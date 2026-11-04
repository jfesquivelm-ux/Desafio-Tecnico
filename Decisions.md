# Decisiones Técnicas (ADR) — TI-3188

---

# ADR 0001: Bloqueo pesimista sobre filas padre (`trabajadores` y `puestos`)

- **Estado:** Aceptado
- **Fecha:** 2026-11-04
- **Ticket:** TI-3188

## Contexto
[`AsignarTurno::procesar`](../../app/Clinica/Logicas/Turnos/AsignarTurno.php) validaba el cupo disponible (`TurnosActivosPorBloque::contar`) y si el trabajador ya tenía turno en ese bloque antes de insertar, pero sin transacción ni bloqueo (*Time-of-Check to Time-of-Use*). Bajo peticiones simultáneas (varias coordinadoras en la mañana o doble clic), dos procesos leían el mismo conteo (`0` ocupados) y ambos insertaban, superando el cupo del puesto (`id=165` e `id=182` creados en el mismo segundo `2026-10-13 09:12:03`) o duplicando el turno del trabajador.

## Decisión
Envolver la validación y la creación del turno en `DB::transaction(..., 3)` y adquirir al inicio `SELECT ... FOR UPDATE` (`lockForUpdate()`) sobre la fila de `trabajadores` y luego sobre la fila de `puestos` mediante `AsignarTurno::bloquearTrabajadorYPuesto($idTrabajador, $idPuesto)` (siempre en orden estricto `trabajador -> puesto`). La misma estrategia y orden de bloqueo se reutilizan en [`MoverTurno`](../../app/Clinica/Logicas/Turnos/MoverTurno.php).

## Alternativas descartadas
1. **Índice `UNIQUE` en `turnos (puesto_id, fecha, bloque)`:** No es viable porque existen puestos con cupo mayor a 1 (ej. `URG Triage` bloque `M` tiene cupo 2), los cupos cambian según vigencia temporal (`cupos.vigente_desde` / `vigente_hasta`), los turnos cancelados (`cancelado_at IS NOT NULL`) permanecen en la misma tabla y ya existen duplicados históricos en la base de datos que harían fallar la migración.
2. **`lockForUpdate()` directo sobre `turnos` en la consulta de conteo (`TurnosActivosPorBloque::contar`):** Cuando aún no existen turnos activos en ese puesto/fecha/bloque (`COUNT = 0`), InnoDB no tiene filas que bloquear y aplica *gap locks* sobre el índice `fecha`, bloqueando inserciones de otros puestos en el mismo día y aumentando la probabilidad de *deadlocks*. Además, modificar `TurnosActivosPorBloque` afectaría innecesariamente al reporte de dotación.
3. **Locks de aplicación (`Cache::lock` / `GET_LOCK` de MySQL):** `Cache::lock` depende del driver configurado (`array`/`file` en local y testing, que no comparte estado entre procesos o contenedores), y `GET_LOCK` opera a nivel de sesión sin liberarse automáticamente ante un *rollback* de transacción como los *row locks* de InnoDB.

## Trade-offs asumidos
- Todas las asignaciones concurrentes que involucren al mismo puesto (incluso en fechas distintas) o al mismo trabajador se serializan durante los pocos milisegundos que dura la transacción. Dado el volumen operativo de una clínica (coordinadoras humanas planificando turnos, no miles de transacciones por segundo sobre el mismo puesto), esa espera es imperceptible y elimina por completo tanto los *gap locks* como las condiciones de carrera.

---

# ADR 0002: Implementar "mover turno" como cancelar origen + crear nuevo y contrato HTTP

- **Estado:** Aceptado
- **Fecha:** 2026-11-04
- **Ticket:** TI-3188

## Contexto
Al no existir la acción de mover turno, las coordinadoras cancelaban el turno original (`POST /api/turnos/{id}/cancelar`) y luego intentaban asignar otro manualmente (`POST /api/turnos`). Si la segunda petición fallaba por cupo lleno (`409`), el turno original ya quedaba cancelado ("desaparecía") y el saldo semanal quedaba negativo. Además, cuando un turno se mueve de día puede cruzar de una semana a otra o de un contrato a otro (ej. el cambio de contrato de Bruno Soto el `2026-10-15`), y el desafío requiere documentar el contrato de respuestas de `PATCH /api/turnos/{id}/mover`.

## Decisión
Implementar [`MoverTurno::procesar`](../../app/Clinica/Logicas/Turnos/MoverTurno.php) (`PATCH /api/turnos/{id}/mover` con `{fecha, bloque, puesto_id?}`) dentro de una única transacción `DB::transaction(..., 3)` que:
1. Bloquea en orden fijo `trabajador -> puesto destino -> turno origen` (`lockForUpdate()`).
2. Valida que ni el turno origen ni el periodo destino estén exportados a nómina, que exista contrato vigente en la fecha destino y que haya cupo disponible en destino.
3. Marca `cancelado_at` en el turno origen, crea una nueva instancia de `Turno` con el `contrato_id` vigente en la fecha destino y las `horas` del nuevo bloque, y registra la acción `moverTurno` en `auditoria`.

### Contrato de respuestas HTTP de `PATCH /api/turnos/{id}/mover`

| Caso | Código HTTP | Respuesta JSON | Justificación |
|---|---|---|---|
| **Éxito** | `200 OK` | `{"ok": true, "id": <id_nuevo>, "id_anterior": <id_origen>}` | Consistente con `cancelar` (`200`) y `asignar` (`ok`, `id`), entregando ambos IDs al frontend. |
| **Cupo lleno o trabajador ya tiene turno en destino** | `409 Conflict` | `{"error": "Cupo completo para el bloque"}` / `{"error": "El trabajador ya tiene turno en ese bloque"}` | Mismo código y mensaje que `POST /api/turnos`; hace *rollback* sin cancelar el turno origen. |
| **Turno origen cancelado o periodo origen/destino exportado** | `409 Conflict` | `{"error": "El turno esta cancelado"}` / `{"error": "El turno ya fue exportado a nomina"}` | Conflicto con el estado actual del recurso o del periodo de nómina cerrado. |
| **Datos inválidos (`fecha`, `bloque`, `puesto_id`, sin contrato o mismo slot)** | `422 Unprocessable Entity` | `{"error": "<mensaje de validación>"}` | Mismo estándar `422` que usa `AsignarTurno` para errores de entrada o reglas de dominio previas. |
| **Turno inexistente o `turnos_mover_atomico = 0`** | `404 Not Found` | `{"error": "Turno no existe"}` / `{"error": "No encontrado"}` | Mismo estándar `404` que usa `CancelarTurno` cuando el ID no existe. |

## Alternativas descartadas
- **Modificar (`UPDATE`) la misma fila de `turnos` cambiando `fecha`, `bloque` y `puesto_id` en el lugar:**
  - Si el turno se mueve de una semana `A` a una semana `B` (o cambia de `contrato_id`), [`TurnoObserver::saved`](../../app/Observers/TurnoObserver.php) solo recibe los atributos nuevos (`$turno->contrato_id` y `$turno->fecha` nuevos) y recalcularía únicamente el saldo de la semana destino, dejando el saldo de la semana origen desactualizado con horas fantasma.
  - Sobrescribir la fila borra el rastro histórico en la planilla del día original (`planilla_mostrar_cancelados`) y dificulta la auditoría operacional.

## Trade-offs asumidos
- Cada movimiento genera una fila nueva en `turnos` (dejando el turno anterior con `cancelado_at` poblado) y dispara dos veces `TurnoObserver::saved` (al cancelar el origen y al crear el destino). A cambio, los saldos de ambas semanas y contratos se recalculan de forma exacta reutilizando la lógica existente, se preserva la trazabilidad completa y si el destino falla toda la transacción revierte sin perder el turno original.

---

# ADR 0003: Comportamiento bajo `turnos_mover_atomico` y preservación de paridad

- **Estado:** Aceptado
- **Fecha:** 2026-11-04
- **Ticket:** TI-3188

## Contexto
El desafío exige que todo comportamiento nuevo entre bajo la configuración `turnos_mover_atomico` (tabla `configurations`, apagada por defecto `'0'` tanto en una base recién migrada como tras `make fresh`), que con ella apagada todo lo que hoy funciona siga respondiendo exactamente igual (mismas respuestas, mismo reporte de dotación, mismo archivo de nómina, misma auditoría), y que la garantía de un turno por cupo aplique siempre.

## Decisión
1. **Fix de concurrencia siempre activo en [`AsignarTurno`](../../app/Clinica/Logicas/Turnos/AsignarTurno.php):** El bloqueo transaccional no altera códigos HTTP, mensajes JSON, formato de reportes ni registros de `auditoria` (`validarCupo` se confirma incluso cuando devuelve `409`); únicamente serializa peticiones concurrentes para impedir violaciones de cupo.
2. **Persistencia dual de `turnos_mover_atomico = 0`:** Se registra tanto en la migración [`2026_11_04_000001_insert_config_turnos_mover_atomico.php`](../../database/migrations/2026_11_04_000001_insert_config_turnos_mover_atomico.php) (usando `exists()` para no tocar filas existentes) como en [`PlanillaSeeder::configuraciones()`](../../database/seeders/PlanillaSeeder.php) para sobrevivir a `make fresh`.
3. **`PATCH /api/turnos/{id}/mover` devuelve `404` cuando `turnos_mover_atomico = 0`:** Se evalúa `Configuracion::configHabilitada('turnos_mover_atomico')` al inicio de `MoverTurno::procesar` antes de abrir transacción ni tocar la base de datos.
4. **No modificar el `catch` ni las claves heredadas en `AsignarTurno` y `CancelarTurno`:** Se mantuvieron sin cambios el bloque `catch (Throwable $e)` de `AsignarTurno` (que responde `201` con `id = 0`) y la lectura de `'bloquearPeriodoExportado'` en las clases existentes para garantizar paridad estricta.

## Alternativas descartadas
- **Devolver `403 Forbidden` o `501 Not Implemented` cuando `turnos_mover_atomico = 0`:** Con `404 Not Found` el endpoint se comporta como inexistente mientras la funcionalidad está apagada.
- **Corregir en este mismo PR el `catch` (`201`/`id=0` -> `500`) y el typo `bloquearPeriodoExportado` -> `bloquear_periodo_exportado` en `AsignarTurno`/`CancelarTurno`:** Aunque son defectos reales de deuda técnica, corregirlos cambiaría las respuestas actuales de los endpoints existentes con la configuración apagada y rompería la regla de no tocar lo que no corresponde al dolor del ticket. En `MoverTurno` (código nuevo bajo la flag) sí se valida directamente `PeriodoNomina::estaExportado()`.

## Trade-offs asumidos
- Si en `AsignarTurno` se agotaran los 3 reintentos de transacción por contención extrema en base de datos, el `catch` heredado seguiría devolviendo `201` con `id = 0` (aunque gracias a la transacción hará *rollback* completo y no dejará turnos ni saldos corruptos a medias). Este hallazgo queda documentado en [`plans/PLAN.md`](../../plans/PLAN.md) para abordarse en un ticket s
