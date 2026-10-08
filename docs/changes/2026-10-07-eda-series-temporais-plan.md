# EDA de séries temporais (tempo, frequência, DTW/clustering/shapelets, identificação de sistemas)

## Context

O projeto já tem pré-processamento de séries temporais em `src/mimo_sys/preprocessors/timeseries.py` (`TimeSeriesPreprocessor`: imputação, remoção de outliers, EWT/detrend, filtro adaptativo, diferenciação, normalização) usado nos notebooks de treino, mas nenhuma notebook de EDA explora formalmente as propriedades estatísticas de série temporal da série MIMO (tendência, estacionariedade, autocorrelação, causalidade, colinearidade, clustering/shapelets). O usuário pediu uma EDA dedicada a isso, reaproveitando o `TimeSeriesPreprocessor` para gerar a versão decomposta/detrended da série, seguindo o fluxo `ds-change`.

Os pacotes necessários (`statsmodels` para decomposição/ADF/ACF-PACF/Granger/VIF, `dtaidistance` para DTW e clustering hierárquico baseado em DTW) **não estão instalados** — serão adicionados via `uv add`. Para shapelets, a lib mais usual (`tsfresh`/`pyts`) traz muitas dependências pesadas; dado "prefira simples" do projeto, vou implementar uma extração de shapelets simplificada (busca de subsequências discriminativas via distância euclidiana/DTW em janelas), documentada no notebook, em vez de puxar uma lib extra — evita inchar o `pyproject.toml` por uma única técnica exploratória. Vou perguntar ao usuário se prefere isso ou instalar `tsfresh`/`pyts` mesmo assim.

## Dados

Usar `data/processed/mimo_data.parquet` (série completa, 2216 x 7, índice `timestamp` horário, colunas `freq_b1, freq_b2, freq_b3, vazao_entrada, vazao_distribuicao, nivel_res, pressao`) — é a série já limpa, com índice temporal real, igual ao padrão usado em `notebooks/eda/01_exploracao_reservatorio_mimo.ipynb`.

## Implementação

1. **`uv add statsmodels dtaidistance`** (grupo principal ou `--group plot`? — manter em deps principais, pois são usados para cálculo, não só plot).

2. **Novo notebook `notebooks/eda/03_eda_series_temporais.ipynb`** (segue a numeração de `01_exploracao_reservatorio_mimo.ipynb` e `02_split_vs_dataset_completo.ipynb`), estruturado em seções:
   - **Carregamento**: `pd.read_parquet(DATA_PROCESSED / "mimo_data.parquet")`, igual ao padrão de `data/processed` já usado.
   - **Preprocessamento auxiliar**: instanciar `TimeSeriesPreprocessor(apply_ewt=True, detrend=True, apply_filter=True, normalize=False, differencing=False, remove_outliers=True)` com `matplotlib.use("Agg")` antes (como em `03_seq2seq_attention_preprocessor.ipynb`) para suprimir os plots internos de diagnóstico, chamando `fit_transform` sobre o array `.to_numpy()` das 7 colunas; usar `get_trend_component()` e `get_ewt_components()` para tendência/decomposição por feature.
   - **Tendência e decomposição**: plot da série original vs. tendência (`get_trend_component`) por feature; complementar com `statsmodels.tsa.seasonal.STL` ou `seasonal_decompose` para decomposição trend/seasonal/resid clássica, comparando com a decomposição EWT.
   - **Estacionariedade**: teste ADF (`statsmodels.tsa.stattools.adfuller`) por feature, na série original e na detrended/differenced.
   - **Autocorrelação**: ACF e PACF (`statsmodels.graphics.tsaplots.plot_acf/plot_pacf`) por feature, original e nos resíduos pós-detrend.
   - **Causalidade de Granger**: `statsmodels.tsa.stattools.grangercausalitytests` entre pares de variáveis (ex.: `vazao_entrada -> nivel_res`, `freq_b* -> vazao_distribuicao`), tabela de p-valores por lag.
   - **Cointegração**: teste de Engle-Granger par a par (`statsmodels.tsa.stattools.coint`) entre as 7 features, tabela de p-valores; complementar com o teste de Johansen (`statsmodels.tsa.vector_ar.vecm.coint_johansen`) no conjunto completo para checar rank de cointegração multivariado.
   - **DTW**: matriz de distância DTW entre as 7 features (`dtaidistance.dtw.distance_matrix_fast`), heatmap.
   - **Clustering de séries temporais**: clustering hierárquico (`scipy.cluster.hierarchy` + matriz de distância DTW) das 7 features, dendrograma.
   - **Shapelets**: implementação simples, sem dependência nova — busca de subsequências discriminativas via janelas deslizantes + ranking por distância euclidiana/variância entre segmentos, usando `numpy`/`scipy` já disponíveis.
   - **Domínio da frequência (sinais/controle)**: usando só `scipy.signal` (já disponível):
     - FFT/periodograma (`scipy.signal.periodogram`) e densidade espectral de potência via Welch (`scipy.signal.welch`) por feature, para identificar frequências/periodicidades dominantes (ex. ciclo diário de 24h na série horária).
     - Espectrograma (`scipy.signal.spectrogram`) para ver não-estacionariedade espectral ao longo do tempo.
     - Densidade espectral cruzada e coerência (`scipy.signal.csd`, `scipy.signal.coherence`) entre pares entrada/saída (ex. `vazao_entrada` vs `nivel_res`, `freq_b*` vs `vazao_distribuicao`) — equivalente no domínio da frequência à causalidade de Granger, mostrando em quais faixas de frequência a relação é mais forte.
   - **Identificação de sistemas (Guidorzi / controle clássico)**: tratando pares entrada→saída já usados no projeto (`freq_b1/2/3, vazao_entrada` como entradas; `vazao_distribuicao, nivel_res, pressao` como saídas, mesma convenção de `INPUT_COLS`/`OUTPUT_COLS` do notebook `03_seq2seq_attention_preprocessor.ipynb`):
     - Estimativa de resposta ao impulso/FRF empírica via correlação cruzada normalizada (`scipy.signal.correlate`) entre entrada e saída, como aproximação simples de função de resposta ao impulso (sem exigir excitação controlada).
     - Ajuste de um modelo ARX simples (regressão linear de `y[k]` em lags de `y` e `u`, via `numpy.linalg.lstsq` ou `sklearn.linear_model.LinearRegression`) para cada par entrada-saída, reportando o erro de predição um passo à frente — identificação de sistemas no estilo do Guidorzi (structs ARX/ARMAX), mantendo a implementação simples e sem lib de identificação dedicada (ex. `sippy`), que não está instalada.
     - Diagrama de polos/zeros do modelo ARX ajustado (`scipy.signal.tf2zpk` + `numpy.roots`) para checar estabilidade, ligando com a leitura de controle clássico.

3. **`docs/changes/2026-10-07-eda-series-temporais.md`**: nota curta no formato existente (`Date`, `Status: done`, `Type: eda`, `Goal`, `Approach` citando o notebook e os módulos tocados, `Data / model impact` com os principais achados: features não-estacionárias, pares com causalidade de Granger significativa, pares cointegrados, clusters DTW).

4. **`CHANGELOG.md`**: entrada em `## [Unreleased]` / `### Added` apontando para a nota acima, seguindo o padrão das entradas existentes.

## Verificação

- Rodar o notebook ponta a ponta: `uv run --extra ml jupyter nbconvert --to notebook --execute notebooks/eda/03_eda_series_temporais.ipynb` (ou execução manual célula a célula), confirmando que não há exceções e que os gráficos/tabelas (ADF, Granger, VIF, DTW, dendrograma) são gerados.
- `uv run ruff check src/ tests/` (notebook não é alvo do ruff, mas qualquer código novo em `src/` — não previsto aqui — precisaria passar).
- Conferir que `pyproject.toml`/`uv.lock` refletem as novas dependências após `uv add`.
