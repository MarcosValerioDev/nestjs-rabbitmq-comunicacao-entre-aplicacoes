# Comunicação entre aplicações com RabbitMQ e NestJS

Monorepo NestJS com duas aplicações que se comunicam de forma assíncrona pelo RabbitMQ:

- **`api`**: aplicação HTTP (porta `3333`) que **publica** mensagens, direto numa fila ou numa exchange.
- **`process`**: worker sem HTTP que **consome** as filas `email` e `notifications` e processa as mensagens.

A comunicação usa o [`amqplib`](https://github.com/amqp-node/amqplib) diretamente, encapsulado num módulo NestJS (`RabbitmqModule`) presente em cada aplicação.

## Arquitetura

```
                    ┌──────────────────────────── RabbitMQ ────────────────────────────┐
                    │                                                                  │
 GET /queue ──────► │  sendToQueue ─────────────────────────────► [ fila: email ]  ────┼──► EmailService
                    │                                                     ▲            │
 GET /exchange ───► │  publish ─► ( amq.direct ) ── routing key "process" ┤            │
     (api)          │                                                     ▼            │    (process)
                    │                                           [ fila: notifications ]┼──► NotificationService
                    └──────────────────────────────────────────────────────────────────┘
```

- **`/queue`**: publica direto na fila `email`. Só o `EmailService` recebe.
- **`/exchange`**: publica na exchange `amq.direct` com a routing key `process`. Como as duas filas estão ligadas (bind) a essa routing key, a mensagem é copiada para `email` **e** `notifications` (fan-out por routing key).

## Tecnologias

- Node.js 18 + TypeScript
- NestJS 10 (modo monorepo)
- RabbitMQ 3.8 (com painel de gerenciamento)
- `amqplib` / `amqp-connection-manager`
- Docker / Docker Compose / Dev Container
- PostgreSQL 16 (já está no compose, mas ainda **não é usado** pelo código, veja os comentários `// save on database`)

## Estrutura

```
apps/
├── api/                          # Produtor (HTTP :3333)
│   └── src/
│       ├── main.ts
│       ├── app.module.ts
│       ├── app.controller.ts     # rotas /default-nest, /queue, /exchange
│       ├── app.service.ts        # publica nas filas/exchange
│       ├── http.http             # requisições prontas (REST Client)
│       └── rabbitmq/             # provider (conexão/canal) + service
└── process/                      # Consumidor (worker)
    └── src/
        ├── main.ts
        ├── process.module.ts
        ├── email.service.ts          # consome a fila "email"
        ├── notification.service.ts   # consome a fila "notifications"
        └── rabbitmq/                 # provider (conexão/canal) + service
```

### Módulo RabbitMQ

- **`RabbitMQProvider`** (`RABBITMQ_PROVIDER`): lê `RABBITMQ_URI` via `ConfigService`, abre a conexão e cria um `Channel`.
- **`RabbitmqService`** expõe:
  - `publishInQueue(queue, message)`: `channel.sendToQueue`
  - `publishInExchange(exchange, routingKey, message)`: `channel.publish`
  - `consume(queue, callback)`: consome e faz `ack` depois de executar o callback

Os nomes aceitos são tipados: filas `'email' | 'notifications'`, exchange `'amq.direct'`, routing key `'process'`.

## Pré-requisitos

- Docker e Docker Compose
- (Opcional) VS Code com a extensão **Dev Containers**

## Configuração

Crie o arquivo `.env` a partir do exemplo:

```bash
cp .env.example .env
```

| Variável       | Padrão                             | Descrição                     |
| -------------- | ---------------------------------- | ----------------------------- |
| `RABBITMQ_URI` | `amqp://admin:admin@rabbitmq:5672` | URI de conexão com o RabbitMQ |

> O host `rabbitmq` só funciona dentro da rede do Docker. Se for rodar as aplicações direto na sua máquina, use `amqp://admin:admin@localhost:5672`.

## Subindo o ambiente

### Opção 1: Dev Container (recomendado)

Abra o projeto no VS Code e escolha **"Dev Containers: Reopen in Container"**. Os serviços `api`, `rabbitmq` e `postgres` sobem juntos e o editor abre dentro do container `api`.

### Opção 2: Docker Compose

```bash
docker compose up -d
docker compose exec api bash
```

O container `api` fica ocioso (`sleep infinity`); as aplicações são iniciadas manualmente de dentro dele.

### Serviços

| Serviço  | Endereço                 | Credenciais     |
| -------- | ------------------------ | --------------- |
| API      | http://localhost:3333    | —               |
| RabbitMQ | amqp://localhost:5672    | `admin`/`admin` |
| RabbitMQ (painel) | http://localhost:15672 | `admin`/`admin` |
| Postgres | localhost:5432           | `admin`/`admin` |

## Preparando o RabbitMQ (obrigatório)

O código **não cria** as filas nem os binds (não há `assertQueue`/`bindQueue`). Antes do primeiro uso, crie tudo no painel em http://localhost:15672:

1. **Queues and Streams → Add a new queue**: crie as filas `email` e `notifications`.
2. **Exchanges → `amq.direct` → Bindings**: adicione dois binds com a routing key `process`:
   - `amq.direct` → `email` (routing key `process`)
   - `amq.direct` → `notifications` (routing key `process`)

Sem as filas, o `process` falha ao consumir e as mensagens publicadas pela `api` são descartadas.

## Executando

Dentro do container (ou localmente, depois de ajustar o `.env`):

```bash
npm install

# terminal 1: consumidor
npm run start:dev -- process

# terminal 2: produtor
npm run start:dev -- api
```

### Testando

Use o arquivo [apps/api/src/http.http](apps/api/src/http.http) (extensão REST Client) ou `curl`:

```bash
curl http://localhost:3333/default-nest   # rota padrão, não faz nada
curl http://localhost:3333/queue          # publica só na fila "email"
curl http://localhost:3333/exchange       # publica em "email" e "notifications"
```

Saída esperada no terminal do `process`:

```
{"message":"Email enviado para queue"}
{"message":"Enviando informações para exchange (email and notifications)"}
{"message":"Enviando informações para exchange (email and notifications)"}
```

## Scripts

| Script                       | Descrição                                     |
| ---------------------------- | --------------------------------------------- |
| `npm run start:dev -- <app>` | Inicia `api` ou `process` em modo watch       |
| `npm run start:debug -- <app>` | Inicia em modo debug + watch                |
| `npm run build -- <app>`     | Compila a aplicação para `dist/`              |
| `npm run start:prod:api`     | Executa a `api` compilada                     |
| `npm run start:prod:process` | Executa o `process` compilado                 |
| `npm run lint`               | ESLint com correção automática                |
| `npm run format`             | Prettier                                      |
| `npm test` / `npm run test:e2e` | Testes unitários / e2e                     |

## Pontos de atenção e próximos passos

- **Filas e binds manuais**: dá para automatizar com `channel.assertQueue` + `channel.bindQueue` na inicialização.
- **Código duplicado**: o módulo `rabbitmq/` é idêntico nas duas aplicações; poderia ir para uma `lib` compartilhada do monorepo (`libs/rabbitmq`).
- **Ack incondicional**: `consume` faz `ack` mesmo se o callback lançar erro, então a mensagem se perde. Vale tratar erros com `nack`/dead-letter queue.
- **Persistência**: o Postgres está provisionado, mas o salvamento (`// save on database`) ainda não foi implementado.
- **Testes e2e**: os arquivos em `apps/*/test` são os gerados pelo Nest e não refletem as rotas atuais.
