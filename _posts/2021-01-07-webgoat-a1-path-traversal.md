---
title: WebGoat - A1 - Path Traversal
author: Christian Gutierrez
date: 2021-01-07 14:10:00 -0500
categories: [WebGoat, A1]
tags: [PathTraversal]
image: /assets/webgoat/A1/3_path_traversal/portada.PNG
---
## Introducción
De forma breve, como lo indica la descripción de la vulnerabilidad el **Path Traversal** nos permite crear, leer, editar o eliminar archivos fuera la ubicación definida por el aplicativo; Algunos posibles escenarios de explotación son los siguientes:

- Enumeración de archivos internos:
    - Pensemos en un aplicativo que retorna un mensaje de existencia como _"No es posible leer el archivo"_ cuando se intenta crear un archivo en una ruta por ejemplo como: ``../../ultra_secreto.txt``, esto nos permite realizar ataques por diccionario con nombres comunes, alternando entre directorios ``../``. 

- Sobrescribir un archivo interno:
    - La manera más directa de ver el impacto, es cuando además de la ruta de creación, también controlamos el nombre con el que se almacena el archivo, por lo que podemos sobrescribir componentes internos; un ejemplo sería identificar la ruta del ``index.php`` y resubir un archivo con el mismo nombre ``../../index.php`` y que en su contenido se incluya algún tipo de sniffer para el formulario login. 

- Eliminar un archivo interno: 
    - Evidentemente, si podemos sobrescribir un archivo podemos eliminarlo o llevar su contenido a "vacio", posibilitando la eliminación de componentes de seguridad o de core que afecten el aplicativo. Un caso sería cuando se controla la ruta y el nombre, pero existe un filtro de contenido que evita la inclusión de código, se puede inyectar un archivo vacio o con información basura. 

- Lectura de archivos internos: 
    - Manipulando las rutas mediante los saltos de directorio ``../`` y ``..\`` en una funcionalidad de lectura, es posible acceder al contenido de archivos internos de alta criticidad, como por ejemplo: web.config, .htaccess, config.php, wp-config.php, etc...

## Path Traversal - #2

![Ejercicio #2](/assets/webgoat/A1/3_path_traversal/01_ejercicio_2.png)
_Ejercicio #2_

Se tiene un formulario común de actualización de perfil de usuario, en el que se puede subir una imagen/foto/avatar, y además se indican 3 parametros a rellenar, que vistos desde la herramienta ```BurpSuite``, se envian al aplicativo los siguientes datos: 
- ``uploadFile``
- ``fullName``
- ``email``
- ``password``

Como consejo general, se recomienda primero ver el comportamiento de las funcionalidades, la información enviada y la información que se recibe para tomarlo como base y así identificar de forma sencilla cambios o comportamientos que se enlazan a los ataques que inyectamos. 

![Paquete Inicial](/assets/webgoat/A1/3_path_traversal/01_paquete_inicial.png)
_Paquete Inicial_

Para este ejemplo se ingreso como nombre **test**; de la respuesta se pueden deducir varias cosas: 
1. El archivo se almacenará bajo el nombre agregado en el parámetro ``fullName``.
2. Aunque inicialmente se almacena con una extensión, se puede agregar en el ``fullname``.
3. No hay un filtro ni ninguna regla que nos obligue a subir archivos de IMAGEN o algo particular. 

Teniendo lo anterior en mente, se realiza la prueba básica de agregar al futuro nombre el archivo un salto de directorio ``../`` y comprobar si la aplicación fija los archivos a un directorio destino, y si el aplicativo cuenta con filtros de este tipo de caracteres: 

![Reto Superado](/assets/webgoat/A1/3_path_traversal/01_vulnerabilidad_pasada.png)
_Reto Superado_

Como se observa de la respuesta, hemos pasado el reto ;)... sin embargo, veamos a nivel de código qué sucede: 
- [**Código base de la lección Path Traversal**](https://github.com/WebGoat/WebGoat/tree/develop/webgoat-lessons/path-traversal/src/main/java/org/owasp/webgoat/path_traversal)

Veremos únicamente el código afectado, pero puedes profundizar en el enlace anterior, accedienco a ``ProfileUploadBase.java``.

```java
@PostMapping(value = "/PathTraversal/profile-upload", consumes = ALL_VALUE, produces = APPLICATION_JSON_VALUE)
@ResponseBody
public AttackResult uploadFileHandler(@RequestParam("uploadedFile") MultipartFile file, @RequestParam(value = "fullName", required = false) String fullName) {
    return super.execute(file, fullName);
}
```
Como se observa solo se requieren los parametros ``uploadedFile`` y ``fullName``, los cuales no tienen ningún proceso de filtrado o saneamiento, estos se pasan directamente a la funcionalidad [execute()](https://github.com/WebGoat/WebGoat/blob/60c7fdd0dbcbc09aaa22f5c772666c716344f8cf/webgoat-lessons/path-traversal/src/main/java/org/owasp/webgoat/path_traversal/ProfileUploadBase.java#L26) que se encarga de almacenar el archivo el directorio ``"/PathTraversal/" + webSession.getUserName()``, por lo cual, al inyectar el payload en ``fullName`` se logra activar la vulnerabilidad.


# Path Traversal - #3

El problema nos indica que el desarrollador ha solucionado la vulnerabilidad, por lo que debemos encontrar otro vector para explotarla. De nuevo revisamos los datos enviados y recibidos. 

![Paquete Inicial](/assets/webgoat/A1/3_path_traversal/02_paquete_inicial.png)
_Paquete Inicial_

Un punto importante es que ahora la funcionalidad que se consume es ``/profile-upload-fix``, y la primera prueba a intentar es replicar la vulnerabilidad anterior, para verificar si ha sido mitigada. 

Ahora que sabemos que ha sido mitigada, nos planteamos varios escenarios o soluciones, se pudo implementar: 
- Una fijación de directorio. 
- Un filtrado de caracteres. 
- Un cambio de nombre del archivo final, por uno aleatorio o estatico.

Existen varias opciones, unas más seguras que otras. Pero recordemos el enunciado del problema y la literalidad y falta de contexto con que se le indica un error a un desarrollador; En muchos casos si me piden solucionar un fallo X no hay tiempo para pensar en XY o XZ, es por esto que al aplicar una mitigación sin un conocimiento general o técnico de la vulnerabilidad se puede incurrir en la creación de una brecha nueva. 

> Si me piden eliminar los caracteres ``../``, yo lo hago.
> Desarrollador, 2021 :v

Imaginemos que una de las opciones para hacer la eliminación es utilizar una funcionalidad de ``replace("../", "")`` en donde no hay cabida para los caracteres que explotan la vulnerabilidad inicial, sin embargo, no se contemplo la inyección de payload ofuscados o pre-construidos. 
- ..**../**/

Si inyectamos la cadena anterior, el filtro solo detectará y reemplazará la parte central donde encuentra el patrón, no obstante, al eliminarlo o reemplazarlo por vacio "", la unión de la cadena restante resulta nuevamente en el payload malicioso. 

- ..**../**/..**../**/etc/passwd
- ../../etc/passwd

Pongamoslo en practica: 

![Reto Superado](/assets/webgoat/A1/3_path_traversal/02_vulnerabilidad_pasada.png)
_Reto Superado_

La utilización de funcionalidades de ``replace`` se es susceptible a estos comportamientos en que se inyectan payload's pre-construidos, ya que no es usual encontrar que estas funcionalidades sean iterativas hasta identificar que no estan los caracteres maliciosos. Ahora veamos el código fuente: 
- [**Código Path Traversal ProfileUploadFix**](https://github.com/WebGoat/WebGoat/blob/develop/webgoat-lessons/path-traversal/src/main/java/org/owasp/webgoat/path_traversal/ProfileUploadFix.java)

```java
@PostMapping(value = "/PathTraversal/profile-upload-fix", consumes = ALL_VALUE, produces = APPLICATION_JSON_VALUE)
@ResponseBody
public AttackResult uploadFileHandler(
        @RequestParam("uploadedFileFix") MultipartFile file,
        @RequestParam(value = "fullNameFix", required = false) String fullName) {
    return super.execute(file, fullName != null ? fullName.replace("../", "") : "");
}
```

El código es bastante simple, se identifica el comportamiento que habiamos previsto y se observa que únicamente es aplicado al parametro ```fullName```.

# Path Traversal - #4

La descripción del reto nos indica que el desarrollador esta aburrido con este error y como se da en el parametro ``fullName``, decide no utilizarlo. (¿En principio, por qué lo usaba :| ?). Como es costumbre, debemos verificar sí en realidad se realizo la mitigación. 

![Paquete Inicial](/assets/webgoat/A1/3_path_traversal/03_paquete_inicial.png)
_Paquete Inicial_

Revisando la respuesta con detenimiento efectivamente ya no hace uso del dato inyectado en ``fullName``, pero entonces ¿De dónde saca el nombre de la imagen?. 
- Vemos que el nombre con el que se guarda la imagen es el original de la imagen. 
- Ahora archivo es almacenado con extensión. 
- ¿Hay algún filtro de tipo de archivo? ¿Podemos subir cualquier archivo?.

En este reto se explota un error muy común en los aplicativos: 
> !Todo lo enviado por el cliente es manipulable y debe ser validado¡ 

Así que nuestro desarrollador cree que al utilizar directamente el nombre del archivo se mitiga la vulnerabilidad por que estos no pueden contener código malicioso o no pueden ser manipulados. Ya veremos que no es así.

![Reto Superado](/assets/webgoat/A1/3_path_traversal/03_vulnerabilidad_pasada.png)
_Reto Superado_

Esta vez el payload va en el parametro a ``uploadedFileRemoveUserInput`` el cual transporta el cuerpo de la imagen/archivo que se intenta subir, así pues el aplicativo no aplica filtros sobre este parametro y lo inyecta a la ruta donde se almacenará. 

Ahora veamos el código fuente: 
- [**Código Path Traversal ProfileUploadRemoveUserInput**](https://github.com/WebGoat/WebGoat/blob/develop/webgoat-lessons/path-traversal/src/main/java/org/owasp/webgoat/path_traversal/ProfileUploadRemoveUserInput.java)

```java
@PostMapping(value = "/PathTraversal/profile-upload-remove-user-input", consumes = ALL_VALUE, produces = APPLICATION_JSON_VALUE)
@ResponseBody
public AttackResult uploadFileHandler(@RequestParam("uploadedFileRemoveUserInput") MultipartFile file) {
    return super.execute(file, file.getOriginalFilename());
}
```

Efectivamente el aplicativo realiza la lectura del nombre del archivo directamente de las propiedades del objeto ``file`` y, lo ejecuta sin validación o filtrado, por lo que es vulnerable. 


# Path Traversal - #5

![Ejercicio #2](/assets/webgoat/A1/3_path_traversal/04_ejercicio_5.png)
_Ejercicio #2_

Este nuevo reto nos indica que debemos leer el archivo ``path-traversal-secret.jpg``, pero aparentemente no contamos con ninguna funcionalidad ni formulario en el cual inyectar el ataque, solo tenemos gaticos tiernos. 

Aunque en el paquete enviado no aparecen parametros que podamos inyectar, se identifica algo "raro" en las cabeceras de respuesta: 

![Paquete Inicial](/assets/webgoat/A1/3_path_traversal/04_paquete_inicial.png)
_Paquete Inicial_

Vemos que la cabecera ``location`` contiene información del uso de un parametro ``id`` en la funcionalidad, así que presumimos se puede usar para realizar nuestro ataque, pero tenemos los siguientes inconvenientes: 

- Al utilizar los caracteres ``..`` y ``/`` nos retorna un mensaje de error indicando que son caracteres no validos. 

Para que nuestro desarrollador a tomado medidas y ahora filtra los caracteres corruptos por separado. Entonces ¿Cómo podemos evadir el filtro si ambos caracteres estan betados? 

Vamos a utilizar una técnicas de cambio de codificación, en la que enviamos el mismo contenido pero en una presentación distinta y así evitamos que el filtro detecte los caracteres que intentamos inyectar. 

- ``../`` 
- ``%2e%2e%2f``

Para este caso los payloads anteriores son equivalentes ya que representan lo mismo, el primero se encuentra en representación ASCII habitual y, el segundo esta codificado en URL; La diferencia es que la presentación en URL no contiene los caracteres invalidos que nos indica la aplicación y nos permitirá realizar el ataque. 

![Primera Inyeccion de Payload](/assets/webgoat/A1/3_path_traversal/04_primera_inyeccion.png)
_Primera Inyeccion de Payload_

Vemos un comportamiento distintos que nos lista el contenido del directorio ``../``, así que debemos buscar el archivo ``path-traversal-secret.jpg`` e intentamos acceder a más subdirectorios. 

![Reto Superado](/assets/webgoat/A1/3_path_traversal/04_payload_final.png)
_Reto Superado_

- ``id  = ../../ path-traversal-secret``
- ``id  = %2e%2e%2f%2e%2e%2fpath-traversal-secret``

Ahora veamos el código fuente: 
- [**Código Path Traversal ProfileUploadRemoveUserInput**](https://github.com/WebGoat/WebGoat/blob/develop/webgoat-lessons/path-traversal/src/main/java/org/owasp/webgoat/path_traversal/ProfileUploadRetrieval.java)


Condición de filtrado: 
```java
if (queryParams != null && (queryParams.contains("..") || queryParams.contains("/"))) {
    return ResponseEntity.badRequest().body("Illegal characters are not allowed in the query params");
}
```

Una vez de pasa la condición anterior, se procede a enviar el nombre a la funcionalidad que crea el archivo: 

```java
var id = request.getParameter("id");
var catPicture = new File(catPicturesDirectory, (id == null ? RandomUtils.nextInt(1, 11) : id) + ".jpg");
```

