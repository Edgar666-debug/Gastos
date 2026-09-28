# Finanzas MX Features and Beta Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Completar CSV, presupuestos, inversiones, panel, controles de cuenta y validación beta Android.

**Architecture:** Estas funciones usan los repositorios y el outbox ya existentes. Todo cálculo parte de datos locales; inversión usa precio manual y los duplicados CSV se presentan al usuario sin eliminarlos.

**Tech Stack:** Flutter/Dart, paquete `csv`, selector de archivos, Drift, Supabase Flutter e integración Flutter.

**Spec:** `docs/superpowers/specs/2026-09-27-finanzas-mx-design.md`

## Global Constraints

- CSV acepta únicamente fechas `yyyy-MM-dd` y `dd/MM/yyyy`; fechas ambiguas o filas no interpretables se rechazan.
- Duplicados CSV permanecen visibles para decisión humana.
- No hay cotizaciones en vivo: el precio de inversión es manual.
- La eliminación de cuenta exige confirmación explícita y no puede afectar a otro usuario.

## Review Focus

- Campos CSV entrecomillados con separador interno deben conservarse.
- Símbolos y miles en importes CSV deben producir centavos correctos.
- Venta parcial y comisión deben conservar coste promedio y posición correcta.
- Una posición en cero no debe producir división por cero ni valor fantasma.
- Un presupuesto excedido debe exponer `spent`, `available` y estado excedido correctos.

---

### Task 1: Importar CSV con vista previa y duplicados visibles

**Files:**
- Create: `lib/features/csv/csv_import_service.dart`, `lib/features/csv/csv_import_page.dart`, `test/features/csv/csv_import_service_test.dart`
- Modify: `lib/features/finance/data/finance_repository.dart`

**Interfaces:**
- Produces: `CsvPreview`, `CsvRowIssue`, `CsvDuplicate`, `CsvImportService.preview()` y `commit()`.

- [ ] **Step 1: Escribir pruebas de CSV entrecomillado, formato MXN, fila inválida y duplicado**

El duplicado se detecta por cuenta, fecha, centavos y descripción normalizada, pero `commit()` requiere selección humana para incluirlo o excluirlo.

- [ ] **Step 2: Ejecutar prueba para confirmar el fallo**

Run: `flutter test test/features/csv/csv_import_service_test.dart`

Expected: FAIL porque el servicio no existe.

- [ ] **Step 3: Implementar detección de delimitador, mapeo y preview**

Usar el parser CSV instalado y `Money.parse`; aceptar solo dos formatos de fecha y adjuntar `CsvRowIssue` por cada fila rechazada.

- [ ] **Step 4: Implementar página de selección, mapeo y confirmación**

La pantalla presenta cabeceras, asignación de columnas, errores y duplicados antes de guardar en transacción.

- [ ] **Step 5: Ejecutar pruebas y commit**

Run: `flutter test test/features/csv/csv_import_service_test.dart`

```bash
git add lib/features/csv lib/features/finance test/features/csv
git commit -m "feat: import CSV"
```

### Task 2: Añadir presupuestos mensuales

**Files:**
- Create: `lib/features/budgets/budget_repository.dart`, `lib/features/budgets/budgets_page.dart`, `test/features/budgets/budget_repository_test.dart`

**Interfaces:**
- Produces: `BudgetStatus(spent, available, exceeded)` y `watchMonthlyBudgetStatus(DateTime month)`.

- [ ] **Step 1: Escribir prueba de presupuesto excedido**

Usar gastos locales de una categoría y verificar centavos exactos para gastado, disponible y excedido.

- [ ] **Step 2: Implementar repositorio y página reactiva**

Obtener gastos del mes desde Drift y conservar presupuesto como entidad sincronizable.

- [ ] **Step 3: Ejecutar pruebas y commit**

Run: `flutter test test/features/budgets/budget_repository_test.dart`

```bash
git add lib/features/budgets test/features/budgets
git commit -m "feat: manage monthly budgets"
```

### Task 3: Añadir inversiones y panel local

**Files:**
- Create: `lib/features/investments/investment_repository.dart`, `lib/features/investments/investments_page.dart`, `lib/features/dashboard/dashboard_page.dart`, `test/features/investments/investment_repository_test.dart`, `test/features/dashboard/dashboard_test.dart`

**Interfaces:**
- Produces: `InvestmentPosition(quantity, averageCost, marketValue, profitLoss)` y `DashboardSummary`.

- [ ] **Step 1: Escribir pruebas de compra, venta parcial, comisión, posición cero y panel**

La posición usa precio manual; el panel suma flujo mensual, categorías, patrimonio e inversiones desde SQLite.

- [ ] **Step 2: Implementar repositorio de inversiones**

Actualizar cantidad y coste promedio por operación; impedir ventas que lleven cantidad por debajo de cero.

- [ ] **Step 3: Implementar las pantallas**

Mostrar posición, precio manual, ganancia/pérdida y resumen del panel sin solicitar datos de red.

- [ ] **Step 4: Ejecutar pruebas y commit**

Run: `flutter test test/features/investments test/features/dashboard`

```bash
git add lib/features/investments lib/features/dashboard test/features/investments test/features/dashboard
git commit -m "feat: add investments dashboard"
```

### Task 4: Exportar datos, eliminar cuenta y preparar beta

**Files:**
- Create: `lib/features/account/export_service.dart`, `lib/features/account/account_page.dart`, `integration_test/app_test.dart`
- Modify: `android/app/build.gradle.kts`, `README.md`

**Interfaces:**
- Consumes: Edge Function `delete-account`.
- Produces: exportación del usuario y flujo de eliminación confirmado.

- [ ] **Step 1: Escribir pruebas de exportación y borrado aislado**

La prueba de borrado confirma explícitamente, invoca la función y verifica que datos de otro usuario siguen existiendo.

- [ ] **Step 2: Implementar exportación y eliminación**

Exportar los datos del usuario autenticado; bloquear eliminación hasta confirmar y limpiar sesión/local tras éxito.

- [ ] **Step 3: Escribir prueba de integración del recorrido beta**

Cubrir acceso, alta offline, sincronización simulada, CSV y cierre de sesión.

- [ ] **Step 4: Ejecutar calidad y empaquetado**

Run: `flutter analyze && flutter test && flutter test integration_test && flutter build apk --debug`

Expected: PASS y APK debug generado.

- [ ] **Step 5: Commit**

```bash
git add lib/features/account integration_test android README.md
git commit -m "test: prepare Android beta"
```
