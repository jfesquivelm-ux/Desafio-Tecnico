DESAFÍO TÉCNICO FULLSTACK: TICKET TI-3188

De: Jaime Esquivel Marchant

Propuesta de Arquitectura, Plan de Mitigación Operacional y Entregables de Continuidad

1. Análisis de Causa Raíz y Mecanismo del Fallo Causa del Duplicado (Race Condition / TOCTOU):

   La verificación en capa de aplicación (ej. SELECT COUNT(*) < cupo) sin cerrojo pesimista (FOR UPDATE) ni
   restricción a nivel relacional es vulnerable a condiciones de carrera (Time-of-Check to Time-of-Use). Durante las mañanas de alta concurrencia, múltiples
   coordinadoras evalúan la disponibilidad del mismo puesto y bloque en la misma ventana de milisegundos; ambas leen cupo disponible y proceden a insertar,
   generando sobrecupo en la base de datos.
   Causa de la Desaparición y Saldo Negativo: Al no existir una operación atómica para 'mover turno', el usuario ejecuta una secuencia manual:
   cancela (o elimina) el turno en el día origen y posteriormente intenta asignar en el día destino. Si la asignación en el destino falla
   (por cupo lleno o concurrencia), el turno origen ya fue destruido o cancelado. El trabajador pierde esas horas de su planificación semanal (8 o 10 hrs),
   dejando un saldo de horas negativo contra su contrato laboral.

   MECANISMO DE QUIEBRE DE CONCURRENCIA:
   
   Lectura simultánea (Time-of-Check): Dos coordinadoras asignan personal en el mismo puesto, día y bloque de manera simultánea durante las mañanas. Ambas
   peticiones (Req A y Req B) consultan la disponibilidad al mismo instante y leen que el cupo está libre (cupo_actual < cupo_maximo).

   Aprobación en memoria: La capa lógica de la aplicación valida exitosamente ambas solicitudes de forma independiente porque los datos que ambas consultaron
   aún reflejan estado disponible.

   Escritura no coordinada (Time-of-Use): Ambas conexiones ejecutan sus respectivas sentencias INSERT y confirman la transacción (COMMIT).

   Violación de integridad: Al no existir una restricción única (UNIQUE constraint), un lock pesimista (SELECT ... FOR UPDATE) ni un nivel de aislamiento
   serializable, ambos registros persisten, dejando el puesto con sobrecupo (p. ej., 2 trabajadores en un puesto de cupo 1).

   POR QUE A VECES DESAPARECEN TURNO Y EL SALDO QUEDA NEGATIVO:
   
    "El frontend no tiene implementada la operación atómica de mover. Para simular un traslado, la coordinadora debe":   DELETE / Cancelar
    el turno original en el día origen.   POST / Asignar el turno en el día destino.   Si el paso 1 tiene éxito pero el paso 2 falla
    (por ejemplo, porque otra coordinadora ocupó el cupo destino una fracción de segundo antes, o por error de red), la acción queda a medio
    camino.

    El turno original fue eliminado, el nuevo nunca se creó, el trabajador pierde horas semanales y su saldo resulta negativo.

   FALTA DE ATOMOCIDAD EN EL "MOVER" MANUAL:

    Al no existir un (endpoint) para mover, las coordinadoras ejecutan: Cancelar turno origen, Asignar turno destino.

    Si la asignación de destino falla (por cupo lleno u error de red) tras haber cancelado el origen, el turno se pierde, resultando
    en la desaparición del turno y un saldo semanal negativo.

    Si dos personas intentan reubicar personal al mismo puesto libre al mismo tiempo, ambas cancelan sus orígenes y chocan en el destino,
    provocando sobrecupo o estados inconsistentes. 

3. Plan de Desarrollo (PLAN.md).

   2.1 Descomposición del Problema y Supuestos
   
       • Inconsistencia de Cupo: Ausencia de concurrencia serializada/pesimista sobre el puesto/bloque.

       • Pérdida de Turnos: Falta de atomicidad transaccional en la acción de traslado entre días/bloques.

       • Paridad Operacional: Riesgo de mutar reportes consolidados o alterar el archivo de nómina previo al cierre de mes.

       • Supuesto: El cupo es estricto por regla de negocio; la nómina de fin de mes es inmutable para registros cerrados.

   2.2 Alternativas Evaluadas y Descartadas.
   
       Paso 1 (Integridad y Mitigación): Script de detección/corrección para Operaciones e incorporación del bloqueo pesimista
       transaccional (FOR UPDATE) en toda asignación/movimiento, garantizando que el sobrecupo no pueda ocurrir bajo ninguna
       circunstancia.

       Paso 2 (Endpoint de Negocio): Implementación del endpoint PATCH /api/turnos/{id}/mover bajo la configuración
       turnos_mover_atomico.

       Paso 3 (Compatibilidad y Pruebas): Test suite de concurrencia y regresión (nómina, reportes y paridad con
       flag apagado). 
        
   
   2.3 Alcance Incremental y Delimitación.

       • Prioridad 1: Corrección de concurrencia en la validación de cupo (aplica globalmente para frenar duplicados). 

       • Prioridad 2: Configuración turnos_mover_atomico en tabla configurations (off por defecto).

       • Prioridad 3: Implementación del endpoint PATCH /api/turnos/{id}/mover protegido por la bandera.

       • Lo que NO se toca: Motor de exportación de nómina, contratos históricos ni esquemas cerrados de auditoría.

          Estructura del cálculo y exportación de nómina: Es el cierre mensual de remuneraciones y no admite alteraciones de formato o cálculo.
          Lógica del reporte de dotación: Solo debe reflejar los turnos reales; su motor de agregación no se altera.
          Contratos y esquemas de permisos: Ajenos al problema de concurrencia en la grilla de turnos.
