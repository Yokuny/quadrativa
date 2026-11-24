# Modelo de dados do Bora Jogar (PostgreSQL + PostGIS)

> **Estado:** estudo de arquitetura de 24/11/2026. Ainda não é migração nem decisão: o que é decisão de produto está nas [perguntas abertas](#10-perguntas-abertas) e o que depende de teste no sandbox do Asaas está marcado como **a confirmar**.
>
> **Fontes:** `.specs/STATE.md` (AD-001 a AD-007), `briefing.md` (regras de negócio, sessões 1 a 11), `reports/Gateways de pagamento para marketplace.md` (Asaas e as cinco regras em conflito) e os dados de exemplo de `Bora Jogar · protótipo.html` (`SEED_T`, `SEED_Q`, `SEED_J`, `SEED_A`, `SEED_G`, `SEED_D`, `TARIFAS`), usados só como fonte de campos e estados. Os campos do Asaas foram conferidos no MCP oficial (`/v3/accounts`, `/v3/customers`, `/v3/payments`, `/v3/payments/{id}/refund`, `/v3/installments/{id}/refund`, `/v3/payments/{id}/escrow`, `/v3/escrow/{id}/finish`, `/v3/accounts/{id}/escrow` e as páginas de idempotência de webhook e de split com Conta Escrow). Os fatos de Postgres, PostGIS e pg-boss foram conferidos no Context7.
>
> **Diagramas:** [`data-model.excalidraw`](data-model.excalidraw) mostra a visão geral (tabelas e setas). Há também um arquivo por domínio, com as colunas principais: `data-model.pessoas`, `.quadras`, `.times`, `.jogos`, `.contratacoes`, `.pagamentos`, `.contabilidade`, `.reputacao` e `.notificacoes` (todos `.excalidraw`). Abrem no excalidraw.com ou na extensão do VS Code. Nos diagramas, ◆ marca coluna de estado e ¢ marca dinheiro em centavos. Caixa tracejada é tabela de outro domínio. O rótulo da seta é a cardinalidade pai:filho.

## Convenções

- Tabelas e colunas em inglês, `snake_case`, nome da tabela no plural. O termo é o do glossário e vale igual em tabela, contrato (`packages/contracts`) e código.
- Chave primária `uuid` com `gen_random_uuid()`. As exceções são `ledger_entries` e `jobs`, que usam `bigint` identity porque são volumosas e só crescem.
- Tempo sempre em `timestamptz`. Os prazos ("2h, terminando 1h antes") são calculados no fuso da cidade (`cities.timezone`) e **gravados** na linha (`respond_by`, `pay_by`, `withdraw_until`). A regra de "metade do tempo até o limite" depende do momento do convite, por isso o prazo não pode ser derivado depois.
- Dinheiro em **centavos inteiros** (`bigint`, sufixo `_cents`), nunca `numeric` nem `float`. O Asaas trabalha com reais decimais (`value: 129.9`). A conversão é feita só na borda (cliente HTTP do Asaas): o valor é lido como texto decimal e convertido para centavos sem passar por `float`.
- Estados são `ENUM` do Postgres. Transições válidas ficam no service do módulo dono, e o banco garante as invariantes que não dependem de fluxo (`CHECK`, `UNIQUE` parcial, `EXCLUDE`).
- Usuário nunca é apagado: é anonimizado (`users.anonymized_at`), porque há lançamentos financeiros ligados a ele. Por isso as FKs para tabelas com dinheiro ou histórico são `ON DELETE RESTRICT`. `CASCADE` só aparece em filhos puros (fotos, habilidades, tokens de push, candidaturas).
- Extensões: `postgis`, `btree_gist` (exclusão de reservas), `citext` (e-mail), `pgcrypto` (só se o Postgres for anterior ao 13, para ter `gen_random_uuid()`).
- O ORM ou query builder (Drizzle, Kysely ou SQL puro) não está decidido. Os tipos aqui são SQL.

## 1. Glossário

Um termo em inglês por conceito. Os termos marcados com † são propostas que ainda precisam de confirmação (Q18).

| pt-BR (briefing) | Inglês (tabela, contrato, código) | Observação |
| :--- | :--- | :--- |
| usuário, pessoa | `User` (`users`) | uma pessoa com vários perfis |
| equipe do app, dono do app | `admin` (`users.app_role`) | vê sugestões e denúncias |
| cidade, região | `City` (`cities`) | filtro, não tenant |
| quadra | `Court` (`courts`) | |
| dono de quadra | `court owner` (`courts.owner_user_id`) | não é tabela própria |
| modalidade (futsal, society) | `court_type` | enum `futsal`, `society` |
| piso | `surface` | |
| estrutura (vestiário etc.) | `amenities` | |
| time | `Team` (`teams`) | |
| capitão | `captain` (`teams.captain_user_id`) | |
| nível (iniciante, intermediário, avançado) | `skill_level` | `beginner`, `intermediate`, `advanced` |
| desafio (aberto) | `Challenge` (`challenges`) | |
| vaga / "time pedindo jogador" | `PlayerOpening` (`player_openings`) | † |
| "Quero jogar" | `OpeningApplication` (`opening_applications`) | † |
| reserva, horário | `Booking` (`bookings`) | a ocupação da faixa de tempo da quadra |
| jogo | `Match` (`matches`) | |
| time que marcou / time convidado | `home team` / `away team` | |
| divisão 50/50 | `split_mode` | não confundir com o split do gateway |
| convite de jogo (time convidado) | `TeamInvitation` (`team_invitations`) | |
| contratação | `Hire` (`hires`) | serve para árbitro e jogador |
| convite (para jogar ou apitar) | `Hire` em `invited` | o convite é o início da contratação |
| árbitro | `Referee` (`referees`) | |
| função do árbitro (principal, segundo, anotador) | `referee_function` | `main_referee`, `second_referee`, `scorekeeper` |
| jogador de aluguel | `RentalPlayer` (`rental_players`) | † alternativa: `HiredPlayer` |
| posição | `player_position` | |
| passe livre | `free_pass` | jogador sem cobrança |
| habilidade / característica | `ProfileSkill` (`profile_skills`) | uma tabela para os dois |
| confirmação de habilidade por um time | `SkillEndorsement` (`skill_endorsements`) | |
| parte de cada time na contratação | `HireShare` (`hire_shares`) | |
| check-in | `CheckIn` (`check_ins`) | |
| W.O. | `Walkover` (`walkovers`) | † alternativa: `NoShow` |
| cartão vermelho | `Suspension` (`suspensions`) | |
| placar | `score` (`matches.home_score`, `away_score`) | |
| avaliação do time | `TeamReview` (`team_reviews`) | |
| pontualidade, jogo limpo, comprometimento, organização | `punctuality`, `fair_play`, `commitment`, `organization` | |
| selos | `badges` | derivados, sem tabela |
| denúncia de perfil | `ProfileReport` (`profile_reports`) | |
| revisão automática / texto bloqueado | `moderation` (`moderation_blocks`, `blocked_terms`) | |
| sugestão | `Suggestion` (`suggestions`) | |
| notificação | `Notification` (`notifications`) | |
| recebedor / dados de recebimento | `Payee` (`payees`) | = subconta Asaas |
| pagador | `PayerCustomer` (`payer_customers`) | = cliente Asaas |
| cobrança (tela de pagamento) | `Charge` (`charges`) | a nossa |
| cobrança no Asaas | `GatewayPayment` (`gateway_payments`) | `pay_…` |
| linhas do valor | `ChargeItem` (`charge_items`) | |
| taxa de serviço (5%) | `service_fee` | |
| tarifa do meio de pagamento | `payment_tariff` | |
| acréscimo do parcelamento | `installment_surcharge` | |
| caução | `Deposit` (`referee_deposit`) | † depende da Q1 |
| split (gateway) | `ChargeSplit` (`charge_splits`) | |
| estorno / reembolso | `Refund` (`refunds`) | |
| retenção até o fim do jogo | `EscrowHold` (`escrow_holds`) | Conta Escrow |
| liberação / repasse | `release` | encerrar a garantia |
| acerto (tabela do W.O., do cancelamento) | `Settlement` (`settlements`, `settlement_lines`) | |
| pendência, saldo devedor | `Debt` (`debts`, `debt_items`) | |
| taxa administrativa de cancelamento (10%) | `cancellation_fee` | taxa em `settlements.rates` |
| registro contábil | `Ledger` (`ledger_*`) | |

## 2. Domínios e módulos donos

Pela arquitetura de referência do Ethica, **cada tabela tem um único módulo dono**: só o repository desse módulo lê e escreve nela, e os outros módulos passam pelo service dele. O agrupamento em domínios do diagrama é visual. O módulo é a fronteira de código.

| Módulo (`apps/api/src/modules/…`) | Tabelas de que é dono |
| :--- | :--- |
| `identity` | `users` |
| `profiles` | `referees`, `rental_players`, `profile_skills` |
| `geo` | `cities` |
| `courts` | `courts`, `court_photos` |
| `teams` | `teams` |
| `matches` | `challenges`, `bookings`, `matches`, `team_invitations`, `check_ins` |
| `hiring` | `player_openings`, `opening_applications`, `hires`, `hire_shares` |
| `payments` | `payees`, `payer_customers`, `charges`, `charge_items`, `gateway_payments`, `charge_splits`, `refunds`, `refund_splits`, `escrow_holds`, `asaas_webhook_events` |
| `settlement` | `settlements`, `settlement_lines`, `debts`, `debt_items` |
| `ledger` | `ledger_accounts`, `ledger_transactions`, `ledger_entries` |
| `reputation` | `team_reviews`, `provider_reviews`, `skill_endorsements`, `walkovers`, `suspensions` |
| `community` | `blocked_terms`, `moderation_blocks`, `profile_reports`, `suggestions` |
| `notifications` | `notifications`, `push_devices` |
| `worker` | `jobs` |

Dependências esperadas, sem ciclos: `identity`, `geo` ← `profiles`, `courts`, `teams` ← `matches` ← `hiring` ← `payments` ← `settlement` ← `ledger`. `reputation`, `community` e `notifications` reagem por evento. `matches` precisa saber se um time está bloqueado por pendência. Essa consulta vai ao service de `settlement`, e essa dependência inverte o sentido. Para não haver ciclo, `settlement` emite `settlement:debt_opened` e `settlement:debt_paid`, e `teams` mantém uma marca de bloqueio. Uma alternativa é `matches` consultar uma função pura de `settlement` que não volta a importar `matches`. A escolha fica para o design da `foundation`.

## 3. Tabelas

### Pessoas e perfis

#### `users`

Módulo dono: **`identity`**. Pessoa que usa o app. Um usuário pode ser capitão, dono de quadra, árbitro e jogador ao mesmo tempo (os perfis apontam para ele).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `name` | `text` | `NOT NULL`, passa pela moderação |
| `email` | `citext` | `UNIQUE`; login |
| `phone` | `text` | celular, opcional |
| `home_city_id` | `uuid` | **FK** → `cities`; cidade escolhida no topo |
| `app_role` | `app_role` | `member` ou `admin` (equipe do app: sugestões e denúncias) |
| `cpf_last4` | `text` | só os 4 últimos dígitos, para exibição; o CPF completo é a pergunta aberta Q10 |
| `anonymized_at` | `timestamptz` | LGPD: usuário é anonimizado, nunca apagado (há lançamentos financeiros ligados) |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (anonymized_at IS NULL OR email IS NULL)`.

#### `referees`

Módulo dono: **`profiles`**. Perfil de árbitro (sessão 8). Um por usuário.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users`, `UNIQUE` |
| `city_id` | `uuid` | **FK** → `cities` |
| `neighborhood` | `text` |  |
| `court_types` | `court_type[]` | modalidades; `CHECK (cardinality(court_types) > 0)` |
| `functions` | `referee_function[]` | `main_referee`, `second_referee`, `scorekeeper`; ao menos uma |
| `years_experience` | `smallint` | `CHECK (>= 0)` |
| `certification` | `text` | curso ou certificação |
| `price_cents` | `bigint` | valor por jogo; `CHECK (> 0)` |
| `bio` | `text` | moderada |
| `payee_id` | `uuid` | **FK** → `payees`; obrigatório para publicar |
| `status` | `profile_status` | `draft`, `published`, `hidden` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (status <> 'published' OR payee_id IS NOT NULL)`.

Índices: `(city_id, status)` para a lista de Contratar.

#### `rental_players`

Módulo dono: **`profiles`**. Perfil de jogador de aluguel (sessão 8). Um por usuário.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users`, `UNIQUE` |
| `city_id` | `uuid` | **FK** → `cities` |
| `neighborhood` | `text` |  |
| `court_types` | `court_type[]` | modalidades |
| `positions` | `player_position[]` | ao menos uma; o filtro da lista usa goleiro × linha |
| `level` | `skill_level` | `beginner`, `intermediate`, `advanced` |
| `free_pass` | `boolean` | passe livre: sem cobrança e sem taxa |
| `price_cents` | `bigint` | valor por jogo |
| `bio` | `text` | moderada |
| `payee_id` | `uuid` | **FK** → `payees`; ver Q16 para passe livre |
| `status` | `profile_status` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK ((free_pass AND price_cents = 0) OR (NOT free_pass AND price_cents > 0))`; `CHECK (status <> 'published' OR free_pass OR payee_id IS NOT NULL)`.

Índices: `(city_id, status)`.

#### `profile_skills`

Módulo dono: **`profiles`**. Habilidade do jogador ou característica do árbitro (lista fixa ou escrita à mão). A contagem de times que confirmaram vem de `skill_endorsements`.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `referee_id` | `uuid` | **FK** → `referees` `ON DELETE CASCADE` |
| `rental_player_id` | `uuid` | **FK** → `rental_players` `ON DELETE CASCADE` |
| `label` | `text` | moderado quando `is_custom` |
| `is_custom` | `boolean` | escrita à mão |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (num_nonnulls(referee_id, rental_player_id) = 1)`; `UNIQUE NULLS NOT DISTINCT (referee_id, rental_player_id, label)`.

### Cidades e quadras

#### `cities`

Módulo dono: **`geo`**. Região. A cidade é filtro de tudo, não tenant (AD-005).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `name` | `text` | `NOT NULL` |
| `state` | `char(2)` | UF |
| `ibge_code` | `integer` | `UNIQUE` |
| `timezone` | `text` | `DEFAULT 'America/Sao_Paulo'`; prazos são calculados neste fuso |
| `is_pilot` | `boolean` | São Mateus do Sul |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (name, state)`.

#### `courts`

Módulo dono: **`courts`**. Quadra de futsal ou society (sessões 2 e 8).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `owner_user_id` | `uuid` | **FK** → `users` (dono) |
| `city_id` | `uuid` | **FK** → `cities` |
| `name` | `text` | moderado |
| `court_type` | `court_type` | `futsal` ou `society` |
| `surface` | `text` | piso (lista por tipo no protótipo) |
| `description` | `text` | moderada |
| `address` | `text` | `NOT NULL` |
| `neighborhood` | `text` | `NOT NULL` |
| `location` | `geography(Point,4326)` | `NOT NULL`; ponto no mapa |
| `hourly_price_cents` | `bigint` | `CHECK (> 0)`; preço de 1h30 e 2h é proporcional |
| `opens_at` | `time` | abertura |
| `closes_at` | `time` | fechamento; `24:00` é válido em `time` |
| `amenities` | `text[]` | estrutura: vestiário, estacionamento… |
| `whatsapp` | `text` |  |
| `payee_id` | `uuid` | **FK** → `payees`; obrigatório para publicar |
| `status` | `profile_status` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (closes_at > opens_at)`; `CHECK (status <> 'published' OR payee_id IS NOT NULL)`.

Índices: `GIST (location)` para mapa e distância do check-in; `(city_id, court_type, status)`.

#### `court_photos`

Módulo dono: **`courts`**. Até 6 fotos por quadra, já reduzidas no aparelho.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `court_id` | `uuid` | **FK** → `courts` `ON DELETE CASCADE` |
| `position` | `smallint` | `CHECK (position BETWEEN 1 AND 6)` |
| `storage_key` | `text` | chave no armazenamento de objetos (fornecedor a definir) |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (court_id, position)` (o limite de 6 sai daqui).

### Times e desafios

#### `teams`

Módulo dono: **`teams`**. Time; o capitão é o usuário dono (sessão 8). Bloqueio por pendência é derivado de `debts` abertas.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `captain_user_id` | `uuid` | **FK** → `users` |
| `city_id` | `uuid` | **FK** → `cities` |
| `name` | `text` | moderado |
| `initials` | `text` | sigla gerada do nome; `CHECK (char_length BETWEEN 1 AND 3)` |
| `level` | `skill_level` |  |
| `crest_color` | `text` | `CHECK (crest_color ~ '^#[0-9A-Fa-f]{6}$')` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Índices: `(city_id, level)` para ranking e rivais.

#### `challenges`

Módulo dono: **`matches`**. Desafio aberto (sessão 1). Pode existir sem reserva (quem aceita reserva e paga) ou apontar para um jogo com horário já pago ("horário já reservado").

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `team_id` | `uuid` | **FK** → `teams` (criador) |
| `city_id` | `uuid` | **FK** → `cities` (cópia para o filtro) |
| `court_id` | `uuid` | **FK** → `courts` |
| `starts_at` | `timestamptz` | dia e horário propostos |
| `duration_minutes` | `smallint` | `CHECK (IN (60, 90, 120))` |
| `note` | `text` | recado; moderado |
| `match_id` | `uuid` | **FK** → `matches`; preenchido quando o horário já está pago |
| `accepted_by_team_id` | `uuid` | **FK** → `teams` |
| `status` | `challenge_status` | `open`, `accepted`, `withdrawn`, `expired` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (status <> 'accepted' OR accepted_by_team_id IS NOT NULL)`; `CHECK (accepted_by_team_id IS DISTINCT FROM team_id)`.

Índices: parcial `(city_id, starts_at) WHERE status = 'open'`.

#### `player_openings`

Módulo dono: **`hiring`**. Vaga "time pedindo jogador" da visão Sou jogador (sessão 1). Ver Q15.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `team_id` | `uuid` | **FK** → `teams` |
| `match_id` | `uuid` | **FK** → `matches` |
| `position_label` | `text` | ex.: Goleiro, 2 jogadores de linha |
| `slots` | `smallint` | `CHECK (> 0)` |
| `status` | `opening_status` | `open`, `filled`, `closed` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

#### `opening_applications`

Módulo dono: **`hiring`**. Toque em "Quero jogar" numa vaga.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `opening_id` | `uuid` | **FK** → `player_openings` `ON DELETE CASCADE` |
| `rental_player_id` | `uuid` | **FK** → `rental_players` |
| `hire_id` | `uuid` | **FK** → `hires` quando o time aceita |
| `status` | `application_status` | `sent`, `accepted`, `rejected`, `withdrawn` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (opening_id, rental_player_id)`.

### Jogos e reservas

#### `bookings`

Módulo dono: **`matches`**. Ocupação do horário da quadra. Faixa e status ficam na mesma linha porque a exclusão parcial só enxerga a própria linha.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `court_id` | `uuid` | **FK** → `courts` `ON DELETE RESTRICT` |
| `slot` | `tstzrange` | `[início, fim)` |
| `status` | `booking_status` | `held`, `reserved`, `released` |
| `hold_expires_at` | `timestamptz` | só em `held` (Q6) |
| `released_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (NOT isempty(slot) AND lower_inc(slot) AND NOT upper_inc(slot))`; `CHECK (upper(slot) - lower(slot) IN (interval '60 minutes', interval '90 minutes', interval '120 minutes'))`; `CHECK ((status = 'held') = (hold_expires_at IS NOT NULL))`; `EXCLUDE USING gist (court_id WITH =, slot WITH &&) WHERE (status IN ('held', 'reserved'))`.

Índices: o índice GiST da exclusão serve à grade de horários.

#### `matches`

Módulo dono: **`matches`**. Jogo marcado entre dois times numa reserva (sessões 3 e 5).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `booking_id` | `uuid` | **FK** → `bookings`, `UNIQUE` (1:1) |
| `court_id` | `uuid` | **FK** → `courts` (cópia de leitura) |
| `city_id` | `uuid` | **FK** → `cities` |
| `home_team_id` | `uuid` | **FK** → `teams`, quem marcou |
| `away_team_id` | `uuid` | **FK** → `teams`; nulo enquanto for desafio aberto |
| `starts_at` | `timestamptz` | = `lower(bookings.slot)` |
| `ends_at` | `timestamptz` | = `upper(bookings.slot)`; fim automático e repasses |
| `split_mode` | `boolean` | divisão 50/50; sempre `true` com desafio aceito |
| `court_price_cents` | `bigint` | preço da duração, congelado na reserva |
| `status` | `match_status` | ver máquina de estados |
| `cancelled_by_team_id` | `uuid` | **FK** → `teams` |
| `cancelled_at` | `timestamptz` |  |
| `home_score` | `smallint` | informado pela quadra |
| `away_score` | `smallint` |  |
| `score_reported_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (away_team_id IS DISTINCT FROM home_team_id)`; `CHECK ((home_score IS NULL) = (score_reported_at IS NULL))`.

Índices: `(home_team_id, starts_at)`, `(away_team_id, starts_at)`; parcial `(ends_at) WHERE status = 'confirmed'` para o encerramento.

#### `team_invitations`

Módulo dono: **`matches`**. Convite para um time específico confirmar o jogo (sessão 5, Confirmação). Histórico: recusa vira desafio aberto e outro time pode entrar.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `match_id` | `uuid` | **FK** → `matches` |
| `team_id` | `uuid` | **FK** → `teams` convidado |
| `respond_by` | `timestamptz` | 2h, terminando no máximo 1h antes; gravado, não derivado |
| `status` | `team_invitation_status` | `pending`, `confirmed`, `declined`, `expired`, `cancelled` |
| `responded_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Índices: `UNIQUE (match_id) WHERE status = 'pending'`.

#### `check_ins`

Módulo dono: **`matches`**. Chegada de capitão, árbitro ou jogador contratado (sessão 5). Evidência para W.O. e contestação.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `match_id` | `uuid` | **FK** → `matches` |
| `team_id` | `uuid` | **FK** → `teams`, quando é o capitão |
| `hire_id` | `uuid` | **FK** → `hires`, quando é árbitro ou jogador |
| `user_id` | `uuid` | **FK** → `users` que fez (ou dono da quadra que confirmou) |
| `method` | `check_in_method` | `gps` ou `court_confirmed` |
| `location` | `geography(Point,4326)` | só em `gps` |
| `distance_m` | `integer` | `ST_Distance(location, courts.location)` no momento |
| `checked_in_at` | `timestamptz` | entre início −1h e início +15min |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (num_nonnulls(team_id, hire_id) = 1)`; `CHECK ((method = 'gps') = (location IS NOT NULL))`; `UNIQUE (match_id, team_id)`, `UNIQUE (hire_id)`.

### Contratações

#### `hires`

Módulo dono: **`hiring`**. Contratação de árbitro ou jogador para um jogo (sessão 4). Prazos gravados porque a regra de "metade do tempo até o limite" não é derivável depois.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `match_id` | `uuid` | **FK** → `matches` |
| `hiring_team_id` | `uuid` | **FK** → `teams` |
| `kind` | `hire_kind` | `referee` ou `player` |
| `referee_id` | `uuid` | **FK** → `referees` |
| `rental_player_id` | `uuid` | **FK** → `rental_players` |
| `role_label` | `text` | posição ou função pedida |
| `referee_slot` | `smallint` | 1 ou 2: limite de 2 árbitros por jogo |
| `price_cents` | `bigint` | preço cheio do prestador, congelado no convite |
| `free_pass` | `boolean` | aceite já confirma, sem cobrança |
| `status` | `hire_status` | ver máquina de estados |
| `respond_by` | `timestamptz` | prazo do aceite |
| `pay_by` | `timestamptz` | prazo do pagamento, gravado no aceite |
| `withdraw_until` | `timestamptz` | início −2h |
| `invited_at` | `timestamptz` |  |
| `accepted_at` | `timestamptz` |  |
| `closed_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK ((kind = 'referee') = (referee_id IS NOT NULL) AND (kind = 'player') = (rental_player_id IS NOT NULL))`; `CHECK ((kind = 'referee') = (referee_slot IS NOT NULL))`, `CHECK (referee_slot IN (1, 2))`; `UNIQUE (match_id, referee_slot) WHERE status IN ('invited', 'awaiting_payment', 'confirmed')`; `UNIQUE (match_id, referee_id) WHERE status IN (…ativos)`, idem `rental_player_id`.

Índices: parcial `(respond_by) WHERE status = 'invited'` e `(pay_by) WHERE status = 'awaiting_payment'`.

#### `hire_shares`

Módulo dono: **`hiring`**. Parte de cada time no valor da contratação: 1 linha (jogador, ou árbitro sem 50/50) ou 2 (árbitro com 50/50). Permite "pagar por ele".

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `hire_id` | `uuid` | **FK** → `hires` `ON DELETE RESTRICT` |
| `team_id` | `uuid` | **FK** → `teams` responsável pela parte |
| `paid_by_team_id` | `uuid` | **FK** → `teams`; difere de `team_id` quando o contratante paga pelo adversário |
| `share_cents` | `bigint` | parte do preço do prestador, sem taxa |
| `pay_by` | `timestamptz` | 2h após o aceite, até 1h antes |
| `status` | `hire_share_status` | `pending`, `paid`, `covered`, `waived`, `expired` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (hire_id, team_id)`.

### Pagamentos (Asaas)

#### `payees`

Módulo dono: **`payments`**. Recebedor = subconta Asaas em nome próprio (AD-007). Dados bancários e documentos ficam no Asaas; aqui só ids e resumo mascarado (LGPD). Grão (por pessoa ou por perfil) é a Q9.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users` |
| `person_type` | `person_type` | `individual` ou `company` |
| `document_last4` | `text` | últimos dígitos do CPF/CNPJ |
| `asaas_account_id` | `text` | `UNIQUE`; `id` de `POST /v3/accounts` |
| `asaas_wallet_id` | `text` | `UNIQUE`; `walletId`, usado no `split` |
| `api_key_secret_ref` | `text` | referência ao cofre, nunca a chave (Q11) |
| `kyc_status` | `payee_kyc_status` | de `ACCOUNT_STATUS_GENERAL_APPROVAL_*` |
| `payout_summary` | `text` | resumo mascarado, ex.: "Banco do Brasil ag. 1234 conta …89-0" |
| `escrow_days_to_expire` | `smallint` | cópia do `daysToExpire` configurado na subconta |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

#### `payer_customers`

Módulo dono: **`payments`**. Cliente pagador no Asaas (`POST /v3/customers` exige `name` e `cpfCnpj`).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users`, `UNIQUE` |
| `asaas_customer_id` | `text` | `UNIQUE`; `cus_…` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

#### `charges`

Módulo dono: **`payments`**. Cobrança nossa: o que um usuário paga num checkout (parte da quadra, parte de contratação, pendência). `id` vai como `externalReference` no Asaas.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `payer_user_id` | `uuid` | **FK** → `users` |
| `payer_team_id` | `uuid` | **FK** → `teams`, quando paga em nome do time |
| `purpose` | `charge_purpose` | `court_share`, `hire_share`, `debt_payment` |
| `billing_type` | `billing_type` | `pix` ou `credit_card` |
| `installment_count` | `smallint` | 1 a 6; `CHECK (billing_type = 'credit_card' OR installment_count = 1)` |
| `total_cents` | `bigint` | = soma de `charge_items` |
| `status` | `charge_status` | ver mapeamento Asaas |
| `external_reference` | `text` | `UNIQUE`; = `id::text` |
| `asaas_installment_id` | `text` | `UNIQUE`; só parcelado |
| `due_at` | `timestamptz` | prazo de pagamento do nosso lado |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (installment_count BETWEEN 1 AND 6)`; `CHECK (total_cents > 0)`.

Índices: parcial `(due_at) WHERE status = 'pending'`.

#### `charge_items`

Módulo dono: **`payments`**. Linhas do valor mostradas na tela de pagamento. Cada linha diz de qual prestador é e qual taxa foi aplicada, para que as duas saídas das regras em conflito caibam sem mudar o esquema.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `charge_id` | `uuid` | **FK** → `charges` `ON DELETE RESTRICT` |
| `kind` | `charge_item_kind` | `court_share`, `referee_fee`, `player_fee`, `service_fee`, `payment_tariff`, `installment_surcharge`, `referee_deposit`, `debt_principal` |
| `match_id` | `uuid` | **FK** → `matches` |
| `hire_share_id` | `uuid` | **FK** → `hire_shares` |
| `debt_id` | `uuid` | **FK** → `debts` |
| `payee_id` | `uuid` | **FK** → `payees`; nulo para taxa, tarifa e acréscimo |
| `amount_cents` | `bigint` | `CHECK (>= 0)` |
| `rate_bp` | `integer` | taxa aplicada em pontos-base (500 = 5%), quando houver |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (num_nonnulls(match_id, hire_share_id, debt_id) >= 1)`; `CHECK (kind NOT IN ('court_share', 'referee_fee', 'player_fee', 'referee_deposit', 'debt_principal') OR payee_id IS NOT NULL)`: dinheiro de terceiro nunca fica sem recebedor (AD-007).

#### `gateway_payments`

Módulo dono: **`payments`**. Cobrança no Asaas (`pay_…`). Pix e cartão à vista: 1 por `charge`; cartão parcelado: 1 por parcela.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `charge_id` | `uuid` | **FK** → `charges` |
| `asaas_payment_id` | `text` | `UNIQUE` |
| `installment_number` | `smallint` | `installmentNumber` |
| `value_cents` | `bigint` | `value` |
| `net_value_cents` | `bigint` | `netValue` (após a tarifa) |
| `asaas_status` | `text` | status bruto do Asaas |
| `estimated_credit_date` | `date` | `estimatedCreditDate` |
| `credit_date` | `date` | `creditDate`: dinheiro disponível ≠ escrow liberado |
| `last_synced_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (charge_id, installment_number)`.

#### `charge_splits`

Módulo dono: **`payments`**. Split por recebedor em cada `gateway_payment`, sempre por valor fixo (`fixedValue`/`totalFixedValue`), nunca percentual.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `gateway_payment_id` | `uuid` | **FK** → `gateway_payments` |
| `payee_id` | `uuid` | **FK** → `payees` |
| `amount_cents` | `bigint` | `fixedValue` |
| `asaas_split_id` | `text` | `UNIQUE`; `split[].id`, usado em `splitRefunds` |
| `external_reference` | `text` | `UNIQUE`; `split[].externalReference` |
| `status` | `split_status` | espelha `split[].status` |
| `cancellation_reason` | `text` | `split[].cancellationReason` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (gateway_payment_id, payee_id)`.

#### `refunds`

Módulo dono: **`payments`**. Estorno pedido por nós. O Asaas não devolve id de estorno: o nosso `id` é a chave e a conciliação usa cobrança, valor, `dateCreated` e `refundedSplits[].id`.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `charge_id` | `uuid` | **FK** → `charges` |
| `gateway_payment_id` | `uuid` | **FK** → `gateway_payments` |
| `settlement_id` | `uuid` | **FK** → `settlements` que mandou estornar |
| `reason` | `refund_reason` | `match_cancelled`, `walkover`, `hire_withdrawn`, `hire_no_show`, `hire_cancelled`, `deposit_return`, `other` |
| `amount_cents` | `bigint` | `value` |
| `status` | `refund_status` | `requested`, `sent`, `pending`, `awaiting_authorization`, `done`, `cancelled`, `failed` |
| `idempotency_key` | `text` | `UNIQUE`; ex.: `settlement:<id>:payment:<id>` |
| `asaas_date_created` | `timestamptz` | `refunds[].dateCreated`, preenchido na conciliação |
| `end_to_end_id` | `text` | Pix |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (amount_cents > 0)`.

#### `refund_splits`

Módulo dono: **`payments`**. Quanto sai de cada split (`splitRefunds[{id, value}]`). O resto sai da conta-pai (taxa da plataforma).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `refund_id` | `uuid` | **FK** → `refunds` |
| `charge_split_id` | `uuid` | **FK** → `charge_splits` |
| `amount_cents` | `bigint` |  |
| `done` | `boolean` | `refundedSplits[].done` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (refund_id, charge_split_id)`.

#### `escrow_holds`

Módulo dono: **`payments`**. Garantia da Conta Escrow. Sem webhook de mudança: o worker consulta `GET /v3/payments/{id}/escrow`. Grão por cobrança ou por recebedor: a confirmar no sandbox.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `gateway_payment_id` | `uuid` | **FK** → `gateway_payments` |
| `payee_id` | `uuid` | **FK** → `payees`; nulo se o grão for a cobrança |
| `asaas_escrow_id` | `text` | `escrow.id`, usado em `POST /v3/escrow/{id}/finish` |
| `status` | `escrow_status` | `active`, `release_requested`, `released`, `expired`, `ended_by_refund`, `ended_other` |
| `release_due_at` | `timestamptz` | fim do horário reservado ou registro do W.O. |
| `expiration_date` | `date` | `escrow.expirationDate` |
| `finished_at` | `date` | `escrow.finishDate` |
| `finish_reason` | `text` | `escrow.finishReason` |
| `last_polled_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE NULLS NOT DISTINCT (gateway_payment_id, payee_id)`.

Índices: parcial `(release_due_at) WHERE status = 'active'`.

#### `asaas_webhook_events`

Módulo dono: **`payments`**. Eventos recebidos do Asaas. Entrega é "at least once": o `id` do evento é a chave única; responde 200 depois de gravar e processa no worker.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `asaas_event_id` | `text` | `UNIQUE`; `id` do evento (`evt_…`) |
| `event_type` | `text` | `event`, ex.: `PAYMENT_RECEIVED` |
| `asaas_payment_id` | `text` | para achar a `gateway_payment` (sem FK: o evento pode chegar antes) |
| `payload` | `jsonb` | corpo inteiro |
| `status` | `webhook_event_status` | `pending`, `processing`, `done`, `failed`, `ignored` |
| `attempts` | `smallint` |  |
| `received_at` | `timestamptz` |  |
| `processed_at` | `timestamptz` |  |
| `last_error` | `text` |  |

Índices: parcial `(received_at) WHERE status = 'pending'`.

### Acerto e contabilidade

#### `settlements`

Módulo dono: **`settlement`**. Acerto de um desfecho: fim normal, cancelamento, W.O., desistência. Congela as regras aplicadas (versão e taxas) para que a mudança das regras em conflito não reescreva acertos antigos.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `match_id` | `uuid` | **FK** → `matches` |
| `hire_id` | `uuid` | **FK** → `hires`, quando o desfecho é só de uma contratação |
| `kind` | `settlement_kind` | `match_finished`, `match_cancelled`, `walkover_one_team`, `walkover_both_teams`, `hire_withdrawn`, `hire_no_show`, `hire_cancelled` |
| `policy_version` | `text` | versão da tabela de regras usada |
| `rates` | `jsonb` | taxas usadas: 5%, 10%, 30%… |
| `at_fault_team_id` | `uuid` | **FK** → `teams`, quem cancelou ou faltou |
| `status` | `settlement_status` | `computed`, `executing`, `done`, `failed` |
| `computed_at` | `timestamptz` |  |

Restrições: `UNIQUE (match_id, kind) WHERE hire_id IS NULL`; `UNIQUE (hire_id, kind)`.

#### `settlement_lines`

Módulo dono: **`settlement`**. Uma linha por movimento do acerto: liberar para recebedor, estornar para pagador, reter taxa, virar pendência. É a tabela do acerto mostrada no cartão do W.O.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `settlement_id` | `uuid` | **FK** → `settlements` |
| `action` | `settlement_action` | `release_to_payee`, `refund_to_payer`, `keep_platform_fee`, `absorb_by_platform`, `create_debt` |
| `charge_split_id` | `uuid` | **FK** → `charge_splits` de onde sai o valor |
| `payee_id` | `uuid` | **FK** → `payees` beneficiado |
| `team_id` | `uuid` | **FK** → `teams` reembolsado ou devedor |
| `amount_cents` | `bigint` | `CHECK (> 0)` |
| `refund_id` | `uuid` | **FK** → `refunds` gerado |
| `debt_id` | `uuid` | **FK** → `debts` gerada |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

#### `debts`

Módulo dono: **`settlement`**. Pendência ("saldo devedor"): o faltoso que não pagou o bastante, e as alternativas "nova cobrança" das regras em conflito. Time com `debts` em aberto não marca, não aceita e não contrata.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `debtor_team_id` | `uuid` | **FK** → `teams` |
| `match_id` | `uuid` | **FK** → `matches` |
| `settlement_id` | `uuid` | **FK** → `settlements` de origem |
| `reason` | `debt_reason` | `walkover_shortfall`, `cancellation_other_team_fee`, `referee_share_replacement` |
| `amount_cents` | `bigint` | = soma de `debt_items` |
| `status` | `debt_status` | `open`, `charging`, `paid`, `written_off` |
| `settlement_charge_id` | `uuid` | **FK** → `charges` (`purpose = 'debt_payment'`) |
| `paid_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Índices: parcial `(debtor_team_id) WHERE status IN ('open', 'charging')` para o bloqueio.

#### `debt_items`

Módulo dono: **`settlement`**. Para quem vai cada parte da pendência (quadra, árbitro), para avisar "valor pendente" e "pendência quitada".

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `debt_id` | `uuid` | **FK** → `debts` |
| `payee_id` | `uuid` | **FK** → `payees` |
| `amount_cents` | `bigint` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (debt_id, payee_id)`.

#### `ledger_accounts`

Módulo dono: **`ledger`**. Conta do razão. Uma por papel e dono: pagador, recebedor retido, recebedor liberado, taxa da plataforma, tarifa do gateway, pendência a receber.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `kind` | `ledger_account_kind` | `payer`, `payee_escrow`, `payee_released`, `platform_fee`, `platform_absorbed`, `gateway_tariff`, `debt_receivable` |
| `payee_id` | `uuid` | **FK** → `payees` |
| `team_id` | `uuid` | **FK** → `teams` |
| `user_id` | `uuid` | **FK** → `users` (pagador) |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE NULLS NOT DISTINCT (kind, payee_id, team_id, user_id)`.

#### `ledger_transactions`

Módulo dono: **`ledger`**. Fato contábil (cobrança recebida, split, taxa, tarifa, estorno, liberação, pendência). Nunca editado: correção é uma transação de estorno contábil.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `kind` | `ledger_tx_kind` | `charge_received`, `refund`, `escrow_release`, `debt_created`, `debt_paid`, `reversal` |
| `idempotency_key` | `text` | `UNIQUE`: a mesma causa não lança duas vezes |
| `charge_id` | `uuid` | **FK** → `charges` |
| `refund_id` | `uuid` | **FK** → `refunds` |
| `settlement_id` | `uuid` | **FK** → `settlements` |
| `reverses_id` | `uuid` | **FK** → `ledger_transactions` |
| `occurred_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (reverses_id)`; trigger `BEFORE UPDATE OR DELETE` levanta erro.

#### `ledger_entries`

Módulo dono: **`ledger`**. Lançamento em partidas dobradas, em centavos com sinal. A soma por transação é zero.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `bigint` | **PK** `GENERATED ALWAYS AS IDENTITY` |
| `transaction_id` | `uuid` | **FK** → `ledger_transactions` `ON DELETE RESTRICT` |
| `account_id` | `uuid` | **FK** → `ledger_accounts` |
| `amount_cents` | `bigint` | `CHECK (<> 0)`; débito +, crédito − |
| `charge_item_id` | `uuid` | **FK** → `charge_items` explicado pelo lançamento |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: constraint trigger `DEFERRABLE INITIALLY DEFERRED`: `sum(amount_cents) = 0` por `transaction_id`; trigger `BEFORE UPDATE OR DELETE` levanta erro; `REVOKE UPDATE, DELETE` do papel da aplicação.

Índices: `(account_id, created_at)` para saldo e extrato.

### Reputação e comunidade

#### `team_reviews`

Módulo dono: **`reputation`**. Avaliação de um time pelo adversário (sessão 6): 4 notas obrigatórias, destaques e comentário.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `match_id` | `uuid` | **FK** → `matches` |
| `reviewer_team_id` | `uuid` | **FK** → `teams` |
| `reviewed_team_id` | `uuid` | **FK** → `teams` |
| `punctuality` | `smallint` | `CHECK (BETWEEN 1 AND 5)` |
| `fair_play` | `smallint` | idem |
| `commitment` | `smallint` | idem |
| `organization` | `smallint` | idem |
| `highlights` | `text[]` | destaques opcionais |
| `comment` | `text` | moderado |
| `visible_at` | `timestamptz` | preenchido nas duas linhas quando a segunda avaliação chega |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (match_id, reviewer_team_id)`; `CHECK (reviewer_team_id <> reviewed_team_id)`.

Índices: `(reviewed_team_id) WHERE visible_at IS NOT NULL` para o ranking.

#### `provider_reviews`

Módulo dono: **`reputation`**. Nota de quadra, árbitro e jogador (o briefing mostra nota e avaliações, mas não diz quem avalia: Q14).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `match_id` | `uuid` | **FK** → `matches` |
| `reviewer_team_id` | `uuid` | **FK** → `teams` |
| `court_id` | `uuid` | **FK** → `courts` |
| `referee_id` | `uuid` | **FK** → `referees` |
| `rental_player_id` | `uuid` | **FK** → `rental_players` |
| `rating` | `smallint` | `CHECK (BETWEEN 1 AND 5)` |
| `comment` | `text` | moderado |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (num_nonnulls(court_id, referee_id, rental_player_id) = 1)`; `UNIQUE NULLS NOT DISTINCT (match_id, reviewer_team_id, court_id, referee_id, rental_player_id)`.

#### `skill_endorsements`

Módulo dono: **`reputation`**. Time que confirmou uma habilidade ou característica depois do jogo (a contagem do perfil).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `profile_skill_id` | `uuid` | **FK** → `profile_skills` `ON DELETE CASCADE` |
| `team_id` | `uuid` | **FK** → `teams` |
| `match_id` | `uuid` | **FK** → `matches` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `UNIQUE (profile_skill_id, team_id, match_id)`.

#### `walkovers`

Módulo dono: **`reputation`**. W.O. registrado (sessão 5). Fica no histórico e entra no ranking como 10 notas 1 em pontualidade e comprometimento, calculado a partir das linhas, sem criar avaliações falsas.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `match_id` | `uuid` | **FK** → `matches` |
| `team_id` | `uuid` | **FK** → `teams` faltoso |
| `hire_id` | `uuid` | **FK** → `hires` faltoso |
| `kind` | `walkover_kind` | `team_no_show`, `referee_no_show`, `player_no_show` |
| `rating_weight` | `smallint` | `DEFAULT 10`, congelado |
| `recorded_at` | `timestamptz` | fechamento do check-in |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (num_nonnulls(team_id, hire_id) = 1)`; `UNIQUE (match_id, team_id)`, `UNIQUE (hire_id)`.

#### `suspensions`

Módulo dono: **`reputation`**. Cartão vermelho: 7 dias sem convocações para árbitro ou jogador que deu W.O.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `referee_id` | `uuid` | **FK** → `referees` |
| `rental_player_id` | `uuid` | **FK** → `rental_players` |
| `walkover_id` | `uuid` | **FK** → `walkovers` |
| `period` | `tstzrange` | `[W.O., W.O. + 7 dias)` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (num_nonnulls(referee_id, rental_player_id) = 1)`; `UNIQUE (walkover_id)`.

Índices: `GIST (period)` para "suspenso agora".

#### `blocked_terms`

Módulo dono: **`community`**. Lista do filtro de palavras (palavrões e termos racistas).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `term` | `text` | `UNIQUE`, normalizado sem acento e minúsculo |
| `category` | `blocked_term_category` | `profanity`, `racism`, `other` |
| `active` | `boolean` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

#### `moderation_blocks`

Módulo dono: **`community`**. Registro de texto bloqueado (não publicado). Guarda só o motivo e um resumo, não o texto inteiro.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users` |
| `field` | `moderated_field` | `review_comment`, `team_name`, `challenge_note`, `skill`, `description`, `name` |
| `reason` | `text` | mostrado à pessoa |
| `engine` | `text` | `word_filter`; `ai` fica desligado |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

#### `profile_reports`

Módulo dono: **`community`**. Denúncia de perfil (time, jogador, árbitro). Só a equipe do app vê.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `reporter_user_id` | `uuid` | **FK** → `users` |
| `team_id` | `uuid` | **FK** → `teams` |
| `referee_id` | `uuid` | **FK** → `referees` |
| `rental_player_id` | `uuid` | **FK** → `rental_players` |
| `reason` | `report_reason` | `offense`, `racism_or_prejudice`, `violence`, `did_not_pay`, `fake_profile`, `other` |
| `details` | `text` | relato opcional |
| `status` | `report_status` | `open`, `in_review`, `actioned`, `dismissed` |
| `handled_by_user_id` | `uuid` | **FK** → `users` admin |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Restrições: `CHECK (num_nonnulls(team_id, referee_id, rental_player_id) = 1)`.

Índices: parcial `(created_at) WHERE status IN ('open', 'in_review')`.

#### `suggestions`

Módulo dono: **`community`**. Sugestão do botão flutuante, lida pela equipe do app.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users` |
| `body` | `text` |  |
| `screen` | `text` | aba de onde veio |
| `status` | `suggestion_status` | `new`, `read`, `done` |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

### Notificações e jobs

#### `notifications`

Módulo dono: **`notifications`**. Aviso do sino e do push (sessão 9), com destino de navegação e marca de lida.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users` destinatário |
| `kind` | `notification_kind` | um valor por evento da sessão 9 |
| `body` | `text` | texto pronto, com valor e horário limite |
| `target_route` | `text` | ex.: `jogos`, `perfil:arbitro` |
| `target_params` | `jsonb` | ids do assunto |
| `dedupe_key` | `text` | `UNIQUE`: o mesmo fato não notifica duas vezes |
| `read_at` | `timestamptz` | abrir o sino marca todas |
| `push_sent_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Índices: parcial `(user_id, created_at DESC) WHERE read_at IS NULL` para o contador.

#### `push_devices`

Módulo dono: **`notifications`**. Token de push do aparelho (Capacitor).

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `uuid` | **PK** `DEFAULT gen_random_uuid()` |
| `user_id` | `uuid` | **FK** → `users` `ON DELETE CASCADE` |
| `platform` | `device_platform` | `android`, `ios`, `web` |
| `token` | `text` | `UNIQUE` |
| `last_seen_at` | `timestamptz` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

#### `jobs`

Módulo dono: **`worker`**. Fila do worker no próprio Postgres, se não for usado o `pg-boss` (Q23). Prazos de 2h, check-in, encerramento, polling do escrow.

| Coluna | Tipo | Regra |
| :--- | :--- | :--- |
| `id` | `bigint` | **PK** `GENERATED ALWAYS AS IDENTITY` |
| `kind` | `text` | ex.: `hire.respond_deadline`, `match.finish`, `escrow.poll` |
| `run_at` | `timestamptz` | quando rodar |
| `payload` | `jsonb` |  |
| `dedupe_key` | `text` | `UNIQUE`; ex.: `match.finish:<id>` |
| `status` | `job_status` | `queued`, `running`, `done`, `failed`, `cancelled` |
| `attempts` | `smallint` |  |
| `locked_until` | `timestamptz` | `SELECT … FOR UPDATE SKIP LOCKED` |
| `last_error` | `text` |  |
| `created_at` | `timestamptz` | `NOT NULL DEFAULT now()` |

Índices: parcial `(run_at) WHERE status = 'queued'`.

**Views (sem tabela):** `team_ratings` (média por critério, nota geral, total de avaliações, peso dos W.O.), `team_badges` (Pontual, Jogo limpo, Comprometido, Jogo pegado), `court_slots` (grade de Livre, Ocupado ou Passou de 7 dias, gerada de `courts.opens_at`/`closes_at` menos `bookings`) e `payee_balances` (saldo retido e liberado de cada recebedor, a partir de `ledger_entries`). O ranking pode virar `MATERIALIZED VIEW` quando o volume pedir.

## 4. Relacionamentos

| Pai | Filho | Coluna (FK) | Cardinalidade | `ON DELETE` |
| :--- | :--- | :--- | :--- | :--- |
| `users` | `referees` | `user_id` | 1:0..1 | `RESTRICT` |
| `users` | `rental_players` | `user_id` | 1:0..1 | `RESTRICT` |
| `users` | `teams` | `captain_user_id` | 1:N | `RESTRICT` |
| `users` | `courts` | `owner_user_id` | 1:N | `RESTRICT` |
| `users` | `payees` | `user_id` | 1:N | `RESTRICT` |
| `users` | `payer_customers` | `user_id` | 1:0..1 | `RESTRICT` |
| `users` | `charges` | `payer_user_id` | 1:N | `RESTRICT` |
| `users` | `notifications` | `user_id` | 1:N | `RESTRICT` |
| `users` | `push_devices` | `user_id` | 1:N | `CASCADE` |
| `users` | `profile_reports` | `reporter_user_id` | 1:N | `RESTRICT` |
| `users` | `suggestions` | `user_id` | 1:N | `RESTRICT` |
| `users` | `moderation_blocks` | `user_id` | 1:N | `RESTRICT` |
| `cities` | `courts` | `city_id` | 1:N | `RESTRICT` |
| `cities` | `teams` | `city_id` | 1:N | `RESTRICT` |
| `cities` | `referees` | `city_id` | 1:N | `RESTRICT` |
| `cities` | `rental_players` | `city_id` | 1:N | `RESTRICT` |
| `courts` | `court_photos` | `court_id` | 1:0..6 | `CASCADE` |
| `courts` | `bookings` | `court_id` | 1:N | `RESTRICT` |
| `courts` | `challenges` | `court_id` | 1:N | `RESTRICT` |
| `referees` | `profile_skills` | `referee_id` | 1:N | `CASCADE` |
| `rental_players` | `profile_skills` | `rental_player_id` | 1:N | `CASCADE` |
| `payees` | `courts` | `payee_id` | 1:N | `RESTRICT` |
| `payees` | `referees` | `payee_id` | 1:0..1 | `RESTRICT` |
| `payees` | `rental_players` | `payee_id` | 1:0..1 | `RESTRICT` |
| `teams` | `challenges` | `team_id` | 1:N | `RESTRICT` |
| `matches` | `challenges` | `match_id` | 1:0..1 | `RESTRICT` |
| `teams` | `player_openings` | `team_id` | 1:N | `RESTRICT` |
| `matches` | `player_openings` | `match_id` | 1:N | `RESTRICT` |
| `player_openings` | `opening_applications` | `opening_id` | 1:N | `CASCADE` |
| `rental_players` | `opening_applications` | `rental_player_id` | 1:N | `RESTRICT` |
| `bookings` | `matches` | `booking_id` | 1:1 | `RESTRICT` |
| `teams` | `matches` | `home_team_id / away_team_id` | 1:N (×2) | `RESTRICT` |
| `matches` | `team_invitations` | `match_id` | 1:N | `RESTRICT` |
| `teams` | `team_invitations` | `team_id` | 1:N | `RESTRICT` |
| `matches` | `check_ins` | `match_id` | 1:N | `RESTRICT` |
| `teams` | `check_ins` | `team_id` | 1:N | `RESTRICT` |
| `hires` | `check_ins` | `hire_id` | 1:0..1 | `RESTRICT` |
| `matches` | `hires` | `match_id` | 1:N | `RESTRICT` |
| `teams` | `hires` | `hiring_team_id` | 1:N | `RESTRICT` |
| `referees` | `hires` | `referee_id` | 1:N | `RESTRICT` |
| `rental_players` | `hires` | `rental_player_id` | 1:N | `RESTRICT` |
| `hires` | `hire_shares` | `hire_id` | 1:1..2 | `RESTRICT` |
| `teams` | `hire_shares` | `team_id` | 1:N | `RESTRICT` |
| `teams` | `charges` | `payer_team_id` | 1:N | `RESTRICT` |
| `charges` | `charge_items` | `charge_id` | 1:N | `RESTRICT` |
| `matches` | `charge_items` | `match_id` | 1:N | `RESTRICT` |
| `hire_shares` | `charge_items` | `hire_share_id` | 1:N | `RESTRICT` |
| `payees` | `charge_items` | `payee_id` | 1:N | `RESTRICT` |
| `charges` | `gateway_payments` | `charge_id` | 1:1..6 | `RESTRICT` |
| `gateway_payments` | `charge_splits` | `gateway_payment_id` | 1:N | `RESTRICT` |
| `payees` | `charge_splits` | `payee_id` | 1:N | `RESTRICT` |
| `charges` | `refunds` | `charge_id` | 1:N | `RESTRICT` |
| `gateway_payments` | `refunds` | `gateway_payment_id` | 1:N | `RESTRICT` |
| `refunds` | `refund_splits` | `refund_id` | 1:N | `RESTRICT` |
| `charge_splits` | `refund_splits` | `charge_split_id` | 1:N | `RESTRICT` |
| `gateway_payments` | `escrow_holds` | `gateway_payment_id` | 1:N | `RESTRICT` |
| `payees` | `escrow_holds` | `payee_id` | 1:N | `RESTRICT` |
| `matches` | `settlements` | `match_id` | 1:N | `RESTRICT` |
| `settlements` | `settlement_lines` | `settlement_id` | 1:N | `RESTRICT` |
| `charge_splits` | `settlement_lines` | `charge_split_id` | 1:N | `RESTRICT` |
| `payees` | `settlement_lines` | `payee_id` | 1:N | `RESTRICT` |
| `teams` | `settlement_lines` | `team_id` | 1:N | `RESTRICT` |
| `settlements` | `debts` | `settlement_id` | 1:N | `RESTRICT` |
| `teams` | `debts` | `debtor_team_id` | 1:N | `RESTRICT` |
| `matches` | `debts` | `match_id` | 1:N | `RESTRICT` |
| `charges` | `debts` | `settlement_charge_id` | 1:0..1 | `RESTRICT` |
| `debts` | `debt_items` | `debt_id` | 1:N | `RESTRICT` |
| `payees` | `debt_items` | `payee_id` | 1:N | `RESTRICT` |
| `payees` | `ledger_accounts` | `payee_id` | 1:N | `RESTRICT` |
| `teams` | `ledger_accounts` | `team_id` | 1:N | `RESTRICT` |
| `ledger_transactions` | `ledger_entries` | `transaction_id` | 1:N (≥2) | `RESTRICT` |
| `ledger_accounts` | `ledger_entries` | `account_id` | 1:N | `RESTRICT` |
| `charges` | `ledger_transactions` | `charge_id` | 1:N | `RESTRICT` |
| `refunds` | `ledger_transactions` | `refund_id` | 1:N | `RESTRICT` |
| `settlements` | `ledger_transactions` | `settlement_id` | 1:N | `RESTRICT` |
| `matches` | `team_reviews` | `match_id` | 1:0..2 | `RESTRICT` |
| `teams` | `team_reviews` | `reviewer_team_id / reviewed_team_id` | 1:N (×2) | `RESTRICT` |
| `matches` | `provider_reviews` | `match_id` | 1:N | `RESTRICT` |
| `teams` | `provider_reviews` | `reviewer_team_id` | 1:N | `RESTRICT` |
| `courts` | `provider_reviews` | `court_id` | 1:N | `RESTRICT` |
| `referees` | `provider_reviews` | `referee_id` | 1:N | `RESTRICT` |
| `rental_players` | `provider_reviews` | `rental_player_id` | 1:N | `RESTRICT` |
| `profile_skills` | `skill_endorsements` | `profile_skill_id` | 1:N | `CASCADE` |
| `teams` | `skill_endorsements` | `team_id` | 1:N | `RESTRICT` |
| `matches` | `skill_endorsements` | `match_id` | 1:N | `RESTRICT` |
| `matches` | `walkovers` | `match_id` | 1:N | `RESTRICT` |
| `teams` | `walkovers` | `team_id` | 1:N | `RESTRICT` |
| `hires` | `walkovers` | `hire_id` | 1:0..1 | `RESTRICT` |
| `walkovers` | `suspensions` | `walkover_id` | 1:0..1 | `RESTRICT` |
| `referees` | `suspensions` | `referee_id` | 1:N | `RESTRICT` |
| `rental_players` | `suspensions` | `rental_player_id` | 1:N | `RESTRICT` |
| `teams` | `profile_reports` | `team_id` | 1:N | `RESTRICT` |
| `referees` | `profile_reports` | `referee_id` | 1:N | `RESTRICT` |
| `rental_players` | `profile_reports` | `rental_player_id` | 1:N | `RESTRICT` |

Os N:N são resolvidos por tabelas com dados próprios: time × time num jogo (`matches`, com `home_team_id` e `away_team_id`), time × prestador (`hires`, que tem prazos e estado), time × habilidade (`skill_endorsements`), cobrança × recebedor (`charge_splits`) e estorno × split (`refund_splits`).

## 5. Máquinas de estado

Cada estado é um valor de `ENUM`. A transição só acontece pelo service do módulo dono, numa transação com `UPDATE … WHERE status = '<de>'` (transição atômica: se 0 linhas mudarem, outro processo chegou antes).

### 5.1 Jogo (`matches.status`) e reserva (`bookings.status`). Briefing §2, §3, §5 e §11

| De | Para | Quando |
| :--- | :--- | :--- |
| (novo) | `awaiting_payment` | o time escolhe quadra, horário, duração, adversário ou desafio aberto, e 50/50. Reserva `held` (ver Q6) |
| `awaiting_payment` | `awaiting_opponent` | o time que marcou pagou a sua parte (ou tudo, sem 50/50) e convidou um time. Reserva `reserved`: "o horário trava quando o primeiro time paga a quadra" |
| `awaiting_payment` | `open_challenge` | pagou sem adversário: desafio aberto com "horário já reservado" |
| `awaiting_payment` | `abandoned` | o pagamento não chegou até `hold_expires_at`. Reserva `released` |
| `awaiting_opponent` | `confirmed` | o convidado pagou a parte (50/50) ou tocou em Confirmar (sem 50/50) dentro de `team_invitations.respond_by` |
| `awaiting_opponent` | `open_challenge` | o convidado recusou ou o prazo passou ("vira desafio aberto") |
| `open_challenge` | `confirmed` | outro time aceitou o desafio e pagou a parte (50/50 sempre ligado) |
| `open_challenge` | `cancelled` | o criador cancelou até 2h antes, ou ninguém aceitou até o início (Q8) |
| `awaiting_opponent`, `confirmed` | `cancelled` | um time cancelou até 2h antes. Depois disso, o cancelamento é bloqueado. Reserva `released` |
| `confirmed` | `walkover` | o check-in fechou (início + 15 min) com pelo menos um time sem check-in |
| `confirmed` | `finished` | fim do horário reservado (`ends_at`), com os dois times presentes. O placar chega depois, pela quadra |

`finished`, `walkover`, `cancelled` e `abandoned` são finais. A reserva de jogo `finished` ou `walkover` continua `reserved`, que é o histórico de ocupação. Reserva: `held` → `reserved` | `released`; `reserved` → `released`.

### 5.2 Convite de time (`team_invitations.status`). Briefing §5 Confirmação e §10 Prazos

`pending` → `confirmed` (pagou ou confirmou) | `declined` | `expired` (`respond_by`: 2h, terminando no máximo 1h antes) | `cancelled` (o jogo foi cancelado antes). Só um convite `pending` por jogo.

### 5.3 Desafio (`challenges.status`). Briefing §1

`open` → `accepted` (sai da lista para todos) | `withdrawn` (Retirar) | `expired` (a data passou; desafios vencidos somem).

### 5.4 Contratação (`hires.status`). Briefing §4, §7 e §10

| De | Para | Quando |
| :--- | :--- | :--- |
| (novo) | `invited` | convite enviado até 1h30 antes. Nada é cobrado. `respond_by` = mín(convite + 2h, início − 1h), ou metade do tempo até o limite se faltar pouco |
| `invited` | `awaiting_payment` | aceite de prestador cobrado. `pay_by` = mín(aceite + 2h, início − 1h). Cria `hire_shares` |
| `invited` | `confirmed` | aceite de jogador de passe livre: "o aceite já confirma" |
| `invited` | `declined` / `expired` / `cancelled` | recusou / passou de `respond_by` ("convite estornado") / o jogo foi cancelado |
| `awaiting_payment` | `confirmed` | todas as `hire_shares` estão `paid` ou `covered` |
| `awaiting_payment` | `payment_expired` | passou de `pay_by`; "contratação cancelada", os dois lados são avisados |
| `awaiting_payment` | `cancelled` | o adversário não pagou e o contratante escolheu cancelar o árbitro, ou o jogo foi cancelado |
| `confirmed` | `withdrawn` | o prestador desistiu até `withdraw_until` (início − 2h). O valor volta |
| `confirmed` | `cancelled` | o jogo foi cancelado |
| `confirmed` | `no_show` | sem check-in no fechamento: W.O. do prestador, valor volta a quem pagou, cartão vermelho |
| `confirmed` | `settled` | fim do jogo, ou W.O. de um time: valor liberado ("Recebido") |

Painel do prestador (§7): Novo = `invited`; Aguardando pagamento = `awaiting_payment`; Confirmado e pago = `confirmed`; Recebido = `settled`; Encerrados = `declined`, `expired`, `payment_expired`, `cancelled`, `withdrawn` e `no_show`.

Parte de cada time (`hire_shares.status`): `pending` → `paid` | `covered` (o contratante pagou pelo adversário) | `waived` (contratação desfeita) | `expired`.

### 5.5 Cobrança (`charges.status`). Ver o mapeamento Asaas na §7

`pending` → `confirmed` (cartão autorizado, ainda não creditado) → `received`; `pending` → `received` (Pix); `pending` → `expired` | `cancelled` | `failed` (cartão recusado ou risco reprovado); `confirmed`/`received` → `partially_refunded` → `refunded`; `confirmed`/`received` → `chargeback`. Para o produto, uma cobrança "paga" é `confirmed` ou `received`.

### 5.6 Estorno (`refunds.status`)

`requested` (linha gravada **antes** da chamada ao Asaas, com `idempotency_key`) → `sent` (a chamada respondeu 200) → `pending` / `awaiting_authorization` (status do Asaas) → `done` | `cancelled`. Uma falha 4xx leva a `failed`, que exige revisão: o estorno não é reenviado sozinho.

### 5.7 Garantia do escrow (`escrow_holds.status`)

`active` → `release_requested` (o worker chamou `POST /v3/escrow/{id}/finish` em `release_due_at`) → `released`; `active` → `expired` (`finishReason = EXPIRED`: o worker falhou e o prazo da subconta venceu); `active` → `ended_by_refund` (`PAYMENT_REFUNDED`); `active` → `ended_other` (`CHARGEBACK`, `INSUFFICIENT_BALANCE`, `CUSTOMER_CONFIG_DISABLED`), que gera alerta. **Liberado não quer dizer disponível**: no cartão, o dinheiro só entra no saldo em `gateway_payments.credit_date` (D+32 por parcela).

### 5.8 Acerto (`settlements.status`) e pendência (`debts.status`)

Acerto: `computed` → `executing` → `done` | `failed` (se falhar, volta para `executing` na nova tentativa; cada linha é idempotente pela `idempotency_key` do estorno e da transação contábil).

Pendência: `open` → `charging` (cobrança criada) → `paid`; `charging` → `open` (a cobrança venceu); `open` → `written_off` (decisão do admin). Time com pendência `open` ou `charging` não marca jogo, não aceita desafio e não contrata.

### 5.9 Check-in (`check_ins`). Briefing §5

Não tem enum: a linha existe ou não. A janela vai de início − 1h a início + 15 min. `method = gps` exige `location` e grava `distance_m` (raio na Q17). `method = court_confirmed` é a confirmação do dono da quadra para quem está sem sinal ou GPS. No fechamento (job `checkin.close`), quem não tem linha vira `walkovers` e o jogo ou a contratação mudam de estado.

### 5.10 Denúncia (`profile_reports.status`). Briefing §6

`open` → `in_review` → `actioned` (o perfil pode ser suspenso ou escondido, `profile_status = hidden`) | `dismissed`.

## 6. O que o banco garante

### 6.1 Reserva sem sobreposição

Uma faixa `tstzrange` semiaberta e uma restrição de exclusão parcial na mesma tabela. O padrão `EXCLUDE USING gist (sala WITH =, faixa WITH &&)` com `btree_gist` é o exemplo da documentação do Postgres (Range Types, *Constraints on Ranges*). O predicado `WHERE` cria um índice parcial e só enxerga a própria linha. Por isso o status mora em `bookings`, e não em `matches`.

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TYPE booking_status AS ENUM ('held', 'reserved', 'released');

CREATE TABLE bookings (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  court_id        uuid NOT NULL REFERENCES courts(id) ON DELETE RESTRICT,
  slot            tstzrange NOT NULL,
  status          booking_status NOT NULL,
  hold_expires_at timestamptz,
  released_at     timestamptz,
  created_at      timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT bookings_slot_bounds CHECK (NOT isempty(slot) AND lower_inc(slot) AND NOT upper_inc(slot)),
  CONSTRAINT bookings_duration CHECK (upper(slot) - lower(slot) IN
    (interval '60 minutes', interval '90 minutes', interval '120 minutes')),
  CONSTRAINT bookings_hold CHECK ((status = 'held') = (hold_expires_at IS NOT NULL)),
  CONSTRAINT bookings_no_overlap EXCLUDE USING gist (court_id WITH =, slot WITH &&)
    WHERE (status IN ('held', 'reserved'))
);
```

- Faixa `[início, fim)`: um jogo das 19h às 20h e outro das 20h às 21h não se sobrepõem.
- Uma segunda inserção conflitante falha com `exclusion_violation` (SQLSTATE `23P01`). A API traduz para "horário acabou de ser reservado".
- Cancelar ou deixar expirar muda o status para `released` e libera a faixa sem apagar a linha. Isso cumpre a regra "jogos cancelados liberam o horário".
- Horário de funcionamento, "horário de hoje que já passou" e antecedência máxima (Q7) são validados no service, porque dependem de `courts` e do relógio.
- O Postgres mais recente tem `UNIQUE (court_id, slot WITHOUT OVERLAPS)`, uma chave temporal que equivale ao `EXCLUDE`. Por ser uma restrição `UNIQUE`, ela não leva predicado `WHERE`, então não substitui a exclusão parcial por status.

### 6.2 Dinheiro em centavos e registro contábil que nunca é editado

- Todo valor é `bigint` em centavos com `CHECK (>= 0)`. Só `ledger_entries.amount_cents` tem sinal.
- **Razão em partidas dobradas**: `ledger_transactions` (o fato) e `ledger_entries` (as pernas). Sinal positivo é entrada na conta e negativo é saída. A soma das pernas de uma transação é zero, garantida por um *constraint trigger* `DEFERRABLE INITIALLY DEFERRED` que confere no `COMMIT`.
- **Append-only no banco**: trigger `BEFORE UPDATE OR DELETE` que levanta erro nas duas tabelas, mais `REVOKE UPDATE, DELETE` do papel da aplicação. Um erro se corrige com uma transação nova (`kind = 'reversal'`, `reverses_id`, que é único: cada transação só pode ser revertida uma vez).
- **Sem lançamento duplicado**: `ledger_transactions.idempotency_key UNIQUE` (ex.: `charge_received:<gateway_payment_id>`, `refund:<refund_id>`, `escrow_release:<escrow_hold_id>`).
- **Cada centavo explicado**: a perna aponta para o `charge_item_id` que a originou (parte da quadra, taxa de 5%, tarifa, árbitro, caução, pendência). O acerto (`settlements.rates`, `policy_version`) registra que regra foi aplicada.

```sql
CREATE FUNCTION forbid_change() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN RAISE EXCEPTION '% is append-only', TG_TABLE_NAME; END $$;

CREATE TRIGGER ledger_entries_append_only BEFORE UPDATE OR DELETE ON ledger_entries
  FOR EACH ROW EXECUTE FUNCTION forbid_change();

CREATE FUNCTION ledger_tx_balanced() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  IF (SELECT coalesce(sum(amount_cents), 0) FROM ledger_entries
      WHERE transaction_id = NEW.transaction_id) <> 0 THEN
    RAISE EXCEPTION 'ledger transaction % does not balance', NEW.transaction_id;
  END IF;
  RETURN NULL;
END $$;

CREATE CONSTRAINT TRIGGER ledger_entries_balanced AFTER INSERT ON ledger_entries
  DEFERRABLE INITIALLY DEFERRED FOR EACH ROW EXECUTE FUNCTION ledger_tx_balanced();
```

Exemplo do briefing (quadra de R$ 100, 50/50; o time A paga a sua parte no Pix com tarifa `t`, cujo valor depende do contrato):

| Transação | Conta | Centavos |
| :--- | :--- | ---: |
| `charge_received` | `payer` (time A) | −(5.250 + t) |
| | `payee_escrow` (quadra) | +5.000 |
| | `platform_fee` | +250 |
| | `gateway_tariff` | +t |
| `escrow_release` (fim do jogo) | `payee_escrow` (quadra) | −5.000 |
| | `payee_released` (quadra) | +5.000 |
| `refund` (se A cancelar: 90% do split da quadra, regra atual) | `payee_escrow` (quadra) | −4.500 |
| | `payer` (time A) | +4.500 |

### 6.3 Idempotência com o Asaas

O Asaas não tem header de idempotência. A proteção fica do nosso lado:

- **Webhook**: a entrega é "at least once", e um reenvio mantém o mesmo `id` de evento (documentação do Asaas, *Como implementar idempotência em Webhooks*). `asaas_webhook_events.asaas_event_id UNIQUE`; `INSERT … ON CONFLICT (asaas_event_id) DO NOTHING`; responde `200` **depois** de gravar; o worker processa as linhas `pending`. O token vem no header `asaas-access-token` e é conferido antes de gravar. O processamento não depende da ordem: relê o objeto no Asaas (`GET /v3/payments/{id}`) e aplica a transição atômica.
- **Criar cobrança**: `charges.id` vai como `externalReference` (`UNIQUE` em `charges.external_reference`). Antes de tentar de novo depois de um timeout, o worker procura a cobrança pelo `externalReference` e só recria se não achar. O mesmo vale para `split[].externalReference` (`charge_splits.external_reference`).
- **Estorno**: a resposta do Asaas **não tem id de estorno** (`PaymentRefundGetResponseDTO` traz `dateCreated`, `status`, `value`, `refundedSplits`, mas nenhum `id`). A linha em `refunds` é gravada antes da chamada, com `idempotency_key UNIQUE`. Se a chamada falhar sem resposta, o worker relê a cobrança e compara `refunds[]` por valor, `dateCreated` e `refundedSplits[].id` antes de chamar de novo.
- **Liberação**: `finish` num escrow já encerrado é inofensivo do nosso lado, porque `escrow_holds` só sai de `active` uma vez. O comportamento do Asaas numa segunda chamada está **a confirmar**.

### 6.4 Localização com PostGIS

- `courts.location` e `check_ins.location` são `geography(Point,4326)`. Em `geography`, `ST_DWithin` e `ST_Distance` trabalham em **metros** (documentação do PostGIS), então o raio do check-in é um número em metros: `ST_DWithin(check_ins.location, courts.location, :raio_m)`.
- Índice `GIST (location)` em `courts`, para o mapa e para "quadras perto de mim".
- O mapa do app (Google Maps ou Mapbox) só lê `ST_Y`/`ST_X`. O ponto é salvo pelo toque no mapa ou pelo GPS no cadastro.

### 6.5 Fila de jobs do worker

Os prazos não podem depender de celular aberto (§9: "vira uma rotina no servidor"). Tipos de job: `booking.hold_expire`, `team_invitation.expire`, `hire.respond_deadline`, `hire.payment_deadline`, `hire_share.payment_deadline`, `checkin.close` (início + 15 min), `match.finish` (`ends_at`), `escrow.finish` (`release_due_at`), `escrow.poll` (não há webhook de mudança do escrow), `webhook.process`, `charge.reconcile`, `notification.push` e `challenge.expire`.

Há duas saídas, ainda **sem escolha definitiva** (Q23):

1. **`pg-boss`**: cria o próprio schema e oferece `send(name, data, { startAfter, singletonKey, retryLimit })` e `schedule(name, cron, data, { tz })`. A opção `db` permite gravar o job na mesma transação da regra de negócio. Nesse caso, a tabela `jobs` deste documento não existe.
2. **Tabela própria `jobs`**: `run_at`, `dedupe_key UNIQUE` (o mesmo prazo não é agendado duas vezes) e consumo com `SELECT … FOR UPDATE SKIP LOCKED`.

Nas duas, cada job é idempotente: relê o estado e só age se a transição ainda for válida (um `match.finish` atrasado num jogo já `cancelled` não faz nada).

### 6.6 Dados bancários ficam no Asaas (LGPD)

- Chave Pix, banco, agência, conta, documento e selfie vão direto para a subconta Asaas (KYC em `POST /v3/accounts` e no fluxo de aprovação). O banco guarda só `asaas_account_id`, `asaas_wallet_id`, `kyc_status`, `document_last4` e `payout_summary` (o resumo mascarado que os outros veem).
- A resposta de `POST /v3/accounts` traz `accessToken.apiKey` da subconta. Ela **não** vai para coluna comum. Se for necessária, fica num cofre de segredos, e o banco guarda só `api_key_secret_ref` (Q11).
- `POST /v3/customers` exige `name` e `cpfCnpj` do pagador. O CPF completo não é guardado (Q10). Fica só `asaas_customer_id` e os últimos dígitos.

### 6.7 Notificações

`notifications.kind` tem um valor por linha da tabela da §9 do briefing: `hire_invited`, `hire_accepted`, `referee_share_due`, `hire_declined`, `hire_expired`, `hire_paid`, `hire_payment_expired`, `hire_withdrawn`, `match_invited`, `match_confirmed`, `match_declined`, `challenge_accepted`, `opponent_paid_court`, `match_cancelled`, `walkover`, `debt_due`, `debt_pending_for_payee`, `debt_paid`, `suspension`, `match_finished_review`, `payout_released`, `score_reported`, `team_reviewed`, `match_invitation_expired` e `check_in_confirmed_by_court`. `target_route` e `target_params` levam à tela do assunto. Abrir o sino faz `UPDATE … SET read_at = now() WHERE user_id = $1 AND read_at IS NULL`. `dedupe_key` impede que um job reexecutado avise duas vezes. O push sai por job (`notification.push`) para os `push_devices` do usuário.

### 6.8 Avaliações, W.O. e ranking

- `team_reviews`: as 4 notas são obrigatórias (`NOT NULL` e `CHECK` de 1 a 5). Há uma avaliação por time por jogo (`UNIQUE (match_id, reviewer_team_id)`).
- "Só aparece quando os dois avaliarem": quando a segunda avaliação do jogo é gravada, o service preenche `visible_at` nas duas linhas na mesma transação. As consultas públicas filtram `visible_at IS NOT NULL`. Sem a segunda avaliação, a primeira não aparece (ver Q14b sobre um prazo para liberar).
- W.O. pesa como 10 avaliações de 1 estrela em pontualidade e comprometimento. A view `team_ratings` soma, para cada `walkovers` do time, `rating_weight` notas 1 nesses dois critérios. Não são criadas avaliações falsas, e o W.O. continua no histórico como linha própria. Conferência com o exemplo do briefing: um time com 4,8 e 51 jogos cai para cerca de 4,5 com um W.O.
- Selos (Pontual, Jogo limpo, Comprometido com nota a partir de 4,6/4,7; Jogo pegado com jogo limpo abaixo de 4) e o rótulo "Novo" são calculados na view, com limites guardados como constantes de configuração.

### 6.9 Moderação, denúncias e administração

- `blocked_terms` alimenta o filtro de palavras. O texto bloqueado **não é gravado** na tabela de destino: a API devolve o motivo e registra o bloqueio em `moderation_blocks`, sem o texto inteiro. A revisão por IA fica desligada (`engine = 'word_filter'`).
- `profile_reports` só pode ser lida por `app_role = 'admin'`. A regra é aplicada no service, e há também a opção de *row-level security* do Postgres.
- `suggestions` e `profile_reports` formam a lista "Para o dono do app" (§7).

## 7. Mapeamento Asaas

### 7.1 Ids do Asaas em cada tabela

| Objeto Asaas | Id / campo | Onde fica | Observação |
| :--- | :--- | :--- | :--- |
| Subconta (recebedor) | `id` de `POST /v3/accounts` | `payees.asaas_account_id` | usado em `POST /v3/accounts/{id}/escrow` |
| Carteira da subconta | `walletId` | `payees.asaas_wallet_id` | usado em `split[].walletId` |
| Chave de API da subconta | `accessToken.apiKey` | **fora do banco** (`payees.api_key_secret_ref`) | Q11 |
| Configuração do escrow | `daysToExpire`, `enabled`, `isFeePayer` | `payees.escrow_days_to_expire` (cópia) | por subconta, vale só para recebimentos novos |
| Cliente pagador | `id` (`cus_…`) | `payer_customers.asaas_customer_id` | `cpfCnpj` obrigatório |
| Cobrança | `id` (`pay_…`), `externalReference` | `gateway_payments.asaas_payment_id`, `charges.external_reference` | uma por parcela no cartão parcelado |
| Parcelamento | `installment` | `charges.asaas_installment_id` | |
| Split | `split[].id`, `split[].externalReference` | `charge_splits.asaas_split_id`, `.external_reference` | `fixedValue` (ou `totalFixedValue` no parcelamento) |
| Garantia do escrow | `escrow.id` | `escrow_holds.asaas_escrow_id` | `GET /v3/payments/{id}/escrow` devolve **um** objeto por cobrança; grão por recebedor **a confirmar** |
| Estorno | **sem id**; `dateCreated`, `value`, `refundedSplits[].id` | `refunds` (id nosso) e `refund_splits` | conciliação por campos |
| Evento de webhook | `id` (`evt_…`), `event` | `asaas_webhook_events.asaas_event_id`, `.event_type` | chave única |

### 7.2 Estados do Asaas para estados nossos

**Cobrança** (`payment.status` → `gateway_payments.asaas_status` bruto; o service deriva `charges.status`):

| Asaas | `charges.status` |
| :--- | :--- |
| `PENDING`, `AWAITING_RISK_ANALYSIS` | `pending` |
| `CONFIRMED` | `confirmed` (cartão aprovado, crédito futuro) |
| `RECEIVED` | `received` |
| `OVERDUE` | `expired` |
| `REFUND_REQUESTED`, `REFUND_IN_PROGRESS` | sem mudança; `refunds` em `pending` |
| `REFUNDED` | `refunded` |
| reembolso parcial (evento `PAYMENT_PARTIALLY_REFUNDED`) | `partially_refunded`; o `status` que o Asaas mostra depois de um estorno parcial está **a confirmar** |
| `CHARGEBACK_REQUESTED`, `CHARGEBACK_DISPUTE`, `AWAITING_CHARGEBACK_REVERSAL` | `chargeback` (alerta ao admin) |
| `RECEIVED_IN_CASH`, `DUNNING_REQUESTED`, `DUNNING_RECEIVED` | não usados (boleto e dinheiro); `ignored` |
| eventos `PAYMENT_CREDIT_CARD_CAPTURE_REFUSED`, `PAYMENT_REPROVED_BY_RISK_ANALYSIS` | `failed` |
| evento `PAYMENT_DELETED` | `cancelled` |

**Split** (`split[].status` → `charge_splits.status`, mesmo nome em minúsculas): `PENDING`, `PROCESSING`, `AWAITING_CREDIT`, `DONE`, `CANCELLED`, `REFUNDED`, `PROCESSING_REFUND` e `BLOCKED_BY_VALUE_DIVERGENCE`. O evento `PAYMENT_SPLIT_DONE` traz `additionalInfo.splitId`. Um split `DONE` numa subconta com escrow **continua retido**.

**Estorno** (`refunds[].status` → `refunds.status`): `PENDING` → `pending`; `AWAITING_CRITICAL_ACTION_AUTHORIZATION` e `AWAITING_CUSTOMER_EXTERNAL_AUTHORIZATION` → `awaiting_authorization`; `DONE` → `done`; `CANCELLED` → `cancelled`. Evento `PAYMENT_REFUND_DENIED` → `failed`.

**Escrow** (`escrow.status` e `finishReason` → `escrow_holds.status`): `ACTIVE` → `active`; `DONE` com `REQUESTED_BY_CUSTOMER` → `released` (que esse motivo corresponde ao `finish` pela API está **a confirmar**); `EXPIRED` → `expired`; `PAYMENT_REFUNDED` → `ended_by_refund`; `CHARGEBACK`, `INSUFFICIENT_BALANCE` e `CUSTOMER_CONFIG_DISABLED` → `ended_other`.

**Subconta** (eventos `ACCOUNT_STATUS_GENERAL_APPROVAL_*` → `payees.kyc_status`): `PENDING` → `pending_documents`; `AWAITING_APPROVAL` → `awaiting_approval`; `APPROVED` → `approved`; `REJECTED` → `rejected`. Perfil com recebedor fora de `approved` não é publicado.

Eventos assinados no webhook: `PAYMENT_CREATED`, `PAYMENT_CONFIRMED`, `PAYMENT_RECEIVED`, `PAYMENT_OVERDUE`, `PAYMENT_DELETED`, `PAYMENT_REFUNDED`, `PAYMENT_PARTIALLY_REFUNDED`, `PAYMENT_REFUND_IN_PROGRESS`, `PAYMENT_REFUND_DENIED`, `PAYMENT_CREDIT_CARD_CAPTURE_REFUSED`, `PAYMENT_REPROVED_BY_RISK_ANALYSIS`, `PAYMENT_CHARGEBACK_REQUESTED`, `PAYMENT_CHARGEBACK_DISPUTE`, `PAYMENT_AWAITING_CHARGEBACK_REVERSAL`, `PAYMENT_SPLIT_DONE`, `PAYMENT_SPLIT_CANCELLED`, `PAYMENT_SPLIT_DIVERGENCE_BLOCK`, `PAYMENT_SPLIT_DIVERGENCE_BLOCK_FINISHED` e `ACCOUNT_STATUS_GENERAL_APPROVAL_*`.

### 7.3 A confirmar no sandbox

1. `splitRefunds` com a garantia do escrow ainda ativa (o go/no-go do AD-007).
2. **Estorno de cartão parcelado por recebedor**: `POST /v3/installments/{id}/refund` aceita só `value`, sem `splitRefunds`. Não se sabe se `POST /v3/payments/{id}/refund` com `splitRefunds` em cada parcela resolve, nem como fica o escrow de parcelas futuras. Isso afeta todo cancelamento e W.O. pago em 2x a 6x (Q12).
3. Grão do escrow: `GET /v3/payments/{id}/escrow` devolve um objeto por cobrança, mas `daysToExpire` é por subconta e cobranças com vários recebedores podem ter prazos diferentes. Falta saber se `finish` libera todos os splits da cobrança e com qual chave de API ele é chamado (conta-pai ou subconta).
4. Valor máximo de `daysToExpire` (limita o horizonte de reserva, Q7).
5. `status` da cobrança depois de um estorno parcial, e se é possível filtrar a lista de cobranças por `externalReference`.
6. Se o Asaas Checkout hospedado aceita split, escrow e parcelamento (escopo PCI).

## 8. As cinco regras em conflito com o AD-007

O modelo não decide nenhuma delas. As duas saídas cabem sem mudar o esquema, porque o acerto é feito de linhas (`settlement_lines`) com a regra congelada (`settlements.policy_version`, `rates`), as linhas do valor dizem de quem é cada parte (`charge_items.kind`, `payee_id`) e a pendência é uma tabela genérica (`debts.reason`). Nenhum `CHECK` ou enum fixa um dos lados.

| Regra do briefing | Saída do briefing no modelo | Saída do relatório no modelo |
| :--- | :--- | :--- |
| Caução da arbitragem | `charge_items.kind = 'referee_deposit'` com `payee_id` do árbitro (split com escrow), devolvida por `refunds.reason = 'deposit_return'`. o `CHECK` de `charge_items` exige `payee_id` e impede caução na conta da plataforma | sem `referee_deposit`; no W.O., `debts.reason = 'referee_share_replacement'` cobra do faltoso a metade que falta |
| Quem cancela arca com os 5% do outro time | `debts.reason = 'cancellation_other_team_fee'` (nova cobrança a quem cancelou) | `settlement_lines.action = 'absorb_by_platform'`: o estorno da taxa sai da conta-pai (a parte de `value` fora de `splitRefunds`) |
| W.O.: presente perde 30% da sua metade | linha `release_to_payee` com 30% da metade do presente para a quadra | 30% da quadra inteira sai da parte do faltoso; a falta vira `debts.reason = 'walkover_shortfall'` |
| 30% no W.O. e 5% que não volta | `rates = {"walkover_court": 0.30, ...}`, linha `keep_platform_fee` | tabela escalonada em `rates` e `policy_version`; a taxa volta por `refund_to_payer` quando quem pagou não deu causa |
| Liberação no fim do jogo para cartão; tarifa não devolvida | `escrow_holds.released` no fim do jogo; `charge_items.payment_tariff` fora do estorno | texto do aviso usa `gateway_payments.estimated_credit_date`; o estorno pode incluir a tarifa para o time inocente |

## 9. Conferência do briefing, item a item

Sessões financeiras e de prazo (3, 4, 5, 10 e 11). "Q" aponta a pergunta aberta quando a regra não cabe sem decisão.

| § | Regra | Onde fica no modelo |
| :--- | :--- | :--- |
| 3 | Na reserva o time paga só a quadra; arbitragem depois do aceite | `charges.purpose = 'court_share'`; `hire_shares` criadas no aceite |
| 3 | Adversário ou desafio aberto | `matches.away_team_id` nulo + `challenges.match_id` |
| 3 | Duração 1h, 1h30, 2h com preço proporcional | `bookings_duration` `CHECK`; `matches.court_price_cents` congelado |
| 3 | Divisão 50/50 liga ou desliga; sempre ligada com desafio aceito | `matches.split_mode`; regra no service de `matches` |
| 3 | 0, 1 ou 2 árbitros, só se faltar mais de 1h | `hires.referee_slot IN (1,2)` + índice único parcial; janela no service |
| 3 | Resumo com taxa de 5% e tarifa repassada | `charge_items` (`service_fee`, `payment_tariff`, `rate_bp`) |
| 3 | Pix (QR) ou cartão, até 6x com acréscimo | `charges.billing_type`, `installment_count`, `charge_items.installment_surcharge`; Q12 |
| 3 | Horário trava quando o primeiro time paga | `bookings.status`: `held` → `reserved`; Q6 |
| 3 | Convidado confirma pagando ou tocando Confirmar; recusa vira desafio | `team_invitations` + §5.1 |
| 3 | Caução = metade do outro time, volta no fim | Q1; `referee_deposit` cabe |
| 3 | Adversário não paga a arbitragem em 2h: pagar por ele ou cancelar | `hire_shares.paid_by_team_id` (`covered`) ou `hires` → `cancelled` |
| 3 | Split pelo gateway | `charge_splits` com `fixedValue` |
| 4 | Contratar até 1h30 antes | validação no service; `hires.invited_at` |
| 4 | Aceitar em 2h, terminando 1h antes; perto do jogo, metade do tempo | `hires.respond_by` gravado |
| 4 | Pagar em 2h após o aceite, até 1h antes | `hires.pay_by`, `hire_shares.pay_by` |
| 4 | Desistir até 2h antes; depois é W.O. | `hires.withdraw_until`; `withdrawn` / `no_show` |
| 4 | Perder prazo desfaz sem cobrar o que não foi pago | nada é cobrado antes do aceite; `expired` / `payment_expired` sem `charges` |
| 4 | Jogador: valor + 5%; passe livre confirma no aceite | `hires.free_pass`; `charge_items` `player_fee` + `service_fee` |
| 4 | Árbitro: até 2 por jogo; 50/50 com metade + 5% + caução | `referee_slot`; duas `hire_shares`; Q1 |
| 4 | Valor retido e liberado no fim ou no W.O. | `escrow_holds.release_due_at`; job `escrow.finish`; Q5 |
| 4 | Filtros por modalidade e posição; perfil com habilidades confirmadas, W.O. e avaliações | `court_types`, `positions`, `profile_skills` + `skill_endorsements`, `walkovers`, `provider_reviews` (Q14) |
| 5 | Jogo termina sozinho no fim do horário; repasses nesse momento | job `match.finish` em `matches.ends_at`; `settlements.kind = 'match_finished'` |
| 5 | Cartão do jogo: o que foi pago, o que falta, quem confirmou | `charges` + `hire_shares` + `team_invitations` + `check_ins` |
| 5 | + Jogador / + Árbitro enquanto faltar mais de 1h; cancelar até 2h antes | validação no service |
| 5 | Check-in de capitão, árbitro e jogador; GPS ou quadra; abre −1h, fecha +15min; quem não faz leva W.O. | `check_ins` (§5.9); job `checkin.close` |
| 5 | Placar informado pela quadra | `matches.home_score`, `away_score`, `score_reported_at` |
| 5 | Tabela do acerto no W.O. e no cancelamento | `settlements` + `settlement_lines` |
| 5 | Cancelamento: volta a parte da quadra −10% (fica para a quadra), árbitro, jogadores e caução; 5% não volta; arca com os 5% do outro; tarifa não volta | `settlement_lines` + `rates`; Q2, Q4, Q5 |
| 5 | O outro time recebe 100%, incluindo a taxa | `refund_to_payer` com a taxa (estorno da conta-pai) |
| 5 | Contratados avisados, contratação desfeita | `hires` → `cancelled`; `notifications.kind = 'match_cancelled'` |
| 5 | W.O.: faltoso arca com 30% da quadra e a arbitragem inteira; presente perde 30% da metade; quadra fica com 30% | `settlements.kind = 'walkover_one_team'`, linhas por recebedor; Q3, Q4 |
| 5 | Faltoso sem saldo: pendência, bloqueio, avisos a quadra e árbitro | `debts` + `debt_items`; bloqueio por pendência aberta; `debt_pending_for_payee`, `debt_paid` |
| 5 | Os dois faltaram: metade dos custos para cada um | `settlements.kind = 'walkover_both_teams'`, `at_fault_team_id` nulo |
| 5 | Faltoso recebe o equivalente a 10 notas 1 | `walkovers.rating_weight` na view `team_ratings` |
| 5 | Árbitro ou jogador sem check-in: W.O., valor volta, cartão vermelho de 7 dias | `walkovers`, `refunds.reason = 'hire_no_show'`, `suspensions.period` |
| 10 | Prestador recebe sempre o preço cheio | `charge_splits.amount_cents` = `hires.price_cents` / `courts` (split fixo) |
| 10 | Passe livre sem taxa | `free_pass` → nenhum `charge_items` |
| 10 | Taxa administrativa de 10% para a quadra | `rates.cancellation_fee`; linha `release_to_payee` |
| 10 | Exemplo de W.O. com 50/50 (presente R$ 65, quadra R$ 30, árbitro R$ 30, goleiro R$ 30, faltoso R$ 35) | 5 `settlement_lines`; o teste de aceitação da spec de pagamentos usa estes números |
| 10 | Exemplo sem 50/50 (R$ 238 volta, quadra R$ 42 + R$ 42 pendentes) | estorno de 23.800 do split da quadra + `debts` de 4.200 com `debt_items` da quadra |
| 10 | Prazos (tabela) | `respond_by`, `pay_by`, `withdraw_until`, `hold_expires_at`, `ends_at`, `suspensions.period`, jobs da §6.5 |
| 10 | Recebimento por Pix ou conta bancária | na subconta Asaas; `payees.payout_summary` |
| 10 | Aviso de reembolso com prazo (Pix na hora, cartão até 2 faturas) | `charges.billing_type` no texto da notificação |
| 11 | Tudo termina 1h antes; contratar só até 1h30 | prazos gravados e validação no service |
| 11 | Duração 1h, 1h30, 2h | `CHECK` em `bookings` |
| 11 | Quem confirma o W.O.: check-in; sem sinal, a quadra | `check_ins.method` |
| 11 | Placar pela quadra | `matches.*_score` |
| 11 | W.O. sem 50/50: faltoso paga tudo; dois sem check-in: metade | `settlements.kind` |
| 11 | Saldo devedor bloqueia | `debts` |
| 11 | Taxa do inocente: quem cancelou arca | Q2 |
| 11 | Tarifas repassadas e não devolvidas | `charge_items.payment_tariff`; Q5 |
| 11 | Parcelamento com acréscimo | `installment_surcharge`; Q12 |
| 11 | Moderação por filtro de palavras; denúncia | `blocked_terms`, `moderation_blocks`, `profile_reports` |
| 11 | Cartão vermelho para todos (7 dias) | `suspensions` |

Duas regras vieram do relatório e não estão no briefing: o **horizonte máximo de reserva** (Q7) e o **cadastro de recebedor com KYC real** (`payees.kyc_status`).

## 10. Perguntas abertas

Cada pergunta traz uma recomendação, e nenhuma está decidida. As marcadas com **[pagamentos]** bloqueiam a spec de pagamentos.

| # | Pergunta | Recomendação |
| :--- | :--- | :--- |
| Q1 **[pagamentos]** | Caução da arbitragem: manter (no split do árbitro com escrow, com estorno em todo jogo normal) ou eliminar? | Eliminar. No W.O., o árbitro fica com a metade já paga pelo faltoso, e a outra metade vira pendência do faltoso. Assim, nenhum jogo normal gera estorno |
| Q2 **[pagamentos]** | Os 5% do time inocente no cancelamento: a plataforma absorve ou vira cobrança nova para quem cancelou? | A plataforma absorve (estorno da própria taxa). Uma cobrança de centavos a quem cancelou custa mais em tarifa e atrito do que vale |
| Q3 **[pagamentos]** | W.O.: o time presente continua perdendo 30% da sua metade? | Não. O faltoso paga os 30% da quadra inteira com a parte dele; se não cobrir, a diferença vira pendência |
| Q4 **[pagamentos]** | Percentuais de retenção (30% no W.O., 10% no cancelamento) e taxa de 5% que nunca volta | Tabela escalonada por antecedência com teto de 10% a 20%, mostrada antes do pagamento com aceite expresso. A taxa volta para quem não deu causa. Validar com advogado |
| Q5 **[pagamentos]** | Promessa "pagamento liberado no fim do jogo" no cartão (D+32 por parcela) e tarifa não devolvida | Prometer "Pix no fim do jogo; cartão conforme a liquidação", com a data mostrada no painel. Devolver a tarifa ao time inocente |
| Q6 **[pagamentos]** | Quando o horário trava: ao abrir o checkout (reserva temporária) ou só quando o pagamento confirma? O briefing diz "assim que um jogo ativo é criado" (§2) e "quando o primeiro time paga" (§3) | Reserva temporária (`held`) de 10 a 15 min ao abrir o checkout, que vira `reserved` na confirmação. Sem isso, dois times podem pagar pelo mesmo horário e um deles precisa ser estornado |
| Q7 **[pagamentos]** | Com quanta antecedência um jogo pode ser marcado? O `daysToExpire` do escrow é fixo por subconta | 30 dias, com `daysToExpire` = 30 + margem (a confirmar contra o máximo do Asaas) |
| Q8 **[pagamentos]** | Desafio aberto com quadra paga que ninguém aceita até o início: o que acontece com o dinheiro? | Fecha sozinho até 2h antes como cancelamento sem culpa: estorno integral da quadra sem os 10%. Se a taxa volta segue a Q4 |
| Q9 **[pagamentos]** | Recebedor: uma subconta por pessoa (CPF) ou por perfil (quadra, árbitro, jogador)? Quadra com CNPJ? | Uma subconta por documento (CPF ou CNPJ). A mesma pessoa usa a subconta do CPF para árbitro e jogador, e a quadra pode ter a subconta do CNPJ. Isso evita pagar mensalidade de escrow em dobro |
| Q10 **[pagamentos]** | O Asaas exige CPF do pagador. O app guarda o CPF completo? | Não. Pedir no primeiro pagamento, enviar ao Asaas e guardar só `asaas_customer_id` e os 4 últimos dígitos |
| Q11 **[pagamentos]** | Guardar a `apiKey` da subconta devolvida pelo Asaas? | Só se algum fluxo exigir operar como a subconta (ex.: envio de documentos do KYC). Nesse caso, num cofre de segredos, com referência no banco |
| Q12 **[pagamentos]** | Parcelamento em até 6x: o estorno de parcelamento não aceita `splitRefunds` | Lançar com Pix e cartão à vista até o sandbox provar estorno parcial por recebedor em cobrança parcelada |
| Q13 | Multa por "pendência": o devedor é o time ou o capitão (pessoa)? | O time, com o capitão como responsável perante o Asaas na cobrança da pendência. Validar com advogado (relatório: "Quem é o devedor da dívida de W.O.") |
| Q14 | Quem avalia quadra, árbitro e jogador, e com que critérios? (O briefing mostra nota e número de avaliações, mas só descreve a avaliação entre times.) b) Se um time não avaliar, a nota do outro aparece depois de um prazo? | a) O time contratante dá nota de 1 a 5 ao árbitro e ao jogador, os dois times dão nota à quadra e as habilidades podem ser confirmadas na mesma tela. b) Liberar em 7 dias |
| Q15 | Vaga "time pedindo jogador" e "Quero jogar" (§1): como viram contratação? | A candidatura aceita pelo time vira `hires` em `invited`, já aceito pelo jogador, seguindo para o pagamento com os mesmos prazos |
| Q16 | Jogador de passe livre precisa cadastrar recebimento? O §8 lista "recebimento" como obrigatório | Não exigir para passe livre, porque ele não recebe nada |
| Q17 | Raio do check-in por GPS | 150 m, guardado como configuração |
| Q18 | Termos do glossário marcados com † (`RentalPlayer`, `Walkover`, `Deposit`, `PlayerOpening`) | Manter `RentalPlayer` e `Walkover` (são os nomes do domínio) |
| Q19 | Time tem só o capitão, ou outros membros podem pagar e fazer check-in? | V1 só com o capitão, como no protótipo |
| Q20 | Com duração de 1h30, a grade de blocos de 1h fica com meio bloco livre. Início em hora cheia ou meia hora? | Grade de 30 min e início em hora cheia ou meia hora |
| Q21 | Retenção de dados (LGPD) para usuário que pede exclusão | Anonimizar o perfil e manter o financeiro pelo prazo legal (a definir com o contador) |
| Q22 | Antecipação do cartão: a plataforma antecipa e embute o custo no acréscimo? | Ligada à Q5. Não antecipar no lançamento |
| Q23 | Fila de jobs: `pg-boss` ou tabela própria `jobs`? | `pg-boss`, que já traz `startAfter`, `singletonKey`, novas tentativas, cron com fuso e envio na mesma transação. Decidir no design da `foundation` |
