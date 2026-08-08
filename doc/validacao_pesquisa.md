# PortaFlow — Validação da visão contra os melhores sistemas (pesquisa 2026-08-08)

> Síntese de 3 pesquisas paralelas na internet (stack WhatsApp+IA, plataformas prontas BR, tracking+pagamentos),
> todas com fontes primárias verificadas na data. Rótulos: **[primária]** = doc oficial; **[relato]** = caso real
> independente (issue/fórum); **[vendor]** = material comercial, usar com cautela.

## 1. Veredito geral

**A visão dos docs está validada.** O funil Atrair → Qualificar → Orçar → Fechar com IA no WhatsApp,
orçamento automático, pagamento no chat e humano só em exceção é exatamente o padrão que os melhores
sistemas de 2025/2026 implementam. Três reforços importantes:

1. **Não existe case público de vendedor IA de portas/materiais de construção no Brasil** — busca extensa
   não achou nenhum. O mais próximo é o OrçaFácil da Leroy Merlin (mai/2025, orçamento por IA no WhatsApp,
   sem números publicados). O nicho está aberto.
2. **O melhor case de produção auditável encontrado** (8 meses rodando, cliente nomeado, Dubai) usa
   n8n + WhatsApp Cloud API + Chatwoot e reduziu a mediana de primeira resposta de 15,4 min para 13 s.
   A regra de preço deles, citada literalmente: *"If an item is not in the catalog, no figure may be
   quoted, not even a range"* — nasceu de um incidente real em que o modelo inventou uma taxa. **[relato]**
3. A própria Meta lançou um "Meta Business Agent" (IA nativa dela, ~US$0,04–0,05/mensagem) — validação
   de que o mercado vai nessa direção; rejeitado para nós porque é caixa-preta e não garante preço
   determinístico do nosso catálogo.

## 2. As 7 decisões que a pesquisa fundamenta

### D1. Preço NUNCA sai do LLM — sempre de um motor determinístico
O padrão dos sistemas sérios: o LLM **extrai** os dados (medida, acabamento, quantidade), o **código
calcula** o preço em cima do catálogo/tabela, e o LLM só **narra** o resultado. Respaldo: Anthropic
"Building Effective Agents" (gates determinísticos entre etapas) **[primária]**; incidente real de preço
inventado no case de Dubai **[relato]**; caso Air Canada — tribunal condenou a empresa a honrar informação
errada do chatbot **[jornalismo sobre decisão real]**. No Brasil, o CDC tende a tornar oferta precisa
**vinculante**: preço errado dito pela IA pode virar obrigação. Mitigação: todo orçamento com ID único +
validade explícita + tabela de preço versionada (nunca sobrescrever, criar registro novo e desativar o
antigo — padrão Stripe Quotes) e checagem programática de que qualquer valor em R$ na resposta bate com o
retorno da tool.

### D2. Evolution API primeiro é aceitável, MAS com os olhos abertos (sua decisão, com ressalvas)
A pesquisa recomendaria API oficial direto. Evidência dura contra o não-oficial **[relatos, issues GitHub
2025–2026]**: bans de contas com 3+ anos; casos de "banido ao escanear o QR"; e o **Error 463 "Reachout
Timelock"** (jul/2026) — o WhatsApp bloqueia **iniciar** conversa com quem não tem ~28 dias de
relacionamento, via cliente não-oficial. Nuance que joga a seu favor: **nosso funil é inbound** (o lead
manda a primeira mensagem vindo do anúncio/landing) — o Error 463 atinge cold outreach, não resposta a
lead que chegou. Condições para o Evolution ser tolerável no início: chip dedicado (nunca o número
principal), só responder quem escreveu primeiro, **zero disparo frio em massa** pelo número do robô
(reativação de base antiga fica para a API oficial), backup contínuo (CRM e conversas são nossos, um ban
não apaga nada) e gatilhos de migração definidos no plano. Também é o motivo de existir a **camada de
adaptador de canal** na arquitetura: trocar Evolution → Cloud API sem tocar no resto.

### D3. Desenhar a conversa para POUCAS mensagens
**A partir de 01/10/2026 a Meta cobra por mensagem também as de "service"** (texto livre), hoje grátis na
janela de 24h **[primária, developers.facebook.com, tarifas exatas até 01/09/2026]**. Tarifa utility BR
hoje ≈ US$0,007/msg (~R$0,04) — duas fontes independentes convergem; o "US$0,68" na tabela da Meta é erro
de casa decimal. Consequência de projeto: agente objetivo, sem "tagarelice", coleta de dados agrupada.
Isso vale desde já (no Evolution é grátis, mas o desenho precisa nascer pronto para o custo da migração).
Custo LLM na mesma ordem de grandeza: conversa completa ≈ US$0,05–0,18 (Haiku/Sonnet, antes de prompt
caching).

### D4. Atribuição: capturar a origem NA PRIMEIRA MENSAGEM ou perder para sempre
- Anúncio Meta clique-para-WhatsApp (CTWA): o webhook traz objeto `referral` com `ctwa_clid` **só na
  mensagem-gatilho** — persistir imediatamente, keyed por telefone **[primária]**.
- `wa.me` não carrega UTM (só `?text=`): landing → WhatsApp exige **código curto injetado no texto** da
  primeira mensagem, que o backend parseia para recuperar gclid/UTMs.
- **Otimizar campanhas pelo evento "Lead qualificado" com valor proxy** (qtd portas × ticket médio), não
  por "Venda": Meta precisa de ~50 eventos/semana por conjunto e Google recomenda ~30 conversões/30 dias —
  com ticket alto e poucas vendas/mês, só o evento de qualificação tem volume **[primária, ambos]**.
- Meta CAPI para mensageria: `action_source: "business_messaging"` + `user_data.ctwa_clid` **[primária]**.
- ⚠️ Google Ads: conversões offline migram para a **Data Manager API em 15/06/2026** — implementar já na
  API nova **[primária]**.
- Ferramentas BR (Tintim, UTMify) existem mas nenhuma cobre funil de IA custom ponta a ponta; o "glue
  code" (persistir origem → disparar eventos) é nosso de qualquer jeito. Preços delas: não confirmados
  (páginas SPA) — checar se um dia interessar.

### D5. Pagamento: Asaas, link Pix no chat, sinal + saldo em duas cobranças
- **Asaas**: Pix R$0,99–1,99/transação sem mensalidade, API madura (`POST /payments` + Checkout) e campo
  `externalReference` que volta no webhook — o elo pagamento↔lead **[primária]**. Alternativa: Mercado
  Pago (tarifário oficial não confirmado na checagem). Stripe descartado (Pix só por convite no BR).
  InfinitePay tem Pix 0% mas sem doc de webhook confirmada — risco de furo na atribuição.
- "Pagamento nativo no WhatsApp" (Orders API BR) é só camada de UI sobre o SEU gateway — a própria Meta:
  *"WhatsApp does NOT support payment reconciliations"* **[primária]**. Link como mensagem normal é o
  padrão de mercado e resolve.
- Ticket alto: **sinal via Pix** (irreversível por padrão, sem chargeback de arrependimento) + segunda
  cobrança do saldo via API (nenhum gateway tem produto "sinal+saldo" nativo).

### D6. Build (seu caso) — com as lições de quem comprou
Para empresa sem time técnico, a conta favorece SaaS (Chatvolt R$549, GPT Maker R$397, Blip Go R$299/mês) —
a manutenção de um build (5–15 h/mês) supera a assinatura. **No seu caso a conta inverte**: você é
técnico, opera a própria infra, quer Evolution (nenhum SaaS oferece) e quer SaaS próprio no futuro.
O critério que separa as plataformas boas das ruins — **tool-calling em tempo de conversa para buscar
preço real** — vira requisito central do nosso código. E `tenant_id` em tudo desde o dia 1 custa quase
nada agora e habilita o SaaS depois.

### D7. Regras operacionais que os relatos de produção tornam obrigatórias
1. **Dedup** de webhook por constraint de banco (caso real: 837 cópias da mesma mensagem em 25 h por
   falta disso) **[relato, issue Evolution ago/2026]**.
2. **Debounce** (~8–12 s) antes de processar rajada de mensagens do lead.
3. **Handoff sem auto-silenciamento**: marcar mensagens do próprio bot para não confundir com intervenção
   humana; humano assume → bot pausa naquela conversa.
4. **Caminho de escalação humana sempre visível** — exigência contratual da Meta, além de boa prática
   **[primária, Business Policy]**.
5. **LGPD + opt-in Meta**: aviso de uso de dados no primeiro contato; opt-in separado por categoria de
   mensagem (exigência da Meta, mais restritiva que a LGPD); PII hasheada (SHA-256) antes de enviar a
   Meta/Google; envio de transcrições a LLM nos EUA = transferência internacional (art. 33) — validar
   com jurídico antes de escalar.

## 3. O que ajusta na visão original dos docs

| Item dos docs | Ajuste pós-pesquisa |
|---|---|
| Público: cliente final primeiro | **B2B primeiro (lojas e revendas)** — sua decisão; orçamento por volume, tabela por perfil, recompra |
| "IA recomenda e orça" | IA **extrai e narra**; quem orça é o motor determinístico (D1) |
| Follow-up automático livre | Follow-up com cadência desenhada para custo pós-out/2026 e, na API oficial, via templates aprovados (D3) |
| Métrica lead→margem | Confirmada; acrescenta: **evento de otimização das campanhas = lead qualificado com valor proxy** (D4) |
| "Link de pagamento" genérico | Asaas, Pix para sinal, `externalReference` amarrando pagamento→lead→anúncio (D5) |
| CRM | CRM próprio leve (Postgres) + operação via Telegram; Kanban visual só quando doer (YAGNI) |

## 4. Gaps explícitos (verificar antes de gastar dinheiro)

- Tarifa **Marketing** BR da API oficial (tabela interativa da Meta — checar ao vivo ao planejar remarketing).
- Tarifas exatas pós-01/10/2026 (Meta publica até 01/09/2026).
- Preços Tintim/UTMify (se formos usar; provavelmente não no MVP).
- Tarifário oficial Mercado Pago (se Asaas não servir).
- Jurídico: CDC/oferta vinculante e LGPD art. 33 (transferência internacional) — consulta dedicada antes de escalar.
- Elegibilidade da Orders API do WhatsApp para empresa nova (não precisamos dela; só se um dia quisermos UI nativa).

## 5. Fontes principais

**Meta/WhatsApp [primárias]:** developers.facebook.com/documentation/business-messaging/whatsapp/pricing ·
…/pricing/non-template-messages · whatsapp.com/legal/business-policy · …/business-terms ·
developers.facebook.com/docs/marketing-api/conversions-api/business-messaging ·
…/documentation/business-messaging/whatsapp/webhooks/reference/messages/text (objeto `referral`/`ctwa_clid`) ·
…/documentation/business-messaging/whatsapp/payments/payments-br/overview

**Google [primárias]:** support.google.com/google-ads/answer/2998031 (migração Data Manager API) ·
…/answer/6268632 (30 conversões/30 dias) · facebook.com/business/help/269269737396981 (50 eventos/semana)

**Padrões de agente [primárias]:** anthropic.com/engineering/building-effective-agents ·
docs.anthropic.com tool use · python.useinstructor.com (validação de slots) · docs.stripe.com/api/quotes
(ID/validade/versionamento) · rasa.com/docs/rasa/forms (slot filling obrigatório)

**Risco não-oficial [relatos]:** github.com/WhiskeySockets/Baileys/issues (#1869, #2707 Error 463) ·
github.com/evolution-foundation/evolution-api/issues (#2497, #1870, #2675) ·
github.com/yredsmih/whatsapp-blacklisted-providers (C&D real contra Chat API)

**Case de produção [relato]:** community.n8n.io/t/-/304159 + xaphor.github.io/case-studies (15,4 min → 13 s)

**Pagamentos [primárias]:** asaas.com/precos-e-taxas · docs.asaas.com/reference/criar-nova-cobranca ·
stripe.com/br/payment-method/pix (Pix por convite) · cbc.ca (caso Air Canada)

**Plataformas BR [vendor, comparadas]:** Chatvolt · GPT Maker · Zaia · Blip Go · Kommo · z-api.io/planos ·
relatório completo com ~60 URLs no artifact da pesquisa de plataformas.
