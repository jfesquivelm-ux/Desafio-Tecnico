PROTOCOLO DE SEGURIDAD OPERACIONAL

Regla de Oro: Queda estrictamente prohibido ejecutar DELETE masivos o sentencias UPDATE compuestas con JOINs de múltiples tablas. 
Todo ajuste operacional debe realizarse por ID unívoco bajo transacción.

4.1 Consultas de Detección

A. Puestos con Sobrecupo (Turnos Duplicados):

  SELECT t.fecha, t.bloque, t.puesto_id, p.nombre AS puesto_nombre,
       COUNT(t.id) AS total_asignados, c.cupo
  FROM turnos t
  JOIN puestos p ON p.id = t.puesto_id
  JOIN cupos c ON c.puesto_id = t.puesto_id AND c.bloque = t.bloque
  WHERE t.estado = 'asignado'
  GROUP BY t.fecha, t.bloque, t.puesto_id, p.nombre, c.cupo
  HAVING COUNT(t.id) > c.cupo;

B. Trabajadores con Descuadre de Saldo Semanal:

SELECT tr.id AS trabajador_id, CONCAT(tr.nombre, ' ', tr.apellido) AS trabajador,
       SUM(CASE WHEN t.bloque IN ('M','T') THEN 8 WHEN t.bloque = 'N' THEN 10 ELSE 0 END) AS horas,
       ct.horas_semanales,
       (SUM(CASE WHEN t.bloque IN ('M','T') THEN 8 WHEN t.bloque = 'N' THEN 10 ELSE 0 END) - ct.horas_semanales) AS saldo
  FROM trabajadores tr
  JOIN contratos ct ON ct.trabajador_id = tr.id AND ct.activo = true
  LEFT JOIN turnos t ON t.trabajador_id = tr.id AND t.fecha BETWEEN '2026-10-12' AND '2026-10-18' AND t.estado = 'asignado'
  GROUP BY tr.id, tr.nombre, tr.apellido, ct.horas_semanales
  HAVING saldo < 0;

4.2 Procedimiento de Corrección Acotada

  1. Para cancelar turno sobrante identificado: 

     UPDATE turnos SET estado = 'cancelado', updated_at = NOW() WHERE id = :turno_id AND estado = 'asignado';

  3. Para reactivar turno desaparecido: 

     UPDATE turnos SET estado = 'asignado', updated_at = NOW() WHERE id = :turno_id AND estado = 'cancelado';

4.3 Criterios de Escalamiento
 
   • La sentencia UPDATE arroja 0 o más de 1 fila afectada.

   • El caso involucra días pertenecientes a un cierre de nómina ya consolidado o exportado a finanzas.

   • El saldo sigue descuadrado debido a múltiples contratos vigentes o licencias no computadas.
   
5. Guía de QA para Pull Request.

   Tabla de Verdad del Feature Flag:
   
        COMPORTAMIENTO / FLUJO            FLAG OFF (turnos_mover_atomico=0)  FLAG ON (turnos_mover_atomico=1)
   
      Control de Cupo (Asignación)          Estricto (Bloqueo pesimista)      Estricto (Bloqueo pesimista)
   
      Endpoint PATCH /api/turnos/{id}/mover 501 Not Implemented / Flujo       Legacy 200 OK Atómico / 409 Rollback
   
      Cálculo de Nómina y Reporte           Paridad Idéntica 100%             Paridad Idéntica 100%
   
      Riesgo de Pérdida de Turnos           Mitigado por procedimiento Ops    Eliminado a nivel de API


   


