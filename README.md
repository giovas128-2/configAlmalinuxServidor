# Implementación de Servidor NGINX y PHP compilados desde Código Fuente

---

## Carátula

| | |
|---|---|
| **Escuela** | Tecnologico de Estudios Superiores del Oriente del Estado de Mexico |
| **Materia** | Taller de sistemas Operativos |
| **Profesor** | Gustavo Moises Romero Gonzalez |
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
```

<img width="800" height="400" alt="prueba1" src="https://github.com/user-attachments/assets/fed871e4-686c-44d7-950a-7dd2e66f82da" />

```bash
dnf install -y gcc gcc-c++ make cmake zlib-devel openssl-devel \
  libcurl-devel libpng-devel freetype-devel sqlite-devel \
  bison tar wget git oniguruma-devel epel-release nano
```

<img width="800" height="400" alt="prueba2" src="https://github.com/user-attachments/assets/250d597d-397e-4f45-ab8a-64ed88e2ef53" />

```bash
dnf install -y re2c
```

<img width="800" height="400" alt="prueba3" src="https://github.com/user-attachments/assets/9b93d1a3-3f6a-4be7-843e-a8fc4d2c1e56" />


### 2. Creación de usuarios y directorios del sistema

Se crearon los usuarios y grupos del sistema necesarios para ejecutar los servicios con privilegios mínimos:

```bash
groupadd -r nginx
useradd -r -g nginx -s /sbin/nologin -d /srv/nginx nginx
useradd -r -g nginx -s /sbin/nologin php
```

<img width="800" height="400" alt="prueba4" src="https://github.com/user-attachments/assets/67dbb2bf-2535-4555-b936-fc66efdd73d7" />

```bash
mkdir -p /srv/nginx
chown -R nginx:nginx /srv/nginx
```

<img width="800" height="400" alt="prueba5" src="https://github.com/user-attachments/assets/9d065623-fb2e-47b5-8b3b-e04dc1beb5e0" />


### 3. Compilación e instalación de NGINX 1.31.1

Se descargó el código fuente de NGINX versión 1.31.1 desde el sitio oficial y se compiló con las siguientes opciones:

```bash
cd /tmp
wget https://nginx.org/download/nginx-1.31.1.tar.gz
tar -xzvf nginx-1.31.1.tar.gz
cd nginx-1.31.1
```

<img width="800" height="400" alt="prueba6" src="https://github.com/user-attachments/assets/255c75bd-7e46-40cd-9374-800bfaf2fa59" />


```bash
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

<img width="800" height="400" alt="prueba7" src="https://github.com/user-attachments/assets/59b1775c-9d21-4817-b207-b471488f30ea" />


```bash
make && make install
```

<img width="800" height="400" alt="prueba8" src="https://github.com/user-attachments/assets/dcdcddc5-665e-4372-b6d2-2c5f672234b5" />

Se verificó la instalación con:

```bash
/srv/nginx/sbin/nginx -v
# nginx version: nginx/1.31.1
```

<img width="800" height="400" alt="prueba9" src="https://github.com/user-attachments/assets/f6022655-5de9-4d2a-aa3e-9ee92a57f9dc" />


### 4. Compilación e instalación de PHP 8.4.7

Se descargó el código fuente de PHP versión 8.4.7 y se compiló habilitando FPM, soporte de imágenes (GD), internacionalización (intl) y calendario:

```bash
cd /tmp
wget https://www.php.net/distributions/php-8.4.7.tar.gz
tar -xzvf php-8.4.7.tar.gz
cd php-8.4.7
```

<img width="800" height="400" alt="prueba10" src="https://github.com/user-attachments/assets/bae1e482-89b6-4675-b02e-3470cde44b6e" />

```bash
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
```

<img width="800" height="400" alt="prueba11" src="https://github.com/user-attachments/assets/318db1b9-5cb4-41ee-a949-bb1656dab1f5" />


```bash
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

<img width="800" height="400" alt="prueba12" src="https://github.com/user-attachments/assets/e2158230-5885-439c-9c82-745f22f1403b" />


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

<img width="880" height="400" alt="prueba13" src="https://github.com/user-attachments/assets/791f4e3c-0c2a-468f-bf20-c3fa677af4c6" />


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

<img width="1600" height="899" alt="prueba14" src="https://github.com/user-attachments/assets/3be1be6d-3d00-4838-90e8-3c2e7ee32ef6" />


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

<img width="800" height="400" alt="prueba15" src="https://github.com/user-attachments/assets/529b91d7-b431-4354-a669-e25134b5b3c6" />


Se habilitaron y arrancaron ambos servicios:

```bash
systemctl daemon-reload
systemctl enable nginx php-fpm8.4
systemctl start php-fpm8.4
systemctl start nginx
```

<img width="800" height="400" alt="prueba16" src="https://github.com/user-attachments/assets/42a17103-c869-4bf2-85f5-8de4b9b4512d" />


### 8. Verificación del funcionamiento

Se creó el archivo de prueba `phpinfo.php`:

```bash
echo '<?php phpinfo(); ?>' > /srv/nginx/html/phpinfo.php
```

<img width="858" height="402" alt="prueba17" src="https://github.com/user-attachments/assets/44a1875f-e9ce-41a3-9761-a7c9c4e66e2a" />



Se configuró el reenvío de puertos en VirtualBox (host 8080 → guest 80) y se accedió desde el navegador a:

```
http://127.0.0.1:8080/phpinfo.php
```

El resultado mostró correctamente la página de información de PHP 8.4.7 con Server API: FPM/FastCGI, confirmando la comunicación exitosa entre NGINX y PHP-FPM.

<img width="1600" height="850" alt="resultado" src="https://github.com/user-attachments/assets/eb03975b-4a84-4cf5-895d-39250d9fff9b" />


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
