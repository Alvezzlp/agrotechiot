# 🌿 AgroTech — Estufa Inteligente Conectada (IoT)

> **Protótipo de monitoramento microclimático e automação para estufas agrícolas usando ESP32 e ThingSpeak.**

---

## 📌 Sobre o Projeto

O **AgroTech** é uma solução de IoT desenvolvida para monitorar continuamente os parâmetros ambientais críticos de uma estufa (temperatura, umidade do ar e iluminação). 

O sistema identifica anomalias em tempo real e dispara alertas locais (sonoros e visuais)[cite: 1, 2], além de transmitir toda a telemetria via Wi-Fi para um dashboard na nuvem, permitindo o acompanhamento remoto do cultivo[cite: 1, 2].

---

## 🚀 Funcionalidades

- 🌡️ **Leitura Térmica e Hídrica:** Coleta contínua de temperatura e umidade com o sensor DHT22[cite: 1, 2].
- ☀️ **Monitoramento de Luminosidade:** Leitura dos níveis de luz com sensor LDR[cite: 1, 2].
- 🚨 **Alerta Local Automático:** Ativa LED Vermelho + Buzzer se a temperatura passar de **30°C** ou se a luz cair abaixo do limite seguro.
- 📊 **Telemetria Cloud:** Envio dos dados a cada 15 segundos para o **ThingSpeak** via HTTP[cite: 1, 2].

---

## 🧰 Hardware & Mapeamento de Pinos

| Componente | Função | Pino no ESP32 |
| :--- | :--- | :---: |
| **ESP32 NodeMCU** | Microcontrolador principal + Wi-Fi | -- |
| **DHT22** | Sensor de Temperatura e Umidade | `GPIO 4` |
| **LDR** | Sensor de Luminosidade (Analógico) | `GPIO 34` |
| **LED Verde** | Sinalização de Status OK | `GPIO 26` |
| **LED Vermelho** | Sinalização de Alerta | `GPIO 27` |
| **Buzzer** | Alarme Sonoro Local | `GPIO 25` |

---

## 🌐 Simulação Interativa (Wokwi)

Você pode simular este projeto diretamente no navegador sem precisar de componentes físicos!

[![Simular no Wokwi](https://img.shields.io/badge/▶️_Testar_no-Wokwi-blue?style=for-the-badge&logo=wokwi)](https://wokwi.com/projects/476141274344863745)

🔗 **Link direto do circuito:** [Clique aqui para abrir a simulação no Wokwi](https://wokwi.com/projects/476141274344863745)

> **Como testar:**
> 1. Clique no botão de **Play** verde para iniciar a simulação.
> 2. Interaja com o **DHT22** subindo a temperatura acima de 30°C ou ajuste o **LDR** para escuro (< 1500).
> 3. Observe o LED Vermelho acender e o Buzzer tocar!

---

## 💻 Código-Fonte (C++)

<details>
<summary><b>Clique para expandir o código do ESP32</b></summary>

```cpp
#include <WiFi.h>
#include "ThingSpeak.h"
#include <DHT.h>

#define DHTPIN 4
#define DHTTYPE DHT22
#define LDR_PIN 34
#define LED_VERDE 26
#define LED_VERMELHO 27
#define BUZZER_PIN 25

DHT dht(DHTPIN, DHTTYPE);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

unsigned long CHANNEL_ID = 3509320; 
const char* WRITE_API_KEY = "I7M86IW46DOLD7IN"; 

WiFiClient client;

const float TEMP_LIMITE = 30.0;
const int LUZ_LIMITE = 1500;

const unsigned long INTERVALO_ENVIO = 15000;
unsigned long ultimoEnvio = 0;

void setup() {
  Serial.begin(115200);

  pinMode(LED_VERDE, OUTPUT);
  pinMode(LED_VERMELHO, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  dht.begin();
  WiFi.mode(WIFI_STA);
  ThingSpeak.begin(client);

  conectarWiFi();
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    conectarWiFi();
  }

  float temperatura = dht.readTemperature();
  float umidade = dht.readHumidity();
  int luminosidade = analogRead(LDR_PIN);

  if (isnan(temperatura) || isnan(umidade)) {
    Serial.println("Falha ao ler o DHT22!");
    delay(1000);
    return;
  }

  bool alerta = (temperatura > TEMP_LIMITE || luminosidade < LUZ_LIMITE);

  if (alerta) {
    digitalWrite(LED_VERMELHO, HIGH);
    digitalWrite(LED_VERDE, LOW);
    digitalWrite(BUZZER_PIN, HIGH);
  } else {
    digitalWrite(LED_VERMELHO, LOW);
    digitalWrite(LED_VERDE, HIGH);
    digitalWrite(BUZZER_PIN, LOW);
  }

  if (millis() - ultimoEnvio >= INTERVALO_ENVIO || ultimoEnvio == 0) {
    ThingSpeak.setField(1, temperatura);
    ThingSpeak.setField(2, umidade);
    ThingSpeak.setField(3, luminosidade);

    int httpCode = ThingSpeak.writeFields(CHANNEL_ID, WRITE_API_KEY);
    if (httpCode == 200) {
      Serial.println("-> Dados enviados ao ThingSpeak com sucesso!");
    }
    ultimoEnvio = millis();
  }

  delay(500);
}

void conectarWiFi() {
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
}
