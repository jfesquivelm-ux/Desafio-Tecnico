# Mitigación TI-3188: turnos duplicados y saldos que no cuadran

> **PROTOCOLO DE SEGURIDAD OPERACIONAL — Regla de Oro:** Queda estrictamente prohibido ejecutar `DELETE` masivos o sentencias `UPDATE` compuestas con `JOIN` de múltiples tablas. Todo ajuste operacional debe realizarse por `ID` unívoco bajo transacción (`START TRANSACTION ... COMMIT`).

**Para quién:** operaciones, con SQL básico y acceso de lectura/escritura a MySQL (base `planilla`).
No hace falta conocer el código.
**Cuándo usarlo:** la clínica reporta "dos personas en un puesto donde cabe una" o "el turno
desapareció / el saldo de horas quedó negativo", y el fix todavía no está en producción.

---

## 0. Reglas antes de empezar

1. **Solo `SELECT` y `UPDATE` por `id`.** Nunca `DELETE`, nunca `UPDATE` sin `WHERE id = …`.
2. **Un caso a la vez**, dentro de una transacción (`START TRANSACTION … COMMIT`).
3. **Guarda en el ticket** el resultado de cada consulta de detección **antes** de corregir.
4. **Confirma con la jefatura de la unidad** cuál turno queda. El SQL muestra el problema, pero no
   decide quién trabajó.
5. Si algo no calza con lo que dice este documento, **detente y escala** (sección 4).

### Glosario mínimo

| Término | En la base |
|---|---|
| Turno **activo** | `turnos.cancelado_at IS NULL` **y** la fecha del turno cae dentro del contrato (`contratos.fecha_inicio <= fecha` y `fecha_fin` nula o `>= fecha`). Un turno fuera de su contrato **no cuenta** en la planilla, el reporte ni la nómina. |
| **Cupo** | `cupos.cupo` vigente para puesto + bloque + fecha. Si no hay fila, el sistema asume **1**. El cupo cambia en el tiempo (ej. URG Box 1, mañana: 2 hasta el 10-10-2026 y 1 desde el 11-10-2026). |
| **Bloque** | `M` mañana (8 h), `T` tarde (8 h), `N` noche (10 h). |
| **Periodo exportado** | `periodos_nomina.estado = 'EXPORTADO'` para el mes (`'2026-09'`). Ese archivo ya se envió a remuneraciones. |
| **Saldo** | Tabla `saldos`, una fila por contrato y semana (`semana` = lunes). `saldo = horas_asignadas - horas_contrato`. |

---

## 1. Detectar

### 1.1 Puestos sobre su cupo (el duplicado del ticket)

```sql
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
ORDER BY t.fecha, t.puesto_id, t.bloque;
```

**Cómo leerlo:** cada fila es un puesto/día/bloque con más personas que su cupo. `turnos_ids` está
ordenado del más antiguo al más nuevo según `creado_at`. Por cada fila, mira el detalle con la
consulta 1.3.

### 1.2 Trabajador con dos turnos en el mismo bloque (doble clic / reintento)

```sql
SELECT t.trabajador_id, tr.nombre, t.fecha, t.bloque, COUNT(*) AS turnos,
       GROUP_CONCAT(t.id ORDER BY t.creado_at, t.id) AS turnos_ids
FROM turnos t
JOIN contratos k ON k.id = t.contrato_id
                AND k.fecha_inicio <= t.fecha
                AND (k.fecha_fin IS NULL OR k.fecha_fin >= t.fecha)
JOIN trabajadores tr ON tr.id = t.trabajador_id
WHERE t.cancelado_at IS NULL
GROUP BY t.trabajador_id, tr.nombre, t.fecha, t.bloque
HAVING COUNT(*) > 1;
```

**Cómo leerlo:** la misma persona aparece dos veces en el mismo día y bloque, y se le cuentan
horas dobles. Siempre es un error: nadie hace dos turnos en el mismo bloque.

### 1.3 Detalle de un caso (antes de decidir)

Reemplaza los ids por los de `turnos_ids`:

```sql
SELECT t.id, t.trabajador_id, tr.nombre, t.puesto_id, t.fecha, t.bloque, t.horas,
       t.creado_at, t.cancelado_at, t.exportado_en,
       k.id AS contrato_id, k.centro AS centro_contrato,
       pn.estado AS estado_periodo
FROM turnos t
JOIN trabajadores tr ON tr.id = t.trabajador_id
JOIN contratos k ON k.id = t.contrato_id
LEFT JOIN periodos_nomina pn ON pn.periodo = DATE_FORMAT(t.fecha, '%Y-%m')
WHERE t.id IN (165, 182);
```

### 1.4 Saldos que no cuadran con los turnos

Recalcula el saldo como lo hace el sistema y muestra solo las filas donde lo guardado difiere:

```sql
SELECT * FROM (
  SELECT s.id AS saldo_id, s.contrato_id, k.trabajador_id, s.semana,
         s.horas_asignadas, s.saldo, k.horas_semanales,
         COALESCE((SELECT SUM(t.horas) FROM turnos t
                   WHERE t.contrato_id = s.contrato_id
                     AND t.cancelado_at IS NULL
                     AND t.fecha BETWEEN s.semana AND DATE_ADD(s.semana, INTERVAL 6 DAY)
                     AND t.fecha >= k.fecha_inicio
                     AND (k.fecha_fin IS NULL OR t.fecha <= k.fecha_fin)), 0)
       - COALESCE((SELECT SUM(pe.horas) FROM permisos pe
                   WHERE pe.trabajador_id = k.trabajador_id
                     AND pe.tipo = 'SIN_GOCE'
                     AND pe.fecha BETWEEN s.semana AND DATE_ADD(s.semana, INTERVAL 6 DAY)), 0)
         AS asignadas_esperadas
  FROM saldos s
  JOIN contratos k ON k.id = s.contrato_id
) z
WHERE z.horas_asignadas <> z.asignadas_esperadas;
```

**Cómo leerlo:** `horas_asignadas` es lo guardado y `asignadas_esperadas` lo que debería decir. Si
hay filas, el saldo quedó desactualizado: dos cambios simultáneos se pisaron o alguien corrigió un
turno por SQL y no actualizó el saldo.

### 1.5 Saldos negativos y su posible causa

Un saldo negativo **no siempre es un error**. Esta consulta muestra pistas para clasificarlo:

```sql
SELECT s.id AS saldo_id, k.trabajador_id, tr.nombre, s.contrato_id, s.semana, s.saldo,
       (SELECT COUNT(*) FROM permisos pe
         WHERE pe.trabajador_id = k.trabajador_id AND pe.tipo = 'SIN_GOCE'
           AND pe.fecha BETWEEN s.semana AND DATE_ADD(s.semana, INTERVAL 6 DAY)) AS permisos_sin_goce,
       (k.fecha_inicio > s.semana
         OR (k.fecha_fin IS NOT NULL AND k.fecha_fin < DATE_ADD(s.semana, INTERVAL 6 DAY))) AS contrato_parcial,
       (SELECT GROUP_CONCAT(t.id) FROM turnos t
         WHERE t.contrato_id = s.contrato_id AND t.cancelado_at IS NOT NULL
           AND t.fecha BETWEEN s.semana AND DATE_ADD(s.semana, INTERVAL 6 DAY)) AS turnos_cancelados
FROM saldos s
JOIN contratos k ON k.id = s.contrato_id
JOIN trabajadores tr ON tr.id = k.trabajador_id
WHERE s.saldo < 0
ORDER BY s.semana, tr.nombre;
```

| Pista | Significado | ¿Se corrige? |
|---|---|---|
| `permisos_sin_goce > 0` | El permiso descuenta horas. | **No.** Es correcto. |
| `contrato_parcial = 1` | El contrato empieza o termina a mitad de semana. Hoy el sistema compara contra las horas semanales completas. | **No.** Es una limitación conocida, no es de este ticket. |
| `turnos_cancelados` con ids | Posible **turno desaparecido**: se canceló para "moverlo" y la reasignación nunca se hizo (o falló porque el cupo destino se llenó). | **Solo si la jefatura confirma** (sección 2.3). |

---

## 2. Corregir

### 2.1 Puesto sobre su cupo

1. Con 1.3, confirma que **todos** los turnos del caso tienen `exportado_en` NULL y
   `estado_periodo` distinto de `EXPORTADO`. Si alguno está exportado, **no sigas**: escala.
2. Pregunta a la jefatura **quién debe quedar**. Si no responde, se mantiene el turno creado
   **primero** (`creado_at` menor): el último fue el que entró de más. Si dos turnos tienen el mismo
   `creado_at`, **no elijas tú**: espera la respuesta de la jefatura.
3. Cancela **solo** los turnos que sobran, uno por uno:

```sql
START TRANSACTION;

UPDATE turnos
SET cancelado_at = NOW(), updated_at = NOW()
WHERE id = 182                 -- el turno que sobra
  AND cancelado_at IS NULL
  AND exportado_en IS NULL;
-- Debe decir "1 row affected". Si dice 0 o más de 1: ROLLBACK; y escala.

COMMIT;
```

4. **Actualiza el saldo** de ese trabajador y esa semana (2.4). Al cancelar por SQL, el sistema
   **no** recalcula el saldo solo.

> Se usa `cancelado_at` y no `DELETE` porque así queda el historial y se puede revertir
> (`SET cancelado_at = NULL`). La planilla, el reporte y la nómina ya ignoran los turnos cancelados.

### 2.2 Trabajador con dos turnos en el mismo bloque

Igual que 2.1, pero no hace falta preguntar quién queda: se mantiene el **más antiguo** y se
cancela el otro. Después, 2.4.

### 2.3 Turno desaparecido (saldo negativo por un cancelado)

1. Confirma con la jefatura que el trabajador **sí debía hacer ese turno** en esa fecha y bloque.
   - Si lo querían **en otro día**, no lo arregles por SQL: la coordinadora lo asigna en la
     planilla, que recalcula todo sola.
2. Verifica que el cupo de ese puesto, fecha y bloque **tenga espacio** (1.1 para esa fecha) y que el
   trabajador no tenga otro turno en ese bloque (1.2).
3. Reactívalo: Antes de reactivar un turno cancelado hay que comprobar el cupo, o se vuelve a crear el duplicado.

```sql
START TRANSACTION;

UPDATE turnos
SET cancelado_at = NULL, updated_at = NOW()
WHERE id = 161                 -- ej. turno cancelado de Carla Muñoz el 2026-10-13
  AND cancelado_at IS NOT NULL
  AND exportado_en IS NULL;
-- "1 row affected" o ROLLBACK + escalar.

COMMIT;
```

4. Actualiza el saldo (2.4) y vuelve a correr 1.1 para esa fecha: no debe aparecer.

### 2.4 Saldo desactualizado

Corre la consulta 1.4 para obtener `saldo_id` y `asignadas_esperadas` del contrato modificado y actualiza **esa única fila**:

```sql
START TRANSACTION;

UPDATE saldos
SET horas_asignadas = 32.0,                 -- asignadas_esperadas según 1.4
    saldo           = 32.0 - horas_contrato,
    actualizado_at  = NOW(),
    updated_at      = NOW()
WHERE id = 57;                              -- saldo_id según 1.4
-- "1 row affected" o ROLLBACK + escalar.

COMMIT;
```

Si la fila no existe en 1.4 (por ejemplo, la semana no tiene saldo), **no la crees por SQL**: escala.

### 2.5 Lo que **no** se toca, aunque se parezca

| Caso | Por qué no |
|---|---|
| Turnos con `exportado_en` o de un **periodo exportado** | La nómina ya se envió y remuneraciones concilia contra ese archivo. Cambiarlo es un cambio silencioso en la nómina. **Escalar.** |
| Dos personas en un puesto **antes de un cambio de cupo** (ej. URG Box 1 M hasta el 10-10-2026, cupo 2) | Estaba permitido. La consulta 1.1 ya usa el cupo vigente de cada fecha. |
| Turnos **fuera del contrato** del trabajador | No son activos: no suman en la planilla, el reporte ni la nómina. Si en una tabla "cruda" parecen duplicados, no lo son. **No tocar**; avisar a desarrollo si aparecen nuevos. |
| Saldos negativos por **permiso sin goce** o **cambio de contrato a mitad de semana** | Son correctos según las reglas actuales. |
| Turnos cancelados que la jefatura confirma como **cancelaciones reales** | No son desaparecidos. |
| Turnos de prueba en fechas futuras | Si ves turnos 2027 de trabajadores "Test …", son del equipo de desarrollo. Avisar, no tocar. |

---

## 3. Verificar

**Antes** de corregir (pegar en el ticket):
- Salida de 1.1 / 1.2 para la fecha del caso y salida de 1.3 con los ids.
- Fila de `saldos` del trabajador y la semana (`SELECT * FROM saldos WHERE contrato_id = ? AND semana = ?`).

**Después:**
1. 1.1 y 1.2 para esa fecha **ya no muestran** el caso.
2. 1.4 **no muestra** el contrato ni la semana corregidos.
3. En la app, la planilla de ese día (o `GET /api/planilla/dia?fecha=AAAA-MM-DD`) muestra a la
   persona correcta y el turno sobrante como cancelado.
4. El **reporte de dotación** puede tardar hasta **10 minutos** en reflejar el cambio (tiene caché).
   Si pasado ese tiempo sigue igual, escala.
5. Ningún periodo cambió de estado: `SELECT periodo, estado, sucio FROM periodos_nomina;` debe dar lo
   mismo que antes.

---

## 4. Escalar a desarrollo (detente y avisa)

- El `UPDATE` afecta **0 o más de 1 fila**.
- Algún turno del caso está **exportado** o el periodo está `EXPORTADO`.
- Hay `sucio = 1` en `periodos_nomina` para el mes del caso.
- Dos turnos empatan en `creado_at` y la jefatura no responde.
- Un mismo puesto aparece sobre su cupo **más de una vez por semana**, o hay **más de 3 casos
  nuevos por día**: puede ser otro problema.
- Un saldo no cuadra y **no hay** duplicado, cancelado, permiso ni cambio de contrato que lo explique.
- Crees que hay que **borrar** algo o **crear** filas.

Al escalar, incluye: número de ticket, fecha, puesto y bloque, salidas de 1.1, 1.3 y 1.4, y lo que
ya corregiste (ids y hora).

---

## 5. Casos detectados en la base actual

| Caso | Dónde | Qué hacer |
|---|---|---|
| URG Box 1, M, 14-10-2026: Ana Pérez (`id=165`) + Gabriela Núñez (`id=182`), cupo 1 | 1.1 (`165,182`) | Duplicado real en periodo abierto. **Los dos tienen el mismo `creado_at` (`2026-10-13 09:12:03`)**, así que hay que preguntar a la jefatura antes de cancelar (pista: el contrato de Gabriela es de UCI y Ana Pérez tiene el turno regular en URG Box 1). |
| UCI Cama 3, T, 22-09-2026: Carla Muñoz (`id=87`) + Elena Vidal (`id=179`), cupo 1 | 1.1 (`87,179`) | Periodo **2026-09 exportado** (`sucio = 1`) y el turno `179` se creó después del export. **No tocar: escalar.** |
| URG Triage, T, 08-10-2026: Diego Rojas (`id=180`) + Felipe Ortiz (`id=181`) | (no aparece en 1.1) | El contrato de Diego terminó el 30-09: su turno no es activo. **No tocar.** |
| Carla Muñoz, T, 13-10-2026 cancelado (`id=161`) | 1.5 | Posible desaparecido: confirmar con la jefatura (2.3). |
| Elena Vidal (semana 05-10) y Bruno Soto (semana 05-10) negativos | 1.5 | Permiso sin goce: correcto. |
| Bruno Soto, semana 12-10-2026 | 1.5 | Cambio de contrato el 15-10: correcto según reglas actuales. |

