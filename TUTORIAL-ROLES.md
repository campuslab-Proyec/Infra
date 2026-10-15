# Cómo asignar roles a los usuarios de CampusLab

CampusLab usa los **roles de aplicación de Microsoft Entra ID** para decidir qué puede hacer cada usuario. Los roles se asignan desde el portal de Azure; no hay que tocar código.

## Qué permite cada rol

| Acción | Estudiante / sin rol | Docente | Admin |
|---|:---:|:---:|:---:|
| Ver catálogo | ✅ | ✅ | ✅ |
| Crear laboratorios y recursos | ❌ | ❌ | ✅ |
| Ver y crear reservas | ✅ | ✅ | ✅ |
| Cambiar el estado de una reserva | ❌ | ✅ | ✅ |
| Ver Auditoría y Reportes | ❌ | ❌ | ✅ |

Un usuario sin rol asignado puede entrar y queda con los mismos permisos que un Estudiante.

## Antes de empezar

- Necesitas una cuenta con permisos de administración en el directorio **appclientM** (tenant `aff2eb38-d6cf-4220-8527-53f1cb6aed15`).
- El usuario al que le vas a asignar el rol **tiene que existir en ese directorio**. Si no existe, primero agrégalo (ver la sección "Agregar un usuario nuevo").

## Asignar un rol (paso a paso)

1. Entra a <https://portal.azure.com> o a <https://entra.microsoft.com>.
2. Arriba a la derecha, verifica que estás en el directorio **appclientM**. Si no, haz clic en tu cuenta → **Cambiar directorio**.
3. Busca **Aplicaciones empresariales** en la barra de búsqueda y ábrela.
4. Abre la aplicación **ms-campuslab-bff**.

   > Ojo: debe ser **ms-campuslab-bff**, no frontend-campuslab. Los roles están definidos en el BFF.

5. En el menú izquierdo, entra a **Usuarios y grupos**.
6. Haz clic en **+ Agregar usuario o grupo**.
7. En **Usuarios**, haz clic en *Ninguno seleccionado*, busca a la persona, márcala y presiona **Seleccionar**.
8. En **Seleccionar un rol**, elige **Administrador**, **Docente** o **Estudiante** y presiona **Seleccionar**.
9. Haz clic en **Asignar**.

Listo: la persona aparece en la lista con su rol.

## El usuario debe volver a iniciar sesión

El rol viaja dentro del token de inicio de sesión, así que **no se aplica hasta que la persona cierre sesión y vuelva a entrar** en CampusLab.

Para comprobar que funcionó, mira el encabezado de la aplicación junto al botón **Cerrar Sesión**: debe aparecer el nombre del usuario y su rol, por ejemplo `Juan Pérez · Docente`. Si aparece **"Sin rol"**, revisa la sección de problemas comunes.

## Cambiar o quitar un rol

En **Aplicaciones empresariales → ms-campuslab-bff → Usuarios y grupos**:

- **Quitar:** marca al usuario y presiona **Quitar**.
- **Cambiar:** quita la asignación actual y vuelve a asignar el rol nuevo. Una persona puede tener más de un rol; en ese caso obtiene los permisos de ambos.

Igual que al asignar, el cambio se aplica cuando el usuario vuelve a iniciar sesión.

## Agregar un usuario nuevo al directorio

Si la persona no aparece al buscarla en el paso 7, primero hay que agregarla:

1. Ve a **Usuarios → Todos los usuarios → + Nuevo usuario**.
2. Elige una de estas opciones:
   - **Crear nuevo usuario:** crea una cuenta `nombre@...onmicrosoft.com`. Copia la contraseña que se genera y entrégasela a la persona; en el primer inicio de sesión le pedirá cambiarla.
   - **Invitar a un usuario externo:** escribe el correo de la persona (por ejemplo, su cuenta `@duocuc.cl`). Le llega un correo y debe hacer clic en **Aceptar invitación**.
3. Después, vuelve a **Asignar un rol** y asígnale uno.

## Problemas comunes

| Síntoma | Causa | Solución |
|---|---|---|
| Aparece **"Sin rol"** en el encabezado | El usuario no cerró sesión después de la asignación, o el rol se asignó en la app equivocada | Cierra sesión y vuelve a entrar. Verifica que el rol esté en **ms-campuslab-bff** |
| No aparecen **Auditoría** ni **Reportes** | El usuario no es Admin | Asígnale el rol **Administrador** |
| Error **403** al guardar o al cambiar un estado | El rol no tiene permiso para esa acción | Revisa la tabla del inicio |
| Error **AADSTS650052** al iniciar sesión | La cuenta pertenece a otra organización y no está en el directorio appclientM | Invítala como usuario externo (ver "Agregar un usuario nuevo") |
| El usuario no aparece en la búsqueda | No existe en el directorio | Agrégalo primero |

## Opcional: que solo entren usuarios con rol

Por defecto, cualquier usuario del directorio puede iniciar sesión, aunque no tenga rol. Para que solo entren las personas con un rol asignado:

**Aplicaciones empresariales → ms-campuslab-bff → Propiedades → ¿Asignación requerida? = Sí → Guardar.**

Con eso activado, si alguien sin rol intenta entrar, Azure lo bloquea en la pantalla de inicio de sesión.
