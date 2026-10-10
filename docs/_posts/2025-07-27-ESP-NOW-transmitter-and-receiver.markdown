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

'
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
'
