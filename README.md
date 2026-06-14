# AWS Serverless Brokerage - Simulação de Ambiente Real

Este repositório documenta a **jornada de construção de um sistema financeiro distribuído** usando práticas e padrões de arquitetura utilizados por corretoras reais. O objetivo é dominar a stack AWS Serverless através de implementação prática, simulando cenários reais de:

- **Processamento assíncrono** de ordens de compra/venda
- **Alta disponibilidade** e recuperação de desastres
- **Compliance e auditoria** (requisitos CVM/reguladores)
- **Streaming de dados** em tempo real (cotações da bolsa)
- **Segurança bancária** (criptografia, secrets, IAM)
- **Observabilidade** completa (logs, métricas, traces)

### Diagrama

<img width="2221" height="1181" alt="AWS_Serveless (1)" src="https://github.com/user-attachments/assets/4ba5d11d-d5f9-4306-963c-a766ac86e4aa" />

### LocalStack

LocalStack permite simular **todos os serviços AWS localmente**, sem custos e com iteração rápida.

### Metodologia de Estudo

Este é um **projeto evolutivo** dividido em fases incrementais. Cada fase adiciona complexidade e simula novos desafios reais de produção. O código não é descartável - cada melhoria se soma à anterior, construindo um sistema progressivamente mais robusto.

---

## Projeto .NET

O repositório inclui uma API em .NET para simular o backend financeiro, integrando com DynamoDB e outros serviços AWS. O foco principal segue sendo a arquitetura serverless e o uso do LocalStack.

### Como rodar localmente

1. Instale o [.NET 9 SDK](https://dotnet.microsoft.com/download/dotnet/9.0)
2. Execute o script de desenvolvimento que automatiza o LocalStack, aplica o Terraform (se necessário) e inicia a API:

```bash
./dev/start.sh
```

O script realiza automaticamente:

- Sobe o LocalStack via Docker
- Aplica os recursos do Terraform (se ainda não existirem)
- Inicia a aplicação `Brokerage.Api` com `dotnet watch run`

A API deverá ficar disponível em http://localhost:5032 (ou na porta mostrada pelo dotnet).

> **Nota:** Se preferir executar manualmente, você pode subir o LocalStack, aplicar o Terraform e rodar `cd src/Brokerage.Api && dotnet run`.

## Estrutura do Projeto

- **dev**: scripts de desenvolvimento e helpers (ex: `dev/start.sh`)
- **infra**: definições de infra local com `docker-compose.yml` e a pasta `terraform` com os recursos (SQS, SNS, DynamoDB, S3, etc)
- **src**: código-fonte .NET

## Roadmap

### Fase 1: Fundação Básica

**Foco:** Fazer o fluxo funcionar ponta a ponta

- [x] Configuração Docker/LocalStack
- [x] IaC com Shell Script
- [x] Fluxo Assíncrono Simples (Lambda → SQS → Lambda)
- [x] Persistência (DynamoDB/S3)

### Fase 2: Resiliência Financeira

**Foco:** Em corretoras, perder uma mensagem = perder dinheiro do cliente

- [x] **Dead Letter Queue (DLQ):** Se o worker falhar 3x (ex: erro de validação), mover para uma fila de "Rejeitados" para análise manual
- [x] **Idempotência no DynamoDB:** Usar `ConditionExpressions` para garantir que a ordem ORD-123 não seja debitada duas vezes do saldo
- [x] **Retry Policies:** Configurar "Exponential Backoff" na SQS (tentar de novo em 2s, depois 4s, depois 8s...)

### Fase 3: Notificações & Fan-out (Padrão SNS)

**Foco:** Uma ordem executada dispara várias ações simultâneas

- [x] **SNS (Simple Notification Service):** Criar um tópico `OrderEvents`
- [x] **Padrão Fan-out:** Quando o Worker confirmar a compra:
  - [x] Publicar mensagem no SNS
  - [x] SNS entrega para uma SQS de "Notificações" (simulada)
  - [x] SNS entrega para uma SQS de "Auditoria" (Compliance)
  - [x] SNS entrega para uma SQS de "Relatórios" (BackOffice)
- [x] **Lambda de Notificação:** Consumir fila e simular envio de email/SMS ao cliente

### Fase 4: Orquestração de Transações (Step Functions)

**Foco:** Compra de ações não é só um passo, é um fluxo de estados

- [x] **AWS Step Functions:** Substituir a lógica simples do Worker por uma Máquina de Estados:
  1. **Validar Saldo** → Se insuficiente, rejeitar
  2. **Bloquear Saldo** → Debitar do saldo disponível
  3. **Executar Ordem** → Chamar API simulada da B3
  4. **Confirmar Transação** → Gravar no DynamoDB
  5. **Rollback:** Se falhar no passo 3, devolver o dinheiro do passo 2
- [x] **Padrão Saga:** Implementar compensação automática em caso de falha

