
# Báo cáo IoT 1

# Các thư viện bao gồm 

```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/event_groups.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "esp_log.h"
#include "esp_nimble_hci.h"
#include "nimble/nimble_port.h"
#include "nimble/nimble_port_freertos.h"
#include "host/ble_hs.h"
#include "services/gap/ble_svc_gap.h"
#include "services/gatt/ble_svc_gatt.h"
#include "sdkconfig.h"
#include <string.h>
#include "esp_err.h"
#include "esp_event_base.h"
#include "esp_netif_types.h"
#include "esp_wifi_types.h"
#include "freertos/projdefs.h"
#include "esp_mac.h"
#include "esp_wifi.h"
#include "lwip/err.h"
#include "lwip/sys.h"
#include <stdlib.h>
#include <esp_http_server.h>
```

- Các thư viện này cung cấp các tính năng như FreeRTOS (cho quản lý đa nhiệm), NVS (Non-Volatile Storage) để lưu dữ liệu, 
ESP32 SDK để truy cập WiFi, Bluetooth, và giao thức HTTP.

# Các macro và định nghĩa hằng số

```
#define EXAMPLE_ESP_WIFI_SSID      "myssid" 
#define EXAMPLE_ESP_WIFI_PASS      "mypassword"
/* EXAMPLE_ESP_WIFI_SSID: Định nghĩa SSID của mạng WiFi mà ESP32 sẽ kết nối. Ở đây, SSID mặc định là "myssid".
EXAMPLE_ESP_WIFI_PASS: Định nghĩa mật khẩu cho mạng WiFi, trong ví dụ này là "mypassword". */

#define EXAMPLE_ESP_WIFI_CHANNEL   1 
#define EXAMPLE_MAX_STA_CONN       4
#define MIN(x,y) ((x) <(y) ? (x) : (y)) // Đây là một macro đơn giản để trả về giá trị nhỏ hơn giữa hai giá trị x và y.
#define EXAMPLE_ESP_MAXIMUM_RETRY 10 // Số lần thử kết nối WiFi tối đa

#define WIFI_CONNECTED_BIT BIT0
#define WIFI_FAIL_BIT      BIT1
/* WIFI_CONNECTED_BIT: Biến bit này sẽ được đặt (set) khi WiFi đã kết nối thành công.
WIFI_FAIL_BIT: Biến bit này sẽ được đặt nếu quá trình kết nối WiFi thất bại. */

static const char *TAG_WIFI="wifi ";

static const char *TAG_HTTP="HTTP SERVER";
static int s_retry_num=0;
static EventGroupHandle_t s_wifi_event_group;
\\Đây là biến kiểu EventGroupHandle_t dùng để tạo một nhóm các sự kiện. 
Các bit như WIFI_CONNECTED_BIT và WIFI_FAIL_BIT sẽ được đặt hoặc xóa trong nhóm sự kiện này để phản ánh trạng thái kết nối WiFi.
char *TAG = "BLE-Server";
uint8_t ble_addr_type;
void ble_app_advertise(void);
\\Đây là hàm khởi động quá trình quảng bá BLE. Nó có thể được gọi để bắt đầu việc phát sóng thông tin BLE, 
cho phép các thiết bị khác tìm và kết nối đến ESP32 qua BLE.

```
# Xử lý HTTP Request và Gửi Form WiFi 
```
static esp_err_t root_get_handler(httpd_req_t *req)
{
const char resp[] ="<!DOCTYPE html>"
"<html>"
    "<head>"
        "<meta charset=\"UTF-8\">"
    "</head>"
    "<body>"
        "<form id=\"wifiForm\">"
            "<input type=\"text\" id=\"ssid\" name=\"ssid\" placeholder=\"SSID\" required>"
            "<h1></h1>"
            "<input type=\"password\" id=\"password\" name=\"password\" placeholder=\"Password\" required>"
            "<h1></h1>"
            "<button type=\"button\" onclick=\"sendWiFiInfo()\">Send WiFi Info</button>"
        "</form>"

        "<script>"
            "function sendWiFiInfo() {"
                "var ssid = document.getElementById(\"ssid\").value;"
                "var pass = document.getElementById(\"password\").value;"
                "var data = \"ssid=\" + encodeURIComponent(ssid) + \"&password=\" + encodeURIComponent(pass);"

                "var xhttp = new XMLHttpRequest();"
                "xhttp.open(\"POST\", \"/post\", true);"
                "xhttp.setRequestHeader(\"Content-type\", \"application/x-www-form-urlencoded\");"
                "xhttp.send(data);"
            "}"
        "</script>"
    "</body>"
"</html>";
httpd_resp_send(req,resp,HTTPD_RESP_USE_STRLEN);


return ESP_OK;
}

Đây là handler cho HTTP GET, gửi một trang HTML đơn giản với form để nhập SSID và mật khẩu.
Khi người dùng nhập thông tin và nhấn nút, JavaScript sẽ gửi thông tin bằng phương thức POST đến /post.

```
# Xử Lý Dữ Liệu POST

```
static esp_err_t set_post_handler(httpd_req_t *req)
{
    char buf[100];
    int ret, data_len = req->content_len;

    while (data_len > 0) {
        if ((ret = httpd_req_recv(req, buf, MIN(data_len, sizeof(buf)))) <= 0) {
            if (ret == HTTPD_SOCK_ERR_TIMEOUT) {
                continue;
            }
            return ESP_FAIL;
        }
        data_len -= ret;
    }
    buf[req->content_len] = '\0';

    char ssid[32] = {0};
    char password[64] = {0};
    sscanf(buf, "ssid=%31[^&]&password=%63s", ssid, password);
    ESP_LOGI(TAG_HTTP, "Parsed SSID: %s", ssid);
    ESP_LOGI(TAG_HTTP, "Parsed Password: %s", password);
    
    nvs_handle_t nvs_handle;
    ESP_ERROR_CHECK(nvs_open("storage", NVS_READWRITE, &nvs_handle));
    ESP_ERROR_CHECK(nvs_set_str(nvs_handle, "ssid", ssid));
    ESP_ERROR_CHECK(nvs_set_str(nvs_handle, "password", password));
    ESP_ERROR_CHECK(nvs_commit(nvs_handle));
    nvs_close(nvs_handle);

    httpd_resp_send(req, "WiFi configuration received", HTTPD_RESP_USE_STRLEN);
    vTaskDelay(1000 / portTICK_PERIOD_MS);
    esp_restart();
    return ESP_OK;
}

Đây là handler cho HTTP POST, nhận thông tin SSID và mật khẩu, sau đó lưu vào NVS (bộ nhớ không thay đổi) của ESP32.
Sau khi lưu thông tin WiFi, ESP32 sẽ khởi động lại (esp_restart) để kết nối với mạng WiFi mới.
```

# Xử lí kết nối wifi
```
static void wifi_event_handler(void* arg, esp_event_base_t event_base,
                                    int32_t event_id, void* event_data)
{
    if (event_id == WIFI_EVENT_AP_STACONNECTED) {
        wifi_event_ap_staconnected_t* event = (wifi_event_ap_staconnected_t*) event_data;
        ESP_LOGI(TAG_WIFI, "station "MACSTR" join, AID=%d",
                 MAC2STR(event->mac), event->aid);
    } else if (event_id == WIFI_EVENT_AP_STADISCONNECTED) {
        wifi_event_ap_stadisconnected_t* event = (wifi_event_ap_stadisconnected_t*) event_data;
        ESP_LOGI(TAG_WIFI, "station "MACSTR" leave, AID=%d",
                 MAC2STR(event->mac), event->aid);
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START){
		esp_wifi_connect();
	} else if (event_base == WIFI_EVENT && event_id ==WIFI_EVENT_STA_DISCONNECTED){
		if(s_retry_num <EXAMPLE_ESP_MAXIMUM_RETRY){
			esp_wifi_connect();
			s_retry_num++;
			ESP_LOGI(TAG_WIFI,"retry to connect to the AP");
			
		} else {
			xEventGroupSetBits(s_wifi_event_group, WIFI_FAIL_BIT);
			
		}
		ESP_LOGI(TAG, "connect to the AP fail");
	} else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP){
		ip_event_got_ip_t* event = (ip_event_got_ip_t*)event_data;
		ESP_LOGI(TAG_WIFI,"got ip:" IPSTR,IP2STR(&event->ip_info.ip));
		s_retry_num=0;
		xEventGroupSetBits(s_wifi_event_group, WIFI_CONNECTED_BIT);
	}   
}

Hàm xử lý sự kiện WiFi giúp kiểm soát các sự kiện khác nhau như kết nối thành công, ngắt kết nối, hoặc nhận địa chỉ IP.
```
# wifi_init_softap
```
void wifi_init_softap(void)
{
    esp_netif_create_default_wifi_ap();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    /* wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();: Tạo cấu hình mặc định cho WiFi.
ESP_ERROR_CHECK(esp_wifi_init(&cfg));: Khởi tạo WiFi driver với cấu hình đã định nghĩa.*/

    ESP_ERROR_CHECK(esp_event_handler_instance_registe(WIFI_EVENT, ESP_EVENT_ANY_ID,&wifi_event_handler,NULL,NULL));

    /* esp_event_handler_instance_register: Đăng ký một hàm xử lý sự kiện (wifi_event_handler) để lắng nghe các sự kiện WiFi, như kết nối, ngắt kết nối, hay các lỗi khác. */

    wifi_config_t wifi_config = {
        .ap = {
            .ssid = EXAMPLE_ESP_WIFI_SSID,
            .ssid_len = strlen(EXAMPLE_ESP_WIFI_SSID),
            .channel = EXAMPLE_ESP_WIFI_CHANNEL,
            .password = EXAMPLE_ESP_WIFI_PASS,
            .max_connection = EXAMPLE_MAX_STA_CONN,
#ifdef CONFIG_ESP_WIFI_SOFTAP_SAE_SUPPORT
            .authmode = WIFI_AUTH_WPA3_PSK,
            .sae_pwe_h2e = WPA3_SAE_PWE_BOTH,
#else /* CONFIG_ESP_WIFI_SOFTAP_SAE_SUPPORT */
            .authmode = WIFI_AUTH_WPA2_PSK,
#endif
            .pmf_cfg = {
                    .required = true,
            },
        },
    };
    if (strlen(EXAMPLE_ESP_WIFI_PASS) == 0) {
        wifi_config.ap.authmode = WIFI_AUTH_OPEN;
    }

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_AP));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());

    ESP_LOGI(TAG, "wifi_init_softap finished. SSID:%s password:%s channel:%d",
             EXAMPLE_ESP_WIFI_SSID, EXAMPLE_ESP_WIFI_PASS, EXAMPLE_ESP_WIFI_CHANNEL);

}

```
# wifi_init_sta
``` 
void wifi_init_sta(const char* ssid, const char* password)
{
    s_wifi_event_group = xEventGroupCreate();

    //ESP_ERROR_CHECK(esp_netif_init());

   // ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));

    //esp_event_handler_instance_t instance_any_id;
    //esp_event_handler_instance_t instance_got_ip;
    ESP_ERROR_CHECK(esp_event_handler_instance_register(WIFI_EVENT,
                                                        ESP_EVENT_ANY_ID,
                                                        &wifi_event_handler,
                                                        NULL,
                                                        NULL));
    ESP_ERROR_CHECK(esp_event_handler_instance_register(IP_EVENT,
                                                        IP_EVENT_STA_GOT_IP,
                                                        &wifi_event_handler,
                                                        NULL,
                                                        NULL));

    wifi_config_t wifi_config = {
        .sta = {
            .ssid = " ",
            .password = " ",
            /* Authmode threshold resets to WPA2 as default if password matches WPA2 standards (pasword len => 8).
             * If you want to connect the device to deprecated WEP/WPA networks, Please set the threshold value
             * to WIFI_AUTH_WEP/WIFI_AUTH_WPA_PSK and set the password with length and format matching to
             * WIFI_AUTH_WEP/WIFI_AUTH_WPA_PSK standards.
             */
            .threshold.authmode = WIFI_AUTH_WPA2_PSK,
            
        },
    };
    
    strncpy((char*)wifi_config.sta.ssid, ssid, sizeof(wifi_config.sta.ssid));
    strncpy((char*)wifi_config.sta.password, password, sizeof(wifi_config.sta.password));
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA) );
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config) );
    ESP_ERROR_CHECK(esp_wifi_start() );

    ESP_LOGI(TAG, "wifi_init_sta finished.");

    /* Waiting until either the connection is established (WIFI_CONNECTED_BIT) or connection failed for the maximum
     * number of re-tries (WIFI_FAIL_BIT). The bits are set by event_handler() (see above) */
    EventBits_t bits = xEventGroupWaitBits(s_wifi_event_group,
            WIFI_CONNECTED_BIT | WIFI_FAIL_BIT,
            pdFALSE,
            pdFALSE,
            pdMS_TO_TICKS(10000));

    /* xEventGroupWaitBits() returns the bits before the call returned, hence we can test which event actually
     * happened. */
    if (bits & WIFI_CONNECTED_BIT) {
        ESP_LOGI(TAG_WIFI, "connected to ap SSID:%s password:%s",
                 ssid, password);
    } else if (bits & WIFI_FAIL_BIT) {
        ESP_LOGI(TAG_WIFI, "Failed to connect to SSID:%s, password:%s",
        		ssid, password);
    } else {
        ESP_LOGE(TAG_WIFI, "UNEXPECTED EVENT");
    }
}
```
# Ghi dữ liệu qua BLE
```
static int device_write(uint16_t conn_handle, uint16_t attr_handle, struct ble_gatt_access_ctxt *ctxt, void *arg)
{
    // printf("Data from the client: %.*s\n", ctxt->om->om_len, ctxt->om->om_data);

    char * data = (char *)ctxt->om->om_data;
    
    char *ssid = strtok(data, "&");
    char *password = strtok(NULL, "&");

    printf("ssid: %s, password: %s\n", ssid, password);
    nvs_handle_t nvs_handle;
	ESP_ERROR_CHECK(nvs_open("storage",NVS_READWRITE,&nvs_handle));
	ESP_ERROR_CHECK(nvs_set_str(nvs_handle,"ssid",ssid));
	ESP_ERROR_CHECK(nvs_set_str(nvs_handle,"password",password));
	ESP_ERROR_CHECK(nvs_commit(nvs_handle));
	nvs_close(nvs_handle);
    wifi_init_sta(ssid, password);
    return 0;
}
```

# gatt_svcs[]
```
định nghĩa các dịch vụ GATT (Generic Attribute Profile) cho BLE (Bluetooth Low Energy).
Nó được dùng để mô tả các dịch vụ và đặc tính (characteristics) mà một thiết bị BLE cung cấp, 
chẳng hạn như việc đọc và ghi dữ liệu.


static const struct ble_gatt_svc_def gatt_svcs[] = {
    {.type = BLE_GATT_SVC_TYPE_PRIMARY,
     .uuid = BLE_UUID16_DECLARE(0x180),                 // Define UUID for device type
     .characteristics = (struct ble_gatt_chr_def[]){
         {.uuid = BLE_UUID16_DECLARE(0xFEF4),           // Define UUID for reading
          .flags = BLE_GATT_CHR_F_READ,
          .access_cb = device_read},
         {.uuid = BLE_UUID16_DECLARE(0xDEAD),           // Define UUID for writing
          .flags = BLE_GATT_CHR_F_WRITE,
          .access_cb = device_write},
         {0}}},
    {0}};
```

# ble_gap_event()

```
static int ble_gap_event(struct ble_gap_event *event, void *arg)
{
    switch (event->type)
    {
    // Advertise if connected
    case BLE_GAP_EVENT_CONNECT:
        ESP_LOGI("GAP", "BLE GAP EVENT CONNECT %s", event->connect.status == 0 ? "OK!" : "FAILED!");
        if (event->connect.status != 0)
        {
            ble_app_advertise();
        }
        break;
    // Advertise again after completion of the event
    case BLE_GAP_EVENT_DISCONNECT:
        ESP_LOGI("GAP", "BLE GAP EVENT DISCONNECTED");
        break;
    case BLE_GAP_EVENT_ADV_COMPLETE:
        ESP_LOGI("GAP", "BLE GAP EVENT");
        ble_app_advertise();
        break;
    default:
        break;
    }
    return 0;
}

BLE_GAP_EVENT_CONNECT: Sự kiện này xảy ra khi một thiết bị BLE kết nối.
ESP_LOGI: In ra thông tin kết nối, nếu event->connect.status == 0 thì kết nối thành công, ngược lại kết nối thất bại.
Nếu kết nối thất bại (status != 0), hàm ble_app_advertise() sẽ được gọi để tiếp tục quảng bá BLE, cho phép các thiết bị khác kết nối.

```
# ble_app_advertise()
``` 
Hàm ble_app_advertise() được sử dụng để khởi động quá trình quảng bá (advertising) cho thiết bị BLE,
giúp các thiết bị khác có thể phát hiện và kết nối với nó.
Quá trình quảng bá BLE bao gồm việc thiết lập các thông số về tên thiết bị và chế độ kết nối.

void ble_app_advertise(void)
{
    // GAP - device name definition
    struct ble_hs_adv_fields fields;
    const char *device_name;
    memset(&fields, 0, sizeof(fields));
    device_name = ble_svc_gap_device_name(); // Read the BLE device name
    fields.name = (uint8_t *)device_name;
    fields.name_len = strlen(device_name);
    fields.name_is_complete = 1;
    ble_gap_adv_set_fields(&fields);

    // GAP - device connectivity definition
    struct ble_gap_adv_params adv_params;
    memset(&adv_params, 0, sizeof(adv_params));
    adv_params.conn_mode = BLE_GAP_CONN_MODE_UND; // connectable or non-connectable
    adv_params.disc_mode = BLE_GAP_DISC_MODE_GEN; // discoverable or non-discoverable
    ble_gap_adv_start(ble_addr_type, NULL, BLE_HS_FOREVER, &adv_params, ble_gap_event, NULL);
}
struct ble_gap_adv_params adv_params: Cấu trúc này lưu trữ các tham số cần thiết cho việc quảng bá BLE.
memset(&adv_params, 0, sizeof(adv_params)): Đặt toàn bộ bộ nhớ của adv_params về 0.
adv_params.conn_mode = BLE_GAP_CONN_MODE_UND: Đặt chế độ kết nối thành "Undirected" (không hướng đích), 
cho phép thiết bị BLE quảng bá mà không nhắm vào một thiết bị cụ thể, và các thiết bị khác có thể kết nối.
adv_params.disc_mode = BLE_GAP_DISC_MODE_GEN: Đặt chế độ khám phá thành "General Discoverable", 
cho phép thiết bị BLE được phát hiện bởi tất cả các thiết bị BLE khác.
ble_gap_adv_start(ble_addr_type, NULL, BLE_HS_FOREVER, &adv_params, ble_gap_event, NULL): Bắt đầu quá trình quảng bá với các tham số sau:
ble_addr_type: Loại địa chỉ BLE (được định nghĩa trước đó, ví dụ BLE_ADDR_PUBLIC hoặc BLE_ADDR_RANDOM).
NULL: Không sử dụng địa chỉ khác.
BLE_HS_FOREVER: Thời gian quảng bá là mãi mãi (thiết bị sẽ quảng bá liên tục).
&adv_params: Các tham số quảng bá được cấu hình ở trên.
ble_gap_event: Hàm callback xử lý các sự kiện GAP (được định nghĩa ở phần trước).
NULL: Không truyền dữ liệu bổ sung nào khác cho callback.
```
