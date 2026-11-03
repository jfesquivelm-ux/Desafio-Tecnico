Respuestas Resumidas (RESPUESTAS.md).

1. ¿Por qué se duplican los turnos si el código valida antes de guardar?.

    Porque la lectura del cupo y la inserción se ejecutan de forma concurrente sin cerrojos ni serialización en base de datos.
    Múltiples procesos leen cupo disponible simultáneamente y escriben en paralelo.
   
2. ¿En qué condiciones se rompe esa validación y cómo reproducirlo?.

    Se quiebra durante ventanas de alta contención matutina cuando dos coordinadoras interactúan con el mismo puesto/bloque. Se reproduce
    de manera determinista mediante un test automatizado con peticiones concurrentes asíncronas paralelas.
   
3. ¿De dónde sale el saldo negativo? ¿Es el mismo mecanismo?.

    Nace de la falta de una operación atómica: el usuario cancela en el día A y luego intenta asignar en el día B; si el día B rechaza la inserción
    por cupo, el turno original ya fue destruido, restando 8 o 10 horas semanales.
   
4. ¿Cómo debería funcionar mover como acción de verdad?.

   Como una transacción de base de datos única y atómica: reserva el cupo en destino mediante bloqueo pesimista y actualiza el turno; si el destino está lleno,
   revierte todo (rollback) manteniendo intacto el día origen.
   
5. ¿Qué pasa con el saldo, reporte y nómina al mover un turno?.

   Dentro de la misma semana el saldo neto no se altera (variación cero). El reporte de dotación actualiza el día origen y destino instantáneamente sin
   sobrecupos, y la nómina mensual no cambia a menos que cruce fin de mes.
   
6. ¿Qué entregas primero para que el dolor pare ya?.

   Primero el bloqueo de concurrencia en la validación de cupos para frenar nuevos duplicados, junto con la guía de mitigación SQL para Operaciones y el 
   endpoint PATCH atómico detrás de la configuración.
   
7. ¿Qué le entregas a operaciones mientras tanto?.

    Un manual técnico operacional (ops/MITIGACION.md) con consultas SQL de diagnóstico, updates puntuales por ID para subsanar turnos sin borrar datos y
    criterios de escalamiento.
    
8. ¿Cómo llegaste a la causa raíz?.

    Inspeccionando el flujo de asignación: comprobé que el SELECT COUNT(*) no usaba FOR UPDATE ni nivel de aislamiento Serializable, coincidiendo con los
    reportes de concurrencia matutina.
    
9. ¿Qué alternativas descartaste y por qué?.

    Descarté restricciones UNIQUE en base de datos por puestos con cupo mayor a uno y datos históricos duplicados, así como locks en Redis para evitar
    infraestructura adicional a días del cierre de nómina.

10. ¿Cómo usaste la IA en el proceso?.
    Se utilizó para acelerar la redacción de plantillas Markdown, contrastar sintaxis de locking pesimista y generar casos de prueba de concurrencia.
    Se rechazaron propuestas de validación exclusiva en cliente o limpiezas automáticas destructivas.
    
11. ¿Qué harías distinto con más tiempo?.
    Implementaría WebSockets para actualizar en tiempo real la planilla visual de las coordinadoras, una cola asíncrona de asignación y un dashboard
    interno de autocorrección para Operaciones.
