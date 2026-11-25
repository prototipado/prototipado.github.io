---
title: "Del ECG al ECG–PPG: un modelo acoplado por eventos"
author: Juan I. Cerrudo
date: 2026-11-25
lang: es
page_id: ecg-ppg-event-model
permalink: /desarrollos/generador_ecg_sintetico_2/
categories:
  - Desarrollos
tags:
  - ECG
  - PPG
  - Raspberry Pi Pico
  - Python
layout: single
toc: true
toc_sticky: true
excerpt: "Un modelo por eventos genera ECG y PPG sincronizados: la actividad auricular, la conducción y cada latido ventricular determinan también el pulso periférico."
---

## Una nueva implementación de generación sintética

Esta serie empezó con un generador que reproducía muestras de ECG almacenadas. En el [primer post](/desarrollos/generador_ecg/) contamos ese desarrollo y sus distintas mejoras de hardware.

Después, en [Del registro al modelo: generación sintética de ECG en tiempo real](/desarrollos/generador_ecg_sintetico_1/), dimos el paso de la reproducción a la generación sintética: implementamos en la Raspberry Pi Pico un modelo matemático de conducción cardíaca basado en osciladores acoplados. Ese modelo nos permitió modificar la frecuencia y explorar distintas morfologías sin depender de un registro pregrabado.

Este post presenta una nueva implementación de esa generación sintética, basada en otro enfoque. Como referencia tomamos el trabajo [“An ECG–PPG Physiological Signal Emulator for Calibration and Validation of Cardiovascular Monitoring Devices”](https://doi.org/10.3390/s26175578), publicado en *Sensors* en 2026. Además de proponer otra manera de generar el ECG, el artículo incorpora algo que no estaba en la implementación anterior: una onda de fotopletismografía (PPG) vinculada temporalmente con cada latido ventricular.

## Un modelo por eventos

El modelo separa el ciclo en bloques: marcapasos auricular → conducción auriculoventricular → actividad ventricular → generación del pulso periférico. En vez de construir todas las ondas alrededor de una secuencia fija de picos R, mantiene eventos auriculares y ventriculares por separado.

Cada evento auricular puede conducirse o bloquearse. Si se conduce, el evento ventricular correspondiente aparece después del intervalo PR. Esa separación permite representar, por ejemplo, un bloqueo AV de primer grado —con PR prolongado—, un Mobitz I —con PR progresivamente mayor y luego un impulso bloqueado— o un bloqueo AV completo, en el que la actividad ventricular sigue su propio ritmo.

Las ondas del ECG se construyen con funciones gaussianas: la onda P se asocia a los eventos auriculares; Q, R, S y T, a los ventriculares. Luego, un vector cardíaco tridimensional se proyecta sobre los electrodos para obtener derivaciones con relaciones espaciales coherentes. El modelo contempla además clases de latido: un latido ventricular ectópico puede producir un QRS más ancho y tardío mediante otro conjunto de parámetros.

## PPG sincronizada

La novedad que más nos interesa es cómo se genera la PPG. Cada pulso periférico nace de un evento ventricular y aparece después de un retardo electromecánico:

**tiempo del pie del pulso = evento ventricular + PAT**  
**PAT = PEP + longitud del trayecto vascular / velocidad de onda de pulso**

En estas expresiones, PEP es el período preeyectivo y PAT el tiempo desde la activación eléctrica hasta la llegada del pulso. La forma de cada pulso se representa con cinco componentes gaussianas. Así se pueden ajustar su amplitud y morfología sin perder el vínculo temporal con el latido que lo originó.

Esto hace que las dos señales respondan juntas a los cambios del modelo. Si un impulso auricular se bloquea, no aparece un latido ventricular conducido ni un pulso periférico asociado. Si el latido ventricular es ectópico, el modelo puede generar a la vez un QRS más ancho y un pulso más tardío y atenuado. No hace falta inventar una segunda secuencia de pulsos ni alinearla a mano después.

También es importante qué significa aquí “salida PPG”: el artículo modela la componente pulsátil como una señal eléctrica analógica para conectar a una entrada de medición. No genera luz ni reemplaza un sensor óptico; tampoco modela la intensidad óptica absoluta o su componente continua.

## Qué cambia respecto del modelo de osciladores acoplados

En el post anterior el corazón se representaba mediante osciladores acoplados: el modelo describía la conducción y generaba el ECG a partir de las componentes asociadas a sus ondas. El artículo de 2026 organiza el proceso de otra manera: mantiene secuencias separadas de eventos auriculares y ventriculares, representa la conducción AV entre ellas y deriva tanto el ECG como la PPG de esos eventos.

| Aspecto | Modelo anterior (osciladores) | Nueva implementación del artículo |
|---|---|---|
| Organización del modelo | Osciladores acoplados para representar la conducción cardíaca | Eventos auriculares, conducción AV y eventos ventriculares separados |
| Generación del ECG | Componentes asociadas a las ondas, dentro del modelo de osciladores | Ondas gaussianas asociadas a eventos auriculares y ventriculares |
| Salidas | ECG | ECG y PPG sincronizados |
| Relación entre latido y pulso | No se generaba una PPG acoplada | Cada evento ventricular da origen a un pulso con un retardo electromecánico |

No se trata de que una implementación sea simplemente una versión corregida de la otra. El modelo de osciladores acoplados ya generaba ECG sintético y permitía explorar su frecuencia y morfología. El enfoque nuevo cambia la representación del ritmo y, sobre todo, amplía el modelo para incluir la respuesta periférica: la PPG no es una señal independiente añadida al final, sino que depende de los eventos ventriculares y de sus características.

El trabajo de 2026 también contempla reproducir ECG clínicos de 12 derivaciones convirtiéndolos a potenciales de electrodos para que sigan el mismo flujo de salida. Esa función de reproducción complementa el modelo sintético, pero la novedad que nos interesa en esta etapa es la generación acoplada de ECG y PPG.

El modelo permite modular los intervalos de los eventos con una componente sinusoidal asociada a la respiración. Sin embargo, en la evaluación del artículo esa modulación se desactivó para aislar las pruebas de temporización y de la cadena analógica. Es un detalle que conviene tener presente: que el modelo admita variabilidad no significa que todas las señales medidas en el trabajo la incorporen.

## Simulador del modelo en el navegador

Al igual que en el post anterior, dejamos acá una implementación interactiva del modelo corriendo en el navegador, implementada en JavaScript. Permite modificar los parámetros del marcapasos auricular, explorar distintos ritmos y bloqueos de conducción AV, y observar en tiempo real cómo cada evento ventricular genera tanto el complejo QRS como la onda de pulso periférico (PPG) con su retardo PAT correspondiente:

<a href="{{ '/assets/sims/ecg_ppg_model_sim.html' | relative_url }}"
   target="_blank" rel="noopener"
   style="display:block; max-width:720px; margin:1.5em auto;">
  <img src="{{ '/assets/images/generador_ecg_sintetico_2/sim.png' | relative_url }}"
       alt="Simulador ECG–PPG event-driven — abrir en pestaña nueva"
       style="width:100%; border-radius:10px; display:block;">
</a>

## Del modelo al hardware

El equipo del artículo implementó su emulador con un STM32F407 y conversores DAC para producir diez salidas analógicas sincronizadas: nueve potenciales de electrodos de ECG y un canal PPG, a 500 muestras por segundo. Esa arquitectura mantiene alineados los canales, incluido el retardo entre la actividad eléctrica y el pulso.

En la implementación anterior, nuestra Pico estaba organizada alrededor de ocho salidas de ECG: I, II y V1 a V6; las derivaciones de los miembros restantes se calculaban a partir de las señales disponibles. Por eso, incorporar la PPG no consiste solo en dibujar otra curva en la interfaz: implica agregar y calibrar un canal analógico, definir su amplitud y asegurar que su actualización quede sincronizada con las salidas de ECG.

Realizamos pruebas reutilizando uno de los canales existentes para poder generar la PPG, lo ideal seria agregar mas salidas analógicas al generador para poder incorporar en el futuro aún mas señales como por ejemplo respiración, presión arterial o temperatura.

## Bonus: fantoma de dedo

A partir de la búsqueda bibliográfica, encontramos el trabajo [An Optical Signal Simulator for the Characterization of Photoplethysmographic Devices](https://www.mdpi.com/1424-8220/24/3/1008), que demostraba que era posible simular señales PPG mediante la modulación de la luz emitida por LEDs. Tomando esa idea como punto de partida, quisimos comprobar si podíamos reproducir el principio de funcionamiento con una implementación más sencilla, prescindiendo inicialmente tanto de la etapa de sincronismo con el sensor como de las fuentes de corriente controladas utilizadas en el artículo. Para ello, tuvimos que considerar que la relación entre la tensión aplicada al LED, la corriente que circula por él y la intensidad luminosa emitida no es perfectamente lineal, y que depende de parámetros como la tensión directa del LED, la resistencia serie y los límites de funcionamiento de la etapa de salida. En lugar de incorporar una etapa adicional de control de corriente, optamos por modificar directamente la forma de onda de entrada, buscando compensar esas no linealidades para obtener una variación de luminosidad similar a la de una señal PPG real. Mediante un proceso iterativo de prueba y error, utilizando un generador de señales arbitrarias y observando la respuesta óptica obtenida, fuimos ajustando la forma de onda hasta conseguir una señal cuya morfología se aproximaba bastante a la buscada.

Para evaluar este enfoque en un escenario real, el paso siguiente fue fabricar un fantoma físico que se pudiera acoplar directamente a un oxímetro de pulso de dedal comercial y comprobar si el equipo era capaz de "leer" la señal óptica generada.

### El molde y la silicona

El molde se imprimió en ABS a partir del STEP de una mano. Dentro se colocaron tres leds: verde, rojo e infrarrojo. Una vez ensamblados los leds y sus conexiones, se coló silicona en el molde para aportarle la consistencia y la dispersión óptica necesarias para que la luz atraviese el material de forma parecida al tejido biológico. El resultado fue un fantoma sólido listo para insertar en el dedal. En los ensayos se excitó únicamente el led rojo, dejando el verde y el infrarrojo sin conectar.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/molde.png" alt="Fabricación del fantoma de dedo" caption="Fabricación del fantoma. De izquierda a derecha y de arriba abajo: los tres leds (verde, rojo e IR) antes del ensamble; el molde de ABS; los leds posicionados dentro del molde; el fantoma de silicona terminado." class="align-center" %}

### El desafío del sincronismo

Un oxímetro comercial no es un simple receptor pasivo: multiplexa en el tiempo la emisión roja e infrarroja y toma muestras intercaladas de luz ambiente para cancelarla. Al emitir luz desde el fantoma de forma continua y asincrónica, parte de nuestra señal puede caer en las ventanas de fondo y ser descartada. Sin embargo, en las pruebas comprobamos que el dedal sí respondía a la modulación externa, lo que abrió la puerta a ensayar distintas formas de onda.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/setup.jpg" alt="Setup del experimento con el fantoma y el oxímetro de dedal" caption="Setup completo: el fantoma de dedo insertado en el oxímetro de dedal, conectado al generador ECG–PPG." class="align-center" %}

### De la señal cruda a la compensación empírica

Al inyectar inicialmente la forma de onda de PPG directa desde el generador, la respuesta en la pantalla del oxímetro era irreconocible. Aunque la frecuencia de pulso coincidía, la morfología aparecía completamente distorsionada.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/dedo_ppg_generador.png" alt="Cuatro paneles: dedo real, fantoma con PPG, generador de funciones y configuración" caption="Arriba izquierda: captura del oxímetro con un dedo real (referencia). Arriba derecha: señal obtenida con el fantoma excitado con la PPG del generador ECG–PPG. Abajo izquierda: señal lograda con el generador de funciones (rampa asimétrica). Abajo derecha: configuración del generador de funciones Siglent SDG1050 usada en el experimento." class="align-center" %}

Para entender cómo reaccionaba el instrumento, recurrimos al generador de funciones Siglent SDG1050 y exploramos diversas señales patrón. Dos observaciones resultaron clave:

1. **Inversión de polaridad:** En un oxímetro, un aumento del volumen sanguíneo implica mayor absorción y menor luz recibida. Por ende, la señal lumínica debe tener la polaridad invertida respecto de la curva gráfica habitual de PPG para que en pantalla suba cuando corresponde.
2. **Efecto diferenciador del front-end:** La cadena analógica y digital del oxímetro se comporta como un filtro paso-alto que atenúa niveles continuos y acentúa cambios bruscos. Por ejemplo, al aplicar una rampa asimétrica (subida lenta y caída rápida), el equipo devolvía un pulso con un flanco ascendente marcado seguido de una meseta.

### Ajuste en el firmware

Con estas pautas en mente, adaptamos el firmware del generador para pre-compensar la señal digitalmente antes de enviarla a la modulación PWM del led:

- Se invirtió la polaridad de la curva sintética.
- Se aplicó una etapa de pre-énfasis (un integrador con fuga que contrarresta la derivada introducida por el paso-alto del dedal).
- Se mapeó el rango resultante a la ventana de trabajo útil del led rojo, respetando su tensión directa de encendido y el rango de salida de los operacionales con su resistencia serie de 120 Ω.

{% include figure popup=true image_path="/assets/images/generador_ecg_sintetico_2/ppg_preemph_plot.png" alt="Señal de PPG original y señal con pre-énfasis" caption="Señal de PPG original (gaussiana, invertida) y la versión con pre-énfasis enviada al PWM que excita el led rojo." class="align-center" %}

El resultado fue una mejora notable: el oxímetro comenzó a sincronizar de manera estable la frecuencia cardíaca y a trazar una curva con una morfología mucho más cercana a una pulsación real.

<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/generador_ecg_sintetico_2/comparacion.mp4" type="video/mp4">
    Tu navegador no soporta la reproducción de video.
  </video>
  <figcaption>Comparación lado a lado: dedo real (arriba) y fantoma con firmware de pre-énfasis (abajo). La frecuencia es correcta; la morfología todavía no es perfecta.</figcaption>
</figure>

### Estado actual y limitaciones

El fantoma constituye una prueba de concepto muy útil para validar el generador acoplado de forma no invasiva, aunque presenta limitaciones conocidas:

- **Operación asincrónica:** Al no estar enganchado en fase con el reloj de muestreo del oxímetro, parte de la señal emitida coincide con las fases de descarte de luz ambiente del equipo.
- **Canal único:** Al excitar solamente el led rojo, el cociente entre rojo e infrarrojo no está controlado, por lo que el valor de SpO₂ numérico que muestra el dedal no tiene validez física.
- **Respuesta no calibrada:** La pre-compensación se determinó de manera empírica; una linealización completa requeriría caracterizar formalmente la función de transferencia óptica y eléctrica del dispositivo receptor.

Aun así, validar que una señal sintética acoplada al ECG puede reproducirse ópticamente y ser interpretada por un equipo médico estándar demostró la viabilidad de la idea y dejó asentadas las bases para futuros refinamientos.


El código fuente del firmware, el diseño del hardware y los archivos del simulador están disponibles en el [repositorio del Laboratorio](https://github.com/prototipado/ECG_Phantom).

---

[![Hits](https://hits.sh/prototipado.github.io/desarrollos/generador_ecg_sintetico_2/.svg)](https://hits.sh/prototipado.github.io/desarrollos/generador_ecg_sintetico_2/)
