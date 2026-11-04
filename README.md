# DataRetail 360 DWH — Pipeline ETL · DataRetail 360

> Proyecto Bottom-Up de Data Warehouse para DataRetail 360, comenzando por la creación de Datamarts.
> IDY1102 · Unidad 2 · Experiencia 2 · Actividad 2.3 · RA2 · IL 2.3

---

## 1. Descripción

Este pipeline ETL integra datos heterogéneos de la empresa ficticia **DataRetail 360** (retail con presencia nacional en tecnología, hogar y accesorios) hacia un Data Warehouse dimensional. La organización consolida manualmente información en archivos Excel y CSV, generando cifras distintas para la misma pregunta de negocio por diferencias de filtros, fechas de corte, anulaciones, devoluciones y criterios de clasificación.

El pipeline resuelve esa necesidad consolidando las fuentes en un modelo dimensional único (6 dimensiones + 4 fact tables) con reglas de calidad explícitas, cuarentena de rechazos, trazabilidad por logs y carga idempotente. El modelo sigue un enfoque **Bottom-Up** a partir de la Bus Matrix: cada proceso de negocio tiene su propia fact table con PK simple (`fact_key`) y FKs solo a las dimensiones marcadas **X** en la Bus Matrix.

---

## 2. Arquitectura

```
Preflight idempotencia ─► Generación Bronce heterogénea ─► CSV {fuente}_{YYYYMMDD}.csv
        │                                                          │
        │                                                          ▼
        │                                                 EXTRACT ─► TRANSFORM ─► LOAD (SQLite)
        │                                                          │
        │                                                          ▼
        │                                       rejected/Rejected_{fuente}_{YYYYMMDD}.csv
        ▼
DataRetail360.db (etl_control)                         MASK ─► DataRetail360_Masked.db
```

- **Idempotencia total:** antes de generar, extraer, transformar, cargar o enmascarar, el pipeline consulta `etl_control` en modo lectura. Si la fecha ya existe, termina con `OMITIDO_POR_IDEMPOTENCIA` sin crear archivos nuevos.
- **Bronce heterogéneo:** no imita el modelo dimensional. Genera 10 fuentes sintéticas con semilla fija (`numpy.random.default_rng(42)`) que conservan el esquema de su sistema de origen: `crm_clientes` (CRM REST API), `erp_ventas` (ERP PostgreSQL OLTP), `wms_inventario` (WMS API), `mkt_campanas` (Marketing SaaS), `pos_devoluciones` (POS SQL Server) y los maestros `mdm_productos`, `mdm_categorias`, `mdm_sucursales`, `mdm_canales`, `core_calendario`. Se guardan como `data/{fuente}_{YYYYMMDD}.csv`, sin surrogate keys ni columnas de hechos.
- **Transformaciones:** validación de claves nulas, FKs inexistentes, RUT chileno (módulo 11), emails, montos negativos, descuentos inválidos, cantidades no positivas, stock negativo, rangos de campaña, fechas fuera de calendario, estados fuera de dominio y duplicados por clave natural.
- **Clave de cliente:** el CRM no expone una ID de cliente; su llave primaria es el RUT. `crm_clientes` no tiene `customerId`, y las fuentes transaccionales referencian al cliente con `CUSTOMER_RUT`/`customer_rut`. En el datamart, `rut_cliente` + `dv_cliente` son la clave natural y `cliente_key` es la ID de almacenamiento generada.
- **Destino:** SQLite `DataRetail360.db` — 6 dimensiones (`dim_fecha`, `dim_categoria`, `dim_producto`, `dim_sucursal`, `dim_cliente`, `dim_canal`) + 4 fact tables (`fact_ventas`, `fact_inventario`, `fact_campanas`, `fact_devoluciones`), esquema estrella con PK simple y `dim_categoria` como dimensión snowflake de `dim_producto`.
- **Capa enmascarada:** `DataRetail360_Masked.db` replica el modelo procesado con RUT y email enmascarados en `dim_cliente`, más una tabla `masking_audit` que registra la técnica aplicada.

---

## 3. Requisitos

- Python 3.10 o superior (Google Colab ya incluye todo).
- Librerías: `pandas` y `numpy`. El notebook verifica e instala lo faltante; `pytest` es opcional y no se usa como runner.
- Base de datos: **SQLite** (incluida en la biblioteca estándar de Python, no requiere servidor).
- Espacio en disco: mínimo 500 MB para datos, rechazos y logs.

---

## 4. Instalación

```bash
git clone [url-del-repositorio]
cd [carpeta-del-proyecto]
python -m venv .venv
source .venv/bin/activate        # En Windows: .venv\Scripts\activate
pip install pandas numpy
```

> **Importante**: Verificar que `.gitignore` excluye `data/`, `logs/`, `rejected/`, `*.db` y artefactos locales antes del primer commit.

---

## 5. Configuración

El pipeline **no requiere credenciales ni variables de entorno**: SQLite es local y los datos son sintéticos.

Todos los parámetros viven en la celda *"Parámetros, fuentes y logging"* del notebook:

```python
SEED           = 42             # semilla reproducible
N_VENTAS       = 10_000_000     # filas fact_ventas (antes de rechazos)
N_INVENTARIO   = 120_000        # filas fact_inventario (antes de rechazos)
N_CAMPANAS     = 25_000         # filas fact_campanas (antes de rechazos)
N_DEVOLUCIONES = 300_000        # filas fact_devoluciones (antes de rechazos)
N_CLIENTES     = 200_000        # filas crm_clientes → dim_cliente (antes de rechazos)
FECHA_INICIO   = "2025-01-01"
FECHA_FIN      = "2026-12-31"
DB_NAME        = "DataRetail360.db"
MASKED_DB_NAME = "DataRetail360_Masked.db"
```

`RUN_DATE_STR` (`YYYYMMDD`) se usa en nombres de datos y rechazos; `RUN_TIME` (`YYYYMMDD_HHMMSS`) se usa solo en el nombre del log.

---

## 6. Uso

Abrir `ETL_DataRetail360.ipynb` en Google Colab o Jupyter y ejecutar **Entorno de ejecución → Ejecutar todo**.

Al ejecutarse, el pipeline:

1. **Verifica idempotencia** leyendo `etl_control`: si la fecha ya fue procesada, se detiene sin generar ni transformar datos.
2. **Genera** las 10 fuentes Bronce heterogéneas (RUT válidos ≥ 90%) y las guarda como `data/{fuente}_{YYYYMMDD}.csv`.
3. **Extrae** leyendo esos CSV; si alguno falta, lo regenera y lo persiste antes de leerlo.
4. **Transforma** aplicando las reglas de calidad y enviando las filas inválidas a `rejected/Rejected_{fuente}_{YYYYMMDD}.csv`, un archivo por fuente.
5. **Carga** dimensiones primero y luego hechos en `DataRetail360.db` con DDL explícito, `INSERT`/append y `PRAGMA foreign_keys = ON`.
6. **Enmascara** RUT y email en una copia del modelo y la guarda en `DataRetail360_Masked.db`.
7. **Registra** la corrida en `etl_control` para bloquear una segunda ejecución el mismo día.

Resultados y artefactos:

- `data/{fuente}_{YYYYMMDD}.csv` — datos Bronce con esquema de origen.
- `rejected/Rejected_{fuente}_{YYYYMMDD}.csv` — cuarentena separada por fuente.
- `logs/etl_{YYYYMMDD_HHMMSS}.log` — log de la corrida (consola + archivo); la primera línea siempre es `INICIO DE PIPELINE`.
- `DataRetail360.db` — base SQLite con el modelo dimensional completo.
- `DataRetail360_Masked.db` — copia enmascarada de ejemplo con auditoría.

---

## 7. Estructura del proyecto

```
.
├── ETL_DataRetail360.ipynb   # Deliverable: pipeline ETL completo (Colab-ready)
├── Tablas.md                 # Esquema del datamart (6 dims + 4 facts)
├── Bus Matrix.md             # 4 procesos × 5 dimensiones (semántica X/O)
├── ETL Plan.md               # Requisitos del pipeline
├── Checklist.md              # Validación del pipeline
├── README.md                 # Este documento
├── data/                     # CSV Bronce por fuente y fecha (NO se versiona)
├── rejected/                 # Rejected_{fuente}_{YYYYMMDD}.csv (NO se versiona)
├── logs/                     # etl_{YYYYMMDD_HHMMSS}.log (NO se versiona)
├── DataRetail360.db          # Base SQLite generada al ejecutar (NO se versiona)
└── DataRetail360_Masked.db   # Copia enmascarada de ejemplo (NO se versiona)
```

---

## 8. Calidad de datos

Los motivos están centralizados en el diccionario `MOTIVOS` del notebook. Cada rechazo se escribe con la especificación completa:

`rechazo_id`, `fecha_proceso`, `fuente`, `sistema_origen`, `tabla_destino`, `fila_origen`, `clave_registro`, `campo_afectado`, `valor_rechazado`, `motivo_codigo`, `motivo_descripcion`, `regla`, `severidad`, `accion_sugerida` y `payload_origen` (fila completa en JSON).

| Código de motivo | Dimensión de calidad | Acción | Severidad |
|---|---|---|---|
| `CLAVE_NULA` | Completitud | Enviar a rechazos | ALTA |
| `CLAVE_DUPLICADA` | Unicidad | Conservar la primera, rechazar el resto | MEDIA |
| `RUT_INVALIDO` | Validez | Enviar a rechazos | ALTA |
| `EMAIL_INVALIDO` | Validez | Enviar a rechazos | MEDIA |
| `NOMBRE_NULO` / `REGION_NULA` | Completitud | Enviar a rechazos | MEDIA |
| `FK_INEXISTENTE` | Integridad referencial | Enviar a rechazos | ALTA |
| `FECHA_INVALIDA` / `FECHA_FUERA_CALENDARIO` | Validez | Enviar a rechazos | ALTA |
| `MONTO_NEGATIVO` / `DESCUENTO_INVALIDO` | Validez / Consistencia | Enviar a rechazos | ALTA |
| `CANTIDAD_INVALIDA` / `STOCK_NEGATIVO` | Validez | Enviar a rechazos | ALTA |
| `ESTADO_DESCONOCIDO` / `DOMINIO_INVALIDO` | Validez | Enviar a rechazos | MEDIA |
| `VALOR_NO_NUMERICO` | Validez | Enviar a rechazos | ALTA |
| `SKU_INVALIDO` / `PRECIO_INVALIDO` / `COSTO_MAYOR_PRECIO` | Validez / Consistencia | Enviar a rechazos | ALTA / MEDIA |
| `FECHA_CAMPANA_INVALIDA` | Consistencia | Enviar a rechazos | MEDIA |

Cada fuente tiene su propio archivo `rejected/Rejected_{fuente}_{YYYYMMDD}.csv`; no existe un archivo agregado. El test `test_motivos_mapeados` verifica que todo motivo emitido exista en `MOTIVOS`.

---

## 9. Logging y manejo de errores

- Los registros se guardan en `logs/etl_{YYYYMMDD_HHMMSS}.log` (y se imprimen en consola) con formato `timestamp | nivel | [paso] | mensaje`.
- La primera línea del log es siempre `INICIO DE PIPELINE`, con `run_id`, fecha, base principal y base enmascarada.
- Niveles usados: `DEBUG`, `INFO`, `WARNING`, `ERROR`.
- Códigos de paso: `E` (generación/extracción), `T` (transformación y rechazos), `L` (carga y enmascarado), `S` (sistema, tests y orquestación).
- Los rechazos se registran por fuente: `RECHAZO fuente=... filas=N archivo=... motivos={...}`, incluyendo las fuentes con cero rechazos.
- **Idempotencia total:** `etl_control(run_date PRIMARY KEY)` se consulta antes de generar Bronce. Una segunda corrida el mismo día solo registra la omisión y no crea datos, rechazos ni base enmascarada.
- La carga usa DDL + `INSERT`/append (no `to_sql(if_exists="replace")`) para no eliminar las restricciones de clave foránea.
- Errores recuperables (reintento): archivo CSV ausente (se regenera).
- Errores fatales (detener y alertar): fallo de conexión/escritura en SQLite o violación de FK (`PRAGMA foreign_key_check`).

---

## 10. Pruebas

El notebook incluye `run_tests()`, que ejecuta 14 pruebas autocontenidas y muestra una tabla PASS/FAIL:

| Test | Qué valida |
|---|---|
| `test_bronze_heterogeneo` | Bronce usa nombres y esquemas de origen, sin `dim_*`/`fact_*`, surrogate keys ni `customerId` en el CRM |
| `test_cliente_sin_id_origen` | `dim_cliente` no tiene `cliente_id`; `rut_cliente` es clave natural y `cliente_key` la ID de almacenamiento |
| `test_rut_valido_90` | ≥ 90% de RUT con DV válido |
| `test_timestamps_solo_fecha` | Datos y rechazos usan `YYYYMMDD`; solo el log incluye hora |
| `test_extract_lee_csv` | EXTRACT devuelve todas las fuentes desde CSV |
| `test_extract_regenera_si_falta` | Si falta un CSV, se regenera, persiste y extrae |
| `test_rechazos_por_fuente` | Un archivo de rechazo por fuente, sin agregado |
| `test_rechazo_detalle` | Columnas de detalle completas y payload JSON válido |
| `test_motivos_mapeados` | Todo `motivo_codigo` existe en `MOTIVOS` |
| `test_logs_inicio_y_separacion` | `INICIO DE PIPELINE` es la primera línea y los rechazos se registran por fuente |
| `test_transform_rechaza_rut_invalido` | RUT inválidos van a rechazos, no a `dim_cliente` |
| `test_load_sqlite` | Conteos por tabla y `PRAGMA foreign_key_check` sin violaciones |
| `test_enmascarado` | La base enmascarada no conserva RUT/email crudos y mantiene los hechos |
| `test_idempotencia_bloquea_antes_de_generar` | Una corrida previa bloquea generación, transformación y enmascarado |

```python
# En una celda del notebook:
run_tests()
```

---

## 11. Equipo

- **Nombre del Estudiante** — IDY1102, Unidad 2, Experiencia 2
- **Profesor/tutor** — Curso IDY1102, Duoc UC

---

## 12. Licencia / uso académico

Proyecto académico para el curso IDY1102 (Duoc UC). Datos de ejemplo del caso ficticio DataRetail 360. Este modelo dimensional y el pipeline ETL son para fines educativos y pueden extenderse para uso profesional considerando mejores prácticas de arquitectura de datos, gobernanza y seguridad.

---

## Referencias y Artefactos del Proyecto

| Archivo | Descripción |
|---|---|
| `Checklist.md` | 42 ítems de validación para revisar pipeline antes de integrar (merge) |
| `ETL_DataRetail360.ipynb` | Deliverable: pipeline ETL completo, ejecutable en Google Colab |
| `ETL_Plan.md` | Definición de procesos completables |
| `README.md` | Este documento |
