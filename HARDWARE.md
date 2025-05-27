# Hardware Components for MAX30102 Heartbeat Sensor Project

## 1. MAX30102 Sensor 
- **Overview:** Measures heart rate and oxygen saturation using infrared and red LEDs. Communicates via I2C and works with 1.8V–3.3V voltage.
- **Description:** This sensor works by being in contact with skin, preferably a thin part of the body, such as a fingertip or an earlobe. Let's see how it works in more detail. The amount of Hemoglobing, an oxygen carrier protein, in our blood varies between heart beats. This protein also absorbs infrared light.  Hence, by sending infrared light into the skin and measuring the amount that is reflected, using a photoreceptor, the sensor can measure the changes in Hemoglobin levels and see a pattern that matches our heart beat. This method of heart-rate measurement is called Photoplethysmography (PPG). This sensor can also measure oxygen saturation by pulse oximetry, a method that used two lights: infrared (880nm) and red light (660nm). This is possible because oxygenated hemoglobin absorbs more infrared light while deoxygenated hemoglobin absorbs more red light. Similar to the heart rate measurement, the sensor sends both wavelengths of light into the skin and compares the amount of red and infrared light absorbed. This allows it to calculate the percentage of hemoglogin that carries oxygen. 
## 2. Bidirectional Logic Level Converter
- **Description:** Safely converts logic signals (electrical signals) between 5V (Arduino) and 3.3V (MAX30102) to prevent damage. *See the last section for more detail*

## 3. Microcontroller (Arduino Uno)
- **Description:** Runs the code that controls the sensor, reads data via I2C, and processes or sends it to the computer.

## 6. Breadboard and Jumper Wires
- **Description:** Used for building the circuit.

## 7. Computer
- **Description:** Used to write, upload, and monitor code via the Arduino IDE. 

## Additional components for data storage
### 9. MicroSD Card Module
- **Description:** Enables local storage of data to a microSD card.

### 10. microSD Card 
- **Description:** Removable memory card where the heart rate data is logged.

## Explanations for the logic level converter
<p align="justify">
When connecting the MAX30102 sensor to the Arduino Uno, a logic level converter is needed because they don't operate with the same voltage. The Arduino Uno uses 5V logic, meaning that its pins send signals at 5V. In contrast, the MAX30102 operates on a 3.3V logic, which cannot tolerate 5V signals. While the Arduino does have a 3.3V power output, this only provides power, it does not change the voltage leve of the data signals, which remain at 5V. This is an issue, because we are working with an $I^2C$ communication protocol, which is how the Arduino and the sensor exchange data. $I^2C$ uses two lines: SDA (data) and SCL(clock). If the Arduino's 5V signals are being sent directly to the sensor's 3.3V SDA and SCL pins, it can damage the sensor. A bidirectional logic level converter solves this by acting as an intermediary, translating the 5V signals to 3.3V and vice-versa. It is bidirectional because the $I^2C$ protocal works both ways, both the Arduino and the sensor must be able to send and receive data on the same lines. 
</p>

The following image shows a logic level converter like the one used in this project. One side of the converter is for high-voltage, this is the side directy connected to the Arduino. The other side, the low-voltage, is connected to the sensor.
![image](https://github.com/user-attachments/assets/4d74c182-f520-446f-8f4f-3591454fbcdd)
