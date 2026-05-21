# Plano de Implementação: Configuração do Skillcore-Site no Google Cloud Run com CI/CD via GitHub

> **Para executores:** REQUISITO SUB-HABILIDADE: Use `superpowers:executing-plans` para implementar este plano passo a passo.

**Objetivo:** Criar um novo projeto no Google Cloud Platform (`Skillcore-Site`), configurar um container Docker com Nginx para servir o site estático no Google Cloud Run, mapear o domínio `skillcore.com.br` e configurar um pipeline de CI/CD via GitHub Actions para deploy automático a cada commit na branch `main`.

**Arquitetura:** 
- **Servidor Web:** Docker contendo servidor leve Nginx (Alpine) configurado para expor a porta 8080 (padrão do Cloud Run).
- **Hospedagem:** Google Cloud Run (Serverless, auto-scaling de zero a N).
- **Registro de Imagens:** Google Artifact Registry.
- **CI/CD:** GitHub Actions autenticado via Service Account Key com permissões mínimas necessárias.
- **Domínio:** Mapeamento de domínio customizado para `skillcore.com.br`.

**Tech Stack:**
- Google Cloud SDK (CLI)
- Docker & Nginx (Alpine)
- GitHub Actions (CI/CD)

---

### Tarefa 1: Criação e Configuração Inicial do Projeto GCP

**Arquivos:** N/A (Operações CLI do gcloud)

- [ ] **Passo 1: Criar o novo projeto no Google Cloud**
  Criar o projeto com nome `Skillcore-Site` e um ID de projeto único. Vamos tentar usar o ID `skillcore-site-prod`.
  Execute: `gcloud projects create skillcore-site-prod --name="Skillcore-Site"`
  *Nota: Se o ID já estiver em uso por outro usuário GCP no mundo, tentaremos um sufixo como `skillcore-site-2026`.*

- [ ] **Passo 2: Vincular o projeto ao faturamento (Billing)**
  Vincular o projeto criado à conta de faturamento ativa `SkillCore Billing` (ID: `0163DF-17FF5F-0022E6`).
  Execute: `gcloud billing projects link skillcore-site-prod --billing-account=0163DF-17FF5F-0022E6`

- [ ] **Passo 3: Ativar as APIs necessárias no projeto**
  Definir o projeto ativo e ativar os serviços do Cloud Run, Artifact Registry, Cloud Build e IAM.
  Execute:
  ```bash
  gcloud config set project skillcore-site-prod
  gcloud services enable run.googleapis.com \
                         artifactregistry.googleapis.com \
                         cloudbuild.googleapis.com \
                         iam.googleapis.com
  ```

---

### Tarefa 2: Criação do Dockerfile, Configuração do Nginx e .dockerignore

**Arquivos:**
- Criar: `Dockerfile`
- Criar: `nginx.conf`
- Criar: `.dockerignore`

- [ ] **Passo 1: Criar o arquivo `nginx.conf`**
  Configurar o Nginx para rodar na porta 8080 (padrão esperado pelo Cloud Run) e servir os arquivos estáticos de forma otimizada.
  Conteúdo do arquivo `/Users/thiagonascimento/skillcore/sknovo/nginx.conf`:
  ```nginx
  server {
      listen 8080;
      server_name localhost;

      location / {
          root /usr/share/nginx/html;
          index index.html index.htm;
          try_files $uri $uri/ /index.html;
      }

      error_page 500 502 503 504 /50x.html;
      location = /50x.html {
          root /usr/share/nginx/html;
      }
  }
  ```

- [ ] **Passo 2: Criar o arquivo `Dockerfile`**
  Usar imagem Alpine leve do Nginx, copiar as configurações e os arquivos do site.
  Conteúdo do arquivo `/Users/thiagonascimento/skillcore/sknovo/Dockerfile`:
  ```dockerfile
  FROM nginx:alpine
  COPY nginx.conf /etc/nginx/conf.d/default.conf
  COPY . /usr/share/nginx/html
  EXPOSE 8080
  CMD ["nginx", "-g", "daemon off;"]
  ```

- [ ] **Passo 3: Criar o arquivo `.dockerignore`**
  Ignorar arquivos de desenvolvimento, pastas Git e documentação para reduzir o tamanho da imagem de build.
  Conteúdo do arquivo `/Users/thiagonascimento/skillcore/sknovo/.dockerignore`:
  ```dockerignore
  .git
  .github
  docs
  Dockerfile
  nginx.conf
  .dockerignore
  ```

---

### Tarefa 3: Criação do Repositório do Artifact Registry e Service Account

**Arquivos:** N/A (Operações CLI do gcloud)

- [ ] **Passo 1: Criar o repositório Docker no Artifact Registry**
  Criar o repositório na região `southamerica-east1`.
  Execute: `gcloud artifacts repositories create skillcore-site-repo --repository-format=docker --location=southamerica-east1 --description="Docker repository for Skillcore static site"`

- [ ] **Passo 2: Criar a Service Account para o GitHub Actions**
  Criar conta de serviço dedicada com privilégios limitados de deploy.
  Execute: `gcloud iam service-accounts create github-actions-deployer --display-name="GitHub Actions Deployer"`

- [ ] **Passo 3: Conceder permissões para a Service Account**
  Conceder permissões de escrita no Artifact Registry e administração no Cloud Run.
  Execute:
  ```bash
  # Permissão para gerenciar e fazer deploy no Cloud Run
  gcloud projects add-iam-policy-binding skillcore-site-prod \
    --member="serviceAccount:github-actions-deployer@skillcore-site-prod.iam.gserviceaccount.com" \
    --role="roles/run.admin"

  # Permissão para agir como a Service Account na execução do Cloud Run
  gcloud projects add-iam-policy-binding skillcore-site-prod \
    --member="serviceAccount:github-actions-deployer@skillcore-site-prod.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"

  # Permissão para enviar imagens para o Artifact Registry
  gcloud projects add-iam-policy-binding skillcore-site-prod \
    --member="serviceAccount:github-actions-deployer@skillcore-site-prod.iam.gserviceaccount.com" \
    --role="roles/artifactregistry.writer"
  ```

- [ ] **Passo 4: Gerar e baixar a chave JSON da Service Account**
  Gerar a chave JSON para autenticação do GitHub Actions.
  Execute: `gcloud iam service-accounts keys create sa-key.json --iam-account=github-actions-deployer@skillcore-site-prod.iam.gserviceaccount.com`

---

### Tarefa 4: Configuração dos Segredos e Variáveis no GitHub

**Arquivos:** N/A (Operações CLI via gh)

- [ ] **Passo 1: Adicionar a chave da Service Account como segredo no repositório GitHub**
  Usando o GitHub CLI `gh` logado, adicionar o conteúdo de `sa-key.json` como o segredo `GCP_SA_KEY`.
  Execute: `gh secret set GCP_SA_KEY < sa-key.json`

- [ ] **Passo 2: Adicionar o ID do Projeto GCP como variável no GitHub**
  Configurar a variável `GCP_PROJECT_ID` para apontar para `skillcore-site-prod`.
  Execute: `gh variable set GCP_PROJECT_ID --body "skillcore-site-prod"`

- [ ] **Passo 3: Limpar o arquivo de chave local**
  Remover o arquivo `sa-key.json` local por motivos de segurança.
  Execute: `rm sa-key.json`

---

### Tarefa 5: Configuração do Workflow do GitHub Actions

**Arquivos:**
- Criar: `.github/workflows/deploy.yml`

- [ ] **Passo 1: Criar o arquivo `.github/workflows/deploy.yml`**
  Criar o pipeline que roda no push para a branch `main`, constrói a imagem Docker, envia para o Artifact Registry e faz o deploy no Cloud Run.
  Conteúdo do arquivo `/Users/thiagonascimento/skillcore/sknovo/.github/workflows/deploy.yml`:
  ```yaml
  name: Deploy to Google Cloud Run

  on:
    push:
      branches:
        - main

  env:
    PROJECT_ID: ${{ vars.GCP_PROJECT_ID }}
    REGION: southamerica-east1
    SERVICE_NAME: skillcore-site
    REPOSITORY_NAME: skillcore-site-repo

  jobs:
    deploy:
      name: Build and Deploy Static Site
      runs-on: ubuntu-latest

      steps:
        - name: Checkout Code
          uses: actions/checkout@v4

        - name: Google Auth
          id: auth
          uses: google-github-actions/auth@v2
          with:
            credentials_json: ${{ secrets.GCP_SA_KEY }}

        - name: Set up Cloud SDK
          uses: google-github-actions/setup-gcloud@v2

        - name: Authorize Docker pushes
          run: |
            gcloud auth configure-docker southamerica-east1-docker.pkg.dev --quiet

        - name: Build and Push Docker Image
          run: |
            IMAGE_TAG="southamerica-east1-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY_NAME }}/${{ env.SERVICE_NAME }}:${{ github.sha }}"
            docker build -t $IMAGE_TAG .
            docker push $IMAGE_TAG
            echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV

        - name: Deploy to Cloud Run
          uses: google-github-actions/deploy-cloudrun@v2
          with:
            service: ${{ env.SERVICE_NAME }}
            region: ${{ env.REGION }}
            image: ${{ env.IMAGE_TAG }}
            flags: '--allow-unauthenticated'
  ```

---

### Tarefa 6: Primeira Execução e Mapeamento do Domínio `skillcore.com.br`

**Arquivos:** N/A (Operações CLI do gcloud)

- [ ] **Passo 1: Realizar o primeiro deploy manualmente ou via commit**
  Como acabamos de configurar o Dockerfile e o workflow, faremos um commit inicial das configurações para a branch `main` (ou faremos o deploy manual uma vez para garantir que o serviço Cloud Run existe antes de mapear o domínio).
  Fazer o deploy inicial diretamente para criar o serviço Cloud Run:
  Execute: `gcloud run deploy skillcore-site --source . --region=southamerica-east1 --allow-unauthenticated`

- [ ] **Passo 2: Mapear o domínio `skillcore.com.br` para o Cloud Run**
  Configurar o mapeamento do domínio customizado no Cloud Run na região `southamerica-east1`.
  Execute: `gcloud beta run domain-mappings create --service=skillcore-site --domain=skillcore.com.br --region=southamerica-east1`

- [ ] **Passo 3: Exibir registros DNS necessários**
  A execução do mapeamento de domínio irá gerar registros DNS (como registros A, AAAA e TXT para verificação do domínio). Exibiremos esses registros para o usuário configurar em seu provedor de DNS (Cloudflare, GoDaddy, etc.).
  Execute: `gcloud beta run domain-mappings describe --domain=skillcore.com.br --region=southamerica-east1`
