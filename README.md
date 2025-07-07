
/*
 * PROJECT: Industrial Production Line Monitor with Dashboard Status
 * AUTHOR: [Your Name Here]
 * DATE: July 2025
 *
 * DESCRIPTION:
 * This firmware is for an ESP32-based IoT device designed to monitor a production line.
 * It uses an E18-D80NK photoelectric sensor to count products and calculates machine downtime.
 * Data is sent to the Ubidots cloud platform for real-time visualization and analysis.
 * A dedicated status variable is sent to Ubidots to power a visual indicator on the dashboard.
 *
 * FEATURES:
 * - Real-time product counting with instant updates to the cloud for live monitoring.
 * - Per-minute analysis of production rate and machine downtime.
 * - Per-minute machine status updates (Running/Stopped) for dashboard indicators.
 * - NTP time synchronization for accurate timestamps and scheduled daily operations.
 * - Automatic end-of-day (18:00) final count logging and counter reset.
 */

#include "UbidotsEsp32Mqtt.h"
#include <time.h>

// --- USER CONFIGURATION ---
const char* WIFI_SSID = "SangPC";
const char* WIFI_PASSWORD = "Ngoc@1010";
const char* UBIDOTS_TOKEN = "BBUS-xMPaxkx6gAU1ALNuereFRkO6aga1Y4"; // USE THE NEW TOKEN YOU REGENERATED

const char* DEVICE_LABEL = "may-san-xuat-01";
const char* VAR_LABEL_PRODUCT_RATE = "san-pham-moi-phut";
const char* VAR_LABEL_DOWNTIME = "thoi-gian-dung-phut";
const char* VAR_LABEL_PRODUCT_REALTIME = "tong-san-pham-realtime";
const char* VAR_LABEL_PRODUCT_FINAL = "tong-san-pham-chot-so";
const char* VAR_LABEL_STATUS = "machine-status";
const char* VAR_LABEL_SPEED_PPM = "toc-do-san-xuat-ppm"; // NEW: For accurate production speed

// --- SENSOR AND LOGIC CONFIGURATION ---
const int SENSOR_PIN = 15; // Digital pin connected to the E18-D80NK sensor's black wire
const long SEND_INTERVAL_MS = 2000; // Gửi dữ liệu tổng hợp mỗi 2 giây
const long DOWNTIME_THRESHOLD_MS = 5000; // Sau 5 giây không có sản phẩm -> coi là máy dừng
//const long SEND_INTERVAL_MS = 60000; // Send aggregated data every 1 minute
//const long DOWNTIME_THRESHOLD_MS = 20000; // After 20s with no product, assume machine has stopped

// --- TIME CONFIGURATION (NTP) ---
const char* NTP_SERVER = "pool.ntp.org";
const long  GMT_OFFSET_SEC = 7 * 3600; // GMT+7 Timezone
const int   DAYLIGHT_OFFSET_SEC = 0;   // No daylight saving

// --- GLOBAL VARIABLES ---
int productsInCurrentMinute = 0;
long dailyProductTotal = 0;
unsigned long lastProductTime_ms = 0;
bool isMachineStopped = true;
unsigned long downtimeStart_ms = 0;
float downtimeInCurrentInterval_sec = 0;
unsigned long lastSendTime = 0;
bool objectPresent = false;
bool dailyResetDone = false;

Ubidots ubidots(UBIDOTS_TOKEN);

void setup() {
  Serial.begin(115200);
  pinMode(SENSOR_PIN, INPUT_PULLUP);
  
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to WiFi...");
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println("\nWiFi connected.");

  configTime(GMT_OFFSET_SEC, DAYLIGHT_OFFSET_SEC, NTP_SERVER);
  Serial.print("Syncing NTP time...");
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) { delay(1000); Serial.print("."); }
  Serial.println("\nTime successfully synced.");
  
  ubidots.setup();
  ubidots.reconnect();
  lastProductTime_ms = millis();
  downtimeStart_ms = millis();
  lastSendTime = millis();
}

void loop() {
  if (!ubidots.connected()) { ubidots.reconnect(); }
  ubidots.loop();

  // === PRODUCT COUNTING LOGIC (POLLING METHOD) ===
  bool isProductDetected = (digitalRead(SENSOR_PIN) == LOW);
  if (isProductDetected) {
    if (!objectPresent) {
      objectPresent = true;
      productsInCurrentMinute++;
      dailyProductTotal++;
      lastProductTime_ms = millis();
      Serial.print("Product detected! Daily total: "); Serial.println(dailyProductTotal);
      
      ubidots.add(VAR_LABEL_PRODUCT_REALTIME, dailyProductTotal);
      ubidots.publish(DEVICE_LABEL);
      Serial.println("--> Real-time update sent!");
    }
  } else {
    objectPresent = false;
  }
  
  // === MACHINE DOWNTIME DETECTION LOGIC ===
  if (!isMachineStopped && (millis() - lastProductTime_ms > DOWNTIME_THRESHOLD_MS)) {
    isMachineStopped = true;
    downtimeStart_ms = lastProductTime_ms;
  }
  if (isMachineStopped && (lastProductTime_ms > downtimeStart_ms)) {
    isMachineStopped = false;
    downtimeInCurrentInterval_sec += (float)(lastProductTime_ms - downtimeStart_ms) / 1000.0;
  }
  
  // === PERIODIC AGGREGATED DATA PUBLISH LOGIC (every minute) ===
  if (millis() - lastSendTime > SEND_INTERVAL_MS) {
    float finalDowntimeToSend = downtimeInCurrentInterval_sec;
    if(isMachineStopped) {
      finalDowntimeToSend += (float)(millis() - downtimeStart_ms) / 1000.0;
      downtimeStart_ms = millis();
    }
    
    // --- ACCURATE SPEED CALCULATION ---
    float productionSpeed_PPM = 0;
    float runningTime_sec = (SEND_INTERVAL_MS / 1000.0) - finalDowntimeToSend;
    if (runningTime_sec > 1) {
      float runningTime_min = runningTime_sec / 60.0;
      productionSpeed_PPM = productsInCurrentMinute / runningTime_min;
    }
    
    Serial.println("--------------------------------");
    Serial.println("SENDING PERIODIC AGGREGATED DATA:");
    Serial.printf("- Products in the last minute: %d\n", productsInCurrentMinute);
    Serial.printf("- Downtime in the last minute: %.2f seconds\n", finalDowntimeToSend);
    Serial.printf("- Current Machine Status: %s\n", isMachineStopped ? "STOPPED" : "RUNNING");
    Serial.printf("- Calculated Production Speed: %.2f PPM\n", productionSpeed_PPM);
    

    ubidots.add(VAR_LABEL_PRODUCT_RATE, productsInCurrentMinute);
    ubidots.add(VAR_LABEL_DOWNTIME, finalDowntimeToSend);
    ubidots.add(VAR_LABEL_STATUS, isMachineStopped ? 0 : 1);
    ubidots.add(VAR_LABEL_SPEED_PPM, productionSpeed_PPM);
    ubidots.publish(DEVICE_LABEL);
    
    productsInCurrentMinute = 0;
    downtimeInCurrentInterval_sec = 0;
    lastSendTime = millis();
    Serial.println("--------------------------------");
  }

  // === END-OF-DAY SAVE AND RESET LOGIC (at 18:00) ===
  struct tm timeinfo;
  if(getLocalTime(&timeinfo)){
    if (timeinfo.tm_hour == 18 && timeinfo.tm_min == 0 && !dailyResetDone) {
      Serial.println("==============================================");
      Serial.println("IT'S 18:00 - SAVING FINAL COUNT AND RESETTING");
      
      ubidots.add(VAR_LABEL_PRODUCT_FINAL, dailyProductTotal);
      ubidots.publish(DEVICE_LABEL);
      
      dailyProductTotal = 0;
      productsInCurrentMinute = 0;
      downtimeInCurrentInterval_sec = 0;
      
      dailyResetDone = true; 
      Serial.println("COUNTERS HAVE BEEN RESET TO 0 FOR THE NEXT DAY.");
      Serial.println("==============================================");
    }
    
    if (timeinfo.tm_hour == 0 && dailyResetDone) {
      dailyResetDone = false;
    }
  }
}
