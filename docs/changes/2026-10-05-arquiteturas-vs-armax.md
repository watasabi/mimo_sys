# Todas as arquiteturas vs. ARMAX/MOGWO (split 70/15/15)

- Date: 2026-10-05
- Status: done
- Type: model

## Goal

Treinar as quatro arquiteturas de `src/mimo_sys/architectures/`
(`MPNNForecaster`, `SymbolicGraphNetwork`, `StemGNN`,
`Seq2SeqLatentGNN`) no mesmo split 70/15/15 e protocolo de
`02_seq2seq_attention_splits_vs_armax.ipynb`, e comparar com
`Seq2SeqAttention` e o ARMAX/MOGWO publicado.

## Approach

- `notebooks/training/04_arquiteturas_vs_armax.ipynb`: contrato comum
  `(batch, input_window, n_nodes) -> (batch, output_window, n_targets)`
  das 4 arquiteturas, `hidden_size=64`, `message_dim=8` em
  `MPNN`/`SymbolicGraphNetwork`. Mesmo forecast encadeado de 12 passos
  do notebook 02, carregando pesos já treinados em
  `models/<nome>_70_15_15.pth`.
- Cada arquitetura foi treinada em processo separado (script fora do
  repositório): treinar as 4 em sequência no mesmo processo esgotava a
  RAM da máquina de desenvolvimento (7,7 GB) — StemGNN e
  Seq2SeqLatentGNN sozinhos já usam a maior parte.

## Data / model impact

- Escreve `models/{mpnn,symbolicgraphnetwork,stemgnn,seq2seqlatentgnn}_70_15_15.pth`.
- Resultado: todas as 5 redes superam o ARMAX de 12 passos nas 3
  saídas; `SymbolicGraphNetwork` tem a melhor vazão e pressão entre as
  redes e é destilável; `StemGNN` é o único a melhorar o nível, mas
  não é destilável. Tabela e ressalvas em
  `docs/2_split_artigo_vs_dataset_completo.md`, seção 10.
