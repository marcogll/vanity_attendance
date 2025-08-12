## ⏱️ Control de Asistencia — ESP32 + RC522 + OLED + RTC DS1302
Dispositivo de registro de asistencia con:

* **ESP32** DevKit v1
* **OLED 128×64** (SH1106, I²C)
* **RFID RC522** (SPI)
* **RTC DS1302** con batería CR2032
* **Buzzer + LEDs**

Envío de JSON a Webhook HTTPS

Optimizado para respuesta rápida, UI minimalista, lectura NDEF (name/curp/branch) con fallback MIFARE Classic y cache local, UUID alfanumérico de 10 para evitar duplicados, y time sync robusto (RTC+NTP+build time).

***

## 📦 Dependencias (Arduino IDE)
Instala desde Library Manager:

* `U8g2` (OLED SH1106/SSD1306)
* `MFRC522`
* `NDEF` (NDEF_MFRC522)
* `ArduinoJson`
* `WiFi`, `WiFiClientSecure`, `HTTPClient`
* `Ds1302` (la que usa constructor `Ds1302(CLK, IO, CE)`)

***

## 🔌 Pinout (para soldar / KiCad)

### ESP32 ↔ OLED SH1106 (I²C)
| Señal | OLED      | ESP32 |
| :---- | :-------- | :---- |
| SDA   | SDA       | 21    |
| SCL   | SCK/SCL   | 22    |
| VCC   | 3.3V      | 3.3V  |
| GND   | GND       | GND   |

> **Tip**: si tu OLED trae `RST` suelto, amárralo a 3.3 V o asigna un GPIO y resetea al inicio.

### ESP32 ↔ RC522 (SPI)
| Señal    | RC522     | ESP32 |
| :------- | :-------- | :---- |
| SS (SDA) | SDA/SS    | 5     |
| SCK      | SCK       | 18    |
| MOSI     | MOSI      | 23    |
| MISO     | MISO      | 19    |
| RST      | RST       | 17    |
| VCC      | 3.3V      | 3.3V  |
| GND      | GND       | GND   |
| IRQ      | NC        | —     |

### ESP32 ↔ RTC DS1302
| Señal | DS1302    | ESP32 |
| :---- | :-------- | :---- |
| CE    | RST/CE    | 25    |
| IO    | DAT/IO    | 26    |
| CLK   | CLK       | 27    |
| VCC   | 3.3V      | 3.3V  |
| GND   | GND       | GND   |
| VBAT  | CR2032    | —     |

> **OJO**: Esta librería usa orden (`CLK`, `IO`, `CE`) en el constructor.

### LEDs, Buzzer, Botón
| Dispositivo | ESP32 | Notas                                    |
| :---------- | :---- | :--------------------------------------- |
| LED WiFi    | 13    | 220 Ω a GND                              |
| LED Estado  | 14    | 220 Ω a GND                              |
| Buzzer      | 4     | `tone()`; directo a GND                  |
| Botón Reset | 32    | a GND, `INPUT_PULLUP`, reset largo (3 s) |

***

## 🖥️ UI (OLED, 128×64, blanco sobre negro)

### Fuentes U8g2:
* **Tiny**: `u8g2_font_6x12_tr`
* **Mediana**: `u8g2_font_helvB12_tf` (o `u8g2_font_logisoso16_tf`)
* **Grande**: `u8g2_font_logisoso18_tf`

### Pantallas
* **Welcome (0.7 s)**
    * “BIENVENIDA A:” (Tiny)
    * “VANITY” / “LOS PINOS” (Med)
* **Conectando Wi‑Fi (hasta 20 s máx)**
    * “CONECTANDO…” (Tiny) + SSID (Med) + puntos animados
* **Wi‑Fi OK (0.7 s)**
    * ✓ SSID (Med)
    * “LISTO” (Tiny)
    * LED verde parpadea 3× + beep corto
* **Home / Reloj (refresco 1 s)**
    * “VANITY” (Med)
    * DOM | 8 DE AGO (Tiny; día/mes MAYÚS 3 letras)
    * HH:MM (Grande) + ss (Tiny)
* **Leyendo tarjeta (momento)**
    * “LEYENDO…” (Med) + animación breve
* **Registro OK (2.5 s)**
    * “HOLA” (Tiny)
    * {NAME} (Med)
    * “REGISTRO” (Tiny) + HH:MM grande + ss tiny
    * Buzzer OK + LED estado 3× rápido
* **Registro Falló (2.5 s)**
    * ✗ FALLO REGISTRO (Med)
    * :( (Med)
    * INTENTA DE NUEVO (Tiny, no se corta)
    * Buzzer 3× + LED estado 3× lento
* **Sin Wi‑Fi (0.6 s)**
    * “SIN WIFI” (Med) + SSID (Tiny)

El Home es el estado idle.

***

## 🕒 Tiempo (RTC + NTP + fallback)
* Se intenta usar RTC DS1302; si arranca inválido, se usa hora del sistema (NTP o build).
* Se configura NTP una sola vez al conectar Wi‑Fi.
* Se ajusta el RTC solo si el desfase es > 10 s.
* **Fallback**: si no hay NTP, usa hora de compilación (build).
* **Comando Serial** para fijar manualmente el RTC:
    ```
    SET YYYY-MM-DD HH:MM:SS
    ```

***

## 🪪 Lectura RFID y datos
* Si la tarjeta tiene NDEF con JSON `{name, curp, branch}`, se usa.
* Si no, **fallback MIFARE Classic**: busca `{...}` JSON simple en sectores iniciales.
* Si no hay datos, envia modo genérico:
    `name = "CARD_" + UIDHEX`, `curp = UIDHEX`, `branch = "pinos"`
* **Cache local (RAM)** por UID: si una vez ya se obtuvo `{name, curp, branch}`, se reutiliza aunque la siguiente lectura falle (mejor UX).

***

## 🌐 Webhook
* **Método**: HTTP `POST` HTTPS (con `WiFiClientSecure` sin cert)
* **User-Agent**: `clock_pinos_1`
* **JSON enviado**:
    ```json
    {
      "uuid": "AbC123xYz9",
      "name": "Marco",
      "curp": "GAMM9105...",
      "branch": "pinos",
      "date": "2025-08-08",
      "time": "13:45:00"
    }
    ```
* **uuid**: 10 caracteres alfanuméricos (`[0-9A-Za-z]`).

***

## 🐞 Logs en Serial (relevantes)
* Boot, estado del RTC, conexión Wi‑Fi/NTP.
* Al leer tarjeta: tipo/UID, NDEF encontrado o fallback.
* Webhook: URL, Body, código y respuesta.
* Cache: HIT/PUT por UID.
* Los ticks del reloj NO se imprimen cada segundo (solo UI).

***

## 🧠 Tips de hardware (I²C)
* Si OLED parpadea o da NACK:
    * Pull‑ups 4.7 kΩ a 3.3 V en `SDA`/`SCL` (si tu módulo no los trae).
    * Capacitores 100 nF + 10 µF en `VCC`‑`GND` del OLED.
    * Cableado corto, `SDA=21`, `SCL=22` (no cruzar).
* Si OLED tiene `RST` expuesto, amárralo a 3.3 V o usa GPIO con pulso.

***

## 🧩 Sketch final (comentado)
Ajusta `SSID`, `PASSWORD`, `WEBHOOK_URL` a tu entorno.

```cpp
/***************
 * Proyecto: Control de Asistencia (ESP32 + RC522 + OLED + DS1302)
 * UI minimalista, lectura NDEF + fallback Classic, cache de tarjetas,
 * Webhook HTTPS con UUID alfanumérico (10), manejo robusto de hora (RTC+NTP).
 *
 * Pinout en README. Este código asume:
 * - OLED SH1106 I2C en SDA=21, SCL=22 (3.3V)
 * - RC522 SPI (SS=5, RST=17, SCK=18, MOSI=23, MISO=19)
 * - DS1302 (CE=25, IO=26, CLK=27) + CR2032
 * - LED WiFi=13, LED Estado=14, Buzzer=4, Botón Reset=32 (a GND)
 ***************/
#include <Wire.h>
#include <U8g2lib.h>
#include <SPI.h>
#include <MFRC522.h>
#include <NfcAdapter.h>          // librería NDEF para MFRC522
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <time.h>
#include <sys/time.h>
#include <Ds1302.h>
#include <ArduinoJson.h>

// ===== CONFIGURACIÓN =====
#define DEBUG 1
#define DBG(...) do{ if(DEBUG){ Serial.printf(__VA_ARGS__); } }while(0)

// Wi-Fi + Webhook
const char* SSID        = "TU_SSID";
const char* PASSWORD    = "TU_PASS";
const char* WEBHOOK_URL = "[https://tu-servidor.com/webhook/ID](https://tu-servidor.com/webhook/ID)";

// Zona horaria (UTC-6 sin DST; ajusta a tu región)
const long GMT_OFFSET = -6 * 3600;
const int  DST_OFFSET = 0;

// ===== PINOUT =====
#define OLED_SDA   21
#define OLED_SCL   22
#define RC522_SS    5
#define RC522_RST  17
#define RTC_RST    25       // CE
#define RTC_DAT    26       // IO
#define RTC_CLK    27       // CLK
#define LED_WIFI   13
#define LED_STAT   14
#define BUZZER_PIN  4
#define BTN_RESET  32

// ===== OBJETOS =====
U8G2_SH1106_128X64_NONAME_F_HW_I2C u8g2(
  U8G2_R0, U8X8_PIN_NONE, OLED_SCL, OLED_SDA
);
MFRC522   mfrc522(RC522_SS, RC522_RST);
NfcAdapter nfc(&mfrc522);
// IMPORTANTE: esta lib usa orden (CLK, IO, CE)
Ds1302    rtc(RTC_CLK, RTC_DAT, RTC_RST);

// ===== TEXTOS =====
const char* WEEKDAYS[7]  = {"DOM","LUN","MAR","MIE","JUE","VIE","SAB"};
const char* MONTHS_3[12] = {"ENE","FEB","MAR","ABR","MAY","JUN","JUL","AGO","SEP","OCT","NOV","DIC"};

// ===== ESTADO/UI =====
enum UIState { ST_HOME, ST_READING, ST_FEEDBACK_OK, ST_FEEDBACK_FAIL, ST_WIFI_OK };
UIState ui = ST_HOME;
unsigned long stateT0=0, lastClock=0, cooldownUntil=0, wifiOkUntil=0;
bool redraw=true;

// Timings de UI
const unsigned long WIFI_OK_MS   = 700;     // Wi-Fi OK: 0.7 s
const unsigned long NO_WIFI_MS   = 600;     // SIN WIFI: 0.6 s
const unsigned long FEEDBACK_MS  = 2500;    // OK/FAIL: 2.5 s
const unsigned long READ_COOLDOWN_MS = 900; // anti-doble

// NTP control
bool ntpConfigured=false, ntpPending=false;
unsigned long ntpDeadline=0, ntpLastTry=0;
const unsigned long NTP_MAX_MS  = 20000;

// Anti-doble UID
String lastCardUid=""; unsigned long lastCardAt=0;

// Botón reset largo (3s)
bool btnLast=true; unsigned long btnT0=0; const unsigned long BTN_RESET_MS=3000;

// ===== PROTOTIPOS =====
void uiWelcome(), uiConnecting(const char* ssid), uiWifiOK(const char* ssid), uiNoWifi(const char* ssid);
void uiHome(), uiRegOK(const String& name,uint8_t hh,uint8_t mm,uint8_t ss), uiRegFail();
void setState(UIState s);

bool    rtcIsValid(const Ds1302::DateTime& d);
bool    setRtcFromString(const char* s);
bool    setSystemTimeFromBuild();
bool    fillNow(Ds1302::DateTime& out);
bool    syncRtcIfSkewGT(uint32_t thr);
String dateISO_fromAny(), timeISO_fromAny();

String ndefRecordToString(NdefRecord& rec);
bool    readClassicJsonFallback(String& name, String& curp, String& branch);

bool    readAndSendOnce(String &nameOut, uint8_t &hhOut, uint8_t &mmOut, uint8_t &ssOut);
bool    sendWebhook(const String& uuid,const String& name,const String& curp,const String& branch,
                      const String& dateStr,const String& timeStr,String& errOut);
void    startNtpOnce();

// ===== UTILS =====
static const char* wifiStatusStr(wl_status_t s){
  switch(s){
    case WL_NO_SHIELD: return "NO_SHIELD";
    case WL_IDLE_STATUS: return "IDLE";
    case WL_NO_SSID_AVAIL: return "NO_SSID";
    case WL_SCAN_COMPLETED: return "SCAN_DONE";
    case WL_CONNECTED: return "CONNECTED";
    case WL_CONNECT_FAILED: return "CONNECT_FAILED";
    case WL_CONNECTION_LOST: return "CONNECTION_LOST";
    case WL_DISCONNECTED: return "DISCONNECTED";
    default: return "UNKNOWN";
  }
}
void beepOK(){ tone(BUZZER_PIN, 2200, 160); delay(170); noTone(BUZZER_PIN); }
void beepErr(){ for(int i=0;i<3;i++){ tone(BUZZER_PIN,1100,120); delay(140); noTone(BUZZER_PIN); delay(80);} }
void ledWifiBlink3(){ for(int i=0;i<3;i++){ digitalWrite(LED_WIFI,LOW); delay(70); digitalWrite(LED_WIFI,HIGH); delay(70);} }
void ledStatBlink(int n,int onMs,int offMs){ for(int i=0;i<n;i++){ digitalWrite(LED_STAT,HIGH); delay(onMs); digitalWrite(LED_STAT,LOW); delay(offMs);} }

// UUID alfanumérico (10 chars) [0-9A-Za-z]
String uuid10(){
  static const char s[]="0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
  char b[11]; for(int i=0;i<10;i++){ uint32_t r=esp_random(); b[i]=s[r%62]; } b[10]=0; return String(b);
}

// ===== TIEMPO / RTC =====
bool rtcIsValid(const Ds1302::DateTime& d){
  return (d.year<=99)&&(d.month>=1&&d.month<=12)&&(d.day>=1&&d.day<=31)&&
         (d.hour<=23)&&(d.minute<=59)&&(d.second<=59);
}
static int64_t epochFromRtc(const Ds1302::DateTime& dt){
  struct tm t={}; t.tm_year=(dt.year+2000)-1900; t.tm_mon=dt.month-1; t.tm_mday=dt.day;
  t.tm_hour=dt.hour; t.tm_min=dt.minute; t.tm_sec=dt.second;
  time_t e=mktime(&t); return (e==-1)?0:(int64_t)e;
}
static int64_t epochFromTm(const struct tm& ti){ struct tm t=ti; time_t e=mktime(&t); return (e==-1)?0:(int64_t)e; }

// Llena "out" con RTC si es válido; si no, con hora de sistema (NTP/build); si no, 1970-01-01 00:00:00.
bool fillNow(Ds1302::DateTime& out){
  Ds1302::DateTime r; rtc.getDateTime(&r);
  if(rtcIsValid(r)){ out=r; return true; }
  struct tm ti; if(getLocalTime(&ti,50)){
    out.year=(uint8_t)((ti.tm_year+1900)-2000); out.month=ti.tm_mon+1; out.day=ti.tm_mday;
    out.hour=ti.tm_hour; out.minute=ti.tm_min; out.second=ti.tm_sec; out.dow=ti.tm_wday?ti.tm_wday:7;
    return true;
  }
  memset(&out,0,sizeof(out)); out.month=1; out.day=1; out.dow=1; return false;
}
String dateISO_fromAny(){ Ds1302::DateTime d; fillNow(d); char b[11]; sprintf(b,"%04u-%02u-%02u",d.year+2000,d.month,d.day); return b; }
String timeISO_fromAny(){ Ds1302::DateTime d; fillNow(d); char b[9];  sprintf(b,"%02u:%02u:%02u",d.hour,d.minute,d.second); return b; }

// Ajusta RTC desde NTP si desfase > thr (seg)
bool syncRtcIfSkewGT(uint32_t thr){
  struct tm ti; if(!getLocalTime(&ti,2000)) { DBG("[NTP] getLocalTime timeout\n"); return false; }
  Ds1302::DateTime r; rtc.getDateTime(&r);

  auto do_set = [&](const struct tm& ti_){
    Ds1302::DateTime dt;
    dt.year=(uint8_t)((ti_.tm_year+1900)-2000); dt.month=ti_.tm_mon+1; dt.day=ti_.tm_mday;
    dt.dow=ti_.tm_wday?ti_.tm_wday:7; dt.hour=ti_.tm_hour; dt.minute=ti_.tm_min; dt.second=ti_.tm_sec;
    rtc.setDateTime(&dt);
    Ds1302::DateTime chk; rtc.getDateTime(&chk);
    DBG("[NTP] post-set read: %04u-%02u-%02u %02u:%02u:%02u (valid=%d)\n",
        chk.year+2000, chk.month, chk.day, chk.hour, chk.minute, chk.second, rtcIsValid(chk));
  };

  if(!rtcIsValid(r)){
    DBG("[NTP] RTC inválido → seteando\n");
    do_set(ti); return true;
  }
  int64_t delta = llabs(epochFromRtc(r)-epochFromTm(ti));
  DBG("[NTP] Δ RTC-NTP = %lld s\n",(long long)delta);
  if((uint64_t)delta>thr){
    DBG("[NTP] ✅ RTC ajustado (> %us)\n", thr);
    do_set(ti); return true;
  }
  DBG("[NTP] RTC dentro del umbral\n");
  return false;
}

// Fija hora del sistema desde hora de compilación (fallback)
bool setSystemTimeFromBuild(){
  const char* months = "JanFebMarAprMayJunJulAugSepOctNovDec";
  char monStr[4]; int dd, yyyy, hh, mm, ss;
  if (sscanf(__DATE__, "%3s %d %d", monStr, &dd, &yyyy) != 3) return false;
  if (sscanf(__TIME__, "%d:%d:%d", &hh, &mm, &ss) != 3) return false;
  int mon = (strstr(months, monStr) - months) / 3 + 1;
  struct tm t={}; t.tm_year=yyyy-1900; t.tm_mon=mon-1; t.tm_mday=dd; t.tm_hour=hh; t.tm_min=mm; t.tm_sec=ss;
  time_t epoch=mktime(&t); struct timeval tv={ .tv_sec=epoch, .tv_usec=0 }; settimeofday(&tv,nullptr);
  DBG("[SYS] System time set from build: %04d-%02d-%02d %02d:%02d:%02d\n", yyyy, mon, dd, hh, mm, ss);
  return true;
}

// Comando por Serial: SET YYYY-MM-DD HH:MM:SS
bool setRtcFromString(const char* s){
  int Y,M,D,h,m,sec; if (sscanf(s, "SET %d-%d-%d %d:%d:%d", &Y,&M,&D,&h,&m,&sec)!=6) return false;
  if (Y<2000||Y>2099||M<1||M>12||D<1||D>31||h>23||m>59||sec>59) return false;
  Ds1302::DateTime dt; dt.year=(uint8_t)(Y-2000); dt.month=M; dt.day=D; dt.hour=h; dt.minute=m; dt.second=sec;
  struct tm t={}; t.tm_year=Y-1900; t.tm_mon=M-1; t.tm_mday=D; mktime(&t); dt.dow=t.tm_wday?t.tm_wday:7;
  rtc.setDateTime(&dt);
  Ds1302::DateTime chk; rtc.getDateTime(&chk);
  DBG("[RTC] Set from serial: %04d-%02d-%02d %02d:%02d:%02d (dow=%d) post-read valid=%d\n",
      Y,M,D,h,m,sec,dt.dow, rtcIsValid(chk));
  return true;
}

// ===== OLED helpers =====
void drawCentered(const char* t,int y,const uint8_t* f){ u8g2.setFont(f); int w=u8g2.getUTF8Width(t); u8g2.drawUTF8((128-w)/2,y,t); }

// ===== UI =====
void uiWelcome(){
  u8g2.clearBuffer();
  drawCentered("BIENVENIDA A:",10,u8g2_font_6x12_tr);
  drawCentered("VANITY",28,u8g2_font_helvB12_tf);
  drawCentered("LOS PINOS",46,u8g2_font_helvB12_tf);
  u8g2.sendBuffer(); delay(700);
}
void uiConnecting(const char* ssid){
  unsigned long t0=millis(); int step=0;
  DBG("[WiFi] Connecting to '%s'...\n", ssid);
  while(WiFi.status()!=WL_CONNECTED && millis()-t0<20000){
    u8g2.clearBuffer();
    drawCentered("CONECTANDO...",20,u8g2_font_6x12_tr);
    drawCentered(ssid,44,u8g2_font_helvB12_tf);
    const char* dots[]={" ",".","..","..."}; drawCentered(dots[step++%4],62,u8g2_font_6x12_tr);
    u8g2.sendBuffer(); delay(200);
  }
}
void uiWifiOK(const char* ssid){
  u8g2.clearBuffer();
  drawCentered("✓",18,u8g2_font_helvB12_tf);
  drawCentered(ssid,36,u8g2_font_helvB12_tf);
  drawCentered("LISTO",56,u8g2_font_6x12_tr);
  u8g2.sendBuffer();
}
void uiNoWifi(const char* ssid){
  u8g2.clearBuffer();
  drawCentered("SIN WIFI",26,u8g2_font_helvB12_tf);
  drawCentered(ssid,44,u8g2_font_6x12_tr);
  u8g2.sendBuffer(); delay(NO_WIFI_MS);
}
void uiHome(){
  Ds1302::DateTime d; fillNow(d);
  char dateLine[40]; snprintf(dateLine,sizeof(dateLine),"%s | %u DE %s",
    (d.dow>=1&&d.dow<=7)?WEEKDAYS[d.dow%7]:WEEKDAYS[0],
    d.day, (d.month>=1&&d.month<=12)?MONTHS_3[d.month-1]:"???");
  char hhmm[6]; sprintf(hhmm,"%02u:%02u",d.hour,d.minute);
  char ss[3];   sprintf(ss,"%02u",d.second);
  u8g2.clearBuffer();
  drawCentered("VANITY",12,u8g2_font_helvB12_tf);
  drawCentered(dateLine,26,u8g2_font_6x12_tr);
  u8g2.setFont(u8g2_font_logisoso18_tf);
  int w=u8g2.getUTF8Width(hhmm), x=(128-w)/2;
  u8g2.drawUTF8(x,62,hhmm);
  u8g2.setFont(u8g2_font_6x12_tr);
  u8g2.drawUTF8(x+w+4,50,ss);
  u8g2.sendBuffer();
}
void uiRegOK(const String& name,uint8_t hh,uint8_t mm,uint8_t ss){
  char hhmm[6]; sprintf(hhmm,"%02u:%02u",hh,mm); char sss[3]; sprintf(sss,"%02u",ss);
  u8g2.clearBuffer();
  drawCentered("HOLA",12,u8g2_font_6x12_tr);
  drawCentered(name.length()?name.c_str():"",28,u8g2_font_helvB12_tf);
  drawCentered("REGISTRO",40,u8g2_font_6x12_tr);
  u8g2.setFont(u8g2_font_logisoso18_tf);
  int w=u8g2.getUTF8Width(hhmm), x=(128-w)/2; u8g2.drawUTF8(x,62,hhmm);
  u8g2.setFont(u8g2_font_6x12_tr); u8g2.drawUTF8(x+w+4,50,sss);
  u8g2.sendBuffer();
  beepOK(); ledStatBlink(3,80,60);
  delay(FEEDBACK_MS);
}
void uiRegFail(){
  u8g2.clearBuffer();
  drawCentered("✗ FALLO REGISTRO",26,u8g2_font_helvB12_tf);
  drawCentered(":(", 44, u8g2_font_helvB12_tf);
  drawCentered("INTENTA DE NUEVO",60,u8g2_font_6x12_tr);
  u8g2.sendBuffer();
  beepErr(); ledStatBlink(3,150,150);
  delay(FEEDBACK_MS);
}

// ===== NDEF utils =====
String ndefRecordToString(NdefRecord& rec){
  const byte* p=rec.getPayload(); int len=rec.getPayloadLength(); if(!p||len<=0) return "";
  const byte* t=rec.getType(); int tl=rec.getTypeLength(); String type; for(int i=0;i<tl;i++) type+=(char)t[i];
  // Texto con lang code
  if(type=="T" && len>=1){ uint8_t status=p[0], langLen=status&0x3F; int st=1+langLen; String s; for(int i=st;i<len;i++) s+=(char)p[i]; return s; }
  // JSON explícito
  if(type.indexOf("json")>=0 || type=="application/json"){ String s; for(int i=0;i<len;i++) s+=(char)p[i]; return s; }
  // Default: bytes como texto
  String s; for(int i=0;i<len;i++) s+=(char)p[i]; return s;
}

// ===== Fallback MIFARE Classic (rápido) =====
bool readClassicJsonFallback(String& name, String& curp, String& branch){
  MFRC522::MIFARE_Key key; for (byte i=0;i<6;i++) key.keyByte[i]=0xFF;
  String acc; acc.reserve(256); bool found=false;
  for (byte sector=1; sector<=4 && !found; sector++){
    for (byte block=sector*4; block<sector*4+3 && !found; block++){
      MFRC522::StatusCode st = mfrc522.PCD_Authenticate(MFRC522::PICC_CMD_MF_AUTH_KEY_A, block, &key, &(mfrc522.uid));
      if (st != MFRC522::STATUS_OK) continue;
      byte buf[18]; byte size=18; st = mfrc522.MIFARE_Read(block, buf, &size);
      if (st == MFRC522::STATUS_OK){
        for (int i=0;i<16;i++){ char c=(char)buf[i]; if (c>=32 && c<=126) acc += c; }
        int s = acc.indexOf('{'); int e = acc.indexOf('}', s+1);
        if (s>=0 && e>s){
          String json = acc.substring(s, e+1);
          DBG("[Classic] JSON detectado: %s\n", json.c_str());
          StaticJsonDocument<256> jd;
          if (deserializeJson(jd, json)==DeserializationError::Ok){
            if(!name.length())   name   = jd["name"]   | jd["NAME"]   | jd["Name"]   | "";
            if(!curp.length())   curp   = jd["curp"]   | jd["CURP"]   | jd["Curp"]   | "";
            if(!branch.length()) branch = jd["branch"] | jd["BRANCH"] | jd["Branch"] | "";
            found=true;
          }
        }
      }
    }
  }
  mfrc522.PCD_StopCrypto1();
  return found && (name.length() || curp.length() || branch.length());
}

// ===== Cache simple por UID (RAM) =====
struct CacheEntry { String uid,name,curp,branch; uint32_t last; };
CacheEntry cache[8]; // anillo de 8
int cacheIdx=0;
bool cacheGet(const String& uid, String& name, String& curp, String& branch){
  for(auto &e: cache){ if(e.uid==uid){ name=e.name; curp=e.curp; branch=e.branch; e.last=millis(); return true; } }
  return false;
}
void cachePut(const String& uid,const String& name,const String& curp,const String& branch){
  cache[cacheIdx] = {uid,name,curp,branch,millis()};
  cacheIdx = (cacheIdx+1)%8;
}

// ===== HTTP =====
bool sendWebhook(const String& uuid,const String& name,const String& curp,const String& branch,
                 const String& dateStr,const String& timeStr,String& errOut){
  StaticJsonDocument<256> out;
  out["uuid"]=uuid; if(name.length()) out["name"]=name;
  if(curp.length()) out["curp"]=curp; if(branch.length()) out["branch"]=branch;
  out["date"]=dateStr; out["time"]=timeStr;
  String body; serializeJson(out,body);
  DBG("[HTTP] POST → %s\n", WEBHOOK_URL);
  DBG("[HTTP] Body: %s\n", body.c_str());
  int code=-1;
  if(WiFi.status()==WL_CONNECTED){
    WiFiClientSecure client; client.setInsecure();
    client.setTimeout(1500);
    HTTPClient http; http.setTimeout(2000);
    if(!http.begin(client,WEBHOOK_URL)){ errOut="begin()"; return false; }
    http.setUserAgent("clock_pinos_1");               // ← Agent
    http.addHeader("Content-Type","application/json");
    code=http.POST(body);
    String resp=http.getString();
    DBG("[HTTP] code=%d, resp=%s\n", code, resp.c_str());
    http.end();
  } else { errOut="no_wifi"; return false; }
  return (code>0 && code<400);
}

// ===== FSM helper =====
void setState(UIState s){ ui=s; stateT0=millis(); redraw=true; }

// ===== NTP control =====
void startNtpOnce(){
  if(ntpConfigured) return;
  configTime(GMT_OFFSET, DST_OFFSET, "pool.ntp.org", "time.google.com", "time.nist.gov");
  ntpConfigured=true; ntpPending=true; ntpDeadline=millis()+NTP_MAX_MS; ntpLastTry=0;
  DBG("[NTP] start\n");
}

// ===== LECTURA + ENVÍO =====
bool readAndSendOnce(String &nameOut, uint8_t &hhOut, uint8_t &mmOut, uint8_t &ssOut){
  // 1) Detectar tarjeta
  u8g2.clearBuffer(); drawCentered("LEYENDO...",42,u8g2_font_helvB12_tf); u8g2.sendBuffer();
  bool tp = nfc.tagPresent();
  bool piccNew = mfrc522.PICC_IsNewCardPresent();
  bool piccRead = false;
  if (piccNew) piccRead = mfrc522.PICC_ReadCardSerial();
  DBG("[RFID] tagPresent=%d, PICC_IsNew=%d, ReadCardSerial=%d\n", (int)tp, (int)piccNew, (int)piccRead);
  if (!(tp || (piccNew && piccRead))) return false;

  // 2) UID + tipo
  String uidHex;
  if (mfrc522.uid.size) {
    MFRC522::PICC_Type t = mfrc522.PICC_GetType(mfrc522.uid.sak);
    Serial.print("[RFID] PICC type: "); Serial.println(MFRC522::PICC_GetTypeName(t));
    for (byte i=0;i<mfrc522.uid.size;i++){ if(mfrc522.uid.uidByte[i]<0x10) uidHex+='0'; uidHex+=String(mfrc522.uid.uidByte[i],HEX); }
    uidHex.toUpperCase();
    Serial.print("[RFID] UID="); Serial.println(uidHex);
  }

  // 3) Ventana anti-doble
  if(uidHex.length() && uidHex==lastCardUid && (millis()-lastCardAt)<READ_COOLDOWN_MS){
    DBG("[RFID] UID repetido en ventana\n");
    mfrc522.PICC_HaltA(); mfrc522.PCD_StopCrypto1(); return false;
  }

  // 4) Extraer datos: NDEF → Classic → Cache → Genérico
  String name, curp, branch;

  // 4a) NDEF directo
  {
    NfcTag tag = nfc.read();
    if (tag.hasNdefMessage()){
      NdefMessage msg = tag.getNdefMessage();
      DBG("[NDEF] records=%d\n", msg.getRecordCount());
      for (int i=0;i<msg.getRecordCount();i++){
        NdefRecord rec = msg.getRecord(i);
        String payload = ndefRecordToString(rec); payload.trim();
        DBG("[NDEF] rec%d: %s\n", i, payload.c_str());
        StaticJsonDocument<256> jd;
        if(!deserializeJson(jd,payload)){
          if(!name.length())   name   = jd["name"]   | jd["NAME"]   | jd["Name"]   | "";
          if(!curp.length())   curp   = jd["curp"]   | jd["CURP"]   | jd["Curp"]   | "";
          if(!branch.length()) branch = jd["branch"] | jd["BRANCH"] | jd["Branch"] | "";
        }
      }
    } else {
      DBG("[NDEF] sin NDEF\n");
    }
  }

  // 4b) Cache por UID (si alguna vez la tuvimos)
  if((!name.length() || !curp.length() || !branch.length()) && uidHex.length()){
    String cn,cc,cb;
    if (cacheGet(uidHex, cn, cc, cb)){
      DBG("[CACHE] HIT %s → %s/%s/%s\n", uidHex.c_str(), cn.c_str(), cc.c_str(), cb.c_str());
      if(!name.length()) name=cn; if(!curp.length()) curp=cc; if(!branch.length()) branch=cb;
    }
  }

  // 4c) Fallback Classic (rápido)
  if (!name.length() || !curp.length() || !branch.length()){
    if (readClassicJsonFallback(name,curp,branch)){
      DBG("[Classic] OK: name=%s curp=%s branch=%s\n", name.c_str(), curp.c_str(), branch.c_str());
    }
  }

  // 4d) Fallbacks finales / genérico
  if(!name.length())   name   = "CARD_" + (uidHex.length()?uidHex:"UNKN");
  if(!curp.length())   curp   = uidHex.length()?uidHex:"UNKN";
  if(!branch.length()) branch = "pinos";

  // 5) UUID + Hora
  String uuid = uuid10();
  Ds1302::DateTime d; fillNow(d);
  String dateStr = dateISO_fromAny();
  String timeStr = timeISO_fromAny();

  // 6) POST
  String err; bool ok=sendWebhook(uuid,name,curp,branch,dateStr,timeStr,err);
  DBG("[WEBHOOK] ok=%d, uuid=%s, name=%s, curp=%s, branch=%s, date=%s, time=%s\n",
      ok, uuid.c_str(), name.c_str(), curp.c_str(), branch.c_str(), dateStr.c_str(), timeStr.c_str());

  // 7) Actualiza cache para esta UID
  if(uidHex.length()){
    DBG(ok ? "[CACHE] PUT %s\n" : "[CACHE] UPDATE %s\n", uidHex.c_str());
    cachePut(uidHex,name,curp,branch);
  }

  // 8) Salida para UI
  lastCardUid=uidHex; lastCardAt=millis();
  nameOut=name; hhOut=d.hour; mmOut=d.minute; ssOut=d.second;

  // 9) Cierra crypto y HALT
  mfrc522.PICC_HaltA(); mfrc522.PCD_StopCrypto1();

  return ok;
}

// ===== SETUP =====
void setup(){
  Serial.begin(115200); delay(50);
  DBG("\n[BOOT] Arrancando...\n");

  pinMode(LED_WIFI,OUTPUT); digitalWrite(LED_WIFI,LOW);
  pinMode(LED_STAT,OUTPUT); digitalWrite(LED_STAT,LOW);
  pinMode(BUZZER_PIN,OUTPUT); noTone(BUZZER_PIN);
  pinMode(BTN_RESET,INPUT_PULLUP);

  u8g2.begin(); uiWelcome();

  // RFID
  SPI.begin();
  mfrc522.PCD_Init(); delay(50);
  mfrc522.PCD_SetAntennaGain(mfrc522.RxGain_max);
  nfc.begin();

  // RTC
  DBG("[RTC] pins CE=%d IO=%d CLK=%d\n", RTC_RST, RTC_DAT, RTC_CLK);
  rtc.init(); rtc.start();
  Ds1302::DateTime d0; rtc.getDateTime(&d0);
  DBG("[RTC] init read: y=%u m=%u d=%u %02u:%02u:%02u (valid=%d)\n",
      d0.year+2000,d0.month,d0.day,d0.hour,d0.minute,d0.second, rtcIsValid(d0));

  // Hora sistema desde build (por si NTP tarda)
  setSystemTimeFromBuild();

  // Wi-Fi
  WiFi.mode(WIFI_STA); WiFi.disconnect(true); delay(300);
  WiFi.begin(SSID,PASSWORD);
  uiConnecting(SSID);

  if (WiFi.status() == WL_CONNECTED) {
    digitalWrite(LED_WIFI, HIGH); ledWifiBlink3();
    DBG("[WiFi] CONNECTED  IP=%s  RSSI=%d dBm  SSID=%s\n",
        WiFi.localIP().toString().c_str(), WiFi.RSSI(), WiFi.SSID().c_str());
    uiWifiOK(SSID); wifiOkUntil = millis() + WIFI_OK_MS; setState(ST_WIFI_OK);
    ntpConfigured=false; startNtpOnce();
  } else {
    DBG("[WiFi] FAIL status=%s\n", wifiStatusStr(WiFi.status()));
    digitalWrite(LED_WIFI, LOW); uiNoWifi(SSID);
  }
}

// ===== LOOP =====
void loop(){
  unsigned long now=millis();

  // Reset largo (3s)
  bool btn=digitalRead(BTN_RESET);
  if(btn==LOW && btnLast==HIGH) btnT0=now;
  if(btn==LOW && (now-btnT0)>=BTN_RESET_MS){
    u8g2.clearBuffer(); drawCentered("REINICIANDO...",36,u8g2_font_helvB12_tf); u8g2.sendBuffer(); beepOK(); delay(150); ESP.restart();
  }
  btnLast=btn;

  // FSM de UI
  switch(ui){
    case ST_WIFI_OK: if (now > wifiOkUntil) setState(ST_HOME); break;
    case ST_HOME:
      if(redraw || now-lastClock>1000){ lastClock=now; redraw=false; uiHome(); }
      if(now>=cooldownUntil && (nfc.tagPresent() || mfrc522.PICC_IsNewCardPresent())) setState(ST_READING);
      break;
    case ST_READING:{
      String nm; uint8_t hh=0,mm=0,ss=0;
      bool ok=readAndSendOnce(nm,hh,mm,ss);
      cooldownUntil=now+READ_COOLDOWN_MS;
      if(ok) uiRegOK(nm,hh,mm,ss), setState(ST_FEEDBACK_OK);
      else   uiRegFail(),         setState(ST_FEEDBACK_FAIL);
    } break;
    case ST_FEEDBACK_OK:
    case ST_FEEDBACK_FAIL:
      if(now-stateT0>FEEDBACK_MS) setState(ST_HOME);
      break;
  }

  // NTP en segundo plano (no bloquear)
  if (ntpPending && (millis() - ntpLastTry > 500)) {
    ntpLastTry = millis();
    struct tm ti;
    if (getLocalTime(&ti, 10)) {
      syncRtcIfSkewGT(10);
      ntpPending = false;
      DBG("[NTP] first sync OK\n");
    } else if (millis() > ntpDeadline) {
      DBG("[NTP] timeout\n");
      ntpPending = false;
    }
  }

  // Logs de cambios de estado Wi-Fi
  static wl_status_t lastWiFiStatus = WL_NO_SHIELD;
  wl_status_t s = WiFi.status();
  if (s != lastWiFiStatus) {
    DBG("[WiFi] status=%s (%d)\n", wifiStatusStr(s), s);
    if (s == WL_CONNECTED) {
      DBG("[WiFi] IP=%s  RSSI=%d dBm  SSID=%s\n",
          WiFi.localIP().toString().c_str(), WiFi.RSSI(), WiFi.SSID().c_str());
      ntpConfigured=false; startNtpOnce();
    }
    if (s != WL_CONNECTED) digitalWrite(LED_WIFI, LOW);
    lastWiFiStatus = s;
  }

  // Comando Serial: SET YYYY-MM-DD HH:MM:SS
  static String line;
  while (Serial.available()) {
    char c = Serial.read();
    if (c=='\r' || c=='\n') {
      if (line.length()) {
        if (!setRtcFromString(line.c_str())) DBG("[RTC] Bad cmd. Use: SET YYYY-MM-DD HH:MM:SS\n");
        line = "";
      }
    } else if (line.length() < 64) line += c;
  }
}
✅ Checklist antes de probar
OLED: cable corto, pull‑ups 4.7 k si no trae, capacitores 100 nF + 10 µF en VCC/GND.

RC522: 3.3 V y antena despejada.

RTC: batería CR2032 buena, pines CE=25, IO=26, CLK=27.

GND común para todo.

Webhook responde 200 OK.

🧪 Diagnóstico rápido
Hora mala → SET 2025-08-08 14:00:00 por Serial.

Sin NDEF → ve si el log muestra Classic JSON detectado.

Envía pero falla a veces → revisa RSSI Wi‑Fi, HTTP code y timeouts.

OLED se cae → revisa pull‑ups, desacople, baja I²C a 50 kHz.
