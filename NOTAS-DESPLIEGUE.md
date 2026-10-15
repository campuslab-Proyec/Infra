# Notas de despliegue de CampusLab

Guía para levantar, actualizar y diagnosticar CampusLab en la EC2. Para la configuración de Azure ver [GUIA-AZURE-TENANT.md](GUIA-AZURE-TENANT.md), y para asignar roles ver [TUTORIAL-ROLES.md](TUTORIAL-ROLES.md).

## Arquitectura

```
Navegador ──HTTPS──> ngrok ──> EC2:80 nginx (frontend Angular)
                                   │  /api/*
                                   v
                              BFF :8080 ──> bookings :8081 ──> audit :8085
                                        ├─> catalog  :8082
                                        ├─> notify   :8083
                                        ├─> report   :8084
                                        └─> audit    :8085
           MySQL :3306 (bookings_db, catalog_db, audit_db)   RabbitMQ :5672
```

- El login es con Microsoft Entra ID. El frontend obtiene un token para la API del BFF y el BFF lo reenvía a los microservicios.
- Los eventos de reservas viajan por RabbitMQ: bookings publica y notify consume.
- Report lee directamente la tabla `bookings` de `bookings_db`.

## Repositorios

Cada servicio es un repositorio propio en <https://github.com/campuslab-Proyec>. Todos deben clonarse **en la misma carpeta**, porque `Infra/compose.yaml` usa rutas relativas (`../ms-campuslab-bff`, etc.).

| Repositorio en GitHub | Nombre de carpeta |
|---|---|
| Infra | `Infra` |
| ms-campuslab-audit | `ms-campuslab-audit` |
| ms-campuslab-bff | `ms-campuslab-bff` |
| ms-campuslab-bookings | `ms-campuslab-bookings` |
| ms-campuslab-catalog | `ms-campuslab-catalog` |
| ms-campuslab-notify | `ms-campuslab-notify` |
| ms-campuslab-report | `ms-campuslab-report` |
| frontend-campuslab | **`ngfrontend-campuslab`** ⚠️ |

⚠️ El repositorio del frontend se llama `frontend-campuslab`, pero la carpeta tiene que llamarse `ngfrontend-campuslab`:

```bash
git clone https://github.com/campuslab-Proyec/frontend-campuslab.git ngfrontend-campuslab
```

## Instalación desde cero en una EC2

```bash
ORG=https://github.com/campuslab-Proyec
cd ~
for r in Infra ms-campuslab-audit ms-campuslab-bff ms-campuslab-bookings ms-campuslab-catalog ms-campuslab-notify ms-campuslab-report; do
  git clone $ORG/$r.git
done
git clone $ORG/frontend-campuslab.git ngfrontend-campuslab

sudo systemctl enable --now docker
docker network create campuslab-net
cd ~/Infra
docker compose up -d --build
```

La primera vez, el build tarda varios minutos. Si la instancia se queda sin memoria, usa `docker compose build --parallel 1` y luego `docker compose up -d`.

Después, configura ngrok como servicio (sección **ngrok**).

## Actualizar la EC2 con lo último de GitHub

```bash
cd ~
for d in Infra ms-campuslab-* ngfrontend-campuslab; do
  echo "== $d"; git -C $d fetch origin && git -C $d reset --hard origin/main
done
cd ~/Infra
docker compose up -d --build
```

`reset --hard` descarta cualquier cambio hecho a mano en la EC2; la fuente de verdad es GitHub.

## Al reiniciar el laboratorio de AWS

Cuando termina la sesión del lab, AWS detiene la instancia. Al volver a iniciarla:

- **La IP pública cambia**, porque no hay IP elástica. Usa la IP nueva que aparece en la consola de EC2 para conectarte por SSH:
  ```powershell
  ssh -i "ruta\a\tu-clave.pem" ec2-user@<IP-NUEVA>
  ```
- **Docker, los contenedores y ngrok arrancan solos.** El compose tiene `restart: unless-stopped`, Docker está habilitado al arranque y ngrok corre como servicio de systemd.
- Los servicios Spring tardan 1 o 2 minutos en estar listos. Mientras tanto, la página puede mostrar un error 502.
- Los datos de MySQL se conservan, porque están en el volumen `mysql-data`.

## ngrok

La aplicación se publica en <https://handoff-sprint-protrude.ngrok-free.dev>. Ese dominio solo puede estar activo **en una máquina a la vez**: si otra máquina tiene ngrok corriendo con el mismo dominio, la página mostrará esa otra versión. En <https://dashboard.ngrok.com>, en **Agents**, se ve desde dónde está conectado.

Configurarlo como servicio (una sola vez por instancia):

```bash
sudo tee /etc/systemd/system/ngrok.service > /dev/null <<EOF
[Unit]
Description=ngrok tunnel CampusLab
After=network-online.target docker.service
Wants=network-online.target

[Service]
User=ec2-user
ExecStart=$(which ngrok) http --url=handoff-sprint-protrude.ngrok-free.dev 80 --log=stdout
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now ngrok
sudo systemctl status ngrok --no-pager
```

MSAL exige HTTPS, salvo en `localhost`. Por eso no se puede usar la app entrando por `http://<IP-de-la-EC2>`; hay que entrar por el dominio de ngrok.

## Configuración de Azure (Entra ID)

- **Directorio:** appclientM, tenant `aff2eb38-d6cf-4220-8527-53f1cb6aed15`.
- **frontend-campuslab** (cliente `718a6ef2-45a4-4f06-84c9-6f54de8c3514`):
  - **Autenticación → SPA**, URIs de redirección:
    - `https://handoff-sprint-protrude.ngrok-free.dev/`
    - `http://localhost/` (para correrlo en un PC con Docker)
    - `http://localhost:4200/` (para `ng serve`)
  - **Permisos de API:** `ms-campuslab-bff → access_as_user`, con consentimiento de administrador otorgado.
- **ms-campuslab-bff** (cliente `8d19bb45-1842-43ce-b76c-4513b979df81`):
  - **Exponer una API:** URI `api://8d19bb45-1842-43ce-b76c-4513b979df81` y el scope `access_as_user`.
  - **Roles de aplicación:** `Admin`, `Docente` y `Estudiante`. Ver [TUTORIAL-ROLES.md](TUTORIAL-ROLES.md).

Los backends solo aceptan tokens emitidos por este tenant y para esta API. Una cuenta de otra organización (por ejemplo, una cuenta `@duocuc.cl` que no se haya invitado) no puede entrar.

## Diagnóstico de problemas

Comandos útiles, desde `~/Infra`:

```bash
docker compose ps -a                              # estado de los contenedores
docker compose logs <servicio> --tail 50          # log de un servicio
curl -I http://localhost                          # nginx debe responder 200
```

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| ngrok muestra **502 Bad Gateway** en la página principal | El contenedor `frontend-campuslab` no está corriendo, o ngrok apunta a otra máquina | `docker compose ps -a` y revisar el log del frontend; revisar **Agents** en el dashboard de ngrok |
| **502** en llamadas a `/api/...` | El BFF está caído o todavía arrancando | `docker compose logs ms-campuslab-bff --tail 50` |
| **401** en todas las llamadas | El token no es de este tenant o de esta API | Iniciar sesión con una cuenta del directorio appclientM |
| **403** en una acción | El usuario no tiene el rol necesario | Ver [TUTORIAL-ROLES.md](TUTORIAL-ROLES.md) |
| **AADSTS650052** al iniciar sesión | La cuenta es de otra organización, o se está sirviendo una versión vieja del frontend | Invitar la cuenta al directorio; verificar la versión (fila siguiente) |
| La app no refleja los últimos cambios | Se está sirviendo un build viejo | En F12 → Network, revisar el nombre de `main-*.js`; reconstruir con `docker compose up -d --build` |
| `network campuslab-net ... could not be found` | Falta la red de Docker | `docker network create campuslab-net` |
| El build se corta o la instancia se congela | Falta de memoria al compilar todo en paralelo | `docker compose build --parallel 1` |

## Correrlo en un PC (Windows con Docker Desktop)

```powershell
docker network create campuslab-net
cd Infra
docker compose up -d --build
```

Después se abre <http://localhost>. La URI `http://localhost/` tiene que estar registrada en Azure, en frontend-campuslab.

## Pendientes conocidos

- **catalog y notify no validan JWT.** Solo los protege el BFF, y sus puertos (8082 y 8083) están publicados en el compose. Conviene no abrir esos puertos en el Security Group de la EC2.
- **Security Group:** con ngrok basta con abrir el puerto **22** (SSH). Los puertos 3307, 5672, 15672 y 8080–8085 no deberían estar abiertos a internet.
