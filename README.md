# Fertilizer Sales Data Pipeline

Pipeline de datos de punta a punta (Bronze → Silver → Gold) construido en **Databricks** con **PySpark** y **Delta Lake**, sobre un dominio simulado de venta de fertilizantes. Arquitectura medallion, modelo dimensional y análisis de negocio orientado a un problema real del sector.

>**Datos 100% sintéticos.** Generados por mí y modelados sobre mi experiencia en la industria de fertilizantes. No contienen información real de ninguna empresa.

---


## Contexto de negocio

Una empresa de fertilizantes produce bajo demanda, pero esa demanda depende del **clima**: cuando se espera buena temporada de lluvias, los cultivos piden más fertilizante. El problema es que los pronósticos de lluvia son poco fiables, y cuando la empresa fabrica de más, queda con **sobre-stock de producto** — algo especialmente costoso porque muchos fertilizantes son **perecederos** (vida útil limitada) y almacenar materia prima tiene un costo fijo diario.

Este pipeline modela ese flujo (venta → producción → sobre-stock) y responde tres preguntas de negocio:


1. ¿Qué tan ligada está la demanda a la lluvia?
2. ¿Cuánto cuesta el sobre-stock y cómo evoluciona?
3. ¿Qué productos concentran el riesgo real por ser perecederos?

## Arquitectura (medallion)
Bronze (crudo) → Silver (limpio) → Gold (listo para negocio)
ventas deduplicado modelo estrella
clima nulos tratados tabla resumen mensual
maestros calidad marcada análisis de negocio


- **Bronze** — ingesta de datos crudos tal cual llegan (ventas, clima, maestros).
- **Silver** — limpieza y calidad: deduplicación con `ROW_NUMBER`, tratamiento de nulos con `COALESCE`, y una bandera `es_precio_imputado` para no perder trazabilidad de qué datos se corrigieron.
- **Gold** — modelo estrella (`dim_producto`, `dim_cliente`, `fact_ventas`) y una tabla resumen materializada (`resumen_ventas_mensual`) al grano mes × producto × línea × departamento, lista para Power BI.

## Stack

| Componente | Herramienta |
|---|---|
| Procesamiento | Apache Spark (PySpark) |
| Almacenamiento | Delta Lake |
| Plataforma | Databricks Free Edition (serverless, Unity Catalog) |
| Consumo | Power BI |

## Equivalencia en Azure

Este proyecto corre en Databricks Free Edition (gratis), pero el mismo diseño se traslada directo a un entorno Azure de producción:

| En este proyecto | Equivalente en Azure |
|---|---|
| Ingesta en notebooks | Azure Data Factory |
| Delta Lake / Unity Catalog | Azure Data Lake Storage Gen2 + Delta |
| Databricks Free Edition | Azure Databricks |
| Power BI | Power BI |

## Estructura del repositorio

| Notebook | Qué hace |
|---|---|
| `01_setup` | Configuración inicial y catálogo |
| `02_bronze_ventas` | Ingesta de ventas crudas |
| `02_bronze_clima` | Ingesta de datos de lluvia |
| `02_bronze_maestros` | Ingesta de maestros (productos, clientes) |
| `03_silver_ventas` | Limpieza, deduplicación y calidad de datos |
| `04_gold_ventas` | Modelo estrella + tabla resumen mensual |
| `05_analisis_correlacion` | Análisis de negocio (clima, sobre-stock, perecederos) |

## Cómo ejecutarlo

1. Importa la carpeta en Databricks (Free Edition sirve).
2. Ejecuta los notebooks en orden: `01` → `02_*` → `03` → `04` → `05`.
3. El cómputo serverless se enciende solo; no requiere configurar clúster.

## Hallazgos principales

- **Demanda ligada a la lluvia:** correlación de **0.67** entre precipitación y volumen vendido — positiva y coherente con el comportamiento agronómico real.
- **Costo del sobre-stock creciente:** **15.355 → 35.062 → 76.014 USD** en tres años (supuesto declarado: 2 USD/TM/mes de almacenamiento).
- **Riesgo concentrado en perecederos:** el producto de mayor volumen de sobre-stock resultó ser el de menor riesgo; en cambio, ~22.500 TM de sobre-stock caen en productos de vida corta — esa es la zona de riesgo real que el negocio debe vigilar.

## Autor

Juan Camilo Montoya — [GitHub: Juan-mp28](https://github.com/Juan-mp28)