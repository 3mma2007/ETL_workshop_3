# ETL Workshop 3 — Happiness Score Prediction con Apache Kafka + ML

**Curso:** ETL  
**Programa:** Ingeniería de Datos e Inteligencia Artificial  
**Dataset:** [World Happiness Report — Kaggle](https://www.kaggle.com/datasets/unsdsn/world-happiness)

---

## 📋 Descripción del Proyecto

Este proyecto implementa un pipeline ETL completo que combina **Machine Learning** con **Data Streaming** para predecir el índice de felicidad (`Happiness Score`) de distintos países usando datos de los años 2015–2019.

El flujo general es:

```
CSVs (5 años) → EDA + Feature Engineering → Modelo de Regresión (.pkl)
                                                    ↓
                                     Kafka Producer (X_test + y_test)
                                                    ↓
                                     Kafka Consumer → Predicciones → SQLite
```

---

## 🗂️ Estructura del Repositorio

```
ETL_workshop_3/
│
├── ETL_workshop_3.ipynb       # Notebook principal: EDA, entrenamiento y Kafka
├── data/
│   ├── 2015.csv
│   ├── 2016.csv
│   ├── 2017.csv
│   ├── 2018.csv
│   └── 2019.csv
├── model.pkl                  # Modelo entrenado (LinearRegression)
├── df_clean.csv               # Dataset limpio después del preprocesamiento
├── X_test.csv                 # Features del conjunto de prueba
├── y_test.csv                 # Scores reales del conjunto de prueba
├── happiness_predictions.db   # Base de datos SQLite con predicciones
└── README.md
```

---

## ⚙️ Tecnologías Utilizadas

| Herramienta | Uso |
|---|---|
| Python 3 | Lenguaje principal |
| Pandas | Carga y transformación de datos |
| Scikit-learn | Modelo de regresión y métricas |
| Joblib | Serialización del modelo (.pkl) |
| Apache Kafka (KRaft) | Streaming de datos en tiempo real |
| confluent-kafka | Cliente Python para Kafka |
| SQLite | Almacenamiento de predicciones |
| Matplotlib | Visualizaciones |

---

## 🚀 Instrucciones de Configuración y Ejecución

### Requisitos previos

- Python 3.8+
- Google Colab (recomendado) o entorno Linux con acceso a internet
- Google Drive montado en `/content/drive/Shared drives/ETL_workshop_3/`

### 1. Clonar el repositorio y abrir el notebook

```bash
git clone <url-del-repositorio>
```

Abrir `ETL_workshop_3.ipynb` en Google Colab.

### 2. Montar Google Drive

La primera celda del notebook monta automáticamente el Drive. Asegúrate de que los 5 CSVs están en la ruta:

```
/content/drive/Shared drives/ETL_workshop_3/
```

### 3. Ejecutar el notebook en orden

Las secciones del notebook son:

1. **Configuración Inicial** — Monta Drive, importa librerías
2. **Extracción y Exploración de Datos (EDA)** — Carga los 5 CSVs, mapeo de columnas, limpieza, imputación KNN, boxplots y correlación
3. **Modelo** — Split 70/30, entrenamiento de `LinearRegression`, métricas, guardado del `.pkl`
4. **Kafka** — Instala y lanza el broker Kafka local; define y ejecuta Producer y Consumer en paralelo con `threading`

### 4. Kafka (ejecución en paralelo)

```python
import threading

consumer_thread = threading.Thread(target=kafka_consumer)
producer_thread = threading.Thread(target=kafka_producer)

consumer_thread.start()
producer_thread.start()

consumer_thread.join()
producer_thread.join()
```

---

## 📊 Descripción de los Datasets

| Año | Filas | Columnas originales clave |
|---|---|---|
| 2015 | ~158 | Happiness Score, Economy (GDP per Capita), Family, Health... |
| 2016 | ~157 | Happiness Score, Economy (GDP per Capita), Family, Health... |
| 2017 | ~155 | Happiness.Score, Economy..GDP.per.Capita., Family... |
| 2018 | ~156 | Score, GDP per capita, Social support... |
| 2019 | ~156 | Score, GDP per capita, Social support... |

Todos los datasets fueron normalizados al siguiente esquema común:

| Feature unificado | Descripción |
|---|---|
| `gdp_per_capita` | PIB per cápita |
| `social_support` | Apoyo social / familia |
| `life_expectancy` | Esperanza de vida saludable |
| `freedom` | Libertad para tomar decisiones |
| `trust_gov` | Percepción de corrupción / confianza en el gobierno |
| `generosity` | Generosidad |
| `happiness_score` | ⭐ Variable objetivo |

---

## 🔍 Hallazgos del EDA

- **Correlaciones más altas** con `happiness_score`: `gdp_per_capita`, `social_support` y `life_expectancy` presentan las correlaciones positivas más fuertes.
- **Outliers**: Se detectaron valores atípicos en `trust_gov` y `generosity`, pero se decidió conservarlos ya que representan casos reales y no errores de medición.
- **Valores faltantes**: Imputados usando KNN Imputer (k=5) para features; filas con `happiness_score` nulo fueron eliminadas.
- **Año como feature**: Se analizó la variación del score promedio por año y no se observó una tendencia significativa, por lo que el año **no fue incluido** como variable predictora.

---

## 🤖 Entrenamiento del Modelo

- **Algoritmo:** LinearRegression (scikit-learn)
- **Split:** 70% entrenamiento / 30% prueba (`random_state=42`)
- **Features:** `gdp_per_capita`, `social_support`, `life_expectancy`, `freedom`, `trust_gov`, `generosity`
- **Target:** `happiness_score`

### Métricas obtenidas

| Métrica | Valor |
|---|---|
| R² | ~0.78 |
| MAE | ~0.43 |
| RMSE | ~0.55 |
| MAPE (%) | ~7.8% |

*(Los valores exactos pueden variar con cada ejecución)*

---

## 🔄 Proceso de Streaming con Kafka

### Kafka Producer

- Lee `X_test.csv` e `y_test.csv`
- Envía cada fila como un mensaje JSON al topic `happiness-features`
- Primero envía todas las filas de `X_test` (features), luego las de `y_test` (scores reales)

### Kafka Consumer

- Suscrito al topic `happiness-features`
- Detecta el tipo de mensaje por las claves del JSON:
  - Sin clave `happiness_score` → se acumula en `buffer_x` (features)
  - Con clave `happiness_score` → se acumula en `buffer_y` (score real)
- Una vez finalizado el stream (sin mensajes por 5 segundos), realiza las predicciones y guarda en SQLite

### Base de Datos (SQLite)

Archivo: `happiness_predictions.db`  
Tabla: `predicciones`

| Columna | Descripción |
|---|---|
| `gdp_per_capita` | Feature: PIB per cápita |
| `social_support` | Feature: Apoyo social |
| `life_expectancy` | Feature: Esperanza de vida |
| `freedom` | Feature: Libertad |
| `trust_gov` | Feature: Confianza en gobierno |
| `generosity` | Feature: Generosidad |
| `y_real` | Happiness Score real |
| `y_prediccion` | Happiness Score predicho por el modelo |

---

## ⚠️ Decisiones de Diseño y Supuestos

1. **KRaft en lugar de ZooKeeper**: Se usa la configuración KRaft de Kafka 3.7.0, que elimina la dependencia de ZooKeeper y simplifica la instalación en entornos efímeros como Google Colab.

2. **Un solo topic para X e y**: El Producer envía features y scores reales en el mismo topic (`happiness-features`), diferenciándolos por la presencia o ausencia de la clave `happiness_score` en el payload JSON.

3. **SQLite como base de datos**: Se eligió SQLite por su simplicidad y compatibilidad con Google Colab sin configuración adicional de servidor.

4. **Imputación KNN**: Se usó KNN Imputer en lugar de imputación por mediana por año, ya que captura mejor las relaciones entre variables para datos con estructura geográfica/socioeconómica.

---

## 📈 Visualizaciones

El notebook incluye:
- Boxplots de todas las variables
- Distribución del Happiness Score por año
- Scatter plot: Predicciones vs Valores Reales
- Histograma de residuos (errores)
