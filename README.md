# Sheets + Worker externo + Playwright para análise competitiva no Mercado Livre

## Objetivo

Para cada GTIN/EAN + URL de exemplo informados na planilha:

- localizar anúncios equivalentes no Mercado Livre
- validar se é realmente o mesmo produto
- separar kits/combos/variações diferentes
- enriquecer com vendedor, preço, frete, total estimado, vendas/proxy, nota/avaliações, URL
- devolver resumo executivo, top 10, menor/maior/médio e conclusão prática

## Por que esta arquitetura

A API oficial do Mercado Livre está muito boa para autenticação, detalhes de item, preço e listagens por vendedor conhecido, mas não expõe de forma estável uma busca aberta de concorrentes cross-seller para o caso de uso competitivo. A documentação atual de Items & Searches destaca busca/listagem por seller e seller_id, e o fluxo de User Products também é centrado no seller. Já GTIN é um identificador oficial de produto e o access token deve ser usado no header Authorization, com renovação por refresh token.

Por isso a arquitetura usa:

1. **Playwright** para descobrir concorrentes na SERP pública do marketplace.
2. **API oficial** para enriquecer e validar os anúncios encontrados.
3. **Google Sheets / Apps Script** só como entrada e apresentação.

## Arquitetura

```text
Google Sheets
   │
   │ Apps Script (bridge)
   ▼
Worker HTTP externo (Node + Express)
   ├── Autenticação ML (access_token + refresh_token)
   ├── Playwright (descoberta na busca pública)
   ├── ML API (/items, /users, /sale_price)
   ├── Validador de equivalência (GTIN / catálogo / marca / modelo / categoria)
   └── Relatório JSON
   ▼
Apps Script escreve:
   - INDICE
   - TOP_<GTIN>
   - RESUMO_<GTIN>
   - CONCLUSAO_<GTIN>
   - DIAG_<GTIN>
```

## Fluxo

### 1) Entrada na planilha
Usuário informa de 1 a 10 linhas no formato:

```text
7893595568672 | https://www.mercadolivre.com.br/.../up/MLBU...?...item_id:MLB...
```

### 2) Bridge no Apps Script
O Apps Script faz `POST /analyze` no worker e envia:

```json
{
  "items": [
    {"gtin": "7893595568672", "url": "https://..."}
  ]
}
```

### 3) Worker cria a âncora do produto
- extrai `item_id`, `catalog_product_id` e `user_product_id` da URL
- consulta `/items/{id}` quando há `item_id`
- consolida sinais fortes:
  - GTIN
  - marca
  - modelo
  - categoria
  - catálogo
  - seller da âncora

### 4) Worker descobre candidatos
- monta queries a partir de GTIN, modelo, marca+modelo e título
- usa Playwright para abrir a busca pública do Mercado Livre
- captura cards da listagem e extrai:
  - itemId
  - título
  - URL
  - preço visível
  - frete grátis
  - avaliações/nota quando houver

### 5) Worker enriquece candidatos
- `/items?ids=...&include_attributes=all`
- `/users?ids=...`
- `/items/{id}/sale_price?context=channel_marketplace`
- fallback opcional: abrir página do item com Playwright

### 6) Validação de equivalência
O score prioriza:
- mesmo GTIN
- mesmo `catalog_product_id`
- mesmo modelo
- mesma marca
- mesma categoria
- similaridade de título

Kits/combos recebem penalidade e continuam separados.

### 7) Relatório
O worker devolve JSON pronto para a planilha escrever.

## Estrutura do projeto

```text
ml_sheets_playwright_bundle/
  ├─ .env.example
  ├─ package.json
  ├─ README.md
  ├─ src/
  │  ├─ config.js
  │  ├─ ml-auth.js
  │  ├─ ml-api.js
  │  ├─ normalize.js
  │  ├─ scraper.js
  │  ├─ matcher.js
  │  ├─ report.js
  │  └─ server.js
  └─ apps-script/
     └─ bridge.gs
```

## Instalação local

```bash
npm install
cp .env.example .env
# preencha as variáveis
npm start
```

## Variáveis necessárias

No worker:

- `ML_CLIENT_ID`
- `ML_CLIENT_SECRET`
- `ML_ACCESS_TOKEN`
- `ML_REFRESH_TOKEN`
- `WORKER_API_KEY`

No Apps Script:

- `WORKER_BASE_URL`
- `WORKER_API_KEY`

## Endpoints do worker

### `GET /health`
Resposta simples de saúde.

### `POST /analyze`
Corpo:

```json
{
  "items": [
    {
      "gtin": "7893595568672",
      "url": "https://www.mercadolivre.com.br/..."
    }
  ]
}
```

Resposta:

```json
{
  "ok": true,
  "generatedAt": "2026-03-13T12:00:00.000Z",
  "results": [
    {
      "anchor": {"title": "...", "gtin": "..."},
      "stats": {"min": 100, "max": 200, "avg": 150, "strongestSeller": "Loja X"},
      "rows": [...],
      "executiveSummary": {...},
      "diagnostics": {...}
    }
  ]
}
```

## Deploy sugerido

Você pode subir o worker em qualquer ambiente Node que rode Chromium do Playwright:

- Cloud Run
- Railway
- Render
- VM/Docker próprio

## Recomendações práticas

- limite a análise a 1–10 itens por execução
- mantenha a concorrência do worker baixa
- trate Playwright como fonte principal de descoberta, não a API
- trate a API como fonte principal de **enriquecimento e validação**, não de descoberta
- guarde logs de diagnóstico por execução
- rotacione client secret e tokens se eles tiverem sido expostos

## Limitações honestas

- avaliações/nota podem continuar como `não visível` em alguns anúncios
- frete total exato pode não estar disponível publicamente para todos os cards
- páginas do Mercado Livre mudam o HTML com frequência, então o scraper precisa de manutenção leve periódica

## Próximos passos recomendados

1. subir o worker externo
2. testar `/health`
3. configurar `WORKER_BASE_URL` no Apps Script
4. rodar 1 GTIN real
5. revisar a aba `DIAG_<GTIN>`
6. ajustar heurísticas por categoria, se necessário
