# PRACTICA 2 

Página de Enunciado: https://jderobot.github.io/RoboticsAcademy/exercises/Drones/rescue_people 

El objetivo de esta práctica es desarrollar un programa de control para un dron capaz de buscar supervivientes en una zona determinada, detectar sus caras mediante visión artificial y registrar sus posiciones para facilitar su posterior rescate.

En nuestro caso el programa sigue una estrategia de búsqueda en espiral cuadrada que permite recorrer progresivamente la zona de interés. Durante el vuelo, el dron análiza las imágenes de la cámara y estima las coordenadas de las personas detectadas.

## Configuración Inicial 

### Coordenadas de referencia

### Configuración de la cámara y del detector 

### Parámetros de la misión


<br>
<br>

## Funcionamiento

### Movimiento - Búsqueda 

### Detección de supervivientes 

### Gestión de la batería y recarga 

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
