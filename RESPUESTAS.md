# Respuestas Resumidas (TI-3188)

### 1. ¿Por qué se duplican los turnos si el código valida antes de guardar?
Porque la lectura del cupo (`SELECT COUNT(*)`) y la inserción (`INSERT`) en `AsignarTurno::procesar` se ejecutan de forma concurrente sin transacción ni bloqueo pesimista en base de datos. Múltiples procesos leen cupo disponible simultáneamente y escriben en paralelo (condición de carrera *TOCTOU*; ver [PLAN.md](../plans/PLAN.md)).

### 2. ¿En qué condiciones se rompe esa validación y cómo reproducirlo?
Se quiebra durante ventanas de alta contención matutina cuando dos o más coordinadoras asignan sobre el mismo puesto/fecha/bloque en la misma ventana de milisegundos. Se reproduce de manera determinista mediante [`AsignarTurnoConcurrenciaTest`](../tests/Feature/AsignarTurnoConcurrenciaTest.php) y con peticiones HTTP paralelas reales en [`AsignarTurnoConcurrenciaRealTest`](../tests/Feature/AsignarTurnoConcurrenciaRealTest.php) y [`simular_carrera_asignar.py`](../tests/concurrencia/simular_carrera_asignar.py).

### 3. ¿De dónde sale el saldo negativo? ¿Es el mismo mecanismo?
Nace de la falta de una operación atómica para trasladar turnos: la coordinadora cancela el turno en el día A (`POST /api/turnos/{id}/cancelar`) y luego intenta asignar en el día B (`POST /api/turnos`). Si el día B rechaza la inserción por cupo lleno (`409`), el turno original ya quedó cancelado, restando 8 o 10 horas semanales al saldo del trabajador (además de los saldos negativos legítimos por permisos sin goce o cambios de contrato).

### 4. ¿Cómo debería funcionar "mover" como acción de verdad?
Como una transacción de base de datos única y atómica (`PATCH /api/turnos/{id}/mover` en [`MoverTurno.php`](../app/Clinica/Logicas/Turnos/MoverTurno.php)): adquiere bloqueo pesimista (`FOR UPDATE`), valida disponibilidad, cancela el turno origen y crea el nuevo turno en destino (ver [ADR 0002](decisions/0002-mover-turno-como-cancelar-y-crear.md)). Si el destino está lleno o es inválido, revierte todo (*rollback*) manteniendo intacto el turno del día origen.

### 5. ¿Qué pasa con el saldo, reporte y nómina al mover un turno?
Dentro de la misma semana el saldo neto no se altera (variación cero), y si cruza de semana `TurnoObserver` recalcula automáticamente el saldo de la semana origen y de la semana destino. El reporte de dotación invalida su caché y actualiza ambos días sin sobrecupos, y la nómina mensual solo contabiliza el turno activo final (bloqueando movimientos si el periodo ya fue exportado).

### 6. ¿Qué entregas primero para que el dolor pare ya, y qué dejas para después?
Primero se entrega el bloqueo transaccional en la validación de cupos (`AsignarTurno`) para frenar nuevos duplicados de inmediato, junto con la guía de mitigación SQL para Operaciones y el endpoint `PATCH /api/turnos/{id}/mover` detrás de la configuración `turnos_mover_atomico` (apagada por defecto; ver [ADR 0003](decisions/0003-aislamiento-feature-flag-y-paridad.md)). Se deja para después la integración visual en el frontend React y el saneamiento de configuraciones heredadas.

### 7. ¿Qué le entregas a operaciones mientras tanto?
Se entrega un manual técnico operacional ([`ops/MITIGACION.md`](../ops/MITIGACION.md)) con consultas SQL de diagnóstico para identificar sobrecupos activos y saldos desfasados sin confundirlos con falsos positivos. Incluye sentencias `UPDATE` puntuales acotadas por `id` dentro de transacciones para subsanar turnos sin borrar datos, el paso obligatorio de recálculo de saldo y criterios claros de escalamiento.

### 8. ¿Cómo llegaste a la causa raíz?
Inspeccioné el flujo de asignación en `AsignarTurno` y `TurnosActivosPorBloque`, comprobando que el `SELECT COUNT(*)` y el guardado ocurrían sin `DB::transaction` ni `lockForUpdate()`. Luego confirmé la evidencia en la base de datos y en `PlanillaSeeder`, donde los turnos duplicados (`id=165` e `id=182`) comparten exactamente el mismo timestamp de creación (`2026-10-13 09:12:03`).

### 9. ¿Qué alternativas descartaste y por qué?
Descarté restricciones `UNIQUE` en base de datos porque existen puestos con cupo mayor a uno, vigencias temporales de cupo y turnos cancelados en la misma tabla. También descarté bloquear el `COUNT(*)` directo en `turnos` por los *gap locks* de InnoDB sobre el índice `fecha`. Asimismo, descarté locks externos en caché/Redis para no sumar infraestructura a días del cierre de nómina (ver [ADR 0001](decisions/0001-bloqueo-pesimista-por-filas-padre.md)).

### 10. ¿Cómo usaste la IA en el proceso?
Se utilizó para acelerar la exploración del repositorio, estructurar plantillas Markdown, contrastar el orden de bloqueo pesimista (`trabajador -> puesto`) y generar el script y los tests de concurrencia. Se rechazaron o corrigieron suposiciones iniciales como probar bloqueos de base de datos en una misma conexión o realizar limpiezas automáticas destructivas en migraciones.

### 11. ¿Qué harías distinto con más tiempo?
Conectaría la acción de mover en el frontend React y corregiría en un PR separado el typo heredado `bloquearPeriodoExportado` junto con el `catch` silencioso de `AsignarTurno`. Además, implementaría actualización en tiempo real (WebSockets/polling) en la planilla de las coordinadoras y una herramienta interna asistida para Operaciones.
