# Case Study: Troubleshooting SSH Socket Deadlock on Huawei VRP

  ## 1. Problem Description

  A newly deployed access switch was inaccessible via **SSH/STelnet** from a remote management subnet.

  - **L2 Connectivity**: Successful pings between the local aggregation gateway and the access switch.

  - **L3 Connectivity**: ARP was visible on the gateway, but the remote management server could not reach the switch initially.

  - **Service Failure**: Terminal emulators (SecureCRT) reported `Connection refused` on the standard **Port 22**.
---

## 2. Diagnostic Steps & Findings

### Step 1: Service Verification

First, confirm if the SSH service is globally enabled within the VRP system.

- **Command**: `display ssh server status`
  
- **Observation**: `STELNET IPv4 server` was set to `Enable`. However, remote access remained blocked.
  

### Step 2: Identifying the "Ghost Port" (Critical Finding)

The most crucial step was inspecting the actual **TCP Socket listeners** at the OS level.

- **Command**: `display tcp status`
  
- **Finding**: The switch was **not** listening on Port 22. Instead, the process was bound to **TCP 64443**.
  
- **Conflict**: Even after manually forcing the port via `ssh server port 22`, the listener remained stubbornly stuck on `64443`.
  

### Step 3: Application Layer Debugging

To rule out firewall or ACL interference, we attempted to trace the handshake.

- **Commands**:`terminal monitor`,`terminal debugging`,`debugging ssh server all`
  
- **Observation**: **Zero logs** were generated during connection attempts.
  
- **Conclusion**: The TCP handshake was being rejected at the **Transport Layer (TCP)** before reaching the **Application Layer (SSH)**. Because the listener was on the wrong port, the switch sent a TCP RST (Reset) immediately.
  

---

## 3. Root Cause & Resolution

### **Root Cause: Socket Deadlock**

The failure was due to a **Socket Deadlock** within the VRP management plane. The system's TCP stack failed to unbind the high-order port (`64443`) and re-bind to the standard port (`22`) despite valid CLI configuration changes. This is typically caused by:

1. Legacy security hardening templates pre-applied to the hardware.
  
2. A hanging management process failing to release the socket resources.
  

### **Resolution**

A **system reboot** was performed to flush the kernel TCP stack and restart the management listener processes.

- **Result**: Post-reboot, `display tcp status` confirmed `0.0.0.0:22` in the `Listening` state. Remote access was fully restored.

---

## 4. Key Takeaways for Network Engineers

1. **Trust `display tcp status`**: In network OSs, the application status (SSH Enabled) doesn't always reflect the socket reality. If the port isn't "Listening" in the TCP table, the service is effectively down.
  
2. **Debugging Limits**: Application-level debugging (SSH) is invisible if the failure occurs at the Transport Layer (TCP).
  
3. **Process vs. Config**: A "Success" message in the CLI means the configuration was saved to the database, but it does not guarantee the underlying OS process has updated its state. When logic fails, a reboot is often required to reset the kernel hooks.
  

---

## 🛠️ Appendix: Complete Command Reference & Diagnostic Logic

To ensure a structured approach for future incidents, the following commands used in this case are categorized by their role in the troubleshooting lifecycle.

### 1. Connectivity & Layer 3 Diagnostics

*Before troubleshooting the application (SSH), bidirectional IP reachability must be confirmed.*

- **`display ip routing-table`**
  
  - **Purpose**: To verify if the switch has a valid return path to the management subnet.
    
  - **Finding**: Confirmed the lack of a default gateway for cross-segment traffic.
    
- **`ip route-static 0.0.0.0 0.0.0.0 <Gateway_IP>`**
  
  - **Purpose**: Configures the default static route to ensure the switch can reply to remote requests.
- **`ping -a <Source_IP> <Target_IP>`**
  
  - **Purpose**: Performs a ping from a specific source interface. This tests if the management VLAN interface is correctly routing packets.
- **`display arp`**
  
  - **Purpose**: Checks the Address Resolution Protocol table to ensure Layer 2 to Layer 3 mapping is healthy.

### 2. Protocol Stack & Socket Investigation

*This phase moves from the "Configuration" to the "Kernel Runtime."*

- **`display ssh server status`**
  
  - **Purpose**: To check if the SSH application is globally enabled.
- **`display tcp status`** (⭐ **The "Smoking Gun" Command**)
  
  - **Purpose**: To view active listeners in the TCP stack.
    
  - **Finding**: Discovered that Port 22 was missing and the system was erroneously listening on Port `64443`.
    
- **`ssh server port 22`**
  
  - **Purpose**: Manually attempts to re-bind the SSH service to the standard port.

### 3. Real-Time System Debugging

*Used to confirm if packets are reaching the application or being dropped by the TCP stack.*

- **`terminal monitor`**
  
  - **Purpose**: Enables the display of system logs and debug messages on the current console session.
- **`terminal debugging`**
  
  - **Purpose**: Activates the terminal's ability to show low-level debug outputs.
- **`debugging ssh server all`**
  
  - **Purpose**: Captures every step of the SSH handshake.
    
  - **Conclusion**: No output was generated, proving the TCP handshake was rejected before the SSH process could intervene.
    

### 4. Recovery & System Restoration

*Actions taken to resolve the "Socket Deadlock" and initialize encryption.*

- **`reboot`**
  
  - **Purpose**: The final solution to flush the kernel's hung TCP stack and force management processes to re-bind to the correct sockets.
- **`rsa local-key-pair create`**
  
  - **Purpose**: Generates the RSA key pairs required for SSH encryption (recommended size: 2048 bits).
- **`ssh server compatible-enable`**
  
  - **Purpose**: Enables algorithm compatibility for older SSH clients (e.g., legacy SecureCRT or PuTTY versions).
- **`ssh server-source all-interface`**
  
  - **Purpose**: Ensures the SSH service listens on all valid IP interfaces.

### 5. Security Hardening & VTY Access Control

*Post-troubleshooting steps to secure the device.*

- **`local-user <User> service-type ssh`**
  
  - **Purpose**: Restricts the specific user to SSH access only (disabling Telnet/FTP for that user).
- **`undo telnet server enable`**
  
  - **Purpose**: Globally disables the insecure Telnet protocol.
- **`user-interface vty 0 4`**
  
  - **Purpose**: Enters the Virtual Type Terminal lines configuration.
- **`protocol inbound ssh`**
  
  - **Purpose**: The "Physical Barrier"—rejects any non-SSH traffic (like Telnet) from even attempting a connection on the VTY lines.

---
