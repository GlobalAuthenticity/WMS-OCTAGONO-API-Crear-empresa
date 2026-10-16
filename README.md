# User Data API (Octógono WMS)

> API REST con Bearer Token para consultar perfil, área, cursos y test de un usuario y crear empresas.

| Dato | Valor |
|---|---|
| Nombre del plugin | User Data API (Octógono WMS) |
| Slug / carpeta | `user-data-api` |
| Versión | 1.12.0 |
| Autor | Cristian |
| Requisitos | WordPress 5.8+ · PHP 7.4+ · tablas `{prefix}user_members` y `{prefix}areas` · LearnDash (opcional, para `courses`) |
| Namespace REST | `user-data-api/v1` |

---

## ¿Qué hace?

El plugin expone **tres rutas REST (POST)** protegidas con el mismo Bearer Token y un **panel de administración** para configurarlas y probarlas sin Postman.

| Ruta | Para qué sirve |
|---|---|
| `POST /wp-json/user-data-api/v1/lookup` | Consultar la información completa de **un** usuario (recomendada) |
| `POST /wp-json/user-data-api/v1/user` | Igual que `/lookup` (ruta original, se mantiene por compatibilidad) |
| `POST /wp-json/user-data-api/v1/company` | Crear una **empresa** (usuario `group_admin`) y, opcionalmente, sus **áreas** |

> **¿Por qué hay dos rutas para consultar?** Firewalls como Wordfence bloquean por patrón cualquier ruta REST terminada en `/user` o `/users` (protección anti-enumeración). `/lookup` hace exactamente lo mismo, pero el firewall no la bloquea.

---

## Autenticación

Todas las rutas usan el mismo token, guardado en la opción `uda_bearer_token`.

```
Authorization: Bearer TU_TOKEN
Content-Type: application/json
```

- Al **activar** el plugin se genera automáticamente un token aleatorio de 48 caracteres si no existe uno.
- Se compara con `hash_equals()`, así que la comparación tarda lo mismo sin importar el token (protege contra *timing attacks*).
- Si **no hay token configurado**, se rechazan todas las peticiones: la API nunca queda abierta por accidente.
- El token solo admite `A-Z a-z 0-9 - _ .`.

| Situación | HTTP | Código | Mensaje |
|---|---|---|---|
| Falta `Authorization` | 401 | `uda_unauthorized` | Falta la cabecera Authorization. |
| Formato distinto de `Bearer TOKEN` | 401 | `uda_unauthorized` | Formato de cabecera Authorization inválido… |
| No hay token en el servidor | 401 | `uda_unauthorized` | El token de autenticación no ha sido configurado… |
| Token incorrecto | 401 | `uda_unauthorized` | Token inválido. |

---

## 1. Consultar usuario — `POST /lookup` (o `/user`)

### Parámetros (body JSON)

Envía **uno** de los dos. Si llegan ambos, se usa `user_id`.

| Campo | Tipo | Descripción |
|---|---|---|
| `user_id` | integer | ID del usuario en WordPress |
| `email` | string | Correo exacto del usuario |

```bash
curl -X POST "https://octagono.wmslatam.com/wp-json/user-data-api/v1/lookup" \
  -H "Authorization: Bearer TU_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"usuario@ejemplo.com"}'
```

### Qué devuelve

| Bloque | Contenido | De dónde sale |
|---|---|---|
| Datos básicos | `id`, `email`, `username`, `first_name`, `last_name`, `full_name`, `roles`, `registered_at` | `wp_users` / `wp_usermeta` |
| `profile.completed` | `true` si los 7 campos obligatorios tienen valor: `first_name`, `last_name`, `user_age`, `user_gender`, `country`, `phone_number`, `user_position` | `wp_usermeta` |
| `area` | `id`, `name`, `quota`, `status`, `expires_at` y `membership` (`status`, `expires_at`) | Fila **más reciente** de `{prefix}user_members` + `{prefix}areas` |
| `company` | `id`, `name`, `email` | Usuario `grupo_id` del área, solo si tiene el rol `group_admin` |
| `courses` | Por curso: `id`, `title`, `status`, `progress_percentage`, `completed_steps`, `total_steps`, `completed` | Funciones oficiales de LearnDash |
| `test` | `completed`, `question_test_ids` (respuestas crudas) y `results` | Meta `question_test_ids` |

`company` y `area` vienen en `null` si el usuario no tiene membresía. `courses` viene vacío si LearnDash no está activo o el usuario no tiene cursos.

### Resultados del test "Sentido Vital"

Replica exactamente el cálculo del tema (`/resultado-test/` y los informes por área/empresa):

- **Promedio** = suma de los `value` / 17 (escala 0 a 3).
- **Clasificación**:

| Nivel | `key` | Rango |
|---|---|---|
| Búsqueda de sentido vital | `busqueda-sentido` | 0 – 1.99 |
| Medio sentido vital | `medio-sentido` | 2.00 – 2.34 |
| Alto sentido vital | `alto-sentido` | 2.35 – 2.64 |
| Plenitud de sentido | `plenitud-sentido` | 2.65 – 3 |

`results` incluye `average`, `scale`, `classification` (nivel obtenido con su descripción) y `levels` (los 4 niveles, para que quien consuma la API no tenga que duplicar la escala). Si el usuario no hizo el test: `{ "completed": false, "question_test_ids": null, "results": null }`.

### Ejemplo de respuesta (200)

```json
{
  "success": true,
  "user": {
    "id": 123,
    "email": "usuario@ejemplo.com",
    "username": "usuario",
    "first_name": "Juan",
    "last_name": "Pérez",
    "full_name": "Juan Pérez",
    "roles": ["subscriber"],
    "registered_at": "2026-01-01 10:00:00",
    "profile": { "completed": true },
    "company": { "id": 10, "name": "Empresa Ejemplo", "email": "contacto@empresa-ejemplo.com" },
    "area": {
      "id": 5,
      "name": "Recursos Humanos",
      "quota": 20,
      "status": "Activado",
      "expires_at": "2026-12-31",
      "membership": { "status": "ACTIVO", "expires_at": "31-12-2026" }
    },
    "courses": [
      {
        "id": 120,
        "title": "Curso de ejemplo",
        "status": "in_progress",
        "progress_percentage": 65,
        "completed_steps": 13,
        "total_steps": 20,
        "completed": false
      }
    ],
    "test": {
      "completed": true,
      "question_test_ids": [{ "question": 1, "answer": 2, "value": 1 }],
      "results": {
        "average": 2.12,
        "scale": { "min": 0, "max": 3, "question_count": 17 },
        "classification": {
          "key": "medio-sentido",
          "label": "Medio sentido vital",
          "description": "Personas que entienden algunas de las razones por las que actúan en su vida…",
          "range": { "min": 2, "max": 2.34 }
        },
        "levels": ["… los 4 niveles …"]
      }
    }
  }
}
```

### Errores

| HTTP | Mensaje |
|---|---|
| 400 | `Debes enviar "user_id" o "email".` |
| 400 | `El formato del correo electrónico no es válido.` |
| 404 | `Usuario no encontrado.` |

---

## 2. Crear empresa — `POST /company`

Crea un usuario de WordPress con rol `group_admin` (la "empresa") y sus áreas en la misma petición. Replica la lógica de los botones **"Crear Empresa"** y **"Crear Área"** del plugin *Administrar Empresa Octágono*, pero no depende de él.

### Parámetros (body JSON)

| Campo | Obligatorio | Si se omite, llega vacío o `null` |
|---|---|---|
| `nombre_empresa` | **Sí** — único (sin distinguir mayúsculas) | — |
| `user_email` | No | Se autogenera `admin.<slug>@wmslatam.com` |
| `nombre_encargado` | No | `José Martínez` |
| `contacto` | No | Celular aleatorio de 10 dígitos que empieza por `3` |
| `ocupacion` | No | `Administrador WMS` |
| `areas` | No | Se crea **una** área: `Área demo`, 2 cupos, vence hoy + 6 meses |

**Cada elemento de `areas`:**

| Campo | Obligatorio | Regla |
|---|---|---|
| `nombre` | Sí | Texto no vacío |
| `cupos` | Sí | Entero ≥ 1 |
| `fecha_expiracion` | No | `AAAA-MM-DD`, no anterior a hoy. Por defecto, hoy + 6 meses |

`codigo` y `licencia_id` se aceptan sin error, pero **no se guardan** (este sitio no maneja licencias).

```bash
curl -X POST "https://octagono.wmslatam.com/wp-json/user-data-api/v1/company" \
  -H "Authorization: Bearer TU_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "nombre_empresa": "Acme Demo S.A.S.",
    "areas": [
      { "nombre": "Ventas", "cupos": 20 },
      { "nombre": "Recursos Humanos", "cupos": 5, "fecha_expiracion": "2027-03-23" }
    ]
  }'
```

### Cómo se genera el correo automático

1. Se toma `nombre_empresa`, se quitan los acentos y se pasa a minúsculas.
2. Se elimina la palabra **"demo"** cuando aparece como palabra completa, en cualquier posición.
3. Se eliminan los espacios y todo lo que no sea letra o número (máx. 40 caracteres).
4. Resultado: `admin.<slug>@wmslatam.com`. Ej.: `Acme Demo S.A.S.` → `admin.acmesas@wmslatam.com`.
5. Si el correo ya existe, se prueba `admin.<slug>2@…` hasta `20`, y como último recurso un sufijo aleatorio.
6. Si el slug queda vacío (ej. `"Demo"`), se usa `empresa<6 letras aleatorias>`.

### Qué crea internamente

**Usuario empresa**

- `user_login` y `user_email`: el correo final.
- `nickname` y `first_name`: `nombre_empresa`.
- Contraseña aleatoria de 14 caracteres alfanuméricos.
- Rol `group_admin`.
- Metas: `nombre_encargado`, `phone_number`, `occupation`, `grupo = true`, `membresia_activa = true`.
- Metas de autologin (`mi_autologin_token`, `mi_autologin_expires` = 15 min). Se generan, pero desde la v1.8.1 la URL ya no se devuelve.

**Áreas** (`{prefix}areas`): `nombre`, `cupo`, `estado = Activado`, `tipo = companies`, `grupo_id` = ID de la empresa y `fecha_expiracion`.

### Reglas de consistencia

- **Todo el arreglo `areas` se valida antes de crear nada.** Si una sola área viene mal, se responde 400 y no se crea ni la empresa ni ninguna área.
- Si la empresa ya se creó y falla el `INSERT` de un área puntual, la empresa y las demás áreas se conservan. Esa área aparece en la respuesta con `success: false` para reintentarla.

### Respuesta exitosa (201)

```json
{
  "success": true,
  "message": "La empresa se ha registrado correctamente.",
  "empresa": {
    "user_id": 456,
    "nombre_empresa": "Acme Demo S.A.S.",
    "email": "admin.acmesas@wmslatam.com",
    "password": "a8Kd73LmQp20Xz",
    "rol": "company"
  },
  "areas": [
    { "id": 31, "nombre": "Ventas", "cupos": 20, "fecha_expiracion": "2027-04-16", "success": true },
    { "id": 32, "nombre": "Recursos Humanos", "cupos": 5, "fecha_expiracion": "2027-03-23", "success": true }
  ],
  "defaults_aplicados": ["email", "nombre_encargado", "contacto", "ocupacion"]
}
```

- `empresa.rol` es solo una etiqueta legible (`company`); el rol real en WordPress es `group_admin`.
- `empresa.password` va **en texto plano**. Es el único canal para entregarla: el endpoint no envía correo de bienvenida ni sincroniza con otro sitio.
- `defaults_aplicados` lista los campos que se completaron automáticamente (`email`, `nombre_encargado`, `contacto`, `ocupacion`, `areas`).

### Errores

| HTTP | Código interno | Causa |
|---|---|---|
| 400 | `nombre_requerido` | Falta `nombre_empresa` |
| 400 | `correo_invalido` | `user_email` con formato inválido |
| 400 | `areas_invalidas` / `area_invalida` | `areas` no es un arreglo, o un elemento no es objeto |
| 400 | `area_nombre_requerido` | Un área sin `nombre` |
| 400 | `area_cupos_invalido` | `cupos` ausente o menor que 1 |
| 400 | `area_fecha_invalida` / `area_fecha_pasada` | Fecha ilegible o anterior a hoy |
| 409 | `empresa_existente` | Ya hay una empresa con ese nombre |
| 409 | `usuario_existente` | Ya existe un usuario con ese correo |

Forma del error: `{ "success": false, "message": "..." }`.

---

## Panel de administración

Menú lateral **User Data API** (icono de API), solo para usuarios con `manage_options`:

**Submenú "API info usuario"**
- Ver, copiar y regenerar el token. El botón "Generar token seguro" lo guarda de inmediato.
- Documentación del endpoint de consulta y comando cURL listo para copiar.
- Herramienta de **previsualización**: buscar un usuario con filtros por rol y por estado del test (realizó / no realizó) y ver exactamente el JSON que devolvería la API, con tarjetas visuales y barras de progreso de cursos.

**Submenú "API Crear empresa"**
- Documentación del endpoint `/company`.
- Formulario de prueba que **crea una empresa real** (con áreas en JSON libre) sin usar Postman.

En ambas páginas, el botón **"Descargar documentación (.md)"** genera un Markdown por endpoint (`user-data-api-consultar-usuario.md` y `user-data-api-crear-empresa.md`), pensado como referencia para agentes de IA. El token aparece como `TU_TOKEN_AQUI`.

Las acciones AJAX (`uda_generate_token`, `uda_search_users`, `uda_preview_user`, `uda_crear_empresa_test`) validan nonce y capacidad.

---

## Estructura de archivos

```
user-data-api/
├── user-data-api.php            # Bootstrap, constantes, activación/desactivación
├── uninstall.php                # Borra la opción uda_bearer_token
├── readme.txt
├── assets/
│   ├── css/admin.css            # Estilos del panel (identidad WMS Latam)
│   └── js/admin.js              # Lógica del panel (búsqueda, preview, pruebas, copiar)
└── includes/
    ├── class-uda-loader.php     # Carga e inicializa las clases
    ├── class-uda-settings.php   # Opción del token (Settings API)
    ├── class-uda-auth.php       # Validación del Bearer Token
    ├── class-uda-api.php        # Rutas /user y /lookup
    ├── class-uda-user-data.php  # Arma el perfil: empresa, área, cursos, test
    ├── class-uda-company-api.php# Ruta /company
    ├── class-uda-company.php    # Creación de la empresa (group_admin)
    ├── class-uda-area.php       # Validación y creación de áreas
    └── class-uda-admin.php      # Menús, páginas, AJAX, descarga de .md
```

---

## Instalación

1. **Plugins → Añadir plugin → Subir plugin** → `user-data-api.zip` → **Instalar ahora** → **Activar**.
2. Ir a **User Data API → API info usuario** y copiar el token generado (o generar uno nuevo).
3. Probar con el cURL de ejemplo, o con la previsualización del panel.

Al **desinstalar** se elimina la opción `uda_bearer_token`. Los usuarios y las áreas creados no se tocan.

---

## Seguridad

- Bearer Token comparado con `hash_equals()`. Sin token configurado, no responde a nadie.
- Consulta de **un solo usuario por petición**, por ID o correo exactos: no permite listar ni enumerar usuarios.
- No expone `user_pass`, hashes, activation keys ni otros metadatos internos.
  - **Excepción:** `/company` devuelve la contraseña recién generada de la empresa.
- Todas las consultas usan `$wpdb->prepare()` y `$wpdb->prefix`.
- Panel protegido con nonces y `current_user_can('manage_options')`.

---

## Consideraciones

- **Contraseña en texto plano:** quien consuma `/company` debe transmitirla por HTTPS y no guardarla en logs.
- **Prueba del panel = datos reales:** el formulario "API Crear empresa" crea empresas y áreas de verdad en la base de datos.
- **Membresía única:** si un usuario tiene varias filas en `user_members`, solo se reporta la más reciente (mayor `id`).
- **Autologin sin uso:** los metas `mi_autologin_*` se siguen generando al crear la empresa, aunque la URL ya no se devuelva.
- **Sin rollback de áreas:** si falla el `INSERT` de un área después de crear la empresa, la empresa no se revierte.
