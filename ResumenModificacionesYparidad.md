Resumen de Modificaciones, Paridad y Plan de Pruebas — TI-3188
1. Modificaciones Realizadas
1.1 Archivos existentes modificados
app/Clinica/Logicas/Turnos/AsignarTurno.php

Se envolvió la validación de cupo y el guardado del turno en una transacción (DB::transaction(..., 3)) con bloqueo pesimista (lockForUpdate) sobre las filas de trabajadores y puestos en orden fijo (bloquearTrabajadorYPuesto):

diff
--- a/app/Clinica/Logicas/Turnos/AsignarTurno.php
+++ b/app/Clinica/Logicas/Turnos/AsignarTurno.php
@@ -12,6 +12,7 @@
 use App\Models\Puesto;
 use App\Models\Trabajador;
 use App\Models\Turno;
+use Illuminate\Support\Facades\DB;
 use Log;
 use Throwable;
 
@@ -54,24 +55,32 @@ public static function procesar($datos)
                 return self::respuesta(409, 'El periodo ya fue exportado a nomina');
             }
 
-            if (Configuracion::configHabilitada('validar_cupo')) {
-                $resultado = self::validarCupo($puesto->id, $fecha, $bloque, $trabajador->id);
-                if ($resultado !== true) {
-                    return self::respuesta(409, $resultado);
+            // TI-3188: validar y guardar en la misma transaccion, con el trabajador y el
+            // puesto bloqueados, para que dos asignaciones simultaneas no pasen ambas
+            // la validacion de cupo. Orden de bloqueo fijo (trabajador -> puesto) para
+            // no generar deadlocks; MoverTurno usa el mismo orden.
+            return DB::transaction(function () use ($trabajador, $puesto, $contrato, $fecha, $bloque) {
+                self::bloquearTrabajadorYPuesto($trabajador->id, $puesto->id);
+
+                if (Configuracion::configHabilitada('validar_cupo')) {
+                    $resultado = self::validarCupo($puesto->id, $fecha, $bloque, $trabajador->id);
+                    if ($resultado !== true) {
+                        return self::respuesta(409, $resultado);
+                    }
                 }
-            }
 
-            $turno = new Turno();
-            $turno->trabajador_id = $trabajador->id;
-            $turno->contrato_id = $contrato->id;
-            $turno->puesto_id = $puesto->id;
-            $turno->fecha = $fecha;
-            $turno->bloque = $bloque;
-            $turno->horas = Fechas::horasBloque($bloque);
-            $turno->creado_at = date('Y-m-d H:i:s');
-            $turno->save();
-
-            return ['ok' => true, 'codigo' => 201, 'id' => $turno->id];
+                $turno = new Turno();
+                $turno->trabajador_id = $trabajador->id;
+                $turno->contrato_id = $contrato->id;
+                $turno->puesto_id = $puesto->id;
+                $turno->fecha = $fecha;
+                $turno->bloque = $bloque;
+                $turno->horas = Fechas::horasBloque($bloque);
+                $turno->creado_at = date('Y-m-d H:i:s');
+                $turno->save();
+
+                return ['ok' => true, 'codigo' => 201, 'id' => $turno->id];
+            }, 3);
         } catch (Throwable $e) {
             report($e);
             Log::debug('AsignarTurno', ['datos' => $datos]);
@@ -118,6 +127,21 @@ public static function validarCupo($idPuesto, $fecha, $bloque, $idTrabajador)
         return true;
     }
 
+    public static function bloquearTrabajadorYPuesto($idTrabajador, $idPuesto)
+    {
+        Trabajador::whereKey($idTrabajador)->lockForUpdate()->first();
+        Puesto::whereKey($idPuesto)->lockForUpdate()->first();
+    }
app/Http/Controllers/PlanillaController.php y routes/api.php

Se agregó el endpoint PATCH /api/turnos/{id}/mover que delega en MoverTurno::procesar:

diff
--- a/routes/api.php
+++ b/routes/api.php
@@ -10,6 +10,7 @@
 Route::post('/turnos', [PlanillaController::class, 'asignar']);
 Route::post('/turnos/{id}/cancelar', [PlanillaController::class, 'cancelar']);
+Route::patch('/turnos/{id}/mover', [PlanillaController::class, 'mover']);
diff
--- a/app/Http/Controllers/PlanillaController.php
+++ b/app/Http/Controllers/PlanillaController.php
@@ -120,4 +121,16 @@ public function cancelar($id)
         return response()->json($resultado, $codigo);
     }
+
+    public function mover(Request $request, $id)
+    {
+        $resultado = MoverTurno::procesar((int) $id, $request->only(['fecha', 'bloque', 'puesto_id']));
+        $codigo = $resultado['codigo'];
+        unset($resultado['codigo']);
+        if (! $resultado['ok']) {
+            return response()->json(['error' => $resultado['mensaje']], $codigo);
+        }
+
+        return response()->json($resultado, $codigo);
+    }
 }
database/seeders/PlanillaSeeder.php

Se agregó la configuración turnos_mover_atomico con valor '0' para que exista también tras un make fresh:

diff
--- a/database/seeders/PlanillaSeeder.php
+++ b/database/seeders/PlanillaSeeder.php
@@ -58,6 +58,7 @@ private function configuraciones()
             ['nombre' => 'planilla_mostrar_cancelados', 'valor' => '1', 'descripcion' => 'Muestra los turnos cancelados en la planilla del dia'],
             ['nombre' => 'validarCupo', 'valor' => '0', 'descripcion' => null],
+            ['nombre' => 'turnos_mover_atomico', 'valor' => '0', 'descripcion' => 'Habilita PATCH /api/turnos/{id}/mover (mover turno atomico)'],
         ];
1.2 Archivos nuevos creados
Archivo	Propósito
app/Clinica/Logicas/Turnos/MoverTurno.php	Lógica transaccional y atómica de mover turno (verifica turnos_mover_atomico, bloquea trabajador + puesto destino + turno origen, valida contrato/cupo/exportación, cancela origen, crea nuevo turno y audita moverTurno).
database/migrations/2026_11_04_000001_insert_config_turnos_mover_atomico.php	Migración que inserta turnos_mover_atomico = 0 en configurations si no existe.
tests/Feature/AsignarTurnoConcurrenciaTest.php	Tests de caracterización de la ventana check-then-act en AsignarTurno.
tests/Feature/AsignarTurnoConcurrenciaRealTest.php	Test de concurrencia real con Http::pool contra múltiples workers de php-fpm.
tests/Feature/MoverTurnoTest.php	Suite de tests para PATCH /api/turnos/{id}/mover (flag apagada/encendida, saldos, cupo lleno, validaciones).
tests/Feature/ParidadConfigApagadaTest.php	Tests de paridad con la flag apagada (snapshots de reporte de dotación, nómina, respuestas de asignar y auditoría).
tests/concurrencia/simular_carrera_asignar.py	Script en Python con threading.Barrier para disparar peticiones concurrentes contra POST /api/turnos.
ops/MITIGACION.md	Procedimiento SQL de detección, corrección acotada por id, verificación y escalamiento para Operaciones.
2. ¿Por qué no afectan a ningún programa que ya está funcionando?

Con la configuración turnos_mover_atomico apagada (0, su valor por defecto), el sistema se comporta exactamente igual que en main para todas las operaciones normales:

Consultas, reportes y nómina intactos (GET /api/planilla/*, /api/dotacion, /api/dotacion/csv, GenerarArchivoNomina):

No se modificó TurnosActivosPorBloque, CalcularSaldo, GenerarArchivoNomina, ReporteDotacionController ni CancelarTurno.

Asignación de turnos (POST /api/turnos en AsignarTurno::procesar):

Devuelve los mismos códigos HTTP (201, 409, 422) y el mismo formato JSON.
Cuando rechaza por cupo lleno (409), el closure de DB::transaction retorna sin lanzar excepción, por lo que hace COMMIT del registro en auditoria (validarCupo), igual que antes.
El observer TurnoObserver::saved corre en la misma conexión, por lo que ve el turno recién insertado y recalcula el saldo igual que antes.

Nuevo endpoint (PATCH /api/turnos/{id}/mover) aislado:

Es una ruta nueva que ningún proceso existente invoca hoy, y mientras turnos_mover_atomico = 0 responde 404 inmediatamente sin tocar la base de datos.
Únicos dos cambios bajo el capó (intencionales)
Peticiones simultáneas al mismo puesto/trabajador: Antes podían entrar dos a la vez y superar el cupo; ahora MySQL las serializa durante unos milisegundos gracias a lockForUpdate(), la primera gana (201) y la segunda recibe 409 Cupo completo para el bloque.
Fallo inesperado a mitad del guardado: Antes, si fallaba el observer después de $turno->save(), el turno quedaba guardado a medias sin saldo recalculado; ahora la transacción hace rollback completo para no dejar datos inconsistentes. La respuesta del catch sigue siendo la misma.
3. Plan de Pruebas y Tabla de Verdad

El detalle paso a paso del plan de pruebas (tabla de verdad, comandos make test, script de Python, curl manuales y validación SQL) se encuentra en el artefacto plan_de_pruebas_qa.md.
