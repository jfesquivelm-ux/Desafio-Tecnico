# DESAFÍO TÉCNICO FULLSTACK: TICKET TI-3188

**De:** Jaime Esquivel Marchant  
**Propuesta de Arquitectura, Plan de Mitigación Operacional y Entregables de Continuidad**

---

## 1. Problema y Análisis de Causa Raíz (TOCTOU y Falta de Atomicidad)

En Clínica Norte se reportan dos dolores recurrentes (~5 reclamos/semana), sobre todo en las mañanas cuando varias coordinadoras ajustan la planilla al mismo tiempo:

1. **Sobrecupo (turnos duplicados en un mismo puesto/bloque):** El `2026-10-14` en `URG Box 1` bloque `M` (cupo = 1) aparecen dos trabajadoras activas asignadas al mismo tiempo (Ana Pérez `id=165` y Gabriela Núñez `id=182`).
2. **Turnos que "desaparecen" y dejan saldo semanal negativo:** Al mover el turno de un trabajador a otro día, a veces el trabajador pierde el turno original sin quedar asignado en el nuevo día, quedando con saldo negativo en la semana (ej. Carla Muñoz `id=161` el `2026-10-13`).

### Mecanismo de quiebre de concurrencia (Race Condition / TOCTOU)

La verificación en capa de aplicación (`SELECT COUNT(*) < cupo`) sin cerrojo pesimista (`FOR UPDATE`) ni restricción a nivel relacional es vulnerable a condiciones de carrera (*Time-of-Check to Time-of-Use*):

- **Lectura simultánea (*Time-of-Check*):** Dos coordinadoras asignan personal en el mismo puesto, día y bloque de manera simultánea durante las mañanas. Ambas peticiones (`Req A` y `Req B`) consultan la disponibilidad al mismo instante y leen que el cupo está libre (`ocupados < cupo`).
- **Aprobación en memoria:** La capa lógica de la aplicación valida exitosamente ambas solicitudes de forma independiente porque los datos que ambas consultaron aún reflejan estado disponible.
- **Escritura no coordinada (*Time-of-Use*):** Ambas conexiones ejecutan sus respectivas sentencias `INSERT` sin bloqueo previo (`SELECT ... FOR UPDATE`), por lo que ambos registros persisten y dejan el puesto con sobrecupo (2 trabajadores en un puesto de cupo 1).

### Falta de atomicidad en el "mover" manual (causa de la desaparición y saldo negativo)

Al no existir una operación ni endpoint atómico para mover un turno, para simular un traslado la coordinadora debe ejecutar manualmente dos pasos separados:
1. `POST /api/turnos/{id}/cancelar`: cancela el turno original en el día origen (`cancelado_at`).
2. `POST /api/turnos`: intenta asignar el turno en el día destino.

Si el paso 1 tiene éxito pero el paso 2 falla (porque otra coordinadora ocupó el cupo destino una fracción de segundo antes, o por un error de validación/red), la acción queda a medio camino: el turno origen ya quedó cancelado, el nuevo nunca se creó, el trabajador pierde esas horas de su planificación semanal (8 o 10 hrs) y su saldo resulta negativo contra su contrato laboral. Asimismo, si dos coordinadoras intentan reubicar personal hacia el mismo puesto libre al mismo tiempo, ambas cancelan sus orígenes y chocan en el destino.

---

## 2. Hallazgos (evidencia en código y base de datos)

- **Condición de carrera (*check-then-act*) en `AsignarTurno::procesar`:** `validarCupo()` consulta `Cupo::vigente()`, cuenta los turnos activos con `TurnosActivosPorBloque::contar()` y verifica si el trabajador ya tiene turno en ese bloque; luego `procesar()` hace `$turno->save()` fuera de una transacción y sin bloqueo (`SELECT ... FOR UPDATE`). En la base (`turnos`), `id=165` e `id=182` tienen exactamente el mismo `creado_at = '2026-10-13 09:12:03'`.
- **Inexistencia de "mover turno" atómico:** La aplicación no tiene un endpoint para mover. Las coordinadoras cancelan el turno (`POST /api/turnos/{id}/cancelar`) y luego intentan crear otro (`POST /api/turnos`). Si en el intertanto otra coordinadora ocupa el cupo destino, el segundo paso responde `409`, pero el turno original ya quedó con `cancelado_at` poblado y `TurnoObserver` recalculó el saldo semanal a la baja.
- **Excepciones silenciadas en `AsignarTurno::procesar`:** El bloque `catch (Throwable $e)` devuelve `['ok' => true, 'codigo' => 201, 'id' => 0, 'mensaje' => 'Proceso Completado']`, ocultando errores inesperados como si fueran éxitos.
- **Anomalías adicionales en datos (`PlanillaSeeder`):**
  - `2026-09-22` (`UCI Cama 3`, `T`, `ids=87,179`): sobrecupo creado el `2026-10-02` sobre el periodo `2026-09` ya `EXPORTADO`, dejando `periodos_nomina.sucio = 1`.
  - `2026-10-08` (`URG Triage`, `T`, `ids=180,181`): aparente duplicado en tabla cruda, pero `id=180` (Diego Rojas) tiene contrato vencido el `2026-09-30`, por lo que `Turno::activos()` lo excluye correctamente.
  - `Configuracion::configHabilitada('bloquearPeriodoExportado')` busca en camelCase mientras la fila en `configurations` es `bloquear_periodo_exportado` (snake_case), y existe una fila huérfana `validarCupo = 0` junto a `validar_cupo = 1`.

---

## 3. Descomposición

1. **Integridad de cupo y de trabajador por bloque (siempre activa):** Serializar la validación y la inserción de turnos para impedir tanto el sobrecupo de un puesto como la doble asignación de un mismo trabajador en el mismo día/bloque.
2. **Operación atómica de mover turno (`PATCH /api/turnos/{id}/mover`):** Cancelar el turno origen y crear el turno destino dentro de una misma transacción, controlada por la configuración `turnos_mover_atomico` (apagada por defecto).
3. **Mitigación operativa (`ops/MITIGACION.md`):** Consultas SQL de diagnóstico y corrección acotada por `id` para que Operaciones limpie datos en producción sin intervención de desarrollo.

---

## 4. Supuestos

- El aislamiento transaccional de MySQL 8 (InnoDB `REPEATABLE READ`) está activo en producción igual que en el stack Docker.
- Con `turnos_mover_atomico = 0`, `PATCH /api/turnos/{id}/mover` debe comportarse como inexistente (`404 Not Found`) sin alterar ningún dato.
- Al mover un turno, conservar trazabilidad (fila origen con `cancelado_at` + nueva fila activa) es preferible a sobrescribir la fila en el lugar (`UPDATE`), porque dispara `TurnoObserver` en ambas fechas (recalculando el saldo de ambas semanas si cambia de semana) y preserva el historial.

---

## 5. Alternativas consideradas y descartadas

1. **Índice `UNIQUE (puesto_id, fecha, bloque)` en `turnos`:** Descartado. Existen puestos con `cupo = 2` (ej. `URG Triage` bloque `M`), el cupo varía en el tiempo (`cupos.vigente_desde`/`vigente_hasta`), los turnos cancelados comparten las mismas columnas y la migración fallaría sobre datos existentes con duplicados.
2. **`SELECT ... FOR UPDATE` sobre `turnos` en `TurnosActivosPorBloque::contar()`:** Descartado. Cuando hay `0` turnos activos para esa fecha/bloque, InnoDB recurre a *gap locks* sobre el índice `fecha`, bloqueando inserciones de otros puestos en el mismo día y aumentando el riesgo de *deadlocks*; además, modificar `TurnosActivosPorBloque` afectaría innecesariamente al reporte de dotación.
3. **Lock distribuido en caché (`Cache::lock`):** Descartado. En pruebas/local el driver de caché es `array`/`file` y acoplaría la integridad transaccional de la base de datos a un componente externo.
4. **Elegida — Transacción + `lockForUpdate()` sobre filas padre (`trabajadores` y luego `puestos`):** Las filas de `trabajadores` y `puestos` siempre existen, el bloqueo tiene granularidad exacta por trabajador y puesto, y al adquirir los locks siempre en el mismo orden (`trabajador -> puesto`) se evitan *deadlocks* entre `AsignarTurno` y `MoverTurno`.

---

## 6. Orden de entrega (valor incremental)

1. **Reproducción y línea base de paridad:** Tests que demuestran el fallo de concurrencia (`AsignarTurnoConcurrenciaTest`, `simular_carrera_asignar.py`) y snapshots de paridad (`ParidadConfigApagadaTest`) para reporte de dotación, CSV y nómina.
2. **Fix de integridad en `AsignarTurno`:** Transacción con `bloquearTrabajadorYPuesto` siempre activa y verificada con `AsignarTurnoConcurrenciaRealTest` (`Http::pool`).
3. **Configuración `turnos_mover_atomico` y `MoverTurno`:** Migración, seeder, endpoint `PATCH /api/turnos/{id}/mover` y su suite `MoverTurnoTest`.
4. **Mitigación operativa (`ops/MITIGACION.md`):** Consultas verificadas contra el seeder.

---

## 7. Qué NO se toca y por qué

- **Estructura del cálculo y motor de exportación de nómina (`GenerarArchivoNomina`):** Es el cierre mensual de remuneraciones y no admite alteraciones silenciosas de formato, hash ni cálculo.
- **Lógica del reporte de dotación (`TurnosActivosPorBloque`, `ReporteDotacionController`) y cálculo de saldos (`CalcularSaldo`):** Solo deben reflejar los turnos activos reales; su motor de agregación funciona correctamente una vez garantizada la integridad de `turnos`.
- **Contratos históricos, esquemas de permisos y datos existentes en migraciones/seeders:** Son ajenos al problema de concurrencia en la grilla de turnos y representan producción; las anomalías de datos se corrigen de forma controlada vía `ops/MITIGACION.md`.
- **El typo `bloquearPeriodoExportado` en `AsignarTurno`/`CancelarTurno` y el `catch` que devuelve `201 / id=0`:** Corregirlos cambiaría las respuestas actuales de endpoints existentes con la configuración apagada; se dejan documentados como deuda técnica para tratar en un ticket propio.
- **Frontend React:** Fuera del camino crítico; se priorizó blindar el backend y las pruebas de concurrencia/paridad.

---

## 8. Riesgos y mitigación

- **Contención o *deadlocks* en horas punta:** Mitigado acotando el lock únicamente a la fila del trabajador y del puesto involucrado (transacción de pocos milisegundos), imponiendo orden estricto de bloqueo (`trabajador -> puesto -> turno`) y configurando `DB::transaction(..., 3)` con 3 reintentos automáticos.
- **Desfase de saldos al corregir turnos manualmente por SQL en Operaciones:** Mitigado incluyendo en `ops/MITIGACION.md` la consulta de verificación de saldos (1.4) y el paso obligatorio de actualización de `saldos` (2.4), ya que un `UPDATE` en SQL no dispara `TurnoObserver`.

---

## 9. Decisiones abiertas (para alinear antes de encender la flag en producción)

- Confirmar con Frontend si el contrato de respuesta de `PATCH /api/turnos/{id}/mover` (`200 {"ok": true, "id": <nuevo>, "id_anterior": <origen>}` y errores `{"error": "..."}`) calza con su manejo de estado o requiere algún campo adicional.
- Definir con Producto si en un siguiente PR se corrige el nombre de la configuración `bloquearPeriodoExportado` -> `bloquear_periodo_exportado` y el `catch` de `AsignarTurno` que responde `201` con `id = 0`.

