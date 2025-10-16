# 🔐 Flujo de Autenticación JWT - Diagrama Visual

## Flujo Completo de Autenticación

```
┌──────────────┐                                    ┌──────────────┐
│              │                                    │              │
│   CLIENTE    │                                    │   SERVIDOR   │
│  (Frontend)  │                                    │  (Backend)   │
│              │                                    │              │
└──────┬───────┘                                    └──────┬───────┘
       │                                                   │
       │  1. POST /api/login                              │
       │     { email, password }                          │
       ├─────────────────────────────────────────────────►│
       │                                                   │
       │                                                   │ 2. Verificar
       │                                                   │    credenciales
       │                                                   │    en DB
       │                                                   │
       │                                                   │ 3. Generar
       │                                                   │    JWT token
       │                                                   │
       │  4. Respuesta con token                          │
       │     { success: true, token: "eyJ..." }           │
       │◄─────────────────────────────────────────────────┤
       │                                                   │
       │  5. Guardar token                                │
       │     localStorage.setItem('jwt_token', token)     │
       │                                                   │
       │                                                   │
       │  6. GET /api/protected                           │
       │     Headers: Authorization: Bearer eyJ...        │
       ├─────────────────────────────────────────────────►│
       │                                                   │
       │                                                   │ 7. Validar token
       │                                                   │    - Firma válida?
       │                                                   │    - No expiró?
       │                                                   │
       │  8. Datos protegidos                             │
       │     { message: "Acceso concedido", ... }         │
       │◄─────────────────────────────────────────────────┤
       │                                                   │
       │                                                   │
       │  9. GET /api/admin                               │
       │     Headers: Authorization: Bearer eyJ...        │
       ├─────────────────────────────────────────────────►│
       │                                                   │
       │                                                   │ 10. Validar token
       │                                                   │     Y verificar
       │                                                   │     rol admin
       │                                                   │
       │  11a. Si es admin: 200 OK                        │
       │       { message: "Acceso admin" }                │
       │◄─────────────────────────────────────────────────┤
       │                                                   │
       │  11b. Si NO es admin: 403 Forbidden              │
       │◄─────────────────────────────────────────────────┤
       │                                                   │
       │                                                   │
       │  12. Cerrar sesión (logout)                      │
       │      localStorage.removeItem('jwt_token')        │
       │                                                   │
       │                                                   │
```

---

## Estructura del Token JWT

```
┌─────────────────────────────────────────────────────┐
│                   TOKEN JWT                         │
│                                                     │
│  <TOKEN>             │
│  <TOKEN> │
│  IiwibmFtZSI6IkpvaG4gRG9lIiwiaXNfYWRtaW4iOmZhbHN │
│  lLCJleHAiOjE3MzY5ODc2MDAsImlhdCI6MTczNjkwMTIwMH │
│  0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c    │
│                                                     │
└─────────────────────────────────────────────────────┘
            │                  │              │
            ▼                  ▼              ▼
       ┌────────┐         ┌────────┐     ┌──────────┐
       │ HEADER │         │ PAYLOAD│     │SIGNATURE │
       └────────┘         └────────┘     └──────────┘
```

### Desglose del Token:

#### 1. HEADER (Base64)
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### 2. PAYLOAD (Base64) - Claims
```json
{
  "sub": "1",                      // User ID
  "email": "usuario@ejemplo.com",  // Email
  "name": "Juan Pérez",            // Nombre
  "is_admin": false,               // Rol
  "exp": 1736987600,               // Expira: timestamp
  "iat": 1736901200                // Emitido: timestamp
}
```

#### 3. SIGNATURE (HMAC SHA-256)
```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

---

## Proceso de Validación del Guardián

```
┌────────────────────────────────────────────────┐
│ Request recibido                               │
└───────────────┬────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────┐
│ ¿Tiene header "Authorization"?                │
└───────┬────────────────────────────────────────┘
        │
        ├─── NO ──►  401 Unauthorized
        │
        │ SÍ
        ▼
┌────────────────────────────────────────────────┐
│ ¿Comienza con "Bearer "?                      │
└───────┬────────────────────────────────────────┘
        │
        ├─── NO ──►  401 Unauthorized
        │
        │ SÍ
        ▼
┌────────────────────────────────────────────────┐
│ Extraer token (después de "Bearer ")          │
└───────┬────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────────┐
│ Decodificar JWT                                │
│ - Verificar firma con secret                   │
│ - Verificar formato                            │
└───────┬────────────────────────────────────────┘
        │
        ├─── INVÁLIDO ──►  401 Unauthorized
        │
        │ VÁLIDO
        ▼
┌────────────────────────────────────────────────┐
│ Verificar expiración (exp > now)              │
└───────┬────────────────────────────────────────┘
        │
        ├─── EXPIRADO ──►  401 Unauthorized
        │
        │ NO EXPIRADO
        ▼
┌────────────────────────────────────────────────┐
│ [Para AdminUser] ¿is_admin == true?           │
└───────┬────────────────────────────────────────┘
        │
        ├─── NO ──►  403 Forbidden
        │
        │ SÍ (o no aplica para AuthenticatedUser)
        ▼
┌────────────────────────────────────────────────┐
│ ✓ Acceso Concedido                            │
│ Claims extraídos y disponibles en la ruta     │
└────────────────────────────────────────────────┘
```

---

## Guardianes Implementados

### 1. AuthenticatedUser
```rust
┌──────────────────────────────────────────┐
│ AuthenticatedUser                        │
├──────────────────────────────────────────┤
│ Verifica:                                │
│  ✓ Token presente                        │
│  ✓ Token válido                          │
│  ✓ Token no expirado                     │
├──────────────────────────────────────────┤
│ Provee:                                  │
│  • Claims del usuario (id, email, etc)   │
├──────────────────────────────────────────┤
│ Uso:                                     │
│  #[get("/api/protected")]                │
│  fn route(user: AuthenticatedUser) {...} │
└──────────────────────────────────────────┘
```

### 2. AdminUser
```rust
┌──────────────────────────────────────────┐
│ AdminUser                                │
├──────────────────────────────────────────┤
│ Verifica:                                │
│  ✓ Todo lo de AuthenticatedUser          │
│  ✓ is_admin == true                      │
├──────────────────────────────────────────┤
│ Provee:                                  │
│  • Claims del admin (id, email, etc)     │
├──────────────────────────────────────────┤
│ Uso:                                     │
│  #[get("/api/admin")]                    │
│  fn route(admin: AdminUser) {...}        │
└──────────────────────────────────────────┘
```

---

## Códigos de Respuesta HTTP

```
┌──────────────────────────────────────────────────────┐
│ 200 OK                                               │
│ ✓ Autenticación exitosa                             │
│ ✓ Acceso a ruta protegida concedido                 │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ 401 UNAUTHORIZED                                     │
│ ✗ Token no proporcionado                            │
│ ✗ Token inválido (firma incorrecta)                 │
│ ✗ Token expirado                                     │
│ ✗ Token malformado                                   │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ 403 FORBIDDEN                                        │
│ ✗ Token válido pero permisos insuficientes          │
│ ✗ Usuario autenticado intenta acceder a ruta admin  │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ 500 INTERNAL SERVER ERROR                            │
│ ✗ Error en el servidor                              │
│ ✗ Error de base de datos                            │
│ ✗ Error al generar token                            │
└──────────────────────────────────────────────────────┘
```

---

## Comparación: Usuario Normal vs Administrador

```
┌─────────────────────────────────────────────────────────┐
│                     USUARIO NORMAL                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Puede acceder:                                         │
│    ✓ GET  /api/protected                               │
│    ✓ GET  /api/me                                      │
│    ✓ POST /api/posts (crear contenido)                │
│                                                         │
│  NO puede acceder:                                      │
│    ✗ GET  /api/admin                (403 Forbidden)    │
│    ✗ GET  /api/admin/dashboard      (403 Forbidden)    │
│    ✗ PUT  /api/users/:id            (403 Forbidden)    │
│    ✗ DELETE /api/users/:id          (403 Forbidden)    │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                     ADMINISTRADOR                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Puede acceder:                                         │
│    ✓ GET  /api/protected                               │
│    ✓ GET  /api/me                                      │
│    ✓ POST /api/posts                                   │
│    ✓ GET  /api/admin                                   │
│    ✓ GET  /api/admin/dashboard                         │
│    ✓ PUT  /api/users/:id                               │
│    ✓ DELETE /api/users/:id                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Ciclo de Vida del Token

```
   CREACIÓN            VALIDEZ             EXPIRACIÓN
      │                   │                     │
      ▼                   ▼                     ▼
┌─────────┐         ┌──────────┐         ┌──────────┐
│         │         │          │         │          │
│ Login   │────────►│ Token    │────────►│ Token    │
│ Exitoso │         │ Válido   │         │ Expirado │
│         │         │ (24h)    │         │          │
└─────────┘         └──────────┘         └──────────┘
     │                    │                     │
     │                    │                     │
     │                    ├─► Acceso OK         │
     │                    │                     │
     │                    │                     ├─► 401 Unauthorized
     │                    │                     │
     │                    │                     └─► Debe hacer login
     │                    │                         de nuevo
     │                    │
     └────────────────────┴────► Uso del token
                                 en cada request
```

---

## Mejores Prácticas Visualizadas

### ✅ Hacer
```
Cliente                          Servidor
  │                                │
  │  Authorization: Bearer token   │
  ├───────────────────────────────►│
  │                                │
  │  Enviar token en HEADER        │
  │  ✓ Seguro                      │
  │  ✓ Estándar                    │
```

### ❌ NO Hacer
```
Cliente                          Servidor
  │                                │
  │  ?token=abc123                 │
  ├───────────────────────────────►│
  │                                │
  │  Enviar token en URL           │
  │  ✗ Inseguro (logs)             │
  │  ✗ Queda en historial          │
```

---

## Arquitectura del Sistema

```
┌────────────────────────────────────────────────────────┐
│                      FRONTEND                          │
│                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ login.   │  │ balance. │  │ jwt-test.│           │
│  │ html     │  │ html     │  │ html     │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│                                                        │
│  ┌────────────────────────────────────────────┐      │
│  │ localStorage / sessionStorage              │      │
│  │ jwt_token: "eyJhbGc..."                   │      │
│  └────────────────────────────────────────────┘      │
└───────────────────┬────────────────────────────────────┘
                    │
                    │ HTTP Requests
                    │ Authorization: Bearer <token>
                    │
┌───────────────────▼────────────────────────────────────┐
│                     BACKEND                            │
│                                                        │
│  ┌────────────────────────────────────────────┐      │
│  │              Rocket Router                 │      │
│  └────────────────┬───────────────────────────┘      │
│                   │                                    │
│         ┌─────────┴─────────┐                         │
│         │                   │                         │
│         ▼                   ▼                         │
│  ┌────────────┐      ┌────────────┐                  │
│  │  Rutas     │      │  Rutas     │                  │
│  │  Públicas  │      │  Protegidas│                  │
│  │            │      │            │                  │
│  │ /login     │      │ /api/me    │◄─── Guardianes  │
│  │ /api/login │      │ /api/admin │     - Authenticated
│  └────────────┘      └────────────┘     - AdminUser  │
│                             │                         │
│                             ▼                         │
│                      ┌────────────┐                   │
│                      │   JWT      │                   │
│                      │   Utils    │                   │
│                      │            │                   │
│                      │ - create   │                   │
│                      │ - decode   │                   │
│                      │ - validate │                   │
│                      └────────────┘                   │
│                             │                         │
│                             ▼                         │
│                      ┌────────────┐                   │
│                      │ PostgreSQL │                   │
│                      │            │                   │
│                      │ - usuarios │                   │
│                      │ - asignaturas                  │
│                      └────────────┘                   │
└────────────────────────────────────────────────────────┘
```

---

**Este diagrama muestra el flujo completo del sistema JWT implementado.**

Para más detalles técnicos, consulta:
- `JWT_README.md` - Documentación completa
- `SECURITY_BEST_PRACTICES.md` - Guía de seguridad
- `jwt_examples.rs` - Ejemplos de código
