**Use the following code in arduino**
```arduino
#include <Wire.h>
#include "MAX30105.h"

MAX30105 sensor;

void setup() {
  Serial.begin(115200);
  while (!Serial);  // Waits for Serial Monitor to open

  if (!sensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 not found. Check wiring.");
    while (1);
  }

  sensor.setup(50, 4, 2, 100, 411, 4096); 
  Serial.println("Place your finger on the sensor...");
}

void loop() {
  
  uint32_t irValue = sensor.getIR();  // Read only IR
  Serial.println(irValue);     // Send to Serial
  delay(50);                          // Sample every 50ms 
}
```
**Use the following code in python**
```python
import serial
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from scipy.signal import find_peaks
import time

# === CONFIGURATION ===
PORT = 'COM6'           
BAUD_RATE = 115200
LIST_SIZE = 300
PEAK_DISTANCE = 10
PROMINENCE = 100
SAMPLE_INTERVAL = 0.05  # 50 ms

# === STATIC SETUP ===
try:
    ser = serial.Serial(PORT, BAUD_RATE, timeout=1)
    time.sleep(2)  # Give Arduino time to reset
except Exception as e:
    print(f"Error: Cannot open {PORT}. {e}")
    exit()

data = [0] * LIST_SIZE
timestamps = [time.time()] * LIST_SIZE

# === PLOT SETUP ===
plt.style.use("ggplot")
fig, ax = plt.subplots(figsize=(10, 5))
line, = ax.plot([], [], lw=2, color='deepskyblue', label='IR Signal')
peaks_line, = ax.plot([], [], 'ro', label='Detected Peaks')

ax.set_title("Real-Time IR Signal (Heartbeat)", fontsize=14, weight='bold')
ax.set_xlabel("Sample Index")
ax.set_ylabel("IR Value")
ax.set_xlim(0, LIST_SIZE)
ax.legend(loc="upper right")
ax.grid(True, linestyle='--')

# === ANIMATION FUNCTION ===
def animate(frame, data, timestamps, ser):
    while ser.in_waiting:
        try:
            raw = ser.readline().decode('utf-8').strip()
            ir = int(raw)
            now = time.time()
            data.append(ir)
            timestamps.append(now)
            if len(data) > LIST_SIZE:
                data.pop(0)
                timestamps.pop(0)
        except:
            continue

    y = np.array(data)
    x = np.arange(LIST_SIZE)  # Fixed x-axis length
    line.set_data(x, y)

    # Peak detection
    peaks, _ = find_peaks(y, prominence=PROMINENCE, distance=PEAK_DISTANCE)
    peaks_line.set_data(peaks, y[peaks])

    # BPM calculation
    if len(peaks) >= 5 and np.ptp(timestamps) > 2:
        peak_times = [timestamps[p] for p in peaks]
        intervals = np.diff(peak_times)
        valid_intervals = intervals[intervals > 0]
        
        if len(valid_intervals) > 0:
            mean_interval = np.mean(valid_intervals)
            avg_bpm = 60.0 / mean_interval
            if 30 < avg_bpm < 200:  
                ax.set_title(f"Real-Time IR Signal (BPM: {avg_bpm:.1f})", fontsize=14, weight='bold')
            else:
                ax.set_title("Real-Time IR Signal (BPM: --)", fontsize=14, weight='bold')
        else:
            ax.set_title("Real-Time IR Signal (BPM: --)", fontsize=14, weight='bold')
    else:
        ax.set_title("Real-Time IR Signal (BPM: --)", fontsize=14, weight='bold')

    # Auto-center Y-axis
    y_center = np.mean(y)
    y_range = (np.max(y) - np.min(y)) / 2 + 500
    ax.set_ylim(y_center - y_range, y_center + y_range)

    return line, peaks_line

# === LAUNCH ANIMATION ===
ani = animation.FuncAnimation(fig, animate, fargs=(data, timestamps, ser), interval=50)
plt.tight_layout()
plt.show()
```

