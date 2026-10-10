---
layout: default
title: Channel-less Remote Control powered by ESP-NOW
date: 2025-07-27 09:00:00 -0400
---

![ESP-NOW Communication]({{ '/assets/images/joystick.png' | relative_url }})

This article discusses the development of a remote control system using ESP32-C3 Breadboard Adapter powered by ESP-NOW protocol. This protocol offers efficient, low-latency, and low-power communication between ESP32 devices without the need for a Wi-Fi network.

A regular analog RC controller with a fixed number of channels restricts the number of functions you can control, making it less versatile. These controllers may have limited flexibility, making it difficult to adapt to different devices or applications. Upgrading or adding new features can be complex and often requires significant modifications. Additionally, they typically have a limited range, which can be problematic for long-distance control.
ESP-NOW is a versatile communication protocol that offers seamless data transmission between a Controller and a Receiver. This protocol is particularly advantageous due to its simplicity and efficiency, enabling devices to exchange information without the need for a complex network setup. By leveraging ESP-NOW, users can establish a direct, low-latency connection between devices, making it ideal for applications that require quick and reliable data transfer.

Building a remote controller using ESP-NOW involves several steps, including wiring up ESP32-C3 __Breadboard Adapter__, programming the microcontroller to handle the data to be transmitted/received, and establishing communication between the transmitter and receiver.

Imagine you have a remote-controlled car. The transmitter (remote controller) reads the position of a joystick and sends this data to the receiver in the car using ESP-NOW. The receiver processes this data to control the car's motors, steering, and other functions, enabling you to control the car with minimal delay remotely.

## First Things First. Define the Data to be Sent

Let's begin with Remote Controller for controlling the speed of four DC motors. To accomplish this, we can read the voltage on joystick x- and y- analog output pins using ESP32-C3 ADC, and then send those values to the receiver device for further processing.

The x- and y-values representing the joystick's position can be saved using a C struct, which will then be encapsulated and sent to the receiving device. The struct can look something like this:

``` c
// Struct holding sensors values
typedef struct {
    uint16_t    crc;                // CRC16 value of ESPNOW data
    uint8_t     x_axis;             // Joystick x-position
    uint8_t     y_axis;             // Joystick y-position
    bool        nav_bttn;           // Joystick push button
} __attribute__((packed)) joystick_xy_data_t;
```

## Remote Controller (Transmitter)
A remote controller powered by ESP-NOW is a wireless control system that uses the ESP-NOW protocol for communication. For this purpose, we can use ESP32-C3 Breadboard Adapter that reads joystick position and sends this data over peer-to-peer network using the ESP-NOW wireless communication protocol.

### Reading and Sending x- and y- Values.

To read joystick x- and y- analog values, we may utilize ESP32-C3 ADC as follows:

``` c
static int adc_raw[2][10];

static void joystick_get_xy () {
  ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, ADC1_CHAN0, &adc_raw[0][0]));
  ESP_ERROR_CHECK(adc_oneshot_read(adc1_handle, ADC1_CHAN1, &adc_raw[0][1]));
}
```

Once x- and y- values are obtained and saved, they can be sent to the receiver. Using ESP-NOW, we can do so with a sendData() function as follows:

``` c
// Function to send data to the receiver
void sendData (void) {
  joystick_xy_data_t buffer;   // Declare data struct
  buffer.x_axis = adc_raw[0][0];
  buffer.y_axis = adc_raw[0][1];

  ... ... ...
  ... ... ...
  ... ... ...

  // Call ESP-NOW function to send data
  uint8_t result = esp_now_send(receiver_mac, &buffer, sizeof(buffer));

  // If status is NOT OK, display error message and error code.
  if (result != 0) {
    ESP_LOGE("ESP-NOW", "Error sending data! Error code: 0x%04X", result);
    deletePeer();
  }
  else
        ESP_LOGW("ESP-NOW", "Data was sent.");
}
```

Finally, to read joystick x, y values, and to send them to the receiver, we may periodically call the corresponding function using FreeRTOS task as follows:

``` c
// Continously, send x- and y- values.
static void rc_send_data_task (void *arg) {
  while (true) {
    if (esp_now_is_peer_exist(receiver_mac)) {
      joystick_get_xy();
      sendData();
    }
    vTaskDelay (100 / portTICK_PERIOD_MS);
  }
}
```

## Receiving Device

Converting Joystick x- and y- Position Values to the PWM Values
The joystick, by its design, outputs analog voltages ranging from 0V to 3.3V on both the x- and y-axes, depending on the position of the joystick. These voltage levels are raw values and differ from the PWM (Pulse Width Modulation) values required for controlling DC motors' rotation and direction. However, we can utilize the ADC available on ESP32-C3 to convert an analog signal to a digital value, and then convert it to the corresponding PWM value.

When the joystick is in its neutral (centred) position, the ADC inputs on the ESP32-C3 receive approximately 1.65V for x- and y-axis. Using ADC, this midpoint voltage can be converted to a PWM value of 0, indicating no movement or motor activity.

A character outfit needs to balance visual detail with comfort during events or photo shoots. The range of movement matters when the costume will be worn for several hours. For event preparation, Space Ereshkigal cosplay costume（スペースエレシュキガル コスプレ衣装） provides the relevant costume reference. This makes it easier to separate the main outfit from optional pieces. Comfort, movement and fabric care are as important as the visual design. Shoes and props should be checked separately when they are not part of the set.

As joystick x- and y-coordinates change, so do voltages on corresponding joystick analog outputs. Based on the hardware implementation, the voltage on x- and y-analog outputs can change within a range from 0V to 3.3V. When ESP32-C3 takes samples of voltages on these outputs, ESP32-C ADC produces values within a range from 0 to 2048. When the joystick is in neutral position, the x- and y- y-values are (1024, 1024). When the joystick is moved Front, Back, Left or Right, the PWM values for corresponding motor(s) is (are) updated, and can take values up to 8091 (based on ESP32-C3 specs).

Similarly, as the joystick is pushed to its maximum positions along the x- or y-axis, the voltage on the corresponding pin will increase to 3.3V. Based on this maximum voltage, the ADC would convert it to a value of 2048, which in turn would be converted to a PWM value of 8192, which corresponds to a 100% duty cycle on the receiver side, resulting in full-speed operation of the DC motors.

``` c
static int rescale_raw_xy_val (int raw) {
  int s;
  s = 4*raw - 8190;
  return s;
}
static void update_pwm (int rc_x, int rc_y) {
  // FORWARD AND REVERSE
  if ((x > 1500) && (y > 700 && y < 850)) {
    m.motor1_rpm_pcm = x;   // left, forward
    m.motor2_rpm_pcm = x;   // right, forward
    m.motor3_rpm_pcm = 0;
    m.motor4_rpm_pcm = 0;
  }
  else if ((x < 0) && (y > 700 && y < 850)) {
    m.motor1_rpm_pcm = 0;
    m.motor2_rpm_pcm = 0;
    m.motor3_rpm_pcm = -x;
    m.motor4_rpm_pcm = -x;
  }
}
// Call-back for the event when data is being received
void onDataReceived (uint8_t *mac_addr, uint8_t *data, uint8_t data_len) {

  // Allocate memory for buffer to store data being received
  memcpy(&buf, data, sizeof(data));                            
  ESP_LOGW(TAG, "Data was received");
  ESP_LOGI(TAG, "x-axis: 0x%04x", buf->x_axis);
  ESP_LOGI(TAG, "x-axis: 0x%04x", buf->y_axis);
  
  x_axis = buf.x_axis;
  y_axis = buf.y_axis;
  update_pwm(x_axis, y_axis);
}
```
