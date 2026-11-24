# Pagar.me (Stone) e Zoop (iFood): fichas para o marketplace Bora Jogar

Pesquisa feita em 2026-11-24. Legenda: **[CONF]** confirmado em doc oficial ou página oficial lida nesta pesquisa; **[INF]** inferência minha; **[NÃO ENC]** não encontrado. Nenhuma página oficial de preços ou documentação trazia data visível, salvo quando indicado.

Notas de método:
- Quase todo o conteúdo da docs.pagar.me e da docs.zoop.co foi lido pelos índices `llms.txt` e pelas páginas `.md` linkadas a partir deles.
- Várias perguntas (hold nativo na Pagar.me, split no estorno da v5, assinatura de webhook, idempotência) não têm resposta nas páginas lidas. Essas lacunas são o principal resultado a levar para o contato comercial e técnico.

## Split (divisão de uma cobrança entre vários recebedores)

### Takeaway
Os dois gateways fazem split nativo por valor fixo ou por porcentagem, no cartão e no Pix.
- **Pagar.me:** tem as três flags que definem quem assume custos (`liable`, `charge_processing_fee`, `charge_remainder_fee`). Não documenta limite de recebedores. O split precisa ser enviado na autorização.
- **Zoop:** limita a 20 splits por transação. Aceita split **a posteriori**, até a véspera da primeira liquidação do recebível.

### Cited Findings
**Pagar.me**
- [CONF] Cada regra de split tem `amount` (inteiro), `type` (`flat` ou `percentage`), `recipient_id` (`rp_` + 16 caracteres) e `options`. Dentro de `options`:
  - `liable`: o recebedor responde por chargeback.
  - `charge_processing_fee`: o recebedor paga a taxa da transação.
  - `charge_remainder_fee`: o recebedor fica com o resto da divisão.
  - Fonte: [Pagar.me – Split](https://docs.pagar.me/reference/split-1)
- [CONF] Ao menos um recebedor precisa ser `liable`, ao menos um precisa pagar a taxa de processamento e ao menos um precisa receber o resto — [Pagar.me – Split](https://docs.pagar.me/reference/split-1)
- [CONF] Aceita N recebedores, com uma ou várias regras por recebedor. A página **não informa limite máximo** — [Pagar.me – Split](https://docs.pagar.me/reference/split-1)
- [CONF] O split (com `recipient_id`) precisa ir na **autorização**, não na captura, porque as bandeiras exigem isso — [Pagar.me – Split](https://docs.pagar.me/reference/split-1)
- [CONF] O recurso é "disponível apenas para clientes PSP" — [Pagar.me – Split](https://docs.pagar.me/reference/split-1)
- [CONF] Split no Pix: o array `split` vai dentro de `payment` e é "suportado para clientes com afiliação Pagar.me" — [Pagar.me – Pix](https://docs.pagar.me/reference/pix-2.md)

**Zoop**
- [CONF] Campos de uma regra de split:
  - `amount`: valor fixo em centavos.
  - `percentage`: a divisão de taxa só é possível nesse tipo.
  - `liable`: 1 ou 0, responde por chargeback.
  - `charge_processing_fee`: 0 calcula o split sobre o valor bruto, 1 sobre o líquido.
  - `is_gross_amount`
  - `charge_recipient_processing_fee`: o recebedor assume a taxa inteira.
  - Não existe `charge_remainder`.
  - Fonte: [Zoop – Split rules](https://docs.zoop.co/docs/split-rules.md)
- [CONF] Limite: "Só serão possível realizar 20 Splits por transações". Não se coloca `on_behalf_of` nas regras, porque o seller principal é identificado automaticamente — [Zoop – Split rules](https://docs.zoop.co/docs/split-rules.md)
- [CONF] As regras valem para transações CP, CNP, boleto e Pix. Há exemplos só de crédito e boleto — [Zoop – Split rules](https://docs.zoop.co/docs/split-rules.md)
- [CONF] O split pode ser criado "a priori" (junto com a transação) ou "a posteriori" (pelo endpoint de split-rules) "até a data anterior a primeira liquidação do recebível". O marketplace escolhe se as taxas administrativas são divididas — [Zoop – Split de Pagamentos](https://docs.zoop.co/docs/split-de-pagamentos.md); endpoint: [POST split-rules por transação](https://docs.zoop.co/reference/post_v1-marketplaces-marketplace-id-transactions-transaction-id-split-rules-1.md)

### Inferences
- [INF] Os 4 recebedores do Bora Jogar (quadra, árbitro, jogador de aluguel, plataforma) cabem com folga nos dois gateways.
- [INF] Para a plataforma ficar com 5% e não pagar a taxa do gateway, o caminho na Pagar.me é `charge_processing_fee=true` no dono da quadra (ou rateio). Hoje nada no lado da Pagar.me impede essa configuração.
- [INF] O split a posteriori da Zoop ajuda quando o árbitro ou o jogador de aluguel são escolhidos depois do pagamento. Mas a janela vai só até a véspera da 1ª liquidação, e no Pix ela é curta.

### Gaps
- [NÃO ENC] Limite máximo de recebedores na Pagar.me.
- [NÃO ENC] Se a Pagar.me permite alterar o split depois da autorização. A doc sugere que não.
- [NÃO ENC] Exemplo oficial da Zoop de split no Pix.

## Custódia / escrow (reter até o fim do jogo e liberar por API ou por data)

### Takeaway
Nenhuma das páginas oficiais lidas descreve um "hold"/escrow nativo, isto é, reter um recebível e liberá-lo por comando de API ou numa data. Nos dois casos o que existe é:
- desligar a transferência automática do recebedor, deixando o dinheiro no saldo dele no gateway;
- depois, disparar a transferência ou o saque por API.

Isso é um escrow **emulado**: o dinheiro já está no saldo do recebedor, não está retido pela plataforma.

### Cited Findings
**Pagar.me**
- [CONF] `transfer_settings.transfer_enabled` (boolean) define se o recebedor recebe automaticamente. Também existem `transfer_interval` (exemplos "Daily"/"Weekly") e `transfer_day`. A página de criar recebedor **não fala em reter nem bloquear fundos** — [Pagar.me – Criar recebedor](https://docs.pagar.me/reference/criar-recebedor-1.md)
- [CONF] Existem endpoints de transferência e saque: criar transferência, cancelar transferência e saque — [Criando uma transferência](https://docs.pagar.me/reference/criando-uma-transferência.md), [Saque](https://docs.pagar.me/reference/saque-1.md), [Operações de saldo](https://docs.pagar.me/reference/operação-de-saldo.md). Listados no [índice llms.txt](https://docs.pagar.me/llms.txt); não li o conteúdo das páginas.
- [CONF] O índice oficial da doc v5 **não tem página** sobre "liberação", "hold" ou "bloqueio" de recebíveis — [docs.pagar.me/llms.txt](https://docs.pagar.me/llms.txt)
- [CONF] Saldo do recebedor: `GET /core/v5/recipients/{id}/balance`. O campo `transferred_amount` foi descontinuado e passa a voltar `null`. Novos recebedores podem ter ID `pp_` (`payment_profile_id`) além de `re_` — [Pagar.me – Liquidação, transferências e recebedores (changelog)](https://docs.pagar.me/docs/liquidação-transferências-e-recebedores.md)
- [CONF / fonte secundária] No Reclame Aqui, a Pagar.me respondeu a um lojista que vendas no cartão são "liberadas automaticamente após 30 dias" mesmo com saldo bloqueado, conforme contrato. Isso descreve bloqueio de risco, não escrow comercial — [Reclame Aqui – Pagar.me](https://www.reclameaqui.com.br/empresa/pagar-me/faq/por-que-meu-saldo-foi-bloqueado_7EErT0zHEasoky68/)

**Zoop**
- [CONF] Política de recebimento:
  - Automática: o saldo disponível é transferido **todo dia** para a conta bancária.
  - Manual (padrão quando a automática não é escolhida): o valor fica no saldo da conta Zoop do seller, que pode transferir para conta de mesma titularidade.
  - A política muda por API ou pelo portal do seller, mas "não será possível realizar alteração da política de recebimento através do dashboard".
  - Fonte: [Zoop – Política de recebimento](https://docs.zoop.co/docs/politica-de-recebimento.md)
- [CONF] A "Conta Gráfica Zoop" é uma conta digital criada automaticamente para cada seller, com extrato e saque por TED para conta de mesma titularidade. A página **não descreve custódia nem retenção com liberação posterior** — [Zoop – Conta Gráfica](https://docs.zoop.co/docs/conta-grafica-zoop.md)
- [CONF] Estorno de transação já liquidada só é possível se houver saldo disponível na "conta gráfica". Sem isso, é preciso uma liberação especial — [Zoop – Cancelando transações cartão](https://docs.zoop.co/docs/estornando-transacoes.md)
- [CONF] Desde 2026, a Zoop divulga split como forma de o marketplace "liquidar e manter só os próprios recursos". O motivo é a **Resolução BCB 522 (nov/2025)**, que exige que marketplaces reestruturem fluxos para liquidar apenas recursos próprios — [Startups.com.br, 03/11/2026](https://startups.com.br/negocios/fintech/zoop-vira-adquirente-para-atender-marketplaces-sob-nova-regra-do-bc/)

### Inferences
- [INF] Escrow emulado nos dois:
  1. Criar o recebedor ou seller com transferência automática desligada.
  2. Fazer o split na hora do pagamento.
  3. Após o jogo, a plataforma dispara transferência ou saque por API.
  - Ressalva: o valor já é **do recebedor** (está no saldo dele). Num cancelamento, o estorno por split precisa debitar esse saldo.
  - Se o recebedor puder sacar por conta própria (por exemplo, pelo portal do seller na Zoop), a retenção não está garantida. Isso precisa ser validado com o comercial.
- [INF] A alternativa de concentrar 100% no recebedor da plataforma e repassar depois conflita com o espírito da Resolução BCB 522 (marketplace liquidar só recursos próprios). Também pode configurar subcredenciamento ou custódia irregular. Vale consultar um jurídico.
- [INF] O split a posteriori da Zoop ("até a véspera da 1ª liquidação") permite adiar a decisão de quem recebe. Não segura o dinheiro além da data de liquidação.
- [INF] Pré-autorização no cartão (autorizar e capturar depois) seria outro mecanismo de "hold", mas é limitada a poucos dias pelas bandeiras. Não confirmei o prazo na Pagar.me nem na Zoop.

### Gaps
- [NÃO ENC] Recurso oficial "split com liberação manual" ou "hold de recebível" na Pagar.me v5. Pesquisei e não está no índice da doc. Perguntar ao comercial se existe algo para contas PSP ou Marketplace.
- [NÃO ENC] Prazo máximo de retenção nos dois gateways.
- [NÃO ENC] Na Zoop, se o marketplace consegue impedir o seller de sacar e controlar a liberação (transferência P2P marketplace↔seller ou bloqueio de saldo).
- [NÃO ENC] Prazo de captura de `auth_only` (pré-autorização) na Pagar.me v5.

## Estornos parciais com valor diferente por recebedor

### Takeaway
- **Zoop:** confirmado. O estorno aceita `void_rules`, com valor por recebedor.
- **Pagar.me v5:** o endpoint de cancelamento aceita `amount` parcial, mas a página lida **não mostra regras de split no cancelamento**. A v2 tinha "estorno parcial com split". Na v5, isso precisa ser confirmado.

### Cited Findings
**Pagar.me**
- [CONF] `DELETE /core/v5/charges/{charge_id}` com body `amount` opcional (sem ele, estorna o total) e `reason` (códigos 4000–4352; 4352 marca suspeita de fraude). No cartão, a cobrança vira `canceled` com `canceled_amount`. No boleto, fica `processing` / `pending_refund`. A página **não fala de split nem de Pix** — [Pagar.me – Cancelar cobrança](https://docs.pagar.me/reference/cancelar-cobran%C3%A7a)
- [CONF] Estorno de Pix: pede-se o cancelamento da cobrança pelo ID, com status `pending_refund` → `refunded` — [Pagar.me – Pix](https://docs.pagar.me/reference/pix-2.md)
- [CONF, versão antiga] A API **v2** tinha "estorno parcial com split", com `split_rules` obrigatório — [Pagar.me v2 – Estorno parcial com split](https://docs.pagar.me/v2/reference/estorno-parcial-com-split) (desatualizado, não é v5)

**Zoop**
- [CONF] Cancelamento parcial só em transações de crédito **capturadas**, e pode ser repetido até zerar. O `amount` diminui e o `original_amount` é mantido. A transação fica `succeeded` enquanto `amount` for maior que 0 e vira `canceled` (webhook `transaction.canceled`) quando zera — [Zoop – Cancelando transações cartão](https://docs.zoop.co/docs/estornando-transacoes.md)
- [CONF] Estorno com split: há uma entrada em `void_rules` por recebedor (`recipient` + `amount`), e o `amount` do seller principal tem que ser igual à soma (ex.: 300 + 500 = 800). Pelo dashboard, o estorno sai do seller principal — [Zoop – Cancelando transações cartão](https://docs.zoop.co/docs/estornando-transacoes.md); [API void](https://docs.zoop.co/reference/post_v1-marketplaces-marketplace-id-transactions-transaction-id-void-1.md)
- [CONF] O cancelamento parcial só é permitido "no dia seguinte" à transação, e o total vale no mesmo dia. Transação liquidada só pode ser estornada se houver saldo na conta gráfica — [Zoop – Cancelando transações cartão](https://docs.zoop.co/docs/estornando-transacoes.md)
- [CONF] Existe página própria para estorno de Pix ([Estorno da transação Pix](https://docs.zoop.co/docs/estorno-transacao-pix.md)), mas não li o conteúdo. Uma reportagem de 2020 já citava estorno parcial em split na Zoop ([Mobile Time, 2020](https://www.mobiletime.com.br/noticias/31/03/2020/zoop-oferece-estorno-parcial-em-split-de-pagamento/)), antiga.

### Inferences
- [INF] O caso "quadra fica com 10% ou 30%, taxa da plataforma não volta" encaixa bem no `void_rules` da Zoop: estorna-se X do dono da quadra, 100% do árbitro e do jogador, e 0 da plataforma.
- [INF] Na Pagar.me v5, se não houver split no cancelamento, o estorno parcial sairia pelas regras padrão (provavelmente proporcional ou do `charge_remainder`). Isso inviabiliza a regra de negócio sem ajuste manual por transferência entre recebedores. **Validar antes de escolher.**

### Gaps
- [NÃO ENC] Na doc v5 lida, o campo de split no `DELETE /charges` da Pagar.me.
- [NÃO ENC] Estorno parcial de Pix (MED/devolução) por recebedor nos dois gateways.
- [NÃO ENC] Prazo máximo para estorno de Pix.

## Pix (QR dinâmico + copia e cola)

### Takeaway
Os dois geram QR dinâmico e copia e cola por API. A Pagar.me documenta expiração de até 10 anos e cobra 0,99% no Pix pelo preço público. A Zoop não publica preço.

### Cited Findings
- [CONF] Pagar.me:
  - `payment_method: "pix"` com `pix.expires_in` (segundos, obrigatório) ou `expires_at` (UTC).
  - "Prazo máximo para expiração de um QRCode Pix é de 10 anos".
  - Em `last_transaction` vêm `qr_code` (o payload, ou seja, o copia e cola) e `qr_code_url` (PNG).
  - Há `additional_information`.
  - O customer precisa ter `name`, `email`, `document` e `phones`.
  - Pix "só para contas configuradas com o gateway Pagar.me".
  - Fonte: [Pagar.me – Pix](https://docs.pagar.me/reference/pix-2.md)
- [CONF] Pagar.me tem um simulador de Pix para testes — [Simulador Pix](https://docs.pagar.me/docs/simulador-pix.md) (listado no índice)
- [CONF] Zoop: a resposta traz o campo EMV para gerar o QR. O consumidor paga por QR ou "Copia e cola". O produto é voltado a clientes Zoop Payments com negócio online — [Zoop – Pix online](https://docs.zoop.co/docs/cobrando-pix-online.md)
- [CONF] Zoop: o seller precisa ser preparado para Pix, e há regras de payout de Pix e estorno de Pix em páginas próprias — [Preparando o seller para Pix](https://docs.zoop.co/docs/preparando-o-seller-para-oferta-do-pix.md), [Regras de payout Pix online](https://docs.zoop.co/docs/geração-de-recebível-payout-pix.md). Listadas, não lidas.

### Inferences
- [INF] O payload `qr_code` da Pagar.me e o EMV da Zoop são o copia e cola padrão BR Code.

### Gaps
- [NÃO ENC] Expiração do Pix na Zoop.
- [NÃO ENC] Taxa de Pix da Zoop.
- [NÃO ENC] Prazo de liquidação do Pix (D+0 ou D+1) documentado nos dois.

## Cartão de crédito (até 6x com juros para o comprador, antecipação, liquidação)

### Takeaway
A Pagar.me publica só a taxa de crédito à vista (4,19%) e "recebimento no dia seguinte*". A Pagar.me tem API completa de antecipação. Não encontrei nas páginas lidas como repassar juros de parcelamento ao comprador em nenhum dos dois. Na prática, o valor da cobrança é calculado pelo próprio app.

### Cited Findings
- [CONF] Pagar.me: crédito à vista 4,19%, Pix 0,99%, "Recebimento: Dia seguinte*" (asterisco sem explicação). Parcelado, boleto, antecipação, transferência e mensalidade não são publicados. A página não tem data — [pagar.me/precos](https://pagar.me/precos), acesso em 2026-11-24
- [CONF] Pagar.me tem endpoints de antecipação (criar, simular spot, limites, cancelar pendente) e `automatic_anticipation_settings` no recebedor (`enabled`, `type` full ou 1025, `volume_percentage`, `days`, `delay`) — [Criar recebedor](https://docs.pagar.me/reference/criar-recebedor-1.md); [Guia rápido sobre antecipações](https://docs.pagar.me/page/guia-rápido-sobre-antecipações.md) (índice)
- [Fonte secundária, sem data] Um blog da Shopify cita "Plano Básico: 3,99% + R$0,40" no crédito da Pagar.me. Conflita com os 4,19% da página oficial e provavelmente está desatualizado — [Shopify BR](https://www.shopify.com/br/blog/taxas-de-processamento-de-cartoes-de-credito)

### Inferences
- [INF] Repasse de juros: o padrão de mercado é o lojista calcular o valor total com juros e enviar `amount` maior com `installments=N`. Precisa de confirmação na doc de cartão.
- [INF] Com `delay` / antecipação, a Pagar.me aceita antecipar recebíveis de parcelado por recebedor. Isso conflita com a ideia de reter fundos.

### Gaps
- [NÃO ENC] Taxas de parcelado 2–6x nos dois.
- [NÃO ENC] Na Zoop, MDR, prazo de liquidação de crédito (D+30 por parcela?) e taxas de antecipação.
- [NÃO ENC] Parâmetro de "juros pagos pelo comprador" nos dois.

## Recebedores / sellers e KYC (CPF e CNPJ)

### Takeaway
- **Pagar.me v5:** desde 2024 exige `register_information` completo, com renda, profissão, endereço e telefone. O sócio do CNPJ precisa constar no QSA. A conta bancária precisa ter o mesmo documento do recebedor. Há "link de KYC" (prova de vida via webapp/QR) para KYC `partially_denied`.
- **Zoop:** exige selfie e documento com foto desde 13/03/2026. Promete análise em até 3 minutos em 99% dos casos. Também exige conta bancária de mesma titularidade.
- Nenhum dos dois documenta chave Pix como destino de repasse.

### Cited Findings
**Pagar.me**
- [CONF] Novo contrato desde 29/02/2024. Contrato antigo só até 30/11/2024. Exige `register_information`:
  - `type: individual` (CPF): `name`, `email`, `document`, `site_url`, `birthdate`, `monthly_income`, `professional_occupation`, `phone_numbers`, endereço completo.
  - `type: corporation` (CNPJ): `company_name`, `trading_name`, `annual_revenue`, `main_address` etc. e exatamente 1 `managing_partners`, que deve ser sócio qualificado no QSA (administradores e procuradores não são aceitos).
  - Fonte: [Pagar.me – Novas regras recebedores v5](https://docs.pagar.me/page/novas-regras-para-criação-de-sellers-de-marketplace-c-v5.md)
- [CONF] `default_bank_account.holder_document` "deve ser igual ao documento do recebedor". Tipos `checking` e `savings`. **Não há campo para chave Pix** — [Pagar.me – Criar recebedor](https://docs.pagar.me/reference/criar-recebedor-1.md)
- [CONF] `POST /core/v5/recipients/{id}/kyc_link` devolve `url`, `base64_qrcode` e `expires_at` para acessar o webapp. O 404 de exemplo diz "doesn't have any kyc with partially_denied status" — [Pagar.me – Criar link KYC](https://docs.pagar.me/reference/criar-link-recebedor.md)
- [CONF] Existem páginas sobre "Adição do fluxo de Prova de Vida (v5)" e "Ambiente de teste para prova de vida" — [índice](https://docs.pagar.me/llms.txt). Não li o conteúdo.

**Zoop**
- [CONF] Sellers PF (CPF) e PJ (CNPJ).
  - PF: nome, e-mail, CPF, nascimento, renda, endereço, MCC, **selfie e documento com foto**.
  - PJ: dados do sócio, razão social, CNPJ, faturamento, data de abertura, MCC, selfie e documento do sócio.
  - Documentos exigidos desde 13/03/2026. Formatos PNG, JPEG, WebP, HEIC etc., até 5 MB.
  - Fonte: [Zoop – Credenciamento](https://docs.zoop.co/docs/credenciamento.md)
- [CONF] Fluxo: cadastro por API → upload de documentos → tokenização da conta bancária → análise (KYC, documentos, risco e antifraude) → habilitado. "Até 3 minutos para 99% dos casos". Até 3 dias úteis para S/A ou LTDA com S/A na cadeia. Resultado por webhook. Status: pending, active, enabled, denied, deleted, disabled — [Zoop – Credenciamento](https://docs.zoop.co/docs/credenciamento.md)
- [CONF] "Dados bancários tokenizados de mesma titularidade" — [Zoop – Credenciamento](https://docs.zoop.co/docs/credenciamento.md)
- [CONF] No sandbox, a aprovação ou rejeição de seller é manual, por e-mail a integracao@zoop.com.br — [Zoop – Credenciamento](https://docs.zoop.co/docs/credenciamento.md)

### Inferences
- [INF] Árbitros e jogadores PF vão ter de informar renda mensal, profissão e endereço completo na Pagar.me, e selfie e documento na Zoop. É fricção alta no onboarding, mas inevitável com qualquer PSP regulado.
- [INF] Pagamento via chave Pix ao recebedor não aparece como opção. O repasse é para conta bancária.

### Gaps
- [NÃO ENC] Custo por recebedor ou seller e tempo de aprovação do KYC na Pagar.me.
- [NÃO ENC] Validação automática de titularidade (penny test ou similar) nos dois.

## Desenvolvedor (API, SDK, sandbox, webhooks, idempotência, tokenização, PCI)

### Takeaway
- **Pagar.me v5:** base `https://api.pagar.me/core/v5`, autenticação Basic com secret key. Tokenização client-side por endpoint público com public key (`appId`), o que serve para WebView Capacitor.
- **Zoop:** a maioria dos endpoints exige **mTLS**. Tem SDKs focados em POS e Tap to Pay. Não encontrei SDK Node.
- Nos dois: assinatura de webhook, política de retry e idempotency key **não foram confirmadas** nas páginas lidas.

### Cited Findings
- [CONF] Pagar.me "Criar token cartão" usa a public key como query `appId` (chamada feita do front) — [Pagar.me – Criar token cartão](https://docs.pagar.me/reference/criar-token-cart%C3%A3o-1); OpenAPI espelhada em [apis.io](https://apis.io/apis/pagarme/pagarme-cards-api/) (secundária)
- [CONF] Objeto webhook da Pagar.me: `id`, `url`, `event`, `status` (pending, sent, failed), `attempts`, `last_attempt`, `response_status`, `data`. Portas 80 e 443. **Sem menção a HMAC, retries ou idempotência** — [Pagar.me – Visão geral webhooks](https://docs.pagar.me/reference/visão-geral-sobre-webhooks.md)
- [CONF] Pagar.me tem checkout hospedado e link de pagamento ([Checkout](https://docs.pagar.me/docs/checkout-about.md)) e network token — [índice](https://docs.pagar.me/llms.txt)
- [CONF] O pacote npm `@pagarme/sdk` aparece em busca ([npm](https://npmjs.com/package/@pagarme/sdk)). A página bloqueou o acesso (403), então não verifiquei publicador nem versão.
- [CONF] Zoop: a referência da API tem "a maioria [dos endpoints] exigindo mTLS". SDKs: Tap to Pay iOS/Android, mPOS, SmartPOS, PINPad. Há sandbox, webhooks (inclusive "Webhook com autorização") e replay de eventos — [docs.zoop.co](https://docs.zoop.co/), [Webhook com autorização](https://docs.zoop.co/docs/webhook-com-autorização.md), [Replay](https://docs.zoop.co/reference/post_v1-marketplaces-marketplace-id-events-actions-replay.md), [Sandbox](https://docs.zoop.co/docs/ambiente-de-sandbox.md)
- [CONF] Zoop: endpoint de token de cartão e mudança recente no endpoint de tokenização (changelog) — [Token de cartão](https://docs.zoop.co/reference/post_v1-marketplaces-marketplace-id-cards-tokens-1.md), [Changelog tokenização](https://docs.zoop.co/changelog/atualização-no-endpoint-de-tokenização.md)
- [CONF] A Zoop tem "MCP Server" na documentação — [docs.zoop.co](https://docs.zoop.co/)

### Inferences
- [INF] Pagar.me: com tokenização por public key no front, o PCI fica provavelmente em SAQ A-EP ou SAQ A. Com checkout hospedado, SAQ A.
- [INF] Zoop: exigir mTLS no backend é viável em Node/Fastify (`https.Agent` com cert), mas aumenta o esforço de integração. Não está claro se a tokenização no front também exige mTLS. Se exigir, o cartão teria de passar pelo backend, o que aumenta o escopo PCI.
- [INF] Os dois parecem não ter SDK oficial Node/TS bem mantido. O plano é cliente HTTP próprio a partir do OpenAPI.

### Gaps
- [NÃO ENC] Assinatura HMAC, política de retry e header `Idempotency-Key` nos dois.
- [NÃO ENC] `tokenizecard.js` da Pagar.me: só achei o endpoint de token.
- [NÃO ENC] Requisitos de mTLS na tokenização da Zoop.

## Preços e modelo comercial

### Takeaway
- **Pagar.me:** publica Pix 0,99% e crédito à vista 4,19% (página sem data, lida em 24/11/2026). Split e marketplace exigem conta **PSP** ou afiliação Pagar.me, o que indica contrato comercial e não autoatendimento puro.
- **Zoop:** não publica preço e é vendida por contato comercial.

### Cited Findings
- [CONF] [pagar.me/precos](https://pagar.me/precos): Pix 0,99%, crédito à vista 4,19%, recebimento "Dia seguinte*". Split listado como produto ("O split mais robusto do mercado") sem preço.
- [CONF] Split "disponível apenas para clientes PSP" — [Pagar.me – Split](https://docs.pagar.me/reference/split-1)
- [CONF] Site da Zoop: sem preços. Os CTAs ("Quero ser cliente", "Comece já") levam ao comercial — [zoop.com.br](https://zoop.com.br/)

### Inferences
- [INF] Marketplace com split nos dois gateways = negociação comercial. As taxas de parcelado e de transferência viriam na proposta.

### Gaps
- [NÃO ENC] Taxas de parcelado, transferência, mensalidade e volume mínimo nos dois.

## Público-alvo, propriedade e status em 2026

### Takeaway
- **Zoop:** é "uma empresa iFood", de infraestrutura e embedded finance para marketplaces, bancos, fintechs e ERPs. Em nov/2026 obteve aval de Visa e Mastercard para virar **adquirente**. Seus clientes são grandes (iFood, Olist, Itaú, Nubank, OLX, Sympla). Não publica tamanho mínimo, mas o perfil é enterprise.
- **Pagar.me:** "agora é Stone" e atende de pequenos a grandes clientes, com preço público.

### Cited Findings
- [CONF] Zoop:
  - Aval de Visa e Mastercard para atuar como adquirente, com foco em marketplaces (integrações Mastercard até jan/2027 e Visa no 1º semestre de 2027, expectativa).
  - Mais de 50 contas, entre elas iFood, Olist, Banco Cora, TOTVS, Itaú, Nubank, Conta Azul, Decolar, Sympla e OLX.
  - TPV de 12 meses de R$155 bi.
  - Verticais: bancos, fintechs, ERPs.
  - Fonte: [Startups.com.br, 03/11/2026](https://startups.com.br/negocios/fintech/zoop-vira-adquirente-para-atender-marketplaces-sob-nova-regra-do-bc/)
- [Secundária] Zoop: faturamento de R$700 mi em 2025 e meta de R$2 bi em 2026 — resumo de busca de [Exame](https://exame.com/negocios/zoop-mira-r-1-bi-apostando-em-celular-como-maquininha/) e [Startups.com.br](https://startups.com.br/negocios/fintech/zoop-vira-adquirente-para-atender-marketplaces-sob-nova-regra-do-bc/). Números não conferidos no texto original.
- [CONF] Rodapé do site: Zoop é "uma empresa iFood" — [zoop.com.br](https://zoop.com.br/)
- [CONF, 2024 – possivelmente desatualizado] Parceria Itaú, Rede e Zoop para split em grandes empresas — [Itaú RI](https://www.itau.com.br/relacoes-com-investidores/noticias/itau-unibanco-rede-e-zoop-se-unem-para-oferecer-solucao-de-split-de-pagamentos-para-grandes-empresas-impulsionarem-seus-negocios/?lang=en), [Mobile Time 14/03/2024](https://www.mobiletime.com.br/noticias/14/03/2024/zoop-fecha-parceria-para-oferecer-split-de-pagamento-com-itau-e-rede/)
- [CONF] Pagar.me: "A partir de agora, Pagar.me é Stone". Selo verificado no Reclame Aqui, com tempo de resposta de cerca de 5 dias (jan–jun/2026) — [Reclame Aqui – Pagar.me](https://www.reclameaqui.com.br/empresa/pagar-me/)

### Inferences
- [INF] Para uma startup em fase inicial como o Bora Jogar, a Zoop provavelmente é grande demais ou exige volume. O modelo de >50 contas grandes sugere que não atende o long tail. Precisa de confirmação comercial.

### Gaps
- [NÃO ENC] Tamanho mínimo ou volume mínimo da Zoop.
- [NÃO ENC] Programa para startups nos dois.

## Limitações conhecidas, reclamações e congelamento de saldo

### Takeaway
No Reclame Aqui há casos de saldo bloqueado na Pagar.me, ligados a análise de risco do lojista e a disputas MED/Pix em lojas fraudulentas. Não achei reclamações específicas sobre a Zoop, e ela é B2B, com pouca exposição ao consumidor.

### Cited Findings
- [CONF] FAQ da Pagar.me no RA: a empresa "tem a responsabilidade de acompanhar o perfil de todos os clientes". Bloqueios acontecem por monitoramento — [RA – Por que meu saldo foi bloqueado](https://www.reclameaqui.com.br/empresa/pagar-me/faq/por-que-meu-saldo-foi-bloqueado_7EErT0zHEasoky68/)
- [CONF] Casos de MED/Pix: a Pagar.me/Stone responde que a loja recebedora é responsável pela disputa — [RA – Contestação de Pix site falso](https://www.reclameaqui.com.br/pagar-me/contestacao-de-pix-por-site-falso-stone_c20AOhbIYEsFrJdX/)
- [CONF] Reclamação de antecipação bloqueada pela Pagar.me — [RA – Foco Radical](https://www.reclameaqui.com.br/foco-radical/a-bloqueou-o-adiantamento-no-site_spKwiS1xd1Cj4RIl/)
- [CONF] Zoop: estorno de transação liquidada depende de saldo na conta gráfica ou de "liberação especial", uma limitação operacional — [Zoop – Cancelando transações](https://docs.zoop.co/docs/estornando-transacoes.md)
- [CONF] Contexto regulatório: a Resolução BCB 522 (nov/2025) dá às bandeiras mais poder sobre critérios de risco e garantias de subcredenciadores e marketplaces. Isso pode levar a mais exigências de colateral ou retenção em 2026–27 — [Startups.com.br](https://startups.com.br/negocios/fintech/zoop-vira-adquirente-para-atender-marketplaces-sob-nova-regra-do-bc/)

### Inferences
- [INF] Um marketplace de serviço futuro (jogo agendado), com fundos retidos e cancelamentos frequentes, tem perfil de risco que tende a gerar retenções. Combinar isso por contrato.

### Gaps
- [NÃO ENC] Relatos de desenvolvedores sobre qualidade e estabilidade da API Zoop ou Pagar.me v5 em 2025–26.
- [NÃO ENC] Reclamações Reclame Aqui da Zoop.
