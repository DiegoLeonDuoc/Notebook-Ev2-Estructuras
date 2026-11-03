Lista de verificación para revisar un pipeline antes de integrarlo
Cómo usar: El revisor marca cada ítem antes de aprobar el Pull Request. Si algo no se cumple, se comenta y se corrige antes del merge.

1. Funcionalidad
☐  El pipeline cumple su objetivo (extrae, transforma y carga lo esperado).
☐  El resultado coincide con una verificación manual o un conteo de referencia.
☐  Carga las dimensiones antes que los hechos (claves foráneas válidas).

2. Manejo de errores
☐  Usa try/except para fallas esperables; no se cae silenciosamente.
☐  Las filas inválidas van a una tabla/registro de rechazos con su motivo.
☐  Distingue errores recuperables (reintento) de fatales (detener y alertar).

3. Logging y trazabilidad
☐  Usa logging (no print) con niveles adecuados (INFO/WARNING/ERROR).
☐  Registra inicio/fin y conteos (leídas, cargadas, rechazadas).
☐  No registra datos personales ni secretos en los logs.

4. Idempotencia y repetibilidad
☐  Ejecutar el pipeline dos veces no duplica datos.
☐  Soporta carga incremental o recarga controlada.

5. Validación de datos
☐  Incluye aserciones o validaciones finales (sin nulos en claves, rangos válidos).
☐  Aplica las reglas de calidad relevantes (unicidad, consistencia, validez).

6. Seguridad y secretos
☐  No hay contraseñas ni claves de API en el código.
☐  Las credenciales se leen de variables de entorno o gestor de secretos.
☐  El .gitignore excluye datos, .env y artefactos locales.

7. Legibilidad y mantenibilidad
☐  Nombres claros de funciones y variables; código comentado donde aporta.
☐  Configuración (rutas, parámetros) separada del código.
☐  Funciones cortas y con una responsabilidad clara.

8. Documentación y versionado
☐  El repositorio tiene un README (plantilla 2.3.3) actualizado.
☐  Los mensajes de commit son claros y descriptivos.
☐  El cambio llega por una rama y un Pull Request, no directo a la rama principal.

9. Veredicto de la revisión
Resultado	Significado
Aprobado	Cumple todo lo esencial; se puede integrar (merge).
Aprobado con comentarios	Cambios menores sugeridos; integrar tras ajustarlos.
Requiere cambios	Hay ítems críticos sin cumplir; corregir y volver a revisar.
