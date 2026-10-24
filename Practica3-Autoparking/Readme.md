# PRACTICA 3

Página de Enunciado: `https://jderobot.github.io/RoboticsAcademy/exercises/AutonomousCars/autoparking/`

El objetivo de esta práctica es desarrollar un algoritmo de aparcamiento automático, es decir, nuestro vehículo debe detectar los coches aparcados, localizar un hueco con espacio suficiente y realizar la maniobra sin colisionar con los obstáculos.

Para esta solución usaremos el LiDAR 3D para detectar obstáculos y la odometría para estimar la posición y orientación del vehículo.

## Parámetros 

Los valores utilizados en el algoritmo proceden de tres fuentes, en primer lugar, la geometría del vehículo, en segundo, la configuración del sensor en el modelo del simulador, y para finalizar, los parámetros de funcionamiento ajustados para la solución.

Antes de programar las maniobras, es necesario conocer las dimensiones del vehículo y la posición del sensor LiDAR. Estos datos permiten interpretar correctamente las medidas del sensor y calcular las distancias respecto a los límites del coche.

<br>

### Geometría del vehículo

A partir del modelo 3D del vehículo se establecen los siguientes parámetros:

- `X_DELANTE = 2.19`: posición del extremo delantero respecto al centro del vehículo.

- `X_DETRAS = -2.43`: posición del extremo trasero.

- `MEDIO_ANCHO = 1.0`: mitad de la anchura del vehículo.

- `X_EJE = -1.52`: posición del eje trasero, utilizada como referencia para calcular el movimiento.

De esta forma, La longitud total del vehículo se obtiene mediante la diferencia entre sus extremos: `LONGITUD = X_DELANTE - X_DETRAS = 4.62 m`

Finalmente, usamos estos valores para determinar si un obstáculo está delante, detrás o a un lado del vehículo.

<br>

### Configuración del LiDAR

La posición y orientación del sensor se obtienen de su configuración en el modelo del simulador:

- `X_LIDAR = -1.25`: desplazamiento longitudinal.

- `Z_LIDAR = 1.725`: altura del sensor.

- `YAW_LIDAR = -1.578`: orientación del sensor en el plano horizontal.

También se establece un intervalo de altura entre `ALTURA_MIN = 0.3` y `ALTURA_MAX = 1.4` para seleccionar los puntos que resultan útiles para detectar obstáculos.

<br>

Los parámetros de geometría proceden del modelo del vehículo y de la configuración del sensor donde ha sido necesario la ayuda de foros e incluso de la IA para determinar con exactitud dichas medidas. 

Por otro lado, las velocidades y los márgenes de seguridad han sido ajustados y comprobados experimentalmente en la simulación.

## Funcionamiento 

El algoritmo combina los datos del LiDAR con la odometría para detectar un hueco adecuado y ejecutar el aparcamiento de forma autónoma. El funcionamiento se divide en los siguientes pasos:

`1. Inicialización`: esperamos a que el simulador nos proporcione una pose válida, luego, se guarda la posición inicial del vehículo y se calculan las referencias de la fila de aparcamiento a partir de los datos del LiDAR.

`2. Percepción del entorno`: se procesan los puntos del LiDAR, se transforman al sistema de referencia del vehículo y se descartan los puntos que corresponden a la propia geometría del coche.

`3. Estimación del movimiento`: se utiliza la odometría para calcular el avance, el desplazamiento lateral y la orientación del vehículo respecto a su posición inicial.

`4. Detección del hueco`: se analizan los obstáculos y las distancias libres para comprobar si existe un espacio suficiente y si el vehículo está correctamente alineado.

`5. Ejecución de las maniobras`: una máquina de estados determina el movimiento que debe realizar el vehículo en cada momento y las condiciones para pasar a la siguiente fase.

`6. Finalización`: cuando el coche está suficientemente centrado en el hueco, se detiene por completo.



El proceso se repite continuamente a una frecuencia objetivo de 20 Hz, actualizando la percepción y las decisiones de movimiento en cada iteración.

### Detección del hueco de aparcamiento

### Maquina de Estados 


## Video 

https://github.com/user-attachments/assets/68e7c766-124a-46fd-9430-fbeb1348cc76

<img width="838" height="220" alt="Captura desde 2026-10-24 11-26-03" src="https://github.com/user-attachments/assets/88ac4b29-ea24-4e15-adf0-bac010ca56b0" />
