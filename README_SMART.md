#include <Wire.h>
#include <WiFi.h>
#include <LiquidCrystal_I2C.h> // include Library ของจอ LCD
#include <TridentTD_LineNotify.h> // include Library ของ Line Notify
#include <TimeLib.h> // include Library ของ Time ไว้แสดงเวลา
#include <NTPClient.h> // include Library ของ Time ดึงข้อมูลเวลาปัจจุบัน

LiquidCrystal_I2C lcd(0x27, 16, 2); //สร้างอ็อบเจ็กต์ของคลาส LiquidCrystal_I2C ควบคุม LCD / 0x27 ระบุที่อยู่ของชิป / 16 ความกว้าง(จำนวนอักษรแนวนอน) / 2 ความสูง(จำนวนแถว)

/* จัดการ WiFi และ Line Token */
const char* ssid = "CHANOM";
const char* password = "0955438698zax";
const char* LINE_TOKEN = "Xvx6Iwn6p7WIVX8f9jpDbz2it8V6d4Gk5x98GVEFwyU";
/*          สิ้นสุด            */

bool lineNotified = false; // ประกาศตัวแปลเป็น Boolean

WiFiUDP ntpUDP; // ดึงข้อมูลเวลาปัจจุบัน
NTPClient timeClient(ntpUDP, "pool.ntp.org"); // ดึงข้อมูลเวลาปัจจุบัน

void setup() {
  pinMode(2, OUTPUT); //ไฟสถานะเช็คที่บอร์ด
  Serial.begin(115200); //ความเร็ว bit per secondในการแสดงผ่าน Serial monitor
  Serial.println(LINE.getVersion()); //แสดงเวอร์ชันของไลบรารี Line Notify ผ่าน Serial Monitor
  LINE.setToken(LINE_TOKEN); //ตั้งค่า Token เก็บไว้ที่ LINE_TOKEN

  WiFi.begin(ssid, password); //เริ่มต้นการใช้งาน WiFi

  while (WiFi.status() != WL_CONNECTED) { //ตรวจสอบสถานะการเชื่อมต่อ WiFi โดยใช้ฟังก์ชัน WiFi.status() != คือ ไม่เท่ากับ
    delay(1000);
    Serial.println("Connecting to WiFi..."); //แสดงผ่าน Serial monitor
    lcd.begin(); //เริ่มต้นใช้จอ LCD
    lcd.backlight(); //เปิดไฟหลังจอ LCDให้แสดงที่แสงน้อย
    lcd.setCursor(0, 1); //ตั้งค่าตำแหน่งที่จะแสดงข้อความบนจอ LCD ตำแหน่งที่ 0 แถวที่ 1 (แถวแรกตำแหน่งแรก) 
    lcd.print("Connecting WiFi ..."); //แสดงผ่าน LCD
  }

  Serial.println("Connected to WiFi"); 
  Serial.print("IP Address: ");
  Serial.println(WiFi.localIP()); //แสดง IP ผ่าน Serial monitor

  lcd.begin(); //เริ่มใช้จอ LCD
  lcd.backlight();
  lcd.setCursor(2, 1); //ตั้งค่าตำแหน่งที่จะแสดงข้อความบนจอ LCD ตำแหน่งที่ 2 แถวที่ 1 (แถวแรกตำแหน่งสอง)
  lcd.print("WiFi Success");
  delay(5000);
  lcd.clear();

  // เริ่มต้นให้เวลาที่ ESP32 จะใช้ NTPClient หรือเรียกเวลาปัจจุบัน
  timeClient.begin();
  timeClient.setTimeOffset(7 * 3600); // ตั้งค่าตาม UTC  ของท้องถิ่น (ในที่นี้เป็น UTC+7) ตย.กำหนดค่า UTC  เป็นวินาที ซึ่งคำนวณจาก 7 ชั่วโมง และมี 1 ชั่วโมงมีค่าเท่ากับ 3600 วินาที
  timeClient.update(); // ปรับเวลาทันทีหลังจากเริ่มต้น
}

const int numSensors = 6; //constant ประกาศเป็นค่าคงที่
int sensorPins[numSensors] = {18, 17, 19, 15, 16, 4}; //ประกาศตัวแปรสำหรับใช้งาน Sensor
int totalSlots = numSensors; //เก็บค่านับจำนวนเซนเซอร์ทั้งหมดไว้ที่ totalSlots
int detectedSlots = 0; //กำหนดค่าเริ่มต้นเป็น 0 ใช้เก็บจำนวนของเซ็นเซอร์ที่ตรวจจับได้หรือตรวจจับแล้ว

void loop() {
  if (WiFi.status() != WL_CONNECTED) { //ตรวจสอบสถานะWiFi ถ้าไม่มีการเชื่อมต่อหรือการเชื่อมต่อหลุด
    lcd.clear();
    lcd.backlight();
    lcd.setCursor(2, 1); //ตั้งค่าตำแหน่งที่จะแสดงข้อความบนจอ LCD ตำแหน่งที่ 2 แถวที่ 1 (แถวแรกตำแหน่งสอง)
    lcd.print("Lost Connect"); //แสดงคำว่า Lost Connect ผ่านจอ LCD หาก WiFi หลุด
    delay(5000); //หน่วง 5 วิ
    lcd.clear(); //ล้างค่าหน้าจอ
    lcd.setCursor(2, 1); //ตั้งค่าตำแหน่งที่จะแสดงข้อความบนจอ LCD ตำแหน่งที่ 2 แถวที่ 1 (แถวแรกตำแหน่งสอง)
    lcd.print("Retry Connect"); //แสดงข้อความเชื่อมต่ออีกครั้งที่จอ LCD
    
    WiFi.begin(ssid, password);

    while (WiFi.status() != WL_CONNECTED) { //ใช้ตรวจสอบสถานะ WiFi ถ้าหากยังไม่มีการเขื่อมต่อ
      delay(1000);
      Serial.println("Connecting to WiFi..."); //แสดงผ่าน Serial monitor
    }
    lcd.clear(); //ล้างหน้าจอ
    lcd.setCursor(2, 1); //ตั้งค่าตำแหน่งที่จะแสดงข้อความบนจอ LCD ตำแหน่งที่ 2 แถวที่ 1 (แถวแรกตำแหน่งสอง)
    lcd.print("WiFi Success"); //แสดงเชื่อมต่อสำเร็จผ่าน LCD
    digitalWrite(2, HIGH); //ไฟสถานะ
    delay(5000); //หน่วง 5 วิ
    lcd.clear(); //ล้างหน้าจอ
  }

  detectedSlots = 0; //กำหนตัวแปรมาเก็บค่าเซนเซอร์ที่ตรวจจับได้ กำหนดเป็น 0

  for (int i = 0; i < totalSlots; i++) { //ตัวแปร i เป็นตัวนับ ที่เริ่มต้นที่ 0 และวนลูปจนถึง totalSlots - 1
    int sensorValue = digitalRead(sensorPins[i]); //อ่านค่าขาของเซ็นเซอร์ในตำแหน่งที่ i ในรอบที่กำลังวนลูป และเก็บค่านั้นไว้ในตัวแปร sensorValue

    if (sensorValue == HIGH) { //อ่านค่าเซนเซอร์ที่จับได้หากจับได้ให้ detectedSlotsเพิ่มค่า ++
      detectedSlots++;
    }
  }
  lcd.clear();

  // ดึงเวลาจากเซิร์ฟเวอร์ NTP
  timeClient.update();

  if (detectedSlots > 0) { //เช็คการตรวจจับเซนเซอร์
    // ดึงข้อมูลเวลาปัจจุบันจาก NTPClient
    time_t now = timeClient.getEpochTime();
    struct tm *tm_info;
    tm_info = localtime(&now);

    // สร้าง timestamp
    String timestamp = String("Timestamp: ") +
                       String(tm_info->tm_mday) + "/" +
                       String(tm_info->tm_mon + 1) + "/" +
                       String(tm_info->tm_year + 1900) + " " +
                       String(tm_info->tm_hour) + ":" +
                       String(tm_info->tm_min) + ":" +
                       String(tm_info->tm_sec);

    lcd.backlight();
    lcd.setCursor(0, 1);
    lcd.print("Parking Space " + String(detectedSlots)); //แสดงผ่านจอ LCD

    if (!lineNotified) { //เช็คสถานะการส่ง Line Nofity
      LINE.notify("✅ ที่จอดรถว่าง ✅\n" + timestamp); //ส่งไลน์ว่าว่าง
      lineNotified = true; //ตรวจสอบการส่งซ้ำโดยใช้ตัวแปร lineNotified เซ็ตค่าเป็น true จากเดิม false
    }
  } else if (detectedSlots == 0) { //เช็คการตรวจจับเซนเซอร์
    // ดึงข้อมูลเวลาปัจจุบันจาก NTPClient
    time_t now = timeClient.getEpochTime();
    struct tm *tm_info;
    tm_info = localtime(&now);

    // สร้าง timestamp
    String timestamp = String("Timestamp: ") +
                       String(tm_info->tm_mday) + "/" +
                       String(tm_info->tm_mon + 1) + "/" +
                       String(tm_info->tm_year + 1900) + " " +
                       String(tm_info->tm_hour) + ":" +
                       String(tm_info->tm_min) + ":" +
                       String(tm_info->tm_sec);

    lcd.backlight();
    lcd.setCursor(1, 1); //ตั้งค่าตำแหน่งที่จะแสดงข้อความบนจอ LCD ตำแหน่งที่ 1 แถวที่ 1 (แถวแรกตำแหน่งสอง)
    lcd.print("Statusg  : FULL!! "); //แสดงผ่านจอ LCD

    if (lineNotified) {
      LINE.notify("❌ ที่จอดรถเต็ม ❌\n" + timestamp); //ส่งLine ว่าเต็ม
      lineNotified = false; //ตรวจสอบการส่งซ้ำโดยใช้ตัวแปร lineNotified เซ็ตค่าคืนเป็น false ก่อนที่จะนำไปวนเงื่อนไขใหม่
    }
  }

  delay(100);
}
