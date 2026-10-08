# EDA de séries temporais — tempo, frequência e identificação de sistemas

- Date: 2026-10-07
- Status: done
- Type: eda

## Goal

Explorar formalmente as propriedades de série temporal da série MIMO
(reservatório): tendência/decomposição, estacionariedade, autocorrelação,
causalidade de Granger, cointegração, DTW/clustering, shapelets, domínio da
frequência e identificação de sistemas, reaproveitando o
`TimeSeriesPreprocessor`.

## Approach

- `notebooks/eda/03_eda_series_temporais.ipynb`, carregando
  `data/processed/mimo_data.parquet` e imputando os poucos `NaN` (8 por
  coluna) antes das análises.
- Usa `TimeSeriesPreprocessor` (`src/mimo_sys/preprocessors/timeseries.py`)
  para a decomposição EWT/tendência, comparada com `STL` (`statsmodels`).
- Estacionariedade (ADF), ACF/PACF, causalidade de Granger e cointegração
  (Engle-Granger + Johansen) via `statsmodels`.
- DTW e clustering hierárquico das 7 features via `dtaidistance` +
  `scipy.cluster.hierarchy`.
- Shapelets: extração simples (sem dependência nova) por janelas de maior
  variância, ranking por distância euclidiana.
- Domínio da frequência: periodograma, PSD (Welch), espectrograma,
  densidade espectral cruzada e coerência entrada/saída via `scipy.signal`.
- Identificação de sistemas: resposta ao impulso empírica via correlação
  cruzada, e modelo ARX (mínimos quadrados, ordem 2) por par
  entrada→saída, com polos/estabilidade.
- Gráficos complementares: histogramas/boxplots e pairplot das 7 features,
  média móvel ±1 desvio, heatmap de correlação (Pearson), heatmaps de
  p-valor de Granger e de cointegração, predito-vs-real do melhor ARX e
  diagrama de polos no plano complexo.
- Novas dependências: `statsmodels`, `dtaidistance` (via `uv add`).

Report completo (18 gráficos extraídos do notebook + implicações para o TCC): ver
`docs/5_eda-series-temporais.md` e `reports/figures/eda_series_temporais/`.

## Data / model impact

- Todas as 7 features são estacionárias após detrend (ADF p < 0.05).
- Causalidade de Granger mais forte: `vazao_entrada -> nivel_res` (lag 1,
  p ≈ 5e-124) e `vazao_entrada -> vazao_distribuicao` (lag 8).
- Cointegração (Engle-Granger) significativa entre `vazao_entrada` e as 3
  saídas, e entre `vazao_distribuicao`/`nivel_res`/`pressao` entre si —
  consistente com serem o mesmo sistema físico de reservatório. Johansen
  indica rank de cointegração alto (trace stat >> crit 99% até o 6º
  autovalor).
- ARX (ordem 2) mais preciso para `nivel_res` (RMSE ≈ 0.10–0.12), pior para
  `vazao_distribuicao` (RMSE ≈ 9.2–9.7); todos os modelos ajustados ficaram
  estáveis (polos dentro do círculo unitário).
- Sem mudança em `src/`; apenas notebook de EDA + deps novas em
  `pyproject.toml`/`uv.lock`.
