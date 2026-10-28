# Perguntas de Risco: Aula 04 - Saga da Compra

Os estados são os do [apêndice](apendice-maquina-de-estados.md).

Três serviços, três bancos, mensagens pelo outbox e pelo Kafka. A ordem é reservar (15 min), cobrar o Pix, confirmar a reserva. A regra: o cliente não fica cobrado sem produto e sem devolução.

Cada pergunta responde direto, depois o porquê, depois um exemplo e o teste.

---
## 1. E se a confirmação do Pix chegar depois de o pedido ter sido cancelado por falta de pagamento?

**Resposta:**

Eu manteria o pedido cancelado e iniciaria uma ação compensatória da Saga para realizar o estorno do Pix.

Como a reserva já expirou e o produto foi liberado para outras compras, não temos mais a garantia de que aquela unidade esteja disponível.

Ao receber a confirmação atrasada do pagamento, o orquestrador identificaria que o pedido já foi cancelado, registraria a necessidade de estorno e enviaria um comando `EstornarPagamento` para o serviço de Pagamentos.

**Por quê?**

Em uma arquitetura de microsserviços, as operações acontecem de maneira distribuída e assíncrona. Por isso, uma confirmação de pagamento pode chegar depois de o pedido ter sido cancelado.

Não podemos simplesmente ignorar esse evento, porque o cliente realmente realizou o pagamento.

Também não seria seguro reabrir automaticamente o pedido, pois a unidade reservada pode já ter sido comprada por outro cliente.

O estorno seria a ação compensatória responsável por manter a consistência financeira da operação.

Para evitar estornos duplicados, utilizaria uma chave de idempotência associada ao identificador do pagamento.

**Exemplo:**

Imagine que um cliente reservou a última televisão disponível até as 23h55.

1. Às 23h55, o prazo expira e o sistema cancela o pedido por falta de pagamento.
2. O serviço de Estoque libera a unidade reservada.
3. Outro cliente realiza a compra dessa unidade.
4. Às 23h55min20s, o PSP confirma que o Pix do primeiro cliente foi recebido.
5. O orquestrador identifica que o pedido está em `CANCELADO_SEM_COBRANCA` e inicia a compensação, passando para `ESTORNANDO`.
6. O serviço de Pagamentos solicita o estorno utilizando uma chave idempotente vinculada ao pagamento.
7. Após a confirmação do PSP, o pedido passa para `CANCELADO_COM_ESTORNO`.

Caso o PSP não confirme imediatamente a devolução, manteríamos o estorno pendente e utilizaríamos mecanismos de retry e reconciliação para acompanhar o resultado.

**Como testamos:**

Utilizaria testes de integração com Spring Boot, Kafka, PostgreSQL, Testcontainers e WireMock para simular os serviços e o PSP.

1. Criaria um pedido com reserva de curta duração e pagamento ainda não confirmado.
2. Simularia a expiração da reserva e verificaria se o pedido foi cancelado e o estoque liberado.
3. Enviaria posteriormente um evento de confirmação do Pix.
4. Verificaria se o orquestrador identificou o pagamento atrasado e publicou o comando de estorno.
5. Confirmaria que o PSP recebeu apenas uma solicitação efetiva de estorno para aquele pagamento, mesmo diante de mensagens duplicadas.
6. Verificaria se, após a confirmação da devolução, o pedido chegou a `CANCELADO_COM_ESTORNO` e a reserva permaneceu `LIBERADA`.

---

## 2. E se o PSP devolver o Pix, a resposta se perder, e o Pagamento tentar devolver de novo?

**Resposta:**

Eu utilizaria uma combinação de idempotência, reconciliação, Retry com Exponential Backoff e DLQ para garantir que o estorno seja realizado apenas uma vez e que nenhuma operação fique sem acompanhamento.

Se o serviço de Pagamentos recebesse um timeout após solicitar o estorno, não consideraria automaticamente que a operação falhou.

Primeiro, consultaria o PSP utilizando a mesma chave de idempotência para verificar se a devolução já foi realizada.

Se o PSP confirmasse o estorno, atualizaria o estado do pagamento. Caso ainda não fosse possível confirmar o resultado, manteria a operação pendente e realizaria novas tentativas de consulta com intervalos progressivos.

**Por quê?**

Em sistemas distribuídos, uma operação pode ser concluída com sucesso mesmo quando sua resposta não chega à aplicação.

Se realizássemos uma nova solicitação de estorno com uma chave diferente, poderíamos provocar uma devolução duplicada.

Por isso, utilizaria uma chave de idempotência associada ao identificador do pagamento, garantindo que todas as tentativas representem a mesma operação financeira.

Também manteria o mesmo identificador lógico do comando para impedir que mensagens duplicadas gerassem novos estornos.

O Exponential Backoff reduziria a frequência das tentativas durante uma indisponibilidade do PSP.

Caso as tentativas de reconciliação fossem esgotadas, encaminharia a operação para revisão, utilizando uma DLQ e alertas para acompanhamento.

**Exemplo:**

Imagine que o serviço de Pagamentos solicite um estorno ao PSP.

1. O PSP realiza a devolução em 1 segundo.
2. A resposta se perde e nossa aplicação recebe um timeout após 3 segundos.
3. O serviço mantém o estorno como `SOLICITADO`, sem iniciar uma nova devolução.
4. Uma consulta ao PSP, utilizando a mesma chave, confirma que o dinheiro já foi devolvido.
5. O serviço registra o estorno como `CONCLUIDO`.
6. O pedido passa para `CANCELADO_COM_ESTORNO`.

Se o PSP estivesse indisponível para consulta, aplicaríamos intervalos progressivos de 1 s, 5 s, 30 s e 2 minutos, conforme o cenário do exercício.

Após cinco tentativas sem confirmação, o pedido passaria para `ESTORNO_EM_REVISAO`, com encaminhamento para DLQ e geração de um alerta.

Se a situação permanecesse sem confirmação por 24 horas, o pedido poderia passar para `ESTORNO_FALHOU`, conforme a política estabelecida, mantendo uma rotina de reconciliação para identificar uma eventual devolução posterior.

O pedido só seria considerado `CANCELADO_COM_ESTORNO` quando o PSP confirmasse a devolução.

**Como testamos:**

Utilizaria testes de integração com Spring Boot, PostgreSQL, Kafka, Testcontainers e WireMock para simular o PSP.

1. Simularia um estorno realizado com sucesso, mas com a resposta HTTP atrasada além do timeout da aplicação.
2. Verificaria se o serviço iniciou a reconciliação e identificou a devolução já realizada.
3. Confirmaria que houve apenas um estorno efetivo, mesmo diante de timeouts e mensagens duplicadas.
4. Simularia respostas HTTP 503 durante a reconciliação e verificaria os intervalos progressivos das novas tentativas.
5. Simularia cinco falhas consecutivas e verificaria se o pedido passou para `ESTORNO_EM_REVISAO`, com registro na DLQ e geração de alerta.
6. Verificaria que o pedido não foi marcado como `CANCELADO_COM_ESTORNO` sem confirmação do PSP.

---

## 3. E se a tela "Meus Pedidos" receber "concluído" antes de "pago"?

**Resposta:**

Eu utilizaria versionamento de eventos para garantir que a tela "Meus Pedidos" sempre apresentasse o estado mais recente do pedido.

Cada evento publicado no Kafka teria um campo `version`, representando a versão do estado do pedido.

O consumidor responsável por atualizar a projeção no Redis verificaria a versão recebida e só realizaria a atualização caso ela fosse maior que a versão armazenada.

Também garantiria que essa comparação e a atualização acontecessem de forma atômica no Redis, evitando conflitos entre diferentes instâncias do consumidor.

**Por quê?**

Em uma arquitetura de microsserviços com comunicação assíncrona, mensagens podem ser reprocessadas ou chegar fora de ordem.

Se atualizássemos o Redis simplesmente com o último evento recebido, poderíamos sobrescrever um status mais recente com outro mais antigo.

Isso poderia fazer a tela do cliente voltar de `CONCLUIDO` para `PAGO`, causando confusão e prejudicando a experiência de compra.

Com o versionamento, garantimos que eventos antigos não façam o estado da projeção retroceder.

**Exemplo:**

Imagine que o serviço de Pedidos publique dois eventos:

- `PagamentoAprovado`, versão 3, status `PAGO`.
- `PedidoConcluido`, versão 4, status `CONCLUIDO`.

Por causa de um reprocessamento, o consumidor recebe primeiro a versão 4.

O Redis armazena:

```json
{
  "pedido_id": "123",
  "status": "CONCLUIDO",
  "version": 4
}
```

Dois segundos depois, o consumidor recebe o evento `PagamentoAprovado`, versão 3.

Antes de atualizar, verifica que a versão recebida é menor que a armazenada.

Como `3 < 4`, o evento é ignorado e o pedido permanece como `CONCLUIDO`.

Essa verificação seria executada atomicamente no Redis, por exemplo, utilizando um script Lua.

Também utilizaria o `pedido_id` como chave de particionamento no Kafka para favorecer a ordem dos eventos relacionados ao mesmo pedido.

**Como testamos:**

Utilizaria testes unitários e testes de integração com Kafka e Redis, utilizando Testcontainers.

1. Enviaria primeiro o evento `PedidoConcluido`, versão 4, e depois `PagamentoAprovado`, versão 3.
2. Verificaria que a projeção permaneceu com status `CONCLUIDO` e versão 4.
3. Enviaria o mesmo evento duas vezes e confirmaria que não houve alteração indevida.
4. Executaria consumidores concorrentes tentando atualizar a mesma projeção e verificaria que apenas a versão mais recente prevaleceu.
5. Consultaria a API utilizada pela tela "Meus Pedidos" para confirmar que o status não voltou de `CONCLUIDO` para `PAGO`.
