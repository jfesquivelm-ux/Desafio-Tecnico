Plan de Pruebas y Guía de QA — TI-3188

Este plan cubre la verificación completa de lo implementado (fix de concurrencia, configuración turnos_mover_atomico, endpoint PATCH /api/turnos/{id}/mover, paridad y mitigación SQL) y está listo para usarse también como Guía de QA en el cuerpo del Pull Request (Entregable 5 de DESAFIO.md).

1. Tabla de Verdad de Configuraciones

La garantía de integridad de cupo (transacción + SELECT ... FOR UPDATE sobre trabajador y puesto en AsignarTurno) aplica siempre que validar_cupo = 1. El comportamiento nuevo de mover turno está controlado por turnos_mover_atomico.

Escenario / Acción	turnos_mover_atomico = 0 (Default)	turnos_mover_atomico = 1
Existencia de la flag tras make migrate o make fresh	Existe en configurations con valor = '0'	Existe en configurations (se activa con UPDATE)
POST /api/turnos secuencial (cupo libre / cupo lleno / ya tiene turno)	201 / 409 / 409 (idéntico a main)	201 / 409 / 409 (idéntico a main)
POST /api/turnos concurrente (N peticiones simultáneas a cupo = 1)	1 gana (201), N-1 reciben 409. Dotación activa = 1	1 gana (201), N-1 reciben 409. Dotación activa = 1
PATCH /api/turnos/{id}/mover (destino con cupo libre)	404 {"error": "No encontrado"} (no modifica el turno ni audita)	200 {"ok": true, "id": <nuevo>, "id_anterior": <id>}. Cancela origen, crea nuevo, recalcula saldos de ambas semanas y registra auditoría moverTurno
PATCH /api/turnos/{id}/mover (destino con cupo lleno)	404 (sin cambios)	409 {"error": "Cupo completo para el bloque"}. Rollback: el turno origen sigue activo y su saldo no baja
PATCH /api/turnos/{id}/mover (turno origen cancelado o exportado)	404 (sin cambios)	409 (El turno esta cancelado / El turno ya fue exportado a nomina)
PATCH /api/turnos/{id}/mover (datos inválidos / mismo slot)	404 (sin cambios)	422 con mensaje de validación; turno origen intacto
Reporte de dotación (GET /api/dotacion y /api/dotacion/csv)	Salida y formato idénticos a main	Se invalida caché automáticamente al mover y refleja el nuevo puesto/fecha/bloque
Archivo de nómina (GenerarArchivoNomina::generar)	Contenido CSV y hash SHA-256 idénticos a main	Suma las horas del turno nuevo activo e ignora el origen cancelado
Auditoría (auditoria)	Registra validarCupo con el mismo JSON que main	Además registra moverTurno al mover un turno exitosamente
2. Preparación del Entorno y Captura de Línea Base (Paridad)

IMPORTANT

La primera vez que corre ParidadConfigApagadaTest.php, si no existen los archivos en tests/fixtures/paridad/, los genera y marca esos 3 tests como I (Incomplete). En la siguiente ejecución compara contra esos archivos y queda en verde (OK).

bash
# 1. Levantar contenedores, migrar y sembrar datos desde cero
make go
make fresh

Verificar que la configuración quedó creada y apagada por defecto:

bash
docker compose exec -T db mysql -uplanilla -pplanilla planilla \
  -e "SELECT nombre, valor, descripcion FROM configurations WHERE nombre = 'turnos_mover_atomico';"
Resultado esperado: 1 fila con nombre = turnos_mover_atomico y valor = 0.
3. Pruebas Automatizadas (PHPUnit)

Ejecutar la suite completa:

bash
make test

(Si en la primera corrida ParidadConfigApagadaTest grabó los snapshots y marcó 3 tests como Incomplete, vuelve a correr make test inmediatamente para verificar que pasen en verde).

Qué valida cada archivo de test
PlanillaDiaTest.php
Smoke test original del repositorio (GET /api/planilla/dia?fecha=2026-10-14 responde 200).
AsignarTurnoConcurrenciaTest.php (Caracterización del mecanismo)
Demuestra que secuencialmente el cupo se respeta, pero si otra operación se intercala entre validarCupo() y el INSERT sin bloqueo entre conexiones distintas, ambas validaciones aprueban.
AsignarTurnoConcurrenciaRealTest.php (Prueba real del fix con workers paralelos)
Dispara 6 peticiones POST /api/turnos simultáneas con Http::pool contra http://web/api/turnos sobre un puesto de cupo 1.
Resultado esperado: códigos ordenados [201, 409, 409, 409, 409, 409] y exactamente 1 turno activo en base de datos.
ParidadConfigApagadaTest.php (Demostración de paridad con flag en 0)
Compara byte por byte el JSON y CSV de /api/dotacion (2026-09-01 a 2026-10-31) y los CSV de nómina (2026-09 y 2026-10).
Comprueba los códigos y mensajes exactos (201, 409, 422) de POST /api/turnos, el payload de auditoria (validarCupo) y que PATCH /api/turnos/{id}/mover devuelva 404.
MoverTurnoTest.php (Cobertura completa del endpoint nuevo)
Flag apagada (404), movimiento exitoso entre semanas (cancela origen, crea nuevo, actualiza saldos en ambas semanas y registra auditoria), cupo lleno en destino (409 sin perder el turno origen), cambio de puesto, turno ya cancelado (409), periodo exportado (409), turno inexistente (404) y 5 casos de datos inválidos (422).
4. Prueba de Estrés / Concurrencia con Script Python

Con el stack arriba (http://localhost:38081), correr simular_carrera_asignar.py:

Caso A: 5 coordinadoras asignan distintos trabajadores al mismo puesto/fecha/bloque (cupo = 1)
bash
python3 tests/concurrencia/simular_carrera_asignar.py --rondas 10 --trabajadores 1,3,5,6,7
Resultado esperado (con el fix):
En cada ronda: respuestas={201: 1, 409: 4} activos=1 -> ok.
Resumen final: Rondas con cupo superado: 0/10.
Caso B: Doble clic / reintento simultáneo del mismo trabajador en el mismo bloque
bash
python3 tests/concurrencia/simular_carrera_asignar.py --mismo-trabajador 1 --paralelas 5 --rondas 5
Resultado esperado (con el fix):
En cada ronda: respuestas={201: 1, 409: 4} activos=1 -> ok.
Resumen final: Rondas con cupo superado: 0/5.
5. Pruebas Manuales por API (curl) con Datos Concretos
5.1 Verificar que con turnos_mover_atomico = 0 el endpoint mover está deshabilitado

Buscar un turno activo de Ana Pérez (trabajador_id = 1) en octubre:

bash
curl -s "http://localhost:38081/api/planilla/dia?fecha=2026-10-15"

(Anotar el id del turno de Ana Pérez en URG Box 1, bloque M, por ejemplo ID_TURNO).

Intentar moverlo con la configuración apagada:

bash
curl -i -X PATCH "http://localhost:38081/api/turnos/ID_TURNO/mover" \
  -H "Content-Type: application/json" \
  -d '{"fecha":"2026-10-17","bloque":"M"}'
Esperado: HTTP/1.1 404 Not Found con {"error":"No encontrado"}. El turno del 2026-10-15 sigue activo (cancelado: false).
5.2 Activar turnos_mover_atomico = 1 y probar mover

Encender la configuración:

bash
docker compose exec -T db mysql -uplanilla -pplanilla planilla \
  -e "UPDATE configurations SET valor = '1' WHERE nombre = 'turnos_mover_atomico';"

Caso de rechazo atómico (cupo lleno → el turno NO desaparece): Intentar mover el turno de Ana Pérez (2026-10-15, M) al 2026-10-16 bloque N en URG Box 1 (puesto_id = 1), donde Felipe Ortiz (trabajador_id = 6) ya ocupa el cupo de 1:

bash
curl -i -X PATCH "http://localhost:38081/api/turnos/ID_TURNO/mover" \
  -H "Content-Type: application/json" \
  -d '{"fecha":"2026-10-16","bloque":"N","puesto_id":1}'
Esperado: HTTP/1.1 409 Conflict con {"error":"Cupo completo para el bloque"}.
Consultar GET /api/planilla/dia?fecha=2026-10-15: el turno original de Ana Pérez sigue activo y su saldo_semana no bajó.

Caso de éxito (mover a un día libre): Mover ese mismo turno al sábado 2026-10-17, bloque M (cupo libre):

bash
curl -i -X PATCH "http://localhost:38081/api/turnos/ID_TURNO/mover" \
  -H "Content-Type: application/json" \
  -d '{"fecha":"2026-10-17","bloque":"M"}'
Esperado: HTTP/1.1 200 OK con {"ok":true,"id":<ID_NUEVO>,"id_anterior":<ID_TURNO>}.
En GET /api/planilla/dia?fecha=2026-10-15, ID_TURNO aparece con "cancelado": true.
En GET /api/planilla/dia?fecha=2026-10-17, aparece <ID_NUEVO> activo con "saldo_semana" sin alteración negativa.

Volver a dejar la configuración apagada:

bash
docker compose exec -T db mysql -uplanilla -pplanilla planilla \
  -e "UPDATE configurations SET valor = '0' WHERE nombre = 'turnos_mover_atomico';"
6. Verificación de las Consultas SQL de ops/MITIGACION.md

Sobre la base recién sembrada (make fresh), ejecutar las consultas de detección de ops/MITIGACION.md para comprobar que detectan exactamente las anomalías plantadas en el seeder:

bash
# Consulta 1.1: Puestos sobre su cupo
docker compose exec -T db mysql -uplanilla -pplanilla planilla -e "
SELECT t.puesto_id, p.nombre AS puesto, t.fecha, t.bloque,
       COUNT(*) AS dotacion,
       COALESCE((SELECT c.cupo FROM cupos c
                 WHERE c.puesto_id = t.puesto_id AND c.bloque = t.bloque
                   AND c.vigente_desde <= t.fecha
                   AND (c.vigente_hasta IS NULL OR c.vigente_hasta >= t.fecha)
                 ORDER BY c.vigente_desde DESC LIMIT 1), 1) AS cupo,
       GROUP_CONCAT(t.id ORDER BY t.creado_at, t.id) AS turnos_ids
FROM turnos t
JOIN contratos k ON k.id = t.contrato_id
                AND k.fecha_inicio <= t.fecha
                AND (k.fecha_fin IS NULL OR k.fecha_fin >= t.fecha)
JOIN puestos p ON p.id = t.puesto_id
WHERE t.cancelado_at IS NULL
GROUP BY t.puesto_id, p.nombre, t.fecha, t.bloque
HAVING dotacion > cupo
ORDER BY t.fecha, t.puesto_id, t.bloque;"
Resultado esperado en Consulta 1.1:
2026-09-22 | UCI Cama 3 (puesto_id=2) | T | dotacion=2, cupo=1 → Caso en periodo 2026-09 (EXPORTADO y sucio=1), que la guía indica no tocar y escalar.
2026-10-14 | URG Box 1 (puesto_id=1) | M | dotacion=2, cupo=1 → El caso del ticket (Ana Pérez + Gabriela Núñez).
(Nota: el caso del 2026-10-08 en URG Triage no debe aparecer como duplicado activo porque el contrato de Diego Rojas venció el 2026-09-30).
