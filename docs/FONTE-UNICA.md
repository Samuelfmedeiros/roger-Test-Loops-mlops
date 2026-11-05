# ROGER — Fonte única (leia antes de editar qualquer coisa)

## Regra de ouro

**O código do Roger é UM só.** Ele vive em `~/.hermes/scripts` e é espelhado nos perfis
por `scripts/roger-mirror-sync.py` (cron diário 06:20). Editar
`~/.hermes/profiles/<perfil>/scripts/roger*.py` cria uma segunda verdade: o espelho
sobrescreve na próxima sincronização (o backup fica em `~/.hermes/backups/mirror-<ts>/`).

**Config por grupo é o lugar do ajuste fino**, não o código:
`~/.hermes/config/test-loop/<projeto>.json` (128 configs).

## Por que essa regra existe (medido em 24/10/2026)

| Sintoma | Medida |
|---|---|
| `loop_orchestrator_unified.py` (coração do loop) em 3 versões | raiz, ajudante, arachne |
| `roger-nopaid-guard.py` / `roger-gate-retry.py` velhos | `gateway_telegram` divergia da raiz |
| Clones mortos | ajudante 34 arquivos / 0 crons; arachne 9 / 1 |
| Total divergente antes da cura | **35 arquivos em 3 perfis** |
| Depois do espelho | **0 divergentes** |

Mesmo código, nota diferente e comportamento diferente dependendo de quem rodava.

## Estrutura deste repo

```
engine/ router/ gates/ pilots/     # motor do Roger v3 (já existia)
scripts/                           # utilitários de router/vault/wsl (já existia)
scripts/hermes/                    # SNAPSHOT do Roger que roda em ~/.hermes/scripts
  _manifest.json                   # hash de cada arquivo — base do check-drift
  check-drift.py                   # falha se o snapshot não for igual ao Roger vivo
  tests/                           # contratos (pytest)
.github/workflows/roger-ci.yml     # contratos + drift + atestado (runner self-hosted)
docs/                              # ARQUITETURA, GATES, CONFIG, OPERACAO, RUNBOOK-FALHAS
variants/tatuengine/               # variante isolada do TatuEngine (pin próprio)
```

## Fases da convergência

1. **Feito (24/10)**: espelho raiz→perfis; divergência zerada; snapshot + CI + docs.
2. **Agora**: PR obrigatório; CI roda contratos + drift + atestado (`roger-qualidade.py`
   precisa dar **APTO**); `CHANGELOG` por mudança de comportamento.
3. **Próximo**: repo vira a fonte (symlink de `~/.hermes/scripts/roger*` para cá) e o
   `check-drift` inverte a direção. Requer teste de symlink × systemd (`realpath`,
   `NRestarts`) — pitfall já catalogado.
4. **TatuEngine**: variante própria em `variants/tatuengine/`, versionada à parte.

## Antes de commitar

```bash
python3 scripts/hermes/check-drift.py --sync   # traz o Roger vivo para o snapshot
python3 -m pytest scripts/hermes/tests -q      # contratos
python3 ~/.hermes/scripts/roger-qualidade.py   # tem que dar APTO
git add -A && git commit -m "roger: <o que mudou>"
```

Mudança de comportamento exige: `CHANGELOG` + critério de rollback no corpo do commit.
