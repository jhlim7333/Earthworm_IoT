# Earthworm_IoT

#include <Debug.h>
#include <JSN270.h>
#include <Arduino.h>
#include <SoftwareSerial.h>
#include <LiquidCrystal.h>
#include <DHT11.h>

//Pin number set up for three-color LED
#define RED 6
#define GREEN 10
#define BLUE 9

//Define variables for connecting of Internet
#define SSID      "iptime"    // your wifi network SSID
#define KEY       "12345678"    // your wifi network password
#define AUTH       "NONE"     // your wifi network security (NONE, WEP, WPA, WPA2)
#define USE_DHCP_IP 1

#if !USE_DHCP_IP
#define MY_IP          "192.168.1.133"
#define SUBNET         "255.255.255.0"
#define GATEWAY        "192.168.1.254"
#endif

#define HOST_IP        "210.117.139.168"
#define REMOTE_PORT    8888
#define PROTOCOL       "TCP"
SoftwareSerial mySerial(3, 2); // RX, TX
JSN270 JSN270(&mySerial);
String sCommand;

//Pin number set up for LCD Module
LiquidCrystal lcd(12, 11, 5, 4, 13, 7);
int pin=8;

//Analog pin number set up for Temperature-Humidity sensor
DHT11 dht11(pin); 
boolean connectt = false;
int err;
float temp, humi;
int pumpPIN = A0;
int resetPin = A3;
int btnControlPin = A4;
int buttonPin = A5;

String msg;
char cc;


void setup() {

  mySerial.begin(9600);
  Serial.begin(9600); //Serial Monitor Setup!
  digitalWrite(resetPin, HIGH);
  pinMode(12, OUTPUT);
  lcd.begin(16, 2); //LCD Module size => 16x2

  digitalWrite(12,LOW);
  
  pinMode(resetPin,OUTPUT);
  pinMode(pumpPIN,OUTPUT);  //Pump Setup!
  pinMode(buttonPin, INPUT);
  pinMode(btnControlPin, OUTPUT);
  err=dht11.read(humi, temp);
  WiFisetup();  //Call func!
} //setup


void loop() {
  pumpControl(1);
  
  if(JSN270.available()) {
    cc = (char)JSN270.read();
    if(cc=='z'){
      // Serial.println("here1");
      pumpControl(2);
    } else if(cc=='q'){
     // Serial.println("here2");
      temp_humi(0);
    }
  }
  colorLED();
  buttonControl();
} //loop

//Function for identify whether button is pressed or not
//When the botton is pressed, LCD module is turned on for 10 seconds.
void buttonControl(){ 
  int buttonState=0;
  buttonState = digitalRead(buttonPin);
  if (buttonState == HIGH) {
        digitalWrite(btnControlPin,HIGH);
        temp_humi(1);
        delay(10000);
        digitalWrite(btnControlPin,LOW);        
  }
  else {
    digitalWrite(btnControlPin,LOW);
  }
}


//This function is for activating the pump,following humidity value automatically
void pumpControl(int state) { 
  if(state==1){
    if(humi<=25) {
      digitalWrite(pumpPIN,HIGH);
      delay(1000);
      digitalWrite(pumpPIN,LOW);  
    }
  } else if(state==2){
    digitalWrite(pumpPIN,HIGH);
    delay(1000);
    digitalWrite(pumpPIN,LOW);
  }

  

}


//This function is display the values of temperature and humidity at LCD module
void temp_humi(int a){ 
  if((err=dht11.read(humi, temp))==0) {
    lcd.clear();
    lcd.setCursor(0,0);
    lcd.print("temp:");
    lcd.print(temp);
    lcd.setCursor(0,1);
    lcd.print("humi:");
    lcd.print(humi);
    lcd.print("%");

    JSN270.print("module1/temp/");
    JSN270.println(temp);
    delay(500);
    JSN270.print("module1/humi/");
    JSN270.println(humi);
  }
  else  {
   lcd.setCursor(0,0);
    lcd.println();
    lcd.print("Error No :");
    lcd.print(err);
    lcd.println();    
  }
  


}// This method have function which get value of temperature and humidity and send that data to LCD Module.


void colorLED(){
  if(temp >=27.00 && temp <=28.00) {
    analogWrite(RED, random(0));
    analogWrite(GREEN, random(255));
    analogWrite(BLUE, random(0));
  }
  else if(temp < 27.00) {
    analogWrite(RED, random(255));
    analogWrite(GREEN, random(0));
    analogWrite(BLUE, random(0));
  }
  else {
    analogWrite(RED, random(0));
    analogWrite(GREEN, random(0));
    analogWrite(BLUE, random(255));
  }
} //Following the value of temperature and humidity, LED'light color is changed.


void WiFisetup(){
  String line1;
  String line2;
  char c;
  digitalWrite(btnControlPin,HIGH);
  Serial.println("----- GrungE~ -----");
  line1 = "---- GrungE ----";
  lcd.setCursor(0,0);
  lcd.print(line1);
  lcd.setCursor(0,1);
  line2 = "Connecting....";
  lcd.print(line2);


  // wait for initilization of JSN270
  delay(5000);
  //JSN270.reset();
  delay(1000);

  //JSN270.prompt();
  JSN270.sendCommand("at+ver\r");
  delay(5);
  lcd.setCursor(0,0);
  while(JSN270.receive((uint8_t *)&c, 1, 1000) > 0) {
    Serial.print((char)c);
  }
  delay(1000);

  #if USE_DHCP_IP
  JSN270.dynamicIP();
  #else
    JSN270.staticIP(MY_IP, SUBNET, GATEWAY);
  #endif    

  if (JSN270.join(SSID, KEY, AUTH)) {
    Serial.println("WiFi connect to " SSID);
  }
  else {
    Serial.println("Failed WiFi connect to " SSID);
    Serial.println("Restart System");
    digitalWrite(resetPin,LOW);

    //return;
  }
  delay(1000);

  JSN270.sendCommand("at+wstat\r");
  delay(5);
  while(JSN270.receive((uint8_t *)&c, 1, 1000) > 0) {
    Serial.print((char)c);
  }
  delay(1000);        

  JSN270.sendCommand("at+nstat\r");
  delay(5);
  while(JSN270.receive((uint8_t *)&c, 1, 1000) > 0) {
    Serial.print((char)c);
  }
  delay(1000);

  if (!JSN270.client(HOST_IP, REMOTE_PORT, PROTOCOL)) {
    Serial.println("Failed connect to " HOST_IP);
    Serial.println("Restart System");
    digitalWrite(resetPin,LOW);
  } else {
    Serial.println("Socket connect to " HOST_IP);
    delay(2000);
    // Enter data mode
    JSN270.sendCommand("at+exit\r");
    delay(5);
    
    JSN270.print("module1/connect");
    delay(500);
   
    lcd.setCursor(0,0);
    lcd.setCursor(0,1);
    line2 = "Connect Wifi !!";
    lcd.print(line2);
    delay(5000);
    digitalWrite(btnControlPin,LOW);
    
  }
}
