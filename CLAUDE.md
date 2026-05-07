# CLAUDE.md — Antike Cotizador

**Antike** — sistema de cotización para empresa mexicana de espejos, vidrio y marcos metálicos. Stack: React 18 + Vite 5 + React Router 6 + Supabase + Vercel Serverless (Python PDF). CSS custom en `src/styles/global.css`, sin Tailwind.

---

## Roles y acceso

| Rol | Rutas | Auth |
|---|---|---|
| `admin` | Todo | Email/Google |
| `vendedor` | `/nueva-cotizacion`, `/historial` | Email/Google |
| cliente externo | `/cotizar` (pública) | Sin cuenta |

Rol en `public.users.rol`. Promover: `UPDATE public.users SET rol = 'admin' WHERE email = '...'`.

---

## Archivos críticos

| Archivo | Qué hace |
|---|---|
| `src/lib/calculos.js` | **Fuente única de verdad de precios.** `calcularPartida(form, tarifas, variante?)`, `validarPartida()`, `TARIFAS_DEFAULT`, `TIPO_A_NOMBRE`. Lo usan el frontend y `api/pdf.py`. |
| `src/lib/supabase.js` | Cliente Supabase único + todos los helpers: `clientesDB`, `cotizacionesDB`, `partidasDB`, `tarifasDB`, `solicitudesDB`, `categoriasDB`, `variantesDB`, `usuariosDB`, `pdfDB`. |
| `src/lib/renderEspejo.js` | `renderEspejo(canvas, opciones)` — render fotorrealista (sombra, marco por stroke, bisel, glow LED exterior). Usado en `Cotizador.jsx` y `Solicitar.jsx`. |
| `src/hooks/index.js` | `useAuth()`, `useClientes()`, `useCotizaciones()`, `useTarifas()`, `useCategorias()`. |
| `src/pages/Solicitar.jsx` | Wizard público 6 pasos. Ruta `/cotizar`. Layout 60/40 canvas/config. Envía campos planos a `solicitudesDB.crear()`. |
| `api/crear-usuario.js` | Serverless Node. Crea usuarios con `admin.createUser()` (sin sobreescribir sesión del admin). Requiere JWT de admin. |
| `api/pdf.py` | Serverless Python. Genera PDF con ReportLab, sube a Storage, retorna URL firmada. `generar_cotizacion.py` debe estar en la raíz. |

---

## Reglas de arquitectura

- **Un solo cliente Supabase** — `src/lib/supabase.js`. Nunca crear una segunda instancia.
- **Precios solo en `calculos.js`** — si cambian, actualizar `TARIFAS_DEFAULT` ahí únicamente.
- **Tarifas versionadas** — cada guardar desactiva la versión anterior e inserta nueva. Nunca sobreescribir.
- **`solicitudesDB.crear()` sin `.select()`** — RLS de `solicitudes` solo da `INSERT` al rol `anon`. Agregar `.select()` causa error. No "corregir".
- **Folio generado en DB** — `supabase.rpc('siguiente_folio', { p_user_id: uid })`.
- **Creador en dos queries** — `cotizaciones.user_id` → `auth.users`, no joineable por PostgREST. Se resuelve client-side con `.in('id', ids)` a `public.users`.
- **Toasts con color hardcoded** — `background: #1e1e1c`. No cambiar a variable CSS o el texto queda invisible en dark mode.
- **`calcularPartida` acepta variante opcional** — `calcularPartida(form, tarifas, variante = null)`. Si se pasa variante con `precio_m2`, reemplaza la tarifa base. Retrocompatible.
- **Variantes de material** — `categoriasDB.listar()` devuelve categorías con variantes embebidas. Derivar variantes activas en render: `categorias.find(c => c.nombre === TIPO_A_NOMBRE[tipo])?.variantes_producto.filter(v => v.activa)`.

---

## Lógica de negocio — fórmulas de precio

```
Área:
  Rectángulo:  (ancho/1000) × (alto/1000)
  Circular:    π × (ancho/2000) × (alto/2000)
  Irregular/arco: ((ancho + bbox_mm)/1000) × ((alto + bbox_mm)/1000)

Perímetro: 2 × ((ancho + alto) / 1000)  [siempre, sin importar figura]

Costos:
  Vidrio:   tarifa_vidrio[tipo] × área
  Procesos por ml: tarifa_proc_ml[proc] × perímetro  (bisel, chaflán, pulido, resaques)
  Procesos por m²: tarifa_proc_area[proc] × área     (templado, backing)
  Barrenos: tarifa_barreno × n_barrenos
  Marco:    perímetro × (tarifa_marco_ml[mat] + tarifa_acabado[acab]) + tarifa_armado
  LED:      tarifa_led[tipo] × perímetro

Multiplicadores de precio venta:
  Vidrio+procesos ×2.0 | Marco ×2.5 | LED ×2.5 | Costo fijo ×1.5

Total: subtotal + indirectos (prorr. por m²) + transporte_esp + riesgo% − descuento%
```

**Restricciones:**
- Bisel solo en espesor ≥ 6 mm
- Latón solo acabado cepillado
- LED solo en espejos (no vidrio)

---

## Variables de entorno

```
# Frontend (VITE_ prefix — expuestas al navegador)
VITE_SUPABASE_URL
VITE_SUPABASE_ANON_KEY

# Serverless only — NUNCA con prefijo VITE_
SUPABASE_URL
SUPABASE_SERVICE_KEY   ⚠️ bypasea RLS — solo en api/pdf.py y api/crear-usuario.js
```

---

## Schema de base de datos

```sql
users          (id uuid → auth.users, nombre, email, empresa, tel, rol DEFAULT 'vendedor')
clientes       (id, user_id, nombre, empresa, tel, email, obra, notas)
cotizaciones   (id, user_id, cliente_id, folio, version, estado, obra, total,
                m2_total, n_partidas, peso_total, n_rejas, c_empaque, c_envio,
                c_instalacion, c_transporte_esp, pct_riesgo, pct_descuento, notas, fecha)
partidas       (id, cotizacion_id, user_id, orden, descripcion, tipo, espesor,
                ancho, alto, cantidad, figura, procesos text[], n_barrenos,
                marco, marco_acabado, led, variante_id uuid FK nullable,
                area_unit, peso_unit, perimetro, c_vidrio, c_marco, c_led, c_fijo,
                costo_unit, precio_unit, precio_total)
tarifas        (id, user_id, version int, activa bool, datos jsonb, notas)
solicitudes    (id, nombre, empresa, email, tel, estado DEFAULT 'pendiente', created_at
                + columnas planas del bespoke — ver migración pendiente abajo)
categorias_producto  (id, nombre, orden)
variantes_producto   (id, categoria_id, nombre, precio_m2, imagen_url, activa)
pdf_exports    (id, cotizacion_id, user_id, storage_path, url_firmada)
```

Trigger `handle_new_user` crea fila en `public.users` + fila en `tarifas` al registrarse. Usar `jsonb_build_object()` para el JSON — nunca strings literales (CRLF en Windows rompe el JSON).

---

## ⚠️ SQL pendiente de ejecutar en Supabase

### Políticas RLS (ejecutar si no están activas)

```sql
CREATE POLICY "admin ve todos los clientes" ON public.clientes FOR SELECT TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');
CREATE POLICY "admin ve todas las cotizaciones" ON public.cotizaciones FOR SELECT TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');
CREATE POLICY "authenticated read users" ON public.users FOR SELECT TO authenticated USING (true);
CREATE POLICY "admin update rol" ON public.users FOR UPDATE TO authenticated
  USING ((SELECT rol FROM public.users WHERE id = auth.uid()) = 'admin');
```

### Migración `003_solicitudes_bespoke.sql` — REQUERIDA para `/cotizar` en producción

```sql
ALTER TABLE public.solicitudes
  DROP COLUMN IF EXISTS partidas,
  DROP COLUMN IF EXISTS total,
  ADD COLUMN IF NOT EXISTS forma          text,
  ADD COLUMN IF NOT EXISTS ancho          numeric,
  ADD COLUMN IF NOT EXISTS alto           numeric,
  ADD COLUMN IF NOT EXISTS categoria_id   uuid REFERENCES categorias_producto(id) ON DELETE SET NULL,
  ADD COLUMN IF NOT EXISTS variante_id    uuid REFERENCES variantes_producto(id) ON DELETE SET NULL,
  ADD COLUMN IF NOT EXISTS variante_nombre text,
  ADD COLUMN IF NOT EXISTS procesos       text[],
  ADD COLUMN IF NOT EXISTS n_barrenos     int,
  ADD COLUMN IF NOT EXISTS marco          text,
  ADD COLUMN IF NOT EXISTS marco_acabado  text,
  ADD COLUMN IF NOT EXISTS led            text;
```

---

## Funcionalidades pendientes (alta prioridad)

- **Envío de email** al recibir solicitud de cliente (Resend desde Vercel function)
- **"Convertir solicitud en cotización"** — botón en `/solicitudes` que pre-rellena `/nueva-cotizacion`
- **Deploy a Vercel** — configurar las 4 variables de entorno en Vercel Dashboard

---

## Comandos de desarrollo

```bash
npm run dev          # frontend solo → http://localhost:5173
vercel dev           # frontend + serverless → http://localhost:3000
npm run build
npx vercel --prod
```

> En Windows si `npm` no se reconoce: `& "C:\Program Files\nodejs\npm.cmd" <comando>`
