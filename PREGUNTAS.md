Respuestas estructuradas a las preguntas clave del desafío TI-3188 basadas en el documento: 

PUNTO DE VISTA TEORICO.

1. ¿Por qué se duplican los turnos si el código "valida el cupo" antes de guardar? 

Porque existe una condición de carrera del tipo TOCTOU (Time-of-Check to Time-of-Use). La 
validación se ejecuta en el código de la aplicación (haciendo una consulta SELECT para contar 
los turnos asignados) antes de ejecutar el INSERT o UPDATE. Como esa lectura y la escritura 
posterior no forman una transacción con bloqueo pesimista (FOR UPDATE) ni existe una 
restricción de integridad a nivel de motor de base de datos, dos operaciones concurrentes leen 
que el cupo sigue disponible y ambas guardan con éxito. 

2. ¿En qué condiciones se rompe esa validación? ¿Cómo lo reproduce? 

● Condiciones: Ocurre durante picos de concurrencia, específicamente en las mañanas 
cuando varias coordinadoras ajustan la plani la simultáneamente sobre los mismos 
puestos, fechas y bloques. 

● Reproducción confiable: No se puede reproducir mediante clics manuales secuenciales. 
Se reproduce mediante un script de pruebas automatizado o prueba de integración que 
dispare dos o más peticiones HTTP concurrentes (en paralelo) para asignar un turno 
en el mismo puesto, fecha y bloque con cupo 1. 

3. ¿De dónde sale el saldo negativo? ¿Es el mismo mecanismo que el duplicado? 

● Mecanismo: No es el mismo mecanismo técnico (el duplicado es concurrencia; el saldo 
negativo es falta de atomicidad), aunque están encadenados operativamente. 

● Origen: Al no existir la función de "mover", la coordinadora debe cancelar el turno actual 
(lo que resta horas al saldo semanal) e intentar asignar el nuevo turno a mano. Si al 
momento de reasignar el cupo destino está ocupado, fala la conexión o colisiona con 
otra usuaria, el turno nuevo no se crea y el original ya fue cancelado, dejando al 
trabajador con menos horas de las contractuales. 

4. ¿Cómo debería funcionar "mover" como acción de verdad (no cancelar y asignar a mano)? 

Debe implementarse mediante el endpoint acordado PATCH /api/turnos/{id}/mover ejecutado 
como una única transacción atómica: 

  1. Iniciar transacción y bloquear la fila del turno y/o el recurso destino mediante SELECT ... 
     FOR UPDATE para evitar lecturas sucias concurrentes. 

  2. Validar que el nuevo puesto/bloque/fecha tenga cupo real disponible. 

  3. Actualizar la fecha, bloque y/o puesto en una sola instrucción. 

  4. Si el destino no tiene cupo o fala cualquier validación, aplicar ROLLBACK total para que 
     el turno de origen permanezca intacto, sin restar horas ni perder el registro. 

5. ¿Qué pasa con el saldo, el reporte y la nómina cuando un turno se mueve? 

● Saldo de horas: Si el movimiento es dentro de la misma semana y entre bloques de igual 
  duración (por ejemplo, mañana a tarde: 8 hrs), el saldo queda inalterado. Si cambia entre 
  bloques de distinta duración (mañana 8 hrs vs. noche 10 hrs) o cruza semanas distintas, 
  el saldo se recalcula sumando o restando el diferencial exacto. 

● Reporte de dotación: Refleja de inmediato el decremento de 1 persona en el origen y el 
  incremento de 1 persona en el destino en pantala y exportación CSV. 

● Nómina: Si el turno se mueve dentro del período activo previo al cierre, impacta el total 
  acumulado del mes. Si el turno corresponde a un período o mes que ya fue 
  exportado a remuneraciones, no se debe permitir la modificación silenciosa, ya que 
  la nómina enviada es inmutable para conciliación. 

6. ¿Qué entregas primero para que el dolor pare ya, y qué dejas para después? 

 ● Primero (Camino crítico e inmediato): 
 
  1. La garantía de integridad en base de datos (bloqueo / restricción de cupo) para evitar 
     que sigan naciendo turnos duplicados en producción desde hoy. 
  
  2. El endpoint atómico PATCH /api/turnos/{id}/mover respaldado bajo la configuración turnos_mover_atomico. 
  
  3. El manual de mitigación con SQL para Operaciones. 

 ● Para después (Fase 2 / Backlog): 
 
  1. Integración de la acción en la interfaz visual de React (clic derecho  modal de mover). 
  
  2. Bloqueo de edición sobre semanas o meses con nómina cerrada/exportada. 

  3. Notificaciones en tiempo real (WebSockets / SSE) para refrescar la planila de otras 
     coordinadoras cuando un puesto sea ocupado. 

7. ¿Qué le entregas a operaciones mientras tanto? 

   Se entrega el manual ops/MITIGACION.md estructurado en cuatro secciones: 

     1. Detectar: Consultas SQL simples de solo lectura que identifiquen los puestos que 
        exceden el cupo configurado por bloque/fecha (COUNT(*) > cupo) y los trabajadores con 
        descuadre entre horas asignadas y su contrato semanal. 
     
     2. Corregir: Scripts con sentencias UPDATE puntuales referenciando identificadores 
        directos (WHERE id = ...) para cancelar o reasignar el turno sobrante sin usar DELETE 
        masivos ni joins destructivos. 
     
     3. Verificar: Consultas de validación para ejecutar antes y después de cada corrección, 
        asegurando que la dotación volvió al cupo y los saldos semanales cuadraron a cero.

        
4. Escalar: Criterios claros de detención (por ejemplo: si el turno duplicado pertenece a un 
mes con nómina ya enviada o si hay inconsistencias en contratos múltiples) para 
congelar el caso y derivarlo a desarrolo.

EN FUNCION DE LO PRACTICADO.

# Respuestas — TI-3188

### 1. ¿Por qué se duplican los turnos si el código "valida el cupo" antes de guardar?
Porque [`AsignarTurno::procesar`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/app/Clinica/Logicas/Turnos/AsignarTurno.php) ejecutaba una lectura (`validarCupo` → `COUNT(*)`) y luego una escritura (`$turno->save()`) sin transacción ni bloqueo pesimista (*check-then-act*). Cuando dos peticiones llegan al mismo tiempo, ambas leen `ocupados = 0` antes de que la otra inserte y ambas aprueban la validación (`id=165` e `id=182` tienen el mismo `creado_at`).

### 2. ¿En qué condiciones se rompe esa validación? ¿Cómo lo reproduces?
Se rompe cuando dos o más peticiones concurrentes asignan sobre el mismo puesto/fecha/bloque (o al mismo trabajador) dentro de la ventana de milisegundos entre el `SELECT COUNT(*)` y el `INSERT`. Se reproduce de forma determinista en [`AsignarTurnoConcurrenciaTest`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/tests/Feature/AsignarTurnoConcurrenciaTest.php) (interceptando el evento `Turno::creating`) y con concurrencia HTTP real mediante [`simular_carrera_asignar.py`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/tests/concurrencia/simular_carrera_asignar.py) y [`AsignarTurnoConcurrenciaRealTest`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/tests/Feature/AsignarTurnoConcurrenciaRealTest.php).

### 3. ¿De dónde sale el saldo negativo? ¿Es el mismo mecanismo que el duplicado?
No es el mismo mecanismo, pero sí consecuencia del mismo flujo de trabajo en horas punta: como no existía "mover turno", la coordinadora cancelaba primero el turno (`POST /api/turnos/{id}/cancelar`) y luego intentaba asignarlo en el nuevo día (`POST /api/turnos`). Si en ese instante otra coordinadora ocupaba el cupo destino, la asignación fallaba con `409`, el turno original quedaba cancelado y [`TurnoObserver`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/app/Observers/TurnoObserver.php) recalculaba el saldo semanal con 8 horas menos (además de los casos legítimos por permisos sin goce o cambio de contrato a mitad de semana).

### 4. ¿Cómo debería funcionar "mover" como acción de verdad (no cancelar y asignar a mano)?
En [`MoverTurno::procesar`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/app/Clinica/Logicas/Turnos/MoverTurno.php) (`PATCH /api/turnos/{id}/mover`), toda la operación corre en una sola transacción bloqueando en orden `trabajador -> puesto destino -> turno origen` ([ADR 0002](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/docs/decisions/0002-mover-turno-como-cancelar-y-crear.md)). Si el destino tiene cupo y contrato vigente (y ningún periodo está exportado), cancela el turno origen y crea el nuevo; si cualquier validación falla, hace *rollback* y el turno original nunca se pierde.

### 5. ¿Qué pasa con el saldo, el reporte y la nómina cuando un turno se mueve?
Al cancelar el turno origen y crear el nuevo como dos modelos `Turno`, [`TurnoObserver::saved`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/app/Observers/TurnoObserver.php) se dispara para ambos: recalcula el saldo de la semana/contrato origen y el de la semana/contrato destino, e invalida la caché del reporte de dotación (`dotacion_version`). La nómina solo considera `Turno::activos()`, por lo que ignora el turno origen cancelado y contabiliza el nuevo sin duplicar horas.

### 6. ¿Qué entregas primero para que el dolor pare ya, y qué dejas para después?
Primero se entrega el guía de mitigación operativa ([`ops/MITIGACION.md`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/ops/MITIGACION.md)) y el bloqueo transaccional en [`AsignarTurno`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/app/Clinica/Logicas/Turnos/AsignarTurno.php) (siempre activo), junto con el endpoint `PATCH /api/turnos/{id}/mover` apagado por defecto bajo `turnos_mover_atomico`. Se deja para después la integración en el frontend React y la corrección de deuda técnica heredada que cambiaría contratos actuales (`bloquearPeriodoExportado` y el `catch` que devuelve `201/id=0`; ver [PLAN.md](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/plans/PLAN.md)).

### 7. ¿Qué le entregas a operaciones mientras tanto?
El runbook [`ops/MITIGACION.md`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/ops/MITIGACION.md) con consultas SQL listas para copiar y pegar que detectan sobrecupos activos (`165,182` en octubre y `87,179` en septiembre), turnos dobles de un mismo trabajador y saldos desfasados, diferenciándolos de falsos positivos (contratos vencidos como `id=180` o permisos sin goce). Incluye sentencias `UPDATE ... WHERE id = ?` en transacción para cancelar/reactivar y recalcular `saldos`, además de reglas claras de cuándo detenerse y escalar (ej. periodos ya exportados como `2026-09`).

### 8. ¿Cómo llegaste a la causa?
Inspeccionando el camino de `POST /api/turnos` en [`AsignarTurno.php`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/app/Clinica/Logicas/Turnos/AsignarTurno.php) y [`TurnosActivosPorBloque.php`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/app/Clinica/Consultas/TurnosActivosPorBloque.php), donde vi que el conteo de ocupados y el `save()` ocurrían sin transacción ni `lockForUpdate()`, y cruzándolo con [`PlanillaSeeder.php`](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/database/seeders/PlanillaSeeder.php), que planta los dos turnos del `2026-10-14` con el mismo timestamp `2026-10-13 09:12:03` y el turno cancelado de Carla Muñoz el `2026-10-13`. Luego confirmé el comportamiento con el test de caracterización y el script multihilo en Python contra los workers de `php-fpm`.

### 9. ¿Qué alternativas descartaste y por qué?
Descarté crear un índice `UNIQUE (puesto_id, fecha, bloque)` porque hay cupos mayores a 1, cupos variables en el tiempo y turnos cancelados en la misma tabla; y descarté hacer `lockForUpdate()` directo sobre `turnos` porque con 0 filas activas InnoDB bloquea por *gap locks* sobre el índice `fecha`, afectando otros puestos ([ADR 0001](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/docs/decisions/0001-bloqueo-pesimista-por-filas-padre.md)). En `MoverTurno` descarté hacer `UPDATE` sobre la misma fila porque no recalcularía el saldo de la semana origen al cambiar de semana ([ADR 0002](file:///home/jfesq/proyectos/rflex-semi-senior-fullstack-challenge-2026-11-4e1b/docs/decisions/0002-mover-turno-como-cancelar-y-crear.md)).

### 10. ¿Cómo usaste la IA (qué le pediste, qué aceptaste, qué rechazaste)?
La usé para explorar rápidamente las consultas Eloquent (`TurnosActivosPorBloque`, `validarCupo`), generar el andamiaje del script de concurrencia en Python (`threading.Barrier`), redactar los tests de PHPUnit y estructurar las consultas SQL de `ops/MITIGACION.md`. Acepté la estrategia de bloquear las filas padre (`trabajadores` y `puestos`) en orden fijo y el diseño de `MoverTurno` como cancelar + crear; corregí una primera suposición sobre usar el test de evento `Turno::creating` para validar el fix (ya que corre en la misma conexión y no prueba locks entre transacciones, por lo que sumamos `AsignarTurnoConcurrenciaRealTest` con `Http::pool`) y ajusté la aserción de JSON en `ParidadConfigApagadaTest` porque MySQL reordena las llaves de columnas `JSON`.

### 11. ¿Qué harías distinto con más tiempo?
Conectaría el menú contextual de la planilla en React al nuevo endpoint `PATCH /api/turnos/{id}/mover` cuando la flag esté activa, y abriría un segundo PR para corregir el typo `bloquearPeriodoExportado` → `bloquear_periodo_exportado`, eliminar la clave duplicada `validarCupo` en `configurations` y reemplazar el `catch` de `AsignarTurno` que devuelve `201` con `id = 0` por un error `500` real monitoreado.


