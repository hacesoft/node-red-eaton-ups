[🇨🇿 **Česky**](README_CZ.md) | [🇬🇧 English](README.md)

---


# Module UPS Eaton / NUT pro Node-RED

Tento modul zajišťuje integraci zdroje nepřerušitelného napájení (UPS) Eaton (a jakýchkoli dalších záložních zdrojů podporujících protokol **NUT – Network UPS Tools**) do prostředí Node-RED.

Součástí modulu je:
* **Přímé dotazování na NUT server** přes TCP socket (využívá standardní Node.js modul `net`).
* **Detekce stavů a událostí** (výpadek napájení, obnovení sítě, nízký stav baterie, nabíjení atd.).
* **Odesílání notifikací** prostřednictvím služby **ntfy**.
* **Komplexní uživatelské rozhraní** (Dashboard v2 Panel & dvouosý SVG graf historie).
* **Konfigurační rozhraní** pro nastavení parametrů přímo z Dashboardu bez nutnosti úpravy kódu.
* **Ukládání konfigurace** do persistentního JSON souboru.
* **Poskytování provozních dat** do nadřazených systémů přes globální proměnné a LINEA / REST API.

---

## 🛠 Jak modul funguje a jak se napojuje na systém

1. **Plánování a komunikace (UPS Scheduler / NUT Request):**
   * Plánovač (`inject`) spouští dotaz na NUT server každých 5 sekund (při běžném provozu).
   * Pomocí vestavěného Node.js modulu `net` (který si uzel načítá přímo v nastavení uzlu bez nutnosti zásahu do `settings.js`) se otevře TCP spojení na definovaný Host a Port NUT serveru a odešle se příkaz `LIST VAR <upsName>`.
   * Přečte se celá odpověď a spojení se korektně ukončí.

2. **Zpracování dat (Parser UPS):**
   * Odpověď je rozpadnuta na jednotlivé proměnné (`VAR <upsName> <klic> "<hodnota>"`).
   * Probíhá detekce provozního stavu (příznaky `OL` = Síť OK, `OB` = Baterie, `LB` = Nízká baterie, `CHRG` = Nabíjení atd.).
   * Vypočítávají se provozní hodnoty (činný/zdánlivý výkon, výkonová rezerva, zbývající čas v minutách).
   * **Globální proměnné:**
     * `global.get("ups")` – kompletní zpracovaný objekt stavu UPS dostupný kdekoliv v Node-RED.
     * `global.get("lineaApiUpsState")` – čistý read-only snapshot určený pro externí API.
   * **Detekce změn (Change detection):** Při přechodu mezi stavy (`POWER_LOST`, `POWER_RESTORED`, `BATTERY_LOW`) se vygeneruje událost pro notifikace.

3. **Notifikace (ntfy):**
   * Při vzniku události se připraví textová zpráva a priority.
   * Pokud je v konfiguraci zadané `ntfyUrl`, odesílá se HTTP POST požadavek na daný ntfy server/topic.

4. **Watchdog komunikace:**
   * Node-RED kontroluje věk poslední úspěšné odpovědi z NUT serveru. Pokud nepřijde odpověď déle než 15 sekund, stav se přepne na `NUT OFFLINE`.

---

## 💾 Co a jak se ukládá (Persistenca a Stav)

Modul rozlišuje dva typy uložení dat: **konfigurační stav na disk** a **provozní stav v paměti (RAM)**.

### 1. Konfigurace (`config.upsConfig`)
Při změně nastavení v Dashboard UI (Host, Port, Název UPS, Timeout, ntfy URL) a stisku tlačítka **Uložit konfiguraci**:
1. Uzel `UPS CONFIG handler` aktualizuje živé hodnoty v paměti: `global.set("config.upsConfig", ...)`
2. Výstupní signál projde přes uzel `link out` (`UPS CONFIG -> SAVE config_hacesoft.json`).
3. Sběrné flow pro zápis konfigurace (ve výchozím stavu v systému LINEA) sloučí aktuální globální objekt `config` a uloží jej do souboru na disk (např. `/data/config_hacesoft.json` nebo do souboru určeného vaší aplikací).
4. Při startu Node-RED se tento JSON načte z disku a vloží zpět do `global.config`, odkud si ho modul UPS při inicializaci zkontoluje a použije.

**Struktura ukládané konfigurace (JSON):**
```json
{
  "upsConfig": {
    "host": "192.168.1.100",
    "port": 3493,
    "upsName": "eaton",
    "timeoutMs": 4000,
    "ntfyUrl": "https://ntfy.sh/moje-ups-notifikace"
  }
}
```

### 2. Provozní stav v paměti (`global.ups`)
V paměti RAM se po každém cyklu vyhodnocení (každých 5 s) udržuje aktuální objekt `global.get("ups")`. Tento objekt obsahuje živá data:

```javascript
{
  "online": true,
  "statusText": "Síť OK",
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

## ⚙️ Nastavení zdroje NUT (Synology, pfSense, OPNsense)

Aby se Node-RED mohl k UPS připojit, musí zařízení, ke kterému je UPS připojena USB kabelem, fungovat jako **NUT Server** a mít povolený přístup ze sítě (standardně port `3493`).

### 1. Synology NAS
Pokud je UPS připojena přes USB k Synology NAS:
1. Otevřete **Ovládací panel $\rightarrow$ Napájení $\rightarrow$ UPS**.
2. Zaškrtněte **Povolit podporu UPS**.
3. Jako typ UPS vyberte **Synology UPS server**.
4. Klikněte na tlačítko **Povolená zařízení UPS** (Povolené IP adresy).
5. Přidejte **IP adresu zařízení, kde běží váš Node-RED**.
6. **Důležité parametry pro Node-RED:**
   * **Host:** IP adresa Synology NAS
   * **Port:** `3493`
   * **UPS name:** `ups` (Synology interně používá název `ups`)

---

### 2. Firewall pfSense
1. **Instalace:** Jděte do **System $\rightarrow$ Package Manager $\rightarrow$ Available Packages**, vyhledejte a nainstalujte balíček **nut**.
2. **Konfigurace:** Jděte do **Services $\rightarrow$ UPS / NUT**.
   * **UPS Mode:** `Local USB` (připojení přes USB k pfSense).
   * **UPS Name:** např. `eaton` nebo `ups`.
   * **Driver:** Vyberte ovladač pro vaši UPS (např. `usbhid-ups`).
3. **Povolení přístupu ze sítě:**
   V záložce **Advanced settings** (do pole pro konfiguraci `upsd.conf`) přidejte:
   ```text
   LISTEN 0.0.0.0 3493
   ```
4. **Firewall:** V **Firewall $\rightarrow$ Rules** povolte TCP provoz na port `3493` z IP adresy Node-REDu.

---

### 3. Firewall OPNsense
1. **Instalace:** Jděte do **System $\rightarrow$ Firmware $\rightarrow$ Plugins**, vyhledejte a nainstalujte plugin **os-nut**.
2. **Konfigurace:** Jděte do **Services $\rightarrow$ Nut $\rightarrow$ Configuration**.
   * Na záložce **General Settings** zaškrtněte **Enable Nut**.
   * Na záložce **USB Service**: zaškrtněte **Enable**, zadejte **Name** (`eaton` nebo `ups`) a vyberte driver (`usbhid-ups`).
3. **Síťový server (UPS Network Service):**
   * Zaškrtněte **Enable**.
   * **Listen Address:** `0.0.0.0`
   * **Port:** `3493`
4. **Firewall:** V **Firewall $\rightarrow$ Rules** přidejte pravidlo povolující **TCP** port **3493** z IP adresy Node-RED.

---

## 🚀 Jak integrovat modul do vlastního / jiného Node-RED projektu (Mimo LINEA)

Tento modul byl navržen tak, aby jej bylo možné snadno vyjmout z projektu LINEA a použít v jakékoli jiné instanci Node-REDu.

### Krok 1: Import a závislosti
1. V Node-RED zvolte **Import** a vložte JSON kód tohoto modulu.
2. Ujistěte se, že máte nainstalován **Dashboard v2** (`@flowfuse/node-red-dashboard`).

### Krok 2: Přiřazení do UI skupin
1. Otevřete pravý panel v Node-RED (záložka **Dashboard**).
2. Vyhledejte uzly:
   * `UI CONFIG` (Konfigurační karta)
   * `UPS Panel` (Hlavní provozní karta)
   * `GRAF_SVG_2osy` (Graf historie)
3. Přiřaďte tyto uzly do vašich vytvořených skupin (`ui-group`) a stránek (`ui-page`).

### Krok 3: Řešení ukládání konfigurace (Persistence)
Pokud ve svém projektu **nepoužíváte** standardní souborový ukládač systému LINEA (`SAVE config_hacesoft.json`):

* **Možnost A (Jednoduchá – Uložení do souboru):**
  Zrušte spojení z uzel `link out` (`UPS CONFIG -> SAVE config_hacesoft.json`) a napojte výstup z `UPS CONFIG handler` přímo na standardní uzel `write file`, který zapíše obsah do vámi zvoleného JSON souboru.
* **Možnost B (Uložení do Flow/Global Context):**
  V nastavení Node-REDu (`settings.js`) můžete povolit persistentní kontext v RAM/souboru (`contextStorage: { default: { module: 'localfilesystem' } }`). V takovém případě se data uložená do `global.set("config.upsConfig", ...)` zachovají automaticky i po restartu a nemusíte řešit žádné externí ukládání na disk.

### Krok 4: Využití dat v ostatních částech vašich flow
Jakmile modul běží, můžete z jakéhokoli jiného uzlu/flow přistupovat k datům UPS:
* **V uzlu `Function`:**
  ```javascript
  const ups = global.get("ups");
  if (ups && !ups.online) {
      node.warn("UPS je offline nebo komunikace selhala!");
  } else if (ups && ups.statusFlags.OB) {
      node.warn(`Běžíme na baterii! Zbývá ${ups.batteryRuntimeMin} minut.`);
  }
  ```
* **V uzlu `Change`:**
  Zvolte *Set* `msg.payload` *to* `global.ups.batteryCharge`.

---

## 🔧 Popis Konfiguračního UI (`UPS CONFIG`)

Konfigurační karta v Dashboardu umožňuje nastavovat parametry spojení bez nutnosti úpravy zdrojového kódu.

### Nastavitelné položky:
| Pole | Typ | Popis / Příklad |
| :--- | :--- | :--- |
| **NUT host** | Text | IP adresa nebo hostname serveru, kde běží NUT (např. Synology, pfSense, OPNsense nebo Linux server). |
| **Port** | Číslo | Port NUT serveru (výchozí standardní port je `3493`). |
| **UPS name** | Text | Název UPS zadaný v konfiguraci NUT serveru (Synology = `ups`, pfSense/OPNsense = např. `eaton` nebo podle vašeho nastavení). |
| **TCP timeout [ms]** | Číslo | Časový limit pro odpověď v milisekundách (výchozí: `4000` ms, min. 500 ms). |
| **ntfy URL** | Text *(volitelné)* | Adresa pro zasílání push notifikací (např. `https://ntfy.sh/moje-ups-alarmy`). Pokud je pole prázdné, notifikace přes ntfy se neodesílají. |

### Tlačítka a stav testu:
* **Test NUT:** Vykoná jednorázový testovací dotaz na zadanou kombinaci Host + Port + Název UPS. Výsledek (úspěch/chyba) se okamžitě zobrazí vedle tlačítka bez nutnosti konfiguraci ukládat.
* **Uložit konfiguraci:** Uloží zadané parametry do globálního stavu `config.upsConfig` a vyvolá uložení do konfiguračního souboru. Modul začne ihned používat nové údaje.

---

## 📊 Popis Provozního UI (Dashboard)

Uživatelské rozhraní se skládá ze dvou hlavních vizuálních bloků:

### 1. UPS Panel (`UPS Panel`)
Zobrazuje aktuální stav záložního zdroje v reálném čase:

* **Stavový banner (nahoře):**
  * Zobrazuje aktuální režim napájení: `Síť OK` (zelená), `BĚŽÍ NA BATERII` (oranžová, bliká), `NÍZKÁ BATERIE` (červená, bliká).
  * Zobrazuje doplňkové vlastnosti (např. *nabíjí, vybíjí, přetížení, vyměnit baterii, bypass*).
* **Baterie a výdrž:**
  * Grafické znázornění kapacity baterie (v %) s barevným odlišením podle nabití.
  * Velký odpočet zbývající doby provozu na baterii (v minutách a sekundách).
  * Informace o hranici vypnutí (např. *vypnutí při 20 %*).
* **Výkon a Zátěž:**
  * Ukazatel vytížení UPS v % (zelená $\rightarrow$ oranžová $\rightarrow$ červená při $>80\%$).
  * Činný příkon (**W**), zdánlivý výkon (**VA**), dostupná rezerva (**W**) a nominální maximum (**W**).
* **Síť a Baterie:**
  * Vstupní napětí (**V**), výstupní napětí (**V**), frekvence sítě (**Hz**) a typ baterie.
* **Patička:**
  * Výrobce, model UPS a verze firmwaru.

### 2. Graf historie (`GRAF_SVG_2osy`)
Integrovaný dynamický SVG graf pro sledování průběhu v čase:

* **Měřené veličiny:**
  * 🟠 **Příkon (W)** – činný odběr připojené zátěže.
  * 🟢 **Zátěž (%)** – procentuální vytížení zdroje.
  * 🔵 **Napětí (V)** – stupnice na pravé ose pro kontrolu kvality/výpadků vstupní sítě (200–250 V).
* **Funkce grafu:**
  * **Interaktivní legenda:** Kliknutím na příslušnou řadu v legendě lze danou křivku skrýt nebo znovu zobrazit.
  * **Tooltip (Cursor):** Při najetí myší/prstem nad graf se zobrazí vertikální osa a přesné naměřené hodnoty v daném čase.
  * **Kapacita:** Uchovává historii až pro 720 bodů.