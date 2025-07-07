# 3rd-EYE
#include "esp_camera.h"
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <ESP_Mail_Client.h> // For sending email
#define CAMERA_MODEL_AI_THINKER

#include "camera_pins.h"

// Wi-Fi credentials
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Panic button pin
#define PANIC_BUTTON_PIN 12
#define ALERT_LED 33

bool alertSent = false;

// Email config (you can also use Blynk or Firebase instead)
SMTPSession smtp;
ESP_Mail_Session session;
SMTP_Message message;

void startCameraServer(); // optional, for live stream

void setup() {
  Serial.begin(115200);
  pinMode(PANIC_BUTTON_PIN, INPUT_PULLUP);
  pinMode(ALERT_LED, OUTPUT);

  // Initialize camera
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer   = LEDC_TIMER_0;
  config.pin_d0       = Y2_GPIO_NUM;
  config.pin_d1       = Y3_GPIO_NUM;
  config.pin_d2       = Y4_GPIO_NUM;
  config.pin_d3       = Y5_GPIO_NUM;
  config.pin_d4       = Y6_GPIO_NUM;
  config.pin_d5       = Y7_GPIO_NUM;
  config.pin_d6       = Y8_GPIO_NUM;
  config.pin_d7       = Y9_GPIO_NUM;
  config.pin_xclk     = XCLK_GPIO_NUM;
  config.pin_pclk     = PCLK_GPIO_NUM;
  config.pin_vsync    = VSYNC_GPIO_NUM;
  config.pin_href     = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn     = PWDN_GPIO_NUM;
  config.pin_reset    = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  config.frame_size = FRAMESIZE_QVGA;
  config.jpeg_quality = 10;
  config.fb_count = 1;

  if (!esp_camera_init(&config)) {
    Serial.println("Camera init success");
  } else {
    Serial.println("Camera init failed");
    return;
  }

  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }
  Serial.println("\nWiFi connected");
}

void loop() {
  if (digitalRead(PANIC_BUTTON_PIN) == LOW && !alertSent) {
    alertSent = true;
    digitalWrite(ALERT_LED, HIGH);
    sendAlert(); // Send image/email
    delay(5000);
    digitalWrite(ALERT_LED, LOW);
  }
}

void sendAlert() {
  camera_fb_t * fb = esp_camera_fb_get();
  if (!fb) {
    Serial.println("Camera capture failed");
    return;
  }

  session.server.host_name = "smtp.gmail.com";
  session.server.port = 465;
  session.login.email = "YOUR_EMAIL@gmail.com";
  session.login.password = "YOUR_APP_PASSWORD";
  session.login.user_domain = "";

  message.sender.name = "Soulo 3rd Eye";
  message.sender.email = "YOUR_EMAIL@gmail.com";
  message.subject = "ALERT: Panic Button Triggered!";
  message.addRecipient("User", "TO_EMAIL@gmail.com");

  message.text.content = "Emergency! Safety alert triggered by the user.";
  message.text.charSet = "us-ascii";
  message.text.transfer_encoding = Content_Transfer_Encoding::enc_7bit;

  SMTP_Attachment att;
  att.descr.filename = "alert.jpg";
  att.descr.mime = "image/jpg";
  att.file.data = fb->buf;
  att.file.size = fb->len;
  att.descr.transfer_encoding = Content_Transfer_Encoding::enc_base64;
  message.addAttachment(att);

  if (!smtp.connect(&session)) return;

  if (!MailClient.sendMail(&smtp, &message)) {
    Serial.println("Error sending email, " + smtp.errorReason());
  } else {
    Serial.println("Alert sent!");
  }

  esp_camera_fb_return(fb);
  smtp.closeSession();
}
