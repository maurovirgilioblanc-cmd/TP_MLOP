<div align="center">

<h1>🧠 Predicción de ACV</h1>

<p>
TP Final de la materia MLOps — Posgrado en Inteligencia Artificial (CEIA, UBA)
</p>

<p>
Pipeline end-to-end para predecir la probabilidad de accidente cerebrovascular (ACV/stroke) usando Regresión Logística con SMOTE
</p>

<br>

<h2>👥 Integrantes</h2>

<p>
Mauro Virgilio Blanc<br>
Juan Pablo Imbrogno<br>
Sofía Belén Caselli<br>
Miguel Angel Leiva Martinez<br>
Andrea Viviana Ferenaz
</p>

<br>

<h2>🏗️ Arquitectura</h2>

<pre>
MinIO (s3)   <- dataset de entrada (bucket: data)
     |
Airflow      <- orquesta el pipeline de entrenamiento (DAG: train_stroke_model)
     |
MLflow       <- registra experimentos y versiona el modelo (bucket: mlflow)
     |
FastAPI      <- sirve predicciones en tiempo real (puerto 8800)
</pre>

<p>
Servicios de soporte: PostgreSQL (backend de Airflow y MLflow), Redis (broker de Celery)
</p>

<br>

<h2>🧰 Stack tecnológico</h2>

<table align="center">
<tr>
<th>Servicio</th>
<th>Imagen / Framework</th>
<th>Puerto</th>
</tr>

<tr><td>Airflow</td><td>custom (CeleryExecutor)</td><td>8080</td></tr>
<tr><td>MLflow</td><td>custom</td><td>5000</td></tr>
<tr><td>MinIO</td><td>minio/minio:latest</td><td>9000 / 9001</td></tr>
<tr><td>FastAPI</td><td>custom (uvicorn)</td><td>8800</td></tr>
<tr><td>PostgreSQL</td><td>custom</td><td>5432</td></tr>

</table>

<br>

<h2>⚙️ Requisitos previos</h2>

<p>
Docker &gt;= 24<br>
Docker Compose &gt;= 2.20<br>
Al menos 4 GB de RAM y 10 GB de disco disponibles para Docker
</p>

<pre>
echo "AIRFLOW_UID=$(id -u)" >> .env
</pre>

<br>

<h2>🚀 Despliegue paso a paso</h2>

<p><b>1. Clonar el repositorio</b></p>

<pre>
git clone &lt;https://github.com/maurovirgilioblanc-cmd/TP_MLOP&gt;
cd ceia-mlops
</pre>

<p><b>2. Construir y levantar servicios</b></p>

<pre>
docker compose --profile all up --build -d
</pre>

<p>
Esto levanta todos los servicios: Airflow, MLflow, MinIO, FastAPI y PostgreSQL. Esperar a que los healthchecks pasen (puede tardar 2-3 minutos).
</p>

<p><b>3. Cargar dataset en MinIO</b></p>

<p>
Este paso es obligatorio antes de ejecutar el DAG. El pipeline lee stroke-data.csv desde el bucket data de MinIO.
</p>
<pre>
docker cp data/stroke-data.csv minio:/tmp/stroke-data.csv
docker exec minio mc alias set local http://localhost:9000 minio minio123
docker exec minio mc cp /tmp/stroke-data.csv local/data/stroke-data.csv
</pre>

<p>
Alternativamente, usar la UI de MinIO en http://localhost:9001 (usuario: minio, contrasena: minio123), ingresar al bucket data y subir el archivo data/stroke-data.csv.
</p>

<p><b>4. Ejecutar DAG de entrenamiento</b></p>

<p>
1. Abrir Airflow en http://localhost:8080 (usuario: airflow, contraseña: airflow)<br>
2. Activar y ejecutar el DAG <b>train_stroke_model</b><br>
3. Esperar a que todas las tareas terminen en estado <b>success</b>
</p>

<p>
El modelo quedará registrado en MLflow como <b>stroke-predictor</b>.
</p>

<p><b>5. Reiniciar API</b></p>
<p>La API carga el modelo al iniciar. Si fue levantada antes de que el DAG terminara, reiniciarla para que tome el modelo recien registrado:</p>
<pre>
docker compose restart fastapi
</pre>

<p>
En ejecuciones posteriores (el modelo ya existe en MLflow) este paso no es necesario.
</p>

<br>

<h2>🌐 Servicios y URLs</h2>

<table align="center">
<tr>
<th>Servicio</th>
<th>URL</th>
<th>Credenciales</th>
</tr>

<tr><td>Airflow</td><td>http://localhost:8080</td><td>airflow / airflow</td></tr>
<tr><td>MLflow</td><td>http://localhost:5000</td><td>-</td></tr>
<tr><td>MinIO</td><td>http://localhost:9001</td><td>minio / minio123</td></tr>
<tr><td>FastAPI</td><td>http://localhost:8800/docs</td><td>-</td></tr>

</table>

<br>

<h2>🔎 Uso de la API</h2>

<p><b>Verificar estado</b></p>
<pre>
curl http://localhost:8800/
</pre>

<p><b>Predecir riesgo de ACV</b></p>
<pre>
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
</pre>

<br>

<h2>Respuesta</h2>

<pre>
{
  "probabilidad_acv": 0.312,
  "prediccion": 1,
  "riesgo": "Medio",
  "threshold_usado": 0.17847
}
</pre>

<br>

<h2>📌 Valores válidos</h2>

<ul align="left" style="display: inline-block; text-align: left;">
  <li><b>gender:</b> "Male", "Female", "Other"</li>
  <li><b>work_type:</b> "Private", "Self-employed", "Govt_job", "children", "Never_worked"</li>
  <li><b>Residence_type:</b> "Urban", "Rural"</li>
  <li><b>smoking_status:</b> "formerly smoked", "never smoked", "smokes", "Unknown"</li>
</ul>

<br>

<h2>🧪 Evidencia</h2>

<p><b>1. DAG ejecutado en Airflow</b></p>
<img src="docs/airflow_dag.png" width="700"/>

<br><br>

<p><b>2. Experimento registrado en MLflow</b></p>
<img src="docs/mlflow_experiment.png" width="700"/>

<br><br>

<p><b>3. Respuesta del endpoint /predict</b></p>
<img src="docs/fastapi_predict1.png" width="700"/>

<br><br>

<img src="docs/fastapi_predict2.png" width="700"/>

<br>

<h2>🛑 Detener servicios</h2>

<pre>
docker compose --profile all down
</pre>

<p>
Para eliminar también los volúmenes (base de datos, modelos y datos en MinIO):
</p>

<pre>
docker compose --profile all down -v
</pre>

</div>