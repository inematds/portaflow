# PortaFlow — Plano de Execução (máquina de vendas B2B de portas via WhatsApp)

> **Para trabalhadores agênticos (Sonnet/Opus):** este é o plano MESTRE. Antes de codar cada fase,
> gere o plano detalhado da fase com a skill `superpowers:writing-plans` (formato TDD, tarefas
> bite-sized) salvando em `doc/planos/AAAA-MM-DD-fase-N-<nome>.md`, e execute com
> `superpowers:subagent-driven-development` (recomendado) ou `superpowers:executing-plans`.
> Passos usam checkbox (`- [ ]`) para rastreio. Feche o loop SEMPRE: rodar `pytest` após editar,
> reportar resultado real. Base de decisões: `doc/validacao_pesquisa.md` (D1–D7) e
> `doc/lista_dados_necessarios.md` (dados que o Nei fornece).

**Objetivo:** transformar leads B2B (lojas e revendas de portas internas brancas) em pedidos pagos
com mínima intervenção humana: WhatsApp → IA qualifica → orçamento determinístico → link Pix (sinal)
→ follow-up → humano só em exceção, com notificações internas via Telegram e funil medido até margem.

**Arquitetura:** monólito modular Python (FastAPI) + Postgres + Redis, com **camada de adaptador de
canal** (Evolution API agora, WhatsApp Cloud API oficial na escala) e **motor de orçamento
determinístico** separado do LLM (o LLM extrai e narra; o código calcula). Multi-tenant desde o dia 1
(`tenant_id` em toda tabela de domínio) para virar SaaS depois. Operação interna via bot Telegram.

**Stack:** Python 3.12 · FastAPI · SQLAlchemy 2 + Alembic · Postgres 16 · Redis 7 · Docker Compose ·
Evolution API v2 (container) · anthropic SDK (Claude) · pydantic v2 · aiogram 3 (Telegram) ·
Asaas (pagamentos) · pytest.

## Restrições globais (valem para TODAS as tarefas)

- **Preço NUNCA sai do LLM.** Todo valor em R$ mostrado ao cliente vem de `catalog/pricing.py`
  (função pura, testada com golden tests). Item fora do catálogo → nenhum número, nem faixa (D1).
- **Guardrail de saída:** antes de enviar, validar que qualquer valor monetário na resposta do agente
  bate com o retorno das tools daquele turno; divergiu → bloquear e reformular.
- **Orçamento = ID curto único + validade explícita + versão da tabela de preço.** Tabelas de preço
  são versionadas: nunca sobrescrever, criar registro novo e desativar o antigo (D1/CDC).
- **Dedup de webhook** por constraint UNIQUE no banco (`provider_message_id`) com
  `ON CONFLICT DO NOTHING`; **debounce** de ~10 s (Redis) antes de processar rajada (D7).
- **Origem do lead capturada na PRIMEIRA mensagem** (`ctwa_clid`/código curto da landing) e
  persistida imediatamente — depois é irrecuperável (D4).
- **Conversa enxuta:** desenho para ≤ ~6 trocas até o orçamento; mensagens agrupadas. A partir de
  out/2026 a API oficial cobra por mensagem de serviço (D3).
- **Handoff:** humano assume (comando Telegram) → bot pausa naquela conversa; mensagens do bot são
  marcadas para nunca serem confundidas com intervenção humana; "falar com atendente" sempre
  disponível ao cliente (D7).
- **Sem disparo frio** pelo número do robô (Evolution): só responder quem escreveu primeiro.
  Reativação de base antiga fica para a fase da API oficial (D2).
- **`tenant_id`** em toda tabela de domínio; consultas sempre filtradas por tenant.
- **LGPD:** aviso de uso de dados no primeiro contato; PII hasheada (SHA-256) antes de enviar a
  Meta/Google; valores de tokens/chaves jamais impressos em log.
- **Segredos:** carregar em runtime de `.env` local (chaves de LLM já existem em
  `~/projetos/openpcbotv2/.env` ou `~/projetos/wifi/.env`); nunca copiar valores para o repo.
- **Modelos LLM:** default conversa `claude-sonnet-5`, triagem/extração barata
  `claude-haiku-4-5-20251001`; ambos configuráveis por env. Prompt caching no system prompt/catálogo.
- **Mensagens ao cliente em PT-BR**, tom profissional B2B, sem emoji em excesso.
- **Git:** autor/committer `inematds <inematds@gmail.com>`; commits pequenos e frequentes;
  versão semver `vX.XX.YY` na regra do Nei (patch incrementa YY; minor incrementa XX e CARREGA o YY;
  só major zera tudo).
- **TDD:** teste falhando → implementação mínima → teste passando → commit. `pytest` verde antes de
  qualquer "pronto".

## Estrutura de arquivos (mapa do monólito)

```
portaflow/
├── docker-compose.yml          # postgres, redis, evolution-api, app
├── .env.example                # todas as vars documentadas, sem valores reais
├── pyproject.toml
├── alembic/                    # migrations
├── app/
│   ├── main.py                 # FastAPI: /webhook/evolution, /webhook/asaas, /health
│   ├── config.py               # pydantic-settings, carrega .env
│   ├── db.py                   # engine, session, Base
│   ├── models/                 # SQLAlchemy: tenant, lead, conversation, message,
│   │                           #   product, price_table, quote, order, payment,
│   │                           #   followup, event
│   ├── channels/
│   │   ├── base.py             # ChannelAdapter (interface)
│   │   ├── evolution.py        # adapter Evolution API v2
│   │   └── cloud_api.py        # adapter oficial (Fase 7)
│   ├── ingest/
│   │   ├── webhook.py          # parse + dedup + persist origem (referral/código curto)
│   │   └── debounce.py         # Redis, junta rajada em um turno
│   ├── agent/
│   │   ├── engine.py           # loop do turno: contexto → Claude → tools → guardrail → envio
│   │   ├── slots.py            # pydantic: PerfilLead, PedidoPortas (validação de faixas)
│   │   ├── tools.py            # definições tool-use p/ Claude
│   │   ├── guardrails.py       # checagem de R$ vs retorno de tools; alçada de desconto
│   │   └── prompts/            # system prompt, few-shots (arquivos .md)
│   ├── catalog/
│   │   ├── importer.py         # importa catálogo real do Nei (formato definido na Fase 2)
│   │   └── pricing.py          # MOTOR DETERMINÍSTICO: itens+perfil+qtd+CEP → orçamento
│   ├── quotes/service.py       # cria quote (ID curto, validade, versão de tabela), render msg
│   ├── payments/asaas.py       # cria cobrança (externalReference=quote_id), webhook confirmação
│   ├── followup/scheduler.py   # cadências (1h/24h/72h/7d), fila no Postgres, worker
│   ├── crm/service.py          # estágios, score, motivos de perda, métricas de funil
│   ├── tracking/
│   │   ├── attribution.py      # ctwa_clid, código curto landing, gclid
│   │   └── conversions.py      # Meta CAPI + Google Data Manager API (Fase 6)
│   └── ops/
│       └── telegram_bot.py     # aiogram: notificações + /assumir /liberar /desconto /resumo
├── landing/                    # Fase 6 (página estática + short-codes)
├── tests/                      # espelha app/; golden tests de pricing; evals de conversa
└── doc/                        # docs existentes + planos por fase em doc/planos/
```

## Modelo de dados (núcleo — detalhar nas migrations da Fase 0/2)

| Tabela | Campos-chave |
|---|---|
| `tenants` | id, nome, config JSON |
| `leads` | id, tenant_id, wa_id/telefone (unique por tenant), nome, perfil (`loja\|revenda\|construtor\|arquiteto\|consumidor`), cnpj?, cidade, uf, cep, origem (`ctwa_clid\|utm\|gclid\|codigo_curto\|organico`), score (`quente\|morno\|frio\|ruim`), estagio (`novo→qualificando→qualificado→orcado→negociando→fechado\|perdido\|escalado`), motivo_perda?, created_at |
| `conversations` | id, tenant_id, lead_id, status (`bot\|humano\|encerrada`), canal |
| `messages` | id, conversation_id, direcao, autor (`lead\|bot\|humano`), texto, provider_message_id **UNIQUE**, ts |
| `products` + `price_tables` | esquema final definido na Fase 2 a partir do catálogo real; `price_tables` com `versao`, `valid_from`, `ativo`, degraus de volume, perfil |
| `quotes` | id curto (ex. `PF-2026-0001`), tenant_id, lead_id, itens JSON, subtotal, frete, total, validade_ate, price_table_versao, status (`rascunho→enviado→aceito→expirado→cancelado`) |
| `orders` | quote_id, valor_sinal, valor_saldo, status (`aguardando_sinal→sinal_pago→saldo_cobrado→pago→entregue`) |
| `payments` | asaas_id, external_reference=quote_id, tipo (`sinal\|saldo`), status, valor, ts |
| `followups` | lead_id, due_at, tipo, payload, status |
| `events` | tenant_id, lead_id, tipo (`lead_novo\|lead_qualificado\|orcamento_enviado\|venda\|perda`), valor, atribuicao JSON, enviado_capi?, enviado_google?, ts |

## Interfaces centrais (contratos entre fases)

```python
# app/channels/base.py
class ChannelAdapter(Protocol):
    async def send_text(self, tenant_id: int, to: str, text: str) -> str: ...   # -> provider_message_id
    async def send_image(self, tenant_id: int, to: str, url: str, caption: str = "") -> str: ...
    def parse_webhook(self, payload: dict) -> IncomingMessage | None: ...
    # IncomingMessage: wa_id, texto, provider_message_id, ts, referral: dict | None

# app/catalog/pricing.py  — PURA, sem I/O, 100% coberta por golden tests
def calcular_orcamento(itens: list[ItemPedido], perfil: str, qtd_total: int,
                       cep: str, tabela: PriceTable) -> Orcamento: ...
# Orcamento: linhas[], subtotal, frete, total, tabela_versao — Decimal, nunca float

# app/agent/tools.py — tools expostas ao Claude
consultar_catalogo(filtros)          # busca produtos; sem preço unitário solto
calcular_orcamento(itens, cep)       # chama pricing.py; única fonte de R$
escalar_humano(motivo)               # pausa bot + notifica Telegram
agendar_followup(quando, contexto)
gerar_link_pagamento(quote_id, tipo) # Asaas; tipo: sinal|saldo (Fase 4)

# Valor proxy do lead qualificado (D4): valor = qtd_portas × ticket_medio_por_porta (env, recalibrar mensal)
```

---

## Fases (cada uma vira um plano TDD detalhado antes de codar)

### Fase 0 — Fundação ✅ pré-requisito: nenhum
Repo git iniciado (autor `inematds`), `pyproject.toml`, `docker-compose.yml` (postgres, redis,
evolution-api v2 com sua própria instância de banco, app), `config.py`, `db.py`, migrations iniciais
(tenants, leads, conversations, messages, events, followups), seed do tenant 1, `.env.example`,
pytest configurado com banco de teste.
**Aceite:** `docker compose up` sobe tudo; `pytest` verde; `GET /health` responde; migration
aplica e reverte limpa.

### Fase 1 — Canal WhatsApp + ingest + Telegram ops ✅ pré-requisito: Fase 0; chip dedicado para testar de verdade
`ChannelAdapter` + `EvolutionAdapter` (conexão por QR, envio/recebimento), webhook `/webhook/evolution`
com **dedup** (UNIQUE + ON CONFLICT) e **debounce** (Redis 10 s), persistência de mensagens e
**captura de origem na primeira mensagem** (objeto `referral`/`ctwa_clid` quando existir; código curto
`#PF-xxxx` no texto quando vier da landing), criação automática de lead. Bot Telegram (aiogram) com
notificações (lead novo, pedido de humano) e comandos `/assumir <lead>`, `/liberar <lead>`,
`/resumo` (digest do dia). Eco de teste: mensagem recebida → resposta fixa de confirmação.
**Aceite:** conversa real de um segundo número chega, não duplica sob replay do webhook, rajada de
3 mensagens vira 1 turno, origem persistida, notificação chega no Telegram, `/assumir` pausa o eco.

### Fase 2 — Catálogo + motor de orçamento 🔒 BLOQUEADA por: itens A–C de `doc/lista_dados_necessarios.md`
Ao receber a amostra real do catálogo do Nei: definir schema `products`/`price_tables` espelhando a
estrutura real (medidas, acabamentos, componentes do kit, degraus de volume, perfis, frete por
região), `importer.py` idempotente, `pricing.py` puro com **golden tests** (planilha de casos
validada pelo Nei: cada linha = itens+perfil+qtd+CEP → total esperado, `Decimal`), `quotes/service.py`
(ID curto, validade default 7 dias, versão de tabela, render da mensagem de orçamento).
**Aceite:** todos os golden tests passam; reimportar catálogo não duplica; orçamento reproduzível
por ID com a versão de tabela da época mesmo após atualização de preços.

### Fase 3 — Agente vendedor IA ✅ pré-requisito: Fases 1–2
`slots.py` (pydantic, faixas validadas: largura 40–120 cm etc.), `engine.py` (turno: histórico +
system prompt cacheado → Claude com tools → guardrail → envio), `guardrails.py` (R$ na resposta ≡
retorno de tool; desconto acima da alçada → `escalar_humano`), prompts em `agent/prompts/` (persona
vendedor B2B, coleta agrupada: perfil+cidade → mix de portas+medidas+acabamento → prazo; aviso LGPD
no primeiro contato; sempre oferecer atendente), scoring quente/morno/frio ao qualificar, evento
`lead_qualificado` com valor proxy. **Eval harness:** conversas simuladas em
`tests/evals/*.jsonl` (lead ideal, lead vago, pedido fora do catálogo, pedido de desconto, pedido de
humano) rodadas contra o engine com asserts (nunca inventou preço; escalou quando devia; ≤ N trocas
até orçamento).
**Aceite:** evals verdes; conversa real ponta a ponta: lead novo → qualificado → orçamento correto
enviado no WhatsApp → notificação "orçamento enviado" no Telegram.

### Fase 4 — Fechamento + pagamento ✅ pré-requisito: Fase 3; conta Asaas (sandbox primeiro)
`payments/asaas.py` (criar cobrança Pix com `externalReference=quote_id`, webhook
`/webhook/asaas` idempotente), fluxo sinal (% configurável por tenant) + saldo (segunda cobrança
disparada por comando ou gatilho de entrega), `orders`, aceite do orçamento no chat ("fechar pedido")
→ link do sinal → confirmação → evento `venda` + notificação Telegram com 🎉.
**Aceite:** no sandbox, ciclo completo: aceite → link → pagamento simulado → order `sinal_pago` →
Telegram notificado; replay do webhook não duplica pagamento.

### Fase 5 — Follow-up + CRM mínimo ✅ pré-requisito: Fase 3 (ideal: 4)
`followup/scheduler.py` (worker: cadência default 1h/24h/72h/7d para orçamento sem resposta, cancela
ao responder/fechar; mensagens curtas com argumento diferente por toque; respeitando "sem disparo
frio" — follow-up é continuação de conversa iniciada pelo lead), motivos de perda (IA classifica ao
encerrar), `crm/service.py` com métricas do funil (investimento→leads→qualificados→orçamentos→
vendas→receita→margem), `/resumo` do Telegram vira relatório diário com o funil.
**Aceite:** orçamento ignorado gera follow-ups nos horários certos e para quando o lead responde;
`/resumo` mostra números reais do banco.

### Fase 6 — Captação + tracking de conversões ✅ pré-requisito: Fase 5 rodando com leads reais
Landing page (uma página, mobile-first, formulário mínimo → botão WhatsApp com código curto
`#PF-xxxx` gerado por request, guardando gclid/UTMs), `tracking/conversions.py`: Meta CAPI
(`action_source=business_messaging`, `ctwa_clid`, eventos `Lead` qualificado com valor proxy e
`Purchase` com valor real) e Google **Data Manager API** (não a API legada — corte 15/06/2026),
PII hasheada. Guia de campanha: Google Search (termos de fundo de funil B2B) e Meta CTWA
(objetivo Engajamento/Leads, nunca Tráfego), otimizando por `Lead` qualificado (D4).
**Aceite:** clique de anúncio de teste → lead com origem correta → evento aparece no Gerenciador de
Eventos da Meta / diagnóstico do Google; funil por origem no `/resumo`.

### Fase 7 — Migração para API oficial 🔒 gatilho: 1º dos três — (a) >2.000 conversas/mês, (b) receita recorrente provada (ex. 3 meses de vendas), (c) qualquer sinal de restrição no número Evolution
`CloudAPIAdapter` (mesma interface), verificação Meta Business, templates aprovados para follow-up
fora da janela de 24h, opt-in por categoria, orçar custo/mensagem com a tarifa vigente
(pós-out/2026), reativação da base antiga (agora permitida) como campanha de templates.
**Aceite:** trocar `CHANNEL=cloud_api` no env migra o canal sem tocar em agent/pricing/crm; suíte
inteira verde nos dois adapters (testes de contrato da interface).

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Ban do número no Evolution (real, documentado) | Chip dedicado; inbound-only; sem massa fria; backup: CRM/conversas no NOSSO banco (ban não perde nada); número reserva preparado; gatilho (c) da Fase 7 |
| IA inventar preço (incidente real em case análogo; CDC pode vincular oferta) | D1: motor determinístico + guardrail de saída + quote com ID/validade/versão + confirmação antes de cobrança |
| Custo por mensagem pós-out/2026 na oficial | Conversa enxuta por design desde a Fase 3; medir msgs/conversa no funil |
| Webhook duplicado / rajadas (caso real: 837 duplicatas) | Dedup por UNIQUE + debounce Redis (Fase 1, testado) |
| Lead sem atribuição | Captura na 1ª mensagem (Fase 1), código curto na landing (Fase 6) |
| LGPD / transferência internacional | Aviso no 1º contato; hash de PII; consulta jurídica antes de escalar (registrada como pendência) |
| Dependência dos dados do catálogo | Fase 2 bloqueada explicitamente; Fases 0–1 não dependem; cobrar itens A–C da lista |

## Ordem de execução e paralelismo

`F0 → F1 → (F2 quando dados chegarem) → F3 → F4 → F5 → F6 → F7`.
F2 pode andar em paralelo com F1 assim que o catálogo chegar. F5 pode começar pelo scheduler em
paralelo com F4. Nada de F3 antes de F2 (agente sem motor de preço viola D1).

## Definição de pronto do MVP (fim da Fase 4)

Um lead real que chega pelo WhatsApp é qualificado pela IA, recebe orçamento correto calculado do
catálogo real, pode pagar o sinal por Pix, e o Nei recebe no Telegram: lead quente, orçamento
enviado, pedido de humano e pagamento confirmado — com todos os números do funil consultáveis via
`/resumo`.
