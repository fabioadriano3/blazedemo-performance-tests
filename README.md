# Projeto de Teste de Performance — BlazeDemo (JMeter)

## 🎯 Objetivo
Executar testes de performance do fluxo principal de **compra de passagem aérea com sucesso** no site **BlazeDemo** (`https://www.blazedemo.com`), com:

- **Teste de carga (Load Test)**: sustentado em **250 RPS**
- **Teste de pico (Spike Test)**: subida brusca **0 → 250 → 500 RPS**
- **Critério de aceite**: **p90 < 2s** (percentil 90 do tempo de resposta)

> Importante: **não execute** 250–500 RPS contra ambientes de terceiros sem permissão explícita. Use um ambiente de homologação/staging ou um espelho controlado. Este repositório entrega **os artefatos** e a **estratégia**, e você roda/mede no ambiente autorizado.

---

## 📦 Estrutura do projeto

- `jmeter/test-plans/`
  - `blazedemo_load_250rps.jmx`
  - `blazedemo_spike_0_250_500rps.jmx`
- `jmeter/data/`
  - `routes.csv` (origem/destino)
  - `passengers.csv` (dados do comprador)
- `jmeter/results/` (saída dos testes)

---

## ✅ Pré-requisitos

- **Java 11+** (recomendado Java 17)
- **Apache JMeter 5.6+**
- **JMeter Plugins** (recomendado para controle de RPS por “arrivals/throughput shaping”)

### Plugins necessários (recomendado)
Os planos foram construídos para usar:

- **Concurrency Thread Group** (JMeter Plugins)
- **Throughput Shaping Timer** (JMeter Plugins)

Instalação sugerida:

1. Baixe/instale o **Plugins Manager** do JMeter
2. No JMeter: `Options → Plugins Manager`
3. Instale os componentes:
   - `Concurrency Thread Group`
   - `Throughput Shaping Timer`

> Se você preferir “só JMeter puro”, dá para adaptar para `Thread Group` + `Constant Throughput Timer`, mas a precisão para 250 RPS sustentado costuma ser pior e depende muito do número de threads/latência.

---

## 🧪 O que o script testa (fluxo E2E)

1. **Acessar página inicial** (`GET /`)
2. **Selecionar origem e destino** (`POST /reserve`)
3. **Buscar voos** (retorno da lista em `/reserve`)
4. **Selecionar voo** (`POST /purchase`)
5. **Preencher formulário de compra** (dados vindos de `passengers.csv`)
6. **Confirmar compra** (`POST /confirmation`)
7. **Validar mensagem de sucesso** (assertion “Thank you for your purchase today!”)

---

## ▶️ Como executar (CLI)

> Execute a partir da raiz do projeto para manter os caminhos relativos dos CSVs.

### 1) Load test — 250 RPS sustentado

```bash
jmeter -n \
  -t jmeter/test-plans/blazedemo_load_250rps.jmx \
  -l jmeter/results/load_250rps.jtl \
  -e -o jmeter/results/load_250rps_report
```

### 2) Spike test — 0 → 250 → 500 RPS

```bash
jmeter -n \
  -t jmeter/test-plans/blazedemo_spike_0_250_500rps.jmx \
  -l jmeter/results/spike_0_250_500rps.jtl \
  -e -o jmeter/results/spike_0_250_500rps_report
```

### Propriedades do plano (`-J`)

Os planos usam `${__P(NOME,valor_padrao)}` nas **User Defined Variables**. Você pode sobrescrever na linha de comando com `-J`, por exemplo:

| Propriedade | Exemplo | Uso |
|-------------|---------|-----|
| `TARGET_RPS` | `-JTARGET_RPS=250` | Throughput alvo (load) |
| `RAMP_UP_SECONDS` | `-JRAMP_UP_SECONDS=60` | Ramp-up do Thread Group |
| `HOLD_SECONDS` | `-JHOLD_SECONDS=900` | Duração do teste (load) |
| `THREADS_MAX` | `-JTHREADS_MAX=600` | Teto de threads |
| `BASE_DOMAIN` | `-JBASE_DOMAIN=www.blazedemo.com` | Host alvo |
| `TEST_DURATION_SECONDS` | `-JTEST_DURATION_SECONDS=600` | Duração (spike) |

Exemplo com smoke (similar ao CI):

```bash
jmeter -n \
  -t jmeter/test-plans/blazedemo_load_250rps.jmx \
  -l jmeter/results/smoke.jtl \
  -e -o jmeter/results/smoke_report \
  -JTARGET_RPS=5 \
  -JRAMP_UP_SECONDS=10 \
  -JHOLD_SECONDS=120 \
  -JTHREADS_MAX=40
```

---

## 🔄 CI/CD (GitHub Actions)

O repositório inclui um workflow que instala o **Apache JMeter**, o plugin **Throughput Shaping Timer** (`jpgc-tst`) e executa um **smoke test** (carga reduzida) para validar que o plano `.jmx` e o fluxo HTTP continuam íntegros.

| Item | Detalhe |
|------|---------|
| Arquivo | `.github/workflows/jmeter-ci.yml` |
| Disparo automático | `push` e `pull_request` nas branches `main` e `master` |
| Disparo manual | **Actions** → **JMeter — CI** → **Run workflow** (`workflow_dispatch`) |
| Job | `jmeter-smoke`: load test com `-JTARGET_RPS=5`, `-JHOLD_SECONDS=120`, etc. |
| Artefatos | `jmeter-smoke-<número_do_run>`: `ci_load.jtl` e pasta `ci_load_report/` (HTML) |
| Falha do job | Se `statistics.json` → `Total.errorPct` for maior que zero |

> O smoke na CI **não** substitui o teste de 250 RPS em ambiente autorizado; serve para regressão do script e detecção rápida de quebras (HTTP, assertions, plugin).

### Como executar a pipeline no GitHub

1. Faça push deste repositório para o GitHub (ou importe o projeto).
2. Abra **Actions** no repositório.
3. Selecione o workflow **JMeter — CI**.
4. Para rodar sem commit: **Run workflow** (branch desejada) e confirme.
5. Ao terminar, abra o run → seção **Artifacts** → baixe `jmeter-smoke-*` e abra `ci_load_report/index.html` no navegador.

### Validação local equivalente à CI

Com JMeter e o plugin instalados na sua máquina:

```bash
mkdir -p jmeter/results
jmeter -n \
  -t jmeter/test-plans/blazedemo_load_250rps.jmx \
  -l jmeter/results/ci_load.jtl \
  -e -o jmeter/results/ci_load_report \
  -JTARGET_RPS=5 \
  -JRAMP_UP_SECONDS=10 \
  -JHOLD_SECONDS=120 \
  -JTHREADS_MAX=40
```

---

## 📊 Como abrir o relatório

- Relatório HTML:
  - `jmeter/results/load_250rps_report/index.html`
  - `jmeter/results/spike_0_250_500rps_report/index.html`

Métricas que você deve checar no relatório:
- **Throughput** (req/s)
- **Response Times** (Average, Median, **p90**, p95)
- **Error %**
- **Active Threads Over Time** (se aplicável)

---

## 🧾 Relatório de execução (modelo pronto)

Preencha com os dados do relatório HTML/JTL após a execução.

### Load test (250 RPS por 10–15 min)
- **Throughput médio**: ___ RPS (alvo: **250 RPS**)
- **p90**: ___ ms (alvo: **< 2000 ms**)
- **p95**: ___ ms
- **Erro %**: ___ %
- **Observações**:
  - estabilidade do throughput (variação, quedas)
  - degradação ao longo do tempo (fila/latência crescente)

### Spike test (0 → 250 → 500 RPS)
- **Degrau 250 RPS — p90**: ___ ms
- **Degrau 500 RPS — p90**: ___ ms
- **Erro % no pico**: ___ %
- **Sinais típicos de saturação**:
  - aumento abrupto de latência
  - timeouts
  - HTTP 5xx/429
  - conexão resetada

### ✅ Critério de aceitação atendido?
- **Resposta**: **(Sim/Não)**
- **Justificativa objetiva**:
  - p90 = ___ ms (limite 2000 ms)
  - erro % = ___ %
  - throughput = ___ RPS (alvo 250 RPS)

> Se “Não”: descreva **onde** ocorreu o gargalo (rede, DNS, TLS, app, banco, limites de conexão, CPU/memória, pool de threads, rate limiting).

---

## 🧠 Análise crítica (QA Sr — diferencial)

### Limitações deste teste (o que pode distorcer resultados)
- **Ambiente alvo**: BlazeDemo é um **site de demonstração**; a infraestrutura pode variar, ter rate limiting e não representar um ambiente real.
- **Teste externo**: latência de internet e roteamento influenciam fortemente p90/p95.
- **Gerador de carga**: para 250–500 RPS, **uma única máquina** pode ser insuficiente; use **geração distribuída** (múltiplos load generators) se necessário.
- **Dados e cache**: resultados podem melhorar/piorar conforme cache do servidor/CDN, warm-up, TLS session reuse.

### Melhorias recomendadas
- **Separar cenários**: medir `/` e `/confirmation` separadamente para isolar gargalos.
- **Adicionar métricas de infra/app**: APM, logs, CPU/mem, GC, pool de conexões.
- **Distribuir carga**: JMeter remoto (ou containers) para evitar gargalo no gerador.
- **Revisar limites**: keep-alive, `httpclient4` pool, DNS cache, max connections.

### Sugestões técnicas (se p90 não bater 2s)
- **Cache/CDN** para páginas estáticas
- **Escala horizontal** e autoscaling
- **Otimização de I/O** (DB, chamadas externas), filas assíncronas
- **Tuning de pools** (thread pools / connection pools)
- **Rate limiting** controlado e respostas 429 bem definidas

