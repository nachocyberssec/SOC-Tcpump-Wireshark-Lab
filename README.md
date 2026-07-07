# SOC-Tcpump-Wireshark-Lab

## Project Goal
The goal of this project is investigating a network alert for a potential port scan. I simulated a stealth port scan (SYN scan) in a controlled Linux environment, captured the raw network traffic using `tcpdump`, and analyzed the logs in `Wireshark` to find the attacker's footprints (IoCs) and figure out what happened.

---

## Key Skills Demonstrated
* **Traffic Analysis:** Using Wireshark filters to cut through network noise and isolate suspicious activity.
* **Network Monitoring:** Capturing live traffic on a Linux system via terminal using `tcpdump`.
* **Understanding Attacks:** Knowing how an attacker runs reconnaissance (Nmap) and how the TCP three-way handshake works behind the scenes.

---

## Tools Used
* **Analysis:** Wireshark
* **Packet Capture:** Tcpdump (Linux Terminal)
* **Attack Simulation:** Nmap (Mapping to MITRE ATT&CK [T1046 - Network Service Scanning](https://attack.mitre.org/techniques/T1046/))

---

## Step-by-Step Walkthrough

### 1. Catching the Traffic (Tcpdump)
To catch the attack without messing with the system's performance, I started a packet capture with `tcpdump` on the local loopback interface (`lo`) and saved the raw data into a `.pcap` file:
```bash
sudo tcpdump -i lo -w escaneo_local.pcap
```
### 2. Simulating the Attack (Nmap SYN Scan)
Next, I simulated an attacker trying to map out open doors on the system by running a **SYN Scan (Stealth Scan)**. 
*Quick Technical Note:* This type of scan requires `sudo` privileges because Nmap needs to craft raw network packets manually instead of letting the OS handle the connection normally. It drops the connection mid-way (never sending the final `ACK`) to try and stay under the radar of basic logs.
```bash
sudo nmap -sS -F localhost
```

**What Nmap found:**
* **Host status:** UP
* **Open ports discovered:** 22/tcp (SSH) and 443/tcp (HTTPS).

---

## Investigating in Wireshark (SOC Analysis)

The capture file recorded a total of 901 packets in just a couple of seconds. To see if this was a false alarm or a real threat, I opened the `.pcap` file in Wireshark.

### A. Isolating the Attacker
To filter out the background noise and see exactly what the scan was doing, I used this specific Wireshark filter:
```text
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

**Why this filter?** It looks only for packets asking to open a connection (`SYN`) without any confirmation (`ACK`). Seeing a massive flood of these packets hitting different ports consecutively in milliseconds is the textbook definition of an automated port scan.

### B. The Evidence (Screenshot)

<img width="1552" height="1012" alt="wire" src="https://github.com/user-attachments/assets/57816d85-8158-472d-80c2-11e2acfcc40e" />

### C. Final Analyst Findings
1. **Confirmed Incident:** This is a **True Positive**. An internal tool or attacker ran an automated, rapid sweep to find open entry points.
2. **Exposed Services:** The scan successfully detected that the system has two active services listening: **Port 22 (SSH)** and **Port 443 (HTTPS)**.
3. **SSH Traffic Check:** While looking at the logs, I also checked earlier control traffic and confirmed that the SSH sessions on Port 22 are traveling fully encrypted (`Server/Client: Encrypted packet`). This means that even though the attacker found the port, they cannot read any session data.
