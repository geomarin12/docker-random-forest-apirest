# Manual Práctico: Arquitectura de Simulación IoT, API REST, IA (Random Forest) y Monitoreo

Este manual guía paso a paso el despliegue de un sistema completo de simulación IoT, microservicio API REST con Inteligencia Artificial y panel de visualización en tiempo real usando **Docker**, **Python** y **MobaXterm**.

---

## 🏗️ Arquitectura del Sistema

```
  +-----------------------------------------------------------------------------------------+
  |                             ENTORNO DOCKER (WSL / MobaXterm)                           |
  |                                                                                         |
  |   [ Node-RED ] (Simula telemetría)                                                       |
  |        │                                                                                |
  |        ├───(HTTP POST /predict)───> [ API REST Flask / Python ]                         |
  |        │                            └──(Modelo Random Forest) ──> [ Decisión / JSON ]   |
  |        │                                                                                |
  |        ├───(Métricas / Alertas)───> [ Consola Debug ]                                   |
  |        │                                                                                |
  |        └───(Envía métricas)───────> [ Grafana ] (Dashboard)                             |
  +-----------------------------------------------------------------------------------------+
```

---

## 🚀 Guía Paso a Paso

### Fase 1: Preparación del Entorno (MobaXterm & Docker)

1. Abre **MobaXterm** y conéctate a tu sesión local de WSL/Linux.
2. Crea la estructura del proyecto e ingresa a la carpeta:

```bash
mkdir -p ~/proyecto_iot_ia
cd ~/proyecto_iot_ia
```

---

### Fase 2: Microservicio API REST con Random Forest (Python)

1. Crea el archivo `requirements.txt` con las dependencias necesarias:

```bash
cat << 'EOF' > requirements.txt
flask
scikit-learn
numpy
EOF
```

2. Crea el servicio web `app.py` que entrenará el modelo **Random Forest** y expondrá la **API REST**:

```bash
cat << 'EOF' > app.py
from flask import Flask, request, jsonify
from sklearn.ensemble import RandomForestClassifier
import numpy as np

app = Flask(__name__)

# Entrenamiento inicial del modelo Random Forest
# Datos de entrenamiento simulados: [Temperatura (°C), Humedad (%)]
# Clases: 0 = NORMAL, 1 = ALERTA CRÍTICA
X_train = np.array([
    [25.0, 40.0],
    [30.0, 45.0],
    [50.0, 55.0],
    [75.0, 70.0],
    [85.0, 85.0],
    [92.0, 90.0]
])
y_train = np.array([0, 0, 0, 1, 1, 1])

# Instancia y entrenamiento del algoritmo
modelo_rf = RandomForestClassifier(n_estimators=10, random_state=42)
modelo_rf.fit(X_train, y_train)

@app.route('/predict', methods=['POST'])
def predict():
    try:
        data = request.get_json()
        temp = float(data.get('temperatura', 0))
        hum = float(data.get('humedad', 50))

        # Realizar predicción con Random Forest
        prediccion = modelo_rf.predict([[temp, hum]])[0]
        probabilidades = modelo_rf.predict_proba([[temp, hum]])[0]

        estado = "CRITICO" if prediccion == 1 else "NORMAL"
        prob_critico = float(probabilidades[1]) if len(probabilidades) > 1 else float(prediccion)

        return jsonify({
            "status": "success",
            "temperatura": temp,
            "humedad": hum,
            "decision_rf": estado,
            "probabilidad_fallo": round(prob_critico, 2),
            "accion": "ENVIAR ALERTA" if estado == "CRITICO" else "SISTEMA ESTABLE"
        }), 200

    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 400

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

3. Crea un `Dockerfile` para empaquetar el microservicio de IA:

```bash
cat << 'EOF' > Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

---

### Fase 3: Despliegue con Docker Compose

1. Crea el orquestador `docker-compose.yml` para integrar Node-RED, Grafana y la API de Python:

```bash
cat << 'EOF' > docker-compose.yml
version: '3.8'

services:
  api_ia:
    build: .
    container_name: api_random_forest
    ports:
      - "5000:5000"
    restart: always

  nodered:
    image: nodered/node-red:latest
    container_name: nodered_simulador
    ports:
      - "1880:1880"
    volumes:
      - nodered_data:/data
    depends_on:
      - api_ia
    restart: always

  grafana:
    image: grafana/grafana:latest
    container_name: grafana_dashboard
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    restart: always

volumes:
  nodered_data:
  grafana_data:
EOF
```

2. Construye y ejecuta la arquitectura completa:

```bash
docker compose up -d --build
```

3. Comprueba el estado de los tres contenedores:

```bash
docker ps
```

---

### Fase 4: Configuración del Flujo en Node-RED

1. Ingresa desde tu navegador a `http://localhost:1880`.
2. Construye la siguiente secuencia de nodos:

#### A) Simulación de Sensor IoT:
* **Nodo `inject`:** Repetir cada 5 segundos.
* **Nodo `function` (Simulador de Telemetría):**
  ```javascript
  // Genera temperatura (20°C a 95°C) y humedad (30% a 90%)
  const temp = (Math.random() * (95 - 20) + 20).toFixed(2);
  const hum = (Math.random() * (90 - 30) + 30).toFixed(2);

  msg.payload = {
      temperatura: parseFloat(temp),
      humedad: parseFloat(hum)
  };
  msg.headers = {
      'Content-Type': 'application/json'
  };
  return msg;
  ```

#### B) Consumo de API REST (Machine Learning):
* **Nodo `http request`:**
  * **Method:** `POST`
  * **URL:** `http://api_ia:5000/predict`
  * **Return:** `a parsed JSON object`

#### C) Procesamiento del Resultado y Notificaciones:
* **Nodo `debug`:** Conéctalo a la salida del nodo HTTP Request para verificar la respuesta devuelta por **Random Forest** (`decision_rf`, `probabilidad_fallo` y `accion`).

#### D) Endpoint HTTP para Grafana:
* **Nodo `http in`:** Método `GET`, URL `/api/telemetria`.
* **Nodo `http response`:** Retorna la información a Grafana.

3. Presiona el botón **Deploy**.

---

### Fase 5: Visualización en Tiempo Real (Grafana)

1. Ingresa a `http://localhost:3000` *(admin / admin)*.
2. Agrega una **Data Source** tipo **JSON API** o **Infinity** apuntando a:
   `http://nodered_simulador:1880/api/telemetria`
3. Agrega un Panel de **Gauge** o **Time Series** para monitorear las métricas y el estado emitido por el modelo en tiempo real.

---

## 🛠️ Resumen de Componentes del Proyecto

| Componente | Tecnología | Función |
|---|---|---|
| **Gestión/Terminal** | MobaXterm + WSL | Consola de administración remota y comandos |
| **Microservicio IA** | Python (Flask + Scikit-Learn) | Servicio REST con el modelo **Random Forest** |
| **Orquestador IoT** | Node-RED | Generación de telemetría y llamadas HTTP POST |
| **Visualización** | Grafana | Dashboard interactivo con gráficos en tiempo real |
| **Contenedores** | Docker & Docker Compose | Despliegue empaquetado de todos los servicios |
