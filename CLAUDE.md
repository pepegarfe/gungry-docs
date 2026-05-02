# CLAUDE.md — Contexto completo del proyecto Antike

> **Propósito de este archivo:** Transferencia de contexto para que una IA (u otro desarrollador) pueda continuar el trabajo sin perder ninguna decisión tomada, patrón acordado ni advertencia crítica. Generado al final de la sesión de desarrollo inicial.

---

## 1. Descripción del proyecto

**Antike** es un sistema de cotización digital para una empresa mexicana de fabricación e instalación de espejos, vidrio y marcos metálicos. Reemplaza un proceso basado en Excel.

**Volumen:** ~200 proyectos/mes  
**Moneda:** MXN (pesos mexicanos)  
**Mercado:** México (fechas, idioma y formatos en español)

### Productos que maneja el sistema
- Espejos: natural, color, avejentado
- Vidrio: flotado, texturizado
- Con o sin marco metálico (acero, inoxidable, latón)
- Con o sin iluminación LED (básico, dimmable, touch, antifog)
- Procesos adicionales: bisel, chaflán, pulido recto, barrenos, resaques, templado, backing

---

## 2. Archivos del proyecto y propósito de cada uno

### Raíz del proyecto (`antike/`)

| Archivo | Propósito |
|---|---|
| `index.html` | Punto de entrada HTML. Carga fuentes DM Sans + DM Mono desde Google Fonts. |
| `vite.config.js` | Configuración del bundler Vite con plugin de React. Puerto 5173. |
| `package.json` | Dependencias: React 18, React Router 6, Supabase JS SDK v2, Vite 5, Tailwind 3. |
| `netlify.toml` | Configura deploy en Netlify: build command, publish dir, Node 18, redirect SPA y directorio de funciones (`netlify/functions/`) con runtime Python 3.9. |
| `generar_cotizacion.py` | Motor PDF con ReportLab. Genera el PDF profesional con miniaturas de cada pieza dibujadas en canvas. Función `generar_pdf(datos, output_path)`. |
| `.env.example` | Plantilla de variables de entorno. Ver sección de variables críticas abajo. |
| `.gitignore` | Excluye `node_modules/`, `dist/`, `.env.local`, `__pycache__/`. |
| `README.md` | Instrucciones de instalación, estructura y flujo completo. |

### `supabase/migrations/`

| Archivo | Propósito |
|---|---|
| `001_schema_inicial.sql` | Schema completo de PostgreSQL. Crear tablas, RLS, triggers, función `siguiente_folio()`, bucket de Storage para PDFs. **Ejecutar una sola vez en Supabase SQL Editor.** |

### `netlify/functions/`

| Archivo | Propósito |
|---|---|
| `netlify/functions/pdf.py` | Función serverless de Netlify (Python). Recibe `cotizacion_id` + JWT del usuario, valida con Supabase Auth, trae datos de la DB, ejecuta `generar_cotizacion.py`, sube el PDF a Supabase Storage y devuelve una URL firmada válida por 3600 segundos. Handler: `def handler(event, context)`. |

### `src/lib/`

| Archivo | Propósito |
|---|---|
| `src/lib/supabase.js` | **Archivo más crítico del frontend.** Cliente Supabase único + todos los helpers de DB organizados por dominio: `auth`, `clientesDB`, `cotizacionesDB`, `partidasDB`, `tarifasDB`, `pdfDB`. Toda operación de DB pasa por aquí. |
| `src/lib/calculos.js` | Motor de cálculo central. Funciones: `calcArea()`, `calcPerimetro()`, `calcPeso()`, `calcularPartida()`, `validarPartida()`, `dibujarMiniatura()`, `calcularRejas()`. **Este módulo es la única fuente de verdad para precios** — lo usan tanto el frontend como `api/pdf.py`. |

### `src/hooks/`

| Archivo | Propósito |
|---|---|
| `src/hooks/index.js` | Cuatro hooks de React: `useAuth()`, `useClientes()`, `useCotizaciones(clienteId?)`, `useTarifas()`. Cada hook encapsula loading, error y las operaciones CRUD. `useClientes` incluye suscripción Realtime de Supabase. |

### `src/pages/`

| Archivo | Propósito |
|---|---|
| `src/pages/Login.jsx` | Autenticación con email+contraseña y Google OAuth. Maneja registro y login. Redirige a `/historial` tras autenticarse. |
| `src/pages/Cotizador.jsx` | Página principal. Compuesta por: `FormPartida` (configurador de una partida con validaciones y preview canvas), `TablaPartidas` (lista editable), `PanelIndirectos` (empaque, envío, instalación, prorrateo, total). Guarda todo en Supabase al hacer clic en "Guardar". |
| `src/pages/CotizacionDetalle.jsx` | Vista de solo lectura de una cotización guardada. Muestra partidas, desglose financiero, historial de versiones del mismo folio con deltas de precio, comparativa automática contra versión anterior, y botón de generación de PDF. |
| `src/pages/Historial.jsx` | Layout de dos columnas: sidebar oscuro con lista de clientes (búsqueda + filtros por estado) y área principal con timeline de versiones agrupadas por folio. CRUD completo de clientes. |
| `src/pages/Tarifas.jsx` | Motor de tarifas con 7 pestañas. Al guardar, crea una nueva versión en la tabla `tarifas` de Supabase (no sobreescribe, versiona). Incluye simulador de impacto y exportación a JSON. |

### `src/`

| Archivo | Propósito |
|---|---|
| `src/App.jsx` | Router principal con React Router 6. Componente `Privado` como auth guard (redirige a `/login` si no hay sesión). Nav lateral común. Carga páginas con `lazy()` + `Suspense`. |
| `src/main.jsx` | Punto de entrada React. Monta `<App />` en `#root`. Importa `global.css`. |
| `src/styles/global.css` | **Todos los estilos de la app en un solo archivo.** Variables CSS (colores, tipografía, radios). Dark mode con `@media (prefers-color-scheme: dark)`. Tipografía: DM Sans (texto) + DM Mono (números, folios, código). |

### Archivos HTML standalone (prototipos previos a React)

Estos archivos son **versiones autónomas de demostración** — no forman parte del build final de React. Están en los outputs como referencia visual:

| Archivo | Propósito |
|---|---|
| `antike_historial.html` | Prototipo standalone del historial con datos demo hardcodeados. Referencia visual del diseño acordado. |
| `antike_motor_tarifas.html` | Prototipo standalone del motor de tarifas. Totalmente funcional sin backend. |

---

## 3. Arquitectura y decisiones de diseño

### Stack tecnológico

```
Frontend: React 18 + React Router 6 + CSS custom (sin Tailwind en runtime)
Build:    Vite 5
DB/Auth:  Supabase (PostgreSQL + Auth + Storage + Realtime)
API:      Netlify Functions (Python 3.9 para PDF)
PDF:      Python + ReportLab (no jsPDF, no Puppeteer)
Hosting:  Netlify (frontend + API functions en el mismo repo)
Fuentes:  DM Sans + DM Mono (Google Fonts)
```

### Decisión: CSS custom en lugar de Tailwind en runtime

Se decidió usar CSS custom con variables en lugar de clases de Tailwind en los componentes. Razón: los prototipos HTML standalone funcionan sin build pipeline, y Tailwind requiere compilador para las clases de utilidad. Todo el CSS vive en `src/styles/global.css`.

### Decisión: un solo cliente Supabase

`src/lib/supabase.js` exporta **una única instancia** `supabase` creada con `createClient()`. Todos los módulos importan desde ahí. Nunca crear una segunda instancia.

### Decisión: motor de cálculo como módulo independiente

`src/lib/calculos.js` es importado tanto por el frontend React como por `api/pdf.py` (via subprocess). Esto garantiza que el precio mostrado en pantalla y el precio impreso en el PDF **siempre coincidan**. Si cambias la lógica de precios, cámbiala solo aquí.

### Decisión: tarifas versionadas, nunca sobreescritas

La tabla `tarifas` en Supabase nunca actualiza un registro existente. Cada `guardar()` desactiva la versión anterior (`activa = false`) e inserta una nueva fila. Esto genera un historial inmutable de cambios de precios.

### Decisión: folio autogenerado en el servidor

El folio (`COT-2025-XXXX`) lo genera la función PostgreSQL `siguiente_folio(user_id)`, no el frontend. Esto evita colisiones en caso de múltiples usuarios o pestañas abiertas simultáneamente.

### Patrón de hooks

Cada hook sigue el mismo patrón:
```js
const { datos, loading, error, crear, actualizar, eliminar } = useRecurso()
```
Los hooks manejan el estado local optimista: actualizan el array en memoria inmediatamente y luego persisten en Supabase. Si Supabase falla, se debe agregar rollback (pendiente en Fase 2).

### Row Level Security (RLS)

**Todas** las tablas tienen RLS activo. La política en todas es `auth.uid() = user_id`. Esto significa:
- Nunca se filtra manualmente por `user_id` en el frontend — Supabase lo hace automáticamente
- La `service_role` key bypasea RLS — solo usarla en `api/pdf.py` (servidor), nunca en el frontend

---

## 4. Lógica de negocio crítica

### Fórmulas de precio (NO modificar sin revisar `calculos.js`)

```
Área:
  Rectángulo:  (ancho/1000) × (alto/1000)
  Circular:    π × (ancho/2000) × (alto/2000)
  Irregular:   ((ancho + bbox_mm)/1000) × ((alto + bbox_mm)/1000)
  donde bbox_mm = tarifas.bbox_mm (default: 80)

Perímetro: siempre 2 × ((ancho + alto) / 1000), sin importar la figura

Costo vidrio:  tarifa_vidrio[tipo] × área
Costo proceso: tarifa_proc_ml[proceso] × perímetro  (bisel, chaflán, pulido, resaques)
               tarifa_proc_area[proceso] × área       (templado, backing)
               tarifa_barreno × n_barrenos            (barrenos)
Costo marco:   perímetro × (tarifa_marco_ml[material] + tarifa_acabado[acabado]) + tarifa_armado
Costo LED:     tarifa_led[tipo] × perímetro

Precio venta:
  Vidrio y procesos × mult.vidrio  (default 2.0)
  Marco             × mult.marco   (default 2.5)
  LED               × mult.led     (default 2.5)
  Costo fijo        × 1.5          (siempre fijo)

Total cotización:
  Subtotal partidas
  + indirectos (empaque + envío + instalación)   ← prorrateable por m²
  + transporte especial                          ← no prorrateable
  + riesgo (% sobre base)
  − descuento (% sobre base + riesgo)
```

### Restricciones de validación (hardcodeadas en `calculos.js` y en UI)

| Restricción | Implementación |
|---|---|
| Bisel solo en espesor ≥ 6 mm | `validarPartida()` + chip deshabilitado en UI |
| Latón solo acabado cepillado | `validarPartida()` + select forzado en UI |
| LED solo en espejos (no vidrio) | `validarPartida()` + sección LED oculta en UI |
| Tamaños máximos por material | `tarifas.max_size[tipo]` configurable en Tarifas |
| Límite logístico 2,100 mm | `tarifas.lim_transporte` configurable, alerta (no bloqueo) |

### Cálculo de rejas de transporte

```
Por cada partida:
  Si peso_unitario > kg_reja  O  ancho > 1,200 mm  O  alto > 1,200 mm:
    → 1 reja por pieza (no se pueden combinar)
  Si no:
    → agrupar hasta kg_reja kg por reja
```

---

## 5. Variables de entorno

### Frontend (prefijo `VITE_` — expuestas al navegador)
```
VITE_SUPABASE_URL        URL del proyecto Supabase
VITE_SUPABASE_ANON_KEY   Clave anon/public (segura para exponer)
```

### API serverless (sin prefijo — solo en servidor)
```
SUPABASE_URL             Misma URL que arriba
SUPABASE_SERVICE_KEY     ⚠️ Clave service_role — NUNCA exponer al frontend
                         Bypasea RLS. Solo para api/pdf.py.
```

**En desarrollo:** archivo `.env.local` (ignorado por git)  
**En producción:** Netlify Dashboard → Site configuration → Environment variables

---

## 6. Schema de base de datos

### Tablas principales

```sql
users          (id uuid PK → auth.users, nombre, email, empresa, tel)
clientes       (id, user_id FK, nombre, empresa, tel, email, obra, notas)
cotizaciones   (id, user_id FK, cliente_id FK, folio, version, estado,
                obra, total, m2_total, n_partidas, peso_total, n_rejas,
                c_empaque, c_envio, c_instalacion, c_transporte_esp,
                pct_riesgo, pct_descuento, notas, fecha)
partidas       (id, cotizacion_id FK, user_id FK, orden, descripcion,
                tipo, espesor, ancho, alto, cantidad, figura,
                procesos text[], n_barrenos, marco, marco_acabado,
                led, area_unit, peso_unit, perimetro,
                c_vidrio, c_marco, c_led, c_fijo,
                costo_unit, precio_unit, precio_total)
tarifas        (id, user_id FK, version int, activa bool, datos jsonb, notas)
pdf_exports    (id, cotizacion_id FK, user_id FK, storage_path, url_firmada)
```

### Trigger automático al registrarse

`handle_new_user()` se dispara en `auth.users INSERT`. Crea:
1. Fila en `public.users`
2. Fila en `tarifas` con las tarifas por defecto en JSON

### Función `siguiente_folio(user_id)`

Retorna el siguiente folio del año actual en formato `COT-YYYY-NNNN`. Busca el máximo número en folios del año y suma 1. Llamar con `supabase.rpc('siguiente_folio', { p_user_id: uid })`.

---

## 7. Flujo de datos completo

```
Usuario abre /cotizar
  → useClientes() carga lista de clientes desde Supabase
  → useTarifas() carga tarifas activas (JSON) desde Supabase
  → FormPartida usa calculos.js para precio en tiempo real (sin red)
  → "Agregar partida" → estado local (array de partidas)
  → "Guardar cotización":
      1. cotizacionesDB.crear() → INSERT en cotizaciones
         (folio generado por siguiente_folio() en PostgreSQL)
      2. partidasDB.actualizarLote() → DELETE + INSERT de partidas
      3. navigate('/cotizar/:id')

Usuario hace clic en "Descargar PDF"
  → pdfDB.generar(cotizacion_id)
  → POST /.netlify/functions/pdf con JWT en header
  → netlify/functions/pdf.py verifica token con Supabase Auth
  → Trae cotización + partidas desde Supabase (service key)
  → Llama a generar_cotizacion.py con los datos
  → Sube PDF a Supabase Storage en pdfs/{user_id}/{folio}-{version}.pdf
  → Registra en pdf_exports
  → Retorna URL firmada (válida 60 min)
  → Frontend abre URL en nueva pestaña
```

---

## 8. ⚠️ Advertencias y cosas que NO tocar sin cuidado

### 1. `src/lib/calculos.js` — fuente única de verdad de precios
Cualquier cambio en las fórmulas de precio afecta **tanto el cotizador web como los PDFs ya generados**. Si cambias un multiplicador o fórmula aquí, avisa al equipo comercial porque los precios históricos en PDF no cambiarán (son inmutables), pero los nuevos sí.

### 2. `supabase/migrations/001_schema_inicial.sql` — ejecutar solo una vez
Si se ejecuta dos veces en el mismo proyecto, fallará por duplicados. Para hacer cambios al schema en producción, crear un nuevo archivo `002_nombre_cambio.sql` con `ALTER TABLE` o `CREATE TABLE` adicionales.

### 3. `SUPABASE_SERVICE_KEY` — nunca al frontend
Esta key bypasea todas las políticas de RLS. Si llega al navegador (por ejemplo, si se pone con prefijo `VITE_`), cualquier usuario podría leer datos de todos los demás. Solo vive en `netlify/functions/pdf.py`.

### 4. La restricción `unique (user_id, activa)` en `tarifas` es `DEFERRABLE`
La transacción de guardar tarifas hace primero `UPDATE activa=false` y luego `INSERT activa=true`. Si se hace en orden inverso o fuera de transacción, viola el unique. El código en `tarifasDB.guardar()` hace esto en dos queries separados — si Supabase no ejecuta ambos en la misma transacción implícita, puede haber un momento de inconsistencia. En producción con mucho tráfico, considerar wrappear en una función RPC PostgreSQL.

### 5. `netlify/functions/pdf.py` importa `generar_cotizacion` con `sys.path.insert`
El PDF generator (`generar_cotizacion.py`) debe estar en el directorio raíz del proyecto para que `netlify/functions/pdf.py` pueda importarlo. El path está configurado como `../../` relativo a la función. Si se mueve a un subdirectorio, actualizar el `sys.path.insert` en `netlify/functions/pdf.py`.

### 6. Canvas API en `calculos.js` — solo funciona en navegador
La función `dibujarMiniatura()` usa `canvas.getContext('2d')` que es una API del navegador. Si se intenta usar en Node.js (tests unitarios, SSR), fallará. En `api/pdf.py`, las miniaturas se generan con la clase `MiniaturaEspejo` de ReportLab (no reusan `calculos.js`).

### 7. El trigger `on_auth_user_created` requiere que `public.users` exista antes
Si se intenta crear un usuario en Supabase Auth antes de ejecutar la migración SQL, el trigger fallará silenciosamente y el usuario no tendrá perfil ni tarifas. Siempre ejecutar `001_schema_inicial.sql` antes del primer registro.

---

## 9. Estado actual del sistema — qué está hecho y qué falta

### ✅ Completado en esta sesión

| Módulo | Archivo(s) | Estado |
|---|---|---|
| Configurador de partida (UI + validaciones + canvas) | `Cotizador.jsx`, `calculos.js` | ✅ Completo |
| Lista de partidas + indirectos + prorrateo | `Cotizador.jsx` | ✅ Completo |
| Motor de tarifas editable | `Tarifas.jsx`, `tarifasDB` | ✅ Completo |
| Generador de PDF profesional | `generar_cotizacion.py`, `netlify/functions/pdf.py` | ✅ Completo |
| Historial de clientes y versiones | `Historial.jsx` | ✅ Completo |
| Detalle de cotización + comparativa de versiones | `CotizacionDetalle.jsx` | ✅ Completo |
| Auth (email + Google) | `Login.jsx`, `useAuth` | ✅ Completo |
| Schema de base de datos | `001_schema_inicial.sql` | ✅ Completo |
| Cliente Supabase + helpers CRUD | `supabase.js` | ✅ Completo |
| Estilos completos | `global.css` | ✅ Completo |
| Arquitectura de integración | `App.jsx`, `netlify.toml` | ✅ Completo |

### 🔜 Pendiente (Fase 2)

| Funcionalidad | Prioridad | Notas |
|---|---|---|
| Rollback optimista en hooks | Alta | Si Supabase falla, revertir estado local |
| Tests unitarios de `calculos.js` | Alta | Especialmente las fórmulas de precio |
| Página de configuración de perfil de usuario | Media | Cambiar nombre, empresa, logo |
| Logo de empresa en PDF | Media | Subir a Supabase Storage, leer en `generar_cotizacion.py` |
| Dashboard de métricas | Media | Cotizaciones por mes, m² fabricados, tasa de aprobación |
| Órdenes de producción | Baja | Tabla `ordenes` generada al aprobar una cotización |
| CRM básico | Baja | Estados de seguimiento, recordatorios, notas por cliente |
| Módulo de logística | Baja | Rutas de entrega, instalación, evidencia fotográfica |
| Multi-empresa / multi-usuario | Baja | Tabla `empresas`, invitar colaboradores |
| App móvil | Baja | React Native o PWA (el CSS ya es responsive) |

---

## 10. Comandos de desarrollo

```bash
# Instalar dependencias
npm install

# Desarrollo local (solo frontend)
npm run dev                 # → http://localhost:5173

# Desarrollo local con funciones serverless
netlify dev                 # → http://localhost:8888

# Build de producción
npm run build               # genera /dist

# Preview del build
npm run preview

# Deploy a Netlify (primera vez)
netlify login
netlify deploy              # deploy de preview

# Deploy a Netlify (producción)
netlify deploy --prod

# Ejecutar generador de PDF localmente (requiere reportlab)
pip install reportlab --break-system-packages
python generar_cotizacion.py   # genera cotizacion_antike.pdf de demo
```

---

## 11. Decisiones pendientes que requieren input del cliente

1. **Logo en PDF** — Antike no proporcionó un archivo de logo. El PDF actual usa el texto "ANTIKE" en tipografía. Cuando esté disponible el logo (PNG/SVG), agregarlo en `generar_cotizacion.py` en la sección `── ENCABEZADO ──`.

2. **Multiplicadores de precio** — Los valores actuales (vidrio ×2.0, marco ×2.5, LED ×2.5) son los proporcionados en el brief. Confirmar si reflejan exactamente la política comercial actual antes de poner en producción.

3. **Condiciones comerciales en PDF** — El PDF incluye 4 condiciones por defecto. Antike debería revisarlas y actualizarlas en `generar_cotizacion.py` → sección `condiciones` del demo, y en `construir_datos_pdf()` en `api/pdf.py`.

4. **Dominio** — El README asume `cotizador.antike.mx`. Si el dominio es diferente, actualizar en Netlify Dashboard → Domain management y en Supabase Auth → URL Configuration → Site URL.

5. **Equipo de ventas** — Si múltiples personas van a cotizar, decidir si comparten una cuenta o tienen cuentas separadas. El esquema ya soporta multi-usuario (RLS por `user_id`), pero las tarifas también son por usuario. Si deben compartir tarifas, hay que añadir un nivel de `empresa` entre `user` y las tarifas.

---

## 12. Dependencias externas y versiones fijadas

```json
{
  "@supabase/supabase-js": "^2.39.0",
  "react":                 "^18.2.0",
  "react-dom":             "^18.2.0",
  "react-router-dom":      "^6.22.0",
  "vite":                  "^5.1.0"
}
```

**Python (para PDF):**
```
reportlab  (instalado con pip install reportlab --break-system-packages)
```
No hay `requirements.txt` aún — crear uno si se agrega más dependencias Python.

---

## 13. Contexto de la conversación original

Este sistema fue diseñado y construido en una sola sesión de trabajo con Claude. El orden de desarrollo fue:

1. **Sprint 0** — Definición del negocio, productos, fórmulas de costo y restricciones
2. **Sprint 1** — Configurador de partida (widget interactivo HTML)
3. **Sprint 2** — Lista de partidas + cotización completa con indirectos (widget interactivo HTML)
4. **Sprint 3** — Motor de tarifas editable (archivo HTML standalone)
5. **Sprint 4** — Generador de PDF profesional (Python + ReportLab)
6. **Sprint 5** — Historial de clientes y versiones (archivo HTML standalone)
7. **Sprint 6** — Arquitectura de integración (diagrama + plan de sprints)
8. **Sprint 7** — Sprint 1 de integración: Schema SQL de Supabase + estructura React
9. **Sprint 8** — Sprint 2-4 de integración: páginas completas React conectadas a Supabase

Los archivos HTML standalone (historial, motor de tarifas) son prototipos funcionales que se usaron como referencia para construir las versiones React. Se mantienen en los outputs como documentación visual.

**Sesión 4 — Migración de Vercel a Netlify (2 mayo 2025)**
- `vercel.json` eliminado
- `netlify.toml` creado: build command, publish dir, Node 18, redirect SPA `/* → /index.html 200`, directorio de funciones `netlify/functions/`, runtime Python 3.9
- `api/pdf.py` movido a `netlify/functions/pdf.py` y handler reescrito de `BaseHTTPRequestHandler` al formato `def handler(event, context)` de Netlify Functions
- `public/_redirects` creado como respaldo SPA: `/*  /index.html  200`
- `src/lib/supabase.js`: URL de fetch `/api/pdf` → `/.netlify/functions/pdf`
- `README.md`: todas las referencias a Vercel CLI y comandos actualizadas a equivalentes Netlify
- `CLAUDE.md`: entradas de archivos, stack, advertencias y comandos actualizados a Netlify

---

*Documento generado el 30 de abril de 2025. Actualizado el 2 de mayo de 2025 (migración Netlify). Para continuar el trabajo, la IA debe leer este archivo completo antes de hacer cualquier cambio al código.*
