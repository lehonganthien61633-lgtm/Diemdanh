#include <WiFi.h>
#include <Firebase_ESP_Client.h>
#include <addons/TokenHelper.h> 
#include <SPI.h>
#include <MFRC522.h>
#include <time.h> 
#include <Preferences.h>
#include <WebServer.h>
#include <DNSServer.h>
#include "esp_camera.h"
#include "quirc.h"
#include "esp_http_server.h"
#include "img_converters.h"
 
const byte DNS_PORT = 53; 
DNSServer dnsServer;
 
// ================= CẤU HÌNH FIREBASE =================
#define API_KEY "AIzaSyD8UhkdBIf2WF9EWXGaXRAgjLORrT3E450"
#define DATABASE_URL "rfid-project-c315c-default-rtdb.asia-southeast1.firebasedatabase.app"
 
// ================= CẤU HÌNH CHÂN (ĐÃ NÉ CHÂN PSRAM VÀ CAMERA) =================
#define SCK_PIN  39
#define MISO_PIN 40
#define MOSI_PIN 41
#define SS_PIN   21
#define RST_PIN  14
 
#define RGB_PIN  48   
#define BOOT_PIN 0    
#define AUDIO_PIN 42  
 
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;
bool signupOK = false;
 
MFRC522 mfrc522(SS_PIN, RST_PIN);
Preferences prefs;
WebServer server(80);
 
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 7 * 3600; 
const int   daylightOffset_sec = 0;
 
unsigned long lastCheckTime = 0;
int currentMode = 0; 
bool rfidEnabled = true;
bool ledEnabled = true;
 
// Dữ liệu NVS 
bool isAdminConfigured = false;
String deviceID = "";
String deviceName = "";
String savedSSID = "";
String savedPASS = "";
 
String lastScannedUID = "";
unsigned long lastScanTime = 0;
 
// Trạng thái Setup
bool inSetupMode = false;
unsigned long bootPressStart = 0;
 
String targetSSID = "";
String targetPASS = "";
int wifiSetupState = 0; 
unsigned long wifiConnectStart = 0;
 
// Biến cho QR Code Task
TaskHandle_t QRCodeReader_Task;
String detectedQR = "";
volatile bool isQRDetected = false; // BẮT BUỘC có volatile để đồng bộ Core
unsigned long lastQRScanTime = 0;
 
void safeNeoPixelWrite(uint8_t r, uint8_t g, uint8_t b) {
  static uint8_t lastR = 255, lastG = 255, lastB = 255;
  if (!ledEnabled) { r = 0; g = 0; b = 0; }
  if (r == lastR && g == lastG && b == lastB) return; 
  lastR = r; lastG = g; lastB = b;
  neopixelWrite(RGB_PIN, r, g, b);
}
 
// ================= GIAO DIỆN CAPTIVE PORTAL =================
const char CAPTIVE_PORTAL_HTML[] PROGMEM = R"=====(
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>Cài Đặt Thiết Bị</title>
  <style>
    body { font-family: 'Segoe UI', Roboto, Helvetica, sans-serif; margin: 0; background-color: #f0f2f5; color: #1c1e21; overflow-x: hidden; }
    .header { background: #f0f2f5; padding: 15px 20px; display: flex; align-items: center; position: sticky; top: 0; z-index: 10; }
    .header .title { font-size: 20px; margin-left: 20px; flex: 1; font-weight:bold;}
    .content-box { background: white; margin: 10px; border-radius: 12px; overflow: hidden; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    .wifi-item { display: flex; align-items: center; padding: 15px 20px; border-bottom: 1px solid #eee; cursor: pointer; transition: 0.2s;}
    .wifi-item:active { background: #f8f9fa; }
    .wifi-icon { font-size: 24px; margin-right: 15px; }
    .wifi-info { flex: 1; }
    .wifi-name { font-size: 16px; font-weight: 500; color: #000; }
    .wifi-status { font-size: 13px; color: #606770; margin-top: 3px; }
    .screen { display: none; padding: 20px; animation: fadeIn 0.3s ease-in-out; }
    .screen.active { display: block; }
    @keyframes fadeIn { from { opacity: 0; transform: translateX(10px); } to { opacity: 1; transform: translateX(0); } }
    input { width: 100%; padding: 14px; margin-top: 10px; border: 1px solid #ccc; border-radius: 8px; font-size: 16px; box-sizing: border-box; background: #f8f9fa; outline:none;}
    input:focus { border-color: #007c8a; background: #fff;}
    button { width: 100%; padding: 14px; margin-top: 20px; border: none; border-radius: 8px; font-size: 16px; font-weight: bold; cursor: pointer; transition: 0.2s;}
    .btn-primary { background: #007c8a; color: white; }
    .btn-primary:active { background: #005f69; }
    .btn-cancel { background: #e4e6eb; color: #1c1e21; margin-top: 10px; }
    .btn-cancel:active { background: #d8dadf; }
    .loading-spinner { border: 4px solid #f3f3f3; border-top: 4px solid #007c8a; border-radius: 50%; width: 40px; height: 40px; animation: spin 1s linear infinite; margin: 40px auto; }
    @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
  </style>
</head>
<body>
  <!-- MÀN 1: CHỌN WI-FI -->
  <div id="screen-wifi" class="screen active">
    <div class="header" style="margin:-20px -20px 10px -20px;"><div class="title">Wi-Fi Khả Dụng</div></div>
    <div class="content-box" id="wifi-list"><div style="text-align:center; padding: 20px; color: #606770;">Đang quét mạng...</div></div>
  </div>
  <!-- MÀN 2: NHẬP PASS WI-FI -->
  <div id="screen-pass" class="screen">
    <h2 id="selected-ssid" style="margin-top:0; color:#007c8a; word-break:break-all;">SSID</h2>
    <p style="color:#606770; margin-bottom: 5px;">Nhập mật khẩu mạng</p>
    <div style="position: relative;">
      <input type="password" id="wifi-pass" placeholder="Mật khẩu..." style="padding-right: 40px;">
      <span id="eye-wifi" style="position: absolute; right: 15px; top: 22px; cursor: pointer; font-size: 20px;" onclick="toggleVisibility('wifi-pass', 'eye-wifi')">👁️</span>
    </div>
    <div id="wifi-noti" style="color:#e74c3c; font-size:14px; margin-top:10px; display:none;"></div>
    <button class="btn-primary" onclick="connectWifi()">Kết Nối</button>
    <button class="btn-cancel" onclick="showScreen('screen-wifi')">Hủy</button>
  </div>
  <!-- MÀN 3: ĐANG KẾT NỐI -->
  <div id="screen-loading" class="screen" style="text-align:center; padding-top:50px;">
    <h2 style="color:#007c8a;">Đang kiểm tra kết nối...</h2>
    <p style="color:#606770;">Vui lòng giữ nguyên trang web này.</p>
    <div class="loading-spinner"></div>
  </div>
  <!-- MÀN 4: ĐĂNG NHẬP ADMIN -->
  <div id="screen-admin" class="screen">
    <div style="text-align:center; margin-bottom: 20px;">
      <div style="background:#eafaf1; color:#27ae60; padding:10px; border-radius:8px; font-weight:bold; margin-bottom:20px;">✓ Wi-Fi đã kết nối!</div>
      <h2 style="margin:0; color:#2c3e50;">Xác Thực Firebase</h2>
      <p style="color:#606770; font-size:14px;">Bảo mật thiết bị với cơ sở dữ liệu</p>
    </div>
    <input type="email" id="admin-email" placeholder="Email (admin@test.com)" value="admin@test.com">
    <div style="position: relative;">
      <input type="password" id="admin-pass" placeholder="Mật khẩu (12345678)" style="padding-right: 40px;">
      <span id="eye-admin" style="position: absolute; right: 15px; top: 22px; cursor: pointer; font-size: 20px;" onclick="toggleVisibility('admin-pass', 'eye-admin')">👁️</span>
    </div>
    <div id="admin-noti" style="color:#e74c3c; font-size:14px; margin-top:10px; display:none;"></div>
    <button class="btn-primary" onclick="verifyAdmin()">Lưu & Hoàn Tất</button>
  </div>
  <!-- MÀN 5: THÀNH CÔNG -->
  <div id="screen-success" class="screen" style="text-align:center; padding-top:50px;">
    <h1 style="color:#27ae60; font-size: 60px; margin:0;">✓</h1>
    <h2 style="color:#2c3e50;">Cài đặt hoàn tất!</h2>
    <p style="color:#606770;">Thiết bị đã được cấp Mã Định Danh và đang kết nối. Bạn có thể đóng trang web này.</p>
  </div>
  <script>
    function toggleVisibility(inputId, iconId) {
      let input = document.getElementById(inputId);
      let icon = document.getElementById(iconId);
      if (input.type === "password") { input.type = "text"; icon.innerText = "🙈"; } 
      else { input.type = "password"; icon.innerText = "👁️"; }
    }
    let currentSsid = "";
    function showScreen(id) {
      document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
      document.getElementById(id).classList.add('active');
    }
    function fetchNetworks() {
      if(document.getElementById('screen-wifi').classList.contains('active')) {
        fetch('/scan').then(res => res.json()).then(data => {
          let html = '';
          data.forEach(net => {
            let icon = net.secure ? '🔒' : '📶';
            let secText = net.secure ? 'Được bảo mật' : 'Mở';
            html += `<div class="wifi-item" onclick="selectWifi('${net.ssid}')"><div class="wifi-icon">${icon}</div><div class="wifi-info"><div class="wifi-name">${net.ssid}</div><div class="wifi-status">${secText}</div></div></div>`;
          });
          document.getElementById('wifi-list').innerHTML = html;
        }).catch(err => {});
      }
    }
    fetchNetworks(); setInterval(fetchNetworks, 6000);
    function selectWifi(ssid) {
      currentSsid = ssid; document.getElementById('selected-ssid').innerText = ssid;
      document.getElementById('wifi-pass').value = ''; document.getElementById('wifi-noti').style.display = 'none';
      showScreen('screen-pass');
    }
    function connectWifi() {
      let pass = document.getElementById('wifi-pass').value; showScreen('screen-loading');
      let formData = new URLSearchParams(); formData.append('ssid', currentSsid); formData.append('pass', pass);
      fetch('/connect', { method: 'POST', body: formData }).then(() => { checkStatus(); });
    }
    function checkStatus() {
      let checkInt = setInterval(() => {
        fetch('/status').then(res => res.text()).then(state => {
          if (state == "2") { clearInterval(checkInt); showScreen('screen-admin'); } 
          else if (state == "3") {
            clearInterval(checkInt);
            document.getElementById('wifi-noti').innerText = "Sai mật khẩu hoặc mất sóng!";
            document.getElementById('wifi-noti').style.display = 'block'; showScreen('screen-pass');
          }
        });
      }, 2000);
    }
    function verifyAdmin() {
      let email = document.getElementById('admin-email').value; let pass = document.getElementById('admin-pass').value;
      let noti = document.getElementById('admin-noti');
      let formData = new URLSearchParams(); formData.append('email', email); formData.append('pass', pass);
      fetch('/saveadmin', { method: 'POST', body: formData }).then(res => res.text()).then(resp => {
        if(resp == "ok") { showScreen('screen-success'); } else { noti.innerText = "Sai tài khoản!"; noti.style.display = 'block'; }
      });
    }
  </script>
</body>
</html>
)=====";
 
// ================= CÁC HÀM XỬ LÝ SERVER =================
void handleRoot() { server.send(200, "text/html", CAPTIVE_PORTAL_HTML); }
void handleScan() {
  int n = WiFi.scanNetworks(); String json = "[";
  for (int i = 0; i < n; ++i) {
    if (i > 0) json += ",";
    json += "{\"ssid\":\"" + WiFi.SSID(i) + "\",\"secure\":" + String(WiFi.encryptionType(i) != WIFI_AUTH_OPEN ? "true" : "false") + "}";
  }
  json += "]"; server.send(200, "application/json", json);
}
void handleConnect() {
  targetSSID = server.arg("ssid"); targetPASS = server.arg("pass");
  WiFi.disconnect(); delay(100);
  wifiSetupState = 1; wifiConnectStart = millis();
  WiFi.begin(targetSSID.c_str(), targetPASS.c_str());
  server.send(200, "text/plain", "connecting");
}
void handleStatus() {
  if (wifiSetupState == 1) {
    if (WiFi.status() == WL_CONNECTED) { wifiSetupState = 2; } 
    else if (millis() - wifiConnectStart > 20000) { wifiSetupState = 3; WiFi.disconnect(); }
  }
  server.send(200, "text/plain", String(wifiSetupState));
}
void handleSaveAdmin() {
  if (server.arg("email") == "admin@test.com" && server.arg("pass") == "12345678") {
    prefs.putString("ssid", targetSSID); prefs.putString("pass", targetPASS); prefs.putBool("admin_ok", true);
    server.send(200, "text/plain", "ok"); delay(1000); ESP.restart();
  } else { server.send(200, "text/plain", "fail"); }
}
 
void startSetupMode() {
  inSetupMode = true;
  WiFi.mode(WIFI_AP_STA); WiFi.softAP("ESP32-S3 Setup"); 
  dnsServer.start(DNS_PORT, "*", WiFi.softAPIP());
  server.on("/", handleRoot); server.on("/scan", handleScan);
  server.on("/connect", HTTP_POST, handleConnect); server.on("/status", handleStatus);
  server.on("/saveadmin", HTTP_POST, handleSaveAdmin);
  server.onNotFound([]() { server.sendHeader("Location", String("http://") + WiFi.softAPIP().toString(), true); server.send(302, "text/plain", ""); });
  server.begin();
  Serial.println("\n[!] DANG MO CHE DO CAI DAT: ESP32-S3 Setup (Khong mat khau)");
}
 
// ================= HỆ THỐNG ÂM THANH =================
#define NOTE_E5  659
#define NOTE_G5  784
#define NOTE_A5  880
#define NOTE_AS5 932
#define NOTE_B5  988
#define NOTE_C6  1047
#define NOTE_D6  1175
#define NOTE_E6  1319
#define NOTE_F6  1397
#define NOTE_G6  1568
#define NOTE_A6  1760
 
void playStartupMelody() {
  int notes[] = { NOTE_E6, NOTE_E6, NOTE_E6, NOTE_C6, NOTE_E6, NOTE_G6, NOTE_G5, NOTE_C6, NOTE_G5, NOTE_E5, NOTE_A5, NOTE_B5, NOTE_AS5, NOTE_A5, NOTE_G5, NOTE_E6, NOTE_G6, NOTE_A6, NOTE_F6, NOTE_G6, NOTE_E6, NOTE_C6, NOTE_D6, NOTE_B5 };
  int tempo[] = { 150, 300, 150, 150, 300, 600, 600, 450, 150, 450, 300, 300, 150, 300, 200, 200, 200, 300, 150, 300, 300, 150, 150, 450 };
  for (int i = 0; i < 24; i++) { tone(AUDIO_PIN, notes[i]); delay(tempo[i] * 0.8); noTone(AUDIO_PIN); delay(tempo[i] * 0.2); }
}
void playSuccessMelody() {
  int notes[] = {NOTE_C6, NOTE_E6, NOTE_G6, NOTE_C6 * 2};
  for(int i = 0; i < 4; i++) { tone(AUDIO_PIN, notes[i]); delay(100); noTone(AUDIO_PIN); delay(50); }
}
void playWrongMelody() {
  tone(AUDIO_PIN, 300); delay(300); noTone(AUDIO_PIN); delay(50);
  tone(AUDIO_PIN, 150); delay(500); noTone(AUDIO_PIN);
}
 
// ================= TIẾN TRÌNH QUÉT QR CAMERA (RTOS TASK) =================
void QRCodeReaderTask(void * pvParameters) {
  struct quirc *q = quirc_new();
  if (!q) {
      Serial.println("Loi: Khong the tao bo nho Quirc!");
      vTaskDelete(NULL);
  }
  
  while(true) {
      if (isQRDetected || currentMode == 1) { 
          vTaskDelay(200 / portTICK_PERIOD_MS);
          continue;
      }
      
      camera_fb_t * fb = esp_camera_fb_get();
      if (!fb) {
          vTaskDelay(100 / portTICK_PERIOD_MS);
          continue;
      }
      
      if (quirc_resize(q, fb->width, fb->height) == 0) {
          uint8_t *image = quirc_begin(q, NULL, NULL);
          if (image != NULL) {
              memcpy(image, fb->buf, fb->len);
              quirc_end(q);
              
              int count = quirc_count(q);
              if (count > 0) {
                  struct quirc_code code;
                  quirc_extract(q, 0, &code);
                  struct quirc_data data;
                  if (!quirc_decode(&code, &data)) {
                      String payload = (const char *)data.payload;
                      if (payload.startsWith("QR_EXIT|") && (millis() - lastQRScanTime > 4000)) {
                          detectedQR = payload;
                          isQRDetected = true;
                          lastQRScanTime = millis();
                          Serial.println("Da quet duoc QR: " + detectedQR);
                          
                          // ĐÈN XANH DƯƠNG: Báo hiệu Camera đã đọc thành công mã QR (Đang chờ Firebase)
                          neopixelWrite(RGB_PIN, 0, 0, 255);
                      }
                  }
              }
          }
      }
      esp_camera_fb_return(fb);
      vTaskDelay(30 / portTICK_PERIOD_MS); // Rút ngắn delay để quét tốc độ cao ~30FPS
  }
}
 httpd_handle_t stream_httpd = NULL;
#define PART_BOUNDARY "123456789000000000000987654321"
static const char* _STREAM_CONTENT_TYPE = "multipart/x-mixed-replace;boundary=" PART_BOUNDARY;
static const char* _STREAM_BOUNDARY = "\r\n--" PART_BOUNDARY "\r\n";
static const char* _STREAM_PART = "Content-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n";

static esp_err_t stream_handler(httpd_req_t *req) {
  camera_fb_t * fb = NULL;
  esp_err_t res = ESP_OK;
  size_t _jpg_buf_len = 0;
  uint8_t * _jpg_buf = NULL;
  char * part_buf[64];

  res = httpd_resp_set_type(req, _STREAM_CONTENT_TYPE);
  if(res != ESP_OK) return res;

  while(true) {
    fb = esp_camera_fb_get();
    if (!fb) {
      res = ESP_FAIL;
    } else {
      // Vì camera đang chụp dạng Đen/Trắng (GRAYSCALE) để đọc QR, ta phải ép nén sang JPEG để truyền Web
      if (fb->format != PIXFORMAT_JPEG) {
        bool jpeg_converted = frame2jpg(fb, 80, &_jpg_buf, &_jpg_buf_len);
        esp_camera_fb_return(fb);
        fb = NULL;
        if (!jpeg_converted) res = ESP_FAIL;
      } else {
        _jpg_buf_len = fb->len;
        _jpg_buf = fb->buf;
      }
    }
    
    if (res == ESP_OK) {
      size_t hlen = snprintf((char *)part_buf, 64, _STREAM_PART, _jpg_buf_len);
      res = httpd_resp_send_chunk(req, (const char *)part_buf, hlen);
    }
    if (res == ESP_OK) res = httpd_resp_send_chunk(req, (const char *)_jpg_buf, _jpg_buf_len);
    if (res == ESP_OK) res = httpd_resp_send_chunk(req, _STREAM_BOUNDARY, strlen(_STREAM_BOUNDARY));
    
    if (fb) { esp_camera_fb_return(fb); fb = NULL; _jpg_buf = NULL; } 
    else if (_jpg_buf) { free(_jpg_buf); _jpg_buf = NULL; }
    
    if (res != ESP_OK) break;
    
    // Rất quan trọng: Nghỉ 50ms giữa các frame để chừa CPU cho Task QR quét ngầm
    vTaskDelay(50 / portTICK_PERIOD_MS); 
  }
  return res;
}

void startStreamServer() {
  httpd_config_t config = HTTPD_DEFAULT_CONFIG();
  config.server_port = 81; // Khởi động Web Server thứ 2 trên Port 81 (Tránh đụng Captive Portal Port 80)
  
  httpd_uri_t stream_uri = {
    .uri = "/stream",
    .method = HTTP_GET,
    .handler = stream_handler,
    .user_ctx = NULL
  };
  
  if (httpd_start(&stream_httpd, &config) == ESP_OK) {
    httpd_register_uri_handler(stream_httpd, &stream_uri);
    Serial.println("Khoi dong thanh cong Camera Stream Server (Port 81)");
  }
}
// ================= SETUP =================
void setup() {
  Serial.begin(115200);
  pinMode(BOOT_PIN, INPUT_PULLUP);
  
  // Khởi tạo SPI Custom Pins cho RC522 để né Camera và RAM
  SPI.begin(SCK_PIN, MISO_PIN, MOSI_PIN, SS_PIN); 
  mfrc522.PCD_Init();
  mfrc522.PCD_SetAntennaGain(mfrc522.RxGain_max);
  
  // Cấu hình Camera chuẩn ESP32S3_EYE (Dành cho OV3660 / OV5640)
  camera_config_t configCam;
  configCam.ledc_channel = LEDC_CHANNEL_0;
  configCam.ledc_timer = LEDC_TIMER_0;
  configCam.pin_d0 = 11; 
  configCam.pin_d1 = 9;  
  configCam.pin_d2 = 8;  
  configCam.pin_d3 = 10; 
  configCam.pin_d4 = 12; 
  configCam.pin_d5 = 18; 
  configCam.pin_d6 = 17; 
  configCam.pin_d7 = 16; 
  configCam.pin_xclk = 15;
  configCam.pin_pclk = 13;
  configCam.pin_vsync = 6; 
  configCam.pin_href = 7;
  configCam.pin_sccb_sda = 4; 
  configCam.pin_sccb_scl = 5; 
  configCam.pin_pwdn = -1;
  configCam.pin_reset = -1;
  configCam.xclk_freq_hz = 20000000;
  configCam.pixel_format = PIXFORMAT_GRAYSCALE;
  configCam.frame_size = FRAMESIZE_VGA; // Nâng lên VGA (640x480) để mã QR sắc nét hơn
  
  if (psramFound()) {
      configCam.fb_count = 2;
      configCam.fb_location = CAMERA_FB_IN_PSRAM;
  } else {
      configCam.fb_count = 1;
      configCam.fb_location = CAMERA_FB_IN_DRAM;
  }
 
  if (esp_camera_init(&configCam) == ESP_OK) {
      // Ép phần cứng quang học để dễ đọc màn hình điện thoại hơn
      sensor_t * s = esp_camera_sensor_get();
      if (s != NULL) {
          s->set_vflip(s, 1);    // Lật ảnh dọc (Tránh bị ngược QR)
          s->set_contrast(s, 2); // Tăng cường độ tương phản (Quan trọng)
      }
      
      // Cấp 20480 Bytes RAM cho Task
      xTaskCreatePinnedToCore(QRCodeReaderTask, "QRCodeReader_Task", 20480, NULL, 1, &QRCodeReader_Task, 1);
      Serial.println("Camera & QR Task OK!");
  } else {
      Serial.println("Loi Camera!");
  }
 
  pinMode(AUDIO_PIN, OUTPUT);
  digitalWrite(AUDIO_PIN, LOW);
  playStartupMelody();
 
  prefs.begin("config", false);
  deviceID = prefs.getString("id", "");
  if (deviceID == "") {
    const char charset[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    for (int i = 0; i < 10; i++) { deviceID += charset[esp_random() % 36]; }
    prefs.putString("id", deviceID);
  }
  
  if (prefs.getBool("deleted", false)) {
    Serial.println("\n[!] THIET BI DANG O TRANG THAI NGHI.");
    while(true) {
      if (digitalRead(BOOT_PIN) == LOW) {
        delay(5000); 
        if (digitalRead(BOOT_PIN) == LOW) {
          prefs.putBool("deleted", false); 
          ESP.restart();
        }
      }
      neopixelWrite(RGB_PIN, 200, 150, 0); delay(500); neopixelWrite(RGB_PIN, 0, 0, 0); delay(500);
    }
  }
 
  isAdminConfigured = prefs.getBool("admin_ok", false);
  deviceName = prefs.getString("name", "Cổng " + deviceID.substring(6)); 
  savedSSID = prefs.getString("ssid", "");
  savedPASS = prefs.getString("pass", "");
  
  safeNeoPixelWrite(255, 0, 0); 
  WiFi.mode(WIFI_STA); WiFi.disconnect(); delay(100);
 
  bool needSetup = false;
  if (savedSSID == "") { needSetup = true; } 
  else {
    WiFi.begin(savedSSID.c_str(), savedPASS.c_str());
    int attempts = 0;
    while (WiFi.status() != WL_CONNECTED && attempts < 20) { delay(500); Serial.print("."); attempts++; }
    if (WiFi.status() != WL_CONNECTED) needSetup = true;
  }
  if (!isAdminConfigured) needSetup = true;
 
  if (needSetup) { startSetupMode(); return; }
  
  // Bật Server truyền hình ảnh Camera lên ngay khi có mạng Wi-Fi
  startStreamServer();
  
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) { delay(500); }
  
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;
  auth.user.email = "admin@test.com";
  auth.user.password = "12345678";
  signupOK = true;
  config.token_status_callback = tokenStatusCallback; 
  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
 
  fbdo.setBSSLBufferSize(16384, 4096);
  fbdo.setResponseSize(8192);
 
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/name", deviceName);
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ip", WiFi.localIP().toString());
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/mac", WiFi.macAddress());
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ssid", WiFi.SSID());
  Firebase.RTDB.setBool(&fbdo, "/Devices/" + deviceID + "/modules/rfid", true);
  Firebase.RTDB.setBool(&fbdo, "/Devices/" + deviceID + "/modules/led", true);
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/action", "none");
}
 
// ================= LOOP =================
void loop() {
  if (digitalRead(BOOT_PIN) == LOW) {
    if (bootPressStart == 0) bootPressStart = millis();
    else if (millis() - bootPressStart > 5000 && !inSetupMode) { startSetupMode(); }
  } else { bootPressStart = 0; }
 
  if (inSetupMode) {
    dnsServer.processNextRequest(); 
    server.handleClient();
    if (millis() % 1000 < 500) safeNeoPixelWrite(0, 0, 255); else safeNeoPixelWrite(0, 0, 0);
    return;
  }
 
  if (WiFi.status() != WL_CONNECTED) {
    if (millis() % 500 < 250) safeNeoPixelWrite(255, 0, 0); else safeNeoPixelWrite(0, 0, 0);
    return;
  }
 
  time_t now; time(&now);
 
  if (millis() - lastCheckTime > 2000) {
    lastCheckTime = millis();
    if (Firebase.ready() && signupOK) {
      Firebase.RTDB.setInt(&fbdo, "/Devices/" + deviceID + "/lastPing", now);
      if (Firebase.RTDB.getBool(&fbdo, "/Devices/" + deviceID + "/modules/rfid")) rfidEnabled = fbdo.boolData();
      if (Firebase.RTDB.getBool(&fbdo, "/Devices/" + deviceID + "/modules/led")) ledEnabled = fbdo.boolData();
      if (Firebase.RTDB.getInt(&fbdo, "/Command/mode")) currentMode = fbdo.intData();
 
      if (Firebase.RTDB.getString(&fbdo, "/Devices/" + deviceID + "/Command/action")) {
        String action = fbdo.stringData();
        if (action == "update_info") {
          Firebase.RTDB.getString(&fbdo, "/Devices/" + deviceID + "/Command/new_name"); 
          String newName = fbdo.stringData();
          if(newName != "") {
            prefs.putString("name", newName);
            Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/action", "none"); 
            ESP.restart();
          }
        }
        else if (action == "connect_wifi") {
          Firebase.RTDB.getString(&fbdo, "/Devices/" + deviceID + "/Command/ssid"); String nSsid = fbdo.stringData();
          Firebase.RTDB.getString(&fbdo, "/Devices/" + deviceID + "/Command/pass"); String nPass = fbdo.stringData();
          if(nSsid != "") {
            WiFi.disconnect(); delay(100); WiFi.begin(nSsid.c_str(), nPass.c_str());
            int attempts = 0; bool connectSuccess = false;
            while (attempts < 20) { 
              if (WiFi.status() == WL_CONNECTED) { connectSuccess = true; break; }
              delay(500); attempts++;
            }
            if (connectSuccess) {
              prefs.putString("ssid", nSsid); prefs.putString("pass", nPass);
              savedSSID = nSsid; savedPASS = nPass;
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ip", WiFi.localIP().toString());
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ssid", WiFi.SSID());
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/action", "none"); 
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/wifi_status", "success");
            } else {
              WiFi.disconnect(); delay(100); WiFi.begin(savedSSID.c_str(), savedPASS.c_str());
              while (WiFi.status() != WL_CONNECTED) { delay(500); }
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/action", "none"); 
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/wifi_status", "fail");
            }
          }
        }
        else if (action == "delete") {
          Firebase.RTDB.deleteNode(&fbdo, "/Devices/" + deviceID); 
          prefs.clear(); prefs.putBool("deleted", true); ESP.restart();
        }
      }
    }
  }
 
  if (currentMode == 1) safeNeoPixelWrite(0, 255, 0); else safeNeoPixelWrite(255, 0, 0); 
 
  // ================= XỬ LÝ QUÉT QR RA SỚM =================
  if (isQRDetected) {
      bool success = false;
      if (Firebase.ready() && signupOK) {
          String qrPayload = detectedQR;
          if (qrPayload.startsWith("QR_EXIT|")) {
              int firstPipe = qrPayload.indexOf('|');
              int secondPipe = qrPayload.indexOf('|', firstPipe + 1);
              if (firstPipe != -1 && secondPipe != -1) {
                  String uid = qrPayload.substring(firstPipe + 1, secondPipe);
                  String reason = qrPayload.substring(secondPipe + 1);
                  struct tm timeinfo; 
                  if (getLocalTime(&timeinfo)) {
                      char timeStringBuff[20]; 
                      strftime(timeStringBuff, sizeof(timeStringBuff), "%Y-%m-%d %H:%M:%S", &timeinfo);
                      String timestamp = String(timeStringBuff);
                      time_t now; time(&now);
                      
                      Firebase.RTDB.setString(&fbdo, "/Members/" + uid + "/lastStatus", "Ra");
                      String saveStatus = "Ra (Về sớm)";
                      FirebaseJson jsonLog; 
                      jsonLog.set("time", timestamp); 
                      jsonLog.set("status", saveStatus);
                      jsonLog.set("reason", reason);
                      Firebase.RTDB.setJSON(&fbdo, "/Members/" + uid + "/Attendance/" + String(now), &jsonLog);
                      
                      playSuccessMelody();
                      safeNeoPixelWrite(0, 255, 0); delay(1000); safeNeoPixelWrite(0, 0, 0); // XANH LÁ: Lưu Firebase thành công
                      success = true;
                  }
              }
          }
      }
      
      // Nếu có lỗi (mất mạng, mã sai định dạng, đồng bộ thời gian xịt)
      if (!success) {
          playWrongMelody();
          for(int i = 0; i < 4; i++) { safeNeoPixelWrite(255, 0, 0); delay(150); safeNeoPixelWrite(0, 0, 0); delay(150); } // ĐỎ: Thất bại
      }
      
      isQRDetected = false; // Mở khóa cho lần quét tiếp theo
  }
 
  // ================= XỬ LÝ QUÉT THẺ RFID =================
  if (rfidEnabled) {
    if (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) return;
 
    String uid = "";
    for (byte i = 0; i < mfrc522.uid.size; i++) {
      uid += String(mfrc522.uid.uidByte[i] < 0x10 ? "0" : ""); uid += String(mfrc522.uid.uidByte[i], HEX);
    }
    uid.toUpperCase();
 
    if (uid == lastScannedUID && (millis() - lastScanTime < 5000)) { mfrc522.PICC_HaltA(); return; }
    lastScannedUID = uid; lastScanTime = millis();
 
    struct tm timeinfo; if (!getLocalTime(&timeinfo)) return;
    char timeStringBuff[20]; strftime(timeStringBuff, sizeof(timeStringBuff), "%Y-%m-%d %H:%M:%S", &timeinfo);
    String timestamp = String(timeStringBuff);
 
    if (Firebase.ready() && signupOK) {
      if (currentMode == 1) {
        Firebase.RTDB.setString(&fbdo, "/Command/new_uid", uid);
        Firebase.RTDB.setInt(&fbdo, "/Command/mode", 0); currentMode = 0;
        playSuccessMelody(); 
        for(int i = 0; i < 3; i++) { safeNeoPixelWrite(0,0,0); delay(200); safeNeoPixelWrite(0,255,0); delay(200); }
      } else {
        if (Firebase.RTDB.getJSON(&fbdo, "/Members/" + uid)) {
            FirebaseJson &jsonNode = fbdo.jsonObject(); FirebaseJsonData result;
            jsonNode.get(result, "name");
            if (!result.success) { 
                playWrongMelody(); 
                for(int i = 0; i < 4; i++) { safeNeoPixelWrite(0,0,255); delay(150); safeNeoPixelWrite(0,0,0); delay(150); }
                return;
            }
            
            String currentStatus = "Vào";
            jsonNode.get(result, "lastStatus"); if (result.success && result.stringValue == "Vào") currentStatus = "Ra";
            
            String activeStart = "07:00", activeEnd = "17:00", p_start = "", p_end = "";
            jsonNode.get(result, "p_start"); if (result.success) p_start = result.stringValue;
            jsonNode.get(result, "p_end"); if (result.success) p_end = result.stringValue;
            if (p_start != "") { activeStart = p_start; activeEnd = p_end; }
            
            String today = timestamp.substring(0, 10);
            jsonNode.get(result, "tempShifts");
            if (result.success) {
                FirebaseJson tsJson; tsJson.setJsonData(result.stringValue);
                size_t len = tsJson.iteratorBegin();
                for (size_t i = 0; i < len; i++) {
                    int type; String key, val; tsJson.iteratorGet(i, type, key, val);
                    FirebaseJson item; item.setJsonData(val); FirebaseJsonData itemData;
                    item.get(itemData, "s_from"); String s_from = itemData.stringValue;
                    item.get(itemData, "s_to"); String s_to = itemData.stringValue;
                    if (today >= s_from && today <= s_to) {
                        item.get(itemData, "s_start"); activeStart = itemData.stringValue;
                        item.get(itemData, "s_end"); activeEnd = itemData.stringValue;
                        break;
                    }
                }
            }
            
            String currentTime = timestamp.substring(11, 16); String eval = "Đúng giờ";
            if (currentStatus == "Vào" && currentTime > activeStart) eval = "Đi trễ";
            else if (currentStatus == "Ra") {
                if (currentTime < activeEnd) {
                    // Cấm quẹt thẻ ra sớm - Yêu cầu quét QR
                    playWrongMelody();
                    for(int i = 0; i < 4; i++) { safeNeoPixelWrite(255,165,0); delay(150); safeNeoPixelWrite(0,0,0); delay(150); }
                    mfrc522.PICC_HaltA();
                    return; 
                } else if (currentTime > activeEnd) {
                    eval = "Tăng ca";
                }
            }
            
            Firebase.RTDB.setString(&fbdo, "/Members/" + uid + "/lastStatus", currentStatus);
            String saveStatus = currentStatus + " (" + eval + ")";
            FirebaseJson jsonLog; jsonLog.set("time", timestamp); jsonLog.set("status", saveStatus); 
            Firebase.RTDB.setJSON(&fbdo, "/Members/" + uid + "/Attendance/" + String(now), &jsonLog);
            
            playSuccessMelody(); 
            safeNeoPixelWrite(0, 255, 0); delay(500);
        } else {
            playWrongMelody(); 
            for(int i = 0; i < 4; i++) { safeNeoPixelWrite(0,0,255); delay(150); safeNeoPixelWrite(0,0,0); delay(150); }
        }
      }
    }
    mfrc522.PICC_HaltA(); 
  }
}

