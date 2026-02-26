---
title: "Tonometro de pulso"
author: Juan I. Cerrudo
date: 2026-02-05
categories:
  - Tutorial
tags:
  - tonometro
  - diseño
  - prototipado
layout: single
toc: true
toc_sticky: true
---

## Introducción

Hace unos años se acercó al Laboratorio de Prototipado un médico con la inquietud de poder medir la presión arterial utilizando un tonometro de pulso.
Un tonometro de pulso es un dispositivo que mide la presión sanguínea y las características de la onda de pulso directamente sobre una arteria superficial, generalmente en la muñeca. El senor aplana la arteria radial contra el hueso y mide la presión necesaria para mantenerla colapsada.
La idea era poder incorporar esta funcionalidad al BioAmp, de manera de poder medir simultáneamente la presión arterial y la actividad eléctrica del corazón. La medición simultánea de ECG y presión permite estimar parámetros como el tiempo de tránsito de pulso (PTT), correlacionado con la rigidez arterial y utilizado como indicador temprano de riesgo cardiovascular.

Inicialmente se intentó replicar un sistema, con el que ya estaban trabajando, que implicaba el uso de un sensor de presion invasivo. Este es un dispositivo médico diseñado para la monitorización continua y precisa de la presión arterial y otros parámetros fisiológicos. Funciona mediante la inserción de un catéter directamente en un vaso sanguíneo (habitualmente una arteria), el cual transmite la presión a un transductor externo que convierte la señal mecánica en una señal eléctrica. Las tubuladuras del sensor se llenaban con agua, el extremo del catéter era modificado con una membrana flexible (dedo de guante de latex o un globo) y se colocaba en la muñeca del paciente. La presión arterial movia la membrana y con ella la columna de agua, permitiendo medir la presión arterial. El sensor se conectaba a uno de los canales diferenciales del BioAmp, la salida tipo puente de Wheatstone generaba una diferencia de potencial proporcional a la presión ejercida que podia ser monitoreada junto con la señal de ECG.
Este sistema no resulto muy efectivo, al menos en nuestra implementacion, con dificultades para el llenado de las tubuladuras y problemas para evitar burbujas de aire dentro de las mismas. Ademas, desde el punto de vista dinámico, el sistema hidráulico introducía una compliancia elevada y un comportamiento fuertemente amortiguado, lo que degradaba la fidelidad de la forma de onda de presión y reducia la sensibilidad del sensor.


## Rediseño

Decidimos rediseñar el sistema desde cero, modificando el transductor, buscando algo más facil de utilizar. Durante la investigacion bibliografioa que emprendemos cuando llevamos adelante un proyecto, una de las fuentes de busqueda son patentes. Una que nos resulto particularmnente interesante es la patente [US20050177047A1](https://patents.google.com/patent/US20050177047A1/en); "Device for, and a method of, transcutaneous pressure waveform sensing of an artery and a related target apparatus".

![US20050177047A1-20050811-D00000](/assets/images/tonometro/US20050177047A1-20050811-D00000.png){: .align-center}

En la imagen del dispositvio se podia observar el sensor de presion, una suerte de cono de gel que transmitia la presion arterial desde la superficie exterior al sensor. Ya estabamos en la busqueda de sensores de presion invasiva para evaluar su reutilizacion, y de la imagen nos parecio reconocer el encapsulado del sensor utilizado, si bien no estamos 100% seguros se parece mucho al [MPX2300DT1](https://www.nxp.com/docs/en/data-sheet/MPX2300D.pdf) de NXP 

![MPX2300DT1](/assets/images/tonometro/MPX2300DT1.png){: .align-center}

Este sensor es un transductor de presión de silicio de 0 a 2.3 MPa (0 a 23 bar) con salida de voltaje compensada en temperatura, utilizado en dispositivos de monitoreo de presión invasiva.  


## Modelado CAD

Comenzamos el modelado CAD del dispositivo con el objetivo de reproducir la geometría descripta en la patente. Para el cono de gel se optó por utilizar silicona, diseñando y fabricando previamente un molde en PLA mediante impresión 3D. El molde se fijaba en una morsa para asegurar estabilidad durante el proceso de colado.

![corte_a](/assets/images/tonometro/corte_a.png){: .align-center}

En lugar de emplear silicona líquida convencional (RTV), se utilizó una pistola termofusible. Aunque coloquialmente se las denomina “pistolas de silicona”, las barras empleadas están compuestas en realidad por polímeros termoplásticos (habitualmente EVA u otros copolímeros). No obstante, debido a la similitud en la consistencia final del material solidificado y su facilidad de procesamiento, se decidió evaluar esta alternativa como solución práctica y de bajo costo. 

![molde](/assets/images/tonometro/molde_A.png){: .align-center}

Fueron necesarios multiples prototipos, tanto del molde como del cono de gel, para optimizar la geometría y logragr un buen "colado". La estructura del resto del dispositivo también pasó por varias iteraciones, buscando una geometría que permitiera sujetar el sensor de forma segura y que a la vez fuera cómoda de utilizar.

![prototipos](/assets/images/tonometro/prototipos.jpg){: .align-center}

## Pruebas

En el video podemos ver la utilizacion del prototipo conectado al BioAmp y visualizando la señal de presión en el software BrainBay. Dada la versatilidad de este zoftware es posible armar un diagrama de flujo para procesar la señal y obtener la presión arterial directamente en mmHg.

<p align="center">
  <video controls width="80%">
    <source src="/assets/images/tonometro/video_test.mp4" type="video/mp4">
  </video>
</p>

Si además se suma la captura de la señal de ECG, se obtienen registros como el siguiente:

![ECG_presion](/assets/images/tonometro/ECG_presion.png){: .align-center} 

El rediseño permitió obtener una señal de presión con morfología compatible con una onda de pulso radial fisiológica, evitando las limitaciones dinámicas del sistema hidráulico inicial. La integración con el BioAmp demostró la viabilidad de registrar simultáneamente ECG y presión arterial mediante un transductor no invasivo de bajo costo.

Si bien el sistema aún requiere calibración formal contra un método clínico de referencia, los resultados preliminares validan el concepto de tonometría radial integrada y abren la puerta al desarrollo de herramientas para la estimación de rigidez arterial y parámetros hemodinámicos derivados.