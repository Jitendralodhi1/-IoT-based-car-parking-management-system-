# -IoT-based-car-parking-management-system-

IoT-based car parking management system using Arduino code and components:

Components Required:

Arduino Uno
Ultrasonic Sensor (HC-SR04)
Wi-Fi Module (ESP8266)
LED (for status indication)
Breadboard and jumper wires

Arduino code:

#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>

// Replace with your Wi-Fi credentials
const char* ssid = "Your_SSID";
const char* password = "Your_PASSWORD";

// Replace with your web server IP address
const char* serverIP = "192.168.0.100";
const int serverPort = 80;

// Ultrasonic sensor pins
const int trigPin = 2;
const int echoPin = 3;

// LED pin
const int ledPin = 13;

// Variables
long duration;
int distance;
bool isOccupied = false;

ESP8266WebServer server(80);

void setup() {
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(ledPin, OUTPUT);

  Serial.begin(9600);

  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }

  Serial.println("Connected to WiFi");

  // Start the server
  server.begin();
  Serial.println("Server started");

  // Register handler for updating parking status
  server.on("/update", handleUpdate);
}

void loop() {
  // Measure the distance
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  duration = pulseIn(echoPin, HIGH);
  distance = duration * 0.034 / 2;

  // Check if parking space is occupied
  if (distance < 10) {
    if (!isOccupied) {
      isOccupied = true;
      updateServerStatus(true);
      digitalWrite(ledPin, HIGH);  // Turn on the LED to indicate occupied space
    }
  } else {
    if (isOccupied) {
      isOccupied = false;
      updateServerStatus(false);
      digitalWrite(ledPin, LOW);  // Turn off the LED to indicate vacant space
    }
  }

  server.handleClient();
}

void updateServerStatus(bool status) {
  WiFiClient client;

  if (client.connect(serverIP, serverPort)) {
    // Send HTTP POST request to update server status
    client.println("POST /update-status HTTP/1.1");
    client.println("Host: " + String(serverIP));
    client.println("Content-Type: application/x-www-form-urlencoded");
    client.print("status=");
    client.println(status ? "occupied" : "vacant");
    client.println("Connection: close");
    client.println();
    delay(500);
    client.stop();
  }
}

void handleUpdate() {
  if (server.method() == HTTP_POST) {
    String status = server.arg("status");
    if (status == "occupied") {
      isOccupied = true;
      digitalWrite(ledPin, HIGH);  // Turn on the LED to indicate occupied space
    } else if (status == "vacant") {
      isOccupied = false;
      digitalWrite(ledPin, LOW);  // Turn off the LED to indicate vacant space
    }
    server.send(200, "text/plain", "Status updated");
  } else {
    server.send(400, "text/plain", "Invalid request");
  }
}

