# CLAUDE.md — CuentasClaras

## Proyecto
- **Nombre:** CuentasClaras
- **Producto:** Mi Presupuesto 50/30/20
- **Descripción:** app de presupuesto personal basada en la regla 50/30/20. Gratis, sin registro, funciona offline. Los datos viven solo en el dispositivo del usuario.
- **Producción:** https://miscuentasclara.netlify.app
- **Idioma de la UI y del código de dominio:** español rioplatense (voseo: "Agregá", "Exportá").

## Stack
- HTML + CSS + JS puro en **un solo archivo: `index.html`** (~2260 líneas: `<style>` arriba, markup en el medio, `<script>` al final).
- Sin bundler, sin npm, sin framework, sin librerías JS.
- Única dependencia externa: Google Fonts (DM Sans e Inter) vía `<link>`.
- El donut se dibuja a mano con `<canvas>` (no hay Chart.js).

## Versionado
- `const APP_BUILD = 'YYYY.MM.DD.N'` — formato fecha + contador del día (ej. `2026.10.30.2`). Definido en `index.html` dentro del `<script>`, bajo `// ── VERSIÓN DE LA APP ──` (~línea 1360).
- **Cada cambio publicable requiere incrementar `APP_BUILD`**: mismo día → subir `N`; día nuevo → fecha de hoy y `N = 1`.
- Se guarda en `localStorage['cc_app_build']`; si difiere del build cargado, se muestra el banner de actualización.
- No confundir con:
  - `APP_VERSION` (número entero, hoy `2`): versión del **esquema de datos**, se usa en la migración. Solo se sube si cambia la estructura de `state`.
  - El texto `v1.1` en `.nav-brand` y `v1.0` en el instructivo/footer: etiquetas visuales, no están atadas a `APP_BUILD`.

## Deploy
- Flujo: **local → GitHub (rama `main`) → Netlify**, auto-deploy en ~30 s.
- Archivo local: `C:\Users\diazs\OneDrive\Escritorio\Workspace\cuentas-claras\index.html`
- Repo: https://github.com/diazsergio63/cuentas-claras
- Netlify: sitio `miscuentasclara`, Build command **vacío**, Publish directory **`.`**. Hay Netlify CLI instalado y `.netlify/` (en `.gitignore`) enlazado al sitio.
- Verificar el deploy: `curl` a la URL de producción y buscar el texto esperado.

## Arquitectura de la app
Seis secciones (`<section class="seccion" id="tab-…">`), navegación por tabs (`.tab[data-tab]` + `irTab(id)`):

| Tab | id | Qué hace |
|---|---|---|
| Instructivo | `instructivo` | Guía de uso; sección activa por defecto |
| Inicio | `inicio` | Hero, 4 pasos, regla visible, instalar PWA, export/import |
| Ajustes | `ajustes` | Año, moneda, porcentajes, categorías |
| Movimientos | `movimientos` | Alta/baja de ingresos, gastos y deudas por mes |
| Metas y Deudas | `metas` | Metas de ahorro (+ Abonar) y deudas (bola de nieve) |
| Resumen | `resumen` | Stats, donut, gasto por categoría, metas, movimientos del mes |

- Todo el estado está en el objeto global `state`; se persiste con `guardar()` y se carga con `cargar()`.
- Render imperativo: cada sección tiene su `render*()` (`renderAjustes`, `renderMovimientos`, `renderMetasDeudas`, `renderResumen`), invocado desde `irTab()`.
- Tema claro/oscuro: atributo `data-tema` en `<html>`, clave `cc_tema`.

### localStorage
| Clave | Contenido |
|---|---|
| `cc_presupuesto_v2` | `state` completo (JSON) |
| `cc_presupuesto_v1` | legacy; se migra y se borra |
| `cc_app_build` | último `APP_BUILD` visto |
| `cc_ultimo_aviso_export` | `"año-mes"` (mes 0-indexado) del último aviso de export |
| `cc_tema` | `claro` / `oscuro` |
| `sessionStorage.cc_banner_cerrado` | banner PWA cerrado en esta sesión |

### Forma de `state`
```
{ nombre, _version,
  ajustes: { anio, moneda, pctN, pctD, pctA, categorias: [{id, nombre, tipo, presupuesto}] },
  movimientos: [{id, tipo, concepto, monto, categoriaId, fecha:'YYYY-MM-DD'}],
  metas:  [{id, nombre, objetivo, actual}],
  deudas: [{id, nombre, total, pagado, cuota}],
  tipoMovActivo }
```
Los `id` nuevos son `Date.now()`; las categorías por defecto usan ids chicos (1–11).

## Regla de negocio 50/30/20
- Porcentajes **editables** por el usuario en Ajustes (`pctN`, `pctD`, `pctA`; default 50/30/20). Deben sumar 100 o `guardarAjustes()` rechaza el guardado.
- Se aplican sobre el **ingreso del mes** seleccionado.
- **Tipos de categoría:** `N` Necesidad, `D` Deseo, `S` Ahorro, `A` Deuda, `I` Ingreso (solo las categorías fijas).
- **Tipos de movimiento:** `ingreso`, `gasto`, `deuda`. (El ahorro NO se registra como movimiento: va por Metas → + Abonar. `renderMovimientos` aún conserva un icono `ahorro` por legado.)
- Donut: bloque "Necesidades" = gastos con categoría `N`; "Deseos" = gastos con categoría `D`; "Ahorro y Deuda" = ver más abajo.

## Categorías fijas
- **Sueldo / Salario** (id 10) y **Otros ingresos** (id 11), tipo `I`. **NO se pueden eliminar.**
- Viven en `CATS_FIJAS`; `renderCatLista()` las oculta de la lista de Ajustes (así no tienen botón ×) y `poblarCatSelect()` / `renderResumen()` las anteponen siempre. Mantener ese patrón (`[...CATS_FIJAS, ...categorias.filter(no fijas)]`) en cualquier lugar nuevo que liste categorías.

## Cosas críticas que NO romper
1. **Migración automática de datos v1→v2** (`migrar()`, `cargar()`): v1 tenía tipo `A` = "Ahorro/Deuda"; v2 lo separa en `S` y `A` según el nombre (regex ahorro|emergencia|fondo|reserva|meta). `cargar()` lee `cc_presupuesto_v2` y cae a `cc_presupuesto_v1`; si venía de v1 guarda en v2 y borra v1. Una futura v3 debe sumar un bloque `if (v < 3)` y subir `APP_VERSION`, sin tocar los anteriores.
2. **Banner de actualización** (`#banner-update`, `verificarActualizacion()`, `aplicarActualizacion()`, `cerrarBannerUpdate()`): detecta cambios de `APP_BUILD` contra `cc_app_build`. Si `APP_BUILD` no se incrementa, los usuarios existentes no se enteran de la actualización.
3. **Modal de export automático** (`verificarRecordatorioExport()`): se muestra los días 1–5 del mes, una vez por mes, solo si hay movimientos; se corre desde `init()`.
4. **Lógica del donut** (`renderResumen()` / `dibujarDonut()`): "Ahorro y Deuda" = `ahorros` (suma de `actual` de todas las metas, es decir, lo abonado a metas cuenta como Ahorro) + gastos con categoría `S` + movimientos `deuda` + movimientos con categoría `A`. Ojo: el total de metas no se filtra por mes. No cambiar esto sin avisar.
5. **PWA instalable en Android/iOS/PC:** meta tags (`apple-mobile-web-app-*`, `theme-color`, etc.), manifest generado como blob en JS, service worker inline (cache `cc-v1`) e iconos SVG como data URI. **No modificar los meta tags ni el manifest sin revisar el impacto** en la instalación y en la actualización (el service worker sirve primero desde caché).

## Convenciones de código
- **Idioma:** funciones, variables y comentarios en español, `camelCase` (`agregarMovimiento`, `renderResumen`, `poblarMesSelector`). Prefijos recurrentes: `render*` (pintar), `agregar*` / `borrar*` (CRUD), `toggle*`, `mostrar*` / `cerrar*` (modales), `verificar*`, `actualizar*`.
- **Script en una sola pieza:** secciones separadas por banners `// ═══ TÍTULO ═══` (ESTADO GLOBAL, NAVEGACIÓN, AJUSTES, MOVIMIENTOS, METAS Y DEUDAS, RESUMEN, EXPORT/IMPORT, INIT, TEMA OSCURO, PWA).
- **Eventos:** handlers inline (`onclick="agregarMeta()"`), por lo que las funciones deben seguir siendo globales. Las tabs usan `addEventListener`.
- **IDs del DOM:** kebab-case con prefijo de sección: `mov-concepto`, `meta-nombre`, `deuda-total`, `cat-tipo`, `res-stats`, `pct-n`.
- **Montos:** siempre mediante `fmt(n)`; meses mediante `mesStr(y, m)`; fechas como string `YYYY-MM-DD`, filtrado de mes con `fecha.startsWith('YYYY-MM')`.
- **Feedback al usuario:** `showToast(msg)` para confirmaciones; `alert()` / `confirm()` solo en validaciones y en import.
- **Persistencia:** llamar `guardar()` después de cada mutación de `state`. Excepción conocida: `borrarCat` y `agregarCategoria` solo persisten cuando el usuario toca "Guardar ajustes"; no cambiar sin pensarlo, es el comportamiento actual.
- **HTML dinámico:** plantillas con template literals en `innerHTML` (sin escapado de texto del usuario — tenerlo en cuenta si se toca).
- **CSS:** variables en `:root` (`--verde`, `--azul`, `--amarillo`, `--rojo`, `--gris-*`, `--radio`); modo oscuro con overrides `[data-tema="oscuro"] …`; el CSS responsive está al final del bloque principal (`@media (max-width: 400px)` y `@media (max-width: 480px)` para `.agregar-cat`). Si se agrega un componente con colores propios, agregar también su override oscuro.
- **Commits:** mensajes cortos en español con prefijo convencional (`fix:`, `chore:`, `test:`).

## Antes de cada sesión
- [ ] Leer este archivo
- [ ] Ver `APP_BUILD` actual
- [ ] Confirmar con el usuario qué funcionalidad se va a tocar
- [ ] No reescribir bloques enteros si solo hay que editar
