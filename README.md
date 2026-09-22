# Code Review Assistant com IA — Passo a Passo

Sistema que analisa Pull Requests do GitHub automaticamente, dá feedback
baseado em boas práticas, e mantém histórico/métricas de qualidade de
código ao longo do tempo.

## Arquitetura geral

```
GitHub PR aberto/atualizado
        │  (webhook)
        ▼
┌─────────────────┐      ┌──────────────┐      ┌─────────────────┐
│  Backend FastAPI │─────▶│  LLM (API)   │      │   PostgreSQL     │
│  (webhook + API) │◀─────│  Anthropic/  │      │   (Supabase)     │
└────────┬─────────┘      │  OpenAI      │      └────────┬─────────┘
         │                 └──────────────┘               │
         │  comenta no PR via GitHub API                   │
         ▼                                                  │
   GitHub (comentário automático no PR)                     │
                                                              │
┌─────────────────┐                                          │
│ Frontend (React/ │◀─────────────────────────────────────────┘
│ HTML dashboard)  │  lê histórico e métricas
└──────────────────┘
```

## Stack sugerida (reaproveitando o que você já domina)

- **Backend:** FastAPI (Python) — você já usou nos projetos de IoT
- **Banco:** Supabase (Postgres) — você já usou
- **LLM:** API da Anthropic (Claude) ou OpenAI, via chamada HTTP simples
- **Frontend:** pode ser tão simples quanto uma página HTML+JS consumindo a API, ou React se quiser mais robustez
- **Hospedagem backend:** Render, Railway, ou Fly.io (têm tier gratuito)
- **Autenticação com GitHub:** GitHub App ou Personal Access Token (comece com PAT, é mais simples)

---

## Fase 1 — Fundação (webhook + persistência básica)

**Objetivo:** receber o evento do GitHub e salvar no banco, sem IA ainda.

1. Criar repositório do projeto com estrutura:
   ```
   code-review-assistant/
   ├── app/
   │   ├── main.py
   │   ├── webhooks.py
   │   ├── github_client.py
   │   ├── llm_client.py
   │   ├── models.py
   │   └── db.py
   ├── requirements.txt
   └── README.md
   ```

2. Criar um repositório de teste no GitHub (pode ser um repo qualquer seu)
   e configurar um **webhook** apontando pro seu endpoint (`Settings →
   Webhooks` no GitHub), evento `pull_request`.

3. Implementar endpoint que recebe o payload do webhook:
   ```python
   # app/main.py
   from fastapi import FastAPI, Request

   app = FastAPI()

   @app.post("/webhook/github")
   async def github_webhook(request: Request):
       payload = await request.json()
       action = payload.get("action")
       if action in ("opened", "synchronize"):
           pr_number = payload["pull_request"]["number"]
           repo_full_name = payload["repository"]["full_name"]
           # próximos passos: buscar o diff, processar
           return {"status": "recebido", "pr": pr_number}
       return {"status": "ignorado"}
   ```

4. Para testar localmente, use **ngrok** ou **localtunnel** pra expor seu
   `localhost` publicamente (o GitHub precisa de uma URL acessível pra
   enviar o webhook).

5. Criar a tabela inicial no Supabase:
   ```sql
   create table pull_requests (
       id bigserial primary key,
       repo_full_name text not null,
       pr_number int not null,
       titulo text,
       autor text,
       status text default 'recebido',
       criado_em timestamptz default now()
   );
   ```

**Checkpoint da Fase 1:** você abre um PR de teste e vê a linha aparecer
no banco. Sem IA ainda — só a esteira funcionando.

---

## Fase 2 — Buscar o diff e chamar o LLM

**Objetivo:** processar o conteúdo real do PR e gerar uma análise.

1. Buscar o diff do PR via API do GitHub:
   ```python
   # app/github_client.py
   import httpx

   GITHUB_TOKEN = "seu_personal_access_token"

   async def buscar_diff(repo_full_name: str, pr_number: int) -> str:
       url = f"https://api.github.com/repos/{repo_full_name}/pulls/{pr_number}"
       headers = {
           "Authorization": f"token {GITHUB_TOKEN}",
           "Accept": "application/vnd.github.v3.diff",
       }
       async with httpx.AsyncClient() as client:
           resp = await client.get(url, headers=headers)
           return resp.text
   ```

2. Definir o "playbook" de boas práticas como um prompt estruturado —
   aproveite o que você já estudou (SOLID, Clean Code do Gilded Rose):
   ```python
   # app/llm_client.py
   PLAYBOOK = """
   Você é um revisor de código sênior. Analise o diff abaixo e aponte:
   1. Violações de princípios SOLID, se houver
   2. Problemas de nomenclatura ou legibilidade
   3. Riscos de bugs ou edge cases não tratados
   4. Sugestões de melhoria concisas

   Responda em formato JSON com esta estrutura:
   {"resumo": "...", "problemas": [{"tipo": "...", "descricao": "...", "linha": "..."}], "nota_geral": 1-10}

   Diff:
   {diff}
   """

   async def analisar_diff(diff: str) -> dict:
       # chamada à API da Anthropic ou OpenAI aqui
       ...
   ```

3. Pedir resposta em **JSON estruturado** (não texto livre) — isso facilita
   salvar no banco e exibir no frontend depois. Modelos de LLM modernos
   seguem bem instruções de formato quando você é explícito.

4. Salvar o resultado da análise numa nova tabela:
   ```sql
   create table analises (
       id bigserial primary key,
       pr_id bigint references pull_requests(id),
       resumo text,
       problemas jsonb,
       nota_geral int,
       criado_em timestamptz default now()
   );
   ```

**Checkpoint da Fase 2:** abrir um PR de teste gera uma análise real salva
no banco, com problemas identificados pelo LLM.

---

## Fase 3 — Comentar automaticamente no PR

**Objetivo:** fechar o ciclo — a IA não só analisa, ela **age** (posta o
comentário de volta no GitHub).

```python
async def comentar_no_pr(repo_full_name: str, pr_number: int, comentario: str):
    url = f"https://api.github.com/repos/{repo_full_name}/issues/{pr_number}/comments"
    headers = {"Authorization": f"token {GITHUB_TOKEN}"}
    async with httpx.AsyncClient() as client:
        await client.post(url, headers=headers, json={"body": comentario})
```

Formate o `comentario` de forma legível (Markdown), reaproveitando o JSON
da análise — isso é o momento "uau" pro vídeo: mostrar um PR real
recebendo um comentário automático da sua IA.

**Checkpoint da Fase 3:** PR de teste recebe comentário automático visível
no GitHub.

---

## Fase 4 — Dashboard (frontend)

**Objetivo:** visualizar o histórico e métricas — a camada que mostra
"sistema completo", não só um script.

Endpoints necessários no backend:
```python
@app.get("/api/prs")
async def listar_prs():
    # retorna lista de PRs analisados com nota geral

@app.get("/api/prs/{pr_id}")
async def detalhe_pr(pr_id: int):
    # retorna análise completa de um PR

@app.get("/api/metricas")
async def metricas_gerais():
    # nota média ao longo do tempo, problemas mais comuns, etc.
```

No frontend, comece simples: uma tabela com os PRs analisados + nota, e
um gráfico de linha da nota média ao longo do tempo (dá pra usar Chart.js
ou Recharts). Não precisa ser bonito na primeira versão — funcional
primeiro, polimento depois.

**Checkpoint da Fase 4:** você abre a página e vê o histórico real de PRs
analisados, com gráfico.

---

## Fase 5 — Observabilidade e polimento

**Objetivo:** cobrir o bloco de MLOps/Observabilidade do edital de forma
concreta.

1. Logar cada chamada ao LLM: tempo de resposta, tokens usados, custo
   estimado — numa tabela `logs_llm`.
2. Adicionar um campo pro desenvolvedor "avaliar" se a sugestão da IA foi
   útil (👍/👎) — isso vira uma métrica de qualidade real do sistema ao
   longo do tempo, e é um ótimo dado pra mostrar no vídeo.
3. Tratamento de erros: o que acontece se a API do LLM falhar? Tenha um
   fallback (retry simples, ou marcar como "falhou" no banco sem quebrar
   o webhook).

---

## Ordem de prioridade se o tempo apertar

Se em algum momento o tempo ficar curto, essa é a ordem de corte (do que
cortar primeiro pro que nunca cortar):

1. ✂️ Dashboard bonito — funcional simples já basta
2. ✂️ Sistema de 👍/👎 — legal mas não essencial
3. ⚠️ Métricas agregadas — importante, mas pode ser uma versão simples
4. 🔒 Webhook + LLM + comentário automático no PR — **este é o núcleo do
   projeto, nunca corte isso**

## Próximos passos imediatos

1. Criar o repositório e a estrutura de pastas
2. Gerar um Personal Access Token no GitHub (`Settings → Developer
   settings → Personal access tokens`)
3. Criar conta/pegar chave de API do LLM que for usar
4. Implementar a Fase 1 (webhook + banco) e testar com ngrok antes de
   avançar pra Fase 2
