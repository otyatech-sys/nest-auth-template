# 📐 Plan Técnico de Ejecución: Plantilla B2B Modular Monolith en NestJS (Fases de Implementación)

> **Objetivo:** Construir una plantilla (boilerplate) de backend desacoplada, segura y escalable para aplicaciones B2B SaaS en NestJS con arquitectura de Monolito Modular inspirada en DDD, multitenencia estricta, gestión de sesiones multidispositivo y workers asíncronos.

---

## 1. Alcance (Scope)

**Dentro del alcance:**

* Configuración de infraestructura local con Docker Compose (PostgreSQL 15 + Redis 7) y Dockerfile multi-stage.
* Modelo de datos completo en Prisma ORM e integración con extensión global mediante `AsyncLocalStorage` para aislamiento automático de tenants y Soft Delete en un período de gracia de 30 días.
* Módulos del dominio: `AuthModule`, `UsersModule`, `OrganizationsModule`, `SessionsModule`, `SecurityModule`, `MailModule`, `SocialModule`.
* Hash de contraseñas con `argon2` y hashing rápido con `SHA-256` para códigos de recuperación 2FA.
* Control de sesiones multidispositivo (`ua-parser-js`), Refresh Token Rotation con detección de reutilización y validación estricta con `@RequireStrictValidation()`.
* Autenticación multifactor (2FA TOTP con `otplib`/`qrcode`) y defensa contra ataques de fuerza bruta (Account Lockout tras 10 intentos y `nestjs-throttler-storage-redis`).
* Tareas asíncronas en segundo plano con `BullMQ` (limpieza diaria de sesiones expiradas y purga física *hard delete* tras 30 días).
* Validación de archivos (*Magic Numbers*) usando `file-type` con un patrón adaptador para storage (S3/Cloudinary).
* Observabilidad estructurada en JSON (`nestjs-pino`) inyectando `traceId`, `userId` y `organizationId`.
* Estrategia de testing de alta fidelidad E2E con `Testcontainers`.

**Fuera del alcance (Postergado / No contemplado en esta etapa):**

* Implementación de paneles de administración frontend (únicamente API REST y BFF pattern en el lado del cliente).
* Integración de pasarelas de pago específicas (Stripe/Paddle) dentro del núcleo base.

---

## 2. Mapa de Archivos Afectados

### Archivos de Configuración e Infraestructura (Creación / Modificación)

* `docker-compose.yml` — **[Creación]** — *Definición de servicios PostgreSQL 15 y Redis 7.*
* `Dockerfile` — **[Creación]** — *Construcción multi-stage (deps, builder, production).*
* `.env.example` — **[Creación]** — *Variables de entorno base para desarrollo y producción.*
* `package.json` — **[Modificación]** — *Adición de dependencias de producción y desarrollo especificadas en el SRS.*
* `src/main.ts` — **[Modificación]** — *Configuración global de ValidationPipe, Helmet, Pino Logger, CORS y Swagger condicional.*
* `src/app.module.ts` — **[Modificación]** — *Orquestación de módulos de dominio y middlewares de contexto.*

### Capa de Persistencia y Base de Datos

* `prisma/schema.prisma` — **[Creación]** — *Modelo de datos relacional B2B multitenant completo.*
* `src/common/prisma/prisma.service.ts` — **[Creación]** — *Servicio de conexión Prisma e integración de cliente extendido.*
* `src/common/prisma/prisma-tenant.extension.ts` — **[Creación]** — *Prisma Client Extension con AsyncLocalStorage para aislamiento de tenant y Soft Delete.*

### Módulos del Sistema

* `src/common/` — **[Creación]** — *Guards, Decorators, Interceptors, Storage Adapters e i18n.*
* `src/modules/auth/` — **[Creación]** — *Estrategias JWT/Local, rotación de tokens y endpoints `/auth`.*
* `src/modules/organizations/` — **[Creación]** — *Gestión de workspaces, miembros y asignación de roles.*
* `src/modules/users/` — **[Creación]** — *CRUD de usuarios, gestión de perfiles y borrado suave.*
* `src/modules/sessions/` — **[Creación]** — *Extracción de metadatos `ua-parser-js`, revocación remota e inspección.*
* `src/modules/security/` — **[Creación]** — *Módulos RBAC (`PermissionsGuard`), 2FA/TOTP y Security Logs.*
* `src/modules/social/` — **[Creación]** — *Estrategia Google OAuth2, flujo PKCE y prevención de account takeover.*
* `src/modules/mail/` — **[Creación]** — *Configuración Mailer con Nodemailer y plantillas multi-idioma.*
* `src/workers/` — **[Creación]** — *Procesadores de BullMQ para purga de sesiones y hard delete de entidades.*

---

## 3. Secuencia de Implementación Paso a Paso

### Fase 1: Infraestructura Base, Dependencias e Identidad

1. **Instalación de Dependencias:**
* Instalar paquetes requeridos: `@prisma/client`, `prisma`, `argon2`, `helmet`, `@nestjs/throttler`, `nestjs-throttler-storage-redis`, `otplib`, `qrcode`, `ua-parser-js`, `@nestjs-modules/mailer`, `nodemailer`, `redis`, `@nestjs/cache-manager`, `nestjs-i18n`, `multer`, `file-type`, `bullmq`, `@nestjs/bullmq`, `nestjs-pino`, `pino-http`, `@nestjs/passport`, `passport`, `passport-jwt`, `passport-local`, `passport-google-oauth20`, `class-validator`, `class-transformer`, `@nestjs/terminus`.


2. **Entorno Docker:**
* Crear `docker-compose.yml` especificando contenedores PostgreSQL 15-alpine y Redis 7-alpine con persistencia de volúmenes.
* Crear `Dockerfile` optimizado en 3 etapas (`deps`, `builder`, `production`) garantizando que `npx prisma migrate deploy` se ejecute fuera del arranque directo del contenedor de la app.


3. **Punto de Entrada (`src/main.ts`):**
* Configurar `ValidationPipe` global con `whitelist: true`, `forbidNonWhitelisted: true` y `transform: true`.
* Integrar `helmet()` para cabeceras HTTP de seguridad.
* Habilitar `app.enableShutdownHooks()` y reemplazar el logger nativo por `Logger` de `nestjs-pino`.
* Exponer Swagger (`@nestjs/swagger`) únicamente cuando `process.env.NODE_ENV === 'development'`.



### Fase 2: Esquema Prisma y Capa de Aislamiento Multitenant

1. **Definición del Schema (`prisma/schema.prisma`):**
* Implementar los modelos: `Organization`, `OrganizationMember`, `User`, `Session`, `ApiKey`, `UserSecurity`, `Profile`, `SocialAccount`, `Role`, `Permission`, `RolePermission`, `SecurityLog`.
* Definir índices críticos (`Session.expiresAt`, `Profile[firstName, lastName]`, `SecurityLog.createdAt`).


2. **Aislamiento de Tenants y Soft Delete (`AsyncLocalStorage` + Prisma Extension):**
* Crear un middleware/interceptor HTTP que capture la cabecera `x-organization-id` y el ID del usuario en la solicitud HTTP y los almacene en la instancia `AsyncLocalStorage`.
* Implementar una Prisma Client Extension que intercepte consultas de lectura y escritura (`findMany`, `findFirst`, `update`, `delete`) inyectando automáticamente el `organizationId` activo y filtrando registros donde `deletedAt` sea `null`.



### Fase 3: Núcleo de Seguridad, Rate Limiting y Correos (MailModule)

1. **Configuración de Rate Limiting Distribuido:**
* Configurar `ThrottlerModule` utilizando `nestjs-throttler-storage-redis` para evitar discrepancias de contadores entre múltiples instancias.


2. **Sistema de Logs Estructurados (Observabilidad):**
* Configurar `LoggerModule` de `nestjs-pino` para generar eventos JSON inyectando contextualmente `traceId` (UUID de la petición), `userId` y `organizationId`.


3. **Módulo Transaccional de Emails (`MailModule`):**
* Integrar `@nestjs-modules/mailer` respaldado por `nodemailer`.
* Configurar `nestjs-i18n` para resolver dinámicamente mensajes de error, validaciones de DTOs y plantillas de correos electrónicos a través del header `Accept-Language`.



### Fase 4: Autenticación, Sesiones y Dominio de Usuarios (`AuthModule`, `UsersModule`, `OrganizationsModule`)

1. **Estrategia Hash y Gestión de Tokens:**
* Implementar servicio para hash de contraseñas con `argon2` (prohibido bcrypt).
* Crear flujos de emisión de JWT con tiempos de expiración configurables (`JWT_ACCESS_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN`).


2. **Sesiones y Detección de Reutilización (`SessionsModule`):**
* Extraer metadatos de usuario (`browser`, `os`, `device`) utilizando `ua-parser-js` al autenticarse.
* Guardar el hash del `refreshToken` en la tabla `Session`.
* Implementar rotación estricta de tokens al consumir `/auth/refresh`. En caso de detectar un reuso de token invalidado, revocar automáticamente la familia entera de tokens asociada a la sesión.


3. **Decorador `@RequireStrictValidation()`:**
* Crear Guard que consulte Redis para comprobar de forma síncrona que el token no ha sido revocado y que la cuenta permanece activa (`isActive: true`) antes de permitir el paso a endpoints sensibles.


4. **Organizaciones y Usuarios:**
* Lógica de registro y creación atómica dentro de una transacción `$transaction` de Prisma (creando `User`, `Profile` y asignando la primera `Organization` como propietario).



### Fase 5: Autorización Granular (RBAC) y Seguridad Avanzada (`SecurityModule`)

1. **Control de Acceso basado en Roles (`PermissionsGuard`):**
* Implementar guard que resuelva el contexto: si la ruta requiere `x-organization-id`, evalúa el rol específico de la tabla pivote `OrganizationMember`; si la ruta es del sistema, evalúa el rol global con `organizationId: null`.


2. **Autenticación Multifactor (2FA/TOTP):**
* Implementar flujo de generación y verificación de secreta 2FA (`otplib` y códigos QR con `qrcode`).
* Generación de 10 códigos de recuperación almacenados mediante hash ligero `SHA-256` para evitar cuellos de botella de CPU en Node.js.
* Emitir JWT con scope exclusivo `{ scope: '2fa_only' }` tras login inicial cuando 2FA esté habilitado, bloqueando todas las rutas excepto `/auth/2fa/verify`.


3. **Account Lockout:**
* Incrementar `failedLoginAttempts` tras login erróneo. Bloquear la cuenta mediante el campo `lockedUntil` por 30 minutos al alcanzar 10 intentos e interrumpir la verificación de hash inmediatamente devolviendo estado `423 Locked`.



### Fase 6: Autenticación Social y Procesamiento Asíncrono (`SocialModule` y BullMQ)

1. **Login Social (`SocialModule`):**
* Configurar estrategia `passport-google-oauth20`.
* Verificar explícitamente `email_verified: true` del proveedor para evitar *Account Takeover*. Registrar o asociar la cuenta automáticamente si el email ya existe y fue previamente verificado localmente.


2. **Workers en Segundo Plano (`BullMQ`):**
* Configurar worker con cron diario para ejecutar `DELETE FROM Session WHERE expiresAt < NOW()`.
* Configurar worker para el proceso de purga física de datos (*Hard Delete*) para registros cuyo `deletedAt` sea superior a 30 días, permitiendo que la restricción de base de datos `ON DELETE CASCADE` limpie las dependencias relacionadas.



### Fase 7: Carga de Archivos, Diagnósticos y Cobertura de Pruebas

1. **Servicio de Carga Segura de Archivos:**
* Implementar interceptor de carga de imágenes de perfil con `multer`. Validar los *Magic Numbers* del archivo enviado utilizando la biblioteca `file-type` antes de procesar la subida al almacenamiento de destino mediante una interfaz adaptadora (`StorageAdapter`).


2. **Monitorización de Salud (`@nestjs/terminus`):**
* Crear endpoint `/health` que verifique el estado de conexión de la base de datos PostgreSQL mediante Prisma y el estado de la instancia de Redis.


3. **Pruebas de Integración y E2E:**
* Configurar entorno de pruebas E2E con Vitest e integrar `Testcontainers` para instanciar contenedores reales de PostgreSQL y Redis durante el ciclo de testing.



---

## 4. Criterios de Aceptación (Verificables)

* [ ] La aplicación compila sin errores TypeScript (`npm run build`) bajo modo estricto.
* [ ] La infraestructura de desarrollo se levanta correctamente con `docker-compose up -d`.
* [ ] La extensión global de Prisma intercepta automáticamente todas las consultas inyectando el filtro del tenant activo extraído de `AsyncLocalStorage`.
* [ ] El proceso de login firma los hash de contraseñas utilizando únicamente `argon2`.
* [ ] La rotación de `refreshToken` invalida la sesión completa al reusar un token revocado.
* [ ] Al intentar autenticarse con una cuenta con 10 intentos fallidos se recibe una respuesta con código `423 Locked` sin ejecutar validaciones de hashing adicionales.
* [ ] El endpoint `/auth/2fa/verify` acepta el token firmado con scope `2fa_only` y rechaza dicho token en cualquier otro endpoint.
* [ ] La carga de avatares rechaza archivos ejecutables con extensión modificada mediante la comprobación de *Magic Numbers*.
* [ ] Los logs generados en consola por `nestjs-pino` incluyen los atributos `traceId`, `userId` y `organizationId` estructurados en formato JSON.
* [ ] La suite de pruebas E2E ejecuta los tests sobre un contenedor de datos de `Testcontainers` pasando en un 100%.

---

## 5. Decisiones Tomadas y Riesgos

* **Decisión:** Sustituir la comprobación de códigos de recuperación 2FA con `argon2` por `SHA-256`.
*Justificación:* Los códigos de recuperación poseen alta entropía generada con `crypto.randomBytes`. Usar `argon2` para validar múltiples códigos en bucle saturaría el pool de hilos de Node.js y la CPU del servidor.
* **Decisión:** Separar las migraciones de base de datos del comando de arranque del contenedor Docker.
*Justificación:* Evita bloqueos de tablas y condiciones de carrera cuando se despliegan múltiples réplicas de la aplicación simultáneamente en entornos como Kubernetes.
* **Riesgo / Mitigación:** *Fuga de datos entre organizaciones debido a consultas manuales sin cláusula `where`.*
*Mitigación:* Implementación forzada de una Prisma Client Extension con `AsyncLocalStorage`, que actúa a nivel de ORM interceptando cualquier consulta antes de ser enviada a la base de datos.
