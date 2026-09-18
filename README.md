# Fertilizer Sales Data Pipeline

Pipeline de datos end-to-end (Bronze → Silver → Gold) en **Databricks** con **PySpark** y **Delta Lake**, sobre un dominio de venta de fertilizantes. Incluye arquitectura medallion, modelo dimensional, análisis de negocio y una capa de ML puesta en producción con MLflow y un job programado.

> **Datos 100% sintéticos.** Los generé yo, modelados sobre mi experiencia en la industria de fertilizantes. No uso datos reales de ninguna empresa por confidencialidad.

---

## Contexto de negocio

Una empresa de fertilizantes produce bajo demanda, y esa demanda depende del clima: cuando se esperan buenas lluvias, los cultivos piden más fertilizante. El problema es que los pronósticos de lluvia fallan, y cuando la empresa fabrica de más queda con sobre-stock. Eso duele especialmente porque muchos fertilizantes son perecederos (vida útil corta) y almacenar materia prima tiene un costo fijo diario.

El pipeline modela ese flujo (venta → producción → sobre-stock) y responde tres preguntas:

1. ¿Qué tan ligada está la demanda a la lluvia?
2. ¿Cuánto cuesta el sobre-stock y cómo evoluciona?
3. ¿Qué productos concentran el riesgo por ser perecederos?

## Arquitectura (medallion)
Bronze (crudo) Silver (limpio) Gold (negocio)
ventas → deduplicado → modelo estrella
clima nulos tratados tabla resumen mensual
maestros calidad marcada predicción de demanda (ML)


- **Bronze** — ingesta de datos crudos tal cual llegan (ventas, clima, maestros). No se transforma nada; queda como copia fiel de la fuente.
- **Silver** — limpieza y calidad: deduplicación con `ROW_NUMBER` (por `pedido_id`, conservando el más reciente), nulos con `COALESCE`, y una bandera `es_precio_imputado` para no perder rastro de qué se corrigió.
- **Gold** — modelo estrella (`dim_producto`, `dim_cliente`, `fact_ventas`), una tabla resumen mensual (`resumen_ventas_mensual`) al grano mes × producto × línea × departamento, y la tabla de predicciones del modelo (`prediccion_demanda`).

## Stack

| Componente | Herramienta |
|---|---|
| Procesamiento | Apache Spark (PySpark) |
| Almacenamiento | Delta Lake |
| Plataforma | Databricks Free Edition (serverless, Unity Catalog) |
| ML | scikit-learn + MLflow |
| Orquestación | Databricks Jobs |
| Consumo | Power BI |

## Capa de ML y producción

Sobre el pipeline monté un modelo que predice la demanda mensual por departamento a partir del pronóstico de lluvia.

- **Modelo:** regresión lineal (`scikit-learn`). Features: pronóstico de lluvia + departamento (one-hot). Elegí el pronóstico y no la lluvia real a propósito: en producción, al momento de planear, solo tienes el pronóstico disponible.
- **Evaluación:** R² ≈ 0.64 con validación cruzada de 5 folds. Un único split train/test daba lecturas inestables (entre 0.22 y 0.74 según la partición) por el tamaño del dataset (180 filas), así que la validación cruzada da la medida confiable.
- **Producción:** el modelo se registra en **MLflow**, un notebook de producción (`07_produccion_prediccion`) lo carga sin reentrenar, predice y escribe a `gold.prediccion_demanda`.
- **Orquestación:** un job de Databricks (`prediccion_demanda_mensual`) ejecuta ese notebook el día 1 de cada mes.

> Es una prueba de concepto sobre datos sintéticos: el valor está en el flujo end-to-end (entrenar → registrar → servir → orquestar), no en la precisión del modelo.

## Equivalencia en Azure

Corre en Databricks Free Edition, pero el diseño se traslada directo a un entorno Azure de producción:

| En este proyecto | Equivalente en Azure |
|---|---|
| Ingesta en notebooks | Azure Data Factory |
| Delta Lake / Unity Catalog | Azure Data Lake Storage Gen2 + Delta |
| Databricks Free Edition | Azure Databricks |
| MLflow (tracking del modelo) | Azure ML / MLflow gestionado |
| Power BI | Power BI |

## Estructura del repositorio

| Notebook | Qué hace |
|---|---|
| `01_setup` | Configuración inicial y catálogo |
| `02_bronze_ventas` | Ingesta de ventas crudas |
| `02_bronze_clima` | Ingesta de datos de lluvia |
| `02_bronze_maestros` | Ingesta de maestros (productos, clientes) |
| `03_silver_ventas` | Limpieza, deduplicación y calidad |
| `04_gold_ventas` | Modelo estrella + tabla resumen mensual |
| `05_analisis_correlacion` | Análisis de negocio (clima, sobre-stock, perecederos) |
| `06_ml_prediccion_demanda` | Entrenamiento y evaluación del modelo |
| `07_produccion_prediccion` | Carga el modelo desde MLflow y escribe predicciones a Gold |

## Cómo ejecutarlo

1. Importa la carpeta en Databricks (Free Edition sirve).
2. Ejecuta los notebooks en orden: `01` → `02_*` → `03` → `04` → `05` → `06`.
3. El cómputo serverless se enciende solo; no hay que configurar clúster.
4. Para la predicción en producción, corre `07` (requiere que el modelo ya esté registrado en MLflow por el `06`).

## Hallazgos

- **Demanda ligada a la lluvia:** correlación de 0.65–0.67 entre precipitación y volumen vendido.
- **Sobre-stock creciente:** 15.355 → 35.062 → 76.014 USD en tres años (supuesto: 2 USD/TM/mes de almacenamiento).
- **Riesgo en perecederos:** el producto de mayor volumen de sobre-stock resultó ser el de menor riesgo; en cambio ~22.500 TM de sobre-stock caen en productos de vida corta — esa es la zona a vigilar.

## Autor

Juan Camilo Montoya — [GitHub: Juan-mp28](https://github.com/Juan-mp28)