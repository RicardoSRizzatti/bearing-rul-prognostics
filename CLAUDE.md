# CLAUDE.md

Contexto para o Claude Code neste repositório. O README tem a descrição completa do desafio; aqui ficam o estado atual, as decisões e o que não pode ser quebrado.

## O projeto

Estimar o RUL (vida útil remanescente) de rolamentos a partir de vibração, com os dados do **IEEE PHM 2012 Prognostic Challenge** (plataforma PRONOSTIA, FEMTO-ST).

- **Dados:** `make data` baixa o zip oficial do repositório da NASA PCoE e descompacta em `data/raw/femto` (~3 GB). `data/` nunca vai para o git.
- **Estrutura:** pastas `Learning_set/` (6 rolamentos até a falha) e teste (11 rolamentos truncados); confira os nomes reais depois de descompactar. Cada `acc_xxxxx.csv` é um trecho de 0,1 s (2560 amostras a 25,6 kHz) com colunas hora, minuto, segundo, microssegundo, aceleração horizontal e vertical; os trechos são registrados a cada 10 s. Há arquivos com separador `;`.
- **Falha:** amplitude de vibração acima de 20 g.
- **Condições:** 1 = 1800 rpm / 4000 N, 2 = 1650 rpm / 4200 N, 3 = 1500 rpm / 5000 N.
- **Métrica oficial:** `Er = 100·(RUL_real − RUL_prev)/RUL_real`; `A = exp(−ln(0,5)·Er/5)` se `Er ≤ 0`, `A = exp(ln(0,5)·Er/20)` se `Er > 0`; score = média de `A` nos 11 rolamentos de teste. Implementar em `ml/evaluate.py` com teste unitário (ex.: `Er = −5` e `Er = 20` dão `A = 0,5`).
- **RUL real do teste:** divulgado após o desafio (entre 339 s e 7570 s); o espelho <https://github.com/Lucky-Loek/ieee-phm-2012-data-challenge-dataset> traz os ensaios de teste completos. Confirmar os valores antes de usar no score.

## Estado atual

O repositório foi criado a partir do template `datascience_template` e **o pipeline ainda roda o exemplo do Iris**. O que é específico do Iris e precisa ser trocado:

- `etl/extract.py` (carrega o Iris do scikit-learn), `etl/transform.py`
- `configs/config.yaml` (`project`, `data.table`, `data.target`, `data.features`, `display_names`, `split`, `model.params`)
- `ml/train.py` → `build_pipeline()` (hoje `StandardScaler` + `LogisticRegression`; aqui será regressão)
- `models/model.onnx` e `models/metadata.json` (regerar com `make all`)
- `tests/test_api.py`, `tests/test_etl.py` e o payload do smoke test em `.github/workflows/ci.yml`
- `pyproject.toml` ainda usa o nome de pacote `datascience-template`. Para renomear, troque `name` e rode `uv lock` (a CI usa `uv sync --locked` e falha se o lock estiver desatualizado).

O checklist do projeto está na seção **Roteiro** do README: marque os itens conforme forem concluídos.

## Decisões de projeto

- **Uma linha por trecho de 0,1 s** depois do transform: identificador do rolamento, condição, tempo desde o início, features (RMS, curtose, fator de crista, pico, energia por banda e do envelope, nos dois eixos) e, para o aprendizado, o RUL naquele instante. O sinal bruto não passa do `extract`/`transform`: ler arquivo a arquivo e agregar, sem carregar os 3 GB num DataFrame.
- **Split por rolamento, não aleatório.** O `ml/split.py` do template faz `train_test_split` estratificado; aqui o teste são os 11 rolamentos oficiais e a validação é deixando um rolamento de fora (GroupKFold por rolamento). Trechos do mesmo rolamento nunca podem estar em treino e validação ao mesmo tempo.
- **Previsão no último trecho de cada rolamento de teste**: o score oficial usa uma estimativa por rolamento, no fim do registro truncado.
- **Features que o modelo usa na previsão** (escala, combinações) ficam no `Pipeline` do `ml/train.py`, para irem no ONNX. As features de sinal (RMS, espectro) são calculadas no ETL, e a API recebe essas features.

## Regras do template (verificadas em `tests/test_project_rules.py` e na CI)

1. `etl`, `ml` e `app` importam de `tools`; `tools` não importa nenhum deles; `app` não importa `etl`, `ml`, scikit-learn nem pandas.
2. Modelos só em ONNX, salvos por `tools.model_io.save_onnx_model` (valida e compara com o modelo original). Nada de `joblib`/`pickle` em `models/`. LightGBM/XGBoost convertem via `onnxmltools`.
3. Pré-processamento que o modelo precisa na previsão fica dentro do `Pipeline`.
4. `ml/evaluate.py` mede o próprio `model.onnx`.
5. `data/` fora do git; `models/` versionado (acima de 50 MB, usar Git LFS).

## Comandos

```bash
make setup   # dependências Python (uv) e do front (npm)
make data    # baixa os dados para data/raw/femto
make all     # ETL + treino + avaliação → models/model.onnx + metadata.json
make app     # API + front em http://localhost:8000
make check   # lint + testes (o mesmo que a CI)
make help    # todos os comandos
```

## Convenções

- Documentação, comentários e docstrings em português, como no resto do template.
- Antes de comitar: `make check`. Ao trocar o exemplo do Iris, rodar `make all` e comitar `models/` junto.
- A CI roda `make all` sem acesso aos dados: quando o extract passar a ler `data/raw`, prever um modo de amostra pequena (fixture em `tests/`) para a CI continuar verde.
- Notebooks de exploração em `notebooks/`; o que for definitivo vai para `etl/` ou `ml/`.
