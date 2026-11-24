# Asaas e Iugu como gateways para o marketplace Bora Jogar (split, custódia, estornos, Pix, cartão, subcontas, DX, preços)

Pesquisa feita em 2026-11-24. Legenda: **[CONFIRMADO]** = está na documentação oficial ou em página oficial (a data `updatedAt` da página vai entre parênteses quando existe). **[INFERIDO]** = conclusão do pesquisador. **[NÃO ENCONTRADO]** = sem fonte. Páginas atualizadas antes de 2025 aparecem marcadas como possivelmente desatualizadas. Todas as páginas de docs do Asaas consultadas têm `updatedAt` entre ago e nov/2026. Boa parte das páginas da Iugu traz `updatedAt: 2025-10-31`, que parece uma re-sincronização em massa e não uma revisão de conteúdo.

---

## Q1. Split de pagamentos (vários recebedores, fixo ou %, limite, quem paga a taxa, Pix e cartão)

### Takeaway
As duas plataformas fazem split nativo por cobrança entre contas internas (subcontas ou outras contas da mesma plataforma), com valor fixo e percentual. O Asaas não limita o número de recebedores e calcula o % sobre o valor líquido, já descontada a taxa do gateway, que fica com o emissor da cobrança. A Iugu também cobra a taxa da conta que cria a fatura e permite configurar o split por meio de pagamento e por número de parcelas. Nenhuma das duas faz split para quem não tem conta na plataforma.

### Cited Findings
**Asaas**
- [CONFIRMADO] O split aceita `fixedValue` (até 2 casas decimais) e `percentualValue` (até 4 casas) e permite combinar os dois na mesma cobrança. Split só com % não pode passar de 100%. O total não pode passar do valor líquido. — [Split de pagamentos](https://docs.asaas.com/docs/split-de-pagamentos)
- [CONFIRMADO] Não há limite de recebedores: "Não há limite de quantidade de `walletId` informado para os recebedores." — [FAQ do Split](https://docs.asaas.com/docs/faq-do-split)
- [CONFIRMADO] Cada recebedor precisa ter conta Asaas e `walletId`. O `walletId` do próprio emissor não entra no split, e o líquido que sobra fica com o emissor. — [Split de pagamentos](https://docs.asaas.com/docs/split-de-pagamentos); [FAQ do Split](https://docs.asaas.com/docs/faq-do-split)
- [CONFIRMADO] O % é aplicado sobre o `netValue`, "o valor da cobrança após o desconto das taxas aplicáveis" (exemplo da doc: líquido de R$ 98 com 50% dá R$ 49). Taxas de antecipação também reduzem esse líquido. Na prática, quem absorve a taxa do gateway é o emissor da cobrança, o que bate com a regra "o líquido não direcionado permanece com quem criou". — [FAQ do Split](https://docs.asaas.com/docs/faq-do-split); [Split para contas com Conta Escrow](https://docs.asaas.com/docs/split-para-contas-com-conta-escrow)
- [CONFIRMADO] Split não pode ser agendado para data futura: ele roda quando a cobrança é recebida. Se o split for maior que o líquido disponível, a cobrança fica bloqueada (`PAYMENT_SPLIT_DIVERGENCE_BLOCK`) e há 2 dias úteis para corrigir. Passado esse prazo, os splits são cancelados. — [FAQ do Split](https://docs.asaas.com/docs/faq-do-split)
- [CONFIRMADO] Contas novas vêm com split desabilitado. A habilitação é feita em Integrações > Segurança, com confirmação por SMS ou Token App. Sem ela, a API devolve `invalid_action`. Webhooks do split: `PAYMENT_SPLIT_DONE`, `PAYMENT_SPLIT_DIVERGENCE_BLOCK`, `PAYMENT_SPLIT_DIVERGENCE_BLOCK_FINISHED`. Status possíveis: PENDING, AWAITING_CREDIT, DONE, CANCELLED, REFUSED, REFUNDED. — [Split de pagamentos](https://docs.asaas.com/docs/split-de-pagamentos)
- [CONFIRMADO] Existem guias específicos de split para cobranças avulsas, parcelamentos (`totalFixedValue`, endpoint "Atualizar splits do parcelamento"), assinaturas e cobranças antecipadas. — [índice docs Asaas](https://docs.asaas.com/docs/português/llms.txt)
- [INFERIDO] O split vale para qualquer `billingType` (Pix, cartão, boleto), porque o objeto `split` fica na cobrança genérica. As páginas de split não falam de restrição por meio de pagamento.

**Iugu**
- [CONFIRMADO] O split exige o "Plano Marketplace" (Conta Mestre + Subcontas) e transfere os valores automaticamente quando a fatura é paga. "Hoje, a divisão de valores é realizada apenas entre contas iugu (mestre e sub)." — [Split de Pagamentos](https://dev.iugu.com/docs/split-de-pagamentos)
- [CONFIRMADO] Os parâmetros por recebedor são `recipient_account_id`, `cents`, `percent`, `permit_aggregated` (fixo + % ao mesmo tempo, só via API), `pix_cents/percent`, `credit_card_cents/percent`, `bank_slip_cents/percent` e `credit_card_{1..12}x_cents/percent` (split diferente por número de parcelas, até 12x, ou 18x em contas antigas). — [Split de Pagamentos](https://dev.iugu.com/docs/split-de-pagamentos)
- [CONFIRMADO] Há split padrão por conta ("Configurar Conta") e split variável por fatura ou assinatura (array `splits` em `POST /v1/invoices`). — [Split de Pagamentos](https://dev.iugu.com/docs/split-de-pagamentos); [Split de pagamento (conceito)](https://dev.iugu.com/docs/split-de-pagamento)
- [CONFIRMADO] Ressalva: "A soma dos Splits nunca deve totalizar 100% do valor da fatura; caso contrário, a divisão... não será processada, e... o valor total será destinado à conta criadora." — [Split de Pagamentos](https://dev.iugu.com/docs/split-de-pagamentos)
- [CONFIRMADO] "O split é liquidado sempre na conta que está criando a transação, ou seja, a conta que cria a transação paga as taxas iugu." Exemplo da doc: fatura de R$ 100 na mestre com 70% para a subconta e taxa de R$ 2,50 deixa R$ 27,50 para a mestre. — [Split de pagamento (conceito)](https://dev.iugu.com/docs/split-de-pagamento)
- [CONFIRMADO] O site comercial cita "MultiSplit" ("aplica múltiplas regras na mesma transação") e "saldo reserva". — [iugu.com](https://www.iugu.com/)
- [CONFIRMADO] A doc tem um "Criar Splits" ("multi splits para uma conta; criar um novo multi split sobrepõe o que já está configurado"). — [Criar Splits](https://dev.iugu.com/reference/criar-split-1)

### Inferences
- Para o Bora Jogar (dono da quadra + árbitro + jogador de aluguel + 5% da plataforma), o desenho natural nas duas é a plataforma emitir a cobrança na conta-pai, colocar os 3 recebedores no split e ficar com o resto (5% menos a taxa do gateway). Como a taxa sai da parte do emissor, os 5% brutos da plataforma diminuem. Uma alternativa é a cobrança ser emitida pela subconta do dono da quadra, que então absorve a taxa.
- Cada árbitro e cada jogador precisa ter subconta (Asaas ou Iugu), com KYC. Ver Q6.

### Gaps
- [NÃO ENCONTRADO] Número máximo de recebedores por fatura na Iugu. O índice cita uma página sobre "Limitações do Split de Pagamento, como o número máximo de contas", mas o trecho que li não trazia o número.
- [NÃO ENCONTRADO] Se há tarifa extra por split nas duas.

---

## Q2. Custódia/escrow (reter até o fim do jogo e liberar por API ou por data)

### Takeaway
**O Asaas tem custódia nativa: a "Conta Escrow".** Ela é configurada por subconta, retém os recebimentos por `daysToExpire` e libera automaticamente em `expirationDate` ou manualmente via `POST /v3/escrow/{id}/finish`, com mensalidade por subconta habilitada. **Não encontrei escrow nativo na Iugu.** Lá a custódia teria de ser emulada: receber na conta-mestre e transferir depois via API, ou segurar o saque das subcontas (`auto_withdraw` desligado), o que não impede o saldo de ficar disponível para a subconta.

### Cited Findings
**Asaas: Conta Escrow**
- [CONFIRMADO] "A Conta Escrow permite reter temporariamente valores recebidos por uma subconta até o encerramento da garantia. Enquanto a garantia estiver ativa, o recebimento existe, mas o valor não fica disponível no saldo da subconta." Casos de uso citados: "marketplaces que liberam o valor após a confirmação da entrega; plataformas que aguardam a conclusão de um serviço". (updatedAt 2026-09-02) — [Introdução - Conta Escrow](https://docs.asaas.com/docs/introducao-conta-escrow)
- [CONFIRMADO] "pode ser configurada diretamente pela API para as subcontas, sem necessidade de solicitar uma liberação prévia ao Asaas". A configuração é por subconta ou por um padrão que vale para todas. Só recebimentos posteriores à habilitação entram no fluxo. Ao desabilitar, todas as garantias da subconta são encerradas e os valores liberados. — [Introdução - Conta Escrow](https://docs.asaas.com/docs/introducao-conta-escrow)
- [CONFIRMADO] Configuração: `POST /v3/accounts/{id}/escrow` com `enabled`, `isFeePayer`, `daysToExpire`. O prazo pertence à subconta e não ao array `split`. Mudanças em `daysToExpire` só afetam recebimentos novos. — [Split para contas com Conta Escrow](https://docs.asaas.com/docs/split-para-contas-com-conta-escrow); [ref. Conta Escrow](https://docs.asaas.com/reference/conta-escrow)
- [CONFIRMADO] Consulta: `GET /v3/payments/{id}/escrow`, com a API key da própria subconta, devolve status, `expirationDate` e o id da garantia. Também existe o objeto `escrow` nas listagens de cobranças. — [Valores sob garantia](https://docs.asaas.com/docs/bloqueio-dos-valores)
- [CONFIRMADO] Liberação manual: `POST /v3/escrow/{id}/finish`, onde `{id}` é o id da garantia e não o da cobrança. "A liberação manual encerra imediatamente a garantia, independentemente da data configurada em `expirationDate`." Liberação automática "Ao atingir `expirationDate`. Nenhuma chamada à API é necessária." Uma garantia encerrada não pode ser reativada. — [Liberação dos valores em garantia](https://docs.asaas.com/docs/desbloqueio-dos-valores)
- [CONFIRMADO] Com split: "não trate um valor como disponível apenas porque o respectivo Split foi liquidado". Cada subconta recebedora segue o próprio `daysToExpire` (os exemplos da doc usam 10 e 15 dias). — [Split para contas com Conta Escrow](https://docs.asaas.com/docs/split-para-contas-com-conta-escrow)
- [CONFIRMADO] "Não há atualmente um evento de Webhook específico documentado para alterações no estado da garantia." É preciso consultar por polling. — [Valores sob garantia](https://docs.asaas.com/docs/bloqueio-dos-valores)
- [CONFIRMADO] Custo: há mensalidade por subconta com Escrow habilitado, cobrada a partir do dia seguinte à habilitação. `isFeePayer: true` faz a subconta pagar; `false` ou omitido, paga a conta principal. O valor "consulte Configurações > Taxas" não é público. — [Cobrança da taxa por subconta](https://docs.asaas.com/docs/cobran%C3%A7a-da-taxa-por-subconta-configurada)

**Iugu**
- [NÃO ENCONTRADO] Nos índices `llms.txt` de dev.iugu.com, não há recurso de escrow/custódia/retenção com liberação por API. Busquei "custód", "reten", "bloque", "liber", "escrow" e "garantia". — [índice dev.iugu.com](https://dev.iugu.com/llms.txt)
- [CONFIRMADO] Ferramentas que servem para emular: "Transferir Valor entre contas iugu" (só em live_mode, com Assinatura RSA obrigatória) — [Transferir Valor](https://dev.iugu.com/reference/transferir-valor). Configuração de subconta com `auto_withdraw`, `auto_withdraw_type`, `auto_advance` etc. — [Configurar subconta por API](https://dev.iugu.com/docs/configurar-subconta-por-api). "Cobrança em Duas Etapas (Captura Tardia)", com `two_step_transaction`, só para cartão — [Cartão — Configurações extras](https://dev.iugu.com/docs/cc-configuracoes-extras).
- [CONFIRMADO] O site comercial cita "controle saldo reserva", sem documentação técnica que eu tenha encontrado. — [iugu.com](https://www.iugu.com/)

### Inferences
- [INFERIDO] **Período máximo do escrow Asaas:** a doc não traz limite para `daysToExpire`. Como dá para liberar manualmente, um `daysToExpire` com folga (por exemplo, 7 a 15 dias depois da data do jogo) seguido de `finish` quando o jogo acaba atende o requisito "reter até o fim do jogo e liberar por API ou automaticamente".
- [INFERIDO] **Limitação importante do escrow Asaas:** o prazo é por subconta, não por cobrança. Não dá para dar a cada reserva a data do seu jogo como `expirationDate`. Daí vem o padrão "prazo fixo longo + finish manual".
- [INFERIDO] **A plataforma (5%) não está em escrow:** a parte que fica na conta-pai é liberada no fluxo normal. Num W.O. ou cancelamento, os 5% já estariam disponíveis, o que combina com a regra "taxa da plataforma não é reembolsada".
- [INFERIDO] **Emular escrow** (cobrar tudo na conta da plataforma e transferir depois), seja na Iugu, seja no Asaas sem Escrow, tem implicações (não verifiquei em fontes jurídicas nesta pesquisa; ver `regulatorio.md`):
  - a plataforma passa a deter recursos de terceiros, o que pode caracterizá-la como prestadora de serviço de pagamento ou subcredenciadora sob as regras do BCB;
  - toda a receita bruta (quadra + árbitro + jogador) entra na conta da plataforma, com risco de tributação (ISS, PIS/COFINS, IRPJ) sobre o total se não houver segregação contábil e documental clara de "valores de terceiros";
  - a responsabilidade de pagar o recebedor e o risco de chargeback ficam inteiramente com a plataforma;
  - as transferências entre contas podem gerar tarifa e exigem conciliação própria.
  - O escrow nativo, com split direto para subcontas, evita que o dinheiro dos recebedores passe pela conta da plataforma.

### Gaps
- [NÃO ENCONTRADO] Valor da mensalidade da Conta Escrow por subconta (só aparece logado).
- [NÃO ENCONTRADO] Limite máximo de `daysToExpire`.
- [NÃO ENCONTRADO] O que acontece com estorno ou chargeback de cobrança cuja garantia ainda está ativa. A doc de liberação e a de valores retidos não tratam disso.
- [NÃO ENCONTRADO] Se a Iugu tem produto de custódia sob contrato comercial ("saldo reserva").

---

## Q3. Estornos parciais e por recebedor (Pix devolução e cartão estorno, depois do split liquidado)

### Takeaway
**O Asaas atende o requisito central.** O endpoint de estorno aceita `value` (total) e `splitRefunds[{id, value}]`, que define quanto sai de cada split, e o resto é debitado da cobrança principal. Funciona para Pix (vários estornos parciais) e cartão. **A Iugu não atende bem.** Pix só aceita estorno integral. No cartão, o estorno parcial é rateado proporcionalmente entre as contas, sem escolha por recebedor.

### Cited Findings
**Asaas**
- [CONFIRMADO] "Estorna total ou parcialmente uma cobrança paga por cartão de crédito ou Pix." "Cobranças recebidas via Pix permitem estorno integral ou múltiplos estornos parciais, desde que a soma... não ultrapasse o valor total." Cartão pode ser estornado quando CONFIRMED ou RECEIVED e leva até 10 dias úteis para aparecer na fatura do pagador. (updatedAt 2026-09-08) — [Estornar cobrança](https://docs.asaas.com/reference/estornar-cobranca)
- [CONFIRMADO] Regras do `value` + `splitRefunds`: "Se apenas `splitRefunds` for informado, o valor total será a soma dos valores definidos nos Splits. Se `value` e `splitRefunds` forem informados, `value` define o total. Os valores definidos em `splitRefunds` são debitados dos respectivos participantes e o restante é debitado da cobrança principal. Se nenhum dos campos for informado, o estorno será integral." Schema `PaymentRefundSplitRequestDTO {id: id do split, value: valor a estornar do split}`. — [Estornar cobrança](https://docs.asaas.com/reference/estornar-cobranca)
- [CONFIRMADO] "As taxas da cobrança, como compensação e notificação, não são devolvidas no estorno." Se faltar saldo (por exemplo, estorno integral de Pix logo após receber), a API retorna 400. — [Estornar cobrança](https://docs.asaas.com/reference/estornar-cobranca)
- [CONFIRMADO] A resposta traz um array `refunds` com `status` (PENDING/CANCELLED/DONE), `endToEndIdentifier`, `transactionReceiptUrl` e `refundedSplits[{id, value, done}]`. Uma cobrança pode ter vários estornos. — [Estornos](https://docs.asaas.com/docs/estornos)
- [CONFIRMADO] Existem endpoints separados: estornar parcelamento, estornar boleto, listar estornos. — [índice ref. Asaas](https://docs.asaas.com/reference/llms.txt)
- [CONFIRMADO] Num estorno de cobrança com split, "os repasses relacionados à cobrança também são estornados". — [FAQ do Split](https://docs.asaas.com/docs/faq-do-split)

**Iugu**
- [CONFIRMADO] Tabela oficial: Cartão, parcial ✅, até 180 dias, restituição em 30 a 60 dias. Pix, parcial ❌, até 90 dias, restituição instantânea. Boleto, sem estorno. — [Reembolsar Faturas por API](https://dev.iugu.com/docs/realizar-o-reembolso-de-faturas-estorno-por-api) (updatedAt 2026-09-01)
- [CONFIRMADO] Pix: "o valor do estorno precisa ser integral, ou seja, não são permitidos reembolsos parciais." (updatedAt 2026-09-03) — [Reembolso Pix](https://dev.iugu.com/reference/reembolso-pix)
- [CONFIRMADO] Cartão com split: "o valor reembolsado será distribuído proporcionalmente entre as contas envolvidas" (exemplo da doc: 20/80, estorno de R$ 10 tira R$ 2 da mestre e R$ 8 da sub). — [Reembolso Cartão](https://dev.iugu.com/reference/reembolso-cartao-de-credito)
- [CONFIRMADO] O endpoint `POST /v1/invoices/{id}/refund` tem `partial_value_refund_cents` ("somente cartão de crédito"). Não há parâmetro por split no request. — [Reembolsar Fatura](https://dev.iugu.com/reference/reembolsar-fatura)
- [CONFIRMADO] "O estorno gera cobrança da taxa de MDR", debitada da conta responsável pelo estorno. Em cartão parcelado, o estorno sai do saldo disponível (parcelas já liquidadas) e do saldo a receber (parcelas futuras). Sem saldo disponível, o reembolso não é feito. — [Reembolsar Faturas por API](https://dev.iugu.com/docs/realizar-o-reembolso-de-faturas-estorno-por-api)

### Inferences
- [INFERIDO] Num cenário de W.O. no Asaas (quadra fica com 30%, árbitro recebe integral, plataforma não devolve os 5%), dá para expressar tudo em um único estorno: `splitRefunds` com 70% do split da quadra, 0 do árbitro e o valor do jogador conforme a regra. Os 5% ficam porque o "restante" só sai da cobrança principal se `value` for maior que a soma dos splitRefunds.
- [INFERIDO] Com Escrow ativo, a ordem ideal é estornar antes de liberar a garantia. Não está documentado se o estorno debita valores ainda retidos.
- [INFERIDO] Na Iugu, a regra de W.O. exigiria estorno integral do Pix e depois nova transferência ou cobrança para recompor as partes. Ou, alternativa melhor, não estornar e pagar os recebedores com transferências a partir da conta-mestre, o que leva de volta à custódia emulada.

### Gaps
- [NÃO ENCONTRADO] Prazo máximo para estorno de Pix no Asaas. A regra do BCB (MED/devolução) costuma ser 90 dias, mas não confirmei na doc Asaas.
- [NÃO ENCONTRADO] Comportamento do `splitRefunds` quando o recebedor já sacou o saldo (Asaas).

---

## Q4. Pix (QR dinâmico + copia e cola, expiração, tarifas)

### Takeaway
As duas geram QR dinâmico e copia e cola via API. No Asaas, o QR dinâmico vence 12 meses depois do vencimento da cobrança. A tarifa padrão de Pix recebido no Asaas é fixa por transação e a página oficial mostrou valores conflitantes entre consultas. Na Iugu, a tarifa é percentual (≈0,99% segundo uma fonte terceira de out/2026).

### Cited Findings
- [CONFIRMADO] Asaas: `POST /v3/lean/payments` com `billingType: PIX`, depois `GET /v3/payments/{id}/pixQrCode`, que devolve `encodedImage` (base64), `payload` (copia e cola) e `expirationDate`. "O QR Code é dinâmico, possui vencimento e pode ser pago uma única vez. Ele expira 12 meses após a data de vencimento da cobrança." Sem chave Pix cadastrada, o QR fica vinculado a uma instituição parceira e vale até 23:59 do mesmo dia. Essa modalidade será descontinuada. (updatedAt 2026-10-13) — [Cobranças via Pix](https://docs.asaas.com/docs/cobrancas-via-pix)
- [CONFIRMADO] Asaas: existe o Asaas Checkout (página hospedada) para Pix, com expiração e URLs de redirecionamento. — [índice docs](https://docs.asaas.com/docs/português/llms.txt)
- [CONFIRMADO] Iugu: Pix é ativado via `v1/payments/pix` (exige conta verificada). A cobrança é feita com `POST /v1/invoices` e `payable_with: ["pix"]`. A fatura traz QR Code e copia e cola. Ao alterar o vencimento, "o QR Code PIX já..." continua o mesmo id. (página de 2025-10-31) — [Cobrar com PIX por API](https://dev.iugu.com/docs/realizar-cobrança-com-pix-por-api); [índice ref.](https://dev.iugu.com/reference/llms.txt)
- [CONFIRMADO] Tarifa Pix Asaas, no texto da página de preços capturado em 2026-11-24: "Pix R$ 2,15 / R$ 1,08 por transação recebida". R$ 1,08 parece ser a taxa promocional de 3 meses para novos clientes ("promoção válida até 24/02/2027" para quem se cadastra hoje). Uma WebFetch anterior no mesmo dia retornou "R$ 0,79 / R$ 1,59", e um comparador de out/2026 cita "R$ 1,99 por Pix ou boleto". **Há conflito entre leituras, possivelmente teste A/B ou mudança recente.** Confirmar na conta. — [Asaas Preços e taxas](https://www.asaas.com/precos-e-taxas); [quasa.io (terceiro)](https://quasa.io/pt/media/asaas-ou-iugu-tarifa-publicada-vence-ate-o-volume-pedir-proposta)
- [CONFIRMADO, fonte terceira] Iugu plano Essencial (out/2026): Pix 0,99%, boleto R$ 2,19, cartão 3,34%, mais R$ 0,40 de processamento por boleto ou cartão. Mensalidade "consulte". — [quasa.io](https://quasa.io/pt/media/asaas-ou-iugu-tarifa-publicada-vence-ate-o-volume-pedir-proposta)

### Inferences
- [INFERIDO] Para tickets de quadra (R$ 100 a R$ 300), uma tarifa fixa de Pix (Asaas, R$ 1 a 2) tende a sair mais barata que 0,99% (Iugu) acima de cerca de R$ 200.

### Gaps
- [NÃO ENCONTRADO] Prazo de liquidação do Pix (D+0) nas docs oficiais das duas. O Pix costuma cair na hora, mas não achei a frase oficial.
- [NÃO ENCONTRADO] Página oficial de preços da Iugu: `/precos` e `/taxas` dão 404 e `/planos` redireciona para a home sem valores.

---

## Q5. Cartão: até 6x, juros repassados ao comprador, antecipação, prazos (D+x)

### Takeaway
O Asaas parcela até 12x (21x para Visa/Master) e cobra MDR por faixa de parcelas. **Não achei um parâmetro nativo de "repassar juros ao comprador"**: o campo `interest` da API é juro por atraso, então o valor com juros teria de ser calculado pela plataforma. A Iugu tem `max_installments_without_interest` na configuração da conta, o que indica repasse nativo de juros a partir de N parcelas. Recebimento de cartão no Asaas: 1 parcela a cada ~32 dias, com antecipação de 1,25% a.m. (à vista) ou 1,70% a.m. (parcelado).

### Cited Findings
**Asaas**
- [CONFIRMADO] "até 21 parcelas para Visa e Mastercard" e "até 12 parcelas para as demais bandeiras". É preciso informar `remoteIp` (IP do pagador). — [Cobranças via cartão](https://docs.asaas.com/docs/cobrancas-via-cartao-de-credito); [Preços](https://www.asaas.com/precos-e-taxas)
- [CONFIRMADO] O parcelamento é criado com `installmentCount` + `installmentValue` ou `totalValue`. Se não dividir certo, a diferença vai para a última parcela. (updatedAt 2026-11-16) — [Criar cobrança parcelada](https://docs.asaas.com/docs/criar-uma-cobranca-parcelada)
- [CONFIRMADO] O `PaymentInterestRequestDTO` da API é "Percentual de juros ao mês ... para pagamento após o vencimento", ou seja, juro por atraso e não juro de parcelamento. — [Criar parcelamento (ref)](https://docs.asaas.com/reference/criar-parcelamento)
- [CONFIRMADO] O blog do Asaas lembra que juros no parcelamento são legais se informados antes e incluídos no CET. — [Blog Asaas - juros em compras parceladas](https://blog.asaas.com/cobranca-de-juros-em-compras-parceladas/)
- [CONFIRMADO] MDR padrão (página de 2026-11-24, padrão / promocional de 3 meses): à vista R$ 0,49 + 2,99% / 1,99%; 2 a 6x R$ 0,49 + 3,49% / 2,49%; 7 a 12x R$ 0,49 + 3,99% / 2,99%; 13 a 21x R$ 0,49 + 4,29% / 3,29%, "sobre o valor total da venda". Débito R$ 0,35 + 1,89%. — [Preços](https://www.asaas.com/precos-e-taxas)
- [CONFIRMADO] Prazo: o simulador da página mostra "1 parcela a cada 32 dias". O FAQ diz que à vista o dinheiro "estaria disponível... (D+32)". Antecipação de cartão: "1,25% ao mês" (à vista) e "1,70% ao mês" (a prazo), com "recebimento em até 2 dias úteis", sujeito a análise de crédito. Antecipação de boleto a partir de 5,79% a.m. — [Preços](https://www.asaas.com/precos-e-taxas)
- [CONFIRMADO] Antecipação via API: `POST /v3/anticipations/simulate`, depois `POST /v3/anticipations` (por cobrança ou parcelamento inteiro). Webhooks `RECEIVABLE_ANTICIPATION_*`. Cobranças com split seguem regras específicas ("Split em cobranças antecipadas"). — [Antecipação](https://docs.asaas.com/docs/antecipacoes)

**Iugu**
- [CONFIRMADO] `credit_card.installments` e `max_installments` (1 a 12) via `POST v1/accounts/configuration`. "Número máximo de parcelas sem juros: Se deseja cobrar juros a partir de determinada quantidade de parcelas, é possível utilizando o parâmetro `max_installments_without_interest`" (só via API). (página de 2025-10-31) — [Cartão — Configurações extras](https://dev.iugu.com/docs/cc-configuracoes-extras)
- [CONFIRMADO] Split diferente por número de parcelas (`credit_card_Nx_percent/cents`). — [Split de Pagamentos](https://dev.iugu.com/docs/split-de-pagamentos)
- [CONFIRMADO] Antecipação automática por subconta (`auto_advance`, `auto_advance_type`, `auto_advance_option`) e endpoints de antecipação de recebíveis. — [Configurar subconta por API](https://dev.iugu.com/docs/configurar-subconta-por-api); [índice](https://dev.iugu.com/llms.txt)
- [CONFIRMADO, terceiro] Antecipação na Iugu "sob consulta". — [quasa.io](https://quasa.io/pt/media/asaas-ou-iugu-tarifa-publicada-vence-ate-o-volume-pedir-proposta)

### Inferences
- [INFERIDO] Asaas: para repassar juros, o backend calcula o `totalValue` com juros (tabela Price ou o MDR da faixa) e cria o parcelamento com esse valor, exibindo o CET ao comprador. Isso também infla a base do split, então os splits devem ser fixos (`fixedValue`/`totalFixedValue`) para que os recebedores não fiquem com parte dos juros.
- [INFERIDO] Iugu: com `max_installments_without_interest: 1`, a Iugu aplicaria juros ao comprador a partir de 2x. A taxa de juros usada não foi encontrada.

### Gaps
- [NÃO ENCONTRADO] Taxa de juros que a Iugu aplica quando `max_installments_without_interest` é usado.
- [NÃO ENCONTRADO] Prazo D+x de cartão na Iugu (costuma ser D+30 por parcela, sem fonte oficial nesta pesquisa).
- [NÃO ENCONTRADO] Se o Asaas Checkout (hospedado) oferece "juros por conta do comprador".

---

## Q6. Subcontas e recebedores (criação, KYC CPF/CNPJ, documentos, aprovação, conta bancária, custo)

### Takeaway
As duas criam subcontas por API e aceitam pessoa física (CPF). No Asaas, a conta-pai precisa ser CNPJ, os novos integradores passam por um "período de avaliação regulatória" (10 subcontas, R$ 2 mil por subconta, até 60 dias), há dois modelos (não-BaaS, em que o titular entra no Asaas, e BaaS/White Label, com onboarding na marca da plataforma) e o KYC pede documento e selfie. Na Iugu, a verificação exige documento de identidade em base64, aprovação em até 2 dias úteis e domicílio bancário de mesma titularidade, e cada subconta tem custo de manutenção.

### Cited Findings
**Asaas**
- [CONFIRMADO] "A conta-pai que cria as subcontas deve ser de pessoa jurídica (CNPJ)... Contas de pessoa física (CPF) não podem criar subcontas" (Res. Conjuntas BCB nº 16 e 17, BaaS). `POST /v3/accounts` com `incomeValue` obrigatório. `birthDate` só para PF. `companyType`/`taxRegime` (MEI, NATIONAL_SIMPLE...) só para PJ. Webhooks podem ser configurados na criação. (updatedAt 2026-11-17) — [Criação de subcontas](https://docs.asaas.com/docs/criacao-de-subcontas)
- [CONFIRMADO] Período de avaliação regulatória: até 10 subcontas, até R$ 2.000,00 em cobranças por subconta, até 60 dias corridos a partir da primeira subconta em produção. Ao atingir qualquer limite, o bloqueio é automático até concluir o checklist. Também passa a ser "obrigatório evidenciar a marca, os links e os textos de responsabilidade do Asaas" em cada ponto de contato. — [Fluxo de aprovação de subcontas](https://docs.asaas.com/docs/detalhamento-do-fluxo-de-aprovação-de-subcontas)
- [CONFIRMADO] Não-BaaS: o titular recebe e-mail de ativação e envia documento e selfie na interface Asaas. BaaS: a plataforma conduz o fluxo, espera ≥15 s, chama `GET /v3/myAccount/documents` e envia via `onboardingUrl` ou `POST /v3/myAccount/documents/{id}`. Aprovação acompanhada pelo webhook `ACCOUNT_STATUS_GENERAL_APPROVAL_APPROVED`. O BaaS "deve ser previamente alinhado e habilitado". — [Fluxo de aprovação](https://docs.asaas.com/docs/detalhamento-do-fluxo-de-aprovação-de-subcontas); [Subcontas](https://docs.asaas.com/docs/subcontas)
- [CONFIRMADO] Há uma taxa de criação de subconta ("Subcontas não-BaaS encerradas em até 48 horas após a criação não geram cobrança da taxa de criação, quando aplicável"). O valor fica em Configurações > Taxas e não é público. Há confirmação anual de dados comerciais. — [Subcontas](https://docs.asaas.com/docs/subcontas)
- [CONFIRMADO] Transferências: Pix (PJ) com 30 grátis por mês e depois R$ 2,00 cada; Pix PF grátis; TED R$ 5,00. Saques de subconta para a conta bancária do recebedor seguem essas tarifas. — [Preços](https://www.asaas.com/precos-e-taxas)
- [CONFIRMADO] "Mecanismo para validação de saque via webhooks" permite que a plataforma aprove ou recuse transferências. — [índice docs](https://docs.asaas.com/docs/português/llms.txt)

**Iugu**
- [CONFIRMADO] `POST /v1/marketplace/create_account` (sem parâmetro obrigatório), depois `POST /v1/accounts/{id}/request_verification` com o `user_token`. Só é possível verificar "em até 24h após sua criação", e a chamada não pode ser repetida após 200. Campos: `person_type` (Física ou Jurídica), `cpf` (PF), `cnpj` + `resp_cpf` (PJ), `estimated_revenue`, `bank`, agência, conta, `automatic_transfer`. Arquivos em base64 (≤15 MB): `identification` (RG/CNH/Passaporte, obrigatório), `address_proof` (≤90 dias), documentos adicionais. "O prazo para torná-la verified é de até 2 dias úteis." Gatilhos `referrals.verification` e `referrals.document_status_change`. — [Criar, Verificar e Configurar Subconta](https://dev.iugu.com/docs/criar-verificar-e-configurar-subconta); [Configurar subconta por API](https://dev.iugu.com/docs/configurar-subconta-por-api)
- [CONFIRMADO] O saque vai para o "domicílio bancário da mesma titularidade" (`POST v1/bank_verification` para alterar). Existe PIX/TED Out para terceiros. — [índice ref.](https://dev.iugu.com/reference/llms.txt)
- [CONFIRMADO] "Existe um custo de manutenção para cada subconta criada." Valor não publicado. — [Diferença subconta x mestre](https://dev.iugu.com/docs/diferença-de-subconta-para-conta-mestre)
- [CONFIRMADO] O split exige o "Plano Marketplace". — [Split de Pagamentos](https://dev.iugu.com/docs/split-de-pagamentos)

### Inferences
- [INFERIDO] Árbitros e jogadores PF (CPF, sem CNPJ) são aceitos nas duas. A fricção de onboarding (selfie e documento) é real e precisa ser desenhada no app. No Asaas, o modelo BaaS evita mandar o usuário ao app Asaas, mas exige habilitação comercial e branding Asaas obrigatório.
- [INFERIDO] O período de avaliação do Asaas (10 subcontas, R$ 2 mil cada) limita um piloto com muitos árbitros e jogadores. Planejar o checklist regulatório antes do lançamento.

### Gaps
- [NÃO ENCONTRADO] Valores das taxas de criação (Asaas) e manutenção (Iugu) de subconta.
- [NÃO ENCONTRADO] SLA de aprovação de KYC no Asaas.
- [NÃO ENCONTRADO] Se o Asaas valida a titularidade da conta bancária de destino do saque das subcontas.

---

## Q7. Developer experience (API, SDK Node/TS, sandbox, webhooks, idempotência, tokenização no cliente, MCP)

### Takeaway
**Asaas:** docs de 2026 muito atualizadas (llms.txt e MCP oficial em `https://docs.asaas.com/mcp`), sandbox, webhooks com `authToken` no header `asaas-access-token`, entrega at-least-once e fila pausada após 15 falhas. Fraquezas: SDK oficial só em Java, **sem header de idempotência** (a doc recomenda `externalReference` e controle próprio) e **sem lib JS documentada de tokenização no cliente** (cartão enviado pelo backend; tokenização em produção sujeita a habilitação). **Iugu:** `Idempotency-Key` nativo (24 h), `iugu.js` para tokenizar no cliente (inclusive em WebView), webhooks com Basic auth opcional e Content-Type form-urlencoded. Fraqueza: o SDK Node oficial está abandonado (último push em 2021).

### Cited Findings
**Asaas**
- [CONFIRMADO] MCP Server público em `https://docs.asaas.com/mcp`, baseado em docs + OpenAPI. Lista endpoints e schemas e "executar requisições quando a ferramenta estiver autenticada". Também há `llms.txt`. (updatedAt 2026-10-13) — [MCP Server (IA)](https://docs.asaas.com/docs/mcp-1)
- [CONFIRMADO] "Atualmente, o SDK oficial do Asaas está disponível para Java", também usável em Kotlin. Não há SDK Node oficial. — [SDK](https://docs.asaas.com/docs/sdks)
- [CONFIRMADO] Sandbox em `https://api-sandbox.asaas.com`, com guias de saldo de teste e pagamento de QR Pix no sandbox. — [Estornar cobrança (OpenAPI servers)](https://docs.asaas.com/reference/estornar-cobranca); [índice docs](https://docs.asaas.com/docs/português/llms.txt)
- [CONFIRMADO] Autenticação por header `access_token`. Limites: 25.000 requisições por conta a cada 12 h, 50 GETs concorrentes, rate limit por endpoint com headers `RateLimit-*`, HTTP 429 sem `Retry-After`. (updatedAt 2026-10-27) — [Limites da API](https://docs.asaas.com/reference/rate-e-quota-limit)
- [CONFIRMADO] Webhooks: até 10 por conta, entrega "at least once", deduplicação pelo `id` do evento, só HTTP 200 conta como sucesso. Novas tentativas em intervalos progressivos. "Após 15 falhas consecutivas, a fila... é interrompida". Eventos antigos de fila parada são removidos depois de um tempo. `authToken` (definido pelo integrador, >32 caracteres) enviado no header `asaas-access-token`, sem assinatura HMAC. Há whitelist de IPs. — [Introdução - Webhooks](https://docs.asaas.com/docs/sobre-os-webhooks); [FAQ de Webhooks](https://docs.asaas.com/docs/faq-de-webhooks); [Criar webhook pela API](https://docs.asaas.com/docs/criar-novo-webhook-pela-api)
- [CONFIRMADO] Idempotência na criação: a doc trata "Cobrança duplicada após retry sem idempotência" e recomenda identificador interno, `externalReference`, e consultar antes de recriar. Não cita header de idempotência. — [Cobrança duplicada após retry](https://docs.asaas.com/docs/cobrança-duplicada-após-retry-sem-idempotência)
- [CONFIRMADO] Tokenização `POST` (cartão vinculado ao `customer`): "disponível para homologação no Sandbox. Em Produção, a funcionalidade depende de habilitação e está sujeita à análise." Os exemplos de cobrança enviam os dados do cartão por chamada server-side e exigem HTTPS. — [Tokenização](https://docs.asaas.com/reference/tokenizacao-de-cartao-de-credito); [Cobranças via cartão](https://docs.asaas.com/docs/cobrancas-via-cartao-de-credito)
- [CONFIRMADO] Existe o Asaas Checkout hospedado (Pix, cartão, parcelamento, callbacks). — [índice docs](https://docs.asaas.com/docs/português/llms.txt)

**Iugu**
- [CONFIRMADO] `Idempotency-Key` no header, válido por 24 h. Repetição devolve 409, e `resource_id: "processing"` indica que a requisição original ainda roda. (updatedAt 2026-09-16) — [Chave de Idempotência](https://dev.iugu.com/docs/chave-de-idempotência-1)
- [CONFIRMADO] `iugu.js` tokeniza no cliente ("dados... enviados para o cofre da iugu e... retorna um token"). O endpoint `/v1/payment_token` chamado do servidor "sujeitará auditorias do PCI". Para mobile, "Utilize um webview como alternativa". Existe Zero Auth. — [Tokenização de Cartão](https://dev.iugu.com/docs/tokenizacao-de-cartao-de-credito)
- [CONFIRMADO] SDKs listados: PHP, Ruby, .NET, Java, node.js, Python. — [Bibliotecas SDK](https://dev.iugu.com/docs/bibliotecas). O repo `iugu/iugu-node` tem 33 estrelas, `pushed_at` 2021-11-27 e não está arquivado: **abandonado na prática** (verificado via GitHub API em 2026-11-24). — [github.com/iugu/iugu-node](https://github.com/iugu/iugu-node)
- [CONFIRMADO] Gatilhos (webhooks): `Content-Type: application/x-www-form-urlencoded` (formato `data[...]`), IP de saída 98.82.243.132 (mudou em 04/08/2025), limite de 20 gatilhos. O campo `authorization` funciona como chave "Basic Authentication". Há reenvio manual por log e por período. — [Gatilhos](https://dev.iugu.com/docs/gatilhos); [Criar Gatilho](https://dev.iugu.com/reference/criar-gatilho); [Reenviar gatilhos](https://dev.iugu.com/docs/reenviar-gatilhos-para-o-webhook)
- [CONFIRMADO] Assinatura RSA obrigatória em endpoints sensíveis (ex.: transferência entre contas). Changelog de 28/07/2026: "Assinatura RSA: Múltiplos Tokens e Novo Parâmetro X-Signature-Token-Id". Há receita Node.js de RSA. — [índice dev.iugu.com](https://dev.iugu.com/llms.txt); [Transferir Valor](https://dev.iugu.com/reference/transferir-valor)
- [CONFIRMADO] Os docs da Iugu também publicam `llms.txt` (readme.io). — [dev.iugu.com/llms.txt](https://dev.iugu.com/llms.txt)

### Inferences
- [INFERIDO] Asaas e PCI: sem lib JS de tokenização no cliente, o PAN passa pelo backend Fastify, o que coloca o backend no escopo PCI-DSS (SAQ D). As alternativas são o Asaas Checkout hospedado (fora do escopo) ou a tokenização com dados enviados uma única vez. Esse é um ponto relevante contra o Asaas para o app em Capacitor.
- [INFERIDO] Iugu: o `iugu.js` roda na WebView do Capacitor (a própria doc sugere WebView para mobile), o que reduz o escopo PCI (SAQ A-EP).
- [INFERIDO] Nas duas, o cliente Node/TS seria escrito em casa (fetch + tipos gerados do OpenAPI, que as duas expõem nos `.md` de referência).
- [INFERIDO] Webhooks: nenhuma das duas assina com HMAC. Usar token secreto ou Basic auth, whitelist de IP e reconsulta do recurso pela API antes de efetivar.

### Gaps
- [NÃO ENCONTRADO] Política de retentativa automática dos gatilhos da Iugu (intervalos e número).
- [NÃO ENCONTRADO] MCP oficial da Iugu.

---

## Q8. Preços (data da página)

### Takeaway
O Asaas publica preços (página capturada em 2026-11-24, com valores padrão e promocionais de 3 meses e alguma inconsistência entre capturas). A Iugu não tem página pública de preços acessível: são planos com mensalidade "sob consulta", e o único dado encontrado vem de um comparador terceiro de out/2026.

### Cited Findings
- [CONFIRMADO] Asaas (2026-11-24, padrão / promo): Pix R$ 2,15 / R$ 1,08; boleto R$ 2,15 / R$ 1,08; crédito à vista R$ 0,49 + 2,99% / 1,99%; 2 a 6x 3,49% / 2,49%; 7 a 12x 3,99% / 2,99%; 13 a 21x 4,29% / 3,29% (+R$ 0,49); débito R$ 0,35 + 1,89%; antecipação de cartão 1,25% a.m. (à vista) e 1,70% a.m. (parcelado); antecipação de boleto a partir de 5,79% a.m.; Pix out PJ com 30 grátis por mês e depois R$ 2,00; TED R$ 5,00; notificação WhatsApp R$ 0,55; e-mail + SMS R$ 0,99; NF-e R$ 0,49. Sem mensalidade nem adesão. "As taxas apresentadas são padrão. Para conferir as condições aplicadas ao seu contrato, consulte-as diretamente na sua conta." — [Asaas Preços e taxas](https://www.asaas.com/precos-e-taxas)
- [CONFLITO] Outra leitura do mesmo URL no mesmo dia mostrou Pix e boleto a "R$ 0,79 / R$ 1,59", e o quasa.io (out/2026) cita R$ 1,99. — [Asaas Preços](https://www.asaas.com/precos-e-taxas); [quasa.io](https://quasa.io/pt/media/asaas-ou-iugu-tarifa-publicada-vence-ate-o-volume-pedir-proposta)
- [NÃO ENCONTRADO] Asaas: tarifa de chargeback, de subconta e de Conta Escrow (todas só aparecem logado, em Configurações > Taxas).
- [CONFIRMADO, terceiro] Iugu Essencial (out/2026): Pix 0,99%, boleto R$ 2,19, cartão 3,34% + R$ 0,40 de processamento, mensalidade "consulte", antecipação sob consulta. — [quasa.io](https://quasa.io/pt/media/asaas-ou-iugu-tarifa-publicada-vence-ate-o-volume-pedir-proposta)
- [CONFIRMADO] Iugu: o estorno cobra MDR, e há custo de manutenção por subconta (valor não publicado). — [Reembolsar Faturas](https://dev.iugu.com/docs/realizar-o-reembolso-de-faturas-estorno-por-api); [Subconta x mestre](https://dev.iugu.com/docs/diferença-de-subconta-para-conta-mestre)

### Inferences
- [INFERIDO] Para comparar custo total no Bora Jogar, falta pedir proposta à Iugu (Plano Marketplace) e confirmar no painel Asaas as taxas de subconta, Escrow e chargeback.

### Gaps
- [NÃO ENCONTRADO] Tarifa de chargeback das duas, e parcelado vs à vista da Iugu.

---

## Q9. Limitações, reclamações, bloqueios

### Takeaway
Há relatos isolados de bloqueio de saldo ou saque nas duas (Reclame Aqui, 2025–2026), sem padrão quantificado. O Asaas tem bons índices no Reclame Aqui (98,3% respondidas, nota 8,5). As principais limitações técnicas e regulatórias que encontrei vêm das próprias docs: período de avaliação, branding obrigatório e split desabilitado por padrão no Asaas; Pix sem estorno parcial, split proporcional no estorno e SDK Node parado na Iugu.

### Cited Findings
- [CONFIRMADO] Perfil do Asaas no Reclame Aqui: 98,3% respondidas, 91,5% resolvidas, nota 8,5/10 (6 meses), segundo resumo da busca. — [Reclame Aqui Asaas](https://www.reclameaqui.com.br/empresa/asaas-gestao-financeira/)
- [CONFIRMADO, relato de usuário] Reclamação "Sem suporte, saldo bloqueado" no Asaas. — [Reclame Aqui](https://www.reclameaqui.com.br/asas/sem-suporte-saldo-bloqueado_VUBT-EHMbDtlC1AT/). Comentários no blog do Asaas citam "dinheiro retido desde 17/11/2025". — [Blog Asaas Ouvidoria](https://blog.asaas.com/release/ouvidoria-do-asaas/)
- [CONFIRMADO, relato de usuário] Iugu: "Conta Bloqueada para Saque Após Solicitação de Revisão de Tarifas" (jun/2026, marcada como resolvida) — [Reclame Aqui](https://www.reclameaqui.com.br/iugu/conta-bloqueada-para-saque-apos-solicitacao-de-revisao-de-tarifas_VNnBNAq1DsWPDB77/). Limite de saque diário de 30 mil e mensal de R$ 100 mil numa conta Iugu vinculada a ERP (mar/2026) — [Reclame Aqui](https://www.reclameaqui.com.br/egs-sistemas/problemas-com-limites-de-saque-e-transferencia-na-conta-digital-iugu-vinculada-ao-erp-egs-sistemas_DGPsNCS8Zq68m_BK/)
- [CONFIRMADO] Limitações documentadas: limites do período de avaliação regulatória e branding Asaas obrigatório — [Fluxo de aprovação](https://docs.asaas.com/docs/detalhamento-do-fluxo-de-aprovação-de-subcontas); split desabilitado por padrão — [Split](https://docs.asaas.com/docs/split-de-pagamentos); sem webhook de estado do escrow — [Valores sob garantia](https://docs.asaas.com/docs/bloqueio-dos-valores); tokenização em produção sujeita a análise — [Tokenização](https://docs.asaas.com/reference/tokenizacao-de-cartao-de-credito).
- [CONFIRMADO] Iugu: split só entre contas Iugu; Pix sem estorno parcial; split de 100% é ignorado — [Split](https://dev.iugu.com/docs/split-de-pagamentos); [Reembolso Pix](https://dev.iugu.com/reference/reembolso-pix).

### Inferences
- [INFERIDO] Num marketplace com muitos recebedores PF, o risco de bloqueio cautelar (antifraude/PLD) existe nas duas. Para mitigar: KYC bem feito, descrição clara do negócio no onboarding e volume crescente.

### Gaps
- [NÃO ENCONTRADO] Histórico de indisponibilidades (status pages) das duas em 2025–2026.
- [NÃO ENCONTRADO] Nota agregada da Iugu no Reclame Aqui.
- [NÃO ENCONTRADO] Discussões de desenvolvedores (fóruns, GitHub issues) recentes.

---

## Q10. Resumo de aderência ao Bora Jogar (síntese do pesquisador)

### Takeaway
**O Asaas cobre nativamente os 3 requisitos críticos:** split para N recebedores, custódia com liberação por API ou data (Conta Escrow) e estorno parcial com valor escolhido por recebedor (`splitRefunds`), inclusive em Pix. **A Iugu falha em custódia nativa e em estorno parcial por recebedor/Pix**, mas é melhor em idempotência, tokenização client-side (iugu.js) e repasse de juros do parcelamento (`max_installments_without_interest`).

### Cited Findings
- Escrow nativo Asaas — [Introdução - Conta Escrow](https://docs.asaas.com/docs/introducao-conta-escrow); `splitRefunds` — [Estornar cobrança](https://docs.asaas.com/reference/estornar-cobranca); Pix parcial não suportado na Iugu — [Reembolso Pix Iugu](https://dev.iugu.com/reference/reembolso-pix); rateio proporcional na Iugu — [Reembolso Cartão Iugu](https://dev.iugu.com/reference/reembolso-cartao-de-credito); iugu.js — [Tokenização Iugu](https://dev.iugu.com/docs/tokenizacao-de-cartao-de-credito); Idempotency-Key Iugu — [Chave de Idempotência](https://dev.iugu.com/docs/chave-de-idempotência-1).

### Inferences
- [INFERIDO] Pontos a validar no Asaas antes de decidir:
  - mensalidade da Escrow × número de árbitros e jogadores (cada recebedor retido paga ou gera mensalidade);
  - como `splitRefunds` interage com uma garantia ativa;
  - escopo PCI sem tokenização no cliente (avaliar Asaas Checkout ou tokenização em produção);
  - repasse de juros calculado pela própria plataforma;
  - limites do período de avaliação regulatória.

### Gaps
- Ver os Gaps de cada seção. Os maiores: valores de taxas por subconta e Escrow (Asaas), preços oficiais da Iugu, prazos D+x da Iugu.
