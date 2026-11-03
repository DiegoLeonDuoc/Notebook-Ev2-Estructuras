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
Generación Bronze (numpy/pandas)  ─►  CSV con timestamp  ─►  EXTRACT  ─►  TRANSFORM  ─►  LOAD (SQLite)
                                                                              │
                                                                              ▼
                                                                  rejected/Rejected_{ts}.csv
```

- **Fuentes:** datos sintéticos generados por código con semilla fija (`numpy.random.default_rng(42)`) para las 10 tablas del datamart, persistidos como `data/{tabla}_{timestamp}.csv`. No se usa el LLM para generar datos.
- **Transformaciones:** validación de claves nulas, FKs inexistentes, RUT chileno (módulo 11), montos negativos o `monto_neto > monto_bruto`, cantidades no positivas, estados fuera de dominio y duplicados por clave natural.
- **Destino:** SQLite `DataRetail360.db` — 6 dimensiones (`dim_fecha`, `dim_categoria`, `dim_producto`, `dim_sucursal`, `dim_cliente`, `dim_canal`) + 4 fact tables (`fact_ventas`, `fact_inventario`, `fact_campanas`, `fact_devoluciones`), esquema estrella con PK simple y `dim_categoria` como dimensión snowflake de `dim_producto`.

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

Todos los parámetros viven en la celda *"Parámetros del laboratorio"* del notebook:

```python
SEED           = 42            # semilla reproducible
N_VENTAS       = 2_000         # filas fact_ventas
N_INVENTARIO   = 600           # filas fact_inventario
N_CAMPANAS     = 150           # filas fact_campanas
N_DEVOLUCIONES = 300           # filas fact_devoluciones
N_CLIENTES     = 500           # filas dim_cliente
FECHA_INICIO   = "2025-01-01"
FECHA_FIN      = "2025-12-31"
DB_NAME        = "DataRetail360.db"
```

`RUN_TS` y `RUN_DATE` se calculan en tiempo de ejecución y se usan en nombres de CSV, rechazos y logs.

---

## 6. Uso

Abrir `ETL_DataRetail360.ipynb` en Google Colab o Jupyter y ejecutar **Entorno de ejecución → Ejecutar todo**.

Al ejecutarse, el pipeline:

1. **Genera** las 10 tablas Bronze con funciones pseudoaleatorias (RUT válidos ≥ 90%) y las guarda como `data/{tabla}_{RUN_TS}.csv`.
2. **Extrae** leyendo esos CSV; si alguno falta, lo regenera y lo persiste antes de leerlo.
3. **Transforma** aplicando las reglas de calidad y enviando las filas inválidas a `rejected/Rejected_{RUN_TS}.csv` con la columna `motivo_rechazo`.
4. **Carga** dimensiones primero y luego hechos en `DataRetail360.db` (DDL de `Tablas.md`).
5. **Registra** la corrida en `etl_control` para bloquear una segunda ejecución el mismo día.

Resultados y artefactos:

- `data/{tabla}_{timestamp}.csv` — datos Bronze con timestamp.
- `rejected/Rejected_{timestamp}.csv` — cuarentena con `motivo_rechazo`.
- `logs/etl_{timestamp}.log` — log de la corrida (consola + archivo).
- `DataRetail360.db` — base SQLite con el modelo dimensional completo.

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
├── data/                     # CSV Bronze con timestamp (NO se versiona)
├── rejected/                 # Rejected_{ts}.csv (NO se versiona)
├── logs/                     # etl_{ts}.log (NO se versiona)
└── DataRetail360.db          # Base SQLite generada al ejecutar (NO se versiona)
```

---

## 8. Calidad de datos
## REVISAR Y ACOMODAR

| Regla | Dimensión de calidad | Acción | Motivo de rechazo |
|---|---|---|---|
| Clave primaria o natural nula | Completitud | Enviar a rechazos | `clave_nula` |
| FK nula o inexistente en dimensión | Integridad referencial | Enviar a rechazos | `fk_nula` |
| RUT chileno con DV inválido | Validez | Enviar a rechazos | `rut_invalido` |
| Monto negativo | Validez | Enviar a rechazos | `monto_negativo` |
| `monto_neto > monto_bruto` o negativo | Consistencia | Enviar a rechazos | `monto_neto_invalido` |
| Cantidad ≤ 0 | Validez | Enviar a rechazos | `cantidad_invalida` |
| Estado fuera de dominio | Validez | Enviar a rechazos | `estado_desconocido` |
| Clave natural duplicada | Unicidad | Conservar la primera, rechazar el resto | `duplicado` |

Los motivos están centralizados en el diccionario `REGLAS` del notebook y el test `test_motivos_mapeados` verifica que todo motivo emitido exista allí.

---

## 9. Logging y manejo de errores
## REVISAR 

- Los registros se guardan en `logs/etl_{timestamp}.log` (y se imprimen en consola) con formato `timestamp | nivel | [paso] | mensaje`.
- Niveles usados: `DEBUG`, `INFO`, `WARNING`, `ERROR`.
- Códigos de paso: `E` (generación/extracción), `T` (transformación), `L` (carga), `S` (sistema, tests y orquestación).
- Las filas inválidas se persisten en `rejected/Rejected_{timestamp}.csv` con su `motivo_rechazo` para intervención manual.
- El pipeline es **idempotente**: `etl_control(run_date PRIMARY KEY)` bloquea una segunda corrida el mismo día y la carga usa `if_exists="replace"`, por lo que un reintento manual deja el mismo estado final sin duplicados.
- Errores recuperables (reintento): archivo CSV ausente (se regenera).
- Errores fatales (detener y alertar): fallo de conexión/escritura en SQLite o violación de FK (`PRAGMA foreign_key_check`).

---

## 10. Pruebas
## REVISAR

El notebook incluye `run_tests()`, que ejecuta 10 pruebas autocontenidas y muestra una tabla PASS/FAIL:

| Test | Qué valida |
|---|---|
| `test_generacion_bronze` | 10 tablas generadas; columnas clave; ausencia de FKs marcadas **O** |
| `test_rut_valido_90` | ≥ 90% de RUT con DV válido |
| `test_csv_con_timestamp` | Los 10 CSV existen con `RUN_TS` |
| `test_extract_lee_csv` | EXTRACT devuelve todas las tablas desde CSV |
| `test_extract_regenera_si_falta` | Si falta un CSV, se regenera, persiste y extrae |
| `test_transform_rechaza_rut_invalido` | RUT inválidos van a rechazos, no a `dim_cliente` |
| `test_motivos_mapeados` | Todo `motivo_rechazo` existe en `REGLAS` |
| `test_rechazos_con_motivo` | `Rejected.csv` tiene `motivo_rechazo` sin nulos |
| `test_load_sqlite` | Conteos por tabla y `PRAGMA foreign_key_check` sin violaciones |
| `test_idempotencia_diaria` | Registrar dos veces la misma fecha no duplica `etl_control` (DB temporal) |

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
