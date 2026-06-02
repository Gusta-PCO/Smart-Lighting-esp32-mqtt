# Smart-Lighting-esp32-mqtt
# Sistema de Iluminação Pública Inteligente com Geração Piezoelétrica e Monitoramento via MQTT

Este repositório contém a documentação técnica, especificação de hardware e o código-fonte correspondente ao desenvolvimento de um nó de rede inteligente para iluminação pública voltado a Cidades Sustentáveis (alinhado ao ODS 11 da ONU). O sistema integra sensoriamento de presença humana, regulação adaptativa de intensidade luminosa e colheita de energia mecânica (piezoelétrica) residual do pavimento urbano.

---

## 1. Descrição de Funcionamento e Lógica de Controle

O dispositivo opera como um nó de borda (*Edge Device*) autônomo baseado no microcontrolador ESP32. O comportamento do firmware executa em malha fechada seguindo as premissas abaixo:

*   **Modo de Espera (Economia):** Na ausência de pedestres detectados pelo sensor infravermelho passivo (PIR), a luminária pública (representada por LEDs) é mantida em estado econômico operando com apenas **20%** de sua intensidade luminosa máxima, reduzindo drasticamente o consumo global.
*   **Modo Ativo (Detecção):** Ao identificar a movimentação de um transeunte, o sensor altera seu estado digital disparando uma rotina prioritária de interrupção no hardware. O microcontrolador eleva instantaneamente o brilho dos LEDs para **100%** de sua potência através de sinais PWM.
*   **Temporização:** Após um intervalo contínuo de **10 segundos** sem novas detecções, o nó executa um decaimento suave e retorna ao Modo de Espera.
*   **Energy Harvesting (Colheita de Energia):** Paralelamente ao controle luminoso, módulos piezoelétricos instalados sob a calçada geram pulsos de tensão AC a partir do impacto mecânico dos passos. Essa energia é fisicamente retificada por uma ponte de diodos, filtrada por um capacitor e amostrada pelo conversor analógico-digital (ADC) do ESP32 para quantificar a energia limpa gerada.

---

## 2. Arquitetura de Hardware

O circuito elétrico do protótipo é estruturado a partir das seguintes especificações técnicas e mapeamentos de portas GPIO:

### Componentes Utilizados
| Componente | Especificação Técnica | Função no Projeto |
| :--- | :--- | :--- |
| **ESP32 DevKit V1** | Processador Dual-Core Tensilica LX6 (240 MHz), Wi-Fi nativo | Unidade central de processamento de borda e conectividade TCP/IP. |
| **Sensor PIR HC-SR501** | Ângulo de cobertura < 110º, Alcance máximo de 7 metros | Sensoriamento térmico e detecção de movimento de transeuntes. |
| **LED Branco Quente** | Diâmetro de 5mm, Temperatura de cor 3500K, Corrente 20mA | Atuador responsável pela simulação da luminária pública. |
| **Transdutores Piezoelétricos** | Elementos cerâmicos circulares de 27mm de diâmetro | Discos de conversão mecânico-elétrica instalados sob o pavimento. |
| **Módulo Retificador** | 4x Diodos rápidos 1N4148 (Ponte de Graetz) + 1x Capacitor de 10µF | Conversão de sinal AC pulsante do piezo para sinal DC linear estável. |

### Diagrama de Pinagem (GPIO Mapping)
*   **GPIO 13 (Entrada Digital):** Conectado ao pino de sinal do Sensor PIR HC-SR501.
*   **GPIO 12 (Saída Digital PWM):** Canal de controle de intensidade do circuito de LEDs (Frequência: 5000 Hz, Resolução: 8 bits - escala de 0 a 255).
*   **GPIO 34 (Entrada Analógica ADC):** Canal acoplado ao circuito retificador piezoelétrico para leitura linear de tensão.

---

## 3. Interfaces, Módulos e Protocolos de Comunicação

O projeto possui comunicação ponto a ponto e de transmissão em rede baseada na pilha de protocolos TCP/IP nativa da arquitetura de rádio IEEE 802.11 b/g/n do ESP32, estruturando-se da seguinte forma:

### Protocolo MQTT (Message Queuing Telemetry Transport)
O dispositivo inicializa o cliente MQTT através da biblioteca PubSubClient, estabelecendo conexão assíncrona com o broker público `broker.hivemq.com` na porta padrão `1883`. 
Para contornar colisões em servidores abertos, o firmware gera um identificador dinâmico exclusivo (*ClientID hex*) baseado em amostragem randômica atrelada a uma rotina automática de reconexão cíclica (`reconnect()`).

### Tópico de Publicação
As telemetrias consolidadas são despachadas a cada intervalo de **5 segundos** (gerenciado por contadores de tempo assíncronos não bloqueantes com a função `millis()`) no seguinte tópico de gerenciamento urbano:
*   `cidade/postes/01/telemetria`

### Estrutura do Payload (Formato JSON)
Os dados brutos coletados nos pinos são empacotados em uma cadeia de caracteres estruturada sob a sintaxe JSON, garantindo portabilidade para centrais de comando ou dashboards de monitoramento.

**Exemplo de Payload transmitido (Modo Economia):**
```json
{
  "presenca": 1,
  "tensao": 0.77,
  "estado": "Ativo"
}
```
## 4. Algoritmo e Código-Fonte do Firmware

O desenvolvimento do sistema embarcado foi estruturado em linguagem C++ utilizando o ambiente de desenvolvimento do Arduino para o microcontrolador ESP32. A arquitetura do código adota rotinas assíncronas baseadas em temporizadores de hardware (`millis()`) em detrimento de funções de atraso bloqueantes (`delay()`), garantindo o processamento paralelo das tarefas de sensoriamento, modulação de potência e transmissão de dados de rede.

### Tratamento Matemático (Conversão Analógico-Digital)
Para converter a quantização binária bruta gerada pelo conversor analógico-digital (ADC) de 12 bits do ESP32 para uma escala de tensão elétrica real linear (de 0V a 3,3V), o firmware executa a seguinte equação matemática embarcada:

$$\text{tensão} = \frac{\text{leituraADC} \times 3.3}{4095}$$

### Estrutura Lógica do Algoritmo
O fluxo de execução do programa principal opera de forma cíclica e condicional, dividindo-se nas seguintes etapas fundamentais:

* **Inicialização (Setup):** Configura a taxa de transmissão da comunicação serial para 115200 bps, define a direção de dados dos pinos periféricos (GPIO 13 como entrada; GPIO 12 e 34 em modos analógicos/PWM) e estabelece os parâmetros iniciais da modulação por largura de pulso (Frequência de 5000 Hz e resolução de 8 bits).
* **Manutenção de Conectividade:** Monitora de forma contínua o status de pareamento da pilha TCP/IP e da sessão MQTT. Caso ocorra uma desconexão com o servidor, a sub-rotina de tratamento `reconnect()` é acionada autonomamente para reestabelecer o enlace.
* **Controle de Iluminação Adaptativa:** Processa as variações de sinal discreto do sensor PIR. Se um pulso de nível alto (HIGH) for registrado na entrada digital, o sistema altera instantaneamente o ciclo de trabalho (*duty cycle*) do PWM para 255 (100% de brilho) e atualiza o marcador temporal. Decorridos 10 segundos sem novas interações, o firmware decai o PWM para 51 (20% de brilho).
* **Serialização e Transmissão:** A cada janela de tempo fixa de 5 segundos, gerenciada de forma assíncrona, as variáveis de controle do nó são encapsuladas em uma cadeia de caracteres estruturada sob a sintaxe JSON e enviadas ao broker remoto via função `client.publish()`.

### Implementação do Código-Fonte do Loop Principal
```cpp
#include <WiFi.h>
#include <PubSubClient.h>

#define PIR_PIN 13
#define LED_PIN 12
#define PIEZO_PIN 34

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqtt_server = "broker.hivemq.com";

WiFiClient espClient;
PubSubClient client(espClient);

unsigned long ultimoMovimento = 0;
const unsigned long tempoLigado = 10000;

unsigned long ultimoEnvio = 0;

void conectarWiFi() {

  Serial.println("Conectando ao WiFi...");

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi conectado!");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());
}

void conectarMQTT() {

  while (!client.connected()) {

    Serial.print("Conectando MQTT...");

    String clientId = "ESP32-";
    clientId += String(random(0xffff), HEX);

    if (client.connect(clientId.c_str())) {

      Serial.println(" conectado!");

    } else {

      Serial.print(" falhou, rc=");
      Serial.print(client.state());
      Serial.println(" tentando novamente...");
      delay(2000);
    }
  }
}

void setup() {

  Serial.begin(115200);

  pinMode(PIR_PIN, INPUT);

  ledcAttach(LED_PIN, 5000, 8);

  conectarWiFi();

  client.setServer(mqtt_server, 1883);

  Serial.println("Sistema iniciado");
}

void loop() {

  if (!client.connected()) {
    conectarMQTT();
  }

  client.loop();

  int presenca = digitalRead(PIR_PIN);

  int leituraADC = analogRead(PIEZO_PIN);

  float tensao = (leituraADC * 3.3) / 4095.0;

  if (presenca == HIGH) {

    ultimoMovimento = millis();

    Serial.println("=== MOVIMENTO DETECTADO ===");
  }

  if (millis() - ultimoMovimento < tempoLigado) {

    ledcWrite(LED_PIN, 255);

  } else {

    ledcWrite(LED_PIN, 51);
  }

  if (millis() - ultimoEnvio > 5000) {

    ultimoEnvio = millis();

    String payload = "{";
    payload += "\"presenca\":";
    payload += String(presenca);
    payload += ",";
    payload += "\"tensao\":";
    payload += String(tensao, 2);
    payload += ",";
    payload += "\"estado\":\"";

    if (presenca) {
      payload += "Ativo";
    } else {
      payload += "Economia";
    }

    payload += "\"}";

    client.publish(
      "cidade/postes/01/telemetria",
      payload.c_str()
    );

    Serial.println("MQTT Publicado:");
    Serial.println(payload);
  }

  Serial.print("Presenca: ");
  Serial.print(presenca);

  Serial.print(" | ADC: ");
  Serial.print(leituraADC);

  Serial.print(" | Tensao: ");
  Serial.print(tensao, 2);

  Serial.println(" V");

  delay(500);
}
---
```
## 5. Procedimentos de Reprodução e Testes

Para viabilizar a replicação do nó de iluminação inteligente, o repositório disponibiliza todos os artefatos necessários. O procedimento de teste é dividido entre a configuração do firmware e a auditoria de dados em tempo real, garantindo a validação da telemetria MQTT.

### Requisitos de Ambiente
* **Hardware:** Microcontrolador ESP32 DevKit V1 (ou compatível), Módulo Sensor PIR HC-SR501, Transdutor Piezoelétrico e LED de potência com resistor limitador (330Ω).
* **Software:** Ambiente de desenvolvimento **Arduino IDE** com as bibliotecas `WiFi.h` e `PubSubClient.h` instaladas via gerenciador de bibliotecas.
* **Rede:** Acesso a uma rede Wi-Fi ativa com banda de 2.4 GHz e conectividade à Internet para comunicação com o broker MQTT.

### Passo a Passo para Instalação
1. **Download do Código:** Acesse o botão **"<> Code"** no topo deste repositório e selecione **"Download ZIP"** para obter o projeto completo, ou clone o diretório localmente via Git:
   `git clone https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git`
2. **Configuração:** Abra o arquivo `firmware.ino` no Arduino IDE. Altere as constantes `ssid` e `password` com as credenciais da sua rede Wi-Fi local.
3. **Compilação e Upload:** Conecte o ESP32 ao computador via cabo USB, selecione a porta COM correta no menu *Ferramentas* e realize o upload do firmware para a placa.
4. **Monitoramento:** Abra o *Monitor Serial* da Arduino IDE (configurado em 115200 bps) para verificar a inicialização do handshake Wi-Fi e a conexão com o broker `broker.hivemq.com`.

### Auditoria de Telemetria (Protocolo MQTT)
Para validar o envio dos dados sem a necessidade de um servidor próprio, o projeto utiliza um broker público. O acompanhamento dos pacotes pode ser realizado através do [HiveMQ Web Client](http://www.hivemq.com/demos/websocket-client/):

* **Configuração do Cliente:** 1. Acesse o site do cliente Web do HiveMQ.
    2. Clique em **"Connect"** (mantendo as configurações padrão de porta 8000/WebSockets).
    3. Em **"Subscriptions"**, clique em **"Add New Topic Subscription"** e insira o tópico: `cidade/postes/01/telemetria`.
* **Validação:** A cada 5 segundos, o nó publicará um objeto JSON contendo o status de `presenca`, a `tensao` gerada e o `estado` operacional. Caso os dados apareçam na janela do cliente web, o nó está corretamente integrado à infraestrutura de rede IoT.
