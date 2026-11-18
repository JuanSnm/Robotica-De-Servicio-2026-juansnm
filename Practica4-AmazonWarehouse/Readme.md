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



## CASO_2: 




## CASO_3: Robot Ackermann 
