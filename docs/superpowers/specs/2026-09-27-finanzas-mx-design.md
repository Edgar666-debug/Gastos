# Finanzas MX — diseño del MVP

## Objetivo

Aplicación Android personal para registrar finanzas en MXN, operar sin conexión e importar CSV. Supabase guarda un respaldo privado por usuario. El MVP no incluye banca, cotizaciones en vivo, multimoneda ni colaboración.

## Plataforma y configuración

Flutter orientado primero a Android, con `minSdk` 23, paquete `mx.finanzas.app`, español (`es-MX`) y nombre visible **Finanzas MX**. La URL y la clave publicable de Supabase se reciben solo por `--dart-define`; el repositorio no guarda secretos. La autenticación usa confirmación de correo, recuperación con `finanzasmx://auth-callback` y PKCE.

## Arquitectura

La interfaz consume repositorios. Drift/SQLite es la fuente local de lectura y Supabase PostgreSQL el respaldo remoto. Las entidades son inmutables y el dinero se representa como `Money(minorUnits)` en centavos enteros.

Al guardar, editar o borrar una entidad, el repositorio actualiza SQLite y agrega un cambio a `pending_changes` dentro de la misma transacción. Los borrados son lógicos y su tombstone se conserva hasta sincronizar.

`SyncService.syncNow()` envía la cola en orden, descarga los cambios del usuario y actualiza SQLite. Un fallo de red no elimina cambios locales. Si el servidor tiene una versión más nueva, se conserva el cambio local pendiente y se muestra un aviso; nunca se descarta silenciosamente.

## Datos y seguridad

Las tablas remotas son `accounts`, `categories`, `transactions`, `budgets`, `investments`, `investment_movements` y `csv_imports`. Cada fila tiene UUID, `user_id`, marcas de creación/actualización y `deleted_at`.

RLS se habilita en cada tabla. Las políticas se aplican a `authenticated` y usan `auth.uid() = user_id` tanto en `USING` como en `WITH CHECK` cuando corresponde. La Edge Function `delete-account` valida el JWT y elimina solo al usuario que la invoca; su clave administrativa es un secreto del entorno, nunca del cliente.

## Experiencia de usuario

Las rutas separan acceso, recuperación y aplicación. Una sesión inválida va a acceso; el cierre de sesión limpia SQLite. El primer corte funcional incluye panel, cuentas, categorías y movimientos manuales. Un formulario acepta importes positivos, con tipo determinado por la categoría, y rechaza cero, negativos o más de dos decimales antes de persistir.

Después se incorporan CSV, presupuestos e inversiones sin modificar este flujo. El CSV detecta delimitador, muestra el mapeo y una vista previa. Solo acepta fechas `yyyy-MM-dd` y `dd/MM/yyyy`; filas no interpretables se rechazan y los duplicados se conservan visibles para decisión humana.

## Verificación

Cada bloque aporta pruebas unitarias o de widgets de sus reglas. El backend prueba que un usuario autenticado solo puede leer y modificar sus propias filas y que un usuario sin sesión no accede. La sincronización cubre altas sin conexión, tombstones, reintentos y conflictos. La beta añade pruebas de integración para registro, movimiento offline, sincronización, CSV y cierre de sesión.

## Entrega incremental

1. Base Flutter, configuración y formato MXN.
2. Esquema Supabase seguro y modelo local.
3. Autenticación y primer corte offline: cuentas, categorías y movimientos.
4. Sincronización persistente.
5. CSV, presupuestos, inversiones, panel y controles de cuenta.
6. Pruebas de integración y preparación de beta Android.
