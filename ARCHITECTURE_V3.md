# Roger V3: Arquitetura, Estado de Produção e Histórico de Infraestrutura

> Registro vivo. Atualizado em 2026-09-29, pós Write real em produção.
> Pacote: `~/projetos/roger-v3` (WSL) | Plugin: `roger-inbound` (Windows).

## 1. Visão geral da arquitetura

Motor autônomo de agentes + memória procedural em Python puro, tipagem
estrita via Pydantic, **sem** LangChain/LangGraph (decisão registrada: o
stack existente já tem Pydantic v2 e o LangGraph seria adição de stack sem
ganho nesta escala).

**Componentes centrais:**

| Componente | Papel | Onde vive |
|---|---|---|
| `roger-inbound` | Hook `pre_gateway_dispatch`; captura a DM do operador e delega ao pipeline | Windows (plugin Hermes) |
| `wsl_bridge.py` | Ponte stdin/stdout → pipeline; usa REST do Qdrant via urllib | WSL |
| `router/` | Classificador (LLM + Pydantic, threshold 0.85), memória com versionamento, pipeline | `~/projetos/roger-v3` |
| `engine.py` + `schemas.py` | Contratos: `RogerDraftState`, `RogerCriticDecision`, `FinalSoulEnvelope` | idem |
| `gates/registry.py` | 10 Held-out Gates declarados (ARACHNE…MARVIN) | idem |
| `vault.py` | Cofre Capivara: resolução por referência, JWT efêmero, scrub | idem |
| `tatu.py` | Render + `sendMessage` do envelope na DM | idem |
| `yurumi_procedural` | Memória procedural no Qdrant (bge 384 + colibri 768) | Qdrant :6333 |

**Regra de Ouro:** o LLM pode *sugerir* que terminou; só evidência externa
(Held-out Gate + Judge) declara `SUCCESS`.

## 2. Decisões críticas de infraestrutura (WSL2 / host)

**Restrição de hardware:** host com **15.7 GB** de RAM física (não 20 GB —
medido em 2026-09-29).

**`.wslconfig` INTOCÁVEL:** `memory=9GB`, `swap=16GB`.
O arquivo carrega o registro de **12 BSODs** (bugcheck 0x1a/0x3b) causados
por teto de RAM excessivo: com 15 GB de teto e 15.7 GB físicos, o Windows
perde a folga de commit e o agendador falha páginas. **Não elevar a RAM sem
medir o host livre primeiro.**

**Boot storm (incidente de 2026-09-29).** O WSL reiniciava em ciclo
(`load average` chegando a 64) porque a soma dos serviços estourava o teto
de 8.9 GB no boot. Diagnóstico eBak:
- Raio-X Docker + `ps` revelaram o gargalo real: **pipeline de mídia
  (Tdarr/exiftool)**, `clamscan` (62% CPU), `k3s-server`, `jellyfin` e
  `llama-server`/`ollama` ociosos.
- Contenção: Tdarr/exiftool mortos, clamscan encerrado (origem: timer
  `clamav-freshclam-once` + cron `security-scan-weekly.sh` — **exclusões
  pendentes**), jellyfin morto, k3s parado + `systemctl disable`, Ollama
  morto, `docker system prune -f`.
- **Qdrant nunca foi o culpado** (32% CPU / 1.6 GB = uso saudável).
- Resultado: load 64 → 2.0–2.8 estáveis, 12/12 amostras sem reset.
- `k3s-server` roda como `root` → exige `sudo` para matar/mascarar.

## 3. Estado do pipeline (2026-09-29)

**Rota de entrada: PROVADA.** Tráfego real capturado em
`pre_gateway_dispatch` (`~/.hermes/roger-v3/inbound_shadow.jsonl` no
**Windows** — não no WSL; erro de leitura já cometido uma vez).

**Bridge Qdrant: CORRIGIDO.** Qdrant 1.19 rejeita
`points: [{"id": ...}]` em `set_payload`; exige IDs soltos
(`points: ["uuid"]`). Ergo o `RestMemory.set_payload` envia IDs diretos.
Achado só pelo teste real — a teoria aceitaria ambos.

**Versionamento atômico: PROVADO em produção.** Write real
(`Mídia, a partir de agora adicione sarcasmo`) → classificador LLM
(0.99) → vetorização (bge+colibri) → v1 depreciada, v2 `active`, com
`supersedes` linkado.

**Modo inline: ARMADO** (`ROGER_INLINE=1`), aguardando o poller renascer
com o código novo (drenagem de `/restart` em curso; PID antigo ainda
carrega a versão shadow em memória).

**Limiares calibrados (dados vivos, não chutes):**
- Hedge no denso colibri-768: **0.55 global**, **0.45 com pre-filtro
  `soul_target`** (o 0.65 original punia match legítimo: 0.50 vs 0.29).
- Classificador: threshold **0.85**, fail-closed para `AMBIGUOUS` em erro
  de parse (nunca exceção para cima).

## 4. Débito técnico / próximos passos

1. **`scripts/roger-handoff.py` — receptor cego.** O receptor recebe só
   slug/nota/rodada e depende de consultas externas no host para entender o
   escopo. Enriquecer o payload com `branch`, `diff --stat` e o caminho do
   arquivo de estado: transforma o handoff de aviso de passagem em
   transferência de contexto rico, blindando o merge humano contra
   dependência de execução externa. *(Backlog: entra assim que o
   relé/relay voltar a operar com o host estável.)*
2. **Exclusões do ClamAV** — o scan semanal volta a varrer sem exclusões
   para `/mnt/c` e diretórios de build (origem: timer
   `clamav-freshclam-once` + cron `security-scan-weekly.sh`).
3. **Motores das outras SOULs** — só ARACHNE executa live; as demais
   escalam honestamente (`ESCALATED_TO_DM`), nada é forjado.
4. **Gates sem prova ao vivo** — CAPIVARA (EXPLAIN), LAVANDA (inventário),
   PORTIFOLIO (LHCI), MARVIN (ffprobe): declarados + unit-tested, sem
   comando real do host.
5. **k3s/jellyfin/Tdarr e o bloco de mídia travados** (`restart=no`) —
   religar com supervisão quando o host respirar; hoje só
   `infinity-bge` e `infinity-colibri` sobem no boot.
6. **Poller com código shadow em memória** — o `/restart` precisa
   concluir o drain para o `roger-inbound` carregar a versão inline.
