# PRACTICA 3

Página de Enunciado: `https://jderobot.github.io/RoboticsAcademy/exercises/AutonomousCars/autoparking/`

El objetivo de esta práctica es desarrollar un algoritmo de aparcamiento automático, es decir, nuestro vehículo debe detectar los coches aparcados, localizar un hueco con espacio suficiente y realizar la maniobra sin colisionar con los obstáculos.

Para esta solución usaremos el LiDAR 3D para detectar obstáculos y la odometría para estimar la posición y orientación del vehículo.

<br>
<br>

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

<br>
<br>

## Funcionamiento 

El algoritmo combina la información del LiDAR con la odometría para detectar un hueco adecuado y realizar las maniobras necesarias para aparcar de forma autónoma. El LiDAR permite conocer la posición de los obstáculos, mientras que la odometría permite saber cómo se desplaza y gira el vehículo durante el proceso. 

El funcionamiento se divide en los siguientes pasos:


`1. Inicialización`: al comenzar, esperamos a que el simulador proporcione los datos de posición del vehículo. Después, se guarda su posición inicial y se utilizan las primeras lecturas del LiDAR para establecer las referencias de la fila de aparcamiento.

`2. Percepción del entorno`: recogemos los puntos detectados por el sensor y los filtramos para conservar los que resultan útiles. A continuación, se transforman al sistema de referencia del vehículo y se eliminan los puntos que pertenecen al propio coche. Así, el algoritmo puede identificar los obstáculos y calcular las distancias respecto al vehículo.

`3. Estimación del movimiento`: mediante la odometría, calculamos cuánto ha avanzado el coche, cuánto se ha desplazado lateralmente y cuánto ha girado respecto a su posición inicial. Esta información permite conocer su posición durante la búsqueda del hueco y las distintas maniobras.

`4. Detección del hueco`: mientras el vehículo avanza, analizamos los obstáculos y las distancias libres para comprobar si existe un espacio suficientemente grande para aparcar. También comprobamos que el coche esté correctamente alineado con la fila antes de comenzar la maniobra.

`5. Ejecución de las maniobras`:  cuando se encuentra un hueco válido, entra en funcionamiento una máquina de estados que controla las distintas fases del aparcamiento. Según la posición del vehículo y el espacio disponible, el algoritmo decide cuándo debe retroceder, avanzar, girar o corregir su posición.

`6. Finalización`: una vez que el vehículo ha entrado en el hueco, comprobamos la distancia disponible delante y detrás para ajustar su posición. Cuando se considera que está suficientemente centrado, se detiene completamente.

Todo este proceso se ejecuta de forma continua mediante un bucle de control con una frecuencia objetivo de 20 Hz. En cada ciclo se actualizan los datos del LiDAR, se calcula el movimiento del vehículo y se decide la siguiente acción, permitiendo adaptar las maniobras a las condiciones detectadas durante el aparcamiento.

<br>

### Maquina de Estados 

El aparcamiento se organiza mediante seis estados. Cada uno representa una fase de la maniobra y define las condiciones necesarias para cambiar al siguiente.

| Estado | Funcionamiento y movimiento | Condiciones de transición |
|---|---|---|
| `BUSCAR_HUECO` | El vehículo avanza a velocidad `V = 1.0` y corrige su orientación para mantenerse alineado con la fila de aparcamiento. Analiza los puntos del LiDAR para localizar un espacio libre con longitud suficiente. | Pasa a `ATRAS_DERECHA` cuando el vehículo está alineado con la fila (`abs(linea - lateral) < 0.2`), detecta un coche que delimita el hueco y comprueba que el espacio disponible tiene la longitud necesaria. |
| `ATRAS_DERECHA` | El vehículo retrocede a velocidad `V_LENTA = 0.5` mientras gira a la derecha con velocidad angular `GIRO = 1.0`. Esta maniobra permite introducir la parte trasera del coche en el hueco. | Pasa a `ATRAS_IZQUIERDA` cuando alcanza la posición lateral prevista (`lateral < mitad`) o cuando la distancia libre detrás es inferior al margen de seguridad (`detras < MARGEN`). |
| `ATRAS_IZQUIERDA` | El vehículo continúa retrocediendo, pero gira a la izquierda para modificar su orientación y avanzar en la entrada al hueco. | Pasa a `CENTRAR` cuando el ángulo relativo es inferior a un grado (`angulo < math.radians(1)`). Si no se cumple esta condición, pasa a `ADELANTE_DERECHA` cuando la distancia libre detrás es inferior al margen (`detras < MARGEN`) o el vehículo sobrepasa la referencia lateral prevista (`lateral < fondo - MARGEN / 2`). |
| `ADELANTE_DERECHA` | El vehículo avanza a velocidad reducida (`V_LENTA = 0.5`) mientras gira a la derecha. Se utiliza para corregir la posición y la orientación cuando no puede continuar retrocediendo adecuadamente. | Pasa a `CENTRAR` cuando el ángulo relativo es inferior a un grado (`angulo < math.radians(1)`). Si no se cumple esta condición y la distancia libre delante es inferior al margen de seguridad (`delante < MARGEN`), vuelve a `ATRAS_IZQUIERDA`. |
| `CENTRAR` | El vehículo compara las distancias libres delante (`delante`) y detrás (`detras`). Si necesita corregir su posición, realiza pequeños desplazamientos hacia delante o hacia atrás a una velocidad lineal de `0.3`, utilizando el ángulo para corregir su orientación. | Pasa a `APARCADO` cuando la diferencia entre las distancias libres delante y detrás es inferior a `0.5 m` (`abs(delante - detras) < 0.5`). Si no se cumple, avanza cuando hay más espacio libre delante y retrocede en caso contrario. |
| `APARCADO` | El vehículo se encuentra en la fase final y deja de moverse. | No tiene transiciones de salida. El programa establece las velocidades lineal y angular a cero mediante `mover(0, 0)`. |

Los cambios de estado se comprueban continuamente durante la ejecución del programa. De esta forma, el vehículo adapta sus maniobras a su posición, orientación y distancia a los obstáculos.

<br>
<br>

## Métodos y Funciones 


### Procesamiento del LiDAR

- `leer_puntos()` → Filtra los datos del LiDAR y transforma los puntos al sistema de referencia del vehículo.


### Odometría y transformaciones

- `eje_trasero()` → Calcula la posición del eje trasero a partir de la pose del vehículo proporcionada por el simulador. Este punto se utiliza como referencia para analizar el movimiento.

- `leer_pose()` → Calcula el avance longitudinal, el desplazamiento lateral y el cambio de orientación respecto a la pose inicial. Estas magnitudes permiten evaluar la evolución de las maniobras.

- `a_la_calle(x, y, pose)` → Transforma un punto al sistema de referencia de la calle, permitiendo analizar la posición de los obstáculos durante la búsqueda del hueco.

### Cálculo de distancias

- `hay_algo(valores, desde, hasta)` → Comprueba si existen valores dentro de un intervalo determinado.

- `libre_delante(puntos)` → Calcula la distancia al obstáculo más cercano situado delante del vehículo.

- `libre_detras(puntos)` → Calcula la distancia al obstáculo más cercano situado detrás.

- `libre_derecha(puntos)` → Calcula la distancia libre en el lateral derecho.

### Control del movimiento

- `mover(v, w)` → Establece las velocidades lineal y angular del vehículo mediante HAL.setV() y HAL.setW().

- `maniobrar(v, w)` → Ejecuta las maniobras de aparcamiento y aplica un pequeño desplazamiento inicial durante un tiempo limitado para facilitar el comienzo del movimiento.

- `cambiar(nuevo)` → Actualiza el estado actual, registra el instante del cambio y muestra por pantalla la fase en la que se encuentra el algoritmo.

## Video 

https://github.com/user-attachments/assets/68e7c766-124a-46fd-9430-fbeb1348cc76

<img width="838" height="220" alt="Captura desde 2026-10-24 11-26-03" src="https://github.com/user-attachments/assets/88ac4b29-ea24-4e15-adf0-bac010ca56b0" />
