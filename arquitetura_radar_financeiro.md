# Arquitetura do Sistema — Radar Financeiro

> Documento de arquitetura técnico para implementação do projeto **Radar Financeiro**.

---

## 1. Visão geral

Radar Financeiro é uma aplicação web (dashboard) que agrega cotações, conversões de moeda, índices econômicos e notícias financeiras. A arquitetura proposta prioriza clareza para um projeto educacional, facilidade de manutenção e possibilidade de evolução.

**Principais objetivos arquiteturais**:

- Separação clara entre frontend e backend.
- API simples e bem documentada.
- Persistência mínima (histórico opcional) com migrações e controle de versão.
- Deploy containerizado (Docker) e configuração pronta para ambiente local e produção.

---

## 2. Visão em alto nível (componentes)

- **Frontend (Next.js + React)**

  - Páginas/Componentes: Header, Cotacoes, Conversor, Indices, Noticias, Footer.
  - Comunicação: HTTP/HTTPS para backend (REST). SSR/SSG para páginas estáticas (Next.js) quando conveniente.
  - Storybook para componentes isolados.

- **Backend (FastAPI)**

  - Endpoints REST: `/cotacoes`, `/converter`, `/indices`, `/noticias`, endpoints de auth (opcional), e endpoints administrativos para histórico.
  - Integrações: AwesomeAPI, CoinGecko, Banco Central (SGS), IBGE, NewsAPI.
  - Persistência: PostgreSQL via SQLAlchemy; Alembic para migrações.

- **Banco de Dados**

  - PostgreSQL (padrão para produção) — versões ou SQLite para desenvolvimento/instrução (opcional).

- **Infraestrutura e Deploy**

  - Docker + Docker Compose para desenvolvimento local.
  - Possível deploy em VPS, Docker Swarm, ou serviços como Render/Heroku/Cloud Run.

- **Observability**

  - Logs (stdout), health-check endpoints, métricas básicas (Prometheus expor em /metrics, opcional).

---

## 3. Diagrama de alto nível (ASCII)

```
[Client Browser] <----HTTPS----> [Next.js Frontend] <--HTTP--> [FastAPI Backend]
                                         |                     |
                                         |                     +--> External APIs: AwesomeAPI, CoinGecko, NewsAPI, BCB/IBGE
                                         |
                                         +--> CDN (opcional) / Static assets

Database: PostgreSQL <--- SQLAlchemy/Alembic ---> Backend
Cache: Redis (opcional) <--- Backend cache for frequent requests --->
```

---

## 4. Contratos de API (endpoints principais)

### GET /cotacoes

- **Descrição:** Retorna cotações atuais (BRL/USD, BRL/EUR, BTC/BRL, Ibovespa).
- **Query params (opcional):** `symbols=usd,eur,btc,ibov`
- **Resposta (200):**

```json
{
  "updated_at": "2025-08-12T12:00:00Z",
  "data": {
    "USD": { "price": 5.12, "change_pct": 0.35 },
    "EUR": { "price": 5.60, "change_pct": -0.12 },
    "BTC": { "price": 56000.12, "change_pct": 2.5 },
    "IBOV": { "index": 116000.5, "change_pct": -0.8 }
  }
}
```

### GET /converter

- **Descrição:** Converte um valor entre moedas usando a cotação atual (base BRL).
- **Query params:** `amount` (float), `to` (string, ex: "USD"), `from` (string, default "BRL")
- **Resposta (200):**

```json
{ "from": "BRL", "to": "USD", "amount": 100.0, "result": 19.53 }
```

### GET /indices

- **Descrição:** Retorna Selic, IPCA (12m), CDI, Poupança.
- **Resposta (200):**

```json
{
  "selic": 13.75,
  "ipca_12m": 4.9,
  "cdi": 13.65,
  "poupanca": 0.5,
  "as_of": "2025-08-01"
}
```

### GET /noticias

- **Descrição:** Retorna lista paginada de notícias.
- **Query params:** `page`, `page_size`, `q` (filtro, ex: economia)
- **Resposta (200):** Lista com título, url, source, imagem, publishedAt.

---

## 5. Modelo de dados (básico)

> O armazenamento não é obrigatório para o MVP, mas é recomendado para funcionalidades extras (histórico de cotações, log de conversões, favoritos do usuário).

### Tabelas sugeridas (PostgreSQL)

- **users** (opcional)

  - id (PK)
  - email
  - password\_hash
  - created\_at

- **conversion\_history**

  - id (PK)
  - user\_id (FK nullable)
  - from\_currency
  - to\_currency
  - amount
  - result
  - created\_at

- **quotes\_history** (opcional: registros periódicos)

  - id
  - symbol (USD, EUR, BTC, IBOV)
  - value
  - change\_pct
  - recorded\_at

- **favorites**

  - id
  - user\_id
  - symbol
  - created\_at

---

## 6. Caching e taxa de requisições (rate-limits)

- Muitas APIs públicas aplicam rate-limit. Estratégia recomendada:
  - **Cache no backend (Redis)** para resultados de `/cotacoes` e `/indices` por 30s a 60s.
  - **Caching de artigos** por 5 a 15 minutos.
  - Para ambiente sem Redis, usar cache em memória (LRU) com TTL.

---

## 7. Segurança

- **Autenticação:** JWT (PyJWT) + endpoints de login/register (se houver usuário). Para MVP público não precisa auth.
- **Segurança nas requisições às APIs externas:** guardar chaves em variáveis de ambiente (never commit `.env`).
- **Rate-limiting:** Implementar limite por IP (ex: 60 req/min) se o backend for público.
- **Headers de segurança:** HSTS, CSP (via Next.js), CORS configurado apenas para domínios permitidos.

---

## 8. Observabilidade e testes

- **Logs:** estrutura JSON simples emitida para stdout (compatível com Docker). Logger configurado no FastAPI (uvicorn logger + loguru/standard).
- **Health checks:** `/health` no backend que verifica conexões essenciais (DB opcional).
- **Métricas (opcional):** expor `/metrics` para Prometheus se quiser medir latência/throughput.
- **Testes:** usar pytest para testes unitários e de integração (mock das APIs externas).
- **Testes frontend:** Storybook + testes de componentes (Jest + React Testing Library).

---

## 9. CI / CD (sugestão simples)

- **CI** (GitHub Actions):
  - rodar `lint` (flake8/isort), `pytest` no backend e `npm test` no frontend, build de produção.
- **CD:**
  - Deploy automático no push para `main` (opcional) para Cloud Run / Render / DigitalOcean App Platform.

---

## 10. Dockerização e docker-compose (exemplo simplificado)

**docker-compose.yml (resumo)**

```yaml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - 8000:8000
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/radar
      - NEWSAPI_KEY=${NEWSAPI_KEY}
    depends_on:
      - db
      - redis

  frontend:
    build: ./frontend
    ports:
      - 3000:3000
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: radar
    volumes:
      - db_data:/var/lib/postgresql/data

  redis:
    image: redis:alpine

volumes:
  db_data:
```

**Observação:** configure `Dockerfile` no backend para usar `uvicorn` com gunicorn/uvicorn workers em produção e no frontend `next start`.

---

## 11. Fluxos de dados e sequência típica

1. O usuário abre o frontend (Next.js) — SSR/SSG entrega HTML/CSS/JS.
2. O componente `Cotacoes` faz requisição a `GET /cotacoes` no backend.
3. Backend verifica cache (Redis). Se cache inválido, consulta APIs externas (AwesomeAPI / CoinGecko), compõe resposta e atualiza cache.
4. Frontend renderiza os cartões com valores e variações. Um timer local solicita atualização a cada X segundos.
5. Conversões usam `GET /converter` (que internamente consulta cotação atual ou usa cache).
6. Notícias carregadas via `GET /noticias`, cache por 5-15 minutos.

---

## 12. Sugestão de roadmap de implementação (MVP -> v1)

**MVP (entregável para o curso):**

- Backend com endpoints `/cotacoes`, `/converter`, `/indices` (cache simples em memória).
- Frontend com páginas principais e gráficos estáticos (recharts), layout responsivo.
- Docker-compose para rodar localmente.

**v1 (após curso):**

- Adicionar `noticias` com integração NewsAPI e caching em Redis.
- Persistência de histórico de cotações e histórico de conversões.
- Autenticação simples e favoritos.
- CI com testes automatizados e deploy.

---

## 13. Variáveis de ambiente recomendadas

- `DATABASE_URL` — string de conexão com PostgreSQL
- `REDIS_URL` — conexão Redis
- `NEWSAPI_KEY` — chave NewsAPI
- `COINGECKO_KEY` — (se necessário)
- `AWESOMEAPI_KEY` — (se necessário)
- `JWT_SECRET` — segredo para tokens JWT
- `ENV` — development|production

---

## 14. Observações finais

O desenho aqui é propositalmente pragmático: fácil de implementar no prazo de um curso, com boas práticas que facilitam a evolução. Se quiser, eu adapto o documento para incluir diagramas UML (PNG/SVG), um `terraform` básico para deploy em nuvem, ou um arquivo `docker-compose` completo e `Dockerfile`s prontos para frontend e backend.

---

*FIM do documento.*

