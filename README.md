# CamScanner
'''cpp

#include "esp_camera.h"
#include <WiFi.h>
#include "esp_http_server.h"

// Configuración de la red WiFi
const char* ssid = "Red_Erick";
const char* password = "Erick456@";

// Pines de la cámara para el ESP32-CAM (ajusta según tu modelo)
#define PWDN_GPIO_NUM    32
#define RESET_GPIO_NUM   -1
#define XCLK_GPIO_NUM    0
#define SIOD_GPIO_NUM    26
#define SIOC_GPIO_NUM    27

#define Y9_GPIO_NUM      35
#define Y8_GPIO_NUM      34
#define Y7_GPIO_NUM      39
#define Y6_GPIO_NUM      36
#define Y5_GPIO_NUM      21
#define Y4_GPIO_NUM      19
#define Y3_GPIO_NUM      18
#define Y2_GPIO_NUM      5
#define VSYNC_GPIO_NUM   25
#define HREF_GPIO_NUM    23
#define PCLK_GPIO_NUM    22

#define FLASH_GPIO_NUM   4  // Ajusta este número según tu conexión del flash

void startCameraServer();
void setupFlash();

void setup() {
  Serial.begin(115200);
  Serial.setDebugOutput(true);
  Serial.println();

  // Configura el pin del flash
  setupFlash();

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

  if(psramFound()){
    config.frame_size = FRAMESIZE_UXGA;
    config.jpeg_quality = 10;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_SVGA;
    config.jpeg_quality = 12;
    config.fb_count = 1;
  }
  
  // Inicializa la cámara
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Error inicializando la cámara: 0x%x", err);
    return;
  }

  // Conecta a la red WiFi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi conectado.");
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.localIP());

  // Inicia el servidor de la cámara
  startCameraServer();
  Serial.println("Servidor de cámara iniciado.");
}

void loop() {
  // No se necesita código en el loop principal para este ejemplo
}

// Configura el pin del flash
void setupFlash() {
  pinMode(FLASH_GPIO_NUM, OUTPUT);
  digitalWrite(FLASH_GPIO_NUM, HIGH); // Enciende el flash
}

// Implementación de la función startCameraServer
void startCameraServer() {
  httpd_config_t config = HTTPD_DEFAULT_CONFIG();
  httpd_handle_t server = NULL;

  httpd_uri_t uri_get = {
    .uri       = "/",
    .method    = HTTP_GET,
    .handler   = [](httpd_req_t *req) {
      char part_buf[64];
      static const char* _STREAM_CONTENT_TYPE = "multipart/x-mixed-replace;boundary=frame";
      static const char* _STREAM_BOUNDARY = "\r\n--frame\r\n";
      static const char* _STREAM_PART = "Content-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n";

      httpd_resp_set_type(req, _STREAM_CONTENT_TYPE);

      while (true) {
        camera_fb_t * fb = esp_camera_fb_get();
        if (!fb) {
          Serial.println("Camera capture failed");
          httpd_resp_send_500(req);
          return ESP_FAIL;
        }

        size_t hlen = snprintf(part_buf, 64, _STREAM_PART, fb->len);
        httpd_resp_send_chunk(req, _STREAM_BOUNDARY, strlen(_STREAM_BOUNDARY));
        httpd_resp_send_chunk(req, part_buf, hlen);
        httpd_resp_send_chunk(req, (const char *)fb->buf, fb->len);
        httpd_resp_send_chunk(req, "\r\n", 2);
        esp_camera_fb_return(fb);

        // Añadir un pequeño retraso para reducir la carga en la red y evitar problemas
        delay(30);
      }
      return ESP_OK;
    },
    .user_ctx  = NULL
  };

  if (httpd_start(&server, &config) == ESP_OK) {
    httpd_register_uri_handler(server, &uri_get);
  }
}



'''
