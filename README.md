#RoboGrip X5 is a robotic arm project designed for pick-and-place operations, developed using a combination of mechanical design and embedded systems.
#This project utilizes an Arduino Uno r3 and 5 servo motors, which are controlled through a Python-based GUI interface. The GUI allows users to operate the robotic arm in #a simple and interactive way.
#Key Features:
#Python GUI for real-time control
#Smooth movement using 5 servo motors
#Arduino-based hardware integration
#Designed for pick-and-place applications
#Hardware Used:
#Arduino Uno R3
#5 × Servo Motors (SG90)
#Power Supply (5V Battery)
#The system enables precise control of the robotic arm through a user-friendly interface. It demonstrates the integration of software and hardware for automation tasks,
#making it suitable for educational and prototype-level industrial applications.

# RoboticArm
import tkinter as tk
from tkinter import ttk, messagebox
import serial
import serial.tools.list_ports
import threading
import time

BAUD_RATE = 9600
SERVO_NAMES = ["Base", "Shoulder", "Elbow", "Wrist", "Gripper"]
DEFAULT_ANGLES = [90, 90, 90, 90, 90]

PRESETS = {
    "Home":       [90,  90,  90,  90,  90],
    "Pick Up":    [90,  45,  120, 60,  30],
    "Drop Off":   [180, 80,  100, 70,  30],
    "Wave":       [90,  130, 45,  90,  90],
    "Full Extend":[90,  90,  45,  90,  90],
}

class RoboticArmGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("🦾 Robotic Arm Controller")
        self.root.geometry("680x700")
        self.root.configure(bg="#0d1117")

        self.serial_conn = None
        self.connected = False

        self.slider_vars = []
        self.angle_labels = []
        self.last_sent = [None]*5  # prevent spam

        self.status_var = tk.StringVar(value="● Disconnected")

        self._build_ui()

    # ── UI ─────────────────────────────
    def _build_ui(self):
        tk.Label(self.root, text="🦾 ROBOTIC ARM",
                 font=("Courier New", 20, "bold"),
                 fg="#00e5ff", bg="#0d1117").pack(pady=10)

        # Connection
        frame = tk.Frame(self.root, bg="#161b22")
        frame.pack(fill="x", padx=20, pady=10)

        self.port_var = tk.StringVar()
        self.port_combo = ttk.Combobox(frame, textvariable=self.port_var, width=15)
        self.port_combo.pack(side="left", padx=5)
        self._refresh_ports()

        tk.Button(frame, text="Refresh", command=self._refresh_ports).pack(side="left")

        self.conn_btn = tk.Button(frame, text="CONNECT", command=self._toggle_connection)
        self.conn_btn.pack(side="left", padx=10)

        tk.Label(frame, textvariable=self.status_var,
                 fg="red", bg="#161b22").pack(side="right")

        # Sliders
        for i, name in enumerate(SERVO_NAMES):
            frame = tk.Frame(self.root, bg="#161b22")
            frame.pack(fill="x", padx=20, pady=5)

            tk.Label(frame, text=name, fg="white", bg="#161b22", width=10).pack(side="left")

            var = tk.IntVar(value=90)
            self.slider_vars.append(var)

            slider = tk.Scale(frame, from_=0, to=180, orient="horizontal",
                              variable=var,
                              command=lambda val, idx=i: self._on_slider(idx, val))
            slider.pack(side="left", fill="x", expand=True)

            lbl = tk.Label(frame, text="90°", fg="cyan", bg="#161b22")
            lbl.pack(side="left", padx=5)
            self.angle_labels.append(lbl)

        # Buttons
        btn_frame = tk.Frame(self.root, bg="#0d1117")
        btn_frame.pack(pady=10)

        tk.Button(btn_frame, text="RESET", command=self._reset_all).pack(side="left", padx=5)
        tk.Button(btn_frame, text="STATUS", command=self._get_status).pack(side="left", padx=5)

        for name in PRESETS:
            tk.Button(btn_frame, text=name,
                      command=lambda p=name: self._run_preset(p)
                      ).pack(side="left", padx=3)

    # ── SERIAL ─────────────────────────
    def _refresh_ports(self):
        ports = [p.device for p in serial.tools.list_ports.comports()]
        self.port_combo["values"] = ports
        if ports:
            self.port_combo.set(ports[0])

    def _toggle_connection(self):
        if self.connected:
            self._disconnect()
        else:
            self._connect()

    def _connect(self):
        try:
            self.serial_conn = serial.Serial(self.port_var.get(), BAUD_RATE, timeout=1)
            time.sleep(2)
            self.connected = True
            self.status_var.set("● Connected")
            self.conn_btn.config(text="DISCONNECT")
            threading.Thread(target=self._read_serial, daemon=True).start()
        except Exception as e:
            messagebox.showerror("Error", str(e))

    def _disconnect(self):
        self.connected = False
        if self.serial_conn:
            self.serial_conn.close()
        self.status_var.set("● Disconnected")
        self.conn_btn.config(text="CONNECT")

    def _send(self, cmd):
        if self.connected and self.serial_conn:
            try:
                self.serial_conn.write((cmd+"\n").encode())
            except:
                self._disconnect()

    def _read_serial(self):
        while self.connected:
            try:
                if self.serial_conn.in_waiting:
                    print(self.serial_conn.readline().decode().strip())
            except:
                break

    # ── CONTROLS ───────────────────────
    def _on_slider(self, idx, val):
        angle = int(val)
        self.angle_labels[idx].config(text=f"{angle}°")

        # prevent spamming same value
        if self.last_sent[idx] == angle:
            return

        self.last_sent[idx] = angle

        # slight delay to smooth movement
        self.root.after(50, lambda: self._send(f"S{idx+1},{angle}"))

    def _run_preset(self, name):
        angles = PRESETS[name]

        def run():
            for i, a in enumerate(angles):
                self.slider_vars[i].set(a)
                self._send(f"S{i+1},{a}")
                time.sleep(0.15)

        threading.Thread(target=run, daemon=True).start()

    def _reset_all(self):
        self._send("RESET")
        for i in range(5):
            self.slider_vars[i].set(90)

    def _get_status(self):
        self._send("STATUS")


if __name__ == "__main__":
    root = tk.Tk()
    app = RoboticArmGUI(root)
    root.mainloop()
