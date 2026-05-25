markdown_content = """# TECNOLÓGICO DE ESTUDIOS SUPERIORES DEL ORIENTE DEL ESTADO DE MÉXICO

**Carrera:** Ingeniería en Sistemas Computacionales  
**Materia:** Taller de Sistemas Operativos  
**Grupo:** 6s11  
**Docente:** Gustavo Moisés Romero González  

---

### Tema: Servidor NGINX y PHP-FPM Compilados desde Código Fuente en AlmaLinux

**Integrantes del Equipo:**
* García González Miguel Ángel
* Ventura Vázquez Lany Camila

**Fecha de Entrega:** 26 de Mayo de 2026  

---

## 1. OBJETIVO GENERAL
Implementar un entorno de servidor web de alto rendimiento mediante la compilación desde el código fuente de NGINX (versión 1.31.x) y PHP (versión 8.4.x) en una distribución empresarial AlmaLinux virtualizada en VirtualBox, garantizando su integración nativa mediante Sockets UNIX y la automatización del ciclo de vida de los servicios mediante el gestor de sistema SystemD.

## 2. OBJETIVOS ESPECÍFICOS
* Configurar el entorno de desarrollo y resolver la matriz de dependencias iniciales mediante gestores de paquetes (`dnf`, repositorios EPEL y CRB).
* Compilar e instalar el servidor web NGINX bajo especificaciones rígidas de arquitectura, aplicando un prefijo de instalación unificado en `/srv/nginx` y asignando privilegios mínimos mediante un usuario de sistema dedicado.
* Compilar e instalar PHP incorporando soporte nativo para el Administrador de Procesos FastCGI (PHP-FPM) y extensiones avanzadas de procesamiento gráfico (GD), internacionalización (intl) y manejo de fechas.
* Diseñar e implementar archivos de unidad (slices) de SystemD personalizados para el control automatizado y persistente de ambos demonios durante el arranque del sistema operativo.
* Configurar un canal de comunicación local optimizado mediante un Socket UNIX en `/tmp/php84.sock`, mitigando latencias de red en comparación con sockets TCP tradicionales.
* Diagnosticar y resolver contingencias en entornos de producción académica como colisiones de puertos con servicios preexistentes (Apache httpd), aislamiento de directorios temporales de SystemD (`PrivateTmp`) y mapeo de variables FastCGI.

---

## 3. DESARROLLO DEL PROYECTO

### Paso 1: Preparación del Entorno y Resolución de Dependencias
Para iniciar el proceso de compilación en AlmaLinux, se requirió la activación de herramientas de desarrollo y librerías específicas que no se incluyen por defecto en la instalación base del sistema operativo.

Se actualizaron los repositorios locales y se habilitó el repositorio **CRB (Code Ready Builder)** y **EPEL**, indispensables para obtener bibliotecas de desarrollo como `libicu-devel` u `oniguruma-devel`:

## A continuación, se instalaron las librerías necesarias para compilar ambos servicios con las directivas solicitadas (Soporte SSL/TLS, compresión de datos, procesamiento de imágenes y localización lingüística):

# Dependencias de NGINX
sudo dnf install pcre-devel zlib-devel openssl-devel -y

# Dependencias de PHP (XML, SQLite, Expresiones Regulares, Gráficos GD e Internacionalización INTL)
sudo dnf install libxml2-devel sqlite-devel oniguruma-devel libpng-devel libjpeg-turbo-devel freetype-devel libicu-devel -y
## Paso 2: Creación de Usuarios y Grupos de Sistema
Por directivas de seguridad informática, los servicios web jamás deben ejecutarse bajo privilegios de root. Se crearon usuarios de sistema sin directorio de inicio (home) y con shells restringidas (/sbin/nologin) para aislar los procesos.

* Grupo del sistema unificado: nginx

* Usuario del servicio web: nginx

* Usuario del procesador PHP: php (asignado al grupo nginx para permitir la intercomunicación fluida).

sudo groupadd nginx
sudo useradd -r -g nginx -s /sbin/nologin nginx
sudo useradd -r -g nginx -s /sbin/nologin php

## Paso 3: Descarga, Compilación e Instalación de NGINX 1.31.x
Se descargó el código fuente oficial de NGINX dentro del directorio temporal de fuentes /usr/local/src. Posteriormente se configuró el instalador definiendo el prefijo (Prefix) solicitado en /srv/nginx, asignando los usuarios correspondientes y activando el módulo criptográfico SSL:

cd /usr/local/src
sudo wget [https://nginx.org/download/nginx-1.31.0.tar.gz](https://nginx.org/download/nginx-1.31.0.tar.gz)
sudo tar -zxvf nginx-1.31.0.tar.gz
cd nginx-1.31.0

# Configuración de compilación con parámetros específicos
sudo ./configure \
    --prefix=/srv/nginx \
    --user=nginx \
    --group=nginx \
    --with-http_ssl_module

# Compilación de binarios e instalación física en el Prefix
sudo make
sudo make install

## Paso 4: Registro del Servicio SystemD para NGINX
Para automatizar el arranque del servidor web junto con el sistema operativo, se programó un archivo de unidad de servicio en la ruta /etc/systemd/system/nginx.service:

[Unit]
Description=Servidor web NGINX (Compilado de Fuente)
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/srv/nginx/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=false

[Install]
WantedBy=multi-user.target

*Nota metodológica: Inicialmente la directiva PrivateTmp se configuró en true, pero durante las pruebas de integración fue modificada a false para evitar el aislamiento del espacio de nombres del directorio /tmp, permitiendo que NGINX accediera al socket UNIX compartido.

# Se procedió a recargar el demonio de SystemD y habilitar el servicio para el inicio automático en el objetivo multiusuario (multi-user.target):
sudo systemctl daemon-reload
sudo systemctl enable nginx

## Paso 5: Descarga, Compilación e Instalación de PHP 8.4.x

Se procedió a descargar y extraer la versión más reciente del intérprete de PHP 8.4.x para su compilación. Se configuró el proceso garantizando la habilitación de PHP-FPM, los usuarios asignados y las extensiones obligatorias de la práctica:
cd /usr/local/src
sudo wget [https://www.php.net/distributions/php-8.4.0.tar.gz](https://www.php.net/distributions/php-8.4.0.tar.gz)
sudo tar -zxvf php-8.4.0.tar.gz
cd php-8.4.0

# Configuración incorporando GD, INTL y FPM
sudo ./configure \
    --prefix=/srv/nginx \
    --enable-fpm \
    --with-fpm-user=php \
    --with-fpm-group=nginx \
    --enable-gd \
    --with-jpeg \
    --with-freetype \
    --enable-intl

# Proceso de compilación y despliegue en directorios del Prefix
sudo make
sudo make install

## Paso 6: Configuración de PHP-FPM y Despliegue del Socket UNIX
Para estructurar el comportamiento de PHP-FPM, se clonaron los archivos de configuración de producción provistos en el código fuente:

sudo cp php.ini-production /srv/nginx/lib/php.ini
sudo cp /srv/nginx/etc/php-fpm.conf.default /srv/nginx/etc/php-fpm.conf
sudo cp /srv/nginx/etc/php-fpm.d/www.conf.default /srv/nginx/etc/php-fpm.d/www.conf

Se editó el archivo del Pool de conexiones de procesos de procesamiento web (/srv/nginx/etc/php-fpm.d/www.conf) mediante el editor nano para forzar la escucha a través de un Socket UNIX localizado en el sistema de archivos compartido, asignando los permisos pertinentes:

user = php
group = nginx

; Definición del Socket UNIX solicitado
listen = /tmp/php84.sock

; Permisos de acceso estructural sobre el archivo de Socket
listen.owner = php
listen.group = nginx
listen.mode = 0660

## Paso 7: Registro del Servicio SystemD para PHP-FPM
Se dio de alta el slice de servicio correspondiente en /etc/systemd/system/php-fpm8.4.service para asegurar el procesamiento automático de scripts PHP desde el arranque:
[Unit]
Description=PHP FastCGI Process Manager 8.4
After=network.target

[Service]
Type=simple
PIDFile=/srv/nginx/var/run/php-fpm.pid
ExecStart=/srv/nginx/sbin/php-fpm --nodaemonize --fpm-config /srv/nginx/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
PrivateTmp=false

[Install]
WantedBy=multi-user.target 

Se guardaron los cambios, se recargaron las configuraciones del sistema y se iniciaron los demon de procesamiento:
sudo systemctl daemon-reload
sudo systemctl enable php-fpm8.4
sudo systemctl start php-fpm8.4

## Paso 8: Integración de NGINX con FastCGI (Intercomunicación)
Para culminar el enlace de infraestructura, se configuró el archivo principal del servidor web /srv/nginx/conf/nginx.conf de manera limpia. Se definieron las rutas absolutas del Document Root y se activó la directiva de enlace FastCGI hacia el Socket UNIX:

user  nginx;
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   /srv/nginx/html;
            index  index.php index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /srv/nginx/html;
        }

        # Configuración del puente de comunicación FastCGI por Socket UNIX
        location ~ \.php$ {
            root           /srv/nginx/html;
            fastcgi_pass   unix:/tmp/php84.sock;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  /srv/nginx/html$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
}

## Paso 9: Bitácora de Resolución de Fallos (Troubleshooting de Práctica)
Durante la etapa de pruebas finales se manifestaron diversas problemáticas, las cuales fueron solventadas con base en un análisis lógico secuencial:

Fallo de Arranque inicial de NGINX: El servicio de SystemD fallaba debido a una colisión en el puerto 80. Al inspeccionar el entorno (ss -tulpn | grep :80), se descubrió la presencia de hilos huérfanos de NGINX activos fuera del entorno de control. Se erradicaron mediante sudo killall -9 nginx, permitiendo que el gestor SystemD tomara el control exclusivo de manera exitosa.

Error 502 Bad Gateway: Tras iniciar ambos servicios, NGINX arrojaba un error 502 al intentar leer el recurso de prueba. Al verificar los descriptores con ls -la /tmp/php84.sock, se demostró que el socket existía pero la directiva de seguridad PrivateTmp=true en el servicio de NGINX aislaba el directorio /tmp en un entorno virtualizado exclusivo de la unidad. El cambio del parámetro a false restableció la visibilidad mutua del socket.

Error File Not Found / Retorno de 0 bytes: Tras solventar la comunicación, PHP-FPM comenzó a emitir un error de archivo ausente. Se detectó que el archivo de prueba /srv/nginx/html/info.php se encontraba vacío (0 bytes) a causa de un fallo de sincronización con el portapapeles bidireccional del software de virtualización VirtualBox. Se ingresó directamente al archivo mediante el editor nano y se transcribió manualmente el script clásico de diagnóstico.

## Paso 10: Validación del Sistema
Una vez solventada la integridad de los datos, se creó el script definitivo de verificación académica:

sudo nano /srv/nginx/html/info.php
Contenido del script:


<?php
phpinfo();
?>
Se aplicó un reinicio general a la infraestructura de servicios para validar la persistencia post-contingencia y se probó la respuesta localmente con la herramienta curl:


sudo systemctl restart nginx
curl http://localhost/info.php
El servidor web retornó exitosamente el volcado de la estructura HTML con los encabezados correspondientes de la versión PHP 8.4.x, confirmando que el entorno opera en un ciclo cerrado de producción local.

## 4. CONCLUSIONES
La realización de esta práctica permitió comprender a bajo nivel el funcionamiento operativo de una pila de servidores web empresarial en arquitecturas Linux modernas. La instalación manual a partir del aislamiento y compilación del código fuente ofrece ventajas sustanciales frente a la instalación genérica por repositorios precompilados, permitiendo una optimización estricta de las directivas del compilador, inyección selectiva de módulos necesarios (como el módulo criptográfico y las extensiones geográficas e idiomáticas de PHP) y un control granular de rutas mediante el prefijo /srv/nginx.

Asi mismo, el diseño de la comunicación mediante archivos de Sockets UNIX demostró ser una alternativa sumamente eficiente para servidores monolíticos o entornos virtualizados locales, al erradicar la sobrecarga de la pila de protocolos TCP/IP, agilizando el intercambio de descriptores de archivos entre el servidor web proxy y el administrador de procesos CGI. Finalmente, la integración con SystemD consolida la adquisición de competencias clave en la administración de servidores, enseñando la relevancia de configurar directivas avanzadas de seguridad y aislamiento de procesos de forma correcta.

## 5. BIBLIOGRAFÍA
Navinda, D. (2021, noviembre 25). Installing Nginx from source. Medium. https://medium.com/@dahamne/installing-nginx-from-source-e13517ba0e9c

Vultr. (2025, julio 16). Cómo instalar el servidor web Nginx en AlmaLinux 9. Vultr Docs. https://docs.vultr.com/how-to-install-nginx-web-server-on-almalinux-9
