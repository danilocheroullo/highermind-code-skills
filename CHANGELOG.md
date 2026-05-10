# Changelog

Todas as mudanças notáveis neste projeto serão documentadas aqui.

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/).

---

## [3.0.0] — 2026-05-10

LLM-native evolution + 4 skills novas + orquestrador. Motivado por aprendizados rodando as 5 skills num ship real (hm-oracle: tarot multi-rodada + módulo astro + cross-channel context + home meditativa). Onde as skills v2 deixaram passar — bug de chat history sem limite, singleton stale, race condition em geração cara, falta de retry no streaming, falta de fallback manual em forms — agora pegam.

### Adicionado

#### Nova skill: `/hm-validate-all` — orquestrador
- Dispara as 5 skills de validação em ordem otimizada (security → engineer → qa → designer → deploy)
- Consolida findings em UM report priorizado (bloqueante ship / corrigir antes uso real / technical debt aceitável)
- Reconcilia severidades quando duas skills marcam mesmo issue
- Para no Security Gate se CRITICO encontrado (não desperdiça tempo nas outras 4)
- Tempo médio: 15-20 min pra projeto médio

#### Nova skill: `/hm-llm-guardrails` — patterns LLM-app
- 12 patterns obrigatórios pra app que integra Claude/GPT/Gemini
- Sliding window de chat history (corrige bug de context overflow após 30+ turns)
- Lazy client factory (sem singleton stale quando user troca API key em runtime)
- In-flight dedupe pra geração cara (evita double billing em refresh duplo)
- Rate limit por endpoint LLM (anti-runaway cost)
- Streaming abort cleanup + retry route com marker no DB
- Schema validation no response (Zod/Pydantic, sem JSON.parse cego)
- Token budget explícito (nunca sem max_tokens)
- Cross-channel context safety (LLM-A injetando em LLM-B com user_notes adversarial)
- Cost tracking + estimativa por sessão
- Multi-provider failover (L3, opcional)
- Tool calling guardrails (sandbox, max iterations, schema validation)

#### Nova skill: `/hm-data-integrity` — dados sagrados
- Backup strategy (atômico, criptografado, versionado, testado, off-site)
- Migration safety (idempotente, reversível ou roll-forward only, backup antes)
- Operações destrutivas (confirmação, soft-delete default, audit log)
- Runtime integrity (transactions, FK, constraints, idempotency keys)
- Schema validation runtime (JSON.parse + Zod sempre)
- DR plan (RPO/RTO, drill executado, runbook escrito)
- Compliance LGPD/GDPR/HIPAA (right to erasure/access, breach notification)
- File/blob integrity (checksum, versioning, lifecycle)
- Observabilidade pra detectar problemas cedo

#### Nova skill: `/hm-perf` — performance profiling
- Bundle size (alvos por framework: Next, Vite)
- Render performance (Core Web Vitals: LCP, INP, CLS)
- API latency (p50/p95/p99)
- Database (indexes, slow queries, cursor pagination)
- LLM tokens (cost por turn, prompt caching, latency)
- Network (HTTP/2, compression, CDN, cache headers)
- Memory (leaks, listeners, bound caches)
- Build performance (HMR <2s, cold build <60s)
- Tools por stack (bundle-analyzer, React DevTools Profiler, pg_stat_statements, web-vitals)

#### Nova skill: `/hm-ux-flow` — validação de fluxo
- Não substitui `/hm-designer` (visual). Foca em DECISÃO do user.
- Detecta 3 tipos de friction: decisão desnecessária, mal posicionada, sem informação
- Hierarquia de decisão (funil: o que → como → confirmar)
- Reversibilidade (acao reversível vs irreversível vs destrutiva)
- Recovery de erro (form preserve dados, msgs acionáveis, dead-ends)
- Friction points conhecidos (onboarding, choice paralysis, missing affordance)
- Mobile vs desktop (touch targets, gestures, bottom sheets)
- Empty states + loading states (shimmer obrigatório, sem spinner genérico)

### Mudado

#### `/hm-engineer` v3 — LLM patterns no padrão senior
- Padrão senior inegociável ganhou 3 itens: zero singletons stale, zero JSON.parse + cast sem validação, zero history unbounded em LLM
- Nova seção "LLM-app patterns" entre Performance e Custo: 10 patterns recorrentes que scanners não pegam (sliding window, lazy client, in-flight dedupe, streaming abort, cross-channel safety, etc)
- Cross-reference com `/hm-llm-guardrails` pra deep audit

#### `/hm-security` v2.2 — LLM-app gotchas expandidos
- Domínio 12 (AI/LLM Security) ganhou 4 sub-domínios novos:
  - 12.5 PII em prompts ganhou check de "disclaimer transparente ao user"
  - 12.6 Cross-channel context safety (LLM-A → LLM-B, confused deputy mitigation)
  - 12.7 API key lifecycle e singleton stale
  - 12.8 Streaming endpoint safety (abort, resumability, backpressure, in-flight billing)
  - 12.9 Sliding window obrigatório (CRITICO se chat route sem `.limit(N)`)

#### `/hm-deploy` v3 — multi-modelo distribution
- Nova seção 0: Distribution Model (identifica modelo antes da auditoria)
- Checks específicos por modelo: Container/Docker, Serverless/Edge, Desktop (Electron), Mobile (Expo/RN), Library/SDK, CLI tool
- Antes da v3, era 100% Docker-centric e exigia adaptação mental pra outros modelos
- Pula seções não aplicáveis ao modelo (ex: Electron não tem `.dockerignore` → pula Domínio 1.1)

#### `/hm-qa` v3 — edge case checklist
- Nova seção: Edge case checklist (8 categorias × N checks cada)
- Categorias: Formulários (fallback manual!), Streaming endpoints, Erros 4xx/5xx (CTA acionável!), Estados de UI, Mobile, LLM-app, Concorrência, Dados sagrados
- Bugs recorrentes em 80% dos projetos quando ninguém testa de verdade

### Filosofia

A v2 era pra times que sabem o que estão fazendo. A v3 é pra times que sabem o que estão fazendo COM LLM. Padrão senior sobrevive — só ganhou patterns nativos do mundo onde apps tem agente, conversa, e chamada externa cara em todo lugar.

---

## [2.1.0] — 2026-04-08

Security-first evolution. Motivado por falhas reais no Orion Finance (Dockerfile com `npm run dev`, sem `.dockerignore`, `--reload` no entrypoint) que passaram pela v2.

### Adicionado

#### Nova skill: `/hm-security` — Auditoria de seguranca dedicada
- Skill world-class baseada em OWASP ASVS 5.0, CIS Benchmarks, SLSA, e metodologias de Tempest/CrowdStrike/Trail of Bits
- 3 niveis de auditoria: L1 (baseline), L2 (enterprise), L3 (critical systems)
- 8 dominios: Container, Aplicacao (OWASP Top 10 + API Top 10), Auth, Dados, Dependencias, Infra, Logging, Crypto
- Secrets scan automatico com patterns
- Supply chain audit (lock files, SBOM, provenance)
- Business logic testing (race conditions, IDOR, privilege escalation)
- Compliance mapping (LGPD, GDPR, PCI-DSS)
- Severidades padronizadas: CRITICO bloqueia, sem excecao

#### Skills existentes — Security gate integrado
- `/hm-deploy`: Security Gate como secao 0 (bloqueante antes de tudo)
- `/hm-engineer`: OWASP Top 10 + Container Security como primeira auditoria
- `/hm-init`: Seguranca obrigatoria desde o commit zero (.dockerignore, multi-stage, non-root)
- `/hm-qa`: Security Audit como primeira etapa antes de testes

### Mudado

- Seguranca movida de "camada de auditoria" para "gate bloqueante" em todas as skills
- Output de todas as skills agora tem secao de seguranca no topo
- Tabela de ports atualizada com todos os projetos ativos

---

## [2.0.0] — 2026-04-03

Evolução completa das skills baseada em aprendizados reais de 5+ projetos construídos com o framework (higher-mind-os, Scout, HM Finance, Orion, higher-mind-community).

### Adicionado

#### Nova skill: `/hm-deploy`
- Validação completa de infraestrutura, containers e reprodutibilidade
- Checklist de Docker (subida, rebuild, dados sagrados)
- Validação de environment e secrets
- Checklist de database e migrations
- Health checks e monitoramento
- Teste de reprodutibilidade (clone limpo)
- Segurança de deploy (ports, CORS, HTTPS, secrets)
- Tabela de ports dos projetos Higher Mind pra evitar colisões

#### `/hm-init` — Framework de decisão de stack
- Tabela de critérios ponderados pra avaliação de stack (fit, performance, custo, maturidade, ecossistema, DX, hiring)
- Anti-patterns de escolha explícitos
- Seção de arquitetura agent-first como default (quando aplicável)
- Infraestrutura local com Docker Compose desde o dia 1
- Restrições de custo como parte do design (API calls, hosting, bandwidth)
- Documentação obrigatória via ARCHITECTURE.md
- Princípio "dados são sagrados" desde o primeiro docker-compose.yml

#### `/hm-engineer` — Padrão senior e novas camadas de auditoria
- Baseline de engenheiro senior inegociável (zero bare except, zero any types, zero fire-and-forget, zero secrets hardcoded, zero queries sem limit)
- Nova camada: **Custo x Performance** (API calls justificadas, contexto mínimo em LLMs, token usage consciente)
- Nova camada: **Dados sagrados** (nenhuma operação destrutiva sem confirmação, volumes nomeados, migrations não-destrutivas)
- Nova camada: **Infraestrutura** (Docker rebuild vs restart, migrations automáticas, health checks, ports)
- Expansão de Performance: I/O paralelo, memoização
- Expansão de Arquitetura: validação de agent loops (max iterations, token limits, timeout)
- Regra: dados em risco é sempre CRÍTICO

#### `/hm-designer` — Agent-first UI e pixel perfect
- Filosofia agent-first: UI = visibilidade + override, não input principal
- Referência A24 adicionada às referências estéticas
- Seção **Pixel perfect**: zero tolerância a desalinhamentos, quebras, cortes
- Padrões técnicos: full-width layout, shimmer (não spinner), dark-first, inline styles quando framework não coopera, transições 200-300ms
- Novos anti-patterns: formulários onde agente deveria executar, spinners genéricos, layout centralizado em telas grandes
- Regra: desalinhamento arquitetural (formulário vs agente) é finding

#### `/hm-qa` — Infraestrutura, agente e custo como teste
- Nova seção: **Verificação de infraestrutura** (containers, migrations, ports, volumes, rebuild, .env)
- Nova seção: **Verificação de agente** (tool loops, alucinação de tools, token usage, custo por interação)
- Nova seção: **Integridade de dados** (persistência, migrations não-destrutivas, backups, operações destrutivas)
- Nova seção: **Check de custo** (API calls por fluxo, contexto mínimo, calls redundantes, custo por usuário/mês)
- Output expandido com seções de Infraestrutura, Agente, Integridade de Dados e Custo

### Mudado

- Contagem de skills: 4 → 5 (adição de `/hm-deploy`)
- SKILL.md parent atualizado pra refletir 5 skills
- Todas as skills agora consideram agent-first como paradigma (quando aplicável)
- Performance expandida de "não ter N+1" pra incluir custo de APIs externas e token management

---

## [1.0.0] — 2026-03-12

Release inicial.

### Skills
- `/hm-init` — Início de projeto com melhores ferramentas e estrutura
- `/hm-engineer` — Validação de código em todas as camadas
- `/hm-designer` — Validação de interface contra o mais alto padrão
- `/hm-qa` — Quality assurance completo

### Infraestrutura
- Setup script com symlinks automáticos
- CLAUDE.md.template como ponto de partida
- Instalação global e por projeto
