#include <databaseNEW.h>
//absen 1x dengan pembatasan jam dan mengirim ke excel
//SDA = D4
//SCK = D5
//MOSI = D7
//MISO = D6
//GND = G
//RST = D0
//3,3V = 3V
#include <ESP8266WiFi.h>
#include <ESP8266HTTPClient.h>
#include <WiFiClientSecure.h>
#include <SPI.h>
#include <MFRC522.h>
#include <NTPClient.h>
#include <WiFiUdp.h>

#include <Wire.h>
#include <LiquidCrystal_PCF8574.h>
LiquidCrystal_PCF8574 lcd(0x27);  // Sesuaikan dengan alamat I2C LCD (0x27 atau 0x3F)


// WiFi credentials
const char* ssid = "iPhone";         // Ganti dengan nama WiFi kamu
const char* password = "ip123456";  // Ganti dengan password WiFi kamu

// Google Apps Script Web App URL
const char* serverName = "https://script.google.com/macros/s/AKfycbx5k3Jy1vgko_bT_yinSJJLHNSU__L4vRHbfxF8vYzwIG0qQ8ABQk4hjXed0ifBshAQ/exec";

// Pin RFID untuk NodeMCU
#define RST_PIN D3  
#define SS_PIN D4   
#define Buzz_pin D0
#define Led_pin D8

MFRC522 mfrc522(SS_PIN, RST_PIN);  // Buat instance MFRC522

// NTP Client Setup
WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "pool.ntp.org", 7 * 3600, 60000);  // UTC+7 untuk WIB

// Menyimpan UID yang sudah absen hari ini
String absensiHariIni[10];  // Maksimal 10 orang, bisa diperbesar sesuai kebutuhan
int totalAbsensi = 0;
String tanggalHariIni = "";

void setup() {
  Serial.begin(9600);
  SPI.begin();         
  mfrc522.PCD_Init();  
  lcd.begin(16, 2);
  pinMode (Buzz_pin, OUTPUT);
  pinMode (Led_pin, OUTPUT); 

  // Koneksi ke WiFi
  WiFi.begin(ssid, password);
  Serial.print("Menghubungkan ke WiFi...");
  lcd.setBacklight(255); // Nyalakan backlight (nilai antara 0-255)
  lcd.setCursor(0, 0);
  lcd.print("Menghubungkan ke wifi .."); 
  delay(1000);
  lcd.clear();
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nTerhubung ke WiFi!");
  lcd.setBacklight(255); 
  lcd.setCursor(0, 1);
  lcd.print("Terhubung wifi"); // Cetak teks ke LCD
  digitalWrite (Buzz_pin, HIGH);
  delay(1000);
  // Mulai NTP Client
  timeClient.begin();      

  // Inisialisasi waktu
  timeClient.update();
  tanggalHariIni = getFormattedDate(timeClient.getEpochTime());  // Ambil tanggal (yyyy-mm-dd)
  Serial.println("Tanggal Hari Ini: " + tanggalHariIni);
  Serial.println("Tempelkan kartu RFID...");
  lcd.setBacklight(255); 
  lcd.setCursor(0, 0);
  lcd.print("Tempelkan kartu");
}

void loop() {
  timeClient.update();
  String tanggalSekarang = getFormattedDate(timeClient.getEpochTime());
    lcd.setBacklight(255); 
    lcd.setCursor(0, 0);
    lcd.print("Tempelkan Kartu");
    
  if (tanggalSekarang != tanggalHariIni) {
    resetAbsensiHarian();
    tanggalHariIni = tanggalSekarang;
  }

  int currentHour = timeClient.getHours();   // Ambil jam saat ini
  int currentMinute = timeClient.getMinutes();  // Ambil menit saat ini

  // Batasi hanya bisa digunakan antara jam 08:15 - 21:00
  if ((currentHour < 8) || (currentHour == 8 && currentMinute < 15) || (currentHour >= 21 && currentMinute < 5)) {
    Serial.println("RFID tidak aktif. Waktu operasional: 08:15 - 21:05");
    lcd.setBacklight(255); 
    lcd.setCursor(0, 0);
    lcd.print("RFID TIDAK AKTIF");
    digitalWrite (Led_pin, HIGH);
    delay(1000);  // Cek setiap 10 detik
    return;
  }

  // Cek kartu RFID
  if (!mfrc522.PICC_IsNewCardPresent() || !mfrc522.PICC_ReadCardSerial()) {
    return;
  }

  String uidString = getUIDString();

  // Cek apakah sudah absen hari ini
  if (sudahAbsenHariIni(uidString)) {
    Serial.println("Kartu ini sudah absen hari ini.");
    lcd.setBacklight(255); 
    lcd.setCursor(0, 0);
    lcd.print("Sudah Absen");
    digitalWrite (Led_pin, HIGH);
    delay(3000);
    lcd.clear();
    return;
  }

  String nama = "";
  String divisi = "";
  String angkatan = "";

  // Cek UID dan SAK untuk menentukan data
  if (isUIDValid(validUID1, sizeof(validUID1)) && mfrc522.uid.sak == validSAK1) {
    nama = "Muhammad Khoirul Rozi"; // nama masing"
    angkatan = "23"; //angkatan
    divisi = "Elektrik"; // Divisi
  } else {
    Serial.println("Data tidak valid!");
    lcd.setBacklight(255); 
    lcd.setCursor(0, 0);
    lcd.print("KTP tidak valid");
    delay (2000);
    lcd.clear();
    return;
  }

  Serial.println("Data valid! Mengirim ke Google Spreadsheet...");
  lcd.setBacklight(255); 
  lcd.setCursor(0, 0);
  lcd.print("KTP Valid");
  sendDataToGoogleSheets(divisi, nama, angkatan);


  // Tambahkan UID ke daftar absensi hari ini
  absensiHariIni[totalAbsensi] = uidString;
  totalAbsensi++;

  // Hentikan komunikasi dengan kartu
  mfrc522.PICC_HaltA();
  mfrc522.PCD_StopCrypto1();


  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("BERHASIL ABSEN");
  lcd.setCursor(0, 1);
  lcd.print(nama);
  digitalWrite (Buzz_pin, HIGH);
  delay (1500);
  lcd.clear();

  // delay(3000);

  delay(3000); // Delay untuk menghindari pembacaan berulang
}

// Fungsi untuk memeriksa apakah UID kartu valid
bool isUIDValid(const byte* validUID, byte length) {
  if (mfrc522.uid.size != length) {
    return false;
  }
  for (byte i = 0; i < length; i++) {
    if (mfrc522.uid.uidByte[i] != validUID[i]) {
      return false;
    }
  }
  return true;
}

// Fungsi untuk mendapatkan UID dalam bentuk string
String getUIDString() {
  String uidStr = "";
  for (byte i = 0; i < mfrc522.uid.size; i++) {
    uidStr += String(mfrc522.uid.uidByte[i], HEX);
  }
  return uidStr;
}

// Fungsi untuk mengecek apakah kartu sudah absen hari ini
bool sudahAbsenHariIni(String uid) {
  for (int i = 0; i < totalAbsensi; i++) {
    if (absensiHariIni[i] == uid) {
      return true;
    }
  }
  return false;
}

// Fungsi untuk mereset absensi harian
void resetAbsensiHarian() {
  totalAbsensi = 0;
  Serial.println("Daftar absensi harian di-reset.");
}

// Fungsi untuk mengirim data ke Google Sheets
void sendDataToGoogleSheets(String divisi, String nama, String angkatan) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    WiFiClientSecure client;
    client.setInsecure();  // Mengabaikan sertifikat SSL, cocok untuk testing

    String serverPath = String(serverName);

    // Data yang akan dikirim ke Google Apps Script (URL-encoded)
    String postData = "sheet_name=Sheet1&command=insert_row&values=" + nama + "," + divisi + "," + angkatan;

    http.begin(client, serverPath);  // Menggunakan WiFiClientSecure
    http.addHeader("Content-Type", "application/x-www-form-urlencoded");

    int httpResponseCode = http.POST(postData);

    if (httpResponseCode > 0) {
      String response = http.getString();
      Serial.println("Response dari server: " + response);
    } else {
      Serial.print("Error mengirim data. Kode HTTP: ");
      Serial.println(httpResponseCode);
      digitalWrite (Led_pin, HIGH);
    }

    http.end();
  } else {
    Serial.println("WiFi tidak terhubung!");
  }
}

// Fungsi untuk memformat waktu epoch menjadi tanggal (yyyy-mm-dd)
String getFormattedDate(unsigned long epochTime) {
  time_t rawTime = epochTime;
  struct tm* timeInfo = localtime(&rawTime);

  char buffer[11];  // Format: yyyy-mm-dd\0
  sprintf(buffer, "%04d-%02d-%02d", timeInfo->tm_year + 1900, timeInfo->tm_mon + 1, timeInfo->tm_mday);
  return String(buffer);
}
