# Formato IEEE — Informe de Laboratorio 1 (ESP32, ADC/DAC)

**Título del trabajo:** Evaluación experimental de los módulos ADC y DAC del ESP32 mediante instrumentación básica y prueba cruzada entre grupos  
**Autores:** Daniel David Gomez Britto | Juan Camilo SIlva Velasco  
**Curso:** Internet de las cosas
**Docente:** Fabian Mauricio Paez Rivera
**Fecha:** 04/03/2026

## Abstract
En este laboratorio se implementaron y evaluaron sistemas básicos de adquisición y generación de señales con un módulo ESP32, apoyados por un osciloscopio y un generador de funciones. Se trabajó con los módulos ADC y DAC del microcontrolador para analizar su principio de operación, su comportamiento experimental y sus limitaciones prácticas. Los resultados muestran que el ESP32 es adecuado para aplicaciones educativas y de prototipado, aunque presenta restricciones de resolución, linealidad, ruido y velocidad efectiva de muestreo frente a instrumentación profesional.

## I. Introducción
La adquisición y generación de señales analógicas mediante microcontroladores permite comprender conceptos fundamentales de instrumentación electrónica. En esta práctica se estudió el comportamiento del ADC y DAC del ESP32 mediante pruebas con potenciómetro, generador de señales y osciloscopio, además de una validación cruzada entre grupos (generador ↔ osciloscopio).

## II. Fundamento teórico
### A. Conversión Analógico-Digital (ADC)
Un ADC transforma una señal analógica continua en una representación digital discreta mediante tres etapas: muestreo, cuantización y codificación. Según Nyquist-Shannon, para evitar aliasing se requiere una frecuencia de muestreo al menos doble de la máxima frecuencia de la señal de entrada.

Para un ADC de \(n\) bits existen \(2^n\) niveles. En 12 bits, el rango digital es de 0 a 4095.

### B. Conversión Digital-Analógica (DAC)
El DAC realiza la operación inversa al ADC, generando un voltaje analógico a partir de un valor digital discreto. En el ESP32, el DAC es de 8 bits (0 a 255), por lo que la salida presenta escalonamiento visible en señales continuas.

Modelo ideal:

$$
V_{out}=\frac{D}{2^n-1}\,V_{ref}
$$

### C. Módulos ADC/DAC en ESP32
- **ADC:** dos módulos (ADC1 y ADC2), resolución hasta 12 bits.
- **DAC:** dos canales (GPIO25 y GPIO26), resolución de 8 bits.

Limitaciones observables en práctica:
- DNL/INL,
- ruido interno,
- variación de referencia,
- interferencia por Wi-Fi (especialmente en ADC2),
- limitación de velocidad efectiva por software y transmisión serial.

### D. Método SAR
El ADC del ESP32 utiliza aproximación sucesiva (SAR): evalúa el MSB, compara con referencia interna y ajusta bit a bit hasta converger al código digital final. Es eficiente en consumo y tiempo de conversión.

## III. Metodología experimental
### A. Prueba 1 — ADC con potenciómetro
- Entrada ADC: GPIO34
- Resolución: 12 bits
- Atenuación: 11 dB
- Rango esperado: ~0 a 3.3 V

Se varió manualmente el potenciómetro y se observó la lectura en Serial Monitor/Plotter.

### B. Prueba 2 — ADC con generador de señales
- Señal de entrada: seno de 1 kHz, 2 Vpp, offset 1.65 V
- Entrada ADC: GPIO34

Se evaluó forma de onda y respuesta al incrementar frecuencia.

### C. Prueba 3 — DAC con osciloscopio
Se analizaron señales senoidal, triangular y cuadrada para diferentes frecuencias.

### D. Prueba 4 — Validación cruzada entre grupos
Se conectó la salida DAC del ESP32 emisor a la entrada ADC del ESP32 receptor con GND común para evaluar generación y adquisición en cadena real.

## IV. Resultados y discusión
### A. ADC con potenciómetro
- Rango observado: 0 a 4095
- Comportamiento: aproximadamente lineal
- Fluctuaciones: ±5 a ±10 LSB

Resolución teórica:

$$
LSB_{ADC}=\frac{3.3\,V}{4095}\approx 0.805\,mV
$$

Los datos son consistentes con la teoría para ADC de 12 bits, con ruido atribuible al convertidor y a la referencia.

### B. ADC con generador real
- Senoide claramente visible a 1 kHz
- Respuesta reconocible hasta ~3–5 kHz
- Distorsión marcada >10 kHz
- Aliasing visible >15 kHz
- Error de amplitud aproximado: 2–3%

Conclusión: la tasa efectiva de muestreo con `analogRead()` + serial está por debajo de la capacidad máxima teórica del hardware.

### C. DAC con osciloscopio
- **Senoidal:** buena forma a baja frecuencia, escalones en media y degradación en alta frecuencia.
- **Triangular:** linealidad aceptable, redondeo en vértices por tiempo de establecimiento.
- **Cuadrada:** flancos limitados por `slew rate`, ligero overshoot y ruido en niveles.

### D. Prueba cruzada entre grupos
La señal recibida mostró escalonamiento por doble cuantización (DAC de 8 bits en emisor + ADC de 12 bits en receptor), además de jitter de muestreo y ausencia de filtro anti-aliasing.

## V. Comparación con instrumentación profesional
**Tabla 1. ESP32 vs equipo profesional**

| Aspecto | Sistema ESP32 | Equipo profesional |
|---|---|---|
| Resolución | Escalones visibles | Señal suave/alta resolución |
| Amplitud | Rango fijo (~0–3.3 V) | Rango configurable |
| Frecuencia útil | Limitada (Hz–kHz, según implementación) | Amplia (hasta MHz/GHz) |
| Ruido | Presente | Bajo y mejor blindaje |
| Precisión | Offset/ganancia y no linealidades | Calibración de alta precisión |
| Estabilidad | Sensible a alimentación/entorno | Mayor estabilidad |

## VI. Respuestas de la guía (preguntas del informe)
### 1) Características y funcionamiento general de ADC/DAC (con enfoque en ESP32)
- El **ADC** convierte voltaje analógico en código digital (ESP32: hasta 12 bits, ADC1/ADC2).
- El **DAC** convierte código digital en voltaje analógico (ESP32: 8 bits, GPIO25/GPIO26).
- Son adecuados para docencia/prototipado, con límites en precisión y ruido frente a equipos de laboratorio.

### 2) ¿Se puede alcanzar la máxima velocidad de muestreo del ADC desde código?
En Arduino estándar, de forma sostenida, no normalmente. `analogRead()`, procesamiento en CPU y transmisión serial reducen la tasa efectiva. Para mejorar se recomienda:
- ADC en modo continuo,
- DMA,
- periférico I2S,
- implementación en ESP-IDF.

### 3) ¿Cómo funciona el método SAR?
Muestrea la entrada y aplica una búsqueda binaria interna: prueba bits desde MSB hasta LSB, comparando cada intento contra una referencia. El resultado final es el código digital equivalente al voltaje de entrada.

### 4) ¿Cómo implementar un DAC con resistencias y salidas digitales?
Con una red **R-2R**:
- dos valores resistivos (R y 2R),
- cada bit conmuta a `Vref` o GND,
- la salida es proporcional al código binario.

Requisitos:
- tolerancia ideal ≤1%,
- relación 2:1 bien ajustada,
- estabilidad térmica,
- rango típico práctico: R entre 1 kΩ y 10 kΩ.

### 5) ¿Por qué difiere respecto a equipos profesionales?
Por menor resolución, limitaciones de conversión/muestreo, no linealidades, ruido de alimentación y restricciones del software de adquisición.

## VII. Simulación virtual (Wokwi): control de LED mediante relé
Se validó una conmutación de carga controlada por ESP32:
- pulsador en entrada con `INPUT_PULLUP`,
- relé activado por salida digital,
- LED como carga en contacto NO.

Hallazgos:
- funcionamiento correcto de aislamiento y conmutación,
- limitaciones de relé: rebote, tiempo de conmutación, vida mecánica y ruido electromagnético.

## VIII. Manejo de archivos en ESP32 con microSD (resumen solicitado)
### A. Interfaz y conexiones recomendadas (SPI)
- CS → GPIO5
- MOSI (DI) → GPIO23
- MISO (DO) → GPIO19
- SCK → GPIO18
- VCC → 3.3 V
- GND → GND

### B. Flujo de uso
1. Conexión física SPI
2. Inicialización con `SD.h`
3. Apertura/escritura/lectura/cierre de archivos

### C. Ejemplo de aplicación
Data logger de temperatura que almacena timestamp + valor en `datos.txt` para análisis posterior.

## IX. Conclusiones
El ESP32 permite implementar funciones de adquisición y generación de señales con resultados coherentes para aprendizaje y prototipado. No obstante, sus límites en resolución, linealidad, ruido y velocidad efectiva impiden equipararlo con instrumentación profesional. La práctica permitió identificar claramente qué errores son atribuibles al hardware y cuáles al software/estrategia de medición.

## X. Trabajo futuro
- Muestreo continuo con DMA/I2S.
- Filtrado digital y anti-aliasing analógico.
- Calibración de ganancia/offset.
- Reducción de jitter y desacople de transmisión serial del lazo de adquisición.

---

### Nota de edición
Para entrega formal IEEE en Word/LaTeX, conserva esta estructura y aplica plantilla IEEE con doble columna, numeración de figuras/tablas y referencias bibliográficas según el formato del curso.
