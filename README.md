# PC Selector

Tienda de componentes PC con builder inteligente, asistente IA y pagos reales con Stripe (https://stripe.com/es-us/payments).

---

## Stack

- **Next.js 16** (App Router, React 19)
- **Prisma 6** + **Supabase** (PostgreSQL gestionado)
- **Stripe** (Payment Intents + Checkout Sessions + Webhooks via Node.js SDK)
- **Vercel** (hosting recomendado)
- **Tailwind CSS v4** · **TypeScript**

---

## Cómo funcionan los webhooks de Stripe

> **No se necesita Stripe CLI en producción.**

El Stripe CLI es solo una herramienta de desarrollo local que actúa como proxy para reenviar eventos de Stripe a tu máquina. En Vercel (o cualquier hosting con URL pública), Stripe llama directamente al endpoint `/api/stripe/webhook` de tu app.

La autenticación se hace con el **Node.js SDK** (`stripe.webhooks.constructEvent`), que verifica la firma criptográfica de cada request usando `STRIPE_WEBHOOK_SECRET`. Esto es todo lo que necesitas en producción.

| Entorno | Cómo llegan los eventos | STRIPE_WEBHOOK_SECRET |
|---|---|---|
| **Producción** (Vercel) | Stripe → tu endpoint directamente | Signing secret del Dashboard |
| **Desarrollo local** | Stripe CLI → localhost | whsec_ que imprime la CLI |

---

## Configuración local

### 1. Instalar dependencias

```bash
npm install
```

### 2. Variables de entorno

Este proyecto usa **dos archivos** de entorno por un motivo concreto:

| Archivo | Quién lo lee | Para qué |
|---|---|---|
| `.env` | Prisma CLI, Docker Compose, Next.js | Variables de BD y Stripe |
| `.env.local` | Solo Next.js | Sobreescritura puntual (opcional) |

> **¿Por qué no solo `.env.local`?**  
> `prisma migrate`, `prisma db seed` y `docker-compose` **no leen `.env.local`** — eso es exclusivo de Next.js. Sin `.env`, los comandos de Prisma fallan con `P1012: Environment variable not found`.

Crea tu `.env` copiando el ejemplo:

```bash
cp .env.example .env
```

Edita `.env` con tus valores reales. No toques `.env.local` salvo que necesites sobreescribir algo solo para Next.js (raro en desarrollo local).

### 3. Base de datos y Stripe CLI con Docker

El `docker-compose.yml` levanta dos servicios a la vez:
- **postgres** — PostgreSQL 16
- **stripe-cli** — reenvía eventos de Stripe a tu servidor local

**Primer arranque — autenticación de Stripe:**

```bash
# Asegúrate de exportar tu clave antes de ejecutar docker-compose
export STRIPE_SECRET_KEY="sk_test_..."

# Levanta solo la CLI para hacer el login interactivo
docker-compose run --rm stripe-cli login
```

Abrirá una URL en el navegador. Autorizala con tu cuenta de Stripe.
Las credenciales quedan guardadas en el volumen `stripe_config` y no tendrás que repetir este paso.

**Arranque normal (a partir de ahora):**

```bash
docker-compose up -d
```

Al arrancar, la CLI imprime el webhook signing secret:
```
> Ready! Your webhook signing secret is whsec_xxxxxxxxxxxxxxxx
```
Cópialo en `.env.local` como `STRIPE_WEBHOOK_SECRET`.

En `.env.local` usa las variables de Docker local:
```
DATABASE_URL="postgresql://pcselector:pcselector@localhost:5432/pcselector?schema=public"
DATABASE_URL_UNPOOLED="postgresql://pcselector:pcselector@localhost:5432/pcselector?schema=public"
STRIPE_SECRET_KEY="sk_test_..."
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY="pk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."  # el que imprimió la CLI al arrancar
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

> **Supabase en local:** si prefieres apuntar a Supabase también en desarrollo,
> comenta las variables de Docker en `.env.local` y usa las dos strings de Supabase
> (ver `.env.example` para el formato exacto).

**Ver los eventos en tiempo real:**
```bash
docker-compose logs -f stripe-cli
```

### 4. Crear tablas y poblar el catálogo

```bash
# Crea las tablas (usa DATABASE_URL_UNPOOLED automáticamente)
npx prisma migrate dev --name init

# Inserta todos los productos en la BD
npm run db:seed
```

### 5. Arrancar Next.js

```bash
npm run dev
```

---

## Scripts disponibles

| Comando | Descripción |
|---|---|
| `npm run dev` | Servidor de desarrollo |
| `npm run build` | Build de producción |
| `npm run start` | Servidor de producción |
| `npm run db:migrate` | Nueva migración en desarrollo |
| `npm run db:migrate:prod` | Aplica migraciones en producción |
| `npm run db:seed` | Puebla la BD con el catálogo |
| `npm run db:studio` | Abre Prisma Studio |

---

## Despliegue en producción (Vercel + Supabase)

### 1. Configura Supabase

1. Crea el proyecto en [supabase.com](https://supabase.com)
2. Anota las dos connection strings (Settings → Database)

### 2. Despliega en Vercel

1. Conecta el repo en [vercel.com](https://vercel.com)
2. En **Settings → Environment Variables**, añade todas las variables:

```
DATABASE_URL              → string con pgbouncer (puerto 6543)
DATABASE_URL_UNPOOLED     → string directa (puerto 5432)
STRIPE_SECRET_KEY         → sk_live_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY → pk_live_...
STRIPE_WEBHOOK_SECRET     → whsec_... (del paso 3)
NEXT_PUBLIC_APP_URL       → https://tu-app.vercel.app
```

3. En **Settings → General → Build Command**, configura:
```
prisma migrate deploy && next build
```
Esto aplica las migraciones automáticamente en cada deploy.

### 3. Registra el webhook en Stripe

1. Dashboard de Stripe → **Developers → Webhooks → Add endpoint**
2. URL del endpoint: `https://tu-app.vercel.app/api/stripe/webhook`
3. Eventos a escuchar:
   - `payment_intent.succeeded`
   - `payment_intent.payment_failed`
   - `checkout.session.completed`
4. Copia el **Signing secret** (`whsec_...`) → ponlo en Vercel como `STRIPE_WEBHOOK_SECRET`

### 4. Poblar la BD (una sola vez)

Una vez desplegado y con las migraciones aplicadas, ejecuta el seed apuntando a Supabase:

```bash
# Con las variables de prod en tu entorno local, o desde Vercel CLI
DATABASE_URL_UNPOOLED="postgresql://..." npm run db:seed
```

---

## Arquitectura de la BD

```
Supabase (PostgreSQL)
│
├── Puerto 6543 (PgBouncer pooler)
│   └── DATABASE_URL → Prisma en runtime (Next.js API routes, Server Components)
│
└── Puerto 5432 (conexión directa)
    └── DATABASE_URL_UNPOOLED → prisma migrate deploy (solo en build/deploy)
```

---

## Estructura del proyecto

```
app/
  api/
    assistant/route.ts             ← Asistente IA (RAG desde BD)
    checkout/route.ts              ← Stripe Checkout Sessions
    products/
      route.ts                    ← GET catálogo con filtros
      by-type/route.ts            ← GET productos por tipo (PC Builder)
    stripe/
      payment-intent/route.ts     ← Stripe Payment Intents
      webhook/route.ts            ← Webhook (Node.js SDK, sin Stripe CLI)
    orders/
      confirm/route.ts            ← Consulta estado de pedido
  checkout/
    page.tsx                      ← Pago con Stripe Elements
    success/page.tsx              ← Confirmación con datos reales
  builder/                        ← PC Builder con compatibilidad en tiempo real
  shop/                           ← Tienda (datos desde Supabase)
  product/[slug]/                 ← Páginas de producto
  page.tsx                        ← Home

lib/
  prisma.ts                       ← Cliente Prisma singleton
  stripe.ts                       ← Cliente Stripe + verificación de webhooks (SDK)
  db-to-types.ts                  ← Conversor Prisma → tipos del dominio
  catalog.ts                      ← Catálogo estático (solo para el seed)
  types.ts                        ← Tipos TypeScript compartidos
  compatibility.ts                ← Lógica de compatibilidad de builds
  rag.ts                          ← Asistente IA (async, lee de Supabase)

components/
  pc-builder.tsx                  ← Builder client (carga productos vía API)
  ai-assistant.tsx                ← Chat flotante del asistente
  store-provider.tsx              ← Carrito en localStorage

prisma/
  schema.prisma                   ← Schema con url + directUrl para Supabase
  seed.ts                         ← Seed inicial del catálogo
```
