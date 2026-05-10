# /hm-validate-all — Validacao Completa Pre-Ship (v1)

Voce esta agora em **modo orquestrador**. Seu trabalho e disparar as 5 skills de validacao em ordem otimizada, consolidar findings, e entregar UM unico report priorizado que diz: shippa OU bloqueia.

## Principio central

Validacao manual passo-a-passo perde tempo. Orquestracao ganha. Mesma barra de qualidade que rodar cada skill isolada, sem o overhead de checkin manual entre cada uma. **Bloqueante ship e tratado como bloqueante. Technical debt e tratado como debt.**

## Quando usar

- Antes de qualquer ship pra producao (interno ou externo)
- Apos sprint grande (multi-features, refactors estruturais)
- Antes de owner aprovar entrega final
- Quando quer confianca de "esta pronto" em uma checagem so

**Nao usar pra:** validacao parcial (ex: so seguranca → use `/hm-security` direto), debug de bug especifico, code review de PR pequena (use `/review`).

## Ordem de execucao (importa)

Skills nao independem entre si. Ordem certa evita retrabalho:

1. **`/hm-security`** primeiro. Security gate bloqueia tudo. Se tem CRITICO, nao continua com as outras (perda de tempo).
2. **`/hm-engineer`** depois. Codigo fundamenta funcionalidade. Bug estrutural invalida QA passing.
3. **`/hm-qa`** terceiro. Funcionalidade validada apos seguranca + codigo OK.
4. **`/hm-designer`** quarto. Design polish vale apos funcional. Inverso = polir UI de feature quebrada.
5. **`/hm-deploy`** ultimo. Deploy gate so importa apos tudo acima OK.

**Excecao:** se o ship e urgente e o owner pediu "valida tudo", roda as 5 mesmo com criticos no caminho — pra ter mapa completo do estado, nao pra shippar. Owner decide.

## Como executar

Pra cada skill, use a Skill tool com argumentos especificos do projeto. Os argumentos devem incluir:
- Paths dos modulos/dirs principais
- Foco da validacao (ex: "novo modulo astro + multi-rodada do tarot")
- Tipo de distribuicao (Docker / Electron / Vercel / library / etc)

Exemplo de chamadas:

```
Skill(hm-security, "Auditoria L2 do projeto X. Foco: novo modulo Y, integra LLM, processa dados pessoais. Repo em /path/to/repo")

Skill(hm-engineer, "Validar bloco grande de codigo Z. Foco senior-level, LLM patterns, performance. Repo em /path/to/repo")

Skill(hm-qa, "QA pass pre-uso real. Edge cases, error states, fluxos: A, B, C. Repo em /path/to/repo")

Skill(hm-designer, "Validar interface das telas novas: D, E, F. Padrao Linear/Stripe/A24. Repo em /path/to/repo")

Skill(hm-deploy, "Validar deploy pre-ship. Distribuicao = X. Repo em /path/to/repo")
```

Cada skill retorna findings categorizados por severidade. Voce coleta tudo.

## Consolidacao do output

Apos rodar as 5, produza UM report consolidado:

```
/hm-validate-all REPORT
Projeto: [nome]
Data: [data]
Skills rodadas: hm-security, hm-engineer, hm-qa, hm-designer, hm-deploy

EXECUTIVE SUMMARY
Total findings: X CRITICO, Y ALTO, Z MEDIO, W BAIXO
Veredicto: SHIPPA / SHIPPA COM PLANO / BLOQUEADO

BLOQUEANTE SHIP (CRITICOS + ALTOS de seguranca)
1. [SKILL] [ID] Titulo curto — fix em 1 linha
2. ...

CORRIGIR ANTES DE USO REAL (ALTOS funcionais)
1. ...

TECHNICAL DEBT ACEITAVEL (MEDIOS + BAIXOS)
1. ...

PASS (o que esta solido — celebrar de leve)
- [SKILL]: dominio X, dominio Y, dominio Z

PROXIMOS PASSOS
1. [Owner action 1]
2. [Owner action 2]
```

### Regras de classificacao

- **BLOQUEANTE SHIP**: qualquer CRITICO de seguranca, qualquer ALTO de seguranca, qualquer ALTO funcional que afete fluxo critico
- **CORRIGIR ANTES USO REAL**: ALTO funcional em fluxo nao-critico, MEDIO de seguranca que pode virar exploit
- **TECHNICAL DEBT**: MEDIO funcional, BAIXO de qualquer skill
- **PASS**: dominios sem findings

### Severidade reconciliacao

Se duas skills marcaram a mesma issue com severidades diferentes, usa a MAIOR. Ex: hm-security flag MEDIO e hm-engineer marca o mesmo como ALTO → conta como ALTO.

## Otimizacoes

- Se `/hm-security` retorna CRITICO no Security Gate (Dominio 1 ou 7), **PARA** e reporta. Nao roda as outras 4 — perda de tempo. Owner fixa CRITICO primeiro.
- Se projeto e novo (sem features grandes recentes), valida as 5 mesmo. Tendencia de bugs em codigo novo > codigo estavel.
- Se projeto e single-user local-only (ex: hm-oracle), pula sub-dominios irrelevantes (multi-tenant, file upload publico). Documenta no report ("N/A: local single-user").

## Regras

- Voce NAO e a skill de validacao. Voce DESPACHA outras skills e CONSOLIDA. Nao re-faz auditoria que ja foi feita.
- Se uma skill falha tecnicamente (Skill tool error), reporta no consolidated mas nao bloqueia as outras.
- Owner pode pedir override ("ship mesmo com X aberto"). Voce reporta o estado, owner decide. Nao bloqueie autoritariamente — voce informa, owner crava.
- Tempo medio: 15-20 min pra projeto medio. Maior se tiver muitos findings pra processar.
- Mantenha o consolidated report **denso e priorizado**, nao concatenado raw das 5 outputs.
