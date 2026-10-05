# Seq2Seq com TimeSeriesPreprocessor

- Date: 2026-10-05
- Status: done
- Type: model

## Goal

Medir se o `TimeSeriesPreprocessor` melhora o Seq2Seq com attention no
split 70/15/15, avaliando na escala original.

## Approach

- `notebooks/training/03_seq2seq_attention_preprocessor.ipynb`: quatro
  variantes (normalização; outliers + filtro; EWT/detrend ajustado no
  treino; EWT/detrend ajustado na série toda, com vazamento) mais um
  baseline de persistência. MLflow experiment
  `mimo_sys_seq2seq_attention_preprocessor`.
- `src/mimo_sys/preprocessors/timeseries.py`: corrige `transform` com
  EWT (`_ewt_with_boundaries`), que quebrava com o `ewtpy` instalado.
  Teste em `tests/test_preprocessors.py`.

## Data / model impact

- Resultado: o preprocessor não melhorou o modelo; EWT/detrend piora
  fora da amostra. Tabela e ressalvas em
  `docs/2_split_artigo_vs_dataset_completo.md`, seção 8.
- Dependências: o notebook usa o extra `ml` e o grupo `plot`
  (`uv run --extra ml --group plot ...`).
