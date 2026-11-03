# Prognóstico de vida útil remanescente de rolamentos

Estimar **quanto tempo um rolamento ainda vai funcionar antes de falhar** (RUL, *remaining useful life*) a partir de sinais de vibração, usando os ensaios acelerados até a falha da plataforma **PRONOSTIA** (FEMTO-ST), base do **IEEE PHM 2012 Prognostic Challenge**.

> 🚧 Projeto em andamento. O roteiro e o status de cada etapa estão na seção [Roteiro](#roteiro).

---

## Contexto

Rolamentos estão entre os componentes que mais falham em máquinas rotativas: motores, bombas, ventiladores, redutores. Detectar que um rolamento está com defeito já é útil, mas o que permite planejar a manutenção é saber **quanto tempo resta** até a falha: com essa estimativa dá para programar a troca para a próxima parada em vez de reagir a uma quebra.

O instituto FEMTO-ST (Besançon, França) construiu a plataforma PRONOSTIA para degradar rolamentos de forma acelerada, sob carga radial elevada, e registrar a vibração do início do ensaio até a falha. Os defeitos não foram induzidos artificialmente: cada rolamento se degradou naturalmente, com falhas que envolvem esferas, pistas e gaiola. Esses ensaios foram a base do desafio de prognóstico da conferência IEEE PHM 2012 e hoje são um dos benchmarks mais usados em estimativa de RUL.

## O desafio

São 17 ensaios em três condições de operação:

| Condição | Rotação | Carga radial | Aprendizado (até a falha) | Teste (interrompidos antes da falha) |
|---|---|---|---|---|
| 1 | 1800 rpm | 4000 N | Bearing1_1, Bearing1_2 | Bearing1_3 a Bearing1_7 |
| 2 | 1650 rpm | 4200 N | Bearing2_1, Bearing2_2 | Bearing2_3 a Bearing2_7 |
| 3 | 1500 rpm | 5000 N | Bearing3_1, Bearing3_2 | Bearing3_3 |

Os 6 ensaios de aprendizado vão até a falha, definida como o instante em que a amplitude de vibração passa de 20 g. Os 11 ensaios de teste foram **truncados**: a tarefa é olhar o histórico disponível de cada um e estimar quanto tempo faltava, a partir do último registro, para ele atingir o critério de falha.

O que torna o problema difícil:

- **Pouquíssimos exemplos de falha**: apenas 6 rolamentos com histórico completo, 2 por condição.
- **Grande variação de vida útil** entre rolamentos iguais na mesma condição, típica de degradação natural.
- **Degradação não monotônica**: o sinal pode ficar estável por muito tempo e piorar de forma abrupta no fim.
- **Três condições de operação**, com apenas um rolamento de teste na condição 3.
- **Penalização assimétrica**: prever uma vida útil maior que a real (falha antes do previsto) custa mais caro que prever uma menor.

## Objetivo e métrica

**Objetivo:** estimar o RUL dos 11 rolamentos de teste no instante em que cada registro termina, maximizando o score oficial do desafio, e servir o modelo por uma API que recebe as features de um trecho de vibração e devolve a estimativa.

**Métrica oficial (IEEE PHM 2012).** Para cada rolamento de teste *i*, calcula-se o erro percentual

```
Er_i = 100 × (RUL_real_i − RUL_previsto_i) / RUL_real_i
```

e o score individual

```
A_i = exp(−ln(0,5) · Er_i / 5)    se Er_i ≤ 0   (previsão atrasada: falha antes do previsto)
A_i = exp(+ln(0,5) · Er_i / 20)   se Er_i > 0   (previsão antecipada)
```

O score final é a média dos `A_i` nos 11 rolamentos (1 = perfeito). Um erro de 5% para o lado atrasado já reduz o score do rolamento à metade; para o lado antecipado, isso só acontece com 20% de erro.

Métricas complementares: erro absoluto médio do RUL nos rolamentos de teste e na validação deixando um rolamento de fora.

## Dados

| | |
|---|---|
| **Origem** | Plataforma PRONOSTIA, instituto FEMTO-ST — IEEE PHM 2012 Prognostic Challenge |
| **Download oficial** | NASA Prognostics Center of Excellence — Data Set Repository, item *10. FEMTO Bearing*: <https://www.nasa.gov/intelligent-systems-division/discovery-and-systems-health/pcoe/pcoe-data-set-repository/> (arquivo direto: <https://phm-datasets.s3.amazonaws.com/NASA/10.+FEMTO+Bearing.zip>) |
| **Espelho no GitHub** | <https://github.com/Lucky-Loek/ieee-phm-2012-data-challenge-dataset> (inclui os ensaios de teste completos, usados para conferir o RUL real) |
| **Vibração** | 2 acelerômetros (horizontal e vertical) a 25,6 kHz; um trecho de 2560 amostras (0,1 s) a cada 10 s |
| **Temperatura** | 10 Hz; 600 amostras por minuto |
| **Formato** | CSV, um arquivo por trecho (`acc_xxxxx.csv`, `temp_xxxxx.csv`); cerca de 3 GB descompactado |
| **RUL real do teste** | entre 339 s e 7570 s, conforme divulgado após o desafio |

Os dados **não são versionados** neste repositório. Para baixá-los:

```bash
make data   # baixa o zip oficial e descompacta em data/raw/femto
```

Cada `acc_*.csv` traz hora, minuto, segundo, microssegundo, aceleração horizontal e aceleração vertical. Nem todos os arquivos usam o mesmo separador (há arquivos com `;`), então a leitura no `etl/extract.py` precisa detectá-lo.

## Sobre o projeto

| | |
|---|---|
| **Problema** | Estimar o RUL de rolamentos a partir de vibração, com poucos históricos completos até a falha. |
| **Dados** | FEMTO-ST / PRONOSTIA (IEEE PHM 2012): 6 rolamentos de aprendizado e 11 de teste. |
| **Abordagem (planejada)** | Features por trecho de 0,1 s (RMS, curtose, fator de crista, energia por banda de frequência e envelope), indicador de saúde suavizado, detecção do início da degradação e regressão do RUL. Validação deixando um rolamento de fora. |
| **Métrica principal** | Score oficial do IEEE PHM 2012 (média de `A_i` nos 11 rolamentos de teste). |
| **Resultado** | A preencher: ver `models/metadata.json` ou a seção *Qualidade do modelo* no app. |

## Roteiro

- [ ] Baixar os dados para `data/raw/femto` e adaptar `etl/extract.py` para ler os CSVs de vibração e temperatura
- [ ] Análise exploratória em `notebooks/`: evolução do RMS e da curtose até a falha, espectros no início e no fim de cada ensaio
- [ ] `etl/transform.py`: uma linha por trecho de 0,1 s com as features e o RUL de cada instante (aprendizado)
- [ ] Baseline: regressão do RUL sobre as features no tempo, validada deixando um rolamento de fora
- [ ] Indicador de saúde e detecção do início da degradação (antes dele, o RUL é pouco informativo)
- [ ] Modelo final, exportado em ONNX, e score oficial nos 11 rolamentos de teste
- [ ] Comparação com os resultados publicados para o desafio e análise dos erros por condição de operação

## Referências

- P. Nectoux *et al.*, "PRONOSTIA: An experimental platform for bearings accelerated degradation tests", *IEEE International Conference on Prognostics and Health Management*, Denver, 2012.
- NASA PCoE Data Set Repository: <https://www.nasa.gov/intelligent-systems-division/discovery-and-systems-health/pcoe/pcoe-data-set-repository/>

---

## Início rápido

Requisitos: Linux ou WSL, [uv](https://docs.astral.sh/uv/), Node.js 22+, Docker e `make`.

```bash
make setup        # dependências Python (uv) e do front (npm)
make all          # ETL + treino + avaliação → models/model.onnx
make front-build  # build do React
make app          # http://localhost:8000  (API + front)
```

Ou, só com Docker (usa o modelo já versionado em `models/`):

```bash
make docker       # build + run em http://localhost:8000
```

Para desenvolver o front com hot reload: `make dev` (API na 8000 + Vite na 5173).

`make help` lista todos os comandos.

---

## Estrutura

```
├── configs/config.yaml   # caminhos, seed, destino do load, parâmetros do modelo
├── data/                 # NÃO versionado (só a estrutura)
│   ├── raw/              #   saída do extract: dado como veio da fonte
│   ├── interim/          #   saída do transform: limpo, ainda não publicado
│   ├── processed/        #   saída do load quando o destino é local
│   └── split/            #   train.parquet / test.parquet
├── tools/                # biblioteca própria (usada por etl, ml e app)
│   ├── config.py         #   leitura do config e caminhos
│   ├── data_io.py        #   leitura/escrita: local, BigQuery, GCS
│   ├── model_io.py       #   salvar/carregar modelos (somente ONNX)
│   ├── frames.py         #   operações genéricas com DataFrames
│   ├── git_info.py       #   commit usado no treino
│   └── log.py            #   logger padrão
├── etl/
│   ├── extract.py        # fonte → data/raw
│   ├── transform.py      # data/raw → data/interim (limpeza e regras de negócio)
│   └── load.py           # data/interim → destino (local | bigquery | gcs)
├── ml/
│   ├── split.py          # dados processados → treino/teste
│   ├── train.py          # Pipeline → models/model.onnx + metadata.json
│   └── evaluate.py       # métricas do ONNX no teste → metadata.json
├── models/               # VERSIONADO: somente .onnx + metadata.json
├── app/
│   ├── api/              # FastAPI (carrega o ONNX com onnxruntime)
│   └── frontend/         # React + TypeScript + Vite + ECharts
├── notebooks/            # análise exploratória (fora do pipeline)
├── tests/                # pytest
├── Dockerfile            # multi-stage: build do front + API + modelo
└── Makefile              # automação de todos os comandos
```

### Fluxo

```
fonte ─extract─▶ data/raw ─transform─▶ data/interim ─load─▶ processed (local | BigQuery | GCS)
                                                                   │
                                     ml/split ◀────────────────────┘
                                        │
                     data/split ─train─▶ models/model.onnx ─evaluate─▶ metadata.json (métricas)
                                                │
                                   app/api (onnxruntime) ─▶ app/frontend (React)
```

---

## Regras do repositório

Estas regras são verificadas automaticamente em `tests/test_project_rules.py` e na CI.

1. **Direção das dependências.** `etl`, `ml` e `app` importam de `tools`; `tools` não importa de nenhum deles. `app` não importa `etl`, `ml`, scikit-learn nem pandas: ele só conhece o modelo ONNX e o `metadata.json`.
2. **Modelos somente em ONNX.** A única forma de salvar um modelo é `tools.model_io.save_onnx_model`, que:
   - recusa qualquer caminho que não termine em `.onnx`;
   - valida o arquivo com `onnx.checker`;
   - compara as previsões do ONNX com as do modelo original (tolerância em `model.parity_tolerance`) e **não grava nada** se divergirem.

   Além disso, o `.gitignore` ignora qualquer outro arquivo em `models/`, e um teste falha se aparecer `.joblib`, `.pkl` etc.
3. **Pré-processamento dentro do Pipeline.** O ETL limpa e aplica regras de negócio; escala, encoding e features derivadas que o modelo precisa na previsão ficam no `Pipeline` do `ml/train.py`, para serem exportados no ONNX. Assim o app não reimplementa nada.
4. **Avaliação no artefato final.** O `evaluate.py` mede o próprio `model.onnx`, então a métrica exibida é a do modelo que vai para produção.
5. **Dados fora do git, modelos dentro.** `data/` nunca é versionado. `models/` é: o GitHub avisa acima de 50 MB e bloqueia acima de 100 MB; se um modelo crescer, use Git LFS (instruções no `.gitattributes`).

### ONNX: cuidados

- O ONNX trabalha em float32 por padrão: pequenas diferenças numéricas em relação ao scikit-learn são normais (por isso a tolerância).
- Nem todo componente tem conversor. Os nativos do scikit-learn têm; XGBoost e LightGBM convertem via `onnxmltools`; um transformador customizado exige registrar um conversor no `skl2onnx`.
- Features categóricas (texto): use `ColumnTransformer` com nomes de coluna; `convert_to_onnx` cria um input por coluna automaticamente.

---

## Configuração

Tudo fica em `configs/config.yaml`. Para usar outro arquivo: `CONFIG_PATH=configs/outro.yaml make all`.

### Destino dos dados processados

```yaml
load:
  destination: local   # local (padrão) | bigquery | gcs
```

Para BigQuery ou Cloud Storage:

```bash
make setup-gcp                             # instala o grupo gcp
gcloud auth application-default login      # autenticação
cp .env.example .env                       # e preencha GCP_PROJECT / GCS_BUCKET
```

O `ml/split.py` lê os dados processados pela mesma função (`read_processed`), então sempre encontra os dados onde o load gravou.

### Dependências (grupos do uv)

| Grupo | Para quê | Instalado por |
|---|---|---|
| (projeto) | numpy, pyyaml — comuns a todas as etapas | sempre |
| `etl` | pandas, pyarrow | `make setup` |
| `ml` | scikit-learn, skl2onnx, onnx, onnxruntime | `make setup` |
| `app` | fastapi, uvicorn, onnxruntime | `make setup` e Docker |
| `dev` | pytest, ruff, httpx, ipykernel | `make setup` |
| `gcp` | google-cloud-bigquery, google-cloud-storage | `make setup-gcp` |

A imagem Docker instala só o projeto + grupo `app`.

---

## Webapp

### API

| Rota | Descrição |
|---|---|
| `GET /api/health` | status e se o modelo foi carregado |
| `GET /api/model` | metadata: features, estatísticas, classes, métricas |
| `POST /api/predict` | `{"features": {...}}` → previsão e probabilidades |
| `POST /api/predict/batch` | `{"instances": [{...}, ...]}` → várias previsões |
| `GET /docs` | documentação interativa (Swagger) |

### Front

O front é gerado a partir do `metadata.json`: o formulário, a análise de sensibilidade e as métricas se adaptam a qualquer modelo de classificação ou regressão sem mudar código.

Gráficos padronizados ficam em `app/frontend/src/charts/`:

- `theme.ts` — **única fonte de cores**, com paleta categórica validada para daltonismo, escala sequencial e modo escuro;
- `Chart.tsx` — componente base (ciclo de vida do ECharts, redimensionamento e visão em tabela);
- `BarChart`, `LineChart` (com amostragem LTTB e zoom para séries longas) e `HeatmapChart`.

Para um novo tipo de gráfico: registre-o em `charts/echarts.ts` e crie o componente em `charts/` usando `baseOption`, `valueAxis` e `categoryAxis` do tema.

---

## Qualidade e CI

```bash
make test     # pytest
make lint     # ruff + checagem de tipos do front
make format   # formata o Python
make check    # o mesmo que a CI
```

A CI (`.github/workflows/ci.yml`) roda lint, o pipeline completo, os testes, o build do front e um smoke test da imagem Docker em cada push na `main` e em pull requests.

---

## Dúvidas comuns

- **Aviso `GetPciBusId` do onnxruntime ao iniciar** (comum em WSL e VMs): é inofensivo; o onnxruntime só não reconhece o barramento PCI virtual.
- **`make` reclama de `uv` ou `npm` não encontrados**: instale o uv (`curl -LsSf https://astral.sh/uv/install.sh | sh`) e o Node 22+ (recomendado via `nvm`).
- **Front mostra "Modelo indisponível"**: rode `make all` para gerar `models/model.onnx` e reinicie a API.
- **Mudei o modelo e o app não mudou**: a API carrega o modelo ao iniciar; reinicie (`make app`) ou refaça a imagem (`make docker`).
