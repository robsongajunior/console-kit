# TAREFA: Arquitetar o plano de migração do `azion-console-kit` para um novo monorepo Nuxt

## 0. MODO DE OPERAÇÃO (leia antes de qualquer coisa)

Você atua como **arquiteto de software sênior** especializado em Vue/Nuxt, monorepos e DX para agentes de IA.

Regras não-negociáveis desta tarefa:

1. **NÃO escreva código de aplicação.** O entregável é um conjunto de documentos de planejamento. Snippets ilustrativos são permitidos apenas quando forem a forma mais curta de explicar uma decisão (máx. ~20 linhas cada).
2. **Descoberta antes de proposta.** Execute a Fase 0 (seção 4) lendo o código real dos 4 paths abaixo. Qualquer afirmação sobre o projeto atual deve vir do código, não de suposição.
3. **Não invente.** Se algo não puder ser determinado a partir do código ou de documentação oficial, registre como `❓ OPEN QUESTION` em uma lista consolidada no fim do plano.
4. **Verifique versões.** Antes de fixar qualquer versão em exemplo de `package.json`, confirme o que existe hoje (`npm view <pkg> version`, changelog oficial). Não confie em memória de treino — Nuxt, Tailwind, Storybook e o ecossistema Sentry mudaram recentemente.
5. **Decida, não liste.** Onde houver alternativas, apresente trade-offs e **escolha uma**, justificando. Um plano com 5 "poderíamos usar X ou Y" não é um plano.
6. Antes de escrever o plano final, me faça **até 8 perguntas bloqueantes** (só as que mudam a arquitetura). Depois siga com o que tiver.
7. Documentos em **inglês**; a conversa comigo pode ser em português.

---

## 1. CONTEXTO

Vamos migrar o console atual (Vue 3 + Vite) para um monorepo novo baseado em Nuxt, aproveitando a oportunidade para consolidar padrões, limpar dívida técnica e deixar a base preparada para desenvolvimento assistido por agentes.

### Paths locais (leia todos)

| Path | O que é |
|---|---|
| `/Users/unknown1/www/azion/azion-console-kit/` | Projeto atual a ser migrado (fonte da verdade sobre features, telas e serviços) |
| `/Users/unknown1/www/azion/webkit/packages/webkit` | Design system — publicado como `@aziontech/webkit` |
| `/Users/unknown1/www/azion/webkit/packages/theme` | Tokens/tema — publicado como `@aziontech/theme` |
| `/Users/unknown1/www/azion/webkit/packages/icons` | Ícones — publicado como `@aziontech/icons` |

Repositório público de referência: https://github.com/aziontech/azion-console-kit

---

## 2. OBJETIVO

Um **plano de migração faseado, executável e verificável** que qualquer dev (ou agente) consiga pegar e começar a executar sem precisar reinterpretar intenção.

---

## 3. PRINCÍPIOS E RESTRIÇÕES

- **Reaproveitamento primeiro:** nada que já exista no `@aziontech/webkit` deve ser reimplementado. Componente novo só quando houver gap comprovado (justifique cada um).
- **Baixo custo de remoção:** `packages/services` é uma abstração **temporária**. Deve ser desenhado para ser deletado/substituído pelo SDK da Azion sem refatorar as telas. Aponte explicitamente qual é a fronteira (interfaces, tipos, pontos de acoplamento) que garante isso.
- **Sem publicação em NPM agora.** Modularização é para organização interna e fronteiras claras, não distribuição. Considere isso ao decidir build/bundling dos packages internos.
- **Migração incremental**, não big-bang. Proponha estratégia de coexistência (strangler fig, rotas espelhadas, proxy, feature flag — escolha e justifique) e como validar paridade com o console atual.
- Código limpo, patterns consolidados, convenções únicas e aplicáveis por lint — não por code review manual.

---

## 4. FASE 0 — DESCOBERTA (obrigatória, antes do plano)

Produza estes inventários a partir da leitura do código. Tabelas, sem prosa.

### 4.1 Inventário de telas/rotas
Para cada rota do console atual: path, domínio de negócio, tipo (list / detail / create / edit / wizard / dashboard), complexidade (S/M/L), dependências (serviços, feature flags, permissões), e **onda de migração sugerida**.

### 4.2 Inventário de serviços
Para cada service/API client: endpoint(s), domínio, quem consome, tem tipagem?, tem teste?, padrão de erro atual, e se já existe cobertura no SDK da Azion.

### 4.3 Inventário de componentes + gap analysis vs. webkit
Três colunas de saída:
- **Já existe no webkit** → usar direto (mapear nome antigo → novo)
- **Deveria ir para o webkit** → candidato a contribuição upstream (justifique)
- **Fica em `packages/components`** → componente de plataforma, específico do console (ex.: composições de listagem, formulários de recurso, blocos de dashboard). Este é o principal output esperado desta seção.

### 4.4 Dívida técnica e armadilhas de migração
Padrões atuais que **não** devem ser portados, acoplamentos ocultos, código morto, uso de estado global, e o que quebra ao sair de Vite/SPA para Nuxt.

### 4.5 Decisões de plataforma a resolver
No mínimo: SSR vs. SPA vs. híbrido por rota; estratégia de auth/sessão (onde o token vive, refresh, guards, SSR-safe); data fetching (`useFetch`/`useAsyncData` vs. camada própria); state management; i18n; feature flags; RBAC/permissões; tratamento de erro e loading states; roteamento e code splitting.

---

## 5. ARQUITETURA ALVO (ponto de partida — critique e ajuste)

```
console-kit/
├── apps/
│   ├── console/          # Aplicação Nuxt
│   └── storybook/        # Docs e playground de componentes
├── packages/
│   ├── components/       # Componentes de plataforma reutilizáveis
├── services/  # Abstração de API/SDK (descartável por design)
├── package.json
└── pnpm-workspace.yaml
```

Avalie e proponha ajustes com justificativa: precisa de `packages/config` (eslint/ts/tailwind compartilhados)? `packages/types`? `packages/testing`? `apps/e2e`? Onde vivem os mocks/fixtures? Regras de dependência entre packages (e como impedir import ilegal via lint).

---

## 6. STACK

Trate a lista abaixo como **intenção**, não como decreto. Se algo estiver errado, obsoleto ou conflitante, diga.

### Raiz (devDependencies)
`commitlint` (`@commitlint/cli`, `@commitlint/config-conventional`) · `eslint` (`vue-eslint-parser`, `eslint-plugin-vue`, `eslint-plugin-security`, `eslint-plugin-xss`, `eslint-plugin-no-unsanitized`) · `prettier` · `husky` · `vitest` · `zod`

> Defina: flat config ou eslintrc? Onde mora a config compartilhada? Como o lint roda rápido o suficiente para ser hook de pre-commit no monorepo inteiro? Versionamento/changelog — vale semantic-release aqui ou é overkill sem publicação em NPM?

### `apps/console`
- Nuxt (fixar major após verificação)
- `@aziontech/webkit` ≥ 4.3.0 · `@aziontech/theme` ≥ 4.3.0 · `@aziontech/icons` ≥ 4
- Tailwind 4+ — **atenção:** descreva como Tailwind 4 coexiste com os tokens do `@aziontech/theme` e com o CSS do webkit sem conflito de camadas
- JSONForms: `@jsonforms/core`, `@jsonforms/vue`, `@jsonforms/vue-vanilla` (3.8.0) — proponha a estratégia de **renderer set** custom usando componentes do webkit, e quando usar JSONForms vs. formulário à mão
- `motion-v` ^2.3.0 (confirmar nome do pacote)

### `apps/storybook`
Storybook (última major). Defina: só `packages/components` ou também composições do console? Interaction tests e a11y addon entram no CI?

### Observabilidade
- Sentry para Vue/Nuxt — **verifique o pacote correto e a integração oficial para Nuxt**, incluindo source maps, releases, e captura server-side.
- Além de erro: web vitals, performance budgets, e checagem de disponibilidade (ver seção 7).

---

## 7. AI-FIRST (requisito de primeira classe, não um extra)

O repositório deve ser projetado para que agentes trabalhem nele de forma autônoma e segura. Especifique concretamente:

### 7.1 Contexto para agentes
- `CLAUDE.md` / `AGENTS.md` na raiz e por package: o que é, como rodar, convenções, o que **não** fazer.
- ADRs curtos para decisões arquiteturais (formato e onde ficam).
- Estrutura previsível o suficiente para um agente inferir onde criar um arquivo novo sem perguntar.

### 7.2 Automação de tarefas recorrentes
- Slash commands / skills para os fluxos repetitivos: criar novo service, criar nova tela CRUD, adicionar componente + story + teste, triagem de issue do Sentry.
- Geradores determinísticos (plop/hygen ou script próprio) — o agente invoca o gerador em vez de escrever boilerplate à mão.
- MCPs relevantes e para quê (Sentry, GitHub, Figma, etc.).

### 7.3 Loops de feedback rápidos e determinísticos
Um agente só é bom quanto o sinal que recebe. Defina:
- Comandos únicos e rápidos: `typecheck`, `lint`, `test:unit`, `test:e2e:smoke`, `verify` (o gate completo).
- Mensagens de erro acionáveis; falha rápida; sem flakiness tolerada.
- Estratégia de teste que um agente consegue escrever sozinho: o que é unit (Vitest), o que é component test, o que é E2E (Playwright?), como mocks de API são gerados/mantidos (MSW? fixtures a partir de OpenAPI?).

### 7.4 Sentry → correção automatizada
Descreva o pipeline: como o erro chega estruturado o suficiente para um agente agir. Inclua tagging por domínio/feature/owner, `CODEOWNERS`, source maps confiáveis por release, breadcrumbs úteis, e o runbook do agente (reproduzir → teste que falha → correção → PR).

### 7.5 Performance e saúde
Web vitals em produção, budgets de bundle e Lighthouse no CI (com thresholds que quebram o build), checks sintéticos de disponibilidade, e o que fazer quando um budget estoura.

### 7.6 Guardrails
O que agentes **não** podem alterar sem humano (migrations, auth, config de CI, deps de segurança), como isso é imposto tecnicamente, e como o CI valida um PR gerado por agente.

---

## 8. ENTREGÁVEIS

Crie os arquivos em `docs/plan/`:

| Arquivo | Conteúdo |
|---|---|
| `00-discovery.md` | Todos os inventários da seção 4 |
| `01-architecture.md` | Arquitetura alvo, fronteiras, regras de dependência, decisões de plataforma |
| `02-roadmap.md` | Plano por fases (formato da seção 9) |
| `03-services.md` | Ordem de migração dos serviços + contrato/padrão da camada |
| `04-components.md` | Gap analysis e catálogo dos componentes a construir |
| `05-ai-first.md` | Seção 7 detalhada |
| `06-conventions.md` | Convenções de código, nomenclatura, estrutura de pastas, lint rules |
| `07-open-questions.md` | Tudo que ficou em aberto, com o impacto de cada pendência |

No fim, me devolva no chat um **resumo executivo de no máximo 30 linhas** com: as 5 decisões mais importantes, os 3 maiores riscos, e o que precisa ser respondido para destravar a Fase 1.

---

## 9. FORMATO DE CADA FASE DO ROADMAP

Cada fase deve conter:

- **Objetivo** — em uma frase, orientado a resultado
- **Escopo** — telas, serviços e componentes específicos (nomeados)
- **Pré-requisitos** — o que precisa estar pronto antes
- **Critérios de saída** — verificáveis e binários ("login funciona em SSR com refresh de token e teste E2E verde", não "auth implementado")
- **Riscos e mitigação**
- **Tamanho relativo** — S/M/L (não estime em horas)

Esqueleto sugerido — ajuste conforme a descoberta:
`Fase 0` fundação do monorepo e tooling → `Fase 1` shell da app + auth + layout + design system plugado → `Fase 2` primeira vertical slice completa (um domínio ponta a ponta, provando os padrões) → `Fase 3+` ondas por domínio → `Fase N` desligamento do console antigo.

A **Fase 2 é a mais importante**: ela define os padrões que todas as outras vão copiar. Detalhe-a mais que as demais.

---

## 10. CRITÉRIOS DE ACEITE DO PLANO

O plano está pronto quando:

- [ ] Cada inventário da Fase 0 foi preenchido a partir do código real
- [ ] Cada componente proposto em `packages/components` tem justificativa de por que não vem do webkit
- [ ] A fronteira de descarte do `packages/services` está explícita
- [ ] Todas as decisões de plataforma (4.5) têm uma escolha feita e justificada
- [ ] Cada fase tem critérios de saída binários
- [ ] Versões de dependências foram verificadas, não assumidas
- [ ] Cada requisito AI-First aponta para um artefato concreto do repo
- [ ] Toda incerteza está em `07-open-questions.md`, e não escondida como afirmação

---

## 11. NON-GOALS

- Publicar packages na NPM
- Reescrever ou estender o `@aziontech/webkit` (apenas apontar candidatos a upstream)
- Migrar o backend ou definir contratos de API novos
- Escrever o código da aplicação nesta tarefa