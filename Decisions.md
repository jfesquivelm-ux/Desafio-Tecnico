DECISIONES TECNICAS (ADR) (decisions):

ADR-001: 
Bloqueo Pesimista para Control de Sobrecupo.
Contexto: Múltiples usuarias asignan turnos simultáneamente en horas de la mañana, generando sobrecupo en celdas con cupo 1 o 2.   
Decisión: Ejecutar la verificación de cupos dentro de una transacción con SELECT ... FOR UPDATE sobre el registro de configuración/cupo del puesto para la fecha y bloque objetivo.
Alternativa rechazada: Triggers con bloqueos a nivel de tabla o aislamiento SERIALIZABLE en toda la base, por alto riesgo de bloqueos globales y penalización de rendimiento.Trade-off 
asumido: Pequeña penalización de milisegundos en peticiones que intenten escribir exactamente en el mismo puesto/bloque/fecha, a cambio de consistencia e integridad matemática absoluta.

ADR-002: 
Endpoint Atómico para Mover Turnos.
Contexto: El usuario simula "mover" cancelando y reasignando. Si la segunda falla, el turno original se pierde.   
Decisión: Implementar PATCH /api/turnos/{id}/mover ejecutando la validación del destino, la liberación del origen y la actualización del turno en una única transacción de base de datos (BEGIN ... COMMIT).
Alternativa rechazada: Orquestación en dos llamadas desde el frontend con rollback manual si falla la segunda.Trade-off. 
asumido: Obliga al backend a manejar escenarios de validación compuesta y responder códigos de estado semánticos claros para el cliente.

ADR-003: 
Feature Flag 'turnos_mover_atomico'.
Contexto: Se requiere habilitar la nueva funcionalidad de forma controlada sin alterar el comportamiento productivo preexistente hasta validar estabilidad en producción.
Decisión: Registrar el parámetro turnos_mover_atomico con valor false por defecto en la tabla configurations.
Alternativa rechazada: Variables de entorno estáticas (.env), que obligan a reiniciar contenedores/servicios y no permiten pruebas dinámicas.Trade-off 
asumido: Consulta ligera adicional a la tabla/caché de configuraciones al invocar el endpoint.
