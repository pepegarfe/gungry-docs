# CLAUDE.md — Contexto completo del proyecto Antike

> **Propósito de este archivo:** Transferencia de contexto para que una IA (u otro desarrollador) pueda continuar el trabajo sin perder ninguna decisión tomada, patrón acordado ni advertencia crítica. Actualizado el 5 de mayo de 2026 (sesión 6).

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

### Roles de usuario
El sistema tiene tres roles con acceso diferenciado:

| Rol | Acceso | Autenticación |
|---|---|---|
| `admin` | Todo: cotizador, historial, tarifas, solicitudes de clientes | Email/contraseña o Google |
| `vendedor` | Cotizador + historial (no tarifas, no solicitudes) | Email/contraseña o Google |
| cliente externo | Solo formulario público `/cotizar` | Sin cuenta — solo nombre, empresa, email y tel |

El rol se almacena en `public.users.rol`. Para promover a alguien a admin:
```sql
UPDATE public.users SET rol = 'admin' WHERE email = 'correo@ejemplo.com';
```

---

## 2. Archivos del proyecto y propósito de cada uno

### Raíz del proyecto

| Archivo | Propósito |
|---|---|
| `index.html` | Punto de entrada HTML. Carga fuentes DM Sans + DM Mono desde Google Fonts. |
| `vite.config.js` | Configuración del bundler Vite con plugin de React. Puerto 5173. |
| `package.json` | Dependencias: React 18, React Router 6, Supabase JS SDK v2, Vite 5. |
| `vercel.json` | Configura deploy en Vercel: rutas SPA, función serverless Python para PDF (`api/pdf.py`) con runtime `python3.9` y timeout de 30s. |
| `generar_cotizacion.py` | Motor PDF con ReportLab. Genera el PDF profesional con miniaturas de cada pieza dibujadas en canvas. Función `generar_pdf(datos, output_path)`. |
| `.env.example` | Plantilla de variables de entorno. Ver sección de variables críticas abajo. |
| `.gitignore` | Excluye `node_modules/`, `dist/`, `.env.local`, `__pycache__/`. |

### `supabase/migrations/`

| Archivo | Propósito |
|---|---|
| `001_schema_inicial.sql` | Schema completo de PostgreSQL. Crear tablas, RLS, triggers, función `siguiente_folio()`, bucket de Storage para PDFs. **Ejecutar una sola vez en Supabase SQL Editor.** |

### `api/`

| Archivo | Propósito |
|---|---|
| `api/pdf.py` | Función serverless de Vercel (Python). Recibe `cotizacion_id` + JWT del usuario, valida con Supabase Auth, trae datos de la DB, ejecuta `generar_cotizacion.py`, sube el PDF a Supabase Storage y devuelve una URL firmada válida por 3600 segundos. |
| `api/crear-usuario.js` | Función serverless de Vercel (Node.js). Verifica JWT del solicitante, comprueba que tenga `rol = 'admin'` en `public.users`, luego llama a `supabase.auth.admin.createUser()` con la service_role key. Esto evita que la sesión del admin se sobreescriba (problema que ocurriría con `supabase.auth.signUp()` en el frontend). Devuelve `{ success, userId, email, nombre }`. |

### `src/lib/`

| Archivo | Propósito |
|---|---|
| `src/lib/supabase.js` | **Archivo más crítico del frontend.** Cliente Supabase único + todos los helpers de DB por dominio: `auth`, `clientesDB`, `cotizacionesDB`, `partidasDB`, `tarifasDB`, `pdfDB`, `solicitudesDB`, `usuariosDB`, `categoriasDB`, `variantesDB`. Toda operación de DB pasa por aquí. `clientesDB.listar()` y `cotizacionesDB.listar()` enriquecen cada registro con el campo `creador` (nombre + email del usuario que lo creó), usando una segunda query a `public.users`. |
| `src/lib/calculos.js` | Motor de cálculo central. Funciones: `calcArea()`, `calcPerimetro()`, `calcPeso()`, `calcularPartida()`, `validarPartida()`, `dibujarMiniatura()`, `calcularRejas()`. Exporta `TARIFAS_DEFAULT` (fallback de tarifas) y `TIPO_A_NOMBRE` (mapeo de tipo string → nombre de categoría en DB, ej. `'espejo_natural' → 'Espejo natural'`). **Este módulo es la única fuente de verdad para precios** — lo usan tanto el frontend como `api/pdf.py`. |

### `src/hooks/`

| Archivo | Propósito |
|---|---|
| `src/hooks/index.js` | Cinco hooks de React: `useAuth()`, `useClientes()`, `useCotizaciones(clienteId?)`, `useTarifas()`, `useCategorias()`. `useAuth()` también carga el `rol` desde `public.users`. `useCategorias()` carga las categorías con sus variantes embebidas (`variantes_producto`). |

### `src/components/`

| Archivo | Propósito |
|---|---|
| `src/components/MathBreakdown.jsx` | Componente compartido que muestra el desglose matemático completo de una partida (geometría, costos, multiplicadores, rentabilidad). Exporta también `mxn()` y `dec()` como named exports. Usado en `CotizacionDetalle` y `Solicitudes`. |

### `src/pages/`

| Archivo | Propósito |
|---|---|
| `src/pages/Login.jsx` | Autenticación con email+contraseña y Google OAuth. Solo login — el registro fue eliminado (las cuentas solo las puede crear el admin desde `/usuarios`). Redirige a `/historial` tras autenticarse. |
| `src/pages/Cotizador.jsx` | Página principal. `FormPartida` + `TablaPartidas` + `PanelIndirectos`. Guarda todo en Supabase al hacer clic en "Guardar". |
| `src/pages/CotizacionDetalle.jsx` | Vista de solo lectura de una cotización guardada. Partidas mostradas en layout de filas (`pd-*`). Admin ve botón "Ver lógica matemática" por partida (usa `MathBreakdown`). |
| `src/pages/Historial.jsx` | Layout de dos columnas: sidebar oscuro con clientes + timeline de versiones. CRUD completo de clientes. Admin ve todas las cotizaciones de todos los usuarios; cada tarjeta muestra creador y fecha de creación. Vendedor solo ve las suyas. |
| `src/pages/Tarifas.jsx` | Motor de tarifas editable con 7 pestañas. Solo accesible para admin. Al guardar versiona (no sobreescribe). |
| `src/pages/Solicitar.jsx` | **Formulario público para clientes externos** (sin autenticación). El cliente ingresa datos de contacto, configura productos y **elige variante de material** desde un grid de tarjetas (imagen 80×80 + nombre + precio/m²). Usa `useCategorias()` + `TIPO_A_NOMBRE` para derivar las variantes activas de la categoría seleccionada. `calcPreview()` pasa `varianteObj` como tercer argumento a `calcularPartida()`. Botón "Agregar producto" bloqueado hasta seleccionar variante si la categoría tiene alguna activa. **No muestra precios al cliente.** Importa `TARIFAS_DEFAULT` y `TIPO_A_NOMBRE` de `calculos.js`. Ruta: `/cotizar`. |
| `src/pages/Solicitudes.jsx` | **Vista admin de solicitudes de clientes.** Sidebar con lista + filtros por estado (pendiente/enviada/convertida). Panel de detalle con miniatura canvas por producto y botón "Ver lógica matemática" (usa `MathBreakdown`). Solo accesible para admin. Ruta: `/solicitudes`. |
| `src/pages/Usuarios.jsx` | **Gestión de usuarios (solo admin).** Formulario para crear nuevas cuentas de vendedor (llama a `api/crear-usuario.js`). Tabla de usuarios existentes con selector de rol. Ruta: `/usuarios`. |
| `src/pages/Materiales.jsx` | **Gestión de variantes de materiales (solo admin).** Layout dos columnas: sidebar con las 5 categorías fijas + panel con grid de tarjetas de variantes. Cada tarjeta permite editar precio inline, toggle activa/inactiva y eliminar. Formulario inline para crear variante con upload de imagen. Ruta: `/materiales`. |

### `src/`

| Archivo | Propósito |
|---|---|
| `src/App.jsx` | Router con React Router 6. Componente `Privado` con guard de auth + guard de rol (`soloAdmin`). Nav lateral: links principales (nueva-cotizacion, historial, solicitudes) + submenú de configuración ⚙ en el footer (Tarifas y Usuarios, solo admin). Rutas lazy con `Suspense`. `/cotizar` es pública (sin `Privado`). Rutas privadas del cotizador: `/nueva-cotizacion` y `/nueva-cotizacion/:id`. Rutas admin-only: `/tarifas`, `/solicitudes`, `/usuarios`. |
| `src/main.jsx` | Punto de entrada React. Monta `<App />` en `#root`. Importa `global.css`. |
| `src/styles/global.css` | **Todos los estilos en un solo archivo.** Variables CSS, dark mode, DM Sans + DM Mono. Incluye clases `.sol-*` (formulario público, incl. `.sol-variantes-grid` y `.sol-variante-card/img/placeholder/nombre/precio` para el selector de variantes), `.sq-*` (vista solicitudes admin), `.math-breakdown` / `.mb-*` (desglose matemático), `.pd-*` (cards de partida en filas), `.mat-*` (gestión de materiales), `.usu-*` (gestión de usuarios), `.nav-config-*` (submenú de configuración). Toasts usan `background: #1e1e1c` (hardcoded) para no romper en dark mode. |

---

## 3. Arquitectura y decisiones de diseño

### Stack tecnológico

```
Frontend:   React 18 + React Router 6 + CSS custom
Build:      Vite 5
DB/Auth:    Supabase (PostgreSQL + Auth + Storage + Realtime)
API:        Vercel Serverless Functions (Python 3.9 para PDF)
PDF:        Python + ReportLab
Hosting:    Vercel (frontend + API functions en el mismo repo)
Fuentes:    DM Sans + DM Mono (Google Fonts)
```

### Decisión: CSS custom en lugar de Tailwind en runtime

Todo el CSS vive en `src/styles/global.css` con variables CSS. No se usan clases de Tailwind en los componentes.

### Decisión: un solo cliente Supabase

`src/lib/supabase.js` exporta **una única instancia** `supabase`. Todos los módulos importan desde ahí. Nunca crear una segunda instancia.

### Decisión: motor de cálculo como módulo independiente

`src/lib/calculos.js` es la única fuente de verdad para precios. Lo importan tanto el frontend React como `api/pdf.py`. Si cambias la lógica de precios, cámbiala solo aquí.

### Decisión: tarifas versionadas, nunca sobreescritas

La tabla `tarifas` en Supabase nunca actualiza un registro existente. Cada `guardar()` desactiva la versión anterior (`activa = false`) e inserta una nueva fila.

### Decisión: folio autogenerado en el servidor

El folio (`COT-2025-XXXX`) lo genera la función PostgreSQL `siguiente_folio(user_id)`. Evita colisiones con múltiples usuarios o pestañas simultáneas.

### Decisión: formulario de clientes sin SELECT en Supabase

`solicitudesDB.crear()` hace `INSERT` sin `.select().single()` al final. La razón: la política RLS de `solicitudes` solo otorga `INSERT` al rol `anon` (clientes sin cuenta), no `SELECT`. Agregar `.select()` después del insert causaría error de RLS.

### Decisión: sistema de categorías y variantes de materiales

Las categorías (`categorias_producto`) son fijas (5 tipos: espejo natural/color/avejentado, vidrio flotado/texturizado). Las variantes (`variantes_producto`) son administrables por el admin desde `/materiales`. `categoriasDB.listar()` devuelve categorías con variantes embebidas (`select('*, variantes_producto(*)')`). Su RLS permite lectura anónima, por lo que funciona tanto en páginas autenticadas como en la página pública `/cotizar`.

Tanto `Cotizador.jsx` como `Solicitar.jsx` usan `useCategorias()` para cargar categorías + variantes en un solo request. El mapeo de tipo string a nombre de categoría se hace con `TIPO_A_NOMBRE` (exportado de `calculos.js`). Las variantes activas se derivan en render sin llamadas adicionales: `categorias.find(c => c.nombre === TIPO_A_NOMBRE[form.tipo])?.variantes_producto.filter(v => v.activa)`. `Materiales.jsx` consulta directo a Supabase sin filtro `activa` para ver también las inactivas.

### Decisión: variante en calcularPartida como tercer parámetro opcional

`calcularPartida(form, tarifas, variante = null)` acepta un tercer argumento. Si se pasa variante con `precio_m2`, ese valor sustituye al precio del objeto tarifas. Retrocompatible: todo el código que no pasa variante sigue funcionando con el precio de `tarifas.vidrio[tipo]`.

### Decisión: variante_id en partidas como FK nullable

`partidas.variante_id uuid REFERENCES variantes_producto(id) ON DELETE SET NULL` (migración 003). Al guardar una cotización se persiste la variante elegida. Al cargar la partida para editar, `form.variante_id` se restaura y `varianteObj` se resuelve desde las categorías cargadas en memoria — el preview recalcula con el precio correcto de la variante.

### Decisión: MathBreakdown como componente compartido

El desglose matemático de cada partida vive en `src/components/MathBreakdown.jsx` y se importa tanto en `CotizacionDetalle` como en `Solicitudes`. También exporta `mxn()` y `dec()` para que los archivos que lo importan no necesiten redefinirlos.

### Decisión: TARIFAS_DEFAULT en calculos.js como única fuente de valores por defecto

`calculos.js` exporta `TARIFAS_DEFAULT`, el objeto canónico de tarifas iniciales. Lo usan:
- `Solicitar.jsx` (formulario público) — no tiene acceso autenticado a Supabase, así que usa `TARIFAS_DEFAULT` como constante local. El total se calcula internamente para guardarse en la DB, pero **no se muestra al cliente** — los precios solo se comunican cuando el admin manda la cotización.
- `useTarifas()` — si `tarifasDB.obtenerActivas()` devuelve `null` (sin fila activa en DB) o lanza error, el hook cae a `TARIFAS_DEFAULT` en lugar de quedarse en `loading: true`.

Si los precios base cambian, actualizar **solo** `TARIFAS_DEFAULT` en `calculos.js`.

### Decisión: creación de cuentas vía función serverless (no `signUp` en cliente)

`supabase.auth.signUp()` en el frontend sobreescribiría la sesión activa del admin. Por eso, `Usuarios.jsx` llama a `api/crear-usuario.js` (Node.js serverless), que usa la `SUPABASE_SERVICE_KEY` para llamar a `supabase.auth.admin.createUser()`. La sesión del admin no se ve afectada. Solo debe llamarse desde cuentas admin (la función verifica el JWT entrante).

### Decisión: enriquecimiento de creador en dos queries

`cotizaciones.user_id` apunta a `auth.users`, no a `public.users`, por lo que PostgREST no puede hacer el join automáticamente. El patrón usado: traer registros, extraer los `user_id` únicos, consultar `public.users` con `.in('id', ids)`, y combinar cliente-side. Esto añade el campo `creador: { nombre, email }` a cada registro sin necesidad de cambiar el schema.

### Decisión: nav de configuración en submenú colapsable

Los links de Tarifas y Usuarios se movieron a un submenú `⚙` que aparece junto al email del usuario en el footer de la nav lateral. Esto despeja la nav principal y agrupa las opciones de configuración en un solo lugar. Solo visible para admin.

### Patrón de hooks

```js
const { datos, loading, error, crear, actualizar, eliminar } = useRecurso()
```

### `useAuth()` — patrón actualizado

```js
export function useAuth() {
  // Carga user desde supabase.auth.getSession()
  // Luego carga rol desde public.users (default 'vendedor' si falla)
  // setLoading(false) solo después de tener ambos
  return { user, rol, loading, login, loginConGoogle, logout }
}
```

### Routing con guards de rol

```jsx
function Privado({ children, soloAdmin = false }) {
  const { user, rol, loading } = useAuth()
  if (loading) return <div className="loading-full">Cargando…</div>
  if (!user)   return <Navigate to="/login" replace />
  if (soloAdmin && rol !== 'admin') return <Navigate to="/historial" replace />
  return children
}
```

Rutas admin-only: `/tarifas`, `/solicitudes`, `/usuarios`.  
Ruta pública (sin Privado): `/cotizar` (formulario de clientes externos).  
Rutas del cotizador interno: `/nueva-cotizacion` y `/nueva-cotizacion/:id`.

### Row Level Security (RLS)

**Todas** las tablas internas tienen RLS activo con política `auth.uid() = user_id`. Para `solicitudes`:
- `anon` puede INSERT (clientes externos sin cuenta)
- `authenticated` puede SELECT/UPDATE/DELETE si es admin (verificar por `rol` en `public.users`)

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

### Restricciones de validación

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
                         Bypasea RLS. La usan api/pdf.py y api/crear-usuario.js.
```

**En desarrollo:** archivo `.env.local` (ignorado por git)  
**En producción:** Vercel Dashboard → Settings → Environment Variables

---

## 6. Schema de base de datos

### Tablas principales

```sql
users          (id uuid PK → auth.users, nombre, email, empresa, tel,
                rol text DEFAULT 'vendedor')   ← 'admin' | 'vendedor'

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

solicitudes    (id, nombre, empresa, email, tel,
                partidas jsonb,   ← array de objetos con todos los campos de la partida + precios
                total numeric,
                estado text DEFAULT 'pendiente',   ← 'pendiente' | 'enviada' | 'convertida'
                created_at timestamptz)
```

### Trigger automático al registrarse

`handle_new_user()` se dispara en `auth.users INSERT`. Crea:
1. Fila en `public.users` (con `rol = 'vendedor'` por defecto)
2. Fila en `tarifas` con las tarifas por defecto en JSON

**Importante:** El trigger debe usar `jsonb_build_object()` en lugar de literales de cadena JSON para evitar errores de caracteres CRLF en Windows:
```sql
-- Correcto (usar esto):
INSERT INTO public.users (id, nombre, email)
  VALUES (NEW.id, nombre_val, email_val);

-- Usar jsonb_build_object() para construir JSON, NO concatenar strings
```

### Función `siguiente_folio(user_id)`

Retorna el siguiente folio del año actual en formato `COT-YYYY-NNNN`. Llamar con:
```js
supabase.rpc('siguiente_folio', { p_user_id: uid })
```

### RLS para `solicitudes`

```sql
-- Clientes externos pueden insertar sin cuenta
CREATE POLICY "anon insert" ON solicitudes
  FOR INSERT TO anon WITH CHECK (true);

-- Admins autenticados pueden leer y modificar
CREATE POLICY "admin select" ON solicitudes
  FOR SELECT TO authenticated USING (true);
CREATE POLICY "admin update" ON solicitudes
  FOR UPDATE TO authenticated USING (true);
```

### RLS adicional para visibilidad admin en clientes y cotizaciones

Las políticas base solo permiten que cada usuario vea sus propios registros. Para que el admin vea todos, ejecutar en Supabase SQL Editor:

```sql
-- Admin ve todos los clientes
CREATE POLICY "admin ve todos los clientes"
  ON public.clientes FOR SELECT TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');

-- Admin ve todas las cotizaciones
CREATE POLICY "admin ve todas las cotizaciones"
  ON public.cotizaciones FOR SELECT TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');
```

Supabase combina políticas permisivas con OR, por lo que vendedor sigue viendo solo las suyas y admin ve todo.

### RLS para la tabla `users` (gestión desde `/usuarios`)

```sql
-- Cualquier usuario autenticado puede leer perfiles (para mostrar nombre del creador)
CREATE POLICY "authenticated read users"
  ON public.users FOR SELECT TO authenticated USING (true);

-- Solo admin puede cambiar el rol de otros usuarios
CREATE POLICY "admin update rol"
  ON public.users FOR UPDATE TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');
```

---

## 7. Flujo de datos completo

### Flujo interno (admin/vendedor)

```
Usuario abre /nueva-cotizacion
  → useClientes() carga lista de clientes desde Supabase
  → useTarifas() carga tarifas activas (JSON) desde Supabase
      (si no hay fila activa o error → usa TARIFAS_DEFAULT de calculos.js)
  → FormPartida usa calculos.js para precio en tiempo real (sin red)
  → "Agregar partida" → estado local (array de partidas)
  → "Guardar cotización":
      1. cotizacionesDB.crear() → INSERT en cotizaciones
         (folio generado por siguiente_folio() en PostgreSQL)
      2. partidasDB.actualizarLote() → DELETE + INSERT de partidas
      3. navigate('/nueva-cotizacion/:id')
```

### Flujo de solicitudes de clientes externos

```
Cliente abre /cotizar (público, sin auth)
  → Ingresa datos de contacto (nombre, empresa, email, tel)
  → Configura productos con TARIFAS_DEFAULT (de calculos.js) en Solicitar.jsx
  → Ve precio estimado en tiempo real (canvas preview incluido)
  → "Enviar solicitud":
      solicitudesDB.crear() → INSERT en solicitudes (rol anon, sin SELECT)
      → Pantalla de éxito con total estimado

Admin abre /solicitudes
  → solicitudesDB.listar() carga todas las solicitudes
  → Ve sidebar con lista + filtros (pendiente / enviada / convertida)
  → Selecciona solicitud → ve detalle completo:
      - Miniatura canvas por producto
      - Tags de procesos, marco, LED
      - "Ver lógica matemática" → MathBreakdown (carga tarifas activas de DB)
  → Puede cambiar estado: pendiente → enviada → convertida
```

### Flujo de PDF

```
Admin hace clic en "Descargar PDF"
  → pdfDB.generar(cotizacion_id)
  → POST /api/pdf con JWT en header
  → api/pdf.py verifica token con Supabase Auth
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
Cualquier cambio afecta **tanto el cotizador web como los PDFs ya generados**. Si cambias un multiplicador, avisa al equipo comercial porque los precios históricos en PDF son inmutables.

### 2. `supabase/migrations/001_schema_inicial.sql` — ejecutar solo una vez
Si se ejecuta dos veces, fallará por duplicados. Para cambios al schema en producción, crear `002_nombre_cambio.sql`.

### 3. `SUPABASE_SERVICE_KEY` — nunca al frontend
Esta key bypasea RLS. Si llega al navegador (prefijo `VITE_`), cualquier usuario podría leer datos de todos los demás. La usan `api/pdf.py` y `api/crear-usuario.js`. Nunca agregarla con prefijo `VITE_`.

### 4. La restricción `unique (user_id, activa)` en `tarifas` es `DEFERRABLE`
La transacción hace primero `UPDATE activa=false` y luego `INSERT activa=true`. En producción con mucho tráfico, considerar wrappear en una función RPC PostgreSQL.

### 5. `api/pdf.py` importa `generar_cotizacion` con `sys.path.insert`
`generar_cotizacion.py` debe estar en el directorio raíz. Si se mueve a un subdirectorio, actualizar el path en `api/pdf.py`.

### 6. Canvas API en `calculos.js` — solo funciona en navegador
`dibujarMiniatura()` usa `canvas.getContext('2d')`. En `api/pdf.py` las miniaturas se generan con ReportLab, no con `calculos.js`.

### 7. El trigger `on_auth_user_created` requiere que `public.users` exista antes
Si se crea un usuario antes de ejecutar la migración SQL, el trigger falla silenciosamente. Siempre ejecutar `001_schema_inicial.sql` antes del primer registro.

### 8. `solicitudesDB.crear()` no usa `.select()` después del INSERT
La política RLS de `solicitudes` solo da `INSERT` al rol `anon`. Agregar `.select().single()` causaría error de RLS. No "corregir" esto añadiendo `.select()`.

### 9. Trigger `handle_new_user` — no usar literales de cadena JSON en Windows
En Windows, pegar SQL con strings JSON multilinea puede insertar caracteres CRLF (`0x0d`) que rompen el JSON en PostgreSQL. Usar `jsonb_build_object()` para construir el JSON de tarifas por defecto en el trigger.

### 10. `api/crear-usuario.js` — siempre verificar JWT antes de llamar al admin API
La función verifica que el token Bearer corresponda a un usuario con `rol = 'admin'` en `public.users`. Si se omite esa verificación, cualquiera con un JWT válido podría crear cuentas. No simplificar esta función.

### 11. Toasts — no usar variables CSS que cambien en dark mode
El color de fondo del toast usa `background: #1e1e1c` (hardcoded). Si se cambia a `var(--ink)` u otra variable que se invierte en dark mode, el texto y el fondo quedarán del mismo color y el toast será ilegible. Mantener valores hardcoded para elementos que necesiten color fijo.

---

## 9. Estado actual del sistema

### ✅ Completado

| Módulo | Archivo(s) | Estado |
|---|---|---|
| Configurador de partida (UI + validaciones + canvas) | `Cotizador.jsx`, `calculos.js` | ✅ |
| Lista de partidas + indirectos + prorrateo | `Cotizador.jsx` | ✅ |
| Motor de tarifas editable (solo admin) | `Tarifas.jsx`, `tarifasDB` | ✅ |
| Generador de PDF profesional | `generar_cotizacion.py`, `api/pdf.py` | ✅ |
| Historial con vista de admin (todas las cots. + creador) | `Historial.jsx` | ✅ |
| Detalle de cotización + comparativa de versiones | `CotizacionDetalle.jsx` | ✅ |
| Desglose matemático por partida (solo admin) | `MathBreakdown.jsx` | ✅ |
| Auth login-only (registro eliminado del frontend) | `Login.jsx`, `useAuth` | ✅ |
| Control de acceso por rol (admin / vendedor) | `App.jsx`, `useAuth`, `public.users.rol` | ✅ |
| Gestión de usuarios y creación de cuentas (admin) | `Usuarios.jsx`, `api/crear-usuario.js`, `usuariosDB` | ✅ |
| Submenú de configuración ⚙ en nav lateral | `App.jsx` | ✅ |
| Categorías y variantes de materiales (DB + admin UI) | `002_variantes_material.sql`, `Materiales.jsx`, `categoriasDB`, `variantesDB` | ✅ |
| Selector de variante en cotizador interno (chips con imagen, botón bloqueado sin variante) | `Cotizador.jsx`, `useCategorias`, `calcularPartida` | ✅ |
| Selector de variante en formulario público (grid tarjetas 80×80, botón bloqueado sin variante) | `Solicitar.jsx`, `useCategorias`, `TIPO_A_NOMBRE` | ✅ |
| Formulario público sin precios estimados | `Solicitar.jsx` | ✅ |
| Vista admin de solicitudes de clientes | `Solicitudes.jsx` | ✅ |
| Schema de base de datos (incl. tabla solicitudes) | `001_schema_inicial.sql` | ✅ |
| Cliente Supabase + helpers CRUD completos | `supabase.js` | ✅ |
| Estilos completos | `global.css` | ✅ |

### ⚠️ SQL pendiente de ejecutar en Supabase

Estas políticas deben correrse en **Supabase SQL Editor** antes de que el admin pueda ver todos los registros:

```sql
-- Visibilidad total para admin
CREATE POLICY "admin ve todos los clientes" ON public.clientes FOR SELECT TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');
CREATE POLICY "admin ve todas las cotizaciones" ON public.cotizaciones FOR SELECT TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');

-- Lectura de perfiles de usuario (para mostrar creador en historial)
CREATE POLICY "authenticated read users" ON public.users FOR SELECT TO authenticated USING (true);
CREATE POLICY "admin update rol" ON public.users FOR UPDATE TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');
```

### 🔜 Pendiente

| Funcionalidad | Prioridad | Notas |
|---|---|---|
| Envío de email al recibir solicitud de cliente | Alta | Usar Resend o SendGrid desde una function de Vercel |
| "Convertir solicitud en cotización" | Alta | Botón en `/solicitudes` que pre-rellena `/nueva-cotizacion` con los datos de la solicitud |
| Deploy a Vercel con variables de entorno | Alta | Configurar VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY y SUPABASE_SERVICE_KEY |
| Rollback optimista en hooks | Media | Si Supabase falla, revertir estado local |
| Tests unitarios de `calculos.js` | Media | Especialmente las fórmulas de precio |
| Página de configuración de perfil de usuario | Media | Cambiar nombre, empresa, logo |
| Logo de empresa en PDF | Media | Subir a Supabase Storage, leer en `generar_cotizacion.py` |
| Dashboard de métricas | Media | Cotizaciones por mes, m² fabricados, tasa de aprobación |
| Órdenes de producción | Baja | Tabla `ordenes` generada al aprobar una cotización |
| CRM básico | Baja | Estados de seguimiento, recordatorios, notas por cliente |
| Multi-empresa / multi-usuario | Baja | Tabla `empresas`, invitar colaboradores |

---

## 10. Comandos de desarrollo

```bash
# Instalar dependencias (en PowerShell en Windows)
& "C:\Program Files\nodejs\npm.cmd" install

# Desarrollo local — solo frontend (las rutas /api/* NO funcionan con este comando)
npm run dev                 # → http://localhost:5173

# Desarrollo local con funciones serverless activas (necesario para /api/crear-usuario y /api/pdf)
# Requiere: npm i -g vercel  y  tener SUPABASE_URL + SUPABASE_SERVICE_KEY en .env.local
vercel dev                  # → http://localhost:3000

# Build de producción
npm run build               # genera /dist

# Preview del build
npm run preview

# Deploy a Vercel (primera vez)
npx vercel                  # wizard interactivo

# Deploy a Vercel (producción)
npx vercel --prod

# Ejecutar generador de PDF localmente (requiere reportlab)
pip install reportlab --break-system-packages
python generar_cotizacion.py   # genera cotizacion_antike.pdf de demo
```

> **Nota Windows:** Si `npm` no se reconoce en cmd.exe, usar PowerShell y referenciar la ruta completa: `& "C:\Program Files\nodejs\npm.cmd"`.

---

## 11. Decisiones pendientes que requieren input del cliente

1. **Logo en PDF** — Antike no proporcionó un archivo de logo. El PDF usa el texto "ANTIKE". Cuando esté disponible (PNG/SVG), agregarlo en `generar_cotizacion.py` sección `── ENCABEZADO ──`.

2. **Multiplicadores de precio** — Los valores actuales (vidrio ×2.0, marco ×2.5, LED ×2.5) son los del brief inicial. Confirmar con el equipo comercial antes de producción.

3. **Condiciones comerciales en PDF** — Revisar y actualizar en `generar_cotizacion.py` y en `construir_datos_pdf()` en `api/pdf.py`.

4. **Dominio** — El README asume `cotizador.antike.mx`. Si es diferente, actualizar en Vercel Dashboard y en Supabase Auth → URL Configuration → Site URL.

5. **Tarifas del formulario público** — `Solicitar.jsx` importa `TARIFAS_DEFAULT` de `calculos.js`. Si los precios base cambian, actualizar únicamente `TARIFAS_DEFAULT` en `calculos.js`; `Solicitar.jsx` lo refleja automáticamente.

6. **Email de confirmación al cliente** — Definir qué plataforma usar (Resend recomendado por su integración con Vercel) y qué información incluir en el correo.

---

## 12. Dependencias externas y versiones

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
reportlab  (pip install reportlab --break-system-packages)
```

---

## 13. Historial de desarrollo

### Sesión 1 — Sistema base
1. Definición del negocio, productos, fórmulas y restricciones
2. Configurador de partida (widget interactivo HTML)
3. Lista de partidas + cotización completa (widget HTML)
4. Motor de tarifas editable (HTML standalone)
5. Generador de PDF profesional (Python + ReportLab)
6. Historial de clientes y versiones (HTML standalone)
7. Arquitectura de integración
8. Schema SQL de Supabase + estructura React
9. Páginas completas React conectadas a Supabase

### Sesión 2 — Roles, solicitudes de clientes y desglose matemático
1. Control de acceso por rol (admin / vendedor) en `App.jsx` y `useAuth()`
2. Columna `rol` en `public.users`, trigger `handle_new_user` corregido (CRLF → `jsonb_build_object`)
3. Tabla `solicitudes` con RLS diferenciada (anon INSERT, authenticated SELECT/UPDATE)
4. Formulario público `/solicitar` para clientes externos (sin cuenta)
5. Vista admin `/solicitudes` con sidebar, filtros, miniaturas canvas y gestión de estado
6. Componente compartido `MathBreakdown` con desglose matemático completo
7. Desglose matemático en `CotizacionDetalle` (admin) y en `Solicitudes` (admin)
8. Corrección: `solicitudesDB.crear()` sin `.select()` para cumplir RLS de anon
9. Corrección: `clientesDB.crear()` incluye `user_id` para cumplir FK constraint

### Sesión 3 — Usuarios, configuración y visibilidad admin
1. Página `/usuarios` para que admin cree y gestione cuentas de vendedor
2. `api/crear-usuario.js` (Node.js serverless) — crea usuarios con `admin.createUser()` sin afectar la sesión del admin
3. `Login.jsx` — eliminado el registro; las cuentas solo las crea el admin
4. Submenú de configuración ⚙ en el footer de la nav lateral (Tarifas + Usuarios), solo admin
5. `Solicitar.jsx` — eliminados los precios estimados del formulario público; solo se muestran al recibir la cotización
6. `supabase.js` — añadido `usuariosDB` (listar, cambiarRol, crear); `clientesDB.listar()` y `cotizacionesDB.listar()` enriquecidos con campo `creador` (patrón de dos queries)
7. `Historial.jsx` — admin ve todas las cotizaciones de todos los usuarios; sidebar muestra creador del cliente; tarjetas de cotización muestran creador y fecha de creación
8. `global.css` — nuevas clases `.pd-*` (layout de filas en partidas), `.usu-*` (usuarios), `.nav-config-*` (submenú nav), `.ci-creador`, `.ver-creador`; fix toasts con color hardcoded
9. Corrección: `cotizacionesDB.crear()` incluía `user_id` faltante en el INSERT (mismo bug que `clientesDB.crear()` en sesión 2)
10. Corrección: toasts invisibles en dark mode — `background: var(--ink)` reemplazado por `background: #1e1e1c`
11. SQL pendiente: 4 políticas RLS nuevas (admin ve clientes/cotizaciones de todos; usuarios legibles por todos los autenticados; admin puede cambiar rol)

### Sesión 5 — Sistema de categorías y variantes de materiales
1. `supabase/migrations/002_variantes_material.sql` — tablas `categorias_producto` y `variantes_producto`, RLS pública para lectura, admin para escritura, bucket `materiales` en Storage, datos iniciales (5 categorías base)
2. `supabase.js` — añadidos `categoriasDB` (listar con variantes embebidas) y `variantesDB` (listarPorCategoria con activa=true, listarTodosPorCategoria sin filtro, crear, actualizar, eliminar, subirImagen)
3. `hooks/index.js` — añadido `useCategorias()` que carga categorías con variantes embebidas
4. `calculos.js` — `calcularPartida()` acepta tercer parámetro `variante = null`; precio base = `variante?.precio_m2 ?? T.vidrio[tipo] ?? 380`
5. `Materiales.jsx` — página admin nueva: sidebar con 5 categorías fijas + panel con grid de tarjetas de variantes (imagen, nombre, precio inline editable, toggle activa, eliminar) + formulario inline para crear variante con upload de imagen (max 2MB)
6. `Cotizador.jsx` — `FormPartida` recibe `categorias` como prop; muestra chips de variante después del selector de tipo; al cambiar tipo se resetea la variante; variante seleccionada se pasa a `calcularPartida` como tercer argumento
7. `App.jsx` — ruta `/materiales` (admin-only) + link "Materiales" en submenú ⚙
8. `global.css` — añadidas clases `.mat-*` (layout, sidebar, panel, grid de tarjetas, form nueva variante, chip con imagen)

### Sesión 6 — Variantes conectadas al cotizador interno y formulario público
1. `calculos.js` — exportado `TIPO_A_NOMBRE` (mapeo tipo string → nombre de categoría, reutilizable por cualquier página)
2. `Cotizador.jsx` — botón "Agregar partida" bloqueado cuando la categoría tiene variantes activas y ninguna está seleccionada; nombre de variante visible en `FilaPartida` (columna descripción, junto al tipo y espesor)
3. `Solicitar.jsx` — integración completa de variantes: `useCategorias()` + `TIPO_A_NOMBRE` para derivar variantes sin queries adicionales; grid de tarjetas 3 columnas (imagen 80×80 con `object-fit:cover` o placeholder con inicial, nombre, precio/m²); `calcPreview()` pasa `varianteObj` a `calcularPartida()`; reset de variante al cambiar tipo; botón "Agregar producto" bloqueado sin variante seleccionada; `variante_id`/`variante_nombre` guardados en cada partida de la solicitud
4. `global.css` — 10 clases nuevas `.sol-variante-*` y `.sol-variantes-grid` para el selector de variantes del formulario público

### Sesión 4 — Correcciones de robustez y renombrado de rutas
1. `tarifasDB.obtenerActivas()` — cambiado de `.single()` a `.maybeSingle()` para evitar error 406 cuando hay múltiples filas con `activa = true`
2. `calculos.js` — añadido `export const TARIFAS_DEFAULT` como única fuente de valores por defecto de tarifas
3. `useTarifas()` — ya no se queda en `loading: true` si Supabase devuelve `null` o lanza error; cae a `TARIFAS_DEFAULT` en ambos casos
4. `Solicitar.jsx` — eliminado el objeto `TARIFAS` local duplicado; ahora importa `TARIFAS_DEFAULT` de `calculos.js`
5. Rutas renombradas: `/solicitar` (formulario público) → `/cotizar`; `/cotizar` y `/cotizar/:id` (cotizador interno) → `/nueva-cotizacion` y `/nueva-cotizacion/:id`
6. Actualizado `App.jsx`, `Historial.jsx`, `CotizacionDetalle.jsx` y `Cotizador.jsx` con las nuevas rutas

---

*Para continuar el trabajo, leer este archivo completo antes de hacer cualquier cambio al código.*
