# Sistema de Autenticación JWT - Documentación

## 📋 Descripción General

Este proyecto implementa un sistema completo de autenticación basado en **JSON Web Tokens (JWT)** con guardianes de seguridad para Rocket Framework en Rust.

## 🔑 Características

- ✅ Autenticación mediante JWT
- ✅ Tokens con expiración configurable (por defecto 24 horas)
- ✅ Guardianes (Guards) para proteger rutas
- ✅ Sistema de permisos basado en roles (usuario/administrador)
- ✅ API RESTful con respuestas JSON
- ✅ Compatibilidad con formularios HTML tradicionales

## 🛠️ Estructura del Sistema

### Archivos Principales

```
backend/
├── src/
│   ├── utils/
│   │   ├── jwt.rs          # Lógica de JWT y guardianes
│   │   └── mod.rs
│   ├── routes/
│   │   ├── login.rs        # Rutas de autenticación y protegidas
│   │   └── mod.rs
│   ├── lib.rs              # Configuración principal
│   └── main.rs
├── Cargo.toml              # Dependencias
frontend/
└── jwt-test.html           # Página de prueba de JWT
```

## 📦 Dependencias

Las siguientes dependencias fueron añadidas en `Cargo.toml`:

```toml
jsonwebtoken = "9.3"
chrono = { version = "0.4", features = ["serde"] }
```

## 🔐 Componentes del Sistema JWT

### 1. Claims (jwt.rs)

La estructura `Claims` contiene la información del usuario en el token:

```rust
pub struct Claims {
    pub sub: String,      // ID del usuario
    pub email: String,    // Email del usuario
    pub name: String,     // Nombre del usuario
    pub is_admin: bool,   // Si es administrador
    pub exp: usize,       // Tiempo de expiración
    pub iat: usize,       // Tiempo de emisión
}
```

### 2. Guardianes (Guards)

#### AuthenticatedUser
Valida que el usuario esté autenticado. Se usa en rutas que requieren login.

```rust
#[get("/api/protected")]
pub fn protected_route(user: AuthenticatedUser) -> Json<Response> {
    // Solo usuarios autenticados pueden acceder
}
```

#### AdminUser
Valida que el usuario sea administrador. Se usa en rutas administrativas.

```rust
#[get("/api/admin")]
pub fn admin_route(admin: AdminUser) -> Json<Response> {
    // Solo administradores pueden acceder
}
```

## 🚀 Rutas Disponibles

### Rutas Públicas

#### `POST /api/login` (JSON)
Autentica un usuario y devuelve un token JWT.

**Request:**
```json
{
  "email": "usuario@ejemplo.com",
  "password": "contraseña123"
}
```

**Response (Éxito):**
```json
{
  "success": true,
  "message": "Login exitoso",
  "token": "<TOKEN>",
  "user": {
    "id": "1",
    "name": "Juan Pérez",
    "email": "usuario@ejemplo.com",
    "is_admin": false
  }
}
```

**Response (Error):**
```json
{
  "success": false,
  "message": "Contraseña incorrecta",
  "token": null,
  "user": null
}
```

#### `GET /login` - Página HTML de login
#### `POST /login` - Login con formulario (redirecciona)

### Rutas Protegidas (Requieren Autenticación)

Todas estas rutas requieren el header:
```
Authorization: Bearer <token_jwt>
```

#### `GET /api/protected`
Ruta de ejemplo para usuarios autenticados.

**Response:**
```json
{
  "message": "Acceso concedido a ruta protegida",
  "user": "Juan Pérez",
  "email": "usuario@ejemplo.com"
}
```

#### `GET /api/me`
Obtiene información del usuario actual.

**Response:**
```json
{
  "message": "Información del usuario actual",
  "user": "Juan Pérez",
  "email": "usuario@ejemplo.com"
}
```

### Rutas de Administrador (Requieren rol Admin)

#### `GET /api/admin`
Ruta exclusiva para administradores.

**Response (Admin):**
```json
{
  "message": "Acceso concedido a ruta de administrador",
  "admin": "Admin User"
}
```

**Response (No Admin):**
```
HTTP 403 Forbidden
```

#### `GET /api/admin/dashboard`
Dashboard administrativo.

## 💻 Uso desde el Frontend

### Usando JavaScript/Fetch API

```javascript
// 1. Login y obtener token
async function login(email, password) {
    const response = await fetch('/api/login', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
        },
        body: JSON.stringify({ email, password })
    });
    
    const data = await response.json();
    if (data.success) {
        // Guardar token en localStorage
        localStorage.setItem('jwt_token', data.token);
        return data.token;
    }
}

// 2. Hacer petición a ruta protegida
async function getProtectedData() {
    const token = localStorage.getItem('jwt_token');
    
    const response = await fetch('/api/protected', {
        method: 'GET',
        headers: {
            'Authorization': 'Bearer ' + token
        }
    });
    
    if (response.ok) {
        return await response.json();
    } else {
        console.error('No autorizado');
    }
}

// 3. Cerrar sesión
function logout() {
    localStorage.removeItem('jwt_token');
}
```

### Página de Prueba

Se incluye una página de prueba interactiva en:
```
http://localhost:8000/frontend/jwt-test.html
```

Esta página permite:
- ✅ Iniciar sesión y obtener token
- ✅ Probar rutas protegidas
- ✅ Probar rutas de administrador
- ✅ Ver el token JWT generado
- ✅ Gestionar el token en localStorage

## 🔒 Seguridad

### Importante: Cambiar la clave secreta

**⚠️ CRÍTICO:** En producción, debes cambiar la clave secreta en `jwt.rs`:

```rust
// NO usar esto en producción:
const JWT_SECRET: &str = "tu_clave_secreta_muy_segura_cambiar_en_produccion";

// Mejor práctica: Usar variable de entorno
use std::env;
let secret = env::var("JWT_SECRET").expect("JWT_SECRET debe estar definido");
```

Crea un archivo `.env`:
```env
JWT_SECRET=tu_clave_super_secreta_aleatoria_y_larga_aqui
```

### Mejores Prácticas

1. **HTTPS**: Siempre usar HTTPS en producción
2. **Expiración**: Los tokens expiran en 24 horas por defecto
3. **Headers**: Los tokens deben enviarse en el header `Authorization: Bearer <token>`
4. **Almacenamiento**: Guardar tokens en localStorage o cookies HttpOnly
5. **Validación**: Los tokens se validan en cada petición

## 🧪 Testing

### Prueba Manual

1. Ejecuta el servidor:
```bash
cargo run
```

2. Abre el navegador en:
```
http://localhost:8000/frontend/jwt-test.html
```

3. Prueba las siguientes credenciales (según tu base de datos):
   - Email: tu_usuario@ejemplo.com
   - Password: tu_contraseña

### Prueba con cURL

```bash
# Login
curl -X POST http://localhost:8000/api/login \
  -H "Content-Type: application/json" \
  -d '{"email":"usuario@ejemplo.com","password":"contraseña123"}'

# Ruta protegida
curl http://localhost:8000/api/protected \
  -H "Authorization: Bearer <tu_token_aqui>"

# Ruta admin
curl http://localhost:8000/api/admin \
  -H "Authorization: Bearer <tu_token_aqui>"
```

## 📝 Códigos de Estado HTTP

- **200 OK**: Petición exitosa
- **401 Unauthorized**: Token inválido o no proporcionado
- **403 Forbidden**: Token válido pero sin permisos suficientes
- **500 Internal Server Error**: Error del servidor

## 🎯 Próximos Pasos

Para extender el sistema, puedes:

1. **Agregar refresh tokens** para renovar tokens expirados
2. **Implementar blacklist de tokens** para logout
3. **Agregar más roles** (moderador, editor, etc.)
4. **Implementar cookies HttpOnly** como alternativa a localStorage
5. **Agregar rate limiting** para prevenir ataques de fuerza bruta
6. **Implementar 2FA** (autenticación de dos factores)

## 🐛 Solución de Problemas

### Error: "No autorizado"
- Verifica que el token esté en el header `Authorization: Bearer <token>`
- Verifica que el token no haya expirado
- Verifica que el formato del token sea correcto

### Error: "Prohibido" (403)
- Tu usuario no tiene permisos de administrador
- Verifica el campo `isadmin` en la base de datos

### Token no generado
- Verifica que las credenciales sean correctas
- Verifica la conexión a la base de datos
- Verifica que bcrypt esté validando correctamente

## 📚 Recursos Adicionales

- [JWT.io](https://jwt.io) - Debugger de JWT
- [Rocket Documentation](https://rocket.rs)
- [jsonwebtoken crate](https://docs.rs/jsonwebtoken)

---

**¡Sistema JWT implementado exitosamente! 🎉**
