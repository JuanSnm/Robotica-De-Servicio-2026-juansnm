# PRACTICA 4

El objetivo de esta práctica es desarrollar un sistema de navegación autónoma para un robot logístico capaz de recoger, transportar y depositar estanterías en diferentes ubicaciones de un almacén.

Para ello, el robot dispone de información sobre su posición y de un mapa del entorno, que utiliza para planificar sus desplazamientos y evitar posibles colisiones con los obstáculos. La práctica se desarrolla en el entorno de simulación Amazon Warehouse de Robotics Academy, utilizando Gazebo para representar el almacén y OMPL (Open Motion Planning Library) para realizar la planificación de trayectorias.


Hemos desarrollado tres soluciones diferentes para abordar el problema:

Caso 1: robot holonómico aproximado como un punto, utilizando un mapa de obstáculos engordados.

Caso 2: robot holonómico representado mediante una geometría rectangular, considerando su posición y orientación.

Caso 3: robot Ackermann, teniendo en cuenta sus restricciones cinemáticas y utilizando trayectorias basadas en curvas de Reeds-Shepp.

En los tres casos, el objetivo final es trasladar las seis estanterías a las zonas de almacenamiento señalizadas, actualizando la representación del entorno después de cada movimiento.
