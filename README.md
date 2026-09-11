# Checkpoint 4 - Serverless Order Processing Workflow

Projeto desenvolvido para o Checkpoint 4 da disciplina de **Serverless Computing e Arquiteturas Event-Driven**.

O projeto evolui uma aplicação serverless de processamento de pedidos para uma arquitetura orquestrada utilizando **AWS Lambda, AWS Step Functions, Amazon DynamoDB e Amazon SQS**, adicionando uma camada completa de **observabilidade com Amazon CloudWatch Logs e Amazon CloudWatch Metrics**.

## Arquitetura

```text
HTTP POST
   |
   v
Function URL
   |
   v
StartOrder Lambda
   |
   v
AWS Step Functions
   |
   +--> ValidateOrder Lambda
   |
   +--> ProcessOrder Lambda
   |        |
   |        v
   |    DynamoDB
   |    Idempotency
   |
   +--> FinishOrder Lambda
   |
   v
OrderCompleted
```

### Tratamento de falhas

```text
Lambda Task
   |
   +--> Retry
   |
   +--> Catch
         |
         v
      SendToDLQ
         |
         v
      Amazon SQS
      orders-dlq
         |
         v
      OrderFailed
```

### Observabilidade

```text
Lambda Functions
      |
      +--------------------+
      |                    |
      v                    v
CloudWatch Logs      CloudWatch Metrics
Structured JSON            EMF
      |                    |
      +---------+----------+
                |
                v
    Checkpoint4/OrderProcessing
                |
                v
    CloudWatch Dashboard

AWS Step Functions
        |
        v
CloudWatch State Logs
```

## Tecnologias

* AWS Lambda
* AWS Step Functions
* Amazon DynamoDB
* Amazon SQS
* Amazon CloudWatch Logs
* Amazon CloudWatch Metrics
* AWS IAM
* Python 3.12
* pytest
* Git/GitHub

## Estrutura do projeto

```text
cloud-serverless-checkpoint4/
├── lambdas/
│   ├── start_order/
│   │   └── lambda_function.py
│   ├── validate_order/
│   │   └── lambda_function.py
│   ├── process_order/
│   │   └── lambda_function.py
│   └── finish_order/
│       └── lambda_function.py
├── observability/
│   ├── observability.py
│   └── dashboard.json
├── tests/
│   └── test_lambdas.py
├── workflow/
│   └── order-processing-workflow.json
├── process-response.json
├── process-response-duplicate.json
├── .gitignore
├── README.md
└── requirements.txt
```

## Fluxo de processamento

### 1. StartOrder

A Lambda `start-order` recebe um pedido através de uma **Lambda Function URL** HTTP e inicia uma execução do AWS Step Functions.

Exemplo de entrada:

```json
{
  "order_id": "ORDER-001",
  "customer": "Rafaella",
  "amount": 100
}
```

A função inicia a execução do workflow e retorna HTTP `202` juntamente com o ARN da execução do Step Functions.

A Function URL foi validada em ambiente AWS e não é versionada no repositório.

### 2. ValidateOrder

A Lambda `validate-order` valida os dados básicos do pedido:

* `order_id`
* `customer`
* `amount`
* `amount` maior que zero

Pedidos inválidos geram uma exceção e são direcionados para o tratamento de falhas do workflow.

### 3. ProcessOrder

A Lambda `process-order` realiza o processamento do pedido e utiliza o **Amazon DynamoDB** para garantir idempotência.

A tabela utilizada é:

```text
orders-idempotency
```

A chave de partição é:

```text
order_id
```

Quando um pedido é processado pela primeira vez, seu `order_id` é registrado no DynamoDB.

Quando o mesmo `order_id` é processado novamente, a função identifica o pedido como duplicado e não realiza um novo processamento.

Exemplo de resultado para um pedido duplicado:

```json
{
  "processed": false,
  "duplicate": true,
  "order_id": "ORDER-001",
  "message": "Pedido já processado"
}
```

### 4. FinishOrder

A Lambda `finish-order` finaliza o processamento e retorna o status do pedido.

Exemplo:

```json
{
  "order_id": "ORDER-001",
  "status": "SUCCESS",
  "processed": true,
  "duplicate": false,
  "message": "Pedido ORDER-001 finalizado com sucesso."
}
```

## Orquestração com AWS Step Functions

O AWS Step Functions é responsável por controlar a ordem de execução das funções:

```text
ValidateOrder
      |
      v
ProcessOrder
      |
      v
FinishOrder
      |
      v
OrderCompleted
```

Cada etapa recebe a saída da etapa anterior, permitindo que o processamento seja realizado de forma organizada e controlada.

Em caso de erro, o workflow utiliza `Catch` para direcionar a execução para o fluxo de tratamento de falhas.

O fluxo de falha utiliza a etapa `SendToDLQ`, responsável pelo envio da mensagem para a fila SQS `orders-dlq`.

## Retry

As tarefas Lambda do Step Functions possuem política de retry para erros transitórios da infraestrutura AWS.

São considerados:

* `Lambda.ServiceException`
* `Lambda.AWSLambdaException`
* `Lambda.SdkClientException`

Configuração utilizada:

```text
IntervalSeconds: 2
MaxAttempts: 3
BackoffRate: 2
```

Erros de validação, como um valor negativo para `amount`, não são tratados como erros transitórios e seguem para o fluxo de falha.

## Dead-Letter Queue

O workflow possui tratamento de falhas utilizando **Amazon SQS**.

Quando uma tarefa falha e não consegue ser concluída após as tentativas configuradas, o `Catch` direciona a execução para a etapa `SendToDLQ`.

```text
Catch
  |
  v
SendToDLQ
  |
  v
orders-dlq
  |
  v
OrderFailed
```

A mensagem enviada para a DLQ contém os dados do pedido e as informações relacionadas ao erro.

## Idempotência

A idempotência é implementada na Lambda `process-order` utilizando o Amazon DynamoDB.

Quando um pedido é processado pela primeira vez, seu `order_id` é armazenado na tabela.

Quando o mesmo pedido é enviado novamente, o sistema identifica que o `order_id` já existe e evita um novo processamento.

Exemplo:

```text
Primeira execução:
ORDER-001
processed = true
duplicate = false

Segunda execução:
ORDER-001
processed = false
duplicate = true
```

O resultado da segunda execução está registrado em:

```text
process-response-duplicate.json
```

O resultado de processamento normal está registrado em:

```text
process-response.json
```

# Observabilidade

O Checkpoint 4 adiciona observabilidade ao fluxo de processamento utilizando recursos nativos da AWS, principalmente **Amazon CloudWatch Logs** e **Amazon CloudWatch Metrics**.

A instrumentação foi aplicada às quatro funções Lambda:

* `StartOrder`
* `ValidateOrder`
* `ProcessOrder`
* `FinishOrder`

Também foi habilitado o logging do **AWS Step Functions** para registrar os eventos de execução do workflow.

## Logs estruturados

As Lambdas utilizam logs estruturados em JSON, contendo informações como:

* `timestamp`
* `level`
* `service`
* `event`
* `order_id`
* `status`
* `details`

Exemplo:

```json
{
  "timestamp": "2026-09-11T01:25:06.009668+00:00",
  "level": "INFO",
  "service": "ProcessOrder",
  "event": "order_processed",
  "order_id": "ORDER-004",
  "status": "COMPLETED",
  "details": {
    "duration_ms": 59.31
  }
}
```

Os grupos de logs das funções estão disponíveis no CloudWatch:

```text
/aws/lambda/start-order
/aws/lambda/validate-order
/aws/lambda/process-order
/aws/lambda/finish-order
```

As funções Lambda também estão configuradas para utilizar formato de logging JSON no CloudWatch.

## Módulo de observabilidade

A lógica comum de logging e métricas foi centralizada em:

```text
observability/observability.py
```

O módulo disponibiliza as funções:

```text
log_event()
emit_metric()
```

A função `log_event()` gera eventos estruturados em JSON.

A função `emit_metric()` utiliza **Embedded Metric Format (EMF)** para publicar métricas customizadas no CloudWatch.

O namespace utilizado é:

```text
Checkpoint4/OrderProcessing
```

## Métricas customizadas

As funções emitem métricas utilizando **Embedded Metric Format (EMF)**.

Métricas implementadas:

| Métrica              | Serviço       | Finalidade                        |
| -------------------- | ------------- | --------------------------------- |
| `OrdersStarted`      | StartOrder    | Quantidade de pedidos iniciados   |
| `OrdersValidated`    | ValidateOrder | Quantidade de pedidos validados   |
| `OrdersProcessed`    | ProcessOrder  | Quantidade de pedidos processados |
| `DuplicateOrders`    | ProcessOrder  | Quantidade de pedidos duplicados  |
| `ProcessingDuration` | ProcessOrder  | Tempo de processamento            |
| `OrdersFinished`     | FinishOrder   | Quantidade de pedidos finalizados |
| `FinishDuration`     | FinishOrder   | Tempo de finalização              |

As métricas foram verificadas no CloudWatch utilizando o namespace:

```text
Checkpoint4/OrderProcessing
```

## Dashboard do CloudWatch

Foi criado um dashboard do CloudWatch chamado:

```text
Checkpoint4-OrderProcessing
```

O dashboard apresenta:

* quantidade de pedidos iniciados;
* quantidade de pedidos validados;
* quantidade de pedidos processados;
* quantidade de pedidos finalizados;
* quantidade de pedidos duplicados;
* duração média do processamento;
* duração da etapa de finalização.

O arquivo de configuração do dashboard está em:

```text
observability/dashboard.json
```

O dashboard foi publicado no CloudWatch com validação concluída sem erros.

## Observabilidade do Step Functions

O AWS Step Functions foi configurado para enviar logs para:

```text
/aws/vendedlogs/states/order-processing-workflow
```

O nível de logging utilizado foi:

```text
ALL
```

Com isso, é possível acompanhar eventos como:

* `ExecutionStarted`
* `TaskStateEntered`
* `LambdaFunctionStarted`
* `LambdaFunctionSucceeded`
* `LambdaFunctionFailed`
* `TaskSucceeded`
* `ExecutionSucceeded`
* `ExecutionFailed`

Os logs permitem acompanhar o fluxo completo da execução e identificar em qual etapa ocorreu uma falha.

# Evidências de execução

## Processamento normal

Foi executado um pedido utilizando:

```text
order_id: ORDER-004
customer: Rafaella
amount: 150
```

O workflow foi concluído com sucesso:

```text
ExecutionSucceeded
```

Resultado:

```json
{
  "order_id": "ORDER-004",
  "status": "SUCCESS",
  "processed": true,
  "duplicate": false,
  "message": "Pedido ORDER-004 finalizado com sucesso."
}
```

Durante essa execução foram registrados eventos nas quatro Lambdas e no Step Functions.

A métrica `OrdersProcessed` também foi emitida pelo `ProcessOrder`.

A métrica `ProcessingDuration` registrou o tempo de processamento.

## Idempotência

Foi executado novamente um pedido já existente:

```text
order_id: ORDER-001
```

O sistema identificou a duplicidade através do DynamoDB.

Resultado:

```json
{
  "order_id": "ORDER-001",
  "status": "SUCCESS",
  "processed": false,
  "duplicate": true,
  "message": "Pedido ORDER-001 já havia sido processado."
}
```

O `ProcessOrder` registrou o evento:

```text
duplicate_order
```

e emitiu a métrica:

```text
DuplicateOrders
```

Essa execução demonstrou o funcionamento da idempotência e da observabilidade do caminho de pedido duplicado.

## Falha de validação e DLQ

Foi executado um pedido inválido:

```text
order_id: ORDER-ERROR
customer: Rafaella
amount: -100
```

A validação gerou:

```text
ValueError
amount deve ser um número maior que zero
```

O Step Functions registrou o fluxo de falha:

```text
ExecutionStarted
      |
      v
ValidateOrder
      |
      v
LambdaFunctionFailed
      |
      v
SendToDLQ
      |
      v
Amazon SQS
orders-dlq
      |
      v
OrderFailed
      |
      v
ExecutionFailed
```

A mensagem foi enviada para a fila:

```text
orders-dlq
```

A execução terminou com:

```text
status: FAILED
error: OrderProcessingFailed
cause: Order processing failed after retries
```

Essa evidência demonstra o funcionamento conjunto de:

* logs estruturados;
* Step Functions;
* tratamento de exceções;
* Retry/Catch;
* SQS Dead-Letter Queue;
* observabilidade do fluxo de erro.

# Evidências das métricas

As métricas customizadas foram consultadas utilizando o namespace:

```text
Checkpoint4/OrderProcessing
```

Métricas confirmadas:

```text
FinishDuration
OrdersFinished
OrdersValidated
DuplicateOrders
ProcessingDuration
OrdersStarted
OrdersProcessed
```

Consulta utilizada:

```bash
aws cloudwatch list-metrics \
  --namespace "Checkpoint4/OrderProcessing" \
  --query 'Metrics[].MetricName' \
  --output text
```

# Testes automatizados

Os testes automatizados do projeto foram executados após a instrumentação de observabilidade.

Resultado:

```text
9 passed in 0.05s
```

Comando utilizado:

```bash
python3 -m pytest -q
```

O resultado demonstra que a instrumentação de observabilidade não quebrou os testes existentes do Checkpoint 3.

# Otimizações arquiteturais propostas

## 1. Utilizar escrita condicional no DynamoDB

Atualmente, o `ProcessOrder` realiza primeiro um `get_item` para verificar se o pedido existe e posteriormente um `put_item`.

Uma otimização seria utilizar uma operação de escrita condicional com:

```text
ConditionExpression = attribute_not_exists(order_id)
```

Dessa forma, a existência do pedido seria validada atomicamente durante a própria escrita.

Essa abordagem também evita uma janela de condição de corrida caso duas execuções do mesmo pedido ocorram simultaneamente.

### Benefícios

* menor quantidade de chamadas ao DynamoDB;
* menor latência;
* menor custo;
* maior segurança em cenários concorrentes;
* idempotência mais robusta.

## 2. Substituir a Lambda FinishOrder por estado nativo do Step Functions

A `FinishOrder` atualmente possui uma lógica relativamente simples de formatação do resultado final.

Uma alternativa seria substituir essa Lambda por um estado nativo do Step Functions, como `Pass`, utilizando parâmetros e `ResultPath` para montar a saída final.

### Benefícios

* elimina uma invocação Lambda;
* reduz possibilidade de cold start;
* reduz custo;
* simplifica a arquitetura;
* diminui a quantidade de componentes que precisam ser monitorados;
* reduz a superfície operacional da aplicação.

## 3. Configurar retenção dos logs do CloudWatch

Os grupos de logs podem receber uma política explícita de retenção.

Para um ambiente acadêmico ou de testes, uma política de retenção de aproximadamente:

```text
14 dias
```

pode ser adequada.

Isso evita manter logs indefinidamente e permite controlar o crescimento do armazenamento.

### Benefícios

* redução de custo;
* controle do volume de logs;
* política de retenção explícita;
* menor acúmulo de dados desnecessários.

# Segurança

O projeto não contém chaves de acesso, senhas ou credenciais AWS.

As permissões são controladas através de **AWS IAM Roles**.

Não foram incluídos no repositório:

* AWS Access Keys;
* Secret Keys;
* tokens;
* senhas;
* credenciais de serviços.

A Function URL utilizada nos testes não é armazenada no README nem em arquivos versionados.

# Conclusão

O Checkpoint 4 evolui o workflow serverless do Checkpoint 3 adicionando uma camada completa de observabilidade.

A solução permite acompanhar:

```text
Entrada do pedido
      |
      v
StartOrder
      |
      v
ValidateOrder
      |
      v
ProcessOrder
      |
      +----> DynamoDB / Idempotência
      |
      v
FinishOrder
      |
      v
OrderCompleted
```

Em caso de falha:

```text
Lambda Failure
      |
      v
Retry
      |
      v
Catch
      |
      v
SendToDLQ
      |
      v
Amazon SQS
      |
      v
OrderFailed
```

Os eventos das funções são registrados no CloudWatch Logs utilizando JSON estruturado.

As métricas são publicadas através de Embedded Metric Format no namespace:

```text
Checkpoint4/OrderProcessing
```

Os principais indicadores são apresentados no dashboard:

```text
Checkpoint4-OrderProcessing
```

Dessa forma, a arquitetura passa a oferecer não apenas processamento serverless e tratamento de falhas, mas também mecanismos para acompanhar **execuções, erros, duplicidades, volume de pedidos e duração das operações**, permitindo uma análise mais efetiva da saúde do sistema.
