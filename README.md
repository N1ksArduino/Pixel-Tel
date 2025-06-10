#include <ESP8266WiFi.h>
#include <WiFiUdp.h>
#include <GyverOLED.h>
#include <Wire.h>

// Настройки Wi-Fi
const char* ssid = "Redmi 12";         // Замените на имя вашей сети
const char* password = "159753258";   // Замените на пароль

// Настройки multicast
IPAddress multicastIP(239, 255, 0, 1); // Групповой IP-адрес
const int multicastPort = 1234;         // Порт для обмена

// Пины для кнопок
#define BTN_PREV 14   // D5 - предыдущий символ
#define BTN_NEXT 12   // D6 - следующий символ
#define BTN_SEND 13   // D7 - отправить

// Объекты
GyverOLED<SSD1306_128x32, OLED_NO_BUFFER> oled;
WiFiUDP udp;

// Состояния устройства
char symbols[5] = {'А', 'Б', 'В', 'Г', 'Д'};
uint8_t currentChar = 0;        // Текущий выбранный символ
char receivedChar = ' ';        // Последний полученный символ
unsigned long lastReceive = 0;  // Время последнего приема

void setup() {
  Serial.begin(115200);
  
  // Инициализация дисплея
  Wire.begin();
  oled.init();
  oled.clear();
  oled.setScale(1);
  oled.home();
  oled.print("Загрузка...");
  oled.update();
  
  // Настройка кнопок
  pinMode(BTN_PREV, INPUT_PULLUP);
  pinMode(BTN_NEXT, INPUT_PULLUP);
  pinMode(BTN_SEND, INPUT_PULLUP);
  
  // Подключение к Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    oled.clear();
    oled.home();
    oled.print("Подключение...");
    oled.update();
  }
  
  // Инициализация UDP
  udp.beginMulticast(WiFi.localIP(), multicastIP, multicastPort);
  
  // Отображение начального состояния
  updateDisplay();
}

void loop() {
  // Обработка кнопок
  handleButtons();
  
  // Прием сообщений
  receiveMessages();
  
  // Автоочистка экрана через 5 сек
  if (millis() - lastReceive > 5000 && receivedChar != ' ') {
    receivedChar = ' ';
    updateDisplay();
  }
  
  delay(10);
}

void handleButtons() {
  // Кнопка "Предыдущий символ"
  if (!digitalRead(BTN_PREV)) {
    currentChar = (currentChar == 0) ? 4 : currentChar - 1;
    updateDisplay();
    delay(300);  // Защита от дребезга
  }
  
  // Кнопка "Следующий символ"
  if (!digitalRead(BTN_NEXT)) {
    currentChar = (currentChar + 1) % 5;
    updateDisplay();
    delay(300);
  }
  
  // Кнопка "Отправить"
  if (!digitalRead(BTN_SEND)) {
    sendChar(symbols[currentChar]);
    delay(300);
  }
}

void sendChar(char ch) {
  // Формируем пакет: [длина 1] + [символ]
  uint8_t buffer[1] = { (uint8_t)ch };
  
  udp.beginPacketMulticast(multicastIP, multicastPort, WiFi.localIP());
  udp.write(buffer, 1);
  udp.endPacket();
}

void receiveMessages() {
  int packetSize = udp.parsePacket();
  if (packetSize) {
    uint8_t buffer[packetSize];
    udp.read(buffer, packetSize);
    
    // Проверяем корректность пакета
    if (packetSize == 1) {
      receivedChar = (char)buffer[0];
      lastReceive = millis();
      updateDisplay();
    }
  }
}

void updateDisplay() {
  oled.clear();
  oled.home();
  
  // Строка 1: Выбранный символ
  oled.print("Отправить: ");
  oled.print(symbols[currentChar]);
  
  // Строка 2: Полученный символ
  oled.setCursor(0, 1);
  oled.print("Получено: ");
  oled.print(receivedChar);
  
  oled.update();
}
