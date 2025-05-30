# Arduino Project: Heart-Rate Monitor

## Project Overview

The goal of this project is to build an object capable of measuring the heart rate by simple contact with a sensor, in order to give a health profile to the user. Beyond this interactive facet of the project, it also aims to collect heart rate data and analyse it in a similar way to current pre-existing medical tools such as the EKG machine. This will be done with the Max30102 sensor, whose description can be found in the HARDWARE.md file.

## Hardware Components

- Arduino Uno/Nano
- MAX30102 sensor
- Bidirectional Logic Level Converter
- Breadboard and Jumper wires
- Computer
- MicroSD Card Module
- MicroSD Card
- **More information in HARDWARE.md file**

## Circuit Diagram

[Insert circuit diagram image or description]

## Wiring Instructions

| Arduino Pin | Logic Converter | MAX30102    |Description|
|-------------|--------------   |-------------|-----------|
| 3.3V        | LV              | VIN         |Powers MAX30102 via the low-voltage side (3.3V)|
| 5V          | HV              | ---         |Supplies 5V to high-voltage side of converter. Allows converter to accept the 5V from Arduino|
| GND         | GND(both sides) | GND         |Common ground connection. Reference point, all the devices must share ground|
| A4          | HV1             | ---         |Links Arduino's $I^2C$ data line to converter. Transmits data from Arduino to sensor |
| A5          | HV2             | ---         |Links Arduino's $I^2C$ clock line to converter. Synchronizes the timing of data transfer between Arduino and sensor|
| 2           | ---             | INT         |Optional interrup signal. Alerts Arduino when a new sample of data is ready|
| ---         | LV2             | SCL         |Links converter's $I^2C$ data line to sensor. MAX30102 receives data requests via this line |
| ---         | LV1             | SDA         |Links converter's $I^2C$ clock line to sensor. Guides the sensor in the timing of data transfer|

## Software Requirements

- Arduino IDE [version]
- Libraries:
  - Wire
  - MAX30105
  - math

## Code Explanation (Arduino Version)

### Main logic 
- **Heart rate logic:** Detects peaks in the IR (infrared) signal and calculates the time between peaks to estimate BPM.
- #### SpO_2 calculation:
   The amount of absorbed light measured by the sensor fluctuates over time. This is how we can measure the peaks for the heart beat. However, there is a set value of light absorbed, that is not due to the pulse but to components that do not change over time, like tissue, or venous blood. This baseline is called the DC signal, while the pulsing part is called the AC signal and is caused by the hearbeat. 
  ![image](https://github.com/user-attachments/assets/c123775e-8928-4f3a-9a2a-08b659c497cc)
  There exists a ratio of ratios between AC signal and DC signal which was shown to directly relate to the amount of blood oxygen. Hence, the code is based on this mathematical relationship:
  ![Ratio Equation](https://latex.codecogs.com/svg.image?R%20=%20%5Cfrac%7BAC_%7Bred%7D/DC_%7Bred%7D%7D%7BAC_%7BIR%7D/DC_%7BIR%7D%7D)


   ![SpO2 Equation](https://latex.codecogs.com/svg.image?SpO_2%20=%20104%20-%2017%20%5Ctimes%20R)

  Where red refers to red light and IR refers to infrared light.

  As for the DC and AC signal calculations, they are as follows:

  The DC signal is the average signal over time. To isolate the AC component of the signal, we subtract the DC baseline (average) from each individual sample:

 ```math
 \text{AC}_{\text{IR}} = \text{IR signal} - \text{average signal}
 ```
  However, the AC signal fluctuates. We wish to summarize how "big" or strong it is on average, even though it moves both above and below the baseline. To do this, we calculate its **Root Mean Square (RMS)**, given by the following equation:
  
```math
 \text{RMS}_{\text{AC}} = \sqrt{ \frac{1}{N} \sum_{i=1}^{N} (v_i - V_{\text{DC}})^2 }
```
  where
  - N is the number of samples
  - $V_DC$ is the DC value of the signal
  - $V_i$ is the value of the signal for a specific sample
    


-**Fitness bracket:** Uses the average BPM and the user age and sex to classify cardiovascular fitness

### Code
Before the setup. This is to set our constants and important values
```arduino
#include <Wire.h>
#include "MAX30105.h"
#include <math.h>
//Libraries
MAX30105 particleSensor; //Creates object that lets us talk to the sensor

#define SAMPLE_INTERVAL 250         //We collect data every 250ms=4 times per second
#define IR_THRESHOLD 20000 //Below this, we assume no finger is present
#define PEAK_THRESHOLD 100 //How high the peak hs to be to count as a heartbeat
#define MIN_PEAK_INTERVAL 400 //Time between peaks must be at least 400ms
#define BUFFER_SIZE 100 //How many sample we use for calculations

uint32_t irBuffer[BUFFER_SIZE]; //Stores IR values
uint32_t redBuffer[BUFFER_SIZE]; //Stores red light values
int irIndex = 0; //Tracks index in the buffer (like i=0 in python)

unsigned long lastSampleTime = 0; //When we last took a reading
unsigned long lastPeakTime = 0; //Last time we saw a peak (heartbeat)
int bpm = 0; //current bpm

int age = 0;
char sex = 'M';

int totalBPM = 0; //Sum of all valide bpms (to compute average later)
int validBPMCount = 0;
float totalSpO2 = 0; //sum of all valid oxygen readings (to compute average later)
int validSpO2Count = 0;
//Setup runs once at the beginning
```
The setup:
This part runs for 30 seconds, computing the BPM and the oxygen saturation 4 times per second
```arduino
//Setup runs once at the beginning
void setup() { 
  Serial.begin(115200); //Starts serial monitor
  while (!Serial); //Wait until serial monitor opens

  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 not found. Check wiring.");
    while (1); //Creates infinite loop - freezes program if sensor is not detected
  }

  particleSensor.setup(30, 4, 2, 100, 411, 4096);
 //brightness, average, mode, sample rate, pusle width, ADC range
  UserInfo(); //function for user input
  Serial.println("\nPlace your finger on the sensor. Measuring for 30 seconds...");
  delay(3000); //gives time to place finger

  unsigned long start = millis(); //current time in milliseconds
  while (millis() - start < 30000) { //makes the startup run for 30 seconds
    if (millis() - lastSampleTime >= SAMPLE_INTERVAL) {
      lastSampleTime = millis(); //takes sample every 250ms

      uint32_t ir = particleSensor.getIR(); //infrared light reading
      uint32_t red = particleSensor.getRed(); //red light reading

      irBuffer[irIndex] = ir;
      redBuffer[irIndex] = red;
      irIndex = (irIndex + 1) % BUFFER_SIZE; //circular buffer, always keeps 100 values

      if (ir > IR_THRESHOLD) { //checks if finger is present
        // Heart Rate
        //checks if IR signal increased then drops (peak)
        int prev = (irIndex - 2 + BUFFER_SIZE) % BUFFER_SIZE; //before current value
        int curr = (irIndex - 1 + BUFFER_SIZE) % BUFFER_SIZE; //current value
        int next = irIndex % BUFFER_SIZE; //next value

        if (irBuffer[prev] < irBuffer[curr] &&
            irBuffer[next] < irBuffer[curr] &&
            (irBuffer[curr] - irBuffer[prev] > PEAK_THRESHOLD) &&
            (irBuffer[curr] - irBuffer[next] > PEAK_THRESHOLD)) {
           //if there is a peak, update bmp 
          unsigned long now = millis(); //current time
          unsigned long interval = now - lastPeakTime; //time since last heartbeat was detected
          if (interval > MIN_PEAK_INTERVAL) {
            bpm = 60000 / interval; //60 000 ms is a minute. So we divide a minute by our interval, which gives us the bpm
            lastPeakTime = now;

            if (bpm > 40 && bpm < 180) {
              totalBPM += bpm;
              validBPMCount+=1; //to flush out unrealistic values
            }
          }
        }

        // SpO₂
        float meanIR = 0, meanRed = 0, acIR = 0, acRed = 0; //mean is DC, ac is AC
        for (int i = 0; i < BUFFER_SIZE; i+=1) { //starts at i=0, keeps looping as long as i<100, after each loop increase i by one)
          meanIR += irBuffer[i];
          meanRed += redBuffer[i]; 
        }
        meanIR /= BUFFER_SIZE;
        meanRed /= BUFFER_SIZE; //calculates average of IR and red signals (DC component)

        for (int i = 0; i < BUFFER_SIZE; i+=1) {
          acIR += pow(irBuffer[i] - meanIR, 2);
          acRed += pow(redBuffer[i] - meanRed, 2);
        }
        acIR = sqrt(acIR / BUFFER_SIZE);
        acRed = sqrt(acRed / BUFFER_SIZE);
        // calculate root mean square of each signal (AC component)

        float R = (acRed / meanRed) / (acIR / meanIR); //Ratio of ratios
        float spo2 = 104.0 - 17.0 * R; //formula for SpO2%

        if (spo2 >= 85.0 && spo2 <= 100.0) {
          totalSpO2 += spo2;
          validSpO2Count+=1; //flushes out unrealistic values
        }

        Serial.print("IR: "); Serial.print(ir);
        Serial.print(" | BPM: "); Serial.print(bpm);
        Serial.print(" | SpO2: "); Serial.println(spo2);
      } else {
        Serial.println("No finger detected.");
      }
    }
  }

  // === Results ===
  Serial.println("\n==== 30s Profile Summary ====");

  int avgBPM;
  if (validBPMCount > 0) {
    avgBPM = totalBPM / validBPMCount;
  } else {
    avgBPM = 0;
  } //If we have no valid bpms, then we avoid dividing by 0 to find the average

  float avgSpO2;

  if (validSpO2Count > 0) {
    avgSpO2 = totalSpO2 / validSpO2Count;
  } else {
    avgSpO2 = 0;
  }


  Serial.print("Average HR: "); Serial.print(avgBPM); Serial.println(" BPM");
  Serial.print("Average SpO₂: "); Serial.print(avgSpO2, 1); Serial.println(" %");

  Serial.print("Fitness Profile: ");
  Serial.println(determineFitnessBracket(avgBPM, age, sex));
}
```
The next part is the user info function, called during the setup
```arduino
void UserInfo() { //asks user to enter their information, is called in the setup 
  Serial.println("Enter your age:");
  while (Serial.available() == 0);
  age = Serial.parseInt();
  Serial.read();

  Serial.println("Enter your sex (M/F):");
  while (Serial.available() == 0);
  char input = Serial.read();
  sex = (input == 'f' || input == 'F') ? 'F' : 'M';

  Serial.print("Age: "); Serial.print(age);
  Serial.print(" | Sex: "); Serial.println(sex);
}
```
The last section of the code is used to determine in which fitness bracket the user is. The following code presents only the first category.
```arduino
char* determineFitnessBracket(int avgHR, int age, char sex) { //provides the fitness categories
  if (age >= 18 && age <= 25) {
    if (sex == 'M') {
      if (avgHR <= 61) return "Excellent";
      else if (avgHR <= 65) return "Good";
      else if (avgHR <= 69) return "Above Average";
      else if (avgHR <= 73) return "Average";
      else if (avgHR <= 81) return "Below Average";
      else return "Poor";
    } else {
      if (avgHR <= 65) return "Excellent";
      else if (avgHR <= 69) return "Good";
      else if (avgHR <= 73) return "Above Average";
      else if (avgHR <= 78) return "Average";
      else if (avgHR <= 84) return "Below Average";
      else return "Poor";
    }
  }
```

It is important to note that the void loop is empty because the code only runs for 30 seconds. Hence, there is no need for continuous data collection.

## Code explanation (Python version)

The drawback with using Arduino directly is that the funtion used to calculate the BPM, which relies on finding peaks, is not very accurate. Python has a library, called **spicy.signal** that contains a built-in function capable of finding peaks in a set of data. However, in order to get a live reading of the IR signals, python needs to be synchronized with Arduino. Here is the code for the synchronization. 

## Usage Instructions

1. [Step 1]
2. [Step 2]
3. [Step 3]
4. [Additional steps...]

## Troubleshooting

- **Issue 1**: [Solution]
- **Issue 2**: [Solution]
- **Issue 3**: [Solution]

## Future Improvements

- [Potential improvement 1]
- [Potential improvement 2]
- [Potential improvement 3]

## References

- [Reference 1]
- [Reference 2]
- [Additional references...] 
