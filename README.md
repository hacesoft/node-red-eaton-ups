[🇨🇿 Česky](README_CZ.md) | [🇬🇧 **English**](README.md)

---

# Eaton / NUT UPS Module for Node-RED

This module enables seamless integration of Eaton Uninterruptible Power Supplies (UPS) — as well as any other UPS devices supported by **NUT (Network UPS Tools)** — into Node-RED.

Key Features:
* **Direct NUT Server Communication** over TCP sockets using the native Node.js `net` module.
* **Real-time State & Event Detection** (mains power loss, recovery, battery low, charging state, etc.).
* **Push Notifications** via the **ntfy** service.
* **Rich User Interface** built for Node-RED Dashboard v2 (Status Panel & 2-Axis SVG History Chart).
* **Runtime Configuration UI** allowing on-the-fly parameter changes without editing code.
* **Configuration Persistence** to disk using JSON files.
* **Global State & API Integration** exposing read-only data for external platforms (LINEA / REST APIs).

---

## 🛠 How the Module Works & System Integration

1. **Scheduler & Socket Queries (UPS Scheduler / NUT Request):**
   * A timer (`inject`) triggers a query to the NUT server every 5 seconds under normal conditions.
   * Using the native Node.js `net` module (loaded within the function node setup, eliminating the need to edit `settings.js`), a TCP socket opens to the configured Host and Port, issuing a `LIST VAR <upsName>` command.
   * The response is ingested, processed, and the TCP connection is gracefully closed.

2. **Data Parsing (UPS Parser):**
   * The raw response is parsed into key-value pairs (`VAR <upsName> <key> "<value>"`).
   * Operational statuses are calculated (e.g., `OL` = Online, `OB` = On Battery, `LB` = Low Battery, `CHRG` = Charging).
   * Derived metrics are calculated (real power, apparent power, available headroom, remaining battery runtime in minutes/seconds).
   * **Global Context State:**
     * `global.get("ups")` – complete parsed object accessible across any Node-RED flow.
     * `global.get("lineaApiUpsState")` – clean read-only snapshot for external REST API endpoints.
   * **Change Detection:** Triggers distinct events upon status transitions (`POWER_LOST`, `POWER_RESTORED`, `BATTERY_LOW`).

3. **Notifications (ntfy):**
   * Upon state change events, formatted messages and priority headers are constructed.
   * If `ntfyUrl` is specified in the configuration, an HTTP POST request is sent to the designated ntfy server and topic.

4. **Communication Watchdog:**
   * Node-RED monitors the timestamp of the last successful response. If no response is received within 15 seconds, the state switches to `NUT OFFLINE`.

---

## 💾 Data Persistence & Runtime State

The module separates configuration persistence on disk from live runtime state kept in RAM.

### 1. Configuration (`config.upsConfig`)
When parameters (Host, Port, UPS Name, Timeout, ntfy URL) are modified in the Dashboard UI and the **Save Configuration** button is pressed:
1. The `UPS CONFIG handler` updates live memory: `global.set("config.upsConfig", ...)`
2. An output signal is routed via a `link out` node (`UPS CONFIG -> SAVE config_hacesoft.json`).
3. The host system's configuration saver merges `config` and writes it to a JSON file on disk (e.g., `/data/config_hacesoft.json`).
4. Upon Node-RED restart, the JSON file is reloaded into `global.config`, where the UPS module retrieves it at boot.

**Configuration JSON Example:**
```json
{
  "upsConfig": {
    "host": "192.168.1.100",
    "port": 3493,
    "upsName": "eaton",
    "timeoutMs": 4000,
    "ntfyUrl": "https://ntfy.sh/my-ups-alerts"
  }
}
```

### 2. Runtime State in RAM (`global.ups`)
Every polling cycle (every 5 seconds), the live `global.get("ups")` object is updated in memory:

```javascript
{
  "online": true,
  "statusText": "Online (Mains OK)",
  "batteryCharge": 100,
  "batteryRuntimeSec": 1800,
  "batteryRuntimeMin": 30,
  "loadPct": 15,
  "realPowerW": 120,
  "apparentPowerVA": 150,
  "inputVoltage": 230,
  "outputVoltage": 230,
  "lastUpdate": "2026-09-19T17:00:00.000Z",
  "statusFlags": {
    "OL": true,
    "OB": false,
    "LB": false,
    "CHRG": false
  }
}
```

---

## ⚙️ NUT Server Setup Guide (Synology, pfSense, OPNsense)

For Node-RED to communicate with the UPS, the host system connected to the UPS via USB must run a **NUT Server** listening on port `3493`.

### 1. Synology NAS
If your UPS is connected to a Synology NAS via USB:
1. Navigate to **Control Panel $\rightarrow$ Power $\rightarrow$ UPS**.
2. Check **Enable UPS support**.
3. Select **Synology UPS server** as the UPS type.
4. Click **Permitted UPS devices** (Allowed IP addresses).
5. Add the **IP address of the machine running Node-RED**.
6. **Node-RED Parameters:**
   * **Host:** IP address of the Synology NAS
   * **Port:** `3493`
   * **UPS name:** `ups` (Synology uses `ups` as its internal default name)

---

### 2. pfSense Firewall
1. **Package Installation:** Go to **System $\rightarrow$ Package Manager $\rightarrow$ Available Packages**, search for and install **nut**.
2. **Service Configuration:** Go to **Services $\rightarrow$ UPS / NUT**.
   * **UPS Mode:** Set to `Local USB`.
   * **UPS Name:** Set a name (e.g., `eaton` or `ups`).
   * **Driver:** Select the appropriate driver (typically `usbhid-ups` for Eaton).
3. **Network Listening (upsd.conf):**
   In the **Advanced settings** section (or the custom `upsd.conf` field), add:
   ```text
   LISTEN 0.0.0.0 3493
   ```
4. **Firewall Rule:** Under **Firewall $\rightarrow$ Rules**, create a `Pass` rule for **TCP** traffic on port `3493` originating from your Node-RED IP address.

---

### 3. OPNsense Firewall
1. **Plugin Installation:** Go to **System $\rightarrow$ Firmware $\rightarrow$ Plugins**, search for and install **os-nut**.
2. **Service Configuration:** Go to **Services $\rightarrow$ Nut $\rightarrow$ Configuration**.
   * Under **General Settings**, check **Enable Nut**.
   * Under **USB Service**, check **Enable**, enter a **Name** (`eaton` or `ups`), and select the driver (`usbhid-ups`).
3. **Network Service:**
   * Under **UPS Network Service**, check **Enable**.
   * Set **Listen Address** to `0.0.0.0` (or specific interface IP).
   * Set **Port** to `3493`.
4. **Firewall Rule:** Under **Firewall $\rightarrow$ Rules $\rightarrow$ [Interface]**, add a rule allowing **TCP** traffic to port **3493** from the Node-RED IP.

---

## 🚀 How to Integrate into Custom / Standalone Node-RED Projects

This flow is modular and can easily be ported to any standard Node-RED environment.

### Step 1: Import & Dependencies
1. In Node-RED, select **Import** and paste the flow JSON.
2. Ensure you have **Dashboard v2** (`@flowfuse/node-red-dashboard`) installed.

### Step 2: Assign UI Nodes to Groups
1. Open the Dashboard tab in the right sidebar.
2. Locate the nodes:
   * `UI CONFIG` (Configuration Form)
   * `UPS Panel` (Main Dashboard Card)
   * `GRAF_SVG_2osy` (2-Axis History Chart)
3. Assign each node to your desired Dashboard `ui-group` and `ui-page`.

### Step 3: Handle Configuration Persistence
If your project does not use an external global config handler (`SAVE config_hacesoft.json`):

* **Option A (File System Persistence):**
  Disconnect the `link out` node (`UPS CONFIG -> SAVE config_hacesoft.json`) and route the output of `UPS CONFIG handler` directly into a standard `write file` node saving to a target JSON file.
* **Option B (Context Store Persistence):**
  Configure persistent context in your `settings.js` file (`contextStorage: { default: { module: 'localfilesystem' } }`). In this mode, values saved to `global.set("config.upsConfig", ...)` automatically persist across system reboots without requiring file nodes.

### Step 4: Accessing UPS Data in Custom Flows
Once running, any node can inspect UPS metrics directly from the global context:

* **In a `Function` Node:**
  ```javascript
  const ups = global.get("ups");
  if (ups && !ups.online) {
      node.warn("UPS connection lost!");
  } else if (ups && ups.statusFlags.OB) {
      node.warn(`Operating on battery power! Estimated runtime: ${ups.batteryRuntimeMin} minutes.`);
  }
  ```
* **In a `Change` Node:**
  Set `msg.payload` to `global.ups.batteryCharge`.

---

## 🔧 Configuration UI (`UPS CONFIG`)

The Dashboard configuration card allows modifying connection parameters without editing code.

### Fields:
| Field | Type | Description / Example |
| :--- | :--- | :--- |
| **NUT host** | Text | Hostname or IP address of the NUT server (e.g., Synology, pfSense, OPNsense, or Linux server). |
| **Port** | Number | NUT server TCP port (default standard is `3493`). |
| **UPS name** | Text | Identifier configured in NUT server `ups.conf` (Synology = `ups`, pfSense/OPNsense = e.g., `eaton`). |
| **TCP timeout [ms]** | Number | Socket response timeout in milliseconds (default: `4000` ms, min. 500 ms). |
| **ntfy URL** | Text *(optional)* | Endpoint URL for push notifications (e.g., `https://ntfy.sh/my-ups-alerts`). Leave empty to disable ntfy notifications. |

### Actions:
* **Test NUT:** Executes an immediate, single TCP query to the configured Host + Port + UPS Name. Results (Success/Failure) display next to the button without persisting settings.
* **Save Configuration:** Updates `global.config.upsConfig` in memory and triggers file persistence. The module immediately adopts the new parameters.

---

## 📊 Dashboard UI Description

The UI consists of two core components:

### 1. UPS Panel (`UPS Panel`)
Real-time dashboard displaying key telemetry and status flags:

* **Status Banner:**
  * Primary operational mode: `Mains OK` (Green), `ON BATTERY` (Flashing Orange), `BATTERY LOW` (Flashing Red).
  * Secondary indicators (*charging, discharging, overload, replace battery, bypass*).
* **Battery & Runtime:**
  * Dynamic visual capacity gauge (%) with threshold-based color coding.
  * Prominent remaining runtime counter (minutes & seconds).
  * Low battery shutdown threshold display (e.g., *shutdown at 20%*).
* **Power & Load:**
  * Load percentage meter (% with green $\rightarrow$ orange $\rightarrow$ red transition at $>80\%$).
  * Active Power (**W**), Apparent Power (**VA**), Available Power Reserve (**W**), and Rated Maximum Capacity (**W**).
* **Mains & Battery Telemetry:**
  * Input Voltage (**V**), Output Voltage (**V**), Grid Frequency (**Hz**), and battery chemistry type.
* **Footer:**
  * Manufacturer, UPS model designation, and firmware revision.

### 2. History Chart (`GRAF_SVG_2osy`)
Dynamic, vector-based 2-axis SVG chart tracking long-term trends:

* **Metrics Plotted:**
  * 🟠 **Active Power (W)** – Real load consumption.
  * 🟢 **Load (%)** – Percentage capacity utilization.
  * 🔵 **Input Voltage (V)** – Plotted against the right Y-axis (200–250 V range) to monitor power quality and brownouts.
* **Chart Features:**
  * **Interactive Legend:** Toggle individual series on or off by clicking legend entries.
  * **Crosshair Tooltip:** Hovering over the chart displays a vertical crosshair with precise values for that timestamp.
  * **Buffer Capacity:** Retains up to 720 historical data points.