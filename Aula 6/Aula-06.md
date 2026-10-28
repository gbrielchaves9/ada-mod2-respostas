# Template de System Design: Aula 06 - Observabilidade & Diagnóstico em Produção

> **Contexto do Cenário: O Checkout que Trava sem Dar Erro**
>
> Sua equipe responde pelo serviço de **Checkout** do marketplace, que chama **Pagamento** e **Estoque**. Os três rodam em imagens *Distroless*.
>
> 1. **O Incidente:** Na abertura da Black Friday, com **5.000 TPS**, uma versão nova do Checkout entrou em *Canary*. Minutos depois, os painéis continuavam verdes, com latência média de 120 ms, CPU abaixo de 40% e nenhum pico de HTTP 500.
> 2. **O Impacto:** A conversão despencou. Milhares de reclamações chegaram ao suporte, todas com a mesma cena. O cliente clica em "Finalizar Pedido", a tela fica carregando e o aplicativo desiste por timeout.
> 3. **As Restrições:** A imagem não tem `sh` e não há SSH nos servidores. Cada serviço escreve logs em texto livre, do seu jeito, sem um identificador que ligue a chamada do Checkout à chamada que ele fez ao Pagamento. E dado de cliente que vai parar em log continua sujeito à LGPD.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Logs e Correlação

- **Campos do Log:** Que campos todo log do Checkout precisa ter para você encontrar, no meio de milhares de linhas, as da requisição que travou? Liste os campos e diga para que serve cada um.

  > Eu utilizaria **logs estruturados em JSON**, padronizando os campos registrados pelo Checkout e pelos demais microsserviços.
  >
  > Dessa forma, conseguiríamos pesquisar os logs de uma requisição específica, mesmo com milhares de transações por segundo.
  >
  > **Campos que utilizaria:**
  >
  > | Campo | Finalidade |
  > |---|---|
  > | `timestamp` | Identificar quando o evento aconteceu, utilizando UTC. |
  > | `level` | Classificar o registro como INFO, WARN ou ERROR. |
  > | `service` | Identificar qual microsserviço gerou o log. |
  > | `environment` | Identificar o ambiente, como produção ou homologação. |
  > | `version` | Identificar a versão da aplicação, permitindo comparar o Canary com a versão estável. |
  > | `trace_id` | Relacionar os registros de uma mesma requisição entre diferentes serviços. |
  > | `span_id` | Identificar uma operação específica dentro do trace. |
  > | `event` | Identificar o acontecimento, como início do Checkout, chamada ao Pagamento ou timeout. |
  > | `http_method` | Identificar o método HTTP utilizado. |
  > | `http_route` | Identificar o endpoint, utilizando uma rota normalizada, sem informações pessoais. |
  > | `http_status` | Registrar o status HTTP quando houver uma resposta. |
  > | `duration_ms` | Registrar o tempo gasto na operação concluída ou interrompida. |
  >
  > **Exemplo:**
  >
  > ```json
  > {
  >   "timestamp": "2026-10-28T14:00:00Z",
  >   "level": "WARN",
  >   "service": "checkout",
  >   "environment": "production",
  >   "version": "v2-canary",
  >   "trace_id": "abc123",
  >   "span_id": "span456",
  >   "event": "payment_timeout",
  >   "http_method": "POST",
  >   "http_route": "/checkout",
  >   "http_status": 504,
  >   "duration_ms": 3000
  > }
  > ```
  >
  > Esse é um exemplo ilustrativo de estrutura, com identificadores simplificados.
  >
  > **Por quê?**
  >
  > Como os serviços utilizam imagens Distroless e não existe acesso SSH, precisamos conseguir investigar os incidentes por uma plataforma centralizada de observabilidade.
  >
  > Os logs estruturados permitem filtrar registros por serviço, versão, horário e identificador de rastreamento, facilitando encontrar a requisição problemática.

- **O Caminho do Contexto:** Como o mesmo identificador de rastreamento passa do Checkout para o Pagamento e o Estoque numa chamada HTTP?

  > Eu utilizaria **OpenTelemetry** para instrumentar os três microsserviços e propagar automaticamente o contexto de rastreamento entre as chamadas HTTP.
  >
  > Quando uma requisição chegasse ao Checkout, a aplicação aproveitaria um contexto válido recebido ou iniciaria um novo trace.
  >
  > Esse contexto seria propagado para os serviços de Pagamento e Estoque utilizando o cabeçalho HTTP `traceparent`, definido pelo padrão W3C Trace Context.
  >
  > **Exemplo:**
  >
  > 1. O cliente inicia uma compra.
  > 2. O Checkout recebe ou gera um `trace_id`.
  > 3. Ao chamar Pagamento, envia o contexto no cabeçalho HTTP.
  > 4. Pagamento mantém o mesmo `trace_id` e cria seus próprios spans.
  > 5. O mesmo acontece quando o Checkout chama Estoque.
  > 6. Os logs dos três serviços incluem os identificadores de trace e span correspondentes.
  >
  > Dessa forma, podemos pesquisar pelo `trace_id` e acompanhar toda a operação, identificando qual serviço apresentou lentidão.
  >
  > Cada span teria seu próprio `span_id`, permitindo visualizar as diferentes etapas da requisição.
  >
  > Também configuraria a instrumentação dos clientes HTTP utilizados pelo Spring Boot para garantir que o contexto não fosse perdido durante as chamadas entre serviços.
  >
  > **Por quê?**
  >
  > Sem correlação, encontrar a origem de um timeout em milhares de logs seria muito difícil.
  >
  > Com OpenTelemetry, conseguimos relacionar os registros e visualizar quanto tempo o Checkout gastou aguardando Pagamento e Estoque.

- **LGPD nos Logs:** Quais dados de cliente ficam fora de logs e traces? Para os identificadores que precisam aparecer, qual tratamento você aplica e quem pode ler esses registros?

  > Eu aplicaria o princípio da minimização de dados, registrando somente as informações necessárias para diagnosticar o funcionamento da aplicação.
  >
  > **Dados que não registraria em logs e traces:**
  >
  > - CPF e documentos pessoais.
  > - Senhas, tokens de autenticação e credenciais.
  > - Dados bancários e informações completas de cartões.
  > - Nome completo, endereço, telefone e e-mail, quando desnecessários ao diagnóstico.
  > - Payloads completos contendo informações pessoais ou financeiras.
  >
  > Para identificar operações de negócio, utilizaria identificadores técnicos, como um ID interno de pedido, quando realmente necessário.
  >
  > Esses identificadores poderiam ser pseudonimizados, por exemplo, utilizando HMAC com uma chave protegida, permitindo correlacionar operações sem expor diretamente o identificador original.
  >
  > Também evitaria colocar informações pessoais em atributos de traces ou labels de métricas.
  >
  > **Controle de acesso:**
  >
  > Utilizaria RBAC para permitir a leitura dos registros apenas por profissionais autorizados, conforme suas responsabilidades.
  >
  > Também implementaria retenção limitada, mascaramento de dados sensíveis, proteção dos registros e auditoria dos acessos.
  >
  > **Por quê?**
  >
  > Os logs precisam ajudar a diagnosticar incidentes, mas não devem expor informações pessoais dos clientes.
  >
  > Mesmo identificadores pseudonimizados podem continuar sujeitos à LGPD quando existe possibilidade de associação a uma pessoa.

---

## 2. Métricas

- **Visão do Cliente (RED):** Quais métricas do Checkout entram no painel? Explique como você mede a duração para que o travamento de uma parte dos clientes não desapareça na média.

  > Eu utilizaria a metodologia **RED (Rate, Errors e Duration)** para monitorar o comportamento do Checkout e identificar problemas que afetam os clientes.
  >
  > **Rate — Taxa de requisições:**
  >
  > Monitoraria quantas requisições o Checkout recebe por segundo, separando as versões estável e Canary.
  >
  > Isso permitiria identificar quedas inesperadas no volume e comparar o comportamento das versões.
  >
  > **Errors — Taxa de erros:**
  >
  > Monitoraria a porcentagem de requisições que apresentam falhas, incluindo HTTP 5xx, timeouts e cancelamentos observados no gateway.
  >
  > Também acompanharia as falhas nas chamadas aos serviços de Pagamento e Estoque.
  >
  > Não utilizaria apenas HTTP 500 como indicador de falha, porque uma requisição pode ficar travada e o cliente desistir antes que a aplicação retorne uma resposta.
  >
  > **Duration — Duração:**
  >
  > Monitoraria a latência das requisições utilizando histogramas e percentis como p50, p95 e p99, em vez de depender somente da média.
  >
  > O p95 representa o tempo abaixo do qual aproximadamente 95% das requisições observadas foram concluídas, enquanto o p99 representa aproximadamente 99%.
  >
  > Dessa forma, conseguimos identificar uma parcela de clientes enfrentando lentidão, mesmo quando a maioria das requisições continua rápida.
  >
  > **Como evitar que requisições travadas fiquem invisíveis:**
  >
  > Além dos percentis de requisições concluídas, monitoraria:
  >
  > - Quantidade de requisições em andamento.
  > - Tempo que as requisições abertas já estão aguardando.
  > - Timeouts observados no gateway e no aplicativo.
  > - Taxa de conclusão do Checkout.
  > - Diferença de desempenho entre a versão Canary e a versão estável.
  >
  > **Por quê?**
  >
  > No incidente, a média de 120 ms e a CPU abaixo de 40% não representavam a experiência dos clientes.
  >
  > Uma parte das requisições poderia continuar aguardando sem aparecer nas métricas de latência das operações já concluídas.
  >
  > Por isso, acompanharia tanto os indicadores técnicos quanto a taxa de conversão do Checkout.

- **Visão dos Recursos (USE):** Que recursos você monitora? Que sinal acusaria saturação antes do primeiro HTTP 500?

  > Eu utilizaria a metodologia **USE (Utilization, Saturation e Errors)** para monitorar os recursos utilizados pelo Checkout.
  >
  > Os principais recursos seriam:
  >
  > | Recurso | O que monitorar |
  > |---|---|
  > | CPU | Utilização e possíveis limitações por throttling. |
  > | Memória | Consumo, pressão de memória e reinícios por OOM. |
  > | Pool de conexões do PostgreSQL | Conexões ativas, disponíveis e requisições aguardando conexão. |
  > | Conexões HTTP | Conexões em uso e tempo de espera por conexão. |
  > | Threads e execução | Quantidade de tarefas e possíveis bloqueios. |
  > | Integrações externas | Latência, timeouts e quantidade de chamadas concorrentes. |
  >
  > **Exemplo:**
  >
  > Imagine que a CPU esteja em 35%, mas todas as conexões disponíveis no HikariCP estejam ocupadas.
  >
  > Nesse cenário, novas requisições podem ficar esperando uma conexão de banco, mesmo que a aplicação não esteja consumindo muita CPU.
  >
  > Monitoraria métricas como `hikaricp_connections_active`, `hikaricp_connections_idle` e `hikaricp_connections_pending`, de acordo com os nomes expostos pela instrumentação.
  >
  > **Sinal de saturação:**
  >
  > Um aumento contínuo de requisições aguardando conexões, acompanhado de crescimento na latência, indicaria possível saturação antes de surgirem erros HTTP 500.
  >
  > Também acompanharia filas internas, conexões HTTP esgotadas e chamadas externas demoradas.
  >
  > **Por quê?**
  >
  > CPU baixa não significa necessariamente que o serviço esteja saudável. O gargalo pode estar em recursos compartilhados ou dependências externas.

- **Labels:** Quais labels você permite nessas métricas e quais proíbe? Diga o efeito de cada escolha no servidor de métricas.

  > Eu utilizaria labels de baixa cardinalidade no Prometheus, permitindo separar métricas por características relevantes sem gerar uma quantidade excessiva de séries temporais.
  >
  > **Labels permitidas:**
  >
  > - `service`: nome do microsserviço.
  > - `environment`: ambiente de execução.
  > - `version`: versão ou identificação do Canary.
  > - `http_method`: método HTTP.
  > - `http_route`: rota normalizada, como `/pedidos/{id}`.
  > - `status_code`: código HTTP.
  > - `dependency`: serviço externo chamado, como Pagamento ou Estoque.
  >
  > **Labels que evitaria ou proibiria:**
  >
  > - `user_id`.
  > - `customer_id`.
  > - `pedido_id`.
  > - `trace_id`.
  > - CPF, e-mail e outros dados pessoais.
  > - URLs completas contendo identificadores dinâmicos.
  >
  > **Por quê?**
  >
  > No Prometheus, cada combinação diferente de labels cria uma nova série temporal.
  >
  > Se utilizássemos `pedido_id` como label, poderíamos gerar milhares de séries diferentes em pouco tempo, aumentando o consumo de memória, armazenamento e processamento do servidor de métricas.
  >
  > Isso é conhecido como **alta cardinalidade**.
  >
  > Por isso, utilizaria identificadores específicos de requisições nos logs e traces, e não como labels de métricas.
  >
  > Também limitaria a quantidade de valores permitidos por label para evitar crescimento descontrolado das séries.

---

## 3. Traces e Amostragem

- **A Escolha:** ( ) Head Sampling  (X) Tail Sampling  ( ) Combinação das duas

- **Justificativa:** Guardar 100% dos traces a 5.000 TPS custa caro. Qual é a sua regra de retenção, e como ela evita que justamente as requisições que travaram sejam descartadas?

  > Eu utilizaria **Tail Sampling** com OpenTelemetry, priorizando a retenção dos traces que apresentam erros, timeouts ou latência elevada.
  >
  > Diferentemente do Head Sampling, que decide no início da requisição quais traces serão amostrados, o Tail Sampling permite avaliar o comportamento observado antes de decidir o que manter.
  >
  > **Por quê?**
  >
  > Com 5.000 TPS, armazenar todos os traces continuamente poderia gerar um custo elevado de infraestrutura.
  >
  > Ao mesmo tempo, se utilizássemos apenas uma amostragem aleatória no início da requisição, poderíamos descartar justamente as operações que apresentaram travamentos.
  >
  > **Regra de amostragem:**
  >
  > Configuraria uma política inicial para:
  >
  > 1. Priorizar a retenção de traces com erros e timeouts.
  > 2. Priorizar traces com duração superior a 2 segundos, como exemplo de limite inicial.
  > 3. Manter uma amostra de 5% dos traces considerados saudáveis.
  > 4. Identificar a versão da aplicação para comparar o comportamento do Canary e da versão estável.
  >
  > Os percentuais e limites seriam ajustados conforme testes de carga, capacidade de armazenamento e requisitos de diagnóstico.
  >
  > **Exemplo:**
  >
  > Imagine que o Checkout receba 5.000 requisições por segundo.
  >
  > A maioria termina rapidamente, mas algumas ficam aguardando o serviço de Pagamento durante vários segundos.
  >
  > O Tail Sampling priorizaria a retenção dessas requisições lentas, permitindo visualizar os spans de Checkout e Pagamento e identificar onde o tempo foi gasto.
  >
  > Para implementar isso, utilizaria um OpenTelemetry Collector configurado com o processador de Tail Sampling.
  >
  > Também definiria limites de memória, tempo de decisão e capacidade de processamento para evitar que o próprio coletor se tornasse um gargalo.
  >
  > **Cuidado com requisições que nunca terminam:**
  >
  > O Tail Sampling depende dos spans recebidos e de uma janela de decisão. Uma requisição que permanece aberta por muito tempo pode não gerar imediatamente um span finalizado.
  >
  > Por isso, não dependeria exclusivamente dos traces. Também manteria métricas de requisições em andamento, timeouts do gateway e alertas de duração excessiva.
  >
  > Garantiria ainda que o trace não fosse descartado na origem antes de chegar ao coletor, permitindo que a política de Tail Sampling avaliasse as operações.
  >
  > **Trade-off:**
  >
  > O Tail Sampling exige mais memória e processamento no coletor, pois precisa armazenar temporariamente os spans antes de decidir quais serão mantidos.
  >
  > Em compensação, permite priorizar os traces mais relevantes para investigar o tipo de incidente descrito no exercício.

---

## 4. Estratégia de Testes

- **Qual a nossa estratégia para testar isso e garantir que funciona?** Antes da próxima Black Friday, como você prova que os painéis e os alertas mostrariam este incidente?

  > Eu utilizaria uma combinação de **testes de carga, testes de integração e injeção controlada de falhas** para validar a estratégia de observabilidade antes da próxima Black Friday.
  >
  > O objetivo seria reproduzir um cenário em que o Checkout apresenta lentidão e perda de conversão, mas não necessariamente retorna HTTP 500 ou apresenta CPU elevada.
  >
  > **Plano de testes:**
  >
  > **1. Simular a carga da Black Friday**
  >
  > Utilizaria ferramentas como k6 ou Gatling para gerar tráfego representativo, incluindo um cenário próximo de 5.000 transações por segundo em um ambiente com capacidade adequada.
  >
  > Executaria a versão estável e a versão Canary para comparar seus indicadores.
  >
  > **2. Introduzir lentidão nas dependências**
  >
  > Utilizaria WireMock ou Toxiproxy para simular atrasos nas respostas dos serviços de Pagamento e Estoque.
  >
  > Criaria um cenário em que parte das requisições demorasse vários segundos, enquanto outras continuassem respondendo normalmente.
  >
  > Também simularia requisições que terminassem por timeout no cliente ou no gateway, sem necessariamente produzir HTTP 500 no Checkout.
  >
  > **3. Validar as métricas RED**
  >
  > Verificaria se os dashboards mostram aumento no p95 e p99, crescimento de requisições em andamento, timeouts e queda na taxa de conclusão das compras.
  >
  > Confirmaria que o incidente pode ser detectado mesmo que a latência média e o consumo de CPU permaneçam aparentemente normais.
  >
  > **4. Validar as métricas USE**
  >
  > Simularia saturação de pools de conexão e outros recursos compartilhados.
  >
  > Verificaria se métricas de conexões pendentes, filas e utilização permitem identificar o gargalo antes de uma indisponibilidade completa.
  >
  > **5. Validar logs e correlação**
  >
  > Executaria uma compra de teste com um trace conhecido e verificaria se os registros do Checkout, Pagamento e Estoque podem ser encontrados utilizando o mesmo `trace_id`.
  >
  > Também confirmaria que os logs não contêm dados pessoais ou informações sensíveis desnecessárias.
  >
  > **6. Validar os traces**
  >
  > Verificaria se os traces das requisições lentas são preservados pelo Tail Sampling e se os spans permitem identificar qual serviço provocou a demora.
  >
  > Também confirmaria que uma amostra das requisições saudáveis continua disponível para comparação.
  >
  > **7. Validar alertas e rollback**
  >
  > Configuraria alertas para aumento de latência, timeouts, saturação e queda na taxa de conclusão do Checkout.
  >
  > Durante os testes, verificaria se esses alertas são disparados e enviados ao Slack.
  >
  > Também validaria se o Canary pode ser interrompido e revertido automaticamente quando os indicadores ultrapassarem limites definidos.
  >
  > **Critério de sucesso:**
  >
  > Consideraria a estratégia validada quando conseguíssemos reproduzir o incidente e comprovar que:
  >
  > - Os dashboards identificam o problema, mesmo sem aumento de HTTP 500.
  > - Os alertas são enviados corretamente.
  > - Os logs permitem encontrar uma requisição específica.
  > - Os traces mostram em qual serviço ocorreu a lentidão.
  > - A degradação da versão Canary é identificada.
  > - A mitigação ou rollback recupera os indicadores de desempenho e conclusão das compras.
  >
  > **Por quê?**
  >
  > O principal problema do incidente foi a falta de visibilidade sobre a experiência real dos clientes.
  >
  > Por isso, não basta monitorar se o serviço está ligado ou se a CPU está baixa. Precisamos comprovar que a observabilidade consegue detectar lentidão, saturação e falhas que prejudicam a finalização das compras.
  >
  > Com esses testes, conseguimos validar os painéis, os alertas e os mecanismos de diagnóstico antes que o problema aconteça novamente em produção.