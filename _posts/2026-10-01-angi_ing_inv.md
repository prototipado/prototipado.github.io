---
title: "Ingeniería Inversa del sensor ANGi de Specialized con herramientas de IA - un caso de estudio"
author: Juan I. Cerrudo
date: 2026-10-01
lang: es
page_id: angi-inv
permalink: /desarrollos/angi_ing_inv/
categories:
  - Desarrollos
tags:
  - Hardware
  - BLE
  - Android
  - Kotlin
  - nRF52
  - Inteligencia Artificial
layout: single
toc: true
toc_sticky: true
excerpt: "Ingeniería inversa del sensor Specialized ANGi: reconstrucción del protocolo BLE, desarrollo de apps en Android, análisis del firmware del nRF52 y caracterización eléctrica con herramientas de IA."
---

El ciclismo es una de mis actividades favoritas fuera del trabajo. Cerca de Oro Verde hay rutas que llevan a aldeas y pueblos, con poco tráfico y muchísima gente que las aprovecha para pedalear. El casco es obligatorio y, cuando compré el mío a principios de 2024, [Specialized](https://www.specialized.com/ar/es) ofrecía el sensor [ANGi](https://www.dcrainmaker.com/2018/11/specialized-helmet-sensor.html): un detector de caídas que se conecta al celular y, ante un golpe o movimiento brusco, envía SMS con aviso y ubicación a los contactos que uno configuró. 

Este año Specialized dio de baja la app que lo soportaba y migró a una nueva que no es compatible. Además, los servidores dejaron de responder: no se puede iniciar sesión, recuperar la contraseña ni crear una cuenta, así que estoy bloqueado de seguir usando la app vieja. Me quedé con un dispositivo útil que no puedo usar, y con una pregunta incómoda: ¿realmente era mío? 

### Estructura del documento

Este caso de estudio documenta el proceso completo de ingeniería inversa sobre el sensor ANGi y el desarrollo de una alternativa abierta e independiente (*OpenAngi*) utilizando herramientas de IA. El trabajo está organizado en 5 partes principales:

- **[Parte 1: Claude y la exploración BLE](#parte-1-claude-y-la-exploración-ble)** — Exploración inicial del hardware, enumeración de servicios BLE y descompilación estática del APK original.
- **[Parte 2: OpenCode y el desarrollo de las apps Android](#parte-2-opencode-y-el-desarrollo-de-las-apps-android)** — Construcción del banco de pruebas *AngiTester*, desarrollo de la app final *OpenAngi* (100% independiente de servidores) y comparativa de código.
- **[Parte 3: Hardware y Firmware — Análisis Comparativo](#parte-3-hardware-y-firmware--análisis-comparativo-antigravity-vs-opencode)** — Extracción del firmware del nRF52832, análisis en Ghidra, descubrimiento de la consola UART de producción y caracterización de consumo eléctrico.
- **[Parte 4: Actualización DFU](#parte-4-actualización-dfu)** — Localización del firmware oficial dentro de la APK, actualización por Bluetooth (DFU) e integración del proceso en la app.
- **[Parte 5: Balance del uso de herramientas de IA](#parte-5-balance-del-uso-de-herramientas-de-ia)** — Comparativa de fortalezas y limitaciones de cada modelo, escala/costos del proyecto y consideraciones legales.

Una aclaración inicial: todas las herramientas usadas son de acceso gratuito, por ello se puede ver el uso de diversas en distintos puntos. En algunos casos respondía a intentar herramientas diferentes, limitaciones propias de los modelos (por ejemplo en OpenCode muchos no soportan imágenes), y en otros al hecho de haber llegado a los límites de tokens disponibles.

---

## Parte 1: Claude y la exploración BLE

La primera fase la hicimos enteramente en Claude (Sonnet 5) a través de la interfaz web de claude.ai. Una única sesión continua, un solo modelo, sin interrupciones. 

La primera pregunta fue de viabilidad del proyecto que queríamos encarar. Claude buscó en la web y confirmó que el ANGi corre sobre un [nRF52832](https://www.nordicsemi.com/Products/nRF52832) de Nordic Semiconductor (Cortex-M4F, 512KB flash) — un chip BLE extremadamente documentado con SDK público, identificado a partir del [registro FCC](https://fccid.io/OXXANGI001) del dispositivo.

Como la app oficial estaba muerta (sin login, sin recuperación de contraseña, sin altas nuevas), descartamos capturar tráfico logueado y reordenamos el plan hacia dos frentes paralelos:
1. **Exploración directa del hardware** con nRF Connect for Mobile (herramienta gratuita de Nordic), enumerando servicios y características BLE sin pasar por la app de Specialized.
2. **Decompilación estática del APK** descargado de un mirror (apkmirror.com, ya que Google Play tampoco permitía la descarga) usando [jadx](https://github.com/skylot/jadx) — el código decompilado no depende de que el backend esté vivo.

Conectamos el sensor usando nRF Connect y le pasamos las primeras capturas de pantalla para análisis. De allí salió el mapa inicial de servicios: el servicio custom `afd0ffa0…afd0ffa8`, Battery Service, Device Information, y un servicio de DFU "Buttonless" de Nordic.

### a. Construcción incremental de scripts de exploración

Con Claude fuimos construyendo incrementalmente una serie de scripts en Python usando la librería [bleak](https://bleak.readthedocs.io/en/latest/) (cliente BLE multiplataforma), donde cada script respondía una pregunta concreta que dejaba abierta el anterior:

- `angi_sniffer.py`: se conecta al sensor, se suscribe a las características NOTIFY, y loguea todo el tráfico crudo con timestamp en pantalla y CSV.
- `angi_command_sweep.py`: barre automáticamente los 256 valores posibles (`0x00–0xFF`) en las características WRITE, comparando el registro de STATUS antes y después de cada escritura para detectar reacciones.
- `angi_heartbeat_test.py`: escucha pasiva sin escribir nada, diseñado específicamente para descartar la hipótesis de que ciertos paquetes observados fueran respuestas a comando. Resultado: eran un heartbeat periódico (un contador de uptime incrementando ~1/segundo).

La dinámica era siempre la misma: corríamos el script contra el sensor real → le pegábamos el log o el traceback a Claude → él ajustaba el código → repetir. Simple pero efectivo.

De esta forma avanzábamos pero el proceso era más bien lento. El salto grande vino con la descompilación. Claude nos armó un prompt específico y detallado para ejecutar en [OpenCode](https://opencode.ai/) (otra herramienta de IA, con acceso directo al filesystem), con instrucciones paso a paso: decompilación con jadx, búsqueda dirigida (grep) de los UUIDs y constantes ya identificados por BLE, identificación de clases y métodos por cada UUID, y generación de un documento de protocolo.

El resultado fue un mapa completo del protocolo extraído de las clases Java decompiladas. Con esa información se construyó el `angi_client.py` definitivo: un cliente BLE interactivo y completo que implementa el patrón real de comando extendido del sensor, parsers para los tres formatos de paquete de respuesta (ASCII, reporte IMU de 15 bytes, reporte de impacto de 20 bytes), comandos con nombre, modo raw `<hex>` para experimentación libre, y un autotest que barre los 17 opcodes conocidos × 5 variantes de parámetro con reconexión automática.

### b. Identificación de hardware y sensores en placa

Otra de las líneas de investigación fue por el hardware en sí: abrimos el sensor y le sacamos fotos a la placa. 

{% include figure popup=true image_path="/assets/images/sensor_angi/top_angi.JPG" alt="Vista superior de la placa del sensor ANGi" caption="Vista superior de la PCB del sensor ANGi, con el SoC nRF52832 en encapsulado BGA, sensores y el soporte para pila botón CR2032. La PCB está pegada al gabinate, al sacarla se debe haber flexionado lo suficiente como para cortar algunas pistas, se puede ver un jumper soldado puentenado dos punto sde VCC, necesario para que vuelva a funcionar." class="align-center" %}

{% include figure popup=true image_path="/assets/images/sensor_angi/bottom_angi.JPG" alt="Vista inferior de la placa del sensor ANGi" caption="Vista inferior de la PCB con los puntos de prueba (Test Points)." class="align-center" %}

Claude identificó los chips por su marcado, cruzando contra documentación pública de los fabricantes:

- **Acelerómetro ST LIS3DH**: identificado por la marca "C3H", confirmada contra un post oficial de un ingeniero de STMicroelectronics en el foro de la compañía.
- **Giroscopio ST L3GD20H**: identificado por la marca "3GDH", verificado contra la misma fuente de ST.
- **BLE SoC nRF52832**: en encapsulado BGA bajo una capa de silicona protectora.



{% include figure popup=true image_path="/assets/images/sensor_angi/foto_accel_gyro.JPG" alt="Detalle de los sensores en la placa" caption="Detalle de los sensores STMicroelectronics identificados en la placa: acelerómetro LIS3DH ('C3H') y giroscopio L3GD20H ('3GDH')." class="align-center" %}

### c. Números de la sesión con Claude

| Métrica | Valor |
| :--- | :--- |
| **Modelo** | Claude Sonnet 5 (único, toda la sesión) |
| **Interfaz** | claude.ai (chat web) |
| **Turnos de conversación** | 26 |
| **Llamadas a herramientas** | ~54 (conteo manual) |
| **Imágenes analizadas** | 11 (capturas nRF Connect, fotos de PCB, logos) |
| **Documentos subidos** | 4 |
| **Archivos Python creados** | 4 |
| **Costo** | Plan de suscripción de Claude |

A partir de este punto, teniendo la APK original decompilada, pasamos a OpenCode para continuar el desarrollo de las aplicaciones de prueba y la solución final.

---

## Parte 2: OpenCode y el desarrollo de las apps Android

La segunda fase —construir las apps Android funcionales— la hicimos con OpenCode, un IDE asistido por IA que corre en terminal con acceso directo al filesystem, ejecución de comandos (bash/PowerShell), y herramientas de edición de código. Todo lo que sigue sale de datos reales de la base de datos local de OpenCode (SQLite), no de estimaciones.

OpenCode usó 3 modelos distintos a lo largo de la sesión, todos del tier gratuito:

| Modelo | Periodo | Msgs | Tokens entrada | Tokens salida | Razonamiento |
| :--- | :--- | :--- | :--- | :--- | :--- |
| ling-3.0-flash-fin | 26/09 (arranque) | 231 | 1.859.556 | 85.643 | 39.546 |
| longcat-2.5-preview | 28/09 (exploración) | 24 | 312.099 | 14.417 | 55.808 |
| mimo-v2.6-flash | 28/09–29/09 (grueso del dev) | 483 | 2.171.889 | 218.998 | 256.083 |

 738 mensajes del asistente, 4.3 millones de tokens de entrada, 319K de salida, 351K de razonamiento. El paso por los distintos modelos respondió más a los resultados que obteníamos: en algunos casos notábamos que el modelo comenzaba a tardar más o no lograba resolver los problemas, posiblemente por un contexto elevado en la sesión, por lo que intentábamos con otro modelo.



---

### a. Desarrollo con AngiTester

Antes de construir la app final (OpenAngi), necesitamos contar con un "banco de pruebas de laboratorio" en Android. Así nació **AngiTester** — una aplicación ligera de 6 archivos [Kotlin](https://kotlinlang.org/) cuyo único objetivo era interactuar directamente con el sensor real SHS-D33F y validar cada hipótesis byte a byte.

La pantalla principal de AngiTester se divide en dos zonas clave para inspeccionar el tráfico BLE en tiempo real:

- **Terminal de comandos en vivo (`log [HH:mm:ss.SSS]`)**:
  - Caja tipo consola con auto-scroll inteligente (se pausa si el usuario sube a leer el historial y retoma al volver abajo).
  - Registra todo el tráfico bidireccional (TX/RX), notificaciones ASCII, respuestas de estado y lectura de batería en tiempo real cada 5 minutos.
- **Gráfico custom IMU en tiempo real (`ImuGraphView.kt`)**:
  - Grafica a 5 Hz los 3 ejes del acelerómetro y la velocidad angular en modo TEST.
  - Indexado por número de muestra (`sampleNumber`): evita distorsiones o "saltos" horizontales cuando las notificaciones BLE llegan en ráfagas.
  - Auto-escala Y suavizada para prevenir que un pico repentino aplane la curva continua.
- **Botonera de comandos directa y extendida**:
  - Modos directos (`afd0ffa1`): Activate (sincroniza RTC + MODE `0x01`), Normal (`0x00`), Test (`0x40` streaming IMU).
  - Comandos extendidos (`BLOCK_DATA afd0ffa4` + trigger `0x20`): versión de firmware (`0xC3`), test de acelerómetro/giroscopio (`0xC0/0xC1`), control de LEDs (`0xFE/0xC2`), simulación de impacto (`0xFB`), lectura/reporte de eventos grabados (`0x80/0x81`), entre otros.

AngiTester no fue solo una app de prueba: fue la herramienta empírica que permitió descifrar el comportamiento real del hardware y descubrir aspectos que no estaban claros en el código decompilado:

i. **Decodificación y validación de STATUS (`afd0ffa2`)**:
   Verificamos empíricamente la estructura del bit-field de 1 byte devuelto por el sensor:
   - **Bit 7 (`0x80`)**: Bit de ERROR (si es 1, los 7 bits restantes son el código de error).
   - **Bits 6-5 (`0x60`)**: Modo (`0`=NORMAL, `1`=EXT_CMD, `2`=TEST).
   - **Bit 4 (`0x10`)**: Sesión activa.
   - **Bit 3 (`0x08`)**: Heartbeat de conexión activo.
   - **Bit 0 (`0x01`)**: Último comando completado.

   Catalogamos los valores de estado crudos reales observados durante la operación: `0x00` (recién conectado), `0x01` (NORMAL + sesión activa), `0x03` (sesión activa con eventos pendientes), `0x21` (modo EXT_CMD completado) y `0x82` (ACK con error/transitorio tras comandos de impacto).

ii. **Descubrimiento y calibración de la escala del acelerómetro**:
   La APK original dividía todos los datos del reporte IMU (`IcedotTestReport`) por 100. Sin embargo, al capturar datos en reposo con AngiTester y procesar la evidencia (`angi_static.csv`), descubrimos que los ejes X, Y y Z daban un vector resultante de ~1,73 g, lo cual es físicamente imposible en reposo.

   Mediante pruebas cruzadas derivamos el divisor real para los ejes del acelerómetro:
   `Divisor K = 173.2 ≈ 100 · √3`
   Con esta corrección, los ejes reportaron exactamente ~1,00 g en reposo (error medio de solo 9 mg), mientras que `accelVector` y `rotation` se mantuvieron con divisor 100.

iii. **Confirmación del canal único del giroscopio**:
   Confirmamos que el firmware de Specialized no transmite los 3 ejes independientes del giroscopio en el streaming BLE. En su lugar, reduce la velocidad angular a un único canal escalar (`rotation`) en °/s (bytes 11-12) y su aceleración angular (`rotationalAccel`, bytes 13-14). Determinamos que corresponde a un eje de rotación de basculación (cercano al eje X del casco).

iv. **Validación del flujo de armado de sesión e impacto**:
   Comprobamos que si el RTC de 7 bytes UTC (`afd0ffa7`) no se envía antes de habilitar `MODE 0x01`, los reportes de impacto se registran con timestamp en 0.
   Capturamos y validamos la notificación real de impacto en `afd0ffa6` (paquete de 20 bytes: type, battery, version, timestamp, peakG, peakR, peakRA y deviceID).

<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/sensor_angi/angi_tester.mp4" type="video/mp4">
    Tu navegador no soporta la reproducción de video.
  </video>
  <figcaption>AngiTester: ejecución de la aplicación.</figcaption>
</figure>

---

### b. OpenAngi: la app final independiente

**OpenAngi** es la aplicación definitiva (30 archivos Kotlin, APK de 16.5 MB): una reescritura completa desde cero usando arquitectura moderna en Kotlin + Jetpack Compose Material 3 + MVVM + Room + DataStore + Foreground Service. 

La app funciona **sin servidor, sin cuentas y sin backend**: el propio teléfono manda los SMS directamente mediante `SmsManager`.

**Las pantallas de OpenAngi:**

La app tiene 4 pestañas principales en una barra inferior, más dos pantallas superpuestas (Dispositivo y Alerta):

| Pantalla | Qué hace |
| :--- | :--- |
| **Escanear** | Lista los sensores ANGi cercanos (filtrado por servicio BLE, fallback por nombre SHS-), muestra RSSI de señal. Tocar uno → conecta. Si ya hay sesión activa, muestra un banner de "Escaneo pausado" para no interferir con la radio. |
| **Dispositivo** | El "tablero" principal: pulso animado "Sensor activo y escuchando", batería del sensor, firmware, hardware, status crudo, sesión activa, latido del sensor (heartbeat avanzando), avisos de impacto recibidos (contador de diagnóstico). Botones: Desconectar, Simular impacto, y Declarar emergencia. |
| **Alerta** | Pantalla de emergencia: cuenta regresiva gigante (configurable de 5 a 120 segundos), pico de G del impacto, botón grande "ESTOY BIEN — CANCELAR". Si no cancelás, al terminar muestra confirmación con los SMS enviados y la ubicación. Aparece como notificación full-screen si la app está en background. |
| **Contactos** | Alta de contactos de emergencia (manual o desde el selector del teléfono), edición y baja. Al agregar un contacto, le manda automáticamente un SMS de aviso: "Te han agregado como contacto de emergencia en OpenAngi". |
| **Historial** | Todos los eventos registrados: fecha, chip de estado (cancelado/notificado/fallido), picos de G, ubicación (tap → abre Google Maps). Botón de limpiar (solo cancelados/fallidos, o todo). |
| **Ajustes** | Duración del countdown, sonido/vibración, tu nombre (para personalizar el SMS), simular impacto, declarar emergencia, estado de exclusión de batería, y el disclaimer legal completo. |

{% include figure popup=true image_path="/assets/images/sensor_angi/pantallas.png" alt="Pantallas de la App OpenAngi" caption="Capturas de pantalla de la aplicación OpenAngi para Android, sin servidor y con envío directo de SMS." class="align-center" %}

**Flujo de conexión BLE**

{% include figure popup=true image_path="/assets/images/sensor_angi/flujo_conexion_ble.png" alt="Flujo de conexión BLE" caption="Diagrama del flujo de conexión BLE y sincronización implementado en OpenAngi." class="align-center" %}

Cada operación GATT pasa por un Mutex (una a la vez, con timeout de 8 segundos). Si algún paso falla, la reconexión automática repite todo el setup desde cero, porque el sensor pierde su estado al desconectarse. El backoff crece de 2 a 60 segundos con jitter.

**Flujo de impacto: de la detección al SMS**

Esta es la parte central de la app — la máquina de estados del `ImpactFlowController`:

{% include figure popup=true image_path="/assets/images/sensor_angi/flujo_impact_controller.png" alt="Diagrama de estados de impacto" caption="Máquina de estados del ImpactFlowController para la detección de impacto y envío de emergencia." class="align-center" %}

El flujo corre a nivel de proceso (no de UI): si la app está cerrada pero el Foreground Service está corriendo, la detección funciona igual y la notificación full-screen abre la alarma directamente.

**Declarar emergencia:** hay un botón dedicado en la pantalla de Dispositivo y en Ajustes que dispara exactamente el mismo flujo pero con un evento sintético (tipo DECLARADA, pico 0 g). El SMS dice *"declaró una emergencia (aviso manual)"* en vez de *"sufrió un impacto"*. Útil si necesitás pedir ayuda sin haber tenido un choque.

**Robustez: el Foreground Service**

El servicio `AngiMonitorService` es el dueño de la conexión BLE y corre con una notificación persistente que muestra el estado real:
- 🟢 **Verde:** conectado y vigilando (muestra batería del sensor)
- 🟡 **Ámbar:** reconectando (con cuenta de espera)
- 🔴 **Rojo:** desconectado

Si pasan 5 minutos sin conexión, la app lanza una notificación de alta prioridad: *"OpenAngi perdió conexión con el sensor"*. La filosofía es nunca fallar en silencio — si algo sale mal, que el usuario se entere.

El `BootReceiver` reconecta automáticamente al prender el teléfono, y si Android bloquea el arranque en background, al menos manda una notificación avisando que hay que abrir la app manualmente.


<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/sensor_angi/open_angi.mp4" type="video/mp4">
    Tu navegador no soporta la reproducción de video.
  </video>
  <figcaption>OpenAngi: ejecución de la aplicación. Los avisos no son inmediatos, se van enconlando, se puede ver un segundo aviso Cancelado por el usuario </figcaption>
</figure>


---

### c. Diferencia con la app original: hechos de protocolo vs. código

Uno de los puntos más importantes de este proyecto es entender qué se tomó y qué no de la app original de Specialized, qué tanto las herramientas copiaron de la APK original y qué fue creación original. Corrimos scripts automatizados comparando las 262.495 líneas del APK decompilado contra todo el código nuevo:

| Chequeo | AngiTester | OpenAngi |
| :--- | :--- | :--- |
| Líneas idénticas al decompilado (≥25 chars) | 2 | 1 |
| Textos de UI compartidos | 0 | 0 |
| Recursos copiados (de 879 drawables) | 0 | 0 |

Las 3 líneas "idénticas" son boilerplate idiomático de Android (`if (bluetoothManager != null)`, `if (cursor.moveToFirst())`). Los únicos literales compartidos son UUIDs BLE — datos funcionales indispensables para hablar con el sensor.

De la decompilación extrajimos hechos técnicos del protocolo — la información mínima indispensable para que cualquier app pueda hablar con el sensor:
- UUIDs de las características BLE (`afd0ffa0…ffa8`).
- Offsets y largos de los paquetes: el reporte de impacto son 20 bytes, el RTC son 7 bytes, el reporte IMU son 15 bytes.
- Valores de byte: modos (`0x00` NORMAL, `0x01` ACTIVE, `0x20` EXT_CMD, `0x40` TEST), tipos de reporte (1=LINEAR, 2=ROTATIONAL, 3=LINEARROTATIONAL), bits de STATUS.
- Códigos de error numéricos (33=SUCCESS, 64=INVALID_PARAM).
- Secuencia de activación: escribir RTC + MODE `0x01`.

Todo esto está documentado y citado en los comentarios del código. Son datos funcionales —como documentar que "el puerto HTTP es el 80"—, no expresión creativa copiada.


| Aspecto | App original (Specialized Ride) | OpenAngi |
| :--- | :--- | :--- |
| **Namespace** | `com.specialized.ride` | `com.openangi.app` |
| **Nombres de clases** | `IcedotReport`, `IcedotBLE` | `ImpactReport`, `AngiProtocol` |
| **Parseo de paquetes** | `==20` con excepción (`IcedotException`) | `≥16` tolerante con `parseOrNull` |
| **Arquitectura** | Código legacy Java | Kotlin + Compose M3 + MVVM + Room |
| **Alertas de emergencia** | Backend propio → `POST /events` → servidor manda SMS | SMS directo con `SmsManager` (sin servidor) |
| **UI / Recursos** | Layouts XML, drawables de Specialized | Compose puro, tema oscuro propio, ícono original |
| **Backend / URLs** | `api.prod.s-digital.co` | Ninguno — cero dependencias externas |

El cambio de arquitectura más significativo es el de las alertas: la app original dependía de un backend de Specialized que recibía el evento por API y mandaba el SMS desde su servidor. Cuando Specialized dio de baja el servicio, eso dejó de funcionar. OpenAngi elimina esa dependencia por completo: el teléfono manda el SMS directamente, sin pasar por ningún servidor. Es más simple, más confiable, y no depende de que nadie mantenga un backend vivo.

En resumen: del APK original tomamos los datos del protocolo (UUIDs, formatos de paquete, secuencia de activación) —que son hechos técnicos necesarios para interoperar con el hardware— y construimos una app completamente nueva, con arquitectura distinta, UI distinta, y un mecanismo de alertas fundamentalmente diferente.

---

### d. Balance global del desarrollo de software

Los números del proyecto completo reflejan la dimensión del trabajo asistido por IA:

| Métrica | Claude | OpenCode | Total |
| :--- | :--- | :--- | :--- |
| **Modelos usados** | 1 | 3 | 4 |
| **Turnos / prompts del usuario** | 26 | 75 | 101 |
| **Mensajes del asistente** | 26 | 738 | 764 |
| **Llamadas a herramientas** | ~54 | 1.067 | ~1.121 |
| **Archivos de código creados** | 4 (Python) | 36+ (Kotlin) | 40+ |
| **Costo** | Suscripción Claude | $0 (tier gratuito) | — |
| **Periodo** | 1 sesión | 3 días (26–29/09) | ~3 días |

En el cuadro se observa un detalle de las herramientas utilizadas en OpenCode:

| Herramienta | Llamadas | Descripción |
| :--- | :--- | :--- |
| bash | 378 | Gradle, Python, git, apksigner, comandos del sistema |
| edit | 268 | Ediciones quirúrgicas de código (search & replace) |
| read | 241 | Lectura de archivos del proyecto y del decompilado |
| write | 103 | Creación de archivos nuevos |
| grep | 54 | Búsqueda de patrones en el código |
| glob | 11 | Búsqueda de archivos por nombre |
| todowrite | 6 | Lista de tareas pendientes |
| MCP codebase-memory | 4 | Grafo de conocimiento del codebase |
| task (subagente) | 1 | Delegación a un agente hijo |

Además: 720 trazas de razonamiento, 256 respuestas de texto, y 8 re-compactaciones de contexto (cuando la ventana de contexto se llena, el modelo comprime el historial y sigue).

**¿Qué aprendimos del proceso?**

La IA no reemplazó el trabajo — nosotros seguimos siendo los que conectábamos el sensor, corríamos los scripts, sacábamos las fotos, abríamos el dispositivo, decompilábamos el APK, y tomábamos las decisiones de diseño. Pero aceleró brutalmente cada paso intermedio: en vez de buscar datasheets, leer foros, y escribir parsers de bytes a mano, la conversación con los modelos nos permitió ir de "¿qué es este chip?" a "tenemos un APK firmado y funcionando" en 3 días.

Y es incluso una de las líneas de investigación que estamos incorporando al PID: lograr cerrar el lazo, dar acceso a los LLMs al hardware y a la interacción con herramientas físicas que permitan acelerar el proceso de experimentación.

La combinación de herramientas funcionó bien: 
- **Claude** (chat web con visión, búsqueda y análisis de imágenes) fue ideal para la fase exploratoria: identificar chips por fotos, armar el plan de ataque y construir scripts incrementales.
- **OpenCode** (acceso al filesystem, ejecución de comandos en bash/PowerShell y edición de código) fue ideal para la fase de construcción: compilar, corregir errores de Gradle e iterar sobre 30 archivos Kotlin hasta tener un APK funcional. 

Y todo el desarrollo de las apps Android —738 mensajes, 4.3 millones de tokens de entrada, más de mil llamadas a herramientas— costó exactamente cero dólares usando modelos gratuitos.

---

## Parte 3: Hardware y Firmware — Análisis Comparativo (Antigravity vs. OpenCode)

Además de la ingeniería inversa sobre la app de Android y el protocolo BLE, el proyecto avanzó sobre el firmware binario extraído de la memoria flash del microcontrolador nRF52832 de Nordic Semiconductor (SoC ARM Cortex-M4F que impulsa el sensor ANGi).

Tanto en las fotos publicadas en el sitio de la FCC como en el dispositivo físico abierto, pudimos identificar un conector de debug de 6 pines del tipo TAG-Connect, específicamente el [TC2030](https://www.tag-connect.com/product/tc2030-idc-nl). Nos fue fácil identificar qué pines corresponden a GND y VCC ya que ambos respetaban el pinout estándar del conector, por lo que era de esperar que tanto SWCLK como SWDIO también coincidieran con la distribución estándar. 


{% include figure popup=true image_path="/assets/images/sensor_angi/diy_tag_connect.png" alt="Adaptador casero TAG-Connect TC2030" caption="Adaptador casero tipo TAG-Connect TC2030 fabricado en el laboratorio mediante mecanizado CNC en placas de FR4 y pogo-pins." class="align-center" %}


Los conectores TAG-Connect comerciales son costosos (el TC2030 ronda los 34 USD). En el laboratorio fabricamos una versión propia: piezas de FR4 maquinadas con el CNC, pogo-pins y alambre de clips para los pines de posición:

{% include figure popup=true image_path="/assets/images/sensor_angi/tag_connect.jpg" alt="Lectura del firmware con el J-Link EDU" caption="Extracción del dump completo de la memoria flash del nRF52832 utilizando nRF Connect Programmer." class="align-center" %}

Utilizamos la herramienta de nRF Connect junto con un J-Link EDU para extraer el firmware del microcontrolador; afortunadamente no estaba bloqueada la lectura:

{% include figure popup=true image_path="/assets/images/sensor_angi/nrf_dump.png" alt="Lectura del firmware con nRF Connect" caption="Extracción del dump completo de la memoria flash del nRF52832 utilizando nRF Connect Programmer." class="align-center" %}

### a. Extracción y decompilación base con Claude y Ghidra

Con el dump completo extraído (`nRF_Connect_Programmer_1790785304145.hex`), la primera tarea fue interpretar la estructura de la memoria flash del nRF52832 y preparar un flujo de decompilación automatizado. Esta fase se trabajó utilizando Claude (Sonnet 3.5) a través del chat web.

Claude analizó el archivo Intel HEX y mapeó con la arquitectura de memoria del dispositivo:
- `0x00000–0x00963`: MBR (Master Boot Record de Nordic)
- `0x01000–0x1E73F`: SoftDevice S132 v3.0.0 (stack BLE oficial de Nordic)
- `0x1F000–0x325DB`: Aplicación ANGi (79.324 bytes, con punto de entrada Reset_Handler en `0x1F415`)
- `0x70000–0x72B67`: Datos persistentes en flash (Flash Data Storage / FDS)
- `0x73000–0x7B243`: Bootloader DFU (33 KB)
- `0x7F000`: Página de configuración del bootloader (donde el CRC32 `0x01092589` coincidía exactamente con el hash de la app).

Para convertir este firmware binario crudo en código C navegable, se construyó junto a Claude una canalización de ingeniería inversa headless sobre Ghidra, escribiendo dos scripts en Java:
- `SetupNrf52.java` (Pre-script): Configura automáticamente el entorno de análisis ARM Cortex-M4F en Ghidra. Mapea la RAM (`0x20000000`) y las regiones de registros periféricos del nRF52 (NRF_RADIO, NRF_GPIOTE, NRF_SAADC, RTC, TIMER, etc.), analiza la tabla de vectores e instancia los handlers de interrupción (Reset_Handler, SVC_Handler, *_IRQHandler).
- `DecompileAll.java` (Post-script): Recorre todas las funciones identificadas por el motor de Ghidra y vuelca el código decompilado completo junto a las cadenas de texto (strings) a un único archivo C unificado.

Este proceso permitió obtener el código decompilado original base (`app_decompiled.c`), desentrañando las llamadas al SoftDevice (supervisión SVC 0xA0-0xAD para GATTS) y los literales del servicio BLE del sensor.

Una vez obtenido este archivo C crudo, la pregunta clave era: **¿hasta dónde puede llegar un modelo de lenguaje analizando este decompilado sin conocimiento previo del dispositivo?**

Para responder esto, diseñamos un experimento comparativo entregándole exactamente el mismo prompt inicial y los mismos archivos base a dos herramientas asistidas por IA con entornos y agentes distintos: **Antigravity** (Gemini 3.6 Flash) y **OpenCode** (big-pickle). Como pistas físicas mínimas les proporcionamos:
- La presencia de conectividad BLE (stack SoftDevice de Nordic).
- Los sensores identificados en la PCB: el acelerómetro ST LIS3DH y el giroscopio ST L3GD20H.
- La presencia de dos LEDs de indicación.
- La lectura del voltaje de la batería a través de un canal del conversor analógico-digital (ADC / SAADC).

---

### b. Resumen comparativo: Antigravity vs. OpenCode

A partir del mismo prompt de partida, las dos herramientas abordaron el desafío con filosofías y resultados marcadamente diferentes pero complementarios:

| Métrica / Característica | Antigravity (Gemini 3.6 Flash) | OpenCode (big-pickle) |
| :--- | :--- | :--- |
| **Objetivo efectivo** | Entender el dispositivo y controlarlo en hardware | Documentar y estructurar el decompilado entero |
| **Entregable de código C** | `app_annotated.c` (26 KB, solo prototipos y comentarios) | `app_decompiled_annotated.c` (1,9 MB, 77.793 líneas decompiladas) |
| **Funciones renombradas** | No cuantificado explícitamente en el C | 83 nombres semánticos documentados |
| **Consola UART de producción** | Activada y probada: Pin de entrada, baudrate, comandos y TPs | Tabla de comandos reconstruida (sin pin, baudrate ni TPs) |
| **Verificación** | En hardware real (medición física sobre el sensor) | Mediante scripts automáticos sobre el fichero decompilado |
| **Arquitectura identificada** | Mapa de periféricos y tareas de usuario | Máquina de estados por banderas de evento y bus de mensajes |
| **Límites declarados** | Sin sección de limitaciones | Sección explícita con 6 puntos pendientes de investigación |

---

### c. Logros principales de cada herramienta

#### Antigravity: Dominio del hardware y consola UART

El mayor éxito de Antigravity fue descifrar cómo habilitar la consola de diagnóstico de producción por UART oculta en el firmware. Para activarla determinó que era necesario forzar el pin `P0.30` a GND durante el encendido, operando 9600 o 115200 baudios en 8N1 e identificando que los puntos de prueba corresponden a TXD en TP10 y RXD en TP12, junto con un repertorio de 11 comandos de diagnóstico. 

#### OpenCode: Decompilación y arquitectura de firmware

Por su parte, OpenCode generó el análisis más profundo de la estructura interna del código binario. Produjo 77.793 líneas de C en `app_decompiled_annotated.c`, asignando firmas y parámetros documentados a 83 funciones clave. Asimismo, descubrió la máquina de estados central basada en banderas de eventos (`evt_set` / `evt_clear`), tareas consumidoras y un bus de mensajes de 24 bytes. Finalmente, trazó el mapa completo de la memoria Flash (MBR, SoftDevice BLE, código de aplicación, Bootloader y UICR), fechando la compilación original del firmware en el 6 de octubre de 2021.

---

### d. Descubrimiento de la consola UART y validación en placa real

La identificación de los test points (TP) en la PCB fue un proceso mucho más artesanal: con tester y osciloscopio fuimos identificando las señales físicas. GND y VCC resultaron fáciles de ubicar a partir del holder de la pila. 

Una vez confirmada la existencia de la UART y del pin de consola, la tarea fue encontrarlos en la PCB. Para ello observamos la actividad de todos los TP al energizar el dispositivo e identificamos seis TP con señales de pulsos (posiblemente comunicación con los sensores), otros seis con señal en bajo y un único TP con señal en alto (3V).

{% include figure popup=true image_path="/assets/images/sensor_angi/test_points.png" alt="Test points en la PCB de ANGi" caption="Test points identificados en la PCB de ANGi. TM es el pin P0.30 (Modo Test)" class="align-center" %}

En un principio esperábamos identificar al menos dos TP en alto, correspondientes a la UART en estado de reposo (idle). Al consultar con Antigravity, el modelo explicó que en modo normal la UART no se configura para no consumir energía, y solo se habilita al arrancar con nivel bajo en `P0.30`. Esto nos llevó a pensar que el único TP en alto debería ser el pin de modo test (`P0.30`) con una pull-up.

Le planteamos este escenario al modelo:

**👤 Usuario (step 258):**  
> *"Otra pregunta de los Test Points: hay uno solo a VCC, hay 6 en los que se ve algún tipo de señal/pulsos, y otros 6 que se ve señal bajo a GND. ¿Puede ser que la UART no se configure si no está en modo test? ¿Por qué esperaría ver dos pines más con Idle en 3V de la UART?"*

**🤖 Antigravity (step 259):**  
> *"¡Exactamente! Tu deducción es 100% correcta.
> 
> **1. ¿Por qué no ves la UART en 3V (Idle) en modo normal?**  
> En los microcontroladores Nordic nRF52, la UART no está asignada permanentemente a pines fijos por hardware....
> 
> **2. Explicación de las señales que estás midiendo actualmente (Modo Normal)**  
> - **1 Test Point en VCC (3V):** Alimentación principal de la batería / VDD.  
> - **6 Test Points con pulsos/señales:** Bus de comunicación por bit-banging de los sensores MEMS (LIS3DH + Giroscopio): Clock, MOSI, MISO, Chip Selects (CS). Interrupciones de impacto/movimiento (INT1 / INT2).  
> - **6 Test Points en GND:** Pines deshabilitados, entradas no usadas o pads de tierra del PCB (GND)."*

**👤 Usuario (step 260):**  
> *"No tengo acceso a los pines, el nRF52 es BGA. ¿Sería esperable que entonces el test point a VCC sea ese pin con una pull up?"*

**🤖 Antigravity (step 261):**  
> *"¡Sí, es ALTAMENTE probable! Esa es la arquitectura estándar de test de fábrica en placas con encapsulado BGA."*

Esta respuesta nos descolocó un poco, porque no logramos identificar qué fundamento había detrás de la relación entre el encapsulado BGA del nRF52 y la conclusión de que ese test point correspondía a una pull-up. Si bien le habíamos dado algunas pistas para evaluar si podía llegar al mismo razonamiento al que habíamos llegado nosotros, la conclusión del modelo parecía ir más allá de lo que permitían inferir el contexto disponible en ese momento.

Para probar la teoría, forzamos el estado bajo en `P0.30` durante el encendido: los LEDs titilaron con una secuencia diferente, confirmando el modo test. Al medir nuevamente con osciloscopio, dos test points que antes estaban en bajo pasaron a estar en nivel alto (idle de la UART).

**👤 Usuario (step 262):**  
> *"Lo puse a masa, al encender titilaron rojo y verde, y TP10 y TP12 ahora pasaron a VCC. ¿Deben ser TX y RX, no?"*

**🤖 Antigravity (step 263):**  
> *"¡EXCELENTE NOTICIA! 🎉 ¡Ingresaste con éxito al Modo Test de Fábrica!"*


Paso siguiente, conectamos un conversor serie-USB a ambos pines. Mediante prueba y error determinamos cuál de los TP era RX y cuál TX, y verificamos el baudrate real de **38.400 baudios**:

{% include figure popup=true image_path="/assets/images/sensor_angi/uart.jpg" alt="Conexión serie UART sobre la PCB" caption="Conexión física del conversor USB-Serial a los Test Points de la UART en la placa del sensor ANGi." class="align-center" %}

Con Realterm configurado a 38.400 baudios y el dispositivo en modo test, pudimos interactuar con la consola de producción y verificar el repertorio de comandos:

{% include figure popup=true image_path="/assets/images/sensor_angi/terminal.png" alt="Terminal serie interactuando con ANGi" caption="Captura de la terminal serie interactuando a 38.400 baudios con los comandos de producción del firmware." class="align-center" %}

El modo de testeo permite modificar el modo de trabajo del dispositivo desde el punto de vista de RF para procesos de certificación. Esto se alinea exactamente con la documentación fotográfica del sitio de la FCC: el dispositivo alimentado por fuente y tres cables saliendo por un costado (UART TX/RX y GND):

{% include figure popup=true image_path="/assets/images/sensor_angi/angi_fcc_test_setup.png" alt="Setup de ensayo reportado ante la FCC" caption="Setup de ensayo documentado ante la FCC: alimentación externa y conexión serie para pruebas de certificación RF del sensor ANGi." class="align-center" %}

### e. Modos de funcionamiento BLE

El firmware alterna entre cuatro estados para equilibrar consumo y disponibilidad de conexión. En reposo profundo (`DeepSleep`), la radio está apagada y el acelerómetro puede despertar el sistema al detectar movimiento. Entonces comienza una ventana de **publicidad rápida** (`FastAdvertising`, aproximadamente 20–100 ms entre anuncios) para que la app pueda conectarse; si no hay conexión tras unos 30 segundos, vuelve al reposo.

Al establecerse el enlace BLE, el sensor queda en modo **conectado** y monitorea impactos. Si se pierde la conexión, pasa a **publicidad lenta** (`SlowAdvertising`, aproximadamente 1,28–2,56 s entre anuncios), que permite reconectar sin mover el dispositivo. Si la desconexión y la inactividad se prolongan, vuelve al reposo profundo. Los intervalos y tiempos son aproximados.

{% include figure popup=true image_path="/assets/images/sensor_angi/flujo_estados_dispositivo.png" alt="Diagrama de estados BLE del sensor ANGi" caption="Transiciones entre reposo, publicidad rápida, conexión activa y publicidad lenta tras una desconexión." class="align-center" %}

### f. Algoritmo de detección de caídas

Para entender cómo decide el ANGi que ocurrió una caída, consultamos distintos modelos y contrastamos sus respuestas con el firmware. Gemini (Gemini 3.6 Flash), ChatGPT (GPT 5.6 Luna) y Copilot coincidieron en señalar dos partes relevantes: la configuración de impacto del acelerómetro **LIS3DH** y el análisis triaxial del giroscopio **L3GD20H**. Sin embargo, sus explicaciones no tenían el mismo nivel de cautela. ChatGPT reconoció varias incógnitas; Gemini propuso una secuencia más completa, pero presentó como hechos algunas hipótesis que el código disponible no demostraba. Por ejemplo, interpretó un búfer como circular y previo al impacto, y los conteos **8** y **2** como muestras de confirmación.

La revisión posterior del C mostró que esas interpretaciones iban más allá de la evidencia: el código escribe muestras, pero eso no prueba que luego se lean para decidir si hubo una caída; los valores **8** y **2** aparecen como parámetros, pero no se localizó quién los consume ni qué significado temporal tienen. OpenCode aportó una lectura más detallada de las dos ramas. En su análisis, `impact_detector_config` (`0x2F60C`) asigna por defecto un umbral de **16000** y **8** cuentas; un segundo configurador, `impact_detector_config_b` (`0x2F6BC`), usa **2** cuentas. También reportó que un umbral explícito se multiplica por 10 y que la reconfiguración escribe el registro `0x32` del LIS3DH. OpenCode ubicó el callback, el umbral y las cuentas en los offsets `+0x0C`, `+0x10` y `+0x54` del estado del detector, respectivamente, pero no encontró el consumidor que permita confirmar cómo se aplican las cuentas para decidir un impacto.

Para la inicialización del **LIS3DH**, OpenCode (Big Pickle) anotó una secuencia de parámetros `2000, 2500, 3000, callback, 16000, 1000, 10, 10, 30, 60, 60, 60, 60, 5, 2, 3, 4, 5, 7`; interpretó los valores grandes como umbrales y los últimos como tiempos o duraciones. También describió una tabla de cinco escalas por eje, generada con transformaciones del tipo `sqrt(x*c)` y `sqrt(x²*c)`, y señaló que esta última se usa en la comparación de magnitudes. Para el giroscopio reportó umbrales `0x4F5` (1269), `0x9D4` (2516) y `0x23D` (573), elevados al cuadrado y limitados por un mínimo de `0x800` antes de escribirse en los registros `0x32`–`0x37`. Estos datos sirven como pistas para recorrer la configuración; la interpretación de sus unidades y tiempos no queda demostrada por los valores por sí solos.

En la rama del **L3GD20H**, OpenCode describió `axis_history_task` (`0x28188`) leyendo los tres ejes de 16 bits y calculando una magnitud cuadrática y otra basada en las diferencias entre muestras. Según ese análisis, la tarea rastrea mínimos durante el movimiento, distingue reposo de actividad y descarta ciertos candidatos si el contador no llega a **3** o el acumulador a **20**. También identificó escrituras de `X, Y, Z` seguidas del centinela `0xAAAA` en un búfer de hasta **996** entradas (**249** muestras completas). El almacenamiento de esas palabras aparece en el código; que el búfer se use como historial circular previo al impacto y que su contenido se lea para confirmar una caída sigue siendo una interpretación no verificada.

OpenCode reportó además códigos de autotest del LIS3DH: `0x21` para la prueba satisfactoria y `0x42`, `0x43` y `0x44` para fallas en los ejes X, Y y Z. También atribuyó el callback del detector a `0x1388`; el contraste posterior con el HEX mostró que esa dirección pertenece al SoftDevice y calcula disponibilidad de recursos, no es una función de callback de impacto de la aplicación. Todavía no se encontró la ruta que consume el umbral y las cuentas para invocar un callback.

En el análisis final con Copilot, comparamos el decompilado de Ghidra con el HEX original, en vez de tomar las anotaciones realizadas por los otros LLMs del código como prueba. Esa comparación permitió encontrar código Thumb en `0x27AD4` que Ghidra no había identificado como función. Al seguir la inicialización de la placa, se pudo relacionar ese handler con la configuración de GPIO 9 y observar que compara el pin y su modo. El hallazgo amplía lo que se sabe sobre las interrupciones, pero no identifica qué componente está conectado a esa entrada ni demuestra que participe en la detección de caídas.

Con la evidencia disponible, la reconstrucción más sólida sigue siendo parcial y comprende dos ramas observables:

- **LIS3DH:** se verifica `WHO_AM_I = 0x33`; hay configuradores que guardan un umbral predeterminado de **16000**, conteos **8** o **2** y un puntero asociado. En cierta ruta también se escribe el registro `0x32`. No se localizó el consumidor que indique cómo esos parámetros confirman un impacto.
- **Giroscopio L3GD20H:** una rutina lee sus tres ejes de 16 bits, calcula la suma de cuadrados y la suma de cuadrados de las diferencias entre muestras, y compara los resultados con variables globales y límites visibles de **3** y **20**. También actualiza contadores y mínimos, puede almacenar `X, Y, Z, 0xAAAA` y señalizar un evento. La identidad del sensor está confirmada; lo que no se ha resuelto son las unidades y el significado físico de estos valores. “Energía” y *jerk* son analogías, no conclusiones verificadas.

Una tarea del LIS3DH publica telemetría mientras la rutina del giroscopio L3GD20H analiza muestras. Esto confirma que hay actividad en paralelo, pero no que ambas ramas converjan en una única decisión. Todavía no logramos reconstruir el criterio que transforma esas señales en una caída confirmada ni la ruta que genera una alarma.

{% include figure popup=true image_path="/assets/images/sensor_angi/algoritmo_deteccion.png" alt="Diagrama del algoritmo de detección de caídas" caption="Flujo observado en el firmware para las ramas de análisis de sensores; las relaciones no confirmadas se indican como incógnitas." class="align-center" %}

Podríamos continuar con un análisis más exhaustivo, pero consideramos que lo realizado era suficiente para los objetivos y las actividades del proyecto. Lo que si es seguro,y lo confirma el análisis de la apk, es que el dispositivo informa 3 clases de eventos o alarmas, Linear, Rotational y Linear+Rotanional.

---

### g. Revisión crítica de los archivos C y alucinaciones detectadas

La auditoría minuciosa de los archivos `.c` entregados reveló fortalezas pero también inconsistencias y alucinaciones típicas de los LLMs:

**`app_annotated.c` (Antigravity)**
Aunque su informe en Markdown era muy preciso respecto a la consola UART, el archivo C entregado (630 líneas) fue incompleto:
- **Falta de código decompilado:** Solo contiene prototipos de funciones, tablas de registros y comentarios; no incluye el cuerpo de ninguna función.
- **Inconsistencias técnicas:**
  - Declaró que los sensores utilizaban buses periféricos SPI0/TWI0 y SPI1/TWI1 por hardware, cuando el firmware en realidad implementa bit-banging por software.
  - Identificó el giroscopio como un L3GD20 con registro `WHO_AM_I` de `0xD4` o `0xD7` (en realidad `0xD7` corresponde al L3GD20H).
  - Describió UART0 como simple "logging SEGGER RTT", omitiendo que era la consola de comandos de producción.
  - Omitió en el archivo C los datos de `P0.30`, baudrate y Test Points.

**`app_decompiled_annotated.c` (OpenCode)**
Con 1,9 MB y 77.793 líneas de código decompilado desde Ghidra, fue una entrega masiva pero con fallas de formato:
- **Cuerpos de funciones duplicados:** De las 134 funciones renombradas, aproximadamente 79 contenían el cuerpo anotado nuevo y, justo debajo, el cuerpo original de Ghidra sin borrar con una llave `{` suelta, impidiendo compilar directamente.
- **Confusión en el trigger de la UART:** El código de `main()` chequea si `P0.30` está en nivel bajo para parpadear 3 veces e ingresar a `rf_test_console_poll()`. OpenCode interpretó erróneamente este chequeo como "detección de botón / contacto de carga".
- **Errores de mapa de registros:** Interpretó que el nRF52832 tenía un puerto de pines "P1" (solo posee el puerto P0 con 32 GPIOs), confundió la base de registros GPIO `0x50000500` y descartó la respuesta `0xD7` del sensor por desconocer que correspondía al L3GD20H.

**Veredicto final y flujo de trabajo óptimo**

| Aspecto | Antigravity (Gemini 3.6 Flash) | OpenCode (big-pickle) |
| :--- | :--- | :--- |
| **Código C entregado** | Solo prototipos, estructuras y comentarios (26 KB) | Decompilado C completo con nombres (1,9 MB) |
| **Coherencia interna** | Baja (el .c contradice hallazgos del informe .md) | Media (buen análisis de arquitectura, pero funciones duplicadas en C) |
| **Hallazgos de alto impacto** | Consola UART, pin P0.30, baudrate y TPs (verificados en chip) | Arquitectura de eventos (`evt_set`), bus de 24 B, mapa de flash |
| **Alucinaciones / Errores** | Falso SPI por hardware, omisión de cuerpos de función C | Confusión en registros GPIO nRF52, duplicación de llaves en C |

Ninguna herramienta por sí sola entregó el resultado perfecto, pero juntas se complementaron: Antigravity ganó en la verificación física y de hardware; OpenCode ganó en amplitud de decompilación y arquitectura. El flujo óptimo consistió en tomar la base decompilada de OpenCode, limpiarla de duplicados mediante un script, e incorporar los parámetros de hardware descubiertos por Antigravity.

---

### h. Mediciones de consumo y caracterización eléctrica

Para finalizar la parte de hardware, realizamos mediciones de consumo con el dispositivo en diferentes estados utilizando el [Power Profiler Kit II (PPK2)](https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2) de Nordic Semiconductor.

La herramienta permite medir corrientes desde 200 nA hasta 1 A con frecuencia de muestreo de 100 ksps. Conectamos el ANGi al PPK2 alimentándolo a 3.0 V (simulando la pila botón CR2032) y ensayamos tres modos de funcionamiento representativos del ciclo de vida del dispositivo:

{% include figure popup=true image_path="/assets/images/sensor_angi/ppk2.jpg" alt="Setup de medición con PPK2" caption="Setup de medición con el Nordic Power Profiler Kit II (PPK2) conectado al sensor ANGi para caracterizar el consumo en tiempo real." class="align-center" %}

---

#### i. Actividad normal conectado (Vigilancia en ruta)

Una vez que el sensor despierta (por movimiento) y se conecta exitosamente al teléfono, entra en su régimen de operación normal: el procesador se mantiene vigilando el acelerómetro/giroscopio y emite un latido (heartbeat) periódico por BLE:

{% include figure popup=true image_path="/assets/images/sensor_angi/se_despierta_y_conecta_anotado.png" alt="Consumo al despertar y conectar" caption="Medición en PPK2: Celeste, bajo consumo/durmiendo; Amarillo, despertar por movimiento y búsqueda de conexión; Rosa, Conectado, actividad BLE y consumo normal." class="align-center" %}

Analizando el intervalo estable (a partir de los 20 segundos de captura en adelante, duración 41.86 s):
- **Corriente promedio:** 0.062 mA (62.1 µA)
- **Corriente máxima:** 21.64 mA (picos de radio BLE)
- **Corriente mínima:** 0.004 mA (3.9 µA en sleep inter-eventos)
- **Carga consumida:** 0.000722 mAh
- **Vida útil estimada (CR2032 continua):** 3.706,6 horas (~154.4 días)

---

#### ii. Búsqueda de conexión y Bajo Consumo (Sleep / Standby)

Cuando el sensor se enciende o detecta movimiento, intenta conectar durante una ventana inicial (15 a 38 segundos). Si no encuentra ningún teléfono autorizado, cancela la transmisión y entra en modo de bajo consumo profundo a partir del segundo 40:

{% include figure popup=true image_path="/assets/images/sensor_angi/se_enciende_no_se_conecta_anotado.png" alt="Consumo en búsqueda y standby" caption="Medición en PPK2: Verde, despierta por movimiento; Amarilo, busca conexion y no encuentra; Celeste, bajo consumo/durmiendo." class="align-center" %}


Analizando el periodo de bajo consumo (de 40 a 90.8 segundos, duración 50.85 s):
- **Corriente promedio:** 0.021 mA (21.0 µA)
- **Corriente máxima:** 0.024 mA (24.1 µA)
- **Corriente mínima:** 0.019 mA (18.8 µA)
- **Carga consumida:** 0.000297 mAh
- **Vida útil estimada (CR2032 continua):** 10.949,6 horas (~456.2 días / 1.25 años)

---

#### iii. Secuencia de búsqueda intensa tras desconexión

Si el sensor pierde la conexión BLE con el teléfono durante un trayecto (por ejemplo, al alejarse o perder señal), inicia una secuencia intensa de reintentos repetidos (aproximadamente 15 intentos en un intervalo de 180 a 900 segundos) para intentar restablecer la sesión:

{% include figure popup=true image_path="/assets/images/sensor_angi/buscaba_se_conecta_luego_se_desconecta_anotado.png" alt="Búsqueda, conexión y desconexión" caption="Medición en PPK2: Amarillo, busqueda e intento de conexión; Rosa, Conectado, actividad BLE y consumo normal; Rojo, intenta reconectar luego de desconexión." class="align-center" %}

{% include figure popup=true image_path="/assets/images/sensor_angi/prende_busca_no_conecta_se_mueve_busca_conecta_desconecta_no_reconecta_anotado.png" alt="Ciclo de reconexiones fallidas" caption="Medición en PPK2: Celeste, bajo consumo; Verde, despierta por movimiento; Amarillo, busca conexion y no encuentra; Rosa, Conectado, actividad BLE y consumo normal; Rojo, intenta reconectar luego de desconexión." class="align-center" %}

Analizando el periodo de ráfaga de búsqueda (segundos 180 a 900, duración 720 s):
- **Corriente promedio:** 1.878 mA (1877.8 µA)
- **Corriente máxima:** 23.73 mA (picos continuos de publicidad y escaneo RF)
- **Corriente mínima:** 0.004 mA (3.7 µA)
- **Carga consumida:** 0.375561 mAh
- **Vida útil estimada (si estuviese 100% del tiempo buscando):** 122.5 horas (~5.1 días)

---

#### Tabla resumen de consumo y vida útil estimada

| Estado / Modo | Corriente Promedio | Corriente Máx / Mín | Vida útil estimada (CR2032 225 mAh) |
| :--- | :--- | :--- | :--- |
| **Conectado en ruta (Vigilancia)** | 62.1 µA (0.062 mA) | 21.64 mA / 3.9 µA | ~154.4 días (continuo) |
| **Bajo Consumo (Standby / Sleep)** | 21.0 µA (0.021 mA) | 24.1 µA / 18.8 µA | ~456.2 días (~1.25 años) |
| **Búsqueda activa (Reconexión)** | 1.878 mA (1877.8 µA) | 23.73 mA / 3.7 µA | ~5.1 días (si fuera continuo) |

El manual de Specialized especifica una autonomía típica de 6 meses con salidas de 6 a 8 horas semanales. Con los datos empíricos medidos en el PPK2, podemos realizar la validación técnica:

- En un uso de 7 horas semanales conectado (pedaleando a 62.1 µA):  
  `7 h × 0.0621 mA = 0.435 mAh/semana`
- En el tiempo restante (161 horas semanales) en Standby (a 21.0 µA):  
  `161 h × 0.0210 mA = 3.381 mAh/semana`
- **Consumo total estimado:** ~3,816 mAh/semana.

Para una pila botón CR2032 con una capacidad nominal utilizable de 225 mAh:  
**Autonomía Teórica** = 225 mAh / 3.816 mAh/semana ≈ **58,9 semanas** (~13,5 meses)

Considerando la autodescarga natural de la pila CR2032, las caídas de tensión provocadas por los picos repetidos de radio BLE a 21–23 mA, la variación de temperatura al aire libre y las eventuales búsquedas por desconexión, la estimación del fabricante de 6 meses es un valor sumamente realista y conservador desde la perspectiva de la ingeniería de hardware.

---

## Parte 4: Actualización DFU

Durante el proceso de desramardo del sensor original no fue particularmetne inocuo, la PCB quedó con problemas, ademas de que la apertura del gabinete no fue nada delicada, perdiendose la junta de sellado (IPX7), así que me compré un par de sensores nuevos. Resulta que los sensores venían con un firmware más viejo y eran demasiado sensibles a los golpes, y con la app oficial dada de baja no había forma de actualizarlos. La idea inicial fue tratar de usar el firmware que había extraído con el J-Link y subírselo por Bluetooth, esta vez sin romper el sensor.

### a. Armado del paquete y primer intento con Claude

Le pasamos el dump a Claude (Sonnet 5.5, en claude.ai) y en pocas iteraciones ordenamos el mapa de memoria: MBR, SoftDevice S132 v3.0.0, una aplicación de unos 79 KB, un bootloader Secure DFU de Nordic y una página de settings cuyo CRC coincidía con el de la app. También salió que el sufijo del nombre del sensor (SHS-6356) se deriva de su MAC. Armamos un paquete DFU con [nRF Util](https://www.nordicsemi.com/Products/Development-tools/nRF-Util) y un cliente en Python con bleak que reinicia el sensor en el bootloader, sube el paquete y deja un log de cada paso. Claude lo probó contra un bootloader simulado, pero el hardware real lo fue corrigiendo: buscó un UUID de DFU que no era (el sensor usa el servicio experimental de Nordic SDK 12.x, 8E400001) y el log mostró el correcto.

Con esa corrección el bootloader recibió bien el init packet (el CRC coincidía), pero lo rechazó al validarlo con un error genérico (INVALID_OBJECT) que, en ese bootloader, cubre firma, versión o formato sin decir cuál.

### b. Lectura de la APK con Antigravity y hallazgo del firmware embebido

Ahí cambiamos de herramienta. Con Antigravity (Claude Sonnet 4.6 en modo Thinking y, brevemente, Gemini 3.1 Pro en Low) recorrimos nuevamente la APK descompilada.. Encontramos el código con el que la app oficial arma el DFU y, mejor todavía, el firmware oficial embebido en sus recursos: `shs_v2_6_13.zip`, la misma versión 2.6.13 que había aparecido en los strings del dump. El análisis de los logs anteriores apuntó a una validación de versión del firmware. Con eso se reescribió el script y se documentó el procedimiento en un `README_DFU.md`, y la actualización por fin funcionó: el sensor aceptó el paquete, completó la transferencia y arrancó con la versión nueva.

Un detalle que nos dolió descubrir: sacar el firmware con el J-Link fue innecesario. El binario estuvo siempre ahí, dentro de la APK, empaquetado como un recurso más de la app. Esperábamos lo contrario: que el firmware se descargara de los servidores del fabricante según las actualizaciones que fueran saliendo, y no que viajara adentro de la app. Con esa suposición ni se nos ocurrió mirar los recursos de la APK antes de abrir el sensor y conectarle el programador. Además, el .zip de la APK trae algo que nuestro dump no tenía: el init packet ya armado por el fabricante, que es lo que el bootloader valida. El dump sirvió para entender cómo está organizado el firmware, pero para actualizar el sensor alcanzaba con revisar la APK primero.

{% include figure popup=true image_path="/assets/images/sensor_angi/diagrama_flujo_dfu.png" alt="Diagrama de flujo del proceso de DFU" caption="Diagrama de flujo del proceso de actualización DFU del firmware del sensor ANGi." class="align-center" %}

### c. Integración del DFU en OpenAngi

Con la actualización ya funcionando desde el script de Python, el paso siguiente fue llevarla a la propia app. Le pedimos a OpenCode (con mimo-v2.6-flash-free, un modelo del tier gratuito) que revisara la carpeta del DFU (el README, el .zip, el script y el log de la actualización exitosa) y que agregara a OpenAngi un aviso y la actualización por DFU cuando detecte un firmware viejo. El resultado fue una detección por versión contra la 2.6.13, una fila "Firmware (desactualizado)" en la pantalla del dispositivo y una serie de diálogos: actualización requerida, batería baja, progreso, reinicio y error con botón de reintentar. Por debajo usa la librería DFU de Nordic, con el .zip oficial embebido y los parámetros que había usado la app original. 

La prueba final, verificada en hardware, fue un sensor (SHS-E6F5) que pasó de la 2.5.22 a la 2.6.13: se envió el comando, el sensor confirmó, apareció como DfuTarg, la transferencia tardó 47 s y, al reconectar, la app leyó la versión nueva y la sesión volvió a activarse. Esa ventana de unas 2 h 10 min fue un único modelo, sin subagentes: 213 mensajes del asistente contra 13 nuestros, 291 llamadas a herramientas (125 de bash, 61 ediciones y 59 lecturas), unos 1,05 millones de tokens de entrada, 65 mil de salida y 243 mil de razonamiento, a costo cero.

<figure class="align-center">
  <video controls width="100%">
    <source src="/assets/images/sensor_angi/dfu_process.mp4" type="video/mp4">
    Tu navegador no soporta la reproducción de video.
  </video>
  <figcaption>Actualización DFU del sensor ANGi SHS-E6F5 de la versión 2.5.22 a la 2.6.13.</figcaption>
</figure>

---

## Parte 5: Balance del uso de herramientas de IA

Cada parte de este proyecto usó herramientas de IA de forma diferente. A modo de cierre, vale la pena ver el panorama completo y extraer algunas conclusiones sobre cómo se repartieron las tareas entre modelos y qué funcionó mejor en cada contexto.

### a. Qué hizo cada herramienta

Los detalles de uso de cada sesión están documentados en las partes anteriores ([Parte 1](#parte-1-claude-y-la-exploración-ble), [Parte 2](#parte-2-opencode-y-el-desarrollo-de-las-apps-android), [Parte 3](#parte-3-hardware-y-firmware--análisis-comparativo-antigravity-vs-opencode), [Parte 4](#parte-4-actualización-dfu)). A grandes rasgos, la división fue:

| Herramienta / Modelo | Rol principal en el proyecto |
| :--- | :--- |
| **Claude Sonnet 5** *(claude.ai)* | Exploración BLE inicial, scripts Python, identificación de chips por foto de PCB |
| **Claude Sonnet 5.5** *(claude.ai)* | Análisis del dump hex, mapa de memoria nRF52, scripts Ghidra Java |
| **OpenCode** (`mimo`, `ling`, `big-pickle`) | Desarrollo Android (AngiTester y OpenAngi), decompilado y anotación del C de Ghidra |
| **Antigravity** (`Claude Sonnet 4.6 Thinking`) | Lectura de APK para entender el DFU, extracción del firmware embebido, reescritura del script |
| **ChatGPT** (GPT-5.6 Luna) | Consulta sobre el marco legal de la ingeniería inversa en Argentina |

Ninguna herramienta reemplazó el trabajo de campo: cada hipótesis pasó por medición o ejecución real sobre el sensor. Lo que los modelos aceleraron fue el triage del binario crudo, la escritura y corrección de herramientas, y la lectura de miles de líneas de Java decompilado. Pero no pueden tocar el hardware, y algunas de sus interpretaciones estaban equivocadas.

### b. Escala del proyecto y costo

El volumen de trabajo procesado por los modelos fue considerable. Los números exactos de OpenCode provienen de su base de datos SQLite; los de Claude y Antigravity son estimaciones basadas en los registros de sesión:

| Sesión | Mensajes asistente | Llamadas a herramientas | Tokens entrada (sin caché) | Costo |
| :--- | :--- | :--- | :--- | :--- |
| Claude Web (exploración BLE + firmware) | ~32 | ~68 | N/A (Web UI) | Suscripción |
| OpenCode (apps Android + decompilado C) | 738 + 85 | 1.067 + 209 | ~4,94 M tokens | $0 |
| Antigravity (DFU) | 213 | 291 | ~1,05 M tokens | $0 |
| **Total** | **~1.070** | **~1.635** | **~6 M tokens** | **Mínimo** |

El grueso del desarrollo —apps Android, decompilado del firmware, actualización DFU— corrió íntegramente en tiers gratuitos, a costo cero.

### c. Especialización observada y limitaciones

La selección de herramienta no fue aleatoria: cada una mostró ventajas claras según la tarea.

- **Claude** (interfaz web con visión) fue óptimo para la fase exploratoria: identificar chips por foto, analizar capturas de pantalla de nRF Connect y construir scripts incrementales sin acceso al filesystem.
- **OpenCode** (terminal con bash, edición de archivos y compilación) fue la herramienta natural para construir y depurar aplicaciones reales: compilar con Gradle, corregir errores y mantener coherencia en más de 30 archivos Kotlin a la vez.
- **Antigravity** (agente con acceso al filesystem y modo Thinking) resultó eficaz cuando la tarea requería razonamiento sobre código Java decompilado complejo y depurar errores de protocolo no documentados.

Todas las herramientas mostraron alucinaciones o errores en alguna instancia — detallados en la [sección g de la Parte 3](#g-revisión-crítica-de-los-archivos-c-y-alucinaciones-detectadas) y en la [sección f sobre el algoritmo de detección de caídas](#f-algoritmo-de-detección-de-caídas). La regla de trabajo fue la misma en todos los casos: verificar cada resultado contra el hardware o el binario original antes de tomar decisiones.

Esta es también una de las líneas de investigación que estamos incorporando al PID del Laboratorio: lograr cerrar el lazo, dar acceso a los LLMs al hardware y a la interacción con herramientas físicas que permitan acelerar el proceso de experimentación.

---

## Conclusiones y consideraciones legales

El proceso estuvo condicionado por el nivel de protección que presentaba el dispositivo y su software. La APK analizada no utilizaba ofuscación o cifrado que impidiera examinar su lógica, y el protocolo de comunicación con el ANGi podía ser observado y caracterizado sin necesidad de romper mecanismos criptográficos. Además, durante el análisis del hardware encontramos que el firmware del nRF52 era accesible en las condiciones en las que se encontraba el dispositivo, cuando técnicamente podría haber estado protegido mediante los mecanismos de bloqueo disponibles en el microcontrolador. Todo esto nos facilitó considerablemente el trabajo y nos permitió llevar adelante la ingeniería inversa con herramientas relativamente convencionales de análisis de software, comunicaciones y firmware. Es importante aclarar que estos elementos describen las condiciones encontradas en el dispositivo analizado y no implican que el fabricante haya diseñado deliberadamente el producto para facilitar su ingeniería inversa.

### ¿Qué tan legal es hacer ingeniería inversa?

Llegado este punto apareció otra pregunta: si el dispositivo es nuestro, ¿qué ocurre con la ingeniería inversa que hicimos para volver a utilizarlo? Para este punto recurrimos a ChatGPT (OpenAI, GPT-5.6 Luna) para consultar sobre los aspectos legales del proceso, particularmente en el contexto argentino. La conclusión general fue que hay una diferencia importante entre analizar un software para descubrir la información funcional necesaria para interoperar con un dispositivo propio y copiar, modificar o redistribuir el software propietario original. En este trabajo se utilizó la APK de Ride como objeto de análisis para descubrir el protocolo de comunicación, pero las aplicaciones desarrolladas posteriormente fueron escritas desde cero y no incorporan código, recursos ni layouts de la aplicación original. Del mismo modo, el firmware original del dispositivo no forma parte del desarrollo de OpenAngi; fue utilizado únicamente como material de análisis y, posteriormente, para validar experimentalmente el mecanismo de actualización DFU. De hecho, ni siquiera fue necesario analizar el firmware para implementar OpenAngi: terminó siendo más bien una excusa para explorar el uso de herramientas de IA en el proceso de ingeniería inversa aplicado a firmware.

La cuestión resulta especialmente interesante al pensar en una eventual publicación del proyecto. Una posibilidad, la que se presenta en esta página, sería publicar únicamente el proceso y las conclusiones de la ingeniería inversa; otra, documentar además el protocolo descubierto; y una tercera, eventualmente publicar el código fuente o una APK de una implementación independiente como OpenAngi. En cambio, la publicación del APK original, del código decompilado de Ride o del firmware propietario plantea cuestiones de propiedad intelectual diferentes y, por ese motivo, esos materiales no forman parte de lo que se plantea publicar. En todos los casos, se trata de una evaluación preliminar y no de asesoramiento jurídico profesional: si el proyecto avanzara hacia una distribución pública o comercial, sería necesario revisar específicamente las licencias, la propiedad intelectual y las marcas involucradas.

Hay una excepción interesante dentro del propio desarrollo. Durante las pruebas implementamos también la posibilidad de actualizar el firmware del ANGi mediante DFU. Para realizar esta prueba utilizamos el archivo binario de firmware que se encontraba incluido dentro de la APK original. Esto nos permitió comprobar experimentalmente el proceso de actualización y verificar que era posible implementar el mecanismo de DFU de forma independiente. Sin embargo, el binario utilizado no es código desarrollado por nosotros, sino un componente propietario que provenía de la aplicación original. Por ese motivo, no consideramos apropiado redistribuirlo como parte de una aplicación propia sin analizar previamente sus derechos y condiciones de uso. La solución que estamos evaluando es mantener una aplicación experimental que incluya ese archivo únicamente para las pruebas de DFU y, por separado, una versión de OpenAngi sin el firmware propietario. Esta última sería la candidata para una eventual distribución pública.

[![Hits](https://hits.sh/prototipado.github.io/desarrollos/angi_ing_inv/.svg)](https://hits.sh/prototipado.github.io/desarrollos/angi_ing_inv/)