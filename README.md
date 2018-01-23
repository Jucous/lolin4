# lolin4
//Code pour domotiser poele COMO RIKA sur contact sec.
//Optimiser la régulaztion et avoir une température plus stable dans la pièce (hysteresis trop ample dans la régulation proposée par //RIKA)
//Utilise un Serveur Domoticz installé sur HTPC linux mint pour gérer la programmation nocturne de temp de consigne.
//Arduino ESP8266-12E sert de relais.
//Je mets en place une jauge pour le servoir à pellet pour envoyer des notifications s'ils viennent à manquer.
//Puis j'installerai un bargraphe.



//biblio OTA:
#include <ArduinoOTA.h>
#include <ESP8266mDNS.h>
#include <WiFiUdp.h>



//bibli réseau:
#include <ESP8266WebServer.h>
#include <ESP8266WiFi.h>
#include <ThingSpeak.h>
#include <ESP8266HTTPClient.h>


// Wi-Fi Settings
const char* ssid = "Livebox-dd04_EXT"; // your wireless network name (SSID)
const char* password = "Jucous3058"; // your Wi-Fi network password
const char* host = "192.168.1.2";
const int   port = 8081;


WiFiClient client;
HTTPClient http;
ESP8266WebServer server ( 80 );


const char* TSserver = "api.thingspeak.com";

unsigned long myChannelNumber = 347455;
const char * myWriteAPIKey = "EBBTFRH7UMEP33ST";
const char * myReadAPIKey = "D30ZT2V4K4Q6E8EL";

float  RDC;
float  Temp_Cons;

void updateGpio(){
  Serial.println("Update GPIO command from DOMOTICZ : ");
  for( int8_t i = 0 ; i < server.args(); i++ ) {
    Serial.print( "[" ); Serial.print( __LINE__ ); Serial.print("] : "); Serial.print( server.argName(i) ); Serial.println( server.arg(i) );
  }
  String gpio = server.arg("id"); 
  String token = server.arg("token");
  if ( token != "123abCde" ) {
   Serial.print( "[" ); Serial.print(__LINE__); Serial.print("] : "); Serial.println("Not authentified ");
    return;
  }
  int8_t etat = server.arg("etat").toInt();
 // pinMode(D4, INPUT);
  pinMode(D4, OUTPUT); 
 
  Serial.print("Update GPIO D4");
  if ( etat == 1 ) {
    digitalWrite(D4, HIGH);
    Serial.print("[");Serial.print(__LINE__);Serial.print("] : ");  Serial.println("Relais On");
  } else if ( etat == 0 ) {
    digitalWrite(D4, LOW);
    Serial.print("[");Serial.print(__LINE__);Serial.print("] : ");   Serial.println("Relais off");
  } else {
   Serial.print("[");Serial.print(__LINE__);Serial.print("] : ");   Serial.println(" Erreur");
  }
  Serial.print("[");Serial.print(__LINE__);Serial.print("] : ");   Serial.print( "etat GPIO : " );
  Serial.println( digitalRead( D4 ) );
  
}
/*
********************************************
14CORE ULTRASONIC DISTANCE SENSOR CODE TEST
********************************************
*/
#define TRIGGER 5
#define ECHO    4

// NodeMCU Pin D1 > TRIGGER | Pin D2 > ECHO



void setup() {
  // put your setup code here, to run once:
  Serial.begin(115200);

/*
********************************************
14CORE ULTRASONIC DISTANCE SENSOR CODE TEST
********************************************
*/
  pinMode(TRIGGER, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(BUILTIN_LED, OUTPUT);

    //___________________________OTA
    Serial.println("Booting");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.waitForConnectResult() != WL_CONNECTED) {
    Serial.println("Connection Failed! Rebooting...");
    delay(5000);
    ESP.restart();
  }

 
  ArduinoOTA.setHostname("Lolin4");

 
  ArduinoOTA.onStart([]() {
    Serial.println("Start");
  });
  ArduinoOTA.onEnd([]() {
    Serial.println("\nEnd");
  });
  ArduinoOTA.onProgress([](unsigned int progress, unsigned int total) {
    Serial.printf("Progress: %u%%\r", (progress / (total / 100)));
  });
  ArduinoOTA.onError([](ota_error_t error) {
    Serial.printf("Error[%u]: ", error);
    if (error == OTA_AUTH_ERROR) Serial.println("Auth Failed");
    else if (error == OTA_BEGIN_ERROR) Serial.println("Begin Failed");
    else if (error == OTA_CONNECT_ERROR) Serial.println("Connect Failed");
    else if (error == OTA_RECEIVE_ERROR) Serial.println("Receive Failed");
    else if (error == OTA_END_ERROR) Serial.println("End Failed");
  });
  ArduinoOTA.begin();
  Serial.println("OTA Ready");
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

    while (WiFi.status() != WL_CONNECTED) {
    delay(500);
       Serial.print(".");
  }
    ThingSpeak.begin(client);
      Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.print(WiFi.localIP()); 
  
digitalWrite(D4,LOW);//


  server.on("/gpio", updateGpio);
  server.begin();
}

void loop() {

    ArduinoOTA.handle();
     server.handleClient();

 RDC = (ThingSpeak.readFloatField(myChannelNumber, 1, myReadAPIKey));
 Serial.println ("");
 Serial.print ("Temp_RDC:");
 Serial.println(RDC);
Serial.println ("");

/*
********************************************
14CORE ULTRASONIC DISTANCE SENSOR CODE TEST
********************************************
*/
  long duration; 
  float distance;
  digitalWrite(TRIGGER, LOW);  
  delayMicroseconds(2); 
  
  digitalWrite(TRIGGER, HIGH);
  delayMicroseconds(10); 
  
  digitalWrite(TRIGGER, LOW);
  duration = pulseIn(ECHO, HIGH);
  distance = (duration/2) / 29.1;
  
  Serial.print(distance);
  Serial.println("cm");

  if (distance <50) {
    if (distance >0) {
      String url = "/json.htm?type=command&param=udevice&idx=25&nvalue=0&svalue=";
        url += String(distance); url += ";";
        

            Serial.print("connecting to ");
  Serial.println(host);
  Serial.print("Requesting URL: ");
  Serial.println(url);
  http.begin(host,port,url);
   Serial.println(url);
  http.begin(host,port,url);

  int httpCode = http.GET();
    if (httpCode) {
      if (httpCode == 200) {
        String payload = http.getString();
        Serial.println("Domoticz response "); 
        Serial.println(payload);
      }
    }
    
    }
  }
  
float percentage;
 percentage= (distance/46)* 100; // 46cm taille de la fosse vide. 
 Serial.print("Percentage:");
 Serial.println (percentage);

 if (distance <60) {
    if (distance >0) {
 String url = "/json.htm?type=command&param=udevice&idx=26&nvalue=0&svalue=";
        url += String(percentage); url += ";";

          Serial.print("connecting to ");
  Serial.println(host);
  Serial.print("Requesting URL: ");
  Serial.println(url);
  http.begin(host,port,url);
   Serial.println(url);
  http.begin(host,port,url);

  int httpCode = http.GET();
    if (httpCode) {
      if (httpCode == 200) {
        String payload = http.getString();
        Serial.println("Domoticz response "); 
        Serial.println(payload);
      }
    }
    }
 }

  
    
 
 
    delay(30000);



}
