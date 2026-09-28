# Changelog

Todas as mudanças notáveis neste projeto serão documentadas neste arquivo.

O formato é baseado no [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/), e este projeto adere ao [Versionamento Semântico](https://semver.org/lang/pt-BR/).

## [Unreleased]

### Added
- Setup inicial do projeto via template ds-template-v2.
- Notebook de exploração dos dados de reservatório (`notebooks/eda/`).
- Notebook `notebooks/processing/01_mimo_dataset_from_papper.ipynb` que
  gera os splits de treino/validação usados no artigo, salvos em
  `data/processed/mimo_dataset_sample_from_papper_{train,test}.parquet`.
- Reprodução em Python do MOGWO para identificação ARMAX do sistema
  MIMO (`notebooks/modeling/01_mogwo_armax_reproducao.ipynb`), usando
  os 72 parâmetros do melhor modelo publicados na Tabela 3 do artigo
  (`data/external/tabela3_parametros_armax_papper.csv`), já que o
  código do melhor modelo do artigo (MOCS) não está disponível nos
  dados brutos — ver `docs/changes/2026-09-25-reproduzir-mogwo.md`.

- `src/mimo_sys/preprocessors/` com `TimeSeriesPreprocessor` (imputação,
  remoção de outliers, decomposição EWT/detrend, filtro adaptativo,
  differencing e normalização), reexportado a partir de
  `notebooks/processing/02_preprocessing_ts.py`. Inclui
  `inverse_transform` por lote de janelas, usando os índices de âncora
  de cada janela para desfazer differencing/detrend corretamente
  (em vez de assumir uma única sequência contínua), o que permite
  avaliar previsões de modelo na escala original dos dados.

### Fixed
- Pacote `mimo_sys` agora é instalável (`[build-system]`/`hatchling` em
  `pyproject.toml`); antes o projeto era "virtual" no uv e
  `from mimo_sys import ...` não funcionava.
- `TimeSeriesPreprocessor` força uma cópia gravável do array após a
  imputação, pois `pandas>=3.0` (Copy-on-Write) faz `DataFrame.values`
  devolver arrays somente leitura, quebrando as atribuições in-place
  do pipeline (differencing, detrend).
- `armax_predict` agora trata as 3 primeiras predições como iguais ao
  medido, como o `fun_objetivo.m` do MATLAB. O custo das soluções do
  `MOGWO.mat` passa a bater com o `PARETO_FRONT` dos autores (~3e-15) e
  o modelo da Tabela 3 reproduz a Tabela 2 (R² de y2: 0.67 → 0.95).
- Porte do MOGWO: Beta e Alpha sorteados entre líderes ainda não usados
  e `A` escalar para Beta e Alpha, como no `MOGWO.m`.