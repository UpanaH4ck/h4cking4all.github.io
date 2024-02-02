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
