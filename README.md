//-------------esp32---------------
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>

// =============== LCD ===============
LiquidCrystal_I2C lcd(0x27, 16, 2);

// =============== KEYPAD ===============
const byte ROWS = 4;
const byte COLS = 4;

char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};

byte rowPins[ROWS] = {13, 14, 18, 19};
byte colPins[COLS] = {23, 25, 26, 27};

Keypad keypad = Keypad(
  makeKeymap(keys), rowPins, colPins, ROWS, COLS
);

// =============== PINS ===============
#define ECG_PIN 34
#define LO_PLUS 35
#define LO_MINUS 36
#define BUZZER 32

// SIM800L : RX GPIO16, TX GPIO17
HardwareSerial SIM800(1);

// Communication UNO : RX GPIO33, TX GPIO4
HardwareSerial UNO(2);

// =============== PARAMETERS ===============
const unsigned long SAMPLE_US = 5000;
const unsigned long REFRACTORY_MS = 300;
const unsigned long BEEP_MS = 40;

const int BPM_BRADY = 50;
const int BPM_TACHY = 120;

float baseline = 2048.0;
float envelope = 150.0;

const float THRESH_FRAC = 0.60;

unsigned long lastSampleUs = 0;
unsigned long lastBeatMs = 0;
unsigned long beepUntil = 0;
unsigned long lastAlertMs = 0;

bool aboveThreshold = false;
bool patientValidated = false;
bool buzzerMuted = false;
bool alarmActive = false;

int bpm = 0;
int sampleCounter = 0;
int dangerCounter = 0;

String numero = "";

// =============== SMS QUEUE ===============
QueueHandle_t smsQueue;

void queueSMS(const String &message) {
  char msg[180] = {0};
  message.toCharArray(msg, sizeof(msg));

  xQueueSend(smsQueue, msg, 0);
}

// =============== GSM TASK ===============
void gsmTask(void *parameter) {

  char msg[180];

  SIM800.begin(9600, SERIAL_8N1, 16, 17);

  vTaskDelay(pdMS_TO_TICKS(3000));

  SIM800.println("AT");
  vTaskDelay(pdMS_TO_TICKS(500));

  SIM800.println("ATE0");
  vTaskDelay(pdMS_TO_TICKS(300));

  SIM800.println("AT+CMGF=1");
  vTaskDelay(pdMS_TO_TICKS(500));

  while (true) {

    if (xQueueReceive(smsQueue, &msg, portMAX_DELAY) == pdTRUE) {

      SIM800.print("AT+CMGS=\"+216");
      SIM800.print(numero);
      SIM800.println("\"");

      // Wait for the SMS prompt
      unsigned long start = millis();
      bool prompt = false;

      while (millis() - start < 5000) {
        if (SIM800.available()) {
          if (SIM800.read() == '>') {
            prompt = true;
            break;
          }
        }
        vTaskDelay(pdMS_TO_TICKS(10));
      }

      if (prompt) {
        SIM800.print(msg);
        SIM800.write(26);

        start = millis();
        bool sent = false;

        while (millis() - start < 30000) {
          if (SIM800.available()) {
            String response = SIM800.readStringUntil('\n');
            if (response.indexOf("+CMGS:") >= 0) {
              sent = true;
            }
            if (response.indexOf("ERROR") >= 0) break;
          }

          if (sent) break;
          vTaskDelay(pdMS_TO_TICKS(20));
        }

        Serial.println(sent ? "SMS accepte par reseau"
                            : "Echec confirmation SMS");
      } else {
        Serial.println("SIM800L: pas de prompt SMS");
      }
    }
  }
}

// =============== VALIDATION ===============
void validatePatient() {

  if (numero.length() != 8) {
    lcd.clear();
    lcd.print("8 chiffres requis");
    delay(1200);
    lcd.clear();
    lcd.print("Votre numero:");
    lcd.setCursor(0, 1);
    lcd.print(numero);
    return;
  }

  patientValidated = true;

  lcd.clear();
  lcd.print("Numero valide");

  queueSMS(
    "Inscription validee au systeme de surveillance cardiaque. Numero patient: "
    + numero
  );

  delay(1500);

  lcd.clear();
  lcd.print("Surveillance ON");
  lcd.setCursor(0, 1);
  lcd.print(numero);
}

// =============== COMMANDES UNO ===============
void readUNOCommands() {

  static String command = "";

  while (UNO.available()) {

    char c = UNO.read();

    if (c == '\n') {

      command.trim();

      if (command == "MUTE") {
        buzzerMuted = !buzzerMuted;
      }

      command = "";

    } else if (c != '\r' && command.length() < 20) {
      command += c;
    }
  }
}

// =============== SETUP ===============
void setup() {

  Serial.begin(115200);

  // RX=33, TX=4
  UNO.begin(19200, SERIAL_8N1, 33, 4);

  Wire.begin(21, 22);

  lcd.init();
  lcd.backlight();

  pinMode(LO_PLUS, INPUT);
  pinMode(LO_MINUS, INPUT);
  pinMode(BUZZER, OUTPUT);

  digitalWrite(BUZZER, LOW);

  analogReadResolution(12);

  smsQueue = xQueueCreate(5, 180);

  lcd.setCursor(0, 0);
  lcd.print("Votre numero:");

  xTaskCreatePinnedToCore(
    gsmTask, "GSM", 4096,
    NULL, 1, NULL, 0
  );

  lastSampleUs = micros();
}

// =============== LOOP ===============
void loop() {

  // Keypad
  char key = keypad.getKey();

  if (key) {

    if (key >= '0' && key <= '9') {

      if (!patientValidated && numero.length() < 8) {
        numero += key;
      }

    } else if (key == 'A') {

      if (!patientValidated) validatePatient();

    } else if (key == 'B') {

      if (!patientValidated && numero.length() > 0) {
        numero.remove(numero.length() - 1);
      }

    } else if (key == 'C') {

      numero = "";
      patientValidated = false;

      lcd.clear();
      lcd.print("Votre numero:");
    }

    lcd.setCursor(0, 1);
    lcd.print("                ");
    lcd.setCursor(0, 1);
    lcd.print(numero);
  }

  readUNOCommands();

  // ECG sampling at 200 Hz
  unsigned long nowUs = micros();

  if (nowUs - lastSampleUs < SAMPLE_US) return;

  lastSampleUs += SAMPLE_US;

  unsigned long nowMs = millis();

  bool leadsOff =
    digitalRead(LO_PLUS) ||
    digitalRead(LO_MINUS);

  int raw = leadsOff ? 0 : analogRead(ECG_PIN);

  bool beat = false;

  if (!leadsOff) {

    baseline += (raw - baseline) * 0.002;

    float amp = raw - baseline;
    if (amp < 0) amp = 0;

    float threshold = envelope * THRESH_FRAC;

    if (!aboveThreshold &&
        amp > threshold &&
        (lastBeatMs == 0 ||
         nowMs - lastBeatMs > REFRACTORY_MS)) {

      aboveThreshold = true;
      beat = true;

      unsigned long rr = nowMs - lastBeatMs;

      if (lastBeatMs != 0 && rr >= 300 && rr <= 3000) {
        bpm = 60000UL / rr;
      }

      lastBeatMs = nowMs;

      envelope += (amp - envelope) * 0.25;

      beepUntil = nowMs + BEEP_MS;
    }

    if (amp < threshold * 0.5) {
      aboveThreshold = false;
    }

  } else {

    bpm = 0;
    aboveThreshold = false;
    lastBeatMs = 0;
    dangerCounter = 0;
  }

  // Buzzer
  digitalWrite(
    BUZZER,
    (!leadsOff && !buzzerMuted && nowMs < beepUntil)
      ? HIGH : LOW
  );

  // Danger detection: require repeated abnormal readings
  bool danger =
    !leadsOff && bpm > 0 &&
    (bpm < BPM_BRADY || bpm > BPM_TACHY);

  if (danger) {
    if (dangerCounter < 5) dangerCounter++;
  } else {
    dangerCounter = 0;
    alarmActive = false;
  }

  if (dangerCounter >= 5 &&
      !alarmActive &&
      (lastAlertMs == 0 ||
       nowMs - lastAlertMs >= 300000UL)) {

    if (patientValidated) {
      queueSMS(
        "ALERTE cardiaque: BPM mesure = " +
        String(bpm) +
        ". Frequence hors des limites configurees. Verifier le patient."
      );
    }
    alarmActive = true;
    lastAlertMs = nowMs;
  }
  // Send ECG data to UNO: 100 samples/s
  sampleCounter++;
  if (sampleCounter >= 2) {
    sampleCounter = 0;
    UNO.print(raw);
    UNO.print(',');
    UNO.print(leadsOff ? 1 : 0);
    UNO.print(',');
    UNO.print(beat ? 1 : 0);
    UNO.print(',');
    UNO.println(bpm);
    //---------------------------ardiuno----------
    
#include <MCUFRIEND_kbv.h>
#include <Adafruit_GFX.h>
#include <TouchScreen.h>
#include <SoftwareSerial.h>

// ============ COMMUNICATION ============
// D10 = RX, D11 = TX
SoftwareSerial ESP32(10, 11);

MCUFRIEND_kbv tft;

// ============ TOUCHSCREEN ============
// Brochage courant de certains shields UNO
#define XP 8
#define XM A2
#define YP A3
#define YM 9

TouchScreen ts = TouchScreen(XP, YP, XM, YM, 300);

// Calibration provisoire : a adapter
#define TS_MINX 150
#define TS_MAXX 900
#define TS_MINY 120
#define TS_MAXY 920

// ============ COULEURS ============
#define BLACK  0x0000
#define WHITE  0xFFFF
#define GREEN  0x07E0
#define YELLOW 0xFFE0
#define RED    0xF800
#define GRID   0x2104
#define GREY   0x7BEF

// ============ ECG ZONE ============
const int X0 = 5;
const int X1 = 205;
const int Y0 = 35;
const int Y1 = 115;

int xPos = X0;
int previousY = 75;

int ecg = 0;
int bpm = 0;
int leadsOff = 0;
int beat = 0;

bool muted = false;

char buffer[40];
byte bufferIndex = 0;

unsigned long lastTouch = 0;
unsigned long lastData = 0;

// ============ GRID ============
void drawGrid() {

  for (int x = X0; x <= X1; x += 20) {
    tft.drawFastVLine(x, Y0, Y1-Y0, GRID);
  }

  for (int y = Y0; y <= Y1; y += 16) {
    tft.drawFastHLine(X0, y, X1-X0, GRID);
  }
}

// ============ INTERFACE ============
void drawInterface() {

  tft.fillScreen(BLACK);

  // Titre
  tft.setTextColor(GREEN);
  tft.setTextSize(2);
  tft.setCursor(8, 8);
  tft.print("ECG");

  // Zone ECG
  drawGrid();

  // BPM
  tft.setTextColor(GREEN);
  tft.setTextSize(2);
  tft.setCursor(220, 12);
  tft.print("BPM");

  tft.setTextSize(4);
  tft.setCursor(220, 42);
  tft.print("--");

  // SPO2 indisponible
  tft.setTextColor(YELLOW);
  tft.setTextSize(2);
  tft.setCursor(8, 128);
  tft.print("SPO2");

  tft.setTextColor(WHITE);
  tft.setCursor(220, 128);
  tft.print("-- %");

  // RESP indisponible
  tft.setTextColor(RED);
  tft.setCursor(8, 158);
  tft.print("RESP");

  tft.setTextColor(WHITE);
  tft.setCursor(220, 158);
  tft.print("-- RPM");

  // Connexion
  tft.setTextSize(1);
  tft.setCursor(8, 190);
  tft.setTextColor(WHITE);
  tft.print("AD8232:");

  tft.setCursor(75, 190);
  tft.setTextColor(GREEN);
  tft.print("WAITING");

  // Boutons
  tft.drawRect(5, 207, 75, 30, WHITE);
  tft.drawRect(85, 207, 75, 30, WHITE);
  tft.drawRect(165, 207, 75, 30, WHITE);
  tft.drawRect(245, 207, 70, 30, WHITE);

  tft.setTextSize(1);
  tft.setTextColor(WHITE);

  tft.setCursor(20, 218);
  tft.print("MENU");

  tft.setCursor(94, 218);
  tft.print("SILENCE");

  tft.setCursor(180, 218);
  tft.print("ALARM");

  tft.setCursor(260, 218);
  tft.print("TOOLS");
}

// ============ BPM DISPLAY ============
void updateBPM() {

  tft.fillRect(215, 40, 105, 65, BLACK);

  tft.setCursor(220, 48);
  tft.setTextSize(4);

  if (bpm > 0) {

    if (bpm < 50 || bpm > 120) {
      tft.setTextColor(RED);
    } else {
      tft.setTextColor(GREEN);
    }

    tft.print(bpm);

  } else {

    tft.setTextColor(WHITE);
    tft.print("--");
  }
}

// ============ LEADS STATUS ============
void updateLeads() {

  tft.fillRect(70, 185, 135, 18, BLACK);

  tft.setTextSize(1);
  tft.setCursor(75, 190);

  if (leadsOff) {
    tft.setTextColor(RED);
    tft.print("LEADS OFF");
  } else {
    tft.setTextColor(GREEN);
    tft.print("CONNECTED");
  }
}

// ============ ECG CURVE ============
void drawECG(int value) {

  // Echelle d'affichage a ajuster selon le signal
  int y = map(value, 1500, 2600, Y1, Y0);

  y = constrain(y, Y0, Y1);

  // Effacer l'ancienne colonne
  tft.drawFastVLine(
    xPos, Y0, Y1-Y0+1, BLACK
  );

  // Restaurer les lignes de grille
  if ((xPos-X0) % 20 == 0) {
    tft.drawFastVLine(xPos, Y0, Y1-Y0+1, GRID);
  }

  if ((xPos-X0) % 20 == 0) {
    // Les lignes horizontales seront restaurees
    // lors du passage de cette colonne.
  }

  if (xPos > X0) {
    tft.drawLine(
      xPos-1, previousY,
      xPos, y,
      GREEN
    );
  }

  previousY = y;
  xPos++;

  // Recommencer le balayage
  if (xPos >= X1) {
    xPos = X0;
    previousY = 75;
    drawGrid();
  }
}

// ============ SERIAL RECEIVE ============
void receiveESP32() {

  while (ESP32.available()) {

    char c = ESP32.read();

    if (c == '\n') {

      buffer[bufferIndex] = '\0';

      int values[4];

      if (sscanf(
        buffer, "%d,%d,%d,%d",
        &values[0],
        &values[1],
        &values[2],
        &values[3]
      ) == 4) {

        ecg = values[0];
        leadsOff = values[1];
        beat = values[2];
        bpm = values[3];

        lastData = millis();

        if (!leadsOff) {
          drawECG(ecg);
        }

        static int oldBPM = -1;
        static int oldLeads = -1;

        if (bpm != oldBPM) {
          updateBPM();
          oldBPM = bpm;
        }

        if (leadsOff != oldLeads) {
          updateLeads();
          oldLeads = leadsOff;
        }
      }

      bufferIndex = 0;

    } else if (c != '\r') {

      if (bufferIndex < sizeof(buffer)-1) {
        buffer[bufferIndex++] = c;
      } else {
        bufferIndex = 0;
      }
    }
  }
}

// ============ TOUCH ============
void readTouch() {

  TSPoint p = ts.getPoint();

  // Restaurer les broches tactiles
  pinMode(XM, OUTPUT);
  pinMode(YP, OUTPUT);

  if (p.z < 100 || p.z > 1000) return;

  int x = map(p.x, TS_MINX, TS_MAXX, 0, 320);
  int y = map(p.y, TS_MINY, TS_MAXY, 0, 240);

  x = constrain(x, 0, 319);
  y = constrain(y, 0, 239);

  // Bouton SILENCE
  if (y >= 207 && x >= 85 && x <= 160) {

    muted = !muted;

    ESP32.println("MUTE");

    tft.fillRect(86, 208, 73, 28, BLACK);
    tft.drawRect(85, 207, 75, 30, WHITE);

    tft.setTextSize(1);
    tft.setTextColor(muted ? RED : GREEN);
    tft.setCursor(94, 218);
    tft.print(muted ? "MUTED" : "SILENCE");
  }
}

// ============ SETUP ============
void setup() {

  Serial.begin(115200);
  ESP32.begin(19200);

  uint16_t ID = tft.readID();

  Serial.print("TFT ID = 0x");
  Serial.println(ID, HEX);

  // Meme initialisation que le test TFT
  if (ID == 0xD3D3 || ID == 0x0000) {
    ID = 0x9341;
  }

  tft.begin(ID);
  tft.setRotation(1);

  drawInterface();
}

// ============ LOOP ============
void loop() {

  receiveESP32();

  // Indiquer la perte de communication
  if (lastData != 0 && millis() - lastData > 2000) {
    tft.fillRect(70, 185, 135, 18, BLACK);
    tft.setTextSize(1);
    tft.setTextColor(RED);
    tft.setCursor(75, 190);
    tft.print("NO DATA");
  }

  if (millis() - lastTouch > 200) {
    readTouch();
    lastTouch = millis();
  }
}

  }
}
