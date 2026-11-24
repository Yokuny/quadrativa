# STATE

## Decisions

### AD-001
- **Decision**: TypeScript em todo o monorepo (app, web, API, workers e pacotes compartilhados).
- **Reason**: Uma linguagem só para um time pequeno apoiado por agentes; tipos e esquemas de validação compartilhados entre app e API sem geração de código; mais fácil contratar no Brasil. Desempenho não é gargalo (app regional, limitado por banco e gateway). User: "TypeScript de ponta a ponta com Fastify".
- **Trade-off**: Abre mão do binário único, da menor memória e da concorrência nativa do Go.
- **Scope**: Todos os apps e pacotes.
- **Date**: 2026-11-24
- **Status**: active

### AD-002
- **Decision**: Backend em Fastify puro, como monólito modular, sem framework opinativo (sem NestJS). A arquitetura é imposta por uma skill do projeto + `AGENTS.md` e por regras de fronteira automáticas no CI (dependency-cruiser ou ESLint boundaries), não pelo framework.
- **Reason**: Menos código de estrutura e menos mecanismos implícitos; testes com funções puras e app Fastify em memória; agentes seguem uma skill documentada. Regras de fronteira impedem que a documentação envelheça sem ninguém perceber. User: "TypeScript de ponta a ponta com Fastify".
- **Trade-off**: As convenções (módulos, injeção, erros, transações) são nossas para definir e manter; perde o ecossistema e a familiaridade de mercado do NestJS.
- **Scope**: `apps/api` e workers; o desenho das camadas é definido na funcionalidade `foundation`.
- **Date**: 2026-11-24
- **Status**: active

### AD-003
- **Decision**: App em React + Capacitor para Android e iOS, com o mesmo código publicado também como versão web funcional, totalmente mobile-first.
- **Reason**: Uma base de código para web, Android e iOS. A versão web funcional favorece React DOM + Capacitor em vez de Expo/React Native. User: "Capacitor e uma versão web funcional porem totalmente mobile-first".
- **Trade-off**: A interface roda em WebView: desempenho menor que nativo em Android de entrada (listas longas, mapa) e dependência de plugins para GPS, push e câmera. Cordova descartado.
- **Scope**: `apps/` do cliente e `packages/ui`; toda tela é desenhada primeiro para celular.
- **Date**: 2026-11-24
- **Status**: active

### AD-004
- **Decision**: Banco relacional PostgreSQL (com PostGIS), gerenciado. Fornecedor de hospedagem ainda não definido.
- **Reason**: Domínio relacional e financeiro: transações ACID para divisão, reembolso e W.O.; restrição de exclusão contra reservas sobrepostas; PostGIS para o check-in por localização; SQL para conciliação e ranking. User: "e sql".
- **Trade-off**: Esquema rígido com migrações, contra a flexibilidade de documentos do MongoDB, que este domínio não precisa.
- **Scope**: Toda a persistência da API e dos workers.
- **Date**: 2026-11-24
- **Status**: active

### AD-005
- **Decision**: O fluxo de spec é a skill `tlc-spec-driven` (Specify → Design → Tasks → Execute, com scripts de verificação e verificador independente). Do processo do Ethica entram só o registro vivo por funcionalidade (`docs/features/<slug>/feature.md`), o `docs/roadmap.md` com pendências por dono e o índice gerado com hook de início de sessão; esses três são montados na funcionalidade `foundation`. A arquitetura de referência do Ethica (`monorepo-architecture`) é a base do design da `foundation`, adaptada para Fastify (AD-002) e Postgres (AD-004), sem separação por cliente (multi-tenant).
- **Reason**: As regras de dinheiro e prazo do briefing justificam prova por teste e verificação independente; o registro vivo do Ethica mantém a memória do projeto entre sessões. User: "1 tlc".
- **Trade-off**: Processo mais pesado que o `ethica-spec-driven` em funcionalidades médias; duas fontes (`.specs/` congelado e `docs/features/` vivo) para manter coerentes.
- **Scope**: Todo o desenvolvimento; `.specs/`, `docs/`, `AGENTS.md`.
- **Date**: 2026-11-24
- **Status**: active

### AD-006
- **Decision**: Trabalho direto na `main`, sem branch nem Pull Request. Depois do gate verde, o agente commita e faz `git push origin main` sem perguntar. Commits em Conventional Commits com assunto em português e **sem `Co-Authored-By`, `Generated with` ou qualquer menção a LLM**, mesmo que o harness peça. Nunca `--force` nem reescrita de histórico: erro enviado se corrige com commit novo.
- **Reason**: Escolha do usuário, espelhando o AD-143 do Ethica. User: "2 sem co-authored, 3 direto na main".
- **Trade-off**: Sem revisão humana de código: a qualidade depende do gate local, do CI na `main` e das skills do projeto. Um commit quebrado chega direto à `main`.
- **Scope**: Todos os commits e pushes do repositório.
- **Date**: 2026-11-24
- **Status**: active

## Handoff
