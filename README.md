# Bateria-Dell-E7440-E7240-R3-2018-05-20
Documentacion completa del pack de bateria para intentar diagnosticar 

## Datos del pack

Tecnología: Li-ion
DC: 11.1 VDC
Potencia: 3200 mA/h  36 Wh

## Esquema del Circuito Global

![pack interno](imagen-1.png)

## BMS

![alt text](imagen-5.png)

### IC A2168

El chip A2168 no es un chip original de Texas Instruments (como los BQ estándar de Dell), sino un clon de diseño chino muy común en las baterías de reemplazo genéricas o réplicas (no OEM)

A nivel de software y tramas, este integrado emula exactamente el comportamiento y el mapa de registros de un chip clásico: el BQ20Z45 o el BQ20857

### IC 4407A

MOSFETs de Canal P (AO4407A) de potencia. Su función exclusiva en este BMS es actuar como interruptores electrónicos (llaves de paso). Uno controla la línea de Carga y el otro la de Descarga

![alt text](imagen-4.png)

![alt text](imagen-6.png)

### IC LP1K36A

En la topología de diseño de este tipo de baterías clonadas o genéricas, el circuito se divide de forma muy clara:

1. El chip principal A2168 (lado A) procesa la lógica, cuenta los ciclos, mide la capacidad y gestiona el protocolo de datos SMBus hacia la laptop.
2. El chip secundario LP1K36A (lado B) se encarga del monitoreo analógico puro en tiempo real. Monitorea de manera directa el balance de los voltajes de cada celda y controla las compuertas físicas de los MOSFETs 4407A ante eventos críticos.

![alt text](imagen-7.png)

### IC CUB

Regulador de voltaje LDO (Low-Dropout) de ultra-bajo consumo continuo

![alt text](imagen-8.png)

## Bateria

![alt text](imagen-2.png)

# Arduino

Programa pendiente en evaluar en equipo Arduino Nano + Display Matriz LCD

```java
#include <Wire.h>

// Dirección estándar I2C para Smart Batteries (0x0B)
#define BATTERY_ADDR 0x0B 

// Comandos estándar SBS (Smart Battery Specification)
#define CMD_VOLTAGE         0x09
#define CMD_CURRENT         0x0A
#define CMD_CHARGING_STATUS 0x15
#define CMD_MANUFACTURER_ACCESS 0x00

void setup() {
  Wire.begin();        // Une al bus I2C como maestro
  Serial.begin(9600);  // Abre el monitor serie
  Serial.println("--- Buscando Batería Dell ---");
}

uint16_t readWord(uint8_t cmd) {
  Wire.beginTransmission(BATTERY_ADDR);
  Wire.write(cmd);
  if (Wire.endTransmission(false) != 0) return 0xFFFF; // Error de conexión
  
  Wire.requestFrom(BATTERY_ADDR, 2);
  if (Wire.available() == 2) {
    uint8_t lowByte = Wire.read();
    uint8_t highByte = Wire.read();
    return (highByte << 8) | lowByte;
  }
  return 0xFFFF;
}

void loop() {
  uint16_t voltage = readWord(CMD_VOLTAGE);
  int16_t current = (int16_t)readWord(CMD_CURRENT);
  uint16_t status = readWord(CMD_CHARGING_STATUS);

  if (voltage == 0xFFFF) {
    Serial.println("Error: Batería no detectada. Revisa conexiones o el puente P-PRES.");
  } else {
    Serial.print("Voltaje Total: "); Serial.print(voltage / 1000.0); Serial.println(" V");
    Serial.print("Corriente: "); Serial.print(current); Serial.println(" mA");
    Serial.print("Status Hex: 0x"); Serial.println(status, HEX);
    
    // Si los bits de alarma críticos están en 1, hay falla permanente
    if (status & 0x4000) Serial.println("[ALERTA] TERMINATE_CHARGE_ALARM detectada.");
    if (status & 0x0800) Serial.println("[ALERTA] TERMINATE_DISCHARGE_ALARM detectada (Celdas muertas).");
  }
  
  delay(3000); // Muestreo cada 3 segundos
}
```
