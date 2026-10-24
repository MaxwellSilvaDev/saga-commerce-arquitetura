# Arquitetura Saga — E-commerce distribuído

Projeto acadêmico de desenho e modelagem do padrão **Saga** para a migração de um monólito de comércio eletrônico para microsserviços.

## Cenário

O fluxo de compra envolve três domínios:
- **Order Service**: cria e acompanha o pedido.
- **Inventory Service**: reserva e libera estoque.
- **Payment Service**: autoriza ou recusa o pagamento.
- **Saga Orchestrator**: coordena as etapas, persistindo o estado e iniciando compensações quando necessário.
- **Message Broker**: transporte assíncrono, retentativas e Dead Letter Queue (DLQ).

## Decisão arquitetural

Foi escolhido **Saga Orquestrada**. O processo de compra tem uma sequência crítica conhecida e exige visibilidade central do estado, timeouts, observabilidade e compensações explícitas. A orquestração evita espalhar a lógica do processo entre vários serviços e facilita auditoria e recuperação operacional.

## Fluxo de sucesso

1. O cliente solicita a compra.
2. O Order Service cria o pedido com estado `PENDING`.
3. O orquestrador solicita a reserva de estoque.
4. O Inventory Service confirma a reserva.
5. O orquestrador solicita autorização do pagamento.
6. O Payment Service aprova.
7. O pedido passa para `CONFIRMED` e a Saga termina.

## Falha e compensação

Se o pagamento for recusado após a reserva de estoque:
1. A Saga registra a falha do pagamento.
2. O orquestrador solicita **Cancel Order**.
3. O orquestrador solicita **Release Stock**.
4. Após confirmar as compensações, a Saga termina como `COMPENSATED` e o pedido fica `CANCELLED`.

As compensações são comandos de negócio idempotentes: repetir uma compensação não deve produzir efeitos duplicados.

## Estrutura do repositório

- `diagrams/`: fontes editáveis dos diagramas (PlantUML e Graphviz DOT).
- `docs/decisoes-arquiteturais.md`: decisões, riscos, retries, DLQ, circuit breaker, idempotência e timeouts.
- `README.md`: visão geral e instruções.

## Diagramas

- [Visão geral da arquitetura (DOT)](diagrams/saga-architecture-overview.dot)
- [Sequência de sucesso (PlantUML)](diagrams/sequence-success-flow.puml)
- [Sequência de compensação (PlantUML)](diagrams/sequence-compensation-flow.puml)

### Renderização local

PlantUML: use a extensão PlantUML no editor ou execute `plantuml diagrams/sequence-success-flow.puml diagrams/sequence-compensation-flow.puml`.

Graphviz: execute `dot -Tpng diagrams/saga-architecture-overview.dot -o diagrams/saga-architecture-overview.png`.

## Escopo

Este repositório contém **projeto arquitetural e diagramas**, não uma implementação executável dos microsserviços.
