/**
 * The MySensors Arduino library handles the wireless radio link and protocol
 * between your home built sensors/actuators and HA controller of choice.
 * The sensors forms a self healing radio network with optional repeaters. Each
 * repeater and gateway builds a routing tables in EEPROM which keeps track of the
 * network topology allowing messages to be routed to nodes.
 *
 * Created by Henrik Ekblad <henrik.ekblad@mysensors.org>
 * Copyright (C) 2013-2015 Sensnology AB
 * Full contributor list: https://github.com/mysensors/Arduino/graphs/contributors
 *
 * Documentation: http://www.mysensors.org
 * Support Forum: http://forum.mysensors.org
 *
 * This program is free software; you can redistribute it and/or
 * modify it under the terms of the GNU General Public License
 * version 2 as published by the Free Software Foundation.
 *
 *******************************
 *
 * REVISION HISTORY
 * Version 1.0 - Henrik Ekblad
 *
 * DESCRIPTION
 * The ESP8266 MQTT gateway sends radio network (or locally attached sensors) data to your MQTT broker.
 * The node also listens to MY_MQTT_TOPIC_PREFIX and sends out those messages to the radio network
 *
 * LED purposes:
 * - To use the feature, uncomment any of the MY_DEFAULT_xx_LED_PINs in your sketch
 * - RX (green) - blink fast on radio message recieved. In inclusion mode will blink fast only on presentation recieved
 * - TX (yellow) - blink fast on radio message transmitted. In inclusion mode will blink slowly
 * - ERR (red) - fast blink on error during transmission error or recieve crc error
 *
 * See http://www.mysensors.org/build/esp8266_gateway for wiring instructions.
 * nRF24L01+  ESP8266
 * VCC        VCC
 * CE         GPIO4
 * CSN/CS     GPIO15
 * SCK        GPIO14
 * MISO       GPIO12
 * MOSI       GPIO13
 *
 * Not all ESP8266 modules have all pins available on their external interface.
 * This code has been tested on an ESP-12 module.
 * The ESP8266 requires a certain pin configuration to download code, and another one to run code:
 * - Connect REST (reset) via 10K pullup resistor to VCC, and via switch to GND ('reset switch')
 * - Connect GPIO15 via 10K pulldown resistor to GND
 * - Connect CH_PD via 10K resistor to VCC
 * - Connect GPIO2 via 10K resistor to VCC
 * - Connect GPIO0 via 10K resistor to VCC, and via switch to GND ('bootload switch')
 *
  * Inclusion mode button:
 * - Connect GPIO5 via switch to GND ('inclusion switch')
 *
 * Hardware SHA204 signing is currently not supported!
 *
 * Make sure to fill in your ssid and WiFi password below for ssid & pass.
 */


// Enable debug prints to serial monitor
// #define MY_DEBUG

// Use a bit lower baudrate for serial prints on ESP8266 than default in MyConfig.h
#define MY_BAUD_RATE 9600

// Enables and select radio type (if attached)
#define MY_RADIO_NRF24
//#define MY_RADIO_RFM69

#define MY_GATEWAY_MQTT_CLIENT
#define MY_GATEWAY_ESP8266

// Set this node's subscribe and publish topic prefix
#define MY_MQTT_PUBLISH_TOPIC_PREFIX "mygateway1-out"
#define MY_MQTT_SUBSCRIBE_TOPIC_PREFIX "mygateway1-in"

// Set MQTT client id
#define MY_MQTT_CLIENT_ID "mysensors-2"

// Enable these if your MQTT broker requires usenrame/password
//#define MY_MQTT_USER "username"
//#define MY_MQTT_PASSWORD "password"

// Set WIFI SSID and password
#define MY_ESP8266_SSID "WLAN2"
#define MY_ESP8266_PASSWORD "altes-rumpel-2"

// Set the hostname for the WiFi Client. This is the hostname
// it will pass to the DHCP server if not static.
// #define MY_ESP8266_HOSTNAME "mqtt-sensor-gateway"

// Enable MY_IP_ADDRESS here if you want a static ip address (no DHCP)
// #define MY_IP_ADDRESS 195,147,158,4

// If using static ip you need to define Gateway and Subnet address as well
// #define MY_IP_GATEWAY_ADDRESS 195,147,158,90
// #define MY_IP_SUBNET_ADDRESS 255,255,255,0


// MQTT broker ip address.
#define MY_CONTROLLER_IP_ADDRESS 195, 147, 158, 250
// #define MY_CONTROLLER_IP_ADDRESS 77, 22, 32, 139

// The MQTT broker port to to open
#define MY_PORT 1883

/*
// Enable inclusion mode
#define MY_INCLUSION_MODE_FEATURE
// Enable Inclusion mode button on gateway
#define MY_INCLUSION_BUTTON_FEATURE
// Set inclusion mode duration (in seconds)
#define MY_INCLUSION_MODE_DURATION 60
// Digital pin used for inclusion mode button
#define MY_INCLUSION_MODE_BUTTON_PIN  3
*/
// Set blinking period
#define MY_DEFAULT_LED_BLINK_PERIOD 300
#define MY_WITH_LEDS_BLINKING_INVERSE
// Flash leds on rx/tx/err
#define MY_DEFAULT_ERR_LED_PIN 16  // Error led pin
#define MY_DEFAULT_RX_LED_PIN  5  // Receive led pin
#define MY_DEFAULT_TX_LED_PIN  10  // the PCB, on board LED

#define MY_RAM_ROUTING_TABLE_FEATURE

/*#define RELAY_ON 1  // GPIO value to write to turn on attached relay
#define RELAY_OFF 0 // GPIO value to write to turn off attached relay
#define CHILD_ID_RELAIS 1*/

#include <ESP8266WiFi.h>
#include <MySensors.h>
#include <ESP8266mDNS.h>
#include <ArduinoOTA.h>

/*int RelaisState=0;
unsigned long t0=0;
MyMessage msgLight(CHILD_ID_RELAIS, V_STATUS);*/

void setup()
{
	//pinMode(9,OUTPUT);
	//pinMode(10,OUTPUT);
	//pinMode(22,OUTPUT);
	//pinMode(18,INPUT_PULLUP);
	
	ArduinoOTA.onStart([]() {
		Serial.println("ArduinoOTA start");
	});
	ArduinoOTA.onEnd([]() {
		Serial.println("\nArduinoOTA end");
	});
	ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
		Serial.printf("OTA Progress: %u%%\r", (progress / (total / 100)));
	});
	ArduinoOTA.onError([](ota_error_t error) {
		Serial.printf("Error[%u]: ", error);
		if (error == OTA_AUTH_ERROR) {
			Serial.println("Auth Failed");
		} else if (error == OTA_BEGIN_ERROR) {
			Serial.println("Begin Failed");
		} else if (error == OTA_CONNECT_ERROR) {
			Serial.println("Connect Failed");
		} else if (error == OTA_RECEIVE_ERROR) {
			Serial.println("Receive Failed");
		} else if (error == OTA_END_ERROR) {
			Serial.println("End Failed");
		}
	});
	ArduinoOTA.begin();
	Serial.println("Ready");
	Serial.print("IP address: ");
	Serial.println(WiFi.localIP());
	
}

void before()
{
/*	pinMode(22,OUTPUT);
	pinMode(18,INPUT_PULLUP);
	digitalWrite(22,LOW);
	RelaisState=0;*/
}

void presentation()
{
	// Present locally attached sensors here
	sendSketchInfo("MqttGW_Relais","1.0");
	//present(CHILD_ID_RELAIS,S_BINARY);
}


void loop()
{
	ArduinoOTA.handle();
	/*if((millis()-t0)>30000){
		send(msgLight.set(RelaisState));
	}
	/*if(digitalRead(18)==LOW){
		// Button is pressed toggel Relais
		if(RelaisState==0){
			RelaisState=1;
		}
		else{
			RelaisState=0;
		}
		digitalWrite(22,RelaisState);
		send(msgLight.set(RelaisState));
	}*/
	// Send locally attech sensors data here

}

/*void recieve(const MyMessage &message)
{
	if(message.type==V_STATUS)
	{
		digitalWrite(22,message.getBool()?RELAY_ON:RELAY_OFF);
		RelaisState=message.getInt();
		send(msgLight.set(RelaisState));
		
        Serial.print("Incoming change for sensor:");
        Serial.print(message.sensor);
        Serial.print(", New status: ");
        Serial.println(message.getBool());
		t0=millis();
	}
}
*/