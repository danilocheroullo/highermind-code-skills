# /hm-data-integrity — Dados Sagrados (v1)

Voce esta agora em **modo data integrity**. Seu trabalho e validar que dados do user nao sao perdidos. Nem por bug, nem por crash, nem por migration ruim, nem por operacao destrutiva acidental, nem por disco morto. Dados sao sagrados — esse principio precisa de implementacao concreta, nao so promessa.

## Principio central

A unica perda aceitavel de dado e a explicitamente autorizada pelo owner. Tudo o mais e bug. Se dados podem ser perdidos por um comando errado, um crash mal tratado, ou uma migration apressada, o produto nao esta pronto pra ter usuarios — nem ele mesmo.

## Quando usar

- Antes de shippar projeto que persiste qualquer coisa (DB, files, blob storage)
- Apos mudanca em schema ou migration
- Quando definir politica de backup
- Apos qualquer incidente envolvendo perda de dados
- Periodicamente (auditoria de manutencao, especialmente projetos com dados pessoais/financeiros)

## Niveis

| Nivel | Quando | Foco |
|---|---|---|
| **Local single-user** | Apps pessoais (oracle, family-os) | Backup local automatico, migration safety, undo de destrutivos |
| **Multi-user privado** | CRM interno, ferramentas de time | Tudo acima + replicacao, RTO/RPO, audit trail |
| **Producao publica** | SaaS com clientes pagantes | Tudo acima + DR plan, geo-redundancia, compliance (LGPD/GDPR) |

## Dominios

### 1. Backup strategy

| Check | Criterio |
|---|---|
| Backup automatico | Existe? Quando dispara (schedule, evento)? |
| Backup atomico | Snapshot consistente, nao corrompido por write em curso? (ex: sqlite `.backup`, pg_dump --serializable) |
| Backup criptografado | Em repouso e em transito (se vai pra remoto) |
| Backup versionado | Multiplas geracoes. Nao so o "ultimo" — historico |
| Backup testado | Voce ja restaurou um backup com sucesso? Se nao, nao tem backup, tem **esperanca**. |
| Backup off-site | Pra projetos importantes: copia geografica separada (S3 outra regiao, git remoto, etc) |
| Retention policy | Daily 7d, weekly 4w, monthly 12m? Definido? |

**Patterns:**
- **Single-user app local:** snapshot SQLite no `before-quit` do Electron + commit/push pra repo privado
- **Multi-user:** wal-g pg backups continuous + S3 versioned bucket
- **Files:** rclone sync incremental encrypted

### 2. Migration safety

| Check | Criterio |
|---|---|
| Migrations versionadas | Drizzle, Prisma, Alembic — sequencial, hash-locked |
| Migrations idempotentes | Re-rodar nao quebra (auto-migrate na boot) |
| Migrations roll-forward only? | Production = sim. Dev/local = pode roll-back |
| Migration destrutiva exige confirmacao | DROP COLUMN, DROP TABLE — never silent |
| Backup ANTES de migration grande | Auto ou manual? Documentado? |
| Migration testada em copia de prod | Antes de aplicar em prod real |
| Rollback plan documentado | Mesmo que roll-forward only, plano se algo der errado |
| Schema journal sincronizado | Drizzle/Prisma journal bate com estado real do DB? |

**Anti-patterns:**
- ALTER COLUMN sem default/backfill (NOT NULL em coluna nova com rows existentes)
- DROP COLUMN sem etapa intermediaria de read-only
- Migration que demora muito sem `CONCURRENTLY` em indexes (Postgres lock)
- Schema mudou no codigo sem migration correspondente

### 3. Operacoes destrutivas

| Check | Criterio |
|---|---|
| DELETE sem WHERE bloqueado | ORM raw SQL nao executa "DELETE FROM x" sem clause |
| Soft-delete por default | `deleted_at` ao inves de DROP. Hard-delete e excecao auditada |
| Confirmacao explicita pra hard-delete | UI: dialog com nome do objeto. CLI: --force flag |
| Backup automatico antes de bulk operation | DROP TABLE, TRUNCATE, mass UPDATE — backup primeiro |
| Audit log de destrutivos | Quem fez, quando, o que removeu |
| Recovery window | Soft-delete fica X dias antes de purge real |
| Owner approval pra ops em prod | Migration prod, mass delete, schema change — owner em loop |

**Pattern de confirmacao (CLI):**
```bash
$ tool delete user 123
ERROR: Destructive operation. Add --force to confirm.

$ tool delete user 123 --force
Backing up affected rows to /tmp/backup-...
Deleted 1 user, 47 related records.
```

### 4. Data integrity em runtime

| Check | Criterio |
|---|---|
| Transactions onde atomicidade importa | Multi-table writes em transaction |
| Foreign keys + ON DELETE CASCADE configurado | Sem orphaned records |
| Unique constraints em campos que devem ser unicos | DB enforced, nao so app-level |
| NOT NULL em campos obrigatorios | DB enforced |
| Check constraints pra invariantes | `amount >= 0`, `email LIKE '%@%'`, etc |
| Optimistic concurrency em writes simultaneos | `updated_at`/`version` no WHERE de UPDATE |
| Idempotency keys em operacoes externamente disparadas | Webhook, payment, email send |

### 5. Schema validation runtime (JSON.parse + cast)

| Check | Criterio |
|---|---|
| JSON column tem schema versionado | `payload.schemaVersion` field |
| safeParse via Zod/Pydantic em todo JSON.parse de DB | Sem cast cego |
| Migration de schema versionado documentada | Como fazer v1 → v2 sem perder dados |
| Backwards compat enquanto migra | App lida com v1 E v2 durante transicao |

### 6. Disaster recovery (multi-user / producao)

| Metrica | Definicao | Tipico |
|---|---|---|
| **RPO** (Recovery Point Objective) | Quanto dado pode perder? | 5min a 1h dependendo da criticidade |
| **RTO** (Recovery Time Objective) | Quanto tempo pra voltar? | 15min a 4h |
| **Disaster scenarios documentados** | DB corrompido, regiao AWS down, ransomware, dev errou comando | Pelo menos 4 cenarios cobertos |
| **DR drill executado** | Restore real testado em ambiente paralelo | Ao menos 1x/ano |
| **Runbook de recuperacao** | Passo-a-passo escrito | Acessivel mesmo se app down |

### 7. Compliance (se aplicavel — LGPD/GDPR/HIPAA)

| Check | Criterio |
|---|---|
| Right to erasure | User pode pedir exclusao real? Dados removidos de backups apos retention? |
| Right to access | User pode exportar TODOS seus dados em formato legivel? |
| Audit log de acessos | Quem visualizou dados sensiveis (PII, saude, financeiro)? |
| Breach notification plan | Procedimento de 72h pra ANPD/DPA documentado |
| DPO designado | (LGPD/GDPR) |
| Data minimization | Coleta minima pro proposito declarado |

### 8. File/blob integrity (se app maneja arquivos)

| Check | Criterio |
|---|---|
| Checksum em uploads | SHA256 calculado e verificado |
| Versioning em blob storage | S3 versioned bucket pra recovery de overwrite acidental |
| Lifecycle policy | Auto-archive pra cold storage apos N dias |
| Object lock em arquivos criticos | Imutabilidade WORM pra compliance/legal |
| Backup de blob storage | S3 cross-region replication ou snapshot externo |

### 9. Observabilidade pra detectar problemas cedo

- Alerta quando backup falha
- Alerta quando DB size cresce muito rapido (potencial bug ou ataque)
- Alerta quando query lenta (indica index missing)
- Alerta quando rate de DELETE alto (indica bug ou ataque)
- Dashboard com volume de dados, ultimo backup, tamanho dos backups

## Output

```
DATA INTEGRITY AUDIT
Projeto: [nome]
Nivel aplicado: [Local / Multi-user / Producao publica]
Volume estimado: [rows/files, tamanho]

DOMINIO 1: BACKUP
[Check]: PASS/FAIL — detalhes + fix se FAIL

DOMINIO 2: MIGRATION
[Check]: PASS/FAIL

DOMINIO 3: DESTRUTIVAS
[Check]: PASS/FAIL

DOMINIO 4: RUNTIME INTEGRITY
[Check]: PASS/FAIL

DOMINIO 5: SCHEMA VALIDATION
[Check]: PASS/FAIL

DOMINIO 6: DR (se Multi-user/Producao)
RPO atual: [valor]
RTO atual: [valor]
Drill executado: [data ultimo / nunca]

DOMINIO 7: COMPLIANCE (se aplicavel)
[Check]: PASS/FAIL

DOMINIO 8: FILES (se aplicavel)
[Check]: PASS/FAIL

DOMINIO 9: OBSERVABILIDADE
[Check]: PASS/FAIL

VEREDICTO
Dados protegidos / EM RISCO — X criticos pra resolver primeiro
```

## Regras

- **Toda perda potencial de dado e CRITICO.** Sem MEDIO. Sem BAIXO.
- Backup que nunca foi restaurado nao e backup. Se nunca testou, marca FAIL.
- Migration sem rollback plan e CRITICO em producao.
- DELETE sem confirmacao em CLI/UI publico = CRITICO.
- Disco morto e cenario obrigatorio em DR plan.
- Owner aprova politica de retention. Default conservador: nao apagar nada que pode ser util.
- Se projeto e single-user local: backup automatico no quit + commit pra repo privado e baseline minimo.
- Compliance falha = nao shippa pra mercado regulado. Sem negociacao.
