# ใบงานที่ 5.1: การเชื่อมต่อ Wi-Fi และการค้นหาสัญญาณรอบข้าง (Wi-Fi Connection and Scanning)

## 0. กล่าวนำ (Introduction)
ใบงานในสัปดาห์นี้ จะแสดง console log แบบ forensic  โดยละเอียด จะทำให้นักศึกษาเห็นทั้งฟังก์ชันที่เรียกใช้งานและค่าที่ return กลับมา
เพื่อให้เกิดประสบการณ์ในการวิเคราะห์ปัญหาหน้างานจริงในการใช้ Wi-Fi บน ESP32 ที่รันบน framework ESP-IDF ได้

## 1. วัตถุประสงค์ (Objectives)
1. เรียนรู้และเข้าใจกระบวนการทำงานเบื้องต้นของการใช้งาน ESP32 ในโหมด Station (STA Mode) ผ่านเฟรมเวิร์ก ESP-IDF
2. เรียนรู้การตั้งค่าและใช้งาน API สำหรับสแกนสัญญาณ Wi-Fi (Scan Phase) ในรูปแบบต่าง ๆ ผ่าน `esp_wifi.h`
3. สามารถเขียนโปรแกรมค้นหา Access Point (AP) แบบทั่วไป (General Scan), แบบระบุช่องความถี่ (Channel-Specific Scan) และแบบระบุชื่อเครือข่ายเจาะจง (Targeted SSID Scan)
4. อ่านค่าและแสดงผลรายละเอียดของ AP เช่น SSID, MAC Address (BSSID), RSSI, Channel และ Encryption Type ผ่าน Serial Monitor (ESP-IDF Console) ได้อย่างถูกต้อง

---

## 2. อุปกรณ์และซอฟต์แวร์ที่ใช้ในการทดลอง (Equipment & Tools)
1. บอร์ดไมโครคอนโทรลเลอร์ ESP32 (เช่น ESP32 DevKit V1) จำนวน 1 บอร์ด
2. สายเชื่อมต่อ Micro-USB หรือ USB-C (ตามรุ่นของบอร์ด) จำนวน 1 เส้น
3. คอมพิวเตอร์ที่ติดตั้งโปรแกรม IDE เช่น VS Code พร้อมทั้ง ESP-IDF (อาจจะติดตั้งบนเครื่องหรือบน Docker ก็ได้)

---

## 3. ความรู้พื้นฐานที่เกี่ยวข้อง (Theoretical Background - ESP-IDF Framework)

### 3.1 การเริ่มต้น Wi-Fi Stack บน ESP-IDF
ในเฟรมเวิร์ก ESP-IDF การทำงานของ Wi-Fi จำเป็นต้องเริ่มต้นโมดูลพื้นฐานของระบบก่อนตามลำดับ ได้แก่ NVS Flash (สำหรับจัดเก็บ Configuration), Network Interface (esp_netif) และ Event Loop ดังตัวอย่างต่อไปนี้

```c
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "esp_wifi.h"

void init_wifi_sta(void) {
    // 1. Initialize NVS Flash
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 2. Initialize Network Interface & Event Loop
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    // 3. Initialize Wi-Fi Driver in Station Mode
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_start());
}
```

### 3.2 กระบวนการสแกนสัญญาณด้วย `wifi_scan_config_t`
ในมาตรฐาน IEEE 802.11 กระบวนการ **Scan Phase** เป็นขั้นตอนแรกที่ ESP32 จะสำรวจสภาพแวดล้อมทางคลื่นวิทยุ 2.4GHz โดย ESP-IDF มีโครงสร้างข้อมูล `wifi_scan_config_t` สำหรับกำหนดเงื่อนไขในการสแกน

```c
typedef struct {
    uint8_t *ssid;               // Target SSID (NULL = scan all SSIDs)
    uint8_t *bssid;              // Target MAC address (NULL = scan all BSSIDs)
    uint8_t channel;             // Target channel (0 = scan all channels 1-13)
    bool show_hidden;            // true = include hidden SSIDs in scan result
    wifi_scan_type_t scan_type;  // WIFI_SCAN_TYPE_ACTIVE or WIFI_SCAN_TYPE_PASSIVE
    wifi_scan_time_t scan_time;  // Scan duration per channel
} wifi_scan_config_t;
```

#### แผนผังโครงสร้างข้อมูล (Class Diagram of `wifi_scan_config_t`)

```mermaid
classDiagram
    class wifi_scan_config_t {
        +uint8_t* ssid
        +uint8_t* bssid
        +uint8_t channel
        +bool show_hidden
        +wifi_scan_type_t scan_type
        +wifi_scan_time_t scan_time
    }
    class wifi_scan_type_t {
        <<enumeration>>
        WIFI_SCAN_TYPE_ACTIVE
        WIFI_SCAN_TYPE_PASSIVE
    }
    class wifi_scan_time_t {
        +uint32_t active_min_ms
        +uint32_t active_max_ms
        +uint32_t passive_ms
    }

    wifi_scan_config_t "1" *-- "1" wifi_scan_type_t : uses
    wifi_scan_config_t "1" *-- "1" wifi_scan_time_t : configures
```

**คำสั่งสั่งสแกน**  `esp_wifi_scan_start(&scan_config, true)` โดยหากใส่พารามิเตอร์ตัวหลังเป็น `true` จะเป็นการสแกนแบบ Synchronous (รอจนกว่าการสแกนจะเสร็จสมบูรณ์)

### 3.3 การดึงรายละเอียดของ Access Point (`wifi_ap_record_t`)
เมื่อสแกนเสร็จสิ้น สามารถอ่านจำนวน AP ที่พบด้วย `esp_wifi_scan_get_ap_num()` และอ่านรายละเอียดโครงสร้าง `wifi_ap_record_t` ด้วย `esp_wifi_scan_get_ap_records()` ซึ่งประกอบด้วยสมาชิกสำคัญ ดังต่อไปนี้

#### แผนผังโครงสร้างข้อมูล (Class Diagram of `wifi_ap_record_t`)

```mermaid
classDiagram
    class wifi_ap_record_t {
        +uint8_t[33] ssid
        +uint8_t[6] bssid
        +int8_t rssi
        +uint8_t primary
        +wifi_auth_mode_t authmode
    }
    class wifi_auth_mode_t {
        <<enumeration>>
        WIFI_AUTH_OPEN
        WIFI_AUTH_WEP
        WIFI_AUTH_WPA_PSK
        WIFI_AUTH_WPA2_PSK
        WIFI_AUTH_WPA3_PSK
    }

    wifi_ap_record_t "1" *-- "1" wifi_auth_mode_t : specifies
```

| ตัวแปรสมาชิก | รายละเอียด                 | ข้อมูล                          |
| ------------ | -------------------------- | ------------------------------- |
| `ssid`       | ชื่อเครือข่าย Wi-Fi        | (อาร์เรย์ uint8_t ขนาด 33 ไบต์) |
| `bssid`      | หมายเลข MAC Address ของ AP | (อาร์เรย์ uint8_t ขนาด 6 ไบต์)  |
| `rssi`       | ความแรงของสัญญาณวิทยุ      | (หน่วย dBm)                     |
| `primary`    | ช่องความถี่วิทยุหลัก       | (Channel 1-13)                  |
| `authmode`   | ประเภทการเข้ารหัสความปลอดภัย | (`wifi_auth_mode_t` เช่น `WIFI_AUTH_OPEN`, `WIFI_AUTH_WPA2_PSK`, `WIFI_AUTH_WPA3_PSK`)|

---

## 4. ขั้นตอนและโปรแกรมทดสอบการทดลอง (Experimental Procedures)

ในใบงานนี้ โปรแกรมจะทำการสแกนค้นหาสัญญาณ Wi-Fi ใน 4 กรณีหลักแบบ Forensic Logging และจะ**หยุดการทำงานในเฟสที่ 1 (Scan Phase)** เท่านั้น โดยแสดงผลสรุปผ่าน ESP-IDF Serial Console

### 5.1.1 การสแกนหา AP ทั่วไป (General AP Scan)
เป็นการสแกนครอบคลุมทุกช่องความถี่ (Channel 1-13) และทุก SSID (`.ssid = NULL`, `.channel = 0`) เพื่อสำรวจเครือข่าย Wi-Fi ทั้งหมดที่อยู่บริเวณรอบข้าง

### 5.1.2 การสแกนหา AP โดยกำหนดช่วงความถี่ (Channel-Specific Scan)
เป็นการระบุช่องความถี่เจาะจง (เช่น `.channel = 1`) เพื่อลดเวลาที่ใช้ในการสแกน เหมาะสำหรับกรณีที่ทราบล่วงหน้าว่า AP เป้าหมายทำงานอยู่ใน Channel ใด

### 5.1.3 การสแกนหา AP โดยกำหนดชื่อที่เจาะจงกรณีมีอยู่จริง (Targeted SSID Scan - Existing)
เป็นการระบุชื่อ SSID ที่ต้องการสแกนโดยเฉพาะ (`.ssid = (uint8_t *)"Target_SSID"`) นำชื่อ SSID ที่พบจากการสแกนทั่วไปในข้อ 5.1.1 มาทดสอบสแกนแบบเจาะจง

### 5.1.4 การสแกนหา AP โดยกำหนดชื่อที่เจาะจงกรณีไม่มีอยู่จริง (Targeted SSID Scan - Non-Existent)
เป็นการระบุชื่อ SSID สมมุติที่ไม่มีอยู่จริง (ในใบงานนี้คือ `"NON_EXISTENT_AP_9999"`) เพื่อสังเกตการตอบสนองและ Error Handling ของระบบ (พบ 0 AP)

---

## 5. ซอร์สโค้ดการทดลอง (Complete ESP-IDF Source Code - `main.c`)

ให้นักศึกษานำซอร์สโค้ด C ต่อไปนี้ไปวางในไฟล์ `main/main.c` ของโปรเจกต์ ESP-IDF ทำการ Build และ Flash ลงบอร์ด ESP32 จากนั้นเปิด ESP-IDF Monitor (Baud Rate `115200`) เพื่อสังเกตผลการทำงาน

```c
#include "esp_event.h"
#include "esp_log.h"
#include "esp_netif.h"
#include "esp_system.h"
#include "esp_timer.h"
#include "esp_wifi.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "nvs_flash.h"
#include <stdio.h>
#include <string.h>

static const char *TAG = "LAB_WIFI_SCAN";

// Convert wifi_auth_mode_t enum to readable string
static const char *get_auth_mode_name(wifi_auth_mode_t authmode) {
  switch (authmode) {
  case WIFI_AUTH_OPEN:
    return "OPEN (No Password)";
  case WIFI_AUTH_WEP:
    return "WEP";
  case WIFI_AUTH_WPA_PSK:
    return "WPA_PSK";
  case WIFI_AUTH_WPA2_PSK:
    return "WPA2_PSK";
  case WIFI_AUTH_WPA_WPA2_PSK:
    return "WPA_WPA2_PSK";
  case WIFI_AUTH_WPA2_ENTERPRISE:
    return "WPA2_ENTERPRISE";
  case WIFI_AUTH_WPA3_PSK:
    return "WPA3_PSK";
  case WIFI_AUTH_WPA2_WPA3_PSK:
    return "WPA2_WPA3_PSK";
  case WIFI_AUTH_WAPI_PSK:
    return "WAPI_PSK";
  default:
    return "UNKNOWN";
  }
}

// Perform Wi-Fi scan and display detailed AP records
static void perform_wifi_scan(wifi_scan_config_t *scan_config,
                              const char *test_title, char *found_first_ssid,
                              size_t max_ssid_len) {
  ESP_LOGI(
      TAG,
      "------------------------------------------------------------------");
  ESP_LOGI(TAG, ">>> %s", test_title);
  ESP_LOGI(
      TAG,
      "------------------------------------------------------------------");

  ESP_LOGI(TAG,
           "[FORENSIC]: Call esp_wifi_scan_start(scan_config, block=true)");
  int64_t start_time = esp_timer_get_time();
  esp_err_t err = esp_wifi_scan_start(scan_config, true);
  int64_t duration_ms = (esp_timer_get_time() - start_time) / 1000;
  ESP_LOGI(TAG,
           "[FORENSIC]: esp_wifi_scan_start() returned %s (0x%x) [Duration: "
           "%lld ms]",
           esp_err_to_name(err), err, duration_ms);

  if (err != ESP_OK) {
    ESP_LOGE(TAG, "[STATUS]: Scan failed (Error: %s / Code: 0x%x)",
             esp_err_to_name(err), err);
    return;
  }

  uint16_t ap_count = 0;
  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_scan_get_ap_num(&ap_count)");
  esp_err_t err_ap_num = esp_wifi_scan_get_ap_num(&ap_count);
  ESP_LOGI(
      TAG,
      "[FORENSIC]: esp_wifi_scan_get_ap_num() returned %s (0x%x), ap_count=%u",
      esp_err_to_name(err_ap_num), err_ap_num, ap_count);

  ESP_LOGI(TAG, "[STATUS]: Scan SUCCESS");
  ESP_LOGI(TAG, "[AP COUNT]: %u network(s) found", ap_count);

  if (ap_count == 0) {
    ESP_LOGW(TAG, "[NOTE]: No Access Point found matching the criteria.");
  } else {
    wifi_ap_record_t *ap_info =
        (wifi_ap_record_t *)malloc(sizeof(wifi_ap_record_t) * ap_count);
    if (ap_info == NULL) {
      ESP_LOGE(TAG, "Failed to allocate memory for AP scan records");
      return;
    }

    uint16_t number = ap_count;
    ESP_LOGI(TAG,
             "[FORENSIC]: Call esp_wifi_scan_get_ap_records(&number, ap_info)");
    esp_err_t err_ap_rec = esp_wifi_scan_get_ap_records(&number, ap_info);
    ESP_LOGI(TAG,
             "[FORENSIC]: esp_wifi_scan_get_ap_records() returned %s (0x%x), "
             "records=%u",
             esp_err_to_name(err_ap_rec), err_ap_rec, number);
    ESP_ERROR_CHECK(err_ap_rec);

    // Save first found SSID for targeted scan test
    if (found_first_ssid != NULL && number > 0) {
      snprintf(found_first_ssid, max_ssid_len, "%s", (char *)ap_info[0].ssid);
    }

    printf("\n-----------------------------------------------------------------"
           "---------------------------------\n");
    printf("%-4s | %-24s | %-17s | %-6s | %-4s | %-20s\n", "No.", "SSID",
           "MAC Address (BSSID)", "RSSI", "Chan", "Encryption Type");
    printf("-------------------------------------------------------------------"
           "-------------------------------\n");

    for (int i = 0; i < number; i++) {
      char bssid_str[18];
      snprintf(bssid_str, sizeof(bssid_str), "%02X:%02X:%02X:%02X:%02X:%02X",
               ap_info[i].bssid[0], ap_info[i].bssid[1], ap_info[i].bssid[2],
               ap_info[i].bssid[3], ap_info[i].bssid[4], ap_info[i].bssid[5]);

      const char *ssid_display = (strlen((char *)ap_info[i].ssid) > 0)
                                     ? (char *)ap_info[i].ssid
                                     : "<Hidden SSID>";

      printf("%-4d | %-24s | %-17s | %-4d dBm | %-4d | %-20s\n", i + 1,
             ssid_display, bssid_str, ap_info[i].rssi, ap_info[i].primary,
             get_auth_mode_name(ap_info[i].authmode));
    }
    printf("-------------------------------------------------------------------"
           "-------------------------------\n\n");

    free(ap_info);
  }
}

void app_main(void) {
  // 1. Initialize NVS Flash (Required for Wi-Fi stack)
  ESP_LOGI(TAG, "[FORENSIC]: Call nvs_flash_init()");
  esp_err_t ret = nvs_flash_init();
  ESP_LOGI(TAG, "[FORENSIC]: nvs_flash_init() returned %s (0x%x)",
           esp_err_to_name(ret), ret);
  if (ret == ESP_ERR_NVS_NO_FREE_PAGES ||
      ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_LOGI(TAG, "[FORENSIC]: Call nvs_flash_erase()");
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
    ESP_LOGI(TAG, "[FORENSIC]: nvs_flash_init() retry returned %s (0x%x)",
             esp_err_to_name(ret), ret);
  }
  ESP_ERROR_CHECK(ret);

  // 2. Initialize Network Interface and Default Event Loop
  ESP_LOGI(TAG, "[FORENSIC]: Call esp_netif_init()");
  esp_err_t err_netif = esp_netif_init();
  ESP_LOGI(TAG, "[FORENSIC]: esp_netif_init() returned %s (0x%x)",
           esp_err_to_name(err_netif), err_netif);
  ESP_ERROR_CHECK(err_netif);

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_event_loop_create_default()");
  esp_err_t err_event = esp_event_loop_create_default();
  ESP_LOGI(TAG,
           "[FORENSIC]: esp_event_loop_create_default() returned %s (0x%x)",
           esp_err_to_name(err_event), err_event);
  ESP_ERROR_CHECK(err_event);

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_netif_create_default_wifi_sta()");
  esp_netif_t *sta_netif = esp_netif_create_default_wifi_sta();
  ESP_LOGI(
      TAG,
      "[FORENSIC]: esp_netif_create_default_wifi_sta() returned pointer %p",
      sta_netif);

  // 3. Initialize Wi-Fi Driver in Station Mode
  wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_init(&cfg)");
  esp_err_t err_wifi_init = esp_wifi_init(&cfg);
  ESP_LOGI(TAG, "[FORENSIC]: esp_wifi_init() returned %s (0x%x)",
           esp_err_to_name(err_wifi_init), err_wifi_init);
  ESP_ERROR_CHECK(err_wifi_init);

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_set_mode(WIFI_MODE_STA)");
  esp_err_t err_mode = esp_wifi_set_mode(WIFI_MODE_STA);
  ESP_LOGI(TAG, "[FORENSIC]: esp_wifi_set_mode() returned %s (0x%x)",
           esp_err_to_name(err_mode), err_mode);
  ESP_ERROR_CHECK(err_mode);

  ESP_LOGI(TAG, "[FORENSIC]: Call esp_wifi_start()");
  esp_err_t err_start = esp_wifi_start();
  ESP_LOGI(TAG, "[FORENSIC]: esp_wifi_start() returned %s (0x%x)",
           esp_err_to_name(err_start), err_start);
  ESP_ERROR_CHECK(err_start);

  ESP_LOGI(
      TAG,
      "==================================================================");
  ESP_LOGI(TAG,
           "  Lab 5.1: Wi-Fi Connection and Scanning Phase (ESP-IDF Forensic)");
  ESP_LOGI(
      TAG,
      "==================================================================");

  char first_found_ssid[33] = "";

  // ------------------------------------------------------------------
  // 5.1.1 General AP Scan (All Channels & All SSIDs)
  // ------------------------------------------------------------------
  wifi_scan_config_t scan_config_all = {.ssid = NULL,
                                        .bssid = NULL,
                                        .channel =
                                            0, // 0 = Scan all channels (1-13)
                                        .show_hidden = true,
                                        .scan_type = WIFI_SCAN_TYPE_ACTIVE};
  perform_wifi_scan(&scan_config_all,
                    "Experiment 5.1.1: General AP Scan (All Channels)",
                    first_found_ssid, sizeof(first_found_ssid));

  vTaskDelay(pdMS_TO_TICKS(1000));

  // ------------------------------------------------------------------
  // 5.1.2 Channel-Specific Scan
  // ------------------------------------------------------------------
  uint8_t target_channel = 1; // Scan specifically on Channel 1
  char title_buf[128];
  snprintf(title_buf, sizeof(title_buf),
           "Experiment 5.1.2: Channel-Specific Scan (Channel %d)",
           target_channel);

  wifi_scan_config_t scan_config_chan = {
      .ssid = NULL,
      .bssid = NULL,
      .channel = target_channel, // Scan specified channel only
      .show_hidden = true,
      .scan_type = WIFI_SCAN_TYPE_ACTIVE};
  perform_wifi_scan(&scan_config_chan, title_buf, NULL, 0);

  vTaskDelay(pdMS_TO_TICKS(1000));

  // ------------------------------------------------------------------
  // 5.1.3 Targeted SSID Scan - Existing
  // ------------------------------------------------------------------

  // 5.1.3 Existing SSID (Using first found SSID from 5.1.1)
  const char *target_exist_ssid =
      (strlen(first_found_ssid) > 0) ? first_found_ssid : "WiFi-Test-Guest";
  snprintf(title_buf, sizeof(title_buf),
           "Experiment 5.1.3: Targeted SSID Scan - Existing (\"%s\")",
           target_exist_ssid);

  wifi_scan_config_t scan_config_exist = {.ssid = (uint8_t *)target_exist_ssid,
                                          .bssid = NULL,
                                          .channel = 0,
                                          .show_hidden = true,
                                          .scan_type = WIFI_SCAN_TYPE_ACTIVE};
  perform_wifi_scan(&scan_config_exist, title_buf, NULL, 0);

  vTaskDelay(pdMS_TO_TICKS(1000));

  // ------------------------------------------------------------------
  // 5.1.4 Targeted SSID Scan - Non-Existent
  // ------------------------------------------------------------------
  const char *dummy_ssid = "NON_EXISTENT_AP_9999";
  snprintf(title_buf, sizeof(title_buf),
           "Experiment 5.1.4: Targeted SSID Scan - Non-Existent (\"%s\")",
           dummy_ssid);

  wifi_scan_config_t scan_config_not_exist = {.ssid = (uint8_t *)dummy_ssid,
                                              .bssid = NULL,
                                              .channel = 0,
                                              .show_hidden = true,
                                              .scan_type =
                                                  WIFI_SCAN_TYPE_ACTIVE};
  perform_wifi_scan(&scan_config_not_exist, title_buf, NULL, 0);

  // Complete Scan Phase
  ESP_LOGI(
      TAG,
      "==================================================================");
  ESP_LOGI(TAG, "  [Phase 1 Completed: Wi-Fi Scan Finished]");
  ESP_LOGI(TAG,
           "  Program stopped after scanning. Auth/Assoc Phase not started.");
  ESP_LOGI(
      TAG,
      "==================================================================");
}
```

---

## 6. ตารางบันทึกผลการทดลอง (Experiment Results)

ให้นักศึกษาบันทึกผลลัพธ์จากการสังเกตใน Serial Console ลงในตารางต่อไปนี้:

### 6.1 ตารางสรุปเปรียบเทียบการสแกนทั้ง 4 กรณี

| ข้อการทดลอง | เงื่อนไขการสแกน | สถานะ (Success/Error Code) | จำนวน AP ที่พบ (เครือข่าย) | เวลาที่ใช้ในการสแกน (ms) |
| :---: | :--- | :---: | :---: | :---: |
| **5.1.1** | สแกนทั่วไปทุก Channel | ESP_OK (0x0) | 11 | 2499 |
| **5.1.2** | กำหนดสแกนเฉพาะ Channel 1 | ESP_OK (0x0) | 7 | 200 |
| **5.1.3** | กำหนดสแกน SSID ที่มีจริง | ESP_OK (0x0) | 3 | 2499 |
| **5.1.4** | กำหนดสแกน SSID ที่ไม่มีจริง | ESP_OK (0x0) | 0 | 2498 |

### 6.2 ตารางรายละเอียด AP ที่พบจากการสแกนทั่วไป (ข้อ 5.1.1)

| ลำดับ | ชื่อเครือข่าย (SSID) | MAC Address (BSSID) | ความแรงสัญญาณ (RSSI: dBm) | ช่องความถี่ (Channel) | ประเภทการเข้ารหัส (Encryption Type) |
| :---: | :--- | :--- | :---: | :---: | :--- |
| 1 | KMITL-WIFI | 78:17:BE:C0:7D:A1 | -53 dBm | 1 | OPEN (No Password) |
| 2 | KMITL-Legacy | 78:17:BE:C0:7D:A0 | -55 dBm | 1 | WPA2_ENTERPRISE |
| 3 | KMITL-IoT | 78:17:BE:C0:7D:A2 | -55 dBm | 1 | WPA2_PSK |
| 4 | <Hidden SSID> | 6A:57:07:6F:73:08 | -58 dBm | 6 | WPA2_PSK |
| 5 | FUB | EA:67:85:97:BF:39 | -60 dBm | 10 | WPA2_PSK |

---

## 7. คำถามท้ายการทดลอง (Post-Lab Questions)

1. การกำหนดค่าในโครงสร้าง `wifi_scan_config_t` สำหรับสแกนเจาะจงเฉพาะช่องความถี่ (ข้อ 5.1.2) ช่วยลดเวลาในการสแกนเมื่อเทียบกับการสแกนทุกช่องความถี่ (ข้อ 5.1.1) อย่างไร และมีข้อจำกัดอย่างไร?
```
- การลดเวลา: ช่วยลดเวลาลงได้อย่างมหาศาล (จากตัวอย่างใน Log ลดจาก 2499 ms เหลือเพียง 200 ms) เนื่องจากตัวรับสัญญาณของ ESP32 ไม่ต้องเสียเวลาสลับความถี่ (Channel Dwell Time) ไปรอฟังกดสัญญาณ Beacon Frame ครบทุกช่องสัญญาณ (Channel 1-13) แต่จะรับฟังสัญญาณแค่บนช่องความถี่ที่ระบุเท่านั้น
- ข้อจำกัด: หากตัวแปร AP หรือสถานีเครือข่ายเป้าหมายมีการขยับเปลี่ยนช่องความถี่ (เช่น Router เปิดโหมด Auto Channel) หรือระบบจำเป็นต้องค้นหาตัวเลือกในสภาพแวดล้อมใหม่ทั้งหมด การล็อก Channel จะทำให้ไม่สามารถตรวจพบเครือข่ายรอบข้างที่อยู่คนละช่องความถี่ได้เลย
```
2. เมื่อสังเกตผล Forensic Log ในข้อ 5.1.4 (สแกนหา SSID ที่ไม่มีอยู่จริง) ฟังก์ชัน `esp_wifi_scan_start()`, `esp_wifi_scan_get_ap_num()` และ `esp_wifi_scan_get_ap_records()` ส่งคืนค่าอย่างไร?
```
- esp_wifi_scan_start(): ส่งคืนค่า ESP_OK (0x0) (กระบวนการทำงานของฟังก์ชันสำเร็จ สมบูรณ์ตามสั่ง)
- esp_wifi_scan_get_ap_num(): ส่งคืนค่า ESP_OK (0x0) และตัวแปรเก็บจำนวน ap_count จะเปลี่ยนเป็นค่า 0
- esp_wifi_scan_get_ap_records(): จากเงื่อนไขซอร์สโค้ดที่มีการดัก if (ap_count == 0) ทำให้โปรแกรมข้ามการเรียกใช้งานฟังก์ชันนี้และไม่มีการจอง Memory ให้ตัวแปร ap_info แต่ระบบจะวิ่งไปทำงานในส่วนการโยน Log Warning [NOTE]: No Access Point found matching the criteria. แทน
```
3. ค่าระดับความแรงสัญญาณ (RSSI) ที่แสดงเป็นตัวเลขติดลบ (เช่น -45 dBm กับ -80 dBm) ค่าใดแสดงถึงสัญญาณที่มีความแรงและความเสถียรมากกว่ากัน?
```
ค่า -45 dBm แสดงถึงสัญญาณที่มีความแรงและความเสถียรกว่ามาก -80 dBm
เนื่องจากค่า RSSI เป็นลบ (ค่าเข้าใกล้ 0 มากกว่าจะมีความเข้มของพลังงานคลื่นวิทยุที่ฝั่งรับสูงกว่า)
ขณะที่ค่า -80 dBm เป็นระดับสัญญาณที่ค่อนข้างอ่อนมาก (ใกล้เคียงกับ Noise Floor) ซึ่งอาจทำให้เกิดความล่าช้า 
ข้อมูลตกหล่น (Packet Loss) หรือสัญญาณหลุดระหว่างใช้งานได้ง่าย
```
4. เหตุใดการดึงค่า `authmode` (`wifi_auth_mode_t`) จากโครงสร้าง `wifi_ap_record_t` จึงมีความสำคัญต่อการเตรียมการในเฟสถัดไป (Authentication & Association Phase)?
```
เพราะค่า authmode จะระบุโครงสร้างความปลอดภัยของเป้าหมายอย่างชัดเจน (เช่น OPEN, WPA2_PSK, หรือ WPA2_ENTERPRISE) ข้อมูลนี้จำเป็นต่อการกำหนดโครงสร้าง wifi_config_t ในตัวแปร sta.password ได้ถูกต้อง หากไม่ทราบค่าล่วงหน้าหรือกำหนดตั้งค่าของ Authentication ผิดพลาด (เช่น พยายามใช้การเชื่อมต่อแบบ WPA2 เกาะกับเครือข่ายที่เป็น OPEN) สแต็กของ Wi-Fi Driver จะปฏิเสธการทำ Handshake และสิ้นสุดการเชื่อมต่อลงด้วยความล้มเหลวทันที
```

```
Running idf_monitor in directory D:\Projects\Week-05-Wi-Fi-ESP32\wi-fi
Executing "C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe C:\esp\v6.0.2\esp-idf\tools/idf_monitor.py -p COM6 -b 115200 --toolchain-prefix xtensa-esp32-elf- --target esp32 --revision 0 D:\Projects\Week-05-Wi-Fi-ESP32\wi-fi\build\wi-fi.elf D:\Projects\Week-05-Wi-Fi-ESP32\wi-fi\build\bootloader\bootloader.elf --force-color -m 'C:\Espressif\tools\python\v6.0.2\venv\Scripts\python.exe' 'C:\esp\v6.0.2\esp-idf\tools\idf.py'"...
--- Warning: GDB cannot open serial ports accessed as COMx
--- Using \\.\COM6 instead...
--- esp-idf-monitor 1.9.0 on \\.\COM6 115200
--- Quit: Ctrl+] | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H
3 (SPI_FAST_FLASH_BOOT)%�S      ���� Partition Table:
I (49) boot: ## Label            Usage          Type ST�ets Jul 29 2019 12:21:46

rst:0x1 (POWERON_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
configsip: 0, SPIWP:0xee
clk_drv:0x00,q_drv:0x00,d_drv:0x00,cs0_drv:0x00,hd_drv:0x00,wp_drv:0x00
mode:DIO, clock div:2
load:0x3fff0040,len:6272
load:0x40078000,len:15820
load:0x40080400,len:3920
--- 0x40080400: _invalid_pc_placeholder at C:/esp/v6.0.2/esp-idf/components/xtensa/xtensa_vectors.S:2259
entry 0x40080644
--- 0x40080644: call_start_cpu0 at C:/esp/v6.0.2/esp-idf/components/bootloader/subproject/main/bootloader_start.c:27
I (27) boot: ESP-IDF v6.0.2 2nd stage bootloader
I (27) boot: compile time Aug  3 2026 10:57:32
I (28) boot: Multicore bootloader
I (29) boot: chip revision: v3.1
I (32) boot.esp32: SPI Speed      : 40MHz
I (35) boot.esp32: SPI Mode       : DIO
I (39) boot.esp32: SPI Flash Size : 2MB
I (42) boot: Enabling RNG early entropy source...
I (47) boot: Partition Table:
I (49) boot: ## Label            Usage          Type ST Offset   Length
I (56) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (62) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (69) boot:  2 factory          factory app      00 00 00010000 00100000
I (75) boot: End of partition table
I (79) esp_image: segment 0: paddr=00010020 vaddr=3f400020 size=1a40ch (107532) map
I (124) esp_image: segment 1: paddr=0002a434 vaddr=3ffb0000 size=04528h ( 17704) load
I (132) esp_image: segment 2: paddr=0002e964 vaddr=40080000 size=016b4h (  5812) load
I (134) esp_image: segment 3: paddr=00030020 vaddr=400d0020 size=85fa4h (548772) map
I (331) esp_image: segment 4: paddr=000b5fcc vaddr=400816b4 size=13f54h ( 81748) load
I (365) esp_image: segment 5: paddr=000c9f28 vaddr=50000000 size=00028h (    40) load
I (376) boot: Loaded app from partition at offset 0x10000
I (376) boot: Disabling RNG early entropy source...
I (387) cpu_start: Multicore app
I (395) cpu_start: GPIO 3 and 1 are used as console UART I/O pins
I (395) cpu_start: Pro cpu start user code
I (395) cpu_start: cpu freq: 160000000 Hz
I (397) app_init: Application information:
I (401) app_init: Project name:     wi-fi
I (405) app_init: App version:      37de524-dirty
I (409) app_init: Compile time:     Aug  3 2026 10:56:48
I (414) app_init: ELF file SHA256:  556a3aa54...
I (419) app_init: ESP-IDF:          v6.0.2
I (422) efuse_init: Min chip rev:     v0.0
I (426) efuse_init: Max chip rev:     v3.99 
I (430) efuse_init: Chip rev:         v3.1
I (434) heap_init: Initializing. RAM available for dynamic allocation:
I (440) heap_init: At 3FFAE6E0 len 00001920 (6 KiB): DRAM
I (445) heap_init: At 3FFB8A20 len 000275E0 (157 KiB): DRAM
I (451) heap_init: At 3FFE0440 len 00003AE0 (14 KiB): D/IRAM
I (456) heap_init: At 3FFE4350 len 0001BCB0 (111 KiB): D/IRAM
I (462) heap_init: At 40095608 len 0000A9F8 (42 KiB): IRAM
I (468) spi_flash: detected chip: generic
I (470) spi_flash: flash io: dio
W (473) spi_flash: Detected size(4096k) larger than the size in the binary image header(2048k). Using the size in the binary image header.
I (487) main_task: Started on CPU0
I (487) main_task: Calling app_main()
I (487) LAB_WIFI_SCAN: [FORENSIC]: Call nvs_flash_init()
I (517) LAB_WIFI_SCAN: [FORENSIC]: nvs_flash_init() returned ESP_OK (0x0)
I (517) LAB_WIFI_SCAN: [FORENSIC]: Call esp_netif_init()
I (517) LAB_WIFI_SCAN: [FORENSIC]: esp_netif_init() returned ESP_OK (0x0)
I (527) LAB_WIFI_SCAN: [FORENSIC]: Call esp_event_loop_create_default()
I (527) LAB_WIFI_SCAN: [FORENSIC]: esp_event_loop_create_default() returned ESP_OK (0x0)
I (537) LAB_WIFI_SCAN: [FORENSIC]: Call esp_netif_create_default_wifi_sta()
I (547) LAB_WIFI_SCAN: [FORENSIC]: esp_netif_create_default_wifi_sta() returned pointer 0x3ffbdd78
I (557) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_init(&cfg)
I (567) wifi:wifi driver task: 3ffc0464, prio:23, stack:6656, core=0
I (577) wifi:wifi firmware version: 00ad238
I (577) wifi:wifi certification version: v7.0
I (577) wifi:config NVS flash: enabled
I (577) wifi:config nano formatting: disabled
I (587) wifi:Init data frame dynamic rx buffer num: 32
I (587) wifi:Init static rx mgmt buffer num: 5
I (597) wifi:Init management short buffer num: 32
I (597) wifi:Init dynamic tx buffer num: 32
I (597) wifi:Init static rx buffer size: 1600
I (607) wifi:Init static rx buffer num: 10
I (607) wifi:Init dynamic rx buffer num: 32
I (617) wifi_init: rx ba win: 6
I (617) wifi_init: accept mbox: 6
I (617) wifi_init: tcpip mbox: 32
I (627) wifi_init: udp mbox: 6
I (627) wifi_init: tcp mbox: 6
I (627) wifi_init: tcp tx win: 5760
I (627) wifi_init: tcp rx win: 5760
I (637) wifi_init: tcp mss: 1440
I (637) wifi_init: WiFi IRAM OP enabled
I (637) wifi_init: WiFi RX IRAM OP enabled
I (647) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_init() returned ESP_OK (0x0)
I (647) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_set_mode(WIFI_MODE_STA)
I (657) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_set_mode() returned ESP_OK (0x0)
I (667) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_start()
I (667) phy_init: phy_version 4863,a3a4459,Oct 28 2025,14:30:06
I (757) wifi:mode : sta (14:33:5c:0d:d5:4c)
I (757) wifi:enable tsf
I (757) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_start() returned ESP_OK (0x0)
I (757) LAB_WIFI_SCAN: ==================================================================
I (767) LAB_WIFI_SCAN:   Lab 5.1: Wi-Fi Connection and Scanning Phase (ESP-IDF Forensic)
I (777) LAB_WIFI_SCAN: ==================================================================
I (787) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (787) LAB_WIFI_SCAN: >>> Experiment 5.1.1: General AP Scan (All Channels)
I (797) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (807) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_start(scan_config, block=true)
I (3317) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_start() returned ESP_OK (0x0) [Duration: 2499 ms]
I (3317) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_get_ap_num(&ap_count)
I (3317) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_get_ap_num() returned ESP_OK (0x0), ap_count=11
I (3327) LAB_WIFI_SCAN: [STATUS]: Scan SUCCESS
I (3327) LAB_WIFI_SCAN: [AP COUNT]: 11 network(s) found
I (3337) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_get_ap_records(&number, ap_info)
I (3347) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_get_ap_records() returned ESP_OK (0x0), records=11

--------------------------------------------------------------------------------------------------
No.  | SSID                     | MAC Address (BSSID) | RSSI   | Chan | Encryption Type     
--------------------------------------------------------------------------------------------------
1    | KMITL-WIFI               | 78:17:BE:C0:7D:A1 | -53  dBm | 1    | OPEN (No Password)  
2    | KMITL-Legacy             | 78:17:BE:C0:7D:A0 | -55  dBm | 1    | WPA2_ENTERPRISE     
3    | KMITL-IoT                | 78:17:BE:C0:7D:A2 | -55  dBm | 1    | WPA2_PSK            
4    | <Hidden SSID>            | 6A:57:07:6F:73:08 | -58  dBm | 6    | WPA2_PSK            
5    | FUB                      | EA:67:85:97:BF:39 | -60  dBm | 10   | WPA2_PSK            
6    | Test-WiFi                | 0A:8A:B4:BB:12:61 | -61  dBm | 6    | WPA2_WPA3_PSK       
7    | Pornprom                 | 2E:FE:61:42:3D:24 | -66  dBm | 6    | WPA2_PSK            
8    | eem_Luna                 | 2E:2B:53:FB:4B:AC | -70  dBm | 1    | WPA2_WPA3_PSK       
9    | KMITL-Legacy             | 78:17:BE:A9:94:E0 | -77  dBm | 6    | WPA2_ENTERPRISE     
10   | KMITL-Legacy             | 78:17:BE:C0:66:60 | -84  dBm | 11   | WPA2_ENTERPRISE     
11   | KMITL-WIFI               | 78:17:BE:C0:66:61 | -86  dBm | 11   | OPEN (No Password)  
--------------------------------------------------------------------------------------------------

I (4477) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (4477) LAB_WIFI_SCAN: >>> Experiment 5.1.2: Channel-Specific Scan (Channel 1)
I (4477) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (4487) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_start(scan_config, block=true)
I (4697) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_start() returned ESP_OK (0x0) [Duration: 200 ms]
I (4697) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_get_ap_num(&ap_count)
I (4697) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_get_ap_num() returned ESP_OK (0x0), ap_count=7
I (4707) LAB_WIFI_SCAN: [STATUS]: Scan SUCCESS
I (4707) LAB_WIFI_SCAN: [AP COUNT]: 7 network(s) found
I (4717) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_get_ap_records(&number, ap_info)
I (4727) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_get_ap_records() returned ESP_OK (0x0), records=7

--------------------------------------------------------------------------------------------------
No.  | SSID                     | MAC Address (BSSID) | RSSI   | Chan | Encryption Type     
--------------------------------------------------------------------------------------------------
1    | KMITL-Legacy             | 78:17:BE:C0:7D:A0 | -46  dBm | 1    | WPA2_ENTERPRISE     
2    | KMITL-IoT                | 78:17:BE:C0:7D:A2 | -46  dBm | 1    | WPA2_PSK            
3    | KMITL-WIFI               | 78:17:BE:C0:7D:A1 | -51  dBm | 1    | OPEN (No Password)  
4    | eem_Luna                 | 2E:2B:53:FB:4B:AC | -65  dBm | 1    | WPA2_WPA3_PSK       
5    | KMITL-IoT                | 78:17:BE:C0:66:22 | -78  dBm | 1    | WPA2_PSK            
6    | KMITL-Legacy             | 78:17:BE:C0:66:20 | -79  dBm | 1    | WPA2_ENTERPRISE     
7    | KMITL-WIFI               | 78:17:BE:C0:66:21 | -80  dBm | 1    | OPEN (No Password)  
--------------------------------------------------------------------------------------------------

I (5827) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (5827) LAB_WIFI_SCAN: >>> Experiment 5.1.3: Targeted SSID Scan - Existing ("KMITL-WIFI")
I (5827) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (5837) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_start(scan_config, block=true)
I (8347) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_start() returned ESP_OK (0x0) [Duration: 2499 ms]
I (8347) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_get_ap_num(&ap_count)
I (8347) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_get_ap_num() returned ESP_OK (0x0), ap_count=3
I (8357) LAB_WIFI_SCAN: [STATUS]: Scan SUCCESS
I (8357) LAB_WIFI_SCAN: [AP COUNT]: 3 network(s) found
I (8367) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_get_ap_records(&number, ap_info)
I (8377) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_get_ap_records() returned ESP_OK (0x0), records=3

--------------------------------------------------------------------------------------------------
No.  | SSID                     | MAC Address (BSSID) | RSSI   | Chan | Encryption Type     
--------------------------------------------------------------------------------------------------
1    | KMITL-WIFI               | 78:17:BE:C0:7D:A1 | -51  dBm | 1    | OPEN (No Password)  
2    | KMITL-WIFI               | 78:17:BE:C0:66:21 | -79  dBm | 1    | OPEN (No Password)  
3    | KMITL-WIFI               | 78:17:BE:C0:66:61 | -81  dBm | 11   | OPEN (No Password)  
--------------------------------------------------------------------------------------------------

I (9437) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (9437) LAB_WIFI_SCAN: >>> Experiment 5.1.4: Targeted SSID Scan - Non-Existent ("NON_EXISTENT_AP_9999")
I (9437) LAB_WIFI_SCAN: ------------------------------------------------------------------
I (9447) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_start(scan_config, block=true)
I (11957) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_start() returned ESP_OK (0x0) [Duration: 2498 ms]
I (11957) LAB_WIFI_SCAN: [FORENSIC]: Call esp_wifi_scan_get_ap_num(&ap_count)
I (11957) LAB_WIFI_SCAN: [FORENSIC]: esp_wifi_scan_get_ap_num() returned ESP_OK (0x0), ap_count=0
I (11967) LAB_WIFI_SCAN: [STATUS]: Scan SUCCESS
I (11967) LAB_WIFI_SCAN: [AP COUNT]: 0 network(s) found
W (11977) LAB_WIFI_SCAN: [NOTE]: No Access Point found matching the criteria.
I (11977) LAB_WIFI_SCAN: ==================================================================
I (11987) LAB_WIFI_SCAN:   [Phase 1 Completed: Wi-Fi Scan Finished]
I (11997) LAB_WIFI_SCAN:   Program stopped after scanning. Auth/Assoc Phase not started.
I (12007) LAB_WIFI_SCAN: ==================================================================
I (12007) main_task: Returned from app_main()
```