DECISIONES TECNICAS (ADR) (decisions):

# ADR 0001: Bloqueo pesimista sobre filas padre (`trabajadores` y `puestos`)

## Contexto
`AsignarTurno::procesar` validaba el cupo disponible (`TurnosActivosPorBloque::contar`) y si el trabajador ya tenía turno en ese bloque antes de insertar, pero sin transacción ni bloqueo. Bajo peticiones simultáneas (varias coordinadoras o doble clic), dos procesos leían el mismo conteo (`0` ocupados) y ambos insertaban, superando el cupo del puesto o duplicando el turno del trabajador.

## Decisión
Envolver la validación y la creación del turno en `DB::transaction(..., 3)` y adquirir al inicio `SELECT ... FOR UPDATE` (`lockForUpdate()`) sobre la fila de `trabajadores` y luego sobre la fila de `puestos` (siempre en ese orden: `trabajador -> puesto`). La misma estrategia y orden de bloqueo se reutilizan en `MoverTurno`.

## Alternativas descartadas
1. **Índice `UNIQUE` en `turnos (puesto_id, fecha, bloque)`:** No es viable porque existen cupos mayores a 1 (ej. `URG Triage` mañana tiene cupo 2), los cupos cambian según vigencia temporal (`cupos`), los turnos cancelados permanecen en la misma tabla y ya existen duplicados históricos en producción.
2. **`lockForUpdate()` directo sobre `turnos` en la consulta de conteo:** Cuando aún no existen turnos activos en ese puesto/fecha/bloque, InnoDB aplica *gap locks* sobre el índice `fecha`, bloqueando inserciones de otros puestos en el mismo día y aumentando la probabilidad de *deadlocks*.
3. **Locks de aplicación (`Cache::lock` / `GET_LOCK`):** Dependen del driver de caché configurado (`array`/`file` en local/testing) o no se liberan automáticamente ante un *rollback* de transacción como los *row locks* de InnoDB.

## Trade-offs asumidos
- Todas las asignaciones concurrentes para un mismo puesto (incluso en fechas distintas) o para un mismo trabajador se serializan brevemente durante los pocos milisegundos que dura la transacción. Dado el volumen de una clínica (decenas de coordinadoras humanas, no miles de peticiones por segundo), esa serialización es imperceptible y elimina por completo los *gap locks* y las condiciones de carrera.

# ADR 0002: Implementar "mover turno" como cancelar origen + crear nuevo en una transacción

## Contexto
Al no existir la acción de mover turno, las coordinadoras cancelaban el turno original y creaban otro manualmente. Si la segunda petición fallaba por cupo lleno (`409`), el turno original quedaba cancelado ("desaparecía") y el saldo de horas de la semana quedaba negativo. Además, cuando un turno cambia de día puede cruzar de una semana a otra o de un contrato a otro (como el cambio de contrato de Bruno Soto el `2026-10-15`).

## Decisión
Implementar `MoverTurno::procesar` (`PATCH /api/turnos/{id}/mover`) dentro de una única transacción `DB::transaction` que:
1. Bloquea en orden `trabajador -> puesto destino -> turno origen`.
2. Valida que ni el turno origen ni el periodo destino estén exportados a nómina, que exista contrato vigente en la fecha destino y que haya cupo disponible.
3. Marca `cancelado_at` en el turno origen y guarda una nueva instancia de `Turno` con el `contrato_id` vigente en la nueva fecha y las `horas` del nuevo bloque.

## Alternativas descartadas
- **Modificar (`UPDATE`) la misma fila de `turnos` cambiando `fecha`, `bloque` y `puesto_id`:**
  - Si el turno se mueve de una semana `A` a una semana `B` (o cambia de `contrato_id`), `TurnoObserver::saved` solo ve los valores nuevos (`$turno->contrato_id` y `$turno->fecha` nuevos) y recalcularía únicamente el saldo de la semana destino, dejando el saldo de la semana origen desactualizado con horas fantasma.
  - Además, sobrescribir la fila borra el rastro en la planilla de que existió un turno cancelado/movido en el día original (`planilla_mostrar_cancelados`).

## Trade-offs asumidos
- Cada movimiento genera una fila adicional en `turnos` (con el origen marcado como cancelado) y dispara dos veces `TurnoObserver::saved` (una al cancelar el origen y otra al crear el destino). A cambio, los saldos de ambas semanas y contratos se recalculan de forma exacta sin lógica duplicada, se conserva el historial completo y si el cupo destino está lleno toda la transacción hace *rollback*, por lo que el turno nunca desaparece.

# ADR 0003: Comportamiento bajo `turnos_mover_atomico` y preservación de paridad

## Contexto
El desafío exige que todo comportamiento nuevo entre bajo la configuración `turnos_mover_atomico` (apagada por defecto en migración y en `make fresh`), que con ella apagada todo lo que hoy funciona siga respondiendo exactamente igual, y que la garantía de un turno por cupo aplique siempre.

## Decisión
1. **Fix de concurrencia siempre activo en `AsignarTurno`:** El bloqueo transaccional no altera códigos HTTP, mensajes JSON, formato de reportes ni auditoría en ejecuciones válidas; solo impide que dos peticiones simultáneas violen el cupo.
2. **`PATCH /api/turnos/{id}/mover` devuelve `404` cuando `turnos_mover_atomico = 0`:** Se evalúa la configuración antes de tocar la base de datos y se responde `404 {"error": "No encontrado"}`.
3. **Preservar el `catch` y las configuraciones existentes en `AsignarTurno` y `CancelarTurno`:** No se modificó el bloque `catch` de `AsignarTurno` (que devuelve `201` con `id = 0` ante excepciones no controladas) ni el nombre `'bloquearPeriodoExportado'` en las clases existentes.

## Alternativas descartadas
- **Devolver `403` o `501` cuando `turnos_mover_atomico = 0`:** Con `404` el endpoint se comporta como si la ruta aún no estuviera publicada para cualquier consumidor externo.
- **Corregir de paso el `catch` (`201`/`id=0` -> `500`) y el typo `bloquearPeriodoExportado` en `AsignarTurno`:** Aunque son bugs reales de deuda técnica, cambiarlos alteraría el comportamiento observable actual con la configuración apagada y saldría del alcance del ticket. En `MoverTurno` (código nuevo bajo la flag) sí se verifica directamente `PeriodoNomina::estaExportado()`.

## Trade-offs asumidos
- Si en `AsignarTurno` ocurriera un agotamiento de los 3 reintentos de transacción por bloqueo en base de datos, el `catch` existente seguiría respondiendo `201` con `id = 0` (aunque gracias a la transacción no dejará datos corruptos a medias). Se documenta como hallazgo para corregirlo en un ticket dedicado.

 


