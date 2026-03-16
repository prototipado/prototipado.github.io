---
title: "Algómetro portátil"
author: Juan I. Cerrudo
date: 2026-02-05
categories:
  - Tutorial
tags:
  - algómetro
  - diseño
  - prototipado
layout: single
toc: true
toc_sticky: true
---

## Introducción

Parte de las actividades del Laboratorio de Prototipado involucra brindar soluciones tecnológicas a otros laboratorios de la Facultad de Ingeniería de la UNER. Dentro de esas actividades, comenzamos hace un tiempo a trabajar con el Centro de Ingeniería en Rehabilitación e Investigaciones Neuromusculares apoyando en una de las ramas de investigación que llevan adelante, la que tiene que ver con el estudio del dolor.
En una primera etapa nos pidieron el desarrollo de un algómetro, un dispositivo que permite medir el umbral de dolor de un paciente. El dispositivo completo involucraba una suerte de sistema robótico con un actuador lineal que ejercía presión sobre la piel del paciente, un sensor de fuerza para medir la presión ejercida y un sistema de control del actuador. 

{% include figure image_path="/assets/images/sensor_dolor/robot.png" alt="Algómetro robótico" caption="Izq. Estructura del algómetro y estimulador (de Evaluación de un protocolo de un modelo de dolor con estímulos de sensación cortante utilizando un sistema robótico: estudio piloto). Der. Modelo CAD del estimulador, la versión final con soportes laterales para mejorar rigidez. Modelo CAD del sistema de control." class="align-center" %}

En una segunda etapa nos pidieron el desarrollo de un algómetro de mano, que permitiera medir el umbral de dolor de un paciente de forma manual, simplificando significativamente el sistema. 


## Rediseño
El sistema de control original contaba con 3 fuentes de alimentación: una para el actuador lineal, otra para un motor de pasos y otra para el sistema de control. En el sistema manual buscamos simplificar este esquema, y gracias a la reducción de componentes, logramos alimentar todo el dispositivo con una sola fuente. En la versión original utilizábamos un ESP32 DevKit, pero al no encontrarse en stock al momento de la compra, optamos por un [FireBeetle 2 ESP32-E](https://wiki.dfrobot.com/dfr0654/#tech_specs). Esta placa incluye un cargador de batería LiPo, lo que nos simplificaría la incorporación de una batería en el futuro para hacerlo totalmente portátil.
La interfaz de usuario está compuesta por una pantalla OLED de 128x64 píxeles y un encoder rotatorio con pulsador; también dejamos disponibles los pulsadores de la placa FireBeetle para un uso rápido. 
El sensor de fuerza utilizado es el mismo que usamos en el sistema original, el [FMAMSDXX005WC2C3](https://automation.honeywell.com/content/dam/honeywell-edam/sps/common/en-us/industries/healthcare-and-life-sciences/medical-equipment/documents/sps-medical-fma-series.pdf) de Honeywell. 
También le agregamos un conector BNC para utilizar como entrada con un pulsador (para indicar eventos desde el exterior) o como salida para sincronizar con otros dispositivos.




## Modelado CAD
Basamos el diseño físico en el sistema original, pero simplificamos la estructura para hacerlo significativamente más compacto y liviano. Formamos la estructura general con un stack de piezas impresas en 3D sostenidas por 3 tornillos M3x35mm. Elegimos esta estrategia porque nos permitía una rápida iteración en el rediseño de sectores particulares, además de facilitarnos la mejor orientación de las piezas a la hora de imprimirlas. 

Colocamos el sensor en una suerte de "cuna" diseñada en la pieza plástica que sostiene el sensor y soldamos los cables directamente a sus pads, evadiendo así la necesidad de integrar una PCB adicional.

{% include figure image_path="/assets/images/sensor_dolor/sensor.png" alt="Sensor de presión" caption="Sensor de presión, vista interior." class="align-center" %}

Realizamos la transmisión de fuerza (desde el punto de contacto exterior hacia el sensor interno) mediante una varilla de acero de 2.5mm de diámetro, de deslizamiento suave por estar colocada dentro de una camisa de bronce y sostenida por una pieza plástica. Un resorte se encarga de ejercer fuerza sobre la varilla, manteniéndola siempre en contacto con el sensor con una precarga constante cercana a los 120 gramos. 


{% include figure image_path="/assets/images/sensor_dolor/estimulador.png" alt="Algómetro robótico" caption="Estimulador manual, vista en corte." class="align-center" %}

A la pieza plástica que sostiene la varilla le diseñamos además un pequeño mango para permitirnos separar la varilla del sensor temporalmente y trabarla. De esta manera, podemos proteger de sobrecargas al delicado sensor mientras le estamos cambiando el efector de estimulación al paciente (que puede ser punzante, un filo, entre otros).

{% include figure image_path="/assets/images/sensor_dolor/traba_resorte.png" alt="Traba del resorte" caption="Detalle de la traba del resorte: permite separar la varilla del sensor y trabarla de forma segura para protegerlo mientras cambiamos la punta de prueba. Se ve la traba actuando en sus dos posiciones posibles." class="align-center" %}

Fabricamos además un capuchón especial que se acopla en la parte frontal para proteger tanto el efector como las piezas internas cuando estamos trasladando el dispositivo. En la parte trasera incluimos también un sujetacables, compuesto de una suerte de cuña impresa en material blando (TPU) y una tuerca de roscado rápido, lo cual nos posibilitó fijar firmemente el cable de fuerza al cuerpo del dispositivo, evitando tirones.

Por último, el gabinete para el módulo de control lo estructuramos en dos grandes partes encastradas, unidas por 3 tornillos M3. 

{% include figure image_path="/assets/images/sensor_dolor/gabinete.png" alt="Gabinete" caption="Gabinete del sistema de control, vista interna." class="align-center" %}

Para poder utilizar cómodamente los botones táctiles de la placa FireBeetle, diseñamos e imprimimos unas pequeñas estructuras plásticas flexibles que, una vez presionadas externamente, permiten accionar los botones en placa de forma suave. Si bien usamos iteraciones de este estilo de soluciones previamente, en esta oportunidad los botones de la FireBeetle estaban soldados a 90 grados. Esta orientación nos requirió un análisis más enfocado asegurando que la rotación natural de la pieza impresa convergiera en el desplazamiento lineal óptimo para disparar el "clic" interno mecánico del pulsador sin dañarlo.

<p align="center">
  <video autoplay loop muted playsinline width="80%">
    <source src="/assets/images/sensor_dolor/button_desplazamiento.webm" type="video/webm">
  </video>
</p>

## Pruebas

Para probar el dispositivo, decidimos imprimir unas piezas adicionales de amarre que nos permitían, por un lado, montar con firmeza el estimulador principal en una morsa para evitar vibraciones, y por otro lado, apoyar con seguridad de caídas una pesa estándar de 100 gramos directamente enfocada sobre el cabezal que empujaba hacia nuestro sensor. 

{% include figure image_path="/assets/images/sensor_dolor/setup_pruebas.jpg" alt="Setup de pruebas" caption="Setup de pruebas, con el sensor sobre una pesa de 100 gramos." class="align-center" %}

Realizamos capturas de datos en vacío y con la pesa sobre el sensor, tanto a 10Hz (frecuencia de uso normal con visualización en tiempo real) como a 500Hz (frecuencia máxima durante la realización de mediciones).

{% include figure image_path="/assets/images/sensor_dolor/analisis_0gr_10Hz.png" alt="Análisis en vacío a 10Hz" caption="Resultados del análisis de señal en vacío (0g) a una frecuencia de 10Hz." class="align-center" %}

{% include figure image_path="/assets/images/sensor_dolor/analisis_0gr_500Hz.png" alt="Análisis en vacío a 500Hz" caption="Resultados del análisis de señal en vacío (0g) a una frecuencia de 500Hz." class="align-center" %}

{% include figure image_path="/assets/images/sensor_dolor/analisis_100gr_10Hz.png" alt="Análisis con carga de 100g a 10Hz" caption="Resultados del análisis de señal con una carga de 100g a una frecuencia de 10Hz." class="align-center" %}

{% include figure image_path="/assets/images/sensor_dolor/analisis_100gr_500Hz.png" alt="Análisis con carga de 100g a 500Hz" caption="Resultados del análisis de señal con una carga de 100g a una frecuencia de 500Hz." class="align-center" %}

Estos resultados nos permitieron comprobar que logramos un desempeño de medición sumamente estable utilizando el algómetro en ambos escenarios. Al evaluarlo en vacío (0g), el equipo nos devolvió una señal muy próxima al cero perfecto, manteniéndose oscilando apenas entre ±1.6g en el promedio. Además, registramos que el grueso de esta mínima variabilidad se debe simplemente a una correlación natural con las alteraciones de temperatura ambiente. Al aplicarle la carga de 100g, el dispositivo demostró una precisión excelente: midió promedios exactos alrededor de los 99.8g usando lecturas rápidas de 10Hz y 100.1g subiendo la frecuencia a 500Hz. Para estar totalmente seguros, realizamos análisis espectrales (FFT) los cuales resultaron muy limpios; lo que nos terminó de revelar tanto la ausencia de armónicos nocivos como de influencias por ruidos electromagnéticos ajenos de alta frecuencia. Finalizamos confirmando con estos datos de laboratorio que nuestro rediseño estructural y el ensamble del circuito acoplado nos garantizan en conjunto unas lecturas excepcionalmente limpias y confiables.



<p align="center">
  <video controls width="80%">
    <source src="/assets/images/sensor_dolor/video_test.mp4" type="video/mp4">
  </video>
</p>

