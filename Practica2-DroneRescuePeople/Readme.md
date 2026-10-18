# PRACTICA 2 

Página de Enunciado: https://jderobot.github.io/RoboticsAcademy/exercises/Drones/rescue_people 

El objetivo de esta práctica es desarrollar un programa de control para un dron capaz de buscar supervivientes en una zona determinada, detectar sus caras mediante visión artificial y registrar sus posiciones para facilitar su posterior rescate.

En nuestro caso el programa sigue una estrategia de búsqueda en espiral cuadrada que permite recorrer progresivamente la zona de interés. Durante el vuelo, el dron análiza las imágenes de la cámara y estima las coordenadas de las personas detectadas.

<br>

## Configuración Inicial 

Antes de comenzar la misión, definimos las coordenadas de referencia de la zona de búsqueda, los parámetros de vuelo y la configuración de la cámara y del detector de caras. Estos valores permiten establecer el área que recorrerá el dron, adaptar el procesamiento de las imágenes y controlar los recursos disponibles durante la búsqueda.

<br>

### Coordenadas de referencia

El enunciado nos proporciona las coordenadas GPS del barco y una ubicación aproximada de los supervivientes. Como el simulador trabaja con posiciones en metros, convertimos esas coordenadas a UTM y cálculamos la posición relativa respecto al barco.

- Barco de seguridad: origen local `(0, 0)`.

- Centro de búsqueda: aproximadamente `(40.3, -30.3)` metros respecto al barco.

En esta implementación usamos el eje X para el desplazamiento hacia el este y el eje Y para el desplazamiento hacia el norte, por lo que los valores negativos de Y representan posiciones hacia el sur.

### Configuración de la cámara y del detector 

Para detectar las caras usamos OpenCV con el clasificador Haar Cascade proporcionado por el ejercicio.

- Imagen: `320 × 240 píxeles`.

- Campo de visión horizontal: `59°`.

- Tamaño de las caras aceptadas: entre `20 - 55` píxeles de ancho.

- Altura de vuelo: `4 metros`.

La posición de cada persona la estimamos usando su posición en la imagen, la altura del dron y los parámetros de la cámara. Vamos a ver ahora paso a paso cómo lo calculamos:

1. Calculamos la distancia focal a partir del campo de visión horizontal: `f = (320 / 2) / tan(59° / 2) `

2. Estimamos los desplazamientos respecto al dron mediante la geometría de la cámara: `Δx = ((120 - py) × z) / f` y `Δy = ((160 - px) × z) / f`

3. Calculamos la posición estimada de la persona sumando los desplazamientos a la posición actual del dron: `X_persona = X_dron + Δx` y `Y_persona = Y_dron + Δy`

Donde `px` y `py` son las coordenadas del centro de la cara detectada en la imagen, `z` es la altura del dron, `f` es la distancia focal calculada y `320 × 240` píxeles son las dimensiones de la imagen. Finalmente, de esta forma, obtnemos una aproximación de la posición de la persona en metros.

### Parámetros de la misión

Los principales parámetros utilizados son:

- Altura de vuelo: 4 m.

- Número de tramos de búsqueda: 28, que generan 29 puntos en la trayectoria.

- Separación inicial entre tramos: 2,5 m.

- Batería máxima simulada: 400 m de recorrido.

- Margen de seguridad de batería: 15 m.

- Distancia para considerar una detección repetida: 2 m.

- Tamaño de las caras aceptadas: entre 20 y 55 píxeles de ancho.

Estos parámetros permiten configurar la trayectoria, la detección de personas y la gestión de la batería.


<br>
<br>

## Funcionamiento

En este apartado explicamos cómo funciona el programa durante la misión, desde el desplazamiento inicial hasta la finalización de la búsqueda.

<br>

### Movimiento - Búsqueda 

El dron utiliza una trayectoria en espiral cuadrada para explorar la zona alrededor del centro estimado de los supervivientes. La trayectoria se genera automáticamente y se recorre punto por punto.

Para desplazarse, el programa envía referencias de posición mediante la API HAL. El dron mantiene una altura objetivo de 4 m y se considera que ha alcanzado un punto cuando está a menos de 0,5 m de él.

Durante los desplazamientos de búsqueda, se obtienen imágenes de la cámara ventral para localizar posibles supervivientes.

### Detección de supervivientes 

Las imágenes se procesan mediante OpenCV y el clasificador Haar Cascade, y para mejorar las posibilidades de detección, se analizan ocho orientaciones diferentes, con rotaciones de 45°.

Cuando se detecta una cara, el programa estima la posición de la persona utilizando su ubicación en la imagen, la altura del dron y los parámetros de la cámara.

Finalmente, las coordenadas obtenidas se comparan con las posiciones almacenadas y si una detección se encuentra a menos de 2 m de una persona registrada, se considera repetida, pero en caso contrario, se guarda como una nueva persona.

### Gestión de la batería y recarga 

La batería se simula en función de la distancia recorrida por el dron. Antes de dirigirse al siguiente punto, el programa comprueba si dispone de batería suficiente para llegar y regresar al barco, incluyendo un margen de seguridad.

Si no es suficiente, el dron guarda su posición actual y regresa al barco. Allí aterriza y espera tres segundos para simular la recarga. Después, restablece la batería, despega y vuelve a la posición guardada para continuar la búsqueda.

Esta gestión permite interrumpir temporalmente la misión y reanudarla tras la recarga.

<br>
<br>

#### Desarrollo completo de la misión


1. Inicialización: se configuran las coordenadas, los parámetros de vuelo, la batería y el detector de caras.

2. Planificación: se genera la lista de puntos que forman la espiral cuadrada.

3. Despegue: el dron despega hasta la altura objetivo de 4 metros.

4. Desplazamiento inicial: vuela hasta el centro de la zona de búsqueda sin realizar detecciones.

5. Búsqueda: recorre secuencialmente los puntos de la espiral.

6. Procesamiento de imágenes: durante los desplazamientos de búsqueda, analiza las imágenes ventrales en ocho orientaciones y estima las posiciones de las caras detectadas.

7. Registro: guarda las posiciones de los nuevos supervivientes y evita añadir detecciones cercanas a personas ya registradas.

8. Control de batería: antes de dirigirse a cada punto, comprueba si la batería estimada permite llegar y regresar al barco con la reserva establecida.

9. Recarga y reanudación: cuando es necesario, regresa al barco, aterriza, recarga y vuelve a la posición en la que se interrumpió la misión.

10. Finalización: al completar la espiral, regresa al barco, aterriza y muestra por terminal el número de personas encontradas y las coordenadas estimadas de cada una.





<br>
<br>

## Métodos y Funciones 

`distancia(a, b)` → Calcula la distancia euclídea entre dos posiciones

<br>


`crear_espiral(cx, cy)` → Genera los puntos de la trayectoria de búsqueda
        
        Genera una trayectoria en espiral cuadrada alrededor del centro de búsqueda, de modo que el dron 
        alterna las direcciones este, norte, oeste y sur, aumentando la longitud de los tramos cada dos 
        movimientos. De esta forma, ampliamos progresivamente la zona explorada.

        Como resultado obtenemos 29 movimientos a realizar. 

<br>


`buscar_caras(imagen)` → Detecta caras en ocho orientaciones y devuelve sus centros en píxeles

      Utiliza OpenCV y el clasificador Haar Cascade para detectar caras en las imágenes de la cámara. 
      
      Para mejorar las posibilidades de detección, giramos la imagen ocho veces para observarla en 
      orientaciones diferentes, en incrementos de 45°. Finalmente, filtra las detecciones por tamaño 
      y comprueba que sus centros estén dentro de la imagen.

<br>

`posicion_en_el_mundo(px, py)` → Estima las coordenadas de una cara a partir de la imagen y la posición del dron

      Estima las coordenadas de una persona a partir del centro de su cara en la imagen, la altura del 
      dron y los parámetros de la cámara.

      Calculamos primero la distancia focal a partir del ancho de la imagen y del campo de visión horizontal 
      de la cámara. En nuestro caso, utilizamos una imagen de 320 píxeles de ancho y un campo de visión de 59°.

      A partir de ella, calculamos los desplazamientos respecto al dron, a partir de la posición de su cara en 
      la imagen y de la altura a la que vuela el dron. Cuanto más alejada esté la cara del centro de la imagen,
      mayor será el desplazamiento estimado en esa dirección.

      Finalmente, sumamos estos desplazamientos a la posición actual del dron para obtener las coordenadas 
      estimadas de la persona. 
      
      Este cálculo utiliza una aproximación geométrica de la proyección de la cámara.


`es_persona_nueva(posicion)` → Comprueba si una posición corresponde a una detección nueva

<br>

`buscar_personas(imagen)` → Procesa las detecciones, las representa en la imagen y registra las posiciones nueva

<br>

`mostrar_camaras(imagen_ventral)` → Muestra las cámaras frontal y ventral en la interfaz.

<br>

`ir_a(x, y, buscando=True)` → Controla el movimiento, actualiza la batería y permite buscar personas

<br>

`queda_bateria_para(x, y)` → Estima si existe batería suficiente para llegar al siguiente punto y regresar al barco

<br>

`recargar()` → Regresa al barco, aterriza, simula la recarga y reanuda el vuelo desde la posición guardada

<br>
<br>

## Vídeo

https://urjc-my.sharepoint.com/:v:/r/personal/j_sanmiguel_2023_alumnos_urjc_es/_layouts/15/stream.aspx?id=%2Fpersonal%2Fj_sanmiguel_2023_alumnos_urjc_es%2FDocuments%2FDroneRescue.webm&nav=eyJyZWZlcnJhbEluZm8iOnsicmVmZXJyYWxBcHAiOiJTdHJlYW1XZWJBcHAiLCJyZWZlcnJhbFZpZXciOiJTaGFyZURpYWxvZy1MaW5rIiwicmVmZXJyYWxBcHBQbGF0Zm9ybSI6IldlYiIsInJlZmVycmFsTW9kZSI6InZpZXcifX0%3D&share=cQp0GnjmApzaRK2OcgEiA96cEgUCPaqiJj5Da2YS44-FWCH0BA
