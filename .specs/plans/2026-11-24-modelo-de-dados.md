# Estudo do modelo de dados e diagrama de relacionamento — prompt para sessão nova

Cole tudo abaixo da linha numa sessão nova do Claude Code aberta em `/Users/yokuny/Documents/GitHub/quadrativa`.

---

Você vai fazer o **estudo do modelo de dados** do Bora Jogar e desenhar o **diagrama de relacionamento do banco (Postgres)** com a skill `excalidraw-diagram`. É um estudo de arquitetura: **não escreva código de aplicação, migração nem scaffold**.

## Contexto do projeto

Bora Jogar é um app (Android/iOS via Capacitor + versão web mobile-first) que liga times, donos de quadra, árbitros e jogadores de aluguel de uma mesma cidade, com reserva de quadra, contratação e pagamento dentro do app. Modalidades: futsal e society. Cada cidade é uma região (piloto: São Mateus do Sul/PR). Uma mesma pessoa pode ter vários perfis (capitão de time, dono de quadra, árbitro, jogador). O repositório ainda não tem código: só documentação e decisões.

## Leia antes de tudo, nesta ordem

1. `.specs/STATE.md` — decisões ativas **AD-001 a AD-007**. Elas são restrições, não sugestões:
   - AD-001/002: TypeScript em tudo; API em **Fastify** como monólito modular; arquitetura garantida por skill do projeto e testes de fronteira.
   - AD-003: React + Capacitor + web mobile-first.
   - AD-004: **PostgreSQL + PostGIS**, gerenciado.
   - AD-005: fluxo `tlc-spec-driven`; a arquitetura de referência do Ethica (`/Users/yokuny/Documents/GitHub/Ethica/.agents/skills/monorepo-architecture/`) é a base, adaptada para Fastify + Postgres e **sem multi-tenant** (cidade é filtro, não tenant).
   - AD-006: commits direto na `main`, Conventional Commits em pt-BR, **sem `Co-Authored-By` nem menção a LLM**, push sem perguntar depois de validar.
   - AD-007: gateway **Asaas** — cada recebedor (quadra, árbitro, jogador) é uma **subconta Asaas** em nome próprio; cobrança com **split**; retenção até o fim do jogo pela **Conta Escrow**; cancelamento e W.O. por **estorno parcial com `splitRefunds`**; o dinheiro de terceiros nunca passa pela conta da plataforma.
2. `briefing.md` — **fonte da verdade das regras de negócio** (sessões 1 a 11: início, quadras, reserva e pagamento, contratação, jogos, check-in, W.O., ranking e avaliações, perfil e painéis, cadastros, notificações, regras financeiras e prazos, decisões). Leia inteiro. `fluxo_contratacao.png` é a imagem do fluxo de contratação citada nele. `Documentação App REV1.2.pdf` é a versão em PDF do mesmo conteúdo; consulte só se algo do `briefing.md` parecer cortado.
3. `reports/Gateways de pagamento para marketplace.md` — seções sobre o Asaas e "Cinco regras do briefing não sobrevivem…". Elas dizem quais regras financeiras do briefing vão mudar.
4. `Bora Jogar · protótipo.html` — **só sugestão visual, não vincula** (decisão do usuário). Use-o como fonte de **campos e estados**: o script tem as coleções de exemplo `SEED_T` (times), `SEED_Q` (quadras), `SEED_J` (jogadores), `SEED_A` (árbitros), `SEED_G` (jogos), `SEED_D` (desafios) e as constantes `TARIFAS`, `TIPOS`, `hab`, `carac`. Extraia nomes de campos, enums e estados; não copie o modelo do protótipo como está (ele guarda tudo num documento só via `window.claude.use('db')`).

Não leia as notas em `research_notes/` (o relatório já as consolida).

## O que entregar

### 1. `docs/architecture/data-model.md` — o modelo em texto

- **Glossário**: um termo em inglês por conceito, usado igual em tabela, contrato e código, com o termo em pt-BR do briefing ao lado (ex.: quadra `Court`, desafio `Challenge`, reserva `Booking`, jogo `Match`, contratação `Hire`, árbitro `Referee`, jogador de aluguel `RentalPlayer`, W.O. `Walkover`, caução `Deposit`, cartão vermelho `Suspension`…). Proponha; termos duvidosos vão para as perguntas abertas.
- **Tabelas**: para cada uma, colunas principais com tipo Postgres, chaves, índices relevantes, restrições (`CHECK`, `UNIQUE`, FKs com `ON DELETE`) e qual módulo da API é dono dela (cada tabela tem **um** módulo dono, regra da arquitetura do Ethica).
- **Relacionamentos** com cardinalidade (1:1, 1:N, N:N com tabela de ligação).
- **Máquinas de estado** como enums: jogo/reserva, convite de time, contratação de jogador/árbitro, pagamento, estorno, liberação do escrow, desafio, check-in, denúncia. Para cada uma, os estados e as transições válidas, apontando a seção do briefing.
- **Pontos que o banco precisa garantir**, no mínimo:
  - impedir **reserva sobreposta** na mesma quadra (faixa de tempo `tstzrange` + `EXCLUDE USING gist` com `btree_gist`), com durações de 1h, 1h30 e 2h;
  - **dinheiro em centavos inteiros** e um **registro contábil append-only** (lançamentos que nunca são editados) que explique cada centavo: cobrança, split por recebedor, taxa de 5%, tarifa do meio de pagamento, estorno, liberação, pendência do W.O.;
  - **idempotência** de webhooks e chamadas ao Asaas (o Asaas não tem header de idempotência: usar `externalReference` e uma tabela de eventos recebidos com chave única);
  - geolocalização da quadra e do check-in com **PostGIS** (`geography(Point,4326)`);
  - fila de jobs do worker no próprio Postgres (prazos de 2h, janela de check-in, encerramento automático, polling do escrow) — modele a tabela ou indique `pg-boss`, sem escolher em definitivo;
  - **dados bancários do recebedor ficam no Asaas**: o banco guarda só o id da subconta e um resumo mascarado (LGPD);
  - notificações (sessão 9 do briefing) com destino de navegação e marca de lida;
  - avaliações com as 4 notas, a regra "só aparece quando os dois avaliarem", e o peso do W.O. no ranking;
  - moderação (texto bloqueado) e denúncias de perfil;
  - sugestões dos testadores e o papel de administrador do app.
- **Mapeamento Asaas**: que id do Asaas fica em qual tabela (subconta, cliente pagador, cobrança, split, escrow, estorno, webhook) e quais estados do Asaas viram quais estados nossos. Confirme os nomes de campos com o **MCP do Asaas** (`search`, `get-endpoint`) em vez de supor.
- **Regras do briefing em conflito com o AD-007** (caução da arbitragem, 5% do outro time no cancelamento, 30% no W.O., taxa nunca devolvida, liberação no fim do jogo para cartão): **não decida**. Modele de forma que as duas saídas caibam e liste cada uma nas perguntas abertas com a sua recomendação.
- **Perguntas abertas** ao final: tudo que é decisão de produto do usuário, cada uma com recomendação.

### 2. `docs/architecture/data-model.excalidraw` — o diagrama

Use a skill `excalidraw-diagram` para o desenho (elementos, campos obrigatórios, `fontFamily: 5`, paleta), **com estes ajustes** sobre o que ela manda:
- Salve como **arquivo `.excalidraw` com o JSON puro** (`{"type":"excalidraw","version":2,...}`), **não** no formato Markdown do Obsidian: o time usa VS Code e excalidraw.com.
- Textos em pt-BR para títulos e legendas; nomes de tabelas e colunas como no glossário. **Não** troque aspas por 『』 nem parênteses por 「」: no JSON puro basta escapar.
- Ignore o limite de canvas 1200×800: um diagrama de entidades precisa de mais espaço. Agrupe as tabelas por domínio com um retângulo de fundo e título para cada um (sugestão: Pessoas e perfis · Cidades e quadras · Times e desafios · Jogos e reservas · Contratações · Pagamentos e contabilidade · Reputação e comunidade · Notificações e jobs).
- Cada tabela é um retângulo com o nome em destaque e as colunas principais (PK, FKs, status, valores); cada relacionamento é uma seta ligada às duas caixas (`startBinding`/`endBinding` preenchidos e a seta listada em `boundElements` das caixas), com a cardinalidade escrita junto da seta.
- Se ficar ilegível numa tela só, faça um **diagrama geral** só com tabelas e setas e **um diagrama por domínio** com as colunas (`data-model.<domínio>.excalidraw`).

### 3. Validação antes de commitar

- Cada `.excalidraw` passa em `python3 -c "import json,sys; json.load(open(sys.argv[1]))" <arquivo>`, com ids únicos e toda seta com as duas pontas ligadas a caixas existentes (escreva um verificador curto no scratchpad, não no repo).
- Toda tabela do `.md` aparece no diagrama e vice-versa.
- Toda regra do briefing que muda dado tem lugar no modelo: releia as sessões 3, 4, 5, 10 e 11 do briefing e confira item a item; o que não couber vira pergunta aberta.

## Como trabalhar

- Use o **Context7** para fatos de Postgres/PostGIS/Drizzle (por exemplo `EXCLUDE USING gist`, tipos de faixa, `geography`), e o **MCP do Asaas** para os objetos do Asaas. Não invente API nem campo: o que não confirmar, marque como "a confirmar".
- Não crie decisões `AD-NNN` sozinho: o que for decisão de produto fica nas perguntas abertas para o usuário.
- Ao terminar: commit `docs(modelo-de-dados): …` direto na `main`, sem `Co-Authored-By`, e `git push origin main` (AD-006). Mova este arquivo para `.specs/features/modelo-de-dados/plans/2026-11-24-modelo-de-dados.md` no mesmo commit.
- Responda em pt-BR com: o caminho dos arquivos, um resumo do modelo em poucas linhas, e as perguntas abertas que bloqueiam a spec de pagamentos.
