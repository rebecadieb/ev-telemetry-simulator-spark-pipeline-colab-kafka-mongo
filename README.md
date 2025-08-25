# EV Telemetry Simulator & Spark Pipeline (Colab/Kafka/Mongo)

## Identificação

**Disciplina:** Z104 Big data e tecnologias de armazenagem

**Curso:** 2289 - Especialização em Engenharia de Dados

**Turma:** 02

**Alunos:**
- Fabio Hemerson Araujo de Souza - Matrícula 2519208
- Rebeca Dieb Holanda Silva - Matrícula 2519094

---

## Descrição do Projeto

Projeto de **simulação de telemetria** de uma frota de veículos elétricos (EV) e **pipeline batch** com Apache **Spark** para padronizar, validar, enriquecer e gravar dados em **Parquet** (com *schema evolution* simples).  

> O projeto **suporta Kafka**, mas **nesta implementação** a simulação foi executada usando **sink=`file`** (gerando `.jsonl`), e **após o pipeline** os **dados processados são salvos em Parquet**.

---

## ✨ Principais recursos
- **Simulador** de EVs (posições, temperatura, SOC, etc.) com ruído controlado e distribuição de SOC enviesada à esquerda (poucos casos baixos).
- **Sinks suportados**: `stdout`, arquivo `.jsonl` (**usado nesta execução**), `http`, **Kafka** (opcional).
- **Alertas** (opcional): quando `metrics.soc_pct < threshold`, podem ser enviados ao **Kafka** e/ou gravados no **MongoDB** (`ev_alerts`).
- **Pipeline Spark** (batch): leitura (JSON/CSV/Parquet), padronização, flags de qualidade, quarantine, enriquecimento, escrita **Parquet particionado** (`ano/mes`) + metadados em `_metadata/`.
- **Métricas no Mongo** (opcional): `pipeline_runs` (execução do job) e `ev_metrics_daily` (resumo diário por veículo).

---

## 🧱 Arquitetura (resumo)
- **Simulador (sink=file)** -> grava **JSONL** em `/content/simulation_output/`.
- **Batch Spark** lê esse **JSONL**, aplica qualidade/enriquecimento e escreve **Parquet** processado em `/content/pipeline_output/data_parquet/`.
- (Opcional) **Mongo** para alertas e métricas de execução.
- (Opcional) **Kafka** para telemetria/alertas — não utilizado nesta execução.

---

## 🚀 Quickstart (Colab — execução utilizada)

### 1) Dependências
```bash
!pip -q install pymongo confluent-kafka
```

### 2) (Opcional) Subir **MongoDB** local na VM do Colab (sem apt/GPG)
```bash
%%bash
set -euo pipefail
mkdir -p /content/mongodb/data /content/mongodb/log
urls=(
  "https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu2204-7.0.14.tgz"
  "https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu2204-7.0.13.tgz"
  "https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu2204-7.0.12.tgz"
  "https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu2204-6.0.18.tgz"
  "https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu2004-6.0.18.tgz"
); ok=""
for u in "${urls[@]}"; do
  echo ">> tentando $u"
  if wget -q "$u" -O /content/mongodb.tgz; then
    tar -xzf /content/mongodb.tgz -C /content/
    ok="$(find /content -maxdepth 1 -type d -name 'mongodb-linux-*' | head -n1)"
    break
  fi
done
ln -sfn "$ok/bin/mongod" /content/mongodb/mongod
nohup /content/mongodb/mongod --dbpath /content/mongodb/data --bind_ip 127.0.0.1 --port 27017   --logpath /content/mongodb/log/mongod.log --wiredTigerCacheSizeGB 0.25 >/dev/null 2>&1 &
sleep 2
pgrep -a mongod || { echo "mongod NAO esta rodando"; tail -n 50 /content/mongodb/log/mongod.log; exit 1; }
echo "OK mongod em 127.0.0.1:27017"
```

Sanity check:
```python
from pymongo import MongoClient
MongoClient("mongodb://127.0.0.1:27017", serverSelectionTimeoutMS=3000).admin.command("ping")
```

### 3) Simulador -> FILE (telemetria em JSONL)
> Esta é a forma usada nesta execução. Kafka permanece como opção futura.
```python
simular_frota(
    vehicles=10, hz=2, duration=30,
    sink="file",
    file_path="/content/simulation_output/in.jsonl",
    # (opcional) salvar alertas no Mongo:
    alert_threshold=18.0,
    enable_mongo_alerts=True,
    mongo_uri="mongodb://127.0.0.1:27017",
    mongo_db="telemetria",
    mongo_collection="ev_alerts",
    mongo_ttl_days=30,
)
```

### 4) Pipeline Spark -> Parquet processado
```python
out = run_job(
    input_path="/content/simulation_output/in.jsonl",
    output_path="/content/pipeline_output/data_parquet",
    input_format="json",
    mode="append",

    # (opcional) métricas no Mongo
    save_metrics_to_mongo=True,
    mongo_uri="mongodb://127.0.0.1:27017",
    mongo_db="telemetria",
    mongo_coll_job="pipeline_runs",
    mongo_coll_daily="ev_metrics_daily",
    mongo_ttl_days=30,
)
print(out["metrics"])
```

### 5) Conferir coleções no Mongo
```python
from pymongo import MongoClient
cli = MongoClient("mongodb://127.0.0.1:27017")
print(cli.list_database_names())
print(cli["telemetria"].list_collection_names())
for d in cli["telemetria"]["pipeline_runs"].find({}, {"_id":0,"job_name":1,"records_clean":1}).sort("job_start_utc",-1).limit(3): print(d)
for d in cli["telemetria"]["ev_metrics_daily"].find({}, {"_id":0,"event_date":1,"vehicle_id":1,"rows":1}).sort([("event_date",-1)]).limit(3): print(d)
cli.close()
```

---

## 📊 Leitura & análise com Spark (SOC)
```python
from pyspark.sql import SparkSession, functions as F

spark = (SparkSession.builder
         .appName("check-soc")
         .config("spark.sql.parquet.mergeSchema","true")
         .getOrCreate())

df = (spark.read
      .option("recursiveFileLookup","true")
      .option("pathGlobFilter","*.parquet")
      .parquet("/content/pipeline_output"))

df_soc = df.select("vehicle_id","event_ts_utc", F.col("soc_pct").alias("soc"))
df_soc.select(
    F.count("*").alias("n"),
    F.min("soc").alias("min"),
    F.expr("percentile_approx(soc,0.5)").alias("p50"),
    F.expr("percentile_approx(soc,0.95)").alias("p95"),
    F.max("soc").alias("max")
).show()
```

---

## 💾 Notas de armazenamento
- **Arquivo (JSONL)**: usado como fonte nesta execução para facilitar o desenvolvimento.
- **Parquet**: saída do pipeline com dados processados; mantenha **somente** `.parquet` na pasta `data_parquet/`. Metadados ficam em `_metadata/`.
- **Mongo** (opcional): TTL nos alertas (`ev_alerts`) e/ou nos runs (`pipeline_runs`) para limpeza automática (~a cada 60s).
- **Kafka** (opcional): plataforma de streaming; útil para produção/escala, mas não aplicado nesta execução.

---

## 🧩 Estrutura sugerida (no Colab)

```
/content/
  simulation_output/           # dumps JSONL de telemetria (sink=file)
  pipeline_output/
    data_parquet/              # APENAS arquivos .parquet (dados processados)
    _metadata/
      dicionario_dados.json
      exec_log.json
```

---
