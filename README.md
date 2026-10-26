#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "DFRobot_MPX5700.h"

// --- Deklarasi Layar OLED (128x64 diset Vertikal) ---
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// --- Deklarasi Sensor DFRobot MPX5700AP (I2C) ---
#define I2C_ADDRESS 0x16 
DFRobot_MPX5700 mpx5700(&Wire, I2C_ADDRESS);

// --- Definisi Pin Tombol (Aktif LOW dengan Internal Pull-Up) ---
#define BTN_MODE    19  // Tombol 1: Pindah Mode (Bekam/Spray)
#define BTN_UP      18  // Tombol 2: Speed Naik
#define BTN_DOWN    5   // Tombol 3: Speed Turun
#define BTN_TRIG    17  // Tombol 4: Pelatuk (Trigger Motor)

// --- Definisi Pin Output PWM ke MOSFET D4184 ---
#define MOTOR_VACUUM 25 
#define MOTOR_SPRAY  26 

// --- Variabel Sistem & State ---
bool modeBekam = true; 
int speedBekam = 10;   // Mulai dari 10% di layar
int speedSpray = 10;  
bool btnModeWasPressed = false;
bool btnUpWasPressed = false;
bool btnDownWasPressed = false;

// --- Variabel Keamanan & Kalibrasi Sensor ---
#define BATAS_AMAN_MMHG -700.0  
float tekananAwal_mmHg = 0.0;   
float tekananGaugeSaatIni = 0.0;

// --- Properti PWM ESP32 (API v3.x) ---
const int freq = 5000;
const int res = 8; 

void setup() {
  Serial.begin(115200);
  
  pinMode(BTN_MODE, INPUT_PULLUP);
  pinMode(BTN_UP, INPUT_PULLUP);
  pinMode(BTN_DOWN, INPUT_PULLUP);
  pinMode(BTN_TRIG, INPUT_PULLUP);

  ledcAttach(MOTOR_VACUUM, freq, res);
  ledcAttach(MOTOR_SPRAY, freq, res);
  ledcWrite(MOTOR_VACUUM, 0); 
  ledcWrite(MOTOR_SPRAY, 0);

  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("OLED Gagal!");
    for(;;);
  }
  
  display.setRotation(3); 
  display.clearDisplay();
  
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(2, 40);
  display.print("ALAT BEKAM");
  display.setCursor(2, 55);
  display.print("Kalibrasi...");
  display.display();

  if (mpx5700.begin() == false) {
    display.clearDisplay();
    display.setCursor(2, 50);
    display.print("Sensor Error!");
    display.display();
    while(1); 
  }

  // Proses Auto-Tare (Mengambil referensi titik 0 saat alat dihidupkan)
  float total_kPa = 0;
  for(int i = 0; i < 10; i++) {
    total_kPa += mpx5700.getPressureValue_kpa(0); 
    delay(100);
  }
  tekananAwal_mmHg = (total_kPa / 10.0) * 7.50062; 
}

void loop() {
  bacaTombolMode();
  bacaTombolSpeed();
  bacaSensorMpx();
  eksekusiPelatuk();
  tampilkanLayarAktif();
}

void bacaTombolMode() {
  bool btnState = digitalRead(BTN_MODE) == LOW;

  if (btnState && !btnModeWasPressed) {
    btnModeWasPressed = true;
    modeBekam = !modeBekam; 
    delay(200); 
  } 
  else if (!btnState && btnModeWasPressed) {
    btnModeWasPressed = false;
  }
}

// Tombol naik/turun kelipatan 10% di layar (rentang 10% hingga 100%)
void bacaTombolSpeed() {
  bool btnUpState = digitalRead(BTN_UP) == LOW;
  if (btnUpState && !btnUpWasPressed) {
    btnUpWasPressed = true;
    if (modeBekam) {
      speedBekam += 10;
      if (speedBekam > 100) speedBekam = 100;
    } else {
      speedSpray += 10;
      if (speedSpray > 100) speedSpray = 100;
    }
  } else if (!btnUpState && btnUpWasPressed) {
    btnUpWasPressed = false;
  }
  
  bool btnDownState = digitalRead(BTN_DOWN) == LOW;
  if (btnDownState && !btnDownWasPressed) {
    btnDownWasPressed = true;
    if (modeBekam) {
      speedBekam -= 10;
      if (speedBekam < 10) speedBekam = 10; 
    } else {
      speedSpray -= 10;
      if (speedSpray < 10) speedSpray = 10;
    }
  } else if (!btnDownState && btnDownWasPressed) {
    btnDownWasPressed = false;
  }
}

// Fungsi Pemetaan Agresif: Layar 10% langsung menarik PWM tinggi (205)
int getMappedPWM(int layarPersen) {
  int targetPWM = 0;

  if (layarPersen <= 30) {
    targetPWM = map(layarPersen, 10, 30, 205, 225);
  } 
  else if (layarPersen <= 60) {
    targetPWM = map(layarPersen, 40, 60, 226, 240);
  } 
  else {
    targetPWM = map(layarPersen, 70, 100, 241, 255);
  }

  return targetPWM;
}

void bacaSensorMpx() {
  float current_kPa = mpx5700.getPressureValue_kpa(0);
  float current_mmHg = current_kPa * 7.50062;
  tekananGaugeSaatIni = current_mmHg - tekananAwal_mmHg;
  
  // FILTER NOISE (Toleransi 5.0 mmHg)
  // Menjaga agar angka di layar diam di 0 saat standby
  if (abs(tekananGaugeSaatIni) < 5.0) {
    tekananGaugeSaatIni = 0.0;
  }
}

void eksekusiPelatuk() {
  if (digitalRead(BTN_TRIG) == LOW) { 
    if (modeBekam) {
      if (tekananGaugeSaatIni <= BATAS_AMAN_MMHG) {
        ledcWrite(MOTOR_VACUUM, 0); // Putus arus saat mencapai Cut-Off
      } else {
        int pwmAktif = getMappedPWM(speedBekam);
        ledcWrite(MOTOR_VACUUM, pwmAktif); 
      }
      ledcWrite(MOTOR_SPRAY, 0); 
    } else {
      int pwmAktif = getMappedPWM(speedSpray);
      ledcWrite(MOTOR_SPRAY, pwmAktif); 
      ledcWrite(MOTOR_VACUUM, 0); 
    }
  } else {
    ledcWrite(MOTOR_VACUUM, 0);
    ledcWrite(MOTOR_SPRAY, 0);
  }
}

void tampilkanLayarAktif() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  display.setCursor(0, 2);
  if (digitalRead(BTN_TRIG) == LOW) {
    if (modeBekam && tekananGaugeSaatIni <= BATAS_AMAN_MMHG) {
      display.print("CUT-OFF!"); 
    } else {
      display.print("RUNNING");
    }
  } else {
    display.print("STANDBY");
  }
  
  display.drawLine(0, 12, 64, 12, SSD1306_WHITE); 

  if (modeBekam) {
    display.setCursor(0, 20);
    display.print("M: BEKAM");
    
    display.setCursor(0, 38);
    display.print("Spd:" + String(speedBekam) + "%");
    
    display.setCursor(0, 56);
    display.print("Tekanan:");

    display.setCursor(0, 70); 
    display.setTextSize(2);   
    display.print(tekananGaugeSaatIni, 0); 
    
    display.setTextSize(1);
    display.setCursor(0, 96);
    display.print("mmHg");
    
  } else {
    display.setCursor(0, 20);
    display.print("M: SPRAY");
    
    display.setCursor(0, 38);
    display.print("Spd:" + String(speedSpray) + "%");
    
    display.setCursor(0, 62);
    display.setTextSize(1);
    display.print("Disinfektan");
    display.setCursor(0, 80);
    display.print("Siap Pakai");
  }

  display.display();
}
