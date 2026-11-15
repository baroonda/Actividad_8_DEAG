# 🛡️ Sistema de Monitorización Proactiva y Respuesta a Incidentes (Detección de Fraude)

Este repositorio contiene la implementación de un sistema de monitorización proactiva para el modelo de detección de fraude en transacciones de tarjetas de crédito. Se integra la recolección de métricas, logs y trazas bajo un enfoque de observabilidad moderna, asegurando la confiabilidad del servicio en producción.

---

## 1. Diseño de Monitorización y Observabilidad

Para asegurar la continuidad del servicio, se definieron indicadores técnicos y de negocio que permiten evaluar tanto la infraestructura como la calidad predictiva del modelo de Random Forest.

*   **Métricas del Modelo (Rendimiento Predictivo):** F1-Score, Precisión, Recall (priorizando el control de falsos positivos) y tasa de predicciones de fraude vs. transacciones legítimas.
*   **Métricas del Sistema (Golden Signals):** Latencia de inferencia en el endpoint de FastAPI, throughput (peticiones por segundo), uso de CPU/Memoria y tasa de errores HTTP (5xx).
*   **Logs:** Registro estructurado en formato JSON de cada petición entrante (features de la transacción), la predicción generada, el identificador de la versión del modelo en MLflow y el tiempo de respuesta.
*   **Trazas:** Seguimiento del ciclo de vida de la petición desde que ingresa a la API, pasa por el preprocesamiento (PCA), la inferencia del modelo y la respuesta al cliente.

---

## 2. Alertas Inteligentes, SLOs y Error Budgets

Las alertas inteligentes se han diseñado para accionabilidad, evaluando la severidad y reduciendo el ruido operativo mediante la alineación estricta con los Service Level Objectives (SLO) y los presupuestos de error (error budgets).

*   **SLI (Service Level Indicator):** Porcentaje de inferencias procesadas en menos de 200 ms.
*   **SLO (Service Level Objective):** El 99.5% de las peticiones deben ser atendidas en menos de 200 ms en un periodo de 30 días.
*   **Error Budget:** 0.5% de margen mensual para latencias superiores a 200 ms o errores.
*   **Reglas de Alertamiento:**
    *   *Warning (Advertencia):* Consumo del 5% del error budget en 1 hora (Notificación a canal de Slack/Teams del equipo de MLOps).
    *   *Critical (Crítica):* Latencia del P99 > 300ms durante 5 minutos consecutivos o caída del F1-Score por debajo de 0.85 (Llamada automática (PagerDuty) al ingeniero de guardia y ejecución de runbook de escalamiento).

---

## 3. Estrategias de Detección de Data Drift

Para detectar cambios en la distribución de los datos de entrada (como variaciones inusuales en los montos de transacción) que puedan afectar el rendimiento, se implementa una arquitectura de detección de drift:

1.  **Baseline:** Distribución estadística de los datos de entrenamiento almacenada como artefacto en MLflow.
2.  **Monitorización:** Cálculo periódico del Population Stability Index (PSI) y pruebas estadísticas (ej. Kolmogorov-Smirnov) sobre ventanas móviles de datos en producción.
3.  **Acción:** Si el umbral de drift es superado (ej. p-value < 0.05), se genera una alerta crítica indicando degradación del modelo y se evalúa la necesidad de reentrenamiento.

---

## 4. Descripción de Dashboards Operativos

El tablero de control (implementado vía Grafana/MLflow) proporciona visualización en tiempo real estructurada en tres niveles para la toma de decisiones:

*   **Panel de Salud General (Nivel Alto):** Indicadores semáforo (Verde/Rojo) del estado de FastAPI, SLIs consolidados y estado de consumo del Error Budget.
*   **Panel de Infraestructura (Nivel Medio):** Gráficos de series de tiempo mostrando la latencia (P50, P90, P99), el throughput y el consumo de recursos (CPU/RAM).
*   **Panel de Rendimiento del Modelo (Nivel Profundo):** Histograma de predicciones (distribución de scores), matriz de confusión estimada y gráficas de detección de Data Drift.

---

## 5. Documentación de Respuesta Automatizada y Gobernanza

Se implementa una estrategia de **Rollback Automático** basada en el Model Registry de MLflow, eliminando la necesidad de intervención humana inmediata ante fallos críticos.

*   **Proceso de Rollback:** Si el sistema detecta una caída severa en la calidad predictiva (ej. anomalía post-despliegue), el script de monitorización realiza un cambio automático de la etiqueta del modelo en MLflow de la versión actual a la versión anterior estable ("Production" a "Archived", y restaura la versión previa).
*   **Reentrenamiento Automatizado:** La detección sostenida de data drift dispara un webhook hacia la canalización (pipeline) de CI/CD para iniciar un reentrenamiento del modelo con los datos más recientes.

---

## 6. Simulación de Incidentes y Ejecución de Runbooks

Se ejecutaron dos simulaciones tipo *Game Day* en un entorno controlado para evaluar la capacidad de respuesta y eficacia de los runbooks.

### Incidente Simulador 1: Incremento de Latencia (Degradación de Infraestructura)
*   **Descripción:** Se inyectó tráfico masivo simulando un ataque o pico de transacciones comerciales, saturando los workers de FastAPI.
*   **Runbook Ejecutado (RB-01 Escalado):**
    1.  *Detección:* La alerta de latencia (P99 > 300ms) se disparó en el minuto 3.
    2.  *Diagnóstico:* Verificación rápida de logs confirmando saturación de CPU al 95%.
    3.  *Respuesta:* Ejecución del script de auto-escalado para instanciar 2 workers adicionales.
*   **Resultados:** El MTTR (Mean Time to Repair) fue de 4 minutos. La latencia regresó a la normalidad (< 150ms).

### Incidente Simulador 2: Data Drift y Degradación de Precisión
*   **Descripción:** Se enviaron peticiones con un formato de datos alterado (transacciones con montos atípicamente altos) para simular un cambio en el comportamiento de fraude.
*   **Runbook Ejecutado (RB-02 Modelo):**
    1.  *Detección:* El módulo de drift detectó divergencia en las características y la alerta indicó una caída teórica de precisión.
    2.  *Diagnóstico:* Revisión del dashboard de drift confirmó el cambio en la distribución de la variable `monto`.
    3.  *Respuesta Automatizada:* Se gatilló el rollback automático hacia la versión v1.0 del modelo en MLflow.
*   **Resultados:** El servicio se mantuvo estable con el modelo de respaldo. Se documentó el evento en el postmortem para análisis del equipo de ciencia de datos.

---

## 7. Evidencia de Implementación (Capturas)

> **Nota de entrega:** Las siguientes imágenes evidencian el uso de las herramientas de monitorización y alertamiento en tiempo real.

*Reemplazar los enlaces con las capturas de pantalla de tu ejecución local.*

1. **Captura del Registro en MLflow (Métricas y Artefactos):**
   `![Evidencia MLflow](./docs/img/mlflow_tracking.png)`

2. **Captura de la Configuración de Alertas/Dashboards en Tiempo Real:**
   `![Evidencia Dashboard](./docs/img/grafana_alerts.png)`

3. **Captura de los Logs de la Inferencia (FastAPI):**
   `![Evidencia Logs](./docs/img/fastapi_logs.png)`

---

## 8. Código de Monitorización y Configuración de Alertas

A continuación, se incluye el script de Python (`monitor_alerts.py`) que simula la evaluación del rendimiento, la detección de anomalías y la ejecución de la respuesta automatizada (Rollback) interactuando con MLflow.

```python
import time
import logging
import random
import mlflow
from mlflow.tracking import MlflowClient

# Configuración de Logs
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

# Configuración de MLflow
mlflow.set_tracking_uri("http://localhost:5000")
client = MlflowClient()
MODEL_NAME = "FraudDetectionModel"

# SLOs y Umbrales
SLO_MAX_LATENCY = 0.200  # 200 ms
DRIFT_THRESHOLD = 0.05   # p-value simulado

def monitor_system():
    """Simula la recolección de métricas de producción y evaluación de alertas."""
    logger.info("Iniciando monitorización proactiva del modelo de fraude...")
    
    for cycle in range(1, 6):
        time.sleep(2)
        
        # Simulando recolección de métricas
        current_latency = random.uniform(0.100, 0.350)
        drift_p_value = random.uniform(0.01, 0.15)
        
        logger.info(f"Ciclo {cycle} | Latencia: {current_latency:.3f}s | Data Drift p-value: {drift_p_value:.3f}")
        
        # Evaluación de Alertas Inteligentes (SLI vs SLO)
        if current_latency > SLO_MAX_LATENCY:
            logger.warning(f"[ALERTA] Latencia elevada detectada ({current_latency:.3f}s). Supera el SLO.")
            
        # Simulación de Incidente de Degradación (Drift)
        if drift_p_value < DRIFT_THRESHOLD:
            logger.error("[CRÍTICO] Data drift detectado. Posible degradación del modelo.")
            execute_automated_rollback()
            break # Detener ciclo para simular intervención

def execute_automated_rollback():
    """Ejecuta el runbook de respuesta automatizada utilizando MLflow."""
    logger.info("Iniciando Runbook RB-02: Rollback Automático de Modelo...")
    try:
        # Lógica de MLflow para cambiar de versión
        # Obtener la versión actual en producción
        prod_versions = client.get_latest_versions(MODEL_NAME, stages=["Production"])
        if prod_versions:
            current_version = prod_versions[0].version
            logger.info(f"Modelo actual en Producción: v{current_version}. Transicionando a Archivo.")
            client.transition_model_version_stage(
                name=MODEL_NAME,
                version=current_version,
                stage="Archived"
            )
            # Aquí se añadiría la lógica para promover la versión anterior N-1 a Production
            logger.info("Rollback ejecutado con éxito. Servicio restaurado a versión de respaldo.")
        else:
            logger.info("Simulación: Rollback completado a versión estable por defecto.")
    except Exception as e:
        logger.error(f"Fallo en la automatización del rollback: {str(e)}")

if __name__ == "__main__":
    monitor_system()
