//C++_Arduino
// ============================================
// Secador/Desumidificador 3D - COM PERFIS
// Versão v4 - controlo por humidade + reset
// ============================================
//AQUECEDOR=LED_AZUL 
//VENTOINHA=LED_VERDE

#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ============================================
// PINOS
// ============================================

#define PINO_DHT A0
#define LED_AQUECEDOR 7
#define LED_VENTOINHA 8
#define BOTAO 2
#define BOTAO_RESET 3

#define TIPO_DHT DHT22

// ============================================
// PERFIS DE MATERIAL
// ============================================

enum Material { PLA, PETG, ABS, TPU, NYLON, PC, NUM_MATERIAIS };

struct PerfilSecagem {
  const char* nome;
  float tempIdeal;
  float tempMaxima;
  float humidadeAlvo;
  float humidadeMaxima;
  unsigned long duracao;   // limite máximo (rede de segurança)
};

PerfilSecagem perfis[NUM_MATERIAIS] = {
  {"PLA",     45.0,   55.0,  20.0,    45.0,   4UL  * 3600000UL},
  {"PETG",    65.0,   75.0,  20.0,    45.0,   6UL  * 3600000UL},
  {"ABS",     70.0,   80.0,  20.0,    45.0,   5UL  * 3600000UL},
  {"TPU",     50.0,   60.0,  20.0,    45.0,   6UL  * 3600000UL},
  {"NYLON",   75.0,   85.0,  10.0,    40.0,   10UL * 3600000UL},
  {"PC",      75.0,   85.0,  15.0,    40.0,   7UL  * 3600000UL}
};

int materialAtual = PLA;

// ============================================
// PARÂMETROS GERAIS
// ============================================

#define HISTERESE_TEMP 2.0
#define HISTERESE_HUM 5.0
#define INTERVALO_LEITURA 2000
#define TEMPO_LONGO_PRESSIONAR 2000
#define DEBOUNCE_MS 50

// Tempo que a humidade tem de estar abaixo do alvo para considerar "seco"
#define TEMPO_ESTABILIDADE_HUM 600000UL   // 10 minutos

// ============================================
// OBJETOS E VARIÁVEIS
// ============================================

DHT sensor(PINO_DHT, TIPO_DHT);
LiquidCrystal_I2C lcd(0x27, 16, 2);

float temperatura = 0.0;
float humidade = 0.0;
unsigned long ultimaLeitura = 0;

bool aquecedorLigado = false;
bool ventoinhaLigada = false;
bool modoSeguranca = false;
bool secagemCompleta = false;
bool secagemAtiva = false;

unsigned long inicioSecagem = 0;
unsigned long tempoDecorrido = 0;

// Controlo de estabilidade da humidade
bool humidadeAbaixoAlvo = false;
unsigned long inicioHumidadeOk = 0;

// ============================================
// SETUP
// ============================================

void setup() {
  Serial.begin(9600);
  Serial.println("Secador 3D v4 - Controlo por humidade");

  pinMode(LED_AQUECEDOR, OUTPUT); //AQUECEDOR=LED_AZUL 
  pinMode(LED_VENTOINHA, OUTPUT); //VENTOINHA=LED_VERDE
  pinMode(BOTAO, INPUT_PULLUP); //BOTÃO VERDE
  pinMode(BOTAO_RESET, INPUT_PULLUP); //BOTÃO VERMELHO

  digitalWrite(LED_AQUECEDOR, LOW);
  digitalWrite(LED_VENTOINHA, LOW);

  sensor.begin();

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Secador 3D v4");
  lcd.setCursor(0, 1);
  lcd.print("Iniciando...");

  delay(2000);
  lcd.clear();
  mostrarMaterialSelecionado();
}

// ============================================
// LOOP
// ============================================

void loop() {
  lerBotao();
  lerBotaoReset();

  if (millis() - ultimaLeitura >= INTERVALO_LEITURA) {
    ultimaLeitura = millis();

    lerSensor();
    verificarSeguranca();

    if (secagemAtiva) {
      tempoDecorrido = millis() - inicioSecagem;
      controlarSistema();
    }

    atualizarDisplay();
    enviarSerial();
  }
}

// ============================================
// LEITURA DO BOTÃO PRINCIPAL
// ============================================

void lerBotao() {
  static bool estadoEstavel = HIGH;
  static bool ultimoEstadoLido = HIGH;
  static unsigned long tempoUltimaMudanca = 0;
  static unsigned long tempoInicioPressao = 0;
  static bool longaJaDisparou = false;

  bool leitura = digitalRead(BOTAO);

  if (leitura != ultimoEstadoLido) {
    tempoUltimaMudanca = millis();
    ultimoEstadoLido = leitura;
  }

  if ((millis() - tempoUltimaMudanca) > DEBOUNCE_MS) {

    if (leitura != estadoEstavel) {
      estadoEstavel = leitura;

      if (estadoEstavel == LOW) {
        tempoInicioPressao = millis();
        longaJaDisparou = false;
      }
      else {
        if (!longaJaDisparou) {
          proximoMaterial();
        }
      }
    }

    if (estadoEstavel == LOW && !longaJaDisparou) {
      if (millis() - tempoInicioPressao >= TEMPO_LONGO_PRESSIONAR) {
        iniciarSecagem();
        longaJaDisparou = true;
      }
    }
  }
}

// ============================================
// LEITURA DO BOTÃO DE RESET
// ============================================

void lerBotaoReset() {
  static bool estadoEstavel = HIGH;
  static bool ultimoEstadoLido = HIGH;
  static unsigned long tempoUltimaMudanca = 0;

  bool leitura = digitalRead(BOTAO_RESET);

  if (leitura != ultimoEstadoLido) {
    tempoUltimaMudanca = millis();
    ultimoEstadoLido = leitura;
  }

  if ((millis() - tempoUltimaMudanca) > DEBOUNCE_MS) {
    if (leitura != estadoEstavel) {
      estadoEstavel = leitura;

      if (estadoEstavel == LOW) {
        reiniciarPrograma();
      }
    }
  }
}

// ============================================
// REINICIAR PROGRAMA
// ============================================

void reiniciarPrograma() {
  Serial.println(">>> RESET: a voltar a escolha de material");

  desligarAquecedor();
  desligarVentoinha();

  secagemAtiva = false;
  secagemCompleta = false;
  modoSeguranca = false;
  inicioSecagem = 0;
  tempoDecorrido = 0;
  humidadeAbaixoAlvo = false;
  inicioHumidadeOk = 0;
  materialAtual = PLA;

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("  REINICIADO");
  lcd.setCursor(0, 1);
  lcd.print(" Escolhe mat.");
  delay(1500);

  mostrarMaterialSelecionado();
}

// ============================================
// GESTÃO DE MATERIAIS
// ============================================

void proximoMaterial() {
  if (secagemAtiva) return;

  materialAtual = (materialAtual + 1) % NUM_MATERIAIS;
  Serial.print("Material: ");
  Serial.println(perfis[materialAtual].nome);
  mostrarMaterialSelecionado();
}

void mostrarMaterialSelecionado() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Material: ");
  lcd.print(perfis[materialAtual].nome);
  lcd.setCursor(0, 1);
  lcd.print(perfis[materialAtual].tempIdeal, 0);
  lcd.print("C ");
  lcd.print(perfis[materialAtual].duracao / 3600000UL);
  lcd.print("h [SEGURE]");
}

void iniciarSecagem() {
  secagemAtiva = true;
  secagemCompleta = false;
  inicioSecagem = millis();
  tempoDecorrido = 0;
  humidadeAbaixoAlvo = false;
  inicioHumidadeOk = 0;

  Serial.print(">>> SECAGEM INICIADA: ");
  Serial.println(perfis[materialAtual].nome);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("INICIANDO ");
  lcd.print(perfis[materialAtual].nome);
  delay(1500);
  lcd.clear();
}

// ============================================
// LEITURA DO SENSOR
// ============================================

void lerSensor() {
  float h = sensor.readHumidity();
  float t = sensor.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println("ERRO: Sensor DHT!");
    modoSeguranca = true;
    desligarAquecedor();
    desligarVentoinha();
    return;
  }

  humidade = h;
  temperatura = t;
}

// ============================================
// SEGURANÇA
// ============================================

void verificarSeguranca() {
  PerfilSecagem p = perfis[materialAtual];

  if (temperatura >= p.tempMaxima) {
    modoSeguranca = true;
    desligarAquecedor();
    ligarVentoinha();   // ventoinha ligada para arrefecer
    Serial.println("!!! MODO SEGURANCA !!!");
  }
  else if (modoSeguranca && temperatura < (p.tempMaxima - 10)) {
    modoSeguranca = false;
    Serial.println("Seguranca: OK");
  }
}

// ============================================
// CONTROLO AUTOMÁTICO
// ============================================

void controlarSistema() {
  if (modoSeguranca) return;

  PerfilSecagem p = perfis[materialAtual];

  // --- 1. Estabilidade da humidade ---
  if (humidade <= p.humidadeAlvo) {
    if (!humidadeAbaixoAlvo) {
      humidadeAbaixoAlvo = true;
      inicioHumidadeOk = millis();
      Serial.println(">>> Humidade no alvo, a confirmar estabilidade...");
    }
  } else {
    if (humidadeAbaixoAlvo) {
      Serial.println(">>> Humidade subiu, secagem continua");
    }
    humidadeAbaixoAlvo = false;
    inicioHumidadeOk = 0;
  }

  bool humidadeEstavel = humidadeAbaixoAlvo &&
                         (millis() - inicioHumidadeOk >= TEMPO_ESTABILIDADE_HUM);

  bool tempoTerminado = tempoDecorrido >= p.duracao;

  // --- 2. Fim da secagem ---
  if (humidadeEstavel || tempoTerminado) {
    secagemCompleta = true;
    secagemAtiva = false;
    desligarAquecedor();
    desligarVentoinha();

    if (humidadeEstavel) {
      Serial.println(">>> SECAGEM COMPLETA (humidade no alvo) <<<");
    } else {
      Serial.println(">>> SECAGEM COMPLETA (tempo maximo) <<<");
    }
    return;
  }

  // --- 3. Aquecedor com histerese ---
  if (temperatura < (p.tempIdeal - HISTERESE_TEMP)) {
    ligarAquecedor();
  }
  else if (temperatura >= p.tempIdeal) {
    desligarAquecedor();
  }

  // --- 4. Ventoinha baseada em temperatura E humidade ---
  // Liga se o aquecedor está a trabalhar ou se a humidade ainda está acima do alvo.
  // Desliga quando a temperatura está estável e a humidade já está no alvo.
  if (aquecedorLigado || humidade > p.humidadeAlvo) {
    ligarVentoinha();
  } else {
    desligarVentoinha();
  }
}

// ============================================
// CONTROLO DOS ATUADORES
// ============================================

void ligarAquecedor() {
  if (!aquecedorLigado) {
    digitalWrite(LED_AQUECEDOR, HIGH);
    aquecedorLigado = true;
    Serial.println(">>> AQUECEDOR: ON");
  }
}

void desligarAquecedor() {
  if (aquecedorLigado) {
    digitalWrite(LED_AQUECEDOR, LOW);
    aquecedorLigado = false;
    Serial.println(">>> AQUECEDOR: OFF");
  }
}

void ligarVentoinha() {
  if (!ventoinhaLigada) {
    digitalWrite(LED_VENTOINHA, HIGH);
    ventoinhaLigada = true;
    Serial.println(">>> VENTOINHA: ON");
  }
}

void desligarVentoinha() {
  if (ventoinhaLigada) {
    digitalWrite(LED_VENTOINHA, LOW);
    ventoinhaLigada = false;
    Serial.println(">>> VENTOINHA: OFF");
  }
}

// ============================================
// DISPLAY
// ============================================

void atualizarDisplay() {
  PerfilSecagem p = perfis[materialAtual];

  if (!secagemAtiva && !secagemCompleta) {
    return;
  }

  lcd.setCursor(0, 0);
  lcd.print(p.nome);
  lcd.print(" ");
  lcd.print(temperatura, 1);
  lcd.print("C    ");

  lcd.setCursor(0, 1);

  if (secagemCompleta) {
    lcd.print("H:");
    lcd.print(humidade, 1);
    lcd.print("% PRONTO!  ");
  }
  else if (modoSeguranca) {
    lcd.print("!! QUENTE !!    ");
  }
  else {
    lcd.print("H:");
    if (humidade < 10) lcd.print(" ");
    lcd.print(humidade, 0);
    lcd.print("% ");

    unsigned long restante = (p.duracao > tempoDecorrido)
                             ? (p.duracao - tempoDecorrido)
                             : 0;
    unsigned long horas = restante / 3600000UL;
    unsigned long minutos = (restante % 3600000UL) / 60000UL;

    if (horas < 10) lcd.print("0");
    lcd.print(horas);
    lcd.print(":");
    if (minutos < 10) lcd.print("0");
    lcd.print(minutos);
    lcd.print("   ");
  }
}

// ============================================
// SERIAL
// ============================================

void enviarSerial() {
  Serial.print("[");
  Serial.print(perfis[materialAtual].nome);
  Serial.print("] T:");
  Serial.print(temperatura);
  Serial.print("C H:");
  Serial.print(humidade);
  Serial.print("% AQ:");
  Serial.print(aquecedorLigado ? "ON" : "OFF");
  Serial.print(" VT:");
  Serial.print(ventoinhaLigada ? "ON" : "OFF");

  if (secagemAtiva) {
    unsigned long restante = (perfis[materialAtual].duracao > tempoDecorrido)
                             ? (perfis[materialAtual].duracao - tempoDecorrido)
                             : 0;
    Serial.print(" Resta:");
    Serial.print(restante / 60000UL);
    Serial.print("min");

    if (humidadeAbaixoAlvo) {
      unsigned long est = (millis() - inicioHumidadeOk) / 1000UL;
      Serial.print(" Estab:");
      Serial.print(est);
      Serial.print("s");
    }
  }
  Serial.println();
}
