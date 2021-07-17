# Detección y explotación de SQL injection

### **Introducción al SQL**

Para entender, detectar y explotar las inyecciones de **SQL**, es necesario entender "**Structured Query Language (SQL)**". El **SQL** permite al desarrollador ejecutar las siguientes solicitudes:

- Para **consultar** información deberás usar la sentencia **`SELECT`**
- Para **actualizar** información deberás usar la sentencia **`UPDATE`**
- Para **agregar** información deberás usar la sentencia **`INSERT`**
- Para **borrar** información deberás usar la sentencia **`DELETE`**

Existen mas operaciones (Creación, eliminación y modificación de tablas, bases de datos y triggers) las cuales no son tan usadas en aplicaciones web.

La sentencia mas utilizada en sitios web es el `SELECT`, el cual es usado para extraer información de una base de datos. La sentencia `SELECT` esta seguida de la siguiente sintaxis:

**Ejemplo**
`SELECT columna1, columna2, columna3 FROM tabla1 WHERE columna4='cadena1'  AND columna5=condicionante1 AND columna6=condicionante2;`

En el consulta de ejemplo, se solicita lo siguiente:

- La sentencia **`SELECT`** indica la acción a ejecutar: consulta información;
- La lista de columnas indica que columnas son las solicitadas
- La sentencia **`FROM tabla1`** indica de que tablas se consultaran los registros
- La sentencia **`WHERE`** va seguida de la condición que deberá cumplir para obtener ciertos registros / campos a consultar

La sentencia  **`'cadena1'`** es el valor en forma de texto delimitado por comilla simple, así como la **`condicionante1`** y **`condicionante2`** pueden ser delimitados por comillas simple o solo introducir el valor numérico

### Ejemplo

Quisiéramos obtener información de la siguiente tabla:

[Table1](https://www.notion.so/cae98e0fbfc649f1a9137272552ed9c5)

Usando la siguiente sentencia:

`SELECT columna1, columna2, columna3 FROM table1 WHERE columna4='user' AND columna5=3 AND columna6=4;`

El resultado seria el siguiente:

[Table1](https://www.notion.so/32fc0803089f4572ba81146c84d2a623)

Como podemos observar, solo regresan los valores que cumplan con la sentencia de la condición seguida del **`WHERE`**

Si se pudiera leer el código fuente de algunas bases de datos, de forma común se podría observar algo así `SELECT * FROM nombreTabla`. Donde `*` se usa como comodín en una consulta para poder regresar **TODAS las columnas** sin listar el nombre de cada una.

> *Todos los métodos que se mostraran mas adelante, se basan en el comportamiento general de las bases de datos para encontrar y explotar los ataques SQL injection depende de muchos factores, aunque estos métodos no son 100% confiables, es muy probable que debas probar varios de ellos para garantizar que el parámetro sea vulnerable*

---

### 1.1 Detección basada en números enteros

De forma muy común que se muestran mensajes de error, se podría detectar de forma muy fácil vulnerabilidades en un sitio web. El ataque **SQL injection** de puede detectar utilizando cualquiera de los siguientes métodos:

Imaginemos que tomamos de ejemplo un sitio web de compras, al acceder a la URL **`/cat.php?id=1`**.
La siguiente tabla muestra el contenido por cada valor de **`id`**

[Tabla1](https://www.notion.so/27b7ba29dd1b45be8dcd9c78b3d372ed)

Supongamos que el código de la consulta esta desarrollado en PHP  y fuera de la siguiente manera:

> `<?php
$id = $_GET["id"];
$result= mysql_query("SELECT * FROM articulos WHERE id=".$id);
$row = mysql_fetch_assoc($result);
?>`

Como se puede observar el valor que ingresa el usuario (`$ _GET ["id"]`) se refleja directamente en la consulta SQL. 

Por ejemplo, si quisiéramos acceder a la URL: `/articulo.php?id=1` generará la siguiente consulta: `SELECT * FROM articulos WHERE id=1` 

**Ejemplo 2:**
`/articulos.php?id=2` generará la siguiente consulta `SELECT * FROM articulos WHERE id=2` 

Si un usuario intentara acceder a la URL `/articulo.php?id=2'`, se ejecutaría la siguiente consulta `SELECT * FROM articulos WHERE id=2'`. 

Sin embargo, la sintaxis de esta consulta SQL es incorrecto debido a la comilla simple``` que rompe la estructura de la consulta, por lo que la base de datos arrojaría un error. 

Por ejemplo, si fuera el caso de una base de datos en MySQL arrojaría el siguiente mensaje de error:

`You have an error in your SQL syntax; check the
manual that corresponds to your MySQL server version for the right syntax to use near ''' at line 1`
*Este mensaje de error puede o no ser visible en la respuesta HTTP dependiendo de la configuración de PHP.*

Ahora sabemos que el valor proporcionado en la URL se inserta directamente en la consulta y este valor se considera un número entero, esto permitiría solicitar a la base de datos que realice una operación matemática básica:

**Por ejemplo:**

- Si intentamos acceder a `/articulo.php?id=**2-1**` , se enviaría la siguiente consulta `SELECT * FROM articulos WHERE id=**2-1**` y la información del `articulo 1` se mostrará en la página web, ya que la consulta anterior es equivalente a `SELECT * FROM articulos WHERE id=**1**`(la resta la realizará automáticamente la base de datos).
- Ahora si intentamos acceder a`/articulo.php?id=**2-0**`, se enviaría la siguiente consulta `SELECT * FROM articulos WHERE id=**2-0**` y la información del `articulo 2` se mostrará en la página web ya que la consulta anterior es equivalente a `SELECT * FROM articulos WHERE id=**2**`

    Con esto podríamos deducir las siguientes propiedades son un buen método para detectar un posible SQL injection:

    - Si accedes a `/articulo.php?id=3-2` muestra `articulo 1` y accedes a `articulo.php?id=1-0` muestra `articulo 1`, la base de datos realiza la resta y **es probable** que hayas encontrado un SQL injection.
    - Si accedes a `/articulo.php?id=2-1` muestra el `articulo 2` y accedes a    `articulo.php?id=2-0` también muestra el `articulo 2`, es **poco probable** que tenga una  SQL injection en un número entero, pero podrías intentar con la detección basada en cadenas como veremos en la siguiente sección.
    - Si pones una comilla simple `'` en la URL ( `/articulo.php?id=1'`), deberías recibir un error por la base de datos.

    **NOTA:** Incluso si un valor es un número entero (por ejemplo, `categoria.php?id=1`), se puede usar como una consulta:
    `SELECT * FROM categoria WHERE id=**'1'**`.
    SQL permite ambas sintaxis, sin embargo, usar una **cadena** `' '` en la consulta SQL será más lento que usar un número entero.

    ---

    ### 1.2 Detección basada en cadenas

    Como vimos en la sección de introducción, las consultas SQL están entre comillas cuando van a usar un valor de tipo cadena (ejemplo 'prueba'):
    `SELECT id,nombre FROM usuarios WHERE nombre='prueba';`

    Si la vulnerabilidad de SQL injection está presente en la pagina web, la inyección de una comilla simple `'` romperá la sintaxis de la consulta y esto provocara un error. Además, si insertamos 2 veces una comilla simple `''` ya no romperá la consulta, ya que si se introduce un numero impar de comillas simples arrojara un error y por lo contrario, si se introduce un numero par, no habría un error.

    Tambien es posible comentar el final de una consulta, por lo que en la mayoría de casos no habrá un error (dependiendo del formato de la consulta). Para comentar el final de la consulta se puede usar `' --`

    Por ejemplo, una consulta vulnerable en el campo `prueba`:
    `SELECT id,nombre FROM usuarios WHERE nombre='prueba' AND id=3;`
    Si se introduce `-- '` seguido del valor de `prueba`:
    `SELECT id,nombre FROM usuarios WHERE nombre='prueba' -- ' AND id=3;`
    Seria interpretado por la base de datos, como:
    `SELECT id,nombr FROM usuarios WHERE nombre='prueba'`

    Ahora supongamos que nuestra consulta tuviera una sintaxis con paréntesis:

    `SELECT id,nombre FROM usarios WHERE (nombre='prueba' AND id=3);`

    Al introducir `-- '` , aun así arrojaría un error ya que faltaría el paréntesis derecho, por lo que se puede intentar con uno o mas paréntesis para encontrar un valor que no provoque un error.

    Otra forma de probarlo es usar `' AND '1'='1`, es menos probable que esta sentencia de inyección afecte a la consulta, ya que es menos probable que lo rompa. Por ejemplo, si introducimos en la consulta anterior, podemos observar que la sintaxis sigue siendo correcta:

    `SELECT id,nombre FROM usuarios WHERE (nombre='prueba' AND '1'='1' AND id=3);`

    La vulnerabilidad SQL injection no es como una ciencia exacta y muchos factores pueden afectar el resultado de tus pruebas. Si crees que algo está funcionando, sigue trabajando en la inyección e intenta averiguar qué hace el código con tu inyección para asegurarse de que sea una vulnerabilidad SQL injection.

    Para encontrar una vulnerabilidad SQL injection, debes explorar el sitio web y probar los métodos anteriores en los parámetros de cada una de las paginas, una vez encontrada la vulnerabilidad puedes pasar a la siguiente sección para aprender como explotarla

    ---

    ### 1.3 Explotación de SQL injection

    Ahora que ya sabes como detectar una vulnerabilidad SQL injection, ahora necesitamos explotarla para extraer información, y para esto es necesario saber usar la sentencia `UNION`.

    ### Sentencia `UNION`

    La sentencia `UNION` se utiliza para unir dos consultas:
    **Ejemplo**

    `SELECT * FROM articulos WHERE id=3 UNION SELECT`
    Por lo regular esta sentencia es usada para obtener información de 2 o mas tablas, se puede usar como un payload en un SQL injection. El atacante no puede modificar directamente el inicio de la consulta, ya que esta declarado en el código del backend, sin embargo, al usar UNION el atacante puede manipular el final de la consulta y obtener información de otras tablas:

    `SELECT id,nombre,precio FROM articulos WHERE id=3 **UNION SELECT** id,login,password  **FROM** usuarios`

    Lo importante de este tipo de consultas es que debe devolver el mismo numero de columnas que tiene la tabla, de lo contrario la base de datos generará un error.

    ### Explotación de un SQL injection con `UNION`

    Para realizar una consulta mediante un SQL injection, debes encontrar el numero de columnas que devuelve la primera parte de la consulta, a menos que tengas el código fuente de la aplicación, tendrías que adivinar o deducir este numero.
    Para esto hay 2 métodos para obtener este dato:

    - Usando `UNION SELECT` y aumento el numero de columnas
    - Usando `ORDER BY`
    - Usando `GROUP BY`

    ### 1.3.2 Usando `UNION SELECT`

    Si quisieras usar `UNION SELECT`y el numero de columnas arrojadas por las dos consultas es diferente, la base de datos generará un error.

    `The used SELECT statements have a different number of columns`

    ¡**Manos a la obra!**

    Imaginemos que queremos usar `UNION SELECT` para adivinar el numero de columnas, y para esto la consulta que se usa al ingresar al sitio `/articulo.php?id=1` es:

    `SELECT id,nombre,precio FROM articulos WHERE id=1`

    Realizaremos los siguientes pasos:

    1. Insertaremos la sentencia `1 UNION SELECT 1`, una vez inyectada la sentencia, la sentencia que se ejecutaría en la base de datos seria la siguiente:

        `SELECT id,nombre,precio FROM articulos WHERE id=1 UNION SELECT 1`

        Por lo que la consulta devolverá un error, ya que le numero de columnas es diferente en las 2 consultas.

    2. Continuaremos incrementando la cantidad de columnas en la consulta hasta que la base de datos deje de generar un error, ejemplo:

        **Intento 2:**

        `1 UNION SELECT 1,2`

        **Intento 3:**

        `1 UNION SELECT 1,2,3`

        Dada que el ultimo intento tiene el mismo numero de columnas que la ambas consultas, la base de datos dejara de generar errores y arrojara información

        1.- Los pasos anteriores solo sirven para bases de datos MySQL, pero para otras bases de datos los valores `1,2,3` deben cambiarse `NULL,NULL,NULL` ... esto aplica en bases de datos que requieren tener el mismo valor en ambas consultas unidas por la sentencia `UNION`

        2.- Para las Bases de Datos Oracle cuando se usa `SELECT`, se debe usar la sentencia `FROM`, se puede usar la tabla dual para complementar la solicitud por ejemplo: 
        `UNION SELECT null,null,null,null FROM dual`

        ### 1.3.3 Usando `ORDER BY`

        El otro método usa la sentencia `ORDER BY` . Esta sentencia se usa principalmente para indicar que columnas se debe usar para ordenar los resultados, por ejemplo:

        `SELECT nombre,apellido,edad,grupos FROM usuarios ORDER BY nombre`

        La solicitud anterior devolverá los usuarios ordenados por la columna `nombre` (3er columna).

        Ahora podemos usar esta sentencia para adivinar el numero de columnas, en lugar de indicar el nombre de la columna (`ORDER BY nombre`), indicaremos el número de la columna (`ORDER BY X`) donde `X` es igual al numero de columna por la que quisiéramos ordenar, por ejemplo:

        `SELECT nombre,apellido,edad,grupos FROM usuarios ORDER BY 3`

        Ahora si usamos un número mayor al del total de columnas, la base de datos generara un error, por ejemplo:

        `SELECT nombre,apellido,edad,grupos FROM usuarios ORDER BY 10`

        y el error seria el siguiente:

        `Unknown column '10' in 'order clause'`

        **¡Manos a la obra!**

        Imaginemos que queremos usar la sentencia `ORDER BY` para adivinar el número de columnas, y para esto la consulta que se usa al ingresar al sitio `/articulo.php?id=1` es:

        `SELECT id,nombre,precio FROM articulos WHERE id=1`

        Realizaremos los siguientes pasos:

        1. Insertaremos la sentencia `1 ORDER BY 5`, una vez inyectada la sentencia, la sentencia que se ejecutaría en la base de datos seria la siguiente:

            `SELECT id,nombre,precio FROM articulos WHERE id=1 ORDER BY 5`

            Por lo que la consulta NO arrojara un error, ya que le número de columnas es menor que el del total de columnas de la tabla.

        2. Continuaremos incrementando el número seguido del `ORDER BY` en la consulta hasta que la base de datos generare un error, ejemplo:

            **Intento 2:**

            `1 ORDER BY 2` - Sin error

            **Intento 3:**

            `1 ORDER BY 3` - Sin error

            **Intento 4:**

            `1 ORDER BY 4` - Error: `Unknown column '4' in 'order clause'`

            Con esto podemos deducir que el numero de columnas es 3, ahora podemos usar esta información para construir la consulta final:

            `SELECT id,nombre,precio FROM articulos where id=1 UNION SELECT 1,2,3`
