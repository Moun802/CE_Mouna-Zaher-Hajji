**Use the following code in Arduino**
```arduino
#include <Wire.h>
#include "MAX30105.h"

MAX30105 sensor;

void setup() {
  Serial.begin(115200);
  while (!Serial); 

  if (!sensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 not found. Check wiring.");
    while (1);
  }

  sensor.setup(50, 4, 2, 100, 411, 4096); 
  Serial.println("Place your finger on the sensor...");
}

void loop() {
  
  uint32_t irValue = sensor.getIR(); 
  uint32_t redValue = sensor.getRed(); 
  Serial.print(irValue);
  Serial.print(",");
  Serial.println(redValue);     // Send to Serial
  delay(50);                          // Sample every 50ms 
}
```

**Use the following code in python**

```python
import serial
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation
import time

# === CONFIGURATION ===
PORT = 'COM6'  
BAUD_RATE = 115200
BUFFER_SIZE = 60
FINGER_THRESHOLD = 15000  

# === SERIAL SETUP ===
try:
    ser = serial.Serial(PORT, BAUD_RATE, timeout=1)
    time.sleep(2)  # Give Arduino time to reset
except Exception as e:
    print(f"Could not open port {PORT}: {e}")
    exit()

ir_buffer = []
red_buffer = []

# === SPO2 CALCULATION ===
def calculate_spo2(ir_vals, red_vals):
    ir = np.array(ir_vals)
    red = np.array(red_vals)
    mean_ir = np.mean(ir)
    mean_red = np.mean(red)

    # FINGER DETECTION
    if mean_ir < FINGER_THRESHOLD:
        return None

    ac_ir = np.sqrt(np.mean((ir - mean_ir) ** 2))
    ac_red = np.sqrt(np.mean((red - mean_red) ** 2))

    if mean_ir == 0 or mean_red == 0:
        return None

    R = (ac_red / mean_red) / (ac_ir / mean_ir)
    spo2 = 110 - 25 * R
    return max(min(spo2, 100), 0)

# === VISUAL SETUP: Horizontal Band ===
fig, ax = plt.subplots(figsize=(8, 4))
ax.set_xlim(0, 100)
ax.set_ylim(0, 1.5)
ax.axis('off')

# Initial bar and fixed label above it
bar = ax.barh(0.5, 0, height=0.6, color='lightgray')[0]
label = ax.text(50, 1.1, "SpO₂: --%", ha='center', va='center',
                fontsize=20, weight='bold', color='black')

# === ANIMATION FUNCTION ===
def update(frame):
    global ir_buffer, red_buffer

    # Read serial 
    while ser.in_waiting:
        try:
            raw = ser.readline().decode('utf-8').strip()
            parts = raw.split(",")
            if len(parts) != 2:
                continue
            ir = int(parts[0])
            red = int(parts[1])

            ir_buffer.append(ir)
            red_buffer.append(red)

            if len(ir_buffer) > BUFFER_SIZE:
                ir_buffer.pop(0)
                red_buffer.pop(0)
        except:
            continue

    if len(ir_buffer) < BUFFER_SIZE:
        return bar, label

    spo2 = calculate_spo2(ir_buffer, red_buffer)

    if spo2 is None:
        bar.set_width(0)
        bar.set_color("lightgray")
        label.set_text("Place finger on sensor")
        return bar, label

    # Update the band width
    bar.set_width(spo2)

    # Color based on value
    if spo2 > 95:
        bar.set_color("mediumseagreen")
    elif spo2 > 90:
        bar.set_color("gold")
    else:
        bar.set_color("red")

    # Update label
    label.set_text(f"SpO₂: {spo2:.1f}%")
    return bar, label

# === START ANIMATION ===
ani = FuncAnimation(fig, update, interval=100)
plt.tight_layout()
plt.show()
```
