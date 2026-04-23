<div align="center">
# Prediccion de ACV

TP Final de la materia MLOps — Posgrado en Inteligencia Artificial (CEIA, UBA).

Pipeline end-to-end para predecir la probabilidad de accidente cerebrovascular (ACV/stroke) usando Regresion Logistica con SMOTE.

## Integrantes
- Mauro Virgilio Blanc
- Juan Pablo Imbrogno
- Sofía Belén Caselli
- Miguel Angel Leiva Martinez
- Andrea Viviana Ferenaz.

## Arquitectura

```
MinIO (s3)   <- dataset de entrada (bucket: data)
     |
Airflow      <- orquesta el pipeline de entrenamiento (DAG: train_stroke_model)
     |
MLflow       <- registra experimentos y versiona el modelo (bucket: mlflow)
     |
FastAPI      <- sirve predicciones en tiempo real (puerto 8800)
```

Servicios de soporte: PostgreSQL (backend de Airflow y MLflow), Redis (broker de Celery).

## Stack tecnologico

| Servicio   | Imagen / Framework           | Puerto          |
|------------|------------------------------|-----------------|
| Airflow    | custom (CeleryExecutor)      | 8080            |
| MLflow     | custom                       | 5000            |
| MinIO      | minio/minio:latest           | 9000 / 9001 (UI)|
| FastAPI    | custom (uvicorn)             | 8800            |
| PostgreSQL | custom                       | 5432            |

## Requisitos previos

- Docker >= 24 y Docker Compose >= 2.20
- Al menos 4 GB de RAM y 10 GB de disco disponibles para Docker
- En Linux: exportar `AIRFLOW_UID` antes de levantar los servicios:
  ```bash
  echo "AIRFLOW_UID=$(id -u)" >> .env
  ```

## Despliegue paso a paso

### 1. Clonar el repositorio

```bash
git clone <url-del-repo>
cd ceia-mlops
```

### 2. Construir las imagenes y levantar los servicios

```bash
docker compose --profile all up --build -d
```

Esto levanta todos los servicios: Airflow, MLflow, MinIO, FastAPI y PostgreSQL.
Esperar a que los healthchecks pasen (puede tardar 2-3 minutos).

### 3. Cargar el dataset en MinIO

**Este paso es obligatorio antes de ejecutar el DAG.** El pipeline lee `stroke-data.csv` desde el bucket `data` de MinIO.

```bash
docker cp data/stroke-data.csv minio:/tmp/stroke-data.csv
docker exec minio mc alias set local http://localhost:9000 minio minio123
docker exec minio mc cp /tmp/stroke-data.csv local/data/stroke-data.csv
```

Alternativamente, usar la UI de MinIO en http://localhost:9001 (usuario: `minio`, contrasena: `minio123`), ingresar al bucket `data` y subir el archivo `data/stroke-data.csv`.

### 4. Ejecutar el DAG de entrenamiento

1. Abrir Airflow en http://localhost:8080 (usuario: `airflow`, contrasena: `airflow`)
2. Activar y ejecutar el DAG `train_stroke_model`
3. Esperar a que todas las tareas terminen en estado **success**

El modelo quedara registrado en MLflow como `stroke-predictor`.

### 5. Reiniciar la API (solo en el primer entrenamiento)

La API carga el modelo al iniciar. Si fue levantada antes de que el DAG terminara, reiniciarla para que tome el modelo recien registrado:

```bash
docker compose restart fastapi
```

En ejecuciones posteriores (el modelo ya existe en MLflow) este paso no es necesario.

## Servicios y URLs

| Servicio       | URL                        | Credenciales       |
|----------------|----------------------------|--------------------|
| Airflow UI     | http://localhost:8080      | airflow / airflow  |
| MLflow UI      | http://localhost:5000      | —                  |
| MinIO UI       | http://localhost:9001      | minio / minio123   |
| FastAPI docs   | http://localhost:8800/docs | —                  |

## Uso de la API

### Verificar estado

```bash
curl http://localhost:8800/
```

### Predecir riesgo de ACV

```bash
curl -X POST http://localhost:8800/predict \
  -H "Content-Type: application/json" \
  -d '{
    "age": 67,
    "hypertension": 0,
    "heart_disease": 1,
    "gender": "Male",
    "ever_married": "Yes",
    "work_type": "Private",
    "Residence_type": "Urban",
    "avg_glucose_level": 228.69,
    "bmi": 36.6,
    "smoking_status": "formerly smoked"
  }'
```

Respuesta:

```json
{
  "probabilidad_acv": 0.312,
  "prediccion": 1,
  "riesgo": "Medio",
  "threshold_usado": 0.17847
}
```

**Valores validos:**
- `gender`: `"Male"`, `"Female"`, `"Other"`
- `work_type`: `"Private"`, `"Self-employed"`, `"Govt_job"`, `"children"`, `"Never_worked"`
- `Residence_type`: `"Urban"`, `"Rural"`
- `smoking_status`: `"formerly smoked"`, `"never smoked"`, `"smokes"`, `"Unknown"`

## Evidencia de funcionamiento

### 1. DAG ejecutado en Airflow
![DAG ejecutado](docs/airflow_dag.png)

### 2. Experimento registrado en MLflow
![Experimento en MLflow](docs/mlflow_experiment.png)

### 3. Respuesta del endpoint /predict
![Respuesta /predict](docs/fastapi_predict1.png)

![Respuesta /predict](docs/fastapi_predict2.png)

## Detener los servicios

```bash
docker compose --profile all down
```

Para eliminar tambien los volumenes (base de datos, modelos y datos en MinIO):

```bash
docker compose --profile all down -v
```
</div>