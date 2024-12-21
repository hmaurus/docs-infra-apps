# Plano Diretor da Infraestrutura

Este documento serve como referência centralizada. Ajudando-o sempre que precisar retomar o contexto ou gerar ou ajustar arquivos.

## Visão Geral

A arquitetura proposta é baseada em uma separação clara entre infraestrutura (infra) e aplicativos em repositórios separados. Cada aplicativo tem seu próprio repositório contendo seu backend (NestJS) e frontend (Next.js). A infraestrutura central (reverse proxy, banco de dados, certbot, etc.) fica em um repositório de infra independente.

A infra será compartilhada entre os aplicativos.

A ideia é permitir desenvolvimento local simples com hot-reload e builds locais (`docker-compose.dev.yml`) e um ambiente de produção limpo consumindo imagens pré-buildadas de um registry (`docker-compose.prod.yml`).

## Organização Geral

Teremos uma estrutura de diretórios e repositórios semelhantes a:

```
workspace/
├── infra/
│   ├── docker-compose.yml
│   ├── docker-compose.dev.yml
│   ├── docker-compose.prod.yml
│   ├── .env
│   ├── .env.dev
│   ├── .env.prod
│   ├── docker/
│   │   ├── nginx/
│   │   │   ├── Dockerfile
│   │   │   ├── nginx.conf
│   │   │   └── sites-enabled/
│   │   │       ├── app1.conf
│   │   │       └── app2.conf
│   │   ├── certbot/
│   │   │   ├── Dockerfile
│   │   │   └── scripts/ (scripts de renovação de certificados)
│   │   ├── database/
│   │   │   └── Dockerfile (opcional)
│   │   ├── redis/
│   │   │   └── Dockerfile (opcional)
│   │   └── shared-scripts/
│   │       └── backup.sh (exemplo)
│   ├── data/
│   │   ├── database/
│   │   ├── certs/
│   │   ├── logs/
│   │   └── uploads/
│   └── README.md
│
├── app1/
│   ├── backend/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   ├── src/ (Código fonte NestJS)
│   │   └── ...
│   └── frontend/
│       ├── Dockerfile
│       ├── package.json
│       ├── src/ (Código fonte Next.js)
│       └── ...
│
└── app2/
    ├── backend/
    │   ├── Dockerfile
    │   ├── package.json
    │   ├── src/ (Código fonte NestJS)
    │   └── ...
    └── frontend/
        ├── Dockerfile
        ├── package.json
        ├── src/ (Código fonte Next.js)
        └── ...
```

## Repositórios Separados

- **infra**: Contém a infraestrutura (docker-compose.yml, configurações do Nginx, Certbot, banco de dados, cache, scripts de backup, etc.). Não contém código-fonte de aplicativos.
- **appX**: Cada aplicativo possui seu repositório separado, contendo:
  - `backend/`: Backend NestJS, com seu próprio `Dockerfile`.
  - `frontend/`: Frontend Next.js, também com seu `Dockerfile`.
  
Essas apps não precisam conter a infra de orquestração. Eles apenas fornecem o código-fonte e a lógica dos serviços. As imagens destes serviços serão construídas em pipelines de CI/CD e publicadas em um registry.

## Convenções Adotadas

- **Nomes de serviços no docker-compose**:  
  Ex: `app1-backend`, `app1-frontend`.  
  Cada serviço tem seu próprio contêiner, garantindo isolamento.

- **Arquivos de Compose**:  
  - `docker-compose.yml`: Arquivo base com definição dos serviços.
  - `docker-compose.dev.yml`: Override para desenvolvimento.  
    Neste arquivo utilizamos `build: ../appX/backend` e volumes locais para hot-reload.
  - `docker-compose.prod.yml`: Override para produção.  
    Neste arquivo utilizamos `image: meu-registry.com/appX-backend:tag` em vez de `build:`.  
    Não montamos volumes locais, pois em produção não precisamos do código-fonte, apenas das imagens pré-buildadas.

- **Arquivos .env**:  
  - `.env` base: Variáveis genéricas.  
  - `.env.dev`: Variáveis específicas para desenvolvimento (ex: endpoints locais).  
  - `.env.prod`: Variáveis para produção (URLs, credenciais, etc.).

- **Diretórios de Dados (`data/`)**:  
  Esse diretório existe somente no `infra`, armazena dados persistentes para banco de dados, certificados, uploads e logs. Não é versionado no Git.

## Fluxo de Desenvolvimento

1. **Clonar Repositórios**:  
   Clonar `infra`, `app1`, `app2` lado a lado no mesmo diretório `workspace/`.

2. **Iniciar Ambiente de Desenvolvimento**:  
   No diretório do `infra`:
   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.dev.yml up
   ```
   
   - O `docker-compose.dev.yml` irá construir as imagens dos apps a partir do código-fonte local (`build: ../app1/backend`), montar volumes (`../app1/backend:/usr/src/app`) para permitir hot-reload.
   - O Nginx (definido no `infra`) roteará as requisições para os apps.

3. **Iterar no Código**:  
   Qualquer modificação no código do backend ou frontend será refletida imediatamente no contêiner (graças ao volume montado).

4. **Testes Locais**:  
   Rodar testes tanto no host quanto dentro do contêiner. Ajustar o docker-compose.dev.yml para incluir serviços de teste se necessário.

## Fluxo de Produção

1. **Build das Imagens via CI/CD**:  
   No `app1` e `app2`, pipelines de CI/CD (GitHub Actions, GitLab CI, etc.) constroem as imagens do backend e frontend e fazem push para um registry:
   ```bash
   docker build -t meu-registry.com/app1-backend:version .
   docker push meu-registry.com/app1-backend:version
   ```

2. **Atualização da Produção**:  
   No servidor de produção, somente o `infra` é clonado. O código-fonte dos apps não é necessário.  
   
   No servidor:
   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.prod.yml pull
   docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
   ```
   
   O `prod` vai fazer `pull` das imagens do registry, subir os contêineres e servir o app via Nginx. Sem rebuild ou dependência do código local.

3. **Certificados e Segurança**:  
   O Certbot e o Nginx no `infra` cuidarão de HTTPS. Os certificados ficam em `data/certs/`. Renovação pode ser agendada com scripts e cron.

4. **Persistência de Dados**:  
   Banco de dados, uploads e logs ficam persistidos em `data/`. Assim, se o contêiner reiniciar, os dados permanecem.

5. **Escalabilidade Futura**:  
   Se necessário, pode-se migrar para orquestradores mais complexos (Kubernetes) ou dividir a infra. Mas o setup atual já fornece um caminho sólido para evoluir.

---

## Resumo

- **Repositório separado para infra e cada app**.
- Em **Desenvolvimento**: build a partir do código fonte local, hot-reload, volumes montados.
- Em **Produção**: uso de imagens pré-buildadas do registry, sem necessidade de código fonte no servidor.
- **Simplicidade e Manutenção**: A infra fica isolada, os apps independentes, e o deploy é facilitado.
- **Flexibilidade**: Adicionar novos apps é simples, basta criar novos repositórios, adicionar seus serviços ao `infra` e fazer referência a suas imagens no `docker-compose.prod.yml`.