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

## Configuración de GitLab Runner (Shell Executor + Root) en Fedora 40

Este documento detalla la configuración recomendada para un servidor de despliegue: un GitLab Runner con **ejecutor Shell** corriendo como **Root**.

**¿Por qué esta configuración?**
*   **Simplicidad**: El Runner ejecuta comandos directamente en el servidor (Fedora), eliminando la complejidad de "Docker-in-Docker".
*   **Potencia**: Al ser Root, el Runner puede ejecutar comandos de `podman` directamente, gestionar volúmenes del sistema y reiniciar servicios sin restricciones.

### 1. Instalar el GitLab Runner 🏃‍♂️

Necesitamos el binario del Runner en el servidor.

```bash
# Añadir el repositorio oficial de GitLab
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash

# Instalar el paquete
sudo dnf install gitlab-runner -y
```

### 2. Configurar el Servicio como Root (CRÍTICO) 👑

Por defecto, el runner se instala con un usuario limitado (`gitlab-runner`). Para usar Podman libremente, debemos cambiarlo a `root`.

#### Paso 1: Detener el servicio

```bash
sudo systemctl stop gitlab-runner
```

#### Paso 2: Editar la definición del servicio

Ejecuta este comando para abrir el editor de configuración del sistema:

```bash
sudo systemctl edit --full gitlab-runner
```

Se abrirá un editor. Realiza estos **dos cambios obligatorios**:

1.  Busca `User=gitlab-runner` y cámbialo a `root`.
    
    **Así debe quedar el bloque [Service]**:
    Busca estas dos líneas (suelen estar juntas) y déjalas así:

    ```ini
    [Service]
    User=root
    Group=root
    ```

2.  Busca la línea `ExecStart=`. **Elimina** la parte que dice `--user gitlab-runner`.
    
    Debe quedar parecido a esto:
    ```ini
    ExecStart=/usr/bin/gitlab-runner run --working-directory /home/gitlab-runner --config /etc/gitlab-runner/config.toml --service gitlab-runner
    ```

    **Nota:** A veces, si ya definiste `User=root` en la sección `[Service]`, puedes borrar la parte de `--user ...` de esta línea y funcionará igual, pero ponerlo explícito (`--user root`) es una alternativa segura.

Guarda los cambios (Ctrl+O, Enter) y sal (Ctrl+X).

#### Paso 3: Aplicar y Reiniciar

```bash
# Recargar configuración
sudo systemctl daemon-reload

# Iniciar el servicio como Root
sudo systemctl start gitlab-runner

# Verificar (La columna USER debe decir 'root')
ps aux | grep gitlab-runner
```

#### Paso 4: Configurar Git

Como ahora eres Root, Git necesita permiso para operar en cualquier directorio:

```bash
sudo git config --system --add safe.directory '*'
```

### 3. Registrar el GitLab Runner 📝

Ahora conectaremos el servidor con tu GitLab usando el modo **Shell**.

Necesitarás el **Token de Registro** desde tu GitLab en **Admin Area > Settings > CI/CD > Runners > New Instance Runner**. Crea uno con el tag `deploy-fedora`.

Ejecuta el siguiente comando en tu Fedora (reemplazando los valores):

```bash
sudo gitlab-runner register \
  --non-interactive \
  --url "{IP_DEL_GITLAB}" \
  --token "{TU_TOKEN_GLRT}" \
  --executor "shell" \
  --description "fedora-shell-runner"
```

**Nota**: Al usar `--executor "shell"`, no necesitamos configurar imágenes de docker ni sockets. El runner usará el shell `bash` del sistema directamente.

### 4. Configuración Final (/etc/gitlab-runner/config.toml)

Verifica que el archivo de configuración esté limpio y correcto.

Ruta: `/etc/gitlab-runner/config.toml`

```toml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "fedora-shell-runner"
  url = "http://{IP_DEL_GITLAB}:8929/"
  token = "glrt-xxxxxxxxxxxx"
  executor = "shell"
  [runners.cache]
```

Si necesitas cambiar la URL de clonado (porque GitLab cree que es localhost), agrega `clone_url` dentro de `[[runners]]`:

```toml
  clone_url = "http://{IP_DEL_GITLAB}:8929/"
```

Reinicia el runner si haces cambios manuales: `sudo gitlab-runner restart`.

### 5. Estructura del .gitlab-ci.yml

Tu pipeline ahora es mucho más simple. No necesitas `image: ...` ni `dind`. El script corre como si tú lo escribieras en la terminal del servidor.

```yaml
stages:
  - deploy

variables:
  # Ruta donde queremos el proyecto en el servidor
  PROJECT_PATH: /home/gitlab/proyectos/mi-app

deploy-job:
  stage: deploy
  tags:
    - deploy-fedora
  script:
    - echo "🚀 Iniciando despliegue en Fedora..."
    
    # Asegurar que existe la carpeta
    - mkdir -p $PROJECT_PATH
    
    # Opción A: Usar git pull (rápido)
    - |
      if [ -d "$PROJECT_PATH/.git" ]; then
        cd $PROJECT_PATH
        git pull origin main
      else
        git clone $CI_REPOSITORY_URL $PROJECT_PATH
        cd $PROJECT_PATH
      fi
    
    # Opción B: Ejecutar Podman Compose
    - echo "📦 Construyendo y levantando contenedores..."
    - podman-compose up -d --build
    
    - echo "✅ Despliegue exitoso."
```