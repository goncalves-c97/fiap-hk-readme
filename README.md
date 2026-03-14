# fiap-hk-readme

Repositório que explica sobre a solução desenvolvida no Hackaton, além de agregar os caminhos dos repositórios desenvolvidos.

Trabalho desenvolvido na Pós graduação em Software Architecture na FIAP, para o Hackaton.

URL do vídeo de apresentação no YouTube:

<URL vídeo>

Repositórios envolvidos:

fiap-hk-infra <br />
Repo: https://github.com/goncalves-c97/fiap-hk-infra <br />

fiap-hk-auth-ms <br />
Repo: https://github.com/goncalves-c97/fiap-hk-auth-ms <br />
SonarQube: https://sonarcloud.io/summary/new_code?id=goncalves-c97_fiap-hk-auth-ms <br /><br />
fiap-hk-video-upload-ms <br />
Repo: https://github.com/goncalves-c97/fiap-hk-video-upload-ms <br />
SonarQube: https://sonarcloud.io/summary/new_code?id=goncalves-c97_fiap-hk-video-upload-ms <br /><br />
fiap-hk-video-processing-ms <br />
Repo: https://github.com/goncalves-c97/fiap-hk-video-processing-ms <br />
SonarQube: https://sonarcloud.io/summary/new_code?id=goncalves-c97_fiap-hk-video-processing-ms <br /><br />
fiap-hk-notification-ms <br />
Repo: https://github.com/goncalves-c97/fiap-hk-notification-ms <br />
SonarQube: https://sonarcloud.io/summary/new_code?id=goncalves-c97_fiap-hk-notification-ms <br />

# Architecture Documentation

## Overview

Esta solucao implementa uma plataforma de processamento de videos baseada em microsservicos, com foco em escalabilidade horizontal, resiliencia a picos e persistencia de dados.

Os servicos existentes sao:

- `auth-ms`: autenticação de usuários e emissão de JWT.
- `video-upload-ms`: upload do video, consulta de status e publicação do evento de processamento.
- `video-processing-ms`: processamento assíncrono do video e atualização do status.
- `notification-ms`: envio de notificações em caso de sucesso ou erro.

A infraestrutura alvo foi desenhada para AWS usando:

- `ECS Fargate` para execução dos containers.
- `RDS SQL Server` para persistência relacional.
- `S3` para armazenamento de arquivos.
- `RabbitMQ` em ECS para mensageria assíncrona.
- `ALB` para exposição das APIs HTTP.
- `Secrets Manager` para segredos e configurações sensíveis.
- `CloudWatch Logs` para observabilidade básica.

## Arquitetura



## Responsabilidades Lógicas

### `auth-ms`

- Realiza autenticacao de usuario e senha.
- Emite token JWT compartilhado com os demais servicos.
- Persiste dados de usuario no banco `AuthDb`.

### `video-upload-ms`

- Recebe o upload do video autenticado.
- Persiste o metadado do video no banco `VideoUploadDb`.
- Armazena o arquivo original no `S3`.
- Publica o evento `video-uploaded` no `RabbitMQ`.
- Expoe listagem e consulta de status do processamento.

### `video-processing-ms`

- Consome a fila `video-uploaded`.
- Busca o registro do video no `VideoUploadDb`.
- Atualiza o status do processamento no mesmo banco usado pelo upload.
- Le o video do `S3`, extrai frames e publica o artefato processado no `S3`.
- Publica `video-processed` ou `processing-error`.

### `notification-ms`

- Consome as filas `video-processed` e `processing-error`.
- Envia notificacoes ao usuario, hoje por SMTP/e-mail.

## Topologia de Infraestrutura

### Network

- Uma `VPC` principal.
- Sub-redes publicas para o `ALB`.
- Sub-redes privadas de aplicacao para `ECS`.
- Sub-redes privadas de dados para `RDS`.
- `NAT Gateway` para saida controlada dos workloads privados.

### Compute

- Um cluster `ECS Fargate`.
- Dois servicos HTTP publicos:
  - `auth-ms`
  - `video-upload-ms`
- Tres servicos internos:
  - `video-processing-ms`
  - `notification-ms`
  - `RabbitMQ`

### Dados

- `AuthDb`: persistencia do microsservico de autenticacao.
- `VideoUploadDb`: persistencia compartilhada entre upload e processamento.
- `S3`: armazenamento do video original e do zip final.

## Runtime Flows

### 1. Autenticação

1. O cliente chama `auth-ms`.
2. `auth-ms` valida usuario/senha em `AuthDb`.
3. `auth-ms` devolve um JWT.

### 2. Upload e Processamento

1. O cliente autenticado envia o video para `video-upload-ms`.
2. O servico grava o arquivo em `S3`.
3. O servico persiste o registro em `VideoUploadDb` com status inicial.
4. O servico publica `video-uploaded` no `RabbitMQ`.
5. `video-processing-ms` consome a mensagem.
6. O worker atualiza status no `VideoUploadDb`, processa o arquivo e publica o resultado no `S3`.
7. O worker atualiza o status final no mesmo banco.
8. O worker publica `video-processed` ou `processing-error`.

### 3. Notificação

1. `notification-ms` consome eventos finais.
2. Em caso de erro ou sucesso, envia comunicacao ao usuario.

## Modelo de segurança

- As APIs sao protegidas por JWT.
- O segredo JWT e demais segredos operacionais ficam no `Secrets Manager`.
- O banco de dados não é público.
- O acesso ao `RDS` e ao `RabbitMQ` e feito por security groups internos.
- As imagens são consumidas diretamente do `Docker Hub`.

## Escalabilidade e Resiliência

### Escalabilidade

- `auth-ms` e `video-upload-ms` podem escalar horizontalmente em `ECS`.
- `video-processing-ms` pode aumentar a quantidade de workers para processar videos em paralelo.
- O uso de fila desacopla a recepcao do upload do processamento pesado.

### Resiliência

- A fila evita perda imediata de trabalho em picos de carga.
- O banco e o storage garantem persistencia do estado.
- O desenho suporta retentativa no processamento.

### Limitações atuais

- O `RabbitMQ` esta em uma unica task ECS, sem HA real.
- Ainda nao ha autoscaling automatico configurado.
- A observabilidade atual esta em nivel basico.

## Estratégia de Persistência

- Dados de autenticacao: `AuthDb`.
- Dados de video e status: `VideoUploadDb`.
- Binarios e artefatos: `S3`.

## CI/CD

### Repositórios da Aplicação

Cada microsservico publica sua imagem no `Docker Hub` via GitHub Actions.

### Repositório de Infraestrutura

O repositório `fiap-hk-infra` executa:

- `terraform fmt`
- `terraform validate`
- `terraform plan`
- `terraform apply`

Os valores sensiveis e refs de imagem sao gerados dinamicamente em `terraform.auto.tfvars` a partir de `Secrets` e `Variables` do GitHub Actions.

## Organização do Terraform

### Infraestrutura compartilhada

No repositório `fiap-hk-infra`:

- `network.tf`: VPC, sub-redes, rotas e namespace privado.
- `security.tf`: security groups.
- `database.tf`: bancos RDS.
- `storage.tf`: bucket S3 e credenciais de acesso para a aplicacao.
- `secrets.tf`: secrets operacionais.
- `ecs.tf` e `ecs-workers.tf`: task definitions e services ECS.
- `outputs.tf`: outputs usados pelos demais repositórios.

### Repositórios de Serviço 

Os Terraform dos microsservicos consomem o `remote_state` da stack compartilhada e expoem apenas os outputs relevantes por serviço.

## Prerequisitos Operacionais

- O bucket remoto do backend Terraform precisa existir antes do primeiro `init`.
- Os secrets e variables do GitHub Actions precisam estar configurados.
- As imagens do Docker Hub precisam existir com as tags esperadas.

## Próximos passos

- Configurar autoscaling por CPU, memoria e profundidade de fila.
- Evoluir o `RabbitMQ` para uma estrategia de alta disponibilidade.
- Adicionar health checks mais especificos para aplicação.
- Refinar observabilidade com alarmes e dashboards.
