# Collection Models — experimento MLflow 70/15/15

- Date: 2026-10-09
- Status: done
- Type: model

## Goal

Reunir os modelos num experimento MLflow navegável, um run pai com uma
child run por modelo, cada uma com as métricas, o plot temporal, o plot
no domínio da frequência e o mapa de attention sobre os lags.

## Approach

- `notebooks/training/collection_common.py`: métricas, bandas
  espectrais, plots e logging comuns — para não duplicar a avaliação se
  surgir uma variante futura.
- `notebooks/training/05_collection_models_v1.py`, experimento
  `mimo_sys_collection_70_15_15`. Run pai `collection_70_15_15` + 7
  children: ARMAX 1 passo, ARMAX 12 passos, Seq2SeqAttention, MPNN,
  SymbolicGraphNetwork, StemGNN, Seq2SeqLatentGNN. Carrega os pesos de
  `models/`; **não treina**.

### Correções em relação à primeira versão

A primeira versão deste experimento não dava para ver nada na UI, por
três motivos — todos corrigidos:

1. **Figuras em HTML.** A UI do MLflow pré-visualiza PNG mas serve HTML
   como texto cru/download, então os gráficos eram inúteis pela
   interface. Agora vão como PNG via `mlflow.log_figure` (dependência
   nova: `kaleido`, no grupo `plot`), com o HTML interativo junto em
   `interativo/` para quem quiser zoom/hover.
2. **Nada no run pai.** Era preciso abrir cada child para comparar.
   Agora o pai concentra os heatmaps de resumo: R² modelo × saída e
   erro de energia por banda, um por saída.
3. **Nomes de banda inválidos.** `tendencia_>32h` fazia o
   `mlflow.log_metric` levantar `MlflowException` — a UI rejeita `>` e
   `+` em nome de métrica. Renomeado para `tendencia_acima_32h`.

`log_figure` também tolera falha de render do kaleido (`TimeoutError`
esporádico em lote): registra a tag `png_falhou__*` e segue, em vez de
derrubar o run.

### Variante k-fold (tentada e descartada)

Chegou a existir `06_collection_models_kfold.py`, treinando 4
arquiteturas × 5 folds de `TimeSeriesSplit` do zero (protocolo do
esquema 2 de `02_seq2seq_attention_splits_vs_armax.ipynb`). Rodou com
sucesso (25/25 runs `FINISHED`), mas o fold 0 (338 amostras, ~14 dias)
é instável em todas as arquiteturas — StemGNN deu R² −1,04 no fold 0
contra ~0,55 nos folds 2–4 — e isso puxa a média geral para baixo sem
separar "modelo fraco" de "pouco histórico". Decidido manter só o
70/15/15; o script, o experimento MLflow
(`mimo_sys_collection_kfold_ts`) e os artefatos foram removidos/
arquivados.

## Data / model impact

- Nenhum peso escrito; avaliação derivada de `models/*_70_15_15.pth`.
- R² médio no teste: ARMAX 1 passo 0,826 (referência otimista, usa `y`
  medido nos lags), Seq2SeqAttention 0,655, SymbolicGraphNetwork e
  StemGNN 0,623, MPNN 0,608, Seq2SeqLatentGNN 0,604, ARMAX 12 passos
  0,333.
- **A attention do Seq2SeqAttention é uniforme**: entropia relativa
  0,999 (máximo 1,0 para 24 lags), pesos entre 0,038 e 0,046 — o modelo
  não seleciona lag nenhum, e o heatmap é chapado por isso, não por
  bug. `Seq2SeqLatentGNN` concentra de fato (0,954), com peso crescendo
  nos lags recentes (t-2 ≈ 0,094 contra ~0,033 nos distantes) e os
  primeiros passos de saída atendendo a t-1/t-2. Registrado como
  métrica (`attention_entropia_rel`) para a diferença ficar visível na
  UI, não só no gráfico.
- Só Seq2SeqAttention e Seq2SeqLatentGNN têm attention sobre **lags**.
  StemGNN (`last_adjacency`) e MPNN/SymbolicGraphNetwork
  (`edge_strength`) expõem peso entre **nós** (7×7) — outro eixo,
  plotado à parte como `attention_nos`, não misturado com o de lags.
  `Seq2SeqLatentGNN` calcula os pesos mas os descarta no `forward`; o
  mapa é recuperado com forward hooks no encoder/decoder, e as linhas
  somam 1,0, confirmando que é a mesma distribuição usada internamente.

## Done when

- [x] `uv run ruff check` / `ruff format --check` nos arquivos
- [x] Script roda: 1 parent + 7 children, 23 PNGs
- [x] `uv run --extra ml pytest tests/` (38 passed)
- [x] `CHANGELOG.md` atualizado em `[Unreleased]`

## Como rodar e abrir

```bash
cd notebooks/training
uv run --extra ml --group plot python 05_collection_models_v1.py
uv run --extra ml mlflow ui --backend-store-uri sqlite:///mlflow.db
```

O `cd` é necessário: `sqlite:///mlflow.db` é relativo e os caminhos de
artefato no banco apontam para `notebooks/training/mlruns/`. Na UI, as
figuras ficam na aba **Artifacts** de cada run — os resumos no run pai,
o detalhe por modelo nas children. Para explorar as métricas como
DataFrame sem abrir a UI, ver
`notebooks/eda/04_collection_models_eda.ipynb`
(`docs/changes/2026-10-09-eda-collection-models.md`).
