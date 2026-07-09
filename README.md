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

### PinOut Pack

![alt text](imagen-4.png)

Pinout de Conexión (Mapeo de la Serigrafía)
- P+ / P+: Terminales del Positivo de Potencia (VCC). Ambos pines están puenteados en la placa para soportar la corriente de carga/descarga. Conecte aquí el cable positivo de su fuente o programador.
- C: Línea de reloj del bus de datos, corresponde a SCL (Serial Clock). Conéctelo al pin SCL de su interfaz (EV2300/EV2400/Arduino).
- D: Línea de datos del bus, corresponde a SDA (Serial Data). Conéctelo al pin SDA de su interfaz.
- P-PRES: Pin de presencia de sistema (System Present / System Detect). Para que el chip BMS despierte y abra la comunicación SMBus fuera de la laptop, debe puentear este pin directamente a Masa/GND (P-) utilizando un cable corto o un puente de soldadura temporal.
- ID: Pin de identificación de la batería (normalmente conectado a una resistencia interna térmica o de ID hacia masa). No suele requerirse para el flasheo básico.
- P- / P-: Terminales del Negativo de Potencia / Masa (GND). Conecte aquí el cable de tierra de su analizador y el puente de P-PRES

## Back de tarjeta BMS

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

Programa versión Beta, funcionando con servidor web 

```java
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <time.h>
#include <sys/time.h>
#include "esp_pm.h" 

#define I2C_SDA_PIN 4
#define I2C_SCL_PIN 5
#define LED_PIN 8

WebServer server(80);
bool bmsConectado = false;
unsigned long ultimoEscaneo = 0; 

const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE HTML><html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Kit de diagnóstico de baterias de portatiles v1.0</title>
  <style>
    body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; text-align: center; background-color: #2c3e50; color: #333; padding: 20px; margin: 0; }
    .card { background: #ffffff; padding: 25px; border-radius: 12px; box-shadow: 0px 8px 16px rgba(0,0,0,0.2); max-width: 450px; margin: 0 auto 30px auto; }
    h2 { color: #2c3e50; margin-top: 0; font-size: 1.4em;}
    .status { font-weight: bold; padding: 12px; border-radius: 8px; margin-bottom: 25px; font-size: 1.1em; transition: 0.3s; }
    .data-row { display: flex; justify-content: space-between; padding: 12px 0; border-bottom: 1px solid #eee; font-size: 1.1em; }
    .data-row:last-child { border-bottom: none; }
    .value { font-weight: bold; color: #2980b9; }
    
    /* Botón de Escaneo */
    .btn-scan { background-color: #3498db; color: white; border: none; padding: 12px 20px; font-size: 1em; font-weight: bold; border-radius: 8px; cursor: pointer; width: 100%; margin-top: 15px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); transition: 0.2s; }
    .btn-scan:hover { background-color: #2980b9; }
    .btn-scan:active { transform: scale(0.98); box-shadow: 0 2px 4px rgba(0,0,0,0.1); }

    /* Telemetría */
    .sys-card { background: #ecf0f1; padding: 15px; border-radius: 8px; margin-top: 25px; border: 1px dashed #bdc3c7; }
    .sys-title { font-size: 0.9em; font-weight: bold; color: #7f8c8d; margin-bottom: 10px; text-transform: uppercase; }
    .sys-row { display: flex; justify-content: space-between; font-size: 0.85em; color: #555; margin-bottom: 8px; align-items: center; }
    .sys-value { font-family: monospace; font-weight: bold; color: #34495e; }
    
    /* Semáforo y Badges */
    .badge { padding: 3px 8px; border-radius: 12px; font-size: 0.8em; margin-left: 8px; font-family: sans-serif;}
    .badge-green { background: #d4edda; color: #155724; }
    .badge-orange { background: #fff3cd; color: #856404; }
    .badge-red { background: #f8d7da; color: #721c24; }

    /* Diagrama de Hardware */
    .hw-container { display: flex; justify-content: space-between; align-items: stretch; margin-top: 15px; }
    .hw-board { width: 38%; border-radius: 8px; padding: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.15); display: flex; flex-direction: column; }
    .esp-board { background: #1a252f; color: #ecf0f1; border-top: 4px solid #3498db; }
    .bms-board { background: #229954; color: #ecf0f1; border-top: 4px solid #f1c40f; }
    .hw-title { font-weight: bold; font-size: 0.85em; border-bottom: 1px solid rgba(255,255,255,0.2); padding-bottom: 8px; margin-bottom: 12px; letter-spacing: 1px;}
    .hw-pin { background: rgba(0,0,0,0.3); padding: 5px 8px; border-radius: 4px; font-family: monospace; font-size: 0.75em; margin-bottom: 8px; display: flex; align-items: center; }
    .esp-board .hw-pin { justify-content: flex-end; }
    .bms-board .hw-pin { justify-content: flex-start; }
    .pin-dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin: 0 6px; }
    .dot-vcc { background: #e74c3c; }
    .dot-gnd { background: #7f8c8d; }
    .dot-sda { background: #3498db; }
    .dot-scl { background: #f1c40f; }
    .hw-wires { width: 20%; display: flex; flex-direction: column; justify-content: flex-start; padding-top: 42px; gap: 11px;}
    .wire { font-size: 0.9em; font-weight: bold; letter-spacing: -1px; text-align: center; }
    .diag-note { font-size: 0.75em; color: #c0392b; margin-top: 15px; text-align: left; font-weight: bold; background: #fadbd8; padding: 10px; border-radius: 6px;}

    .footer { font-size: 0.85em; color: #7f8c8d; margin-top: 25px; border-top: 1px solid #eee; padding-top: 15px;}
    .author { font-style: italic; color: #95a5a6; font-size: 0.8em; margin-top: 5px;}
  </style>
</head>
<body>
  
  <div class="card">
    <h2>🔋 Kit de diagnóstico de baterías v1.0 🔋</h2>
    <div id="statusBox" class="status" style="background-color: #f8d7da; color: #721c24;">🔴 Sin conexión</div>
    
    <div class="data-row"><span>⚡ Voltaje:</span> <span id="volt" class="value">0.00 V</span></div>
    <div class="data-row"><span>🔌 Corriente:</span> <span id="curr" class="value">0.00 A</span></div>
    <div class="data-row"><span>🔋 Capacidad:</span> <span id="perc" class="value">0 %</span></div>
    <div class="data-row"><span>🌡️ Temp. Batería:</span> <span id="temp" class="value">0.0 °C</span></div>
    
    <button class="btn-scan" onclick="forzarEscaneo()">🔍 Escanear Ahora</button>

    <div class="sys-card">
      <div class="sys-title">⚙️ Telemetría del ESP32-C3</div>
      <div class="sys-row"><span>🌡️ Temp. CPU:</span> <span id="sys_temp" class="sys-value">-- °C</span></div>
      <div class="sys-row"><span>🧠 RAM Libre:</span> <span id="sys_ram" class="sys-value">-- KB</span></div>
      <div class="sys-row"><span>📶 Clientes Wi-Fi:</span> <span id="sys_wifi" class="sys-value">--</span></div>
      <div class="sys-row"><span>⏱️ Uptime:</span> <span id="sys_uptime" class="sys-value">00:00:00</span></div>
    </div>

    <div class="footer">Sistema ESP32-C3 Super Mini</div>
    <div class="author">by sanchezluys 07-2026 + gemini ✨</div>
  </div>

  <div class="card">
    <h2>🔌 Diagrama Físico de Conexión</h2>
    <div class="hw-container">
      <div class="hw-board esp-board">
        <div class="hw-title">ESP32-C3 MINI</div>
        <div class="hw-pin">5V PIN <span class="pin-dot dot-vcc"></span></div>
        <div class="hw-pin">GND <span class="pin-dot dot-gnd"></span></div>
        <div class="hw-pin">GPIO 4 <span class="pin-dot dot-sda"></span></div>
        <div class="hw-pin">GPIO 5 <span class="pin-dot dot-scl"></span></div>
      </div>
      <div class="hw-wires">
        <div class="wire" style="color: #e74c3c;">━━━▶</div>
        <div class="wire" style="color: #7f8c8d;">━━━▶</div>
        <div class="wire" style="color: #3498db;">◀━━▶</div>
        <div class="wire" style="color: #f1c40f;">━━━▶</div>
      </div>
      <div class="hw-board bms-board">
        <div class="hw-title">BMS BATERÍA</div>
        <div class="hw-pin"><span class="pin-dot dot-vcc"></span> V+ (5V)</div>
        <div class="hw-pin"><span class="pin-dot dot-gnd"></span> GND (-)</div>
        <div class="hw-pin"><span class="pin-dot dot-sda"></span> SDA (Data)</div>
        <div class="hw-pin"><span class="pin-dot dot-scl"></span> SCL (Reloj)</div>
      </div>
    </div>
    <div class="diag-note">
      ⚠️ IMPORTANTE: Recuerda conectar el pin SYS_PRES (System Present) de la batería a GND para despertar el BMS.
    </div>
  </div>

  <script>
    // CORREGIDO: Se eliminó el 'void' incorrecto para JavaScript
    function actualizarUI(data) {
        let statusBox = document.getElementById('statusBox');
        if(data.conectado) {
          statusBox.innerText = "🟢 Conconnected (BMS Detectado)";
          statusBox.style.backgroundColor = "#d4edda";
          statusBox.style.color = "#155724";
        } else {
          statusBox.innerText = "🔴 Sin conexión";
          statusBox.style.backgroundColor = "#f8d7da";
          statusBox.style.color = "#721c24";
        }
        
        document.getElementById('volt').innerText = data.voltaje + " V";
        document.getElementById('curr').innerText = data.corriente + " A";
        document.getElementById('perc').innerText = data.porcentaje + " %";
        document.getElementById('temp').innerText = data.temperatura + " °C";

        let tempVal = parseFloat(data.sys_temp);
        let tempHtml = data.sys_temp + " °C ";
        if(tempVal < 65.0) {
           tempHtml += "<span class='badge badge-green'>🟢 Normal</span>";
        } else if(tempVal >= 65.0 && tempVal < 80.0) {
           tempHtml += "<span class='badge badge-orange'>🟠 Elevada</span>";
        } else {
           tempHtml += "<span class='badge badge-red'>🔴 Crítica</span>";
        }
        document.getElementById('sys_temp').innerHTML = tempHtml;

        document.getElementById('sys_ram').innerText = data.sys_ram + " KB (" + data.sys_ram_pct + "% libre)";
        document.getElementById('sys_wifi').innerText = data.sys_wifi;
        document.getElementById('sys_uptime').innerText = data.sys_uptime;
    }

    setInterval(function() {
      fetch('/datos').then(response => response.json()).then(data => {
         actualizarUI(data);
      });
    }, 2000);

    function forzarEscaneo() {
      let btn = document.querySelector('.btn-scan');
      btn.innerText = "🔍 Escaneando...";
      btn.disabled = true;
      
      fetch('/escanear').then(response => response.json()).then(data => {
         actualizarUI(data);
         btn.innerText = "🔍 Escanear Ahora";
         btn.disabled = false;
      }).catch(err => {
         btn.innerText = "🔍 Escanear Ahora";
         btn.disabled = false;
      });
    }
  </script>
</body>
</html>
)rawliteral";

void ejecutarEscaneoFisico() {
  digitalWrite(LED_PIN, LOW); 
  bool encontrado = false;
  for(byte address = 1; address < 127; address++ ) {
    Wire.beginTransmission(address);
    if (Wire.endTransmission() == 0) {
      if(address == 0x0B) { encontrado = true; }
    }
  }
  bmsConectado = encontrado;
  digitalWrite(LED_PIN, HIGH); 
}

void manejarRaiz() { server.send(200, "text/html", index_html); }

String generarJSONDatos() {
  float cpuTemp = temperatureRead(); 
  float ramLibre = ESP.getFreeHeap() / 1024.0; 
  float ramTotal = ESP.getHeapSize() / 1024.0;
  int ramPorcentaje = (int)((ramLibre / ramTotal) * 100.0);
  int clientesWiFi = WiFi.softAPgetStationNum(); 
  
  unsigned long segTotales = millis() / 1000;
  int horas = segTotales / 3600;
  int minutes = (segTotales % 3600) / 60;
  int segundos = segTotales % 60;
  char uptimeStr[15];
  sprintf(uptimeStr, "%02d:%02d:%02d", horas, minutes, segundos);

  String json = "{";
  json += "\"conectado\":" + String(bmsConectado ? "true" : "false") + ",";
  json += "\"voltaje\":\"0.00\",";
  json += "\"corriente\":\"0.00\",";
  json += "\"porcentaje\":\"0\",";
  json += "\"temperatura\":\"0.0\",";
  json += "\"sys_temp\":\"" + String(cpuTemp, 1) + "\",";
  json += "\"sys_ram\":\"" + String(ramLibre, 1) + "\",";
  json += "\"sys_ram_pct\":\"" + String(ramPorcentaje) + "\",";
  json += "\"sys_wifi\":\"" + String(clientesWiFi) + "\",";
  json += "\"sys_uptime\":\"" + String(uptimeStr) + "\"";
  json += "}";
  return json;
}

void manejarDatos() {
  server.send(200, "application/json", generarJSONDatos());
}

void manejarEscaneoManual() {
  ejecutarEscaneoFisico(); 
  server.send(200, "application/json", generarJSONDatos()); 
}

void setup() {
  setCpuFrequencyMhz(80); 

  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, HIGH); 
  delay(1000);

  struct tm tm;
  tm.tm_year = 2026 - 1900; tm.tm_mon = 7 - 1; tm.tm_mday = 9;           
  tm.tm_hour = 14; tm.tm_min = 15; tm.tm_sec = 0;            
  time_t t = mktime(&tm);
  struct timeval current_time = { .tv_sec = t };
  settimeofday(&current_time, NULL);

  Serial.println("\n--- Kit de Diagnóstico Pro v1.0 (80MHz Opm) ---");
  
  WiFi.mode(WIFI_AP); 
  WiFi.setTxPower(WIFI_POWER_8_5dBm); 
  WiFi.softAP("BMS_Diagnostico", "12345678"); 
  WiFi.setSleep(true); 

  IPAddress IP = WiFi.softAPIP();
  Serial.print("Red creada con éxito. IP: ");
  Serial.println(IP); 

  server.on("/", manejarRaiz);
  server.on("/datos", manejarDatos);
  server.on("/escanear", manejarEscaneoManual); 
  server.begin();
  Serial.println("Servidor Web iniciado.");

  Wire.begin(I2C_SDA_PIN, I2C_SCL_PIN);
}

void loop() {
  server.handleClient();

  if (millis() - ultimoEscaneo > 5000) {
    ultimoEscaneo = millis();
    ejecutarEscaneoFisico();
    
    time_t now; struct tm timeinfo; time(&now); localtime_r(&now, &timeinfo);
    char timeString[25]; strftime(timeString, sizeof(timeString), "%d/%m/%Y %H:%M:%S", &timeinfo);
    Serial.print("["); Serial.print(timeString); Serial.print("] Escaneo Auto: ");
    Serial.println(bmsConectado ? "BMS 0x0B Ok" : "Sin conexion");
  }

  delay(50); 
}
```

<img width="720" height="1600" alt="imagen" src="https://github.com/user-attachments/assets/988beea5-137a-48c6-9b8d-d34d28683ec2" />

<img width="720" height="1600" alt="imagen" src="https://github.com/user-attachments/assets/ebf46343-1d8f-4276-9200-55b04871e023" />



