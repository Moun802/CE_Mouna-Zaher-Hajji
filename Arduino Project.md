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
| A4          | HV1             | ---         |Links Arduino's $I^2C$ data line to converter. Transmits data from sensor Arduino to sensor |
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

## Code Explanation (Interactive part)

### Main logic 
- **Heart rate logic:** Detects peaks in the IR (infrared) signal and calculates the time between peaks to estimate BPM.
- #### SpO_2 calculation:
   The amount of absorbed light measured by the sensor fluctuates over time. This is how we can measure the peaks for the heart beat. However, there is a set value of light absorbed, that is not due to the pulse but to components that do not change over time, like tissue, or venous blood. This baseline is called the DC signal, while the pulsig part is called the AC signal and is caused by the hearbeat. 
  ![image](https://github.com/user-attachments/assets/c123775e-8928-4f3a-9a2a-08b659c497cc)
  There exists a ratio of ratios between AC signal and DC signal which was shown to directly relate to the amount of blood oxygen. Hence, the code is based on this mathematical relationship:
  ![Ratio Equation](https://latex.codecogs.com/svg.image?R%20=%20%5Cfrac%7BAC_%7Bred%7D/DC_%7Bred%7D%7D%7BAC_%7BIR%7D/DC_%7BIR%7D%7D)


   ![SpO2 Equation](https://latex.codecogs.com/svg.image?SpO_2%20=%20104%20-%2017%20%5Ctimes%20R)

  Where red refers to red light and IR refers to infrared light.




- **Fitness bracket:** Uses the average BPM and the user age and sex to classify cardiovascular fitness

```arduino
// Key parts of your code with explanations
void setup() {
  // Explain what's happening in setup
}

void loop() {
  // Explain what's happening in the main loop
}
```

It is important to note that the void loop is empty because the code only runs for 30 seconds. Hence, there is no need for continuous data collection.

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
