# Kifeito Notification

O **Kifeito Notification** é responsável pelo gerenciamento dos lembretes associados às tarefas agendadas.

O serviço recebe eventos publicados pelo **Kifeito Tasks**, gerencia o ciclo de vida dos lembretes e realiza o envio das notificações por e-mail.

A comunicação com o Tasks é **assíncrona**, utilizando **RabbitMQ**. O estado dos lembretes é persistido em **PostgreSQL** e os e-mails são enviados por **SMTP**.

---
<a id="indice"></a>

## 📋 Índice

1. [🎯 Responsabilidade](#-responsabilidade)
2. [📨 Eventos](#-eventos)
3. [⏰ Criar lembrete](#-criar-lembrete)
4. [📧 Enviar lembrete](#-enviar-lembrete)
5. [🔄 Reagendar lembrete](#-reagendar-lembrete)
6. [❌ Cancelar lembrete](#-cancelar-lembrete)
7. [✅ Cancelar lembrete quando a tarefa for concluída](#-cancelar-lembrete-quando-a-tarefa-for-concluída)
8. [🔓 Reativar lembrete](#-reativar-lembrete)
9. [🔄 Recriar lembrete após reabertura](#-recriar-lembrete-após-reabertura)
10. [💾 Persistência](#-persistência)
11. [🔗 Comunicação](#-comunicação)
12. [🛠️ Tecnologias](#️-tecnologias)
13. [📁 Estrutura](#-estrutura)
14. [⚙️ Configuração](#️-configuração)
15. [🐳 Docker](#-docker)
16. [🧪 Testes](#-testes)
17. [🚫 Fora do escopo da versão 1](#-fora-do-escopo-da-v1)
18. [📄 Licença](#-licença)

---

# 🎯 Responsabilidade

O serviço possui uma responsabilidade específica:

> **Gerenciar lembretes de tarefas agendadas e realizar o envio das notificações por e-mail.**

O **Kifeito Tasks** é responsável pela tarefa e pela sua `scheduledAt` (data de agendamento).

O **Kifeito Notification** é responsável pelo lembrete e pelo seu estado.

| Responsabilidade | Serviço |
|---|---|
| Tarefa | Tasks |
| Data do agendamento | Tasks |
| Lembrete | Notification |
| Data de envio | Notification |
| Status do lembrete | Notification |
| Envio de e-mail | Notification |

⬆️ [Voltar ao índice](#indice)

---

# 📨 Eventos

O Kifeito Tasks publica eventos relacionados ao ciclo de vida da tarefa.

O Kifeito Notification consome esses eventos através do RabbitMQ:

```text
TaskCreated
TaskScheduledDateChanged
TaskCompleted
TaskReopened
TaskCancelled
TaskReactivated
TaskDeleted
```

O Kifeito Notification **não altera tarefas**. Ele apenas reage aos eventos e atualiza seu próprio domínio.

### Exemplo de evento

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "TaskCreated",
  "occurredAt": "2026-09-07T10:00:00Z",
  "data": {
    "taskId": "7f8a9c10-1234-5678-9012-abcdef123456",
    "userId": "9a8b7c6d-5432-1098-7654-fedcba654321",
    "title": "Estudar Spring Boot",
    "description": "Estudar Spring Security",
    "scheduledAt": "2026-09-07T15:00:00"
  }
}
```

O `eventId` identifica unicamente o evento e permite o controle de processamento duplicado.

⬆️ [Voltar ao índice](#indice)

---

# ⏰ Criar lembrete

Quando uma tarefa com `scheduledAt` é criada, o Kifeito Tasks publica o evento:

```text
TaskCreated
```

O evento contém as informações necessárias para o Notification criar o lembrete.

Ao receber o evento, o Notification:

1. Verifica se a tarefa possui `scheduledAt`;
2. Calcula o horário do lembrete;
3. Cria o lembrete;
4. Persiste o lembrete no PostgreSQL.

O lembrete será enviado **1 hora antes** da tarefa.

Exemplo:

```text
scheduledAt  = 15:00
scheduledFor = 14:00
```

⬆️ [Voltar ao índice](#indice)

---

# 📧 Enviar lembrete

O Kifeito Notification verifica os lembretes pendentes que chegaram ao horário de envio.

Quando o horário for atingido:

```text
PENDING
   │
   │ horário atingido
   ▼
SENDING EMAIL
   │
   │ sucesso
   ▼
SENT
```

O e-mail contém as seguintes informações da tarefa:

- título;
- descrição;
- data e horário agendados.

Após o envio bem-sucedido:

```text
status = SENT
sentAt = <data e hora do envio>
```

Exemplo:

```text
Reminder
├── status: SENT
└── sentAt: 2026-09-07T14:00:03
```

Se o envio falhar, o lembrete não deve ser marcado como `SENT`.

⬆️ [Voltar ao índice](#indice)

---

# 🔄 Reagendar lembrete

Quando a data de uma tarefa for alterada, o Kifeito Tasks publica:

```text
TaskScheduledDateChanged
```

O Notification localiza o lembrete associado à tarefa e atualiza seu horário de envio.

Exemplo:

```text
Antes:
scheduledAt  = 15:00
scheduledFor = 14:00

Alteração:
scheduledAt  = 18:00
scheduledFor = 17:00
```

O serviço Notification é responsável por recalcular:

```text
scheduledFor = scheduledAt - 1 hora
```

A `scheduledAt` original da tarefa não é armazenada no lembrete.

A fonte continua sendo o serviço Tasks.

⬆️ [Voltar ao índice](#indice)

---

# ❌ Cancelar lembrete

Quando uma tarefa é cancelada, o Kifeito Tasks publica:

```text
TaskCancelled
```

O Notification localiza o lembrete associado e altera seu estado para:

```text
CANCELLED
```

Exemplo:

```text
PENDING
   │
   │ tarefa cancelada
   ▼
CANCELLED
```

Um lembrete com status `CANCELLED` não pode ser enviado.

⬆️ [Voltar ao índice](#indice)

---

# ✅ Cancelar lembrete quando a tarefa for concluída

Quando uma tarefa é concluída antes do horário do lembrete, o Kifeito Tasks publica:

```text
TaskCompleted
```

O Notification verifica o lembrete associado.

Caso ainda esteja pendente:

```text
PENDING
   │
   │ tarefa concluída
   ▼
CANCELLED
```

O lembrete deixa de ser enviado porque a tarefa já foi concluída.

Caso o lembrete já tenha sido enviado, o status permanece:

```text
SENT
```

⬆️ [Voltar ao índice](#indice)

---

# 🔓 Reativar lembrete

Quando uma tarefa cancelada é reativada, o Kifeito Tasks publica:

```text
TaskReactivated
```

O Notification deve verificar novamente a data de agendamento.

### Data futura

Se a tarefa ainda estiver no futuro:

```text
scheduledAt > now
```

Um novo lembrete poderá ser criado.

Exemplo:

```text
Tarefa:
scheduledAt = 18:00
now         = 15:00

Resultado:
novo lembrete
scheduledFor = 17:00
status       = PENDING
```

### Data passada

Se a data já tiver passado:

```text
scheduledAt < now
```

Nenhum lembrete automático será criado.

Nesse caso, o usuário deverá reagendar a tarefa para receber um novo lembrete.

⬆️ [Voltar ao índice](#indice)

---

# 🔄 Recriar lembrete após reabertura

Quando uma tarefa concluída é reaberta, o Kifeito Tasks publica:

```text
TaskReopened
```

O Notification verifica novamente a data de agendamento.

### Data futura

Se ainda houver tempo para o lembrete:

```text
COMPLETED
     │
     │ reabertura
     ▼
PENDING
     │
     ▼
novo lembrete
```

O novo lembrete será criado com:

```text
scheduledFor = scheduledAt - 1 hora
status = PENDING
```

### Data passada

Se a data da tarefa já tiver passado:

```text
scheduledAt < now
```

Nenhum lembrete será criado automaticamente.

O usuário deverá reagendar a tarefa.

⬆️ [Voltar ao índice](#indice)

---

# 💾 Persistência

O Kifeito Notification possui **banco de dados próprio em PostgreSQL**.

### Entidade `Reminder`

| Campo | Descrição |
|---|---|
| `id` | Identificador do lembrete |
| `taskId` | Tarefa associada |
| `userId` | Usuário que receberá a notificação |
| `scheduledFor` | Horário programado para envio |
| `status` | `PENDING`, `SENT` ou `CANCELLED` |
| `sentAt` | Data e hora do envio |

⬆️ [Voltar ao índice](#indice)

---

# 🔗 Comunicação

```text
┌──────────────┐
│     Tasks    │
└──────┬───────┘
       │
       │ Eventos
       ▼
┌──────────────┐
│   RabbitMQ   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Notification │
└──────┬───────┘
       │
    ┌──┴────┐
    ▼       ▼
┌────────┐ ┌──────┐
│Postgres│ │ SMTP │
└────────┘ └──────┘
```

- **RabbitMQ:** comunicação assíncrona entre Tasks e Notification.
- **PostgreSQL:** persistência dos lembretes.
- **SMTP:** envio dos e-mails.

⬆️ [Voltar ao índice](#indice)

---

# 🛠️ Tecnologias

| Tecnologia | Utilização |
|---|---|
| Java 17 | Linguagem |
| Spring Boot | Framework |
| Spring Data JPA | Persistência |
| PostgreSQL | Banco de dados |
| Spring AMQP | Integração com RabbitMQ |
| RabbitMQ | Mensageria |
| Spring Mail | Envio de e-mail |
| Thymeleaf | Template HTML |
| Spring Scheduling | Processamento dos lembretes |
| Gradle | Build |
| Docker | Containerização |
| GitHub Actions | CI |

⬆️ [Voltar ao índice](#indice)

---

# 📁 Estrutura

Estrutura inicial:

```text
kifeito-notification
│
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com.jefferson.notification
│   │   │       │
│   │   │       ├── controller
│   │   │       │   └── EmailController.java
│   │   │       │
│   │   │       ├── dto
│   │   │       │   └── TaskDTO.java
│   │   │       │
│   │   │       ├── entity
│   │   │       │   └── Reminder.java
│   │   │       │
│   │   │       ├── enums
│   │   │       │   └── ReminderStatus.java
│   │   │       │
│   │   │       ├── exception
│   │   │       │   └── EmailException.java
│   │   │       │
│   │   │       ├── repository
│   │   │       │   └── ReminderRepository.java
│   │   │       │
│   │   │       ├── service
│   │   │       │   ├── EmailService.java
│   │   │       │   └── ReminderService.java
│   │   │       │
│   │   │       ├── messaging
│   │   │       │   └── TaskEventConsumer.java
│   │   │       │
│   │   │       ├── scheduler
│   │   │       │   └── ReminderScheduler.java
│   │   │       │
│   │   │       └── NotificationApplication.java
│   │   │
│   │   └── resources
│   │       ├── templates
│   │       │   └── notification.html
│   │       │
│   │       └── application.yaml
│
├── .github
│   └── workflows
│       └── pull-request.yml
│
├── .gitignore
├── Dockerfile
├── build.gradle
├── gradlew
├── gradlew.bat
└── README.md
```

⬆️ [Voltar ao índice](#indice)

---

# ⚙️ Configuração

As configurações de banco de dados e SMTP são fornecidas por variáveis de ambiente.

```text
DATABASE_URL
DATABASE_USERNAME
DATABASE_PASSWORD

SMTP_USERNAME
SMTP_PASSWORD

EMAIL_SENDER
EMAIL_SENDER_NAME
```

> O arquivo `.env` não deve ser versionado.

⬆️ [Voltar ao índice](#indice)

---

# 🐳 Docker

Cada microsserviço possui seu próprio `Dockerfile` e pode ser executado junto aos demais serviços através do Docker Compose.

O Docker garante um ambiente de execução padronizado, facilita a configuração e permite executar os serviços de forma isolada e reproduzível.

⬆️ [Voltar ao índice](#indice)

---

# 🧪 Testes

O serviço terá testes unitários para as regras de negócio e testes de integração para suas principais integrações.

- **Testes unitários:** regras de negócio e ciclo de vida dos lembretes.
- **Testes de integração:** PostgreSQL, RabbitMQ e envio de e-mail.

⬆️ [Voltar ao índice](#indice)

---

# 🚫 Fora do escopo da versão 1

Para manter a complexidade proporcional à necessidade do sistema, a versão 1 não possui:

- SMS;
- WhatsApp;
- Push Notification;
- múltiplos canais;
- preferências de notificação;
- API pública de lembretes.

⬆️ [Voltar ao índice](#indice)

---

# 📄 Licença

O Kifeito está sendo desenvolvido inicialmente para uso próprio e para um grupo limitado de usuários.

Apesar do uso inicial restrito, o projeto está sendo desenvolvido com arquitetura, práticas e estrutura voltadas para um produto comercial, podendo futuramente ser disponibilizado de forma mais ampla.

O código-fonte, a aplicação, a identidade visual, a documentação e demais componentes do projeto são de propriedade do próprio autor.

A utilização, cópia, modificação, distribuição ou comercialização de qualquer parte do projeto depende de autorização expressa do detentor dos direitos.

⬆️ [Voltar ao índice](#indice)
