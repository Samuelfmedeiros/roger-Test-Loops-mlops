# Changelog — Roger

Formato: versão semântica. Cada release tem tag `vX.Y.Z` e entrada aqui.

## v3.1.0 — 2026-11-04
- **Crítico multimodal** (`scripts/critic_multimodal.py`): imagem (VLM grátis), áudio
  (faster-whisper local), PDF (pdftotext) e vídeo (ffmpeg → 3 quadros → VLM).
- Contrato do crítico: sem mídia = `verificavel=false` (nunca aprovação por omissão);
  fail-open em exceção; `--path` inexistente não cai para pastas default.
- 7 testes novos em `tests/test_critic_multimodal.py`.
- Correção: `critic_visual.julga()` devolve `"ok"` no sucesso — antes era lido como erro
  e zerava a nota do crítico visual.
- Docs: `docs/ROGER.md` §4.7.

## v3.0.0 — 2026-11-04
- Crítico visual, ponte Arachne, guard de fonte única, hook de pre-push com `--fast`.
