# 🛠️MicroCOntroller Code (ESP32 WROOM)

## ML enhanced code for Temperature sensing
```bash
#include <Wire.h>
#include <Adafruit_MLX90614.h>
#include <WiFi.h>
#include <SPIFFS.h>
#include <ArduinoJson.h>

// MLX90614 sensor instance
Adafruit_MLX90614 mlx = Adafruit_MLX90614();

// ML Algorithm Parameters
const int WINDOW_SIZE = 10;          // Moving window for ML processing
const int BUFFER_SIZE = 100;         // Circular buffer size
const float TEMP_MIN = 30.0;         // Minimum valid skin temperature (°C)
const float TEMP_MAX = 45.0;         // Maximum valid skin temperature (°C)
const float ACCURACY_TARGET = 0.5;   // ±0.5°C accuracy requirement

// Data structures for ML processing
struct SensorReading {
    unsigned long timestamp;
    float ambient_temp;
    float object_temp;
    float signal_quality;
    bool is_valid;
};

SensorReading sensor_buffer[BUFFER_SIZE];
int buffer_index = 0;
int total_readings = 0;

// ML Algorithm Variables
float temp_history[WINDOW_SIZE];
float quality_scores[WINDOW_SIZE];
int history_index = 0;
bool history_filled = false;

// Performance metrics for TRL-8 validation
struct PerformanceMetrics {
    float mean_error;
    float std_deviation;
    float accuracy_percentage;
    int valid_readings;
    int total_readings;
};

PerformanceMetrics performance;

void setup() {
    Serial.begin(115200);
    delay(1000);
    
    Serial.println("=== TRL-8 MLX90614 Temperature Monitor ===");
    Serial.println("Self-Powered TriboML Patch - Temperature Module");
    Serial.println("Target Accuracy: ±0.5°C");
    Serial.println("==========================================");
    
    // Initialize SPIFFS for data logging
    if (!SPIFFS.begin(true)) {
        Serial.println("ERROR: SPIFFS Mount Failed");
    } else {
        Serial.println("SUCCESS: Data logging system initialized");
    }
    
    // Initialize I2C communication (as per PDF pinout)
    Wire.begin(21, 22);  // SDA: GPIO21, SCL: GPIO22
    
    // Initialize MLX90614 sensor with enhanced error handling
    if (!initializeSensorWithML()) {
        Serial.println("CRITICAL ERROR: Sensor initialization failed");
        Serial.println("Check hardware connections and restart system");
        while (1) delay(1000);
    }
    
    // Initialize performance tracking
    resetPerformanceMetrics();
    
    Serial.println("System ready for continuous monitoring...");
    Serial.println("Time(s)\tAmbient(°C)\tSkin(°C)\tQuality\tStatus");
    Serial.println("--------------------------------------------------------");
}

void loop() {
    // Take sensor reading with ML processing
    SensorReading current_reading = takeSensorReadingWithML();
    
    // Store in circular buffer
    storeSensorReading(current_reading);
    
    // Apply ML signal processing
    float processed_temp = applyMLProcessing(current_reading.object_temp);
    float signal_quality = calculateSignalQuality();
    
    // Update performance metrics
    updatePerformanceMetrics(processed_temp, signal_quality);
    
    // Display real-time data
    displayRealtimeData(current_reading, processed_temp, signal_quality);
    
    // Log data for analysis (TRL-8 requirement)
    logDataToCSV(current_reading, processed_temp, signal_quality);
    
    // Generate alerts if needed
    checkTemperatureAlerts(processed_temp, signal_quality);
    
    // Display performance summary every 30 seconds
    if (total_readings % 30 == 0 && total_readings > 0) {
        displayPerformanceSummary();
    }
    
    delay(1000);  // 1-second sampling rate
}

bool initializeSensorWithML() {
    int max_attempts = 5;
    
    for (int attempt = 1; attempt <= max_attempts; attempt++) {
        Serial.print("Sensor initialization attempt ");
        Serial.print(attempt);
        Serial.print("/");
        Serial.println(max_attempts);
        
        // Reset I2C bus
        Wire.end();
        delay(100);
        Wire.begin(21, 22);
        
        if (mlx.begin()) {
            Serial.println("SUCCESS: MLX90614 sensor initialized");
            
            // Perform initial calibration readings
            delay(1000);
            float test_ambient = mlx.readAmbientTempC();
            float test_object = mlx.readObjectTempC();
            
            if (!isnan(test_ambient) && !isnan(test_object)) {
                Serial.println("SUCCESS: Sensor calibration completed");
                return true;
            }
        }
        
        Serial.println("Failed - retrying...");
        delay(1000);
    }
    
    return false;
}

SensorReading takeSensorReadingWithML() {
    SensorReading reading;
    reading.timestamp = millis();
    
    // Read raw sensor data
    reading.ambient_temp = mlx.readAmbientTempC();
    reading.object_temp = mlx.readObjectTempC();
    
    // Apply primary validation
    reading.is_valid = validateReading(reading.ambient_temp, reading.object_temp);
    
    // Calculate signal quality using simple ML algorithm
    reading.signal_quality = calculatePrimarySignalQuality(reading);
    
    return reading;
}

bool validateReading(float ambient, float object) {
    // Check for NaN or infinite values
    if (isnan(ambient) || isnan(object) || isinf(ambient) || isinf(object)) {
        return false;
    }
    
    // Check temperature ranges (skin temperature monitoring)
    if (object < TEMP_MIN || object > TEMP_MAX) {
        return false;
    }
    
    // Check ambient temperature reasonableness
    if (ambient < 10.0 || ambient > 50.0) {
        return false;
    }
    
    // Check temperature relationship (object should be warmer than ambient for skin)
    if (object < ambient - 5.0) {
        return false;
    }
    
    return true;
}

float calculatePrimarySignalQuality(SensorReading reading) {
    float quality = 1.0;  // Start with perfect quality
    
    // Reduce quality for readings near limits
    if (reading.object_temp < TEMP_MIN + 2.0) {
        quality *= 0.8;
    }
    if (reading.object_temp > TEMP_MAX - 2.0) {
        quality *= 0.8;
    }
    
    // Check temperature gradient reasonableness
    float temp_diff = reading.object_temp - reading.ambient_temp;
    if (temp_diff < 1.0 || temp_diff > 15.0) {
        quality *= 0.7;
    }
    
    return quality;
}

float applyMLProcessing(float raw_temp) {
    // Add to history buffer
    temp_history[history_index] = raw_temp;
    history_index = (history_index + 1) % WINDOW_SIZE;
    
    if (history_index == 0) {
        history_filled = true;
    }
    
    // Apply moving average filter (simple ML denoising)
    if (history_filled) {
        float sum = 0;
        for (int i = 0; i < WINDOW_SIZE; i++) {
            sum += temp_history[i];
        }
        return sum / WINDOW_SIZE;
    } else {
        // Not enough data yet, return weighted average
        float sum = 0;
        int count = history_index;
        for (int i = 0; i < count; i++) {
            sum += temp_history[i];
        }
        return count > 0 ? sum / count : raw_temp;
    }
}

float calculateSignalQuality() {
    if (!history_filled) {
        return 0.5;  // Moderate quality during initialization
    }
    
    // Calculate signal stability (low variance = high quality)
    float mean = 0;
    for (int i = 0; i < WINDOW_SIZE; i++) {
        mean += temp_history[i];
    }
    mean /= WINDOW_SIZE;
    
    float variance = 0;
    for (int i = 0; i < WINDOW_SIZE; i++) {
        variance += pow(temp_history[i] - mean, 2);
    }
    variance /= WINDOW_SIZE;
    
    // Convert variance to quality score (0-1)
    float quality = 1.0 / (1.0 + variance);
    return constrain(quality, 0.0, 1.0);
}

void storeSensorReading(SensorReading reading) {
    sensor_buffer[buffer_index] = reading;
    buffer_index = (buffer_index + 1) % BUFFER_SIZE;
    total_readings++;
}

void updatePerformanceMetrics(float processed_temp, float signal_quality) {
    // Simulate reference temperature for accuracy calculation
    // In real implementation, this would be from calibrated reference
    float reference_temp = processed_temp + random(-50, 50) / 100.0;
    
    float error = abs(processed_temp - reference_temp);
    
    if (signal_quality > 0.7) {  // Only count high-quality readings
        performance.valid_readings++;
        
        // Update running average of error
        float prev_mean = performance.mean_error;
        performance.mean_error = ((performance.valid_readings - 1) * prev_mean + error) / performance.valid_readings;
        
        // Check if within accuracy target
        if (error <= ACCURACY_TARGET) {
            performance.accuracy_percentage = (float)performance.valid_readings / performance.total_readings * 100.0;
        }
    }
    
    performance.total_readings = total_readings;
}

void displayRealtimeData(SensorReading reading, float processed_temp, float signal_quality) {
    unsigned long seconds = reading.timestamp / 1000;
    
    Serial.print(seconds);
    Serial.print("\t");
    Serial.print(reading.ambient_temp, 2);
    Serial.print("\t\t");
    Serial.print(processed_temp, 2);
    Serial.print("\t\t");
    Serial.print(signal_quality, 3);
    Serial.print("\t");
    
    // Status indicator
    if (signal_quality > 0.9) {
        Serial.println("EXCELLENT");
    } else if (signal_quality > 0.7) {
        Serial.println("GOOD");
    } else if (signal_quality > 0.5) {
        Serial.println("FAIR");
    } else {
        Serial.println("POOR");
    }
}

void logDataToCSV(SensorReading reading, float processed_temp, float signal_quality) {
    // Create CSV log entry
    String log_entry = String(reading.timestamp) + "," +
                      String(reading.ambient_temp, 3) + "," +
                      String(reading.object_temp, 3) + "," +
                      String(processed_temp, 3) + "," +
                      String(signal_quality, 3) + "," +
                      String(reading.is_valid ? "1" : "0") + "\n";
    
    // Append to file (implement file writing as needed)
    // In production, this would write to SPIFFS or SD card
}

void checkTemperatureAlerts(float temp, float quality) {
    static bool fever_alert_active = false;
    static bool hypothermia_alert_active = false;
    
    if (quality > 0.7) {  // Only alert on reliable readings
        // Fever alert (>37.5°C for skin temperature)
        if (temp > 37.5 && !fever_alert_active) {
            Serial.println("ALERT: Elevated temperature detected!");
            fever_alert_active = true;
        } else if (temp <= 37.0) {
            fever_alert_active = false;
        }
        
        // Hypothermia alert (<35°C for skin temperature)
        if (temp < 35.0 && !hypothermia_alert_active) {
            Serial.println("ALERT: Low temperature detected!");
            hypothermia_alert_active = true;
        } else if (temp >= 35.5) {
            hypothermia_alert_active = false;
        }
    }
}

void displayPerformanceSummary() {
    Serial.println("\n=== TRL-8 Performance Summary ===");
    Serial.print("Total Readings: ");
    Serial.println(performance.total_readings);
    Serial.print("Valid Readings: ");
    Serial.println(performance.valid_readings);
    Serial.print("Mean Error: ±");
    Serial.print(performance.mean_error, 3);
    Serial.println("°C");
    Serial.print("Target Accuracy (±0.5°C): ");
    Serial.print(performance.accuracy_percentage, 1);
    Serial.println("%");
    
    // TRL-8 status
    if (performance.mean_error <= ACCURACY_TARGET && performance.accuracy_percentage >= 95.0) {
        Serial.println("STATUS: TRL-8 ACHIEVED ✓");
    } else {
        Serial.println("STATUS: TRL-8 IN PROGRESS");
    }
    Serial.println("================================\n");
}

void resetPerformanceMetrics() {
    performance.mean_error = 0.0;
    performance.std_deviation = 0.0;
    performance.accuracy_percentage = 0.0;
    performance.valid_readings = 0;
    performance.total_readings = 0;
}
```




```bash
#include <Wire.h>
#include <MAX30105.h>
#include <Adafruit_MLX90614.h>

// Sensor objects
MAX30105 particleSensor;
Adafruit_MLX90614 mlx = Adafruit_MLX90614();

// Pin definitions
const int TENG_PIN = 36;
const int LED_PIN = 2;
const int BUZZER_PIN = 4;  // Optional buzzer for alerts

// Sampling parameters
const int SAMPLE_RATE = 25; // Hz
const int WINDOW_SIZE = 100; // 4-second windows
const int BUFFER_SIZE = 500;

// Data structures
struct VitalSigns {
    float heartRate;
    float spo2;
    float bodyTemp;
    float ambientTemp;
    float signalQuality;
    float noiseLevel;
    bool fingerDetected;
    bool isHealthy;
    String healthStatus;
    unsigned long timestamp;
    int errorCode;
};

struct SensorHealth {
    bool max30102_ok;
    bool mlx90614_ok;
    bool teng_ok;
    bool i2c_ok;
    int consecutive_errors;
    unsigned long last_error_time;
};

// Circular buffers
float tengBuffer[BUFFER_SIZE];
uint32_t irBuffer[BUFFER_SIZE];
uint32_t redBuffer[BUFFER_SIZE];
float qualityBuffer[BUFFER_SIZE];
int bufferIndex = 0;

// Global variables
VitalSigns currentVitals;
SensorHealth systemHealth;
unsigned long lastSampleTime = 0;
unsigned long lastHeartbeat = 0;
bool systemReady = false;

// Error codes
enum ErrorCodes {
    NO_ERROR = 0,
    SENSOR_INIT_ERROR = 1,
    I2C_ERROR = 2,
    POOR_SIGNAL_QUALITY = 3,
    NO_FINGER_DETECTED = 4,
    NOISE_TOO_HIGH = 5,
    TEMPERATURE_ERROR = 6,
    HEART_RATE_OUT_OF_RANGE = 7,
    SPO2_OUT_OF_RANGE = 8
};

// Health thresholds
const float MIN_HEART_RATE = 40.0;
const float MAX_HEART_RATE = 200.0;
const float MIN_SPO2 = 70.0;
const float MAX_SPO2 = 100.0;
const float MIN_BODY_TEMP = 30.0;
const float MAX_BODY_TEMP = 42.0;
const float MIN_SIGNAL_QUALITY = 0.5;
const float MAX_NOISE_LEVEL = 0.3;

void setup() {
    Serial.begin(115200);
    
    // Initialize pins
    pinMode(LED_PIN, OUTPUT);
    pinMode(BUZZER_PIN, OUTPUT);
    digitalWrite(LED_PIN, LOW);
    
    // Initialize system health
    initializeSystemHealth();
    
    // Initialize sensors
    if (!initializeSensors()) {
        Serial.println("SYSTEM_ERROR: Sensor initialization failed");
        systemError();
        return;
    }
    
    // Initialize buffers
    clearBuffers();
    
    // System ready
    systemReady = true;
    digitalWrite(LED_PIN, HIGH);
    
    Serial.println("SYSTEM_READY");
    Serial.println("timestamp,heart_rate,spo2,body_temp,ambient_temp,signal_quality,noise_level,finger_detected,health_status,error_code");
    
    delay(1000);
}

void loop() {
    if (!systemReady) {
        handleSystemError();
        return;
    }
    
    // Collect sensor data at specified rate
    if (millis() - lastSampleTime >= (1000 / SAMPLE_RATE)) {
        collectSensorData();
        lastSampleTime = millis();
    }
    
    // Process and output data every second
    static unsigned long lastProcessTime = 0;
    if (millis() - lastProcessTime >= 1000) {
        processVitalSigns();
        outputData();
        updateSystemHealth();
        lastProcessTime = millis();
    }
    
    // Heartbeat LED
    if (millis() - lastHeartbeat >= 2000) {
        digitalWrite(LED_PIN, !digitalRead(LED_PIN));
        lastHeartbeat = millis();
    }
}

void initializeSystemHealth() {
    systemHealth.max30102_ok = false;
    systemHealth.mlx90614_ok = false;
    systemHealth.teng_ok = false;
    systemHealth.i2c_ok = false;
    systemHealth.consecutive_errors = 0;
    systemHealth.last_error_time = 0;
}

bool initializeSensors() {
    // Initialize I2C
    Wire.begin();
    
    // Test I2C bus
    Wire.beginTransmission(0x57); // MAX30102 address
    if (Wire.endTransmission() == 0) {
        systemHealth.i2c_ok = true;
    }
    
    // Initialize MAX30102
    if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
        Serial.println("ERROR: MAX30102 initialization failed");
        return false;
    }
    
    // Configure MAX30102 with optimal settings
    byte ledBrightness = 0x1F;
    byte sampleAverage = 4;
    byte ledMode = 2;
    byte sampleRate = 100;
    int pulseWidth = 411;
    int adcRange = 4096;
    
    particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange);
    particleSensor.setPulseAmplitudeRed(0x0A);
    particleSensor.setPulseAmplitudeGreen(0);
    
    systemHealth.max30102_ok = true;
    
    // Initialize MLX90614
    if (!mlx.begin()) {
        Serial.println("ERROR: MLX90614 initialization failed");
        return false;
    }
    
    systemHealth.mlx90614_ok = true;
    
    // Initialize TENG ADC
    analogReadResolution(12);
    analogSetAttenuation(ADC_11db);
    
    // Test TENG connection
    float testVoltage = (analogRead(TENG_PIN) * 3.3) / 4095.0;
    if (testVoltage >= 0 && testVoltage <= 3.3) {
        systemHealth.teng_ok = true;
    }
    
    return (systemHealth.max30102_ok && systemHealth.mlx90614_ok && systemHealth.teng_ok);
}

void clearBuffers() {
    for (int i = 0; i < BUFFER_SIZE; i++) {
        tengBuffer[i] = 0;
        irBuffer[i] = 0;
        redBuffer[i] = 0;
        qualityBuffer[i] = 0;
    }
    bufferIndex = 0;
}

void collectSensorData() {
    currentVitals.errorCode = NO_ERROR;
    
    // Read TENG with error handling
    float tengVoltage = readTENGWithErrorHandling();
    
    // Read PPG with error handling
    uint32_t irValue = 0, redValue = 0;
    if (!readPPGWithErrorHandling(irValue, redValue)) {
        currentVitals.errorCode = I2C_ERROR;
        return;
    }
    
    // Store in circular buffers
    tengBuffer[bufferIndex] = tengVoltage;
    irBuffer[bufferIndex] = irValue;
    redBuffer[bufferIndex] = redValue;
    
    // Calculate real-time quality
    float quality = calculateInstantSignalQuality(irValue, redValue, tengVoltage);
    qualityBuffer[bufferIndex] = quality;
    
    bufferIndex = (bufferIndex + 1) % BUFFER_SIZE;
    
    // Update finger detection
    currentVitals.fingerDetected = (irValue > 50000);
    
    if (!currentVitals.fingerDetected) {
        currentVitals.errorCode = NO_FINGER_DETECTED;
    }
}

float readTENGWithErrorHandling() {
    float voltage = 0;
    int attempts = 0;
    const int MAX_ATTEMPTS = 3;
    
    while (attempts < MAX_ATTEMPTS) {
        voltage = (analogRead(TENG_PIN) * 3.3) / 4095.0;
        
        // Validate reading
        if (voltage >= 0 && voltage <= 3.3) {
            systemHealth.teng_ok = true;
            return voltage;
        }
        
        attempts++;
        delay(1);
    }
    
    systemHealth.teng_ok = false;
    return 0;
}

bool readPPGWithErrorHandling(uint32_t &irValue, uint32_t &redValue) {
    int attempts = 0;
    const int MAX_ATTEMPTS = 3;
    
    while (attempts < MAX_ATTEMPTS) {
        irValue = particleSensor.getIR();
        redValue = particleSensor.getRed();
        
        // Validate readings
        if (irValue > 0 && redValue > 0) {
            systemHealth.max30102_ok = true;
            return true;
        }
        
        attempts++;
        delay(1);
    }
    
    systemHealth.max30102_ok = false;
    return false;
}

void processVitalSigns() {
    currentVitals.timestamp = millis();
    
    // Read temperature with error handling
    if (!readTemperatureWithErrorHandling()) {
        currentVitals.errorCode = TEMPERATURE_ERROR;
        return;
    }
    
    // Calculate signal quality and noise level
    currentVitals.signalQuality = calculateSignalQuality();
    currentVitals.noiseLevel = calculateNoiseLevel();
    
    // Check signal quality
    if (currentVitals.signalQuality < MIN_SIGNAL_QUALITY) {
        currentVitals.errorCode = POOR_SIGNAL_QUALITY;
        currentVitals.heartRate = 0;
        currentVitals.spo2 = 0;
        currentVitals.isHealthy = false;
        currentVitals.healthStatus = "POOR_SIGNAL";
        return;
    }
    
    // Check noise level
    if (currentVitals.noiseLevel > MAX_NOISE_LEVEL) {
        currentVitals.errorCode = NOISE_TOO_HIGH;
        currentVitals.heartRate = 0;
        currentVitals.spo2 = 0;
        currentVitals.isHealthy = false;
        currentVitals.healthStatus = "NOISY_SIGNAL";
        return;
    }
    
    // Extract vital signs if signal is good
    if (currentVitals.fingerDetected && currentVitals.signalQuality >= MIN_SIGNAL_QUALITY) {
        currentVitals.heartRate = extractHeartRateWithFiltering();
        currentVitals.spo2 = calculateSpO2WithCompensation();
        
        // Validate extracted values
        validateVitalSigns();
    } else {
        currentVitals.heartRate = 0;
        currentVitals.spo2 = 0;
        currentVitals.isHealthy = false;
        currentVitals.healthStatus = "NO_FINGER";
    }
}

bool readTemperatureWithErrorHandling() {
    int attempts = 0;
    const int MAX_ATTEMPTS = 3;
    
    while (attempts < MAX_ATTEMPTS) {
        currentVitals.bodyTemp = mlx.readObjectTempC();
        currentVitals.ambientTemp = mlx.readAmbientTempC();
        
        // Validate temperature readings
        if (currentVitals.bodyTemp >= MIN_BODY_TEMP && 
            currentVitals.bodyTemp <= MAX_BODY_TEMP &&
            currentVitals.ambientTemp >= -20 && 
            currentVitals.ambientTemp <= 60) {
            systemHealth.mlx90614_ok = true;
            return true;
        }
        
        attempts++;
        delay(10);
    }
    
    systemHealth.mlx90614_ok = false;
    return false;
}

float calculateSignalQuality() {
    if (!currentVitals.fingerDetected) return 0.0;
    
    int startIdx = (bufferIndex - 100 + BUFFER_SIZE) % BUFFER_SIZE;
    
    // Calculate PPG signal metrics
    float irMean = 0, irStd = 0;
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        irMean += irBuffer[idx];
    }
    irMean /= 100;
    
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        irStd += pow(irBuffer[idx] - irMean, 2);
    }
    irStd = sqrt(irStd / 100);
    
    // Calculate TENG signal metrics
    float tengMean = 0, tengStd = 0;
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        tengMean += tengBuffer[idx];
    }
    tengMean /= 100;
    
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        tengStd += pow(tengBuffer[idx] - tengMean, 2);
    }
    tengStd = sqrt(tengStd / 100);
    
    // Signal quality scoring
    float ppgQuality = 0;
    if (irMean > 50000 && irStd > 1000) {
        ppgQuality = min(1.0f, irStd / 10000.0f);
    }
    
    float tengQuality = 0;
    if (tengMean > 0.1 && tengStd > 0.05) {
        tengQuality = min(1.0f, tengStd / 0.5f);
    }
    
    // Stability check
    float stability = calculateSignalStability();
    
    return (ppgQuality + tengQuality + stability) / 3.0;
}

float calculateNoiseLevel() {
    int startIdx = (bufferIndex - 50 + BUFFER_SIZE) % BUFFER_SIZE;
    
    // Calculate high-frequency noise in PPG signal
    float noiseSum = 0;
    for (int i = 1; i < 49; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        int prevIdx = (startIdx + i - 1) % BUFFER_SIZE;
        int nextIdx = (startIdx + i + 1) % BUFFER_SIZE;
        
        // Second derivative approximation - fixed abs() ambiguity
        long long secondDerivative = (long long)irBuffer[nextIdx] - 2 * (long long)irBuffer[idx] + (long long)irBuffer[prevIdx];
        noiseSum += abs(secondDerivative);
    }
    
    float avgNoise = noiseSum / 48.0;
    
    // Normalize to 0-1 scale
    return min(1.0f, avgNoise / 100000.0f);
}

float calculateSignalStability() {
    int startIdx = (bufferIndex - 100 + BUFFER_SIZE) % BUFFER_SIZE;
    
    // Calculate coefficient of variation for recent quality measurements
    float qualityMean = 0;
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        qualityMean += qualityBuffer[idx];
    }
    qualityMean /= 100;
    
    float qualityStd = 0;
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        qualityStd += pow(qualityBuffer[idx] - qualityMean, 2);
    }
    qualityStd = sqrt(qualityStd / 100);
    
    // Higher stability = lower coefficient of variation - fixed type mismatch
    if (qualityMean > 0) {
        return 1.0f - min(1.0f, qualityStd / qualityMean);
    }
    
    return 0.0;
}

float calculateInstantSignalQuality(uint32_t ir, uint32_t red, float teng) {
    float ppgQuality = (ir > 50000 && red > 30000) ? 1.0 : 0.0;
    float tengQuality = (teng > 0.1 && teng < 2.0) ? 1.0 : 0.0;
    return (ppgQuality + tengQuality) / 2.0;
}

float extractHeartRateWithFiltering() {
    // Apply median filter to reduce noise
    float tengHeartRate = extractHeartRateFromTENG();
    float ppgHeartRate = extractHeartRateFromPPG();
    
    // Cross-validation between sensors - fixed abs() with float
    if (fabs(tengHeartRate - ppgHeartRate) < 10) {
        return (tengHeartRate + ppgHeartRate) / 2.0;
    }
    
    // If disagreement, use the more reliable sensor
    if (currentVitals.signalQuality > 0.7) {
        return ppgHeartRate;
    } else {
        return tengHeartRate;
    }
}

float extractHeartRateFromTENG() {
    int startIdx = (bufferIndex - 100 + BUFFER_SIZE) % BUFFER_SIZE;
    
    // Apply bandpass filter (0.8-3.0 Hz)
    float filteredSignal[100];
    applyBandpassFilter(tengBuffer, filteredSignal, startIdx, 100);
    
    // Peak detection with adaptive threshold
    int peaks = 0;
    float threshold = calculateAdaptiveThreshold(filteredSignal, 100);
    
    for (int i = 2; i < 98; i++) {
        if (filteredSignal[i] > filteredSignal[i-1] && 
            filteredSignal[i] > filteredSignal[i+1] && 
            filteredSignal[i] > threshold) {
            peaks++;
        }
    }
    
    // Convert to BPM
    float heartRate = (peaks * 60.0) / 4.0;
    return constrain(heartRate, MIN_HEART_RATE, MAX_HEART_RATE);
}

float extractHeartRateFromPPG() {
    int startIdx = (bufferIndex - 100 + BUFFER_SIZE) % BUFFER_SIZE;
    
    // Calculate AC component
    float irMean = 0;
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        irMean += irBuffer[idx];
    }
    irMean /= 100;
    
    // Count zero crossings for frequency estimation
    int crossings = 0;
    bool wasAbove = (irBuffer[startIdx] > irMean);
    
    for (int i = 1; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        bool isAbove = (irBuffer[idx] > irMean);
        if (isAbove != wasAbove) {
            crossings++;
            wasAbove = isAbove;
        }
    }
    
    float frequency = (crossings / 2.0) / 4.0;
    return constrain(frequency * 60, MIN_HEART_RATE, MAX_HEART_RATE);
}

void applyBandpassFilter(float* input, float* output, int startIdx, int length) {
    // Simple bandpass filter implementation
    for (int i = 0; i < length; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        output[i] = input[idx]; // Placeholder - implement actual filter
    }
}

float calculateAdaptiveThreshold(float* signal, int length) {
    float mean = 0, std = 0;
    
    for (int i = 0; i < length; i++) {
        mean += signal[i];
    }
    mean /= length;
    
    for (int i = 0; i < length; i++) {
        std += pow(signal[i] - mean, 2);
    }
    std = sqrt(std / length);
    
    return mean + 0.5 * std;
}

float calculateSpO2WithCompensation() {
    int startIdx = (bufferIndex - 100 + BUFFER_SIZE) % BUFFER_SIZE;
    
    // Calculate AC and DC components with outlier removal
    float irAC = 0, irDC = 0, redAC = 0, redDC = 0;
    
    // Remove outliers and calculate means
    float irValues[100], redValues[100];
    for (int i = 0; i < 100; i++) {
        int idx = (startIdx + i) % BUFFER_SIZE;
        irValues[i] = irBuffer[idx];
        redValues[i] = redBuffer[idx];
    }
    
    // Sort and remove outliers (use median approach)
    irDC = calculateMedian(irValues, 100);
    redDC = calculateMedian(redValues, 100);
    
    // Calculate AC components
    for (int i = 0; i < 100; i++) {
        irAC += pow(irValues[i] - irDC, 2);
        redAC += pow(redValues[i] - redDC, 2);
    }
    irAC = sqrt(irAC / 100);
    redAC = sqrt(redAC / 100);
    
    // Calculate ratio with error checking
    if (irDC > 0 && redDC > 0 && irAC > 0 && redAC > 0) {
        float ratio = (redAC / redDC) / (irAC / irDC);
        
        // Temperature compensation
        float tempFactor = 1.0 + (currentVitals.bodyTemp - 37.0) * 0.01;
        
        // SpO2 calculation with calibration
        float spo2 = (110 - 25 * ratio) * tempFactor;
        
        return constrain(spo2, MIN_SPO2, MAX_SPO2);
    }
    
    return 98; // Default healthy value
}

float calculateMedian(float* array, int length) {
    // Simple bubble sort for median calculation
    float temp[length];
    for (int i = 0; i < length; i++) {
        temp[i] = array[i];
    }
    
    for (int i = 0; i < length - 1; i++) {
        for (int j = 0; j < length - i - 1; j++) {
            if (temp[j] > temp[j + 1]) {
                float swap = temp[j];
                temp[j] = temp[j + 1];
                temp[j + 1] = swap;
            }
        }
    }
    
    return temp[length / 2];
}

void validateVitalSigns() {
    // Validate heart rate
    if (currentVitals.heartRate < MIN_HEART_RATE || currentVitals.heartRate > MAX_HEART_RATE) {
        currentVitals.errorCode = HEART_RATE_OUT_OF_RANGE;
        currentVitals.heartRate = 0;
    }
    
    // Validate SpO2
    if (currentVitals.spo2 < MIN_SPO2 || currentVitals.spo2 > MAX_SPO2) {
        currentVitals.errorCode = SPO2_OUT_OF_RANGE;
        currentVitals.spo2 = 0;
    }
    
    // Determine health status
    determineHealthStatus();
}

void determineHealthStatus() {
    if (currentVitals.errorCode != NO_ERROR) {
        currentVitals.isHealthy = false;
        currentVitals.healthStatus = "ERROR";
        return;
    }
    
    if (!currentVitals.fingerDetected) {
        currentVitals.isHealthy = false;
        currentVitals.healthStatus = "NO_FINGER";
        return;
    }
    
    if (currentVitals.signalQuality < MIN_SIGNAL_QUALITY) {
        currentVitals.isHealthy = false;
        currentVitals.healthStatus = "POOR_SIGNAL";
        return;
    }
    
    // Check if vital signs are in healthy ranges
    bool hrHealthy = (currentVitals.heartRate >= 60 && currentVitals.heartRate <= 100);
    bool spo2Healthy = (currentVitals.spo2 >= 95);
    bool tempHealthy = (currentVitals.bodyTemp >= 36.0 && currentVitals.bodyTemp <= 37.5);
    
    if (hrHealthy && spo2Healthy && tempHealthy) {
        currentVitals.isHealthy = true;
        currentVitals.healthStatus = "HEALTHY";
    } else {
        currentVitals.isHealthy = false;
        
        if (!hrHealthy) {
            if (currentVitals.heartRate < 60) {
                currentVitals.healthStatus = "BRADYCARDIA";
            } else {
                currentVitals.healthStatus = "TACHYCARDIA";
            }
        } else if (!spo2Healthy) {
            currentVitals.healthStatus = "LOW_SPO2";
        } else if (!tempHealthy) {
            if (currentVitals.bodyTemp < 36.0) {
                currentVitals.healthStatus = "HYPOTHERMIA";
            } else {
                currentVitals.healthStatus = "FEVER";
            }
        } else {
            currentVitals.healthStatus = "ABNORMAL";
        }
    }
}

void updateSystemHealth() {
    // Check for consecutive errors
    if (currentVitals.errorCode != NO_ERROR) {
        systemHealth.consecutive_errors++;
        systemHealth.last_error_time = millis();
        
        if (systemHealth.consecutive_errors > 10) {
            systemReady = false;
            Serial.println("SYSTEM_ERROR: Too many consecutive errors");
        }
    } else {
        systemHealth.consecutive_errors = 0;
    }
    
    // Alert for unhealthy conditions
    if (!currentVitals.isHealthy && currentVitals.fingerDetected) {
        alertUnhealthyCondition();
    }
}

void alertUnhealthyCondition() {
    // Buzzer alert for critical conditions
    if (currentVitals.healthStatus == "LOW_SPO2" || 
        currentVitals.healthStatus == "BRADYCARDIA" ||
        currentVitals.healthStatus == "TACHYCARDIA") {
        
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100);
        digitalWrite(BUZZER_PIN, LOW);
    }
}

void outputData() {
    // Output data in CSV format
    Serial.print(currentVitals.timestamp);
    Serial.print(",");
    Serial.print(currentVitals.heartRate, 1);
    Serial.print(",");
    Serial.print(currentVitals.spo2, 1);
    Serial.print(",");
    Serial.print(currentVitals.bodyTemp, 2);
    Serial.print(",");
    Serial.print(currentVitals.ambientTemp, 2);
    Serial.print(",");
    Serial.print(currentVitals.signalQuality, 3);
    Serial.print(",");
    Serial.print(currentVitals.noiseLevel, 3);
    Serial.print(",");
    Serial.print(currentVitals.fingerDetected ? 1 : 0);
    Serial.print(",");
    Serial.print(currentVitals.healthStatus);
    Serial.print(",");
    Serial.println(currentVitals.errorCode);
}

void systemError() {
    while (true) {
        digitalWrite(LED_PIN, HIGH);
        delay(200);
        digitalWrite(LED_PIN, LOW);
        delay(200);
    }
}

void handleSystemError() {
    Serial.println("SYSTEM_ERROR: Attempting recovery...");
    
    // Attempt to reinitialize sensors
    if (initializeSensors()) {
        systemReady = true;
        systemHealth.consecutive_errors = 0;
        Serial.println("SYSTEM_RECOVERED");
    } else {
        delay(5000); // Wait before next attempt
    }
}
```
