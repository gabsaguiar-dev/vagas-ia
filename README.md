# 📡 Radar de Vagas — n8n + IA

Um workflow de **n8n** que monitora vagas de estágio/emprego **todos os dias**, usa **IA**
para dar uma nota de fit de 0 a 10 comparando cada vaga com o **seu** perfil, e envia um
ranking direto no **Telegram** — com medalhas para o top 3 e uma justificativa de cada nota.

Você configura o seu perfil uma vez, aponta para as APIs gratuitas de vagas, e recebe
todo dia só o que realmente tem a ver com você — em vez de navegar por dezenas de
anúncios manualmente.

```
Agendamento (diário) ─┬─ API Adzuna  → normaliza ─┐
                      └─ API Jooble  → normaliza ─┴─ junta → remove duplicados
                          → IA (Gemini) avalia todas as vagas em 1 chamada
                          → filtra por nota mínima + ordena por score
                          → 1 mensagem no Telegram por fonte
```

## Como funciona

- **Duas fontes de vagas** (Adzuna e Jooble), normalizadas para um formato único
- **IA como filtro de relevância**: cada vaga recebe um score de 0 a 10 e um motivo de
  uma frase dizendo o que bate ou falta no seu perfil. Só entram na mensagem as vagas
  com score acima da nota mínima configurável (padrão: 4)
- **Busca nacional** por padrão (sem cidade fixa) — você decide se quer filtrar por local
- **Mensagem por fonte**, com as vagas ranqueadas e link direto para cada uma
- **Fallback de segurança**: se a IA falhar, o robô envia as vagas mesmo sem ranking,
  em vez de perder o dia

## Pré-requisitos (tudo gratuito)

| Serviço | Para quê | Onde obter |
|---|---|---|
| n8n | Rodar o workflow (cloud ou self-hosted) | [n8n.io](https://n8n.io) ou Docker (abaixo) |
| Adzuna API | Fonte de vagas 1 (`app_id` + `app_key`) | [developer.adzuna.com](https://developer.adzuna.com) |
| Jooble API | Fonte de vagas 2 (chave por e-mail) | [br.jooble.org/api/about](https://br.jooble.org/api/about) ⚠️ use o domínio do **seu** país — a chave é vinculada a ele |
| Google Gemini | Avaliação das vagas (free tier, sem cartão) | [aistudio.google.com](https://aistudio.google.com) |
| Bot do Telegram | Receber o ranking | [@BotFather](https://t.me/BotFather) (token) + seu `chat_id` |

## Instalação

### Opção A — Docker (recomendado)

```bash
docker compose up -d
# abra http://localhost:5678 e crie a conta local (owner)
```

### Opção B — n8n Cloud

Crie uma conta em [n8n.io](https://n8n.io) e siga os mesmos passos de importação abaixo.

### Importar o workflow

1. No n8n: **Workflow → Import from File** → selecione o `workflow.json` deste repositório
2. Crie as credenciais (o import **não** carrega segredos — você precisa criar e reatribuir):
   - **Telegram**: credencial "Telegram API" com o token do BotFather
   - **Adzuna**: credencial "Custom Auth" com o JSON
     `{ "qs": { "app_id": "SEU_APP_ID", "app_key": "SUA_APP_KEY" } }`
   - **Gemini**: credencial "Google Gemini (PaLM) API" com a chave do AI Studio
3. Substitua os placeholders:
   - Nó **HTTP Request1** (Jooble): troque `SUA_CHAVE_JOOBLE` na URL pela sua chave
   - Nó **Send a text message**: troque `SEU_CHAT_ID` pelo seu chat_id
     (converse com o [@userinfobot](https://t.me/userinfobot) no Telegram para descobrir o seu)
4. **Publish** + ative o toggle **Active** para o agendamento diário funcionar

## Personalizar

| O quê | Onde |
|---|---|
| **Seu perfil** (o que a IA usa para avaliar) | Bloco `PERFIL DO CANDIDATO` no prompt do nó **Basic LLM Chain** — preencha os campos `<SUA ...>` |
| **Cidade / região** | Parâmetro `where` (Adzuna) e `location` (Jooble). Vazio = busca nacional |
| **Termos de busca** | Parâmetro `what` + `what_or` (Adzuna) e `keywords` (Jooble) |
| **Janela de tempo** | Parâmetro `max_days_old` (Adzuna) e `datecreatedfrom` (Jooble) — padrão: 7 dias |
| **Horário do envio** | Nó **Schedule Trigger** (+ timezone nas Settings do workflow) |
| **Nota mínima do ranking** | Constante `NOTA_MINIMA` no penúltimo nó Code |
| **Modelo de IA** | Nó **Google Gemini Chat Model** (qualquer chat model do n8n serve — OpenAI, Anthropic, Ollama local etc.) |

### Dica: termos de busca

A escolha dos termos faz toda a diferença na qualidade dos resultados:

- **Adzuna**: use `what` para o termo principal (ex.: `estágio`) e `what_or` para as
  áreas de interesse separadas por espaço (ex.: `TI desenvolvimento dados inteligência artificial`).
  Mantenha `sort_by=relevance` — ordenar por data costuma trazer ruído de vagas fora da área.
- **Jooble**: a busca é "fuzzy" (interpreta palavras como OU). Um termo enxuto
  (ex.: `estágio TI`) rende resultados muito melhores do que listar várias áreas.

## Alerta de erro (opcional, recomendado)

Crie um segundo workflow com **Error Trigger → Telegram** e o texto
`⚠️ Radar falhou: {{ $json.workflow.name }} — {{ $json.execution.error.message }}`,
publique-o e selecione-o em **Settings → Error Workflow** do workflow principal.

## Observações

- O free tier do Gemini cobre com folga o uso diário (1–2 chamadas/dia)
- Rodando self-hosted, o agendamento só executa com a máquina ligada — ajuste o horário
  para a sua rotina
- Os links das vagas apontam para as páginas originais (Adzuna/Jooble)
- Não há deduplicação entre execuções: se o mesmo anúncio continuar relevante, ele pode
  aparecer de novo no dia seguinte (é uma melhoria bem-vinda se isso te incomodar)

## Licença

[MIT](LICENSE)

---

Baseado no projeto [FelypeSantos-16/radar-vagas-n8n](https://github.com/FelypeSantos-16/radar-vagas-n8n),
expandido com busca nacional, segunda fonte de vagas (Jooble), ranking por IA com
justificativa e configurações personalizáveis.
