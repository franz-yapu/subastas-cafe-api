# Subasta de café — API

Backend de una **plataforma de subastas en línea de café de especialidad** con pujas en tiempo real.
Pensada para instituciones que subastan lotes de café de productores: el administrador publica los
lotes, los compradores pujan en vivo desde el navegador, y al cerrar el sistema adjudica, notifica
por correo y registra la transacción.

Interfaz web (Angular): [`franz-yapu/frontend-c`](https://github.com/franz-yapu/frontend-c)

## Qué resuelve

- **Pujas en vivo y justas.** Todos los compradores ven la puja más alta al instante (WebSockets) y
  reciben aviso cuando alguien los supera.
- **Cierre automático.** Un temporizador de servidor cierra la subasta a la hora fijada y adjudica cada
  lote al ganador, sin que nadie tenga que estar presente.
- **Anti-sniping.** Si entra una puja en los últimos minutos, el cierre se extiende (minutos y activación
  configurables por subasta).
- **Marca blanca.** Cada institución configura sus colores, logo y textos desde un panel, sin tocar código.

## Características

| Área | Qué hay |
| --- | --- |
| Tiempo real | Socket.io en el namespace `/bids`: salas por subasta, eventos `newBid`, `outbid`, `auctionExtended`, `auctionClosed`, cambios de lotes y de fecha de cierre |
| Subastas | Estados de subasta, lotes de café con calidad y proceso, incremento mínimo, extensión automática, cierre programado (`@nestjs/schedule`) |
| Pujas | Validación de monto mínimo, desempate determinista, advertencia de latencia alta al cliente |
| Usuarios y roles | JWT con Passport; roles `ADMIN`, `BUYER`, `SELLER` (solo revisa) y `GUEST`; bitácora de acciones de usuario |
| Correo | Nodemailer con plantillas HTML (bienvenida, restablecer contraseña) y avisos de subasta |
| Notificaciones | Registro de tokens de push para la app móvil |
| i18n | Español e inglés (`nestjs-i18n`), con textos editables desde el panel (*translation overrides*) |
| Marca blanca | Módulo de branding con historial de cambios |
| Documentación | Swagger en `/api` |
| Validación | `class-validator` en todos los DTO |

## Decisiones de diseño que vale la pena mirar

- **Sin doble adjudicación.** Las pujas a un mismo lote se serializan con un *advisory lock* de
  PostgreSQL por lote. Dos pujas simultáneas nunca se aceptan a la vez.
- **Desempate estable.** A igual monto gana siempre la primera en el tiempo
  (`amount DESC, createdAt ASC, id ASC`).
- **Una sola subasta activa, garantizado por la base de datos.** Un índice único parcial
  (`one_active_auction`) lo impone a nivel de BD, no solo en la aplicación.
- **Dinero en `Decimal(12,2)`**, nunca en flotantes.
- **Índices a medida** para las consultas calientes (ganador/precio por lote y últimas pujas).
- **Un solo origen del secreto JWT** compartido por la API REST y el gateway de WebSockets.

## Stack

NestJS 11 · TypeScript · Prisma 6 · PostgreSQL · Socket.io · Passport/JWT · Swagger ·
nestjs-i18n · Nodemailer · `@nestjs/schedule` · Jest · Docker

## Puesta en marcha

Requisitos: Node 20+, PostgreSQL 15+.

```bash
npm install
cp .env.example .env        # rellena DATABASE_URL y JWT_SECRET como mínimo
npx prisma migrate deploy   # crea el esquema
npx prisma db seed          # roles y datos de demostración
npm run start:dev           # http://localhost:3000  ·  Swagger en /api
```

Genera un `JWT_SECRET` fuerte (es **obligatorio en producción**):

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

### Con Docker

```bash
docker build -t subasta-backend .
docker run --env-file .env -p 3000:3000 subasta-backend
```

El contenedor se construye en dos etapas (build y runtime) con Node 20 Alpine.

## Pruebas

```bash
npm test                  # unitarias
npm run test:e2e          # extremo a extremo
RUN_CONCURRENCY_TEST=1 DATABASE_URL=<base de pruebas> npm run test:concurrency
```

La prueba de **concurrencia** corre contra un PostgreSQL real (sin mocks) y verifica el *advisory lock*
y el desempate. Está desactivada por defecto para que nunca toque una base de producción por accidente.

## Estructura

```
src/
  auction/    subastas, temporizador y cierre automático
  bid/        pujas (REST) y gateway de WebSockets
  caffee-lot/ lotes de café
  transaction/ adjudicaciones
  users/ auth/ roles, JWT, bitácora
  email/ notifications/ correo y push
  branding/ translation/ i18n  marca blanca y textos
prisma/       esquema, migraciones y seed
test/         e2e y concurrencia
```

## Estado

Versión **1.0.0** (etiqueta `v1.0.0`), desplegada con Coolify. Rama de trabajo actual: `main`.
Otras ramas (`caritas`, `tecab`, `cafe-pro`, `version_1`, `dev`) conservan ediciones anteriores
configuradas para instituciones concretas.

## Licencia

Todos los derechos reservados © Franz Yapu. El código se publica para que pueda revisarse;
no se concede licencia de uso o redistribución sin autorización escrita.
