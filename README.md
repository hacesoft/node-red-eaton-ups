[🇨🇿 **Česky**](README_CZ.md) | [🇬🇧 English](README.md) | [Linea project](https://github.com/hacesoft/Linea)

# Eaton / NUT UPS Module for Node-RED

A UPS monitoring module using **NUT (Network UPS Tools)**, developed as part of the **LINEA / GridSight** project. It displays power, battery, and load status in Dashboard 2.0, keeps a short history, and sends ntfy notifications for selected state changes.

The module can also be used in a standalone Node-RED project. In that case, assign its Dashboard widgets and configure both saving and loading of settings as described below.

<img width="405" height="640" alt="UPS panel example" src="https://github.com/user-attachments/assets/dedcc061-7939-4db8-b521-5785e930d694" />

## Contents

- [Features](#features)
- [Requirements](#requirements)
- [How the module works](#how-the-module-works)
- [Installation in LINEA](#installation-in-linea)
- [Standalone use](#standalone-use)
- [NUT server and connection checks](#nut-server-and-connection-checks)
- [Dashboard configuration](#dashboard-configuration)
- [Status panel and history](#status-panel-and-history)
- [Events and ntfy](#events-and-ntfy)
- [Data interface](#data-interface)
- [Known limitations of this export](#known-limitations-of-this-export)
- [Troubleshooting](#troubleshooting)
- [Post-installation checklist](#post-installation-checklist)

## Features

- Reads NUT variables over TCP using `LIST VAR <upsName>`.
- Distinguishes mains power, battery operation, low battery, and other status flags.
- Displays available battery charge, runtime, power, load, voltage, and output frequency measurements.
- Provides a configuration form and a one-time connection test.
- Generates `POWER_LOST`, `POWER_RESTORED`, and `BATTERY_LOW` events.
- Sends optional HTTP notifications to the configured ntfy URL.
- Makes data available to other flows through global context and prepares an operational snapshot for the LINEA API.

**This is a monitoring module.** It does not send shutdown commands to the UPS, control outlets, change UPS settings, or automatically shut down a NAS or computer. Protective shutdown must be handled by the appropriate NUT client or host system.

The module is not limited to Eaton. Other UPS devices can be used if their NUT server provides readable variables in the expected format. Available data varies between models; compatibility with a particular model must be verified.

## Requirements

| Component | Requirement |
| --- | --- |
| Node-RED | An environment supporting the Function nodes used here and Dashboard 2.0. The export does not specify a minimum verified Node-RED version. |
| Dashboard | The `@flowfuse/node-red-dashboard` package; the export lists version `1.30.2`. The original `node-red-dashboard` is not a substitute for Dashboard 2.0. |
| TCP library | The `UPS NUT request (CONFIG)` node maps the variable `net` to the built-in Node.js module `net`. |
| NUT server | Reachable from the actual network environment running Node-RED, usually on TCP port `3493`. |
| Configuration | The `global.config.upsConfig` object, which can be created by saving the form. |
| Persistence | LINEA's storage mechanism or one of the standalone options below. |
| ntfy | Optional; not required for measurements or the Dashboard. |

The environment must allow `functionExternalModules` to use the library list in a Function node. If this option is disabled in your installation, add `functionExternalModules: true` to the existing configuration object in `settings.js` and restart Node-RED. Preserve existing settings. Operation without any changes to `settings.js` therefore cannot be guaranteed for every installation. See the [Function node documentation](https://nodered.org/docs/user-guide/writing-functions#using-the-functionexternalmodules-option).

The module itself does not require Modbus, MQTT, Victron, or a Nextcloud connection. It sends NUT queries directly, without an additional dedicated NUT node.

## How the module works

Node names below are preserved exactly as they appear in the supplied flow, including Czech names.

| Node | Role and connections |
| --- | --- |
| `UPS scheduler 1s / NUT 5s` | Sends its first trigger 2 seconds after deployment, then every second. Triggers both the query node and the watchdog. |
| `UPS NUT request (CONFIG)` | Reads configuration, limits normal queries to at least 5 seconds apart, opens a TCP connection, and reads the NUT response. Output 1 feeds the parser and a disabled debug node; output 2 returns test results to the configuration handler. |
| `Parser UPS` | Parses `VAR` lines and updates global data. Output 1 feeds the panel and history; output 2 carries an event when one is generated. |
| `Buffer historie` | Keeps up to 720 points in flow context and sends the entire array to the chart. |
| `Text notifikace` | Builds notification text and, when applicable, HTTP headers from an event. |
| `ntfy URL nastavena?` | Passes messages to the HTTP node only when an ntfy URL is configured. |
| `ntfy POST (UPS config)` | Sends the HTTP request. Its response has no downstream consumer in this export. |
| `UPS CONFIG handler` | Output 1: form response; output 2: save request; output 3: NUT test. |
| `NUT watchdog` | Checks the age of the last successful communication every second. Sends a message only when its state changes; its output is not connected in this export. |

The normal cycle reads configuration again on every trigger. Newly saved values are therefore used on the next permitted query without another Deploy. A test bypasses the five-second limit, but also updates the last-query timestamp.

## Installation in LINEA

1. Export a backup of your current flows before importing.
2. Check whether the `MODULE::UPS::EATON` group already exists. It is already included in the supplied complete project, so another copy is unnecessary.
3. When updating, replace the corresponding module and check its connections rather than running two copies simultaneously.
4. Keep the Dashboard groups or assign all three templates to your existing Dashboard: `UPS CONFIG`, `UPS Panel`, and `GRAF_SVG_2osy`.
5. Check the external link from `UPS CONFIG -> SAVE config_hacesoft.json` to `SAVE_NEW_CONFIG`. The target ID in the reference export is `3c18ac53a47a79d8`; IDs may change during import.
6. Verify initialization and loading of the shared `global.config`, then fill in the UPS configuration, test it, and save it.

### Saving settings in LINEA

The handler uses `global.get("setConfigProperty")` when that function exists. Otherwise, it updates `upsConfig` directly inside `global.config`. Its second output sends only:

```json
{"topic":"save","payload":"save"}
```

The shared `Create/Save File` node reads the actual `global.config` object, serializes it as JSON, and passes it to a file node. In the supplied LINEA project, the default directory is `/data/config_FLOW/` and the main file is `config_hacesoft.json`; the final path depends on the initialized `sValid_Patch` and `filename` values.

For a normal save request, this shared handler also produces outputs for the Shelly and Daikin configurations. It is the project's shared saving branch, not an isolated UPS file writer.

The form displays confirmation as soon as it updates context and sends the request. **This confirmation alone does not prove that writing to disk succeeded.** Check the file and confirm that settings survive a restart.

### API and GridSight integration

The parser writes `global.lineaApiUpsState`. In the supplied LINEA project, `Build status API 1.0.0` reads it and places it in `ups.data` in the response from `GET /api/v1/status`, alongside an `ups.available` flag.

The standalone UPS export contains no `http in` or `http response` node. Creating the snapshot therefore does not expose a REST API by itself. External clients such as GridSight must use the host project's API. Nextcloud archiving and its intervals are not implemented in this module and cannot be confirmed from these files.

## Standalone use

### 1. Import and Dashboard

1. Install Dashboard 2.0 and check that `net` is available in the Function node.
2. Import `eaton_flows_19092026_1854.json` into your chosen flow tab. The export is a selection of a group and configuration nodes; it does not include its own `tab` node.
3. Check the imported Dashboard configuration. The export includes the `/dashboard` base, `/FVE` and `/config` pages, a theme, and two groups. In an existing project, you can use your own assignments instead.
4. Assign `UPS Panel` and the chart to the operational group, and `UPS CONFIG` to the configuration group. Check widget widths and ordering.
5. Disconnect the external link to the unavailable LINEA save handler and choose one persistence option below.

For normal standalone use, the handler's fallback works without `setConfigProperty`. If your project already uses this global function, it must have compatible behavior: setting a value at the path `upsConfig.<property>`.

### 2A. Saving to a separate JSON file

This option leaves runtime context in memory and saves only UPS settings. The additional nodes described below are not included in the supplied export.

**Saving:** Connect the second output of `UPS CONFIG handler` to a new Function node, then connect that node's output to `write file`:

```javascript
// Adapt this path to your installation's persistent storage.
const oConfig = global.get("config") || {};
if (!oConfig.upsConfig) {
    node.error("UPS configuration is unavailable.", msg);
    return null;
}
msg.filename = "/data/ups_config.json";
msg.payload = JSON.stringify({ upsConfig: oConfig.upsConfig }, null, 2);
return msg;
```

Configure `write file` to take its filename from `msg.filename`, overwrite the entire file rather than append, and use UTF-8. The directory must exist and be writable by the Node-RED account. In a container, it must reside on a persistent volume. Do not connect the handler's original output directly to the file node: it would write the text `save`, not the configuration.

**Loading at startup:** Add a one-time Inject, a `read file` node for the same path with UTF-8 output, a JSON node converting text into an object, and this Function:

```javascript
const oLoaded = msg.payload;
const oUpsConfig = oLoaded && oLoaded.upsConfig;
if (!oUpsConfig || typeof oUpsConfig !== "object" ||
    Array.isArray(oUpsConfig) || typeof oUpsConfig.host !== "string" ||
    !oUpsConfig.host.trim()) {
    node.error("The file does not contain a valid upsConfig configuration.", msg);
    return null;
}
const oConfig = global.get("config") || {};
oConfig.upsConfig = oUpsConfig;
global.set("config", oConfig);
return null;
```

Run this loading sequence after any initialization of `global.config` in your project so that another initialization node does not overwrite it. Polling without a host simply waits; it starts working once settings are loaded. On first startup, the file may not exist yet — create the configuration through the form. Catch read, parse, and write errors using a Catch node and display them in Debug. After loading, refresh the configuration page if it was already open.

### 2B. Persistent default Node-RED context

An alternative is a default filesystem context store. Merge the following into your existing `settings.js`:

```javascript
contextStorage: {
    default: {
        module: "localfilesystem"
    }
}
```

After restarting, the unchanged `global.get/set` and `flow.get/set` calls will use this store. Separate file nodes are unnecessary; the external save link can remain disconnected.

**This change affects the instance's entire default context**, including live data, history, and other flows. Writes are normally deferred; the default interval is 30 seconds, so a sudden outage may lose recent changes. Merely adding a named filesystem store while retaining a default memory store is insufficient, because the module does not specify a store name in its `get/set` calls. See [Local Filesystem Context Store](https://nodered.org/docs/api/context/store/localfilesystem).

To separate persistent configuration from runtime data without changing the module, use option 2A. In LINEA, retain its own saving mechanism.

### 3. First startup

Enter the host and actual UPS name, run `Test NUT`, save the configuration, and watch the status panel. Without saving, the tested values do not change normal polling settings. Finally, verify restart behavior using the checklist at the end.

## NUT server and connection checks

The module connects to a NUT server, not directly to a USB device. The server must have a working UPS driver, allow network access, and expose the correct UPS name. Enter the name exactly as configured on the server, including letter case.

| Environment | Preparation |
| --- | --- |
| Synology DSM | For a USB-connected UPS, enable UPS support and the network UPS server. Allow the source address from which Node-RED connects in the permitted devices list. Do not confuse running a network UPS server with acting as a client of a remote UPS. Labels vary by DSM version; see [Synology UPS help](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/system_hardware_ups?version=7). |
| pfSense | Use the NUT package; settings are under `Services > UPS`. Set up local UPS operation and make the server reachable from Node-RED. See [Netgate documentation](https://docs.netgate.com/pfsense/en/latest/packages/nut.html). |
| OPNsense | Install the `os-nut` plugin through `System > Firmware > Plugins`. This topology requires a reachable NUT server; `netclient` mode is a client of a remote server. Verify the precise server setup for your installed version. The [official OPNsense guide](https://docs.opnsense.org/manual/how-tos/nut.html) primarily describes netclient. |
| Linux / another NUT server | Use the correct driver and configure `upsd` to listen on an address reachable from Node-RED. |

The supplied LINEA configuration uses `ups`, whereas the standalone form defaults to `UPS`. Neither value represents automatic name discovery.

If the `upsc` client is available, first list the UPS names, then read the values. Replace the address with your own:

```bash
upsc -l 192.168.1.100:3493
upsc ups@192.168.1.100:3493
```

Syntax and options are described in the [upsc manual](https://networkupstools.org/docs/man/upsc.html). Test from the same network environment as Node-RED; success from a PC does not necessarily confirm access from a container.

On Windows, TCP reachability can be checked separately:

```powershell
Test-NetConnection -ComputerName 192.168.1.100 -Port 3493
```

An open port does not confirm the correct UPS name or valid data. The flow uses plain TCP without login or TLS. Expose the server only to the necessary clients on a trusted network; the form has no settings for connections requiring authentication or TLS.

## Dashboard configuration

| Field | Key | Default / range | Meaning |
| --- | --- | --- | --- |
| NUT host | `host` | Empty outside LINEA configuration | IP address or DNS name, without `http://` or a path. |
| Port | `port` | `3493`; form range 1–65535 | NUT server TCP port. |
| UPS name | `upsName` | `UPS` | Exact UPS name on the server. |
| TCP timeout [ms] | `timeoutMs` | `4000`; form range 500–60000 | Socket inactivity timeout. This is not the polling interval. |
| ntfy URL | `ntfyUrl` | Empty | Full URL of a specific topic; leaving it empty disables sending. |

Example configuration section, not a complete LINEA configuration file:

```json
{
  "upsConfig": {
    "host": "192.168.1.100",
    "port": 3493,
    "upsName": "ups",
    "timeoutMs": 4000,
    "ntfyUrl": ""
  }
}
```

### Test NUT

The test uses the values currently entered in the form without saving them persistently. It returns the TCP/NUT query result to the form.

**A successful test also sends data into the normal parser.** It can overwrite runtime state, add a history point, and trigger a state notification. Testing another UPS is therefore not isolated from operational monitoring. A failed test also updates the shared communication state.

### Save configuration

The `Ulozit konfiguraci` button updates configuration in context and sends a request to the save handler. It does not automatically test the connection or wait for confirmation that the file was written. With a memory context store, the standalone export cannot retain settings across restarts unless persistence is added.

The module does not evaluate the `enabled` property found in LINEA's default configuration. Setting `enabled: false` does not stop polling. To pause polling, disable the scheduler; the manual test remains a separate way to issue a query.

## Status panel and history

### UPS Panel

The panel displays the main status in the priority order **low battery → on battery → mains OK**, additional flags, battery charge, and estimated remaining runtime. Depending on available data, it also shows W, VA, power headroom, rated power, input and output voltage, output frequency, battery type, and device identification.

Runtime is the estimate reported by the UPS during the last query, converted to minutes and seconds. The panel does not independently count it down between queries. The label `vypnuti pri … %` (shutdown at … %) uses `battery.charge.low`; the module does not enforce this threshold, and the value alone does not define the host system's complete shutdown policy.

Real and apparent power are read directly from NUT. The module does not calculate them from load percentage. Headroom is `max(powerNom - realpower, 0)` when both values are available. Missing values are generally displayed as a dash.

### GRAF_SVG_2osy

The chart displays power in W (orange), load in % (green), and input voltage in V (blue). Click a legend item to hide a series. Hovering with the mouse shows the time and values for the selected point. The chart width adapts to its container.

- The buffer holds **720 points**, approximately one hour at a regular five-second sampling interval.
- A point is added only when `realpower` is not `null`. Without this measurement, available voltage and load history are not recorded either.
- The left axis is fixed at **0–100 for both W and %**. Power values above 100 W may therefore be drawn outside the plotting area.
- The right axis is fixed at **200–250 V**. Voltages outside this range, including 0 V during an outage, are not reliably shown within the plotting area.
- Horizontal position follows sample order, not actual time intervals. Communication gaps do not appear as proportionally wide time gaps.
- The tooltip uses `mousemove` and `mouseleave`; touch interaction is not explicitly implemented.
- With the default memory context store, a restart clears history. The module has no history export or long-term archive.

## Events and ntfy

The parser compares the latest main status with the previous one stored in `flow.ups_prevStatus`. Priority is `LB`, then `OB`, then `OL`, otherwise `?`.

| Transition | Event | ntfy priority |
| --- | --- | --- |
| `OL` → `OB` or `LB` | `POWER_LOST` | `high` |
| `OB` or `LB` → `OL` | `POWER_RESTORED` | `default` |
| Another previously recorded main state → `LB`, unless the first row applies | `BATTERY_LOW` | `urgent` |

The first sample without a previous state does not generate a notification. Repeated samples of the same state do not generate one either. A direct `OL` → `LB` transition generates only `POWER_LOST`, not two events simultaneously. A change in `CHRG`, `DISCHRG`, `OVER`, `RB`, or `BYPASS` alone does not generate a separate alarm.

For ntfy, configure a URL such as `https://ntfy.sh/my-ups-topic` and subscribe to the same topic in your client. The flow sends text by POST with the `Title`, `Priority`, `Tags`, and `Content-Type` headers. It uses only `config.upsConfig.ntfyUrl`, not LINEA's shared `config.ntfyConfig.url`.

When the URL is empty, the switch blocks sending. No Dashboard toast is connected in this export, although the node comment mentions one. To display Dashboard notifications, add the appropriate notification node. There is no queue, delivery retry, or HTTP response check. The UI has no ntfy authentication token field; a protected topic requires authentication to be added to the sending branch.

## Data interface

### Context values and their lifetime

| Location | Meaning |
| --- | --- |
| `global.config.upsConfig` | Connection and ntfy settings. |
| `global.ups` | Last successfully parsed sample, including serial number and NUT connection details. |
| `global.upsNutCommState` | Result of the latest completed communication attempt and timestamp of the last success. |
| `global.lineaApiUpsState` | Reduced operational snapshot for further integration. |
| `flow.ups_history` | Array of up to 720 points: `{t, w, load, vin}`. |
| `flow.ups_prevStatus` | Previous main state for event detection. |
| Query node's local context | `lastPoll` limits the frequency of normal queries. |
| Watchdog's local context | `offline` prevents repeated emission of the same communication state. |

Lifetime depends on the default context store. A memory store keeps data in RAM; a filesystem store may also restore old runtime values. Always check their age after restarting.

### `global.ups` — actual field names

| Field | Content / source |
| --- | --- |
| `mfr`, `model`, `serial` | `device.*`, falling back to the corresponding `ups.*`; a missing model defaults to `UPS`. |
| `firmware` | `ups.firmware`. |
| `statusRaw`, `statusText`, `statusColor`, `extras` | Original status, derived description, color, and additional labels. |
| `online`, `onBattery`, `lowBatt` | The `OL`, `OB`, and `LB` flags. **`online` means mains power, not fresh data.** |
| `charging`, `discharging`, `overload`, `replaceBatt`, `bypass` | The `CHRG`, `DISCHRG`, `OVER`, `RB`, and `BYPASS` flags. |
| `charge`, `chargeLow` | `battery.charge` and `battery.charge.low`, in %. |
| `runtimeSec`, `runtimeMin`, `runtimeTxt` | `battery.runtime` in seconds, whole minutes, and an `m:ss` string. |
| `battType`, `battV` | Battery type and voltage. |
| `load` | `ups.load`, in %. |
| `realpower`, `powerNom` | `ups.realpower` and `ups.realpower.nominal`, in W. |
| `powerVA`, `powerNomVA` | `ups.power` and `ups.power.nominal`, in VA. |
| `headroomW` | Calculated non-negative real power headroom. |
| `inputV`, `outputV`, `outputVnom` | Input, output, and nominal output voltage. |
| `freq`, `freqNom` | Output and nominal output frequency, in Hz. |
| `beeper` | `ups.beeper.status`. |
| `ts` | Processing time in Node-RED, as a Unix timestamp in milliseconds. |
| `nutComm` | `{ok, lastOk, host, port, upsName}` as recorded when the sample was created. |

Missing numeric values are `null`; non-numeric text may become `NaN` through `parseFloat`. Consumers should therefore validate numbers. `global.ups` does not contain the `batteryCharge`, `batteryRuntimeMin`, `realPowerW`, or `lastUpdate` fields, or the `statusFlags` object shown in the older documentation.

### Checking freshness before use

The communication state contains `ok`, `lastAttempt`, and, when available, `lastOk`, plus `lastError`, `host`, `port`, and `upsName`. Timestamps are in milliseconds; `lastAttempt` is written when an attempt completes. `ok: false` means the last completed attempt failed, whereas the watchdog allows fifteen seconds since the last success.

This example uses the same tolerance and also checks the age of the parsed sample:

```javascript
const oUps = global.get("ups");
const oComm = global.get("upsNutCommState") || {};
const nNow = Date.now();
const nSampleAge = oUps ? nNow - Number(oUps.ts) : NaN;
const nCommAge = nNow - Number(oComm.lastOk || 0);
const bFresh = !!oUps && Number.isFinite(nSampleAge) &&
    nSampleAge >= 0 && nSampleAge <= 15000 &&
    Number.isFinite(nCommAge) && nCommAge >= 0 && nCommAge <= 15000;

if (!bFresh) {
    node.warn("Fresh UPS data is unavailable; power status cannot be confirmed.");
    return null;
}
if (oUps.onBattery) {
    node.warn("The UPS is running on battery.");
}
msg.payload = {
    onBattery: oUps.onBattery,
    charge: Number.isFinite(oUps.charge) ? oUps.charge : null,
    runtimeSec: Number.isFinite(oUps.runtimeSec) ? oUps.runtimeSec : null
};
return msg;
```

For a simple read in a Change node, choose the **global** source type and the path `ups.charge`. Reading a value alone does not verify its freshness.

### `global.lineaApiUpsState`

The snapshot omits the serial number, IP address, port, and configuration. “Read-only” describes its monitoring purpose; the context object is not technically protected against modification by another flow.

Illustrative structure with example values:

```json
{
  "updatedAt": "2026-09-19T17:00:00.000Z",
  "name": "UPS",
  "online": true,
  "onBattery": false,
  "status": {
    "raw": "OL",
    "lowBattery": false,
    "charging": false,
    "discharging": false,
    "overload": false,
    "replaceBattery": false,
    "bypass": false
  },
  "battery": {"chargePct": 100, "voltageV": 13.5, "runtimeSec": 1800},
  "input": {"voltageV": 230},
  "output": {"voltageV": 230, "frequencyHz": 50},
  "load": {"percent": 15, "realPowerW": 120}
}
```

The snapshot is created from received data, and the watchdog does not invalidate it during a communication failure. Evaluate its age using `updatedAt`; neither the presence of the object nor `online: true` confirms current availability.

## Known limitations of this export

This section describes the actual implementation, not fixes that have already been made.

1. **The watchdog is not connected to the panel or notifications.** After more than 15 seconds without a successful response, it displays `NUT OFFLINE` below the node in the editor. It does not update `global.ups`, the snapshot, or old panel data. Until the first success, it considers communication offline.
2. **Successful tests enter the normal processing path.** Testing another UPS can affect data and alarms; tests and normal queries are not mutually locked.
3. **The chart has fixed axes and requires a power measurement.** It should not be treated as a complete voltage outage record or a general-purpose power chart.
4. **Multiple UPS instances are not isolated.** Copies share global keys; copies on the same flow tab also share history and previous status. Multiple instances require separate namespaces and configuration.
5. **TCP availability is not full data validation.** The query node can accept a response containing a recognized block start or a `VAR` line that the parser subsequently rejects. An “OK” test does not necessarily mean a usable operational sample.
6. **Connection handling has gaps.** There is no protection against overlapping queries with long timeouts and no explicit socket cleanup when the node stops. A connection closing without data has no dedicated error-completion path. Timeout handling uses `socket.end()` rather than forcibly destroying the socket; it does not guarantee complete connection termination.
7. **Save and delivery feedback are not end-to-end confirmations.** The UI does not wait for a file write; the ntfy branch neither evaluates HTTP results nor retries attempts.
8. **The separate UI stale-data indicator is unfinished.** The panel contains a `stale` calculation with a two-second threshold but does not use it in the template. This unused calculation does not provide outage detection, and its threshold does not match normal five-second polling.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| `net is not defined` or a library error | The `net` → `net` mapping and permission to use Function node libraries. |
| `chybi NUT host` (missing NUT host) | The host field and actual contents of `global.config.upsConfig`. |
| `ECONNREFUSED`, timeout | NUT service, address, port, firewall, and the network path from Node-RED or its container. |
| `ERR UNKNOWN-UPS` | UPS name, including case; verify using `upsc -l`. |
| Another `ERR` response | Exact error text and NUT server diagnostics; the cause is not necessarily the UPS name. |
| Test works but normal queries fail | The test uses unsaved values. Save the configuration and check that another initialization node is not overwriting it. |
| UI reports saved, but settings disappear after restart | Save link, JSON serialization, permissions, persistent volume, and startup loading. |
| Panel stays green after communication is lost | It shows the last sample. Check the watchdog, `upsNutCommState.lastOk`, and `ups.ts`. |
| Panel works but chart is empty | Whether the UPS provides `ups.realpower`; without it, the buffer adds no point. |
| Power or voltage is not displayed correctly | Fixed chart ranges of 0–100 and 200–250. |
| No ntfy messages | UPS-specific URL, an actual state transition, the first-sample behavior, and the HTTP response using a temporarily connected Debug node. |
| Notifications ignore LINEA's central ntfy settings | The UPS uses its own `config.upsConfig.ntfyUrl`. |
| No Dashboard toast | The export has no connected toast; one must be added. |
| Two UPS devices overwrite each other's data | The module uses fixed global keys and is designed for one instance. |

## Post-installation checklist

- [ ] After Deploy, there are no unknown nodes or `net` library errors.
- [ ] The test returns data from the correct UPS, and the status panel subsequently receives new samples approximately every 5 seconds.
- [ ] Both `global.ups.ts` and `global.upsNutCommState.lastOk` are updated.
- [ ] Configuration survives a Node-RED restart.
- [ ] When communication is temporarily blocked, the watchdog switches to offline after more than 15 seconds; downstream logic rejects stale data.
- [ ] After communication recovers, the panel and snapshot update again.
- [ ] State transitions and ntfy are verified in a controlled environment; `Test NUT` alone does not verify ntfy delivery.
- [ ] Integrations use the actual field names and independently check data age.