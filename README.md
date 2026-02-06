# Gitlab

## Obtener credenciales iniciales

Visita la URL de GitLab e inicia sesión con el usuario `root` y la contraseña obtenida mediante uno de los siguientes métodos.

### Opción 1: Desde la terminal (Recomendado)

Ejecuta el siguiente comando directamente en tu terminal:

```shell
sudo podman exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

### Opción 2: Entrando al contenedor

Si prefieres acceder al contenedor para buscar el archivo manualmente:

1. Accede al shell del contenedor:
   ```shell
   sudo podman exec -it gitlab /bin/bash
   ```

2. Una vez dentro, lee el contenido del archivo:
   ```shell
   cat /etc/gitlab/initial_root_password
   ```

### Opción 3: Desde un Gestor de Contenedores (GUI)

Si utilizas una interfaz gráfica como Podman Desktop, Docker Desktop, o Portainer:

1. Abre tu gestor de contenedores y localiza el contenedor `gitlab`.
2. Haz clic en el contenedor para ver sus detalles.
3. Busca la pestaña **"Terminal"**, **"Console"** o **"Exec"**.
4. En la terminal integrada que aparece, escribe el siguiente comando y presiona Enter:
   ```shell
   cat /etc/gitlab/initial_root_password
   ```

**Nota:** El archivo de la contraseña se elimina automáticamente en el primer reinicio del contenedor después de 24 horas.

## Configuración de GitLab Runner con Podman en Fedora 40

Este documento detalla la configuración necesaria para ejecutar un GitLab Runner nativo que utiliza Podman como motor de contenedores en lugar de Docker, permitiendo despliegues automáticos en el mismo servidor.

### 1. Requisitos Previos en el Servidor (Fedora 40)

Para que el Runner pueda comunicarse con el motor de contenedores, el socket de Podman debe estar activo y escuchando:

```bash
# Habilitar el socket de Podman a nivel de sistema
sudo systemctl enable --now podman.socket

# Verifica que el socket esté activo
sudo systemctl status podman.socket

# Verificar que el socket existe
ls -l /run/podman/podman.sock
```

### 2. Instalar el GitLab Runner 🏃♂️

Necesitamos el binario del Runner para que pueda "escuchar" las tareas que envíe tu GitLab (que está en el contenedor).

```bash
# Añadir el repositorio oficial de GitLab
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash

# Instalar el paquete
sudo dnf install gitlab-runner -y
```

### 3. Registrar el GitLab Runner 📝

Ahora que tenemos el software instalado, el Runner necesita "presentarse" ante tu instancia de GitLab.

#### 1. Obtener el Token de Registro
Como queremos que este Runner gestione tus despliegues, es ideal crear un Project Runner.

1. Ve a Admin Area.
2. Navega a **Settings ⚙️ > CI/CD**.
3. Expande la sección **Runners**.
4. Haz clic en **Create instance runner**.
5. Añade una etiqueta (tag) como `deploy-fedora`. Esto es vital para que tu archivo `.gitlab-ci.yml` sepa qué Runner usar.
6. Marca la casilla **"Run untagged jobs"** si deseas que este Runner ejecute trabajos sin etiquetas.
7. (Opcional) En **"Runner description"**, añade una descripción para identificarlo fácilmente.
8. Configura la opción **"Protected"** si solo quieres que este Runner se use en ramas protegidas (como `main` o `production`).
9. Al darle a **Create runner**, te aparecerá un comando de registro o un Token que empieza por `glrt-`.

   **Ejemplo:**
   ```bash
   gitlab-runner register  --url http://localhost:8929  --token glrt-HDfu128s_6hkZrU_xjR_S286MQp0OjEKdToxCw.01.121ka1yx4
   ```
   
   **Ejemplo de Token:** `glrt-HDfu128s_6hkZrU_xjR_S286MQp0OjEKdToxCw.01.121ka1yx4`

#### 2. Registro Optimizado (Recomendado)

Copia y pega este comando en tu terminal de Fedora (necesitarás privilegios de sudo):

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "{IP del Gitlab}" \
  --token "glrt-HDfu128s_6hkZrU_xjR_S286MQp0OjEKdToxCw.01.121ka1yx4" \
  --executor "docker" \
  --docker-image "fedora:41" \
  --description "fedora-podman-runner" \
  --docker-host "unix:///run/podman/podman.sock" \
  --docker-privileged
```

**¿Qué acabamos de configurar?**
* **`--executor "docker"`**: Aunque usamos Podman, el Runner utiliza la API de Docker para gestionar los contenedores.
* **`--docker-host`**: Aquí le decimos exactamente dónde encontrar el socket de Podman en Fedora.
* **`--docker-privileged`**: Esto es importante para que el Runner tenga permisos para levantar otros contenedores (necesario para podman-compose).

#### 3. Ejecutar el registro en Fedora (Método Interactivo)
Alternativamente, puedes usar el asistente interactivo:

```bash
sudo gitlab-runner register
```

El asistente te guiará paso a paso. Aquí tienes lo que debes responder:

1.  **Running in system-mode.**
2.  **Enter the GitLab instance URL**: Introduce la URL de tu GitLab (ej. `http://192.168.1.68:8929/`).
3.  **Enter the registration token**: Pega el token `glrt-...` que obtuviste.
    *   *Ejemplo:* `glrt-HDfu128s_6hkZrU_xjR_S286MQp0OjEKdToxCw.01.121ka1yx4`
4.  **Verifying runner... is valid** (Si todo va bien).
5.  **Enter a name for the runner**: Escribe un nombre descriptivo, como `fedora-podman-server`.
    *   *Nota:* Esto solo es para identificarlo en `config.toml`.
6.  **Enter an executor**: Escribe **`docker`**.
7.  **Enter the default Docker image**: Escribe `fedora:40` o `alpine:latest`.

Una vez completado, el registro habrá terminado, pero **aún falta conectar el Runner con el socket de Podman**.

> **⚠️ Solución de problemas:** 
> Si obtienes el error `sudo: gitlab-runner: command not found`, significa que Fallo/No se completó la instalación.
>
> **1. Instalar el repositorio y el Runner**
> ```bash
> # Añadir el repositorio oficial
> curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
>
> # Instalar el paquete del runner
> sudo dnf install gitlab-runner -y
> ```
>
> **2. Verificar la instalación**
> ```bash
> gitlab-runner --version
> ```
> Si devuelve la versión, ¡reintenta el registro!

### 4. Configuración del Runner (/etc/gitlab-runner/config.toml)

El archivo de configuración es la pieza clave. Debimos ajustar el ejecutor docker para que apunte al socket de Podman y tenga permisos de administrador (privileged) para manejar redes y volúmenes.

Ruta: `/etc/gitlab-runner/config.toml`

```toml
concurrent = 1
check_interval = 0
connection_max_age = "15m0s"
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "fedora-podman-server"
  # NOTA: Reemplaza {IP_DEL_GITLAB} por la IP real de tu servidor (ej. 192.168.1.68)
  url = "http://{IP_DEL_GITLAB}:8929/"
  clone_url = "http://{IP_DEL_GITLAB}:8929/"
  id = 1
  token = "glrt-HDfu128s_6hkZrU_xjR_S286MQp0OjEKdToxCw.01.121ka1yx4"
  token_obtained_at = 2026-02-05T22:44:05Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "fedora:41"
    privileged = true
    # NOTA: Mapeo importante para que el contenedor resuelva localhost como la IP del host
    extra_hosts = ["localhost:{IP_DEL_GITLAB}"]
    dns = ["8.8.8.8", "1.1.1.1"]
    network_mode = "host"
    host = "unix:///run/podman/podman.sock"
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = [
    "/cache",
    "/run/podman/podman.sock:/var/run/docker.sock:rw",
    "/home/gitlab/docker-files:/home/gitlab/docker-files:rw"
    ]
    shm_size = 0
    network_mtu = 0
```

**¿Por qué esta configuración?**
Lo vital en este archivo es:
*   **`clone_url`**: Fuerza esta URL para clonar el repositorio, evitando problemas si GitLab cree que es `localhost`.
*   **`extra_hosts`**: Mapea `localhost` a la IP real de GitLab para que el contenedor pueda resolverlo correctamente.
*   **`dns`**: Añade DNS explícitos.
*   **`network_mode`**: Permite que el contenedor use la red real del servidor.
*   **`host = "unix:///run/podman/podman.sock"`**: Conecta el Runner al socket de Podman del host.
*   **`privileged = true`**: Permite ejecutar contenedores dentro de contenedores (Docker-in-Docker / Podman-in-Docker).
*   **`volumes = [...]`**: Mapeamos el socket para control y una carpeta del host `/home/gitlab/docker-files` para persistencia.

#### 🔌 Activación del Socket en Fedora
Para que lo anterior funcione, el "enchufe" (socket) de Podman debe estar escuchando en el sistema. Ejecuta esto para asegurarte:

```bash
sudo systemctl enable --now podman.socket
```

#### 🛡️ Un último detalle: Los Permisos
El Runner se ejecuta bajo el usuario del sistema `gitlab-runner`. Para que el `build-job` pueda escribir en `/home/gitlab/`, necesitamos asegurar que ese usuario tenga permisos.

Podemos solucionar esto rápidamente en tu terminal de Fedora:

```bash
# Crear la carpeta si no existe
sudo mkdir -p /home/gitlab/docker-files

# Darle la propiedad al usuario del runner
sudo chown -R gitlab-runner:gitlab-runner /home/gitlab/
```

#### ❓ Verificación final
Una vez que guardes el archivo (Ctrl+O, Enter, Ctrl+X), reinicia el servicio para que tome los cambios:

```bash
sudo systemctl restart gitlab-runner
```

### 5. Configuración en la Interfaz de GitLab

Para evitar errores de conexión durante el git clone, se debe configurar la URL de clonado global:

1. Ve a **Admin Area > Settings > General**.
2. Expande **Visibility and access controls**.
3. En **Custom Git clone URL for HTTP(S)**, coloca: `http://{IP_DEL_GITLAB}:8929`.

### 6. Estructura del .gitlab-ci.yml

El archivo de integración debe preparar el entorno (instalar herramientas) y manejar la seguridad de Git debido a que la carpeta está montada desde el host.

#### Instalación "al vuelo" (Flexible)
Podemos añadir comandos en una sección llamada `before_script`. Esto instalará las herramientas cada vez que el job comience. Es ideal si quieres probar versiones rápido sin cambiar la imagen base.

Para Fedora, el comando sería algo como: `dnf install -y git podman-compose`

#### Imagen Personalizada (Eficiente)
Podrías crear (o buscar) una imagen que ya incluya todo. Esto ahorra tiempo en cada ejecución del pipeline porque no tiene que descargar e instalar paquetes cada vez.

```yaml
stages:
  - build
  - deploy

variables:
  # Esta ruta ahora es local al servidor donde está el Runner
  FOLDER_ROUTE: /home/gitlab/docker-files/portafolio-dev
   # Instalar herramientas en el runner
before_script:
  - dnf install -y git podman-compose

build-job:
  stage: build
  tags:
    - deploy-fedora  # 🏷️ El tag que le pusiste al Runner
  script:
    - echo "Actualizando código en el servidor..."
    # Como el Runner corre como root/usuario del sistema, accede directo a la carpeta
    - |
      if [ -d "$FOLDER_ROUTE" ]; then
        cd "$FOLDER_ROUTE"
        git pull
      else 
        mkdir -p "$FOLDER_ROUTE"
        git clone --branch dev $CI_REPOSITORY_URL "$FOLDER_ROUTE"
      fi
    - echo "Código actualizado."

deploy-job:
  stage: deploy
  tags:
    - deploy-fedora
  script:
    - echo "Iniciando despliegue con podman-compose..."
    - cd "$FOLDER_ROUTE"
    - podman-compose build
    - podman-compose down
    - podman-compose up -d
    - echo "Despliegue completado."
```