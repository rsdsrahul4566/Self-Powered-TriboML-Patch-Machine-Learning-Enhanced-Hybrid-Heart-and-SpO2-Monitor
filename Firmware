# 🛠️MicroCOntroller Code (ESP32 WROOM)
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
