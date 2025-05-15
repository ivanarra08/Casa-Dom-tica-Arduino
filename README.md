# Casa-Dom-tica-Arduino
#include <Wire.h> 
#include <Adafruit_GFX.h> 
#include <Adafruit_SSD1306.h> 
#include <DHT.h> 
#include <Servo.h> 

 

// Definir pines de sensores y actuadores 
#define DHTPIN 2   
#define DHTTYPE DHT11   
#define LEDTEMP 3  // LED para temperatura alta 
#define LEDLUZ 4   // LED para poca luz 
#define LDR_A A0   // Sensor LDR en A0 
#define TRIG_PIN 6 // Pin Trig del HC-SR04 
#define ECHO_PIN 7 // Pin Echo del HC-SR04 
#define SERVO_PIN 5 // Pin del Servo 
#define LED_BLYNK 8 // LED controlado por Blynk 
#define SMOKE A1  
#define BUZZER 9 
#define PIR 10  
#define LED_PIR 11 

int val = A1;  
int umbralhumo = 500; 
int estadoPIR; 
int distancia; 

DHT dht(DHTPIN, DHTTYPE); 
Servo barreraServo; // Servo de la barrera 
// Configuración de la pantalla OLED 
#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 64 
#define OLED_RESET    -1   
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET); 

void setup() { 
    Serial.begin(9600); 
    Serial.println("Iniciando sensores..."); 
    // Iniciar DHT11 
    dht.begin();  
    // Configurar pines como salida 
    pinMode(LEDTEMP, OUTPUT); 
    pinMode(LEDLUZ, OUTPUT); 
    pinMode(TRIG_PIN, OUTPUT); 
    pinMode(ECHO_PIN, INPUT);
    digitalWrite(LEDTEMP, LOW);   
    digitalWrite(LEDLUZ, LOW); 
    // Iniciar servo 
    barreraServo.attach(SERVO_PIN); 
    barreraServo.write(0); // Inicialmente barrera cerrada 
    // Iniciar pantalla OLED 
    if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {  
        Serial.println("⚠️ Error: No se encontró la pantalla OLED"); 
        while (true); 
    } 
    // Alarma y humo 
    pinMode(BUZZER, OUTPUT); 
    pinMode(SMOKE, INPUT); 
    display.clearDisplay(); 
    display.setTextSize(1); 
    display.setTextColor(SSD1306_WHITE); 
    display.setCursor(10, 10); 
    display.println("Sistema Iniciado..."); 
    display.display(); 
    delay(2000);  
    // PIR y LED 
    pinMode(LED_PIR, OUTPUT); 
    pinMode(PIR, INPUT); 
} 
// Función para medir distancia con el sensor HC-SR04 
float medirDistancia() { 
    digitalWrite(TRIG_PIN, LOW); 
    delayMicroseconds(2); 
    digitalWrite(TRIG_PIN, HIGH); 
    delayMicroseconds(10); 
    digitalWrite(TRIG_PIN, LOW); 
    long duracion = pulseIn(ECHO_PIN, HIGH); 
    float distancia = duracion * 0.034 / 2; 
    return distancia; 
} 

void loop() { 
    float temperatura = dht.readTemperature();   
    float humedad = dht.readHumidity();   
    int luz = analogRead(LDR_A); // Leer nivel de luz 
    float distancia = medirDistancia(); // Medir distancia con HC-SR04  
    // Convertir lectura de LDR a porcentaje 
    int luzPercent = map(luz, 0, 1023, 100, 0);  
    if (isnan(temperatura) || isnan(humedad)) { 
        Serial.println("⚠️ Error al leer el sensor DHT11"); 
        return; 
    } 
    // Control del LED de temperatura (> 17°C) 
    if (temperatura > 17) { 
        digitalWrite(LEDTEMP, HIGH); 
        Serial.println("🔥 LED ENCENDIDO: Temperatura alta!"); 
    } else { 
        digitalWrite(LEDTEMP, LOW); 
    } 
    // Control del LED de Luz (Encender si la luz < 30%) 
    bool lucesEncendidas = false; 
    if (luzPercent > 50) {   
        digitalWrite(LEDLUZ, LOW);// mirar bien esto que esta raro 
        lucesEncendidas = true; 
        Serial.println("💡 LUZ ENCENDIDA: Oscuridad detectada!"); 
    } else { 
        digitalWrite(LEDLUZ, HIGH); 
    } 
    // Control de la barrera del garaje 
    bool barreraAbierta = false; 
    if (distancia < 20) { // Si un coche está a menos de 20 cm 
        barreraServo.write(90); // Abrir barrera (90°) 
        barreraAbierta = true; 
        Serial.print (distancia); 
        Serial.println("🚗 Coche detectado: Barrera abierta!"); 
    } else { 
        barreraServo.write(0); // Cerrar barrera (0°) 
    } 
    // Humo y alarma 
    val = analogRead(SMOKE); // Leer nivel de humo 
    if (val < umbralhumo) { 
        digitalWrite(BUZZER, LOW); // Apagar buzzer si no hay humo 
    } else { 
        digitalWrite(BUZZER, HIGH); // Activar buzzer si se detecta humo 
    } 

    // PIR y LED 
    estadoPIR = digitalRead(PIR);  // Leer el estado del PIR      
    if (estadoPIR == HIGH) {   
        digitalWrite(LED_PIR, HIGH); // Encender LED PIR 
        delay(500); 
        digitalWrite(LED_PIR, LOW);  // Apagar LED PIR 
        delay(500); 
        Serial.println("Movimiento detectado!"); 
    } else { 
        digitalWrite(LED_PIR, LOW);  // Apagar LED PIR 
    }
    delay(100);  // Tiempo de espera para la lectura 
    // Mostrar datos en Monitor Serie 
    Serial.print("🌡️ Temp: "); 
    Serial.print(temperatura); 
    Serial.print("°C  |  💧 Hum: "); 
    Serial.print(humedad); 
    Serial.print("%  |  💡 Luz: "); 
    Serial.print(luzPercent); 
    Serial.print("%  |  📏 Dist: "); 
    Serial.print(distancia); 
    Serial.println(" cm");
    // Mostrar datos en la pantalla OLED 
    display.clearDisplay(); 
    display.setTextSize(1);   
    display.setCursor(10, 10); 
    display.print("Temp: "); 
    display.print(temperatura); 
    display.println(" C"); 
    display.setCursor(10, 25); 
    display.print("Hum: "); 
    display.print(humedad); 
    display.println(" %");  
    display.setCursor(10, 40); 
    display.print("Luz: "); 
    display.print(luzPercent); 
    display.println(" %"); 
    if (lucesEncendidas) { 
        display.setCursor(10, 55); 
        display.println("LUZ ENCENDIDA!"); 
    } 

    if (barreraAbierta) { 
        display.setCursor(10, 55); 
        display.println("BARRERA ABIERTA!"); 
    }
    display.display(); // Mostrar en pantalla 
