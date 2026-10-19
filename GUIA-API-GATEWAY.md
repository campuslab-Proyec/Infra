# API Gateway de AWS para CampusLab

El frontend llama al BFF a través de un **API Gateway (HTTP API) de AWS** que valida el JWT de Entra ID con un *authorizer* antes de reenviar la petición.

```
Navegador ──HTTPS──> API Gateway ──(authorizer JWT: Entra ID)──> ngrok ──> EC2:80 nginx ──/api──> BFF :8080
   (Angular + MSAL, token en Authorization)        │
                                                    └── sin token o token inválido: 401 {"message":"Unauthorized"}
```

- El BFF sigue validando el JWT por su cuenta (defensa en profundidad) y aplica los roles.
- La integración apunta al dominio de ngrok porque es estable; la IP pública de la EC2 cambia en cada reinicio del laboratorio.
- El frontend sabe a qué URL llamar por la variable `API_BASE_URL` del contenedor (ver el paso 6). Vacía = llama al BFF por su mismo origen, como antes.

## Datos que se usan

| Dato | Valor |
|---|---|
| Región | `us-east-1` (la del laboratorio) |
| Tenant de Entra ID | `aff2eb38-d6cf-4220-8527-53f1cb6aed15` |
| Emisor (issuer) del token | `https://login.microsoftonline.com/aff2eb38-d6cf-4220-8527-53f1cb6aed15/v2.0` |
| Audiencia (API ms-campuslab-bff) | `8d19bb45-1842-43ce-b76c-4513b979df81` y `api://8d19bb45-1842-43ce-b76c-4513b979df81` |
| Backend | `https://handoff-sprint-protrude.ngrok-free.dev` |

## 1. Crear la API

1. En la consola de AWS, busca **API Gateway** → **Create API** → **HTTP API** → **Build**.
2. **API name:** `campuslab-api`.
3. En **Integrations**, de momento no agregues ninguna → **Next** → **Next** (las rutas se crean después).
4. **Stage:** deja `$default` con **Auto-deploy** activado → **Next** → **Create**.

## 2. Integración hacia el BFF

1. Menú izquierdo → **Integrations** → **Manage integrations** → **Create**.
2. **Integration type:** *HTTP URI*.
3. **HTTP method:** `ANY`.
4. **URL:** `https://handoff-sprint-protrude.ngrok-free.dev/api/{proxy}`
5. **Create**.

### Parámetro para ngrok
ngrok gratuito muestra una página de advertencia a los navegadores. Para que no la devuelva al gateway:

1. Abre la integración recién creada → **Parameter mapping** → **Create parameter mapping**.
2. **Mapping type:** *All incoming requests*.
3. Agrega: **Modify** `header.ngrok-skip-browser-warning` → **Append** → valor `true`.
4. **Create**.

## 3. Ruta

1. Menú izquierdo → **Routes** → **Create**.
2. Método `ANY`, ruta `/api/{proxy+}` → **Create**.
3. Selecciona la ruta → **Attach integration** → elige la integración del paso 2 → **Attach integration**.

## 4. Authorizer JWT (Entra ID)

1. Menú izquierdo → **Authorization** → pestaña **Manage authorizers** → **Create**.
2. **Authorizer type:** *JWT*.
3. **Name:** `entra-id`.
4. **Identity source:** `$request.header.Authorization`
5. **Issuer URL:** `https://login.microsoftonline.com/aff2eb38-d6cf-4220-8527-53f1cb6aed15/v2.0`
6. **Audience:** agrega las dos:
   - `8d19bb45-1842-43ce-b76c-4513b979df81`
   - `api://8d19bb45-1842-43ce-b76c-4513b979df81`
7. **Create**.
8. Pestaña **Attach authorizers to routes** → selecciona `ANY /api/{proxy+}` → elige `entra-id` → **Attach authorizer**.

## 5. CORS

El frontend (dominio de ngrok o `localhost`) llama al gateway (otro dominio), así que hay que permitirlo:

1. Menú izquierdo → **CORS** → **Configure**.
2. **Access-Control-Allow-Origin:** `https://handoff-sprint-protrude.ngrok-free.dev` y `http://localhost`
3. **Access-Control-Allow-Headers:** `authorization`, `content-type`
4. **Access-Control-Allow-Methods:** `GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`
5. **Access-Control-Max-Age:** `300` → **Save**.

Con CORS configurado, API Gateway responde solo las peticiones `OPTIONS` (preflight) sin pasar por el authorizer.

## 6. Conectar el frontend al gateway

1. En la API, menú **API: campuslab-api** → copia el **Invoke URL** (por ejemplo `https://abc123xyz.execute-api.us-east-1.amazonaws.com`).
2. En la EC2 (consola de AWS), crea el archivo `~/Infra/.env` con esa URL (sin `/` final):
   ```bash
   cat > ~/Infra/.env <<'EOF'
   FRONTEND_DIR=../frontend-campuslab
   API_BASE_URL=https://abc123xyz.execute-api.us-east-1.amazonaws.com
   EOF
   ```
3. Reinicia solo el frontend:
   ```bash
   cd ~/Infra && docker compose up -d frontend-campuslab
   docker logs frontend-campuslab 2>&1 | grep campuslab
   ```
   Debe mostrar `campuslab: API_BASE_URL=https://abc123xyz...`.

`Infra/.env` no se sube a GitHub (está en `.gitignore`), así que `git reset --hard` no lo borra. Para volver a llamar al BFF directo, deja `API_BASE_URL=` vacío y reinicia el frontend.

## 7. Verificar

- **Sin token, el gateway rechaza:**
  ```bash
  curl -i https://abc123xyz.execute-api.us-east-1.amazonaws.com/api/bff/me
  ```
  Debe responder `401` con `{"message":"Unauthorized"}` (lo responde el authorizer, la petición no llega al BFF).
- **Desde la app:** abre https://handoff-sprint-protrude.ngrok-free.dev, inicia sesión y en F12 → **Network** las llamadas deben ir a `https://abc123xyz.execute-api.../api/bff/...` con respuesta `200`.

## Problemas comunes

| Síntoma | Causa | Solución |
|---|---|---|
| `401 {"message":"Unauthorized"}` estando con sesión | El issuer o la audiencia del authorizer no coinciden con el token | Revisar el paso 4. El token debe ser v2 (`requestedAccessTokenVersion = 2` en el manifiesto de ms-campuslab-bff) |
| Error de CORS en la consola del navegador | Falta el origen o el header en la configuración CORS | Revisar el paso 5; el origen debe coincidir exacto, sin `/` final |
| La respuesta es una página HTML de ngrok | Falta el header `ngrok-skip-browser-warning` | Revisar el *parameter mapping* del paso 2 |
| `503` o `504` | ngrok o la EC2 no responden (laboratorio detenido) | Iniciar el laboratorio y revisar `systemctl status ngrok` en la EC2 |
| Las llamadas siguen yendo al dominio de ngrok | El frontend no tomó `API_BASE_URL` | Revisar `~/Infra/.env` y reiniciar el frontend (paso 6); recargar con `Ctrl+F5` |
