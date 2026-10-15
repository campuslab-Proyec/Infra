# Guía del tenant de Azure (Microsoft Entra ID) para CampusLab

Todo lo necesario para administrar el inicio de sesión de CampusLab: qué hay configurado en Azure, cómo dar acceso a otra persona, cómo agregar usuarios, cómo revisar o rehacer la configuración y cómo resolver los errores `AADSTS`.

- Solo asignar roles a usuarios: [TUTORIAL-ROLES.md](TUTORIAL-ROLES.md)
- Desplegar el proyecto: [NOTAS-DESPLIEGUE.md](NOTAS-DESPLIEGUE.md)

---

## 1. Conceptos básicos

| Término | Qué es |
|---|---|
| **Tenant (inquilino / directorio)** | La "organización" de Entra ID donde viven los usuarios y las apps. CampusLab usa el directorio **appclientM**. |
| **Registro de aplicación** | La *definición* de una app: sus URIs de redirección, la API que expone, sus roles y permisos. Está en **Registros de aplicaciones**. |
| **Aplicación empresarial** | La *instancia* de esa app dentro del tenant. Ahí se asignan usuarios y roles. Está en **Aplicaciones empresariales**. |
| **Scope** | Un permiso que una API publica. Aquí es `access_as_user`; el frontend lo pide para poder llamar al BFF. |
| **Rol de aplicación** | Una etiqueta (Admin, Docente o Estudiante) que se asigna a usuarios y viaja dentro del token en el claim `roles`. |

Hay **dos** apps registradas:

- **frontend-campuslab**: la SPA de Angular. Es la que muestra el login de Microsoft.
- **ms-campuslab-bff**: la API. Valida el token y define los roles.

## 2. Datos del tenant y de las apps

| Dato | Valor |
|---|---|
| Directorio | appclientM |
| ID de directorio (tenant) | `aff2eb38-d6cf-4220-8527-53f1cb6aed15` |
| frontend-campuslab, ID de cliente | `718a6ef2-45a4-4f06-84c9-6f54de8c3514` |
| ms-campuslab-bff, ID de cliente | `8d19bb45-1842-43ce-b76c-4513b979df81` |
| ms-campuslab-bff, URI de ID de aplicación | `api://8d19bb45-1842-43ce-b76c-4513b979df81` |
| Scope que pide el frontend | `api://8d19bb45-1842-43ce-b76c-4513b979df81/access_as_user` |
| Tipos de cuenta (ambas apps) | Cuentas de cualquier directorio y cuentas personales de Microsoft |
| URL de la app | `https://handoff-sprint-protrude.ngrok-free.dev` |

Estos IDs no son secretos: cualquiera que abra la app en el navegador los puede ver. Lo que **nunca** debe subirse a GitHub son *secretos de cliente* o *certificados*, y CampusLab no usa ninguno.

## 3. Entrar al tenant correcto

1. Entra a <https://entra.microsoft.com> (o <https://portal.azure.com>).
2. Mira arriba a la derecha, bajo tu nombre: debe decir **appclientM**.
3. Si dice otro directorio (por ejemplo, el de Duoc), haz clic en tu cuenta → **Cambiar directorio** → elige **appclientM**.

Si appclientM no aparece en la lista, tu cuenta no pertenece a ese tenant. Pídele a quien lo administra que te invite (sección 5).

## 4. Dar acceso de administración a un compañero

Para que un compañero pueda gestionar CampusLab en Azure:

1. Primero agrégalo al tenant como usuario (sección 5).
2. Ve a **Identidad → Roles y administradores → Roles y administradores**.
3. Busca y abre **Administrador de aplicaciones** (*Application Administrator*).
4. Haz clic en **+ Agregar asignaciones**, selecciona al compañero y presiona **Agregar**.

Con ese rol puede editar ambos registros de aplicación, asignar roles a usuarios y dar consentimiento de administrador.

**Alternativa con menos permisos:** si solo debe tocar estas dos apps, en **Registros de aplicaciones → [app] → Propietarios → + Agregar propietarios**, agrégalo en ambas apps. Así puede editar sus registros, pero no administrar el resto del tenant. Para que también pueda asignar roles a usuarios, agrégalo además como **propietario** de **ms-campuslab-bff** en **Aplicaciones empresariales**.

Para **crear o invitar usuarios** se necesita además el rol **Administrador de usuarios** (*User Administrator*), o el de **Invitador de usuarios** (*Guest Inviter*) si solo va a invitar cuentas externas.

## 5. Usuarios: crear o invitar

Solo pueden iniciar sesión en CampusLab las cuentas que existen en appclientM.

**Crear un usuario del tenant** (la opción más simple):

1. **Identidad → Usuarios → Todos los usuarios → + Nuevo usuario → Crear nuevo usuario**.
2. Completa el *nombre principal de usuario* (por ejemplo, `prueba`) y el nombre para mostrar.
3. Copia la contraseña generada → **Revisar y crear → Crear**.
4. La persona entra con `prueba@<dominio>.onmicrosoft.com`. En el primer inicio de sesión le pedirá cambiar la contraseña.

**Invitar una cuenta externa** (por ejemplo, `@duocuc.cl` o `@gmail.com`):

1. **Identidad → Usuarios → Todos los usuarios → + Nuevo usuario → Invitar a un usuario externo**.
2. Escribe el correo → **Revisar e invitar → Invitar**.
3. La persona abre el correo y hace clic en **Aceptar invitación**.

Algunas instituciones bloquean las invitaciones externas. Si la invitación no llega o falla, crea un usuario del tenant.

Después de crear o invitar al usuario, asígnale un rol: [TUTORIAL-ROLES.md](TUTORIAL-ROLES.md).

## 6. Configuración de ms-campuslab-bff (la API)

Ruta: **Identidad → Aplicaciones → Registros de aplicaciones → ms-campuslab-bff**.

### 6.1 Exponer una API
- **URI de ID de aplicación:** `api://8d19bb45-1842-43ce-b76c-4513b979df81`.
- Debe existir el scope **`access_as_user`** con estado **Habilitado**. Para crearlo: **+ Agregar un ámbito**, nombre `access_as_user`, *¿Quién puede dar consentimiento?* = **Administradores y usuarios**, completa los textos y **Agregar ámbito**.

### 6.2 Roles de aplicación
Debe haber tres roles, todos con *Tipos de miembro permitidos* = **Usuarios o grupos** y habilitados:

| Nombre para mostrar | Valor (exacto) | Descripción |
|---|---|---|
| Administrador | `Admin` | Gestiona catálogo, reservas, auditoría y reportes |
| Docente | `Docente` | Aprueba y cambia el estado de reservas |
| Estudiante | `Estudiante` | Crea y consulta sus reservas |

El **Valor** es lo que revisa el código (`APPROLE_Admin`, etc.), así que tiene que coincidir exactamente, con mayúscula inicial y sin tildes.

### 6.3 Versión del token
En **Manifiesto**, `"requestedAccessTokenVersion"` (en el manifiesto antiguo, `"accessTokenAcceptedVersion"`) debe ser **2**. Con el tipo de cuenta actual, Azure lo exige de todos modos. Los backends esperan el issuer `https://login.microsoftonline.com/aff2eb38-.../v2.0`.

## 7. Configuración de frontend-campuslab (la SPA)

Ruta: **Registros de aplicaciones → frontend-campuslab**.

### 7.1 Autenticación
En **Autenticación**, dentro de la plataforma **Aplicación de página única (SPA)**, debe haber estas URIs de redirección:

| URI | Para qué |
|---|---|
| `https://handoff-sprint-protrude.ngrok-free.dev/` | Producción (EC2 + ngrok) |
| `http://localhost/` | Correrlo con Docker en un PC |
| `http://localhost:4200/` | Desarrollo con `ng serve` |

- Tienen que estar en la plataforma **SPA**, no en **Web**. Si están en Web, el login falla con `AADSTS9002326`.
- Se escriben exactamente así, con la barra `/` al final.
- Azure no acepta `http://` salvo para `localhost`. Por eso, para entrar desde internet hace falta ngrok (HTTPS).

### 7.2 Permisos de API
En **Permisos de API** debe aparecer **ms-campuslab-bff → access_as_user**:

1. Si no está: **+ Agregar un permiso → Mis API → ms-campuslab-bff → Permisos delegados → access_as_user → Agregar permisos**.
2. Presiona **Conceder consentimiento de administrador para appclientM** → **Sí**. La columna *Estado* debe quedar con un ✅ verde.

Sin el consentimiento, cada usuario ve una pantalla pidiéndole aceptar permisos la primera vez que entra.

## 8. Asignación de roles y acceso a la app

Ruta: **Identidad → Aplicaciones → Aplicaciones empresariales → ms-campuslab-bff**.

- **Usuarios y grupos:** aquí se asigna el rol a cada persona. Paso a paso en [TUTORIAL-ROLES.md](TUTORIAL-ROLES.md).
- **Propiedades → ¿Asignación requerida?**
  - **No** (actual): cualquier usuario del tenant puede entrar. Si no tiene rol, queda con permisos de Estudiante.
  - **Sí**: solo entran los usuarios con un rol asignado; al resto, Azure los bloquea en el login.

Un cambio de rol se aplica cuando el usuario **cierra sesión y vuelve a entrar**.

## 9. Dónde aparecen estos datos en el código

Si alguna vez se cambia de tenant o se vuelven a crear las apps, hay que actualizar estos archivos y subirlos a GitHub:

| Archivo | Qué contiene |
|---|---|
| `ngfrontend-campuslab/src/environments/environment.ts` | `TENANT_ID`, `CLIENT_ID` (frontend) y `BFF_SCOPE` |
| `ngfrontend-campuslab/src/environments/environment.development.ts` | Lo mismo, para `ng serve` |
| `ms-campuslab-bff/src/main/resources/application.yaml` | `issuer-uri`, `jwk-set-uri` y `audiences` (ID del BFF) |
| `ms-campuslab-bookings/src/main/resources/application.yaml` | `issuer-uri` y `audiences` |
| `ms-campuslab-audit/src/main/resources/application.yaml` | `issuer-uri` y `audiences` |
| `ms-campuslab-report/src/main/resources/application.yaml` | `issuer-uri` y `audiences` |

En bookings, audit y report, el tenant se puede sobrescribir sin tocar el código con la variable de entorno `AZURE_TENANT_ID` en `Infra/compose.yaml`.

## 10. Rehacer todo en un tenant nuevo (checklist)

Por si hay que mover CampusLab a otro tenant (por ejemplo, el de un compañero):

1. [ ] Registrar **ms-campuslab-bff**: *Registros de aplicaciones → + Nuevo registro*, sin URI de redirección.
2. [ ] En el BFF: **Exponer una API** → definir el URI de ID de aplicación (`api://<id-del-bff>`) y crear el scope `access_as_user` (6.1).
3. [ ] En el BFF: crear los roles `Admin`, `Docente` y `Estudiante` (6.2).
4. [ ] Registrar **frontend-campuslab** con plataforma **SPA** y las URIs de la sección 7.1.
5. [ ] En el frontend: agregar el permiso `access_as_user` y conceder consentimiento de administrador (7.2).
6. [ ] Crear o invitar a los usuarios (sección 5) y asignarles roles (sección 8).
7. [ ] Actualizar los IDs en los archivos de la sección 9, compilar y subir a GitHub.
8. [ ] En la EC2: actualizar los repos y `docker compose up -d --build` (ver [NOTAS-DESPLIEGUE.md](NOTAS-DESPLIEGUE.md)).

## 11. Errores de inicio de sesión

| Error | Qué significa | Solución |
|---|---|---|
| **AADSTS650052** "...your organization lacks a service principal..." | La cuenta pertenece a otra organización (por ejemplo, Duoc) y la autenticó su propio tenant. También pasa si se está sirviendo una versión vieja del frontend que usa `/common`. | Invitar la cuenta a appclientM (sección 5). Verificar que el sitio sirva el frontend actualizado. |
| **AADSTS50020** "User account ... does not exist in tenant" | La cuenta no está en appclientM. | Invitarla o crear un usuario del tenant (sección 5). |
| **AADSTS50011** "The redirect URI ... does not match" | La URL desde la que se abre la app no está registrada. | Agregar esa URL exacta, con `/` final, en la plataforma SPA (7.1). |
| **AADSTS9002326** "Cross-origin token redemption is permitted only for the 'Single-Page Application' client-type" | La URI de redirección está registrada en la plataforma *Web*. | Borrarla de Web y agregarla en **SPA**. |
| **AADSTS65001** "The user or administrator has not consented..." | Falta el consentimiento del permiso `access_as_user`. | Conceder consentimiento de administrador (7.2). |
| **AADSTS500011** "The resource principal named api://... was not found" | El scope apunta a una API que no existe en este tenant. | Revisar el URI de ID de aplicación del BFF (6.1) y `BFF_SCOPE` en `environment.ts`. |
| **AADSTS50105** "...not assigned to a role for the application" | *Asignación requerida* = Sí y el usuario no tiene rol. | Asignarle un rol (sección 8). |
| La app responde **401** en todas las llamadas | El token es de otro tenant o para otra API (issuer o audiencia incorrectos). | Revisar la sección 9; iniciar sesión con una cuenta de appclientM. |
| Aparece **"Sin rol"** en el encabezado | No tiene rol asignado, o no volvió a iniciar sesión después de asignarlo. | Asignar el rol en **ms-campuslab-bff** y volver a entrar. |

**Consejo:** prueba siempre en una **ventana de incógnito**. MSAL guarda la sesión en `localStorage` y puede reutilizar una cuenta anterior aunque cierres sesión.
