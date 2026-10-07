# Documento de Especificación de Requerimientos (SRS)

## Plantilla B2B Escalable: Autenticación, Perfiles y Seguridad en NestJS

### 1. Visión General

El objetivo de este proyecto es construir una plantilla (boilerplate) de backend robusta, segura y altamente escalable para nivel empresarial (Enterprise/B2B SaaS). La arquitectura seguirá el patrón de **Monolito Modular (Modular Monolith)** inspirado en DDD para garantizar la modularidad y el desacoplamiento. El sistema prioriza la seguridad extrema, la gestión avanzada de sesiones multidiapositivo, el rendimiento y la flexibilidad para encender o apagar módulos en futuros proyectos.

---

### 2. Stack Tecnológico Estricto (Dependencias y Herramientas)

El código debe construirse **estrictamente** utilizando estas herramientas:

* **Framework Principal:** NestJS (`@nestjs/core`, `@nestjs/common`).
* **Base de Datos y ORM:** PostgreSQL 15+ administrado mediante Prisma (`prisma`, `@prisma/client`).
* **Validación Estricta:** `class-validator`, `class-transformer` (Configurados globalmente con `whitelist: true` y `forbidNonWhitelisted: true`).
* **Autenticación y Estrategias:** `@nestjs/passport`, `passport`, `passport-jwt`, `passport-local`, `passport-google-oauth20`.
* **Seguridad y Criptografía:**
  * `argon2` (Obligatorio para hashing de contraseñas, prohibido usar bcrypt).
  * `helmet` (Protección de cabeceras HTTP).
  * `@nestjs/throttler` (Rate limiting integrado).
  * `otplib` y `qrcode` (Generación y validación de tokens 2FA/TOTP).
* **Gestión de Sesiones:** `ua-parser-js` (Para extraer OS, navegador y dispositivo del User-Agent).
* **Mailing:** `@nestjs-modules/mailer` configurado obligatoriamente con `nodemailer` (Para soportar cualquier SMTP como AWS SES, SendGrid, Resend).
* **Caché (Opcional pero estructurado):** `redis` y `@nestjs/cache-manager`.
* **Internacionalización:** `nestjs-i18n` (Para respuestas de error y correos multi-idioma).
* **Uploads:** `multer` y `file-type` (para validación de Magic Numbers). Servicio genérico para S3/Cloudinary implementando un patrón adaptador.
* **Tareas Asíncronas y Workers:** `bullmq` y `@nestjs/bullmq` apoyados en Redis. Obligatorio para purgado de cuentas, limpieza de sesiones y correos, garantizando tolerancia a fallos y evitando condiciones de carrera en entornos distribuidos.
* **Observabilidad y Logs:** `nestjs-pino` y `pino-http` (Obligatorio para generar logs estructurados en JSON con `traceId` y `organizationId`, prohibido usar el logger nativo de NestJS en producción).

---

### 3. Arquitectura de Módulos (Domain-Driven Design)

La aplicación debe estar dividida en módulos aislados. Aunque la capa de datos (Prisma) esté unificada para garantizar integridad referencial, la lógica de negocio estará estrictamente aislada por módulos. La comunicación entre dominios se hará a través de servicios internos o eventos.

1. **`AuthModule`**: Orquestación de login local, registro, y generación de JWT (Access y Refresh).
2. **`UsersModule`**: CRUD de usuarios, gestión de perfiles (Profiles), avatares y control de Soft Delete.
3. **`SocialModule`**: Callbacks y estrategias OAuth2 (Google, etc.).
4. **`SessionsModule`**: Gestión de dispositivos activos, tracking de IPs, invalidación remota y validación de concurrencia.
5. **`SecurityModule`**: Contenedor del `PermissionsGuard` (RBAC), flujos 2FA/MFA, auditoría de seguridad y Rate Limiting.
6. **`MailModule`**: Envío de emails transaccionales (Magic Links, verificación de cuenta, reseteo de contraseña).
7. **`OrganizationsModule` (o TenantsModule):** Gestión de empresas/workspaces, facturación unificada y administración de miembros y roles aislados por organización.

---

### 4. Modelo de Datos (Prisma Schema Completo)

```prisma
generator client {
  provider = "prisma-client"
}

datasource db {
  provider = "postgresql"
}

// NUEVO: Modelo clave para B2B (Multitenencia)
model Organization {
  id          String   @id @default(uuid())
  name        String
  slug        String   @unique
  isActive    Boolean  @default(true)
  
  members     OrganizationMember[]
  roles       Role[]
  apiKeys     ApiKey[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  deletedAt   DateTime? 
}

// NUEVO: Tabla pivote que asocia al usuario con una empresa y le da un rol específico
model OrganizationMember {
  id             String       @id @default(uuid())
  userId         String
  organizationId String
  roleId         String
  
  user           User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  organization   Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  role           Role         @relation(fields: [roleId], references: [id])

  @@unique([userId, organizationId])
}

model User {
  id              String   @id @default(uuid())
  email           String   @unique
  password        String?  
  isEmailVerified Boolean  @default(false)
  isActive        Boolean  @default(true)

  // Eliminado: roleId global. Ahora el rol está en OrganizationMember
  memberships     OrganizationMember[]
  
  profile         Profile?
  security        UserSecurity?
  socialAccounts  SocialAccount[]
  sessions        Session[]
  securityLogs    SecurityLog[]
  apiKeys         ApiKey[]

  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  deletedAt       DateTime? 
}

model Session {
  id               String   @id @default(uuid())
  userId           String
  user             User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  refreshTokenHash String   @unique 
  ipAddress        String?
  browser          String?  // Extraído de ua-parser-js
  os               String?  // Extraído de ua-parser-js
  device           String?  // Extraído de ua-parser-js
  lastActiveAt     DateTime @default(now())
  expiresAt        DateTime
  createdAt        DateTime @default(now())

  @@index([expiresAt]) // Crítico para el worker de limpieza de sesiones expiradas
}

model ApiKey {
  id              String       @id @default(uuid())
  organizationId  String
  organization    Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  createdByUserId String       // Para auditoría (quién creó la llave)
  keyHash         String       @unique 
  name            String    
  scopes          String[] 
  expiresAt       DateTime?
  lastUsedAt      DateTime?
  createdAt       DateTime     @default(now())
}

model UserSecurity {
  id                  String    @id @default(uuid())
  userId              String    @unique
  user                User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // 2FA
  twoFactorSecret     String?
  isTwoFactorEnabled  Boolean   @default(false)
  recoveryCodes       String[]  // Hashes Argon2 de uso único

  // Flujo Seguro y Bloqueo
  pendingEmail        String?   
  failedLoginAttempts Int       @default(0)
  lockedUntil         DateTime? 
}

model Profile {
  id          String   @id @default(uuid())
  userId      String   @unique
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  firstName   String?
  lastName    String?
  avatarUrl   String?
  bio         String?
  phoneNumber String?

  @@index([firstName, lastName])
}

model SocialAccount {
  id                String   @id @default(uuid())
  userId            String
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  provider          String   
  providerAccountId String   @unique
  createdAt         DateTime @default(now())

  @@unique([provider, providerAccountId])
  @@index([userId])
}

model Role {
  id             String          @id @default(uuid())
  name           String
  description    String?
  organizationId String?         // Si es null, es un rol global del sistema. Si tiene ID, es exclusivo de un Tenant.
  organization   Organization?   @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  users          User[]
  permissions    RolePermission[]

  @@unique([name, organizationId]) // Evita nombres duplicados dentro de la misma empresa
}

model Permission {
  id          String           @id @default(uuid())
  action      String           @unique 
  description String?
  roles       RolePermission[]
}

model RolePermission {
  roleId       String
  permissionId String
  role         Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)

  @@id([roleId, permissionId])
}

model SecurityLog {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  action    String   
  ipAddress String?
  userAgent String?
  createdAt DateTime @default(now())

  @@index([userId])
  @@index([createdAt]) // Para Cursor Pagination eficiente
}

```

---

### 5. Lógica de Negocio y Flujos Críticos

#### 5.1. Autenticación y JWT (API Agnóstica y Estrategia Multicliente)

* **Payload de la API y Expiración Configurable:** La API REST devolverá el `accessToken` y el `refreshToken` en el cuerpo JSON de la respuesta. Los tiempos de expiración no estarán "hardcodeados"; se inyectarán estrictamente mediante variables de entorno (`JWT_ACCESS_EXPIRES_IN` y `JWT_REFRESH_EXPIRES_IN`) para permitir flexibilidad según el modelo de negocio (ej. sesiones cortas para B2B estricto, o sesiones de 90+ días para apps B2C).

* **Payload Híbrido:** El `accessToken` no contendrá información sensible ni el perfil completo; únicamente incluirá `userId`, `roleId` y `sessionId`.

* **Arquitectura de Seguridad según Cliente:**
  * **Cliente Web (Next.js):** Se implementará el patrón Backend-for-Frontend (BFF) mediante Server Actions/Route Handlers. Next.js interceptará la respuesta de NestJS y transformará el `refreshToken` en una cookie `HttpOnly`, `Secure` y `SameSite`, blindando la web contra ataques XSS.
  * **Clientes Móviles Nativo/Híbrido:** Consumirán el JSON directamente y almacenarán los tokens exclusivamente en las bóvedas de seguridad del sistema operativo (Keychain / KeyStore).
* **Refresh Token Rotation y Detección de Reutilización:** Para mitigar el robo de tokens, se implementará rotación obligatoria. Cada vez que el cliente consuma `/auth/refresh`, el servidor invalidará el refreshToken usado y emitirá uno nuevo. Si el servidor detecta un intento de uso de un refreshToken previamente invalidado (Reuse Detection), asumirá que la sesión fue comprometida y revocará en cascada toda la familia de tokens asociada a ese sessionId, desconectando al usuario inmediatamente como medida de contención.
* **Operaciones Críticas (Strict Validation):** Aunque la API confía en la naturaleza stateless del JWT durante su corta vida de 10 minutos, se implementará un decorador @RequireStrictValidation() para proteger endpoints destructivos o financieros (ej. borrar cuenta, procesar pagos, cambiar contraseñas). Este decorador forzará una consulta rápida a la caché (Redis) para confirmar en tiempo real que la sesión no ha sido revocada o la cuenta baneada (isActive: false), cerrando la ventana de vulnerabilidad del token asíncrono.

#### 5.2. Gestión de Sesiones y Dispositivos

*Al hacer login, se extraen metadatos del `User-Agent` usando `ua-parser-js` y se crea un registro en `Session`.
* **Auto-Revoke:** Si un usuario cambia su contraseña, todas sus sesiones activas (excepto la que está usando para el cambio) deben ser eliminadas de la base de datos y limpiar las claves Redis de sesiones activas (o actualizar lista blanca).
* Debe existir un endpoint genérico para que el usuario consulte dónde tiene sesiones abiertas y pueda cerrarlas (DELETE `/sessions/:id`).

#### 5.3. Flujo OAuth2 y Auto-Aprovisionamiento (Social Login)
* **Auto-Creación de Cuentas (Sign-up fluido):** Si un usuario inicia sesión con Google/Apple y su correo no existe en la base de datos, el sistema lo registrará automáticamente, creará su Profile extrayendo el nombre y avatar del proveedor, y lo autenticará sin requerir pasos adicionales.

* **Prevención de Account Takeover (Vincular cuentas):** El sistema autovinculará la cuenta social solo si se cumplen dos condiciones simultáneas: 
  1) El correo original tiene isEmailVerified: true en la base de datos local. 
  2) El payload del proveedor (Google/Apple) trae la confirmación explícita de que el correo fue validado por ellos (ej. email_verified: true). Si el proveedor no garantiza la propiedad del correo, se denegará el acceso inmediatamente con un error 403 Forbidden.

* **Autenticación Social Unificada (Web y Móvil):** El login social utilizará el estándar RFC 8252 (siempre vía navegador web). Para clientes móviles, el backend soportará redirecciones dinámicas mediante Deep/Universal Links validados con el estándar PKCE, asegurando que el código de autorización no sea interceptado.

#### 5.4. Roles y Permisos Granulares (RBAC)

* **Autorización Multitenant y Resolución de Contexto:** Los roles dependen de la organización. El sistema exigirá que el cliente envíe una cabecera HTTP x-organization-id en todas las rutas bajo /organizations/.... El PermissionsGuard utilizará esta cabecera para resolver el contexto: si el endpoint es global (ej. /admin/system), validará contra roles con organizationId: null. Si el endpoint es de tenant, cruzará obligatoriamente el rol del usuario en esa empresa específica. Esto permite que un mismo usuario sea "Owner" en la Empresa A y "Viewer" en la Empresa B sin colisión de permisos.
* 
#### 5.5. Seguridad Extrema y Módulo 2FA

* **ThrottlerGuard:** Bloquear IP tras 5 intentos fallidos de login en 15 minutos.
* **Flujo 2FA:** Si `isTwoFactorEnabled` es true, el login devuelve `requires2FA: true` y un token temporal. Este token temporal debe estar firmado con un claim estricto `{ scope: '2fa_only' }`. El `JwtGuard` global rechazará automáticamente cualquier token con este scope, garantizando que solo pueda ser utilizado en el endpoint `POST /auth/2fa/verify`.
* **Códigos de Recuperación (Prevención de Bloqueo de CPU):** Al activar 2FA, se generarán 10 códigos de un solo uso de alta entropía (mediante crypto.randomBytes). A diferencia de las contraseñas humanas, estos códigos no se hashearán con Argon2 para evitar la saturación del Event Loop y del pool de hilos de Node.js. Al ser criptográficamente seguros y no predecibles, se almacenarán usando hashes rápidos nativos (SHA-256), optimizando el rendimiento del servidor.
* **Account Lockout (Defensa Botnet):** Si una cuenta alcanza 10 intentos fallidos (`failedLoginAttempts`), se establece el campo `lockedUntil` por 30 minutos. Cualquier intento de login en este lapso devolverá `423 Locked` de inmediato, sin procesar la verificación Argon2 para evitar saturación de CPU.

#### 5.6. Experiencia de Usuario y Extra Features

* **Estrategia de Retención (Soft a Hard Delete):** Para permitir la recuperación de cuentas, al eliminar un usuario o empresa solo se actualiza su campo `deletedAt` (Soft Delete). Durante este periodo de gracia de 30 días, la Prisma Client Extension global intercepta las consultas para ocultar la entidad. Pasados los 30 días, un worker de BullMQ programado ejecutará un Hard Delete (eliminación física). Es en este momento cuando entra en acción el `onDelete`: Cascade de la base de datos, limpiando automáticamente sus sesiones, perfil y logs relacionales. Al usar BullMQ, se evitan condiciones de carrera entre múltiples contenedores sin necesidad de programar bloqueos distribuidos (locks) manualmente.
* **Magic Links y Recuperación:** Soporte para login sin contraseña mediante un JWT de un solo uso enviado por email (expiración 15 min). Para evitar ataques de reutilización (Replay Attacks), el ID del token (jti) se registrará en Redis al generarse y se eliminará/invalidará inmediatamente tras su primer uso.
* **Email Verification:** Un Guard `@RequireEmailVerification()` para proteger rutas sensibles si el correo no está validado.
* **Security Logs:** Registrar cada intento fallido de login, cambio de password o acceso desde nueva IP en la tabla `SecurityLog` para auditoría.
* **Cambio de Email Seguro:** La actualización del correo no es inmediata. Se guarda en `pendingEmail` y se envía un link de confirmación. El campo `email` principal solo se actualiza tras validar el link.
* **Seguridad en Avatares:** No confiar en la extensión del archivo. Validar la firma hexadecimal real (Magic Numbers) usando `file-type` antes de subirlo a S3/Cloudinary.
* **Internacionalización Dinámica (i18n):** El sistema debe leer el header HTTP `Accept-Language`. Todas las excepciones, los mensajes de error de los DTOs (`class-validator`) y las plantillas de correo (`MailModule`) deben traducirse automáticamente usando `nestjs-i18n`.
* **Transacciones de Base de Datos (Atomicidad):** Cualquier operación que afecte a múltiples tablas (ej. un registro que crea un `User`, un `Profile` y una `Session` simultáneamente) debe ejecutarse obligatoriamente dentro de un `$transaction` de Prisma. Si una inserción falla, se debe hacer rollback de todo.

---

### 6. Infraestructura y Entornos (Docker)

La plantilla debe ser 100% agnóstica a la máquina del desarrollador.

#### 6.1. Entorno Local (Variables y Docker Compose)

El proyecto requerirá un archivo `.env` en la raíz con la configuración de credenciales y los tiempos de expiración de las sesiones:

```env

# Ejemplo de variables requeridas (.env)
JWT_ACCESS_EXPIRES_IN="15m"
JWT_REFRESH_EXPIRES_IN="90d" # Configurable por proyecto (ej. 90d para B2C, 7d para B2B)
DB_USER="postgres"
DB_PASSWORD="password123"
DB_NAME="auth_db"

```

Archivo `docker-compose.yml` que levante la infraestructura requerida:

```yaml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    container_name: auth-template-db
    restart: always
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
      POSTGRES_DB: ${DB_NAME:-auth_db}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    container_name: auth-template-redis
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --save 60 1 --loglevel warning

volumes:
  postgres_data:
  redis_data:
    
```

#### 6.2. Dockerfile (Despliegue Multi-Stage Optimizado)

Archivo `Dockerfile` estructurado para generar la imagen de producción más ligera y segura posible:

```dockerfile
# --- ETAPA 1: Dependencias Base ---
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
COPY prisma ./prisma/
RUN npm ci

# --- ETAPA 2: Build de la Aplicación ---
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build
# Limpiamos dependencias de desarrollo para aligerar la imagen de producción
RUN npm prune --omit=dev

# --- ETAPA 3: Producción ---
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production

# Copiamos solo lo necesario desde el builder
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
COPY --from=builder /app/prisma ./prisma

EXPOSE 3000

# NOTA OPERACIONAL: 'npx prisma migrate deploy' NO debe ir en el CMD para 
# evitar bloqueos en PostgreSQL si se levantan múltiples réplicas simultáneamente. 
# Debe ejecutarse como un InitContainer en Kubernetes o en el pipeline CI/CD.
CMD ["node", "dist/main"]

```

---

### 7. Estándares API, Testing y Operaciones (DevOps)

#### 7.1. Estrategia de Testing (Alta Fidelidad)
* **Testing E2E:** Se implementará `Testcontainers` para las pruebas End-to-End. Cada ejecución de pruebas levantará contenedores efímeros reales de PostgreSQL y Redis, garantizando un entorno idéntico a producción sin depender de mocks de base de datos.
* **Unit Testing:** Se utilizará Jest para probar servicios críticos de forma aislada.

#### 7.2. Estándares de la API
* **Paginación por Cursor:** Todos los endpoints que devuelvan listas masivas (logs, usuarios) utilizarán Cursor Pagination. Esto garantiza rendimiento constante en tablas grandes y facilita el scroll infinito en el frontend.
* **Documentación Privada:** La API estará documentada mediante `@nestjs/swagger`, pero su acceso `/api/docs` estará condicionado estrictamente a entornos de desarrollo (`NODE_ENV=development`).
* **CORS Estricto y Seguridad Móvil:** La API bloqueará cualquier origen web no listado en `ALLOWED_ORIGINS`. Para peticiones móviles, se implementará "App Attestation" (Firebase App Check o Apple DeviceCheck). Un Guard validará el token criptográfico emitido por el SO para asegurar que la petición proviene de un binario oficial no alterado. Queda estrictamente prohibido el uso de cabeceras estáticas como `x-api-key` en clientes móviles.

#### 7.3. Observabilidad y Background Jobs
* **Logs Estructurados:** Se implementará `nestjs-pino` para generar logs en formato JSON. Cada log incluirá automáticamente el `traceId` de la petición HTTP, el `userId` y el `organizationId` para facilitar el rastreo en herramientas de APM (Datadog/CloudWatch).

* **Workers y Tareas Asíncronas:** Se utilizará `BullMQ` apoyado en `Redis`. Existirán workers dedicados para: 1) El purgado físico de cuentas (Hard Delete) tras 30 días de gracia. 2) Un trabajo recurrente cada 24 horas para ejecutar `DELETE FROM Session WHERE expiresAt < now()` y evitar la saturación de la base de datos. `BullMQ` garantizará la no duplicación de tareas y el manejo de reintentos sin necesidad de implementar Distributed Locks manuales.

#### 7.4. Resiliencia de Infraestructura
* **Health Checks:** Implementación de `@nestjs/terminus` exponiendo un endpoint `/health` para monitorización de orquestadores (Docker/Kubernetes).
* **Graceful Shutdown:** Configuración de `app.enableShutdownHooks()` para asegurar que NestJS cierre correctamente las conexiones a Prisma y procesos en curso antes de apagarse.


#### 7.5. Estándares Estrictos de Implementación y Prevención de Fugas
* **Aislamiento de Tenants Forzado (Anti-Data Leaks):** Para evitar que un error humano exponga datos cruzados entre empresas (ej. olvidar añadir `where: { organizationId }` en una consulta), el aislamiento de datos no dependerá de la memoria del desarrollador. Se implementará obligatoriamente una Prisma Client Extension global combinada con el módulo `AsyncLocalStorage` nativo de Node.js. Esta extensión interceptará en tiempo real todas las operaciones de lectura y escritura (`findMany`, `findFirst`, `update`, `delete`) e inyectará automáticamente el `organizationId` extraído del token del usuario, garantizando el aislamiento a nivel de ORM.
* **Rate Limiting Global y Distribuido:** El uso del paquete `@nestjs/throttler` con su almacenamiento por defecto en memoria RAM queda estrictamente prohibido. Para evitar que los límites de intentos se multipliquen por el número de réplicas de la API detrás del balanceador de carga, se configurará obligatoriamente el adaptador `nestjs-throttler-storage-redis`. Esto asegurará que los contadores de intentos fallidos y bloqueos de IP sean centralizados y reales para toda la infraestructura.
* **Migraciones de Base de Datos (Zero-Downtime):** La ejecución de `npx prisma migrate deploy` jamás formará parte del ciclo de arranque de la aplicación (ni en el CMD del Dockerfile ni en el `main.ts`). Las migraciones se ejecutarán exclusivamente en el pipeline CI/CD o mediante `InitContainers` (en Kubernetes) antes de que la nueva versión reciba tráfico. Asimismo, queda prohibido realizar cambios de esquema destructivos (como renombrar o eliminar columnas) en un solo paso; toda migración debe ser retrocompatible en fases (Expand and Contract pattern) para garantizar despliegues sin tiempo de inactividad.
