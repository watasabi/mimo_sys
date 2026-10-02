# Dataset completo: EDA comparativa e splits para treino

- Date: 2026-10-02
- Status: done
- Type: data

## Goal

O artigo usa apenas 1.102 das 2.208 amostras de `SANEPAR_COMPLETO.txt`
(601 estimação + 501 validação). Entender o quanto esse recorte
representa o dataset completo, medir o modelo ARMAX/MOGWO publicado
fora dessa janela, e preparar splits do dataset completo para treinar
modelos (como o Seq2Seq) em mais dados.

## Approach

- `notebooks/eda/02_split_vs_dataset_completo.ipynb`:
  - Compara estatísticas descritivas e distribuições do recorte do
    artigo (1.102 amostras, 49,9% do total) contra o restante da série
    (1.106 amostras nunca vistas no ajuste/validação publicados).
  - Reaplica a mesma `armax_predict`/`armax_objective` de
    `notebooks/modeling/01_mogwo_armax_reproducao.ipynb` (parâmetros
    da Tabela 3, coluna MOGWO) sobre as 2.208 amostras completas, para
    medir a degradação do modelo publicado fora da janela em que foi
    ajustado. Sanity check: os valores no recorte do artigo batem com
    a Tabela 2 do paper.
- `notebooks/processing/03_full_dataset_sample.py`: gera dois esquemas
  de split sobre o dataset completo (2.208 amostras), sem embaralhar
  (preserva a ordem temporal):
  1. `TimeSeriesSplit` (5 folds, sklearn) sobre as primeiras 2.008
     amostras + holdout de teste fixo com as 200 últimas, nunca usado
     nos folds.
  2. Split cronológico simples treino/validação/teste (70/15/15).

## Data / model impact

- Lê: `SANEPAR_COMPLETO.txt`,
  `data/external/tabela3_parametros_armax_papper.csv`.
- Escreve:
  - `data/processed/mimo_full_kfold_assignments.parquet` (atribuições
    fold/role/sample_idx dos 5 folds do `TimeSeriesSplit`).
  - `data/processed/mimo_full_kfold_test.parquet` (holdout de 200
    amostras).
  - `data/processed/mimo_full_sample_{train,val,test}.parquet`
    (1545/331/332 amostras, split cronológico 70/15/15).

**Resultado da EDA** (ARMAX/MOGWO, Tabela 3, MSE/MAE/R²/SMAPE%):

| Saída | MSE (recorte) | MAE (recorte) | R² (recorte) | SMAPE% (recorte) | MSE (completo) | MAE (completo) | R² (completo) | SMAPE% (completo) |
|---|---|---|---|---|---|---|---|---|
| y1 (vazão) | 41.149 | 4.450 | 0.938 | 7.90 | 80.056 | 4.791 | 0.879 | 8.52 |
| y2 (nível) | 0.003 | 0.036 | 0.954 | 1.54 | 0.012 | 0.047 | 0.862 | 2.34 |
| y3 (pressão) | 2.563 | 1.004 | 0.848 | 5.04 | 3.614 | 1.110 | 0.800 | 5.88 |

"Recorte" = 1.102 amostras do artigo (sanity check: bate com a Tabela
2 do paper); "completo" = 2.208 amostras. O modelo publicado degrada
moderadamente fora da janela de ajuste (mais em `y2`, nível), mas não
colapsa — generaliza razoavelmente para o resto da série.
