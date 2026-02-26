---
title: "Tonómetro de pulso"
author: Juan I. Cerrudo
date: 2026-02-05
categories:
  - Tutorial
tags:
  - tonómetro
  - diseño
  - prototipado
layout: single
toc: true
toc_sticky: true
---

## Introducción

Hace unos años se acercó al Laboratorio de Prototipado un médico con la inquietud de poder medir la presión arterial utilizando un tonómetro de pulso.
Un tonómetro de pulso es un dispositivo que mide la presión sanguínea y las características de la onda de pulso directamente sobre una arteria superficial, generalmente en la muñeca. El sensor aplana la arteria radial contra el hueso y mide la presión necesaria para mantenerla colapsada.
La idea era poder incorporar esta funcionalidad al [BioAmp](https://github.com/prototipado/bioamp), de manera de poder medir simultáneamente la presión arterial y la actividad eléctrica del corazón. La medición simultánea de ECG y presión permite estimar parámetros como el tiempo de tránsito de pulso (PTT), correlacionado con la rigidez arterial y utilizado como indicador temprano de riesgo cardiovascular.

Inicialmente se intentó replicar un sistema, con el que ya estaban trabajando, que implicaba el uso de un sensor de presión invasivo. Este es un dispositivo médico diseñado para la monitorización continua y precisa de la presión arterial y otros parámetros fisiológicos. Funciona mediante la inserción de un catéter directamente en un vaso sanguíneo (habitualmente una arteria), el cual transmite la presión a un transductor externo que convierte la señal mecánica en una señal eléctrica. Las tubuladuras del sensor se llenaban con agua, el extremo del catéter era modificado con una membrana flexible (dedo de guante de látex o un globo) y se colocaba en la muñeca del paciente. La presión arterial movía la membrana y con ella la columna de agua, permitiendo medir la presión arterial. El sensor se conectaba a uno de los canales diferenciales del BioAmp, la salida tipo puente de Wheatstone generaba una diferencia de potencial proporcional a la presión ejercida que podía ser monitoreada junto con la señal de ECG.
Este sistema no resultó muy efectivo, al menos en nuestra implementación, con dificultades para el llenado de las tubuladuras y problemas para evitar burbujas de aire dentro de las mismas. Además, desde el punto de vista dinámico, el sistema hidráulico introducía una compliancia elevada y un comportamiento fuertemente amortiguado, lo que degradaba la fidelidad de la forma de onda de presión y reducía la sensibilidad del sensor.


## Rediseño

Decidimos rediseñar el sistema desde cero, modificando el transductor, buscando algo más fácil de utilizar. Durante la investigación bibliográfica que emprendemos cuando llevamos adelante un proyecto, una de las fuentes de búsqueda son patentes. Una que nos resultó particularmente interesante es la patente [US20050177047A1](https://patents.google.com/patent/US20050177047A1/en); "Device for, and a method of, transcutaneous pressure waveform sensing of an artery and a related target apparatus".

{% include figure image_path="/assets/images/tonometro/US20050177047A1-20050811-D00000.png" alt="Patente US20050177047A1" caption="Figura de la patente US20050177047A1 que describe el sistema de tonometría con cono de gel." class="align-center" %}

En la imagen del dispositivo se podía observar el sensor de presión, una suerte de cono de gel que transmitía la presión arterial desde la superficie exterior al sensor. Ya estábamos en la búsqueda de sensores de presión invasiva para evaluar su reutilización, y de la imagen nos pareció reconocer el encapsulado del sensor utilizado, si bien no estamos 100% seguros se parece mucho al [MPX2300DT1](https://www.nxp.com/docs/en/data-sheet/MPX2300D.pdf) de NXP. 

{% include figure image_path="/assets/images/tonometro/MPX2300DT1.png" alt="Sensor MPX2300DT1" caption="Sensor de presión MPX2300DT1 (NXP), encapsulado compatible con el utilizado en sistemas de monitoreo invasivo." class="align-center" %}

Este sensor es un transductor de presión de silicio de 0 a 2.3 MPa (0 a 23 bar) con salida de voltaje compensada en temperatura, utilizado en dispositivos de monitoreo de presión invasiva.  


## Modelado CAD

Comenzamos el modelado CAD del dispositivo con el objetivo de reproducir la geometría descrita en la patente. Para el cono de gel se optó por utilizar silicona, diseñando y fabricando previamente un molde en PLA mediante impresión 3D. El molde se fijaba en una morsa para asegurar estabilidad durante el proceso de colado.

{% include figure image_path="/assets/images/tonometro/corte_a.png" alt="Corte transversal del dispositivo" caption="Corte longitudinal del modelo CAD mostrando la estructura del sensor, la pieza amarilla sostiene el sensor, la pieza verde sujeta el cable. La pieza frontal se imprime en TPU, donde la superficie de contacto es de solo un par de capas de material." class="align-center" %}

En lugar de emplear silicona líquida convencional (RTV), se utilizó una pistola termofusible. Aunque coloquialmente se las denomina “pistolas de silicona”, las barras empleadas están compuestas en realidad por polímeros termoplásticos (habitualmente EVA u otros copolímeros). No obstante, debido a la similitud en la consistencia final del material solidificado y su facilidad de procesamiento, se decidió evaluar esta alternativa como solución práctica y de bajo costo. 

{% include figure image_path="/assets/images/tonometro/molde_A.png" alt="Molde para cono de gel" caption="Molde impreso en PLA fijado para el colado del cono de gel. Se puede ver el orificio de venteo en la parte superior, para asegurar el llenado completo de la cavidad del cono. Además se colocó papel de aluminio en la entrada del molde, a fin de proteger el PLA de la pistola termofusible." class="align-center" %}

Fueron necesarios múltiples prototipos, tanto del molde como del cono de gel, para optimizar la geometría y lograr un buen "colado". La estructura del resto del dispositivo también pasó por varias iteraciones, buscando una geometría que permitiera sujetar el sensor de forma segura y que a la vez fuera cómoda de utilizar.

{% include figure image_path="/assets/images/tonometro/prototipos.jpg" alt="Evolución de prototipos" caption="Diferentes iteraciones de los moldes y la estructura del dispositivo, se buscaba optimizar funcionalidad y facilidad en el armado." class="align-center" %}

## Pruebas

En el video podemos ver la utilización del prototipo conectado al BioAmp y visualizando la señal de presión en el software BrainBay. Dada la versatilidad de este software es posible armar un diagrama de flujo para procesar la señal y obtener la presión arterial directamente en mmHg.

<p align="center">
  <video controls width="80%">
    <source src="/assets/images/tonometro/video_test.mp4" type="video/mp4">
  </video>
</p>

Si además se suma la captura de la señal de ECG, se obtienen registros como el siguiente:

{% include figure image_path="/assets/images/tonometro/ECG_presion.png" alt="Registro ECG y Presión" caption="Captura simultánea de ECG y curva de presión arterial radial obtenida con el prototipo final." class="align-center" %}

El rediseño permitió obtener una señal de presión con morfología compatible con una onda de pulso radial fisiológica, evitando las limitaciones dinámicas del sistema hidráulico inicial. La integración con el BioAmp demostró la viabilidad de registrar simultáneamente ECG y presión arterial mediante un transductor no invasivo de bajo costo.

Si bien el sistema aún requiere calibración formal contra un método clínico de referencia, los resultados preliminares validan el concepto de tonometría radial integrada y abren la puerta al desarrollo de herramientas para la estimación de rigidez arterial y parámetros hemodinámicos derivados.