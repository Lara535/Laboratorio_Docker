# Laboratório Docker

API **ASP.NET Core (.NET 10)** containerizada com Docker, com multi-stage build, imagens versionadas, ambientes de **staging** e **production**, health check, falha controlada e **rollback**.

- **Disciplina:** Integração DevOps — Ciência da Computação
- **Professor:** Daniel Lim-Apo
- **Aluna:** Lara Abreu

## Estrutura

```
DevOpsWebApi/
├── Program.cs
├── DevOpsWebApi.csproj
├── Dockerfile          # multi-stage build (SDK -> aspnet runtime)
├── .dockerignore
├── compose.yaml        # staging + production
└── docs/               # Documento de Evidências
```

## Endpoints

| Rota | Descrição |
|---|---|
| `GET /` | Aplicação, versão e ambiente |
| `GET /health` | Health check (`status`, `version`, `environment`, `timestamp`) |
| `GET /version` | Versão e mensagem (a partir da 1.1.0) |

Configuração por variáveis de ambiente: `APP_ENVIRONMENT` (padrão `development`) e `APP_VERSION` (padrão `local`).

## Execução local (sem Docker)

```bash
dotnet run
```

## Construção

```bash
docker build -t devops-webapi:1.0.0 .
docker images
```

## Execução

```bash
docker run -d --name devops-api -p 8080:8080 devops-webapi:1.0.0
docker ps
docker logs devops-api
docker stop devops-api && docker rm devops-api
```

## Staging

```bash
docker run -d --name devops-api-staging -p 8081:8080 \
  -e APP_ENVIRONMENT=staging -e APP_VERSION=1.0.0 devops-webapi:1.0.0
```

## Production

```bash
docker run -d --name devops-api-prod -p 8082:8080 \
  -e APP_ENVIRONMENT=production -e APP_VERSION=1.0.0 devops-webapi:1.0.0
```

## Health Check

```bash
curl -f http://localhost:8081/health   # staging
curl -f http://localhost:8082/health   # production
# Windows: use curl.exe
```

## Nova versão (1.1.0)

Adiciona o endpoint `/version`. Deploy primeiro em staging, validação e promoção para production.

```bash
docker build -t devops-webapi:1.1.0 .

docker rm -f devops-api-staging
docker run -d --name devops-api-staging -p 8081:8080 \
  -e APP_ENVIRONMENT=staging -e APP_VERSION=1.1.0 devops-webapi:1.1.0

curl -f http://localhost:8081/health     # quality gate

docker rm -f devops-api-prod
docker run -d --name devops-api-prod -p 8082:8080 \
  -e APP_ENVIRONMENT=production -e APP_VERSION=1.1.0 devops-webapi:1.1.0
```

## Falha controlada (1.2.0)

Release **propositalmente quebrada**: `/health` responde HTTP 500. O health check falha e a versão **não deve ser promovida**.

```bash
docker build -t devops-webapi:1.2.0 .

docker rm -f devops-api-staging
docker run -d --name devops-api-staging -p 8081:8080 \
  -e APP_ENVIRONMENT=staging -e APP_VERSION=1.2.0 devops-webapi:1.2.0

curl -f http://localhost:8081/health
# curl: (22) The requested URL returned error: 500
```

## Rollback

Retorno da produção para a última versão estável (`devops-webapi:1.1.0`):

```bash
docker rm -f devops-api-prod
docker run -d --name devops-api-prod -p 8082:8080 \
  -e APP_ENVIRONMENT=production -e APP_VERSION=1.1.0 devops-webapi:1.1.0

curl -f http://localhost:8082/health     # status: healthy
```

## Docker Compose

Staging (`8081`) e production (`8082`) declarados no `compose.yaml` com a imagem `devops-webapi:1.1.0`.

```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose config
docker compose down
```

## Limpeza

```bash
docker compose down
docker rmi devops-webapi:1.0.0 devops-webapi:1.1.0 devops-webapi:1.2.0
```
