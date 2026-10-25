#include "FS.h"
#include "SD_MMC.h"

// Khai báo chân SDMMC chính xác theo sơ đồ mạch của bạn
#define SDMMC_CLK  39 
#define SDMMC_CMD  38 
#define SDMMC_D0   40 

void setup() {
  // Lưu ý: Hãy chắc chắn góc phải dưới của Serial Monitor đang chọn 115200 baud
  Serial.begin(115200);
  delay(2000);
  
  Serial.println("\n--- BẮT ĐẦU KIỂM TRA HỆ THỐNG THẺ NHỚ (CHẾ ĐỘ SDMMC) ---");

  // Thiết lập chân cho SDMMC theo đúng cấu hình mạch
  SD_MMC.setPins(SDMMC_CLK, SDMMC_CMD, SDMMC_D0);

  Serial.print("1. Kiểm tra kết nối mạch & thẻ... ");
  // Tham số 'true' kích hoạt chế độ 1-bit (tiết kiệm chân) phù hợp với sơ đồ
  if (!SD_MMC.begin("/sdcard", true)) {
    Serial.println("❌ THẤT BẠI");
    Serial.println("   -> Lỗi ngàm kết nối SDMMC. Hãy kiểm tra lại thẻ nhớ đã cắm chặt chưa.");
    return;
  }
  Serial.println("✅ THÀNH CÔNG");

  Serial.print("2. Kiểm tra vi điều khiển của thẻ... ");
  uint8_t cardType = SD_MMC.cardType();
  if (cardType == CARD_NONE) {
    Serial.println("❌ LỖI - Không thể đọc dữ liệu.");
    return;
  }
  
  Serial.print("✅ THẺ KHỎE MẠNH (Định dạng: ");
  if (cardType == CARD_MMC) Serial.println("MMC)");
  else if (cardType == CARD_SD) Serial.println("SDSC)");
  else if (cardType == CARD_SDHC) Serial.println("SDHC/SDXC)");
  else Serial.println("Không xác định)");

  // Tính toán dung lượng (Đã sửa lỗi thiếu dấu nhân '*')
  uint64_t cardSize = SD_MMC.cardSize() / (1024 * 1024);
  uint64_t totalBytes = SD_MMC.totalBytes() / (1024 * 1024);
  uint64_t usedBytes = SD_MMC.usedBytes() / (1024 * 1024);
  
  Serial.println("\n--- THÔNG TIN DUNG LƯỢNG ---");
  Serial.printf("Kích thước vật lý : %llu MB\n", cardSize);
  Serial.printf("Dung lượng thực tế: %llu MB\n", totalBytes);
  Serial.printf("Dung lượng đã dùng: %llu MB\n", usedBytes);
  Serial.printf("Dung lượng trống  : %llu MB\n", totalBytes - usedBytes);
  Serial.println("-----------------------------------");
}

void loop() {
  delay(10000);
}
