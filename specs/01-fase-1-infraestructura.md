# SPEC 01 — Infraestructura base, dependencias y configuración de entrada

> **Status:** Implemented  
> **Depends on:** None  
> **Date:** 2026-10-07  
> **Objective:** Establecer los servicios locales en Docker, aprovisionar el catálogo de dependencias estricto para ESM y asegurar el bootstrap con validación global, Helmet, Pino Logger y Swagger condicional.

---

## 1. Contexto

Este spec establece la base operativa para la plantilla B2B Monolito Modular en NestJS 12. Prepara el entorno local de desarrollo con Docker Compose (PostgreSQL 15 y Redis 7), estandariza las dependencias de producción y desarrollo bajo `pnpm` y ESM estricto (`"moduleResolution": "nodenext"`), y refuerza el arranque de la aplicación respetando las restricciones no negociables del SRS.

---

## 2. Scope

**In:**

- Instalación y bloqueo de dependencias de producción y desarrollo requeridas en `package.json` mediante `pnpm`.
- Creación de `.dockerignore`, `docker-compose.yml` (PostgreSQL 15-alpine y Redis 7-alpine) y `Dockerfile` multi-stage optimizado para `pnpm`.
- Creación de archivos `.env.example` y `.env` con variables base para servidor, base de datos, Redis y JWT.
- Integración de `ConfigModule` y `LoggerModule` (`nestjs-pino`) en `src/app.module.ts`, preservando la instrumentación de `ObserveModule`.
- Refuerzo de `src/main.ts`:
  - `ValidationPipe` global (`whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`).
  - Cabeceras de seguridad con `helmet()`.
  - Habilitación de `app.enableShutdownHooks()`.
  - Conexión del logger estructurado `nestjs-pino`.
  - CORS condicional mediante la variable `ALLOWED_ORIGINS`.
  - Exposición de documentación Swagger (`/api/docs`) condicionada exclusivamente a `process.env.NODE_ENV === 'development'`.
- Verificación y ajuste del fixture de pruebas en `test/app.e2e-spec.ts`.

**Out of scope (for future specs):**

- Modelado relacional y esquema en `prisma/schema.prisma` (SPEC 02).
- Prisma Client Extension con `AsyncLocalStorage` para multitenencia y Soft Delete (SPEC 02).
- Implementación de controladores, servicios de dominio y guards específicos (`AuthModule`, `UsersModule`, etc.) (SPECs 03 a 07).

---

## 3. Data model

Este feature no introduce nuevos modelos ni tablas en base de datos. La definición del esquema relacional y las extensiones de cliente Prisma están reservadas para el SPEC 02.

---

## 4. Implementation plan

1. **Aprovisionar dependencias en `package.json`:**
   - Instalar dependencias de producción:
     `@prisma/client`, `argon2`, `helmet`, `@nestjs/throttler`, `nestjs-throttler-storage-redis`, `otplib`, `qrcode`, `ua-parser-js`, `@nestjs-modules/mailer`, `nodemailer`, `redis`, `@nestjs/cache-manager`, `cache-manager`, `nestjs-i18n`, `multer`, `file-type`, `bullmq`, `@nestjs/bullmq`, `nestjs-pino`, `pino-http`, `pino-pretty`, `@nestjs/passport`, `passport`, `passport-jwt`, `passport-local`, `passport-google-oauth20`, `class-validator`, `class-transformer`, `@nestjs/config`, `@nestjs/swagger`, `@nestjs/terminus`.
   - Instalar dependencias de desarrollo:
     `prisma`, `@types/passport-jwt`, `@types/passport-local`, `@types/passport-google-oauth20`, `@types/nodemailer`, `@types/multer`, `@types/qrcode`.
   - Verificar compatibilidad de resolución con `pnpm exec tsc --noEmit`.

2. **Crear configuración de entorno (`.env.example` y `.env`):**
   - Declarar variables:
     - Servidor: `NODE_ENV=development`, `PORT=3000`, `ALLOWED_ORIGINS="http://localhost:3000"`
     - Base de datos: `DB_USER=postgres`, `DB_PASSWORD=postgres`, `DB_NAME=auth_db`, `DATABASE_URL="postgresql://postgres:postgres@localhost:5432/auth_db?schema=public"`
     - Redis: `REDIS_HOST=localhost`, `REDIS_PORT=6379`
     - JWT: `JWT_ACCESS_SECRET="super-secret-access-key"`, `JWT_REFRESH_SECRET="super-secret-refresh-key"`, `JWT_ACCESS_EXPIRES_IN="15m"`, `JWT_REFRESH_EXPIRES_IN="90d"`

3. **Construir infraestructura de contenedores:**
   - Crear `.dockerignore` excluyendo `.git`, `node_modules`, `dist`, `.env` y directorios de cobertura.
   - Crear `docker-compose.yml` configurando los servicios `db` (`postgres:15-alpine`) y `redis` (`redis:7-alpine`) con volúmenes nombrados (`postgres_data`, `redis_data`) y mapeo de puertos 5432 y 6379.
   - Crear `Dockerfile` de producción en 3 etapas (`deps`, `builder`, `production`) usando `node:20-alpine`, `corepack enable pnpm` y ejecutando `CMD ["node", "dist/main.js"]` (sin comandos de migración directa en el arranque).

4. **Configurar módulos globales en `src/app.module.ts`:**
   - Importar `ConfigModule.forRoot({ isGlobal: true, envFilePath: '.env' })`.
   - Importar `LoggerModule.forRoot(...)` configurando `pinoHttp` con formateo legible (`pino-pretty`) en entornos diferentes a `production`.
   - Conservar la inicialización y proveedores de `ObserveModule`.
   - Asegurar extensiones de importación relativa en `.js` (`import { ... } from './...js'`).

5. **Asegurar bootstrap de la aplicación en `src/main.ts`:**
   - Enlazar el logger global mediante `app.useLogger(app.get(Logger))` de `nestjs-pino`.
   - Habilitar `app.enableShutdownHooks()`.
   - Registrar middleware `helmet()`.
   - Configurar CORS con lista blanca parsed desde `process.env.ALLOWED_ORIGINS`.
   - Registrar `ValidationPipe` global con `{ whitelist: true, forbidNonWhitelisted: true, transform: true }`.
   - Configurar `SwaggerModule` bajo la condición estricta `process.env.NODE_ENV === 'development'` apuntando a `/api/docs`.
   - Mantener el paso de `ObserveInstrument` en `NestFactory.create`.

6. **Ajuste de suite de pruebas (`test/app.e2e-spec.ts`):**
   - Asegurar que `app.e2e-spec.ts` compile y pase sin interferencias del logger de Pino o las variables de configuración.
   - Ejecutar la secuencia de verificación completa: `pnpm lint`, `pnpm exec tsc --noEmit`, `pnpm test`, `pnpm run test:e2e`.

---

## 5. Acceptance criteria

- [x] `pnpm install` resuelve todas las dependencias sin conflictos de árbol o pares.
- [x] `pnpm exec tsc --noEmit` y `pnpm run build` completan sin advertencias ni errores en `dist/main.js`.
- [x] `docker compose config` valida el archivo `docker-compose.yml` sin errores de esquema.
- [x] `docker compose up -d` arranca los contenedores `auth-template-db` y `auth-template-redis`.
- [x] `GET /api/docs` responde `200 OK` cuando `NODE_ENV=development` y `404 Not Found` cuando `NODE_ENV=production`.
- [x] Un envío HTTP con propiedades no definidas en un DTO devuelve `400 Bad Request` por la política `forbidNonWhitelisted`.
- [x] Las respuestas HTTP incluyen cabeceras de protección generadas por `helmet` (ej. `X-Content-Type-Options: nosniff`).
- [x] Las trazas de consola se emiten a través de `nestjs-pino`.
- [x] Las suites `pnpm test` y `pnpm run test:e2e` se ejecutan y finalizan al 100% en verde.

---

## 6. Decisions

- **Yes:** Usar `pnpm` en el Dockerfile multi-stage en lugar de `npm`.
  - *Razón:* El repositorio utiliza `pnpm-lock.yaml`; usar npm rompería la reproducibilidad del lockfile.
- **Yes:** Salida de ejecución en `dist/main.js` en lugar de `dist/src/main.js`.
  - *Razón:* `tsconfig.build.json` fija `rootDir: ./src`, aplanando la salida directamente en `dist/`.
- **Yes:** `pino-pretty` solo para `NODE_ENV !== 'production'`.
  - *Razón:* En producción, los logs deben ser JSON puro para ingesta en sistemas de observabilidad externos.
- **No:** `npx prisma migrate deploy` en el CMD del contenedor o en `main.ts`.
  - *Razón:* Prohibición explícita del SRS para evitar condiciones de carrera y locks en réplicas simultáneas.

---

## 7. Risks

| Riesgo | Mitigación |
| :--- | :--- |
| Interferencia de tipos ESM/CJS con `pino-http` o `ua-parser-js` | Uso de `moduleResolution: nodenext` con `esModuleInterop: true` e importaciones relativas con extensión `.js`. |
| Fallo en pruebas E2E al requerir variables de entorno o dependencias de logger | `ConfigModule` configurado como `isGlobal: true` y pasaje del logger mock o buffer en tests si se requiere. |

---

## 8. What is not in this spec

- Creación de tablas, modelos o migraciones Prisma (`specs/02-...`).
- Lógica de autenticación, generación de JWT o hashing de contraseñas (`specs/03-...`).
- Tareas programadas en Redis con BullMQ (`specs/06-...`).

---

### Archivo de configuración inicial de workflow

Guardar en `specs/.spec-config.yml`:

```yaml
# spec workflow configuration
#
# AutoCreateBranch — controls whether /spec-impl creates the git branch automatically.
#   true  (default) → /spec-impl creates and switches to spec-NN-slug without asking
#   false           → /spec-impl asks for [y/N] confirmation before creating the branch
AutoCreateBranch: true
```

---

**Resumen de entrega:**
- **Ruta de la especificación:** `specs/01-infraestructura-base-y-configuracion.md`
- **Estado:** `Draft` (cámbialo a `Approved` tras tu revisión manual).
- **Configuración:** Se incluye `specs/.spec-config.yml` con `AutoCreateBranch: true`.
- **Siguiente paso:** Una vez aprobado, ejecuta `/spec-impl 01-infraestructura-base-y-configuracion` para proceder con su implementación sistemática.
