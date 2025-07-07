Summary: A real-time IoT solution designed to monitor manufacturing efficiency by tracking product throughput, machine downtime, and production speed (PPM), featuring a cloud-based dashboard and multi-channel automated alerts for proactive operational awareness.

Project Objective
The primary goal of this project was to replace manual, error-prone production tracking with an automated, data-driven system. The solution aimed to provide management and operators with real-time insights into the production line's performance, identify bottlenecks, and enable a proactive approach to maintenance and operational improvements.
Key Features
Real-time Product Counting: Utilizes a photoelectric sensor for high-accuracy, non-contact counting of products on a conveyor line.
Automatic Downtime Calculation: The system intelligently detects when the production line stops (if no product is detected within a configurable threshold) and calculates the duration of each stoppage.
Accurate Production Speed Analysis: Calculates the true production speed in Products Per Minute (PPM), considering only the actual machine running time by excluding downtime from the calculation.
Live Data & Instant Updates: The system provides two tiers of data updates:
Instantaneous: The total product count is pushed to the cloud the moment a new product is detected.
Aggregated: A detailed data packet containing per-minute production rate, downtime, and machine status is sent periodically for trend analysis.
Cloud-Based Data Visualization: A comprehensive dashboard was developed on the Ubidots platform, featuring:
Live status indicators (Running/Stopped).
Real-time production counters.
Historical charts for production speed and downtime analysis.
Daily performance summary widgets.
Automated Multi-Channel Alerts: An event-driven alerting system was configured. If the machine remains stopped for a predefined duration (e.g., 5 minutes), notifications are automatically sent via both Email and Telegram to alert relevant personnel.
Time-Aware Operations: The device synchronizes with an NTP server to be aware of the real-world time. This enables scheduled tasks, such as automatically logging the final daily production count at a specific time (e.g., 18:00) and resetting the counters for the next working day.
Technical Stack
Hardware:
Microcontroller: ESP32
Sensor: E18-D80NK Photoelectric (Infrared) Proximity Switch
Firmware & Software:
Language: C++ (within the Arduino Framework)
Libraries: UbidotsEsp32Mqtt, time.h
Key Concepts: Polling for digital inputs, state machine logic for debouncing, non-blocking timers (millis()), NTP time synchronization.
Cloud & Services:
IoT Platform: Ubidots
Protocols: Wi-Fi, MQTT, HTTPS
Features: Ubidots Dashboards & Widgets, Ubidots Events Engine, Webhooks
Third-Party Integration: Telegram Bot API
System Architecture & Data Flow
The E18-D80NK sensor, mounted on the production line, detects a product passing by and sends a LOW signal to the ESP32.
The ESP32 firmware detects this signal change. It immediately increments the daily product counter and publishes this single value to Ubidots for a real-time dashboard update.
The firmware continuously calculates downtime and aggregates per-minute production data.
Periodically (e.g., every 60 seconds), the ESP32 sends a packet of aggregated data (products/minute, downtime/minute, machine status) to Ubidots via MQTT.
The Ubidots Events Engine monitors the machine-status variable. If the status remains "Stopped" (value 0) for a set duration, it triggers two actions.
Action 1: An email is sent to a predefined address.
Action 2: A Webhook calls the Telegram Bot API, sending an alert message to a specified user or group chat.
At 18:00 each day, the ESP32 sends the final total product count to a separate variable on Ubidots and resets its daily counter.

