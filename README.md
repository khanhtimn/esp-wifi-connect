# ESP32 Wi-Fi Connect

## Bổ sung

- **Sửa thư viện**
  → 78\_\_esp-wifi-connect: hàm StartAccessPoint() và Stop() trong wifi_configuration_ap.cc

```
void WifiConfigurationAp::StartAccessPoint()
{
    // Initialize the TCP/IP stack (idempotent)
    esp_err_t err = esp_netif_init();
    if (err != ESP_OK && err != ESP_ERR_INVALID_STATE)
    {
        ESP_ERROR_CHECK(err);
    }

    /************************************************************/
    /*          Đảm bảo event loop mặc định tồn tại             */
    /************************************************************/
    err = esp_event_loop_create_default();
    if (err != ESP_OK && err != ESP_ERR_INVALID_STATE)
    {
        ESP_ERROR_CHECK(err);
    }

    /************************************************************/
    /*         Tạo AP netif mặc định nếu chưa tồn tại           */
    /************************************************************/
    ap_netif_ = esp_netif_get_handle_from_ifkey("WIFI_AP_DEF");
    if (!ap_netif_)
    {
        ap_netif_ = esp_netif_create_default_wifi_ap();
    }
    /************************************************************/
    /*    Đảm bảo default STA netif tồn tại trước khi khởi tạo  */
    /*    Wi-Fi để các hook driver được thiết lập đúng cách     */
    /************************************************************/
    if (esp_netif_get_handle_from_ifkey("WIFI_STA_DEF") == nullptr)
    {
        if (!esp_netif_create_default_wifi_sta())
        {
            ESP_LOGE(TAG, "Failed to create default WIFI_STA_DEF netif");
        }
        else
        {
            ESP_LOGI(TAG, "Created default WIFI_STA_DEF netif for provisioning AP");
        }
    }

    // Set the router IP address to 192.168.4.1
    esp_netif_ip_info_t ip_info;
    IP4_ADDR(&ip_info.ip, 192, 168, 4, 1);
    IP4_ADDR(&ip_info.gw, 192, 168, 4, 1);
    IP4_ADDR(&ip_info.netmask, 255, 255, 255, 0);
    esp_netif_dhcps_stop(ap_netif_);
    esp_netif_set_ip_info(ap_netif_, &ip_info);
    esp_netif_dhcps_start(ap_netif_);
    // Start the DNS server
    dns_server_.Start(ip_info.gw);

    // Initialize the WiFi stack in Access Point mode
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    err = esp_wifi_init(&cfg);
    if (err != ESP_OK)
    {
        if (err == ESP_ERR_WIFI_INIT_STATE)
        {
            ESP_LOGW(TAG, "esp_wifi_init called while already initialised");
        }
        else
        {
            ESP_ERROR_CHECK(err);
        }
    }

    // Get the SSID
    std::string ssid = GetSsid();

    // Set the WiFi configuration
    wifi_config_t wifi_config = {};
    strcpy((char *)wifi_config.ap.ssid, ssid.c_str());
    wifi_config.ap.ssid_len = ssid.length();
    wifi_config.ap.max_connection = 4;
    wifi_config.ap.authmode = WIFI_AUTH_OPEN;

    // Start the WiFi Access Point
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_APSTA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_AP, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
    err = esp_wifi_start();
    if (err != ESP_OK)
    {
        if (err == ESP_ERR_WIFI_CONN || err == ESP_ERR_WIFI_NOT_STOPPED)
        {
            ESP_LOGW(TAG, "esp_wifi_start returned %s", esp_err_to_name(err));
        }
        else
        {
            ESP_ERROR_CHECK(err);
        }
    }

#ifdef CONFIG_SOC_WIFI_SUPPORT_5G
    // Temporarily use only 2.4G Wi-Fi.
    ESP_ERROR_CHECK(esp_wifi_set_band_mode(WIFI_BAND_MODE_2G_ONLY));
#endif

    ESP_LOGI(TAG, "Access Point started with SSID %s", ssid.c_str());

    // 加载高级配置
    nvs_handle_t nvs;
    err = nvs_open("wifi", NVS_READONLY, &nvs);
    if (err == ESP_OK)
    {
        // 读取OTA URL
        char ota_url[256] = {0};
        size_t ota_url_size = sizeof(ota_url);
        err = nvs_get_str(nvs, "ota_url", ota_url, &ota_url_size);
        if (err == ESP_OK)
        {
            ota_url_ = ota_url;
        }

        // 读取WiFi功率
        err = nvs_get_i8(nvs, "max_tx_power", &max_tx_power_);
        if (err == ESP_OK)
        {
            ESP_LOGI(TAG, "WiFi max tx power from NVS: %d", max_tx_power_);
            ESP_ERROR_CHECK(esp_wifi_set_max_tx_power(max_tx_power_));
        }
        else
        {
            esp_wifi_get_max_tx_power(&max_tx_power_);
        }

        // 读取BSSID记忆设置
        uint8_t remember_bssid = 0;
        err = nvs_get_u8(nvs, "remember_bssid", &remember_bssid);
        if (err == ESP_OK)
        {
            remember_bssid_ = remember_bssid != 0;
        }
        else
        {
            remember_bssid_ = false; // 默认值
        }

        // 读取睡眠模式设置
        uint8_t sleep_mode = 0;
        err = nvs_get_u8(nvs, "sleep_mode", &sleep_mode);
        if (err == ESP_OK)
        {
            sleep_mode_ = sleep_mode != 0;
        }
        else
        {
            sleep_mode_ = true; // 默认值
        }

        nvs_close(nvs);
    }
}
```

```
void WifiConfigurationAp::Stop()
{
    // 停止SmartConfig服务
    if (sc_event_instance_)
    {
        esp_event_handler_instance_unregister(SC_EVENT, ESP_EVENT_ANY_ID, sc_event_instance_);
        sc_event_instance_ = nullptr;
    }
    esp_smartconfig_stop();

    // 停止定时器
    if (scan_timer_)
    {
        esp_timer_stop(scan_timer_);
        esp_timer_delete(scan_timer_);
        scan_timer_ = nullptr;
    }

    // 停止Web服务器
    if (server_)
    {
        httpd_stop(server_);
        server_ = nullptr;
    }

    // 停止DNS服务器
    dns_server_.Stop();

    // 注销事件处理器
    if (instance_any_id_)
    {
        esp_event_handler_instance_unregister(WIFI_EVENT, ESP_EVENT_ANY_ID, instance_any_id_);
        instance_any_id_ = nullptr;
    }
    if (instance_got_ip_)
    {
        esp_event_handler_instance_unregister(IP_EVENT, IP_EVENT_STA_GOT_IP, instance_got_ip_);
        instance_got_ip_ = nullptr;
    }

    // Bỏ thay đổi mode Wi-Fi ở đây để giữ nguyên APSTA cho tới khi provisioning END xử lý.
    // Bỏ destroy ap_netif_ ở đây để tránh làm mất handler khi provisioning manager vẫn còn reference.

    // 停止WiFi并重置模式
    /*
    esp_wifi_stop();
    esp_wifi_deinit();
    esp_wifi_set_mode(WIFI_MODE_NULL);

    // 释放网络接口资源
    if (ap_netif_)
    {
        esp_netif_destroy(ap_netif_);
        ap_netif_ = nullptr;
    }
    */

    ESP_LOGI(TAG, "Wifi configuration AP stopped");
}
```


## Component
This component helps with Wi-Fi connection for the device.

It first tries to connect to a Wi-Fi network using the credentials stored in the flash. If this fails, it starts an access point and a web server to allow the user to connect to a Wi-Fi network.

The URL to access the web server is `http://192.168.4.1`.

### Screenshot: Wi-Fi Configuration

<img src="assets/ap_v3.png" width="320" alt="Wi-Fi Configuration">

### Screenshot: Advanced Options

<img src="assets/ap_v3_advanced.png" width="320" alt="Advanced Configuration">

## Changelog: v2.6.0

- Add support for ESP32C5 5G mode.

## Changelog: v2.4.0

- Add ja / zh-TW languages.
- Add advanced tab.
- Add "Connection: close" headers to save open sockets.

## Changelog: v2.3.0

- Add support for language request.

## Changelog: v2.2.0

- Add support for ESP32 SmartConfig(ESPTouch v2)

## Changelog: v2.1.0

- Improve Wi-Fi connection logic.

## Changelog: v2.0.0

- Add support for multiple Wi-Fi SSID management.
- Auto switch to the best Wi-Fi network.
- Captive portal for Wi-Fi configuration.
- Support for multiple languages (English, Chinese).

## Configuration

The Wi-Fi credentials are stored in the flash under the "wifi" namespace.

The keys are "ssid", "ssid1", "ssid2" ... "ssid9", "password", "password1", "password2" ... "password9".

## Usage

```cpp
// Initialize the default event loop
ESP_ERROR_CHECK(esp_event_loop_create_default());

// Initialize NVS flash for Wi-Fi configuration
esp_err_t ret = nvs_flash_init();
if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    ret = nvs_flash_init();
}
ESP_ERROR_CHECK(ret);

// Get the Wi-Fi configuration
auto& ssid_list = SsidManager::GetInstance().GetSsidList();
if (ssid_list.empty()) {
    // Start the Wi-Fi configuration AP
    auto& ap = WifiConfigurationAp::GetInstance();
    ap.SetSsidPrefix("ESP32");
    ap.Start();
    return;
}

// Otherwise, connect to the Wi-Fi network
WifiStation::GetInstance().Start();
```

