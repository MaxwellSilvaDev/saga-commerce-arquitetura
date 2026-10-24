# Decisões arquiteturais — Saga de e-commerce

## 1. Contexto e objetivo

O fluxo de compra depende de serviços independentes para pedidos, estoque e pagamento. Uma transação ACID única não cobre naturalmente vários bancos e serviços distribuídos. O padrão Saga coordena transações locais e, quando uma etapa falha, executa ações compensatórias de negócio.

## 2. Decisão: Saga orquestrada

Adotamos um **Saga Orchestrator** responsável por controlar a ordem das etapas, persistir o estado, aplicar timeouts e iniciar compensações.

**Justificativa:** o fluxo é conhecido, tem ordem explícita e inclui compensações obrigatórias. A orquestração torna o estado observável e simplifica auditoria e suporte. Na coreografia, cada serviço reage a eventos e a lógica do processo pode ficar distribuída, dificultando a visão de ponta a ponta.

## 3. Transação de sucesso

1. Criar pedido como `PENDING`.
2. Reservar estoque.
3. Autorizar pagamento.
4. Confirmar pedido.
5. Marcar Saga como `COMPLETED`.

## 4. Falha de pagamento e compensações

Se o pagamento for recusado depois da reserva:
1. Registrar a falha no estado da Saga.
2. Executar `CancelOrder` no Order Service.
3. Executar `ReleaseStock` no Inventory Service.
4. Confirmar o resultado das compensações e marcar a Saga como `COMPENSATED`.

As compensações não são rollback técnico de uma transação distribuída; são operações de negócio próprias, com resultado monitorado.

## 5. Idempotência e matriz de decisão

Todo comando/evento deve incluir `sagaId`, `orderId`, `correlationId` e um identificador único de mensagem. Os consumidores mantêm deduplicação persistente, por exemplo usando uma tabela de mensagens processadas com restrição única.

| Situação | Ação recomendada |
|---|---|
| Evento duplicado | Reconhecer e ignorar o efeito repetido; devolver resultado já registrado |
| Erro transitório (timeout, 5xx) | Retry com backoff exponencial e jitter, respeitando limite |
| Recusa definitiva de pagamento | Não repetir autorização automaticamente; iniciar compensação |
| Falha após limite de retries | Enviar para DLQ, alertar e permitir investigação/reprocessamento controlado |
| Resposta tardia após timeout | Validar estado atual e versão da Saga antes de aceitar a transição |
| Compensação repetida | Executar de modo idempotente, sem liberar/rescindir duas vezes |

## 6. Timeouts e estado da Saga

Persistir cada transição antes de publicar o próximo comando. Cada etapa possui prazo configurado e timestamp de início. Um processo de recuperação consulta Sagas em estado intermediário expirado e verifica o resultado real da operação antes de repetir ou compensar.

Estados sugeridos: `STARTED`, `STOCK_RESERVED`, `PAYMENT_PENDING`, `COMPENSATING`, `COMPLETED`, `COMPENSATED`, `MANUAL_INTERVENTION`.

Em timeout ambíguo, não presumir falha definitiva: consultar o serviço de destino por `operationId` ou usar uma chave de idempotência antes de reenviar. Usar controle de versão/compare-and-set para evitar que eventos atrasados regridam o estado.

## 7. Retry e Dead Letter Queue (DLQ)

- Aplicar retry somente a falhas transitórias, com backoff exponencial, jitter e número máximo de tentativas.
- Não repetir indiscriminadamente erros de validação ou recusas definitivas de pagamento.
- Depois do limite, encaminhar a mensagem à DLQ com metadados de erro, tentativas, timestamps e correlação.
- Criar alertas e procedimento operacional para inspecionar, corrigir e reprocessar mensagens sem violar idempotência.
- Evitar ciclos de retry infinitos e tempestades de mensagens.

## 8. Circuit Breaker

Usar Circuit Breaker nas chamadas a dependências com falhas persistentes. No estado aberto, interromper chamadas temporariamente e falhar de forma controlada; após intervalo, permitir chamadas de teste no estado half-open. O Circuit Breaker não substitui timeout, retry ou idempotência.

## 9. Riscos mitigados

- **Duplicidade de mensagens:** chaves de idempotência e deduplicação persistente.
- **Falha parcial:** compensações explícitas e estado durável da Saga.
- **Mensagens fora de ordem:** validação de transições e versão do estado.
- **Indisponibilidade de dependência:** timeouts, retry limitado, Circuit Breaker e DLQ.
- **Falha do orquestrador:** persistência do progresso e retomada após reinício.
- **Compensação falha:** retries controlados, monitoramento e estado de intervenção manual.

## 10. Observabilidade e operação

Propagar `sagaId`, `orderId` e `correlationId` em logs e mensagens. Monitorar duração por etapa, Sagas expiradas, taxa de compensação, retries, volume de DLQ e erros por serviço. Definir alertas e runbooks para investigação e reprocessamento.

## 11. Limites do desenho

Este repositório é um blueprint acadêmico: descreve decisões e fluxos, sem implementar serviços, broker, persistência ou infraestrutura.
