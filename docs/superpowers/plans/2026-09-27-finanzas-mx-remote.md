# Finanzas MX Remote Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Añadir el esquema Supabase con RLS y sincronización fiable de los cambios locales.

**Architecture:** Las migraciones versionadas definen las tablas y políticas. La app procesa `pending_changes` en orden, aplica la respuesta del servidor y conserva localmente todo cambio que entre en conflicto.

**Tech Stack:** Supabase CLI, PostgreSQL, Edge Functions (Deno), Supabase Flutter y Drift.

**Spec:** `docs/superpowers/specs/2026-09-27-finanzas-mx-design.md`

## Global Constraints

- RLS en todas las tablas y políticas de propiedad con `TO authenticated`, `USING` y `WITH CHECK`.
- No exponer clave administrativa, ni en Flutter ni en Git.
- Cada tabla incluye UUID, `user_id`, creación, actualización y `deleted_at`.
- Los fallos de red y tombstones pendientes no pueden perder información.

## Review Focus

- Una sesión anónima no puede acceder a filas del esquema público.
- Un usuario no puede reasignar `user_id` al actualizar una fila.
- Las tablas nuevas deben estar explícitamente expuestas al Data API con RLS habilitado.
- Un conflicto conserva el cambio local pendiente y comunica el aviso.
- La Edge Function solo puede borrar al usuario del JWT validado.

---

### Task 1: Inicializar Supabase y crear la migración segura

**Files:**
- Create: `supabase/config.toml`, a CLI-generated migration in `supabase/migrations/`, `supabase/tests/rls_finance.sql`

**Interfaces:**
- Produces: tablas remotas de la especificación y políticas CRUD por propiedad.

- [ ] **Step 1: Descubrir CLI y crear el esqueleto**

Run: `supabase --help`, `supabase init`, `supabase migration new finance_schema`

Expected: la migración recibe un nombre generado por CLI.

- [ ] **Step 2: Escribir prueba RLS con dos usuarios y usuario anónimo**

Comprobar que el segundo usuario no puede seleccionar, actualizar ni borrar una fila del primero; comprobar que anónimo no puede leer.

- [ ] **Step 3: Implementar tablas, índices, triggers y RLS**

Crear las siete tablas, `updated_at` administrado por trigger, índices por `(user_id, updated_at)` y cuatro políticas CRUD de propiedad por tabla. Otorgar acceso Data API a `authenticated` solo tras habilitar RLS.

- [ ] **Step 4: Aplicar y verificar localmente**

Run: `supabase db reset && supabase test db`

Expected: pruebas RLS PASS.

- [ ] **Step 5: Commit**

```bash
git add supabase
git commit -m "feat: add secured supabase schema"
```

### Task 2: Crear `delete-account` con validación de JWT

**Files:**
- Create: `supabase/functions/delete-account/index.ts`, `supabase/functions/delete-account/index_test.ts`

**Interfaces:**
- Produces: `POST /functions/v1/delete-account` que elimina únicamente al sujeto autenticado.

- [ ] **Step 1: Escribir pruebas de JWT ausente, JWT inválido y usuario ajeno**

La prueba espera `401` sin JWT y confirma que la eliminación de un usuario no toca las filas de otro.

- [ ] **Step 2: Ejecutar la prueba para confirmar el fallo**

Run: `deno test supabase/functions/delete-account/index_test.ts`

Expected: FAIL porque la función no existe.

- [ ] **Step 3: Implementar función mínima**

Obtener el usuario desde el JWT del encabezado, rechazar si no existe y llamar la API administrativa solo con secreto de entorno. Revocar sesión antes de eliminar el usuario.

- [ ] **Step 4: Ejecutar prueba y asesor de seguridad**

Run: `deno test supabase/functions/delete-account/index_test.ts && supabase db advisors`

Expected: PASS y sin hallazgos críticos.

- [ ] **Step 5: Commit**

```bash
git add supabase/functions/delete-account
git commit -m "feat: add secure account deletion"
```

### Task 3: Implementar `SyncService.syncNow()`

**Files:**
- Create: `lib/features/sync/sync_service.dart`, `test/features/sync/sync_service_test.dart`
- Modify: `lib/features/finance/data/finance_repository.dart`, `lib/core/database/app_database.dart`

**Interfaces:**
- Consumes: `Future<SyncResult> syncNow()`, outbox Drift y `SupabaseClient`.
- Produces: `SyncResult(sent, received, conflicts, failures)`.

- [ ] **Step 1: Escribir pruebas de alta offline, tombstone, reintento y conflicto**

Simular un cliente remoto: error de red deja cola intacta; tombstone remoto desaparece de lecturas; conflicto deja una entrada pendiente y un aviso.

- [ ] **Step 2: Ejecutar las pruebas para confirmar el fallo**

Run: `flutter test test/features/sync/sync_service_test.dart`

Expected: FAIL porque `SyncService` no existe.

- [ ] **Step 3: Implementar envío ordenado y descarga incremental**

Procesar outbox por fecha, usar `updated_at` del servidor para decidir conflicto y actualizar Drift en transacciones. Borrar una entrada de outbox solo después de confirmación remota.

- [ ] **Step 4: Ejecutar pruebas y análisis**

Run: `flutter test test/features/sync/sync_service_test.dart && flutter analyze`

Expected: PASS.

- [ ] **Step 5: Conectar disparadores de sincronización**

Sincronizar al iniciar sesión, recuperar conectividad y al pulsar acción manual; mostrar conflictos sin bloquear la consulta local.

- [ ] **Step 6: Commit**

```bash
git add lib/features/sync lib/features/finance lib/core/database test/features/sync
git commit -m "feat: sync local finance data"
```
