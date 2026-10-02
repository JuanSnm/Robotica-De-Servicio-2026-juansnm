# PRACITCA 1

Página de Enunciado: `https://jderobot.github.io/RoboticsAcademy/exercises/MobileRobots/vacuum_cleaner_loc`

Vamos a explicar y entender el funcionamiento de Localiced Vacuum Cleaner, donde nuestro objetivo es implementar la lógica de navegación de un robot aspirador haciendo uso de la información de localización del robot dentro de un mapa conocido. El objetivo principal del algoritmo utilizado (Algoritmo BSA, en este caso) es cubrir la mayor superficie posible, tratando de recorrer de forma sistemática las zonas transitables del mapa y reduciendo al mínimo las áreas que quedan sin visitar.

## Funcionamiento 

El funcionamiento del sistema se divide en tres partes principales, donde encontramos el registro del mapa, la planificación y el movimiento. Primero, se relaciona la posición del robot con el mapa para saber en todo momento dónde se encuentra. Después, se realiza una planificación en estático, es decir, antes de que el robot comience a moverse, utilizando el algoritmo BSA y generando una ruta que permita cubrir la mayor parte del entorno. Finalmente, el robot ejecuta la ruta calculada, desplazándose por las distintas zonas y teniendo en cuenta los obstáculos y puntos de retorno.


### 1- Registro del mapa

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


### 2- Algoritmo BSA 

#### Vídeo 

[Grabación de pantalla desde 2026-10-02 14-23-23.webm](https://github.com/user-attachments/assets/610013b8-1f9f-4a49-baa9-463369189449)


### 3 - Movimiento 



## Video
