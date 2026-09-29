# Seq2Seq com attention para o dataset MIMO

- Date: 2026-09-28
- Status: done
- Type: model

## Goal

Treinar um modelo de deep learning (encoder-decoder LSTM com
dot-attention) para prever `y1..y3` (vazão de distribuição, nível,
pressão) do reservatório WRSO, como alternativa ao ARMAX/MOGWO
reproduzido em `notebooks/modeling/01_mogwo_armax_reproducao.ipynb`.

## Approach

- Porte multivariado de `data/external/seq2seq attention.ipynb`
  (originalmente univariado, com dados sintéticos) para os splits
  reais do artigo: mesmos parquets de
  `mimo_dataset_sample_from_papper_{train,test}.parquet`, com `u1..u4`
  + `y1..y3` na janela de entrada e `y1..y3` na janela de saída.
- `notebooks/training/01_seq2seq_attention.ipynb`: pré-processamento
  básico (sem `TimeSeriesPreprocessor` — sem detrend/EWT/differencing),
  só `MinMaxScaler` fitado no treino + janelas deslizantes por batch
  (`INPUT_WINDOW=24`, `OUTPUT_WINDOW=12`), split treino/validação
  85/15, `LightningModule` com `torchmetrics` (MSE/MAE) por estágio,
  `EarlyStopping` + `ModelCheckpoint(monitor="val_mae")`,
  `mlflow.pytorch.autolog()`. Avaliação e métricas finais desfazem a
  normalização apenas nas colunas de saída.
- Plots iguais à referência (`data/external/seq2seq attention.ipynb`):
  forecast de uma janela de teste, forecast do teste completo
  (janelas não sobrepostas encadeadas, como `plot_full_test_prediction`)
  e pesos de attention (heatmap + barras por passo de saída, como
  `plot_attention`).
- Adicionado grupo opcional `ml` em `pyproject.toml`
  (`uv add --optional ml torch lightning torchmetrics mlflow`).

## Data / model impact

- MLflow experiment: `mimo_sys_seq2seq_attention` (ver MLflow UI para
  o run_id).
- Métricas via `torchmetrics` (`MeanSquaredError`, `MeanAbsoluteError`,
  `R2Score`), teste, escala original, `SEED=42`, `MinMaxScaler`:
  - vazão de distribuição: MSE=85.93, MAE=6.98, R²=0.868
  - nível: MSE=0.034, MAE=0.139, R²=0.371
  - pressão: MSE=3.88, MAE=1.49, R²=0.724
- Comparação: pior que o ARMAX/MOGWO publicado (MSE 33.5 / 0.0024 /
  2.32 nas mesmas 3 saídas) — esperado para uma primeira passada sem
  tuning de hiperparâmetros; serve de baseline de deep learning para
  iteração futura.
- Artefato: `models/seq2seq_attention_mimo.pth` (`state_dict`).
