#include <WiFi.h>
#include <Firebase_ESP_Client.h>
#include <addons/TokenHelper.h> 
#include <SPI.h>
#include <MFRC522.h>
#include <time.h> 
#include <Preferences.h>
#include <WebServer.h>
#include <DNSServer.h>

const byte DNS_PORT = 53; 
DNSServer dnsServer;

// Cấu hình Firebase
#define API_KEY "AIzaSyD8UhkdBIf2WF9EWXGaXRAgjLORrT3E450"
#define DATABASE_URL "rfid-project-c315c-default-rtdb.asia-southeast1.firebasedatabase.app"
#define RST_PIN 9
#define SS_PIN 10
#define RGB_PIN 48   
#define BOOT_PIN 0 

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

// Xử lý Wi-Fi nền (Asynchronous) để không bị văng mạng
String targetSSID = "";
String targetPASS = "";
int wifiSetupState = 0; // 0: rảnh, 1: đang thử kết nối, 2: thành công, 3: thất bại
unsigned long wifiConnectStart = 0;

void safeNeoPixelWrite(uint8_t r, uint8_t g, uint8_t b) {
  if (ledEnabled) neopixelWrite(RGB_PIN, r, g, b);
  else neopixelWrite(RGB_PIN, 0, 0, 0); 
}

// ================= GIAO DIỆN CAPTIVE PORTAL LIỀN MẠCH (SPA) =================
// Lưu trên PROGMEM để load siêu nhanh và tiết kiệm RAM
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
    <div class="header" style="margin:-20px -20px 10px -20px;">
      <div class="title">Wi-Fi Khả Dụng</div>
    </div>
    <div class="content-box" id="wifi-list">
      <div style="text-align:center; padding: 20px; color: #606770;">Đang quét mạng...</div>
    </div>
  </div>

  <!-- MÀN 2: NHẬP PASS WI-FI -->
  <div id="screen-pass" class="screen">
    <h2 id="selected-ssid" style="margin-top:0; color:#007c8a; word-break:break-all;">SSID</h2>
    <p style="color:#606770; margin-bottom: 5px;">Nhập mật khẩu mạng</p>
    <input type="password" id="wifi-pass" placeholder="Mật khẩu...">
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
    <input type="password" id="admin-pass" placeholder="Mật khẩu (12345678)">
    <div id="admin-noti" style="color:#e74c3c; font-size:14px; margin-top:10px; display:none;"></div>
    <button class="btn-primary" onclick="verifyAdmin()">Lưu & Hoàn Tất</button>
  </div>

  <!-- MÀN 5: THÀNH CÔNG -->
  <div id="screen-success" class="screen" style="text-align:center; padding-top:50px;">
    <h1 style="color:#27ae60; font-size: 60px; margin:0;">✓</h1>
    <h2 style="color:#2c3e50;">Cài đặt hoàn tất!</h2>
    <p style="color:#606770;">Thiết bị đã được cấp Mã Định Danh và đang kết nối vào hệ thống. Bạn có thể đóng trang web này.</p>
  </div>

  <script>
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
            html += `<div class="wifi-item" onclick="selectWifi('${net.ssid}')">
                      <div class="wifi-icon">${icon}</div>
                      <div class="wifi-info"><div class="wifi-name">${net.ssid}</div><div class="wifi-status">${secText}</div></div>
                    </div>`;
          });
          document.getElementById('wifi-list').innerHTML = html;
        }).catch(err => {});
      }
    }
    
    fetchNetworks();
    setInterval(fetchNetworks, 6000);

    function selectWifi(ssid) {
      currentSsid = ssid;
      document.getElementById('selected-ssid').innerText = ssid;
      document.getElementById('wifi-pass').value = '';
      document.getElementById('wifi-noti').style.display = 'none';
      showScreen('screen-pass');
    }

    function connectWifi() {
      let pass = document.getElementById('wifi-pass').value;
      showScreen('screen-loading');
      let formData = new URLSearchParams();
      formData.append('ssid', currentSsid); formData.append('pass', pass);
      fetch('/connect', { method: 'POST', body: formData }).then(() => { checkStatus(); });
    }

    function checkStatus() {
      let checkInt = setInterval(() => {
        fetch('/status').then(res => res.text()).then(state => {
          if (state == "2") {
            clearInterval(checkInt); showScreen('screen-admin');
          } else if (state == "3") {
            clearInterval(checkInt);
            document.getElementById('wifi-noti').innerText = "Sai mật khẩu hoặc mất sóng! Vui lòng thử lại.";
            document.getElementById('wifi-noti').style.display = 'block';
            showScreen('screen-pass');
          }
        });
      }, 2000);
    }

    function verifyAdmin() {
      let email = document.getElementById('admin-email').value;
      let pass = document.getElementById('admin-pass').value;
      let noti = document.getElementById('admin-noti');
      let formData = new URLSearchParams();
      formData.append('email', email); formData.append('pass', pass);
      fetch('/saveadmin', { method: 'POST', body: formData }).then(res => res.text()).then(resp => {
        if(resp == "ok") { showScreen('screen-success'); } 
        else { noti.innerText = "Sai tài khoản Admin!"; noti.style.display = 'block'; }
      });
    }
  </script>
</body>
</html>
)=====";

// ================= CÁC HÀM XỬ LÝ SERVER =================

void handleRoot() {
  server.send(200, "text/html", CAPTIVE_PORTAL_HTML);
}

void handleScan() {
  int n = WiFi.scanNetworks();
  String json = "[";
  for (int i = 0; i < n; ++i) {
    if (i > 0) json += ",";
    json += "{\"ssid\":\"" + WiFi.SSID(i) + "\",\"secure\":" + String(WiFi.encryptionType(i) != WIFI_AUTH_OPEN ? "true" : "false") + "}";
  }
  json += "]";
  server.send(200, "application/json", json);
}

void handleConnect() {
  targetSSID = server.arg("ssid");
  targetPASS = server.arg("pass");
  
  wifiSetupState = 1; 
  wifiConnectStart = millis();
  WiFi.begin(targetSSID.c_str(), targetPASS.c_str());
  
  server.send(200, "text/plain", "connecting");
}

void handleStatus() {
  if (wifiSetupState == 1) {
    if (WiFi.status() == WL_CONNECTED) {
      wifiSetupState = 2; // Thành công
    } else if (millis() - wifiConnectStart > 10000) { 
      // Timeout sau 10 giây nếu sai mật khẩu
      wifiSetupState = 3; // Thất bại
      WiFi.disconnect();
    }
  }
  server.send(200, "text/plain", String(wifiSetupState));
}

void handleSaveAdmin() {
  String email = server.arg("email");
  String apass = server.arg("pass"); 

  if (email == "admin@test.com" && apass == "12345678") {
    prefs.putString("ssid", targetSSID);
    prefs.putString("pass", targetPASS);
    prefs.putBool("admin_ok", true);
    server.send(200, "text/plain", "ok");
    delay(1000); 
    ESP.restart(); // Khởi động lại toàn bộ mạch để áp dụng NVS
  } else {
    server.send(200, "text/plain", "fail");
  }
}

void startSetupMode() {
  inSetupMode = true;
  // BẬT CHẾ ĐỘ VỪA PHÁT VỪA THU: Điện thoại không bị rớt mạng khi mạch tự test Wi-Fi
  WiFi.mode(WIFI_AP_STA);
  WiFi.softAP("ESP32-S3 Setup"); 
  dnsServer.start(DNS_PORT, "*", WiFi.softAPIP());
  
  server.on("/", handleRoot);
  server.on("/scan", handleScan);
  server.on("/connect", HTTP_POST, handleConnect);
  server.on("/status", handleStatus);
  server.on("/saveadmin", HTTP_POST, handleSaveAdmin);
  server.onNotFound([]() { server.sendHeader("Location", String("http://") + WiFi.softAPIP().toString(), true); server.send(302, "text/plain", ""); });
  
  server.begin();
  Serial.println("\n[!] DANG MO CHE DO CAI DAT: ESP32-S3 Setup (Khong mat khau)");
}

// ================= SETUP CHÍNH =================

void setup() {
  Serial.begin(115200);
  pinMode(BOOT_PIN, INPUT_PULLUP);
  SPI.begin(12, 13, 11, 10); 
  mfrc522.PCD_Init();
  mfrc522.PCD_SetAntennaGain(mfrc522.RxGain_max);
  
  prefs.begin("config", false);

  // TẠO MÃ ĐỊNH DANH NGẪU NHIÊN 10 KÝ TỰ VĨNH VIỄN
  deviceID = prefs.getString("id", "");
  if (deviceID == "") {
    const char charset[] = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    deviceID = "";
    for (int i = 0; i < 10; i++) {
      deviceID += charset[esp_random() % 36];
    }
    prefs.putString("id", deviceID);
  }
  
  if (prefs.getBool("deleted", false)) {
    Serial.println("\n[!] THIET BI DANG O TRANG THAI NGHI. Nhan giu BOOT 5 giay de kich hoat lai.");
    while(true) {
      if (digitalRead(BOOT_PIN) == LOW) {
        delay(5000); 
        if (digitalRead(BOOT_PIN) == LOW) {
          prefs.putBool("deleted", false); 
          Serial.println("\n[!] Khoi dong lai he thong...");
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

  if (savedSSID == "") {
    needSetup = true;
  } else {
    Serial.println("\nDang ket noi Wi-Fi luu: " + savedSSID);
    WiFi.begin(savedSSID.c_str(), savedPASS.c_str());
    int attempts = 0;
    while (WiFi.status() != WL_CONNECTED && attempts < 20) { delay(500); Serial.print("."); attempts++; }
    if (WiFi.status() != WL_CONNECTED) {
      Serial.println("\nKhong the ket noi Wi-Fi. Mo che do Cai dat...");
      needSetup = true;
    }
  }

  if (!isAdminConfigured) needSetup = true;

  if (needSetup) {
    startSetupMode();
    return; 
  }

  Serial.println("\n-> WiFi Da Ket Noi! IP: " + WiFi.localIP().toString());
  
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

  // --- NÂNG CẤP BỘ NHỚ ĐỆM SSL LÊN MỨC TỐI ĐA ---
  fbdo.setBSSLBufferSize(16384, 4096);
  fbdo.setResponseSize(8192);
  // ------------------------------------------------

  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/name", deviceName);
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ip", WiFi.localIP().toString());
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/mac", WiFi.macAddress());
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ssid", WiFi.SSID());
  Firebase.RTDB.setBool(&fbdo, "/Devices/" + deviceID + "/modules/rfid", true);
  Firebase.RTDB.setBool(&fbdo, "/Devices/" + deviceID + "/modules/led", true);
  Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/action", "none");
}

void loop() {
  if (digitalRead(BOOT_PIN) == LOW) {
    if (bootPressStart == 0) bootPressStart = millis();
    else if (millis() - bootPressStart > 5000 && !inSetupMode) {
      startSetupMode();
    }
  } else {
    bootPressStart = 0;
  }

  if (inSetupMode) {
    dnsServer.processNextRequest(); 
    server.handleClient();
    if (millis() % 1000 < 500) neopixelWrite(RGB_PIN, 0, 0, 255); else neopixelWrite(RGB_PIN, 0, 0, 0);
    return;
  }

  if (WiFi.status() != WL_CONNECTED) {
    if (millis() % 500 < 250) neopixelWrite(RGB_PIN, 255, 0, 0); else neopixelWrite(RGB_PIN, 0, 0, 0);
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
            Serial.println("\n[!] Dang kiem tra Wi-Fi moi: " + nSsid);
            
            // 1. Tạm ngắt mạng hiện tại để thử mạng mới
            WiFi.disconnect();
            delay(100);
            WiFi.begin(nSsid.c_str(), nPass.c_str());
            
            // 2. Chờ tối đa 10 giây để kết nối
            int attempts = 0;
            bool connectSuccess = false;
            while (attempts < 20) { 
              if (WiFi.status() == WL_CONNECTED) { connectSuccess = true; break; }
              delay(500); Serial.print(".");
              attempts++;
            }
            
            // 3. Xử lý kết quả
            if (connectSuccess) {
              Serial.println("\n[!] Ket noi THANH CONG Wi-Fi moi!");
              // Lưu Wi-Fi mới vào bộ nhớ
              prefs.putString("ssid", nSsid); 
              prefs.putString("pass", nPass);
              savedSSID = nSsid; // Cập nhật biến RAM
              savedPASS = nPass;
              
              // Cập nhật lại IP và SSID mới lên Firebase
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ip", WiFi.localIP().toString());
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/ssid", WiFi.SSID());
              
              // Báo cáo thành công cho Web
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/action", "none"); 
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/wifi_status", "success");
            } else {
              Serial.println("\n[!] Ket noi THAT BAI! Dang tra ve Wi-Fi cu...");
              WiFi.disconnect();
              delay(100);
              // Khôi phục kết nối Wi-Fi cũ
              WiFi.begin(savedSSID.c_str(), savedPASS.c_str());
              while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
              
              Serial.println("\n[!] Da khoi phuc Wi-Fi cu. Báo loi ve Web.");
              
              // Báo cáo thất bại cho Web
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/action", "none"); 
              Firebase.RTDB.setString(&fbdo, "/Devices/" + deviceID + "/Command/wifi_status", "fail");
            }
          }
        }
        else if (action == "delete") {
          Firebase.RTDB.deleteNode(&fbdo, "/Devices/" + deviceID); 
          prefs.clear(); 
          prefs.putBool("deleted", true); 
          ESP.restart();
        }
      }
    }
  }

  if (currentMode == 1) safeNeoPixelWrite(0, 255, 0); else safeNeoPixelWrite(255, 0, 0); 

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
        Firebase.RTDB.setString(&fbdo, "/Command/new_uid", uid); Firebase.RTDB.setInt(&fbdo, "/Command/mode", 0); currentMode = 0;
        for(int i = 0; i < 3; i++) { safeNeoPixelWrite(0,0,0); delay(200); safeNeoPixelWrite(0,255,0); delay(200); }
      } else {
        if (Firebase.RTDB.getJSON(&fbdo, "/Members/" + uid)) {
        FirebaseJson &jsonNode = fbdo.jsonObject();
        FirebaseJsonData result;
        
        // Kiểm tra xem thẻ có trong danh sách không
        jsonNode.get(result, "name");
        if (!result.success) {
            for(int i = 0; i < 4; i++) { safeNeoPixelWrite(0,0,255); delay(150); safeNeoPixelWrite(0,0,0); delay(150); }
            return;
        }
        
        String currentStatus = "Vào";
        jsonNode.get(result, "lastStatus");
        if (result.success) {
            String ls = result.stringValue;
            if (ls == "Vào") currentStatus = "Ra";
        }
        
        // Lấy dữ liệu Ca làm việc từ JSON Firebase
        String activeStart = "07:00";
        String activeEnd = "17:00";
        String p_start = "", p_end = "";
        
        jsonNode.get(result, "p_start"); if (result.success) p_start = result.stringValue;
        jsonNode.get(result, "p_end"); if (result.success) p_end = result.stringValue;
        
        if (p_start != "") { activeStart = p_start; activeEnd = p_end; }
        
        // Quét danh sách Ca phụ (tempShifts) xem có ca nào đang hoạt động hôm nay không
        String today = timestamp.substring(0, 10);
        jsonNode.get(result, "tempShifts");
        if (result.success) {
            FirebaseJson tsJson;
            tsJson.setJsonData(result.stringValue);
            size_t len = tsJson.iteratorBegin();
            for (size_t i = 0; i < len; i++) {
                int type; String key, val;
                tsJson.iteratorGet(i, type, key, val);
                
                FirebaseJson item; item.setJsonData(val);
                FirebaseJsonData itemData;
                item.get(itemData, "s_from"); String s_from = itemData.stringValue;
                item.get(itemData, "s_to"); String s_to = itemData.stringValue;
                
                if (today >= s_from && today <= s_to) {
                    item.get(itemData, "s_start"); activeStart = itemData.stringValue;
                    item.get(itemData, "s_end"); activeEnd = itemData.stringValue;
                    break; // Ưu tiên ca phụ tìm thấy đầu tiên hợp lệ
                }
            }
            tsJson.iteratorEnd();
        }
        
        // Đánh giá chuyên sâu: Đi trễ hay Tăng ca
        String currentTime = timestamp.substring(11, 16);
        String eval = "Đúng giờ";
        
        if (currentStatus == "Vào") {
            if (currentTime > activeStart) eval = "Đi trễ";
        } else { // "Ra"
            if (currentTime < activeEnd) eval = "Về sớm";
            else if (currentTime > activeEnd) eval = "Tăng ca";
        }
        
        // Nạp cờ Trạng Thái Gốc
        Firebase.RTDB.setString(&fbdo, "/Members/" + uid + "/lastStatus", currentStatus);
        
        // Nạp Lịch Sử có đính kèm Đánh Giá
        String saveStatus = currentStatus + " (" + eval + ")";
        FirebaseJson jsonLog; 
        jsonLog.set("time", timestamp); 
        jsonLog.set("status", saveStatus); 
        Firebase.RTDB.setJSON(&fbdo, "/Members/" + uid + "/Attendance/" + String(now), &jsonLog);
        
        safeNeoPixelWrite(0, 255, 0); delay(500);
      } else {
        // Thẻ lạ
        for(int i = 0; i < 4; i++) { safeNeoPixelWrite(0,0,255); delay(150); safeNeoPixelWrite(0,0,0); delay(150); }
      }
      }
    }
    
    if (currentMode == 1) safeNeoPixelWrite(0, 255, 0); else safeNeoPixelWrite(255, 0, 0);
    mfrc522.PICC_HaltA(); 
  }
}

