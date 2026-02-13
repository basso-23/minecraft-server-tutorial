# Instalación Completa de Pterodactyl en Ubuntu 22.04

Guía paso a paso para instalar Pterodactyl Panel + Wings en un VPS con Ubuntu 22.04 usando IP directamente (sin SSL).

---

## Requisitos Previos

- VPS con Ubuntu 22.04
- Acceso root o usuario con sudo
- IP pública del servidor
- Mínimo 2GB RAM (recomendado 4GB)
- Conexión a internet estable

---

## PARTE 1: INSTALACIÓN DEL PANEL

---

### Paso 1: Actualizar el Sistema

```bash
apt update && apt upgrade -y
```

---

### Paso 2: Instalar Dependencias

```bash
# Agregar repositorios necesarios
apt -y install software-properties-common curl apt-transport-https ca-certificates gnupg

# Agregar repositorio de PHP
LC_ALL=C.UTF-8 add-apt-repository -y ppa:ondrej/php

# Actualizar lista de paquetes
apt update

# Instalar PHP 8.3 y extensiones requeridas
apt -y install php8.3 php8.3-{common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip}

# Instalar MariaDB, NGINX, Redis y herramientas
apt -y install mariadb-server nginx tar unzip git redis-server
```

---

### Paso 3: Instalar Composer

```bash
curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
```

Verificar instalación:
```bash
composer --version
```

---

### Paso 4: Configurar Base de Datos

Acceder a MariaDB:
```bash
mysql -u root -p
```

> **Nota:** Si es instalación nueva, presiona Enter cuando pida contraseña.

Ejecutar los siguientes comandos SQL (reemplaza `TU_PASSWORD_SEGURA` por una contraseña segura):

```sql
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY '123';
CREATE DATABASE panel;
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

> **IMPORTANTE:** Guarda esta contraseña, la necesitarás más adelante.

---

### Paso 5: Descargar el Panel

```bash
# Crear directorio
mkdir -p /var/www/pterodactyl
cd /var/www/pterodactyl

# Descargar última versión
curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz

# Extraer archivos
tar -xzvf panel.tar.gz

# Configurar permisos
chmod -R 755 storage/* bootstrap/cache/
```

---

### Paso 6: Configuración Inicial

```bash
# Copiar archivo de configuración
cp .env.example .env

# Instalar dependencias de PHP
COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader

# Generar clave de aplicación
php artisan key:generate --force
```

> **IMPORTANTE:** Después de ejecutar `key:generate`, verás una clave `APP_KEY` en el archivo `.env`.
> **GUARDA ESTA CLAVE EN UN LUGAR SEGURO.** Si la pierdes, no podrás recuperar datos encriptados.

---

### Paso 7: Configuración del Entorno

Ejecutar el asistente de configuración:

```bash
php artisan p:environment:setup
```

Responde a las preguntas:
- **Egg Author Email:** Tu correo electrónico
- **Application URL:** `http://147.93.118.226` (ejemplo: `http://192.168.1.100`)
- **Application Timezone:** `America/Panama`
- **Cache Driver:** `redis`
- **Session Driver:** `redis`
- **Queue Driver:** `redis`
- **Enable UI based settings editor?:** `yes`
- **Redis Host:** `127.0.0.1`
- **Redis Password:** Dejar vacío (presiona Enter)
- **Redis Port:** `6379`

---

### Paso 8: Configuración de Base de Datos

```bash
php artisan p:environment:database
```

Responde:
- **Database Host:** `127.0.0.1`
- **Database Port:** `3306`
- **Database Name:** `panel`
- **Database Username:** `pterodactyl`
- **Database Password:** La contraseña que creaste en el Paso 4

---

### Paso 9: Configuración de Correo (Opcional)

```bash
php artisan p:environment:mail
```

Responde:
- **Which driver should be used for sending emails?:** `sendmail`
- **Email address emails should originate from:** `admin@localhost`
- **Name that emails should appear from:** `Pterodactyl Panel`

> **Nota:** Para producción puedes configurar SMTP con Gmail u otro servicio.

---

### Paso 10: Migrar Base de Datos

```bash
php artisan migrate --seed --force
```

Este proceso crea todas las tablas necesarias en la base de datos.

---

### Paso 11: Crear Usuario Administrador

```bash
php artisan p:user:make
```

Responde:
- **Is this user an administrator?:** `yes`
- **Email Address:** Tu correo
- **Username:** Tu nombre de usuario
- **First Name:** Tu nombre
- **Last Name:** Tu apellido
- **Password:** Mínimo 8 caracteres, mayúsculas, minúsculas y un número

> **GUARDA ESTAS CREDENCIALES** - Las necesitarás para acceder al panel.

---

### Paso 12: Configurar Permisos

```bash
chown -R www-data:www-data /var/www/pterodactyl/*
```

---

### Paso 13: Configurar Crontab

Abrir el editor de crontab:
```bash
crontab -e
```

Si te pregunta qué editor usar, selecciona `1` para nano.

Agregar esta línea al final del archivo:
```
* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1
```

Guardar y salir (en nano: `Ctrl+X`, luego `Y`, luego `Enter`).

---

### Paso 14: Crear Servicio Queue Worker

Crear el archivo de servicio:
```bash
nano /etc/systemd/system/pteroq.service
```

Pegar el siguiente contenido:
```ini
[Unit]
Description=Pterodactyl Queue Worker
After=redis-server.service

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/pterodactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Guardar y salir (`Ctrl+X`, `Y`, `Enter`).

Habilitar e iniciar el servicio:
```bash
systemctl enable --now redis-server
systemctl enable --now pteroq.service
```

---

### Paso 15: Configurar NGINX

Eliminar configuración por defecto:
```bash
rm /etc/nginx/sites-enabled/default
```

Crear configuración del panel:
```bash
nano /etc/nginx/sites-available/pterodactyl.conf
```

Pegar el siguiente contenido (reemplaza `147.93.118.226` por tu IP real):
```nginx
server {
    listen 80;
    server_name 147.93.118.226;

    root /var/www/pterodactyl/public;
    index index.html index.htm index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/pterodactyl.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

Guardar y salir.

Habilitar el sitio y reiniciar NGINX:
```bash
ln -s /etc/nginx/sites-available/pterodactyl.conf /etc/nginx/sites-enabled/pterodactyl.conf

# Verificar configuración
nginx -t

# Reiniciar NGINX
systemctl restart nginx
```

---

### Paso 16: Verificar Panel

Abre tu navegador y ve a:
```
http://147.93.118.226
```

Deberías ver la página de login de Pterodactyl. Inicia sesión con las credenciales del Paso 11.

---

## PARTE 2: INSTALACIÓN DE WINGS (DAEMON)

Wings es el daemon que ejecuta los servidores de juegos.

---

### Paso 17: Instalar Docker

```bash
curl -sSL https://get.docker.com/ | CHANNEL=stable bash

# Habilitar Docker al inicio
systemctl enable --now docker
```

Verificar instalación:
```bash
docker --version
```

---

### Paso 18: Descargar Wings

```bash
# Crear directorio de configuración
mkdir -p /etc/pterodactyl

# Descargar Wings
curl -L -o /usr/local/bin/wings "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"

# Dar permisos de ejecución
chmod u+x /usr/local/bin/wings
```

---

### Paso 19: Crear Location en el Panel

1. Accede al panel: `http://147.93.118.226`
2. Ve a **Admin** (icono de engranaje arriba a la derecha)
3. En el menú lateral, ve a **Locations**
4. Click en **Create New**
5. Llena los campos:
   - **Short Code:** `local` (o cualquier identificador corto)
   - **Description:** `Servidor Local` (o cualquier descripción)
6. Click en **Create**

---

### Paso 20: Crear Node en el Panel

1. En el panel de Admin, ve a **Nodes**
2. Click en **Create New**
3. Llena los campos:
   - **Name:** `Node-1` (o cualquier nombre)
   - **Description:** Opcional
   - **Location:** Selecciona el location que creaste
   - **FQDN:** `147.93.118.226` (tu IP del servidor)
   - **Communicate Over SSL:** `Use HTTP Connection` (IMPORTANTE)
   - **Behind Proxy:** `Not Behind Proxy`
   - **Daemon Server File Directory:** `/var/lib/pterodactyl/volumes`
   - **Total Memory:** La RAM total de tu servidor en MB (ej: 4096 para 4GB)
   - **Memory Over-Allocation:** `0`
   - **Total Disk Space:** Espacio en disco en MB (ej: 50000 para 50GB)
   - **Disk Over-Allocation:** `0`
   - **Daemon Port:** `8080`
   - **Daemon SFTP Port:** `2022`
4. Click en **Create Node**

---

### Paso 21: Obtener Configuración de Wings

1. Después de crear el Node, click en él para ver los detalles
2. Ve a la pestaña **Configuration**
3. Click en **Generate Token**
4. Copia todo el contenido del cuadro de configuración

---

### Paso 22: Configurar Wings

En tu servidor, crea el archivo de configuración:
```bash
nano /etc/pterodactyl/config.yml
```

Pega el contenido que copiaste del panel.

Debería verse algo así (ejemplo):
```yaml
debug: false
uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
token_id: xxxxx
token: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
api:
  host: 0.0.0.0
  port: 8080
  ssl:
    enabled: false
    cert: ""
    key: ""
  upload_limit: 100
system:
  data: /var/lib/pterodactyl/volumes
  sftp:
    bind_port: 2022
allowed_mounts: []
remote: 'http://147.93.118.226'
```

Guardar y salir.

---

### Paso 23: Probar Wings

Ejecutar Wings en modo debug para verificar que funciona:
```bash
wings --debug
```

Si todo está bien, verás mensajes de inicio sin errores. Presiona `Ctrl+C` para detener.

---

### Paso 24: Crear Servicio de Wings

```bash
nano /etc/systemd/system/wings.service
```

Pegar el siguiente contenido:
```ini
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Guardar y salir.

Habilitar e iniciar Wings:
```bash
systemctl enable --now wings
```

Verificar estado:
```bash
systemctl status wings
```

---

### Paso 25: Crear Allocations (Asignaciones de Puertos)

1. En el panel de Admin, ve a **Nodes**
2. Click en tu Node
3. Ve a la pestaña **Allocation**
4. En **Assign New Allocations**:
   - **IP Address:** Tu IP del servidor (o `0.0.0.0` para todas las interfaces)
   - **Ports:** `25565` (o un rango como `25565-25575`)
5. Click en **Submit**

> **Nota:** El puerto 25565 es el predeterminado para Minecraft. Puedes agregar más puertos según necesites.

---

## PARTE 3: VERIFICACIÓN FINAL

---

### Paso 26: Verificar Todos los Servicios

Ejecuta estos comandos para verificar que todo está corriendo:

```bash
# Verificar NGINX
systemctl status nginx

# Verificar PHP-FPM
systemctl status php8.3-fpm

# Verificar MariaDB
systemctl status mariadb

# Verificar Redis
systemctl status redis-server

# Verificar Queue Worker
systemctl status pteroq

# Verificar Wings
systemctl status wings

# Verificar Docker
systemctl status docker
```

Todos deben mostrar `active (running)`.

---

### Paso 27: Crear Tu Primer Servidor

1. Ve al panel: `http://147.93.118.226`
2. Ve a **Admin** > **Servers**
3. Click en **Create New**
4. Configura:
   - **Server Name:** Nombre de tu servidor
   - **Server Owner:** Selecciona tu usuario
   - **Default Allocation:** Selecciona el puerto que creaste
   - **Node:** Selecciona tu Node
   - **Nest:** Selecciona el tipo (ej: Minecraft)
   - **Egg:** Selecciona el egg específico (ej: Paper, Vanilla, etc.)
   - **Memory:** RAM asignada en MB
   - **Disk:** Espacio en disco en MB
5. Click en **Create Server**

---

## Resumen de Accesos

| Servicio | URL/Puerto |
|----------|------------|
| Panel Web | `http://147.93.118.226` |
| Wings API | `http://147.93.118.226:8080` |
| SFTP | `147.93.118.226:2022` |
| Minecraft (ejemplo) | `147.93.118.226:25565` |

---

## Comandos Útiles

```bash
# Reiniciar todos los servicios
systemctl restart nginx php8.3-fpm mariadb redis-server pteroq wings

# Ver logs de Wings
journalctl -u wings -f

# Ver logs del Queue Worker
journalctl -u pteroq -f

# Actualizar el panel (ejecutar desde /var/www/pterodactyl)
php artisan down
curl -L https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz | tar -xzv
chmod -R 755 storage/* bootstrap/cache/
composer install --no-dev --optimize-autoloader
php artisan migrate --seed --force
php artisan view:clear
php artisan config:clear
chown -R www-data:www-data /var/www/pterodactyl/*
php artisan queue:restart
php artisan up
```

---

## Solución de Problemas Comunes

### El panel no carga
```bash
# Verificar NGINX
nginx -t
systemctl restart nginx

# Verificar permisos
chown -R www-data:www-data /var/www/pterodactyl/*
```

### Wings no conecta
```bash
# Verificar config.yml
cat /etc/pterodactyl/config.yml

# Reiniciar Wings
systemctl restart wings

# Ver logs
journalctl -u wings -f
```

### Error de base de datos
```bash
# Verificar MariaDB
systemctl status mariadb

# Probar conexión
mysql -u pterodactyl -p -h 127.0.0.1 panel
```

---

## Puertos que Deben Estar Abiertos en el Firewall

| Puerto | Protocolo | Uso |
|--------|-----------|-----|
| 80 | TCP | Panel Web |
| 8080 | TCP | Wings API |
| 2022 | TCP | SFTP |
| 25565+ | TCP/UDP | Servidores de juegos |

Si usas UFW:
```bash
ufw allow 80/tcp
ufw allow 8080/tcp
ufw allow 2022/tcp
ufw allow 25565:25575/tcp
ufw allow 25565:25575/udp
ufw reload
```

---

**Listo! Ya tienes Pterodactyl funcionando. Solo entra al panel y crea tus servidores.**
