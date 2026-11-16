# Práctica: Instalación de Moodle en una máquina virtual Ubuntu

![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04%20LTS-E95420?logo=ubuntu&logoColor=white)
![Moodle](https://img.shields.io/badge/Moodle-4.5%20LTS-F98012?logo=moodle&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.3-777BB4?logo=php&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-10.11-003545?logo=mariadb&logoColor=white)

**Duración estimada:** 2 a 3 horas  
**Nivel:** Administración de sistemas / despliegue de aplicaciones web  
**Pila tecnológica:** Ubuntu Server 24.04 LTS, Apache, MariaDB, PHP, Moodle 4.5 LTS (LAMP)

## Índice

1. [Objetivos](#1-objetivos)
2. [Requisitos previos](#2-requisitos-previos)
3. [Parte A: Preparación de la máquina virtual](#3-parte-a-preparación-de-la-máquina-virtual)
4. [Parte B: Instalación de Apache y PHP](#4-parte-b-instalación-de-apache-y-php)
5. [Parte C: Instalación y configuración de MariaDB](#5-parte-c-instalación-y-configuración-de-mariadb)
6. [Parte D: Descarga de Moodle y directorios](#6-parte-d-descarga-de-moodle-y-directorios)
7. [Parte E: Instalación de Moodle](#7-parte-e-instalación-de-moodle)
8. [Parte F: Tareas programadas (cron)](#8-parte-f-tareas-programadas-cron)
9. [Parte G: Verificación y pruebas](#9-parte-g-verificación-y-pruebas)
10. [Solución de problemas frecuentes](#10-solución-de-problemas-frecuentes)
11. [Entregables](#11-entregables)
12. [Preguntas de reflexión](#12-preguntas-de-reflexión)
13. [Ampliaciones opcionales](#13-ampliaciones-opcionales)
14. [Criterios de evaluación sugeridos](#14-criterios-de-evaluación-sugeridos)
15. [Licencia](#15-licencia)

## 1. Objetivos

- Desplegar una plataforma LMS (Moodle) sobre una pila LAMP en una máquina virtual.
- Configurar correctamente servidor web, base de datos, PHP y permisos.
- Completar la instalación web de Moodle y programar las tareas periódicas (cron).
- Verificar el funcionamiento y documentar el proceso.

## 2. Requisitos previos

- Un hipervisor: VirtualBox
- ISO de **Ubuntu Server 24.04 LTS** ([ubuntu.com/download/server](https://ubuntu.com/download/server)).
- Conocimientos básicos de terminal Linux.
- Conexión a Internet en la máquina virtual.

**Recursos recomendados para la VM:**

| Recurso | Valor mínimo |
|---|---|
| CPU | 2 vCPU |
| RAM | 4 GB |
| Disco | 25 GB |
| Red | Adaptador puente (bridge) o NAT con reenvío del puerto 80 |

> **Nota sobre versiones:** esta práctica usa Moodle 4.5 LTS, compatible con PHP 8.3 (el que trae Ubuntu 24.04). Si usas una versión más reciente de Moodle, comprueba antes los requisitos de PHP y base de datos en [docs.moodle.org](https://docs.moodle.org), y ten en cuenta que a partir de Moodle 5.0 el directorio web (`DocumentRoot`) pasa a ser la subcarpeta `public`.

## 3. Parte A: Preparación de la máquina virtual. Instalación de la pila LAMP.

1. Crea la VM con los recursos indicados e instala Ubuntu Server 24.04. Marca la opción **Install OpenSSH server** durante la instalación.
2. Inicia sesión y actualiza el sistema:

```bash
sudo apt update && sudo apt upgrade -y
```

3. Averigua la IP de la VM (la necesitarás más adelante):

```bash
ip -4 addr show | grep inet
```

📸 **Captura 1:** salida del comando anterior.

## 4. Parte B: Instalación de Apache y PHP

1. Instala Apache, PHP y las extensiones que exige Moodle:

```bash
sudo apt install -y apache2 php libapache2-mod-php php-mysql php-xml \
  php-mbstring php-curl php-zip php-gd php-intl php-soap php-ldap \
  php-bcmath php-opcache git unzip graphviz aspell ghostscript
```

2. Comprueba las versiones:

```bash
apache2 -v
php -v
```

3. Ajusta PHP según los requisitos de Moodle (edita **ambos** ficheros, el de Apache y el de la línea de comandos):

```bash
for f in /etc/php/8.3/apache2/php.ini /etc/php/8.3/cli/php.ini; do
  sudo sed -i 's/^;\?max_input_vars.*/max_input_vars = 5000/' $f
  sudo sed -i 's/^upload_max_filesize.*/upload_max_filesize = 100M/' $f
  sudo sed -i 's/^post_max_size.*/post_max_size = 100M/' $f
done
sudo systemctl restart apache2
```

4. Abre `http://IP_DE_LA_VM` en el navegador del equipo anfitrión y comprueba que aparece la página por defecto de Apache.

📸 **Captura 2:** página por defecto de Apache.

## 5. Parte C: Instalación y configuración de MariaDB

1. Instala el servidor de base de datos y asegúralo:

```bash
sudo apt install -y mariadb-server
sudo mysql_secure_installation
```

2. Crea la base de datos (moodleDB) y el usuario de Moodle (moodleTUNOMBRE) (**cambia la contraseña por una propia y moodleTUNOMBRE con los datos correspondientes**):

```bash
sudo mysql
```

```sql
CREATE DATABASE moodleDB DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodleTUNOMBRE'@'localhost' IDENTIFIED BY 'CambiaEstaClave123!';
GRANT ALL PRIVILEGES ON moodleDB.* TO 'moodleTUNOMBRE'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

3. Comprueba que el usuario puede conectarse:

```bash
mysql -u moodleTUNOMBRE -p -e "SHOW DATABASES;"
```

📸 **Captura 3:** resultado del comando anterior mostrando la base de datos `moodle`.

## 6. Parte D: Descarga de Moodle y directorios

1. Descarga el código mediante Git (rama estable 4.5 LTS):

```bash
cd /var/www
sudo git clone -b MOODLE_405_STABLE https://github.com/moodle/moodle.git
```

2. Crea el directorio de datos (`moodledata`), **fuera** del directorio web, y asigna permisos:

```bash
sudo mkdir /var/moodledata
sudo chown -R www-data:www-data /var/moodledata
sudo chmod -R 770 /var/moodledata
```

3. Da al código permisos de solo lectura para el servidor web (más seguro):

```bash
sudo chown -R root:root /var/www/moodle
sudo chmod -R 755 /var/www/moodle
```

4. Configura Apache para servir Moodle. Edita el sitio por defecto:

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Cambia la línea `DocumentRoot` por:

```apache
DocumentRoot /var/www/moodle
```

y añade dentro del bloque `VirtualHost`:

```apache
<Directory /var/www/moodle>
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

5. Reinicia Apache:

```bash
sudo systemctl restart apache2
```

## 7. Parte E: Instalación de Moodle

Tienes dos opciones. Elige **una**.

### Opción 1: Instalador web (recomendada para la práctica)

1. Para que el instalador web pueda escribir el fichero `config.php`, da permisos temporales:

```bash
sudo chmod 777 /var/www/moodle
```

2. Abre `http://IP_DE_LA_VM` en el navegador y sigue el asistente:
   - Idioma: Español.
   - Rutas: directorio de datos `/var/moodledata`.
   - Base de datos: **MariaDB (mariadb/native)**, servidor `localhost`, nombre `moodleDB`, usuario `moodleuserTUNOMBRE` y su contraseña.
   - Acepta la licencia y espera a la comprobación del servidor. Todos los requisitos deben aparecer en verde.
   - Crea la cuenta de administrador y completa el nombre del sitio.

3. Al terminar, **revoca los permisos temporales**:

```bash
sudo chmod 755 /var/www/moodle
sudo chmod 644 /var/www/moodle/config.php
```

### Opción 2: Instalación por línea de comandos

```bash
cd /var/www/moodle
sudo -u www-data /usr/bin/php admin/cli/install.php \
  --lang=es \
  --wwwroot="http://IP_DE_LA_VM" \
  --dataroot="/var/moodledata" \
  --dbtype=mariadb \
  --dbname=moodleDB --dbuser=moodleTUNOMBRE --dbpass='CambiaEstaClave123!' \
  --fullname="Moodle Práctica" --shortname="MoodlePr" \
  --adminuser=admin --adminpass='Admin_2026!' --adminemail=admin@ejemplo.local \
  --agree-license --non-interactive
```

> Si usas esta opción, el directorio `/var/www/moodle` debe ser escribible por `www-data` durante la instalación para crear `config.php`. Después, devuelve los permisos a root como en el punto 3 de la Parte D.

📸 **Captura 4:** pantalla de comprobación de requisitos (o salida final del instalador CLI).  
📸 **Captura 5:** panel principal de Moodle con la sesión de administrador iniciada.

## 8. Parte F: Tareas programadas (cron) OPTATIVO

Moodle necesita ejecutar su cron cada minuto para enviar correos, procesar tareas y mantener el sitio:

```bash
sudo crontab -u www-data -e
```

Añade la línea:

```
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php > /dev/null 2>&1
```

Verifica que funciona ejecutándolo a mano:

```bash
sudo -u www-data /usr/bin/php /var/www/moodle/admin/cli/cron.php
```

En Moodle, entra en **Administración del sitio > Servidor > Tareas > Tareas programadas** y comprueba que las tareas se ejecutan sin errores.

📸 **Captura 6:** listado de tareas programadas con sus últimas ejecuciones.

## 9. Parte G: Verificación y pruebas

Realiza y documenta las siguientes comprobaciones:

1. **Servicios activos:**

```bash
systemctl status apache2 mariadb --no-pager
```

2. **Informe de seguridad y entorno:** en *Administración del sitio > Informes > Comprobaciones de seguridad* y *Informe de rendimiento*. Anota las advertencias y corrige al menos una.
3. **Prueba funcional:** crea un curso de prueba, matricula a un usuario nuevo y comprueba que puede iniciar sesión desde otro navegador.
4. **Reinicio:** reinicia la VM (`sudo reboot`) y comprueba que Moodle vuelve a estar accesible sin intervención.

## 10. Solución de problemas frecuentes

| Síntoma | Causa probable | Solución |
|---|---|---|
| Error 403 Forbidden | Permisos o falta de `Require all granted` | Revisa el bloque `Directory` y los permisos del código |
| Aparece la página por defecto de Apache | `DocumentRoot` mal configurado | Revisa `000-default.conf` y reinicia Apache |
| Faltan extensiones PHP en el instalador | Paquetes no instalados o PHP sin reiniciar | Instala la extensión y ejecuta `sudo systemctl restart apache2` |
| Aviso `max_input_vars` | No se editó el `php.ini` de Apache | Repite el ajuste de la Parte B y reinicia |
| Error de conexión a la base de datos | Usuario, clave o permisos incorrectos | Prueba `mysql -u moodleuser -p` y revisa los `GRANT` |
| No se puede escribir `config.php` | Permisos insuficientes durante la instalación | Aplica el `chmod 777` temporal y revócalo después |
| No accedo desde el anfitrión | Red de la VM | Usa adaptador puente o crea reenvío del puerto 80 en NAT |

## 11. Entregables

Entrega un informe en PDF que incluya:

- Las 6 capturas indicadas, con una breve explicación de cada una.
- Las versiones instaladas de Ubuntu, Apache, PHP, MariaDB y Moodle.
- Una descripción de los problemas encontrados y cómo los resolviste.
- Las respuestas a las preguntas de la sección siguiente.

## 12. Preguntas de reflexión

1. ¿Por qué el directorio `moodledata` debe estar fuera del directorio web?
2. ¿Por qué se recomienda que el código de Moodle no sea escribible por el usuario del servidor web una vez instalado?
3. ¿Qué función cumple el cron en Moodle y qué ocurre si no está configurado?
4. ¿Qué ventajas aportaría usar HTTPS? Indica los pasos generales para activarlo con Let's Encrypt.
5. Propón una estrategia de copias de seguridad para esta instalación (qué elementos hay que respaldar y con qué frecuencia).

Puedes compartirlo y adaptarlo citando la autoría.# moodle
Instalación de un LMS Moodle
