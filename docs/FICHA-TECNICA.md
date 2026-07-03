# Ficha técnica — Granja União (web)

Atualizado: 2026-05-26

## Dois sistemas diferentes

| URL | O quê | Onde roda | Status (26/05/2026) |
|-----|--------|-----------|---------------------|
| **https://granjauniao.com.br** | Site institucional (vitrine, WhatsApp, pedido no site) | GitHub Pages | **Online** (HTTP 200) |
| **https://app.granjauniao.com.br** | Painel admin + API (`/api/*`) | VPS Linux + Docker | **Fora** (HTTP **502**) |

Se o “site não abre”, confira **qual URL**:
- Só o **cardápio/pedido/painel** falha → problema na **API** (`app.`).
- A página principal não carrega → GitHub Pages / DNS do domínio raiz.

---

## app.granjauniao.com.br — Plataforma Ovo

**Repositório:** https://github.com/RivasCode-Ops/Plataforma_ovo  
**Pasta local (se clonar):** `Plataforma_ovo/` (não está no workbench por padrão)

### Função

Sistema de pedidos da granja: estoque, pedidos (WhatsApp, site, presencial), painel administrativo.

### Stack

| Camada | Tecnologia |
|--------|------------|
| API | Node.js 20 + Express (`backend/`) |
| Banco | PostgreSQL 16 (`postgres` no Docker) |
| Painel | React + Vite + Tailwind (`frontend/`) |
| Proxy | Nginx no container `web` + Nginx no host (SSL) |
| Deploy | `infra/docker-compose.prod.yml` |

### DNS (produção)

| Registro | Destino |
|----------|---------|
| `app.granjauniao.com.br` | **A** → `147.93.185.146` (VPS) |

### URLs na VPS

| Caminho | Uso |
|---------|-----|
| `/` | Painel administrativo |
| `/api/health` | Saúde da API |
| `/api/cardapio` | Cardápio público (site) |
| `/api/pedido-site` | Pedido do site React (token) |
| `/api/webhook/granjauniao` | Webhook server-side |

### Variáveis críticas (`infra/.env.prod`)

- `POSTGRES_PASSWORD`, `DATABASE_URL`
- `ADMIN_USER`, `ADMIN_PASSWORD`, `JWT_SECRET`
- `GRANJA_WHATSAPP`
- `SITE_PEDIDO_TOKEN` (mesmo valor do secret `VITE_SITE_PEDIDO_TOKEN` no GitHub do site)
- `CORS_ORIGIN` — deve incluir `https://granjauniao.com.br`

### Containers Docker

| Nome | Serviço |
|------|---------|
| `plataforma_ovo_db` | PostgreSQL |
| `plataforma_ovo_api` | Backend (porta interna 3000) |
| `plataforma_ovo_web` | Nginx + frontend (porta 80 no host) |

---

## granjauniao.com.br — Site

**Repositório:** https://github.com/RivasCode-Ops/granjauniao-site  
**Pasta local:** `D:\PROJETOS\06_CLIENTES\granjauniao-site`

### Stack

| Camada | Tecnologia |
|--------|------------|
| UI | React 19 + Vite 8 + Tailwind |
| Hospedagem | GitHub Pages |
| Domínio | `public/CNAME` → `granjauniao.com.br` |
| CI | `.github/workflows/pages.yml` |

### Integração com a API

Build usa `VITE_API_URL=https://app.granjauniao.com.br`.  
Se a API está em 502, o site **abre**, mas **cardápio/pedido online** mostra indisponível.

---

## Por que app.granjauniao.com.br está fora (502)

**Diagnóstico:** o DNS e o **Nginx na VPS respondem**, mas o **serviço atrás do proxy não responde**.

```
Internet → Nginx (Ubuntu, 147.93.185.146) → proxy → Docker :80 → ❌ backend/web parado ou com erro
```

**502 Bad Gateway** = Nginx no ar, **upstream morto** (containers Docker parados, crash do Node, ou só SSL no host sem `docker compose up`).

### Causas mais comuns

1. VPS reiniciou e os containers não subiram (`restart: unless-stopped` ajuda, mas falha se Docker não iniciar).
2. `docker compose` nunca foi rodado após deploy.
3. Container `backend` ou `postgres` em loop de crash (senha/env errado).
4. Disco cheio / memória na VPS.
5. Deploy incompleto (só certificado SSL no host, app não publicada).

### O que fazer na VPS (SSH)

```bash
ssh usuario@147.93.185.146
cd Plataforma_ovo/infra   # ou onde o projeto está
docker compose --env-file .env.prod -f docker-compose.prod.yml ps
docker compose --env-file .env.prod -f docker-compose.prod.yml logs -f --tail=80 backend
curl -s http://127.0.0.1/api/health
```

Se `ps` estiver vazio ou unhealthy:

```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --build
```

Quando `curl http://127.0.0.1/api/health` retornar OK, `https://app.granjauniao.com.br` deve voltar.

---

## Referências

- Deploy: `Plataforma_ovo/docs/DEPLOY.md`
- Integração site: `Plataforma_ovo/docs/INTEGRACAO_GRANJAUNIAO.md`
- Decisões: `Plataforma_ovo/docs/DECISOES_TECNICAS.md`
