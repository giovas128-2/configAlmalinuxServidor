# Implementación de Servidor NGINX y PHP compilados desde Código Fuente

---

## Carátula

| | |
|---|---|
| **Materia** | Administración de Servidores Linux |
| **Proyecto** | Implementación de NGINX 1.31.x y PHP 8.4.x compilados desde código fuente |
| **Sistema Operativo** | AlmaLinux 10.1 (Heliotrope Lion) |
| **Fecha** | Mayo 2026 |

### Integrantes

- Hernandez Pizano Cesar Giovanni
- Baltazar Migangos Angel Sebastian
- Flores Perez Jacobo Johann

---

## Objetivo General

Implementar un servidor web funcional compilando NGINX 1.31.x y PHP 8.4.x desde código fuente sobre AlmaLinux 10.1, configurando la comunicación entre ambos mediante FastCGI a través de un socket Unix, y registrando ambos servicios en systemd para que arranquen automáticamente con el sistema operativo.

---

## Objetivos Específicos

1. Instalar y configurar AlmaLinux 10.1 en una máquina virtual VirtualBox como entorno de servidor.
2. Compilar NGINX versión 1.31.1 desde código fuente con el prefijo de instalación en `/srv/nginx`.
3. Compilar PHP versión 8.4.7 desde código fuente habilitando soporte para FPM, procesamiento de imágenes, fechas e internacionalización.
4. Configurar PHP-FPM para comunicarse con NGINX mediante socket Unix en `/var/run/php84.sock`.
5. Crear y registrar unidades de servicio systemd para NGINX y PHP-FPM con arranque automático en `multi-user.target`.
6. Verificar el funcionamiento del stack mediante un script PHP visible en el navegador web.

---

## Desarrollo del Proyecto

### 1. Preparación del entorno

Se instaló AlmaLinux 10.1 en una máquina virtual Oracle VirtualBox con las siguientes características:

- RAM: 4096 MB
- CPU: 2 núcleos
- Disco: 20 GB (VDI dinámico)
- Red: NAT con reenvío de puertos (host 8080 → guest 80)

Una vez instalado el sistema, se actualizó y se instalaron las dependencias necesarias para compilar:

```bash
dnf update -y

dnf install -y gcc gcc-c++ make cmake zlib-devel openssl-devel \
  libcurl-devel libpng-devel freetype-devel sqlite-devel \
  bison tar wget git oniguruma-devel epel-release nano

dnf install -y re2c
```

### 2. Creación de usuarios y directorios del sistema

Se crearon los usuarios y grupos del sistema necesarios para ejecutar los servicios con privilegios mínimos:

```bash
groupadd -r nginx
useradd -r -g nginx -s /sbin/nologin -d /srv/nginx nginx
useradd -r -g nginx -s /sbin/nologin php

mkdir -p /srv/nginx
chown -R nginx:nginx /srv/nginx
```

### 3. Compilación e instalación de NGINX 1.31.1

Se descargó el código fuente de NGINX versión 1.31.1 desde el sitio oficial y se compiló con las siguientes opciones:

```bash
cd /tmp
wget https://nginx.org/download/nginx-1.31.1.tar.gz
tar -xzvf nginx-1.31.1.tar.gz
cd nginx-1.31.1

./configure \
  --prefix=/srv/nginx \
  --user=nginx \
  --group=nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_gzip_static_module \
  --with-pcre \
  --with-stream

make && make install
```

Se verificó la instalación con:

```bash
/srv/nginx/sbin/nginx -v
# nginx version: nginx/1.31.1
```

### 4. Compilación e instalación de PHP 8.4.7

Se descargó el código fuente de PHP versión 8.4.7 y se compiló habilitando FPM, soporte de imágenes (GD), internacionalización (intl) y calendario:

```bash
cd /tmp
wget https://www.php.net/distributions/php-8.4.7.tar.gz
tar -xzvf php-8.4.7.tar.gz
cd php-8.4.7

./configure \
  --prefix=/srv/nginx \
  --with-fpm-user=php \
  --with-fpm-group=nginx \
  --enable-fpm \
  --with-openssl \
  --with-zlib \
  --with-curl \
  --enable-gd \
  --with-jpeg \
  --with-webp \
  --with-freetype \
  --enable-intl \
  --enable-mbstring \
  --enable-calendar \
  --with-sqlite3

make -j$(nproc) && make install
```

Se verificó la instalación con:

```bash
/srv/nginx/bin/php -v
# PHP 8.4.7 (cli)
```

### 5. Configuración de PHP-FPM con socket Unix

Se copiaron los archivos de configuración base y se modificó el pool `www` para usar socket Unix:

```bash
cp /srv/nginx/etc/php-fpm.conf.default /srv/nginx/etc/php-fpm.conf
cp /srv/nginx/etc/php-fpm.d/www.conf.default /srv/nginx/etc/php-fpm.d/www.conf
```

En `/srv/nginx/etc/php-fpm.d/www.conf` se configuraron los siguientes parámetros:

```ini
user = php
group = nginx
listen = /var/run/php84.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

### 6. Configuración de NGINX con FastCGI

Se configuró NGINX para procesar archivos PHP mediante FastCGI a través del socket Unix. El archivo `/srv/nginx/conf/nginx.conf` quedó de la siguiente manera:

```nginx
user nginx nginx;
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;
    keepalive_timeout 65;

    server {
        listen      80;
        server_name localhost;
        root        /srv/nginx/html;
        index       index.php index.html;

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
            fastcgi_pass  unix:/var/run/php84.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include       fastcgi_params;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root html;
        }
    }
}
```

### 7. Registro de servicios en systemd

**Servicio NGINX** — `/etc/systemd/system/nginx.service`:

```ini
[Unit]
Description=NGINX HTTP Server
After=network.target

[Service]
Type=forking
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

**Servicio PHP-FPM** — `/etc/systemd/system/php-fpm8.4.service`:

```ini
[Unit]
Description=PHP 8.4 FastCGI Process Manager
After=network.target

[Service]
Type=simple
PIDFile=/srv/nginx/var/run/php-fpm.pid
ExecStart=/srv/nginx/sbin/php-fpm --nodaemonize --fpm-config /srv/nginx/etc/php-fpm.conf
ExecReload=/bin/kill -s USR2 $MAINPID

[Install]
WantedBy=multi-user.target
```

Se habilitaron y arrancaron ambos servicios:

```bash
systemctl daemon-reload
systemctl enable nginx php-fpm8.4
systemctl start php-fpm8.4
systemctl start nginx
```

### 8. Verificación del funcionamiento

Se creó el archivo de prueba `phpinfo.php`:

```bash
echo '<?php phpinfo(); ?>' > /srv/nginx/html/phpinfo.php
```

Se configuró el reenvío de puertos en VirtualBox (host 8080 → guest 80) y se accedió desde el navegador a:

```
http://127.0.0.1:8080/phpinfo.php
```

El resultado mostró correctamente la página de información de PHP 8.4.7 con Server API: FPM/FastCGI, confirmando la comunicación exitosa entre NGINX y PHP-FPM.

---

## Conclusiones

- Se logró compilar exitosamente NGINX 1.31.1 y PHP 8.4.7 desde código fuente sobre AlmaLinux 10.1, demostrando que es posible tener control total sobre las opciones de compilación y módulos incluidos en cada servidor.

- La comunicación entre NGINX y PHP-FPM mediante socket Unix (`/var/run/php84.sock`) es más eficiente que la comunicación TCP, ya que evita la sobrecarga de red al operar ambos servicios en el mismo sistema.

- El uso de systemd para registrar los servicios permite que el servidor web inicie automáticamente con el sistema operativo, garantizando disponibilidad ante reinicios.

- Se identificó que SELinux puede interferir con la ejecución de binarios compilados manualmente en rutas no estándar, por lo que se configuró en modo permissive para el entorno de desarrollo.

- La compilación desde código fuente requiere mayor tiempo y conocimiento técnico que instalar paquetes precompilados, pero ofrece mayor flexibilidad y control sobre las características del servidor.

---

## Bibliografía

nginx. (2026). *nginx 1.31.1 release*. nginx.org. https://nginx.org/en/download.html

PHP Group. (2026). *PHP 8.4.7 release*. php.net. https://www.php.net/downloads.php

AlmaLinux OS Foundation. (2026). *AlmaLinux OS 10.1 documentation*. almalinux.org. https://almalinux.org/

Red Hat, Inc. (2026). *Using systemd unit files*. Red Hat Documentation. https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10

Oracle Corporation. (2026). *Oracle VM VirtualBox user manual*. virtualbox.org. https://www.virtualbox.org/manual/

nginx. (2026). *Beginner's guide — FastCGI*. nginx.org. https://nginx.org/en/docs/beginners_guide.html

PHP Group. (2026). *PHP-FPM configuration*. php.net. https://www.php.net/manual/en/install.fpm.configuration.php
