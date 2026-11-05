# Roger — test-loop com crítico que prova o que afirma

Edição **pública (vitrine)**, gerada automaticamente do repositório privado a partir da
versão `v3.1.0-4-g1768166`. Publicada em 2026-11-05T02:32Z.

Este repositório não contém o engine em produção: ele mostra o **design** do Roger e a
documentação do método. O código executável é privado.

## O que o Roger faz

- **Test-loop com gate de nota**: o builder não avalia o próprio trabalho; um crítico
  separado devolve SCORE / GAPS / DETAIL e a entrega só fecha acima do limiar.
- **Crítico multimodal** (v3.1.0): imagem por VLM, áudio por transcrição local,
  PDF por extração de texto e vídeo por quadros — cada modalidade com regra própria.
- **Sem mídia não é aprovação**: quando não há o que verificar, o veredito é
  `verificavel=false` com motivo, nunca um "ok" por omissão.
- **Fail-open honesto**: exceção no crítico vira resultado não verificável, não derruba
  a campanha nem inventa nota.
- **Roteamento por domínio**: regra que nasce na conversa vira regra do grupo certo.

## Documentação

- [ARCHITECTURE_V3.md](ARCHITECTURE_V3.md) — desenho do pipeline e dos gates
- [CHANGELOG.md](CHANGELOG.md) — versões
- [docs/](docs/) — notas de operação

## Versão

`v3.1.0-4-g1768166` — publicada após suíte verde e atestado APTO do próprio Roger.
