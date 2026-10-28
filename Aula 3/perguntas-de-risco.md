# Perguntas de Risco: Aula 03 - Banco de Dados do Checkout

O desenho usa duas transações curtas (a chamada ao parceiro fica no meio, fora de transação), lock otimista no pagamento, `UPDATE` condicional no estoque e reserva que muda uma vez só. A regra: estoque nunca negativo, pagamento fecha uma vez, falha nunca vira aprovado.

Cada pergunta responde direto, depois o porquê, depois um exemplo e o teste.

---

## 1. E se a resposta atrasada do parceiro e o job fecharem o mesmo pagamento `PENDENTE` ao mesmo tempo, um como `APROVADO` e o outro como `EXPIRADO`?

**Resposta:**

Eu utilizaria Optimistic Locking com a anotação `@Version` do JPA para controlar as atualizações concorrentes do pagamento.

Cada pagamento teria um campo de versão. Quando dois processos tentassem atualizar o mesmo registro, apenas um conseguiria concluir a operação com a versão esperada. O outro encontraria um conflito e precisaria consultar novamente o estado atual, sem sobrescrever a alteração anterior.

**Por quê?**

Se o job de expiração e a confirmação do parceiro tentarem atualizar o mesmo pagamento simultaneamente, podemos ter uma condição de corrida.

Sem controle de concorrência, um processo poderia marcar o pagamento como `APROVADO` e outro sobrescrevê-lo como `EXPIRADO`, provocando inconsistências entre pagamento e estoque.

O `@Version` impede que uma atualização baseada em uma versão desatualizada seja confirmada.

Também manteria a atualização do pagamento e a alteração da reserva na mesma transação do banco, garantindo que um conflito provoque rollback das duas operações.

**Exemplo:**

Imagine um pagamento com:

- Status: `PENDENTE`
- Versão: `3`
- Reserva: `ATIVA`

Dois processos consultam a versão 3.

O processo A recebe a confirmação do pagamento e atualiza o registro para `APROVADO`, incrementando a versão para 4.

O processo B tenta atualizar o mesmo registro para `EXPIRADO`, ainda utilizando a versão 3.

Como a versão já mudou, o JPA identifica o conflito e lança uma exceção de Optimistic Locking.

O processo B deve então reler o pagamento e verificar a regra de negócio aplicável, em vez de sobrescrever o estado `APROVADO`.

**Como testamos:**

Utilizaria testes de integração com Spring Boot, JPA e PostgreSQL, usando Testcontainers.

1. Criaria um pagamento `PENDENTE`, com versão 3 e reserva ativa.
2. Executaria duas transações concorrentes, uma tentando aprovar e outra tentando expirar o pagamento.
3. Verificaria que apenas uma das atualizações é confirmada.
4. Confirmaria que a outra operação recebe uma exceção de conflito de versão e realiza rollback.
5. Verificaria que o estoque permanece consistente e que a reserva não é confirmada ou liberada duas vezes.
6. Repetiria o cenário diversas vezes para identificar possíveis condições de corrida.

---

## 2. E se o pagamento for recusado ou expirar depois que a última unidade já foi reservada?

**Resposta:**

Eu utilizaria um UPDATE condicional no banco de dados para garantir que uma unidade reservada fosse devolvida ao estoque apenas uma vez.

Cada reserva teria um status, como `ATIVA`, `CONFIRMADA` ou `LIBERADA`.

Quando um pagamento fosse recusado ou expirasse, o sistema tentaria alterar a reserva de `ATIVA` para `LIBERADA`. Somente se essa atualização fosse realizada com sucesso, a quantidade disponível no estoque seria incrementada.

**Por quê?**

Como diferentes processos podem tentar liberar a mesma reserva, por exemplo, um job de expiração e o processamento de uma recusa de pagamento, precisamos evitar que a mesma unidade seja devolvida duas vezes.

Se simplesmente incrementássemos a quantidade disponível sempre que uma solicitação de liberação chegasse, poderíamos criar um estoque maior do que a quantidade real de produtos.

O UPDATE condicional garante que apenas a primeira transição de `ATIVA` para `LIBERADA` autorize a devolução.

Também executaria a atualização do pagamento, a alteração da reserva e a devolução da unidade dentro da mesma transação do PostgreSQL, para garantir a consistência entre essas operações.

**Exemplo:**

Imagine que temos uma televisão com apenas uma unidade disponível.

1. O cliente inicia uma compra e a unidade é reservada. O estoque disponível passa de 1 para 0.
2. O parceiro de pagamentos recusa a cobrança.
3. O sistema registra o pagamento como `RECUSADO`.
4. A reserva passa de `ATIVA` para `LIBERADA`.
5. Como a transição ocorreu com sucesso, o estoque disponível volta para 1.

Se um job tentar liberar essa mesma reserva novamente, o UPDATE condicional não encontrará uma reserva com status `ATIVA`.

Nesse caso, nenhuma unidade adicional será devolvida ao estoque.

Uma implementação SQL simplificada seria:

```sql
UPDATE reserva
SET status = 'LIBERADA'
WHERE id = :reservaId
  AND status = 'ATIVA';
```

A aplicação verificaria a quantidade de linhas atualizadas. Apenas se fosse igual a 1, executaria:

```sql
UPDATE estoque
SET disponivel = disponivel + 1
WHERE produto_id = :produtoId;
```

As duas operações devem participar da mesma transação.

**Como testamos:**

Utilizaria testes de integração com Spring Boot, PostgreSQL e Testcontainers, além de WireMock para simular o parceiro de pagamentos.

1. Criaria um produto com uma unidade disponível.
2. Simularia a reserva da última unidade e verificaria que o estoque disponível passou para 0.
3. Simularia uma recusa do pagamento e verificaria que a reserva foi liberada e o estoque voltou para 1.
4. Tentaria liberar a mesma reserva novamente e verificaria que o estoque continuou em 1.
5. Executaria duas tentativas simultâneas de liberação e verificaria que apenas uma incrementou o estoque.
6. Simularia uma falha antes do commit e verificaria que a transação foi revertida, sem deixar a reserva e o estoque inconsistentes.

Também manteria uma restrição `CHECK (disponivel >= 0)` no banco para impedir quantidades negativas.

## 3. E se a migration do índice rodar durante o pico e travar as gravações?

**Resposta:**

Eu utilizaria `CREATE INDEX CONCURRENTLY` no PostgreSQL para criar o índice sem bloquear as operações normais de escrita na tabela durante sua construção.

Também planejaria executar essa migration antes da Black Friday, preferencialmente em um período de menor movimento, reduzindo o risco de impacto no desempenho da aplicação.

**Por quê?**

A criação convencional de um índice no PostgreSQL pode bloquear operações de `INSERT`, `UPDATE` e `DELETE` até que o índice seja construído.

Em uma tabela com milhões de registros, isso poderia provocar lentidão, acúmulo de requisições e saturação do pool de conexões do banco, prejudicando o Checkout.

Com `CREATE INDEX CONCURRENTLY`, o PostgreSQL permite que as operações normais de leitura e escrita continuem acontecendo enquanto o índice é construído.

Porém, essa operação pode levar mais tempo, consumir recursos e não pode ser executada dentro de uma transação explícita.

**Exemplo:**

Imagine uma tabela `pedidos` com 40 milhões de registros. Precisamos criar um índice composto para melhorar as consultas por cliente e status do pedido.

Utilizaria:

```sql
CREATE INDEX CONCURRENTLY idx_pedidos_cliente_status
ON pedidos (customer_id, order_status);
```

Caso estivéssemos utilizando Flyway, configuraria essa migration para ser executada fora de uma transação, utilizando `executeInTransaction=false`.

Também verificaria se o índice foi criado corretamente. Se a operação fosse interrompida e deixasse um índice inválido, planejaria sua remoção segura e uma nova tentativa de criação.

**Como testamos:**

Utilizaria testes de integração com PostgreSQL e Testcontainers.

1. Criaria uma tabela `pedidos` com uma quantidade elevada de registros.
2. Executaria a migration com `CREATE INDEX CONCURRENTLY` enquanto outra rotina realizasse operações de `INSERT`.
3. Verificaria se as operações de escrita continuaram funcionando durante a criação do índice, monitorando suas latências.
4. Confirmaria que o índice foi criado e está válido, consultando o catálogo do PostgreSQL.
5. Utilizaria `EXPLAIN ANALYZE` para avaliar se o índice melhorou o desempenho das consultas esperadas.
6. Simularia uma interrupção durante a criação e validaria o procedimento de recuperação de um índice inválido.

**Trade-off:**

O `CREATE INDEX CONCURRENTLY` reduz o bloqueio das operações normais de escrita, mas pode demorar mais e consumir recursos adicionais do banco.

Por isso, mesmo utilizando essa estratégia, eu evitaria executar migrations pesadas durante o pico de acessos e monitoraria seu impacto durante a execução.
