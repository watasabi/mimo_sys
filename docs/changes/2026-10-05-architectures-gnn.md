# Arquiteturas GNN para identificação MIMO

- Date: 2026-10-05
- Status: done
- Type: model

## Goal

Implementar em `src/` as quatro arquiteturas selecionadas na triagem de
[../3_investigacao-arquiteturas.md](../3_investigacao-arquiteturas.md), com
contrato de entrada/saída único, para que possam ser treinadas e comparadas
nos mesmos splits já usados pelo ARMAX/MOGWO e pelo Seq2Seq.

Nenhum treino nesta mudança — só as arquiteturas e os testes de contrato.
As rotas de extração (equação vs. subgrafo) estão documentadas em
[../4_rotas-de-extracao.md](../4_rotas-de-extracao.md).

## Approach

- Código reutilizável: `src/mimo_sys/architectures/`
  - `mpnn.py` — `MPNNForecaster`, veículo de extração (Modelo B). Expõe
    `message_fn` e `update_fn` como submódulos isolados, com entradas
    nomeadas (janela de lags por nó), para destilação com SymTorch.
  - `stemgnn.py` — `StemGNN`, reprodução da ref. (3): latent correlation
    layer (GRU + self-attention), GFT, Spe-Seq cell (DFT → Conv1D → GLU
    → IDFT), forecast + backcast.
  - `seq2seq_latent_gnn.py` — `Seq2SeqLatentGNN`, reprodução da ref. (4):
    filtro de Fourier low-rank, GCN de correlação latente com máscara
    k-NN, Seq2Seq LSTM com atenção e teacher forcing, suavização
    gaussiana na saída.
  - `symbolic_graph_network.py` — `SymbolicGraphNetwork`, ref.
    arXiv:2603.22380: reaproveita o MPNN e adiciona warm-start
    Savitzky-Golay e injeção dinâmica de ruído; expõe os datasets dos
    dois estágios de destilação (`collect_message_io`,
    `collect_update_io`).
- Testes: `tests/test_architectures.py` (contrato de shape, determinismo
  por seed, e as propriedades específicas de cada arquitetura).
- Sem `torch_geometric`: com N=7 nós a adjacência densa + `einsum` é mais
  simples e evita uma dependência pesada.

## Data / model impact

- Lê / escreve dados: **nada**. Mudança só de código.
- Contrato único: `forward(x) : [B, input_window, n_nodes] -> [B,
  output_window, n_targets]`, com `target_idx` indicando quais nós são
  saídas. Mesma convenção de janela do
  `notebooks/training/01_seq2seq_attention.ipynb` (batch-first aqui; o
  notebook permuta para seq-first internamente).
- Ordem dos nós fixada por `FEATURE_COLS` = `u1..u4` + `y1..y3`
  (`freq_b1`, `freq_b2`, `freq_b3`, `vazao_entrada`,
  `vazao_distribuicao`, `nivel_reservatorio`, `pressao`).
- Contagem de parâmetros medida, `output_window=1`, `n_nodes=7`,
  `hidden_size=64`, `message_dim=2`:

| Arquitetura | `input_window=24` | `input_window=3` (ordem ARMAX) |
|---|---|---|
| `MPNNForecaster` | 13.379 | 9.347 |
| `SymbolicGraphNetwork` | 13.379 | 9.347 |
| `StemGNN` | 27.658 | 22.408 |
| `Seq2SeqLatentGNN` | 46.764 | 44.916 |

O ARMAX do artigo tem 72 parâmetros estimados (eq. 15 / Tabela 3). A
comparação de contagem é argumento para o TCC, não defeito — mas reforça
que o entregável tem de ser a *equação destilada*, não o modelo.

## Achados (dois bugs reais pegos pelos testes)

1. **A eigendecomposição exata do GFT é irreprodutível aqui.** A primeira
   versão usava `torch.linalg.eigh` em vez da aproximação de Chebyshev do
   `microsoft/StemGNN`, com o argumento de que O(N³) é grátis com N=7. É
   grátis e está errado: a atenção aprendida é quase uniforme (0,1429 ≈
   1/7), então a Laplaciana normalizada tem autovalor 0 uma vez e
   autovalor ~1 com multiplicidade 6 (gaps ~1e-4). Os autovetores nesse
   subespaço quase degenerado são numericamente arbitrários, e como a
   Spe-Seq cell fica *entre* a transformada direta e a inversa, os sinais
   não se cancelam — duas chamadas dos mesmos pesos diferiam em até 0,21.
   Trocado por Chebyshev, que é função da Laplaciana e nunca referencia
   autovetores. Coberto por `test_stemgnn_is_reproducible`.
2. **O `backcast` do último bloco do StemGNN era peso morto.** O resíduo
   que ele produz era descartado ao fim do laço, então aqueles parâmetros
   não recebiam gradiente nenhum. No paper o backcast é o ramo
   auto-encoder e entra na loss; agora há `StemGNN.backcast_loss(x)`,
   que o training step deve somar à loss de forecast. Coberto por
   `test_backward_reaches_every_parameter`.

Nota de processo: o `ruff --unsafe-fixes` converteu três somas em
operações *in-place* (`adjacency += ...` sobre a saída de um softmax),
o que quebra o autograd. Revertido; o caso que precisa de reatribuição
tem `noqa: PLR6104` com a razão no comentário.

## Done when

- [x] Importa e roda forward em CPU com shapes do dataset do artigo
- [x] `uv run ruff check src/ tests/` e `uv run ruff format src/ tests/`
      limpos nos arquivos novos (os 13 erros restantes são
      pré-existentes em `preprocessors/timeseries.py`)
- [x] `uv run --extra ml pytest tests/` — 32 passam
- [x] `CHANGELOG.md` atualizado em `[Unreleased]`
- [ ] Treino e comparação (mudança separada, `notebooks/training/`)

**Atenção:** `torch` está no extra `ml`, não nas dependências padrão,
então os testes exigem `uv run --extra ml pytest tests/`.
