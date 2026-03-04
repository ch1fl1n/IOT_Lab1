# Manejo de archivos y almacenamiento en ESP32 (microSD)

## Resumen

El ESP32 puede guardar datos en una tarjeta **SD/microSD** usando interfaz **SPI**, lo que permite registrar mediciones, eventos o logs sin depender de conexión a internet.

Según la documentación de Wokwi para `wokwi-microsd-card`:

- La tarjeta se conecta por pines SPI: `SCK`, `MISO (DO)`, `MOSI (DI)`, `CS`, además de `VCC` y `GND`.
- En simulación, Wokwi crea automáticamente un sistema de archivos **FAT16** al iniciar.
- Por defecto, puede copiar archivos del proyecto a la SD simulada.
- Se puede usar la librería `SD` o `SdFat` para abrir, escribir, leer y listar archivos.

En una práctica de IoT, esto se usa para:

- Guardar datos de sensores en formato `.txt` o `.csv`.
- Tener respaldo local cuando no hay WiFi.
- Leer configuraciones desde archivos al arrancar.

---

## Conexión típica ESP32 ↔ microSD (SPI)

Ejemplo común en ESP32 (bus VSPI):

- `SCK` → GPIO `18`
- `MISO/DO` → GPIO `19`
- `MOSI/DI` → GPIO `23`
- `CS` → GPIO `5` (puede cambiar)
- `VCC` → `3.3V` (o según el módulo)
- `GND` → `GND`

> Nota: revisa el voltaje de tu módulo SD. Muchos módulos para Arduino incluyen regulación y conversión de nivel, pero no todos.

---

## Ejemplo simple (crear, escribir y leer un archivo)

```cpp
#include <SPI.h>
#include <SD.h>

const int SD_CS = 5;

void setup() {
  Serial.begin(115200);

  // Inicializa SPI en pines VSPI del ESP32
  SPI.begin(18, 19, 23, SD_CS); // SCK, MISO, MOSI, CS

  if (!SD.begin(SD_CS)) {
    Serial.println("Error: no se pudo inicializar la tarjeta SD");
    return;
  }

  Serial.println("SD inicializada correctamente");

  // 1) Escribir datos
  File archivo = SD.open("/datos.txt", FILE_WRITE);
  if (archivo) {
    archivo.println("timestamp,temperatura");
    archivo.println("1000,24.8");
    archivo.println("2000,25.1");
    archivo.close();
    Serial.println("Datos guardados en /datos.txt");
  } else {
    Serial.println("No se pudo abrir /datos.txt para escritura");
  }

  // 2) Leer datos
  archivo = SD.open("/datos.txt", FILE_READ);
  if (archivo) {
    Serial.println("Contenido de /datos.txt:");
    while (archivo.available()) {
      Serial.write(archivo.read());
    }
    archivo.close();
  } else {
    Serial.println("No se pudo abrir /datos.txt para lectura");
  }
}

void loop() {
  // En este ejemplo no se usa loop
}
```

---

## Referencia

- Wokwi microSD Card: https://docs.wokwi.com/parts/wokwi-microsd-card
