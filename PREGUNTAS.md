Respuestas estructuradas a las preguntas clave del desafío TI-3188 basadas en el documento: 

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
