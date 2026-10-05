# Seq2Seq em 70/15/15 e k-fold, comparado ao ARMAX/MOGWO

- Date: 2026-10-05
- Status: done
- Type: model

## Goal

Treinar o Seq2Seq com attention no dataset completo nos dois esquemas
de split padronizados (70/15/15 e `TimeSeriesSplit` + holdout) e
compará-lo ao ARMAX/MOGWO publicado em horizontes equivalentes.

## Approach

- `notebooks/training/02_seq2seq_attention_splits_vs_armax.ipynb`: mesmo
  modelo de `01_seq2seq_attention.ipynb`; `train_seq2seq` e `evaluate`
  reutilizados nos dois esquemas, validação cronológica para early
  stopping, MLflow experiment `mimo_sys_seq2seq_attention_splits`.
- Comparação nas mesmas amostras do forecast encadeado de 12 passos:
  ARMAX 1 passo, ARMAX 12 passos (free-run) e Seq2Seq.
- `03_full_dataset_sample.py` volta a gerar só 70/15/15 e k-fold (o
  split 50/20/30, testado antes, foi descartado).

## Data / model impact

- Escreve `models/seq2seq_attention_mimo_70_15_15.pth`.
- Resultado: Seq2Seq vence o ARMAX 12 passos em 8 de 9 combinações
  esquema/saída, com variância alta entre folds. Tabela e ressalvas em
  `docs/2_split_artigo_vs_dataset_completo.md`, seção 7.
