# DOREVIA POS — Handoff Técnico

Respaldo generado: 2026-07-01
Fuente: github.com/michaaaaaail/Dorevia (rama `main`)
App en producción: https://michaaaaaail.github.io/Dorevia/

---

## 1. Qué es

DOREVIA POS es una PWA de punto de venta e inventario para el negocio de joyería/macramé Dorevia (balines de oro 18k, tamaños 3–6, liso y diamantado). Es un **archivo único autocontenido** (`index.html`, ~1535 líneas: HTML + CSS + JS embebido, sin build step, sin frameworks). Se despliega directo como GitHub Pages.

Socios: **Michail** y **José Luis**, ambos con acceso admin.

## 2. Arquitectura

```
Navegador (index.html)
   │
   ├── localStorage  → copia local instantánea (offline-first, respuesta inmediata)
   │
   └── fetch() → Google Apps Script Web App → Google Sheets ("DOREVIA BD")
                  (persistencia real, fuente de verdad compartida entre dispositivos)
```

- **`GAS_URL`** (línea 785): endpoint del Apps Script Web App.
  `https://script.google.com/macros/s/AKfycbx8BAmnv_IlBJTbRXKGF0Y6r_NR0LiSFmWErJ6N57tag4auG4hSTxcwBeU2wJHUjAc7/exec`
- **Google Spreadsheet ID** (backend, "DOREVIA BD"): `1cFbIOufamCCLVzZjZqp31cUGxf8vCdXU1BwWBzK53WE`
- `ld(k,d)` — lee de localStorage (línea 786)
- `sv(k,v)` — escribe en localStorage Y hace POST al Apps Script para sincronizar (línea 787)
- `loadAll()` — al abrir la app, hace GET `?action=getAll` al Apps Script y sobreescribe localStorage con lo que haya en Sheets (línea 788). Así los dos dispositivos quedan sincronizados.
- El código del backend (`Code.gs`) **no vive en este repo** — vive dentro del proyecto de Apps Script asociado al Google Sheet. Si necesitas editarlo, se hace desde Extensiones → Apps Script en el Sheet "DOREVIA BD".

## 3. Login

Usuarios por defecto (`DEFAULT_USERS`): `michail` / `jose`, contraseña `dorevia1`, rol admin.
Los admins pueden crear usuarios nuevos y resetear contraseñas desde la sección de Usuarios (`addUser`, `resetPw`, línea 1064-1067).

## 4. Módulos principales (por función clave)

| Módulo | Funciones clave | Qué hace |
|---|---|---|
| **Dashboard** | `renderDash()`, `kpiDrill()` | KPIs clicables: ventas del mes, ganancia neta, gastos, acumulado. Al tocar un KPI filtra el historial. |
| **Facturación** | `initFac()`, `renderProdGrid()`, `buildItems()`, `calcTotals()`, `applyDiscount()`, `saveVenta()`, `buildFacHTML()`, `dlPDF()` | Toggle Factura/Cotización con numeración independiente (`facNum`/`cotNum`, prefijos FAC-/COT-). Autocompletado de clientes (`setupAC`, `clientSugs`). Descuento manual con botón "Aplicar %" (no automático). Genera PDF standalone vía `buildFacHTML`. |
| **Deuda / Abonos** | `openDetail()`, `addAbono()` | Cada venta puede tener abonos parciales; el `saldo` se recalcula y el estado pasa a "Pagado" cuando llega a 0. |
| **Inventario** | `renderInv()`, `addStock()`, `updateAvgCost()`, `saveInv()`, `addNewProd()`, `delProd()`, `deductStock()`, `restoreStock()` | Costo promedio ponderado: `(stock_actual × costo_prom + qty_nueva × costo_nuevo) / total`. El stock se descuenta automáticamente al vender (`deductStock`) y se restaura si se borra una venta (`restoreStock`). Permite stock negativo si se factura de más. |
| **Caja / Movimientos** | `saveMovimiento()`, `setTipoMov()`, `clearCaja()`, `delMov()` | Registro de gastos/ingresos/inversiones/retiros. `tipoMov` por defecto es "Gasto". |
| **Historial** | `renderHist()`, `fillMeses()` | Filtros por mes y tipo (ventas/gastos). |
| **Contactos** | `renderContactos()`, `openAddContacto()`, `saveNewContacto()`, `openEditContacto()`, `saveEditContacto()`, `deleteContacto()`, `upsertClient()` | CRUD completo de clientes. Editar un nombre propaga el cambio a las ventas históricas de ese cliente (ver línea ~1290). |
| **Exportar** | `renderExportSummary()`, `xlsxReady()`, `makeSheet()`, `dlXlsx()` | Genera `.xlsx` reales vía SheetJS (cargado on-demand desde CDN) para ventas, inventario y contactos. |
| **Usuarios** | `renderUsers()`, `addUser()`, `resetPw()`, `changeMyPw()` | Gestión de accesos, solo admins pueden crear usuarios / resetear contraseñas ajenas. |

## 5. Cálculos de negocio importantes

- **Ganancia neta** = margen bruto de ventas − gastos (no solo margen de ventas).
- **Costo promedio ponderado** al ingresar stock nuevo (línea 851, `updateAvgCost`).
- **Totales de factura** (`calcTotals`, línea 857): subtotal → descuento % → + extra → + IVA opcional (19%) → total. Ganancia = total − costo total de los ítems.
- **IVA**: opcional por factura, 19% cuando se activa.

## 6. Cómo hacer cambios (flujo recomendado)

1. **Nunca edites directo en GitHub web para cambios grandes** — el historial de este proyecto mostró que el editor web rompe template literals y tags `<script>` en archivos grandes.
2. Flujo que sí funciona:
   - Clonar o traer el archivo actual (`raw.githubusercontent.com/michaaaaaail/Dorevia/main/index.html`).
   - Hacer los cambios con reemplazo de string exacto (no regex a ciegas) en un entorno con Python/bash.
   - Verificar antes de subir: balance de tags `<script>`, que las funciones clave sigan presentes.
   - Commit y push usando un GitHub Personal Access Token embebido en la URL remota:
     `git push https://TOKEN@github.com/michaaaaaail/Dorevia.git main`
3. Para cambios en el backend (Code.gs), se edita directo desde Apps Script en el Google Sheet "DOREVIA BD" — no está versionado en Git.
4. **Antes de actualizar el archivo en GitHub, exporta un respaldo en Excel** desde el módulo de Exportar — actualizar el HTML no borra los datos de Sheets, pero es buena práctica tener el respaldo.

## 7. Pendientes / notas sueltas de sesiones anteriores

- Se removió "Piercings" del branding.
- Login sin hints de usuario sugerido (credenciales se comparten en privado).
- El nav inferior pasó de 6 a 8 ítems a medida que se agregaron funciones.
- Cotizaciones no deben mostrar badge de "PAGADO".

## 8. Archivos en esta carpeta

- `index.html` — código fuente completo actual (bajado directo de GitHub, rama main)
- `README.md` — README del repo (mínimo)
- `HANDOFF.md` — este documento
