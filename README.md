# exercicio-pratico-devops

Exercício prático de **Docker**, **GitHub Actions** e **Container Registry** (GHCR) usando uma aplicação simples em **Python / FastAPI**.

O objetivo é praticar o fluxo completo de containerização e integração contínua: da aplicação até a publicação automatizada de uma imagem Docker em um registry, com versionamento por tags (`1.0` e `2.0`).

```
Desenvolvedor -> git push -> Repositório GitHub
              -> GitHub Actions -> Build da imagem Docker
              -> Push para o GHCR
              -> docker pull -> Execução local -> Teste do endpoint
```

## Tecnologias

- Python 3.12 + FastAPI + Uvicorn
- Docker
- GitHub Actions
- GitHub Container Registry (GHCR)

## Estrutura do projeto

```
exercicio-pratico-devops/
├── .github/
│   └── workflows/
│       └── docker.yml      # pipeline de build e push
├── app/
│   ├── __init__.py
│   └── main.py             # aplicação FastAPI
├── .dockerignore
├── Dockerfile
├── README.md
├── requirements.txt
└── VERSION                 # versão usada como tag da imagem
```

## Aplicação

A aplicação expõe um único endpoint:

| Método | Rota     | Resposta (v1.0) | Resposta (v2.0)  |
|--------|----------|-----------------|------------------|
| GET    | `/hello` | `Hello World`   | `Hello World 2`  |

A porta utilizada é a **8080**.

## Executando localmente

### Sem Docker (opcional)

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --port 8080
```

### Com Docker

```bash
docker build -t minha-aplicacao:1.0 .
docker run --rm -p 8080:8080 minha-aplicacao:1.0
```

Em outro terminal:

```bash
curl http://localhost:8080/hello
# Hello World
```

> O `Dockerfile` inicia o Uvicorn com `--host 0.0.0.0`. Sem isso, a aplicação escutaria apenas dentro do container e não responderia ao `curl` feito do host.

## Pipeline (GitHub Actions)

O arquivo `.github/workflows/docker.yml` é executado automaticamente a cada **push na branch `main`** e realiza:

1. Baixa o código do repositório (`actions/checkout`).
2. Lê a versão do arquivo `VERSION`.
3. Faz login no GHCR usando os secrets do repositório.
4. Constrói a imagem Docker.
5. Verifica a imagem: `docker image inspect`, sobe um container e testa `GET /hello`.
6. Publica a imagem no GHCR com a tag igual à versão lida.

### Secrets necessários

Cadastre em **Settings → Secrets and variables → Actions → aba Secrets → New repository secret**:

| Secret              | Conteúdo                                                        |
|---------------------|-----------------------------------------------------------------|
| `REGISTRY_USERNAME` | Usuário do GitHub                                               |
| `REGISTRY_TOKEN`    | Personal Access Token (classic) com o escopo `write:packages`   |

Nenhuma credencial aparece no arquivo da pipeline.

> Os secrets devem ser **Repository secrets**. Secrets criados em *Environment secrets* não chegam ao job, a menos que o job declare `environment:`. Nesse caso o login falha com `Error: Username and password required`.

## Versionamento

A tag da imagem vem do arquivo `VERSION`. Para publicar uma nova versão:

1. Altere o código da aplicação.
2. Atualize o arquivo `VERSION` (por exemplo, de `1.0` para `2.0`).
3. Faça commit e push:

```bash
git add .
git commit -m "Versão 2 da aplicação"
git push
```

A pipeline dispara novamente e publica a nova tag. As tags anteriores continuam disponíveis no registry.

| Versão | Tag da imagem                                | Resposta do `/hello` |
|--------|----------------------------------------------|----------------------|
| 1.0    | `ghcr.io/estevaoaugusto/minha-aplicacao:1.0`    | `Hello World`        |
| 2.0    | `ghcr.io/estevaoaugusto/minha-aplicacao:2.0`    | `Hello World 2`      |

> Substitua `estevaoaugusto` pelo seu usuário do GitHub, **em minúsculas** (o GHCR não aceita maiúsculas no nome da imagem).

## Baixando e executando a imagem publicada

Se o pacote estiver privado, faça login antes:

```bash
echo SEU_TOKEN | docker login ghcr.io -u estevaoaugusto --password-stdin
```

Versão 1.0:

```bash
docker pull ghcr.io/estevaoaugusto/minha-aplicacao:1.0
docker run --rm -p 8080:8080 ghcr.io/estevaoaugusto/minha-aplicacao:1.0
curl http://localhost:8080/hello
# Hello World
```

Versão 2.0 (encerre o container anterior antes, para liberar a porta 8080):

```bash
docker pull ghcr.io/estevaoaugusto/minha-aplicacao:2.0
docker run --rm -p 8080:8080 ghcr.io/estevaoaugusto/minha-aplicacao:2.0
curl http://localhost:8080/hello
# Hello World 2
```

## Problemas comuns

| Sintoma | Causa provável |
|---------|----------------|
| `Error: Username and password required` | Secrets não criados como *Repository secrets*, ou com nome diferente de `REGISTRY_USERNAME` / `REGISTRY_TOKEN` |
| `denied` / `unauthorized` no push | PAT sem o escopo `write:packages`, ou `REGISTRY_USERNAME` diferente do usuário dono do repositório |
| `repository name must be lowercase` | Nome da imagem com letras maiúsculas |
| `curl: connection refused` local | Faltou `--host 0.0.0.0` no `CMD` do Dockerfile, ou a porta do `-p` não bate com a do Uvicorn |
| Pipeline não dispara | A branch principal não se chama `main` |
| `docker pull` retorna `unauthorized` | Pacote privado e sem `docker login ghcr.io` |

Os avisos `DEP0040 punycode` e `logout: true` que aparecem no log do passo de login são normais e não indicam falha.

## Evidências da entrega

- [X] Build e execução local da versão 1.0
- [X] Execução da pipeline (v1.0) com sucesso
- [X] Pacote no GHCR com a tag `1.0`
- [X] `docker pull` + `curl` da imagem `1.0` retornando `Hello World`
- [X] Execução da pipeline (v2.0) com sucesso
- [X] Pacote no GHCR com as tags `1.0` e `2.0`
- [X] `docker pull` + `curl` da imagem `2.0` retornando `Hello World 2`