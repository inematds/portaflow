# PortaFlow — Lista do que preciso de você

> Checklist de dados e acessos para construir a máquina de vendas B2B (lojas e revendas).
> Marcado o que é **[MVP]** (bloqueia o início) e o que é **[Depois]** (pode vir na fase 2+).
> Pode responder por cima deste arquivo, mandar planilha, print ou export — qualquer formato serve.

## A. Catálogo e produto **[MVP]**

- [ ] **Acesso ao seu sistema de catálogo** com o exemplo real que você citou — em qualquer formato: export (planilha/CSV/JSON), acesso ao sistema, ou prints. É a peça mais importante.
- [ ] Estrutura das linhas/modelos de porta (nomes comerciais).
- [ ] Medidas padrão disponíveis (ex.: 60/70/80/90 × 210) e limites de sob medida (se houver).
- [ ] Acabamentos (BP, UV, laqueado/PRIME etc.) e núcleos (colmeia, semi-sólido, sólido).
- [ ] Composição do kit porta pronta: batente/marco (espessuras de parede atendidas — 9/11/14 cm?), guarnição, dobradiças, fechadura — o que é incluso e o que é opcional.
- [ ] Fotos por modelo/acabamento (as melhores que tiver; usadas no orçamento e na venda).
- [ ] Códigos/SKUs, se existirem.

## B. Preço e regras comerciais B2B **[MVP]**

- [ ] Tabela de preço por produto/medida/acabamento (a tabela cheia, de lojista).
- [ ] **Degraus de volume**: preço para 10 / 50 / 100+ portas (ou como você escalona).
- [ ] Diferença de tabela entre perfis: loja/revenda × construtor × cliente final (se atender).
- [ ] **Alçada da IA**: qual desconto máximo o robô pode conceder sozinho antes de chamar você (ex.: até 5% ok, acima disso escala pro humano).
- [ ] Pedido mínimo (valor ou quantidade).
- [ ] Condições de pagamento B2B: à vista Pix (desconto?), sinal + saldo, faturado 28/35 dias (para quem?), cartão/parcelado.
- [ ] Como funciona imposto na venda para revenda (IPI/ST/nota) — o suficiente para o orçamento sair certo.

## C. Logística e operação **[MVP]**

- [ ] Região atendida (cidades/estados/raio de entrega).
- [ ] Frete: quem paga, tabela por região ou por pedido, transportadora própria?
- [ ] Prazo de produção + prazo de entrega (padrão e sob medida).
- [ ] Garantia e política de troca/avaria.
- [ ] Quem é o **humano de escalação** (você?) — nome e como o Telegram deve te chamar.

## D. Canais e identidade **[MVP]**

- [ ] **Número de WhatsApp dedicado** para o robô (recomendo chip novo/dedicado para o Evolution, não o seu pessoal — risco de bloqueio é do número).
- [ ] Nome da empresa/marca a usar nas conversas, CNPJ, cidade sede.
- [ ] **Telegram**: usar o bot do openpcbot ou criar um bot novo? Qual grupo/chat recebe as notificações internas (lead quente, orçamento enviado, fechamento, pedido de humano)?
- [ ] Prova social: fotos de fábrica/estoque/entregas, depoimentos de lojistas, certificações — o que existir.

## E. Acessos técnicos **[MVP]**

- [ ] **Servidor/VPS** onde hospedar (já tem algum? qual? senão, eu especifico um no plano).
- [ ] Conta em gateway de pagamento PJ (Mercado Pago, Asaas ou outro — tem alguma? precisa de conta que gere link Pix/boleto/cartão via API).
- [ ] Chaves de LLM: uso as que já estão em `~/projetos/openpcbotv2/.env` / `~/projetos/wifi/.env`? Confirmar qual provedor prefere para o agente.
- [ ] Domínio para a landing page (tem algum reservado para isso?).

## F. Base existente para reativação **[Depois]**

- [ ] Lista de lojas/revendas que já compraram ou já pediram orçamento (planilha com nome, cidade, WhatsApp).
- [ ] Orçamentos antigos não fechados.
- [ ] Contatos de construtores/arquitetos conhecidos.

## G. Tráfego pago **[Depois — fase de captação]**

- [ ] Conta Google Ads (existe? verba mensal pretendida).
- [ ] Conta Meta Business/Instagram da marca.
- [ ] Verba mensal de mídia pretendida para o piloto.

---

### Observações

1. **O item A (catálogo com exemplo real) é o que destrava tudo** — o orçamento automático é calculado por código em cima dessas tabelas (a IA nunca inventa preço), então a estrutura do catálogo define o modelo de dados do sistema.
2. Sem os itens B (alçada de desconto e condições), a IA opera no modo conservador: só tabela cheia e escala qualquer negociação para você via Telegram.
3. LGPD: vamos guardar dados de contato de leads — a landing e o primeiro contato do robô já vão incluir o aviso de uso de dados; nada extra é necessário de você agora.
