# Prompts para testes de integração de uma API de pedidos

Este repositório registra dois prompts usados com uma LLM e os casos de teste de integração elaborados a partir das respostas. O cenário é didático: o repositório não contém uma aplicação executável nem uma especificação real de API.

## Cenário e premissas

Uma API de pedidos recebe `POST /api/orders` do front-end e coordena três componentes:

1. **API de pedidos**: valida a solicitação, persiste o pedido e coordena o fluxo.
2. **Serviço de estoque**: recebe `POST /inventory/reservations` para reservar os itens.
3. **Gateway de pagamento**: recebe `POST /payments/authorizations` para autorizar o valor.

Para tornar os casos verificáveis, os prompts declaram estas premissas, que devem ser confirmadas ou substituídas pelo contrato real do sistema antes de automatizar os testes:

- Uma solicitação válida retorna `201` com o identificador e o estado do pedido.
- SKU inexistente ou estoque insuficiente retorna `409`; o gateway de pagamento não deve ser chamado.
- Dados de entrada inválidos retornam `400`; nenhum serviço downstream deve ser chamado.
- Uma recusa do pagamento retorna `422`, e a reserva de estoque é liberada.
- Falha ou resposta inválida de um serviço downstream não deve produzir um pedido marcado como pago.
- `Idempotency-Key` identifica uma tentativa lógica: repetir a mesma solicitação não duplica reserva nem autorização.
- Os testes verificam tanto a resposta HTTP quanto as interações e os estados persistidos; não dependem de serviços externos reais.

## Prompt 1 — Geração de cenários funcionais e de contrato

**Objetivo:** cobrir o fluxo normal e regras de negócio entre os componentes.

> Você é especialista em testes de integração de APIs. Considere o seguinte fluxo: o front-end chama `POST /api/orders`; a API de pedidos valida os dados, solicita uma reserva a `POST /inventory/reservations`, autoriza o pagamento em `POST /payments/authorizations` e persiste o pedido.
>
> Use estas premissas: pedido válido retorna `201`; entrada inválida retorna `400`; SKU inexistente ou estoque insuficiente retorna `409` e não chama o gateway; pagamento recusado retorna `422` e libera a reserva; uma chave `Idempotency-Key` evita pedidos, reservas e autorizações duplicados. Não invente outros endpoints ou regras: marque como “a confirmar” qualquer comportamento que dependa de contrato não informado.
>
> Gere de 5 a 7 casos de teste de integração, incluindo sucesso, validação, regra de estoque, pagamento e idempotência. Para cada caso, informe: ID, objetivo, dados/setup, chamada HTTP, resultado esperado (status e campos relevantes), interações downstream esperadas e estado persistido. Priorize casos independentes, observáveis e automatizáveis. Responda em uma tabela e evite testes unitários de métodos isolados.

### Casos gerados e revisados

| ID | Cenário e preparação | Ação | Resultado e verificações de integração |
|---|---|---|---|
| F-01 | Dois itens disponíveis; pagamento aprovado; pedido válido | `POST /api/orders` com chave inédita | `201`; resposta contém ID e estado aprovado. Uma reserva e uma autorização são feitas; pedido persistido com os itens e total corretos. |
| F-02 | Quantidade igual a zero; serviços disponíveis | Enviar pedido inválido | `400`; nenhum pedido é persistido e estoque e gateway não são chamados. |
| F-03 | SKU não cadastrado no serviço de estoque | Enviar pedido com esse SKU | `409`; não há autorização de pagamento e não é persistido pedido pago. |
| F-04 | Estoque disponível; gateway configurado para recusar | Enviar pedido válido | `422`; autorização recusada, reserva liberada e pedido não fica marcado como pago. |
| F-05 | Pedido já processado com uma chave de idempotência | Repetir a mesma chamada e o mesmo corpo com a mesma chave | Resposta representa o mesmo pedido; não ocorre uma segunda reserva ou autorização. |
| F-06 | Pedido já processado com uma chave de idempotência | Reutilizar a chave com corpo diferente | Rejeição conforme o contrato de idempotência (status exato **a confirmar**); nenhum segundo pedido ou efeito downstream é criado. |

## Prompt 2 — Geração de cenários de falha e recuperação

**Objetivo:** explorar indisponibilidade, timeouts, respostas inesperadas e efeitos duplicados.

> Continue usando o cenário e as premissas abaixo. API de pedidos: `POST /api/orders`; dependências: serviço de estoque (`POST /inventory/reservations`) e gateway (`POST /payments/authorizations`). A entrada inválida retorna `400`; SKU sem estoque retorna `409`; pagamento recusado retorna `422` e dispara liberação da reserva; `Idempotency-Key` evita efeitos duplicados. Em caso de timeout de pagamento, o resultado pode ser ambíguo: o gateway aceita a mesma chave de idempotência e a API não deve criar uma segunda autorização em uma repetição.
>
> Produza 5 a 7 testes de integração para falhas e recuperação: timeout/indisponibilidade de cada dependência, resposta downstream malformada, timeout de pagamento com resultado ambíguo e repetição da requisição. Para cada teste, descreva a falha injetada, a chamada inicial e eventual retry, status/resposta esperado, chamadas e efeitos que não podem se duplicar, além do estado final ou pendente do pedido e da reserva. Não presuma retry automático, prazo de timeout, política de compensação ou status HTTP não especificados: identifique essas decisões como “a confirmar”. Use dependências simuladas controláveis e não faça chamadas reais a serviços externos.

### Casos gerados e revisados

| ID | Falha injetada / ação | Resultado e verificações de integração |
|---|---|---|
| R-01 | Serviço de estoque indisponível ao receber a reserva | A API não chama o gateway nem marca o pedido como pago. Verificar status de erro e política de persistência/retry (**status e política a confirmar**). |
| R-02 | Serviço de estoque excede o timeout configurado | A chamada termina dentro do limite esperado; não há autorização de pagamento. Verificar se retry é automático ou fica a cargo do cliente (**a confirmar**). |
| R-03 | Gateway retorna recusa após a reserva | `422`; API solicita liberação da reserva. Confirmar que o estado do pedido reflete a recusa e não há autorização aprovada. |
| R-04 | Gateway excede o timeout, embora possa ter processado a autorização; repetir com a mesma chave | Não criar pedido, reserva ou autorização duplicados. Verificar estado pendente/consulta posterior e resposta da repetição conforme o contrato (**estados e status exatos a confirmar**). |
| R-05 | Serviço de estoque retorna corpo inválido ou incompatível com o contrato | A API não prossegue para pagamento; retorna erro controlado, sem pedido pago. O status exato deve seguir o contrato (**a confirmar**). |
| R-06 | Autorização é recusada e a chamada de liberação da reserva falha | O pedido não fica pago; a falha de compensação fica observável e passível de recuperação, sem ser ignorada. Definir alerta, retry e estado persistido (**a confirmar**). |

## Reflexão sobre os prompts e os resultados

O primeiro prompt, orientado a cenários funcionais e de contrato, produziu uma base mais direta para validar regras de negócio e a ordem das chamadas entre componentes. A tabela com campos obrigatórios tornou os casos observáveis: além do status HTTP, cada teste verifica chamadas downstream e estado persistido. A cobertura incluiu caminhos válidos, inválidos, de estoque, de pagamento e repetição.

O segundo prompt ampliou a diversidade para falhas de infraestrutura e recuperação. A instrução explícita para considerar timeout de pagamento ambíguo trouxe uma situação que um prompt genérico tende a omitir: o serviço pode ter autorizado mesmo sem a resposta chegar. Também evidenciou uma limitação importante: sem contrato para retry, compensação e estados intermediários, a LLM não deve inventar comportamentos. Por isso, os casos sinalizam essas decisões como “a confirmar”, em vez de tratar uma escolha arbitrária como requisito.

Os ajustes que tornaram os resultados mais úteis foram:

- fornecer os componentes, endpoints e sequência do fluxo;
- explicitar premissas de negócio e idempotência;
- pedir campos verificáveis (resposta, chamadas, persistência e efeitos não duplicados);
- separar cobertura funcional da cobertura de falhas;
- pedir dependências simuladas e proibir chamadas externas reais;
- instruir a LLM a identificar lacunas contratuais, em vez de preencher lacunas com suposições.

Como copiloto, a GenAI acelera a exploração de cenários e ajuda a revelar casos de borda, mas não substitui a validação do contrato, a análise de risco nem a execução dos testes no sistema. Antes de transformar estes exemplos em uma suíte automatizada, a equipe deve confirmar os itens marcados “a confirmar”, adaptar os dados aos contratos reais e verificar se os testes são determinísticos. A revisão humana é essencial para evitar que uma resposta plausível da LLM seja confundida com comportamento implementado.
