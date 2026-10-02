# fast-sockets

Serviço **sockets** da plataforma de corridas [fast-platform](https://github.com/guilhermelinosp/fast-platform): consome os eventos do Kafka e os entrega em tempo real por Socket.IO aos apps de motoristas e passageiros. O runtime e os eventos de pedido vêm da biblioteca [fast-platform](https://github.com/guilhermelinosp/fast-platform) (`platform`, `env` e `events`).

[![pipeline](https://github.com/guilhermelinosp/fast-sockets/actions/workflows/pipeline.yml/badge.svg)](https://github.com/guilhermelinosp/fast-sockets/actions/workflows/pipeline.yml)
[![pr-check](https://github.com/guilhermelinosp/fast-sockets/actions/workflows/pr-check.yml/badge.svg)](https://github.com/guilhermelinosp/fast-sockets/actions/workflows/pr-check.yml)
[![CodeQL](https://github.com/guilhermelinosp/fast-sockets/actions/workflows/codeql.yml/badge.svg)](https://github.com/guilhermelinosp/fast-sockets/actions/workflows/codeql.yml)

## Início rápido

O serviço lê o `.env` da própria pasta (`cmd/sockets/.env`, ignorado pelo git): copie o `cmd/sockets/.env.example` e ajuste. É necessário o Kafka.

```bash
cp cmd/sockets/.env.example cmd/sockets/.env
cd cmd/sockets && go run -race main.go
```

## Configuração

| Variável | Descrição |
|---|---|
| `HELLNET_SERVICE`, `HELLNET_ENVIRONMENT` | Nome do serviço e ambiente |
| `HELLNET_PORT` | Porta HTTP do servidor Socket.IO |
| `HELLNET_TELEMETRY_ENDPOINT` | Endpoint OTLP/HTTP (Alloy) |
| `KAFKA_BROKERS`, `KAFKA_SECURITY_PROTOCOL` | Conexão Kafka |
| `KAFKA_TOPIC_ORDER_REQUESTED`, `KAFKA_TOPIC_ORDER_ACCEPTED` | Tópicos dos eventos |
| `SOCKET_DRIVERS_NAMESPACE`, `SOCKET_RIDERS_NAMESPACE` | Namespaces Socket.IO (motoristas e passageiros) |
| `SOCKET_URL` | Só para o client de teste (`go run . client`) |

Variáveis já definidas no ambiente têm prioridade sobre o `.env`. Faltando uma variável obrigatória, o processo falha com um erro claro.

## Arquitetura

```text
Kafka ──► fast-sockets ──► Socket.IO /drivers (apps dos motoristas)
                      └──► Socket.IO /riders  (apps dos passageiros, sala order:<orderId>)
```

O servidor emite cada evento com o **nome do tópico** Kafka: `KAFKA_TOPIC_ORDER_REQUESTED` para os motoristas e `KAFKA_TOPIC_ORDER_ACCEPTED` para os passageiros. O passageiro entra na sala do próprio pedido com o evento `order.subscribe` (sala `order:<orderId>`) e recebe o aceite nela.

## Client de teste

`cmd/sockets/client.go` conecta como motorista e como passageiro, loga cada pacote (namespace, evento e ids) e inscreve o passageiro nos pedidos que o motorista recebe. Lê o mesmo `.env` do servidor:

```bash
cd cmd/sockets && go run . client
```

## Estrutura

```text
cmd/sockets           servidor Socket.IO e client de teste
internal/sockets       servidor Socket.IO e consumers Kafka

importa github.com/guilhermelinosp/fast-platform/{platform,env,events}  (runtime, variáveis de ambiente e eventos de pedido)
```

## Desenvolvimento

```bash
go test -race ./...
go vet ./...
golangci-lint run ./...
```

Os hooks do [Lefthook](.lefthook.yml) rodam `gofmt`, `vet`, testes (com e sem `-race`), build, `go mod tidy`, lint, `govulncheck` e o scan de segredos; instale-os uma vez com `lefthook install`. Commits seguem [Conventional Commits](https://www.conventionalcommits.org/).

## CI/CD

| Workflow | Gatilho | O que faz |
|---|---|---|
| `pr-check` | pull request | shellcheck, estratégia de merge e Conventional Commits (`merge-check`), Gitleaks, labels e o gate de qualidade Go (integridade do módulo, vet, testes com race e cobertura, lint, build, dependency review). O `pr-gate` reúne tudo e é o check obrigatório |
| `pipeline` | push na `main` (ignora `.github/**`) ou manual | guarda de semver (bloqueia major automático), tag imutável + GitHub Release, imagem de container |
| `codeql` | diário ou manual | análise estática (CodeQL) |
| `security` | diário ou manual | scans de Gitleaks e Trivy |
| `auto-pr` | push em `feat/**` ou `fix/**` | abre o pull request automaticamente |
| `dependabot-actions-auto-merge` | pull requests do Dependabot | faz auto-merge das atualizações de GitHub Actions |

Os workflows chamam workflows reutilizáveis de [templates](https://github.com/guilhermelinosp/templates), fixados por SHA de commit. O release precisa do secret `HELLNET_ACTIONS_PRIVATE_KEY` e da variável `HELLNET_ACTIONS_CLIENT_ID`.

## Contribuindo e licença

Veja [CONTRIBUTING.md](CONTRIBUTING.md) e [SECURITY.md](SECURITY.md). Licença [Apache 2.0](LICENSE).
