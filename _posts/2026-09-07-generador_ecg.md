---
title: "Del ECG Real al Corazón Virtual-Construyendo un generador de ECG desde cero"
author: Juan I. Cerrudo
date: 2026-08-18
categories:
  - Desarrollos
tags:
  - ECG
  - Raspberry Pi Pico
  - Python
layout: single
toc: true
toc_sticky: true
excerpt: "Un generador de señales de ECG capaz de producir múltiples derivaciones con la calidad suficiente como para probar equipos de electrocardiografía y realizar demostraciones"
---

## Introducción

A mediados de 2024 surgió, dentro del Laboratorio, la idea de desarrollar un generador de señales de ECG. La inquietud fue planteada por Eduardo Filomena, uno de los integrantes del Laboratorio, con un objetivo bastante concreto: contar con una herramienta que permitiera probar equipos de electrocardiografía, pero que al mismo tiempo pudiera utilizarse con fines didácticos y demostrativos.
Para ese momento, nosotros ya llevábamos un tiempo trabajando en el desarrollo de un electrocardiógrafo inalámbrico de 12 derivaciones. Como parte de ese trabajo necesitábamos probar el equipo y, hasta entonces, las pruebas las realizábamos generalmente utilizando un generador de funciones que incorporaba algunas señales biológicas. El problema era que este equipo estaba limitado a un único canal de ECG.
Contar con un generador capaz de producir múltiples derivaciones y, además, permitirnos personalizar las señales generadas nos resultaba especialmente atractivo. No solo nos permitiría hacer pruebas más completas sobre el electrocardiógrafo, sino también generar distintas condiciones y señales para evaluar su comportamiento, y la posibilidad de automatizar sesiones de testeo.
A partir de esa necesidad comenzó el proyecto que vamos a documentar en esta serie de posts. La intención es recorrer el camino que seguimos durante el desarrollo: desde la recreación del diseño original, pasando por las pruebas y los problemas que fuimos encontrando, hasta las modificaciones y mejoras que finalmente fuimos incorporando.
El punto de partida era una [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico//) (RP2040) programada en Arduino. El microcontrolador tenía almacenados los datos de las señales y, a partir de ellos, generaba mediante PWM las señales correspondientes a las 10 derivaciones convencionales del ECG.
Como la salida del microcontrolador era una señal PWM, esta debía ser acondicionada para obtener una señal analógica. Para ello, las salidas pasaban por una serie de filtros pasa-bajos pasivos, encargados de atenuar la componente de alta frecuencia del PWM y reconstruir la forma de onda del ECG.


{% include figure popup=true image_path="/assets/images/generador_ecg/primer_prototipo.jpeg" alt="Primer prototipo del generador de ECG" caption="Primer prototipo del generador de ECG." class="align-center" %}


Nuestro primer paso fue, entonces, recrear este dispositivo. Antes de modificarlo o pensar en una nueva versión, queríamos entender cómo estaba construido, cómo se generaban las señales y, sobre todo, comprobar qué tan bien funcionaba en la práctica.

<p align="center">
  <video controls width="80%">
    <source src="/assets/images/generador_ecg/ecg_3.mp4" type="video/mp4">
  </video>
</p>


Ese primer prototipo resulto de utilidad para realizar las primeras pruebas de adquisión de señales con el prototipo del electrocardiógrafo que estábamos desarrollando. 

{% include figure popup=true image_path="/assets/images/generador_ecg/setup.jpg" alt="Setup de prueba con el dispositivo y el electrocardiografo en el que se estaba trabajando" caption="Setup de prueba con el dispositivo y el electrocardiografo en el que se estaba trabajando" class="align-center" %}


Inmediatamente después de recrear el dispositivo original, nos pusimos a trabajar en mejorar sus capacidades.
Uno de los primeros cambios fue migrar el código a C. A partir de ese momento, todo el desarrollo posterior lo realizamos en Visual Studio Code, utilizando el SDK oficial de Raspberry Pi para la Pico. Como ocurre cada vez más en nuestros proyectos, buena parte del desarrollo y de las tareas de programación fue asistida por distintas herramientas de inteligencia artificial, que utilizamos como apoyo durante el proceso.
Pero no queríamos quedarnos solamente con una mejora del software. A medida que empezamos a utilizar el generador, también fueron apareciendo nuevas necesidades.
Nos parecía mucho más útil contar con un dispositivo que pudiera transportarse fácilmente y utilizarse de manera independiente, sin depender de una computadora durante las pruebas. Por eso decidimos incorporar una batería (18650 LiPo) como fuente de alimentación, junto con una placa para gestionar su carga. Esto, además de hacerlo portátil, tenía otra ventaja interesante para nuestro caso: permitía trabajar con la referencia de masa flotante respecto de otros equipos durante los ensayos.
También incorporamos una interfaz de usuario propia, basada en una pantalla OLED y un encoder rotativo. De esta manera, podíamos seleccionar las señales y configurar el generador directamente desde el dispositivo, sin necesidad de conectarlo a una PC.

Todo el diseño del gabiente lo fuimos realizando en Fusion360, pasando por varias versiones.

{% include figure popup=true image_path="/assets/images/generador_ecg/evol.png" alt="Evolución del generador de ECG" caption="Evolución del generador de ECG." class="align-center" %}

## El diseño electrónico
El diseño del circuito también fue pasando por varias etapas. A medida que avanzábamos con el desarrollo y entendíamos mejor qué necesitábamos del dispositivo, fuimos modificando tanto la electrónica como la forma de acondicionar las señales. En primera medida cambiamos por una [Raspberry Pi Pico 2](https://www.raspberrypi.com/products/raspberry-pi-pico-2/) (RP2350), para poder mejorar el rendimiento del dispositivo. Si bien el microcontrolador no era el cuello de botella del sistema, si nos permitía contar con más memoria y un procesador mas rápido. 

<p align="center">
  <video controls width="80%">
    <source src="/assets/images/generador_ecg/final_2.mp4" type="video/mp4">
  </video>
</p>


La implementación final cuenta con filtros pasa-bajos activos, específicamente Sallen-Key, diseñados para reconstruir las señales analógicas a partir de las salidas PWM de la Raspberry Pi Pico. El diseño estuvo bastante influenciado por el uso de componentes que ya teníamos disponibles en el laboratorio. Los operacionales son [OPA2335](https://www.ti.com/product/OPA2335) de Texas Instruments, amplificadores operacionales cuya principal caracteristicas que nos interesaba es el hecho qde que son "rail-to-rail", lo que significa que pueden manejar voltajes de entrada y salida que se acercan mucho a los de alimentación. Buscando que componentes pasivos teniamos a disponibilidad, llegamos a una combinacion que nos permitia tener una fc de 256Hz, adecuada para la aplicación.

Antes de llevar el circuito a la placa, realizamos distintas simulaciones para verificar su comportamiento y ajustar los componentes. Esto nos permitió evaluar la respuesta en frecuencia, la atenuación de la componente de alta frecuencia del PWM y la forma de onda obtenida a la salida.

## Simulación
Como se puede ver en la imágen, la simulación se llevó a cabo utilizando [LTspice](https://www.analog.com/en/resources/design-tools-and-calculators/ltspice-simulator.html), un simulador de circuitos electrónicos gratuito desarrollado por Analog Devices.

{% include figure popup=true image_path="/assets/images/generador_ecg/sim.png" alt="Simulación del filtro pasa-bajo activo." caption="Simulación del filtro pasa-bajo activo." class="align-center" %}

La salida de los filtros pasa por un divisor resistivo que reduce la señal, un segundo operacional configurado como seguidor emisor desacopla el divisor de una resistencia de salida de 470 ohm, que pretende simular la impedancia de tejido.

{% include figure popup=true image_path="/assets/images/generador_ecg/Figure_1.png" alt="Simulación del filtro pasa-bajo activo." caption="Resultados de la simulacion temporal, desde arriba hacia abajo; salida del ecg luego del divisor resistivo, señal original, señal de PWM filtrada, señal de PWM sin filtrar." class="align-center" %}

Además, por ese mismo momento estábamos a punto de comenzar en el Laboratorio un nuevo proyecto: el desarrollo de un marcapasos externo.
Esto nos llevó a agregar una funcionalidad que, si bien no era necesaria para el funcionamiento del generador de ECG, podía resultar muy útil a futuro. Incorporamos tres circuitos de entrada conectados al ADC de la Raspberry Pi Pico, destinados a detectar los pulsos generados por un marcapasos.

{% include figure popup=true image_path="/assets/images/generador_ecg/pace_sim.png" alt="Circuito de entrada para evaluacion de marcapasos." caption="Circuito de entrada para evaluacion de marcapasos." class="align-center" %}

La idea detrás de esta incorporación era ir un paso más allá de simplemente generar señales de ECG. A futuro, queríamos utilizar este hardware como parte de un sistema capaz de simular la interacción entre un corazón y un marcapasos, avanzando eventualmente hacia la implementación de un modelo in silico del corazón.
En ese momento todavía era una idea a futuro, pero nos pareció interesante dejar preparada la plataforma para que el mismo dispositivo pudiera formar parte de ese desarrollo.
 
{% include figure popup=true image_path="/assets/images/generador_ecg/final_1.jpg" alt="Prototipo final" caption="Prototipo final del generador de ECG. Se pueden ver secciones de filamento transparente de PETG usados como 'lightpipes' para guiar la luz de los leds hacia el exterior del gabinete." class="align-center" %}


<p align="center">
  <video controls width="80%">
    <source src="/assets/images/generador_ecg/final_3.mp4" type="video/mp4">
  </video>
</p>

## Generación y procesamiento de las señales
Otra de las mejoras que incorporamos fue ampliar las posibilidades de generación de señales. Además de las señales de ECG, agregamos señales de prueba básicas —senoidales, cuadradas y triangulares— con la posibilidad de modificar directamente su amplitud, offset y frecuencia.
Estas señales fueron particularmente útiles durante el desarrollo, ya que nos permitieron probar cada etapa del sistema utilizando formas de onda conocidas y fácilmente parametrizables. De esta manera podíamos verificar por separado la generación mediante PWM, el funcionamiento de los filtros y la respuesta de las salidas antes de pasar a señales más complejas.
Pero la parte más interesante vino al trabajar con señales de ECG reales.
Para esto utilizamos la [Lobachevsky University Electrocardiography Database (LUDB)](https://www.physionet.org/content/ludb/1.0.1/), una base de datos que contiene registros de ECG de 12 derivaciones. Desarrollamos un notebook en Python encargado de tomar estos registros y transformarlos en datos que pudieran ser utilizados directamente por el firmware de la Raspberry Pi Pico.
El primer paso es leer el registro y obtener las 12 derivaciones. A partir de una de ellas se detectan los picos correspondientes a los complejos QRS, que utilizamos como referencia para identificar los distintos ciclos cardíacos presentes en el registro.

{% include figure popup=true image_path="/assets/images/generador_ecg/jupyter_1.png" alt="Ejemplo de senal ecg de la base de datos LUDB." caption="Ejemplo de senal ecg de la base de datos LUDB." class="align-center" %}

En lugar de seleccionar simplemente un latido cualquiera, buscamos aquel que pudiera repetirse de manera continua. Para cada ciclo posible calculamos la diferencia entre su valor inicial y final, considerando todas las derivaciones, y seleccionamos el que presenta la menor diferencia. De esta manera obtenemos un latido que, al comenzar nuevamente después de finalizar, genera una transición lo más suave posible entre ciclos.

{% include figure popup=true image_path="/assets/images/generador_ecg/jupyter_2.png" alt="Ciclo de ECG seleccionado para generacion." caption="Ciclo de ECG seleccionado para generacion." class="align-center" %}

Una vez seleccionado el ciclo, extraemos las muestras correspondientes a las nueve derivaciones que utilizamos en el generador y las llevamos a una escala adecuada para su representación como valores enteros. En nuestro caso, los datos son reescalados mediante la operación 2000 + 30000, quedando centrados alrededor del valor 30000.

Este centrado a media escala responde al funcionamiento de la Raspberry Pi Pico: al alimentarse con una fuente única (0V a 3.3V) y operar con una resolución de PWM de 16 bits (0 a 65535), el valor 30000 establece un nivel de continua o masa virtual de aproximadamente 1.51V (45.7 de ciclo de trabajo). De este modo, se dispone del margen necesario para representar tanto las deflexiones positivas (ondas P, R, T) como las negativas (ondas Q, S) de la señal del ECG sin recortes en 0V.

Finalmente, todas estas muestras se convierten en enteros y se empaquetan automáticamente en un archivo de cabecera .h. El archivo contiene el array con las muestras del latido, junto con una estructura que almacena información adicional del registro, como el ritmo y las características asociadas a la señal.
El resultado es, entonces, un latido de ECG real convertido en un formato directamente utilizable por el firmware. La Raspberry Pi Pico no necesita procesar la señal original ni acceder a la base de datos: simplemente dispone en memoria de las muestras del ciclo y puede reproducirlas de manera repetitiva a través de las salidas PWM.
Este procesamiento también nos permitió algo que buscábamos desde el comienzo: poder incorporar nuevas señales al generador sin tener que modificar manualmente el firmware. A partir de un registro de la base de datos, el notebook se encarga de seleccionar y preparar el latido y generar el archivo que posteriormente incorporamos al proyecto.
De esta manera, el generador pasó a contar con dos grandes grupos de señales: por un lado, señales sintéticas y parametrizables para las pruebas del hardware; y por otro, señales fisiológicas reales, procesadas y preparadas para su reproducción en el dispositivo.


## Comunicación y CLI
Para facilitar y automatizar testeo decidimos tambien agregar una CLI al dispositivo, para poder modificar mediante comandos el comportamiento del mismo, por ejemplo el tipo de señal (sintética o real) y sus parámetros.
Luego creeamos una pequeña App en Python con una interfaz grafica para controlar el dispositivo de manera mas amigable. Otra funcionalidad importante de esta app es que nos permite ver en tiempo real las senales que se están generando. Esto permitiria en un futuro comparar la señal generada con una hipotetica señal adquirida y evaluar la perfimrance del sistema de adquisicion, una manera de "cerrar el lazo".

{% include figure popup=true image_path="/assets/images/generador_ecg/gui.png" alt="Interfaz grafica del software de control del generador de ECG." caption="Interfaz grafica del software de control del generador de ECG." class="align-center" %}

## Una limitación y el próximo paso
A pesar de todas estas mejoras, el generador todavía tenía una limitación importante: las señales que podía reproducir estaban definidas de antemano y almacenadas en memoria. Podíamos elegir entre distintos registros y reproducir señales reales, pero no teníamos una forma sencilla de modificar su morfología o su frecuencia cardíaca en tiempo real.
Esto empezó a resultar cada vez más evidente a medida que utilizábamos el dispositivo para hacer pruebas. Si queríamos cambiar la frecuencia cardíaca, modificar alguna característica de la morfología o generar una condición determinada, necesitábamos preparar previamente una nueva señal y volver a incorporarla al firmware.
La pregunta que apareció entonces fue bastante natural: ¿podíamos generar el ECG en tiempo real, en lugar de limitarnos a reproducir señales previamente almacenadas?
A partir de esta pregunta, en los próximos posts vamos a explorar el uso de modelos matemáticos para la generación de señales de ECG. La idea es pasar de un sistema basado principalmente en la reproducción de registros almacenados a uno en el que podamos controlar de manera mucho más precisa las características de la señal, modificándolas en tiempo real según lo que necesitemos probar.
Esto abre una nueva etapa del proyecto, en la que ya no solo nos interesa reproducir un ECG, sino también entender cómo podemos modelarlo y generarlo.
Pero había además una limitación que no era solamente técnica. Al repetir indefinidamente un mismo registro, también estábamos perdiendo parte de la dinámica propia de un corazón real. Cada ciclo era exactamente igual al anterior: el intervalo RR se repetía, la morfología de las ondas permanecía constante y no existía una variación natural de la frecuencia cardíaca. Tampoco teníamos una dinámica de la línea de base, ni pequeñas variaciones latido a latido en la amplitud y duración de las distintas componentes. En otras palabras, podíamos reproducir muy bien una determinada señal, pero no estábamos reproduciendo el comportamiento dinámico que hay detrás de esa señal.
Esto empezó a ser especialmente evidente cuando pensamos en lo que queríamos hacer con el generador. No nos alcanzaba con tener un ECG que "se pareciera" a un registro real; queríamos poder modificar su frecuencia cardíaca, introducir variabilidad, cambiar su morfología y simular distintas condiciones sin tener que buscar y almacenar un nuevo registro para cada caso. La primera versión seguía siendo muy útil para probar la electrónica y verificar la adquisición de señales reales, pero tenía un límite bastante claro: no estábamos simulando un corazón, estábamos reproduciendo una grabación de un corazón. Y justamente esa diferencia fue la que nos llevó a buscar una forma de generar el ECG a partir de un modelo matemático.

Por lo pronto, los archivos necesarios para la implementación de esta primer versión del generador están disponibles en GitHub, en el [repo del Laboratorio](https://github.com/prototipado/ECG_Phantom).


---

[![Hits](https://hits.sh/prototipado.github.io/desarrollos/generador_ecg/.svg)](https://hits.sh/prototipado.github.io/desarrollos/generador_ecg/)