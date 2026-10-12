qqqqqqqqqqqqqqqqq# Scripts de la práctica Hadoop HDFS — Grupo 01

Este documento reúne en un solo archivo Markdown los seis scripts Bash utilizados en la práctica. Cada bloque corresponde a un archivo `.sh` independiente: copia su contenido en la ruta indicada dentro de la carpeta `scripts/` del proyecto.

> **Importante:** este `.md` sirve como documentación y contenedor del código. No ejecutes todo el documento como si fuera un único script. Guarda cada bloque con el nombre indicado y ejecútalos desde la raíz del proyecto.

## Requisitos previos

- Docker Engine o Docker Desktop con el comando `docker compose` disponible.
- Bash en Linux, macOS, Git Bash o WSL.
- El archivo `docker-compose.yml` y `hadoop.env` de este proyecto en la raíz.
- Espacio de almacenamiento suficiente para generar datasets de 1 GiB y 10 GiB y guardar las réplicas.

## Orden de ejecución

Desde la raíz del proyecto:

```bash
bash scripts/00_iniciar_clúster.sh
bash scripts/01_preparar_hdfs.sh
bash scripts/02_crear_datasets.sh
bash scripts/03_estado_inicial.sh

# Prueba de fallo para el dataset de 1 GiB
bash scripts/04_prueba_fallo.sh 1GB
bash scripts/05_restaurar.sh

# Prueba de fallo para el dataset de 10 GiB
bash scripts/04_prueba_fallo.sh 10GB
bash scripts/05_restaurar.sh
```

La prueba de fallo detiene `bigdata-worker2` y lo deja detenido a propósito para registrar el estado durante la falla. Ejecuta el script de restauración antes de iniciar otra prueba.

---

## 1. `scripts/00_iniciar_clúster.sh`

Valida la configuración de Docker Compose, levanta el clúster y muestra el estado de los contenedores.

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "$ROOT_DIR"

echo "[Grupo 01] Validando configuración de Docker Compose..."
docker compose config

echo "[Grupo 01] Iniciando NameNode y tres DataNodes..."
docker compose up -d

echo "[Grupo 01] Estado actual de los contenedores:"
docker compose ps

echo "Cuando el clúster esté listo, abre http://localhost:9870 y ejecuta scripts/01_preparar_hdfs.sh."
```

## 2. `scripts/01_preparar_hdfs.sh`

Espera a que HDFS salga del modo seguro y crea los directorios para los datasets, resultados y evidencias.

```bash
#!/usr/bin/env bash
set -euo pipefail

MASTER="${MASTER_CONTAINER:-bigdata-master}"

if ! docker inspect -f '{{.State.Running}}' "$MASTER" 2>/dev/null | grep -q true; then
  echo "ERROR: el contenedor $MASTER no está ejecutándose. Inicia el clúster primero." >&2
  exit 1
fi

echo "[Grupo 01] Esperando a que HDFS salga del modo seguro, si corresponde..."
docker exec "$MASTER" bash -lc 'hdfs dfsadmin -safemode wait'

echo "[Grupo 01] Creando directorios de la práctica en HDFS..."
docker exec "$MASTER" bash -lc 'hdfs dfs -mkdir -p /grupo01/dataset_1GB /grupo01/dataset_10GB /grupo01/resultados /grupo01/evidencias'

echo "[Grupo 01] Verificando directorios y nodos..."
docker exec "$MASTER" bash -lc 'hdfs dfs -ls -h /grupo01 && hdfs dfsadmin -report'
```

## 3. `scripts/02_crear_datasets.sh`

Genera datasets de 1 GiB y 10 GiB, los copia a HDFS, solicita un factor de replicación de 2 y elimina las copias temporales del contenedor maestro.

```bash
#!/usr/bin/env bash
set -euo pipefail

MASTER="${MASTER_CONTAINER:-bigdata-master}"

if ! docker inspect -f '{{.State.Running}}' "$MASTER" 2>/dev/null | grep -q true; then
  echo "ERROR: el contenedor $MASTER no está ejecutándose." >&2
  exit 1
fi

create_upload_dataset() {
  local label="$1"
  local count_mib="$2"
  local tmp="/tmp/dataset_grupo01_${label}"
  local hdfs_path="/grupo01/dataset_${label}/dataset_grupo01_${label}"

  echo
  echo "[Grupo 01] Creando dataset ${label} (${count_mib} MiB) en el contenedor maestro..."
  docker exec "$MASTER" bash -lc "dd if=/dev/zero of='${tmp}' bs=1M count=${count_mib} status=progress"

  echo "[Grupo 01] Cargando ${label} a HDFS: ${hdfs_path}"
  docker exec "$MASTER" bash -lc "hdfs dfs -put -f '${tmp}' '${hdfs_path}'"

  echo "[Grupo 01] Solicitando factor de replicación 2 y esperando las réplicas..."
  docker exec "$MASTER" bash -lc "hdfs dfs -setrep -w 2 '${hdfs_path}'"

  echo "[Grupo 01] Verificando archivo cargado..."
  docker exec "$MASTER" bash -lc "hdfs dfs -ls -h '${hdfs_path}'"

  echo "[Grupo 01] Eliminando la copia temporal local del contenedor..."
  docker exec "$MASTER" bash -lc "rm -f '${tmp}'"
}

# 1024 MiB = 1 GiB; 10240 MiB = 10 GiB.
create_upload_dataset "1GB" 1024
create_upload_dataset "10GB" 10240

echo
echo "[Grupo 01] Datasets cargados. Revisa el estado con scripts/03_estado_inicial.sh."
```

## 4. `scripts/03_estado_inicial.sh`

Guarda en `resultados/` el reporte de DataNodes y la ubicación de los bloques de ambos datasets antes de detener un DataNode.

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
MASTER="${MASTER_CONTAINER:-bigdata-master}"
OUT_DIR="$ROOT_DIR/resultados"
STAMP="$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUT_DIR"

run_capture() {
  local label="$1"
  local command="$2"
  local outfile="$OUT_DIR/${STAMP}_inicial_${label}.txt"
  echo "[Grupo 01] Guardando ${label} en ${outfile}"
  docker exec "$MASTER" bash -lc "$command" 2>&1 | tee "$outfile"
}

run_capture "dfsadmin_report" 'hdfs dfsadmin -report'
run_capture "fsck_1GB" 'hdfs fsck /grupo01/dataset_1GB/dataset_grupo01_1GB -files -blocks -locations'
run_capture "fsck_10GB" 'hdfs fsck /grupo01/dataset_10GB/dataset_grupo01_10GB -files -blocks -locations'

echo "[Grupo 01] Estado inicial capturado. No olvides conservar también capturas de pantalla en evidencias/."
```

## 5. `scripts/04_prueba_fallo.sh`

Detiene un DataNode (por defecto `bigdata-worker2`), espera a que el NameNode detecte la caída, registra la ubicación de los bloques, comprueba si el archivo sigue siendo legible y captura el estado posterior a una espera.

Uso: `bash scripts/04_prueba_fallo.sh 1GB` o `bash scripts/04_prueba_fallo.sh 10GB`.

```bash
#!/usr/bin/env bash
set -uo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
MASTER="${MASTER_CONTAINER:-bigdata-master}"
FAILED_CONTAINER="${FAILED_CONTAINER:-bigdata-worker2}"
OUT_DIR="$ROOT_DIR/resultados"
STAMP="$(date +%Y%m%d_%H%M%S)"
REPLICATION_WAIT_SECONDS="${REPLICATION_WAIT_SECONDS:-60}"
SIZE="${1:-}"

case "$SIZE" in
  1GB) HDFS_PATH="/grupo01/dataset_1GB/dataset_grupo01_1GB" ;;
  10GB) HDFS_PATH="/grupo01/dataset_10GB/dataset_grupo01_10GB" ;;
  *) echo "Uso: bash scripts/04_prueba_fallo.sh 1GB|10GB" >&2; exit 2 ;;
esac

mkdir -p "$OUT_DIR"
if ! docker inspect -f '{{.State.Running}}' "$MASTER" 2>/dev/null | grep -q true; then
  echo "ERROR: el contenedor $MASTER no está ejecutándose." >&2
  exit 1
fi
if ! docker inspect -f '{{.State.Running}}' "$FAILED_CONTAINER" 2>/dev/null | grep -q true; then
  echo "ERROR: $FAILED_CONTAINER ya está detenido. Ejecuta scripts/05_restaurar.sh antes de otra prueba." >&2
  exit 1
fi

capture() {
  local name="$1"
  local cmd="$2"
  local outfile="$OUT_DIR/${STAMP}_falla_${SIZE}_${name}.txt"
  echo
  echo "[Grupo 01] Capturando ${name}: ${outfile}"
  docker exec "$MASTER" bash -lc "$cmd" 2>&1 | tee "$outfile"
  return 0
}

echo "[Grupo 01] Deteniendo únicamente $FAILED_CONTAINER para la prueba de ${SIZE}..."
docker stop "$FAILED_CONTAINER" || { echo "ERROR: no se pudo detener $FAILED_CONTAINER" >&2; exit 1; }

echo "[Grupo 01] Esperando a que el NameNode detecte la caída (máximo 240 s)..."
detected=false
deadline=$((SECONDS + 240))
while (( SECONDS < deadline )); do
  report="$(docker exec "$MASTER" bash -lc 'hdfs dfsadmin -report' 2>&1)"
  if printf '%s\n' "$report" | grep -Fq 'Live datanodes (2):'; then
    detected=true
    break
  fi
  sleep 5
done

printf '%s\n' "$report" | tee "$OUT_DIR/${STAMP}_falla_${SIZE}_dfsadmin_report.txt"
if [[ "$detected" != true ]]; then
  echo "AVISO: no se confirmó en 240 s la línea 'Live datanodes (2)'. Revisa el reporte y los logs antes de concluir." | tee "$OUT_DIR/${STAMP}_falla_${SIZE}_aviso.txt"
else
  echo "[Grupo 01] El reporte muestra dos DataNodes activos."
fi

capture "fsck_durante" "hdfs fsck '${HDFS_PATH}' -files -blocks -locations"

read_out="$OUT_DIR/${STAMP}_falla_${SIZE}_lectura.txt"
echo "[Grupo 01] Comprobando si el archivo completo sigue siendo legible..."
if docker exec "$MASTER" bash -lc "hdfs dfs -cat '${HDFS_PATH}' > /dev/null"; then
  echo "LECTURA_OK: el archivo ${SIZE} se pudo leer durante la falla." | tee "$read_out"
else
  echo "LECTURA_ERROR: no se pudo leer el archivo ${SIZE} durante la falla." | tee "$read_out"
fi

if [[ "$REPLICATION_WAIT_SECONDS" =~ ^[0-9]+$ ]] && (( REPLICATION_WAIT_SECONDS > 0 )); then
  echo "[Grupo 01] Esperando ${REPLICATION_WAIT_SECONDS} s para observar una posible recuperación de réplicas..."
  sleep "$REPLICATION_WAIT_SECONDS"
fi

capture "fsck_despues_espera" "hdfs fsck '${HDFS_PATH}' -files -blocks -locations"
capture "dfsadmin_despues_espera" 'hdfs dfsadmin -report'

echo
echo "PRUEBA FINALIZADA. $FAILED_CONTAINER queda detenido a propósito para conservar el estado de falla."
echo "Ejecuta 'bash scripts/05_restaurar.sh' antes de iniciar otra prueba."
echo "Los reportes se guardaron en: $OUT_DIR"
```

## 6. `scripts/05_restaurar.sh`

Vuelve a iniciar el DataNode detenido y espera a que HDFS muestre tres DataNodes activos.

```bash
#!/usr/bin/env bash
set -uo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
MASTER="${MASTER_CONTAINER:-bigdata-master}"
RESTORE_CONTAINER="${RESTORE_CONTAINER:-bigdata-worker2}"
OUT_DIR="$ROOT_DIR/resultados"
STAMP="$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUT_DIR"

echo "[Grupo 01] Iniciando de nuevo $RESTORE_CONTAINER..."
if ! docker start "$RESTORE_CONTAINER"; then
  echo "ERROR: no se pudo iniciar $RESTORE_CONTAINER. Revisa 'docker compose ps' y los logs." >&2
  exit 1
fi

echo "[Grupo 01] Esperando a que HDFS muestre tres DataNodes activos (máximo 240 s)..."
report=""
restored=false
deadline=$((SECONDS + 240))
while (( SECONDS < deadline )); do
  report="$(docker exec "$MASTER" bash -lc 'hdfs dfsadmin -report' 2>&1)"
  if printf '%s\n' "$report" | grep -Fq 'Live datanodes (3):'; then
    restored=true
    break
  fi
  sleep 5
done

printf '%s\n' "$report" | tee "$OUT_DIR/${STAMP}_restauracion_dfsadmin_report.txt"
if [[ "$restored" == true ]]; then
  echo "[Grupo 01] Los tres DataNodes aparecen activos. Comprueba fsck para verificar la recuperación completa de las réplicas."
else
  echo "AVISO: no se confirmó el retorno a tres DataNodes en 240 s. Consulta los logs y el panel del NameNode." >&2
  exit 1
fi
```

---

## Archivos que generan los scripts

Los reportes se guardan en `resultados/` con una marca de fecha y hora en el nombre. Conserva esos `.txt` y añade capturas de pantalla verificables en `evidencias/`. Los datos sobre número de bloques, ubicaciones de réplicas, legibilidad y recuperación deben completarse con los resultados de tu propio clúster; no deben inventarse.

## Notas importantes

- El contenedor DataNode que se detiene por defecto es `bigdata-worker2`.
- Para usar otro contenedor durante la prueba, puedes definir la variable antes de ejecutar el script, por ejemplo: `FAILED_CONTAINER=bigdata-worker3 bash scripts/04_prueba_fallo.sh 1GB`.
- Para cambiar el tiempo de espera para observar la recuperación: `REPLICATION_WAIT_SECONDS=120 bash scripts/04_prueba_fallo.sh 1GB`.
- El script de fallo no restaura automáticamente el DataNode; esto permite mantener el estado de falla mientras se guardan las evidencias. Ejecuta `scripts/05_restaurar.sh` después de cada prueba.
