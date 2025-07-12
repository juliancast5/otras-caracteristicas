#include <WiFi.h>
#include <ArduinoOTA.h>
#include <LiquidCrystal_I2C.h>
#include <Wire.h>
#include <time.h>
#include <WebServer.h>
#include <ArduinoJson.h>
#include <SPIFFS.h>

// --- Configuración WiFi/IP ---
const char* ssid = "ADMIN";
const char* password = "";
IPAddress local_IP(192, 168, 3, 158);
IPAddress gateway(192, 168, 3, 1);
IPAddress subnet(255, 255, 255, 0);
IPAddress primaryDNS(8, 8, 8, 8);
IPAddress secondaryDNS(8, 8, 4, 4);

// --- Zona horaria Bogotá ---
const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = -5 * 3600;
const int daylightOffset_sec = 0;

// --- LCD I2C ---
LiquidCrystal_I2C lcd(0x27, 16, 2);

// --- Servidor Web ---
WebServer server(80);

// --- Autenticación Web ---
const char* auth_user = "frigonel";
const char* auth_pass = "admin2025";

// --- NTC ---
#define PIN_NTC 34
#define R_FIXED 61000.0
#define VIN 3.3

// --- Relés (LÓGICA INVERTIDA: LOW = ON, HIGH = OFF) ---
#define COMPRESOR_PIN 16
#define VENTILADOR_PIN 17
#define RESISTENCIA_PIN 18

// --- Botones RF ---
const int botones[4] = {12, 13, 14, 27}; // B1, B2, B3, B4
bool estadoAnterior[4] = {LOW, LOW, LOW, LOW};

// --- Variables de control requeridas ---
float umbral = -7.0;
int tiempoEspera = 30;
float tempMin = -3.0;
int DsCa = 300;           // Descongelar Cada (antes 'descongelar')
int DsOn = 10;            // Tiempo de descongelado activo (antes 'descongelarOn')
int Tmoff = 30;
int Dsxt = 30;            // Descongelar Extra por Tiempo (antes 'Tdescongelar')

// --- Programación horaria semanal ---
struct HorarioProgramado {
  bool activo;
  uint8_t diaSemana;     // 0=Domingo, 1=Lunes, ..., 6=Sábado
  uint8_t horaInicio;    // 0-23
  uint8_t minutoInicio;  // 0-59
  uint8_t horaFin;       // 0-23
  uint8_t minutoFin;     // 0-59
  // Ya no necesitamos campo 'accion' - siempre será OFF durante el horario
};

const int MAX_HORARIOS = 14; // Máximo 2 horarios por día de la semana
HorarioProgramado horariosSemanales[MAX_HORARIOS];
bool programacionActiva = false;
unsigned long ultimaVerificacionHorario = 0;
bool enHorarioProgramado = false;  // Para saber si estamos en un horario OFF programado

// --- Sistema de almacenamiento histórico mejorado ---
struct DatoHistorico {
  float temperatura;
  uint32_t timestamp; // Unix timestamp
  uint8_t estado;     // 0=Normal, 1=Descongelando, 2=OFF, etc.
};

// Buffer en RAM para datos recientes (últimas 4 horas = 48 puntos cada 5 min)
const int MAX_DATOS = 48;
DatoHistorico bufferDatos[MAX_DATOS];
int indiceDatos = 0;
unsigned long ultimoRegistro = 0;
const unsigned long INTERVALO_REGISTRO = 300000; // 5 minutos en ms

// Control de archivos históricos
unsigned long ultimoArchivo = 0;
const unsigned long INTERVALO_ARCHIVO = 7200000; // 2 horas en ms
const unsigned long RETENCION_MAXIMA = 604800; // 1 semana en segundos
bool spiffsIniciado = false;

// --- Estados de la interfaz ---
enum AppState {
  ESTADO_PRINCIPAL,
  MENU_PRINCIPAL,
  MENU_PARAM_GEN,
  MENU_MODO,
  MODO_OFF_CONFIRM,
  MODO_OFF_EDIT,
  PERM_OFF_SINO,
  MODO_ON_CONFIRM,
  EDIT_GENERIC,
  EDIT_DSON,
  EDIT_GUARDADO,
  EDIT_CANCELADO,
  DESCON_XTIEMPO_CONFIRM,
  DESCON_XTIEMPO_EDIT
};
AppState estadoApp = ESTADO_PRINCIPAL;

// --- Máquina de estados principal ---
enum EstadoSistema {
  NORMAL,
  DESCONGELANDO,
  VENTILADOR_ESPERA,
  OFF_TEMPORIZADO,
  PERM_OFF,
  DESCONGELAR_XTIEMPO
};
EstadoSistema estadoSistema = NORMAL;

// --- Variables de menú ---
int opcionMenuPrincipal = 0;
const int numOpcionesMenuPrincipal = 3;

int opcionParmGen = 0;
const int numOpcionesParmGen = 4;
const char* nombresParmGen[] = {
  "Umbral",
  "Tiempo en espera",
  "Temp. Minima",
  "DsCa"
};

int opcionModo = 0;
const int numOpcionesModo = 3;

int opcionModoOffSiNo = 0;
int opcionPermOffSiNo = 0;
int opcionModoOnSiNo = 0;
int opcionDesconXTiempoSiNo = 0;

int editandoIndice = -1;

float tempFiltrada = -1000;
const float ALFA = 0.05;

static unsigned long tiempoUltimaAccion = 0;
const unsigned long TIMEOUT_MENU = 5000;

const float tabla_resistencias[] = {16.8, 20.1, 23.0, 27.7, 33.0, 36.0};
const float tabla_temperaturas[] = {16.7, 13.1, 8.1, 5.1, 2.6, 0.0};
const int tabla_size = 6;

unsigned long tiempoInicioDsCa = 0;             // Antes tiempoInicioDescongelar
unsigned long tiempoInicioDsOn = 0;             // Antes tiempoInicioDesOn
unsigned long tiempoInicioVentiladorEspera = 0;
unsigned long tiempoInicioModoOff = 0;
unsigned long tiempoInicioDsxt = 0;             // Antes tiempoInicioDescX
unsigned long tiempoInicioTe = 0;               // Nuevo: para controlar el temporizador Te
int tiempoDsxt = 0;
bool teActivo = false;                          // Nuevo: indica si el temporizador Te está activo
bool compresorApagadoPorTe = false;             // Nuevo: indica si el compresor fue apagado por Te

bool alternaFecha = false;

// --- Buffer para evitar parpadeo en línea 2 ---
String bufferLinea2 = "";

// --- Función para actualizar línea 2 sin parpadeo ---
void actualizarLinea2(String nuevoTexto) {
  if (nuevoTexto != bufferLinea2) {
    bufferLinea2 = nuevoTexto;
    lcd.setCursor(0, 1);
    lcd.print("                "); // Limpiar línea 2
    lcd.setCursor(0, 1);
    lcd.print(bufferLinea2);
  }
}

// --- Funciones utilitarias y de máquina ---
void apagarTodosLosRele() {
  digitalWrite(COMPRESOR_PIN, HIGH);
  digitalWrite(VENTILADOR_PIN, HIGH);
  digitalWrite(RESISTENCIA_PIN, HIGH);
}

void refrescarTimeout() {
  tiempoUltimaAccion = millis();
}

void checkTimeoutMenu() {
  if ((estadoApp != ESTADO_PRINCIPAL) && (estadoApp != EDIT_GUARDADO) && (estadoApp != EDIT_CANCELADO)) {
    if (millis() - tiempoUltimaAccion > TIMEOUT_MENU) {
      estadoApp = ESTADO_PRINCIPAL;
      lcd.clear();
      bufferLinea2 = ""; // Limpiar buffer al regresar al estado principal
    }
  }
}

float estimarTemperatura(float r_ntc_kohm) {
  if (r_ntc_kohm <= tabla_resistencias[0]) return tabla_temperaturas[0];
  if (r_ntc_kohm >= tabla_resistencias[tabla_size-1]) return tabla_temperaturas[tabla_size-1];
  for (int i = 1; i < tabla_size; i++) {
    if (r_ntc_kohm < tabla_resistencias[i]) {
      float r1 = tabla_resistencias[i-1];
      float r2 = tabla_resistencias[i];
      float t1 = tabla_temperaturas[i-1];
      float t2 = tabla_temperaturas[i];
      float t = t1 + (r_ntc_kohm - r1) * (t2 - t1) / (r2 - r1);
      return t;
    }
  }
  return tabla_temperaturas[tabla_size-1];
}

void serialPrintEstado(float tempFiltrada, const char* fechaStr, const char* horaStr) {
  Serial.print("Temp: ");
  Serial.print(tempFiltrada, 1);
  Serial.print("°C | Fecha: ");
  Serial.print(fechaStr);
  Serial.print(" | Hora: ");
  Serial.print(horaStr);
  Serial.print(" | U=");
  Serial.print(umbral, 0);
  Serial.print("°C | Tm=");
  Serial.print(tempMin, 0);
  Serial.print("°C | Te=");
  Serial.print(tiempoEspera);
  Serial.print("m | DsCa=");
  Serial.print(DsCa);
  Serial.print("m | DsOn=");
  Serial.print(DsOn);
  Serial.print("m | Toff=");
  Serial.print(Tmoff);
  Serial.print("m | Dsxt=");
  Serial.print(Dsxt);
  Serial.print("m");
  // Estado de relés
  Serial.print(" | Relés-> C:");
  Serial.print(digitalRead(COMPRESOR_PIN) == LOW ? "ON" : "OFF");
  Serial.print(" V:");
  Serial.print(digitalRead(VENTILADOR_PIN) == LOW ? "ON" : "OFF");
  Serial.print(" R:");
  Serial.print(digitalRead(RESISTENCIA_PIN) == LOW ? "ON" : "OFF");
  Serial.println();
}

void procesarSerial() {
  if (Serial.available()) {
    String input = Serial.readStringUntil('\n');
    input.trim();
    input.toLowerCase();

    if (input.startsWith("toff")) {
      if (estadoSistema != PERM_OFF) {
        tiempoInicioModoOff = millis();
        estadoSistema = OFF_TEMPORIZADO;
        Serial.println("Modo OFF iniciado por serial");
      } else {
        Serial.println("No se puede iniciar modo OFF en Perm Off");
      }
      return;
    }
    if (input.startsWith("desc ")) {
      if (estadoSistema != PERM_OFF) {
        int minutos = input.substring(5).toInt();
        if (minutos > 0) {
          tiempoDsxt = minutos;
          tiempoInicioDsxt = millis();
          estadoSistema = DESCONGELAR_XTIEMPO;
          Serial.print("Dsxt iniciado: ");
          Serial.print(minutos);
          Serial.println(" min");
        }
      } else {
        Serial.println("No se puede iniciar Dsxt en Perm Off");
      }
      return;
    }
    if (input.startsWith("permoff")) {
      estadoSistema = PERM_OFF;
      Serial.println("Perm Off activado por serial");
      return;
    }
    if (input.startsWith("modoon")) {
      estadoSistema = NORMAL;
      tiempoInicioDsCa = millis();
      Serial.println("Modo ON activado por serial");
      return;
    }

    int idx = input.indexOf(' ');
    String comando = idx > 0 ? input.substring(0, idx) : input;
    String valorStr = idx > 0 ? input.substring(idx + 1) : "";

    if (comando == "u" && valorStr.length() > 0) {
      float nuevoUmbral = valorStr.toFloat();
      if (nuevoUmbral < 35 && nuevoUmbral > -15) {
        umbral = nuevoUmbral;
        float minPermitida = umbral + 4.0;
        if (tempMin < minPermitida) {
          tempMin = minPermitida;
        }
        Serial.printf("Umbral actualizado a %.2f°C\n", umbral);
      }
    } else if (comando == "te" && valorStr.length() > 0) {
      int nuevoTe = valorStr.toInt();
      if (nuevoTe >= 2 && nuevoTe <= 300) {
        tiempoEspera = nuevoTe;
        Serial.printf("Tiempo en espera actualizado a %d min\n", tiempoEspera);
      }
    } else if (comando == "tm" && valorStr.length() > 0) {
      float nuevoTm = valorStr.toFloat();
      float minPermitida = umbral + 4.0;
      if (nuevoTm >= minPermitida && nuevoTm <= 35) {
        tempMin = nuevoTm;
        Serial.printf("Temp. minima actualizada a %.2f°C\n", tempMin);
      }
    } else if (comando == "dsca" && valorStr.length() > 0) {
      int nuevoDsCa = valorStr.toInt();
      if (nuevoDsCa >= 2 && nuevoDsCa <= 999) {
        DsCa = nuevoDsCa;
        Serial.printf("DsCa actualizado a %d min\n", DsCa);
      }
    } else if (comando == "dson" && valorStr.length() > 0) {
      int nuevoDsOn = valorStr.toInt();
      if (nuevoDsOn > 0 && nuevoDsOn < 1000) {
        DsOn = nuevoDsOn;
        Serial.printf("DsOn actualizado a %d min\n", DsOn);
      }
    } else if (comando == "toff" && valorStr.length() > 0) {
      int nuevoToff = valorStr.toInt();
      if (nuevoToff >= 1 && nuevoToff <= 999) {
        Tmoff = nuevoToff;
        Serial.printf("Tiempo Off actualizado a %d min\n", Tmoff);
      }
    } else if (comando == "dsxt" && valorStr.length() > 0) {
      int nuevoDsxt = valorStr.toInt();
      if (nuevoDsxt >= 1 && nuevoDsxt <= 999) {
        Dsxt = nuevoDsxt;
        Serial.printf("Dsxt actualizado a %d min\n", Dsxt);
      }
    }
  }
}

// --- Limpieza de línea LCD ---
void limpiarLinea2() {
  lcd.setCursor(0, 1);
  lcd.print("                ");
}
// --- Menús y pantallas auxiliares (renombrados) ---

void mostrarMenuPrincipal(int opcion) {
  lcd.clear();
  for (int i = 0; i < 2; i++) {
    lcd.setCursor(0, i);
    int idx = opcion + i;
    if (idx >= numOpcionesMenuPrincipal) idx -= numOpcionesMenuPrincipal;
    if (idx == opcion) lcd.print(">");
    else lcd.print(" ");
    if (idx == 0) lcd.print("Parm. Gene.");
    else if (idx == 1) lcd.print("Modo");
    else if (idx == 2) lcd.print("Dsxt");
  }
}

void mostrarMenuParmGen(int opcion) {
  lcd.clear();
  int base = opcion;
  if (opcion == numOpcionesParmGen - 1) base = opcion - 1;
  if (base < 0) base = 0;
  for (int i = 0; i < 2; i++) {
    int idx = base + i;
    if (idx < numOpcionesParmGen) {
      lcd.setCursor(0, i);
      lcd.print((idx == opcion) ? ">" : " ");
      lcd.print(nombresParmGen[idx]);
    }
  }
}

void mostrarMenuModo(int opcion) {
  lcd.clear();
  int base = opcion;
  if (opcion == numOpcionesModo - 1) base = opcion - 1;
  if (base < 0) base = 0;
  for (int i = 0; i < 2; i++) {
    lcd.setCursor(0, i);
    int idx = base + i;
    if (idx < numOpcionesModo) {
      lcd.print((idx == opcion) ? ">" : " ");
      switch (idx) {
        case 0: lcd.print("Modo off   "); break;
        case 1: lcd.print("Perm. off  "); break;
        case 2: lcd.print("Modo on    "); break;
      }
    }
  }
}

void mostrarModoOffConfirm(int opcion) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Activar modo Off?");
  lcd.setCursor(0, 1);
  lcd.print((opcion == 0) ? ">Si   No" : " Si  >No");
}

void mostrarModoOnConfirm(int opcion) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Activar modo ON?");
  lcd.setCursor(0, 1);
  lcd.print((opcion == 0) ? ">Si   No" : " Si  >No");
}

void mostrarDesconXTiempoConfirm(int opcion) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Act. Dsxt?");
  lcd.setCursor(0, 1);
  lcd.print((opcion == 0) ? ">Si   No" : " Si  >No");
}

void mostrarEdicionTmoff(int valor) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Cuanto Tiempo");
  limpiarLinea2();
  lcd.setCursor(0, 1);
  char buffer[17];
  snprintf(buffer, 17, "%3dm [Edit]", valor);
  lcd.print(buffer);
}

void mostrarEdicionDsxt(int valor) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Cuanto Tiempo");
  limpiarLinea2();
  lcd.setCursor(0, 1);
  char buffer[17];
  snprintf(buffer, 17, "%3dm [Edit]", valor);
  lcd.print(buffer);
}

void mostrarMenuPermOffSiNo(int opcion) {
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("Desea activar");
  limpiarLinea2();
  lcd.setCursor(0,1);
  lcd.print("perm off ");
  lcd.print((opcion == 0) ? ">Si " : " Si ");
  lcd.print((opcion == 1) ? ">No" : " No");
}

void mostrarEdicion(int idx, float valor) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(nombresParmGen[idx]);
  limpiarLinea2();
  lcd.setCursor(0, 1);
  char buffer[17];
  switch (idx) {
    case 0:
    case 2:
      snprintf(buffer, 17, "%3d\xDF""C  | [Edit]", (int)valor);
      break;
    case 1:
    case 3:
      snprintf(buffer, 17, "%3dmin | [Edit]", (int)valor);
      break;
  }
  lcd.print(buffer);
}

void mostrarEdicionDson(int valor) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Tiempo de DsOn.");
  limpiarLinea2();
  lcd.setCursor(0, 1);
  char buffer[17];
  snprintf(buffer, 17, "%3dmin | [Edit]", valor);
  lcd.print(buffer);
}

void mostrarGuardadoOK() {
  lcd.clear();
  lcd.setCursor(2, 0);
  lcd.print("Guardado Ok");
  limpiarLinea2();
}
void mostrarCancelado() {
  lcd.clear();
  lcd.setCursor(3, 0);
  lcd.print("Cancelado");
  limpiarLinea2();
}

void mostrarPantallaTemp(float temp) {
  lcd.setCursor(0,0);
  lcd.print("Temp: ");
  lcd.print(temp, 1);
  lcd.print((char)223);
  lcd.print("C        ");
}

void mostrarPantallaDsCa(int seg, bool showLabel = true) {
  String texto = "";
  if(showLabel) texto += "DsCa=";
  else texto += "     ";
  int minutos = seg / 60;
  int segundos = seg % 60;
  if (minutos < 10) texto += "0";
  texto += String(minutos);
  texto += ":";
  if (segundos < 10) texto += "0";
  texto += String(segundos);
  actualizarLinea2(texto);
}

void mostrarPantallaDsOn(int seg) {
  String texto = "DsOn=";
  int minutos = seg / 60;
  int segundos = seg % 60;
  if (minutos < 10) texto += "0";
  texto += String(minutos);
  texto += ":";
  if (segundos < 10) texto += "0";
  texto += String(segundos);
  actualizarLinea2(texto);
}

void mostrarPantallaTe(int te) {
  String texto = "Te=" + String(te) + "min         ";
  actualizarLinea2(texto);
}

void mostrarPantallaTeContador(int seg) {
  String texto = "Te=";
  int minutos = seg / 60;
  int segundos = seg % 60;
  if (minutos < 10) texto += "0";
  texto += String(minutos);
  texto += ":";
  if (segundos < 10) texto += "0";
  texto += String(segundos);
  actualizarLinea2(texto);
}
void mostrarPantallaReles() {
  // Crear el texto completo
  String textoCompleto = "C:";
  textoCompleto += (digitalRead(COMPRESOR_PIN) == LOW ? "ON" : "OFF");
  textoCompleto += " V:";
  textoCompleto += (digitalRead(VENTILADOR_PIN) == LOW ? "ON" : "OFF");
  textoCompleto += " R:";
  textoCompleto += (digitalRead(RESISTENCIA_PIN) == LOW ? "ON" : "OFF");
  
  // Si el texto cabe en 16 caracteres, mostrarlo completo
  if (textoCompleto.length() <= 16) {
    String texto = textoCompleto;
    while (texto.length() < 16) texto += " "; // Rellenar con espacios
    actualizarLinea2(texto);
  } else {
    // Si no cabe, dividir en dos pantallas que rotan cada segundo
    static unsigned long tiempoUltRotReles = 0;
    static bool mostrarParte1 = true;
    if (millis() - tiempoUltRotReles > 1000) {
      mostrarParte1 = !mostrarParte1;
      tiempoUltRotReles = millis();
    }
    
    if (mostrarParte1) {
      // Primera parte: Compresor y Ventilador
      String texto1 = "C:";
      texto1 += (digitalRead(COMPRESOR_PIN) == LOW ? "ON" : "OFF");
      texto1 += " V:";
      texto1 += (digitalRead(VENTILADOR_PIN) == LOW ? "ON" : "OFF");
      while (texto1.length() < 16) texto1 += " ";
      actualizarLinea2(texto1);
    } else {
      // Segunda parte: Resistencia
      String texto2 = "R:";
      texto2 += (digitalRead(RESISTENCIA_PIN) == LOW ? "ON" : "OFF");
      while (texto2.length() < 16) texto2 += " ";
      actualizarLinea2(texto2);
    }
  }
}
void mostrarPantallaModoOff(float temp, int segundosRestantes) {
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temp, 1);
  lcd.print((char)223);
  lcd.print("C   ");
  String texto = "Modo OFF ";
  int minutos = segundosRestantes / 60;
  int segundos = segundosRestantes % 60;
  if (minutos < 10) texto += "0";
  texto += String(minutos);
  texto += ":";
  if (segundos < 10) texto += "0";
  texto += String(segundos);
  actualizarLinea2(texto);
}
void mostrarPantallaPermOff(float temp) {
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temp, 1);
  lcd.print((char)223);
  lcd.print("C   ");
  String texto = "Perm Off        ";
  actualizarLinea2(texto);
}
void mostrarPantallaDsxt(int seg) {
  String texto = "Dsxt=";
  int minutos = seg / 60;
  int segundos = seg % 60;
  if (minutos < 10) texto += "0";
  texto += String(minutos);
  texto += ":";
  if (segundos < 10) texto += "0";
  texto += String(segundos);
  actualizarLinea2(texto);
}

void mostrarPantallaFechaHora(const char* fechaStr, const char* horaStr) {
  String texto = "F:" + String(fechaStr) + " H:" + String(horaStr);
  actualizarLinea2(texto);
}

void mostrarPantallaUTm() {
  String texto = "U:" + String((int)umbral) + String((char)223) + "C  Tm:" + String((int)tempMin) + String((char)223) + "C";
  actualizarLinea2(texto);
}

// --- Funciones del Servidor Web ---
void handleRoot() {
  // Verificar autenticación
  if (!server.authenticate(auth_user, auth_pass)) {
    return server.requestAuthentication();
  }
  
  String html = "<!DOCTYPE html><html><head><meta charset='UTF-8'><title>Control Temperatura FRIGONEL</title><style>";
  html += "body{font-family:Arial;background:#667eea;padding:20px;margin:0;}";
  html += ".container{max-width:900px;margin:0 auto;background:white;padding:30px;border-radius:10px;box-shadow:0 4px 8px rgba(0,0,0,0.1);}";
  html += ".status{background:#74b9ff;color:white;padding:20px;margin:10px 0;border-radius:8px;}";
  html += ".temp{font-size:2em;text-align:center;}";
  html += ".section{margin:20px 0;border:1px solid #ddd;padding:15px;border-radius:8px;}";
  html += ".section h3{margin-top:0;color:#2d3436;}";
  html += ".input-group{margin:10px 0;display:flex;align-items:center;}";
  html += ".input-group label{width:180px;font-weight:bold;}";
  html += ".input-group input{width:80px;padding:8px;margin:0 10px;border:1px solid #ddd;border-radius:4px;}";
  html += ".btn{background:#74b9ff;color:white;border:none;padding:10px 15px;border-radius:5px;margin:5px;cursor:pointer;}";
  html += ".btn:hover{background:#0984e3;}";
  html += ".btn-danger{background:#ff6b6b;}.btn-danger:hover{background:#e55656;}";
  html += ".btn-success{background:#00b894;}.btn-success:hover{background:#00a085;}";
  html += ".btn-warning{background:#fdcb6e;}.btn-warning:hover{background:#e8b946;}";
  html += ".grid{display:grid;grid-template-columns:1fr 1fr;gap:20px;}";
  html += ".dsxt-group{display:flex;align-items:center;gap:10px;}";
  html += ".horario-item{margin:5px 0;padding:8px;border-left:4px solid #fdcb6e;background:#fff8e1;border-radius:4px;}";
  html += "select,input[type='time']{border:1px solid #ddd;border-radius:4px;font-size:14px;}";
  html += "#formularioHorario{background:#fff8e1;border-color:#fdcb6e;}";
  html += "</style></head><body>";
  html += "<div class='container'>";
  html += "<h1>Control Temperatura Vertical FRIGONEL</h1>";
  
  html += "<div class='status'>";
  html += "<h3>Temperatura: <span id='temp'>--</span> °C | Hora: <span id='hora'>--:--</span></h3>";
  html += "<p><strong>Compresor:</strong> <span id='comp'>--</span> | <strong>Ventilador:</strong> <span id='vent'>--</span> | <strong>Resistencia:</strong> <span id='res'>--</span></p>";
  html += "<p><strong>Estado:</strong> <span id='estado'>--</span> | <strong>Tiempo restante:</strong> <span id='tiempoRest'>--</span></p>";
  html += "<p id='statusProgramacion' style='margin:5px 0;font-size:14px;'><strong>Programación:</strong> <span id='progStatus'>--</span> | <strong>Próximo:</strong> <span id='proximoHorario'>--</span></p>";
  html += "</div>";
  
  html += "<div class='grid'>";
  
  html += "<div class='section'>";
  html += "<h3>Parametros Principales</h3>";
  html += "<div class='input-group'><label>Umbral (°C):</label><input type='number' id='umbral' step='0.1'><button class='btn' onclick='setParam(\"u\",\"umbral\")'>Aplicar</button></div>";
  html += "<div class='input-group'><label>Temp Minima (°C):</label><input type='number' id='tempMin' step='0.1'><button class='btn' onclick='setParam(\"tm\",\"tempMin\")'>Aplicar</button></div>";
  html += "<div class='input-group'><label>Tiempo Espera (min):</label><input type='number' id='tiempoEspera'><button class='btn' onclick='setParam(\"te\",\"tiempoEspera\")'>Aplicar</button></div>";
  html += "</div>";
  
  html += "<div class='section'>";
  html += "<h3>Descongelado Automatico</h3>";
  html += "<div class='input-group'><label>DsCa - Cada (min):</label><input type='number' id='dsca'><button class='btn' onclick='setParam(\"dsca\",\"dsca\")'>Aplicar</button></div>";
  html += "<div class='input-group'><label>DsOn - Duracion (min):</label><input type='number' id='dson'><button class='btn' onclick='setParam(\"dson\",\"dson\")'>Aplicar</button></div>";
  html += "</div>";
  
  html += "</div>";
  
  html += "<div class='section'>";
  html += "<h3>Controles del Sistema</h3>";
  html += "<button class='btn btn-success' onclick='sendCmd(\"modoon\")'>Modo ON</button>";
  html += "<button class='btn btn-danger' onclick='sendCmd(\"toff\")'>Modo OFF Temp</button>";
  html += "<button class='btn btn-danger' onclick='sendCmd(\"permoff\")'>Perm OFF</button>";
  html += "</div>";
  
  html += "<div class='section'>";
  html += "<h3>Descongelado Manual</h3>";
  html += "<div class='dsxt-group'>";
  html += "<label><strong>Tiempo (min):</strong></label>";
  html += "<input type='number' id='dsxt' value='30' min='1' max='300'>";
  html += "<button class='btn btn-warning' onclick='startDsxt()'>Iniciar Dsxt</button>";
  html += "</div>";
  html += "</div>";
  
  html += "<div class='section'>";
  html += "<h3>Programacion Horaria - Apagado Automatico</h3>";
  html += "<p style='margin-bottom:15px;color:#666;font-size:14px;'>Programa horarios para <strong>apagar automáticamente</strong> el equipo. Fuera de estos horarios, el equipo estará <strong>siempre encendido</strong>.</p>";
  html += "<div style='margin-bottom:15px;'>";
  html += "<label style='margin-right:15px;'><input type='checkbox' id='programacionActiva'> Activar Programacion de Apagado</label>";
  html += "<button class='btn' onclick='toggleProgramacion()'>Aplicar</button>";
  html += "</div>";
  html += "<div id='horariosProgramados'>";
  html += "<h4>Horarios de Apagado Configurados</h4>";
  html += "<div id='listaHorarios' style='max-height:200px;overflow-y:auto;border:1px solid #ddd;padding:10px;margin-bottom:10px;'>";
  html += "<p>Cargando horarios...</p>";
  html += "</div>";
  html += "<button class='btn btn-warning' onclick='mostrarFormularioHorario()'>Agregar Horario de Apagado</button>";
  html += "</div>";
  html += "<div id='formularioHorario' style='display:none;margin-top:20px;border:2px solid #fdcb6e;padding:15px;border-radius:8px;background:#fff8e1;'>";
  html += "<h4>Configurar Nuevo Horario de Apagado</h4>";
  html += "<p style='color:#666;font-size:13px;margin-bottom:15px;'><strong>Nota:</strong> Si la hora de fin es menor que la hora de inicio, se asume que el horario cruza medianoche automáticamente.</p>";
  html += "<div style='display:grid;grid-template-columns:1fr;gap:15px;margin-bottom:15px;'>";
  html += "<div>";
  html += "<label>Dia de Inicio:</label>";
  html += "<select id='diaSemana' style='width:100%;padding:8px;margin-top:5px;'>";
  html += "<option value='1'>Lunes</option><option value='2'>Martes</option><option value='3'>Miercoles</option>";
  html += "<option value='4'>Jueves</option><option value='5'>Viernes</option><option value='6'>Sabado</option><option value='0'>Domingo</option>";
  html += "</select>";
  html += "</div>";
  html += "</div>";
  html += "<div style='display:grid;grid-template-columns:1fr 1fr;gap:15px;margin-bottom:15px;'>";
  html += "<div>";
  html += "<label>Hora de Inicio (Apagar):</label>";
  html += "<input type='time' id='horaInicio' style='width:100%;padding:8px;margin-top:5px;'>";
  html += "</div>";
  html += "<div>";
  html += "<label>Hora de Fin (Encender):</label>";
  html += "<input type='time' id='horaFin' style='width:100%;padding:8px;margin-top:5px;'>";
  html += "</div>";
  html += "</div>";
  html += "<div id='infoHorario' style='padding:10px;background:#e3f2fd;border-radius:5px;margin-bottom:15px;font-size:13px;display:none;'>";
  html += "<strong>Vista previa:</strong> <span id='previsualizacion'></span>";
  html += "</div>";
  html += "<div style='text-align:center;'>";
  html += "<button class='btn btn-warning' onclick='guardarHorario()'>Guardar Horario de Apagado</button>";
  html += "<button class='btn' onclick='cancelarHorario()' style='margin-left:10px;'>Cancelar</button>";
  html += "</div>";
  html += "</div>";
  html += "</div>";
  
  html += "<div class='section'>";
  html += "<h3>Graficas de Temperatura</h3>";
  html += "<div style='margin-bottom:15px;'>";
  html += "<button class='btn' onclick='toggleChart()' id='chartBtn'>Mostrar Grafica Actual</button>";
  html += "<button class='btn btn-success' onclick='toggleHistorico()' id='historicoBtn'>Ver Historicos</button>";
  html += "</div>";
  html += "<div id='chartContainer' style='display:none;margin-top:20px;'>";
  html += "<canvas id='tempChart' width='800' height='400'></canvas>";
  html += "</div>";
  html += "<div id='historicoContainer' style='display:none;margin-top:20px;'>";
  html += "<h4>Datos Históricos - Buffer RAM</h4>";
  html += "<div id='listaHistoricos' style='max-height:300px;overflow-y:auto;border:1px solid #ddd;padding:10px;'>";
  html += "<p>Cargando datos históricos...</p>";
  html += "</div>";
  html += "<div id='chartHistorico' style='display:none;margin-top:20px;'>";
  html += "<canvas id='tempChartHistorico' width='800' height='400'></canvas>";
  html += "</div>";
  html += "</div>";
  html += "</div>";
  
  html += "<div style='text-align:center;margin-top:30px;padding:20px;background:#f8f9fa;border-radius:8px;color:#666;'>";
  html += "<p>Programado por <strong>Julian Castel</strong> - Derechos de autor © 2025 para <strong>FRIGONEL SAS</strong></p>";
  html += "<p style='font-size:12px;margin-top:10px;'>";
  html += "🌐 <strong>Acceso Local:</strong> 192.168.3.158 | ";
  html += "🔒 <strong>Usuario:</strong> frigonel | <strong>Contraseña:</strong> admin2025";
  html += "</p>";
  html += "</div>";
  
  html += "</div>";
  html += "<script src='https://cdn.jsdelivr.net/npm/chart.js'></script>";
  html += "<script>";
  html += "function updateStatus(){if(userEditing)return;fetch('/api/status').then(r=>r.json()).then(d=>{";
  html += "document.getElementById('temp').textContent=d.temperatura;";
  html += "document.getElementById('hora').textContent=d.hora;";
  html += "document.getElementById('comp').textContent=d.compresor;";
  html += "document.getElementById('vent').textContent=d.ventilador;";
  html += "document.getElementById('res').textContent=d.resistencia;";
  html += "document.getElementById('estado').textContent=d.estado;";
  html += "document.getElementById('tiempoRest').textContent=d.tiempoRestante || '--';";
  html += "document.getElementById('progStatus').textContent=d.programacionActiva?(d.estadoProgramacion || 'Activa'):'Inactiva';";
  html += "document.getElementById('proximoHorario').textContent=d.proximoHorario || '--';";
  html += "if(!document.activeElement || document.activeElement.tagName!=='INPUT'){";
  html += "document.getElementById('umbral').value=d.umbral;";
  html += "document.getElementById('tempMin').value=d.tempMin;";
  html += "document.getElementById('tiempoEspera').value=d.tiempoEspera;";
  html += "document.getElementById('dsca').value=d.dsca;";
  html += "document.getElementById('dson').value=d.dson;}";
  html += "});}";
  html += "function setParam(p,id){const v=document.getElementById(id).value;";
  html += "fetch('/api/set',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({command:p+' '+v})}).then(r=>r.json()).then(d=>{";
  html += "if(d.success){alert('✅ Actualizado');updateStatus();}else{alert('❌ Error: '+d.mensaje);}});}";
  html += "function sendCmd(c){fetch('/api/set',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({command:c})}).then(r=>r.json()).then(d=>{";
  html += "if(d.success){alert('✅ Ejecutado');updateStatus();}else{alert('❌ Error: '+d.mensaje);}});}";
  html += "function startDsxt(){const min=document.getElementById('dsxt').value;sendCmd('desc '+min);}";
  html += "function toggleProgramacion(){";
  html += "const activa=document.getElementById('programacionActiva').checked;";
  html += "fetch('/api/programacion',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({activa:activa})}).then(r=>r.json()).then(d=>{";
  html += "if(d.success){alert('✅ Programacion de apagado '+(activa?'activada':'desactivada'));loadHorarios();}else{alert('❌ Error: '+d.message);}});};";
  html += "function loadHorarios(){";
  html += "fetch('/api/horarios').then(r=>r.json()).then(d=>{";
  html += "document.getElementById('programacionActiva').checked=d.programacionActiva;";
  html += "const lista=document.getElementById('listaHorarios');";
  html += "if(d.horarios.length===0){lista.innerHTML='<p>No hay horarios de apagado configurados</p>';return;}";
  html += "let html='';";
  html += "d.horarios.forEach((h,i)=>{";
  html += "const dias=['Dom','Lun','Mar','Mie','Jue','Vie','Sab'];";
  html += "const diaInicio=dias[h.diaSemana];";
  html += "const horaInicioStr=String(h.horaInicio).padStart(2,'0')+':'+String(h.minutoInicio).padStart(2,'0');";
  html += "const horaFinStr=String(h.horaFin).padStart(2,'0')+':'+String(h.minutoFin).padStart(2,'0');";
  html += "let descripcion='';";
  html += "if((h.horaFin*60+h.minutoFin)<(h.horaInicio*60+h.minutoInicio)){";
  html += "const diaFin=dias[(h.diaSemana+1)%7];";
  html += "descripcion=`${diaInicio} ${horaInicioStr} - ${diaFin} ${horaFinStr}`;";
  html += "}else{";
  html += "descripcion=`${diaInicio} ${horaInicioStr} - ${horaFinStr}`;";
  html += "}";
  html += "html+=`<div style=\"margin:5px 0;padding:8px;border-left:4px solid #fdcb6e;background:#fff8e1;\">`;";
  html += "html+=`<strong>${descripcion}</strong> `;";
  html += "html+=`<span style=\"color:#e8b946;font-weight:bold;\">APAGAR</span> `;";
  html += "html+=`<button class=\"btn\" style=\"font-size:12px;padding:4px 8px;margin-left:10px;\" onclick=\"eliminarHorario(${i})\">Eliminar</button>`;";
  html += "html+=`</div>`;});";
  html += "lista.innerHTML=html;});}";
  html += "function mostrarFormularioHorario(){";
  html += "document.getElementById('formularioHorario').style.display='block';";
  html += "document.getElementById('horaInicio').addEventListener('change',actualizarPrevisualizacion);";
  html += "document.getElementById('horaFin').addEventListener('change',actualizarPrevisualizacion);";
  html += "document.getElementById('diaSemana').addEventListener('change',actualizarPrevisualizacion);";
  html += "}";
  html += "function actualizarPrevisualizacion(){";
  html += "const dia=parseInt(document.getElementById('diaSemana').value);";
  html += "const horaInicio=document.getElementById('horaInicio').value;";
  html += "const horaFin=document.getElementById('horaFin').value;";
  html += "if(horaInicio&&horaFin){";
  html += "const dias=['Domingo','Lunes','Martes','Miércoles','Jueves','Viernes','Sábado'];";
  html += "const [hI,mI]=horaInicio.split(':').map(Number);";
  html += "const [hF,mF]=horaFin.split(':').map(Number);";
  html += "const minutosInicio=hI*60+mI;";
  html += "const minutosFin=hF*60+mF;";
  html += "let descripcion='';";
  html += "if(minutosFin<minutosInicio){";
  html += "const diaFin=dias[(dia+1)%7];";
  html += "descripcion=`Se apagará el ${dias[dia]} a las ${horaInicio} y se encenderá el ${diaFin} a las ${horaFin}`;";
  html += "}else{";
  html += "descripcion=`Se apagará el ${dias[dia]} a las ${horaInicio} y se encenderá el mismo día a las ${horaFin}`;";
  html += "}";
  html += "document.getElementById('previsualizacion').textContent=descripcion;";
  html += "document.getElementById('infoHorario').style.display='block';";
  html += "}else{";
  html += "document.getElementById('infoHorario').style.display='none';";
  html += "}";
  html += "}";
  html += "function cancelarHorario(){";
  html += "document.getElementById('formularioHorario').style.display='none';";
  html += "document.getElementById('infoHorario').style.display='none';";
  html += "}";
  html += "function guardarHorario(){";
  html += "const dia=parseInt(document.getElementById('diaSemana').value);";
  html += "const horaInicio=document.getElementById('horaInicio').value;";
  html += "const horaFin=document.getElementById('horaFin').value;";
  html += "if(!horaInicio||!horaFin){alert('❌ Debe especificar hora de inicio y fin');return;}";
  html += "const [hI,mI]=horaInicio.split(':').map(Number);";
  html += "const [hF,mF]=horaFin.split(':').map(Number);";
  html += "const horario={diaSemana:dia,horaInicio:hI,minutoInicio:mI,horaFin:hF,minutoFin:mF,activo:true};";
  html += "fetch('/api/horarios',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(horario)}).then(r=>r.json()).then(d=>{";
  html += "if(d.success){alert('✅ Horario de apagado guardado');cancelarHorario();loadHorarios();document.getElementById('horaInicio').value='';document.getElementById('horaFin').value='';}else{alert('❌ Error: '+d.message);}});};";
  html += "function eliminarHorario(indice){";
  html += "if(confirm('¿Eliminar este horario de apagado?')){";
  html += "fetch('/api/horarios',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({eliminar:indice})}).then(r=>r.json()).then(d=>{";
  html += "if(d.success){alert('✅ Horario eliminado');loadHorarios();}else{alert('❌ Error: '+d.message);}});}}";
  html += "let userEditing=false;let editTimeout;";
  html += "function startEditTimeout(){clearTimeout(editTimeout);editTimeout=setTimeout(()=>{userEditing=false;document.activeElement&&document.activeElement.blur();},30000);}";
  html += "document.querySelectorAll('input').forEach(inp=>{";
  html += "inp.addEventListener('focus',()=>{userEditing=true;startEditTimeout();});";
  html += "inp.addEventListener('blur',()=>setTimeout(()=>userEditing=false,100));";
  html += "inp.addEventListener('input',startEditTimeout);});";
  html += "setInterval(updateStatus,2000);updateStatus();";
  html += "setTimeout(loadHorarios,1000);";
  html += "let chart=null;let chartVisible=false;let historicoVisible=false;let chartHistorico=null;";
  html += "function toggleChart(){";
  html += "const container=document.getElementById('chartContainer');";
  html += "const btn=document.getElementById('chartBtn');";
  html += "if(chartVisible){container.style.display='none';btn.textContent='Mostrar Grafica Actual';chartVisible=false;if(chart)chart.destroy();}";
  html += "else{container.style.display='block';btn.textContent='Ocultar Grafica';chartVisible=true;loadChart();}}";
  html += "function toggleHistorico(){";
  html += "const container=document.getElementById('historicoContainer');";
  html += "const btn=document.getElementById('historicoBtn');";
  html += "if(historicoVisible){container.style.display='none';btn.textContent='Ver Historicos';historicoVisible=false;}";
  html += "else{container.style.display='block';btn.textContent='Ocultar Historicos';historicoVisible=true;loadHistoricos();}}";
  html += "function loadChart(){if(chart)chart.destroy();";
  html += "fetch('/api/chart').then(r=>r.json()).then(d=>{";
  html += "if(d.datos.length===0){";
  html += "const ctx=document.getElementById('tempChart').getContext('2d');";
  html += "ctx.clearRect(0,0,800,400);";
  html += "ctx.font='20px Arial';ctx.fillStyle='#666';ctx.textAlign='center';";
  html += "ctx.fillText('No hay datos disponibles',400,200);";
  html += "ctx.font='14px Arial';ctx.fillText('Los datos se registran cada 5 minutos',400,230);";
  html += "return;}";
  html += "const ctx=document.getElementById('tempChart').getContext('2d');";
  html += "const temperaturas=[];const labels=[];const estados=[];";
  html += "d.datos.forEach((punto,i)=>{";
  html += "temperaturas.push(punto.temperatura);";
  html += "labels.push(punto.hora || '00:00');";
  html += "estados.push(punto.estadoTexto || 'NORMAL');});";
  html += "chart=new Chart(ctx,{type:'line',data:{labels:labels,datasets:[";
  html += "{label:'Temperatura °C',data:temperaturas,borderColor:'#74b9ff',backgroundColor:'rgba(116,185,255,0.1)',fill:true,pointRadius:4,pointHoverRadius:6},";
  html += "{label:'Umbral',data:Array(temperaturas.length).fill(d.umbral),borderColor:'#ff6b6b',borderDash:[5,5],fill:false,pointRadius:0},";
  html += "{label:'Temp Mínima',data:Array(temperaturas.length).fill(d.tempMin),borderColor:'#00b894',borderDash:[5,5],fill:false,pointRadius:0}";
  html += "]},options:{responsive:true,plugins:{title:{display:true,text:'Monitoreo de Temperatura - Últimas 4 Horas'},";
  html += "tooltip:{callbacks:{title:function(context){return 'Hora: '+context[0].label;},";
  html += "afterLabel:function(context){return 'Estado: '+estados[context.dataIndex];}}}},";
  html += "scales:{y:{title:{display:true,text:'Temperatura (°C)'}},x:{title:{display:true,text:'Hora'}}}}});});}";
  html += "function loadHistoricos(){";
  html += "const lista=document.getElementById('listaHistoricos');";
  html += "lista.innerHTML='<h5>Datos Históricos del Buffer RAM (Últimas 4 horas)</h5><p>Cargando datos...</p>';";
  html += "fetch('/api/chart').then(r=>r.json()).then(d=>{";
  html += "if(d.datos.length===0){lista.innerHTML='<p>No hay datos disponibles aún.<br>Los datos se registran cada 5 minutos automáticamente.</p>';return;}";
  html += "let html='<div style=\"max-height:300px;overflow-y:auto;\">';";
  html += "html+='<table style=\"width:100%;border-collapse:collapse;font-size:12px;\">';";
  html += "html+='<tr style=\"background:#f0f0f0;font-weight:bold;\"><td style=\"padding:5px;border:1px solid #ddd;\">Hora</td><td style=\"padding:5px;border:1px solid #ddd;\">Temp (°C)</td><td style=\"padding:5px;border:1px solid #ddd;\">Estado</td></tr>';";
  html += "d.datos.forEach(punto=>{";
  html += "const temp=punto.temperatura.toFixed(1);";
  html += "const hora=punto.hora||'--:--';";
  html += "const estado=punto.estadoTexto||'NORMAL';";
  html += "let colorEstado='#28a745';";
  html += "if(estado==='DESCONGELANDO')colorEstado='#fd7e14';";
  html += "else if(estado.includes('OFF')||estado==='PERM_OFF')colorEstado='#dc3545';";
  html += "html+=`<tr><td style=\"padding:4px;border:1px solid #ddd;\">${hora}</td>`;";
  html += "html+=`<td style=\"padding:4px;border:1px solid #ddd;text-align:center;\">${temp}</td>`;";
  html += "html+=`<td style=\"padding:4px;border:1px solid #ddd;color:${colorEstado};font-weight:bold;\">${estado}</td></tr>`;});";
  html += "html+='</table></div>';";
  html += "html+=`<p style=\"margin-top:10px;font-size:11px;color:#666;\">Total: ${d.datos.length} registros de las últimas 4 horas</p>`;";
  html += "lista.innerHTML=html;}).catch(e=>{lista.innerHTML='<p style=\"color:#e74c3c;\">Error cargando datos históricos</p>';});}";
  html += "function verHistorico(nombreArchivo){";
  html += "if(chartHistorico)chartHistorico.destroy();";
  html += "document.getElementById('chartHistorico').style.display='block';";
  html += "fetch('/api/chart?archivo='+encodeURIComponent(nombreArchivo)).then(r=>r.json()).then(d=>{";
  html += "const ctx=document.getElementById('tempChartHistorico').getContext('2d');";
  html += "const temperaturas=[];const labels=[];";
  html += "d.datos.forEach(punto=>{";
  html += "temperaturas.push(punto.temperatura);";
  html += "const fecha=new Date(punto.timestamp*1000);";
  html += "labels.push(fecha.toLocaleTimeString('es-CO',{hour:'2-digit',minute:'2-digit'}));});";
  html += "const titulo=`Historico: ${d.datos[0]?.fecha_hora.split(' ')[0] || 'Archivo'} - ${d.totalPuntos} puntos`;";
  html += "chartHistorico=new Chart(ctx,{type:'line',data:{labels:labels,datasets:[";
  html += "{label:'Temperatura °C',data:temperaturas,borderColor:'#e17055',backgroundColor:'rgba(225,112,85,0.1)',fill:true},";
  html += "{label:'Umbral',data:Array(temperaturas.length).fill(d.umbral),borderColor:'#ff6b6b',borderDash:[5,5],fill:false},";
  html += "{label:'Temp Minima',data:Array(temperaturas.length).fill(d.tempMin),borderColor:'#00b894',borderDash:[5,5],fill:false}";
  html += "]},options:{responsive:true,plugins:{title:{display:true,text:titulo}},";
  html += "scales:{y:{title:{display:true,text:'Temperatura (°C)'}},x:{title:{display:true,text:'Hora'}}}}});});}";
  html += "</script></body></html>";
  
  // Headers para acceso externo
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.sendHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
  server.sendHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
  
  server.send(200, "text/html", html);
}

void addCorsHeaders() {
  server.sendHeader("Access-Control-Allow-Origin", "*");
  server.sendHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
  server.sendHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
}

void handleApiStatus() {
  addCorsHeaders();
  DynamicJsonDocument doc(1024);
  
  doc["temperatura"] = String(tempFiltrada, 1);
  doc["compresor"] = digitalRead(COMPRESOR_PIN) == LOW ? "ON" : "OFF";
  doc["ventilador"] = digitalRead(VENTILADOR_PIN) == LOW ? "ON" : "OFF";
  doc["resistencia"] = digitalRead(RESISTENCIA_PIN) == LOW ? "ON" : "OFF";
  
  // Hora actual del ESP32
  struct tm timeinfo;
  if (getLocalTime(&timeinfo)) {
    char horaStr[6];
    strftime(horaStr, sizeof(horaStr), "%H:%M", &timeinfo);
    doc["hora"] = String(horaStr);
  } else {
    doc["hora"] = "--:--";
  }
  
  // Estado del sistema
  String estadoTexto = "";
  String tiempoRestante = "--:--";
  
  switch(estadoSistema) {
    case NORMAL: 
      estadoTexto = "NORMAL"; 
      // Mostrar tiempo restante hasta próximo DsCa
      {
        unsigned long trans = millis() - tiempoInicioDsCa;
        long totalSegundos = DsCa * 60;
        long transcurrido = trans / 1000;
        int segundosRestantes = totalSegundos - transcurrido;
        if (segundosRestantes < 0) segundosRestantes = 0;
        int minutos = segundosRestantes / 60;
        int segundos = segundosRestantes % 60;
        tiempoRestante = "DsCa: " + String(minutos) + ":" + (segundos < 10 ? "0" : "") + String(segundos);
      }
      break;
    case DESCONGELANDO: 
      estadoTexto = "DESCONGELANDO";
      // Mostrar tiempo restante de DsOn
      {
        unsigned long trans = millis() - tiempoInicioDsOn;
        int total = DsOn * 60;
        int transcurrido = trans / 1000;
        int segundosRestantes = total - transcurrido;
        if (segundosRestantes < 0) segundosRestantes = 0;
        int minutos = segundosRestantes / 60;
        int segundos = segundosRestantes % 60;
        tiempoRestante = "DsOn: " + String(minutos) + ":" + (segundos < 10 ? "0" : "") + String(segundos);
      }
      break;
    case VENTILADOR_ESPERA: 
      estadoTexto = "VENTILADOR_ESPERA"; 
      break;
    case OFF_TEMPORIZADO: 
      estadoTexto = "OFF_TEMPORIZADO";
      // Mostrar tiempo restante de Tmoff
      {
        unsigned long trans = millis() - tiempoInicioModoOff;
        int total = Tmoff * 60;
        int transcurrido = trans / 1000;
        int segundosRestantes = total - transcurrido;
        if (segundosRestantes < 0) segundosRestantes = 0;
        int minutos = segundosRestantes / 60;
        int segundos = segundosRestantes % 60;
        tiempoRestante = "Tmoff: " + String(minutos) + ":" + (segundos < 10 ? "0" : "") + String(segundos);
      }
      break;
    case PERM_OFF: 
      estadoTexto = "PERM_OFF"; 
      break;
    case DESCONGELAR_XTIEMPO: 
      estadoTexto = "DESCONGELAR_XTIEMPO";
      // Mostrar tiempo restante de Dsxt
      {
        unsigned long trans = millis() - tiempoInicioDsxt;
        int total = tiempoDsxt * 60;
        int transcurrido = trans / 1000;
        int segundosRestantes = total - transcurrido;
        if (segundosRestantes < 0) segundosRestantes = 0;
        int minutos = segundosRestantes / 60;
        int segundos = segundosRestantes % 60;
        tiempoRestante = "Dsxt: " + String(minutos) + ":" + (segundos < 10 ? "0" : "") + String(segundos);
      }
      break;
  }
  doc["estado"] = estadoTexto;
  doc["tiempoRestante"] = tiempoRestante;
  
  // Tiempo Te si está activo
  if (teActivo) {
    unsigned long trans = millis() - tiempoInicioTe;
    int total = tiempoEspera * 60;
    int transcurrido = trans / 1000;
    int segundosRestantes = total - transcurrido;
    if (segundosRestantes <= 0) segundosRestantes = 0;
    int minutos = segundosRestantes / 60;
    int segundos = segundosRestantes % 60;
    doc["tiempoTe"] = String(minutos) + ":" + (segundos < 10 ? "0" : "") + String(segundos);
  } else {
    doc["tiempoTe"] = "--:--";
  }
  
  // Parámetros del sistema
  doc["umbral"] = umbral;
  doc["tempMin"] = tempMin;
  doc["tiempoEspera"] = tiempoEspera;
  doc["dsca"] = DsCa;
  doc["dson"] = DsOn;
  
  // Estado de programación horaria
  doc["programacionActiva"] = programacionActiva;
  if (programacionActiva) {
    // Encontrar próximo horario de apagado programado
    struct tm timeinfo;
    if (getLocalTime(&timeinfo)) {
      int diaActual = timeinfo.tm_wday;
      int horaActual = timeinfo.tm_hour;
      int minutoActual = timeinfo.tm_min;
      
      String proximoHorario = "";
      String estadoProgramacion = enHorarioProgramado ? "EN APAGADO PROGRAMADO" : "ACTIVO";
      
      if (!enHorarioProgramado) {
        // Buscar próximo horario de apagado
        for (int i = 0; i < MAX_HORARIOS; i++) {
          if (horariosSemanales[i].activo) {
            int minutosInicio = horariosSemanales[i].horaInicio * 60 + horariosSemanales[i].minutoInicio;
            int minutosFin = horariosSemanales[i].horaFin * 60 + horariosSemanales[i].minutoFin;
            int minutosActual = horaActual * 60 + minutoActual;
            
            // Verificar si es hoy y aún no ha empezado
            if (horariosSemanales[i].diaSemana == diaActual && minutosInicio > minutosActual) {
              proximoHorario = "Apagar a las " + String(horariosSemanales[i].horaInicio) + ":" + 
                             (horariosSemanales[i].minutoInicio < 10 ? "0" : "") + 
                             String(horariosSemanales[i].minutoInicio);
              break;
            }
          }
        }
      } else {
        // Si estamos en apagado programado, mostrar cuándo se enciende
        for (int i = 0; i < MAX_HORARIOS; i++) {
          if (horariosSemanales[i].activo) {
            int minutosInicio = horariosSemanales[i].horaInicio * 60 + horariosSemanales[i].minutoInicio;
            int minutosFin = horariosSemanales[i].horaFin * 60 + horariosSemanales[i].minutoFin;
            int minutosActual = horaActual * 60 + minutoActual;
            
            bool enEsteRango = false;
            if (minutosFin > minutosInicio) {
              // Rango normal
              if (horariosSemanales[i].diaSemana == diaActual) {
                enEsteRango = (minutosActual >= minutosInicio && minutosActual < minutosFin);
              }
            } else {
              // Rango que cruza medianoche
              int diaAnterior = (diaActual == 0) ? 6 : diaActual - 1;
              if (horariosSemanales[i].diaSemana == diaActual && minutosActual >= minutosInicio) {
                enEsteRango = true;
              } else if (horariosSemanales[i].diaSemana == diaAnterior && minutosActual < minutosFin) {
                enEsteRango = true;
              }
            }
            
            if (enEsteRango) {
              if (minutosFin > minutosInicio) {
                // Mismo día
                proximoHorario = "Encender a las " + String(horariosSemanales[i].horaFin) + ":" + 
                               (horariosSemanales[i].minutoFin < 10 ? "0" : "") + 
                               String(horariosSemanales[i].minutoFin);
              } else {
                // Cruza medianoche
                if (horariosSemanales[i].diaSemana == diaActual) {
                  // Estamos en el día de inicio, se enciende mañana
                  proximoHorario = "Encender mañana a las " + String(horariosSemanales[i].horaFin) + ":" + 
                                 (horariosSemanales[i].minutoFin < 10 ? "0" : "") + 
                                 String(horariosSemanales[i].minutoFin);
                } else {
                  // Estamos en el día siguiente, se enciende hoy
                  proximoHorario = "Encender a las " + String(horariosSemanales[i].horaFin) + ":" + 
                                 (horariosSemanales[i].minutoFin < 10 ? "0" : "") + 
                                 String(horariosSemanales[i].minutoFin);
                }
              }
              break;
            }
          }
        }
      }
      
      doc["proximoHorario"] = proximoHorario;
      doc["estadoProgramacion"] = estadoProgramacion;
    }
  }
  
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

void handleApiSet() {
  if (server.hasArg("plain")) {
    DynamicJsonDocument doc(512);
    deserializeJson(doc, server.arg("plain"));
    
    String command = doc["command"];
    
    // Procesar comando igual que por serial
    command.trim();
    command.toLowerCase();
    
    bool success = true;
    String mensaje = "Comando ejecutado";
    
    if (command.startsWith("toff")) {
      if (estadoSistema != PERM_OFF) {
        tiempoInicioModoOff = millis();
        estadoSistema = OFF_TEMPORIZADO;
      } else {
        success = false;
        mensaje = "No se puede iniciar modo OFF en Perm Off";
      }
    } else if (command.startsWith("desc ")) {
      if (estadoSistema != PERM_OFF) {
        int minutos = command.substring(5).toInt();
        if (minutos > 0) {
          tiempoDsxt = minutos;
          tiempoInicioDsxt = millis();
          estadoSistema = DESCONGELAR_XTIEMPO;
        } else {
          success = false;
          mensaje = "Tiempo inválido";
        }
      } else {
        success = false;
        mensaje = "No se puede iniciar Dsxt en Perm Off";
      }
    } else if (command.startsWith("permoff")) {
      estadoSistema = PERM_OFF;
    } else if (command.startsWith("modoon")) {
      estadoSistema = NORMAL;
      tiempoInicioDsCa = millis();
      compresorApagadoPorTe = false; // Resetear flag al activar modo ON
    } else {
      // Procesar parámetros
      int idx = command.indexOf(' ');
      String cmd = idx > 0 ? command.substring(0, idx) : command;
      String valorStr = idx > 0 ? command.substring(idx + 1) : "";
      
      if (cmd == "u" && valorStr.length() > 0) {
        float nuevoUmbral = valorStr.toFloat();
        if (nuevoUmbral < 35 && nuevoUmbral > -15) {
          umbral = nuevoUmbral;
          float minPermitida = umbral + 4.0;
          if (tempMin < minPermitida) {
            tempMin = minPermitida;
          }
        } else {
          success = false;
          mensaje = "Valor fuera de rango";
        }
      } else if (cmd == "te" && valorStr.length() > 0) {
        int nuevoTe = valorStr.toInt();
        if (nuevoTe >= 2 && nuevoTe <= 300) {
          tiempoEspera = nuevoTe;
        } else {
          success = false;
          mensaje = "Valor fuera de rango";
        }
      } else if (cmd == "tm" && valorStr.length() > 0) {
        float nuevoTm = valorStr.toFloat();
        float minPermitida = umbral + 4.0;
        if (nuevoTm >= minPermitida && nuevoTm <= 35) {
          tempMin = nuevoTm;
        } else {
          success = false;
          mensaje = "Valor fuera de rango";
        }
      } else if (cmd == "dsca" && valorStr.length() > 0) {
        int nuevoDsCa = valorStr.toInt();
        if (nuevoDsCa >= 2 && nuevoDsCa <= 999) {
          DsCa = nuevoDsCa;
        } else {
          success = false;
          mensaje = "Valor fuera de rango";
        }
      } else if (cmd == "dson" && valorStr.length() > 0) {
        int nuevoDsOn = valorStr.toInt();
        if (nuevoDsOn > 0 && nuevoDsOn < 1000) {
          DsOn = nuevoDsOn;
        } else {
          success = false;
          mensaje = "Valor fuera de rango";
        }
      } else if (cmd == "toff" && valorStr.length() > 0) {
        int nuevoToff = valorStr.toInt();
        if (nuevoToff >= 1 && nuevoToff <= 999) {
          Tmoff = nuevoToff;
        } else {
          success = false;
          mensaje = "Valor fuera de rango";
        }
      } else if (cmd == "dsxt" && valorStr.length() > 0) {
        int nuevoDsxt = valorStr.toInt();
        if (nuevoDsxt >= 1 && nuevoDsxt <= 999) {
          Dsxt = nuevoDsxt;
        } else {
          success = false;
          mensaje = "Valor fuera de rango";
        }
      } else {
        success = false;
        mensaje = "Comando no reconocido";
      }
    }
    
    DynamicJsonDocument response(256);
    response["success"] = success;
    response["message"] = mensaje;
    
    String responseStr;
    serializeJson(response, responseStr);
    server.send(200, "application/json", responseStr);
  } else {
    server.send(400, "application/json", "{\"success\":false,\"message\":\"No data\"}");
  }
}

void handleApiChart() {
  DynamicJsonDocument doc(2048);
  JsonArray datos = doc.createNestedArray("datos");
  
  // Solo usar datos del buffer RAM - más simple y estable
  int puntosValidos = 0;
  for (int i = 0; i < MAX_DATOS; i++) {
    int idx = (indiceDatos + i) % MAX_DATOS;
    if (bufferDatos[idx].timestamp > 0) {
      JsonObject punto = datos.createNestedObject();
      punto["timestamp"] = bufferDatos[idx].timestamp;
      punto["temperatura"] = bufferDatos[idx].temperatura;
      
      // Convertir estado numérico a texto
      String estadoTexto = "";
      switch(bufferDatos[idx].estado) {
        case 1: estadoTexto = "NORMAL"; break;
        case 2: estadoTexto = "DESCONGELANDO"; break;
        case 3: estadoTexto = "OFF_TEMPORIZADO"; break;
        case 4: estadoTexto = "PERM_OFF"; break;
        case 5: estadoTexto = "DSXT"; break;
        default: estadoTexto = "VENTILADOR"; break;
      }
      punto["estadoTexto"] = estadoTexto;
      
      // Convertir timestamp a hora legible
      struct tm* timeinfo = localtime((time_t*)&bufferDatos[idx].timestamp);
      if (timeinfo) {
        char horaStr[6];
        strftime(horaStr, sizeof(horaStr), "%H:%M", timeinfo);
        punto["hora"] = horaStr;
        puntosValidos++;
      }
    }
  }
  
  // Metadatos básicos
  doc["umbral"] = umbral;
  doc["tempMin"] = tempMin;
  doc["totalPuntos"] = puntosValidos;
  
  String response;
  serializeJson(doc, response);
  addCorsHeaders();
  server.send(200, "application/json", response);
}

void handleApiHistoricos() {
  DynamicJsonDocument doc(4096);
  JsonArray archivos = obtenerArchivosHistoricos();
  doc["archivos"] = archivos;
  doc["totalArchivos"] = archivos.size();
  
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

void handleApiGetHorarios() {
  DynamicJsonDocument doc(2048);
  JsonArray horarios = doc.createNestedArray("horarios");
  
  for (int i = 0; i < MAX_HORARIOS; i++) {
    if (horariosSemanales[i].activo) {
      JsonObject horario = horarios.createNestedObject();
      horario["diaSemana"] = horariosSemanales[i].diaSemana;
      horario["horaInicio"] = horariosSemanales[i].horaInicio;
      horario["minutoInicio"] = horariosSemanales[i].minutoInicio;
      horario["horaFin"] = horariosSemanales[i].horaFin;
      horario["minutoFin"] = horariosSemanales[i].minutoFin;
      horario["activo"] = horariosSemanales[i].activo;
    }
  }
  
  doc["programacionActiva"] = programacionActiva;
  doc["totalHorarios"] = horarios.size();
  
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

void handleApiSetHorarios() {
  if (server.hasArg("plain")) {
    DynamicJsonDocument doc(512);
    deserializeJson(doc, server.arg("plain"));
    
    bool success = true;
    String mensaje = "Horario procesado";
    
    // Verificar si es una eliminación
    if (doc.containsKey("eliminar")) {
      int indice = doc["eliminar"];
      eliminarHorario(indice);
      mensaje = "Horario eliminado";
    } else {
      // Agregar nuevo horario
      HorarioProgramado nuevoHorario;
      nuevoHorario.activo = true;
      nuevoHorario.diaSemana = doc["diaSemana"];
      nuevoHorario.horaInicio = doc["horaInicio"];
      nuevoHorario.minutoInicio = doc["minutoInicio"];
      nuevoHorario.horaFin = doc["horaFin"];
      nuevoHorario.minutoFin = doc["minutoFin"];
      
      // Validaciones
      if (nuevoHorario.diaSemana > 6 || 
          nuevoHorario.horaInicio > 23 || nuevoHorario.minutoInicio > 59 ||
          nuevoHorario.horaFin > 23 || nuevoHorario.minutoFin > 59) {
        success = false;
        mensaje = "Valores fuera de rango";
      } else if (!agregarHorario(nuevoHorario)) {
        success = false;
        mensaje = "No hay espacio para más horarios";
      } else {
        mensaje = "Horario de apagado agregado correctamente";
        guardarHorariosEnSPIFFS();
      }
    }
    
    DynamicJsonDocument response(256);
    response["success"] = success;
    response["message"] = mensaje;
    
    String responseStr;
    serializeJson(response, responseStr);
    server.send(200, "application/json", responseStr);
  } else {
    server.send(400, "application/json", "{\"success\":false,\"message\":\"No data\"}");
  }
}

void handleApiToggleProgramacion() {
  if (server.hasArg("plain")) {
    DynamicJsonDocument doc(256);
    deserializeJson(doc, server.arg("plain"));
    
    programacionActiva = doc["activa"];
    guardarHorariosEnSPIFFS();
    
    DynamicJsonDocument response(256);
    response["success"] = true;
    response["message"] = programacionActiva ? "Programación activada" : "Programación desactivada";
    
    String responseStr;
    serializeJson(response, responseStr);
    server.send(200, "application/json", responseStr);
    
    Serial.println(programacionActiva ? "Programación de apagado ACTIVADA" : "Programación de apagado DESACTIVADA");
  } else {
    server.send(400, "application/json", "{\"success\":false,\"message\":\"No data\"}");
  }
}

void handleApiInfo() {
  addCorsHeaders();
  DynamicJsonDocument doc(1024);
  
  doc["dispositivo"] = "ESP32 Refrigeración FRIGONEL";
  doc["version"] = "2.1.0";
  doc["ip_local"] = WiFi.localIP().toString();
  doc["mac"] = WiFi.macAddress();
  doc["uptime"] = millis();
  doc["wifi_ssid"] = WiFi.SSID();
  doc["wifi_rssi"] = WiFi.RSSI();
  doc["heap_libre"] = ESP.getFreeHeap();
  doc["spiffs_total"] = SPIFFS.totalBytes();
  doc["spiffs_usado"] = SPIFFS.usedBytes();
  doc["autenticacion"] = "Habilitada";
  doc["acceso_externo"] = "Configurar Port Forwarding puerto 8080";
  
  String response;
  serializeJson(doc, response);
  server.send(200, "application/json", response);
}

void setup() {
  Serial.begin(115200);
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Iniciando...");

  pinMode(COMPRESOR_PIN, OUTPUT);
  pinMode(VENTILADOR_PIN, OUTPUT);
  pinMode(RESISTENCIA_PIN, OUTPUT);
  digitalWrite(COMPRESOR_PIN, HIGH);
  digitalWrite(VENTILADOR_PIN, HIGH);
  digitalWrite(RESISTENCIA_PIN, HIGH);

  for (int i = 0; i < 4; i++) {
    pinMode(botones[i], INPUT_PULLDOWN);
    estadoAnterior[i] = LOW;
  }

  WiFi.config(local_IP, gateway, subnet, primaryDNS, secondaryDNS);
  WiFi.begin(ssid, password);
  lcd.setCursor(0, 1);
  lcd.print("Conectando WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("WiFi conectado");
  lcd.setCursor(0, 1);
  lcd.print(WiFi.localIP());

  ArduinoOTA.setHostname("ESP32-OTA");
  ArduinoOTA.begin();

  // --- Configurar servidor web ---
  server.on("/", handleRoot);
  server.on("/api/status", handleApiStatus);
  server.on("/api/set", HTTP_POST, handleApiSet);
  server.on("/api/chart", handleApiChart);
  server.on("/api/historicos", handleApiHistoricos);
  server.on("/api/horarios", HTTP_GET, handleApiGetHorarios);
  server.on("/api/horarios", HTTP_POST, handleApiSetHorarios);
  server.on("/api/programacion", HTTP_POST, handleApiToggleProgramacion);
  server.on("/api/info", handleApiInfo);
  server.begin();

  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  delay(2000);
  
  // Inicializar SPIFFS para almacenamiento histórico
  inicializarSPIFFS();
  
  lcd.clear();

  tiempoUltimaAccion = millis();
  tiempoInicioDsCa = millis();
  teActivo = false; // Inicializar Te como inactivo
  compresorApagadoPorTe = false; // Inicializar flag
  bufferLinea2 = ""; // Inicializar buffer vacío
  
  // Inicializar buffer de datos históricos
  for (int i = 0; i < MAX_DATOS; i++) {
    bufferDatos[i].temperatura = 0.0;
    bufferDatos[i].timestamp = 0;
    bufferDatos[i].estado = 0;
  }
  ultimoRegistro = millis();
  ultimoArchivo = millis();
  
  // Inicializar SPIFFS
  inicializarSPIFFS();
  
  // Inicializar horarios programados
  inicializarHorarios();
  
  // Inicializar estado de programación
  enHorarioProgramado = false;
  
  // Registrar primer dato histórico
  delay(1000); // Esperar a que se estabilice la temperatura
  if (tempFiltrada > -900) {
    registrarDatoHistorico(tempFiltrada);
    
    // Registrar algunos datos iniciales para prueba (simulando lecturas anteriores)
    for (int i = 1; i <= 5; i++) {
      bufferDatos[(indiceDatos + i) % MAX_DATOS].temperatura = tempFiltrada + (random(-20, 20) / 10.0);
      bufferDatos[(indiceDatos + i) % MAX_DATOS].timestamp = (millis() / 1000) - (i * 300); // 5 min atrás cada uno
      bufferDatos[(indiceDatos + i) % MAX_DATOS].estado = obtenerEstadoActual();
    }
    indiceDatos = (indiceDatos + 6) % MAX_DATOS; // Actualizar índice
    
    Serial.println("Datos históricos iniciales creados para demostración");
  }
}
void loop() {
    ArduinoOTA.handle();
    server.handleClient(); // Manejar peticiones web
    procesarSerial();

    // --- Verificar programación horaria cada minuto ---
    if (millis() - ultimaVerificacionHorario >= 60000) {
        verificarProgramacionHoraria();
        ultimaVerificacionHorario = millis();
    }

    // --- Lectura y filtrado de temperatura ---
    int lectura_adc = analogRead(PIN_NTC);
    float v_adc = ((float)lectura_adc / 4095.0) * VIN;
    if (v_adc < 0.01) v_adc = 0.01;
    float r_ntc = R_FIXED * (VIN / v_adc - 1.0);
    float r_ntc_kohm = r_ntc / 1000.0;
    float temp = estimarTemperatura(r_ntc_kohm);

    if (tempFiltrada < -900) tempFiltrada = temp;
    else tempFiltrada = ALFA * temp + (1 - ALFA) * tempFiltrada;

    struct tm timeinfo;
    char horaStr[6] = {'\0'};
    char fechaStr[6] = {'\0'};
    if (getLocalTime(&timeinfo)) {
        strftime(horaStr, sizeof(horaStr), "%H:%M", &timeinfo);
        strftime(fechaStr, sizeof(fechaStr), "%d/%m", &timeinfo);
    } else {
        strcpy(horaStr, "??:??");
        strcpy(fechaStr, "??/??");
    }

    // --- Serial print periódico (cada 4 segundos) ---
    static unsigned long lastSerialPrint = 0;
    if (millis() - lastSerialPrint > 4000) {
        serialPrintEstado(tempFiltrada, fechaStr, horaStr);
        lastSerialPrint = millis();
    }

    // --- Registrar datos históricos cada 5 minutos ---
    if (millis() - ultimoRegistro >= INTERVALO_REGISTRO) {
        registrarDatoHistorico(tempFiltrada);
    }

    // --- Leer botones (edge detection) ---
    bool pulsado[4] = {false, false, false, false};
    for (int i = 0; i < 4; i++) {
        int actual = digitalRead(botones[i]);
        if (actual == HIGH && estadoAnterior[i] == LOW) {
            pulsado[i] = true;
        }
        estadoAnterior[i] = actual;
    }

    checkTimeoutMenu();

    int segundosRestantesDsCa = 0;
    if (estadoSistema == NORMAL) {
        unsigned long trans = millis() - tiempoInicioDsCa;
        long totalSegundos = DsCa * 60;
        long transcurrido = trans / 1000;
        segundosRestantesDsCa = totalSegundos - transcurrido;
        if (segundosRestantesDsCa < 0) segundosRestantesDsCa = 0;
    }
    int segundosRestantesDsOn = 0;
    if (estadoSistema == DESCONGELANDO) {
        unsigned long trans = millis() - tiempoInicioDsOn;
        int total = DsOn * 60;
        int transcurrido = trans / 1000;
        segundosRestantesDsOn = total - transcurrido;
        if (segundosRestantesDsOn < 0) segundosRestantesDsOn = 0;
    }
    int segundosRestantesDsxt = 0;
    if (estadoSistema == DESCONGELAR_XTIEMPO) {
        unsigned long trans = millis() - tiempoInicioDsxt;
        int total = tiempoDsxt * 60;
        int transcurrido = trans / 1000;
        segundosRestantesDsxt = total - transcurrido;
        if (segundosRestantesDsxt < 0) segundosRestantesDsxt = 0;
    }
    
    // --- Cálculo del tiempo restante de Te ---
    int segundosRestantesTe = 0;
    if (teActivo) {
        unsigned long trans = millis() - tiempoInicioTe;
        int total = tiempoEspera * 60;
        int transcurrido = trans / 1000;
        segundosRestantesTe = total - transcurrido;
        if (segundosRestantesTe <= 0) segundosRestantesTe = 0;
    }

    static unsigned long tiempoUltRot = 0;
    static int totalPantallas = 6; // ahora agregamos U y Tm
    static int rotActual = 0;
    if (estadoApp == ESTADO_PRINCIPAL) {
        if (millis() - tiempoUltRot > 3500) {
            rotActual = (rotActual + 1) % totalPantallas;
            tiempoUltRot = millis();
        }
    }

    static float tmpValor = 0.0;
    static int tmpDson = 0;
    static int tmpTmoff = 30;
    static int tmpDsxt = 30;
    static unsigned long tiempoMsg = 0;
    static bool editando = false;
    static bool editandoDson = false;
    static bool editandoTmoff = false;
    static bool editandoDsxt = false;
    static bool compresorPendiente = false;
    static bool ventiladorPendiente = false;
    static unsigned long tiempoInicioCompresor = 0;

    // --- Estados principales y rotación de pantalla ---
    switch (estadoSistema) {
      case NORMAL: {
        // Estado NORMAL: normalmente compresor y ventilador ON, resistencia OFF
        // PERO si fue apagado por Te, mantener compresor OFF hasta que temp >= Tm
        if (!compresorApagadoPorTe) {
            digitalWrite(COMPRESOR_PIN, LOW);  // Compresor ON solo si no fue apagado por Te
        }
        digitalWrite(VENTILADOR_PIN, LOW);
        digitalWrite(RESISTENCIA_PIN, HIGH);

        // --- Lógica del temporizador Te basado en umbral ---
        if (!teActivo && tempFiltrada < umbral) {
            // Activar temporizador Te cuando temp < umbral
            teActivo = true;
            tiempoInicioTe = millis();
        } else if (teActivo && tempFiltrada >= umbral) {
            // Cancelar temporizador Te si temp >= umbral
            teActivo = false;
        }
        
        // Verificar si Te llegó a 0 y aplicar la condición
        if (teActivo) {
            unsigned long tiempoTranscurridoTe = millis() - tiempoInicioTe;
            unsigned long tiempoTotalTe = (unsigned long)tiempoEspera * 60000UL; // Convertir minutos a milisegundos
            
            if (tiempoTranscurridoTe >= tiempoTotalTe) {
                if (tempMin > tempFiltrada) {
                    // Apagar compresor si Tm > temperatura actual
                    digitalWrite(COMPRESOR_PIN, HIGH);
                    compresorApagadoPorTe = true; // Marcar que fue apagado por Te
                }
                teActivo = false; // Desactivar el temporizador
            }
        }
        
        // --- Lógica para reactivar compresor después de Te ---
        if (compresorApagadoPorTe && tempFiltrada >= tempMin) {
            // Reactivar compresor cuando temp >= Tm después de haber sido apagado por Te
            digitalWrite(COMPRESOR_PIN, LOW);
            compresorApagadoPorTe = false; // Resetear flag
        }

        if (estadoApp == ESTADO_PRINCIPAL) {
          // Cuando DsCa llega a 0 y temp <= Tm, inicia DsOn
          if (segundosRestantesDsCa == 0) {
              if (tempFiltrada <= tempMin) {
                  digitalWrite(COMPRESOR_PIN, HIGH);
                  digitalWrite(VENTILADOR_PIN, HIGH);
                  digitalWrite(RESISTENCIA_PIN, LOW);
                  tiempoInicioDsOn = millis();
                  estadoSistema = DESCONGELANDO;
                  rotActual = 1;
                  compresorPendiente = false;
                  ventiladorPendiente = false;
                  teActivo = false; // Cancelar Te al entrar en descongelado
                  compresorApagadoPorTe = false; // Resetear flag al cambiar de estado
              } else {
                  // Si temp > Tm, reinicia DsCa y no entra a DsOn
                  tiempoInicioDsCa = millis();
              }
          }

          mostrarPantallaTemp(tempFiltrada);
          if (rotActual == 0) mostrarPantallaDsCa(segundosRestantesDsCa);
          else if (rotActual == 1) {
              if (teActivo) {
                  mostrarPantallaTeContador(segundosRestantesTe);
              } else {
                  mostrarPantallaTe(tiempoEspera);
              }
          }
          else if (rotActual == 2) mostrarPantallaReles();
          else if (rotActual == 3) mostrarPantallaDsOn(DsOn * 60);
          else if (rotActual == 4) mostrarPantallaFechaHora(fechaStr, horaStr);
          else if (rotActual == 5) mostrarPantallaUTm();
        }
        break;
      }
      case DESCONGELANDO: {
        digitalWrite(COMPRESOR_PIN, HIGH);
        digitalWrite(VENTILADOR_PIN, HIGH);
        digitalWrite(RESISTENCIA_PIN, LOW);

        if (estadoApp == ESTADO_PRINCIPAL) {
          mostrarPantallaTemp(tempFiltrada);
          
          // Rotación entre DsOn y estado de relés cada 2 segundos
          static unsigned long tiempoUltRotDsOn = 0;
          static bool mostrarRelesDsOn = false;
          if (millis() - tiempoUltRotDsOn > 2000) {
            mostrarRelesDsOn = !mostrarRelesDsOn;
            tiempoUltRotDsOn = millis();
          }
          
          if (mostrarRelesDsOn) {
            mostrarPantallaReles();
          } else {
            mostrarPantallaDsOn(segundosRestantesDsOn);
          }
        }
        if (tempFiltrada > tempMin) {
            // Cancelar descongelado si temp sube de Tm
            digitalWrite(RESISTENCIA_PIN, HIGH); // Apagar resistencia al salir del descongelado
            estadoSistema = NORMAL;
            tiempoInicioDsCa = millis();
            compresorApagadoPorTe = false; // Resetear flag al cambiar de estado
        } else if ((millis() - tiempoInicioDsOn) >= ((unsigned long)DsOn*60000UL)) {
            digitalWrite(RESISTENCIA_PIN, HIGH);
            digitalWrite(COMPRESOR_PIN, LOW);   // Enciende compresor (LOW es ON)
            digitalWrite(VENTILADOR_PIN, HIGH); // Ventilador aún OFF
            compresorPendiente = true;
            ventiladorPendiente = false;
            tiempoInicioCompresor = millis();
            estadoSistema = VENTILADOR_ESPERA;
            rotActual = 0;
        }
        break;
      }
      case VENTILADOR_ESPERA: {
        digitalWrite(RESISTENCIA_PIN, HIGH);
        
        unsigned long tiempoTranscurrido = millis() - tiempoInicioCompresor;
        
        // Verificar si el ciclo completo ha terminado
        if (tiempoTranscurrido >= 5 * 60000UL + 1000) { // 1 segundo después de los 5 minutos
            // Al terminar el ciclo, establecer los relés para estado NORMAL primero
            digitalWrite(COMPRESOR_PIN, LOW);   // Estado NORMAL: Compresor ON
            digitalWrite(VENTILADOR_PIN, LOW);  // Estado NORMAL: Ventilador ON
            digitalWrite(RESISTENCIA_PIN, HIGH); // Estado NORMAL: Resistencia OFF
            tiempoInicioDsCa = millis();
            estadoSistema = NORMAL;
            rotActual = 0; // Reiniciar rotación desde el principio
            compresorApagadoPorTe = false; // Resetear flag al cambiar de estado
        } else {
            // Mantener compresor encendido durante todo el estado
            digitalWrite(COMPRESOR_PIN, LOW); // Compresor ON
            
            // Controlar ventilador según el tiempo transcurrido
            if (tiempoTranscurrido >= 5 * 60000UL) {
              digitalWrite(VENTILADOR_PIN, LOW);  // Prende el ventilador después de 5 minutos
              ventiladorPendiente = false;
            } else {
              digitalWrite(VENTILADOR_PIN, HIGH); // Apaga antes de los 5 minutos
              ventiladorPendiente = true;
            }
        }

        // Mostrar pantalla durante la espera del ventilador
        if (estadoApp == ESTADO_PRINCIPAL) {
          mostrarPantallaTemp(tempFiltrada);
          
          // Rotación entre tiempo de ventilador y estado de relés cada 2 segundos
          static unsigned long tiempoUltRotVent = 0;
          static bool mostrarRelesVent = false;
          if (millis() - tiempoUltRotVent > 2000) {
            mostrarRelesVent = !mostrarRelesVent;
            tiempoUltRotVent = millis();
          }
          
          if (mostrarRelesVent) {
            mostrarPantallaReles();
          } else {
            // Mostrar tiempo restante para que se active el ventilador
            int segundosRestantes = (5 * 60) - (tiempoTranscurrido / 1000);
            if (segundosRestantes < 0) segundosRestantes = 0;
            
            String texto = "Vent Off=";
            int minutos = segundosRestantes / 60;
            int segundos = segundosRestantes % 60;
            if (minutos < 10) texto += "0";
            texto += String(minutos);
            texto += ":";
            if (segundos < 10) texto += "0";
            texto += String(segundos);
            texto += "m";
            actualizarLinea2(texto);
          }
        }
        break;
      }
      case OFF_TEMPORIZADO: {
        apagarTodosLosRele();
        if (estadoApp == ESTADO_PRINCIPAL) {
          unsigned long trans = millis() - tiempoInicioModoOff;
          int segundosRestantes = (Tmoff*60) - (trans/1000);
          if (segundosRestantes < 0) segundosRestantes = 0;
          mostrarPantallaModoOff(tempFiltrada, segundosRestantes);
        }
        if ((millis() - tiempoInicioModoOff) >= ((unsigned long)Tmoff*60000UL)) {
            tiempoInicioDsCa = millis();
            estadoSistema = NORMAL;
        }
        break;
      }
      case PERM_OFF: {
        apagarTodosLosRele();
        if (estadoApp == ESTADO_PRINCIPAL) {
          mostrarPantallaPermOff(tempFiltrada);
        }
        break;
      }
      case DESCONGELAR_XTIEMPO: {
        digitalWrite(COMPRESOR_PIN, HIGH);
        digitalWrite(VENTILADOR_PIN, HIGH);
        digitalWrite(RESISTENCIA_PIN, LOW);
        if (estadoApp == ESTADO_PRINCIPAL) {
            mostrarPantallaTemp(tempFiltrada);
            mostrarPantallaDsxt(segundosRestantesDsxt);
        }
        if ((millis() - tiempoInicioDsxt) >= ((unsigned long)tiempoDsxt*60000UL)) {
            digitalWrite(RESISTENCIA_PIN, HIGH);
            tiempoInicioDsCa = millis();
            estadoSistema = NORMAL;
        }
        break;
      }
    }

    // Menú y edición
    switch (estadoApp) {
      case ESTADO_PRINCIPAL:
        if (pulsado[0]) {
          refrescarTimeout();
          estadoApp = MENU_PRINCIPAL;
          opcionMenuPrincipal = 0;
          mostrarMenuPrincipal(opcionMenuPrincipal);
          delay(200);
        }
        break;
      case MENU_PRINCIPAL:
        if (pulsado[1]) {
          refrescarTimeout();
          opcionMenuPrincipal = (opcionMenuPrincipal + 1) % numOpcionesMenuPrincipal;
          mostrarMenuPrincipal(opcionMenuPrincipal);
          delay(120);
        }
        if (pulsado[0]) {
          refrescarTimeout();
          opcionMenuPrincipal = (opcionMenuPrincipal - 1 + numOpcionesMenuPrincipal) % numOpcionesMenuPrincipal;
          mostrarMenuPrincipal(opcionMenuPrincipal);
          delay(120);
        }
        if (pulsado[2]) {
          refrescarTimeout();
          if (opcionMenuPrincipal == 0) {
            estadoApp = MENU_PARAM_GEN;
            opcionParmGen = 0;
            mostrarMenuParmGen(opcionParmGen);
            delay(200);
          } else if (opcionMenuPrincipal == 1) {
            estadoApp = MENU_MODO;
            opcionModo = 0;
            mostrarMenuModo(opcionModo);
            delay(200);
          } else if (opcionMenuPrincipal == 2) {
            if (estadoSistema != PERM_OFF) {
              estadoApp = DESCON_XTIEMPO_CONFIRM;
              opcionDesconXTiempoSiNo = 0;
              mostrarDesconXTiempoConfirm(opcionDesconXTiempoSiNo);
            } else {
              lcd.clear();
              lcd.setCursor(0, 0);
              lcd.print("No disponible en");
              lcd.setCursor(0, 1);
              lcd.print("Perm Off");
              delay(1200);
              estadoApp = ESTADO_PRINCIPAL;
              lcd.clear();
            }
            delay(200);
          }
        }
        if (pulsado[3]) {
          refrescarTimeout();
          estadoApp = ESTADO_PRINCIPAL;
          lcd.clear();
          bufferLinea2 = ""; // Limpiar buffer al salir del menú
          delay(200);
        }
        break;
      case MENU_PARAM_GEN:
        if (pulsado[1]) {
          refrescarTimeout();
          opcionParmGen = (opcionParmGen + 1) % numOpcionesParmGen;
          mostrarMenuParmGen(opcionParmGen);
          delay(120);
        }
        if (pulsado[0]) {
          refrescarTimeout();
          opcionParmGen = (opcionParmGen - 1 + numOpcionesParmGen) % numOpcionesParmGen;
          mostrarMenuParmGen(opcionParmGen);
          delay(120);
        }
        if (pulsado[2]) {
          refrescarTimeout();
          editandoIndice = opcionParmGen;
          switch (editandoIndice) {
            case 0: tmpValor = umbral; break;
            case 1: tmpValor = tiempoEspera; break;
            case 2: tmpValor = tempMin; break;
            case 3: tmpValor = DsCa; break;
          }
          editando = true;
          mostrarEdicion(editandoIndice, tmpValor);
          estadoApp = EDIT_GENERIC;
          delay(200);
        }
        if (pulsado[3]) {
          refrescarTimeout();
          estadoApp = MENU_PRINCIPAL;
          mostrarMenuPrincipal(opcionMenuPrincipal);
          delay(200);
        }
        break;
      case EDIT_GENERIC: {
        static float ultimoValorMostrado = 99999;
        static int ultimoIndiceMostrado = -1;
        if (editando) {
          if (pulsado[1]) refrescarTimeout();
          if (pulsado[0]) refrescarTimeout();
          switch (editandoIndice) {
            case 0:
              if (pulsado[1] && tmpValor < 35) tmpValor += 1;
              if (pulsado[0] && tmpValor > -15) tmpValor -= 1;
              break;
            case 1:
              if (pulsado[1] && tmpValor < 300) tmpValor += 1;
              if (pulsado[0] && tmpValor > 2) tmpValor -= 1;
              break;
            case 2: {
              float minPermitida = umbral + 4.0;
              if (pulsado[1] && tmpValor < 35) tmpValor += 1;
              if (pulsado[0] && tmpValor > minPermitida) tmpValor -= 1;
              break;
            }
            case 3:
              if (pulsado[1] && tmpValor < 999) tmpValor += 1;
              if (pulsado[0] && tmpValor > 2) tmpValor -= 1;
              break;
          }
          if (tmpValor != ultimoValorMostrado || editandoIndice != ultimoIndiceMostrado) {
            mostrarEdicion(editandoIndice, tmpValor);
            ultimoValorMostrado = tmpValor;
            ultimoIndiceMostrado = editandoIndice;
          }
          if (pulsado[2]) {
            refrescarTimeout();
            switch (editandoIndice) {
              case 0: umbral = tmpValor; if (tempMin < umbral + 4.0) tempMin = umbral + 4.0; break;
              case 1: tiempoEspera = (int)tmpValor; break;
              case 2: tempMin = tmpValor; break;
              case 3:
                DsCa = (int)tmpValor;
                tmpDson = DsOn;
                editandoDson = true;
                mostrarEdicionDson(tmpDson);
                ultimoValorMostrado = 99999;
                estadoApp = EDIT_DSON;
                break;
            }
            if (editandoIndice != 3) {
              editando = false;
              mostrarGuardadoOK();
              tiempoMsg = millis();
              ultimoValorMostrado = 99999;
              estadoApp = EDIT_GUARDADO;
            }
          }
          if (pulsado[3]) {
            refrescarTimeout();
            editando = false;
            mostrarCancelado();
            tiempoMsg = millis();
            ultimoValorMostrado = 99999;
            estadoApp = EDIT_CANCELADO;
          }
        }
        break;
      }
      case EDIT_DSON: {
        static int ultimoValorMostrado = 99999;
        if (editandoDson) {
          if (pulsado[1]) refrescarTimeout();
          if (pulsado[0]) refrescarTimeout();
          if (pulsado[1] && tmpDson < 999) tmpDson += 1;
          if (pulsado[0] && tmpDson > 1) tmpDson -= 1;
          if (tmpDson != ultimoValorMostrado) {
            mostrarEdicionDson(tmpDson);
            ultimoValorMostrado = tmpDson;
          }
          if (pulsado[2]) {
            refrescarTimeout();
            DsOn = tmpDson;
            editandoDson = false;
            mostrarGuardadoOK();
            tiempoMsg = millis();
            ultimoValorMostrado = 99999;
            estadoApp = EDIT_GUARDADO;
          }
          if (pulsado[3]) {
            refrescarTimeout();
            editandoDson = false;
            mostrarCancelado();
            tiempoMsg = millis();
            ultimoValorMostrado = 99999;
            estadoApp = EDIT_CANCELADO;
          }
        }
        break;
      }
      case MENU_MODO:
        if (pulsado[1]) {
          refrescarTimeout();
          opcionModo = (opcionModo + 1) % numOpcionesModo;
          mostrarMenuModo(opcionModo);
          delay(120);
        }
        if (pulsado[0]) {
          refrescarTimeout();
          opcionModo = (opcionModo - 1 + numOpcionesModo) % numOpcionesModo;
          mostrarMenuModo(opcionModo);
          delay(120);
        }
        if (pulsado[2]) {
          refrescarTimeout();
          if (opcionModo == 0) { // Modo OFF
            estadoApp = MODO_OFF_CONFIRM;
            opcionModoOffSiNo = 0;
            mostrarModoOffConfirm(opcionModoOffSiNo);
            delay(200);
          } else if (opcionModo == 1) { // Perm Off
            opcionPermOffSiNo = 0;
            mostrarMenuPermOffSiNo(opcionPermOffSiNo);
            estadoApp = PERM_OFF_SINO;
            delay(200);
          } else if (opcionModo == 2) { // Modo ON
            opcionModoOnSiNo = 0;
            mostrarModoOnConfirm(opcionModoOnSiNo);
            estadoApp = MODO_ON_CONFIRM;
            delay(200);
          }
        }
        if (pulsado[3]) {
          refrescarTimeout();
          estadoApp = MENU_PRINCIPAL;
          mostrarMenuPrincipal(opcionMenuPrincipal);
          delay(200);
        }
        break;
      case MODO_ON_CONFIRM:
        if (pulsado[1]) {
          refrescarTimeout();
          opcionModoOnSiNo = (opcionModoOnSiNo + 1) % 2;
          mostrarModoOnConfirm(opcionModoOnSiNo);
          delay(120);
        }
        if (pulsado[0]) {
          refrescarTimeout();
          opcionModoOnSiNo = (opcionModoOnSiNo - 1 + 2) % 2;
          mostrarModoOnConfirm(opcionModoOnSiNo);
          delay(120);
        }
        if (pulsado[2]) {
          refrescarTimeout();
          if (opcionModoOnSiNo == 0) {
            estadoSistema = NORMAL;
            tiempoInicioDsCa = millis();
            mostrarGuardadoOK();
            tiempoMsg = millis();
            estadoApp = EDIT_GUARDADO;
          } else {
            estadoApp = MENU_MODO;
            mostrarMenuModo(opcionModo);
          }
          delay(200);
        }
        if (pulsado[3]) {
          refrescarTimeout();
          estadoApp = MENU_MODO;
          mostrarMenuModo(opcionModo);
          delay(200);
        }
        break;
      case MODO_OFF_CONFIRM:
        if (pulsado[1]) {
          refrescarTimeout();
          opcionModoOffSiNo = (opcionModoOffSiNo + 1) % 2;
          mostrarModoOffConfirm(opcionModoOffSiNo);
          delay(120);
        }
        if (pulsado[0]) {
          refrescarTimeout();
          opcionModoOffSiNo = (opcionModoOffSiNo - 1 + 2) % 2;
          mostrarModoOffConfirm(opcionModoOffSiNo);
          delay(120);
        }
        if (pulsado[2]) {
          refrescarTimeout();
          if (opcionModoOffSiNo == 0) {
            tmpTmoff = Tmoff;
            editandoTmoff = true;
            mostrarEdicionTmoff(tmpTmoff);
            estadoApp = MODO_OFF_EDIT;
          } else {
            estadoApp = MENU_MODO;
            mostrarMenuModo(opcionModo);
          }
          delay(200);
        }
        if (pulsado[3]) {
          refrescarTimeout();
          estadoApp = MENU_MODO;
          mostrarMenuModo(opcionModo);
          delay(200);
        }
        break;
      case MODO_OFF_EDIT: {
        static int ultimoTmoffMostrado = 99999;
        if (editandoTmoff) {
          if (pulsado[1] && tmpTmoff < 999) { tmpTmoff += 1; refrescarTimeout();}
          if (pulsado[0] && tmpTmoff > 1) { tmpTmoff -= 1; refrescarTimeout();}
          if (tmpTmoff != ultimoTmoffMostrado) {
            mostrarEdicionTmoff(tmpTmoff);
            ultimoTmoffMostrado = tmpTmoff;
          }
          if (pulsado[2]) {
            refrescarTimeout();
            Tmoff = tmpTmoff;
            editandoTmoff = false;
            mostrarGuardadoOK();
            tiempoMsg = millis();
            ultimoTmoffMostrado = 99999;
            tiempoInicioModoOff = millis();
            estadoSistema = OFF_TEMPORIZADO;
            estadoApp = EDIT_GUARDADO;
          }
          if (pulsado[3]) {
            refrescarTimeout();
            editandoTmoff = false;
            mostrarCancelado();
            tiempoMsg = millis();
            ultimoTmoffMostrado = 99999;
            estadoApp = EDIT_CANCELADO;
          }
        }
        break;
      }
      case PERM_OFF_SINO:
        if (pulsado[1]) {
          refrescarTimeout();
          opcionPermOffSiNo = (opcionPermOffSiNo + 1) % 2;
          mostrarMenuPermOffSiNo(opcionPermOffSiNo);
          delay(120);
        }
        if (pulsado[0]) {
          refrescarTimeout();
          opcionPermOffSiNo = (opcionPermOffSiNo - 1 + 2) % 2;
          mostrarMenuPermOffSiNo(opcionPermOffSiNo);
          delay(120);
        }
        if (pulsado[2]) {
          refrescarTimeout();
          if (opcionPermOffSiNo == 0) {
            estadoSistema = PERM_OFF;
            mostrarGuardadoOK();
                       tiempoMsg = millis();
            estadoApp = EDIT_GUARDADO;
          } else {
            estadoApp = MENU_MODO;
            mostrarMenuModo(opcionModo);
          }
          delay(200);
        }
        if (pulsado[3]) {
          refrescarTimeout();
          estadoApp = MENU_MODO;
          mostrarMenuModo(opcionModo);
          delay(200);
        }
        break;
      case DESCON_XTIEMPO_CONFIRM:
        if (pulsado[1]) {
          refrescarTimeout();
          opcionDesconXTiempoSiNo = (opcionDesconXTiempoSiNo + 1) % 2;
          mostrarDesconXTiempoConfirm(opcionDesconXTiempoSiNo);
          delay(120);
        }
        if (pulsado[0]) {
          refrescarTimeout();
          opcionDesconXTiempoSiNo = (opcionDesconXTiempoSiNo - 1 + 2) % 2;
          mostrarDesconXTiempoConfirm(opcionDesconXTiempoSiNo);
          delay(120);
        }
        if (pulsado[2]) {
          refrescarTimeout();
          if (opcionDesconXTiempoSiNo == 0) {
            tmpDsxt = Dsxt;
            editandoDsxt = true;
            mostrarEdicionDsxt(tmpDsxt);
            estadoApp = DESCON_XTIEMPO_EDIT;
          } else {
            estadoApp = MENU_PRINCIPAL;
            mostrarMenuPrincipal(opcionMenuPrincipal);
          }
          delay(200);
        }
        if (pulsado[3]) {
          refrescarTimeout();
          estadoApp = MENU_PRINCIPAL;
          mostrarMenuPrincipal(opcionMenuPrincipal);
          delay(200);
        }
        break;
      case DESCON_XTIEMPO_EDIT: {
        static int ultimoDsxtMostrado = 99999;
        if (editandoDsxt) {
          if (pulsado[1] && tmpDsxt < 999) { tmpDsxt += 1; refrescarTimeout();}
          if (pulsado[0] && tmpDsxt > 1) { tmpDsxt -= 1; refrescarTimeout();}
          if (tmpDsxt != ultimoDsxtMostrado) {
            mostrarEdicionDsxt(tmpDsxt);
            ultimoDsxtMostrado = tmpDsxt;
          }
          if (pulsado[2]) {
            refrescarTimeout();
            Dsxt = tmpDsxt;
            tiempoDsxt = Dsxt;
            tiempoInicioDsxt = millis();
            editandoDsxt = false;
            mostrarGuardadoOK();
            tiempoMsg = millis();
            ultimoDsxtMostrado = 99999;
            estadoSistema = DESCONGELAR_XTIEMPO;
            estadoApp = EDIT_GUARDADO;
          }
          if (pulsado[3]) {
            refrescarTimeout();
            editandoDsxt = false;
            mostrarCancelado();
            tiempoMsg = millis();
            ultimoDsxtMostrado = 99999;
            estadoApp = EDIT_CANCELADO;
          }
        }
        break;
      }
      case EDIT_GUARDADO:
        if (millis() - tiempoMsg > 1200) {
          estadoApp = ESTADO_PRINCIPAL;
          lcd.clear();
          bufferLinea2 = ""; // Limpiar buffer al salir
        }
        break;
      case EDIT_CANCELADO:
        if (millis() - tiempoMsg > 1200) {
          estadoApp = ESTADO_PRINCIPAL;
          lcd.clear();
          bufferLinea2 = ""; // Limpiar buffer al salir
        }
        break;
      default:
        estadoApp = ESTADO_PRINCIPAL;
        break;
    }

    delay(50);
}

// --- Funciones SPIFFS para almacenamiento histórico ---
void inicializarSPIFFS() {
  if (!SPIFFS.begin(true)) {
    Serial.println("Error iniciando SPIFFS");
    spiffsIniciado = false;
  } else {
    Serial.println("SPIFFS iniciado correctamente");
    spiffsIniciado = true;
    
    // Mostrar información del sistema de archivos
    size_t totalBytes = SPIFFS.totalBytes();
    size_t usedBytes = SPIFFS.usedBytes();
    Serial.printf("SPIFFS: %u bytes totales, %u bytes usados (%.1f%% usado)\n", 
                  totalBytes, usedBytes, (float)usedBytes/totalBytes*100);
    
    // Listar archivos existentes
    Serial.println("Archivos históricos existentes:");
    File root = SPIFFS.open("/");
    File file = root.openNextFile();
    int contadorArchivos = 0;
    
    while (file) {
      String nombre = file.name();
      if (nombre.startsWith("/hist_") && nombre.endsWith(".csv")) {
        Serial.printf("  - %s (%u bytes)\n", nombre.c_str(), file.size());
        contadorArchivos++;
      }
      file = root.openNextFile();
    }
    root.close();
    
    Serial.printf("Total de archivos históricos: %d\n", contadorArchivos);
    
    // Limpiar archivos antiguos (más de 1 semana)
    limpiarArchivosAntiguos();
  }
}

String obtenerNombreArchivoActual() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    return "/datos_temp.csv"; // Archivo temporal si no hay fecha
  }
  
  // Crear nombre basado en fecha y hora cada 2 horas
  int horaArchivo = (timeinfo.tm_hour / 2) * 2; // 0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22
  
  char buffer[50];
  sprintf(buffer, "/hist_%04d%02d%02d_%02d%02d.csv", 
          timeinfo.tm_year + 1900, 
          timeinfo.tm_mon + 1, 
          timeinfo.tm_mday, 
          horaArchivo, 
          0);
  return String(buffer);
}

uint8_t obtenerEstadoActual() {
  switch(estadoSistema) {
    case NORMAL: return 0;
    case DESCONGELANDO: return 1;
    case VENTILADOR_ESPERA: return 2;
    case OFF_TEMPORIZADO: return 3;
    case PERM_OFF: return 4;
    case DESCONGELAR_XTIEMPO: return 5;
    default: return 0;
  }
}

void registrarDatoHistorico(float temperatura) {
  struct tm timeinfo;
  uint32_t timestamp = 0;
  
  if (getLocalTime(&timeinfo)) {
    timestamp = mktime(&timeinfo);
  } else {
    timestamp = millis() / 1000; // Timestamp relativo si no hay NTP
  }
  
  uint8_t estado = obtenerEstadoActual();
  
  // Agregar al buffer RAM (circular)
  bufferDatos[indiceDatos].temperatura = temperatura;
  bufferDatos[indiceDatos].timestamp = timestamp;
  bufferDatos[indiceDatos].estado = estado;
  indiceDatos = (indiceDatos + 1) % MAX_DATOS;
  
  // Verificar si es hora de archivar (cada 2 horas)
  if (spiffsIniciado && (millis() - ultimoArchivo >= INTERVALO_ARCHIVO)) {
    archivarDatosActuales();
    ultimoArchivo = millis();
  }
  
  ultimoRegistro = millis();
}

void archivarDatosActuales() {
  String nombreArchivo = obtenerNombreArchivoActual();
  
  // Verificar si hay datos válidos para archivar
  int datosValidos = 0;
  for (int i = 0; i < MAX_DATOS; i++) {
    if (bufferDatos[i].timestamp > 0) {
      datosValidos++;
    }
  }
  
  if (datosValidos == 0) {
    return;
  }
  
  File file = SPIFFS.open(nombreArchivo, FILE_APPEND);
  if (!file) {
    return;
  }
  
  // Escribir encabezado solo si el archivo es nuevo (tamaño 0)
  if (file.size() == 0) {
    file.println("timestamp,temperatura,estado,fecha_hora");
  }
  
  // Escribir todos los datos del buffer actual ordenados por timestamp
  for (int i = 0; i < MAX_DATOS; i++) {
    int idx = (indiceDatos + i) % MAX_DATOS;
    if (bufferDatos[idx].timestamp > 0) {
      struct tm* timeinfo = localtime((time_t*)&bufferDatos[idx].timestamp);
      if (timeinfo) {
        char fechaHora[25];
        strftime(fechaHora, sizeof(fechaHora), "%Y-%m-%d %H:%M:%S", timeinfo);
        
        file.printf("%u,%.1f,%u,%s\n", 
                    bufferDatos[idx].timestamp,
                    bufferDatos[idx].temperatura,
                    bufferDatos[idx].estado,
                    fechaHora);
      }
    }
  }
  
  file.close();
  
  // Limpiar archivos antiguos (más de 1 semana)
  limpiarArchivosAntiguos();
}

void limpiarArchivosAntiguos() {
  if (!spiffsIniciado) return;
  
  time_t ahora;
  time(&ahora);
  
  File root = SPIFFS.open("/");
  File file = root.openNextFile();
  
  while (file) {
    String nombre = file.name();
    
    // Solo procesar archivos históricos
    if (nombre.startsWith("/hist_") && nombre.endsWith(".csv")) {
      // Extraer fecha del nombre del archivo
      String fecha = nombre.substring(6, 14); // YYYYMMDD
      String hora = nombre.substring(15, 19);  // HHMM
      
      // Convertir a timestamp
      struct tm tm_archivo = {0};
      tm_archivo.tm_year = fecha.substring(0,4).toInt() - 1900;
      tm_archivo.tm_mon = fecha.substring(4,6).toInt() - 1;
      tm_archivo.tm_mday = fecha.substring(6,8).toInt();
      tm_archivo.tm_hour = hora.substring(0,2).toInt();
      tm_archivo.tm_min = hora.substring(2,4).toInt();
      
      time_t timestampArchivo = mktime(&tm_archivo);
      
      // Si el archivo es más antiguo que una semana, eliminarlo
      if ((ahora - timestampArchivo) > RETENCION_MAXIMA) {
        SPIFFS.remove(nombre);
      }
    }
    
    file = root.openNextFile();
  }
  
  root.close();
}

JsonArray obtenerArchivosHistoricos() {
  DynamicJsonDocument doc(1024);
  JsonArray archivos = doc.createNestedArray();
  
  // Respuesta simple - los datos están en RAM, no en archivos
  if (spiffsIniciado) {
    JsonObject archivo = archivos.createNestedObject();
    archivo["nombre"] = "buffer_ram";
    archivo["tamano"] = MAX_DATOS * 16; // Estimación del tamaño
    archivo["fecha_hora"] = "Datos en RAM (Últimas 4 horas)";
  }
  
  return archivos;
}

JsonArray leerArchivoHistorico(String nombreArchivo) {
  DynamicJsonDocument doc(8192);
  JsonArray datos = doc.createNestedArray();
  
  if (!spiffsIniciado) return datos;
  
  File file = SPIFFS.open(nombreArchivo, FILE_READ);
  if (!file) {
    return datos;
  }
  
  // Saltar encabezado
  file.readStringUntil('\n');
  
  while (file.available()) {
    String linea = file.readStringUntil('\n');
    linea.trim();
    
    if (linea.length() > 0) {
      // Parsear CSV: timestamp,temperatura,estado,fecha_hora
      int pos1 = linea.indexOf(',');
      int pos2 = linea.indexOf(',', pos1 + 1);
      int pos3 = linea.indexOf(',', pos2 + 1);
      
      if (pos1 > 0 && pos2 > pos1 && pos3 > pos2) {
        JsonObject punto = datos.createNestedObject();
        punto["timestamp"] = linea.substring(0, pos1).toInt();
        punto["temperatura"] = linea.substring(pos1 + 1, pos2).toFloat();
        punto["estado"] = linea.substring(pos2 + 1, pos3).toInt();
        punto["fecha_hora"] = linea.substring(pos3 + 1);
      }
    }
  }
  
  file.close();
  return datos;
}

// --- Funciones de programación horaria ---
void inicializarHorarios() {
  // Inicializar todos los horarios como inactivos
  for (int i = 0; i < MAX_HORARIOS; i++) {
    horariosSemanales[i].activo = false;
  }
  programacionActiva = false;
  
  // Cargar horarios desde SPIFFS si existe el archivo
  cargarHorariosDesdeSPIFFS();
  
  Serial.println("Sistema de programación horaria inicializado");
}

bool agregarHorario(HorarioProgramado horario) {
  // Buscar un slot libre
  for (int i = 0; i < MAX_HORARIOS; i++) {
    if (!horariosSemanales[i].activo) {
      horariosSemanales[i] = horario;
      Serial.printf("Horario de apagado agregado en slot %d: Día %d, %02d:%02d-%02d:%02d\n", 
                    i, horario.diaSemana, horario.horaInicio, horario.minutoInicio, 
                    horario.horaFin, horario.minutoFin);
      return true;
    }
  }
  return false; // No hay espacio
}

void eliminarHorario(int indice) {
  int contador = 0;
  for (int i = 0; i < MAX_HORARIOS; i++) {
    if (horariosSemanales[i].activo) {
      if (contador == indice) {
        horariosSemanales[i].activo = false;
        Serial.printf("Horario eliminado del slot %d\n", i);
        guardarHorariosEnSPIFFS();
        return;
      }
      contador++;
    }
  }
}

void verificarProgramacionHoraria() {
  if (!programacionActiva) {
    // Si la programación está desactivada pero estábamos en horario programado, volver a ON
    if (enHorarioProgramado) {
      enHorarioProgramado = false;
      if (estadoSistema == PERM_OFF) {
        estadoSistema = NORMAL;
        tiempoInicioDsCa = millis();
        compresorApagadoPorTe = false;
        Serial.println("Programación desactivada: Volviendo a Modo ON");
      }
    }
    return;
  }
  
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) return;
  
  int diaActual = timeinfo.tm_wday; // 0=Domingo, 1=Lunes, etc.
  int horaActual = timeinfo.tm_hour;
  int minutoActual = timeinfo.tm_min;
  int minutosActual = horaActual * 60 + minutoActual;
  
  bool deberiaEstarOFF = false;
  
  // Verificar si estamos dentro de algún horario de apagado
  for (int i = 0; i < MAX_HORARIOS; i++) {
    if (!horariosSemanales[i].activo) continue;
    
    int minutosInicio = horariosSemanales[i].horaInicio * 60 + horariosSemanales[i].minutoInicio;
    int minutosFin = horariosSemanales[i].horaFin * 60 + horariosSemanales[i].minutoFin;
    
    bool dentroDelRango = false;
    
    if (minutosFin > minutosInicio) {
      // Rango normal (no cruza medianoche) - verificar solo el día configurado
      if (horariosSemanales[i].diaSemana == diaActual) {
        dentroDelRango = (minutosActual >= minutosInicio && minutosActual < minutosFin);
      }
    } else {
      // Rango que cruza medianoche - verificar dos días
      int diaAnterior = (diaActual == 0) ? 6 : diaActual - 1; // Si es domingo (0), el anterior es sábado (6)
      
      if (horariosSemanales[i].diaSemana == diaActual) {
        // Estamos en el día de inicio - verificar si estamos después de la hora de inicio
        dentroDelRango = (minutosActual >= minutosInicio);
      } else if (horariosSemanales[i].diaSemana == diaAnterior) {
        // Estamos en el día siguiente - verificar si estamos antes de la hora de fin
        dentroDelRango = (minutosActual < minutosFin);
      }
    }
    
    if (dentroDelRango) {
      deberiaEstarOFF = true;
      break; // Encontramos un horario de apagado activo
    }
  }
  
  // Aplicar la lógica
  if (deberiaEstarOFF && !enHorarioProgramado) {
    // Entrar en horario de apagado - cambiar a PERM_OFF
    enHorarioProgramado = true;
    if (estadoSistema != PERM_OFF) {
      estadoSistema = PERM_OFF;
      Serial.printf("Programación: Activando apagado programado - %02d:%02d\n", horaActual, minutoActual);
    }
  } else if (!deberiaEstarOFF && enHorarioProgramado) {
    // Salir del horario de apagado - volver a NORMAL
    enHorarioProgramado = false;
    if (estadoSistema == PERM_OFF) {
      estadoSistema = NORMAL;
      tiempoInicioDsCa = millis();
      compresorApagadoPorTe = false;
      Serial.printf("Programación: Fin del apagado programado, volviendo a Modo ON - %02d:%02d\n", horaActual, minutoActual);
    }
  }
}

void guardarHorariosEnSPIFFS() {
  if (!spiffsIniciado) return;
  
  DynamicJsonDocument doc(2048);
  doc["programacionActiva"] = programacionActiva;
  
  JsonArray horarios = doc.createNestedArray("horarios");
  for (int i = 0; i < MAX_HORARIOS; i++) {
    if (horariosSemanales[i].activo) {
      JsonObject horario = horarios.createNestedObject();
      horario["diaSemana"] = horariosSemanales[i].diaSemana;
      horario["horaInicio"] = horariosSemanales[i].horaInicio;
      horario["minutoInicio"] = horariosSemanales[i].minutoInicio;
      horario["horaFin"] = horariosSemanales[i].horaFin;
      horario["minutoFin"] = horariosSemanales[i].minutoFin;
    }
  }
  
  File file = SPIFFS.open("/horarios.json", FILE_WRITE);
  if (file) {
    serializeJson(doc, file);
    file.close();
    Serial.println("Horarios de apagado guardados en SPIFFS");
  }
}

void cargarHorariosDesdeSPIFFS() {
  if (!spiffsIniciado) return;
  
  File file = SPIFFS.open("/horarios.json", FILE_READ);
  if (!file) {
    Serial.println("No se encontró archivo de horarios, usando configuración por defecto");
    return;
  }
  
  DynamicJsonDocument doc(2048);
  if (deserializeJson(doc, file) == DeserializationError::Ok) {
    programacionActiva = doc["programacionActiva"] | false;
    
    JsonArray horarios = doc["horarios"];
    int index = 0;
    
    for (JsonVariant horarioVar : horarios) {
      if (index >= MAX_HORARIOS) break;
      
      JsonObject horario = horarioVar.as<JsonObject>();
      horariosSemanales[index].activo = true;
      horariosSemanales[index].diaSemana = horario["diaSemana"];
      horariosSemanales[index].horaInicio = horario["horaInicio"];
      horariosSemanales[index].minutoInicio = horario["minutoInicio"];
      horariosSemanales[index].horaFin = horario["horaFin"];
      horariosSemanales[index].minutoFin = horario["minutoFin"];
      
      index++;
    }
    
    Serial.printf("Horarios cargados desde SPIFFS: %d horarios, programación %s\n", 
                  index, programacionActiva ? "activa" : "inactiva");
  }
  
  file.close();
}