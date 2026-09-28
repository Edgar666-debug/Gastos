# Finanzas MX Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar la base Android offline-first de Finanzas MX: configuración, dinero MXN, persistencia local, autenticación y movimientos manuales.

**Architecture:** Flutter consume repositorios que escriben Drift/SQLite. Cada mutación de dominio se registra junto a una entrada de outbox local; la sincronización se incorpora en el siguiente plan. Supabase se inicializa exclusivamente con valores de compilación públicos.

**Tech Stack:** Flutter/Dart, Drift + drift_flutter/SQLite, supabase_flutter, flutter_secure_storage, intl y pruebas Flutter.

**Spec:** `docs/superpowers/specs/2026-09-27-finanzas-mx-design.md`

## Global Constraints

- Android primero, `minSdk` 23, paquete `mx.finanzas.app` y nombre **Finanzas MX**.
- Español `es-MX`; MXN fijo; dinero en centavos `int`, nunca `double`.
- URL y clave publicable de Supabase llegan mediante `--dart-define`; ningún secreto va a Git.
- RLS se aplica a todas las tablas remotas; la clave administrativa no llega a Flutter.
- No añadir Docker, backend Node, banca, cotizaciones en vivo ni multimoneda.
- Fijar versiones compatibles en `pubspec.lock` y usar migraciones Supabase versionadas.

## Review Focus

- Un importe `0`, negativo o con más de dos decimales debe fallar antes de tocar SQLite.
- `1,234.56` debe interpretarse como MXN 1,234.56 y no como 1.23456; una coma aislada es decimal.
- Al fallar una escritura, no puede quedar la entidad guardada sin su entrada en `pending_changes`.
- El cierre de sesión debe borrar el archivo SQLite del usuario y no dejar datos visibles.
- Una categoría de ingreso no puede usarse para persistir un gasto, ni la inversa.

---

### Task 1: Crear el proyecto Flutter y su configuración segura

**Files:**
- Create: `pubspec.yaml`, `analysis_options.yaml`, `lib/main.dart`, `lib/app.dart`, `lib/core/config/app_config.dart`, `test/core/config/app_config_test.dart`
- Modify: `android/app/build.gradle.kts`, `android/app/src/main/AndroidManifest.xml`, `.gitignore`

**Interfaces:**
- Produces: `AppConfig.fromEnvironment() -> AppConfig?` y `FinanzasApp(config: AppConfig?)`.

- [ ] **Step 1: Crear el proyecto Flutter con Android y el identificador solicitado**

Run: `flutter create --org mx.finanzas --project-name finanzas_mx .`

Expected: crea el proyecto sin reemplazar `docs/` ni `.git/`.

- [ ] **Step 2: Fijar plataforma y dependencias mínimas**

Configurar `minSdk` 23; añadir Drift, `drift_flutter`, `supabase_flutter`, `flutter_secure_storage`, `intl`, `file_picker`, `csv`, `fl_chart`, y sus herramientas de generación. Ejecutar `flutter pub get` y conservar `pubspec.lock`.

- [ ] **Step 3: Escribir la prueba de configuración ausente y válida**

```dart
expect(AppConfig.fromEnvironment(), isNull);
expect(config.url, 'https://example.supabase.co');
```

- [ ] **Step 4: Implementar `AppConfig.fromEnvironment()`**

Leer `SUPABASE_URL` y `SUPABASE_PUBLISHABLE_KEY` mediante `String.fromEnvironment`; devolver `null` si uno falta. La pantalla de arranque explica cómo proporcionar ambos valores sin mostrarlos.

- [ ] **Step 5: Verificar configuración y análisis**

Run: `flutter test test/core/config/app_config_test.dart && flutter analyze`

Expected: PASS sin advertencias.

- [ ] **Step 6: Configurar enlace profundo y respaldo Android**

Agregar el intent-filter `finanzasmx://auth-callback`, deshabilitar backup de datos sensibles Android y asegurar que `main()` inicialice bindings antes de Supabase.

- [ ] **Step 7: Commit**

```bash
git add pubspec.yaml pubspec.lock analysis_options.yaml android lib test .gitignore
git commit -m "chore: bootstrap flutter app"
```

### Task 2: Crear el núcleo de dinero y formato MXN

**Files:**
- Create: `lib/core/money/money.dart`, `test/core/money/money_test.dart`

**Interfaces:**
- Produces: `Money.parse(String value) -> Money`, `Money.minorUnits`, `Money.formatEsMx()`.

- [ ] **Step 1: Escribir pruebas para cantidades válidas e inválidas**

```dart
expect(Money.parse(r'$1,234.50').minorUnits, 123450);
expect(() => Money.parse('0'), throwsFormatException);
expect(() => Money.parse('1.234'), throwsFormatException);
```

- [ ] **Step 2: Ejecutar la prueba para confirmar el fallo**

Run: `flutter test test/core/money/money_test.dart`

Expected: FAIL porque `Money` no existe.

- [ ] **Step 3: Implementar `Money` inmutable**

Aceptar solo valores positivos con máximo dos decimales; normalizar símbolo y separadores sin usar `double`; formatear con `NumberFormat.currency(locale: 'es_MX', symbol: r'$')` desde centavos.

- [ ] **Step 4: Ejecutar y confirmar las pruebas**

Run: `flutter test test/core/money/money_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/core/money test/core/money
git commit -m "feat: add MXN money value object"
```

### Task 3: Modelar Drift y la cola persistente

**Files:**
- Create: `lib/core/database/app_database.dart`, `lib/core/database/tables.dart`, `test/core/database/app_database_test.dart`
- Generate: `lib/core/database/app_database.g.dart`

**Interfaces:**
- Produces: `AppDatabase`, tablas locales de cuentas, categorías, movimientos, presupuestos, inversiones, movimientos de inversión, importaciones y `pending_changes`.

- [ ] **Step 1: Escribir prueba de migración inicial y transacción atómica**

La prueba abre base temporal, inserta un movimiento y su cambio pendiente en una única transacción; una excepción forzada debe dejar ambas tablas sin filas.

- [ ] **Step 2: Ejecutar la prueba para confirmar el fallo**

Run: `flutter test test/core/database/app_database_test.dart`

Expected: FAIL porque no existe `AppDatabase`.

- [ ] **Step 3: Implementar tablas y `schemaVersion = 1`**

Todas las entidades almacenan UUID, `userId`, `createdAt`, `updatedAt` y `deletedAt`; `pending_changes` almacena `entityType`, `entityId`, operación, carga JSON y fecha. Usar `MigrationStrategy` explícita y generación Drift.

- [ ] **Step 4: Generar código y ejecutar pruebas**

Run: `dart run build_runner build --delete-conflicting-outputs && flutter test test/core/database/app_database_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/core/database test/core/database
git commit -m "feat: add local finance database"
```

### Task 4: Exponer repositorios de cuentas, categorías y movimientos

**Files:**
- Create: `lib/features/finance/domain/entities.dart`, `lib/features/finance/data/finance_repository.dart`, `test/features/finance/finance_repository_test.dart`

**Interfaces:**
- Consumes: `AppDatabase`, `Money`.
- Produces: `watchAccounts()`, `watchTransactions(DateTime month)`, `saveTransaction(Transaction transaction)`, `deleteTransaction(String id)`, `monthlyBalance(String accountId, DateTime month)`.

- [ ] **Step 1: Escribir pruebas de alta, edición, borrado lógico y saldo**

Verificar que `saveTransaction` crea o actualiza exactamente una entrada de outbox y que `deleteTransaction` conserva un tombstone. Incluir categoría/tipo incompatibles como error.

- [ ] **Step 2: Ejecutar la prueba para confirmar el fallo**

Run: `flutter test test/features/finance/finance_repository_test.dart`

Expected: FAIL porque el repositorio no existe.

- [ ] **Step 3: Implementar entidades inmutables y repositorio**

El formulario solo entrega importes positivos. `saveTransaction` guarda entidad y outbox en `AppDatabase.transaction`; `monthlyBalance` suma ingresos y resta gastos de filas no eliminadas.

- [ ] **Step 4: Ejecutar pruebas de repositorio**

Run: `flutter test test/features/finance/finance_repository_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/features/finance test/features/finance
git commit -m "feat: add local finance repositories"
```

### Task 5: Implementar sesión y rutas de autenticación

**Files:**
- Create: `lib/features/auth/data/auth_repository.dart`, `lib/features/auth/presentation/auth_page.dart`, `lib/features/auth/presentation/recovery_page.dart`, `test/features/auth/auth_page_test.dart`
- Modify: `lib/app.dart`, `lib/main.dart`

**Interfaces:**
- Consumes: `AppConfig`, `SupabaseClient`, `AppDatabase`.
- Produces: `AuthRepository.signUp`, `signIn`, `sendRecovery`, `updatePassword`, `signOut`, `authState`.

- [ ] **Step 1: Escribir pruebas de validación y de rutas**

Cubrir correo inválido, contraseña corta, estado de confirmación pendiente y que `signOut` invoca la limpieza de base local.

- [ ] **Step 2: Ejecutar las pruebas para confirmar el fallo**

Run: `flutter test test/features/auth/auth_page_test.dart`

Expected: FAIL porque las páginas y el repositorio no existen.

- [ ] **Step 3: Implementar el repositorio y las rutas**

Usar `signUp`, `signInWithPassword`, `resetPasswordForEmail(redirectTo: 'finanzasmx://auth-callback')`, `updateUser` y `onAuthStateChange`. Persistir secretos solo mediante el almacenamiento configurado por `supabase_flutter` y `flutter_secure_storage`; al cerrar sesión eliminar el archivo SQLite.

- [ ] **Step 4: Ejecutar pruebas de widgets**

Run: `flutter test test/features/auth/auth_page_test.dart`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/features/auth lib/app.dart lib/main.dart test/features/auth
git commit -m "feat: add authenticated app access"
```

### Task 6: Crear pantallas offline de cuentas, categorías y movimientos

**Files:**
- Create: `lib/features/finance/presentation/dashboard_page.dart`, `lib/features/finance/presentation/accounts_page.dart`, `lib/features/finance/presentation/categories_page.dart`, `lib/features/finance/presentation/transaction_form.dart`, `test/features/finance/transaction_form_test.dart`
- Modify: `lib/app.dart`

**Interfaces:**
- Consumes: `FinanceRepository`.
- Produces: formulario de movimiento y pantallas reactivas sin dependencia de red.

- [ ] **Step 1: Escribir prueba de formulario**

Probar que un importe válido crea movimiento, que la categoría decide el tipo y que las entradas inválidas no llaman `saveTransaction`.

- [ ] **Step 2: Ejecutar la prueba para confirmar el fallo**

Run: `flutter test test/features/finance/transaction_form_test.dart`

Expected: FAIL porque no existe el formulario.

- [ ] **Step 3: Implementar las cuatro pantallas mínimas**

Usar `StreamBuilder` sobre los `watch` del repositorio; mostrar saldo por cuenta y movimientos del mes. Edición y borrado llaman al mismo repositorio local.

- [ ] **Step 4: Ejecutar pruebas y análisis completo**

Run: `flutter test && flutter analyze`

Expected: PASS sin advertencias.

- [ ] **Step 5: Commit**

```bash
git add lib/features/finance lib/app.dart test/features/finance
git commit -m "feat: add manual income and expense tracking"
```
