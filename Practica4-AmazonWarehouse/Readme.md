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


### Vídeo del funcionamiento 

https://urjc-my.sharepoint.com/:v:/r/personal/j_sanmiguel_2023_alumnos_urjc_es/_layouts/15/stream.aspx?id=%2Fpersonal%2Fj_sanmiguel_2023_alumnos_urjc_es%2FDocuments%2FROBOTICA%2FAmazonWarehouse_CASO2.mp4&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D&share=cQrYrobnEnr3T5Kb6aeunIDiEgUCyzhZnQxM3LJcIGBEefcj6A

<br>
<br>

## CASO_3: Robot Ackermann 

### Planteamiento

En este último caso trabajamos con un robot de tipo Ackermann, cuyo comportamiento es diferente al de los robots holonómicos utilizados anteriormente.

Este vehículo no puede girar sobre sí mismo, ya que necesita avanzar o retroceder describiendo curvas para cambiar de dirección. Por tanto, debemos tener en cuenta sus restricciones de movimiento durante la planificación. Para ello, representamos el coche mediante una geometría rectangular y utilizamos el espacio de estados ReedsSheppStateSpace de OMPL, que permite generar trayectorias compatibles con movimientos hacia delante y hacia atrás. Además, modificamos el sistema de seguimiento para que el vehículo pueda recorrer las curvas de forma progresiva.


<br>

###  Representación del robot

Al igual que en el Caso 2, utilizamos una serie de puntos distribuidos por la superficie de un rectángulo para representar la geometría del vehículo.

La función `puntos_rectangulo()` genera estos puntos con una separación aproximada de 0,07 metros. Así mismo definimos dos geometrías:

- Coche vacío: rectángulo de 1,3 × 0,75 metros.

- Coche cargado: rectángulo de 2,1 × 0,9 metros, que representa el conjunto formado por el vehículo y la estantería.

También tenemos en cuenta que la posición proporcionada por el simulador no coincide exactamente con el centro geométrico que utilizamos para representar el coche. Para corregirlo, definimos `MEDIO_COCHE = 0.25` y utilizamos la función `pose_coche()`, que desplaza la posición obtenida mediante HAL.getPose3d() para trabajar con el centro de referencia del vehículo, de esta forma, podemos comprobar correctamente la ocupación del espacio teniendo en cuenta la orientación del coche.

<br>


### Comprobación de colisiones

La función `estado_valido()` comprueba si la geometría completa del vehículo puede ocupar una determinada posición y orientación. Para ello, transforma todos los puntos del rectángulo a coordenadas globales utilizando la posición (X, Y) y el ángulo yaw. Posteriormente, comprueba cada punto mediante `hay_obstaculo()` de forma que si alguno coincide con un obstáculo o queda fuera de los límites del mapa, el estado se considera inválido.

Este procedimiento se aplica tanto cuando el coche circula vacío como cuando transporta una estantería, utilizando en cada situación la geometría correspondiente.

<br>


### Planificación con Reeds-Shepp

La principal diferencia respecto a los casos anteriores aparece en el espacio de estados utilizado por OMPL.

En lugar de utilizar `RealVectorStateSpace` o `SE2StateSpace`, empleamos `ReedsSheppStateSpace(RADIO_GIRO)`. Este espacio permite conectar dos poses mediante trayectorias formadas por arcos y segmentos rectos, teniendo en cuenta un radio mínimo de giro. Además, admite desplazamientos tanto hacia delante como hacia atrás.

Utilizamos un radio de giro de 1,2 metros y mantenemos el algoritmo RRT* para buscar una trayectoria entre la posición inicial y la de destino. También utilizamos `PathLengthOptimizationObjective` para minimizar la longitud del recorrido y establecemos un tiempo máximo de planificación de 3 segundos.

Una vez encontrada una solución exacta, interpolamos el camino en 200 estados para obtener una secuencia más detallada de poses que facilite su seguimiento.

<br>


### Seguimiento de la trayectoria

A diferencia de los casos anteriores, el coche no puede ejecutar cada tramo mediante un giro sobre sí mismo seguido de un desplazamiento recto. Por ello, implementamos un controlador basado en Pure Pursuit, que permite seguir una trayectoria curva calculando continuamente la dirección necesaria para alcanzar un punto situado por delante del vehículo.

Primero, la función `partir_en_tramos()` divide la trayectoria en segmentos según el sentido de desplazamiento. De esta manera, identifica cuándo el coche debe avanzar y cuándo debe retroceder. Después, la función `seguir_tramo()` se encarga de seguir cada segmento:

1. Selecciona un punto de la trayectoria situado a una distancia determinada del vehículo.

2. Calcula el ángulo necesario para dirigirse hacia ese punto.

3. Obtiene la curvatura necesaria para corregir la trayectoria.

4. Calcula las velocidades lineal y angular que debe aplicar el vehículo.

Para ello, utilizamos una distancia objetivo de 0,6 metros y limitamos la velocidad lineal máxima a 0,15 m/s. La velocidad y la curvatura se combinan para calcular los valores enviados a HAL.setV() y HAL.setW() y cuando el tramo requiere retroceder, se invierte el sentido de ambas velocidades. De esta forma, el vehículo puede seguir trayectorias curvas y realizar maniobras hacia delante y hacia atrás sin necesidad de detenerse para girar sobre sí mismo.

<br>


### Aproximación a las estanterías

Para mejorar las maniobras de recogida y entrega, no realizamos toda la aproximación mediante el planificador global. En su lugar, calculamos puntos específicos de entrada y salida alrededor de cada estantería.

Utilizamos una distancia de entrada de 3 metros y una pequeña distancia adicional de 1 metro, denominada `CARRERILLA`, para preparar el vehículo antes de realizar la aproximación final. Además la función `recta_hasta()` genera una trayectoria recta interpolada entre la posición actual y el punto objetivo, que posteriormente se ejecuta utilizando el mismo controlador de seguimiento.

Así, el proceso de aproximación se divide en dos partes:

- Planificación global: OMPL calcula una trayectoria hasta un punto situado a cierta distancia de la estantería.

- Aproximación final: el vehículo realiza desplazamientos rectos para alinearse con la estantería y recogerla o depositarla.

<br>


### Recogida y entrega de estanterías

El programa principal calcula para cada estantería los puntos necesarios para realizar la recogida y la entrega. Para cada una de las seis estanterías se realizan las siguientes operaciones:

1. `Preparación de la recogida`: se calcula la posición de aproximación y el punto de entrada situado a 3 metros de la estantería.

2. `Aproximación`: OMPL planifica el recorrido hasta el punto de entrada y el vehículo realiza los desplazamientos rectos necesarios para alcanzar la estantería.

3. `Recogida`: el coche se detiene y utiliza HAL.lift() para levantar la estantería.

4. `Transporte`: se actualiza la geometría del vehículo para representar el conjunto cargado y OMPL calcula una nueva trayectoria hasta el punto situado delante del hueco de destino.

5. `Entrega`: el vehículo realiza la aproximación final y utiliza HAL.putdown() para depositar la estantería.

6. `Actualización y salida`: se actualiza la posición de la estantería en el mapa y el coche retrocede hasta el punto de salida para continuar con la siguiente operación.

Una vez trasladadas las seis estanterías, se muestra el mapa final del almacén.

<br>



### Vídeo del funcionamiento 

https://urjc-my.sharepoint.com/:v:/r/personal/j_sanmiguel_2023_alumnos_urjc_es/_layouts/15/stream.aspx?id=%2Fpersonal%2Fj_sanmiguel_2023_alumnos_urjc_es%2FDocuments%2FROBOTICA%2FAmazonWarehouse_CASO3.mp4&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D&share=cQptf-dgHQxcToE1gvfUZ7WqEgUCAnK-X_QIhg3sqwOyGBG8ww

