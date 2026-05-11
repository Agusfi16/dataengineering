# 🏅 Arquitectura Medallón On-Premise con Python

Pipeline de ingeniería de datos implementado en tres capas (Bronze → Silver → Gold) siguiendo la **arquitectura medallón**, un patrón de diseño ampliamente adoptado en la industria para organizar lagos de datos (*data lakes*) con niveles progresivos de calidad y transformación.

---

## 📐 Arquitectura del Proyecto

```
superstore_raw.csv (REST API)
        │
        ▼
┌───────────────────┐
│   🥉 BRONZE       │  Ingesta sin procesar
│   (Raw Layer)     │  CSV → almacenamiento local
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   🥈 SILVER       │  Limpieza y transformación
│   (Clean Layer)   │  CSV → Parquet
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   🥇 GOLD         │  Modelo estrella (Star Schema)
│   (Business Layer)│  Parquet → Excel / CSV
└───────────────────┘
```

La arquitectura medallón garantiza **trazabilidad** de los datos en cada etapa: si aparece un error en producción, es posible retroceder a Bronze y reprocesar desde cualquier capa sin perder la fuente original.

---

## 🗂️ Estructura de Carpetas

```
proyecto/
│
├── 01_Bronze_Medallion.ipynb     # Ingesta desde REST API pública
├── 02_Silver_Medallion.ipynb     # Limpieza, normalización y enriquecimiento
├── 03_Gold_Medallion.ipynb       # Modelo estrella y exportación
│
└── data/
    ├── bronze/
    │   └── superstore_raw.csv    # Datos crudos (sin modificar)
    ├── silver/
    │   └── superstore_clean.parquet  # Datos limpios y enriquecidos
    └── gold/
        ├── dim_date.xlsx/csv
        ├── dim_customer.xlsx/csv
        ├── dim_product.xlsx/csv
        ├── dim_ship.xlsx/csv
        ├── fact_sales.xlsx/csv
        └── superstore_star_schema.xlsx  # Consolidado (solo modo gerencia)
```

---

## 🥉 Capa Bronze — Ingesta Raw

**Responsabilidad:** Obtener y preservar los datos de origen tal como están, sin ninguna transformación.

La notebook realiza una llamada HTTP `GET` al dataset público de Superstore alojado en GitHub (vía la API de Plotly Datasets), descarga el archivo CSV y lo persiste en disco.

```python
response = requests.get(DATA_URL, timeout=20)
response.raise_for_status()
RAW_CSV_PATH.write_text(response.text, encoding='utf-8')
```

**Principios aplicados:**
- **Inmutabilidad de Bronze:** el archivo `superstore_raw.csv` nunca es modificado por capas posteriores. Esto permite auditoría y re-procesamiento desde el origen.
- **Idempotencia:** ejecutar la notebook múltiples veces produce el mismo resultado.
- **Timeout explícito** en la llamada HTTP para evitar bloqueos indefinidos en entornos de producción.

---

## 🥈 Capa Silver — Limpieza y Transformación

**Responsabilidad:** Tomar los datos crudos de Bronze, aplicar reglas de calidad de datos (*data quality*) y producir un dataset confiable y estandarizado.

Cada transformación está envuelta en bloques `try/except` independientes, lo que garantiza **resiliencia**: si una transformación falla, las demás continúan ejecutándose y el error queda logueado sin interrumpir el pipeline completo.

### Transformaciones aplicadas

| Transformación | Descripción |
|---|---|
| Normalización de columnas | `snake_case`, sin espacios, minúsculas |
| Limpieza de strings | `.strip()` sobre todas las columnas de texto |
| Parseo de fechas | `order_date` y `ship_date` con `errors='coerce'` |
| Campos derivados | `order_year`, `order_month`, `order_quarter` |
| Validación de integridad | Eliminación de filas con fechas nulas, ventas negativas, cantidad ≤ 0 |
| Deduplicación | `.drop_duplicates()` |
| Conversiones numéricas | `sales`, `quantity`, `discount`, `profit` tipados explícitamente |
| Métrica de negocio | `profit_margin = profit / sales` |

### Formato de salida: Parquet

El dataset limpio se almacena en formato **Apache Parquet**, un formato columnar de alto rendimiento que ofrece:
- Compresión eficiente (hasta 10x menos espacio que CSV)
- Lectura parcial por columnas (mejora performance en consultas analíticas)
- Preservación nativa de tipos de datos (fechas, enteros, decimales)
- Compatibilidad directa con herramientas BI y motores SQL distribuidos

---

## 🥇 Capa Gold — Modelo Estrella

**Responsabilidad:** Modelar los datos limpios en un esquema optimizado para análisis y consumo por parte de usuarios de negocio o herramientas de BI.

Se implementa un **Star Schema** (esquema estrella), el modelo dimensional más utilizado en *data warehousing*, compuesto por una tabla de hechos central conectada a tablas de dimensiones mediante claves surrogadas (*surrogate keys*).

### Diagrama del Modelo Estrella

```
          DIM_Date
         (date_key)
              │
DIM_Ship ─────┤         DIM_Customer
(ship_key)    │         (customer_key)
              ▼
         FACT_Sales ───────────────────
         ┌──────────────┐              │
         │ fact_key     │         DIM_Product
         │ order_id     │         (product_key)
         │ date_key FK  │
         │ customer_key │
         │ product_key  │
         │ ship_key     │
         │ sales        │
         │ quantity     │
         │ discount     │
         │ profit       │
         │ profit_margin│
         └──────────────┘
```

### Dimensiones generadas

| Dimensión | Clave | Atributos principales |
|---|---|---|
| `DIM_Date` | `date_key` (YYYYMMDD) | year, quarter, month, month_name, day, weekday |
| `DIM_Customer` | `customer_key` | customer_id, name, segment, country, city, state, region |
| `DIM_Product` | `product_key` | product_id, name, category, sub_category |
| `DIM_Ship` | `ship_key` | ship_mode |

Las claves surrogadas (`*_key`) son enteros secuenciales generados en el pipeline, desvinculando el modelo del sistema fuente y permitiendo cambios en los IDs originales sin impacto en el modelo analítico.

### Modos de exportación

La variable `salida` controla el formato de salida según el consumidor final:

```python
salida = "gerencia"  # Excel: legible, con múltiples hojas consolidadas
salida = "bi"        # CSV: óptimo para Power BI, Tableau, Looker, etc.
```

| Modo | Formato | Caso de uso |
|---|---|---|
| `gerencia` | `.xlsx` individual + consolidado multi-hoja | Reportes ejecutivos, revisiones manuales |
| `bi` | `.csv` por dimensión/hecho | Consumo por herramientas de visualización |

---

## ⚙️ Requisitos

```bash
pip install pandas requests pyarrow openpyxl
```

| Librería | Uso |
|---|---|
| `pandas` | Manipulación y transformación de datos |
| `requests` | Ingesta via REST API (Bronze) |
| `pyarrow` | Serialización en formato Parquet (Silver) |
| `openpyxl` | Exportación a Excel (Gold, modo gerencia) |

---

## 🚀 Cómo ejecutar

Ejecutar las notebooks en orden secuencial:

```bash
# 1. Ingesta
jupyter nbconvert --to notebook --execute 01_Bronze_Medallion.ipynb

# 2. Transformación
jupyter nbconvert --to notebook --execute 02_Silver_Medallion.ipynb

# 3. Modelado y exportación
jupyter nbconvert --to notebook --execute 03_Gold_Medallion.ipynb
```

O simplemente abrirlas en **Jupyter Lab / Notebook** y ejecutar celda por celda.

---

## 📊 Dataset

**Fuente:** [Superstore Sales Dataset — Plotly Datasets](https://raw.githubusercontent.com/plotly/datasets/master/superstore.csv)

Dataset clásico de ventas minoristas con ~10.000 registros de órdenes de una empresa de retail en EE.UU., ampliamente utilizado en la comunidad de data analytics para ejercicios de modelado y visualización.

**Columnas originales clave:** `Order ID`, `Order Date`, `Ship Date`, `Ship Mode`, `Customer ID`, `Customer Name`, `Segment`, `Country`, `City`, `State`, `Postal Code`, `Region`, `Product ID`, `Category`, `Sub-Category`, `Product Name`, `Sales`, `Quantity`, `Discount`, `Profit`

---

## 🔮 Próxima iteración: Despliegue en Azure

Este proyecto está pensado como la **versión on-premise** de referencia. La siguiente iteración replicará el mismo pipeline pero desplegado en la nube de Microsoft Azure, aprovechando servicios nativos para cada capa del medallón:

| Capa | On-Premise (este repo) | Azure (próxima versión) |
|---|---|---|
| **Ingesta** | `requests` + archivos locales | Azure Data Factory / Azure Functions |
| **Bronze** | CSV en disco local | Azure Data Lake Storage Gen2 (ADLS) |
| **Silver** | Parquet en disco local | Delta Lake sobre ADLS + Azure Databricks |
| **Gold** | Excel/CSV en disco local | Azure Synapse Analytics / Dedicated SQL Pool |
| **Orquestación** | Ejecución manual de notebooks | Azure Data Factory Pipelines |
| **Seguridad** | N/A | Azure Key Vault + Managed Identity |

---

## 👤 Autor

Proyecto de portfolio de Ingeniería de Datos.  
Construido con Python · Pandas · Apache Parquet · Star Schema · Medallion Architecture.
