# Practica 01-03

## Despliegue de una aplicación web LAMP sencilla

En esta práctica tenemos archivos importados de la práctica 1, en el directorio *conf* está el archivo *000-default.conf*, también tenemos en el directorio *php* el archivo *index.php* y en el directorio *scripts* tendremos el *.env* (variables distintas a la primera práctica) y el script *install_lamp.sh*, estos archivos provienen de la primera práctica. En esta práctica tenemos un script nuevo llamado *deploy.sh*, que es el script de Bash con la automatización del proceso de instalación de la aplicación web LAMP.

### Creación install_lamp.sh

Empezaremos el script con **"#!/bin/bash"**, que es un indicador que le dice al sistema operativo que el script debe ser ejecutado utilizando el intérprete de Bash.

Tras esto, veremos también **"set -x"** que mostrará todos los comandos que se vayan ejecutando.

Lo siguiente que vemos es el comando **"apt update"**, el cuál actualizará los repositorios. También, tendremos el comando **"apt upgrade -y"** que actualizará los paquetes que hemos instalado en el anterior comando a sus últimas versiones.

#### Instalación del servidor web Apache

A continuación, vamos a instalar el servidor web Apache, con el comando **"apt install apache2 -y"**. Aunque no esté reflejado en el script, necesitamos unos comandos para iniciar Apache. Con el comando **"sudo systemctl start apache2"** y con el comando **"sudo systemctl enable apache2"** dejará activado el servidor y no se apagará cada vez que apaguemos la máquina.

#### Instalación de MySQL

El siguiente paso en el script será instalar el sistema gestor de base de datos de MySQL con el comando **"apt install mysql-server -y"**, al igual que con el servidor Apache tendremos que iniciar el servidor con el comando **"sudo systemctl start mysql"** y lo dejamos habilitado con **"sudo systemctl enable mysql"**. Con MySQL instalado podremos acceder a los archivos de configuración en */etc/mysql/mysql.cnf*, a los archivos de log en */var/log/mysql/error.log* y podremos acceder a MySQL con **"sudo mysql"**.

#### Instalación de PHP

Lo siguiente que tenemos será PHP con sus módulos con el comando **"apt install php libapache2-mod-php php-mysql -y"**. Una vez instalado PHP, tendremos en el directorio php que tenemos de la primera práctica un archivo *index.php* en el cuál tendremos una estructura php con el contenido *phpinfo();* que nos permitirá comprobar que la instalación de PHP se ha completado con éxito, si accedes a tu dirección *IP/info.php* verás la página con PHP.

En el directorio *conf* que hemos importado, veremos un archivo llamado *000-default.conf* para la configuración de Apache. En nuestro script, copiaremos ese archivo con el comando **"cp ../conf/000-default.conf /etc/apache2/sites-available"**.

Tras esto, reiniciamos Apache con **"systemctl restart apache2"**.

Antes hemos visto que hemos importado el archivo *index.php* pero no esta implementado en nuestro script y por lo tanto, no se aplicará. Para que funcione copiaremos este archivo en el script con el comando *cp* -> **"cp ../php/index.php /var/www/html"**.

Por último, modificamos el propietario y el grupo de los directorios de forma recursiva del directorio */var/www/html* a través de **"chown -R www-data:www-data /var/www/html"**.

Lo ejecutamos con **sudo ./install_lamp**, no necesitamos darle permisos porque ya los tendrá de la anterior práctica.

### Creación deploy.sh

Este script, como ya he dicho, nos va a permitir la automatización del proceso de instalación de la aplicación web LAMP. Lo primero que tenemos en este script, son las diez primeras linea del *install_lamp.sh* que tendrán la misma función que en script de LAMP.

Pero antes de actualizar, tenemos el comando **"source.env"**, este comando incluye las variables que tenemos en el archivo *.env* que nos servirán para configurar las variables que nos servirán en comandos posteriores, el contenido de este archivo no es el mismo que en la primera práctica, será:

DB_NAME=aplicacion
<br>
DB_USER=lamp_user
<br>
DB_PASSWORD=lamp_password

### Aplicación web LAMP

Lo siguiente que tenemos, es un comando para eliminar las descargas previas de nuestro repositorio */tmp/iaw-practica-lamp* (un directorio temporal) para que cada vez que ejecutemos el script se pueda descargar la aplicación de nuevo y no de problemas, utilizamos **"rm -rf /tmp/iaw-practica-lamp"**.

Una vez que hemos configurado esto, vamos a clonar un repositorio que contiene el código fuente de la aplicación -> **"git clone https://github.com/josejuansanchez/iaw-practica-lamp.git /tmp/iaw-practica-lamp"** que tiene una base de datos y un directorio *source* (código fuente) que contiene entre otros, el archivo *config.php* que configuraremos ahora y que contiene las variables.

Tras clonar el repositorio, vamos a mover su código fuente al directorio */var/www/html* con el comando *mv*,**"mv /tmp/iaw-practica-lamp/src/* /var/www/html"**. 

Ya que tenemos esto copiado, como he mencionado, dentro de aquí está el archivo *config.php*, el siguiente paso va a ser configurar este archivo:

sed -i "s/database_name_here/$DB_NAME/" /var/www/html/config.php
<br>
sed -i "s/username_here/$DB_USER/" /var/www/html/config.php
<br>
sed -i "s/password_here/$DB_PASSWORD/" /var/www/html/config.php

Lo que acabamos de hacer es reemplazar la variables que tenía la aplicación y hemos puesto las nuestras que tenemos en el archivo *.env*.

Aunque hayamos configurado las variables para el nombre, el usuario y la contraseña de la base de datos, la base de datos no puede tener cualquier nombre que le pongamos ya que se llama *lamp_db* para poder cambiarselo utilizamos de nuevo el comando sed:

**sed -i "s/lamp_db/$DB_NAME/" /tmp/iaw-practica-lamp/db/database.sql**

Una vez realizada toda la configuración, vamos a importar el script de la base de datos con el comando **"mysql -u root < /tmp/iaw-practica-lamp/db/database.sql"**. También, vamos a crear el usuario de la base de datos asignandole privilegios:

mysql -u root <<< "DROP USER IF EXISTS $DB_USER@'%'"
<br>
mysql -u root <<< "CREATE USER $DB_USER@'%' IDENTIFIED BY '$DB_PASSWORD'"
<br>
mysql -u root <<< "GRANT ALL PRIVILEGES ON $DB_NAME.* TO $DB_USER@'%'"

Este comando lo que hace es, primero eliminar usuario si existe el usuario que tenemos en nuestra variable. También, crea usuario identificado por la contraseña asignada en la variable y da todos los privilegios al usuario sobre la base de datos.

Para terminar, como siempre, modificamos el grupo y el propietario de */var/www/html* con **"chown -R www-data:www-data /var/www/html"**.

# IAW - Práctica 01-04
En esta práctica vamos a crear un certificado SSL/TLS autofirmado con la herramienta openssl. Una vez creado vamos a configurar el servidor web Apache para que utilice dicho certificado.

Para esta práctica tenemos un directorio *conf* donde habrá dos archivos de configuración de los que hablaré posteriormente, también en el directorio *scripts* tenemos un archivo *.env* donde estarán las variables, tenemos el script para instalar la pila LAMP (*install_lamp.sh*) y por último, el nuevo script que vamos a tener en esta práctica es el setup_selfsigned_certifice.sh que nos va a permitir crear y configurar un certificado SSL autofirmado.


Todos los archivos excepto *default-ssl.conf* y *setup_selfsigned_certificate* han sido importados de la práctica 01-03, aunque en el *.env* las variables van a cambiar y el *000-default.conf* va a tener otra configuración.

## Creación install_lamp.sh
Empezaremos el script con **"#!/bin/bash"**, que es un indicador que le dice al sistema operativo que el script debe ser ejecutado utilizando el intérprete de Bash.

Tras esto, veremos también **"set -x"** que mostrará todos los comandos que se vayan ejecutando.

Lo siguiente que vemos es el comando **"apt update"**, el cuál actualizará los repositorios. También, tendremos el comando **"apt upgrade -y"** que actualizará los paquetes que hemos instalado en el anterior comando a sus últimas versiones.

### Instalación del servidor web Apache

A continuación, vamos a instalar el servidor web Apache, con el comando **"apt install apache2 -y"**. Aunque no esté reflejado en el script, necesitamos unos comandos para iniciar Apache. Con el comando **"sudo systemctl start apache2"** y con el comando **"sudo systemctl enable apache2"** dejará activado el servidor y no se apagará cada vez que apaguemos la máquina.

### Instalación de MySQL

El siguiente paso en el script será instalar el sistema gestor de base de datos de MySQL con el comando **"apt install mysql-server -y"**, al igual que con el servidor Apache tendremos que iniciar el servidor con el comando **"sudo systemctl start mysql"** y lo dejamos habilitado con **"sudo systemctl enable mysql"**. Con MySQL instalado podremos acceder a los archivos de configuración en */etc/mysql/mysql.cnf*, a los archivos de log en */var/log/mysql/error.log* y podremos acceder a MySQL con **"sudo mysql"**.

### Instalación de PHP

Lo siguiente que tenemos será la instalación de PHP con sus módulos con el comando **"apt install php libapache2-mod-php php-mysql -y"**.

En el directorio *conf* que hemos importado, veremos un archivo llamado *000-default.conf* para la configuración de Apache. En nuestro script, copiaremos ese archivo con el comando **"cp ../conf/000-default.conf /etc/apache2/sites-available"**. Tras esto, reiniciamos Apache con **"systemctl restart apache2"**. El contenido de este archivo cambiará más adelante.

Por último, modificamos el propietario y el grupo de los directorios de forma recursiva del directorio */var/www/html* a través de **"chown -R www-data:www-data /var/www/html"**.

Lo ejecutamos con **sudo ./install_lamp**, no necesitamos darle permisos porque ya los tendrá de la anterior práctica.

## Creación setup_selfsigned_certificate.sh

Lo primero que tenemos son que las 10 primeras líneas son copiadas del *install_lamp.sh* que sirven para lo que he explicado en la pila LAMP.

Antes de todo, importamos el archivo *.env* del cuál las variables que vamos a configurar nos van a servir en el script dentro de unos pasos.

A continuación, vamos a crear un certificado y una clave privada con el comando:
```
openssl req \
  -x509 \
  -nodes \
  -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/ssl/private/apache-selfsigned.key \
  -out /etc/ssl/certs/apache-selfsigned.crt \
  -subj "/C=$OPENSSL_COUNTRY/ST=$OPENSSL_PROVINCE/L=$OPENSSL_LOCALITY/O=$OPENSSL_ORGANIZATION/OU=$OPENSSL_ORGUNIT/CN=$OPENSSL_COMMON_NAME/emailAddress=$OPENSSL_EMAIL"
```
Se ha utilizado la utilidad openssl con los siguientes parámetros, *req* que se utiliza para crear certificados autofirmados, *-x509* que indica que queremos crear un certificado autofirmado en lugar de una solicitud de certificado, *-nodes* que hace que la clave privada del certificado no esté protegida por contraseña y esta sin encriptar, *-days 365* que nos dice la validez del certificado, en este caso de 365 días, *-newkey rsa:2048* para generar una nueva clave privada RSA de 2048 bits junto con el certificado, *-keyout /etc/ssl/private/apache-selfsigned.key* para determinar la ubicación y el nombre del archivo donde se guardará la clave privada generada, *-out /etc/ssl/certs/apache-selfsigned.crt* que indica el lugar y el nombre del archivo donde se guarda la clave privada que vamos a generar. Sin la última parte del comando, cuando lo ejecutemos, nos va a pedir que introduzcamos una serie de datos que irán al certificado, de esta manera sería hacerlo a mano. Para automatizar este proceso, en el *.env* vamos a definir las variables que necesita el certificado y lo vamos a añadir a nuestro script con **-subj "/C=$OPENSSL_COUNTRY/ST=$OPENSSL_PROVINCE/L=$OPENSSL_LOCALITY/O=$OPENSSL_ORGANIZATION/OU=$OPENSSL_ORGUNIT/CN=$OPENSSL_COMMON_NAME/emailAddress=$OPENSSL_EMAIL"**, como vemos tenemos el país, la provincia, localidad, organización, sección de la organización, nombre del dominio y el email sin tener que introducirlo manualmente.

Seguimos viendo el script, y ahora vamos a copiar el archivo de configuración de Apache para HTTPS, vamos a habilitar el tráfico de HTTPS. Su contenido tiene el siguiente significado:

- <VirtualHost *:443>: Permite escuchar por el puerto 443 que es el de HTTPS.
- ServerName: El nombre de dominio que asignemos.
- DocumentRoot: Ruta del directorio raíz del host virtual
- SSLEngine on: Para que se utilice SSL.
- SSLCertificateFile: La ruta del certificado que hemos autofirmado
- SSLCertificateKeyFile: La ruta de la clave privada de nuestro certificado.

Una vez que hemos visto el *default-ssl.conf*, lo copiamos en nuestro script con **"cp ../conf/default-ssl.conf /etc/apache2/sites-available/"** que copiará el archivo de configuración en */etc/apache2/sites-available/*.

Seguimos, y ahora habilitamos el virtual host para HTTPS a través del comando **"a2ensite default-ssl.conf"**, *a2ensite* sirve para eso, para habilitar un sitio web especificado que contiene un bloque VirtualHost dentro de la configuración de apache2. También, habilitamos el modulo SSL con **"a2enmod ssl"**.

### Configuración para que las peticiones a HTTP se redirijan a HTTPS

Vamos a configurar nuestro archivo *000-default.conf* para hacer que las peticiones de HTTP se redirijan a HTTPS, el contenido de este archivo es el siguiente:
```
<VirtualHost *:80>
    DocumentRoot /var/www/html

    # Redirige al puerto 443 (HTTPS)
    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>
```
Por un lado, *RewriteEngine* habilita el motor de reescritura de URLs y nos permite usar reglas de reescritura. *RewriteCond %{HTTPS} off* es una condición que comprueba si la petición recibida utiliza HTTPS o no.

*RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]*: Las reglas de reescritura tienen la siguiente sintaxis *RewriteRule Pattern Substitution [flags]*. Esto tiene el significa de que *Pattern* es el patrón que tiene que cumplirse en la URL para que la regla de reescritura se aplique, *Substitution* se refiere a la URL a la que se redirige la solicitud y los *flags* cambian el comportamiento de la regla, en este caso, indica que es una redirección permanente.

Este archivo vamos a copiarlo en el directorio */etc/apache2/sites-available*, para hacerlo en nuestro script vamos a copiarlo en este directorio con el comando *cp* -> **"cp ../conf/000-default.conf /etc/apache2/sites-available"**.

Antes de terminar nuestro script, tenemos que habilitar el módulo rewrite para hacer la redirección de HTTP a HTTPS, **"a2enmod rewrite"** y reiniciamos el servicio de Apache **"systemctl restart apache2"**.

Para terminar, podemos configurar en el script una variable para que podamos poner cualquier dominio que queramos y no un único dominio siempre, esta marcado como comentario porque no está aplicado. Para aplicarlo con el comando **"sed -i "s//$PUT_YOUR_DOMAIN_HERE" /etc/apache2/sites-available"** modificamos el server name para cualquier dominio y en las variables configuramos el nombre con *PUT_YOUR_DOMAIN_HERE*.

Para ejecutarlo, en el terminal entramos a nuestro directorio *scripts* y le damos permisos al script *setup_selfsigned_certificate.sh* con **chmod +x setup_selfsign