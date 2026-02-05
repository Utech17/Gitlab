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
