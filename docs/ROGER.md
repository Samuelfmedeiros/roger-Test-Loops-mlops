# Roger 🐐 — Test Loop Orchestrator

> Documentação completa do sistema de test-loop obrigatório do ecossistema Samuel.
> Versão: 3.4 · Última atualização: 22/10/2026

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Regra Global](#2-regra-global)
3. [Arquitetura](#3-arquitetura)
4. [Pipeline Completo](#4-pipeline-completo)
5. [Operação](#5-operação)
6. [Configuração por Projeto](#6-configuração-por-projeto)
7. [HTML de Entrega](#7-html-de-entrega)
8. [Roger Router](#8-roger-router)
9. [Cerberus — Red Team](#9-cerberus--red-team)
10. [Crons Integrados](#10-crons-integrados)
11. [Pitfalls & Troubleshooting](#11-pitfalls--troubleshooting)
12. [Histórico de Versões](#12-histórico-de-versões)

---

## 1. Visão Geral

**Roger** é o orquestrador de test-loop obrigatório do ecossistema Samuel. O nome foi batizado por Samuel em 30/08/2026 — ao ler a confirmação radiofônica "Roger" como confirmação de entrega, ele mandou: *"coloca o nome dele como Roger"*.

### Objetivo

Garantir que **toda entrega** passe por um pipeline de qualidade determinístico antes de ser declarada pronta. O Roger não é opcional — é o guardião da qualidade.

### Componentes Principais

| Componente | Localização | Função |
|---|---|---|
| `loop_orchestrator_unified.py` | `scripts/` | Orquestrador principal do test-loop |
| `gen_loop_report_html.py` | `scripts/` | Geração de relatório HTML (primário, <1s) |
| `gen_loop_report_dense.py` | `scripts/` | Geração de relatório PDF (fallback) |
| `loop_handoff.py` | `scripts/` | Handoff dos workers (files_touched, gaps, worker_tail) |
| `roger-router.py` | `profiles/*/scripts/` | Roteamento de crons (pausado) |
| Skills | `skills/` | `roger-test-loop`, `roger-router`, `ai-test-loop`, `test-loop-runner-ops`, `critic-gate` |

### Skills Relacionadas

- **`roger-test-loop`** — Skill principal: regras, comando, pipeline, pitfalls
- **`roger-router`** — Roteamento de crons (v3, pausado)
- **`ai-test-loop`** — Pipeline de avaliação/nota 0-100 (raiz)
- **`test-loop-runner-ops`** — Operação mecânica (config, subprocess, WSL)
- **`critic-gate`** — Critic separado antes de declarar pronto
- **`cerberus-red-team`** — Red Team agêntico (integração 31/08)

---

## 2. Regra Global

**Toda entrega (código, UI, pipeline, feature, fix, infra, config, skill) de QUALQUER projeto/grupo/perfil DEVE passar pelo Roger antes de ser declarada pronta.** Inclusive na DM. "Feito" sem Roger = não entregue (cobrado por Samuel em 02/09/2026).

### Exceções

- **Skills/AGENTS.md/memory/README** — não precisam do loop completo (sem código compilável), mas passam pelo checklist de documentação imediata
- **Projetos sem código** (jellyfin, yurumi, series-guard) — têm `security: false` na config

### Checklist de Documentação Imediata

A cada bloco de trabalho, antes de sair:

- [ ] Memory salva na hora (decisões, pitfalls, descobertas)
- [ ] Skill criada/atualizada no MESMO momento
- [ ] AGENTS.md sincronizado → Yurumi (yurumi-agents-sync.py)
- [ ] Anúncio no Bot Chat de todos os agentes afetados
- [ ] pending.md atualizado (pending-sync.py)

---

## 3. Arquitetura

### Diagrama de Fluxo

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Builder    │ ──→ │   Testes     │ ──→ │   Critic     │
│  (opencode)  │     │ (reais/e2e)  │     │ (separado)   │
└──────────────┘     └──────────────┘     └──────────────┘
       │                     │                    │
       ▼                     ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Reviewer    │ ←── │  Security    │ ←── │  Evidência   │
│  (opencode)  │     │  Gate (CERB) │     │  Visual      │
└──────────────┘     └──────────────┘     └──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│   Gate       │ ──→ │  HTML/PDF    │
│  nota >= thr │     │  Relatório   │
└──────────────┘     └──────────────┘
       │
       ▼
   Entrega ✅
```

### Caminho dos Arquivos

```
scripts/
├── loop_orchestrator_unified.py   ← Orquestrador principal
├── gen_loop_report_html.py        ← HTML generator (stdlib, <1s)
├── gen_loop_report_dense.py       ← PDF generator (ReportLab, fallback)
├── loop_handoff.py                ← Handoff dos workers
├── roger-router.py                ← Roteamento de crons (pausado)
└── sync-all-agents.sh             ← Sync de tudo (memory+skill+AGENTS.md)

config/test-loop/
├── knowledge.md                   ← Base de conhecimento global
├── knowledge-<slug>.md            ← KB por projeto (opcional)
├── <slug>.json                    ← Config por projeto
└── critic_check_<slug>.py         ← Critic determinístico por projeto

profiles/*/roger-router/
├── sentinelas/                    ← Marcadores de entrega (anti-duplicacao)
│   └── <job_id>/
│       └── <chat_id>_<hash>.sent
└── log/
    └── router.log                  ← Log do roteador

output/
├── loop-<slug>-<YYYYMMDD>-<HHMM>-spec.json  ← Payload do relatório
└── loop-<slug>-<YYYYMMDD>-<HHMM>.html       ← Relatório HTML gerado

state/loop/
├── <slug>/
│   └── .loop-state.json          ← Estado do loop (progresso, notas, gaps)
│
└── <projeto>/
    ├── .loop-handoff.json         ← Handoff do worker (files_touched, gaps, worker_tail)
    └── .loop-worker-*.md          ← Arquivos temporários dos workers
```

---

## 4. Pipeline Completo

### 4.1 Builder (opencode CLI)

Executa o opencode CLI para corrigir gaps de programação. O builder **NUNCA avalia o próprio trabalho** — ele apenas produz código.

- **Entrada:** gaps detectados, prompt KB, conhecimento do projeto
- **Saída:** código corrigido, testes, arquivos modificados
- **Parâmetros:** `builder_timeout`, `remote_builder` (offload para Douglas)

### 4.2 Testes Reais

Roda a suíte de testes do projeto:

- **Unitários** — pytest, vitest, etc.
- **E2E** — Playwright, Cypress
- **VRT (Visual Regression Testing)** — comparação visual de screenshots
- **Critic determinístico** — `critic_check_<slug>.py` com evidência real

### 4.3 Critic

Agente determinístico separado (não é o builder) que avalia:

- **Nota 0-100** com evidência real
- **Contexto fresco** — não vê o resultado do builder anterior
- **Critic dedicado** — `critic_check_<slug>.py` por projeto

### 4.4 Reviewer

Segundo opencode que revisa o trabalho do builder:

- **Resposta:** `CONCORDO | DISCORDO | CONCORDO_COM_RESSALVAS`
- **DISCORDO** → reabre o loop (nova rodada)
- **Máximo:** 8 rodadas

### 4.5 Security Gate (Cerberus)

Pipeline de segurança em 2 etapas:

1. **Scanners automáticos** (20 pontos):
   - `gitleaks` — vazamento de secrets
   - `trufflehog` — secrets em commits
   - `bandit` — vulnerabilidades Python
   - `opengrep` — análise SAST

2. **Revisor agêntico OWASP** — analisa cada scanner com contexto OWASP

3. **Cerberus** (`--confirm`) — confirma achados de segurança ativos
   - Integração 31/08: Cerberus é o Red Team agêntico
   - **SECURITY_ISSUE + impacto alto/médio** → reabre o loop

**Projetos sem código** (jellyfin, yurumi, series-guard): `security: false` na config.

### 4.6 Evidência Visual

Obrigatória para mudanças de visual:

- **Screenshots DESKTOP + MOBILE** (viewport configurável por projeto)
- **Vídeo de aprovação** — entregue ao final em mensagem única
- **Ferramentas:** Arachne MCP (screenshot_vision, visual_diff, visual_gates, demo-video-recording)

### 4.7 Gate

Critério de aprovação:

- `threshold` por projeto: UI = 100, código = 85
- **Nota < threshold** → corrigir (máx 8 rodadas)
- **Nota >= threshold** → aprovado

### 4.8 HTML de Entrega

Relatório gerado ao final do pipeline (ver seção 7).

---

## 5. Operação

### 5.1 Comando Principal

```bash
python loop_orchestrator_unified.py --project <slug> --threshold <nota>
```

**Alias:** `roger --project <slug>` (wrapper `~/.local/bin/roger`, criado em 22/10/2026)

**Parâmetros:**
- `--project <slug>` — slug do projeto (arachne, capivara, dogwalk, portifolio, lifelog, tatuengine, leve-lavanda, yurumi, ...)
- `--threshold <nota>` — gate (UI=100, código=85)
- SEMPRE rodar em background + notify quando a suíte demorar (timeout 3600)

### 5.2 Antes de Relançar (após run morto)

```bash
rm -f .loop-state.json .loop-worker-*.md
```

### 5.3 Na DM

A DM não é exceção. Ao concluir trabalho técnico (código, script, config, bridge, infra, skill):

1. **PARAR** e rodar o Roger antes de mandar o resumo final
2. Infra/scripts sem projeto: usar `--project` com slug mais próximo ou criar config test-loop própria
3. NUNCA mandar "feito ✅" sem o Roger ter rodado

### 5.4 Modelos e gate de qualidade (22/10/2026)

| Papel | Principal | Fallback |
|---|---|---|
| Builder opencode | `hermes-9router/rotacao-tokensave` | default da perna |
| Redator / melhorias | `hermes-9router/9router-rotation` | — |
| Painel de reviewers | `rotacao-tokensave` via 9router | `9router-rotation` |
| Security reviewer | `rotacao-tokensave` via 9router | `9router-rotation` |
| VLM / evidência visual | `vision-free` via 9router | `gemini/gemini-3.5-flash-lite` |

O `roger-gate` (`plugins/roger-gate`) usa `pre_verify`: depois de editar código,
o agente não finaliza enquanto não houver state de Roger `done` com score >=
threshold, gerado depois das edições. Kill switch: `ROGER_GATE_ENABLED=0`.

### 5.5 Roger Router (ativo, dedup)

O Roger Router roteava outputs de crons para grupos específicos. **Status: PAUSADO** desde 02/09/2026 — 7 crons foram migrados para deliver nativo com `attach_to_session=true`.

**Para religar:** `cronjob resume b0595dbf870d`

#### Chat IDs dos Grupos

| Grupo | Chat ID |
|---|---|
| Arachne | -1004444788239 |
| Capivara | -1004247157556 |
| DOG_Walk | -1003907121311 |
| Portifólio | -1003963506968 |
| Mídia | -1004404866162 |
| Notificações | -1004315707739 |
| Yurumi | -1003906947487 |
| LEVE LAVANDA | -1003980323020 |
| Ajudante/TATU | -1004402977125 |
| DM Samuel | 6346525016 |

---

## 6. Configuração por Projeto

### 6.1 Estrutura do JSON

`config/test-loop/<slug>.json`:

```json
{
  "path": "//wsl$/kali-linux/home/samuel/projetos/<slug>",
  "threshold": 85,
  "max_rounds": 8,
  "stack": "nextjs",
  "viewports": ["desktop", "mobile"],
  "routes": ["/", "/login", "/dashboard"],
  "remote_builder": false,
  "builder_timeout": 600,
  "parallel": false,
  "workers": 2,
  "reviewers": "panel",
  "handoff_local": true,
  "yurumi_checkpoint": false,
  "security": true
}
```

### 6.2 Campos

| Campo | Descrição | Padrão |
|---|---|---|
| `path` | Caminho do projeto (WSL: `//wsl$/kali-linux/...`) | obrigatório |
| `threshold` | Nota mínima para aprovação | 85 |
| `max_rounds` | Máximo de rodadas | 8 |
| `stack` | Stack tecnológica (nextjs, fastapi, vue) | — |
| `viewports` | Viewports para evidência visual | ["desktop", "mobile"] |
| `routes` | Rotas para screenshot | [] |
| `remote_builder` | Offload para Douglas | false |
| `builder_timeout` | Timeout do builder (s) | 600 |
| `parallel` | Workers paralelos | false |
| `workers` | Número de workers | 2 |
| `reviewers` | "panel" (painel) ou "legacy" | "panel" |
| `handoff_local` | Handoff em disco local | true |
| `yurumi_checkpoint` | Checkpoint Yurumi pós-rodada | false |
| `security` | Rodar security gate | true |

### 6.3 Base de Conhecimento (KB)

- **Global:** `config/test-loop/knowledge.md` — pitfalls conhecidos, regras gerais
- **Por projeto:** `config/test-loop/knowledge-<slug>.md` — específico do projeto

Injetada no prompt do worker/reviewer para evitar repetir erros passados.

### 6.4 GitHub Context

`collect_github_context(cfg)` lê o remote do repo e busca na API GitHub:
- Últimos commits
- PRs abertos
- Última release

Falha silenciosa (não bloqueia o loop). Usado para detectar drift vs upstream.

### 6.5 Regressões

`detect_regressions(st)` compara gaps entre rodadas:
- Gap que fechou (rodada N tinha, N+1 não) → normal
- Gap que REAPARECE (N+2+) → prefixado com `REGRESSÃO: ` e reabre o loop

### 6.6 Melhorias Pós-Convergência

`run_improvements_reviewer(cfg, st)` roda UMA vez quando o loop converge:
- opencode propõe melhorias não vistas (deps, perf, DX, regressão futura)
- NÃO bloqueia a entrega — vai pro relatório seção "Próximo Passo"

---

## 7. HTML de Entrega

### 7.1 Seções Obrigatórias (9)

1. **Veredito** — nota final, threshold, aprovado/reprovado
2. **Divisão de Trabalho** — quem fez o quê
3. **Rodadas e Gaps** — histórico de rodadas, gaps abertos/fechados
4. **Timeline** — linha do tempo da execução
5. **Revisão Independente** — análise do reviewer (opencode)
6. **Arquivos** — arquivos modificados/criados
7. **Como foi feito** — detalhes de implementação
   - 7.1 **Segurança** — gate + revisor OWASP
   - 7.2 **Trabalho Realizado** — cards por rodada com worker, gaps (chips), por-que, arquivos e worker_tail
8. **Estado Final** — resultado dos testes, screenshots
9. **Próximo Passo** — recomendações pós-convergência

### 7.2 Geração

- **Primário:** `gen_loop_report_html.py` (stdlib Python, <1s, mesmo payload JSON)
- **Fallback:** `gen_loop_report_dense.py` (ReportLab, PDF)
- **Payload:** `loop-<slug>-<YYYYMMDD>-<HHMM>-spec.json` gerado por `build_dense_payload()`

### 7.3 Seção 7.2 — Trabalho Realizado

Campos obrigatórios no relatório:

| Campo | Descrição |
|---|---|
| **Worker** | Quem executou (builder, critic, reviewer) |
| **Gaps** | Gaps encontrados (chips coloridos) |
| **Por que** | Motivação da mudança |
| **Como** | Abordagem técnica |
| **Arquivos** | Arquivos tocados |
| **Worker Tail** | Saída do worker (pre colapsado, 1200 chars) |

### 7.4 Encoding

**Problema resolvido (02/09):** Mojibake de UTF-8 capturado como cp1252 no Windows.

**Fixes:**
1. `run()` no orquestrador: `encoding="utf-8"` explícito no `subprocess.run` — previne corrupção em loops novos
2. `fix_mojibake()` no `esc()` do HTML generator — repara dados já corrompidos (loop de 2+ iterações cp1252→utf8)
3. Em-dash `—` e checkmark `✓` convertidos para entidades numéricas HTML (`&#8212;`, `&#10003;`)

### 7.5 Mobile

CSS responsivo integrado:
- Breakpoint: 700px
- Tabelas com scroll horizontal
- KPIs em 2 colunas
- Fontes adaptativas (13px mobile)
- Cards compactos, pre colapsado a 200px

---

## 8. Roger Router

### 8.1 O Que É

Script que roteia outputs de crons para grupos específicos do Telegram, com anti-duplicação de 3 camadas.

### 8.2 Status

**PAUSADO** desde 02/09/2026. Todos os 7 crons de domínio único foram migrados para deliver nativo com `attach_to_session=true` (mensagens reativas — Samuel responde e o agente age no grupo).

### 8.3 Anti-Duplicação (3 Camadas)

1. **Só o output mais recente** por job (mtime, MAX_AGE_S=900)
2. **Output vazio = watchdog pattern** → silêncio
3. **Sentinela por destino** — cada job/chat_id/hash recebe UMA vez

### 8.4 Causa Raiz do Flood (Fixada 02/09)

O Roger hasheava o **arquivo bruto** (incluía `Run Time: HH:MM:SS` que muda a cada execução) → hash nunca repetia → sentinela nunca travava → **toda execução virava mensagem nova** (400-1000/dia). **Fix:** hash calculado sobre o **body** (conteúdo útil) e skip quando body vazio.

### 8.5 Integração com Yurumi

- **Rotas dinâmicas:** `routes.json` com overrides + Yurumi (kind=roger_route_override)
- **Audit trail:** cada roteamento POST no Yurumi (kind=roger_audit)
- **Route map:** Roger publica mapa efetivo no Yurumi (kind=roger_routes)

---

## 9. Cerberus — Red Team

### 9.1 O Que É

Cerberus é o Red Team agêntico que executa varreduras ofensivas de segurança. Integrado ao pipeline do Roger via `--confirm`.

### 9.2 Funcionamento

1. **Scanners automáticos** (gitleaks, trufflehog, bandit, opengrep)
2. **Revisor OWASP** — analisa cada scanner com contexto OWASP
3. **Cerberus `--confirm`** — confirma achados ativos (não falsos positivos)
4. **SECURITY_ISSUE + impacto alto/médio** → reabre o loop

### 9.3 Agendamento

- **Cron semanal:** domingo 05:00 (`31dd96ade6ee`)
- Skill: `cerberus-red-team`

---

## 10. Crons Integrados

| Cron | ID | Schedule | Descrição |
|---|---|---|---|
| CronFix Loop | 0474aee0de50 | a cada 15min | Corrige crons com erro via opencode |
| Launcher Arachne | 1dfe78787b3c | 03:30 | Loop de madrugada do Arachne |
| Cerberus | 31dd96ade6ee | dom 05:00 | Red Team semanal |
| loop-arachne-watchdog | 80ce6e854897 | — | Watchdog Arachne (deliver nativo) |
| bench-portifolio-watchdog | 23d5ccaf86a1 | — | Watchdog Portifólio (deliver nativo) |
| loop-yurumi-watchdog | b2c2923814ce | — | Watchdog Yurumi (deliver nativo) |
| tatu-fase4-watchdog | 4891e511afd5 | — | Watchdog Tatu (deliver nativo) |
| tatu-fase4-progress | 11b35d82aca4 | 60min | Progresso Tatu (deliver nativo) |
| capivara-backup-r2 | ba474c8c6a9b | — | Backup Capivara R2 (deliver nativo) |
| Capivara D1 Sync | b6c2a9463e57 | — | Sync D1 Capivara (deliver nativo) |
| yurumi-agents-sync | 6dafc77f50fc | — | Sync agentes Yurumi (deliver nativo) |

---

## 11. Pitfalls & Troubleshooting

### 11.0 Diagnóstico das 3 classes de `n/d` (revisor) — medidas

`n/d` no painel NÃO é "código ruim": é leitura de AMBIENTE. Separar em um olhar
por `output/loop-reviewers.log`:

| Assinatura | Classe | Cura |
|---|---|---|
| `<15 s` por tentativa + `401`/`429` | **QUOTA** | trocar modelo (`9ROUTER_MODEL`) / usar combo `9router-rotation` |
| `len` na casa dos milhares + `motivo=sem-json` | **TRUNCAMENTO** | teto de tokens do revisor (`9ROUTER_REVIEWER_MAX_TOKENS`, default 3000) |
| `len` baixo + `motivo=nao-json` + corpo `<tool`/`pwd` | **AGÊNTICO** | `_looks_toolcall()` → cai para o próximo modelo da cadeia |

Ausência de veredito nunca é aprovação: `run_debate` devolve
`revisor_indisponivel` e o gap vira `REVISOR_INDISPONIVEL (ambiente)`.

### 11.0b Nota HERDADA/RECICLADA — falso verde de gate (medido 24/10/2026)

Duas camadas, as duas fechadas:

1. **Orquestrador**: `_critic_fp()` é hash só do fonte do critic. Arquivo
   novo/editado no `path` do projeto não mudava o fingerprint e a rodada era
   reaproveitada (`nota já computada — reaproveitando (mesma rubrica)`).
   Cura: `_newest_code_mtime()` + guard **anti-nota-herdada por edição**
   (código mais novo que o `ts` da rodada ⇒ `prev = None`, critic roda de novo;
   log `logs/anti-nota-herdada-edicao.log`).
2. **Plugin do gate** (`roger-gate`): `_state_info` media frescura pelo
   **mtime do state file**, que é reescrito a cada rodada/entrega. Cura:
   `_ts_after()` — exige **`ts` da rodada > `edit_mtime`**; state legado sem
   `ts` mantém o comportamento antigo. Testes:
   `scripts/tests/test_roger_gate_frescura.py`.

Caso real: rodada de `01:56Z` reciclada às `10:20` para arquivo criado `10:06`
→ `RESULT: DONE (nota 100)` sem avaliar o arquivo. Plano completo:
`plans/2026-10-24-gate-roger-nota-reciclada.md`.

### 11.1 Pitfalls Conhecidos

| Problema | Causa | Solução |
|---|---|---|
| **RAM < 1GB** | Swap-thrash | Adiar o loop (critic nota 0 falsa) |
| **WSL freeze** | wsl.exe órfãos | Matar com `kill-wsl-orphans.ps1` antes do loop |
| **Critic timeout** | Timeout < suíte | Usar timeout 2400+ |
| **0 passed no pytest** | Path UNC ou WSL congestionado | Não re-aplicar fix de path |
| **Run morto** | State stale | Limpar `.loop-state.json` + `.loop-worker-*.md` |
| **opencode órfão** | Esgota RAM | `taskkill /F` em cada órfão antes de relançar |
| **UNC //wsl$ instável** | Carga no WSL | Retry + fallback local (state/loop/) |
| **Não spawnar WSL** | Concorrência | Não rodar loop enquanto outro loop WSL roda |
| **Security reviewer** | Ecoa template | Parser fail-closed (eco = n/d, não fail-open) |
| **Mojibake** | cp1252 sem encoding | `encoding="utf-8"` no subprocess.run |
| **Dupla corrupção** | cp1252 em cp1252 | `fix_mojibake()` em loop até estabilizar |
| **Path.home() SYSTEM** | Gateway SYSTEM | Nunca usar Path.home() — derivar de __file__ |

### 11.2 Troubleshooting

**Roger não roda:**
```bash
# Verificar state stale
rm -f .loop-state.json .loop-worker-*.md

# Verificar WSL
wsl --shutdown  # só se necessário
```

**Relatório HTML não abre no celular:**
- Verificar encoding: o HTML tem `charset=utf-8` no `<meta>`
- Verificar CSS mobile: breakpoint 700px, scroll horizontal em tabelas
- Verificar mojibake: `fix_mojibake()` em todas as strings de dados

**Cron não entrega:**
- Verificar se o bot está no grupo
- Verificar deliver config: `telegram:<chat_id>`
- Verificar `attach_to_session=true` para mensagens reativas

**Roger router não roteia:**
- Verificar sentinelas: `rm -rf sentinelas/<job_id>/`
- Verificar log: `tail -50 roger-router/log/router.log`
- Forçar reenvio: `python roger-router.py --force`

---

## 11.3 Atestado de qualidade do próprio Roger (24/10/2026)

O Roger avalia entrega de TODOS os projetos, então defeito nele é defeito em
tudo. `scripts/roger-qualidade.py` mede o Roger contra ele mesmo, por dimensão,
com prova por comando, e emite JSON (`state/roger-qualidade-<ts>.json`) + MD
(`logs/roger-qualidade-<ts>.md`).

| Dim. | Peso | O que mede |
|---|---|---|
| D1 | 30 | pytest dos 8 suites do Roger (contratos determinísticos) |
| D2 | 15 | `loop_orchestrator_unified.py --project X --selftest` |
| D3 | 20 | `e2e_roger_v3.py` (motor completo, 29 testes) |
| D4 | 10 | `roger-nopaid-guard.py --dry-run` silencioso (regra: nunca pago) |
| D5 | 15 | operação viva: gateway, cron do guard, state fresco, sem campanha presa |
| D6 | 10 | `docs/ROGER.md` + skill `roger-test-loop` |

Veredito: **APTO** (todas PASS e ≥90% do teto) · **COM RESSALVAS** (sem FAIL,
≥70%) · **REPROVADO** (qualquer FAIL). `--fast` pula D2/D3. Exit 0/1/2.
Cron semanal `0b4e6b541f67` (segunda 06:00, `roger-qualidade-cron.sh`,
Notificações) fala **só quando não é APTO** — regressão de qualidade vira alerta
sem spam.

**Primeira medida (24/10, antes da cura): REPROVADO 89/100** — D1 e D4 caíram
pelo mesmo motivo: os combos `rotacao-tokensave` (builder dos 45 configs) e
`vision-free` tinham pernas PAGAS (`gemini/*`, `opencode-go/*`). Com a decisão
do dono ("remover"), `scripts/9r-combo-prune-paid.py --apply` limpou as pernas
por `PUT /api/combos/<id>` (backup `~/.9router/combos-backup-<ts>.json` +
`data.sqlite.bak-prune-paid-<ts>`) → **APTO 100/100**.

Defeitos de harness corrigidos no caminho (todos medidos):
- `tests/test_cerberus_ram_gate.py` media `ram_gate()`, removida do wrapper em
  18/10 → estourava **na coleta** e derrubava a suíte inteira. Virou SKIP
  explícito (backup `.bak-20261024-apiremovida`). Teste de API morta deve ser
  skip, nunca erro de coleta.
- `e2e_roger_v3.py` teste 22 exigia a 2ª cópia do watchdog em
  `profiles/gateway_telegram/scripts` (layout Windows). No Linux o gateway roda
  da raiz e nenhum cron consome aquela cópia → exigência virou condicional
  ("se existir cópia, tem de ser idêntica"). 29/29.

## 12. Histórico de Versões

| Data | Versão | Mudanças |
|---|---|---|
| 02/09/2026 | 2.0.0 | Migração PDF→HTML; `fix_mojibake()`; Roger v2 (KB, GitHub context, regressões, melhorias); HTML enriquecido com seção 7.2 |
| 31/08/2026 | 1.4.0 | Integração Cerberus (Red Team); `--confirm` no security gate |
| 30/08/2026 | 1.3.0 | Nome batizado como "Roger"; refatoração do orchestrator |
| 25/08/2026 | 1.2.0 | v2 do test-loop: critic separado, reviewer, security gate, evidência visual |
| 18/08/2026 | 1.1.0 | Regra global: toda entrega passa pelo test-loop |
| 15/08/2026 | 1.0.0 | Primeira versão do test-loop com opencode builder |

---

> **Roger 🐐** — "Feito" sem Roger = não entregue.
> Documentação mantida por Hermes Agent · Atualizada em 02/09/2026