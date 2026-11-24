# Mercado Pago e PagBank como gateways de marketplace com split, custódia e reembolso (Bora Jogar, Brasil, nov/2026)

Legenda: **[CONFIRMADO]** = lido em doc oficial nesta pesquisa (nov/2026). **[INFERIDO]** = dedução ou conhecimento prévio não reverificado. **[NÃO ENCONTRADO]** = buscado sem sucesso. As páginas de doc do MP e do PagBank não exibem data de atualização. Os exemplos de payload do PagBank usam datas de 2025 (ex.: `2025-09-17T12:14:49.216-03:00`), então são material atual.

Limite da pesquisa: cerca de 18 chamadas de busca/fetch. A página oficial de tarifas do MP (`mercadopago.com.br/custos-receber-pagamentos`) devolveu HTTP 403. A tabela oficial de tarifas do PagBank para API não foi localizada.

---

## 1. Split: múltiplos recebedores num mesmo pagamento

### Takeaway
**MP:** há dois produtos. O **Split 1:1** (OAuth + `marketplace_fee`/`application_fee`) aceita só 1 vendedor + a comissão do marketplace. O **Split Payments 1:N** (Orders API) aceita 1 vendedor principal + **até 10 secundários** por pedido, com valor fixo ou percentual, e cobre o caso quadra + árbitro + jogador + plataforma. O 1:N **não é self-service**: exige liberação pelo contato comercial.
**PagBank:** tem split nativo na API Orders, FIXED/PERCENTAGE, com cartão, Pix e boleto. A doc **se contradiz sobre o número máximo de recebedores**: há uma mensagem de erro que sugere limite de 2. Também exige aprovação prévia do produto.

### Cited Findings: Mercado Pago
- O Split 1:1 está disponível em AR, BR, CL, CO, MX, PE e UY. Divide cada transação entre marketplace e vendedor. [CONFIRMADO] — [MP Split landing](https://www.mercadopago.com.br/developers/pt/docs/split-payments/landing)
- No Split 1:1 cada vendedor tem seu `access_token` via OAuth. A comissão vai em `marketplace_fee` (Checkout Pro, `/checkout/preferences`) ou em `application_fee` (Checkout Transparente, `/payments`). Ordem dos descontos: "primeiro, a comissão do Mercado Pago é descontada" e só então a do marketplace sobre o saldo restante. [CONFIRMADO] — [MP Integrar marketplace 1:1](https://www.mercadopago.com.br/developers/pt/docs/split-payments/split-1-1/integration-configuration/integrate-marketplace)
- A doc do 1:1 menciona apenas um vendedor por pagamento. A página diz: "Não são permitidas transferências de instituições financeiras externas" e "somente permite pagamentos com dinheiro em conta entre contas do Mercado Pago". A frase vem da versão AR/ES e foi replicada na BR. Para quem vai receber, o efeito prático exato não ficou claro. [CONFIRMADO texto; impacto INFERIDO] — [MP 1:1 integrate-marketplace](https://www.mercadopago.com.br/developers/pt/docs/split-payments/split-1-1/integration-configuration/integrate-marketplace)
- **Split Payments 1:N** é um "feature of Checkout Transparente (via Orders API) that allows you to split the value of an order between a primary seller and one or more secondary sellers", "with up to 10 secondary sellers per order". [CONFIRMADO] — [MP Split 1:N overview](https://www.mercadopago.com.br/developers/en/docs/split-payments-1-n/overview); [MP Split 1:N (Orders)](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n)
- Regras do 1:N:
  - `config.split_rules.amount_type` aceita `"fixed"` ou `"percentage"`.
  - No percentual, cada valor fica entre 1 e 100 e a soma tem de ser exatamente 100.00. No fixo, a soma tem de bater com o total.
  - O principal entra com `receiver_type: "owner"` (user_id). Os secundários entram com `receiver_type: "partner"` (UUID `receiver_id`).
  - [CONFIRMADO] — [MP Split 1:N](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n.md)
- Acesso ao 1:N: "You can only integrate this product if your commercial contact has shared all the necessary information to do so." [CONFIRMADO] — idem
- No 1:N, "when the payment is processed, each participant receives their share in their own account". Cada participante é creditado direto na própria conta MP. [CONFIRMADO] — idem
- O relatório de vendas do 1:N traz os campos `Mercadopago_fee`, `Financing_fee`, `Total_Paid_Amount` e `Net_Received_Amount`, e menciona um campo de "marketplace fee". A doc não diz quem arca com a tarifa MP no 1:N. [CONFIRMADO / lacuna] — idem
- "Advanced Payments" (`POST /v1/advanced_payments`, array `disbursements` com `collector_id` por vendedor) aparece hoje nas referências públicas apenas sob **Wallet Connect**, em parte rotulado como "legacy". [CONFIRMADO] — [MP ref advanced_payments (Wallet Connect)](https://www.mercadopago.com.br/developers/en/reference/wallet_connect/_advanced_payments/post); [legacy](https://www.mercadopago.com.br/developers/es/reference/online-payments/wallet-connect/advanced-payments-legacy/create-payment/post)

### Cited Findings: PagBank
- O split cria "subtransações para cada pedido". Exemplo da doc: marketplace + vendedor + transportadora. O modelo é de carrinho único: "ao realizar o checkout os items do carrinho devem ser de apenas um vendedor", com o pagamento dividido entre vários recebedores. [CONFIRMADO] — [PagBank Divisão do pagamento](https://developer.pagbank.com.br/reference/divisao-de-pagamento.md)
- Métodos de divisão:
  - `FIXED`: valores em centavos, com soma igual ao total.
  - `PERCENTAGE`: a soma deve ser exatamente 100.
  - "Os meios de pagamento disponíveis para a divisão de um pagamento são: boleto, cartão de crédito e Pix". Sempre existe uma transação primária.
  - [CONFIRMADO] — [Como utilizar a divisão](https://developer.pagbank.com.br/reference/como-utilizar-a-divisao-de-pagamento.md)
- Campos do split: `splits.method`, `splits.receivers[].account.id` (ID de conta PagSeguro, `ACCO_...`), `splits.receivers[].amount.value` e `splits.receivers[].reason`.
  - Regras: mínimo de 2 recebedores; o vendedor primário deve estar na lista e ser dono do token; nenhum recebedor repetido.
  - Na tabela de erros, `invalid_max_receivers` diz que a transação "must have 2 receivers" e manda enviar transações separadas quando houver mais.
  - **Isso contradiz** o exemplo de 3 partes (marketplace, vendedor, transportadora) da página de visão geral.
  - [CONFIRMADO, com contradição] — [Pedido com divisão de pagamento](https://developer.pagbank.com.br/reference/pedido-com-divisao-de-pagamento)
- Quem não estiver habilitado para split recebe HTTP 403. A doc remete a um formulário de aprovação em `dev.pagseguro.uol.com.br/reference/request-approval`. [CONFIRMADO] — idem
- Taxas: é possível zerar o recebimento líquido de alguém desde que as taxas declaradas igualem o bruto. Existem `configurations.liable` (MCC do vendedor principal) e `configurations.chargeback.charge_transfer.percentage`. [CONFIRMADO campos; semântica parcial] — [Como utilizar](https://developer.pagbank.com.br/reference/como-utilizar-a-divisao-de-pagamento.md); [Liable](https://developer.pagbank.com.br/reference/utilizar-o-mcc-vendedor-principal-liable.md); [Chargeback de secundário](https://developer.pagbank.com.br/reference/recuperacao-chargeback-de-secundario.md)
- Fonte de terceiros (apis.io) descreve o split do PagBank como "single charge across multiple receivers (with optional custody / release control)". [TERCEIRO] — [apis.io](https://apis.io/apis/pagseguro/split/)

### Inferences
- MP 1:N com até 10 secundários cobre o caso, com folga: quadra + árbitro + até ~8 jogadores de aluguel, com a plataforma como "owner". O MP 1:1 não serve, porque só comporta 1 vendedor.
- No PagBank, o limite de recebedores precisa ser validado em sandbox ou com o comercial antes de qualquer decisão. Se o limite for mesmo 2, o PagBank não atende o caso de 3 ou mais recebedores numa única cobrança.
- Nos dois provedores, a plataforma (CNPJ) deve ser o recebedor primário/owner. Se a taxa de 5% for modelada como a parcela da plataforma, quem arca com a tarifa do gateway passa a importar. No PagBank isso é configurável. No MP 1:N, não ficou documentado.

### Gaps
- MP 1:N: quem arca com a tarifa do MP (o principal ou proporcionalmente) e como a "marketplace fee" é declarada. A doc não especifica.
- PagBank: número máximo real de recebedores (a doc é contraditória) e custo do produto split.

---

## 2. Custódia e escrow: reter até o fim do jogo e liberar por API ou data

### Takeaway
**PagBank: SIM, documentado.**
- A custódia é por recebedor (`custody.apply`), com data agendada (`custody.release.scheduled`).
- O padrão é de **90 dias** e o máximo de **365 dias**. Ao fim do prazo a liberação é automática.
- Também dá para liberar antecipadamente via `POST /splits/{split_id}/custody/release` para um ou mais recebedores.
- O exemplo da doc usa cartão. Custódia com Pix não ficou confirmada.

**MP:** a doc do 1:N **não descreve custódia nem retenção**: cada um recebe "when the payment is processed". O mecanismo `money_release_date` via disbursements existe no Advanced Payments (SDK oficial), mas a doc pública atual o coloca no contexto do Wallet Connect/legacy. Disponibilidade para um marketplace novo em 2026 não confirmada.

### Cited Findings: PagBank
- Campos de custódia:
  - `splits.receivers[].configurations.custody.apply: true` bloqueia o valor daquele recebedor. "Caso `custody.apply` esteja como `false` ou não seja informado, os valores atribuídos a esse recebedor não serão bloqueados."
  - `configurations.custody.release.scheduled` recebe uma data ISO 8601.
  - `release.released_at` fica null até a liberação.
  - `custody.status` assume `HELD` ou `RELEASED`.
  - [CONFIRMADO] — [Crie e pague um pedido com custódia](https://developer.pagbank.com.br/reference/crie-e-pague-um-pedido-com-custodia.md)
- Prazos: "90 dias: Quando não for informado o prazo da liberação na criação do pedido". O máximo em `scheduled` é de 365 dias. Depois do prazo a liberação é automática. [CONFIRMADO] — idem
- A liberação antecipada antes da liquidação deixa o valor no saldo futuro, "não mais bloqueado" na liquidação. [CONFIRMADO] — idem
- Liberação via API:
  - Chamada: `POST {base}/splits/{split_id}/custody/release`, body `{"receivers":[{"account":{"id":"ACCO_..."}}]}`, resposta 200 vazia.
  - "Esta liberação pode ser feita para um ou mais recebedores envolvidos."
  - "Após esta etapa, os valores estarão disponíveis para transferências das contas dos vendedores."
  - "Apenas o usuário dono da transação pode realizar a liberação da custódia."
  - Limite de 500 requisições a cada 5 minutos.
  - [CONFIRMADO] — [Liberar divisão de pagamento com custódia](https://developer.pagbank.com.br/reference/liberar-divisao-de-pagamento-com-custodia.md)
- A doc não mostra liberação parcial do valor de um recebedor: a liberação é do valor inteiro de cada recebedor. [CONFIRMADO por ausência] — idem

### Cited Findings: Mercado Pago
- O 1:N credita cada participante na própria conta assim que o pagamento é processado. Retenção, liberação manual ou data de liberação não são mencionadas. [CONFIRMADO por ausência] — [MP Split 1:N](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n.md)
- O SDK oficial (Python) tem `advanced_payment.update_release_date`, que envia `money_release_date` ao endpoint de disbursements de advanced payments. [CONFIRMADO no SDK] — [python-mercadopago advanced_payment](https://sources.debian.org/data/main/p/python-mercadopago/2.3.0-1/docs/resources/advanced_payment.html)
- A referência pública de `/v1/advanced_payments` é a do Wallet Connect, com `capture:false` para reservar e captura posterior. Parte das páginas tem o rótulo "legacy". [CONFIRMADO] — [MP ref advanced_payments](https://www.mercadopago.com.br/developers/en/reference/wallet_connect/_advanced_payments/post)
- No relatório de dinheiro liberado, a "release date" é o momento em que a transação impacta o saldo disponível. O campo `SELLER_AMOUNT` traz o valor recebido em compras com split. [CONFIRMADO] — [MP released-money report fields](https://www.mercadopago.com.br/developers/es/docs/checkout-api-payments/additional-content/reports/released-money/report-fields.md)
- Prazo de recebimento do vendedor no MP (na hora, 14 ou 30 dias) é configuração de conta/tarifa, não escrow controlado pela plataforma. [INFERIDO]

### Inferences
- O PagBank tem o recurso de custódia mais aderente ao modelo "segura até o fim do jogo": `scheduled` = data do jogo + margem, mais release via API no check-out da partida. O ponto a confirmar é se a custódia vale para Pix.
- No MP 1:N, a retenção teria de ser feita de outro jeito, já que o dinheiro cai direto na conta MP de cada recebedor (que pode sacar):
  - (a) cobrar 100% na conta da plataforma e repassar depois por transferência, o que configura custódia própria com risco regulatório (subcredenciamento/BCB); ou
  - (b) pedir ao comercial MP o Advanced Payments com `money_release_date`.
- Pré-autorização de cartão (capture=false) não serve como escrow longo: autorizações expiram em poucos dias e o mecanismo não existe para Pix. [INFERIDO]

### Gaps
- MP: o Advanced Payments com `money_release_date` está disponível para novos integradores BR em 2026? Qual o prazo máximo de retenção? Não encontrado.
- PagBank: custódia com Pix e boleto. A doc só mostra exemplo com `CREDIT_CARD`.

---

## 3. Reembolso parcial por recebedor; Pix vs cartão; depois da liberação

### Takeaway
Os dois suportam reembolso parcial com distribuição customizada por recebedor.
- **MP 1:N:** `POST /v1/orders/{order_id}/refund` com `splits[{receiver_id, amount}]`, em valor monetário e não percentual.
- **PagBank:** cancelamento parcial "customizado", com FIXED/PERCENTAGE por recebedor e `fee_liable` para quem paga as taxas.

Nos dois, **o recebedor precisa ter saldo** para que o estorno seja debitado. Depois que o dinheiro é liberado e sacado, o reembolso pode falhar.

### Cited Findings: Mercado Pago
- Opções do 1:N:
  - Reembolso total do pedido: `POST /v1/orders/{order_id}/refund` com body vazio.
  - Total de um participante: `splits` só com `receiver_id`.
  - Parcial de um participante: `splits` com `receiver_id` + `amount`.
  - Vários participantes numa única requisição.
  - "you cannot mix `transactions` and `splits` in the same refund request."
  - "partial or full refund per participant without affecting others".
  - [CONFIRMADO] — [MP Split 1:N](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n.md)
- No 1:1, o reembolso é dividido proporcionalmente entre vendedor e marketplace. "o Marketplace não poderá realizar o reembolso total se o vendedor não tiver dinheiro na conta". [CONFIRMADO] — [MP 1:1](https://www.mercadopago.com.br/developers/pt/docs/split-payments/split-1-1/integration-configuration/integrate-marketplace)

### Cited Findings: PagBank
- Só o recebedor primário pode cancelar. "Todos os recebedores precisam ter saldo em conta para realizar a devolução dos valores": sem saldo de algum deles, o cancelamento não é autorizado. [CONFIRMADO] — [Cancelamento de pedido com divisão](https://developer.pagbank.com.br/reference/cancelamento-de-pedido-com-divisao-de-pagamento.md)
- **Proporcional** (API, iBanking ou central): por exemplo, um parcial de 60% devolve 60% de cada recebedor. [CONFIRMADO] — idem
- **Customizado** (só API):
  - FIXED ou PERCENTAGE por recebedor (percentuais somando 100). Um recebedor pode ser debitado acima do que recebeu, desde que tenha saldo.
  - Depois de um parcial customizado, os seguintes também precisam ser customizados.
  - As taxas ficam com um único recebedor via `fee_liable.percentage = 100`; sem isso, saem do primário.
  - [CONFIRMADO] — idem
- Também há endpoints genéricos de estorno: criar e consultar estorno, listar por vendedor e por pagamento. [CONFIRMADO existência] — [Criar estorno](https://developer.pagbank.com.br/reference/criar-estorno-de-pagamento.md)

### Inferences
- O caso de W.O. funciona nos dois provedores: quadra retém 10% ou 30%, plataforma não devolve a taxa, árbitro e jogador devolvem 100%. No MP, a plataforma simplesmente não inclui o próprio `receiver_id`/owner no array de refund. No PagBank, usa o FIXED customizado com a plataforma em 0.
- No PagBank, se o valor ainda estiver em custódia (`HELD`), o saldo existe e o estorno tende a funcionar. Depois de liberado e sacado, falha. [INFERIDO; a doc não trata do pós-release]
- Pix devolução (MED/devolução Pix) vs estorno de cartão: nenhum dos dois documenta diferença no contexto de split. Pix costuma ter prazo de devolução de até 90 dias. [INFERIDO]

### Gaps
- Regras específicas de devolução Pix vs cartão em split (prazos e devolução parcial Pix) não foram encontradas em nenhum dos dois.
- MP 1:N: se a tarifa MP é devolvida no reembolso.

---

## 4. Pix (QR + copia e cola), parcelamento até 6x com juros para o comprador, prazos e antecipação

### Takeaway
- Os dois suportam Pix via API em split (MP 1:N: `bank_transfer`, exige chave Pix cadastrada; PagBank: pedido com QR Code Pix e split).
- O PagBank documenta o repasse de juros ao comprador em `charges.amount.fees.buyer.interest`. No MP, o repasse de juros não foi confirmado especificamente para o 1:N.

### Cited Findings
- MP 1:N aceita cartão de crédito, débito e Pix (`bank_transfer`). "Pix requires registered Pix keys". O exemplo tem o campo `installments`, mas as regras de parcelamento no split não estão descritas. Os headers obrigatórios incluem `x-idempotency-key` e `x-meli-session-id`. [CONFIRMADO] — [MP Split 1:N](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n.md)
- PagBank: "Pedido com divisão de pagamento com PIX" e "Criar pedido com QR Code PIX" existem como casos documentados. [CONFIRMADO existência] — [Pedido com divisão com Pix](https://developer.pagbank.com.br/reference/pedido-com-divisao-de-pagamento-com-pix.md); [QR Code Pix](https://developer.pagbank.com.br/reference/criar-pedido-com-qr-code-pix-v2.md)
- PagBank: juros de parcelamento repassados ao comprador via `charges.amount.fees.buyer.interest`. Também há "Criar pedido com repasse de taxa ao comprador" e "Consultar taxas de uma transação". [CONFIRMADO] — [Como utilizar a divisão](https://developer.pagbank.com.br/reference/como-utilizar-a-divisao-de-pagamento.md); [Repasse de taxa](https://developer.pagbank.com.br/reference/criar-transacao-com-repasse-de-taxa.md); [Consultar taxas](https://developer.pagbank.com.br/reference/consultar-taxas-transacao.md)
- A resposta do Pix traz QR em imagem/base64 e o texto copia e cola (MP: `point_of_interaction.transaction_data.qr_code` / `qr_code_base64` na API Payments; PagBank: `qr_codes[].text` + links de imagem). [INFERIDO por conhecimento prévio; não reverificado em nov/2026]
- MP: parcelamento "com juros para o comprador" é o comportamento padrão do Checkout (o vendedor pode oferecer "parcelamento sem juros" assumindo o custo). O relatório 1:N tem o campo `Financing_fee`. [INFERIDO; campo CONFIRMADO no relatório 1:N]

### Inferences
- Antecipação: nos dois, o prazo de recebimento (na hora, 14 ou 30 dias) funciona como antecipação embutida na tarifa. Com custódia/escrow, porém, a antecipação perde sentido até a liberação.

### Gaps
- Limite máximo de parcelas e configuração "máx. 6x" no split. Nos dois, isso é controlado pelo front ou pelo parâmetro de parcelas, mas não foi verificado na doc de split.

---

## 5. Recebedores e KYC (pessoa física com CPF)

### Takeaway
Nos dois, **cada recebedor precisa ter conta própria** no provedor.
- **MP 1:N:** conta MP do secundário + convite de consentimento por e-mail aceito na interface do MP (status ACTIVE), com validação de KYC/compliance.
- **PagBank:** conta PagBank/PagSeguro "vendedor" ativa (`ACCO_...`). A conta pode ser criada via API ("Criar conta") ou autorizada via Connect (OAuth/SMS).

Contas de pessoa física (CPF) são comuns nos dois. Aceitação explícita de CPF como recebedor de split: **não encontrada** nas páginas de split.

### Cited Findings: Mercado Pago
- O vendedor principal cria o convite via `POST /v1/split-consent` com `invites[].seller_email`. O secundário aceita "through the Mercado Pago interface". O `receiver_id` (UUID) volta logo na criação.
  - Status: `PENDING` (expira em 30 dias, configurável via `expiration_days`), `ACTIVE` (sem expiração), `REJECTED`, `CANCELED`, `EXPIRED`.
  - O secundário pode cancelar o consentimento a qualquer momento, só pela interface. Depois disso o `receiver_id` fica inválido.
  - [CONFIRMADO] — [MP Split 1:N](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n.md)
- Erros de validação por recebedor: "user not found", "not active", "site mismatch", "seller is using pnf mode, blocked", KYC "compliance validation error" (422). [CONFIRMADO] — idem
- O webhook "Split - Autorização" notifica as transições de consentimento (`consent.accepted`, `consent.expired`, `canceled_by` PP/PS). [CONFIRMADO] — idem
- Split 1:1: o vendedor conecta a conta via OAuth. [CONFIRMADO] — [MP 1:1](https://www.mercadopago.com.br/developers/pt/docs/split-payments/split-1-1/integration-configuration/integrate-marketplace)

### Cited Findings: PagBank
- O recebedor precisa ter "conta de vendedor" ativa (há erros específicos para conta inexistente ou inativa) e estar habilitado a split (sem isso, 403). [CONFIRMADO] — [Pedido com divisão](https://developer.pagbank.com.br/reference/pedido-com-divisao-de-pagamento)
- Há endpoints de conta (Criar conta, Consultar conta, objeto Account) e de Connect (autorização via Connect Authorization ou SMS, obter, renovar e revogar access token). [CONFIRMADO existência] — [Criar conta](https://developer.pagbank.com.br/reference/criar-conta.md); [Connect](https://developer.pagbank.com.br/docs/connect.md); [Connect via SMS](https://developer.pagbank.com.br/docs/connect-via-sms.md)

### Inferences
- Atrito de onboarding: no MP, o jogador ou árbitro precisa ter (ou criar) conta MP e aceitar o convite no app/site do MP, um passo fora do app Bora Jogar. No PagBank, "Criar conta" via API pode reduzir o atrito (conta criada pelo marketplace). Conferir os campos e o KYC exigidos.
- Saque dos recebedores para chave Pix ou conta bancária acontece a partir da conta MP/PagBank deles, não diretamente do split. [INFERIDO]
- Abrir conta MP/PagBank PF costuma ser gratuito. [INFERIDO]

### Gaps
- Aceitação de CPF como recebedor secundário nos dois (provável, mas não confirmada em doc de split). Verificação de titularidade bancária. Custo de onboarding.

---

## 6. Developer: SDK, sandbox, webhooks, idempotência, tokenização, PCI

### Cited Findings
- MP 1:N:
  - Token de cartão gerado com **MercadoPago.js**: "valid for 7 days" e "can only be used once".
  - Headers `x-idempotency-key` (idempotência) e `x-meli-session-id` (device ID). 3DS suportado.
  - Teste exige duas contas de teste Seller do mesmo país, e-mails `{USERNAME}@testuser.com` e header `x-test-token: true`.
  - [CONFIRMADO] — [MP Split 1:N](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n.md)
- PagBank:
  - Cartão criptografado no cliente com chave pública (endpoints criar, consultar e alterar chave pública).
  - Webhooks com "Confirmar autenticidade da notificação". "Chaves públicas e de idempotência".
  - Ambientes sandbox e produção, cartões de teste, simulador e cenários de teste.
  - [CONFIRMADO existência] — [Chaves públicas](https://developer.pagbank.com.br/docs/chaves-publicas.md); [Idempotência](https://developer.pagbank.com.br/docs/chaves-publicas-e-de-idempotencia.md); [Webhooks](https://developer.pagbank.com.br/reference/webhooks.md); [Autenticidade](https://developer.pagbank.com.br/reference/confirmar-autenticidade-da-notificacao.md); [Ambientes](https://developer.pagbank.com.br/docs/ambientes-disponiveis.md); [Cartões de teste](https://developer.pagbank.com.br/docs/cartoes-de-teste.md)
- A base URL da liberação de custódia do PagBank é `sandbox.api.pagseguro.com/splits/` e difere da usada pelo restante da Connect API. A doc é ambígua. [CONFIRMADO] — [Liberar custódia](https://developer.pagbank.com.br/reference/liberar-divisao-de-pagamento-com-custodia.md)
- MP:
  - SDK Node oficial `mercadopago` (npm, v2, TypeScript).
  - Webhooks assinados com header `x-signature` (`ts` + `v1` HMAC-SHA256 sobre `id:...;request-id:...;ts:...;`, usando a secret do painel).
  - Card Payment Brick / Payment Brick (`@mercadopago/sdk-react` ou JS).
  - [INFERIDO por conhecimento prévio; não reverificado em nov/2026. O SDK Python oficial com advanced_payment está CONFIRMADO em [debian sources](https://sources.debian.org/data/main/p/python-mercadopago/2.3.0-1/docs/resources/advanced_payment.html)]
- PagBank: SDK Node oficial para a API Orders **não encontrado**. A integração seria por REST direto. [NÃO ENCONTRADO]

### Inferences
- Tokenização ou criptografia no cliente (MercadoPago.js/Bricks; lib de criptografia do PagBank) roda em WebView do Capacitor por ser JS puro. O PAN não toca o backend, o que mantém o escopo PCI mínimo (SAQ A-EP/SAQ A). O MP exige device ID (`x-meli-session-id`), e é preciso garantir que o script de device fingerprint carregue na WebView. [INFERIDO]
- Lojas de app: venda de serviço físico (quadra/árbitro) pode usar gateway próprio, fora do IAP da Apple/Google. [INFERIDO]

### Gaps
- Mecanismo exato de assinatura dos webhooks do PagBank (não lido). Cobertura do 1:N no SDK Node do MP.

---

## 7. Preços publicados

### Takeaway
- **MP (página oficial de checkouts):** Pix **0,99%**. Cartão de crédito no Checkout Transparente "entre 3,98% e 4,98% conforme o prazo". No link de pagamento: na hora 4,98%, 14 dias 4,48%, 30 dias 3,99%.
- **PagBank:** tabela oficial de API não localizada. Fontes de terceiros divergem: crédito ~4,99% na hora, ~3,99–4,49% em 14 dias, ~3,19–3,99% em 30 dias; Pix 0,99–1,89%.
- Custo dos produtos split/custódia: não publicado em nenhum dos dois.

### Cited Findings
- MP Pix 0,99%; crédito no Checkout Transparente de 3,98% a 4,98% conforme prazo. [CONFIRMADO via snippet da página oficial; a página principal de tarifas devolveu 403] — [MP Checkouts](https://www.mercadopago.com.br/ferramentas-para-vender/check-out?code=barwy); [MP Link de pagamento](https://www.mercadopago.com.br/ferramentas-para-vender/link-de-pagamento); [MP tabela de taxas (knowledge hub)](https://www.mercadopago.com.br/knowledge-hub/tabela-taxas-tarifas_45243)
- PagBank (terceiros):
  - Nuvemshop: crédito à vista 4,99% na hora, 3,99% em 14 dias, 3,19% em 30 dias; Pix até 1,89%.
  - Shopify: 4,98%, 4,49% e 3,99%; Pix 0,99% e, em outro trecho, 1,89%.
  - calculadoradetaxas (jul/2026): Pix grátis e crédito 2,92%, mas em condições de alto faturamento.
  - [TERCEIROS, inconsistentes] — [Nuvemshop](https://www.nuvemshop.com.br/blog/como-cobrar-online-com-o-pagseguro); [Shopify BR](https://www.shopify.com/br/blog/servicos-de-pagamento-para-empresas); [calculadoradetaxas](https://www.calculadoradetaxas.com.br/pagbank)

### Inferences
- As tarifas são negociáveis por volume, e os produtos 1:N e split/custódia passam pelo comercial. Na prática, o preço sai de proposta comercial.

### Gaps
- Tarifa oficial do PagBank para API (Pix e cartão por prazo), tarifa do parcelado por número de parcelas nos dois, tarifa de saque (MP e PagBank costumam ter Pix de saque gratuito, INFERIDO) e taxa de antecipação.

---

## 8. Limitações, bloqueios e reclamações

### Cited Findings
- Limitações estruturais documentadas:
  - MP 1:N: requer liberação comercial; o secundário pode revogar o consentimento a qualquer momento; o secundário não tem acesso ao relatório de split; limite de 10 secundários. [CONFIRMADO] — [MP Split 1:N](https://www.mercadopago.com.br/developers/en/docs/checkout-api-orders/resources/split-payments-1-n.md)
  - PagBank: carrinho de vendedor único; cancelamento só pelo primário; todos os recebedores precisam de saldo para estornar; possível limite de 2 recebedores. [CONFIRMADO] — [Cancelamento](https://developer.pagbank.com.br/reference/cancelamento-de-pedido-com-divisao-de-pagamento.md); [Divisão](https://developer.pagbank.com.br/reference/divisao-de-pagamento.md)
- Reclamações (2026, relatos unilaterais):
  - MP retendo mais de R$ 32 mil "em garantia" (jul/2026, sem resposta). — [Reclame Aqui](https://www.reclameaqui.com.br/bizcapital/mercado-pago-retem-indevidamente-mais-de-r-32-mil-em-valores-de-garantia-mesmo-apos-recusa-do-cliente_5tLAWSkByokgNzmk/)
  - Bloqueio de conta ML/MP sem justificativa. — [Reclame Aqui](https://www.reclameaqui.com.br/mercadolive/bloqueio-indevido-de-conta-e-impossibilidade-de-utilizacao-dos-servicos-do-mercado-livremercado-pago_zcDUWFkcCwRRKqqI/)
  - PagBank condenado (TJMA, 1ª instância) a indenizar R$ 1.000 por bloqueio de conta por 90 dias (mar/2026). — [Terra Brasil Notícias](https://terrabrasilnoticias.com/2026/03/pagbank-e-condenado-por-manter-bloqueio-de-conta-por-prazo-excessivo/)
  - Escritório de advocacia afirma que o entendimento do tribunal (out/2025) sobre bloqueio de saldo alcança instituições de pagamento. Material de advocacia. — [Rosenbaum](https://www.rosenbaum.adv.br/banco-digital-bloqueou-conta-com-saldo-direitos-cliente/)

### Inferences
- Como os recebedores (PF) têm conta própria no gateway, bloqueios por compliance (MP: "pnf mode, blocked", erro de KYC) ou retenções impactam árbitros e jogadores individualmente, e o suporte fica fora do controle da plataforma.
- Em resumo para o Bora Jogar:
  - **PagBank** tem custódia/escrow com liberação por API documentada, mas há dúvida crítica sobre o limite de recebedores.
  - **MP 1:N** comporta até 10 recebedores e reembolso por participante, mas não tem custódia documentada.
  - Nenhum dos dois atende tudo de forma confirmada só pela documentação. Os dois exigem validação com o comercial.

### Gaps
- Nota Reclame Aqui atualizada (índice de solução) de MP e PagBank não coletada. Relatos específicos de marketplaces com split nesses provedores não encontrados.
