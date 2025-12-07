# linuxtips-cicd-reusable

Repositório de workflows reutilizáveis para CI/CD com GitHub Actions, focados em integração e deploy contínuo para AWS (ECS/EKS) com segurança e assinatura de imagens Docker.

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Workflows Disponíveis](#workflows-disponíveis)
- [Workflow: Pipeline CI (Continuous Integration)](#workflow-pipeline-ci-continuous-integration)
- [Workflow: Pipeline CD (Continuous Delivery)](#workflow-pipeline-cd-continuous-delivery)
- [Como Usar nos Projetos](#como-usar-nos-projetos)
- [Configuração de Secrets e Variables](#configuração-de-secrets-e-variables)
- [Fluxo Completo do Projeto Sorteador-Strigus](#fluxo-completo-do-projeto-sorteador-strigus)

---

## 🎯 Visão Geral

Este repositório contém workflows reutilizáveis que implementam uma pipeline completa de CI/CD para aplicações containerizadas, incluindo:

- ✅ Lint de Dockerfile (Hadolint)
- ✅ Build de imagens Docker
- ✅ Scan de vulnerabilidades (Trivy)
- ✅ Assinatura de imagens com Cosign
- ✅ Verificação de assinaturas
- ✅ Deploy automático para AWS ECS

---

## 🔄 Workflows Disponíveis

### 1. `pipeline-ci.yaml` - Continuous Integration

Workflow responsável por construir, testar e assinar imagens Docker.

### 2. `pipeline-cd.yaml` - Continuous Delivery

Workflow responsável por verificar assinaturas e fazer deploy para AWS ECS.

---

## 🔨 Workflow: Pipeline CI (Continuous Integration)

### Descrição

Este workflow realiza todo o processo de integração contínua: valida o Dockerfile, constrói a imagem, faz scan de vulnerabilidades, faz push para o ECR e assina a imagem com Cosign.

### Trigger

- **Tipo**: `workflow_call` (chamado por outros workflows)

### Inputs

| Input               | Tipo   | Padrão        | Descrição                                                |
| ------------------- | ------ | ------------- | -------------------------------------------------------- |
| `working-directory` | string | `"app"`       | Diretório onde está o Dockerfile e arquivos da aplicação |
| `aws-region`        | string | `"us-east-1"` | Região AWS onde está o ECR                               |

### Secrets Obrigatórios

| Secret               | Descrição                                    |
| -------------------- | -------------------------------------------- |
| `AWS_ROLE_TO_ASSUME` | ARN da role AWS para autenticação via OIDC   |
| `COSIGN_KEY`         | Chave privada do Cosign codificada em Base64 |
| `COSIGN_PASSWORD`    | Senha da chave privada do Cosign (opcional)  |

### Variáveis Obrigatórias

| Variable     | Descrição                    |
| ------------ | ---------------------------- |
| `IMAGE_NAME` | Nome da imagem Docker no ECR |

### Passo a Passo Detalhado

#### Job: `build-scan-dockerfile`

**1. Checkout do Código**

```yaml
- name: "Checkout"
  uses: actions/checkout@v4
```

- Faz checkout do código do repositório no runner

**2. Dockerfile Lint (Hadolint)**

```yaml
- name: "Dockerfile Lint (Hadolint)"
  uses: hadolint/hadolint-action@v3.1.0
```

- Analisa o Dockerfile em busca de más práticas e problemas de segurança
- Gera relatório em `lint-report.txt`
- `no-fail: true` permite que o workflow continue mesmo com warnings

**3. Upload do Relatório de Lint**

```yaml
- name: "Upload lint-report.txt"
  uses: actions/upload-artifact@v4
```

- Salva o relatório do Hadolint como artifact (disponível por 30 dias)

**4. Configurar Credenciais AWS (OIDC)**

```yaml
- name: Configurar credenciais AWS (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
```

- Configura autenticação na AWS usando OIDC (OpenID Connect)
- Não usa chaves de acesso, mais seguro
- Requer permissão `id-token: write`

**5. Login no Amazon ECR**

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v2
```

- Autentica no Elastic Container Registry (ECR)
- Retorna o endpoint do registry para uso posterior

**6. Setup Docker Buildx**

```yaml
- name: "Set up Docker Buildx"
  uses: docker/setup-buildx-action@v3
```

- Configura Docker Buildx para builds otimizados
- Suporta builds multi-plataforma

**7. Compilar Nome da Imagem**

```yaml
- name: "Compila nome do repo"
  run: |
    REGISTRY="${{ steps.login-ecr.outputs.registry }}"
    FULL_IMAGE="$( echo "${REGISTRY}/${{ vars.IMAGE_NAME }}:${{ github.sha }}" | tr '[:upper:]' '[:lower:]' )"
```

- Monta o nome completo da imagem: `registry/imagem:commit-sha`
- Converte para minúsculas (requisito do Docker)
- Usa o SHA do commit como tag

**8. Build da Imagem Docker**

```yaml
- name: "Build da imagem"
  uses: docker/build-push-action@v6
  with:
    push: false
    load: true
```

- Constrói a imagem Docker localmente
- `push: false` - não faz push ainda (aguarda validação)
- `load: true` - carrega a imagem no Docker local para scan

**9. Scan de Vulnerabilidades (Trivy)**

```yaml
- name: "Trivy - Scan"
  uses: aquasecurity/trivy-action@0.28.0
  with:
    exit-code: "1"
    severity: "HIGH,CRITICAL"
```

- Escaneia a imagem em busca de vulnerabilidades
- Foca em vulnerabilidades HIGH e CRITICAL
- `exit-code: '1'` - falha o workflow se encontrar vulnerabilidades
- `ignore-unfixed: true` - ignora vulnerabilidades sem patch disponível

**10. Upload do Relatório Trivy**

```yaml
- name: "Upload trivy-report.txt"
  uses: actions/upload-artifact@v4
```

- Salva o relatório de vulnerabilidades como artifact

**11. Push da Imagem para ECR**

```yaml
- name: "Push image to ECR"
  if: ${{ steps.trivy.outcome == 'success' }}
  run: docker push "${{ steps.compile.outputs.full_image }}"
```

- **Condição**: Só executa se o Trivy passou sem erros
- Faz push da imagem para o ECR
- A imagem agora está disponível no registry

**12. Instalar Cosign**

```yaml
- name: "Install Cosign"
  uses: sigstore/cosign-installer@v3.4.0
```

- Instala a ferramenta Cosign para assinatura de imagens
- **Condição**: Só executa se Trivy e Push foram bem-sucedidos

**13. Preparar Chave Privada do Cosign**

```yaml
- name: "Prepare Cosign key"
  run: |
    KEY_PATH="${GITHUB_WORKSPACE}/${{ env.WORKDIR }}/cosign.key"
    echo "${{ secrets.COSIGN_KEY }}" | base64 -d > "$KEY_PATH"
    chmod 600 "$KEY_PATH"
```

- Decodifica a chave privada do Cosign (que está em Base64)
- Salva em arquivo com permissões restritas (600)
- **Condição**: Só executa se etapas anteriores foram bem-sucedidas

**14. Assinar Imagem com Cosign**

```yaml
- name: "Cosign - Sign image"
  run: cosign sign --key "${COSIGN_KEY_PATH}" "${{ steps.compile.outputs.full_image }}"
```

- Assina a imagem Docker com a chave privada
- A assinatura é armazenada no registry junto com a imagem
- Garante integridade e autenticidade da imagem
- **Condição**: Só executa se etapas anteriores foram bem-sucedidas

---

## 🚀 Workflow: Pipeline CD (Continuous Delivery)

### Descrição

Este workflow verifica a assinatura da imagem e faz o deploy para AWS ECS.

### Trigger

- **Tipo**: `workflow_call` (chamado por outros workflows)

### Inputs

| Input               | Tipo   | Padrão        | Descrição                   |
| ------------------- | ------ | ------------- | --------------------------- |
| `working-directory` | string | `"app"`       | Diretório da aplicação      |
| `aws-region`        | string | `"us-east-1"` | Região AWS                  |
| `deploy-type`       | string | `"ecs"`       | Tipo de deploy (ECS ou EKS) |

### Secrets Obrigatórios

| Secret               | Descrição                                     |
| -------------------- | --------------------------------------------- |
| `AWS_ROLE_TO_ASSUME` | ARN da role AWS para autenticação via OIDC    |
| `COSIGN_KEY_PUB`     | Chave pública do Cosign codificada em Base64  |
| `COSIGN_PASSWORD`    | Senha da chave privada (usada na verificação) |

### Variáveis Obrigatórias

| Variable         | Descrição                            |
| ---------------- | ------------------------------------ |
| `IMAGE_NAME`     | Nome da imagem Docker no ECR         |
| `TASK_NAME`      | Nome da task definition do ECS       |
| `CONTAINER_NAME` | Nome do container na task definition |
| `SERVICE_NAME`   | Nome do serviço ECS                  |
| `CLUSTER_NAME`   | Nome do cluster ECS                  |

### Passo a Passo Detalhado

#### Job 1: `prepare`

**1. Checkout do Código**

```yaml
- name: "Checkout"
  uses: actions/checkout@v4
```

- Faz checkout do código (necessário para acessar arquivos locais)

**2. Configurar Credenciais AWS (OIDC)**

```yaml
- name: Configurar credenciais AWS (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
```

- Configura autenticação na AWS via OIDC

**3. Login no Amazon ECR**

```yaml
- name: Login to Amazon ECR
  uses: aws-actions/amazon-ecr-login@v2
```

- Autentica no ECR para acessar a imagem

**4. Compilar Nome da Imagem**

```yaml
- name: "Compila nome do repo"
  run: |
    REGISTRY="${{ steps.login-ecr.outputs.registry }}"
    FULL_IMAGE="$( echo "${REGISTRY}/${{ vars.IMAGE_NAME }}:${{ github.sha }}" | tr '[:upper:]' '[:lower:]' )"
```

- Monta o nome completo da imagem (mesmo processo do CI)
- Output é usado no job de deploy

**5. Instalar Cosign**

```yaml
- name: "Install Cosign"
  uses: sigstore/cosign-installer@v3.4.0
```

- Instala Cosign para verificação de assinaturas

**6. Preparar Chave Pública do Cosign**

```yaml
- name: Prepare Cosign public key
  run: |
    PUB_PATH="${{ inputs.working-directory }}/cosign.pub"
    echo "${{ secrets.COSIGN_KEY_PUB }}" | base64 -d > "$PUB_PATH"
    chmod 644 "$PUB_PATH"
```

- Decodifica a chave pública do Cosign (em Base64)
- Salva em arquivo com permissões de leitura (644)
- **Diferença do CI**: Usa chave pública (para verificar), não privada (para assinar)

**7. Verificar Assinatura da Imagem**

```yaml
- name: Cosign - Verify signature
  run: cosign verify --key "${COSIGN_PUB_PATH}" "${{ steps.compile.outputs.full_image }}"
```

- Verifica se a imagem foi assinada corretamente
- Usa a chave pública para validar a assinatura
- Se a verificação falhar, o workflow é interrompido
- `--insecure-ignore-tlog` ignora o transparency log (opcional)

#### Job 2: `ecs-deploy`

**Condição de Execução**

```yaml
if: ${{ inputs.deploy-type == 'ecs' && needs.prepare.result == 'success' }}
```

- Só executa se:
  - O tipo de deploy é ECS
  - O job `prepare` foi bem-sucedido (assinatura verificada)

**1. Checkout do Código**

```yaml
- name: "Checkout"
  uses: actions/checkout@v4
```

- Faz checkout (necessário para scripts auxiliares, se houver)

**2. Configurar Credenciais AWS (OIDC)**

```yaml
- name: Configurar credenciais AWS (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
```

- Configura autenticação AWS

**3. Obter Task Definition**

```yaml
- name: Get Task Definition
  run: |
    aws ecs describe-task-definition --task-definition ${{ vars.TASK_NAME }} --query taskDefinition > task-definition.json
    echo $(cat task-definition.json | jq 'del(...)') > task-definition.json
```

- Busca a task definition atual do ECS
- Remove campos que não podem ser modificados (ARN, revision, status, etc.)
- Prepara o arquivo para atualização

**4. Renderizar Task Definition**

```yaml
- name: Render Task Definition
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: task-definition.json
    container-name: ${{ vars.CONTAINER_NAME }}
    image: ${{ needs.prepare.outputs.full_image }}
```

- Atualiza a task definition com a nova imagem
- Substitui a imagem do container pela nova versão (com SHA do commit)
- Gera uma nova task definition pronta para deploy

**5. Deploy para Amazon ECS**

```yaml
- name: Deploy to Amazon ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@v2
  with:
    task-definition: ${{ steps.render.outputs.task-definition }}
    service: ${{ vars.SERVICE_NAME }}
    cluster: ${{ vars.CLUSTER_NAME }}
    wait-for-service-stability: true
```

- Faz o deploy da nova task definition no serviço ECS
- Atualiza o serviço com a nova imagem
- `wait-for-service-stability: true` - aguarda o serviço estabilizar antes de finalizar
- O deploy só é considerado completo quando o serviço está rodando com a nova versão

---

## 📦 Como Usar nos Projetos

### Exemplo: Workflow Principal do Sorteador-Strigus

```yaml
name: Pipeline CI/CD DevOps

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: write

jobs:
  ci:
    uses: paulocesar-prog/linuxtips-cicd-reusable/.github/workflows/pipeline-ci.yaml@main
    with:
      working-directory: app
      aws-region: us-east-1
    secrets:
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      COSIGN_KEY: ${{ secrets.COSIGN_KEY }}
      COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}

  cd:
    needs: ci
    uses: paulocesar-prog/linuxtips-cicd-reusable/.github/workflows/pipeline-cd.yaml@main
    with:
      working-directory: app
      aws-region: us-east-1
      deploy-type: ${{ vars.TYPE }}
    secrets:
      AWS_ROLE_TO_ASSUME: ${{ secrets.AWS_ROLE_TO_ASSUME }}
      COSIGN_KEY_PUB: ${{ secrets.COSIGN_KEY_PUB }}
      COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
```

### Explicação do Fluxo

1. **Job `ci`**: Executa o workflow de CI

   - Build, scan e assinatura da imagem
   - Faz push para ECR

2. **Job `cd`**: Executa após o CI (`needs: ci`)
   - Verifica assinatura
   - Faz deploy para ECS

---

## 🔐 Configuração de Secrets e Variables

### Secrets (Settings → Secrets and variables → Actions → Secrets)

| Secret               | Como Obter                                                         |
| -------------------- | ------------------------------------------------------------------ |
| `AWS_ROLE_TO_ASSUME` | ARN da role IAM configurada para OIDC no GitHub                    |
| `COSIGN_KEY`         | Chave privada do Cosign em Base64: `cat cosign.key \| base64 -w 0` |
| `COSIGN_KEY_PUB`     | Chave pública do Cosign em Base64: `cat cosign.pub \| base64 -w 0` |
| `COSIGN_PASSWORD`    | Senha usada ao gerar as chaves Cosign                              |

### Variables (Settings → Secrets and variables → Actions → Variables)

| Variable         | Exemplo                     | Descrição                   |
| ---------------- | --------------------------- | --------------------------- |
| `IMAGE_NAME`     | `sorteador-strigus`         | Nome da imagem no ECR       |
| `TASK_NAME`      | `sorteador-strigus-task`    | Nome da task definition     |
| `CONTAINER_NAME` | `app`                       | Nome do container na task   |
| `SERVICE_NAME`   | `sorteador-strigus-service` | Nome do serviço ECS         |
| `CLUSTER_NAME`   | `production-cluster`        | Nome do cluster ECS         |
| `TYPE`           | `ecs`                       | Tipo de deploy (ecs ou eks) |

### Como Gerar Chaves Cosign

```bash
# Gerar par de chaves
cosign generate-key-pair

# Isso cria:
# - cosign.key (chave privada - NUNCA compartilhar)
# - cosign.pub (chave pública - pode compartilhar)

# Codificar em Base64 para GitHub Secrets
cat cosign.key | base64 -w 0    # Para COSIGN_KEY
cat cosign.pub | base64 -w 0    # Para COSIGN_KEY_PUB
```

---

## 🔄 Fluxo Completo do Projeto Sorteador-Strigus

### Visão Geral do Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│  Trigger: workflow_dispatch (execução manual)                │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  JOB: ci                                                    │
│  Workflow: pipeline-ci.yaml                                 │
└─────────────────────────────────────────────────────────────┘
    │
    ├─► Checkout código
    ├─► Lint Dockerfile (Hadolint)
    ├─► Autenticação AWS (OIDC)
    ├─► Login ECR
    ├─► Build imagem Docker
    ├─► Scan vulnerabilidades (Trivy)
    ├─► Push imagem para ECR
    ├─► Assinar imagem (Cosign)
    │
    └─► ✅ Imagem assinada no ECR
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  JOB: cd (needs: ci)                                        │
│  Workflow: pipeline-cd.yaml                                 │
└─────────────────────────────────────────────────────────────┘
    │
    ├─► Checkout código
    ├─► Autenticação AWS (OIDC)
    ├─► Login ECR
    ├─► Verificar assinatura (Cosign)
    ├─► Obter Task Definition atual
    ├─► Atualizar Task Definition com nova imagem
    ├─► Deploy para ECS
    │
    └─► ✅ Aplicação em produção
```

### Detalhamento por Etapa

#### Fase 1: Continuous Integration (CI)

1. **Validação**

   - Hadolint verifica boas práticas no Dockerfile
   - Relatório salvo como artifact

2. **Build**

   - Imagem Docker construída localmente
   - Tag: `registry/imagem:commit-sha`

3. **Segurança**

   - Trivy escaneia vulnerabilidades
   - Se encontrar HIGH/CRITICAL, workflow falha
   - Relatório salvo como artifact

4. **Publicação**
   - Imagem enviada para ECR (se scan passou)
   - Imagem assinada com Cosign
   - Assinatura armazenada no registry

#### Fase 2: Continuous Delivery (CD)

1. **Verificação**

   - Chave pública do Cosign preparada
   - Assinatura da imagem verificada
   - Se inválida, workflow falha

2. **Preparação**

   - Task definition atual obtida do ECS
   - Campos imutáveis removidos
   - Nova imagem inserida na definição

3. **Deploy**
   - Nova task definition registrada
   - Serviço ECS atualizado
   - Aguarda estabilização do serviço
   - Rollback automático se falhar

### Benefícios desta Arquitetura

✅ **Segurança**

- Scan de vulnerabilidades antes do deploy
- Assinatura e verificação de imagens
- Autenticação sem chaves (OIDC)

✅ **Confiabilidade**

- Deploy só acontece se CI passou
- Verificação de assinatura obrigatória
- Rollback automático em caso de falha

✅ **Rastreabilidade**

- Cada imagem tem tag única (commit SHA)
- Histórico completo de builds e deploys
- Relatórios de segurança preservados

✅ **Reutilização**

- Workflows reutilizáveis em múltiplos projetos
- Configuração centralizada
- Fácil manutenção e atualização

---

## 📝 Notas Importantes

### Permissões Necessárias

O workflow precisa das seguintes permissões:

- **GitHub Actions**: `id-token: write` (para OIDC)
- **AWS IAM Role**: Permissões para:
  - ECR: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:PutImage`
  - ECS: `ecs:DescribeTaskDefinition`, `ecs:RegisterTaskDefinition`, `ecs:UpdateService`, `ecs:DescribeServices`

### Tratamento de Erros

- **Trivy encontra vulnerabilidade**: Workflow falha, imagem não é publicada
- **Assinatura inválida**: Workflow CD falha, deploy não acontece
- **Deploy falha**: ECS faz rollback automático para versão anterior

### Tags de Imagem

As imagens são taggeadas com o SHA do commit (`${{ github.sha }}`), garantindo:

- Rastreabilidade completa
- Impossibilidade de sobrescrever versões
- Facilidade para rollback

---

## 🤝 Contribuindo

Para melhorar estes workflows, faça um Pull Request com suas sugestões!

---

## 📚 Referências

- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [AWS ECR](https://aws.amazon.com/ecr/)
- [AWS ECS](https://aws.amazon.com/ecs/)
- [Cosign](https://github.com/sigstore/cosign)
- [Trivy](https://github.com/aquasecurity/trivy)
- [Hadolint](https://github.com/hadolint/hadolint)
