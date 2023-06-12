#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <DHT.h>

#define DHTPIN 10        // DHT11连接到ESP32的引脚
#define DHTTYPE DHT11     // 使用DHT11传感器

DHT dht(DHTPIN, DHTTYPE); // 初始化DHT对象

BLEServer* pServer = NULL;
BLECharacteristic* pTemperatureCharacteristic = NULL;
BLECharacteristic* pHumidityCharacteristic = NULL;
bool deviceConnected = false;
float temperature = 0.0;
float humidity = 0.0;

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer* pServer) {
    deviceConnected = true;
  }

  void onDisconnect(BLEServer* pServer) {
    deviceConnected = false;
  }
};

void setup() {
  Serial.begin(115200);
  dht.begin();

  BLEDevice::init("ESP32-C3 BLE");
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService* pService = pServer->createService(BLEUUID((uint16_t)0x180F)); // 使用"Generic Access"服务

  pTemperatureCharacteristic = pService->createCharacteristic(
                                  BLEUUID((uint16_t)0x2A6E), // 自定义UUID用于温度特征
                                  BLECharacteristic::PROPERTY_READ |
                                  BLECharacteristic::PROPERTY_NOTIFY
                                );

  pHumidityCharacteristic = pService->createCharacteristic(
                               BLEUUID((uint16_t)0x2A6F), // 自定义UUID用于湿度特征
                               BLECharacteristic::PROPERTY_READ |
                               BLECharacteristic::PROPERTY_NOTIFY
                             );

  pTemperatureCharacteristic->addDescriptor(new BLE2902());
  pHumidityCharacteristic->addDescriptor(new BLE2902());

  pService->start();

  BLEAdvertising* pAdvertising = pServer->getAdvertising();
  pAdvertising->start();
}

void loop() {
  if (deviceConnected) {
    humidity = dht.readHumidity();    // 读取湿度值
    float newTemperature = dht.readTemperature();  // 读取温度值

    if (!isnan(humidity) && !isnan(newTemperature)) {
      if (abs(newTemperature - temperature) >= 1.0) {
        temperature = newTemperature;

        String temperatureString = String(temperature, 1);
        pTemperatureCharacteristic->setValue(temperatureString.c_str());
        pTemperatureCharacteristic->notify();

        if (temperature > 25.0) {
          Serial.println("温度过高！");
        }
        else if (temperature < 10.0) {
          Serial.println("温度过低！");
        }
      }

      if (abs(humidity - humidity) >= 1.0) {
        humidity = humidity;

        String humidityString = String(humidity, 1);
        pHumidityCharacteristic->setValue(humidityString.c_str());
        pHumidityCharacteristic->notify();
      }

      Serial.print("温度: ");
      Serial.print(temperature);
      Serial.print(" 湿度: ");
      Serial.print(humidity);
      Serial.println(" %");
    }
  }

  delay(2000); // 延迟2秒钟
}
