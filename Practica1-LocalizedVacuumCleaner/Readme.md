# PRACTICA 1

Página de Enunciado: `https://jderobot.github.io/RoboticsAcademy/exercises/MobileRobots/vacuum_cleaner_loc`

Vamos a explicar y entender el funcionamiento de Localized Vacuum Cleaner, donde nuestro objetivo es implementar la lógica de navegación de un robot aspirador haciendo uso de la información de localización del robot dentro de un mapa conocido. El objetivo principal del algoritmo utilizado (Algoritmo BSA, en este caso) es cubrir la mayor superficie posible, tratando de recorrer de forma sistemática las zonas transitables del mapa y reduciendo al mínimo las áreas que quedan sin visitar.

## Funcionamiento 

El funcionamiento del sistema se divide en tres partes principales, donde encontramos el registro del mapa, la planificación y el movimiento. Primero, se relaciona la posición del robot con el mapa para saber en todo momento dónde se encuentra. Después, se realiza una planificación en estático, es decir, antes de que el robot comience a moverse, utilizando el algoritmo BSA y generando una ruta que permita cubrir la mayor parte del entorno. Finalmente, el robot ejecuta la ruta calculada, desplazándose por las distintas zonas y teniendo en cuenta los obstáculos y puntos de retorno.


<br>
<br>
<br>




### 1- Registro del mapa _________________________________________________________________________________________________________________________

Empezaremos con el registro del mapa. En esta primera parte necesitamos relacionar las coordenadas del mundo, utilizadas por el robot en Gazebo, con las coordenadas en píxeles de la imagen del mapa.

Esto es necesario porque el robot trabaja con posiciones en metros, mientras que el mapa es una imagen y utiliza píxeles, de forma que, debemos establecer una transformación entre ambos sistemas de coordenadas.

Para realizar esta transformación utilizamos tres valores principales:
   - `ESCALA = 100.4` → indica que un metro en el mundo corresponde aproximadamente a 100.4 píxeles en el mapa.
   - `U0 = 579.34` → posición horizontal de referencia del origen.
   - `V0 = 421.54` - → posición vertical de referencia del origen.


Estos valores los obtuvimos mediante una calibración experimental, donde hemos relacionado posiciones conocidas del robot en Gazebo con sus correspondientes posiciones en píxeles dentro del mapa. 

    Por ejemplo:
    
    Si tenemos el robot en la posición (x, y) = (1, 0.5) m, su posición aproximada en el mapa sería:
    
    - u = 579.34 − 100.4 · 1 = 478.94 píxeles
    - v = 421.54 + 100.4 · 0.5 = 471.74 píxeles
    
    De forma que, la posición (1, 0.5) m del mundo corresponde aproximadamente al píxel (478.94, 471.74) del mapa.
    

De esta forma, podemos saber dónde se encuentra el robot sobre la imagen y, posteriormente, transformar las posiciones de las celdas del mapa a coordenadas reales para que el robot pueda desplazarse hasta ellas.

Además, la transformación también se podrá realizar en sentido contrario, que será necesario posteriormente para convertir los centros de las celdas de la cuadrícula en posiciones reales a las que el robot pueda desplazarse.




<br>
<br>
<br>



### 2- Algoritmo BSA _________________________________________________________________________________________________________________________

#### Creacíón cuadricula

Con el mapa registrado podemos transformarlo en una cuadrícula de celdas de 30 × 30 píxeles.

Para saber si nuestro robot puede pasar por una celda, utilizamos una transformada de distancia, que nos indica qué distancia hay desde cada punto libre hasta el obstáculo más cercano. De esta forma, podemos comprobar que el centro de cada celda tenga suficiente espacio para el robot. La distancia mínima que exigimos viene dada por:

`HOLGURA = (0.178 + 0.06) · ESCALA`


<br>

#### Planificación mediante el algoritmo

Con el mapa ya registrado y la cuadricula creada, realizamos la planificación de la ruta antes de que el robot comience a moverse. De esta forma, usaremos el Backtracking Spiral Algorithm (BSA).

Para el algoritmo, una celda puede considerarse un obstáculo por dos motivos:
- Es un obstáculo real, como una pared.
- Es una celda que ya ha sido visitada, que pasa a actuar como un obstáculo virtual para evitar repetir zonas.

De esta forma, el algoritmo va construyendo una ruta que permite cubrir progresivamente el espacio disponible. 

Durante la planificación se van guardando tres elementos principales, primeramente `visto` que indica qué celdas ya han sido visitadas, después `plan` que almacena el orden de las celdas que debe recorrer el robot y por ultimo `bps` que guarda los puntos de retorno, es decir, celdas desde las que todavía existe alguna zona sin visitar.

Cuando desde una celda existen varias opciones para continuar, el algoritmo elige una de ellas y guarda las demás como puntos de retorno. Cuando ya no puede continuar por la zona actual, busca uno de estos puntos y calcula un camino hasta él para continuar. Y de esta forma, el BSA genera toda la ruta antes de que comience el movimiento del robot, intentando recorrer las zonas libres sin volver a cubrir las celdas que ya han sido visitadas.



<br>

#### Reglas de movimiento 

Para decidir qué hacer en cada celda, nuestro BSA utiliza cuatro reglas principales:

1. `RS1 – Punto crítico`: Si las cuatro direcciones están bloqueadas, significa que el robot no puede continuar desde dicha posición, entonces, la celda se marca como punto crítico y se termina la espiral.

2. `RS2 – Lado de referencia`: Si la celda situada a la izquierda del robot está libre, el robot gira hacia la izquierda y avanza hacia ella.

3. `RS3 – Obstáculo frontal`: Si la celda que tenemos delante está bloqueada, el robot cambia de dirección para poder continuar recorriendo el entorno.

4. `RS4 – Avance`: Si ninguna de las situaciones anteriores ocurre, el robot continúa avanzando en la misma dirección.


<br>

#### Vídeo 

[Grabación de pantalla desde 2026-10-02 14-23-23.webm](https://github.com/user-attachments/assets/610013b8-1f9f-4a49-baa9-463369189449)


<br>
<br>
<br>



### 3 - Movimiento _________________________________________________________________________________________________________________________

Una vez obtenida la ruta mediante el BSA, el robot comienza a ejecutarla desde su posición inicial.

Primero se desplaza hasta el centro de la primera celda de la ruta, y después, se recorren las celdas de plan siguiendo el orden establecido durante la planificación.

Para hacer el movimiento más fluido, se agrupan las celdas consecutivas que están en la misma dirección. De esta forma, en lugar de realizar un movimiento independiente para cada celda, el robot realiza un único movimiento recto pasando por varias celdas, además, para cada tramo se calcula la dirección hacia el siguiente punto mediante: `yaw = atan2(dy, dx)`.

Durante el movimiento se controla la distancia que queda hasta el objetivo y la desviación respecto a la trayectoria y con estos valores se ajustan la velocidad lineal y angular del robot para conseguir que llegue correctamente al siguiente punto. Además, a medida que el robot pasa por las diferentes celdas, estas se van marcando como visitadas en el mapa.


Finalmente, de esta forma, el movimiento consiste simplemente en ejecutar la ruta calculada previamente por el BSA, agrupando los desplazamientos para que el recorrido sea más continuo.


## Métodos y Funciones 

`1. Registro del mapa`
- mundo_a_pixel(x, y) → convierte coordenadas del mundo a píxeles.
- pixel_a_mundo(u, v) → convierte píxeles a coordenadas del mundo.


`2. Localización y HAL`
- **pose() → obtiene la posición del robot y compensa el retraso.**

         Las fórmulas salen de la forma en la que se mueve el robot: avanza en la dirección en la que está orientado y puede girar.
  
         - x += v · cos(θ) · dt → calcula cuánto avanza en X.
         - y += v · sin(θ) · dt → calcula cuánto avanza en Y.
         - θ += w · dt → calcula cuánto cambia su orientación.
  
         Se utiliza cos(θ) y sin(θ) porque el movimiento del robot depende de hacia dónde esté mirando. Estas fórmulas nos permiten estimar su posición durante el pequeño retraso que tenemos al recibir la posición real.

- mandar(v, w) → manda velocidad lineal y angular al robot.
- esperar(tiempo) → mantiene el robot parado durante un tiempo.


`3. Movimiento`
- ang(a) → normaliza un ángulo entre -π y π.
- girar(yaw) → gira el robot hasta alcanzar una orientación.
- **recta(a, b, celdas) → mueve el robot entre dos puntos, corrigiendo la trayectoria y marcando las celdas recorridas.**

      Las fórmulas se obtienen directamente de la posición actual del robot y la posición de la celda a la que queremos llegar.
  
      - yaw = atan2(dy, dx) → sale de la geometría entre los dos puntos y nos indica el ángulo hacia el objetivo.
      - ux = dx / distancia y uy = dy / distancia → convierten la dirección hacia el objetivo en una dirección de tamaño 1.
      - resto → indica la distancia que todavía queda hasta el objetivo.
      - lateral → indica cuánto se ha separado el robot de la trayectoria que debería seguir.
  
      Después utilizamos estos valores para decidir cuánto avanzar y cuánto girar.


`4. Creación de la cuadrícula`
- **cuadricula(ox, oy) → crea una cuadrícula y determina qué celdas son transitables.**

      La cuadrícula se crea dividiendo el mapa en celdas de 30 píxeles.
      El valor de 30 pixeles se ha elegido para tener celdas suficientemente pequeñas para cubrir bien el mapa sin crear demasiadas.
  
      Para comprobar si una celda es segura utilizamos la distancia al obstáculo más cercano y esta holgura se calcula como:
      HOLGURA = (0.178 + 0.06) · ESCALA
  
      Los 0.178 m corresponden a una medida utilizada para tener en cuenta el tamaño del robot y 0.06 m es un margen adicional de seguridad.

- centro(c) → obtiene las coordenadas del centro de una celda.
- vecina(c, d) → obtiene la celda vecina en una dirección determinada (Usada constantemente por BFS y BSA)


`5. Búsqueda de caminos`
- **bfs(origen, destino, pisables) → encuentra un camino entre dos celdas o todas las celdas alcanzables.**


`6. Visualización`
- pintar(c, color) → cambia el color de una celda.
- mostrar(x, y) → muestra el mapa y la posición del robot.
- limpiar(c, x, y) → marca una celda como recorrida.


`7. Planificación BSA`
- bsa(inicio, rumbo) → genera todo el recorrido utilizando el algoritmo BSA.

## Video

https://urjc-my.sharepoint.com/:v:/g/personal/j_sanmiguel_2023_alumnos_urjc_es/IQC4pQM4dtBTTLgfJSPv5mC1AWEd18om2lZkSQoYeBP-JQo?e=1lzvUT&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D
