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

## Problemas Encontrados y Soluciones

Durante el desarrollo del proyecto se presentaron diversos inconvenientes que requirieron investigación y solución. A continuación se documentan cada uno de ellos:

---

### Problema 1 — Error de instalación desatendida en VirtualBox

**Descripción:** Al crear la máquina virtual en VirtualBox con la opción *"Proceed with Unattended Installation"* activada, el sistema arrojó el error `Unknown guest OS major version '10'` con código `E_FAIL (0x80004005)`, impidiendo que la instalación automática iniciara.

**Causa:** VirtualBox no reconoce AlmaLinux 10 en su módulo de instalación desatendida por ser una versión muy reciente.

**Solución:** Se desactivó la instalación desatendida y se realizó la instalación de forma manual, seleccionando la opción *"Install AlmaLinux 10.1"* desde el menú GRUB al arrancar la ISO.

---

### Problema 2 — Paquetes no encontrados al instalar dependencias

**Descripción:** Al intentar instalar las dependencias de compilación, varios paquetes no fueron encontrados en los repositorios de AlmaLinux 10: `re2c`, `pcre2-devel`, `libxml2-devel`, `libicu-devel`, `autoconf` y `perl`.

**Causa:** AlmaLinux 10 reorganizó algunos paquetes en repositorios diferentes o cambió sus nombres respecto a versiones anteriores.

**Solución:** Se instalaron los paquetes disponibles primero, luego se habilitó el repositorio EPEL con `dnf install -y epel-release` y se instaló `re2c` desde ese repositorio. Los demás paquetes se instalaron individualmente identificando sus nombres correctos para AlmaLinux 10.

---

### Problema 3 — Fallo de Makefile en PHP (configure incompleto)

**Descripción:** Al ejecutar `make -j$(nproc) && make install` para compilar PHP, el sistema arrojó el error `No targets specified and no makefile found. Stop.`

**Causa:** El comando `./configure` no había completado correctamente su ejecución, por lo que no generó el archivo Makefile necesario para la compilación.

**Solución:** Se ejecutó nuevamente el comando `./configure` con todos los parámetros y se verificó que terminara con el mensaje `Thank you for using PHP` antes de proceder con `make`.

---

### Problema 4 — Falta de librería oniguruma-devel

**Descripción:** Durante la ejecución del `./configure` de PHP, se obtuvo el error `configure: error: Package requirements (oniguruma) were not met: Package 'oniguruma', required by 'virtual:world', not found`.

**Causa:** La librería `oniguruma-devel` necesaria para el soporte de expresiones regulares multibyte no estaba instalada.

**Solución:** Se instaló con `dnf install -y oniguruma-devel` y se volvió a ejecutar el configure.

---

### Problema 5 — Editor nano no instalado

**Descripción:** Al intentar editar archivos de configuración con `nano`, el sistema respondió `-bash: nano: command not found`.

**Causa:** AlmaLinux 10 en su versión minimal no incluye `nano` por defecto.

**Solución:** Se instaló con `dnf install -y nano` y se continuó con la edición de archivos.

---

### Problema 6 — Archivo nginx.conf vacío accidentalmente

**Descripción:** Al intentar crear el archivo `nginx.conf` con el comando `cat > /srv/nginx/conf/nginx.conf << 'EOF'`, el archivo quedó vacío debido a problemas con el teclado en español que impedía escribir correctamente las comillas simples del delimitador `EOF`.

**Causa:** El teclado estaba configurado en distribución latinoamericana mientras que la laptop física tiene distribución inglesa, causando que algunos caracteres especiales no correspondieran.

**Solución:** Se instaló `nano`, se abrió el archivo vacío con `nano /srv/nginx/conf/nginx.conf` y se escribió la configuración manualmente. Adicionalmente se cambió la distribución del teclado a inglés con `localectl set-keymap us`.

---

### Problema 7 — SELinux bloqueando la ejecución de PHP-FPM

**Descripción:** Al intentar iniciar el servicio `php-fpm8.4` mediante systemd, el servicio fallaba con el error `Unable to locate executable '/srv/nginx/sbin/php-fpm': Permission denied` y código `status=203/EXEC`.

**Causa:** SELinux estaba en modo `Enforcing` y bloqueaba la ejecución de binarios compilados manualmente en rutas no estándar como `/srv/nginx/sbin/`.

**Solución:** Se cambió SELinux a modo permissive de forma temporal con `setenforce 0` y de forma permanente editando `/etc/selinux/config` con `sed -i 's/^SELINUX=.*/SELINUX=permissive/' /etc/selinux/config`.

---

### Problema 8 — Permisos insuficientes en los binarios

**Descripción:** Después de desactivar SELinux, el servicio php-fpm seguía fallando con `Permission denied`.

**Causa:** Los binarios `/srv/nginx/sbin/php-fpm` y `/srv/nginx/sbin/nginx` no tenían el bit de ejecución activado.

**Solución:** Se aplicaron permisos de ejecución con:
```bash
chmod +x /srv/nginx/sbin/php-fpm
chmod +x /srv/nginx/sbin/nginx
```

---

### Problema 9 — Errores de sintaxis en nginx.conf

**Descripción:** Al ejecutar `/srv/nginx/sbin/nginx -t` para verificar la configuración, se obtuvieron múltiples errores de sintaxis: falta de punto y coma en la directiva `index`, llave `{` faltante en `location /`, y llave `}` inesperada.

**Causa:** Al escribir manualmente la configuración en nano con teclado en distribución diferente, algunos caracteres como `{`, `}` y `;` se omitieron o escribieron incorrectamente.

**Solución:** Se corrigieron los errores uno por uno usando `nano +<número_de_línea>` para ir directamente a la línea con error, guiándose por los mensajes de `nginx -t`.

---

### Problema 10 — Socket Unix desapareciendo al reiniciar PHP-FPM

**Descripción:** NGINX reportaba continuamente `connect() to unix:/tmp/php84.sock failed (2: No such file or directory)` aunque PHP-FPM estuviera corriendo.

**Causa:** Al reiniciar el servicio `php-fpm8.4`, el socket antiguo en `/tmp/php84.sock` quedaba bloqueado por una instancia anterior, impidiendo que se creara uno nuevo. Esto causaba que PHP-FPM reportara `Another FPM instance seems to already listen on /tmp/php84.sock`.

**Solución:** Se eliminaron todos los procesos de php-fpm con `pkill php-fpm`, se borró el socket huérfano con `rm -f /tmp/php84.sock`, y se cambió la ubicación del socket a `/var/run/php84.sock` tanto en `www.conf` como en `nginx.conf` para evitar conflictos futuros.

---

### Problema 11 — NGINX corriendo como usuario nobody

**Descripción:** Aunque el socket tenía permisos `srw-rw-rw-` (666), NGINX seguía sin poder conectarse a él.

**Causa:** La configuración de NGINX no tenía la directiva `user` definida, por lo que el proceso worker corría como `nobody` en lugar de `nginx`, causando problemas de acceso.

**Solución:** Se agregó la directiva `user nginx nginx;` al inicio del archivo `nginx.conf` y se reinició NGINX.

---

### Problema 12 — Teclado en español dentro de la VM

**Descripción:** Durante toda la práctica se tuvieron dificultades para escribir caracteres especiales como `{`, `}`, `~`, `\`, `|` y `>` dentro de la terminal de la VM, ya que la distribución del teclado configurada en AlmaLinux (latinoamericana) no coincidía con el teclado físico de la laptop (inglés).

**Causa:** Durante la instalación de AlmaLinux se seleccionó el teclado en español latinoamericano, pero la laptop tiene distribución de teclado inglés americano.

**Solución:** Se cambió la distribución del teclado del sistema con `localectl set-keymap us` para que coincidiera con el teclado físico.

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
