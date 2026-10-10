---
layout: default
title: ESP-NOW Transmitter & Receiver Devices"
date: 2025-07-27 09:00:00 -0400
---

This post introduces ESP-NOW communication between two ESP32s: one sends control data using FreeRTOS tasks; the other receives it via a callback. It highlights MAC registration, shared data structures, and consistent Wi-Fi settings for reliable peer-to-peer messaging.

This post presents a practical introduction to implementing ESP-NOW communication between two ESP32 microcontrollers, one configured as a transmitter and the other as a receiver. The transmitter collects control data—such as joystick positions and motor PWM values—and sends it using a structured format and FreeRTOS tasks to manage concurrent operations. The sendData() function handles data preparation and transmission, while a callback monitors delivery status and manages errors.

On the receiver side, the focus is on registering the transmitter’s MAC address and using the onDataReceived() callback to process incoming data. The post emphasizes the importance of consistent configuration between devices, including shared data structures, Wi-Fi channel settings, and peer registration. This ensures reliable, low-latency communication without the need for a traditional Wi-Fi network, making ESP-NOW a suitable protocol for remote control applications using ESP32.

## COMMON CODE BLOCKS FIRST

ESP-NOW is a wireless protocol that allows devices to exchange data directly without needing a Wi-Fi network. For this to work reliably, both devices must be programmed in a similar fashion.

For example, the code defining and handling the data must be consistent to ensure proper data transmission and reception. This means that the data structs sent between devices must be identical, as well as the initialization of ESP-NOW protocol.

### Defining Data Struct

The following struct defines the format and organization of the data being transmitted from the sender to the receiver in an ESP-NOW communication setup. Each field represents a specific sensor reading or control signal that the receiving device will interpret and act upon accordingly.

``` c
typedef struct {
    uint16_t    crc;                // CRC16 value of ESPNOW data
    int         x_axis;             // Joystick x-position
    int         y_axis;             // Joystick y-position
    bool        nav_bttn;           // Joystick push button
    bool        led;                // LED ON/OFF state
    uint8_t     motor1_rpm_pwm;     // PWMs for four DC motors
    uint8_t     motor2_rpm_pwm;
    uint8_t     motor3_rpm_pwm;
    uint8_t     motor4_rpm_pwm;
} __attribute__((packed)) sensors_data_t;
```

### Getting ESP-NOW Ready

The part of code responsible for initializing ESP-NOW is the same both the systems. The app_main() in both transmitter and receiver devices must begin by initializing the Non-Volatile Storage (NVS) required for Wi-Fi and ESP-NOW operations. After that, calling wifi_init() sets the devices to Wi-Fi station mode necessary for ESP-NOW communication.

This function is essential for both transmitter and receiver devices, as ESP-NOW must be initialized before sending or receiving any data.

`
void app_main(void)
{
    // Initialize NVS
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK( nvs_flash_erase() );
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK( ret );
    wifi_init();
}

void wifi_init()
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK( esp_wifi_init(&cfg) );
    ESP_ERROR_CHECK( esp_wifi_set_storage(WIFI_STORAGE_RAM) );
    ESP_ERROR_CHECK( esp_wifi_set_mode(WIFI_MODE_STA));//ESPNOW_WIFI_MODE));
    ESP_ERROR_CHECK( esp_wifi_start());
    //ESP_ERROR_CHECK( esp_wifi_set_channel(CONFIG_ESPNOW_CHANNEL, WIFI_SECOND_CHAN_NONE));
    ESP_ERROR_CHECK( esp_wifi_set_channel(11, WIFI_SECOND_CHAN_NONE));
    #if CONFIG_ESPNOW_ENABLE_LONG_RANGE
    ESP_ERROR_CHECK( esp_wifi_set_protocol(ESPNOW_WIFI_IF, WIFI_PROTOCOL_11B|WIFI_PROTOCOL_11G|WIFI_PROTOCOL_11N|WIFI_PROTOCOL_LR) );
    #endif
}
`

## MOVING ON TO TRANSMITTER & RECEIVER DEVICES

### Transmitter

On the transmitter device, the code is organized around two main functions: transmission and verification. The transmission function transmission_init() is called from the app_main() function for sending the data to the receiver using ESP-NOW. The verification function ensures that the data has been sent successfully by checking the transmission status using the call-back function. Since the ESP32-C3 microcontroller may be handling multiple tasks simultaneously, the code uses FreeRTOS tasks to allow different operations to run concurrently. This multitasking approach helps maintain responsiveness and efficiency, especially when managing communication alongside other system functions.

The code block below outlines the function responsible for initializing the ESP-NOW protocol on the transmitter device. Specifically, the transmission_init() function begins by calling esp_now_init() to activate the ESP-NOW feature. If initialization is successful, the program proceeds to register a callback function, statusDataSend, which is triggered after each transmission to verify whether the data was sent correctly. A key part of this setup involves specifying the receiving device using its MAC address, along with other configuration parameters such as the communication channel and encryption settings. These details are then registered using esp_now_add_peer(). Finally, a FreeRTOS task named rc_send_data_task is created to manage the periodic transmission of remote control data. This structured approach ensures that the transmitter is properly configured for reliable communication with the receiver using ESP-NOW.

`void transmission_init()
{
    esp_err_t espnow_ret = esp_now_init();
    if (espnow_ret != ESP_OK) {
        ESP_LOGE(TAG, "esp_now_init() failed: %s", esp_err_to_name(espnow_ret));
        return;
    }
    ESP_LOGI(TAG, "ESPNOW initialized successfully");
    esp_now_register_send_cb(statusDataSend);

    // Set ESP-NOW receiver device configuration values
    memcpy(devices.peer_addr, receiver_mac, 6);
    devices.channel = 11;
    devices.encrypt = false;
    esp_now_add_peer(&devices);

    // Defince a task for periodically sending ESPNOW remote control data
    xTaskCreate(rc_send_data_task, "RC", 2048, NULL, 4, NULL);
}
`

The FreeRTOS task for executing the function for sending the data every 100 ms is as follows:

`
// Task to periodically send ESPNOW remote control data
static void rc_send_data_task()
{
    while (true) {
        if (esp_now_is_peer_exist((uint8_t*)receiver_mac)) {
            sendData();
        }
        vTaskDelay (100 / portTICK_PERIOD_MS);
    }
}
`

Finally, shown below is the sendData() function responsible for preparing and transmitting a structured set of control data from the transmitter device to the receiver using the ESP-NOW protocol. It begins by updating the fields of a data buffer, including joystick coordinates, button states, LED status, and PWM values for four motors. Note that the composition of variables corresponds to the data struct defined earlier.

Before sending the data, the function retrieves and logs the current Wi-Fi channel to ensure the device is operating on the correct frequency. It then calls esp_now_send(), passing the receiver’s MAC address, a pointer to the data buffer, and the size of the data. If the transmission fails, the function logs detailed error messages, including the error code and the receiver’s MAC address, and calls deletePeer() to remove the peer configuration. This function plays a central role in the communication process, ensuring that the transmitter sends the data to the receiver while providing feedback in case of transmission issues.

`
static void sendData (void)
{
    buffer.crc = 0;
    buffer.x_axis = 240;
    buffer.y_axis = 256;
    buffer.nav_bttn = 0;
    buffer.led = 0;
    buffer.motor1_rpm_pwm = 0;
    buffer.motor2_rpm_pwm = 0;
    buffer.motor3_rpm_pwm = 0;
    buffer.motor4_rpm_pwm = 0;

    get_joystick_xy(&y, &x);
    //ESP_LOGI("(x, y)", "[ %d, %d ]", x, y);
    buffer.x_axis = x;
    buffer.y_axis = y;

    // Display brief summary of data being sent.
    ESP_LOGI(TAG, "Joystick (x,y) position ( %d, %d )", buffer.x_axis, buffer.y_axis);
    ESP_LOGI(TAG, "pwm 1, pwm 2 [ 0x%04X, 0x%04X ]", (uint8_t)buffer.motor1_rpm_pwm, (uint8_t)buffer.motor2_rpm_pwm);
    ESP_LOGI(TAG, "pwm 3, pwm 4 [ 0x%04X, 0x%04X ]", (uint8_t)buffer.motor3_rpm_pwm, (uint8_t)buffer.motor4_rpm_pwm);

    uint8_t channel;
    esp_wifi_get_channel(&channel, NULL);
    ESP_LOGE(TAG, "ESP-NOW Channel: %d", channel);
    // Call ESP-NOW function to send data (MAC address of receiver, pointer to the memory holding data & data length)
    uint8_t result = esp_now_send((uint8_t*)receiver_mac, (uint8_t *)&buffer, sizeof(buffer));

    // If status is NOT OK, display error message and error code (in hexadecimal).
    if (result != 0) {
        ESP_LOGE(TAG, "Error sending data! Error code: 0x%04X", result);
        ESP_LOGE(TAG, "esp_now_send() failed: %s", esp_err_to_name(result));
        ESP_LOGE(TAG, "Ensure that receiver is powered-on.");
        ESP_LOGE(TAG, "Ensure that received MAC is: %02X:%02X:%02X:%02X:%02X:%02X",
                 receiver_mac[0], receiver_mac[1], receiver_mac[2],
                 receiver_mac[3], receiver_mac[4], receiver_mac[5]);
        
        deletePeer();
    }
}
Lastly, the statusDataSend() function serves as a callback that is automatically triggered after each ESP-NOW data transmission. Its primary role is to check whether the data was sent successfully and to provide feedback based on the result. If the transmission is successful, the function logs a confirmation message along with the MAC address of the receiving device. However, in the event of a failure, the function also removes the peer configuration using deletePeer() and restarts the device with esp_restart() to attempt another transmission session. This callback is essential for monitoring the reliability of communication.

// Callback function to handle the status of data transmission
// This function is called when the data is sent or if there is an error.
static void statusDataSend(const uint8_t *mac_addr, esp_now_send_status_t status)
{
    if (status == ESP_NOW_SEND_SUCCESS) {
        ESP_LOGI(TAG, "Data sent successfully to: %02X:%02X:%02X:%02X:%02X:%02X",
                 mac_addr[0], mac_addr[1], mac_addr[2],
                 mac_addr[3], mac_addr[4], mac_addr[5]);
    } else {
        ESP_LOGE(TAG, "Error sending data to: %02X:%02X:%02X:%02X:%02X:%02X",
                 mac_addr[0], mac_addr[1], mac_addr[2],
                 mac_addr[3], mac_addr[4], mac_addr[5]);
        ESP_LOGE(TAG, "Error sending data. Error code: 0x%04X", status);
        ESP_LOGE(TAG, "esp_now_send() failed: %s", esp_err_to_name(status));
        ESP_LOGE(TAG, "Ensure that receiver is powered-on and MAC is correct.");
        deletePeer();
        esp_restart();
    }
}
`

## Receiver

On the receiver device, the code is slightly simpler, as its primary role is to receive and process incoming data. Within the app_main() function, the transmitter device is registered by specifying its MAC address and communication parameters using the esp_now_peer_info_t structure. This includes setting the Wi-Fi interface, communication channel, and encryption settings. Once the peer information is configured, it is added to the ESP-NOW peer list using esp_now_add_peer(). Importantly, a callback function named onDataReceived is registered using esp_now_register_recv_cb(). This function is automatically triggered whenever data is received, allowing the program to store the incoming information into a predefined data structure. This setup ensures that the receiver is properly configured to recognize the transmitter and handle incoming ESP-NOW messages efficiently.

`
esp_now_peer_info_t transmitterInfo = {0};
memcpy(transmitterInfo.peer_addr, transmitter_mac, ESP_NOW_ETH_ALEN);
transmitterInfo.channel = 0; // Current WiFi channel
transmitterInfo.ifidx = ESP_IF_WIFI_STA;
transmitterInfo.encrypt = false;
ESP_ERROR_CHECK(esp_now_add_peer(&transmitterInfo));

ESP_ERROR_CHECK(esp_now_register_recv_cb((void*)onDataReceived));
The onDataReceived() function is designed to handle incoming data on the receiver device in an ESP-NOW communication setup. When data is received, this callback function is automatically triggered. It begins by logging the MAC address of the transmitting device and the length of the received data, which helps verify the source and size of the transmission. The actual data is then copied into a predefined buffer using memcpy(), allowing the receiver to store and later process the information in a structured format. This function plays a key role in ensuring that incoming data is captured accurately and efficiently for further use within the application.

void onDataReceived (const uint8_t *mac_addr, const uint8_t *data, uint8_t data_len) {

    ESP_LOGI(TAG, "Data received from: %02x:%02x:%02x:%02x:%02x:%02x, len=%d", mac_addr[0], mac_addr[1], mac_addr[2], mac_addr[3], mac_addr[4], mac_addr[5], data_len);

    memcpy(&buf, data, sizeof(buf));
}
`

# CONCLUSION

In conclusion, establishing ESP-NOW communication between two ESP32 devices requires a consistent and well-structured approach. Both the transmitter and receiver must share identical data structures and initialization routines, including the setup of NVS and Wi-Fi in station mode. On the transmitter side, FreeRTOS tasks manage data preparation and transmission, while callback functions monitor success and handle errors. The receiver, in contrast, focuses on registering the transmitter and storing incoming data through a dedicated callback. This setup ensures efficient, reliable, and direct communication between devices without relying on a traditional Wi-Fi network.
