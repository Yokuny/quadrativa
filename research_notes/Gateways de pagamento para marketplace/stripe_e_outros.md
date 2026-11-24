# Stripe Connect Brasil e outros provedores de split (Efí, Vindi, Adyen for Platforms, EBANX) para o Bora Jogar

Pesquisa feita em 2026-11-24. Legenda: **[C]** confirmado em fonte primária; **[I]** inferido; **[NF]** não encontrado. Escopo exclui Asaas, Iugu, Pagar.me, Zoop, Mercado Pago, PagBank.

## Parte A — Stripe Connect no Brasil (disponibilidade, Pix, parcelamento, retenção, reembolsos, KYC, preços, SDK)

### Takeaway
O Stripe Connect funciona para plataforma BR + contas conectadas BR (cobranças e transferências separadas, Pix incluído), e a retenção até o fim do jogo é possível adiando a transferência. O **parcelamento de cartão brasileiro com juros para o comprador não aparece em nenhuma doc da Stripe** (só México/Japão/Mastercard Installments), e isso provavelmente elimina a Stripe para o requisito de até 6x com juros. Também há atritos com Pix: liquidação T+2, processado via EBANX, sem captura manual.

### Cited Findings
**Disponibilidade do Connect / tipos de cobrança**
- [C] O Brasil (BR) está na lista de regiões que suportam "cobranças e transferências separadas" (separate charges and transfers, SCT). — [Stripe Docs: SCT](https://docs.stripe.com/connect/separate-charges-and-transfers.md?platform=web&integration=custom&ui=elements&api-integration=paymentintents)
- [C] Transferências internacionais entre saldos só valem entre US/CA/UK/EEE/CH. "Em outros cenários, sua plataforma e qualquer conta conectada devem estar na mesma região." Para nós isso significa plataforma BR com contas conectadas BR. — mesma fonte
- [C] "Nas transferências que envolvem o Brasil, o parâmetro `source_transaction` é obrigatório." Toda transferência para um recebedor precisa estar ligada a uma cobrança específica. Várias transferências podem usar a mesma `source_transaction`, desde que a soma não passe do valor da cobrança, e isso cobre o caso dono da quadra + árbitro + jogador. — mesma fonte
- [C] Com `source_transaction`, a transferência é aceita mesmo antes da liquidação, mas os fundos só ficam disponíveis no destino quando a cobrança liquidar. — mesma fonte
- [C] Em SCT, as tarifas da Stripe, os reembolsos e os chargebacks são debitados da conta da plataforma. A Stripe recomenda SCT "somente quando você for responsável pelos saldos negativos das suas contas conectadas". — mesma fonte
- [C] Os repasses internacionais (cross-border payouts) só existem para plataformas US/UK/EEE/CA/CH, e o Brasil não está entre elas. — [Stripe Docs: cross-border payouts](https://docs.stripe.com/connect/cross-border-payouts)
- [C] A Stripe restringe transações internacionais e de câmbio (FX) em contas Stripe Brasil, e "as restrições podem limitar também o uso de recursos do Connect". — [Stripe Support: transaction declined BR](https://support.stripe.com/questions/transaction-declined-stripe-brazil-accounts)
- [C] BR aparece nas listas de países das capacidades de conta (account capabilities v1), então as capacidades card_payments/transfers existem para BR. — [Stripe Docs: account capabilities v1](https://docs.stripe.com/connect/account-capabilities.md?accounts-namespace=v1)
- [C] Existe antecipação de recebíveis no Brasil feita por parceiro ("partner-based Brazil anticipations"), e a plataforma pode solicitá-la em nome das contas conectadas. — [Stripe Support](https://support.stripe.com/questions/partner-based-brazil-anticipations)

**Pix**
- [C] Contas Stripe BR aceitam Pix com liquidação em BRL. O Pix vale para contas comuns e para "Platform or marketplace (Connect)" e está marcado como ✓ no Connect. — [Stripe Docs: Pix](https://docs.stripe.com/payments/pix)
- [C] A liquidação do Pix leva "até 2 dias úteis (T+2)". O valor mínimo é R$ 0,50, e o máximo padrão por pagamento é o equivalente a US$ 3.000, "inclusive para empresas no Brasil". — mesma fonte
- [C] O Pix **não tem captura manual nem parcial**, então não há como segurar uma autorização. Aceita reembolso parcial e total em até 90 dias, normalmente concluído em minutos, e o reembolso não pode ser cancelado. Disputas via MED não podem ser contestadas. — mesma fonte
- [C] Em direct charges ou cobranças com `on_behalf_of`, a conta conectada precisa ter Pix ativo (capacidade `pix_payments` para contas Express ou sem dashboard). — mesma fonte
- [C] O parceiro de Pix da Stripe é a **EBANX**, que aparece como recebedor no extrato do cliente e envia recibos por padrão. Descritor de extrato personalizado não é suportado. — mesma fonte; [EBANX press release](https://business.ebanx.com/en/press-room/press-releases/stripe-users-can-now-accept-pix-in-brazil-via-ebanx)
- [C] Pix Automático (recorrente) é suportado. — [Stripe Docs: Pix](https://docs.stripe.com/payments/pix)
- Conflito: o resumo de busca sobre uma página edge-docs dizia que o Pix no Connect exige conta conectada no mesmo país da plataforma. A página principal enquadra a regra pelo país da plataforma no caso de destination charges. Para BR→BR as duas leituras são compatíveis. — [edge-docs Pix](https://edge-docs.stripe.com/payments/pix)

**Parcelamento (parcelamento de cartão)**
- [C] A página "Installments" da Stripe lista apenas Mastercard Installments (4x sem juros), parcelamento do México (meses sin intereses) e do Japão. **O Brasil não aparece.** — [Stripe Docs: Installments](https://docs.stripe.com/payments/installments?locale=en-US)
- [NF/I] Não encontrei nenhuma doc da Stripe sobre parcelamento BR nem sobre juros repassados ao comprador. Uma busca secundária (guia de integração da Digital Manager Guru) diz que a Stripe Brasil aceita só cartão à vista, boleto e Pix. — [Digital Manager Guru: Stripe](https://docs.digitalmanager.guru/integracoes/stripe)
- [C] A Stripe vende "Meses Sin Intereses" na página /br, mas o produto é mexicano. — [stripe.com/br MSI](https://stripe.com/br/payment-method/meses-sin-intereses)

**Retenção de fundos (escrow)**
- [C] Com SCT, a cobrança fica na plataforma e a transferência pode ser criada depois (por exemplo, ao fim do jogo), sempre com `source_transaction` (obrigatório no BR). — [Stripe Docs: SCT](https://docs.stripe.com/connect/separate-charges-and-transfers.md?platform=web&integration=custom&ui=elements&api-integration=paymentintents)
- [C] "Funds segregation" (retenção protegida antes da transferência) está em **prévia privada**, com acesso solicitado à equipe de vendas. — mesma fonte
- [NF] Não encontrei o limite máximo, no Brasil, para manter os fundos na plataforma antes de transferir (a regra global costuma ser 90 dias, exceto nos EUA, mas não confirmei para BR). Também não confirmei `delay_days` máximo nem repasses manuais no BR.

**Reembolsos parciais com split diferente por recebedor**
- [C] Em SCT, reembolsar a cobrança "não tem impacto em nenhuma transferência associada". A plataforma reverte transferências individualmente (`POST /v1/transfers/{id}/reversals`, total ou parcial via `amount`). Isso permite reembolso parcial com proporções diferentes por recebedor, por exemplo devolver 100% do árbitro e 50% da quadra. — mesma fonte
- [C] A reversão só funciona se o saldo disponível da conta conectada cobrir o valor, ou se "connected reserves" estiver habilitado. Se a transferência ainda não tiver sido criada (fundos retidos), basta não transferir ou transferir menos. — mesma fonte
- [I] `reverse_transfer` + `refund_application_fee` são parâmetros de destination charges (um único destinatário). Para quatro recebedores, o modelo adequado é SCT, com reversões manuais.

**Preços (Brasil)**
- [C] Cartão nacional: 3,99% + R$ 0,39. Cartão internacional: +2%. **Pix: 1,19%.** Boleto: R$ 3,45. Disputa: R$ 55. Reembolso sem tarifa, mas a tarifa original não é devolvida. — [stripe.com/br/pricing](https://stripe.com/br/pricing)
- Conflito: a página de preços do Connect BR cita "a partir de 3,99% + R$ 0,50" por cobrança de cartão. — [stripe.com/en-br/connect/pricing](https://stripe.com/en-br/connect/pricing)
- [C] Connect com "Stripe handles pricing" (conta conectada paga a Stripe direto): sem tarifa de plataforma. Com "You handle pricing" (necessário em SCT com contas Express/Custom): **R$ 6 por conta ativa por mês** (ativa = recebeu repasse no mês) + **0,25% + R$ 0,67 por repasse**. — [stripe.com/en-br/connect/pricing](https://stripe.com/en-br/connect/pricing)

**KYC de contas conectadas BR**
- [C/secundário] A Stripe publicou requisitos de verificação específicos para contas conectadas BR, mas não consegui ler o texto. Fonte secundária diz que a Stripe bloqueia o processamento em contas BR até a verificação ser concluída. — [WP Simple Pay](https://wpsimplepay.com/how-to-complete-stripes-brazil-verification-requirements/); [Stripe Support: Connect onboarding](https://support.stripe.com/questions/connect-platforms-manage-onboarding-and-risk-requirements-for-connected-accounts)
- [NF] Não confirmei a lista exata de campos para pessoa física (CPF, data de nascimento, endereço, documento) nem se contas Express aceitam CPF no BR.
- [I] O repasse ao recebedor vai para conta bancária. Não achei repasse direto para **chave Pix** de conta conectada.

**SDK / mobile**
- [C] Fluxo padrão: SDK Node (`stripe`), Payment Element / Stripe.js (React Stripe.js documentado), SDKs nativos iOS/Android/React Native, webhooks `payment_intent.succeeded` e `charge.dispute.created`. O Pix é suportado no Payment Element e em "In-app payments". — [Stripe Docs: Pix](https://docs.stripe.com/payments/pix); [Stripe Docs: SCT](https://docs.stripe.com/connect/separate-charges-and-transfers.md?platform=web&integration=custom&ui=elements&api-integration=paymentintents)
- [I] Em Capacitor, o Payment Element (web) roda dentro da WebView. Existe também o plugin comunitário `@capacitor-community/stripe`, que não verifiquei nesta pesquisa.

### Inferences
- **Veredito Stripe: fraco/inadequado como gateway único.** Pontos fortes: API, SCT multi-recebedor, retenção via transferência atrasada e reembolso parcial por recebedor via reversões. Pontos fracos: (1) parcelamento BR com juros para o comprador não documentado, um bloqueador; (2) Pix com liquidação T+2 via EBANX e extrato mostrando "EBANX"; (3) cartão BR (3,99% + R$ 0,39) é mais caro que a média local; (4) R$ 6 por recebedor ativo por mês + 0,25% + R$ 0,67 por repasse pesam com muitos árbitros e jogadores de baixo ticket; (5) em SCT a plataforma absorve tarifas, chargebacks e saldos negativos.
- Quem paga os juros do parcelamento: sem parcelamento nativo BR, a pergunta perde o sentido na Stripe.

### Gaps
- Limite máximo de retenção antes da transferência no BR, e se `delay_days` e repasses manuais valem para contas conectadas BR.
- Se o parcelamento BR (`payment_method_options.card.installments`) existe em private beta ou por contato com vendas.
- Lista oficial de KYC para pessoa física BR e se contas Express aceitam CPF.
- Repasse via chave Pix para contas conectadas.

## Parte B — Outros provedores brasileiros com split (shortlist)

### Takeaway
Da lista pedida, **nenhum** cobre de forma documentada os três requisitos juntos (split multi-recebedor + retenção/escrow + parcelamento com juros para o comprador) para uma startup pequena. A **Efí** tem split documentado (Pix até 20 contas, cartão sem limite), mas só entre contas Efí, sem escrow e com split Pix imediato. A **Adyen for Platforms** atende tecnicamente (BR, pessoa física, parcelamento, conta de saldo), mas é voltada a grandes empresas. **Vindi** e **EBANX** não têm documentação pública de split marketplace suficiente.

### Cited Findings

**B1. Efí Bank (ex-Gerencianet)**
- [C] Split Pix: "só pode ser realizado entre contas Efí, com limite máximo de 20 contas para o repasse". Não é possível fazer split para a própria conta. Aceita percentual (soma = 100%) ou valor fixo. Funciona com cobrança imediata (QR dinâmico), QR estático/copia e cola e cobrança com vencimento. — [Efí Dev: Split Pix](https://dev.efipay.com.br/docs/api-pix/split-de-pagamento-pix)
- [C] "No momento, lançamentos só podem ser feitos de forma imediata", ou seja, **sem retenção/escrow** no split Pix. O split não pode ser vinculado a cobrança não ATIVA nem desvinculado depois de CONCLUIDA. — mesma fonte
- [C] "A devolução de Split Pix debita apenas da conta integradora": o reembolso sai integralmente da conta da plataforma, e os recebedores não são debitados nem notificados. Prazo de devolução: 90 dias. — mesma fonte
- [C] Split na API de Cobranças (boleto, cartão, link de pagamento): só entre contas Efí (`payee_code`), "sem limite máximo" de contas. Aceita `percentage` ou `fixed`. `mode=1` desconta a tarifa só da conta principal e `mode=2` desconta proporcionalmente (o split fixo exige `mode=1`). O resíduo vai para o integrador. Os exemplos incluem `credit_card` com `installments`. — [Efí Dev: Split Cobranças](https://dev.efipay.com.br/en/docs/api-cobrancas/split-de-pagamento/); [Efí Dev: Cartão](https://dev.efipay.com.br/en/docs/api-cobrancas/cartao/)
- [NF] A doc de split de cobranças não informa quando o repasse ocorre (no pagamento ou na liquidação), retenção, comportamento de reembolso nem quem paga os juros do parcelamento.
- [I] Cada árbitro, jogador e dono de quadra teria de abrir uma **conta digital Efí** (PF/CPF), o que gera atrito de onboarding. A conta Efí PF existe (app na App Store), mas não verifiquei o KYC. — [App Efí](https://apps.apple.com/app/id1443363326)
- [I] Escrow alternativo: receber 100% na conta da plataforma e pagar os recebedores via Pix (API Pix envio) ao fim do jogo. Isso funciona tecnicamente, mas coloca a plataforma na posição de custodiar recursos de terceiros, um risco regulatório/fiscal a validar.
- [NF] Preços públicos da Efí para Pix/cartão não foram coletados nesta pesquisa.
- **Veredito Efí: candidato razoável para Pix com split imediato e custo baixo.** Para o requisito de escrow é fraco: retenção não suportada no split e reembolso debitado só da plataforma. Exige conta Efí para cada recebedor.

**B2. Adyen for Platforms (Brasil)**
- [C] Para Brasil, a tabela de verificação de plataformas aceita os tipos de entidade **Individual**, Organization e Sole Proprietorship (Trust e Unincorporated Partnership não). — [Adyen Docs: verification requirements](https://docs.adyen.com/marketplaces/verification-requirements)
- [C] Os fundos ficam em *balance accounts* até o repasse. BRL só é pago em conta bancária local, sem repasse cross-border. "Pass-through payout is the only available payout model in Africa, Brazil, and India." — [Adyen Docs: supported currencies](https://docs.adyen.com/account/supported-currencies); [Adyen Docs: getting paid](https://docs.adyen.com/account/getting-paid) (o resumo da busca indicou que parte dessas páginas tem mais de cerca de 5 anos, então pode estar **desatualizada**)
- [C] Parcelamento BR: qualquer número de parcelas mensais de 1 a menos de 100. Antecipação é contratada com a Adyen. — fonte secundária: [BigCommerce: Adyen Installments](https://support.bigcommerce.com/articles/en_US/Knowledge/Adyen-Installments); [Microsoft Learn: Adyen Brazil installments](https://learn.microsoft.com/en-us/dynamics365-release-plan/2022wave1/commerce/dynamics365-commerce/payments-installments-dynamics-365-payment-connector-adyen-brazil) (2022, **possivelmente desatualizado**)
- [C] Split configurável por perfil de "split configuration" (regras automáticas) ou por instrução de split por transação. — [Adyen Docs: automatic split configuration](https://docs.adyen.com/pt/platforms/automatic-split-configuration)
- [C] Existe contrato "Pix payment account" da Adyen (Brasil). — [Adyen legal: schedule pix payment account](https://www.adyen.com/legal/schedule-pix-payment-account)
- [I] "Pass-through payout" como único modelo no Brasil sugere restrições para manter saldo e repassar livremente. É preciso confirmar se dá para segurar fundos na balance account até o fim do jogo.
- [NF] Preços (Adyen usa interchange++ mais tarifa por transação) e volume mínimo para o Brasil não foram encontrados publicamente. [I] Historicamente a Adyen mira volumes empresariais, provavelmente acima do porte de um piloto.
- **Veredito Adyen: tecnicamente o mais completo da lista (split, PF, parcelamento, saldo), mas provavelmente inacessível ou desproporcional para uma startup de piloto.** Exige contato comercial.

**B3. EBANX**
- [C] Opera no Brasil nos modelos cross-border e local, processa em BRL, e cartão aceita parcelamento. — [EBANX Docs: Brazil overview](https://docs.ebanx.com/docs/pay-in/processing/payment-methods/country-specific/brazil/brazil-country-overview)
- [C] Material comercial cita split automático entre partes numa transação e pagamentos únicos ou em massa a vendedores, sem detalhar Pix nem sub-recebedores BR. — [EBANX e-commerce](https://business.ebanx.com/en/industries/e-commerce)
- [C] Reembolso Pix pela rede Pix em até 89 dias (90 no processamento local). EBANX é iniciadora de Pix autorizada pelo BCB (2023). É a parceira de Pix da Stripe. — [EBANX Docs](https://docs.ebanx.com/docs/pay-in/processing/payment-methods/country-specific/brazil/brazil-country-overview); [EBANX press: Stripe Pix](https://business.ebanx.com/en/press-room/press-releases/stripe-users-can-now-accept-pix-in-brazil-via-ebanx)
- [C] Jul/Ago 2026: Pix 4x (BNPL com Pagaleve), em que o lojista recebe à vista e a Pagaleve assume o risco. — [EBANX press release](https://business.ebanx.com/en/press-room/press-releases/ebanx-and-pagaleve-blend-pix-and-bnpl-to-expand-access-to-installment)
- [I] O foco da EBANX é merchant global vendendo para a América Latina, não marketplace doméstico com recebedores PF.
- **Veredito EBANX: inadequada como primeira escolha.** Não encontrei documentação de split marketplace com recebedores PF/CPF nem de escrow.

**B4. Vindi**
- [C] A apresentação institucional de 2026 cita "Split" como divisão customizada e automática de valores entre parceiros. — [Vindi 2026 (PDF)](https://blog.vindi.com.br/wp-content/uploads/2026/04/Parceiros-Apresentacao-Institucional-Vindi-2026.pdf)
- [C/secundário] O split divide o pagamento entre vários recebedores. A Vindi tem integração com mais de 30 plataformas de e-commerce. O foco principal é recorrência e assinaturas. — [NFE.io: Vindi](https://nfe.io/blog/assinatura/vindi-o-que-e/)
- [NF] Não encontrei documentação técnica pública do split da Vindi (endpoints, escrow, Pix, parcelamento, KYC).
- **Veredito Vindi: não recomendada para este caso.** Produto orientado a assinatura e o split marketplace não tem documentação pública.

### Inferences
- Sobre o escrow: entre os provedores pesquisados, nenhum oferece "custódia até o evento + liberação por API" de forma documentada e acessível a startup. Na Stripe isso se resolve com SCT e transferência atrasada; na Efí exigiria a plataforma custodiar. Os provedores excluídos deste escopo (Pagar.me, Asaas, Iugu, Zoop) costumam ser os que oferecem custódia/liberação no split, e vale cruzar com as notas dos outros pesquisadores.
- Parcelamento com juros para o comprador: só Adyen (e EBANX, genericamente) aparecem com parcelamento BR. A Efí mostra `installments` em cartão, mas não documenta juros nem split com parcelamento.

### Gaps
- Preços da Efí (Pix/cartão/split) e regras de juros de parcelamento.
- Volume mínimo, preços e capacidade de retenção da Adyen no Brasil.
- Documentação técnica de split da Vindi e da EBANX.
- KYC de recebedor PF (CPF) em Efí, EBANX e Vindi.
