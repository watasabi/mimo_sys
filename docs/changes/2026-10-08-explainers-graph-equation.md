# Explainers: extração de equação e de subgrafo

- Date: 2026-10-08
- Status: done
- Type: features

## Goal

Operacionalizar as duas rotas de `docs/4_rotas-de-extracao.md` como código
reutilizável: Rota A (equação, via destilação simbólica de `message_fn`/
`update_fn`) e Rota B (subgrafo/topologia, via `edge_strength`, com
PGExplainer como fallback opcional).

## Approach

- `src/mimo_sys/explainers/equation/distill.py` — Rota A. Funções que
  recebem os pares `(inputs, outputs)` já produzidos por
  `MPNNForecaster.collect_message_io`/`collect_update_io`
  (`src/mimo_sys/architectures/mpnn.py`) e rodam PySR, nomeando as
  colunas pela convenção documentada em `4_rotas-de-extracao.md` §Rota A
  (`recv_lag*`/`send_lag*` para a mensagem, `own_lag*`/`msg*` para a
  atualização). Import de `pysr` é lazy — o pacote não precisa dele
  instalado para importar `mimo_sys`.
- `src/mimo_sys/explainers/graph/topology.py` — Rota B. `edge_strength_frame`
  nomeia a matriz `[n_nodes, n_nodes]` de `model.edge_strength(x)` (B1,
  grátis). `PGExplainerTopology` é um wrapper fino e opcional sobre
  `torch_geometric.explain.PGExplainer` (B2), import lazy, para o caso
  em que B1 não seja suficiente.
- Dependências novas como extras opcionais (`pyproject.toml`):
  `symbolic` (`pysr`) e `explain` (`torch-geometric`). Nenhuma delas
  entra nas dependências default.
- Testes: `tests/test_explainers.py`, cobrindo os helpers que não
  dependem de `pysr`/`torch-geometric` (nomes de features, framing do
  `edge_strength`, mensagens de erro quando a dependência opcional
  falta).

## Data / model impact

- Nenhum dado é lido ou escrito; são funções sobre tensores já
  produzidos pelos modelos existentes.
- Sem mudança de schema; só código novo em `src/`.

## Done when

- [x] `uv run ruff check src/ tests/`
- [x] `uv run ruff format src/ tests/`
- [x] `uv run pytest tests/`
- [x] `CHANGELOG.md` atualizado em `[Unreleased]`
