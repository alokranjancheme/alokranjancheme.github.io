---
title: IOT Project Wifi Temp Sensor
---

# IOT Project


# Instructions to connect Wemos\_D1\_mini(esp8266)

### **When on the same Network**

- Create a hotspot from your phone/desktop, rename your hotspot as "QMEL\_LAB" and set the password of your hotspot qmel@123.
- Wait for few minutes when a device named "ESP-EAE289" get connected from the hotspot
- Go to setting of your hotspot and copy the connected device ip (192.168.xx.xx) and paste it in your browser
- Now you would be able to see the reading of the sensor.

![image](https://user-images.githubusercontent.com/125783050/223188161-1429a1fe-1d44-4dad-9e1c-c6bb768e24e8.png)


### **When**  **NOT**  **on the same Network**

- Go to the URL [qmellabiitkgp.blogspot.com](http://qmellabiitkgp.blogspot.com/) , click on the give link

![image](https://user-images.githubusercontent.com/125783050/223188411-b1ea0f31-9242-45b1-9b42-c6114957267d.png)

- Open the link Example: 2baa-xxx-110-242-14.in.ngrok.io.( ! This link suppose to change. )

![image](https://user-images.githubusercontent.com/125783050/223188506-436e7770-7f44-45a2-8cb3-d7155299e717.png)

=======================================================================

# Step wise guide to make wifi temperature sensor

- Components required:

 [Generic Wemos D1 Mini V2.2.0 Wifi Internet Development Board Based Esp8266 4Mb Flash Esp-12S Chip](https://www.amazon.in/V2-2-0-Internet-Development-ESP8266-ESP-12S/dp/B077MDHLRC)

![image](https://user-images.githubusercontent.com/125783050/223188757-43970d60-36a5-49c2-a8c8-8e16fbb13a71.png)

  - Any Power supply of 5V/2.4A (any smart phone charger USB MICRO B type)
  - Connecting wire
  - 100 K resistor
  - RTC thermistors ([Generic "10 Pcs Ntc Thermistor Temperature Sensor 10K Ohm Mf52-103 3435 1% "](https://www.amazon.in/Generic-Thermistor-Temperature-Sensor-Mf52-103/dp/B01M8QX36Q))

## Connections:

![image](https://user-images.githubusercontent.com/125783050/223188939-f9405727-c0a3-4c2a-a3c2-0e049f50b601.png)

# [Code](https://github.com/Alok62877/IOT_temp_sensor_-Wifi-/blob/main/code.ino)

```cpp
#include <ESP8266WiFi.h>	
	#include <math.h>
	

	//  http://arduino.esp8266.com/stable/package_esp8266com_index.json
	

	unsigned int Rs = 150000;
	double Vcc = 3.3;
	

	// WiFi parameters
	const char* ssid = "wifi name";
	const char* password = "password";
	

	WiFiServer server(80);
	void setup() {
	Serial.begin(115200);
	delay(10);
	Serial.println();
	// Connect to WiFi network
	WiFi.mode(WIFI_STA);
	Serial.println();
	Serial.println();
	Serial.print("Connecting to ");
	Serial.println(ssid);
	

	WiFi.begin(ssid, password);
	

	while (WiFi.status() != WL_CONNECTED) {
	delay(500);
	Serial.print(".");
	}
	Serial.println("");
	Serial.println("WiFi connected");
	

	// Start the server
	server.begin();
	Serial.println("Server started");
	

	// Print the IP address
	Serial.println(WiFi.localIP());
	}
	

	void loop() {
	

	 delay(1000);
	

	 Serial.println(Thermister(AnalogRead()));
	 float temp = Thermister(AnalogRead());
	
	WiFiClient client = server.available();
	client.println("HTTP/1.1 200 OK");
	client.println("Content-Type: text/html");
	client.println("Connection: close");  // the connection will be closed after completion of the response
	client.println("Refresh: 3");  // refresh the page automatically every 5 sec
	client.println();
	client.println("<!DOCTYPE html>");
	client.println("<html xmlns='http://www.w3.org/1999/xhtml'>");
	client.println("<head>\n<meta charset='UTF-8'>");
	client.println("<title>WifiTempSensor</title>");
	client.println("<link href='https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css' rel='stylesheet'>");
	client.println("</head>\n<body>");
	client.println("<div class='container'>");
	client.println("<div class='panel panel-default' margin:15px>");
	client.println("<div class='panel-heading'>");
	client.println("<H2>Wifi Temp Sensor</H2>");
	client.println("<div class='panel-body' style='background-color: powdergreen'>");
	client.println("<pre>");
	client.print("Temperature (°C)  : ");
	client.println(temp, 2);
	client.println("</pre>");
	client.println("</div>");
	client.println("</div>");
	client.println("</div>");
	client.print("</body>\n</html>");
	}
	

	int AnalogRead() {
	 int val = 0;
	 for(int i = 0; i < 20; i++) {

```

- Use Arduino IDE to upload the code in esp8266
