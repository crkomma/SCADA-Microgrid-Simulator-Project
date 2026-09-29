# Microgrid Hardware-in-the-Loop Testbed
**Phase 1: Solar Dial Telemetry & Local LED Control Room**

Hey! Welcome to my project repository. I'm building a physical microgrid simulation on my desk using an Elegoo Uno (Arduino clone) to mimic a solar power monitoring system. 

My goal here is to use a potentiometer as a "solar input dial" and have the microcontroller read the incoming voltage and trigger different LED alert states depending on how much power is coming in.

---

## Setup: Hardware

Here’s everything I used on my breadboard for this first phase:

* **1x Elegoo Uno** (ATmega328P microcontroller board)
* **1x 10kΩ Potentiometer** (My simulated "Solar Input" knob)
* **4x LEDs** (Green = Normal, Yellow = High Load Warning, Red = Over Voltage, White = Relay Trip)
* **4x 330Ω Resistors** (To prevent burning out the LEDs!)
* Solderless breadboard & some jumper wires

### How I Wired It (Pin Map)

| Component | Connected To: | What It Does |
| :--- | :--- | :--- |
| **Potentiometer Signal Pin** | Analog Pin `A0` | Sends variable analog voltage ($0\text{V} - 5\text{V}$) |
| **Green LED (Long leg)** | Digital Pin `8` | Lights up when solar input is low/stable |
| **Yellow LED (Long leg)** | Digital Pin `9` | Lights up when input hits medium warning levels |
| **Red LED (Long leg)** | Digital Pin `10` | Lights up on high input/trip risk |
| **Short Legs of LEDs** | Breadboard (-) Rail | Bleeds off to GND via $330\,\Omega$ resistors |

---

## 💻 The Code (Arduino C/C++)

The microcontroller uses a 10-bit Analog-to-Digital Converter (ADC). That means it converts the $0\text{V} - 5\text{V}$ coming off the potentiometer into a raw integer between `0` and `1023`. 

I wrote a quick C program to convert that raw number back into an estimated voltage, stream it live to the computer screen via Serial Monitor, and switch the LEDs:

```cpp
/*
  Microgrid Control Room - Hardware Test
  Reads Analog Pin A0 (Solar Dial) and triggers status LEDs based on input level.
*/

const int SOLAR_DIAL_PIN = A0;
const int GREEN_LED      = 8;  // Normal
const int YELLOW_LED     = 9;  // Warning
const int RED_LED        = 10; // Fault / Trip

void setup() {
  // Start serial communication at 9600 baud rate
  Serial.begin(9600);
  
  // Set LED pins as outputs
  pinMode(GREEN_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  Serial.println("--- MICROGRID TELEMETRY INITIALIZED ---");
}

void loop() {
  // Read raw 10-bit value (0 - 1023)
  int rawValue = analogRead(SOLAR_DIAL_PIN);
  
  // Math: Convert raw reading to actual voltage
  float voltage = rawValue * (5.0 / 1023.0);
  
  // Print telemetry data to screen
  Serial.print("Solar Input Raw: ");
  Serial.print(rawValue);
  Serial.print(" | Voltage: ");
  Serial.print(voltage, 2);
  Serial.println(" V");

  // Simple state logic for alert LEDs
  if (rawValue < 350) {
    digitalWrite(GREEN_LED, HIGH);
    digitalWrite(YELLOW_LED, LOW);
    digitalWrite(RED_LED, LOW);
  } 
  else if (rawValue >= 350 && rawValue < 750) {
    digitalWrite(GREEN_LED, LOW);
    digitalWrite(YELLOW_LED, HIGH);
    digitalWrite(RED_LED, LOW);
  } 
  else {
    digitalWrite(GREEN_LED, LOW);
    digitalWrite(YELLOW_LED, LOW);
    digitalWrite(RED_LED, HIGH);
  }
  
  delay(150); // Short delay to keep the serial stream readable
}
```
**Challenges, Debugging, and What I Learned**

Building this wasn't super easy. It was definitely a learning process with a few "oh!" and "a-ha!" moments...
* **Short Circuit:** When I first plugged in my Ground jumper wire, the power light on the microcontroller turned off completely. After some research and pin repositioning, turns out the potentiometer pins were jammed into the same row on the breadboard, which bridged the 5V power straight into Ground, therefore tripping the board's safety fuse.
* **Analog vs. Digital Headers:** I accidentally plugged in the LED control wires into the Analog header side of the microcontroller instead of Digital Pins 8, 9, and 10. I moved them over and fixed the short circuit (again) which got the LEDs responding instantly.
* **Potentiometer Fit:** Since I had a trimpot potentiometer, the legs on it were a little thick; When I plugged a wire in the exact hole next to them, it popped the component right out. To fix this, I spaced the wire out a couple columns over in the same row.

**Results: Testing**

Once all the wiring and positioning issues were fixed, opening the Serial Monitor at 9600 baud gave me smooth, live telemetry readings as I turned the knob on the potentiometer:

```text
--- MICROGRID TELEMETRY INITIALIZED ---
Solar Input Raw: 110 | Voltage: 0.54 V --> [Green LED ON]
Solar Input Raw: 520 | Voltage: 2.54 V --> [Yellow LED ON]
Solar Input Raw: 890 | Voltage: 4.35 V --> [Red LED ON]
```

# Phase 2: Host Python SCADA Pipeline & Real-Time Visualization

Now that the local microcontroller handles real-time edge execution (reading ADC inputs and driving status LEDs), Phase 2 expands the testbed into a true Supervisory Control and Data Acquisition (SCADA) system. 

Using Python on a host PC, the system reads ASCII data streams over the USB serial interface (`COM3` @ 9600 baud), parses raw telemetry into structured numbers, logs timestamped data to disk, and visualizes system state in a real-time dashboard.

## 🐍 Software & Architecture Breakdown

### 1. Telemetry Ingestion & Persistent CSV Logging (`scada_logger.py`)

This script establishes a serial connection using `pySerial`, captures raw lines from the UART buffer, applies Regular Expressions (`re`) to extract numbers from the string, tags the grid operational state, and appends timestamped records directly to `microgrid_telemetry.csv`.

```python
'''
Data Ingestion & Disk Logging:
    Reads text sent over the USB cable from the Elegoo,
    extracts the specific numbers, categorizes the state
    of the solar grid, prints a clean table in the 
    terminal, and saves every data point to a CSV file.
'''    

import serial # lets Python communicate with hardware devices over a serial/USB port
import time # used to pause code execution when initializing hardware connections
import csv # formats data into comma-separated rows so programs like Excel can open them cleanly
import re # "regular expressions" (regex module): acts as a "pattern search" tool which extracts exact numbers out of text strings
from datetime import datetime # imports clock to generate real timestamps for every log entry

# --- CONFIGURATION ---
SERIAL_PORT = 'COM3' # sets USB hardware address where Elegoo is plugged in
BAUD_RATE   = 9600 # sets data transfer speed (9600 bits/second). Python and the Elegoo must match the speed or the letters become corrupted
CSV_FILE    = 'microgrid_telemetry.csv' # names output file where telemetry will be saved on my computer

def run_scada_logger():
    print("=" * 60) # just a banner for decoration and to separate headers
    print("       SCADA TELEMETRY LOGGER & DATA INGESTION ENGINE       ")
    print("=" * 60)
    print(f"Connecting to Microgrid Hardware on {SERIAL_PORT} @ {BAUD_RATE} baud...")
    
    try:
        # Establish serial connection
        ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=2) # opens hardware port COM3 at 9600 baud. 
        # if Python asks for data and heard nothing for 2 seconds, it stops waiting and moves on
        time.sleep(2)  # pauses python for 2 seconds
        
        '''
        when the computer opens a serial connection to the Elegoo, the board auto-resets.
        this 'sleep' allows the microcontroller bootloaded to finish starting up.
        '''
        
        print(f"Connection active! Logging data to '{CSV_FILE}'...")
        print("Press Ctrl+C or click the red Stop button in Spyder to stop.\n")
        
        # Initialize CSV file with structural headers
        with open(CSV_FILE, mode='w', newline='') as file: # safely opens the CSV file for write mode while preventing the addition of extra blank lines between rows.
        # using a 'with' block makes sure Python automatically closes the file if the script ends or crashes
            writer = csv.writer(file) # creates a CSV writer object which handles formatting text into CSV rows
            writer.writerow(['Timestamp', 'Raw_ADC', 'Voltage_V', 'Grid_Status']) # writes the first header line as column names into the CSV file
            
            # Print console header table
            print(f"{'TIMESTAMP':<20} | {'RAW ADC':<8} | {'VOLTAGE':<8} | {'STATUS':<12}") # formatted header row to the terminal. left aligning all the text across different widths to create neat alignment
            print("-" * 58)
            
            while True: # keeps listening for sensor data until stopped manually
                if ser.in_waiting > 0: # asks the serial buffer if there are unread bytes waiting in the hardware queue. there is data to process if greater than zero
                    raw_line = ser.readline().decode('utf-8', errors='replace').strip()
                    
                    '''
                    1. ser.readline(): reads incoming raw bytes until it hits a newline character
                    2. .decode('utf-8', errors='replace'): converts raw hardware bytes into a human-readable text string
                    3. .strip(): cleans up unwanted trailing characters like spaces, carriage returns (\r), or newlines (\n)
                    
                    '''
                    
                    # Regex parsing: extract raw integer and float voltage
                    # Matches lines like: "Solar Input Raw: 520 | Voltage: 2.54 V"
                    match = re.search(r"Solar Input Raw:\s*(\d+)\s*\|\s*Voltage:\s*([\d\.]+)", raw_line)
                    '''
                    \s* ignores any spaces
                    (\d+) Group 1: captures one of more digit numbers (the raw ADC value)
                    \|\s*Voltage:\s* searches for the pipe symbol | and 'Voltage:'
                    ([\d\.]+) Group 2: captures numbers including decimal points (the voltage float)
                    '''
                    if match: # proceeds only if regex successfully finds numbers in the string
                        raw_adc   = int(match.group(1)) # pulls Group 1 out of regex match and converts into integer (ex. 520)
                        voltage   = float(match.group(2)) # pulls Group 2 out and converts into floating-point decimal (ex. 2.54)
                        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S") # reads computer's clock and formats into a standard string YYYY-MM-DD HH-MM-SS
                        
                        '''
                        Apply Grid SCADA Threshold Logic
                        
                        if 10-bit ADC reading (0-1023) is:
                            below 350 (< 1.71 V)
                                tag as NORMAL
                            350-749 (1.71 V to 3.67 V):
                                tag as WARNING
                            above 750 (>= 3.67 V):
                                tag as FAULT/TRIP
                        '''
                        if raw_adc < 350:
                            status = "NORMAL"
                        elif raw_adc < 750:
                            status = "WARNING"
                        else:
                            status = "FAULT/TRIP"
                            
                        # Write record directly to disk
                        writer.writerow([timestamp, raw_adc, voltage, status]) # appends formatted data line to CSV file
                        file.flush()  
                        '''
                        IMPORTANT! forces python to immediately write data out of RAM to the physical hard drive, 
                        so no data is lost if the power drops or program is forcibly closed
                        '''
                        # Live terminal stream
                        print(f"{timestamp:<20} | {raw_adc:<8d} | {voltage:<7.2f}V | {status:<12}") # formats+prints processed data into terminal's live table view
                        # :8d formats integers across 8 spaces
                        # "7.2f formats floats across 7 spaces rounded to 2 decimal places
                        
    except serial.SerialException as e:
        print(f"\n[ERROR] Port failure on {SERIAL_PORT}. Ensure port isn't open elsewhere.")
        print(f"Details: {e}")
    # catches hardware failures like unplugged USB or port in use, printing helpful error instead of crashing
    except KeyboardInterrupt:
        print("\n" + "=" * 60)
        print("Logging paused by user. Telemetry file safely flushed and closed.")
        print("=" * 60)
        # catches when you press Ctrl+C or hit stop
        if 'ser' in locals() and ser.is_open: # checks if varibale 'ser' exists and if the port is open
            ser.close() # closes serial connection properly and releases COM3 back into the OS

if __name__ == "__main__":
    run_scada_logger()
# python entry point, runs program when executed directly
```

### 2. Real-Time Graphical SCADA Dashboard (`scada_dashboard.py`)

This script visualizes incoming voltage telemetry live using `matplotlib.animation.FuncAnimation`. It maintains a fixed-size memory buffer (`collections.deque(maxlen=50)`) to display a rolling 10-second window across dynamic voltage risk zones (Normal, Warning, Fault).

```python
import serial
import time
import re
import collections # specialized data structures module for memory buffers
import sys # system module used to shut down execution safely
import matplotlib.pyplot as plt # graphing library
import matplotlib.animation as animation # matplotlib submodule used for live updating animations

# --- CONFIGURATION ---
SERIAL_PORT = 'COM3'
BAUD_RATE   = 9600
MAX_POINTS  = 50     
# sets window length for plot, displays 50 most recent data points and scrolls continuously as new telemetry data arrives

# Data buffers
time_data    = collections.deque(maxlen=MAX_POINTS)
'''
creates a 'Double-Ended Queue (Dequeue)': when the 51st data point appears, the 1st
(oldest) data point is automatically dropped off the back. keeps RAM usage stable.
'''
voltage_data = collections.deque(maxlen=MAX_POINTS) # same thing, but for the voltage readings
start_time   = time.time() 
# records system time when the script begins and used to calculate relative seconds elapsed (t = 0, 1, 2, ...)

# Establish Serial Bridge
try:
    ser = serial.Serial(SERIAL_PORT, BAUD_RATE, timeout=1)
    time.sleep(2)
    print("Connected to Microgrid Hardware for Live Plotting!")
except Exception as e:
    print(f"Error opening serial port: {e}")
    sys.exit()
    
# try / except block attempts connection to COM3
# if locked or unavailable, prints diagnostic tips and then terminates execution cleanly

# Configure Matplotlib Figure & Axes
plt.style.use('seaborn-v0_8-darkgrid' if 'seaborn-v0_8-darkgrid' in plt.style.available else 'default') # grid aesthetic setup in plot window
fig, ax = plt.subplots(figsize=(10, 5)) # creates a matplotlib window canvas (fig) with one set of axes (ax) styled at 10 in x 5 in
line, = ax.plot([], [], lw=2.5, color='#007acc', label='Solar Voltage (V)') 
# initializes an empty 2D line plot object
# lw=2.5 : line thickness
# #007acc : color (SCADA blue)

# Set Graph Labels & Limits
ax.set_title("SCADA Real-Time Microgrid Solar Ingest", fontsize=14, fontweight='bold', pad=15)
ax.set_xlabel("Elapsed Time (seconds)", fontsize=11)
ax.set_ylabel("Measured Voltage (V)", fontsize=11)
ax.set_ylim(-0.2, 5.2) # locks y-axis from -0.2 V to 5.2 V, representing 0-5V operation range of the microcontroller ADC

# Draw SCADA Threshold Warning Bands
ax.axhline(y=1.71, color='orange', linestyle='--', linewidth=1.5, label='Warning Threshold (1.71V)')
# draws horizontal dashed threshold line across entire plot at 1.71V (orange warning limit)
ax.axhline(y=3.67, color='red', linestyle='--', linewidth=1.5, label='Trip Threshold (3.67V)')
# draws line at 3.67V (red trip limit)
ax.axhspan(0.0, 1.71, color='green', alpha=0.08, label='Normal Zone')
# draws translucent colored background rectangle from 0V to 1.71V: normal zone.
# alpha=0.08 sets transparency level
ax.axhspan(1.71, 3.67, color='orange', alpha=0.08, label='Warning Zone')
# shades warning zone (orange)
ax.axhspan(3.67, 5.00, color='red', alpha=0.08, label='Fault Zone')
# shades fault zone (red)

ax.legend(loc='upper left', frameon=True)
# generates color-coded legend in top left corner

def update_graph(frame): 
# animation engine repeatedly calls function on a timer
# frame argument keeps track of how many loop iterations have run
    while ser.in_waiting > 0: # processes all queued serial lines currently sittin in buffer
        raw_line = ser.readline().decode('utf-8', errors='replace').strip()
        match = re.search(r"Solar Input Raw:\s*(\d+)\s*\|\s*Voltage:\s*([\d\.]+)", raw_line)
        # reads, decodes, and parses incoming serial strings for numbers
        
        if match:
            voltage = float(match.group(2)) # pulls out parsed voltage decimal
            # print(f"[PLOT DEBUG] Plotting Voltage: {voltage}V")
            
            elapsed_time = time.time() - start_time
            # calculates how many seconds passed since script began
            
            time_data.append(elapsed_time)
            voltage_data.append(voltage)
            # pushes new readings into rolling dequeue buffers
            
            line.set_data(time_data, voltage_data)
            # updates plot line object with newly updated memory buffers
            
            # X-axis scrolling window
            if elapsed_time > 10:
                ax.set_xlim(elapsed_time - 10, elapsed_time + 1)
            else:
                ax.set_xlim(0, 10)
                
                '''
                for first 10 seconds, x-axis stays fixed from 0 to 10 seconds
                after 10 seconds,
                slides x-axis window forward in real time, creates rolling 10 second window
                '''
                
    fig.canvas.draw_idle() # repaints matplotlib pixel window canvas on screen 
    return line, # returns updated line tuple back to animation engine

# Launch Real-Time Animation (Updates every 100 ms)
ani = animation.FuncAnimation(
    fig, # canvas figure to draw on
    update_graph, # callback function to run repeatedly
    interval=100, # triggers update function every 100 ms (10 times per second)
    blit=False, # redraws whole plot frame, including scrolling axes
    cache_frame_data=False # disables frame caching to prevent RAM leaking during infinite live streaming
)

plt.tight_layout() # adjusts padding around figure to prevent label overlapping
plt.show() # opens interactive plot window on screen and keeps it active
```

## 🛠️ Debugging & Hardware-Software Challenges

Integrating hardware serial streams with host software introduced several system-level edge cases:
1. Environment & Package Management:
   * Issue: Running `pip install pyserial` in the general system command line installed packages into global Python, leaving Spyder's internal environment missing the module (`ModuleNotFoundError: No module named 'serial'`).
   * Solution: Identified Spyder's exact Python executable using `sys.executable` and installed target packages directly into Spyder's active environment via `path/to/python.exe" -m pip install pyserial`.
2. Serial Resource Contention (`PermissionError: Access is Denied`):
   * Issue: Attempting to run Python scripts while the Arduino IDE Serial Monitor was open threw port access conflicts because Windows locks COM ports to a single active handle.
   * Solution: Ensured the Arduino Serial Monitor was closed before running Python scripts, and implemented `ser.close()` inside clean exception handlers (`KeyboardInterrupt`) to release the serial resource upon exiting.
3. Matplotlib Rendering Backends & RAM Caching:
   * Issue: The dashboard script launched without drawing telemetry because Spyder defaulted to an Inline backend rather than an interactive pop-up window. Additionally, Matplotlib threw unbounded memory warnings (`cache_frame_data`).
   * Solution: Changed Spyder's graphics configuration to Automatic, called `fig.canvas.draw_idle()` on each update frame, and set `cache_frame_data=False` inside `FuncAnimation` to maintain constant RAM overhead during infinite streaming.
  
## 📊 Verification & Data Logging Results

Running `scada_logger.py` produced structured CSV records in `microgrid_telemetry.csv`, confirming accurate timestamping and threshold logic:

```text
2026-08-06 11:23:21,866,4.23,FAULT/TRIP
2026-08-06 11:23:34,767,3.75,FAULT/TRIP
2026-08-06 11:23:36,740,3.62,WARNING
2026-08-06 11:23:45,449,2.19,WARNING
2026-08-06 11:23:45,322,1.57,NORMAL
2026-08-06 11:24:01,78,0.38,NORMAL
```

Simultaneously, `scada_dashboard.py` rendered a live, scrolling waveform showing real-time solar input voltage crossing between operational safety zones as the hardware potentiometer was adjusted.

## 🚀 Next Steps: Phase 3

With data ingestion, disk logging, and live visual monitoring verified, the next phase will focus on:

1. Data analysis of recorded CSV logs using `pandas`.
2. Hardware fault injection testing (simulating grid line trips and high-frequency voltage spikes).
3. Adding visual/auditory pop-up alert mechanisms to the dashboard during FAULT/TRIP states.
