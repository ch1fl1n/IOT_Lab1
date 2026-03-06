# Evaluación experimental de los módulos ADC y DAC del ESP32 mediante instrumentación básica

**Autores:** Daniel David Gomez Britto | Juan Camilo Silva Velasco  
**Curso:** Internet de las Cosas  
**Docente:** Fabian Mauricio Paez Rivera  
**Fecha:** 04/03/2026  

---

# Abstract

Se implementaron y evaluaron experimentos de generación y adquisición analógica usando un ESP32 (DAC interno de 8 bits y ADC hasta 12 bits), un generador de funciones y un osciloscopio. Se realizaron pruebas en bucle **DAC→ADC (loopback)** generando formas triangulares y senoidales y registrando parámetros prácticos: **Vmax/Vmin, escalonamiento por resolución del DAC, ruido y jitter temporal por muestreo software**.

Los resultados muestran la idoneidad del ESP32 para **enseñanza y prototipado** y evidencian limitaciones en **resolución, linealidad y velocidad efectiva**. Finalmente se proponen mejoras de software y acondicionamiento analógico para mejorar precisión y estabilidad.

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

### 1. Loopback DAC→ADC

Se generó una **onda triangular** usando el DAC y se leyó inmediatamente con el ADC.

El potenciómetro conectado al **GPIO35** permite controlar:

- **Amplitud**
- **Frecuencia**

---

### 2. Prueba con generador externo

Se aplicó una **señal senoidal de 1 kHz y 2 Vpp** al ADC para comparar la respuesta del sistema.

---

### 3. Mediciones con osciloscopio

Se registraron los parámetros:

- Vmax
- Vmin
- Vpp
- frecuencia
- forma de onda

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

---

## Simulación B — Sensor MPU6050

Se implementó comunicación **I2C por software**.

El sistema:

- despierta el sensor
- lee el registro **WHO_AM_I**
- obtiene valores del acelerómetro

### Pines usados

| Señal | Pin |
|------|------|
| SDA | 21 |
| SCL | 22 |

El registro **WHO_AM_I** normalmente devuelve:

```
0x68
```

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

# X. Trabajo futuro

Se proponen los siguientes pasos para continuar la experimentación:

1. Calibrar el DAC mediante mediciones reales de salida.
2. Implementar adquisición ADC con **DMA**.
3. Incorporar **amplificadores operacionales** como buffer.
4. Añadir **filtros pasa-bajo** para reducir aliasing.
5. Expandir la lectura del **MPU6050** para incluir todos los ejes del acelerómetro y giroscopio.

---
