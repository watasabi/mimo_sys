# Análise no domínio da frequência: real vs. predito

- Date: 2026-10-09
- Status: done
- Type: analysis

## Goal

Entender **em que faixas de frequência** os modelos erram. O MSE/SMAPE
agregado de `04_arquiteturas_vs_armax.ipynb` diz que as redes superam o
ARMAX de 12 passos, mas não diz se o erro está no ciclo diário, no
harmônico de 12h ou no ruído — informação necessária para justificar o
`nivel_res` ruim (R² 0.25–0.39) sem recorrer a "o modelo é fraco".

## Approach

- `notebooks/training/04_arquiteturas_vs_armax.ipynb`, 4 células novas no
  fim: `amplitude_spectrum` (rfft unilateral), `band_table` (MSE e SMAPE
  espectrais + erro relativo de energia por banda) e um gráfico de
  espectro medido vs. predito em eixo log-log, com marcas em 12h e 24h.
- As bandas são ancoradas na EDA (`docs/5_eda-series-temporais.md` §7),
  não escolhidas ad hoc: tendência (>32h), diário (16–32h), harmônico de
  12h (8–16h), curto (4–8h) e ruído (2–4h). O pico de Welch no conjunto
  de teste cai em 25,6h, consistente com os 23–26h documentados lá.
- Reaproveita `y_true`/`preds`/`armax_12`, que o notebook já constrói a
  partir dos pesos salvos — **nenhum modelo foi retreinado**.

## Data / model impact

- Nenhum dado ou artefato de modelo escrito; a análise é derivada dos
  pesos existentes em `models/*_70_15_15.pth`.
- Achado principal, consistente entre as 5 redes: a banda diária
  (16–32h) é reproduzida quase perfeitamente em `vazao_distribuicao`
  (−0,3% a −9,8% de energia), e é ela que sustenta o R² bom. Acima do
  ciclo diário tudo colapsa: o harmônico de 12h perde 30–70% da energia
  e a banda de ruído (2–4h) perde 63–94% em
  `vazao_distribuicao`/`pressao` — suavização clássica de forecast.
- `nivel_res` se comporta ao contrário: as redes **injetam** energia que
  não existe no medido na banda diária (+58% a +76%) e na de ruído
  (+69% a +235%), enquanto perdem a tendência (−53% a −70%). Isso
  explica melhor o R² baixo do nível do que a leitura agregada.
- MLflow: nada novo registrado (análise pós-treino, sem run).

## Done when

- [x] Notebook roda de ponta a ponta (`jupyter nbconvert --execute`)
- [x] `uv run --extra ml pytest tests/` (38 passed)
- [x] `CHANGELOG.md` atualizado em `[Unreleased]`

## Nota: como abrir o MLflow

O backend é o SQLite em `notebooks/training/mlflow.db` (36 runs, 6
experimentos); `mlruns/` guarda só os artefatos. Os dois caminhos são
relativos, então o comando precisa rodar de dentro de
`notebooks/training/`:

```bash
cd notebooks/training
uv run --extra ml mlflow ui --backend-store-uri sqlite:///mlflow.db
```

Rodar da raiz do repo abre uma UI vazia (ou com links de artefato
quebrados), porque `sqlite:///mlflow.db` aponta para outro caminho.
