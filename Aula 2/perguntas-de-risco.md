# Perguntas de Risco: Aula 02 - Checkout e Pagamento Externo

As respostas usam o `system-design.md`: threads virtuais, timeout de 2 s no cliente HTTP, disjuntor, retry só quando a chamada não chegou, bulkhead de 200, rate limit da cota do contrato e `Idempotency-Key` com `UNIQUE`. A regra: o dinheiro sai uma vez, e falha nunca aparece como compra aprovada.

Cada pergunta responde direto, depois o porquê, depois um exemplo e o teste.

---

## 1. E se o parceiro ficar lento sem chegar a dar timeout (1,8 s) e milhares de compras ficarem esperando ao mesmo tempo?

**Resposta.** A lentidão não segura o banco nem vira fila infinita no parceiro. A conexão do banco não atravessa a chamada, o bulkhead corta em 200, e o disjuntor abre por chamada lenta, sem contar a rejeição do próprio bulkhead.

**Por quê.** Threads virtuais aguentam a espera, então o servidor não trava por falta de thread. O limite real é o pool de conexões (HikariCP, 10 por padrão). Se a transação ficasse aberta durante os 1,8 s, cada compra seguraria uma conexão e a loja inteira esperaria o banco. Abaixo do timeout de 2 s não há erro: um disjuntor que só olhasse falha ficaria fechado e continuaria empurrando carga para um parceiro já lento. A rejeição do bulkhead (`BulkheadFullException`) acontece antes da chamada. Contá-la como falha abriria o circuito por causa da nossa própria fila.

**Exemplo.** 3.000 compras/s × 1,8 s ≈ 5.400 esperas. Com a transação em milissegundos, as 10 conexões dão conta. A chamada 201 ao parceiro espera no máximo 50 ms, recebe `503` com `Retry-After`, sem linha de pagamento e sem somar no disjuntor. Com 60% das últimas 50 chamadas acima de 1,5 s, o circuito abre e responde `503` em milissegundos por 30 s.

**Como testamos.** Integração com Spring e WireMock.

1. WireMock a 1,8 s com `200`, 30 chamadas. Depois de 20 lentas, o `CircuitBreakerRegistry` está `OPEN`. A seguinte volta `503` em menos de 100 ms e o WireMock não recebe chamada nova.
2. Bulkhead de 5 vagas e 20 compras juntas. No máximo 5 chegam ao WireMock. As outras recebem `503` em ~50 ms, sem cobrança e sem abrir o circuito.
3. Pool de 2 conexões, WireMock a 1,8 s, 10 compras e um `GET /pedidos/{id}` no meio. A consulta responde em menos de 200 ms e `hikaricp.connections.pending` fica em 0.
4. Contraprova: `@Transactional` em volta da chamada ao parceiro. O teste 3 fica vermelho.

---

## 2. E se o disjuntor abrir e, quando o parceiro volta, todos os clientes reenviarem ao mesmo tempo?

**Resposta.** A cota do contrato é um número só para a frota. Cada réplica recebe `cota / réplicas` no RateLimiter local, e o gateway segura o teto cheio. O excesso toma `503` na nossa porta. O retry espalha no tempo. A mesma chave impede cobrança dupla.

**Por quê.** O RateLimiter do Resilience4j conta dentro do processo. Dez réplicas com a cota inteira mandam dez vezes o contrato e derrubam o parceiro de novo. O meio-aberto (5 chamadas de teste por réplica) não segura esse pico. `RequestNotPermitted` também não conta como falha do disjuntor: o parceiro não foi chamado.

**Exemplo.** Contrato de 500/s, 10 réplicas, 50/s em cada uma e 500/s no gateway. 6.000 reenvios no mesmo segundo: 500 seguem, o resto recebe `503` sem chegar ao parceiro. Os retries internos esperam ~200 ms, ~400 ms, ~800 ms, com uma parte sorteada, e só repetem `429`, `503` ou conexão que nem abriu. Timeout de leitura vira pagamento `PENDENTE`, não uma segunda cobrança.

**Como testamos.**

1. Limite de 10/s e 100 compras juntas. O WireMock recebe cerca de 10 naquele segundo. O resto volta `503` com `Retry-After`, sem cobrança dupla e sem abrir o circuito.
2. WireMock devolve `429` duas vezes e `200` na terceira. Três tentativas, a mesma `Idempotency-Key`, intervalos crescentes e, em 20 rodadas, intervalos que não são idênticos.
3. Circuito aberto, stub volta a `200`. No meio-aberto passam exatamente 5 chamadas antes de `CLOSED`.
4. Contraprova: `enable-randomized-wait: false`. As tentativas voltam a cair no mesmo instante.

O `Retry-After` devolvido ao app ainda é fixo. Sorteá-lo (20 a 40 s) espalharia também a volta dos clientes. Está registrado no desenho e não muda a divisão da cota.

---

## 3. E se duas requisições com a mesma `Idempotency-Key` chegarem ao mesmo tempo em réplicas diferentes?

**Resposta:**

Eu utilizaria uma combinação de lock distribuído com Redis e uma constraint `UNIQUE` no PostgreSQL para evitar que duas requisições com a mesma chave de idempotência executassem o pagamento simultaneamente.

O Redis seria uma camada adicional para controlar a concorrência entre os pods, enquanto o PostgreSQL seria responsável pela garantia persistente de unicidade.

Também enviaria a mesma `Idempotency-Key` ao parceiro de pagamentos para evitar cobranças duplicadas.

**Por quê?**

Como temos múltiplas réplicas do Checkout, duas requisições podem chegar ao mesmo tempo em pods diferentes.

Apenas consultar o banco antes de inserir não seria suficiente, porque ambas poderiam verificar que a chave ainda não existe e tentar realizar o pagamento.

Com o Redis, somente um pod conseguiria adquirir o lock naquele momento. Além disso, a constraint `UNIQUE` no PostgreSQL garantiria que apenas uma requisição registrasse a chave, mesmo se o lock expirasse ou apresentasse alguma falha.

**Exemplo:**

Imagine que o Pod A e o Pod B recebam simultaneamente uma requisição com a chave `K`.

1. O Pod A consegue adquirir o lock no Redis.
2. O Pod B não consegue adquirir o lock e não inicia uma cobrança.
3. O Pod A tenta registrar a chave `K` no PostgreSQL, com status `EM_PROCESSAMENTO`.
4. Após confirmar o registro, o Pod A envia a cobrança ao parceiro utilizando a mesma chave.
5. Quando o resultado é conhecido, atualiza o pagamento e registra a resposta.
6. Se outra requisição chegar com a mesma chave, o sistema consulta o registro existente e devolve o resultado, caso já esteja concluído.

Se a chave ainda estiver em processamento, a API poderá retornar HTTP 409. Se a mesma chave for utilizada com um conteúdo diferente, retornará HTTP 422.

Caso ocorra um timeout após o envio da cobrança, manteríamos o pagamento como `PENDENTE` e utilizaríamos reconciliação com o parceiro, evitando assumir que a cobrança falhou.

Também utilizaríamos um prazo de recuperação para impedir que operações permaneçam indefinidamente em `EM_PROCESSAMENTO`.

**Como testamos:**

Utilizaria testes de integração com Spring Boot, PostgreSQL, Redis e WireMock.

- Simularia duas requisições simultâneas com a mesma chave, enviadas a instâncias diferentes do Checkout.
- Verificaria que apenas uma cobrança é realizada e que existe somente um registro da chave no PostgreSQL.
- Repetiria uma requisição já concluída e verificaria se a resposta armazenada é devolvida sem realizar outra cobrança.
- Simularia a expiração do lock Redis durante o processamento e verificaria se a constraint `UNIQUE` continua impedindo que outra requisição assuma a mesma operação.
- Simularia a indisponibilidade do Redis e verificaria se a estratégia de contingência mantém a proteção contra duplicidade.
- Simularia um timeout após o envio ao parceiro e verificaria se a reconciliação recupera o resultado sem gerar uma segunda cobrança.

**Trade-off:**

O Redis adiciona complexidade e uma nova dependência à arquitetura. Por isso, utilizaria o lock distribuído como uma camada opcional de controle de concorrência, especialmente se os testes demonstrassem contenção relevante.

A garantia principal continuaria sendo a constraint `UNIQUE` no PostgreSQL, combinada com a idempotência do parceiro de pagamentos.
