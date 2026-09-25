# Reproduzir identificação MIMO com MOGWO

- Date: 2026-09-25
- Status: done
- Type: model

## Goal

Reproduzir em Python o modelo ARMAX MIMO (reservatório São José) do
artigo com o MOGWO, o segundo melhor da Tabela 2. O MOCS (melhor) não
tem código nos dados brutos: nenhum dos 4 zips internos de
`data/raw/WRSO and NSWRSO_draft_all` contém Cuckoo Search.

## Approach

- Fonte de verdade: MATLAB em
  `data/raw/WRSO and NSWRSO_draft_all/Multi_objective_for_MIMO_systems/MOGWO/`.
- Splits do artigo (`main.m`): estimação `550:1150` (601 amostras) e
  validação `1500:2000` (501), gerados por
  `notebooks/processing/01_mimo_dataset_from_papper.ipynb` a partir de
  `SANEPAR_COMPLETO.txt`.
- `notebooks/modeling/01_mogwo_armax_reproducao.ipynb`: modelo ARMAX
  (eq. 15, 72 parâmetros), verificação cruzada com o `.mat` dos autores,
  avaliação dos 72 parâmetros da Tabela 3 (coluna MOGWO, em
  `data/external/tabela3_parametros_armax_papper.csv`) e porte de
  referência do MOGWO (não roda no fluxo principal).

## Data / model impact

- Lê: `SANEPAR_COMPLETO.txt`, `imprime_gráficos/MOGWO.mat`,
  `data/external/tabela3_parametros_armax_papper.csv`.
- Escreve: `data/processed/mimo_dataset_sample_from_papper_{train,test}.parquet`.

**Resultado** (Tabela 3, validação):

|        | y1 (vazão) | y2 (nível) | y3 (pressão) |
|--------|-----------|-----------|--------------|
| MSE artigo | 33.486 | 0.0025 | 2.3168 |
| MSE aqui | 33.500 | 0.0024 | 2.3167 |
| R² artigo | 0.9478 | 0.9526 | 0.8364 |
| R² aqui | 0.9477 | 0.9540 | 0.8364 |

A diferença que sobra em y2 (~0.001 de R²) é compatível com os 3
dígitos dos parâmetros publicados.

**Causas das discrepâncias encontradas pelo caminho**

1. Leitura da Tabela 2: y2 e y3 da linha "best model from validation"
   estavam trocados na minha extração, o que parecia um erro do artigo.
2. Objetivo ARMAX: o MATLAB (`fun_objetivo.m`) substitui as 3 primeiras
   predições pelo valor medido; o porte não fazia isso. Isso dava R² de
   y2 = 0.67 em vez de 0.95. Prova: o custo recalculado das 1500
   soluções do `.mat` passou de 345% de erro máximo (y2) para ~3e-15
   contra o `PARETO_FRONT` salvo pelos autores. Agora é um `assert` no
   notebook.
3. Porte do MOGWO: Beta e Alpha devem ser sorteados entre os líderes
   ainda não usados, e o `A` deles é escalar. Sem isso, uma execução
   ficava com MSE de treino de y2 entre 0.1 e 3.6. Com isso, uma
   execução (200 lobos, 1000 gerações, ~50s) dá 0.004 e 0.016 em duas
   sementes, dentro da faixa dos autores (0.0023–0.0248 nas 50
   execuções).

**Em aberto**: as colunas GA, GWO e CS da Tabela 3 não reproduzem o R²
de y2 da Tabela 2 (GWO dá −9 aqui contra 0.41). Provável sensibilidade
ao arredondamento em parâmetros grandes. Não afeta o MOGWO e não foi
investigado.

## Done when

- [x] Notebook roda do início ao fim com `uv run --group plot jupyter ...`
- [x] Splits em `data/processed/` batem com o artigo (601 e 501 linhas)
- [x] Custo do porte bate com o `PARETO_FRONT` do MATLAB (`assert`)
- [x] MSE/R² da Tabela 3 batem com a Tabela 2
- [x] `CHANGELOG.md` atualizado em `[Unreleased]`
