# EDA reutilizável para os experimentos `collection_models`

- Date: 2026-10-09
- Status: done
- Type: analysis

## Goal

Dar um jeito de comparar os modelos do MLflow como DataFrame, sem abrir
run por run na UI — e deixar reutilizável para qualquer experimento
`collection_*` futuro, não só o 70/15/15.

## Approach

- `notebooks/eda/04_collection_models_eda.ipynb`: `mlflow.search_runs`
  contra `EXPERIMENT_NAME` (variável no topo, hoje
  `mimo_sys_collection_70_15_15`), filtra as children (tem
  `parentRunId`), usa o nome do run como índice.
- Tabelas: R²/SMAPE por saída lado a lado (`metric_table`), resumo
  `r2_medio`/`smape_medio`/`mse_medio`, e a reconstrução em formato
  longo das colunas `freq__<saida>__<banda>__<metrica>` em
  `band_pivot(saida)` — erro de energia por banda, ordenado de
  tendência a ruído.
- Attention: tabela com as tags `attention_lags`/`attention_nos` e as
  métricas `attention_entropia_rel`/`attention_lag_recente_pct`.
- Figuras: `show_artifact(modelo, nome)` baixa um PNG do run via
  `MlflowClient.download_artifacts` e mostra com `IPython.display.Image`
  — não precisa abrir a UI para ver `serie.png`/`espectro.png`/
  `attention_lags.png`. O resumo do run pai (`resumo_r2.png`) também.
- Só lê métricas e artefatos já registrados — não reprocessa pesos nem
  refaz forecast.

### Limpeza no MLflow

O experimento `mimo_sys_collection_70_15_15` tinha 2 tentativas
`FAILED` (do bug do `>`/`+` em nome de métrica, já corrigido) ainda
ativas, que poluiriam qualquer `search_runs`. Marcadas como
`lifecycle_stage=deleted` via SQLite direto — hoje o experimento tem
só o run pai + 7 children `FINISHED`.

## Data / model impact

- Nenhum dado ou peso novo; leitura do MLflow existente
  (`mlflow_sys_collection_70_15_15`).
- Confirma os números já registrados em
  `docs/changes/2026-10-09-collection-models-v1.md`: R² médio
  SymbolicGraphNetwork/ARMAX 1 passo 0,915 em `vazao_distribuicao`,
  attention do Seq2SeqAttention quase uniforme (entropia relativa
  0,999) contra 0,954 do Seq2SeqLatentGNN.

## Done when

- [x] `uv run ruff check` / `ruff format --check` no notebook
- [x] Notebook roda de ponta a ponta (`jupyter nbconvert --execute`)
- [x] Experimento MLflow limpo de runs `FAILED`
- [x] `CHANGELOG.md` atualizado em `[Unreleased]`
