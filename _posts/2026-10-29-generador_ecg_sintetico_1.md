---
title: "Del registro al modelo: generación sintética de ECG en tiempo real"
author: Juan I. Cerrudo
date: 2026-10-29
lang: es
page_id: synthetic-ecg-generator-1
permalink: /desarrollos/generador_ecg_sintetico_1/
categories:
  - Desarrollos
tags:
  - ECG
  - Raspberry Pi Pico
  - Python
layout: single
toc: true
toc_sticky: true
excerpt: "Implementamos en una Raspberry Pi Pico un modelo matemático de conducción cardíaca para generar ECG en tiempo real, variar la frecuencia y explorar distintas morfologías."
---

## Del ECG almacenado al ECG generado en tiempo real

En el [post anterior](https://prototipado.github.io/desarrollos/generador_ecg/) contamos cómo empezamos a construir el generador a partir de señales reales tomadas de una base de datos. La Raspberry Pi Pico almacenaba las muestras y las reproducía. Por eso, para cambiar la frecuencia cardíaca, modificar la morfología o representar una condición determinada, había que preparar otro registro y volver a incorporarlo al firmware.

En definitiva, se parecía más a un reproductor que a un generador.

Hasta ese momento, el camino era más o menos: base de datos → procesamiento → muestras → memoria → reproducción.

Lo que buscábamos era algo diferente: parámetros → modelo matemático → señal → salida.

Esto nos permitiría modificar características de la señal sin tener que preparar previamente una nueva forma de onda. La frecuencia cardíaca, por ejemplo, podría ser un parámetro del modelo y no una característica fija de un conjunto de muestras.

Buscando cómo encarar esto encontramos un trabajo que nos llamó particularmente la atención: [Cardiac Conduction Model for Generating 12 Lead ECG Signals With Realistic Heart Rate Dynamics](https://ieeexplore.ieee.org/document/8466837), publicado en 2018 en IEEE Transactions on NanoBioscience. El trabajo propone justamente un modelo matemático capaz de generar señales de ECG de 12 derivaciones incorporando además dinámica en la frecuencia cardíaca.

## ¿Cómo generamos un ECG a partir de ecuaciones?

El modelo representa distintas partes del sistema de conducción cardíaco mediante osciladores dinámicos. En particular, utiliza tres nodos principales — SA, AV y His-Purkinje — modelados mediante osciladores modificados de tipo [Van der Pol](https://es.wikipedia.org/wiki/Oscilador_de_van_der_Pol), con acoplamiento mediante retardos temporales.

Además aparecen otros cuatro osciladores, basados en el modelo de [Fitz Hugh-Nagumo](https://es.wikipedia.org/wiki/Modelo_de_Fitz_Hugh-Nagumo), que representan las principales componentes que terminamos reconociendo en un ECG: la onda P, la onda Ta, el complejo QRS y la onda T.

La señal final se construye combinando esas componentes. En su forma básica, el modelo utiliza:

ECG = z0 + z1 − z2 + z3 + z4

y, para representar las diferentes derivaciones, introduce distintos pesos sobre cada componente:

ECG = z0 + α1z1 − α2z2 + α3z3 + α4z4

Los coeficientes α1 a α4 no son arbitrarios: fueron obtenidos mediante un procedimiento de ajuste utilizando registros reales de ECG. Dicho de una manera bastante menos matemática: cada componente aporta una parte de la señal y, dependiendo de la derivación, esa contribución cambia.

Para llevar el modelo al microcontrolador implementamos (cuando digo implementamos me refiero a nosotros y a una plétora de LLMs 😅​) las ecuaciones en C y utilizamos integración numérica mediante Runge-Kutta de cuarto orden (RK4), trabajando con una frecuencia de muestreo de 2000 Hz.

Una de las partes que más nos interesaban del modelo era justamente la posibilidad de trabajar con una frecuencia cardíaca que no fuera completamente constante. El trabajo introduce variabilidad de la frecuencia cardíaca mediante un modelo de variabilidad del intervalo RR, con componentes alrededor de 0,1 Hz y 0,25 Hz, asociadas a las bandas de baja y alta frecuencia — las ondas de Mayer y la arritmia sinusal respiratoria.

En la implementación original del paper, esto se construye mediante una señal espectral y una transformada inversa de Fourier para obtener el tacograma RR.

Nosotros hicimos una adaptación más sencilla para la Pico. En lugar de implementar todo ese procedimiento espectral, utilizamos directamente dos modulaciones sinusoidales, una a 0,1 Hz y otra a 0,25 Hz, combinadas con pesos de 60 % y 40 %.

No es exactamente lo mismo que plantea el artículo. Y preferimos dejarlo claro porque, en estos proyectos, decir "implementamos el paper" cuando en realidad hicimos una adaptación puede terminar siendo bastante engañoso.

Pero para nuestro objetivo era un buen punto de partida. Lo importante era que la frecuencia cardíaca dejaba de ser simplemente una propiedad de los datos almacenados. Ahora podía formar parte del comportamiento dinámico del modelo.

El trabajo que utilizamos como referencia genera las 12 derivaciones estándar del ECG y ajusta los coeficientes correspondientes para cada una de ellas. Nuestra implementación, en cambio, está pensada alrededor de la electrónica que ya habíamos desarrollado. Generamos directamente ocho salidas: I, II y V1 a V6. Las derivaciones III, aVR, aVL y aVF se obtienen posteriormente a partir de las señales de los electrodos de los miembros.

Hasta ese momento ya habíamos conseguido algo que para nosotros era bastante significativo: generar un ECG mediante un modelo matemático y modificar su comportamiento sin depender de una señal almacenada.

Pero faltaba probar si el modelo servía para algo más que generar un ritmo sinusal con una frecuencia variable.

El paper incluía, entre sus ejemplos, una simulación de isquemia tipo Wellens 1, donde aparecen modificaciones características de la morfología, especialmente en las derivaciones precordiales.

Eso nos abrió otra posibilidad. En lugar de pensar solamente en "frecuencia cardíaca", empezamos a pensar en condiciones que pudiéramos seleccionar desde el propio equipo. Así fueron apareciendo distintos presets: ritmo sinusal normal, bradicardia sinusal, taquicardia sinusal, bloqueo SA-AV, bloqueo AV-HP y Wellens.

Antes de meternos con las capturas del hardware, dejamos acá una implementación del mismo modelo corriendo en el navegador, en JavaScript en vez de C, pero con las mismas ecuaciones y los mismos parámetros que terminamos usando en la Pico:

## Simulador del modelo de conducción cardíaca

Tiene los mismos presets que el firmware, las 12 derivaciones a la vista al mismo tiempo, y controles para tocar frecuencia cardíaca, variabilidad y ruido en vivo. No reemplaza al generador real, pero sirve para entender de dónde sale cada onda antes de ponerse a mirar un osciloscopio, y para cualquiera que quiera probar el modelo sin tener la placa en la mano.

<a href="{{ '/assets/sims/ecg_model_sim.html' | relative_url }}"
   target="_blank" rel="noopener"
   style="display:block; max-width:720px; margin:1.5em auto;">
  <img src="{{ '/assets/images/generador_ecg_sintetico_1/simulador_web_conduction.png' | relative_url }}"
       alt="Simulador del modelo de conducción cardíaca — abrir en pestaña nueva"
       style="width:100%; border-radius:10px; display:block;">
</a>

## Viendo la señal salir de la placa

Una cosa es que el modelo se comporte bien en la simulación; otra es comprobar la señal analógica a la salida del dispositivo. Para esta prueba usamos el mismo hardware de filtrado y acondicionamiento que describimos en el post anterior.

Con el osciloscopio conectado a la salida del filtro pasa-bajos, pudimos comparar directamente lo que predecía el modelo con lo que efectivamente estábamos entregando.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_1/comparacion.png" alt="Comparación de señales del modelo y la salida del generador. La señal se toma previo al divisor de voltaje final." caption="Comparación de señales del modelo y la salida del generador." class="align-center" %}

## Un panel para no trabajar a ciegas

Como ya contamos en el post anterior, desarrollamos una aplicación en Python que se comunica con la Raspberry Pi Pico por USB y permite controlar el generador desde la computadora. Para esta etapa ampliamos sus controles para trabajar con el nuevo modelo.

Desde la interfaz podemos seleccionar el modelo y el ritmo cardíaco, modificar parámetros como la amplitud, los coeficientes de las distintas componentes del ECG y los retardos entre nodos, además de activar o desactivar la variabilidad de la frecuencia cardíaca.

Pero la parte que más nos servía durante las pruebas era poder visualizar las señales en tiempo real: la aplicación recibe los datos que genera la Pico y los grafica simultáneamente para las distintas derivaciones, permitiéndonos cambiar un parámetro y comprobar prácticamente en el momento cómo se modifica la forma de onda.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_1/gui.png" alt="Interfaz de control del modelo de ECG" caption="Interfaz de control del modelo de ECG." class="align-center" %}

Así fuimos dejando de trabajar a ciegas, mirando solamente lo que ocurría en el código o en el osciloscopio, y pasamos a tener una especie de panel de control para experimentar con el modelo y entender cómo cada cambio afectaba al ECG.

## Calibración de la salida

También necesitábamos controlar mejor la amplitud de la señal de salida. Al principio ajustamos de forma aproximada los valores de PWM para obtener señales del orden de los milivoltios. Aunque la interfaz mostraba las formas de onda, la amplitud graficada no correspondía necesariamente con la amplitud analógica real.

Para mejorar esa estimación, incorporamos un procedimiento de calibración: la Pico genera una señal senoidal y mide su respuesta con uno de sus canales ADC. 

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_1/sine_cal.bmp" alt="Señal senoidal utilizada durante la calibración de la salida" caption="Señal senoidal utilizada durante la calibración de la salida." class="align-center" %}

Con esa lectura podemos relacionar los valores digitales con la amplitud observada y ajustar la ganancia de salida. Una vez calibrada, la GUI también permite ver el valor de la señal recortada cuando se solicita una amplitud que supera el rango de salida del amplificador operacional y este satura.


{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_1/recortado.png" alt="Valor de la señal recortada indicado en la GUI" caption="La GUI muestra el valor de la señal cuando se produce recorte por saturación del amplificador operacional." class="align-center" %}

Comprobamos este recorte con el osciloscopio en la salida V6, donde se observa la saturación de la señal:

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_1/v6_recortado.bmp" alt="Señal de salida V6 recortada por saturación, observada con el osciloscopio" caption="Verificación con el osciloscopio del recorte por saturación en la salida V6." class="align-center" %}

La calibración se incorporó también en el firmware anterior y se actualizo el repo de hardware con los cambios del pcb.

## De reproductor a simulador

La primera versión del proyecto nos había permitido resolver una necesidad concreta: reproducir señales de ECG reales y utilizarlas para probar nuestro electrocardiógrafo. Esta segunda etapa atacó otra limitación: tener control sobre la señal.

Y ese cambio modificó bastante el concepto del dispositivo. Ya no se trataba solamente de almacenar un montón de ECG y elegir cuál reproducir. La idea empezaba a ser otra: tener un modelo capaz de generar una señal y modificarla en función de lo que quisiéramos estudiar o probar.

Todavía quedaban problemas por resolver, por supuesto. La implementación matemática tenía que convivir con las limitaciones del microcontrolador, había que verificar que las señales obtenidas fueran realmente útiles, y después quedaba todo el camino del acondicionamiento analógico y las mediciones.

Pero esa es justamente la parte interesante. Habíamos pasado de preguntarnos "¿qué ECG podemos reproducir?" a preguntarnos "¿qué ECG podemos generar?".

En proximas entregas nos vamos a enfocar en probar diferentes modelos y evaluar la calidad de la señal generada con ellos.

---

[![Hits](https://hits.sh/prototipado.github.io/desarrollos/generador_ecg_sintetico_1/.svg)](https://hits.sh/prototipado.github.io/desarrollos/generador_ecg_sintetico_1/)
