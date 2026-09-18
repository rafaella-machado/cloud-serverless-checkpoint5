# PUC Serverless IA — Checkpoint 5

Projeto desenvolvido para a pós-graduação em **DevOps & Cloud Platform Engineering com IA — PUC Minas**, utilizando serviços serverless da AWS.

Este projeto evolui a arquitetura desenvolvida nos checkpoints anteriores, adicionando **observabilidade, testes automatizados e CI/CD**, com o objetivo de automatizar completamente o processo de integração e implantação das funções serverless.

---

## 1. Objetivo

O objetivo do Checkpoint 5 é implementar um pipeline de **CI/CD (Continuous Integration / Continuous Deployment)** capaz de executar automaticamente:

1. Integração do código no GitHub;
2. Instalação das dependências;
3. Execução dos testes automatizados;
4. Build dos pacotes das funções Lambda;
5. Autenticação segura na AWS;
6. Deploy automático das funções Lambda.

Dessa forma, alterações realizadas no código e enviadas para a branch `master` podem passar pelo processo de validação e implantação de forma automatizada.

---

# 2. Arquitetura

A solução utiliza uma arquitetura baseada em serviços gerenciados e serverless da AWS.

O fluxo principal de processamento de pedidos utiliza:

* AWS Lambda
* AWS Step Functions
* Amazon DynamoDB
* Amazon SQS
* Amazon SNS
* Amazon CloudWatch

Fluxo simplificado:

```text
                    ┌─────────────────────┐
                    │    Start Order      │
                    │    AWS Lambda       │
                    └──────────┬──────────┘
                               │
                               v
                    ┌─────────────────────┐
                    │   Step Functions    │
                    │  Order Processing    │
                    └──────────┬──────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 v                           v
       ┌──────────────────┐        ┌──────────────────┐
       │  Validate Order  │        │   Process Order  │
       │    Lambda        │        │     Lambda       │
       └────────┬─────────┘        └────────┬─────────┘
                │                           │
                └─────────────┬─────────────┘
                              │
                              v
                    ┌─────────────────────┐
                    │    Finish Order     │
                    │       Lambda        │
                    └──────────┬──────────┘
                               │
                               v
                         OrderCompleted
```

Em situações de falha, o workflow possui tratamento de erro, retry e encaminhamento para uma fila DLQ:

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

---

# 3. Funções Lambda

O projeto utiliza quatro funções Lambda principais:

| Função           | Responsabilidade                  |
| ---------------- | --------------------------------- |
| `start-order`    | Inicia o processamento do pedido  |
| `validate-order` | Valida os dados recebidos         |
| `process-order`  | Executa o processamento do pedido |
| `finish-order`   | Finaliza o processamento          |

Os códigos das funções estão organizados no diretório:

```text
lambdas/
├── start_order/
│   └── lambda_function.py
├── validate_order/
│   └── lambda_function.py
├── process_order/
│   └── lambda_function.py
└── finish_order/
    └── lambda_function.py
```

---

# 4. Orquestração com AWS Step Functions

O processamento das etapas é orquestrado utilizando **AWS Step Functions**.

O workflow está armazenado no projeto em:

```text
workflow/order-processing-workflow.json
```

A utilização do Step Functions permite organizar o fluxo de execução, tratamento de erros, retries e encaminhamento para DLQ.

---

# 5. Idempotência

O projeto utiliza o **Amazon DynamoDB** para controle de idempotência.

A tabela utilizada é:

```text
orders-idempotency
```

Esse mecanismo permite identificar pedidos que já foram processados e evitar processamento duplicado.

O cenário de processamento duplicado também possui testes automatizados.

---

# 6. Dead Letter Queue

Para tratamento de falhas que não podem ser resolvidas após as tentativas configuradas, o workflow utiliza uma **Dead Letter Queue (DLQ)** baseada no Amazon SQS.

A DLQ permite preservar as mensagens que falharam para posterior análise e tratamento.

---

# 7. Observabilidade

A solução possui mecanismos de observabilidade utilizando **Amazon CloudWatch**.

Os eventos das funções são registrados utilizando logs estruturados em JSON.

As métricas são publicadas utilizando **Embedded Metric Format (EMF)**.

Namespace utilizado:

```text
Checkpoint4/OrderProcessing
```

Dashboard:

```text
Checkpoint4-OrderProcessing
```

Entre os indicadores acompanhados estão:

* Execuções;
* Erros;
* Duplicidades;
* Volume de pedidos;
* Duração das operações.

O código relacionado à observabilidade está localizado em:

```text
observability/
└── observability.py
```

---

# 8. Testes automatizados

O projeto possui testes automatizados utilizando **pytest**.

Os testes estão localizados em:

```text
tests/test_lambdas.py
```

Os testes abrangem cenários como:

* Validação de pedidos;
* Pedidos válidos;
* Pedidos inválidos;
* Processamento;
* Processamento duplicado;
* Finalização;
* Validação do workflow em JSON.

A suíte atual possui **9 testes automatizados**.

Execução local:

```bash
pytest -v
```

Resultado esperado:

```text
9 passed
```

---

# 9. CI/CD — Checkpoint 5

Nesta etapa foi implementado um pipeline de **Continuous Integration / Continuous Deployment** utilizando **GitHub Actions**.

O workflow está localizado em:

```text
.github/workflows/deploy.yml
```

O pipeline é acionado automaticamente quando ocorre um `push` na branch:

```text
master
```

---

## 9.1 Fluxo do pipeline

```text
Git Push
   |
   v
GitHub Actions
   |
   v
Checkout
   |
   v
Python 3.12
   |
   v
Install Dependencies
   |
   v
Run Tests
   |
   v
Build Lambda Packages
   |
   v
AWS Authentication via OIDC
   |
   v
Deploy AWS Lambda
```

---

# 10. Etapa de Test

O primeiro job do pipeline é responsável pela integração contínua.

O job:

1. Faz checkout do código;
2. Configura Python 3.12;
3. Instala as dependências;
4. Executa os testes automatizados.

Configuração utilizada:

```text
Python 3.12
pytest
```

O job seguinte somente é executado após a conclusão bem-sucedida dos testes.

Fluxo:

```text
Test
  |
  | sucesso
  v
Build and Deploy
```

---

# 11. Build das funções Lambda

Após a aprovação dos testes, o pipeline gera automaticamente os pacotes ZIP das quatro funções:

```text
start-order.zip
validate-order.zip
process-order.zip
finish-order.zip
```

Cada pacote contém:

```text
lambda_function.py
observability/
└── observability.py
```

A construção dos pacotes é realizada automaticamente pelo GitHub Actions.

Os arquivos ZIP são utilizados apenas durante o processo de build/deploy e não são versionados no Git devido à configuração do `.gitignore`.

---

# 12. Deploy automático

Após o build, o pipeline executa o deploy das quatro funções Lambda utilizando a AWS CLI.

Funções atualizadas:

```text
start-order
validate-order
process-order
finish-order
```

O deploy utiliza:

```text
aws lambda update-function-code
```

Dessa forma, uma alteração no código enviada para a branch `master` pode ser automaticamente validada, empacotada e implantada na AWS.

---

# 13. Segurança — AWS OIDC

Para evitar o armazenamento de Access Keys da AWS no GitHub, foi utilizada autenticação através de **OpenID Connect (OIDC)**.

O fluxo de autenticação é:

```text
GitHub Actions
      |
      | OIDC
      v
GitHub OIDC Provider
      |
      v
AWS IAM Role
      |
      v
AWS Lambda
```

A GitHub Actions utiliza uma IAM Role específica para o projeto.

A role possui permissões restritas às quatro funções Lambda utilizadas pelo pipeline.

O identificador da role é armazenado no GitHub Actions como secret:

```text
AWS_ROLE_ARN
```

Nenhuma Access Key ou Secret Access Key da AWS é armazenada no código-fonte.

Também não são versionados arquivos contendo credenciais ou informações sensíveis.

---

# 14. Estrutura do projeto

A estrutura principal do repositório é:

```text
cloud-serverless-checkpoint5/
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── lambdas/
│   ├── start_order/
│   │   └── lambda_function.py
│   ├── validate_order/
│   │   └── lambda_function.py
│   ├── process_order/
│   │   └── lambda_function.py
│   └── finish_order/
│       └── lambda_function.py
│
├── observability/
│   ├── observability.py
│   └── dashboard.json
│
├── tests/
│   └── test_lambdas.py
│
├── workflow/
│   └── order-processing-workflow.json
│
├── requirements.txt
├── .gitignore
└── README.md
```

---

# 15. Pipeline GitHub Actions

O arquivo responsável pela automação é:

```text
.github/workflows/deploy.yml
```

O pipeline possui dois jobs principais:

```text
┌─────────────┐
│    Test     │
└──────┬──────┘
       │
       │ sucesso
       v
┌─────────────────────┐
│ Build and Deploy    │
└─────────────────────┘
```

O segundo job possui uma dependência explícita do primeiro, garantindo que o deploy somente aconteça após a aprovação dos testes automatizados.

---

# 16. Evidência da execução do CI/CD

A execução do pipeline pode ser acompanhada na seção **Actions** do repositório GitHub.

Repositório:

```text
https://github.com/rafaella-machado/cloud-serverless-checkpoint5
```

Workflow:

```text
CI/CD - Deploy Lambdas
```

Para comprovação acadêmica, devem ser registradas evidências da execução contendo:

* Execução do job `Test`;
* Resultado dos testes;
* Execução do job `Build and Deploy`;
* Build dos pacotes Lambda;
* Autenticação AWS;
* Deploy das funções Lambda;
* Resultado final da execução.

### Registro da execução

> **Status:** execução concluída com sucesso.

```text
GitHub Actions Run:
https://github.com/rafaella-machado/cloud-serverless-checkpoint5/actions/runs/35290586528

Resultado:
Pipeline executado com sucesso, incluindo autenticação na AWS via OIDC,
build dos pacotes e deploy automático das funções Lambda:
start-order, validate-order, process-order e finish-order.
```

---

# 17. Conclusão

Com a implementação do Checkpoint 5, o projeto evolui de uma arquitetura serverless com processamento, tratamento de falhas e observabilidade para uma solução que também possui **automação do ciclo de entrega**.

O processo passa a seguir o fluxo:

```text
Alteração no código
       |
       v
Git Push
       |
       v
GitHub Actions
       |
       v
Testes automatizados
       |
       v
Build
       |
       v
Autenticação segura via OIDC
       |
       v
Deploy automático
       |
       v
AWS Lambda
```

Essa abordagem reduz a necessidade de execução manual de comandos de deploy e estabelece uma base de **Continuous Integration e Continuous Deployment** para a aplicação serverless.
