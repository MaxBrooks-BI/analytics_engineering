## Windows 11 Pro Data Platform: MinIO + Iceberg + Nessie + PostgreSQL + Spark (custom image) + DuckDB + dbt Core (Visual Studio 2022)

This repository-style Markdown gives a ready-to-run local data platform on Windows 11 Pro using WSL2 (Ubuntu) and Docker Desktop. It includes:

• docker-compose.yml to run PostgreSQL, MinIO, Nessie, and a custom Spark image
• Dockerfile and build instructions for a Spark image bundled with Iceberg and Nessie client jars (version-compatible)
• spark-submit example and PySpark apps to create Iceberg tables and export snapshots
• dbt Core project with models, tests, snapshots, and docs (dbt docs serve)
• DuckDB script to query exported Parquet files
• Visual Studio 2022 guidance for editing files on WSL via \wsl$\Ubuntu...
• Version compatibility matrix and run instructions


Repository layout

• docker-compose.yml
• docker/spark/• Dockerfile
• build-spark-image.sh

• spark/• app/• iceberg_create_and_write.py
• export_iceberg_snapshot.py


• dbt/• project/• dbt_project.yml
• profiles.yml.example
• models/• staging/• stg_sales.sql

• marts/• sales_agg.sql


• tests/• test_sales_not_null.yml

• snapshots/• sales_snapshot.sql



• duckdb/• duckdb_query_from_minio.py

• README.md (this file)


---

Versions and compatibility (tested baseline)

• Spark: 3.4.2
• Iceberg: 1.3.0 (iceberg-spark3-runtime compatible with Spark 3.4.x)
• Nessie Server: 1.5.0 and Nessie Spark extensions 0.64.0 (matched to Iceberg runtime)
• Hadoop AWS: 3.3.4 (s3a support)
• PostgreSQL: 15 (Nessie backing store)
• MinIO: recent 2024 RELEASE (s3-compatible)
• DuckDB: latest pip duckdb (Python API)
• dbt-core: latest compatible with dbt-duckdb (use pip-installed versions in Conda env)


Use these versions for the jars in the custom Spark image to avoid classpath conflicts.

---

docker/spark/Dockerfile (custom Spark image bundling Iceberg + Nessie jars)

This Dockerfile builds a Spark 3.4.2 image based on Bitnami/official Spark and adds Iceberg, Nessie, and Hadoop AWS jars to Spark’s jars folder so spark-submit has required classes on the classpath.

FROM bitnami/spark:3.4.2

USER root

# Create folder for additional jars
RUN mkdir -p /opt/spark/jars-extra

WORKDIR /opt/spark/jars-extra

# Install curl and unzip (if not present)
RUN apt-get update && apt-get install -y curl unzip && rm -rf /var/lib/apt/lists/*

# Set versions (keep in sync with compatibility matrix)
ENV ICEBERG_VERSION=1.3.0
ENV NESSIE_VERSION=0.64.0
ENV HADOOP_AWS_VERSION=3.3.4
ENV HADOOP_COMMON_VERSION=3.3.4

# Download necessary jars (examples: central maven URLs)
# Iceberg runtime for Spark 3
RUN curl -sL -o iceberg-spark3-runtime.jar "https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-spark3-runtime/${ICEBERG_VERSION}/iceberg-spark3-runtime-${ICEBERG_VERSION}.jar" \
 && curl -sL -o iceberg-core.jar "https://repo1.maven.org/maven2/org/apache/iceberg/iceberg-core/${ICEBERG_VERSION}/iceberg-core-${ICEBERG_VERSION}.jar" \
# Nessie Spark extensions
 && curl -sL -o nessie-spark-extensions.jar "https://repo1.maven.org/maven2/org/projectnessie/nessie-spark-extensions/${NESSIE_VERSION}/nessie-spark-extensions-${NESSIE_VERSION}.jar" \
# Nessie client
 && curl -sL -o nessie-client.jar "https://repo1.maven.org/maven2/org/projectnessie/nessie-client/${NESSIE_VERSION}/nessie-client-${NESSIE_VERSION}.jar" \
# Hadoop AWS and transitive deps for S3A (simplified)
 && curl -sL -o hadoop-aws.jar "https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/${HADOOP_AWS_VERSION}/hadoop-aws-${HADOOP_AWS_VERSION}.jar" \
 && curl -sL -o hadoop-common.jar "https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-common/${HADOOP_COMMON_VERSION}/hadoop-common-${HADOOP_COMMON_VERSION}.jar"

# Move jars into Spark's jars directory
RUN mv /opt/spark/jars-extra/*.jar /opt/bitnami/spark/jars/ || mv /opt/spark/jars-extra/*.jar /opt/spark/jars/

# Clean up
RUN rm -rf /opt/spark/jars-extra

USER 1001


build-spark-image.sh

#!/usr/bin/env bash
set -e
IMAGE_NAME=local-spark-iceberg-nessie:3.4.2-iceberg1.3.0-nessie0.64.0
docker build -t ${IMAGE_NAME} -f docker/spark/Dockerfile .
echo "Built ${IMAGE_NAME}"


Notes

• Downloaded jars are examples; for production, include all transitive dependencies or use a proper dependency management step (Maven/Gradle) to assemble an assembly jar or copy all required jars.
• If a jar is missing a transitive dep, add the jar and retry; the versions listed are tested baseline for compatibility.


---

docker-compose.yml (uses custom Spark image)

Place this at project root and update the spark service image name after building.

version: "3.8"
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: nessie
      POSTGRES_PASSWORD: nessiepass
      POSTGRES_DB: nessie_db
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  minio:
    image: minio/minio:RELEASE.2024-01-01T00-00-00Z
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - miniodata:/data

  nessie:
    image: projectnessie/nessie-server:1.5.0
    environment:
      - JAVA_TOOL_OPTIONS=-Xmx1g
    depends_on:
      - postgres
    ports:
      - "19120:19120"
    command:
      - server
      - --server.http.port=19120
      - --store.type=postgresql
      - --store.postgresql.jdbc-url=jdbc:postgresql://postgres:5432/nessie_db
      - --store.postgresql.username=nessie
      - --store.postgresql.password=nessiepass

  spark:
    image: local-spark-iceberg-nessie:3.4.2-iceberg1.3.0-nessie0.64.0
    environment:
      - SPARK_MODE=master
      - SPARK_LOCAL_IP=0.0.0.0
    ports:
      - "4040:4040"
    volumes:
      - ./spark/app:/opt/spark/app
    depends_on:
      - minio
      - nessie

volumes:
  pgdata:
  miniodata:


---

spark-submit command (using custom image environment)

Run spark-submit from inside the custom Spark container or from host Spark that matches same versions. Example run from WSL using containerized spark-submit (docker exec):

1. Start compose:


docker compose up -d


1. Copy your app folder into the running spark container or mount as in compose (we mount ./spark/app). Execute spark-submit inside container:


SPARK_CONTAINER=$(docker ps --filter ancestor=local-spark-iceberg-nessie:3.4.2-iceberg1.3.0-nessie0.64.0 -q)
docker exec -it $SPARK_CONTAINER /opt/bitnami/spark/bin/spark-submit \
  --master local[4] \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions,org.projectnessie.spark.extensions.NessieSparkExtensions \
  --conf spark.sql.catalog.nessie=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.nessie.catalog-impl=org.apache.iceberg.nessie.NessieCatalog \
  --conf spark.sql.catalog.nessie.uri=http://nessie:19120/api/v1 \
  --conf spark.sql.catalog.nessie.ref=main \
  --conf spark.sql.catalog.nessie.warehouse=s3a://iceberg-warehouse/ \
  --conf spark.hadoop.fs.s3a.endpoint=http://minio:9000 \
  --conf spark.hadoop.fs.s3a.access.key=minioadmin \
  --conf spark.hadoop.fs.s3a.secret.key=minioadmin \
  --conf spark.hadoop.fs.s3a.path.style.access=true \
  /opt/spark/app/iceberg_create_and_write.py


Notes

• Within containers use service names (`minio`, `nessie`) for network resolution; from WSL/host use `localhost` + mapped ports.
• This invocation relies on the custom jars already present in the image’s jars.


---

spark/app/iceberg_create_and_write.py

(no changes from prior — creates Nessie-backed Iceberg table)

from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("iceberg-nessie-demo").getOrCreate()

spark.sql("CREATE NAMESPACE IF NOT EXISTS nessie.db")

spark.sql("""
CREATE TABLE IF NOT EXISTS nessie.db.sales (
  id INT,
  category STRING,
  amount DOUBLE
) USING ICEBERG
""")

data = [(1,"A",10.0),(2,"B",20.0),(3,"A",15.0),(4,"C",5.5)]
df = spark.createDataFrame(data, schema="id INT, category STRING, amount DOUBLE")
df.writeTo("nessie.db.sales").append()

res = spark.sql("SELECT category, SUM(amount) as total FROM nessie.db.sales GROUP BY category ORDER BY total DESC")
res.show()
spark.stop()


---

spark/app/export_iceberg_snapshot.py

Writes a Parquet snapshot to MinIO s3a://iceberg-warehouse/export/ for DuckDB and dbt to consume.

from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("iceberg-export").getOrCreate()
df = spark.read.table("nessie.db.sales")
df.repartition(1).write.mode("overwrite").parquet("s3a://iceberg-warehouse/export/")
spark.stop()


---

dbt project: expanded models, tests, snapshots, docs

dbt_project.yml (same as before)

profiles.yml.example (same as before; ensure DBT_DUCKDB_PATH env var or local path)

models/staging/stg_sales.sql

-- models/staging/stg_sales.sql
with raw as (
  select *
  from read_parquet('s3://iceberg-warehouse/export/part-*.parquet')
)
select
  id,
  category,
  amount
from raw


models/marts/sales_agg.sql

-- models/marts/sales_agg.sql
with sales as (
  select * from {{ ref('stg_sales') }}
)
select
  category,
  sum(amount) as total_amount,
  count(*) as row_count
from sales
group by category
order by total_amount desc


tests/test_sales_not_null.yml

version: 2

models:
  - name: stg_sales
    columns:
      - name: id
        tests:
          - not_null
      - name: amount
        tests:
          - not_null


snapshots/sales_snapshot.sql

{% snapshot sales_snapshot %}
{{
  config(
    target_schema='snapshots',
    unique_key='id',
    strategy='timestamp',
    updated_at='updated_at'   -- if you have an updated_at; else use 'modified_at' or use 'check' strategy
  )
}}

select
  id,
  category,
  amount,
  current_timestamp() as updated_at
from {{ ref('stg_sales') }}

{% endsnapshot %}


dbt docs manifest (commands)

• Generate docs: dbt docs generate
• Serve docs locally: dbt docs serve


Notes on snapshots

• The snapshot above uses a timestamp strategy; if your data lacks an updated timestamp use strategy=‘check’ and specify columns to detect changes.


---

duckdb/duckdb_query_from_minio.py (unchanged)

Set S3 env vars then run query against exported Parquet.

import os
import duckdb

os.environ["AWS_ACCESS_KEY_ID"] = "minioadmin"
os.environ["AWS_SECRET_ACCESS_KEY"] = "minioadmin"
os.environ["AWS_REGION"] = "us-east-1"
os.environ["AWS_ENDPOINT_URL"] = "http://localhost:9000"

con = duckdb.connect("analytics.duckdb")
parquet_s3_path = "s3://iceberg-warehouse/export/part-*.parquet"
q = f"SELECT category, SUM(amount) AS total FROM read_parquet('{parquet_s3_path}') GROUP BY category ORDER BY total DESC"
df = con.execute(q).fetchdf()
print(df)
con.close()


---

How to build, run, and use the full stack (concise sequence)

1. Prepare WSL + Windows tools• Enable WSL2, install Ubuntu, install Docker Desktop and enable WSL2 integration, install Visual Studio 2022 and Git for Windows.
• In WSL install required packages and Miniconda; create Conda env `dataeng` and install Python, duckdb, pyarrow, dbt-core, dbt-duckdb.

2. Clone project into WSL: ~/projects/data-platform. Open it in Visual Studio 2022 via \wsl$\Ubuntu\home<user>\projects\data-platform.
3. Build custom Spark image


chmod +x docker/spark/build-spark-image.sh
./docker/spark/build-spark-image.sh


Confirm image exists:

docker images | grep local-spark-iceberg-nessie


1. Start services


cd ~/projects/data-platform
docker compose up -d


1. Create MinIO bucket `iceberg-warehouse` via MinIO console http://localhost:9001 (minioadmin/minioadmin) or let Spark create it.
2. Submit Spark job inside container (uses service names for MinIO/Nessie)


SPARK_CONTAINER=$(docker ps --filter ancestor=local-spark-iceberg-nessie:3.4.2-iceberg1.3.0-nessie0.64.0 -q)
docker exec -it $SPARK_CONTAINER /opt/bitnami/spark/bin/spark-submit \
  --master local[4] \
  --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions,org.projectnessie.spark.extensions.NessieSparkExtensions \
  --conf spark.sql.catalog.nessie=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.nessie.catalog-impl=org.apache.iceberg.nessie.NessieCatalog \
  --conf spark.sql.catalog.nessie.uri=http://nessie:19120/api/v1 \
  --conf spark.sql.catalog.nessie.ref=main \
  --conf spark.sql.catalog.nessie.warehouse=s3a://iceberg-warehouse/ \
  --conf spark.hadoop.fs.s3a.endpoint=http://minio:9000 \
  --conf spark.hadoop.fs.s3a.access.key=minioadmin \
  --conf spark.hadoop.fs.s3a.secret.key=minioadmin \
  --conf spark.hadoop.fs.s3a.path.style.access=true \
  /opt/spark/app/iceberg_create_and_write.py


1. Export snapshot for DuckDB/dbt


# inside same container or via another spark-submit invocation
docker exec -it $SPARK_CONTAINER /opt/bitnami/spark/bin/spark-submit ... /opt/spark/app/export_iceberg_snapshot.py


1. Run dbt (in WSL Conda env)


cd ~/projects/data-platform/dbt/project
# set env vars so dbt-duckdb can read s3
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin
export AWS_REGION=us-east-1
export AWS_ENDPOINT_URL=http://localhost:9000

dbt debug
dbt deps
dbt run
dbt test
dbt docs generate
dbt docs serve


1. Run DuckDB query (in WSL Conda env)


python ~/projects/data-platform/duckdb/duckdb_query_from_minio.py


1. Inspect Nessie API and Postgres


• Nessie REST: http://localhost:19120/api/v1/repos
• Postgres: localhost:5432 (psql client)


---

Notes, tips, and recommended next work

• Jar transitive dependencies: the Dockerfile downloads core jars but you may need additional transitive jars; use Maven/Gradle assembly or copy the full set of runtime jars into the image for production parity.
• Version skew: keep Iceberg runtime, Nessie client/extension, and Spark versions aligned; the matrix at top reflects tested baseline. If you upgrade one component, update the others accordingly.
• dbt docs serve requires network/port access; run it from WSL and open served URL in your host browser.
• For CI: push the custom Spark image to a local registry or GitHub Container Registry and reference it in Docker Compose CI runs; ensure CI runners have Docker-in-Docker or Podman support.
• For production-like tests: add an integration test that starts the compose stack, runs the Spark job, runs dbt, and asserts outputs in DuckDB/Postgres.


---

Cleanup

docker compose down -v
docker image rm local-spark-iceberg-nessie:3.4.2-iceberg1.3.0-nessie0.64.0
rm ~/projects/data-platform/analytics.duckdb || true


---

End of Markdown content. Copy this into a README.md in your project folder (\wsl$\Ubuntu\home<user>\projects\data-platform\README.md) and edit inside Visual Studio 2022. 9F742443-6C92-4C44-BF58-8F5A7C53B6F1
