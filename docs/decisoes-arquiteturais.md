# Decisões Arquiteturais: SagaFlow

Este documento registra as principais decisões arquiteturais propostas para o SagaFlow, um modelo acadêmico de processamento de compras distribuídas utilizando o padrão Saga por orquestração.

## ADR-001: Saga por orquestração

**Status:** Aceita

**Contexto:** O processamento de uma compra exige etapas ordenadas entre serviços independentes, além de ações compensatórias quando alguma etapa falha.

**Decisão:** Utilizar um Saga Orchestrator persistente, responsável por enviar comandos, processar eventos de resultado e controlar as transições de estado.

**Consequências positivas:**
- Fluxo de execução centralizado e rastreável.
- Compensações explícitas.
- Recuperação a partir do estado persistido.
- Facilidade para incluir novas etapas no processo.

**Consequências negativas:**
- O orquestrador exige alta disponibilidade e monitoramento.
- É necessário controlar concorrência e versionar os fluxos.
- Falhas no orquestrador exigem mecanismos de recuperação.

A persistência do estado e a possibilidade de executar múltiplas instâncias reduzem o risco de indisponibilidade prolongada.

## ADR-002: Mensageria assíncrona

**Status:** Aceita

**Contexto:** Os serviços possuem ciclos de vida e disponibilidades independentes.

**Decisão:** Utilizar um broker de mensagens para transportar comandos e eventos, considerando a possibilidade de entrega pelo menos uma vez.

**Consequências:**
- Os serviços ficam menos dependentes da disponibilidade simultânea uns dos outros.
- Mensagens podem ser duplicadas ou processadas fora da ordem esperada.
- Os consumidores devem validar o estado atual e tolerar duplicidades.
- A ordenação pode ser apoiada por partições, identificadores de pedido e controle de versão, conforme os recursos do broker escolhido.

## ADR-003: Consumidores idempotentes

**Status:** Aceita

**Contexto:** Uma mesma mensagem pode ser entregue mais de uma vez, causando efeitos duplicados se não houver proteção.

**Decisão:** Cada consumidor manterá um registro de mensagens processadas, por exemplo, na tabela `processed_messages`, com unicidade sobre `message_id`.

O registro de deduplicação e a alteração de negócio devem ocorrer na mesma transação local.

**Consequências:**
- Mensagens repetidas não devem duplicar reservas, cancelamentos ou cobranças.
- Respostas anteriores podem ser recuperadas para tratar novas entregas.
- Operações financeiras devem utilizar uma `idempotencyKey` estável quando o gateway oferecer suporte.

A deduplicação por `messageId` não substitui a chave de idempotência da operação de negócio.

## ADR-004: Persistência e recuperação da Saga

**Status:** Aceita

**Contexto:** Uma interrupção pode ocorrer entre etapas, exigindo a retomada do processamento sem reiniciar indevidamente toda a compra.

**Decisão:** Persistir o estado da Saga em uma tabela `saga_instances`, contendo:

| Campo | Finalidade |
|---|---|
| `saga_id` | Identificar a instância da Saga |
| `order_id` | Identificar o pedido relacionado |
| `current_step` | Registrar a etapa atual |
| `status` | Registrar o estado global |
| `attempt_count` | Controlar tentativas |
| `deadline_at` | Registrar o prazo de execução |
| `last_error` | Registrar o último erro conhecido |
| `version` | Controlar concorrência otimista |
| `updated_at` | Registrar a última atualização |

**Decisão complementar:** Adotar o padrão Transactional Outbox para registrar a alteração de estado e o evento a publicar na mesma transação local.

**Consequências:**
- O processamento pode ser retomado após interrupções.
- Um processo de recuperação pode identificar Sagas paradas.
- É necessário monitorar mensagens pendentes de publicação e impedir que instâncias concorrentes avancem a mesma Saga.

## ADR-005: Timeouts e resultados ambíguos

**Status:** Aceita

**Contexto:** Uma operação pode ser concluída mesmo quando sua resposta não chega ao solicitante dentro do prazo.

**Decisão:** Tratar timeouts como resultados potencialmente desconhecidos, e não como confirmação automática de falha.

**Política proposta:**
- Consultar o estado da operação quando houver suporte.
- Repetir operações usando a mesma chave idempotente.
- Manter o pagamento em `UNKNOWN` enquanto o resultado não puder ser confirmado.
- Evitar gerar uma nova chave de pagamento apenas porque a resposta atrasou.
- Utilizar prazos por operação e um prazo global para a Saga.

Os valores de timeout devem ser ajustados de acordo com testes e métricas. Para o exercício, o README apresenta valores ilustrativos.

## ADR-006: Retry e Circuit Breaker

**Status:** Aceita

**Contexto:** Dependências externas podem apresentar falhas temporárias, lentidão ou indisponibilidade.

**Decisão:** Aplicar retentativas limitadas e Circuit Breaker para falhas técnicas transitórias.

**Política proposta:**
- Repetir falhas transitórias, como problemas de conexão e respostas HTTP 5xx.
- Respeitar `Retry-After` quando aplicável.
- Utilizar backoff exponencial com jitter e limite de tentativas.
- Não repetir automaticamente erros permanentes de negócio, como cartão recusado ou estoque insuficiente.
- Abrir o Circuit Breaker após um limiar configurado de falhas técnicas.
- Permitir chamadas de teste para verificar a recuperação da dependência.
- Não manter transações de banco abertas enquanto se aguarda uma resposta de gateway externo.

## ADR-007: Dead-letter queue (DLQ)

**Status:** Aceita

**Contexto:** Algumas mensagens continuam falhando mesmo após as retentativas previstas.

**Decisão:** Encaminhar mensagens que esgotarem a política de tentativas para uma DLQ.

**Procedimento operacional:**
1. Registrar a mensagem, sua origem, as tentativas realizadas e o motivo da falha.
2. Gerar alerta para a equipe responsável.
3. Investigar e corrigir a causa.
4. Reprocessar a mensagem de maneira controlada.
5. Monitorar o resultado do reprocessamento.

O reprocessamento da mesma operação deve preservar a identidade e as chaves idempotentes apropriadas. Uma nova operação de negócio deve possuir identificação própria.

## ADR-008: Compensações de negócio

**Status:** Aceita

**Contexto:** Uma etapa pode falhar depois que etapas anteriores já produziram efeitos em seus respectivos serviços.

**Decisão:** Implementar compensações explícitas e idempotentes, coordenadas pelo orquestrador.

No cenário em que o pagamento é recusado após a reserva de estoque:

1. O orquestrador inicia a compensação.
2. O pedido é cancelado.
3. A reserva de estoque é liberada.
4. A Saga passa para `COMPENSATED` após a confirmação das compensações exigidas.

Se uma compensação falhar temporariamente, ela poderá ser repetida de forma segura. Caso as tentativas se esgotem, a Saga será encaminhada para `MANUAL_REVIEW`, com alerta e trilha de auditoria.

Se um pagamento já tiver sido capturado e uma etapa posterior falhar, pode ser necessário executar `RefundPayment`. Cancelar um pedido não desfaz automaticamente uma transação financeira externa.

## Matriz resumida de riscos e mitigação

| Risco | Mitigação proposta |
|---|---|
| Mensagens duplicadas | Consumidores idempotentes e restrições únicas |
| Falha entre commit e publicação | Transactional Outbox |
| Resposta perdida do gateway | Chave idempotente e reconciliação |
| Saga interrompida | Estado persistido, prazos e recuperação |
| Processamento concorrente | Controle de versão ou bloqueio apropriado |
| Mensagens que falham repetidamente | DLQ, alertas e reprocessamento controlado |
| Dependência externa instável | Timeout, retry limitado e Circuit Breaker |
| Reserva esquecida | TTL, expiração e reconciliação |
| Compensação não concluída | Retentativas, revisão manual e auditoria |

## Referências técnicas

- MICROSOFT. **Saga distributed transactions pattern**. Azure Architecture Center. https://learn.microsoft.com/en-us/azure/architecture/microservices/design/patterns
- MICROSOFT. **Transactional Outbox pattern**. Azure Architecture Center. https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-outbox-cosmos
- RICHARDSON, Chris. **Pattern: Saga**. Microservices.io. https://microservices.io/patterns/data/saga.html
- RICHARDSON, Chris. **Pattern: Transactional Outbox**. Microservices.io. https://microservices.io/patterns/data/transactional-outbox.html

Este documento registra decisões propostas para fins acadêmicos. A implementação deve validar as políticas de retentativa, os prazos, a persistência e as compensações por meio de testes e observabilidade.
