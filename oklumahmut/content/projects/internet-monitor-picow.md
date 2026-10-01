---
title: "24/7 Internet Health & Telemetry Monitor: From Pico W to Cloud-Accessible Edge Appliance"
date: 2026-09-30T00:00:00Z
draft: false
author: "Mahmut Oklu"
tags: ["Raspberry Pi", "Pico W", "IoT", "MicroPython", "Tailscale", "Networking", "Architecture"]
categories: ["Projects"]
description: "An end-to-end IoT and edge computing project designed to continuously measure Wi-Fi link quality, ping latency, jitter, and packet loss, serving an interactive telemetry dashboard accessible globally via Tailscale Funnel."
---

An end-to-end IoT and edge computing project designed to continuously measure Wi-Fi link quality, ping latency, jitter, and packet loss, serving an interactive telemetry dashboard accessible globally via Tailscale Funnel.

**GitHub Repository:** [https://github.com/mahmutoklu/pico-w-internet-monitor](https://github.com/mahmutoklu/pico-w-internet-monitor)

---

## 📌 Background: What is the Raspberry Pi Pico W?

The **Raspberry Pi Pico W** is a high-performance, ultra-low-cost ($6) microcontroller development board designed by Raspberry Pi:
* **Microcontroller vs. SBC:** Unlike a standard Single Board Computer (such as a Raspberry Pi 4 or 5) which boots a full operating system like Linux, the Pico W is a "bare-metal" microcontroller. It executes code directly on hardware with zero operating system overhead, typically programmed via MicroPython or C/C++.
* **Core Silicon (RP2040):** Features a custom Raspberry Pi RP2040 chip equipped with a Dual-core ARM Cortex-M0+ processor running at 133 MHz, 264 KB of internal multi-bank SRAM, and 2 MB of onboard QSPI flash memory.
* **Integrated Wireless (CYW43439):** Contains an onboard Infineon CYW43439 wireless subsystem supporting 2.4 GHz 802.11 b/g/n Wi-Fi and Bluetooth 5.2 with an onboard PCB antenna.
* **Energy Profile:** Draws less than 0.3 Watts under active networking load, making it an ideal candidate for always-on, power-efficient IoT appliances.

---

# PART 1: The Pico W Internet Monitor (Hardware Prototype & Firmware Engine)

---

### 1. Problem Statement

Diagnosing intermittent Wi-Fi degradation and ISP routing issues is notoriously difficult because standard network speed tests are manual, periodic, and consume substantial bandwidth while missing silent micro-dropouts that disrupt real-time applications. In this initial phase of research and development, I set out to create a dedicated, low-power telemetry appliance that continuously tracks home connectivity. Through this embedded implementation, I try to solve the persistent observability gap in residential networking by deploying an autonomous hardware agent that operates 24/7 without requiring an attached host computer.

To build an effective diagnostic tool, the system must capture multidimensional performance metrics rather than simple binary connection states. Therefore, the firmware architecture is engineered **not only** to measure physical Wi-Fi signal attenuation (RSSI in dBm) and latency through lightweight TCP handshakes, but also to evaluate real-time jitter, record timestamped dropouts with flash-wear mitigation, and serve a responsive web dashboard directly from the constrained memory of the microcontroller.

---

### 2. Technical Architecture & How It Works (Part 1)

```
                       +-----------------------------------------------------------+
                       |              Raspberry Pi Pico W (RP2040)                 |
                       |                                                           |
[Public CDN / DNS] <------ TCP Handshake RTT --+-> Metrics Engine                  |
(1.1.1.1 / 8.8.8.8)    |                       |   (Latency, Jitter, Packet Drop)  |
                       |                       |                                   |
                       |                       +-> Wi-Fi Driver (CYW43439 RSSI)    |
                       |                       |                                   |
                       |                       +-> In-Memory RAM Ring Buffer       |
                       |                       |   (Last 120 live data points)     |
                       |                       |                                   |
                       |                       +-> Flash Storage Logger            |
                       |                       |   (Batched CSV summaries)         |
                       |                       |                                   |
                       |                       +-> Async Web Server (Port 80)      |
                       +-------------------------------------+---------------------+
                                                             |
                                         HTTP / JSON API (Port 80)
                                                             |
                                                             v
                                            +---------------------------------+
                                            | Local Network Browser           |
                                            | (http://<pico-local-ip>)        |
                                            +---------------------------------+
```

#### Operational Pipeline:
1. **Lightweight Benchmarking:** Avoids heavy bandwidth-consuming speed tests. Instead, it measures precise round-trip times (RTT) via lightweight TCP SYN/ACK handshakes against public CDN DNS resolvers (`1.1.1.1:53` and `8.8.8.8:53`) using high-resolution millisecond timers (`time.ticks_ms()`).
2. **Dual-Tier Memory Protection:** 
   - High-frequency samples (every 3 seconds) are stored in an in-memory RAM ring buffer(120 points) to power the web UI's real-time graphs without causing flash wear.
   - Long-term statistics (min/avg/max latency, jitter, loss %, disconnect events) are aggregated and flushed to `network_log.csv` only once every 10 minutes to protect the onboard NOR flash memory from write fatigue.
3. **Embedded Web Telemetry:** A native asynchronous HTTP server (`uasyncio`) serves a modern, dark-themed dashboard (`index.html`) with live Chart.js visualizations directly to any device on the local network.

---

### 3. Tech Stack & Development Journey (Part 1)

#### Tech Stack
* **Microcontroller:** Raspberry Pi Pico W (RP2040 Dual-Core ARM Cortex-M0+, Infineon CYW43439 2.4GHz Wi-Fi).
* **Language & Runtime:** MicroPython with `uasyncio`.
* **Frontend:** Responsive HTML5, CSS3 (Dark Mode variables), Vanilla JavaScript (ES6+), Chart.js.
* **Storage:** Flash file system with CSV batching and flash-wear mitigation.

#### How I Developed It (Pair Programming with Google Antigravity)

I initiated the project by prompting Google Antigravity to architect the embedded monitor:

> **User Prompt:**  
> *"I bought raspberry pi pico w and I plugged in to this laptop. I would like to develop a mini dashboard app that monitors and logs internet connection strength and consistency. Think hard search web and propose me the project steps in a guideline."*

**Antigravity's Contribution:**
- Drafted a modular architecture (`config.py`, `wifi_manager.py`, `metrics.py`, `web_server.py`, `main.py`).
- Implemented non-blocking socket timing using `time.ticks_ms()` and `time.ticks_diff()`.
- Designed an async HTTP server that avoids third-party dependency overhead.

When testing the local simulator on macOS, an environment error occurred then I solved it by using Antigravity.

**Antigravity's Contribution:**
- Analyzed the trace and identified an `AttributeError` on `wlan.ifconfig()` caused by running hardware-specific MicroPython calls on macOS standard Python.
- Added platform abstraction layers so the codebase runs seamlessly on both development laptops and microcontrollers.

---
---

# PART 2: Deploying to Raspberry Pi Zero 2 W & Global Access via Tailscale Funnel

---

### 1. Problem Statement

Although the initial microcontroller prototype operated reliably inside the home subnet, its accessibility was strictly confined to local Wi-Fi coverage due to residential NAT and router firewall boundaries. In this second phase of development, I the challenge of achieving secure, perimeter-less remote observability by migrating the monitoring service to an always-on Linux edge appliance (Raspberry Pi Zero 2 W). This transition overcomes the limitations of dynamic residential IP addressing and avoids the severe security hazards of conventional port-forwarding.

By integrating modern encrypted overlay networking, the deployment is structured not only to maintain continuous, background telemetry collection on a dedicated edge device, but also to publish an authenticated, publicly verifiable HTTPS endpoint via Tailscale Funnel that permits global browser access without requiring third-party users or mobile clients to install specialized VPN software.

---

### 2. Technical Architecture & How It Works (Part 2)

```
                            [ Home Network / Edge ]
                           +-------------------------------------------------------+
                           | Raspberry Pi Zero 2 W (Linux OS)                      |
                           |                                                       |
 [Public DNS] <-- TCP RTT -+-> Metrics Engine (RTT, Jitter, Packet Drop, RSSI)     |
                           |                                                       |
                           |   +-> Python Async HTTP Server (Port 8080)            |
                           +---------------------------+---------------------------+
                                                       |
                                           Proxied via localhost:8080
                                                       |
                                                       v
                                        +------------------------------+
                                        |  Tailscale Funnel (Port 443) |
                                        +--------------+---------------+
                                                       |
                                  Encrypted Public HTTPS (Zero Port-Forwarding)
                                                       |
                                                       v
                                  +-----------------------------------------+
                                  | Global Internet Clients (Phones, Laptops|
                                  | https://<your-device>.<tailnet>.ts.net  |
                                  +-----------------------------------------+
```

#### Operational Pipeline:
1. **Linux Edge Migration:** The service was adapted into `pi_monitor.py`, configured to bind to `0.0.0.0:8080` so that all local and virtual network interfaces can access the HTTP dashboard.
2. **Encrypted WireGuard Mesh:** The Raspberry Pi Zero 2 W authenticates into a private Tailscale tailnet, receiving a secure virtual IP.
3. **Tailscale Funnel (Public Ingress):**
   - Using Tailscale Funnel, Tailscale provisions an automated TLS certificate for the device's MagicDNS domain.
   - Inbound requests from the public internet hit Tailscale's edge relays and are forwarded securely over encrypted WireGuard down to port 8080 on the Pi.
   - Zero router port forwarding, no static IP, and no firewall exceptions required.

---

### 3. Tech Stack & Development Journey (Part 2)

#### Tech Stack
* **Edge Hardware:** Raspberry Pi Zero 2 W (Quad-core 64-bit ARM Cortex-A53, 512MB RAM).
* **Operating System:** Raspberry Pi OS Lite (Debian Linux).
* **Language & Runtime:** Python 3 (`asyncio`, `socket`, `json`).
* **Remote Access & Networking:** Tailscale, Tailscale Funnel, Tailscale MagicDNS (`*.ts.net`), TLS/HTTPS reverse proxy.

#### How I Developed It (Pair Programming with Google Antigravity)

I transitioned the project from the microcontroller to Linux edge deployment with Antigravity:

> **User Prompt:**  
> *"Deploy the Pico W internet monitor project I made to one of the Raspberry Pis at home. Make it accessible via Tailscale and connect to it from outside the house. In other words, connect to the RPi from outside the house and use this application."*  
> *"I will deploy it to pi zero 2 w"*

**Antigravity's Contribution:**
- Refactored the core socket binding to `0.0.0.0:8080` in `pi_monitor.py` so the service could accept incoming VPN traffic.
- Outlined the deployment pipeline via `scp`, SSH configuration, and service execution on the Pi Zero 2 W.

When understanding the networking parameters:

> **User Prompt:**  
> *"tailscale ip -4) meaning?"*

**Antigravity's Contribution:**
- Explained CGNAT address spaces (`100.x.y.z`), VPN mesh routing, and how peer-to-peer WireGuard tunnels operate across NAT barriers.

When enabling public sharing without requiring VPN client installation on guest devices:

> **User Prompt:**  
> *"How can other users see this online internet monitoring tool without same tailscale account?"*  
> *"how can i Go to Access Controls / Settings and ensure Funnel is enabled for your account for this problem"*

**Antigravity's Contribution:**
- Configured the Tailscale Access Control Policy (ACL) by adding the required `nodeAttrs` block for Funnel enablement.
- Configured and activated the background funnel proxy (`sudo tailscale funnel --bg 8080`), producing a secure, public HTTPS link (`https://<device>.<tailnet>.ts.net`).

---

## 4. Key Highlights & Summary

| Dimension | Part 1: Microcontroller Prototype | Part 2: Cloud-Accessible Edge Appliance |
| :--- | :--- | :--- |
| **Hardware** | Raspberry Pi Pico W | Raspberry Pi Zero 2 W |
| **Silicon Architecture** | RP2040 Dual-core ARM Cortex-M0+ | Broadcom BCM2710A1 Quad-core ARM Cortex-A53 |
| **Runtime** | MicroPython (`uasyncio`) | Linux / Python 3 (`asyncio`) |
| **Accessibility** | Local Home Wi-Fi Only (`http://192.168.x.x`) | Global Secure HTTPS (`https://<device>.ts.net`) |
| **Network Security** | Local subnet only | Zero-Trust Encrypted WireGuard Tunnel (No port-forwarding) |
| **Storage Model** | NOR Flash-safe batched CSV | Persistent Linux Filesystem |
| **Power Profile** | Ultra-low power (~0.3W) | Low-power edge appliance (~0.7W) |