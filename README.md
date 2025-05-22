#include "esp_camera.h"     // Library to interface with the camera
#include <WiFi.h>           // WiFi library for connecting to the network
#include <WebServer.h>      // Library to set up a web server on ESP32
#include "soc/soc.h"        // Used to disable brownout detector
#include "soc/rtc_cntl_reg.h" // Used for brownout control

// WiFi/hotspot credentials change according to ur deatils.
const char* ssid     =  "suspicious"; 
const char* password = "123456789"; 

// Web server on port 80
WebServer server(80);

// Camera GPIO configuration for AI Thinker Camera board
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

void setup() {
  // Section 1: Disable brownout detector (prevents resets during voltage drops)
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0);

  // Section 2: Initialize Serial for debugging
  Serial.begin(115200);
  Serial.setDebugOutput(true);

  // Section 3: Initialize camera
  if (init_camera() != ESP_OK) {
    Serial.println("Camera init failed");
    return;
  }

  // Section 4: Connect to WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi connected");
  Serial.print(WiFi.localIP());

  // Section 5: Set up the web server to handle MJPEG stream
  server.on("/stream", HTTP_GET, handle_jpg_stream);
  server.begin();
}

void loop() {
  // Section 6: Handle client requests for streaming video
  server.handleClient();
}

// Function to handle the MJPEG stream
void handle_jpg_stream() {
  camera_fb_t * fb = NULL;
  char buffer[64];
  int64_t last_frame = esp_timer_get_time();

  server.setContentLength(CONTENT_LENGTH_UNKNOWN);
  server.send(200, "multipart/x-mixed-replace; boundary=frame");

  while (true) {
    // Section 1: Capture image frame
    fb = esp_camera_fb_get();
    if (!fb) {
      Serial.println("Camera capture failed");
      break;
    }

    // Section 2: Send current frame over the stream
    sprintf(buffer, "--frame\r\nContent-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n", fb->len);
    server.sendContent(buffer);  // Header
    server.sendContent((const char*)fb->buf, fb->len);  // JPEG frame
    server.sendContent("\r\n");  // End of frame

    // Section 3: Return frame buffer to free memory
    esp_camera_fb_return(fb);

    // Section 4: Calculate and print the time per frame
    int64_t now = esp_timer_get_time();
    int64_t frame_time = now - last_frame;
    last_frame = now;
    frame_time /= 1000;
    Serial.printf("MJPG: %u ms (%.1f fps)\n", (uint32_t)frame_time, 1000.0 / (uint32_t)frame_time);

    // Section 5: Break loop if client disconnects
    if (!server.client().connected()) {
      break;
    }
  }
}

// Camera initialization function
esp_err_t init_camera() {
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  config.frame_size = FRAMESIZE_VGA;
  config.jpeg_quality = 12;
  config.fb_count = 2;

  // Initialize the camera
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x", err);
    return err;
  }

  return ESP_OK;
}



