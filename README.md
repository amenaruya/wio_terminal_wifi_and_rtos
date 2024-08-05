# wio_terminal_wifi_and_rtos

zenn: <https://zenn.dev/amenaruya/articles/801a76734428ec>

This is how you can use WiFi while running other tasks on FreeRTOS.

```cpp
#include "rpcWiFi.h"
#include <millisDelay.h>
#include "RTC_SAMD51.h"

#define STACK_SIZE 256
#define WIFI_SSID "SSID"
#define WIFI_PASSWORD "PASSWORD"

using namespace erpc;

const unsigned int  LOCAL_PORT      = 2390;
const char* const   TIME_SERVER     = "time.nist.gov";
const int           NTP_PORT        = 123;
const int           NTP_PACKET_SIZE = 48;
const unsigned long NTP_TO_UNIX     = 2208988800UL;
const long          JST             = 32400UL;
const unsigned long FIVE_SECONDS    = 5000UL;

millisDelay         updateDelay;
byte                packetBuffer[NTP_PACKET_SIZE];
DateTime            currentDateTime;
WiFiUDP             wifiUdp;
unsigned long       ntpTime;
RTC_SAMD51          rtcSamd51;

void                connectToWiFi();
unsigned long       getNtpTime();
void                sendNTPpacket(const char* const address);

static void         ThreadA(void* pvParameters);
static void         ThreadB(void* pvParameters);

static void ThreadA(void* pvParameters) {
    (void) pvParameters;
    Serial.println("Thread A\tStarted");

    while (1) {
        Serial.println("Thread A\tHi");
        delay(1000);
    }
}

static void ThreadB(void* pvParameters) {
    (void) pvParameters;
    Serial.println("Thread B\tStarted");

    while (1) {
        Serial.println("Thread B\tHello");
        delay(2000);
    }
}

void setup() {
    Serial.begin(115200);
    vNopDelayMS(1000);
    while(!Serial);

    Serial.println("\n******************************");
    Serial.println("        Program start         ");
    Serial.println("******************************\n");

    Thread TaskA(
        &ThreadA,
        configMAX_PRIORITIES - 10,
        STACK_SIZE,
        "Task A"
    );
    Thread TaskB(
        &ThreadB,
        configMAX_PRIORITIES - 11,
        STACK_SIZE,
        "Task B"
    );
    Thread* tasksArray[2] = {&TaskA, &TaskB};
    for (Thread* t : tasksArray) {
        t -> start();
    }

    connectToWiFi();
    ntpTime = getNtpTime();
    if (ntpTime == 0) {
        Serial.println("Failed to get time from network time server.");
    }
    if (!rtcSamd51.begin()) {
        Serial.println("Couldn't find RTC");
        while (1);
    }
    currentDateTime = rtcSamd51.now();
    Serial.print("RTC time is: ");
    Serial.println(currentDateTime.timestamp(DateTime::TIMESTAMP_FULL));
    DateTime dtNtp = DateTime(ntpTime);
    Serial.printf(
        "NTP\t%04d/%d/%d %02d:%02d:%02d\n",
        dtNtp.year(),
        dtNtp.month(),
        dtNtp.day(),
        dtNtp.hour(),
        dtNtp.minute(),
        dtNtp.second()
    );
    Serial.println(dtNtp.timestamp(DateTime::TIMESTAMP_FULL));
    rtcSamd51.adjust(dtNtp);
    Serial.println("RTC (boot) time updated.");
    currentDateTime = rtcSamd51.now();
    Serial.printf("Adjusted RTC (boot) time is: ");
    Serial.println(currentDateTime.timestamp(DateTime::TIMESTAMP_FULL));
    updateDelay.start(FIVE_SECONDS);
}

void loop() {
    if (updateDelay.justFinished()) {
        updateDelay.repeat();
        ntpTime = getNtpTime();
        if (ntpTime == 0) {
            Serial.println("Failed to get time from network time server.");
        }
        else {
            rtcSamd51.adjust(DateTime(ntpTime));
            Serial.println("\nrtcSamd51 time updated.");
            currentDateTime = rtcSamd51.now();
            Serial.print("Adjusted RTC time is: ");
            Serial.println(currentDateTime.timestamp(DateTime::TIMESTAMP_FULL));
        }
    }
}


void connectToWiFi() {
    Serial.println("Connecting to WiFi");
    WiFi.disconnect(true);
    Serial.println("Waiting for Wi-Fi connection...");
    WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
    while (WiFi.status() != WL_CONNECTED) {
        WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
        delay(500);
    }
    Serial.println("Connected.");
}
void sendNTPpacket(const char* const address) {
    for (int i = 0; i < NTP_PACKET_SIZE; ++i) {
        packetBuffer[i] = 0;
    }
    packetBuffer[0] = 0b11100011;
    packetBuffer[1] = 0;
    packetBuffer[2] = 6;
    packetBuffer[3] = 0xEC;
    packetBuffer[12] = 49;
    packetBuffer[13] = 0x4E;
    packetBuffer[14] = 49;
    packetBuffer[15] = 52;
    wifiUdp.beginPacket(address, NTP_PORT);
    wifiUdp.write(packetBuffer, NTP_PACKET_SIZE);
    wifiUdp.endPacket();
}
unsigned long getNtpTime() {
    if (WiFi.status() == WL_CONNECTED) {
        wifiUdp.begin(WiFi.localIP(), LOCAL_PORT);
        sendNTPpacket(TIME_SERVER);
        delay(1000);
        if (wifiUdp.parsePacket()) {
            Serial.println("wifiUdp packet received\n");
            wifiUdp.read(packetBuffer, NTP_PACKET_SIZE);
            unsigned long highWord = word(packetBuffer[40], packetBuffer[41]);
            unsigned long lowWord = word(packetBuffer[42], packetBuffer[43]);
            unsigned long secsSince1900 = highWord << 16 | lowWord;
            unsigned long epoch = secsSince1900 - NTP_TO_UNIX;
            unsigned long adjustedTime = epoch + JST;
            return adjustedTime;
        }
        else {
            wifiUdp.stop();
            return 0;
        }
        wifiUdp.stop();
    }
    else {
        return 0;
    }
}
```

```shell

******************************
        Program start         
******************************

Thread A	Started
Thread A	Hi
Thread B	Started
Thread B	Hello
Connecting to WiFi
Waiting for Wi-Fi connection...
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Connected.
Thread B	Hello
Thread A	Hi
wifiUdp packet received

RTC time is: 2000-01-01T00:00:00
NTP	2024/7/27 05:57:53
2024-07-27T05:57:53
RTC (boot) time updated.
Adjusted RTC (boot) time is: 2024-07-27T05:57:53
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi
wifiUdp packet received


rtcSamd51 time updated.
Adjusted RTC time is: 2024-07-27T05:57:59
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi
Thread A	Hi
wifiUdp packet received


rtcSamd51 time updated.
Adjusted RTC time is: 2024-07-27T05:58:04
Thread B	Hello
Thread A	Hi
Thread A	Hi
Thread B	Hello
Thread A	Hi

︙

```
