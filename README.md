#include "FS.h"
#include "SD.h"
#include "SPI.h"

#define SCK_PIN   12
#define MISO_PIN  13
#define MOSI_PIN  11
#define CS_PIN    10

SPIClass spi = SPIClass(FSPI);

void setup() {
  Serial.begin(115200);
  delay(2000); 
  
  Serial.println("\n--- BẮT ĐẦU KIỂM TRA HỆ THỐNG THẺ NHỚ ---");
  spi.begin(SCK_PIN, MISO_PIN, MOSI_PIN, CS_PIN);

  // BƯỚC 1: Kiểm tra kết nối vật lý (Dây dẫn, Mạch SPI và chân tiếp xúc thẻ)
  Serial.print("1. Kiểm tra kết nối mạch & thẻ... ");
  if (!SD.begin(CS_PIN, spi, 10000000)) {
    Serial.println("❌ THẤT BẠI");
    Serial.println("   -> Lỗi giao tiếp SPI. Vui lòng kiểm tra:");
    Serial.println("      - Đã cắm thẻ vào mạch chưa?");
    Serial.println("      - Các dây nối (VCC, GND, MISO, MOSI, SCK, CS) có bị lỏng/sai không?");
    return; 
  }
  Serial.println("✅ THÀNH CÔNG (Mạch đã được kết nối và nhận diện có thẻ)");

  // BƯỚC 2: Kiểm tra sức khỏe của vi điều khiển bên trong thẻ nhớ
  Serial.print("2. Kiểm tra vi điều khiển của thẻ... ");
  uint8_t cardType = SD.cardType();
  if (cardType == CARD_NONE) {
    Serial.println("❌ LỖI");
    Serial.println("   -> Phát hiện có thẻ nhưng không thể đọc dữ liệu. Thẻ có thể đã bị hỏng chip hoặc lỗi định dạng phần cứng nghiêm trọng.");
    return;
  }
  Serial.println("✅ THẺ KHỎE MẠNH");

  // Hiển thị loại thẻ
  Serial.print("   -> Định dạng thẻ: ");
  if (cardType == CARD_MMC) Serial.println("MMC");
  else if (cardType == CARD_SD) Serial.println("SDSC");
  else if (cardType == CARD_SDHC) Serial.println("SDHC/SDXC");
  else Serial.println("Không xác định");

  // BƯỚC 3: Đọc dung lượng
  uint64_t cardSize = SD.cardSize() / (1024 * 1024);
  uint64_t totalBytes = SD.totalBytes() / (1024 * 1024);
  uint64_t usedBytes = SD.usedBytes() / (1024 * 1024);
  uint64_t freeBytes = totalBytes - usedBytes;

  Serial.println("\n--- THÔNG TIN DUNG LƯỢNG ---");
  Serial.printf("Kích thước vật lý : %llu MB\n", cardSize);
  Serial.printf("Dung lượng thực tế: %llu MB\n", totalBytes);
  Serial.printf("Dung lượng đã dùng: %llu MB\n", usedBytes);
  Serial.printf("Dung lượng trống  : %llu MB\n", freeBytes);
  Serial.println("-----------------------------------");
}

void loop() {
  delay(10000); 
}
