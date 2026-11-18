# PRACTICA 4

El objetivo de esta práctica es desarrollar un sistema de navegación autónoma para un robot logístico capaz de recoger, transportar y depositar estanterías en diferentes ubicaciones de un almacén.

Para ello, el robot dispone de información sobre su posición y de un mapa del entorno, que utiliza para planificar sus desplazamientos y evitar posibles colisiones con los obstáculos. La práctica se desarrolla en el entorno de simulación Amazon Warehouse de Robotics Academy, utilizando Gazebo para representar el almacén y OMPL para realizar la planificación de trayectorias.


Hemos desarrollado tres soluciones diferentes para abordar el problema:

- **Caso 1**: robot holonómico aproximado como un punto, utilizando un mapa de obstáculos engordados.

- **Caso 2**: robot holonómico representado mediante una geometría rectangular, considerando su posición y orientación.

- **Caso 3**: robot Ackermann, teniendo en cuenta sus restricciones cinemáticas y utilizando trayectorias basadas en curvas de Reeds-Shepp.

En los tres casos, el objetivo final es trasladar las seis estanterías a las zonas de almacenamiento señalizadas, actualizando la representación del entorno después de cada movimiento.


## CASO_1: Robot holonómico como un punto

### Planteamiento 

En este primer caso simplificamos el robot holonómico representándolo como un punto en el plano. Para tener en cuenta sus dimensiones físicas, utilizamos una técnica conocida como engordamiento de obstáculos. La idea consiste en aumentar el tamaño de los obstáculos del mapa según el radio del robot. De esta forma, OMPL puede planificar los movimientos considerando que el robot es un punto, sin necesidad de comprobar toda su geometría.

Cuando el robot transporta una estantería, aumentamos el radio utilizado para engordar los obstáculos, ya que el conjunto ocupa una superficie mayor.

<br>

### Representación del mapa

Partimos del mapa original del almacén, que hemos obtenido mediante `WebGUI.getMap()`. Este mapa se transforma en una matriz en la que los píxeles representan las zonas libres y ocupadas.

Para trabajar con las coordenadas del entorno de Gazebo, implementamos la función `a_pixel()`, que transforma las coordenadas en metros a coordenadas de píxel. También modificamos el mapa inicial para representar correctamente las estanterías:

- Cuando el robot está vacío, cada estantería se representa mediante sus cuatro patas.

- Cuando transporta una estantería, esta se representa como un obstáculo rectangular de 2,0 × 0,8 metros.

- La estantería que transporta el robot se elimina de los obstáculos, puesto que pasa a formar parte del propio robot.

<br>

### Engordamiento de obstáculos

La función `engordar()` recorre los píxeles ocupados del mapa y dibuja un círculo alrededor de cada uno mediante `cv2.circle()`.

El radio de estos círculos depende de la situación del robot:

- Robot vacío: se utiliza un radio de 0,5 metros.

- Robot cargado: se utiliza un radio de 1,1 metros.

A ambos valores se les añade un margen de seguridad de 0,1 metros y de esta manera, obtenemos un mapa modificado en el que las zonas ocupadas se han ampliado para tener en cuenta el espacio que necesita el robot durante sus desplazamientos.

<br>

### Planificación con OMPL

Para calcular los desplazamientos utilizamos la biblioteca OMPL y el algoritmo RRT* (Rapidly-exploring Random Tree Star).

Como el robot se representa mediante un punto, utilizamos `RealVectorStateSpace(2)`, que define un espacio de estados formado únicamente por las coordenadas X e Y.

Por otro lado, la función `estado_valido()` comprueba si el píxel correspondiente a una posición está libre en el mapa engordado. Si está ocupado, el estado se considera inválido.

Una vez definidos los estados inicial y final, configuramos el problema de planificación utilizando `PathLengthOptimizationObjective`, cuyo objetivo es minimizar la longitud del recorrido y finalmente, ejecutamos RRT* durante un tiempo máximo de 3 segundos y obtenemos una secuencia de puntos que representa el camino calculado.

<br>

### Seguimiento del camino

Una vez obtenido el camino, el robot debe desplazarse siguiendo los puntos generados por OMPL.

Para ello, utilizamos dos funciones principales:

1. `girar_hasta()`: realiza un giro sobre sí mismo hasta alcanzar la orientación deseada.

2. `avanzar_hasta()`: desplaza el robot hacia un punto determinado, ajustando su velocidad lineal y angular.

La función `seguir_camino()` recorre los puntos de la trayectoria. Para alcanzar cada uno, primero calcula la orientación necesaria y después realiza el desplazamiento de esta forma, el movimiento se divide en una sucesión de giros y desplazamientos rectos.

<br>

### Recogida y entrega de estanterías

El programa principal recorre las seis estanterías del almacén y realiza las siguientes operaciones para cada una:

1. `Aproximación`: el robot planifica un camino hasta la estantería sin transportar ninguna carga.

2. `Recogida`: detiene el movimiento y utiliza HAL.lift() para levantar la estantería.

3. `Transporte`: actualiza el mapa para considerar la nueva geometría del conjunto y planifica el desplazamiento hasta la zona de destino.

4. `Entrega`: se aproxima al hueco correspondiente y utiliza HAL.putdown() para depositar la estantería.

5. `Actualización del mapa`: registra la nueva posición de la estantería para que los siguientes cálculos tengan en cuenta su ubicación.

Las seis estanterías se distribuyen entre las dos zonas señalizadas del almacén, colocando tres en cada una. Una vez completadas todas las operaciones, se muestra el mapa final del almacén.

<br>

### Vídeo del funcionamiento 

https://urjc-my.sharepoint.com/:v:/r/personal/j_sanmiguel_2023_alumnos_urjc_es/_layouts/15/stream.aspx?id=%2Fpersonal%2Fj_sanmiguel_2023_alumnos_urjc_es%2FDocuments%2FROBOTICA%2FAmazonWarehouse_CASO1.mp4&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D&share=cQoB4KC4fE_ES7sQEZ0DzqSZEgUCmulyflxd-Xwi7FkzNU4_Bw

<br>
<br>

## CASO_2: Robot holonómico con geometría rectangular

### Planteamiento

En este segundo caso mejoramos la representación del robot. En lugar de considerarlo un punto, tenemos en cuenta su geometría rectangular y su orientación durante la planificación, lo que nos permite comprobar si el robot puede ocupar una determinada posición sin que ninguna parte de su estructura colisione con los obstáculos. Además, cuando transporta una estantería, modificamos la geometría utilizada para representar el conjunto, de manera que también se tenga en cuenta el espacio que ocupa la carga.

A diferencia del caso anterior, ya no necesitamos engordar los obstáculos, puesto que comprobamos directamente si la geometría del robot cabe en las zonas libres del mapa.

<br>

### Representación del robot

Para representar la geometría del robot utilizamos la función `puntos_rectangulo()`, que genera una serie de puntos distribuidos por la superficie de un rectángulo.

La separación entre estos puntos es aproximadamente de 0,07 metros. De esta manera, podemos comprobar diferentes partes del robot sin tener que analizar cada punto del mapa. Definimos dos geometrías diferentes:

- Robot vacío: se representa mediante un cuadrado de 0,8 × 0,8 metros.

- Robot cargado: se representa mediante un rectángulo de 2,1 × 0,9 metros, que incluye el espacio ocupado por la estantería transportada.

Los puntos generados se almacenan en `PUNTOS_ROBOT` y `PUNTOS_CARGADO`, respectivamente.

<br>

### Comprobación de colisiones

La función `robot_libre()` comprueba si todos los puntos que representan al robot se encuentran en zonas libres del mapa. Para ello, transforma cada punto de la geometría local del robot a coordenadas globales, teniendo en cuenta su posición (X, Y) y su orientación (yaw).

A continuación, utiliza la función `hay_obstaculo()` para comprobar si cada punto ocupa una zona libre o una zona con obstáculos, por lo que si alguno de los puntos del robot coincide con un obstáculo o queda fuera de los límites del mapa y la posición se considera inválida. De esta forma, podemos detectar posibles colisiones teniendo en cuenta tanto la posición como la orientación del robot.

<br>

### Planificación con OMPL

Para realizar la planificación utilizamos nuevamente el algoritmo RRT*, pero en este caso cambiamos el espacio de estados por `SE2StateSpace()`. Este espacio permite representar cada estado mediante tres componentes:

- X: posición horizontal del robot.

- Y: posición vertical del robot.

- Yaw: orientación del robot.

La función `estado_valido()` obtiene estos tres valores y utiliza `robot_libre()`  para comprobar si la geometría completa del robot puede ocupar esa posición y orientación.

El resto de la configuración es similar al caso anterior donde utilizamos `PathLengthOptimizationObjective` para minimizar la longitud del recorrido y establecemos un tiempo máximo de planificación de 3 segundos. Como resultado, OMPL genera una trayectoria formada por diferentes poses (X, Y, yaw), en lugar de una simple secuencia de puntos.

<br>

### Comprobación del camino

Además de comprobar que las posiciones de la trayectoria sean válidas, realizamos una comprobación adicional del movimiento entre ellas, que es importante porque el controlador utilizado para ejecutar el camino realiza primero un giro sobre sí mismo y después un desplazamiento en línea recta. Para comprobar que estas maniobras son posibles, implementamos tres funciones:

- `giro_libre()`: comprueba si el robot puede girar sobre sí mismo sin colisionar, evaluando diferentes orientaciones con incrementos de 0,05 radianes.

- `recta_libre()`: comprueba si el robot puede desplazarse entre dos posiciones sin atravesar obstáculos, realizando comprobaciones cada 0,05 metros aproximadamente.

- `camino_libre()`: combina las dos comprobaciones anteriores para verificar todos los tramos de la trayectoria.

Si el camino generado por OMPL no supera esta comprobación, se vuelve a ejecutar la planificación hasta encontrar una trayectoria válida.

<br>

### Seguimiento del camino

Una vez validada la trayectoria, utilizamos un sistema de seguimiento similar al del caso anterior. La función `seguir_camino()` recorre las poses calculadas por OMPL. Para alcanzar cada posición, el robot calcula primero el rumbo necesario, gira hasta alcanzarlo y después avanza hacia el siguiente punto.

Las funciones `girar_hasta()` y `avanzar_hasta()` se encargan de controlar las velocidades lineal y angular, utilizando la posición y orientación actuales obtenidas mediante `HAL.getPose3d()`. De esta forma, el robot ejecuta una secuencia de giros y desplazamientos rectos, pero ahora teniendo en cuenta su geometría durante la planificación y la comprobación del recorrido.

<br>

### Recogida y entrega de estanterías

El proceso de recogida y entrega mantiene la misma estructura que en el caso anterior:

1. `Aproximación`: se planifica el desplazamiento hasta la estantería utilizando la geometría del robot vacío.

2. `Recogida`: se utiliza HAL.lift() para levantar la estantería.

3. `Transporte`: se actualiza la geometría del robot para representar el conjunto cargado y se calcula un nuevo camino hasta la zona de destino.

4. `Entrega`: el robot se aproxima al hueco y utiliza HAL.putdown() para depositar la estantería.

5. `Actualización del mapa`: se registra la nueva posición de la estantería y se prepara el entorno para la siguiente operación.

El programa repite este procedimiento para las seis estanterías y muestra finalmente el mapa actualizado del almacén.

<br>





<br>

### Vídeo del funcionamiento 

https://urjc-my.sharepoint.com/:v:/r/personal/j_sanmiguel_2023_alumnos_urjc_es/_layouts/15/stream.aspx?id=%2Fpersonal%2Fj_sanmiguel_2023_alumnos_urjc_es%2FDocuments%2FROBOTICA%2FAmazonWarehouse_CASO2.mp4&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D&share=cQrYrobnEnr3T5Kb6aeunIDiEgUCyzhZnQxM3LJcIGBEefcj6A

<br>
<br>

## CASO_3: Robot Ackermann 

<br>

### Vídeo del funcionamiento 

https://urjc-my.sharepoint.com/:v:/r/personal/j_sanmiguel_2023_alumnos_urjc_es/_layouts/15/stream.aspx?id=%2Fpersonal%2Fj_sanmiguel_2023_alumnos_urjc_es%2FDocuments%2FROBOTICA%2FAmazonWarehouse_CASO3.mp4&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D&share=cQptf-dgHQxcToE1gvfUZ7WqEgUCAnK-X_QIhg3sqwOyGBG8ww

