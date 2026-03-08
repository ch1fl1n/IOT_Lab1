# Evaluación experimental de los módulos ADC y DAC del ESP32 mediante instrumentación básica

**Autores:** Daniel David Gomez Britto | Juan Camilo Silva Velasco  
**Curso:** Internet de las Cosas  
**Docente:** Fabian Mauricio Paez Rivera  
**Fecha:** 04/03/2026  

---

# Abstract

Se implementaron y evaluaron experimentos de generación y adquisición analógica usando un ESP32 (DAC interno de 8 bits y ADC hasta 12 bits), un generador de funciones y un osciloscopio. Se realizaron pruebas en bucle **DAC→ADC (loopback)** generando formas triangulares y senoidales y registrando parámetros prácticos: **Vmax/Vmin, escalonamiento por resolución del DAC, ruido y jitter temporal por muestreo software**.

Los resultados muestran la idoneidad del ESP32 para **enseñanza y prototipado** y evidencian limitaciones en **resolución, linealidad y velocidad efectiva**. Finalmente se proponen mejoras de software y acondicionamiento analógico para mejorar precisión y estabilidad.

**Keywords—** ESP32, ADC, DAC, SAR, cuantización, jitter, osciloscopio, instrumentación.

---

# I. Introducción

La práctica explora la **generación y adquisición analógica con un ESP32**, usando:

- **DAC de 8 bits** (GPIO25 / GPIO26)
- **ADC de hasta 12 bits** en ADC1

El objetivo fue **caracterizar el comportamiento del sistema** en términos de:

- amplitud
- forma de onda
- ruido
- jitter

Para ello se utilizó un **lazo DAC→ADC en la misma placa**, permitiendo evaluar generación y adquisición en cadena controlada por software.

Adicionalmente se realizaron **simulaciones de periféricos (relé y sensor MPU6050)** para complementar las competencias de integración de dispositivos dentro de sistemas IoT.

---

# II. Fundamento teórico

## ADC (Analog to Digital Converter)

Un **ADC de tipo SAR** realiza tres procesos principales:

1. Muestreo
2. Cuantización
3. Codificación digital

Para un ADC de **n bits**, existen:

\[
2^n
\]

niveles posibles.

En el **ESP32**, el ADC puede entregar hasta **12 bits**, lo que corresponde a valores digitales entre:

```
0 – 4095
```

---

## DAC (Digital to Analog Converter)

Un **DAC convierte un código digital en un voltaje analógico**.

En el ESP32 el DAC es de **8 bits**, por lo que los códigos posibles son:

```
0 – 255
```

La salida teórica puede aproximarse mediante:

\[
V_{out} ≈ \frac{D}{2^n-1} V_{ref}
\]

donde:

- **D** = código digital
- **n** = número de bits
- **Vref** = voltaje de referencia

---

## Errores y limitaciones

Durante la adquisición y generación analógica pueden presentarse:

- **Error de cuantización (LSB)**
- **Errores DNL / INL**
- **Ruido interno**
- **Offsets**
- **Interferencias del sistema (ej. WiFi en ADC2)**
- **Jitter temporal por muestreo software**

---

# III. Metodología experimental

## Equipamiento

- Placa **ESP32**
- **Osciloscopio**
- **Generador de funciones**
- Cables y conexión a tierra común

---

## Configuración del osciloscopio

- **Sonda:** 10×
- **Acoplamiento:** DC
- **Canal:** CH1
- **Escala sugerida:** 500 mV/div
- **Trigger:** por borde ascendente

---

## Conexiones

| Dispositivo | Conexión |
|--------------|-----------|
| DAC ESP32 | GPIO25 |
| ADC ESP32 | GPIO34 |
| Potenciómetro | GPIO35 |
| Osciloscopio | DAC GPIO25 |
| Tierra | GND común |

El **loopback** se realizó conectando:

```
DAC (GPIO25) → ADC (GPIO34)
```

---

## Pruebas realizadas

### 1. Prueba de loopback DAC → ADC

En esta prueba se realizó una verificación directa del funcionamiento conjunto de los módulos **DAC (Digital to Analog Converter)** y **ADC (Analog to Digital Converter)** del ESP32. Para ello, se conectó físicamente el pin **GPIO25**, correspondiente al DAC, con el pin **GPIO34**, configurado como entrada ADC. De esta forma, la señal generada digitalmente por el DAC podía ser leída inmediatamente por el ADC, permitiendo evaluar el comportamiento del sistema de conversión digital–analógico–digital.

El código implementa la generación de una **onda triangular**, la cual se obtiene incrementando y decrementando progresivamente el valor enviado al DAC. Esto se logra mediante la variable `dacValue`, que aumenta o disminuye dependiendo del signo almacenado en `stepSign`. Cuando el valor alcanza el límite superior (`dacMax`) o el límite inferior (0), el signo se invierte, provocando el cambio de dirección de la señal y generando así la forma triangular característica.

El DAC del ESP32 posee una resolución de **8 bits**, por lo que los valores enviados varían entre **0 y 255**. En el código se define un factor de escala:


float dacScale = 3.3 / 255.0;


Este factor permite convertir el valor digital del DAC a su equivalente aproximado en voltaje, considerando una referencia máxima de **3.3 V**. De esta manera, el voltaje teórico generado por el DAC se calcula como:


vDac = (dacValue * dacScale) + dacOffset


Posteriormente, la señal es capturada por el ADC a través del pin **GPIO34** mediante la función `analogRead()`. El ADC del ESP32 se configuró con una resolución de **12 bits** usando:


analogReadResolution(12);


Esto implica que el valor digital obtenido puede variar entre **0 y 4095**. Para convertir este valor nuevamente a voltaje se utiliza la relación:


vAdc = (adcRaw / 4095.0) * 3.3


De esta manera se puede comparar el voltaje teórico generado por el DAC con el voltaje medido por el ADC.

Adicionalmente, el programa calcula una estimación de la **frecuencia de muestreo** del sistema utilizando la función `micros()`. Cada cierto número de muestras se calcula el tiempo transcurrido entre lecturas consecutivas (`dt`) y se estima la tasa de muestreo mediante:


measuredSampleRate = 1000000.0 / dt


Este valor se imprime por el puerto serial junto con el código DAC, la lectura del ADC y el voltaje estimado, permitiendo analizar en tiempo real el comportamiento del sistema.

Para facilitar la experimentación, se utilizó un **potenciómetro conectado al pin GPIO35**, el cual es leído mediante `analogRead()`. Dependiendo del modo de operación definido por la constante `MODE_AMPLITUDE`, este potenciómetro permite controlar distintos parámetros de la señal generada.

En el caso utilizado durante las pruebas (`MODE_AMPLITUDE = true`), el valor del potenciómetro se utiliza para modificar la **amplitud máxima de la señal triangular**, ajustando el límite superior `dacMax` mediante la función `map()`:


dacMax = map(potRaw,0,4095,10,255);


Esto permite variar dinámicamente la amplitud de la señal generada por el DAC, lo cual se refleja inmediatamente en la señal capturada por el ADC. Este comportamiento permite comprobar el correcto funcionamiento del sistema de conversión y verificar la correspondencia entre la señal generada y la señal medida.

Finalmente, todos los parámetros relevantes se envían al monitor serial cada **500 ms**, incluyendo el código digital del DAC, el valor crudo del ADC, el voltaje estimado y la frecuencia de muestreo aproximada.

Los **videos representativos de esta prueba de loopback DAC–ADC se encuentran disponibles en la carpeta de evidencias (ADC-DAC)**.

---

### 2. Prueba con generador de señales externo

La segunda prueba tuvo como objetivo analizar la respuesta del **ADC del ESP32 frente a una señal analógica externa controlada**, utilizando un generador de funciones. Para realizar esta prueba, se desconectó el DAC del circuito y se aplicó directamente una señal al pin **GPIO34**, que actúa como entrada analógica.

Se utilizó una señal **senoidal de 1 kHz y aproximadamente 2 V pico a pico (Vpp)** generada por el equipo de laboratorio. Esta señal fue introducida directamente al pin ADC para observar cómo el microcontrolador digitaliza la señal analógica.

El código empleado en esta prueba es el mismo utilizado en el loopback, ya que la lectura del ADC se realiza de manera independiente mediante:


int adcRaw = analogRead(ADC_LOOP_PIN);


Cada lectura devuelve un valor entre **0 y 4095**, correspondiente a la cuantización de la señal analógica en el rango de referencia de **0 a 3.3 V**. Posteriormente, este valor se transforma a voltaje mediante la relación:


vAdc = (adcRaw / 4095.0) * 3.3


Gracias a esta conversión es posible observar en el monitor serial la variación del voltaje instantáneo correspondiente a la señal senoidal aplicada.

Durante esta prueba se analizaron varios aspectos importantes del comportamiento del ADC:

- **Capacidad de seguimiento de la señal**: se observó cómo los valores leídos por el ADC varían continuamente siguiendo la forma de la señal senoidal.
- **Resolución de cuantización**: al tratarse de un ADC de 12 bits, la señal se discretiza en 4096 niveles posibles.
- **Velocidad de muestreo**: el programa calcula una estimación de la tasa de muestreo usando diferencias de tiempo entre lecturas (`micros()`), lo que permite evaluar si la frecuencia de muestreo es suficiente para representar correctamente la señal de entrada.

La relación entre la frecuencia de la señal (1 kHz) y la frecuencia de muestreo estimada permite comprobar si se cumple adecuadamente el **criterio de Nyquist**, el cual establece que la frecuencia de muestreo debe ser al menos el doble de la frecuencia de la señal para evitar aliasing.

Los valores capturados fueron enviados al monitor serial junto con la estimación de la frecuencia de muestreo, lo cual permitió verificar el comportamiento dinámico del sistema frente a señales externas.

Los **videos correspondientes a esta prueba con el generador de señales externo se encuentran incluidos en la carpeta de evidencias (DAC sin conectar)**.


---
### 3. Mediciones con osciloscopio

La tercera prueba consistió en la observación directa de la señal generada por el **DAC del ESP32** utilizando un **osciloscopio**, con el objetivo de analizar visualmente la forma de onda producida por el sistema.

Para esta medición se desconectó la conexión hacia el ADC y se conectó directamente el pin **GPIO25** (salida DAC) al canal de entrada del osciloscopio. De esta manera fue posible visualizar en tiempo real la señal analógica generada por el microcontrolador.

Como se describe en el código, la señal generada corresponde a una **onda triangular**, obtenida mediante el incremento y decremento progresivo del valor digital enviado al DAC (`dacValue`). El tamaño del incremento está determinado por la variable `stepAbs`, mientras que la dirección del cambio depende de `stepSign`. Cuando el valor alcanza el límite superior (`dacMax`) o el inferior (0), el signo se invierte y la señal comienza a variar en la dirección opuesta.

Este mecanismo produce una señal triangular cuya amplitud depende directamente del valor máximo configurado (`dacMax`), el cual puede ser modificado dinámicamente mediante el potenciómetro conectado al pin **GPIO35**.

Durante la observación en el osciloscopio se registraron los principales parámetros de la señal:

- **Voltaje máximo (Vmax)**
- **Voltaje mínimo (Vmin)**
- **Voltaje pico a pico (Vpp)**
- **Frecuencia de la señal**
- **Forma de onda observada**

Estas mediciones permiten validar experimentalmente la relación entre los valores digitales enviados al DAC y el voltaje analógico generado. Asimismo, la observación directa de la señal permite identificar posibles efectos no ideales, como pequeñas variaciones en la linealidad de la señal o limitaciones propias del convertidor digital–analógico del ESP32.

Adicionalmente, el control de la amplitud mediante el potenciómetro permitió observar cómo la señal triangular cambia su rango de voltaje en tiempo real, confirmando el correcto funcionamiento del mecanismo de control implementado en el código.

Los **videos donde se observa la señal generada en el osciloscopio y su comportamiento dinámico se encuentran disponibles en la carpeta de evidencias (ADC sin conectar)**.
---

# IV. Implementación de software

A continuación se presenta el código utilizado para generar una onda triangular con el DAC y leerla con el ADC.

```cpp
const int DAC_PIN = 25;
const int ADC_LOOP_PIN = 34;
const int ADC_POT_PIN = 35;

const bool MODE_AMPLITUDE = true;

int dacValue = 0;
int stepAbs = 5;
int stepSign = 1;

int dacMax = 255;

unsigned long lastPrint = 0;
unsigned long samples = 0;
unsigned long t_last = 0;

float measuredSampleRate = 0.0;

float dacOffset = 0.0;
float dacScale = 3.3 / 255.0;

void setup() {

  Serial.begin(115200);
  delay(500);

  Serial.println("ESP32 DAC -> ADC loopback");

  analogReadResolution(12);

  t_last = micros();
}

void loop() {

  int potRaw = analogRead(ADC_POT_PIN);

  if (MODE_AMPLITUDE) {
    dacMax = map(potRaw,0,4095,10,255);
    stepAbs = 5;
  }
  else {
    stepAbs = map(potRaw,0,4095,1,20);
    dacMax = 255;
  }

  dacValue += stepSign * stepAbs;

  if (dacValue >= dacMax) {
    dacValue = dacMax;
    stepSign = -1;
  }
  else if (dacValue <= 0) {
    dacValue = 0;
    stepSign = 1;
  }

  dacWrite(DAC_PIN, dacValue);

  int adcRaw = analogRead(ADC_LOOP_PIN);

  float vDac = (dacValue * dacScale) + dacOffset;
  float vAdc = (adcRaw / 4095.0) * 3.3;

  samples++;

  unsigned long now = micros();
  unsigned long dt = now - t_last;
  t_last = now;

  if (samples % 200 == 0) {
    measuredSampleRate = 1000000.0 / dt;
  }

  if (millis() - lastPrint >= 500) {

    Serial.print("dacCode=");
    Serial.print(dacValue);

    Serial.print(" adcRaw=");
    Serial.print(adcRaw);

    Serial.print(" adcV=");
    Serial.print(vAdc);

    Serial.print(" sampleRate=");
    Serial.print(measuredSampleRate);

    Serial.println(" Hz");

    lastPrint = millis();
  }

  delay(1);
}
```

---

# V. Resultados y discusión

## Mediciones representativas

| Medida | Valor |
|------|------|
| Vmax | 3.28 V |
| Vmin | -0.06 V |
| Vpp | 3.34 V |
| Paso DAC | 12.94 mV |
| Resolución ADC | 0.805 mV |

---

## Análisis

El **escalonamiento visible en la señal triangular** se debe a la resolución del **DAC de 8 bits**.

El **ADC de 12 bits** posee suficiente resolución para detectar estos escalones.

El valor negativo observado en **Vmin** probablemente se debe a:

- compensación de la sonda
- conexión a tierra

El **jitter temporal** se origina en el uso de:

- `analogRead()`
- procesamiento del loop
- comunicación serial

---

# V.A Comparación formal ESP32 vs equipo profesional

| Aspecto | Sistema ESP32 | Equipo profesional |
|---|---|---|
| Resolución | DAC de 8 bits y ADC de 12 bits; escalonamiento visible | Mayor resolución efectiva; representación más suave |
| Precisión | Sensible a offset, ganancia y ruido de referencia | Mejor calibración y menor incertidumbre metrológica |
| Frecuencia útil | Limitada por software y serial en pruebas tipo Arduino | Amplio rango operativo y adquisición optimizada |
| Ruido | Mayor susceptibilidad a alimentación, cableado y entorno | Mejor inmunidad y acondicionamiento de señal |
| Estabilidad | Dependiente del montaje y condiciones de laboratorio | Mayor estabilidad temporal y repetibilidad |

Esta comparación confirma que el ESP32 es adecuado para aprendizaje y prototipado, mientras que la instrumentación profesional se mantiene como referencia para medición de alta precisión.

---

# V.B Respuestas de la guía (5 preguntas)

### 1) ¿Cuáles son las características y el funcionamiento general de ADC y DAC en el ESP32?
El **ADC** convierte voltajes analógicos a valores digitales (hasta 12 bits en ESP32), mientras que el **DAC** realiza la operación inversa (8 bits en GPIO25/GPIO26). En conjunto permiten implementar sistemas básicos de adquisición y generación de señales para aplicaciones educativas e IoT.

### 2) ¿Se puede alcanzar la velocidad máxima de muestreo del ADC desde código Arduino estándar?
De forma sostenida, no normalmente. El uso de `analogRead()`, el procesamiento dentro del `loop` y el envío por puerto serial reducen la tasa efectiva de muestreo. Para mejorar rendimiento se recomienda modo continuo, DMA e implementación en ESP-IDF.

### 3) ¿Cómo funciona el método SAR?
El ADC SAR compara iterativamente la señal de entrada contra una referencia interna usando una búsqueda binaria: decide primero el bit más significativo y continúa hasta el menos significativo. Así obtiene una conversión rápida con bajo consumo.

### 4) ¿Cómo implementar un DAC con resistencias y salidas digitales?
Puede implementarse con una red **R-2R**, donde cada bit conmuta a `Vref` o GND. La salida analógica es proporcional al valor binario aplicado. Para buen desempeño se requiere relación 2:1 estable, tolerancia baja (ideal ≤1%) y estabilidad térmica de resistencias.

### 5) ¿Por qué difieren los resultados frente a equipos profesionales?
Por la menor resolución, mayor ruido, limitaciones de linealidad, restricciones de muestreo/tiempo de establecimiento y dependencia del software de adquisición. Estas limitaciones se evidencian en escalonamiento, jitter y menor repetibilidad.

---

# V.C Manejo de archivos en ESP32 con microSD

## Interfaz y conexiones SPI recomendadas

| Módulo microSD | ESP32 |
|---|---|
| CS | GPIO5 |
| MOSI (DI) | GPIO23 |
| MISO (DO) | GPIO19 |
| SCK | GPIO18 |
| VCC | 3.3 V |
| GND | GND |

## Flujo básico de uso
1. Realizar conexión física SPI con tierra común.
2. Inicializar tarjeta con librería `SD.h`.
3. Abrir archivo en modo lectura/escritura.
4. Registrar datos con marca temporal.
5. Cerrar archivo para preservar integridad.

## Ejemplo de aplicación (data logger)
Un caso típico consiste en registrar mediciones periódicas en `datos.txt`, por ejemplo:
- `2026-03-01 10:30:15, Temperatura: 25.3 C`
- `2026-03-01 10:31:15, Temperatura: 25.4 C`
- `2026-03-01 10:32:15, Temperatura: 25.2 C`

Este enfoque permite almacenar evidencia experimental y analizar datos posteriormente en computador.

---

# VI. Simulaciones realizadas

## Simulación A — Relé

### Código

```cpp
const int buttonPin = 17;
const int relayPin = 16;

void setup() {

  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(relayPin, OUTPUT);

}

void loop() {

  if (digitalRead(buttonPin) == LOW) {

    digitalWrite(relayPin, HIGH);

  } else {

    digitalWrite(relayPin, LOW);

  }
}
```

### Descripción

En la simulación se conecta:

- un **pulsador** al pin 17
- un **módulo relé** al pin 16

Al presionar el botón, el relé se activa.

<img width="1600" height="694" alt="image" src="https://github.com/user-attachments/assets/e483245f-cb50-49d4-b751-45e321dbb752" />

---

## Simulación B — Sensor MPU6050

### Código 

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_rom_sys.h"

// Pines I2C por software
#define SDA_PIN 21
#define SCL_PIN 22

#define MPU_ADDR 0x68
#define PWR_MGMT_1 0x6B
#define WHO_AM_I   0x75

// ---------- I2C SOFTWARE ----------
static void i2c_delay() {
    esp_rom_delay_us(5);
}

static void i2c_start() {
    gpio_set_level(SDA_PIN, 1);
    gpio_set_level(SCL_PIN, 1);
    i2c_delay();
    gpio_set_level(SDA_PIN, 0);
    i2c_delay();
    gpio_set_level(SCL_PIN, 0);
}

static void i2c_stop() {
    gpio_set_level(SDA_PIN, 0);
    gpio_set_level(SCL_PIN, 1);
    i2c_delay();
    gpio_set_level(SDA_PIN, 1);
}

static void i2c_write_byte(uint8_t data) {
    for (int i = 0; i < 8; i++) {
        gpio_set_level(SDA_PIN, (data & 0x80) != 0);
        data <<= 1;
        gpio_set_level(SCL_PIN, 1);
        i2c_delay();
        gpio_set_level(SCL_PIN, 0);
    }

    // ACK (ignoramos resultado)
    gpio_set_direction(SDA_PIN, GPIO_MODE_INPUT);
    gpio_set_level(SCL_PIN, 1);
    i2c_delay();
    gpio_set_level(SCL_PIN, 0);
    gpio_set_direction(SDA_PIN, GPIO_MODE_OUTPUT);
}

static uint8_t i2c_read_byte() {
    uint8_t data = 0;
    gpio_set_direction(SDA_PIN, GPIO_MODE_INPUT);

    for (int i = 0; i < 8; i++) {
        gpio_set_level(SCL_PIN, 1);
        i2c_delay();
        data = (data << 1) | gpio_get_level(SDA_PIN);
        gpio_set_level(SCL_PIN, 0);
        i2c_delay();
    }

    // NACK
    gpio_set_direction(SDA_PIN, GPIO_MODE_OUTPUT);
    gpio_set_level(SDA_PIN, 1);
    gpio_set_level(SCL_PIN, 1);
    i2c_delay();
    gpio_set_level(SCL_PIN, 0);

    return data;
}

// ---------- MPU6050 ----------
static void mpu_write(uint8_t reg, uint8_t val) {
    i2c_start();
    i2c_write_byte(MPU_ADDR << 1);
    i2c_write_byte(reg);
    i2c_write_byte(val);
    i2c_stop();
}

static uint8_t mpu_read(uint8_t reg) {
    i2c_start();
    i2c_write_byte(MPU_ADDR << 1);
    i2c_write_byte(reg);
    i2c_start();
    i2c_write_byte((MPU_ADDR << 1) | 1);
    uint8_t val = i2c_read_byte();
    i2c_stop();
    return val;
}

// ---------- MAIN ----------
void app_main(void) {

    gpio_set_direction(SDA_PIN, GPIO_MODE_OUTPUT);
    gpio_set_direction(SCL_PIN, GPIO_MODE_OUTPUT);
    gpio_set_level(SDA_PIN, 1);
    gpio_set_level(SCL_PIN, 1);

    vTaskDelay(pdMS_TO_TICKS(100));

    // Despertar MPU6050
    mpu_write(PWR_MGMT_1, 0x00);

    // Leer WHO_AM_I
    uint8_t id = mpu_read(WHO_AM_I);

    printf("MPU6050 WHO_AM_I = 0x%02X\n", id);

    while (1) {

        uint8_t ax_h = mpu_read(0x3B);
        uint8_t ax_l = mpu_read(0x3C);
        int16_t ax = (ax_h << 8) | ax_l;

        float ax_g = ax / 16384.0;

        printf("AX = %.2f g\n", ax_g);

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

```

Se implementa la comunicación entre el ESP32 y el sensor MPU6050 utilizando el protocolo I2C, pero en este caso mediante una implementación por software (bit-banging) en lugar de utilizar el controlador I2C hardware del microcontrolador. Esto permite comprender con mayor detalle cómo funciona el protocolo a nivel de señales digitales.

Para la comunicación se utilizan dos pines del ESP32:

GPIO21 como línea SDA (Serial Data)

GPIO22 como línea SCL (Serial Clock)

Estas líneas se controlan manualmente mediante funciones que generan las condiciones básicas del protocolo I2C, tales como:

Condición de inicio (START) mediante i2c_start()

Condición de parada (STOP) mediante i2c_stop()

Escritura de un byte mediante i2c_write_byte()

Lectura de un byte mediante i2c_read_byte()

Cada bit se transmite controlando el nivel lógico de la línea SDA mientras se generan pulsos de reloj en la línea SCL, siguiendo el funcionamiento estándar del protocolo I2C. Para garantizar la temporización adecuada se introduce un pequeño retardo usando la función esp_rom_delay_us().

Una vez implementada la capa básica de comunicación I2C, el código define funciones específicas para interactuar con el MPU6050, utilizando su dirección I2C 0x68. Estas funciones son:

mpu_write() para escribir valores en los registros internos del sensor.

mpu_read() para leer valores almacenados en dichos registros.

Durante la inicialización del sistema, el programa envía un comando al registro PWR_MGMT_1 (0x6B) con el valor 0x00, lo cual permite despertar el sensor, ya que por defecto el MPU6050 inicia en modo de bajo consumo.

Posteriormente se realiza una lectura del registro WHO_AM_I (0x75), el cual contiene el identificador del dispositivo. Esta lectura permite verificar que la comunicación I2C se estableció correctamente y que el sensor responde adecuadamente al microcontrolador.

Una vez verificada la conexión, el programa entra en un bucle infinito donde realiza lecturas periódicas del acelerómetro en el eje X. Para ello se leen dos registros consecutivos:

0x3B → parte alta del valor del acelerómetro en X

0x3C → parte baja del valor del acelerómetro en X

Ambos bytes se combinan para formar un valor de 16 bits con signo, que representa la aceleración medida por el sensor. Este valor se convierte posteriormente a unidades físicas de gravedad (g) utilizando el factor de escala del acelerómetro:

ax_g = ax / 16384.0

Este factor corresponde a la configuración típica del sensor en un rango de ±2 g, donde 16384 LSB equivalen a 1 g.

Finalmente, el valor calculado de aceleración se imprime por el puerto serial cada 1 segundo, permitiendo observar cómo cambia la medición al inclinar o mover el sensor.

Este procedimiento permite validar tanto la correcta comunicación I2C como el funcionamiento del acelerómetro del MPU6050, observando en tiempo real las variaciones de aceleración en el eje X.

<img width="1600" height="694" alt="image" src="https://github.com/user-attachments/assets/f24eb99f-9479-42cb-924b-b21bba05080a" />

---

# VII. Fuentes de error

Se identificaron las siguientes fuentes de incertidumbre:

- resolución del **DAC**
- jitter por muestreo software
- ruido eléctrico
- limitaciones del ADC
- interferencias del sistema

---

# VIII. Evidencias experimentales

El informe debe incluir:

- Capturas del **osciloscopio**
- Logs del **monitor serial**
- Capturas de la **simulación del relé**
- Capturas de la **lectura del MPU6050**

---

# IX. Conclusiones

El **ESP32 permite implementar sistemas de generación y adquisición analógica útiles para enseñanza y prototipado**.

Los experimentos mostraron:

- resolución limitada del DAC (8 bits)
- buena capacidad de resolución del ADC (12 bits)
- presencia de jitter debido al muestreo software

Para aplicaciones que requieran **mayor precisión o estabilidad**, se recomienda utilizar:

- adquisición mediante **DMA**
- **ADC continuo**
- acondicionamiento analógico externo.

---

# X. Nota técnica de carga de firmware (botón BOOT)

Durante la práctica, al intentar cargar el programa al ESP32 se observó que, en algunos intentos, el proceso fallaba si no se mantenía presionado el botón **BOOT** durante el inicio de la escritura.

Esto ocurre porque, para programar el chip, el ESP32 debe entrar en **modo bootloader (UART Download Mode)**. Si el circuito de auto-programación de la placa (control de líneas `EN` y `IO0`) no realiza correctamente la secuencia, el microcontrolador inicia en modo de ejecución normal y el cargador reporta error de sincronización.

Al mantener presionado **BOOT**, el pin `GPIO0` se fuerza al estado requerido para entrar en modo descarga, permitiendo que la carga del firmware se complete de forma correcta.

Factores comunes que agravan este problema:
- Cable USB de baja calidad o solo de carga.
- Alimentación inestable del puerto USB.
- Drivers USB-serial incorrectos o con fallas.
- Implementación deficiente del auto-reset en algunos clones de placas.

Como procedimiento práctico, cuando aparece el error de conexión, se recomienda iniciar la carga y mantener **BOOT** presionado hasta que el entorno indique que comenzó la escritura en flash.

---

# XI. Referencias bibliográficas

[1] Espressif Systems, *ESP32 Technical Reference Manual*, Espressif, 2024.

[2] Espressif Systems, *ESP32 Series Datasheet*, Espressif, 2024.

[3] Espressif Systems, *ESP-IDF Programming Guide*, [En línea]. Disponible en: https://docs.espressif.com/

[4] Arduino-ESP32 Core, *Arduino core for the ESP32*, [En línea]. Disponible en: https://github.com/espressif/arduino-esp32

[5] TDK InvenSense, *MPU-6000/MPU-6050 Product Specification*, 2013.


