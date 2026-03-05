# Formato IEEE — Informe de Laboratorio 1 (ESP32, ADC/DAC)

**Título del trabajo:** Evaluación experimental de los módulos ADC y DAC del ESP32 mediante instrumentación básica  
**Autores:** Daniel David Gomez Britto | Juan Camilo Silva Velasco  
**Curso:** Internet de las Cosas  
**Docente:** Fabian Mauricio Paez Rivera  
**Fecha:** 04/03/2026

## Abstract
En este laboratorio se implementaron y evaluaron sistemas básicos de adquisición y generación de señales con un módulo ESP32, apoyados por un osciloscopio y un generador de funciones. Se trabajó con los módulos ADC y DAC del microcontrolador para analizar su principio de operación, su comportamiento experimental y sus limitaciones prácticas. Los resultados muestran que el ESP32 es adecuado para aplicaciones educativas y de prototipado, aunque presenta restricciones de resolución, linealidad, ruido y velocidad efectiva de muestreo frente a instrumentación profesional.


## I. Introducción
La adquisición y generación de señales analógicas mediante microcontroladores permite comprender conceptos fundamentales de instrumentación electrónica. En esta práctica se estudió el comportamiento del ADC y DAC del ESP32 mediante pruebas con potenciómetro, generador de señales y osciloscopio, además de una validación cruzada entre grupos (generador ↔ osciloscopio). Como complemento, se documentaron una simulación de control por relé en Wokwi y un flujo básico de almacenamiento de datos en microSD para escenarios de IoT.

## II. Fundamento teórico
### A. Conversión Analógico-Digital (ADC)
Un ADC transforma una señal analógica continua en una representación digital discreta mediante tres etapas: muestreo, cuantización y codificación. Según Nyquist-Shannon, para evitar aliasing se requiere una frecuencia de muestreo al menos doble de la máxima frecuencia de la señal de entrada.

Para un ADC de \(n\) bits existen \(2^n\) niveles. En 12 bits, el rango digital es de 0 a 4095.

El error de cuantización corresponde a la diferencia entre el valor analógico real y el nivel digital asignado. En términos generales, al aumentar el número de bits disminuye el tamaño del paso de cuantización y mejora la resolución.

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

### E. Implementación de software principal (código base de la práctica)
El programa principal implementó un lazo DAC→ADC en un único ESP32 para validar generación y adquisición de forma controlada. La configuración utilizada fue:
- `GPIO25` como salida DAC (`DAC1`).
- `GPIO34` como entrada ADC de realimentación (loopback).
- `GPIO35` como entrada ADC para potenciómetro de control.

La señal generada fue triangular, actualizada por incrementos/decrementos discretos (`stepAbs` y `stepSign`) entre 0 y un valor máximo de DAC. Se evaluaron dos modos de operación:
- **Modo amplitud:** el potenciómetro ajusta el valor máximo del DAC (aprox. 10 a 255).
- **Modo frecuencia:** el potenciómetro ajusta el tamaño del paso (aprox. 1 a 20), modificando la velocidad de barrido de la onda.

En cada iteración se realizó `dacWrite()` y lectura inmediata con `analogRead()`, además de conversión aproximada a voltaje para correlacionar niveles digitales con magnitudes eléctricas.

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

### E. Análisis integrado de limitaciones observadas
Los resultados de las cuatro pruebas son coherentes entre sí y permiten identificar el origen de los principales errores:
- **Cuantización:** visible en la generación DAC (8 bits) y en la adquisición ADC (12 bits), especialmente cerca de máximos y mínimos de la señal.
- **Muestreo no determinístico:** el ciclo `analogRead()` + procesamiento + envío serial introduce variación temporal entre muestras (jitter).
- **Respuesta en frecuencia limitada por software:** el hardware puede operar más rápido, pero la implementación en Arduino restringe la tasa efectiva en pruebas continuas.
- **Sensibilidad a ruido y referencia:** variaciones de alimentación y ruido interno impactan estabilidad, amplitud y repetibilidad.

### F. Relación entre código implementado y comportamiento medido
La estructura del código principal explica directamente los fenómenos observados en laboratorio:
- El control por pasos discretos en DAC produce escalonamiento visible en la salida analógica.
- El ajuste de `stepAbs` cambia la pendiente de la onda triangular y, por tanto, su frecuencia aparente.
- La lectura secuencial del ADC en el mismo lazo de ejecución incrementa la dependencia temporal del sistema respecto al procesamiento y la comunicación serial.

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
Se validó una conmutación de carga controlada por ESP32 mediante una entrada con `INPUT_PULLUP`, una salida digital hacia el relé y un LED conectado al contacto normalmente abierto (NO).

Hallazgos principales:
- El comportamiento lógico fue consistente: al accionar el pulsador se energiza el relé y se enciende la carga.
- Se confirma el valor didáctico del relé como elemento de aislamiento entre control y carga.
- Para implementación física deben considerarse rebote, tiempo de conmutación, ruido electromagnético y vida útil mecánica de contactos.

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

Ejemplo de registro:
- `2026-03-01 10:30:15, Temperatura: 25.3 C`
- `2026-03-01 10:31:15, Temperatura: 25.4 C`
- `2026-03-01 10:32:15, Temperatura: 25.2 C`

## IX. Conclusiones
El ESP32 permite implementar funciones de adquisición y generación de señales con resultados coherentes para aprendizaje y prototipado. No obstante, sus límites en resolución, linealidad, ruido y velocidad efectiva impiden equipararlo con instrumentación profesional. En esta práctica se evidenció que el desempeño final depende tanto del hardware como de la estrategia de medición y del software de adquisición.

En términos formativos, el laboratorio permitió relacionar teoría y práctica en conceptos clave (cuantización, aliasing, jitter y linealidad), y establecer criterios técnicos para mejorar versiones futuras del sistema, como muestreo continuo, separación entre adquisición y visualización serial, y acondicionamiento analógico de entrada.
