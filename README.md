# API DevOps de Tarefas (devops-python-api)

API didática em Python (FastAPI) usada para demonstrar uma esteira completa de **CI/CD**: testes automatizados, construção de imagem Docker, publicação no **Amazon ECR** e implantação automática em uma instância **Amazon EC2**, tudo orquestrado pelo **GitHub Actions**.

Trabalho final da disciplina **Integração DevOps**.

## Sumário

1. [Descrição da aplicação](#1-descrição-da-aplicação)
2. [Tecnologias utilizadas](#2-tecnologias-utilizadas)
3. [Execução local](#3-execução-local)
4. [Docker](#4-docker)
5. [Integração contínua (CI)](#5-integração-contínua-ci)
6. [Entrega contínua (CD)](#6-entrega-contínua-cd)
7. [Arquitetura de implantação](#7-arquitetura-de-implantação)
8. [Acesso à API publicada](#8-acesso-à-api-publicada)
9. [Como verificar a versão implantada](#9-como-verificar-a-versão-implantada)
10. [Evidências](#10-evidências)
11. [Fluxo de desenvolvimento](#11-fluxo-de-desenvolvimento)

---

## 1. Descrição da aplicação

A aplicação é uma API REST simples para gerenciar tarefas. Ela existe para servir de base à automação: o foco do trabalho é a esteira de entrega, e não a regra de negócio.

| Método | Rota | Descrição |
|---|---|---|
| GET | `/` | Mensagem de boas-vindas e link para a documentação |
| GET | `/health` | Estado da API e **versão implantada** |
| GET | `/tasks` | Lista as tarefas |
| POST | `/tasks` | Cria uma tarefa (campo `title`, de 3 a 80 caracteres) |
| PATCH | `/tasks/{task_id}/done` | Marca uma tarefa como concluída |
| GET | `/docs` | Documentação interativa (Swagger UI) |

As tarefas ficam em memória, portanto são perdidas quando o contêiner é reiniciado.

## 2. Tecnologias utilizadas

- **Python 3.13** e **FastAPI** (com **Uvicorn** como servidor)
- **Pytest** e **pytest-cov** (testes e cobertura mínima de 90%)
- **Ruff** (análise estática e formatação)
- **Docker** (contêiner da aplicação)
- **GitHub Actions** (CI e CD)
- **Amazon ECR** (registro privado de imagens)
- **Amazon EC2** (Amazon Linux 2023, instância `t3.micro`)
- **AWS Academy Learner Lab** (ambiente AWS utilizado)

## 3. Execução local

Pré-requisito: Python instalado.

No PowerShell (Windows):

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
uvicorn app.main:app --reload
```

No Linux ou macOS:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
uvicorn app.main:app --reload
```

A API fica disponível em `http://127.0.0.1:8000` (documentação em `/docs`).

Para rodar as mesmas verificações do CI:

```bash
python -m ruff check .
python -m ruff format --check .
python -m pytest --cov=app
```

## 4. Docker

O `Dockerfile` usa a imagem `python:3.13-slim`, instala as dependências de `requirements.txt`, copia o código, executa a aplicação com um usuário sem privilégios e define um `HEALTHCHECK` que consulta `/health`.

Construir a imagem:

```bash
docker build -t devops-python-api:local .
```

Executar o contêiner:

```bash
docker run -d -p 8000:8000 --name api devops-python-api:local
```

Verificar o funcionamento:

```bash
curl http://localhost:8000/health
```

Parar e remover:

```bash
docker stop api
docker rm api
```

O arquivo `.dockerignore` impede que ambiente virtual, caches, testes e documentação entrem na imagem.

## 5. Integração contínua (CI)

Arquivo: `.github/workflows/ci.yml` (workflow **CI Python**).

**Gatilhos:** abertura ou atualização de Pull Request para a `main`, push na `main` e execução manual.

**Job `Qualidade e testes`:**

1. Baixa o código do repositório.
2. Configura o Python (versão definida em `.python-version`).
3. Instala as dependências de `requirements-dev.txt`.
4. Executa `ruff check` e `ruff format --check`.
5. Executa o `pytest` com cobertura (mínimo de 90%) e gera relatórios.
6. Guarda os relatórios como artefato da execução.

**Job `Build e teste do container`** (só roda se a anterior passar):

1. Constrói a imagem Docker.
2. Inicia o contêiner e consulta `/health` até a API responder.
3. Mostra os logs e remove o contêiner.

Se qualquer etapa falhar, o Pull Request fica marcado como reprovado e a entrega não acontece.

## 6. Entrega contínua (CD)

Arquivo: `.github/workflows/deploy.yml` (workflow **Deploy**).

**Gatilhos:** push na `main` (isto é, merge de um Pull Request) e execução manual.

**Job `Testes`:** executa o `pytest`. O deploy só começa se ele passar.

**Job `Build, ECR e EC2`:**

1. Autentica na AWS usando credenciais guardadas em **GitHub Secrets**.
2. Faz login no Amazon ECR.
3. Constrói a imagem Docker e a identifica com a tag do commit (`github.sha`).
4. Publica a imagem no repositório `devops-python-api` do ECR.
5. Conecta por SSH na instância EC2 (chave guardada em Secret).
6. Na EC2: autentica o Docker no ECR, baixa a imagem do commit, remove o contêiner anterior e inicia o novo (`-p 80:8000`).
7. Confirma que o contêiner está em execução e que `/health` responde (com novas tentativas enquanto a API inicia).

Nenhum comando de implantação é executado manualmente na instância.

### Secrets utilizados

Os valores **não** ficam no repositório. Eles são cadastrados em *Settings → Secrets and variables → Actions*.

| Secret | Finalidade |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial AWS (temporária no Learner Lab) |
| `AWS_SECRET_ACCESS_KEY` | Credencial AWS |
| `AWS_SESSION_TOKEN` | Token de sessão da AWS |
| `AWS_REGION` | Região (`us-east-1`) |
| `EC2_HOST` | Endereço da instância EC2 |
| `EC2_USER` | Usuário SSH (`ec2-user`) |
| `EC2_SSH_KEY` | Chave privada para conectar na EC2 |

A instância EC2 obtém permissão de leitura no ECR por uma **IAM Role** (perfil de instância `LabInstanceProfile` do Learner Lab), sem nenhuma credencial gravada nela.

## 7. Arquitetura de implantação

```
Desenvolvedor
     |
     |  git push (branch) + Pull Request
     v
+-------------------+      PR      +--------------------------+
|      GitHub       | -----------> |  GitHub Actions: CI      |
|   (repositório)   |              |  ruff + pytest + build   |
+-------------------+              +--------------------------+
     |
     |  merge na main
     v
+--------------------------------------------------------------+
|  GitHub Actions: Deploy                                      |
|  1. testes   2. build da imagem   3. push para o ECR         |
|  4. SSH na EC2 -> pull da imagem -> troca do contêiner       |
+--------------------------------------------------------------+
        |                                   |
        | docker push (tag = commit)        | SSH (porta 22)
        v                                   v
+------------------+   docker pull   +---------------------------+
|  Amazon ECR      | <-------------- |  Amazon EC2               |
|  (repo privado)  |  (via IAM Role) |  Docker + contêiner "api" |
+------------------+                 |  porta 80 -> 8000         |
                                     +---------------------------+
                                                  ^
                                                  | HTTP (porta 80)
                                              Usuário / navegador
```

Recursos AWS utilizados:

- **ECR:** repositório privado `devops-python-api`.
- **EC2:** uma instância Amazon Linux 2023 (`t3.micro`) com Docker, IP elástico e a IAM Role `LabInstanceProfile`.
- **Security Group:** entrada nas portas 22 (SSH) e 80 (HTTP), saída liberada.

## 8. Acesso à API publicada

- API: `http://184.192.125.81`
- Estado e versão: `http://184.192.125.81/health`
- Documentação: `http://184.192.125.81/docs`

> Os recursos da AWS são encerrados após a avaliação (exigência da atividade), então o endereço acima deixa de responder depois disso. As evidências da seção 10 comprovam o funcionamento.

## 9. Como verificar a versão implantada

**Pelo navegador ou terminal:**

```bash
curl http://184.192.125.81/health
```

A resposta traz a versão da API:

```json
{"status": "ok", "version": "2.0.0"}
```

**Pela imagem em execução na EC2:**

```bash
docker ps
```

A coluna `IMAGE` mostra a tag, que é o hash do commit implantado (compare com o último commit da `main` no GitHub).

**Pelo ECR:** em *Amazon ECR → devops-python-api → Imagens*, cada deploy gera uma imagem com a tag do respectivo commit.

**Pelo GitHub:** na aba *Actions*, o workflow **Deploy** mais recente mostra o commit e o resultado.

A versão é definida no parâmetro `version` de `FastAPI(...)` em `app/main.py`.

## 10. Evidências

### Pull Requests

| PR | Descrição |
|---|---|
| [#1](https://github.com/brenogamademiranda/devops-python-api/pull/1) | Workflow de deploy (ECR e EC2) |
| [#2](https://github.com/brenogamademiranda/devops-python-api/pull/2) | Correção da verificação de saúde no deploy |
| [#3](https://github.com/brenogamademiranda/devops-python-api/pull/3) | Atualização da versão da API para 2.0.0 |

### Integração contínua

![CI verde no PR #1](docs/evidencias/ci-pr1-verde.png)
![CI verde no PR #3](docs/evidencias/ci-pr3-verde.png)

### Publicação no Amazon ECR

![Imagens no ECR](docs/evidencias/ecr-imagens.png)

### Deploy

![Deploy verde no Actions](docs/evidencias/deploy3-actions-verde.png)
![Contêiner em execução na EC2](docs/evidencias/ec2-docker-ps.png)
![Documentação da API no navegador](docs/evidencias/api-docs.png)

### Atualização automática (duas versões)

Primeiro deploy, versão 1.0.0:

![Versão 1](docs/evidencias/versao-1.png)

Segundo deploy, após o merge do PR #3, versão 2.0.0:

![Versão 2](docs/evidencias/versao-2.png)

## 11. Fluxo de desenvolvimento

Nenhuma alteração é feita direto na `main`:

```
Alteração no código -> Branch -> Pull Request -> Testes automáticos (CI)
-> Merge na main -> Build Docker -> Publicação no ECR -> Deploy na EC2
-> Aplicação atualizada
```

Branches utilizadas: `feature/ci-cd-ec2` (esteira inicial), `fix/deploy-healthcheck` (ajuste da verificação de saúde) e `feature/atualiza-versao` (demonstração da atualização automática).
