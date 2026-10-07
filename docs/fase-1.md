# 📐 Plan Técnico de Ejecución: Fase 1 - Infraestructura Base, Dependencias y Configuración de Entrada

> **Objetivo:** Establecer la infraestructura de servicios locales (PostgreSQL 15 y Redis 7), aprovisionar el catálogo estricto de dependencias para ESM/TypeScript y blindar el punto de entrada de la aplicación (`src/main.ts`) con políticas globales de seguridad, validación y observabilidad.

---

## 1. Alcance (Scope)

**Dentro del alcance:**
- Instalación de dependencias de producción y desarrollo especificadas en el SRS compatibles con Node.js 24, ESM (`"type": "module"`) y `pnpm`.
- Creación de `docker-compose.yml` con servicios de PostgreSQL 15-alpine y Redis 7-alpine con volúmenes persistentes y redes dedicadas.
- Creación de `Dockerfile` multi-stage optimizado para `pnpm` (etapas: `deps`, `builder`, `production`) y su respectivo `.dockerignore`.
- Creación de archivos de variables de entorno `.env.example` y `.env` para desarrollo local.
- Configuración de `src/app.module.ts` integrando `ConfigModule` y `LoggerModule` de `nestjs-pino`.
- Reconfiguración de `src/main.ts`:
  - `ValidationPipe` global con `whitelist: true`, `forbidNonWhitelisted: true` y `transform: true`.
  - Integración de `helmet` para cabeceras HTTP seguras.
  - Activación de `app.enableShutdownHooks()`.
  - Reemplazo del logger nativo por `Logger` de `nestjs-pino`.
  - Exposición de Swagger (`@nestjs/swagger`) condicionada exclusivamente a `process.env.NODE_ENV === 'development'`.
  - Configuración de CORS basada en la variable `ALLOWED_ORIGINS`.

**Fuera del alcance (Postergado a siguientes fases):**
- Definición de tablas y relaciones en `prisma/schema.prisma` (Fase 2).
- Prisma Client Extension con `AsyncLocalStorage` para multitenencia y Soft Delete (Fase 2).
- Módulos de dominio (`AuthModule`, `UsersModule`, `OrganizationsModule`, etc.) y guards específicos (Fases 3 a 6).

---

## 2. Mapa de Archivos Afectados

- `package.json` — **[Modificación]** — *Añadir dependencias de producción (`@prisma/client`, `argon2`, `helmet`, `bullmq`, `nestjs-pino`, etc.) y devDependencies (`prisma`, `@nestjs/swagger`, tipos).*
- `.env.example` — **[Creación]** — *Plantilla de variables de entorno requeridas por la aplicación y contenedores.*
- `.env` — **[Creación]** — *Variables de entorno para desarrollo local.*
- `.dockerignore` — **[Creación]** — *Exclusión de `node_modules`, `dist`, `.git` y artefactos temporales.*
- `docker-compose.yml` — **[Creación]** — *Definición de servicios PostgreSQL 15-alpine y Redis 7-alpine.*
- `Dockerfile` — **[Creación]** — *Construcción multi-stage de producción adaptada a `pnpm` y ESM.*
- `src/app.module.ts` — **[Modificación]** — *Importación e inicialización de `ConfigModule` y `LoggerModule` de `nestjs-pino`.*
- `src/main.ts` — **[Modificación]** — *Configuración de pipes de validación, Helmet, Logger Pino, CORS, shutdown hooks y Swagger condicional.*
- `test/app.e2e-spec.ts` — **[Modificación]** — *Adaptación del fixture de prueba para compatibilidad con `LoggerModule` y `ConfigModule` si es requerido.*

---

## 3. Secuencia de Implementación Paso a Paso

### 1. Gestión de Dependencias (`package.json`)
- **Acción:** Instalar las dependencias exactas usando `pnpm`:
  - **Dependencias de producción:**
    - Persistencia & Cache: `@prisma/client`, `redis`, `@nestjs/cache-manager`, `cache-manager`
    - Criptografía & Auth: `argon2`, `@nestjs/passport`, `passport`, `passport-jwt`, `passport-local`, `passport-google-oauth20`, `otplib`, `qrcode`
    - Seguridad & Rate Limit: `helmet`, `@nestjs/throttler`, `nestjs-throttler-storage-redis`
    - Detección de Dispositivos: `ua-parser-js`
    - Mailer: `@nestjs-modules/mailer`, `nodemailer`
    - i18n & Uploads: `nestjs-i18n`, `multer`, `file-type`
    - Background Jobs: `bullmq`, `@nestjs/bullmq`
    - Observabilidad: `nestjs-pino`, `pino-http`, `pino-pretty`
    - Validación y Salud: `class-validator`, `class-transformer`, `@nestjs/terminus`, `@nestjs/config`, `@nestjs/swagger`
  - **Dependencias de desarrollo:**
    - `prisma`, `@types/passport-jwt`, `@types/passport-local`, `@types/passport-google-oauth20`, `@types/nodemailer`, `@types/multer`, `@types/qrcode`

### 2. Variables de Entorno (`.env.example` y `.env`)
- **Acción:** Definir las variables requeridas en `.env.example` y generar un `.env` funcional para entorno local:
  - Servidor: `NODE_ENV=development`, `PORT=3000`, `ALLOWED_ORIGINS=http://localhost:3000`
  - Base de datos: `DB_USER=postgres`, `DB_PASSWORD=postgres`, `DB_NAME=auth_db`, `DB_PORT=5432`, `DATABASE_URL="postgresql://postgres:postgres@localhost:5432/auth_db?schema=public"`
  - Redis: `REDIS_HOST=localhost`, `REDIS_PORT=6379`
  - JWT: `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `JWT_ACCESS_EXPIRES_IN="15m"`, `JWT_REFRESH_EXPIRES_IN="90d"`

### 3. Infraestructura Docker (`docker-compose.yml`, `Dockerfile`, `.dockerignore`)
- **Punto de intervención:** Raíz del proyecto
- **Acciones:**
  - Crear `.dockerignore` excluyendo `.git`, `node_modules`, `dist`, `.env`, `coverage`.
  - Crear `docker-compose.yml` con servicios `db` (`postgres:15-alpine`) y `redis` (`redis:7-alpine`), mapeo de puertos `5432:5432` y `6379:6379`, variables de entorno referenciadas y volúmenes con nombre (`postgres_data`, `redis_data`).
  - Crear `Dockerfile` de 3 etapas usando imagen base `node:20-alpine` (o `node:22-alpine` compatible con pnpm):
    - Etapa `deps`: instala pnpm, copia `package.json`, `pnpm-lock.yaml` y ejecuta `pnpm install --frozen-lockfile`.
    - Etapa `builder`: compila el proyecto (`pnpm run build`), genera artefactos Prisma si existen.
    - Etapa `production`: copia `node_modules` de producción y `dist`, define `ENV NODE_ENV=production`, expone puerto `3000` y ejecuta `CMD ["node", "dist/src/main.js"]`.

### 4. Orquestación del Módulo Principal (`src/app.module.ts`)
- **Punto de intervención:** `@Module({ imports: [...] })`
- **Acción:** 
  - Incorporar `ConfigModule.forRoot({ isGlobal: true, envFilePath: '.env' })`.
  - Incorporar `LoggerModule.forRoot({ pinoHttp: { transport: process.env.NODE_ENV !== 'production' ? { target: 'pino-pretty' } : undefined } })`.

### 5. Punto de Entrada de la Aplicación (`src/main.ts`)
- **Punto de intervención:** Función `bootstrap()`
- **Acción:**
  - Configurar `app.useLogger(app.get(Logger))` importado de `nestjs-pino`.
  - Configurar `app.enableShutdownHooks()`.
  - Integrar `app.use(helmet())`.
  - Habilitar `app.enableCors({ origin: process.env.ALLOWED_ORIGINS?.split(',') ?? '*' })`.
  - Aplicar `ValidationPipe` global:
    ```typescript
    app.useGlobalPipes(
      new ValidationPipe({
        whitelist: true,
        forbidNonWhitelisted: true,
        transform: true,
        transformOptions: { enableImplicitConversion: true },
      }),
    );
    ```
  - Configurar Swagger condicionalmente:
    ```typescript
    if (process.env.NODE_ENV === 'development') {
      const config = new DocumentBuilder()
        .setTitle('B2B Modular Monolith API')
        .setDescription('API Documentation')
        .setVersion('1.0')
        .addBearerAuth()
        .build();
      const document = SwaggerModule.createDocument(app, config);
      SwaggerModule.setup('api/docs', app, document);
    }
    ```
  - Mantener la sintaxis de importación ESM con extensión `.js` en módulos locales.

### 6. Ajuste y Verificación de Pruebas Unitarias y E2E
- **Punto de intervención:** `test/app.e2e-spec.ts`
- **Acción:** Asegurar que `Test.createTestingModule` inicialice correctamente la aplicación sin errores causados por los nuevos módulos globales (`ConfigModule`, `LoggerModule`).

---

## 4. Criterios de Aceptación (Verificables)

- [ ] `pnpm install` finaliza sin errores de resolución de paquetes o dependencias entre pares.
- [ ] `pnpm run build` compila el proyecto sin errores TypeScript bajo `target: ES2023`, `module: nodenext` y `strict: true`.
- [ ] `docker compose config` valida exitosamente la estructura sintáctica de `docker-compose.yml`.
- [ ] `docker compose up -d` levanta los contenedores `auth-template-db` y `auth-template-redis` en estado `healthy`/`running`.
- [ ] El endpoint `http://localhost:3000/api/docs` responde `200 OK` cuando `NODE_ENV=development` y no está disponible si `NODE_ENV=production`.
- [ ] Peticiones con propiedades no declaradas en un DTO son rechazadas con error `400 Bad Request` gracias a `forbidNonWhitelisted: true`.
- [ ] Las respuestas HTTP incluyen cabeceras de seguridad generadas por `helmet` (ej. `X-Content-Type-Options: nosniff`).
- [ ] Los logs generados en consola por la aplicación son producidos a través de `nestjs-pino`.
- [ ] `pnpm test` y `pnpm run test:e2e` pasan al 100%.

---

## 5. Decisiones Tomadas y Riesgos

- **Decisión:** Mantener la resolución estricta ESM (`NodeNext`) asegurando que cualquier importación relativa use la terminación `.js` (ej. `import { AppModule } from './app.module.js'`).
  - *Razón:* El proyecto está configurado con `"type": "module"` en `package.json` y `"moduleResolution": "nodenext"` en `tsconfig.json`.
- **Decisión:** Adaptar el Dockerfile para utilizar `pnpm` en lugar del `npm` sugerido en el SRS inicial.
  - *Razón:* El repositorio usa `pnpm-lock.yaml`, evitando desincronización de versiones y ahorrando espacio de capas.
- **Riesgo / Mitigación:** *Incompatibilidad de algunos paquetes CJS con `NodeNext` ESM (ej. `pino-http` o `ua-parser-js`).*
  - *Mitigación:* Se utilizarán sintaxis de importación default o `esModuleInterop: true` (ya activo en `tsconfig.json`) y verificación mediante `pnpm run build`.
