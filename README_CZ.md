[🇨🇿 **Česky**](README_CZ.md) | [🇬🇧 English](README.md) | [Linea project](https://github.com/hacesoft/Linea)

# UPS Eaton / NUT pro Node-RED

Modul pro sledování UPS přes **NUT (Network UPS Tools)**, vytvořený jako součást projektu **LINEA / GridSight**. Zobrazuje stav napájení, baterie a zátěže v Dashboardu 2.0, uchovává krátkou historii a při vybraných změnách stavu odesílá upozornění přes ntfy.

Lze jej použít také v samostatném Node-RED projektu. V takovém případě je potřeba přiřadit Dashboard a zajistit ukládání i načítání konfigurace podle tohoto návodu.

<img width="405" height="640" alt="Ukázka panelu UPS" src="https://github.com/user-attachments/assets/dedcc061-7939-4db8-b521-5785e930d694" />

## Obsah

- [Co modul umí](#co-modul-umí)
- [Požadavky](#požadavky)
- [Jak modul pracuje](#jak-modul-pracuje)
- [Instalace v LINEA](#instalace-v-linea)
- [Samostatné použití](#samostatné-použití)
- [NUT server a ověření spojení](#nut-server-a-ověření-spojení)
- [Nastavení v Dashboardu](#nastavení-v-dashboardu)
- [Provozní panel a historie](#provozní-panel-a-historie)
- [Události a ntfy](#události-a-ntfy)
- [Datové rozhraní](#datové-rozhraní)
- [Známá omezení tohoto exportu](#známá-omezení-tohoto-exportu)
- [Řešení problémů](#řešení-problémů)
- [Kontrola po instalaci](#kontrola-po-instalaci)

## Co modul umí

- Čte NUT proměnné přes TCP příkazem `LIST VAR <upsName>`.
- Rozlišuje napájení ze sítě, provoz na baterii, nízkou baterii a další stavové příznaky.
- Zobrazuje dostupné hodnoty kapacity, výdrže, výkonu, zátěže, napětí a výstupní frekvence.
- Nabízí konfigurační formulář a jednorázový test spojení.
- Vytváří události `POWER_LOST`, `POWER_RESTORED` a `BATTERY_LOW`.
- Odesílá volitelné HTTP notifikace na nastavenou ntfy URL.
- Poskytuje data ostatním flow přes globální kontext a připravuje provozní snapshot pro API LINEA.

**Modul je monitorovací.** Neposílá UPS povely k vypnutí, neřídí zásuvky, nemění její nastavení a neprovádí automatické vypnutí NAS nebo počítače. Ochranné vypínání musí zajišťovat příslušný NUT klient nebo hostitelský systém.

Není omezen pouze na Eaton. Jiná UPS je použitelná, pokud její NUT server poskytuje čitelné proměnné v očekávaném formátu. Rozsah údajů není u všech modelů stejný; kompatibilita konkrétního modelu se musí ověřit.

## Požadavky

| Součást | Požadavek |
| --- | --- |
| Node-RED | Prostředí s podporou použitých Function uzlů a Dashboardu 2.0; minimální ověřená verze Node-RED není v exportu uvedena. |
| Dashboard | Balíček `@flowfuse/node-red-dashboard`; export uvádí verzi `1.30.2`. Původní `node-red-dashboard` není náhradou za Dashboard 2.0. |
| TCP knihovna | V uzlu `UPS NUT request (CONFIG)` je uvedeno přiřazení proměnné `net` k vestavěnému Node.js modulu `net`. |
| NUT server | Dostupný ze síťového prostředí, kde skutečně běží Node-RED; obvykle TCP port `3493`. |
| Konfigurace | Objekt `global.config.upsConfig`, případně vytvořený uložením formuláře. |
| Trvalé uložení | Ukládací mechanismus LINEA nebo jedna ze samostatných variant níže. |
| ntfy | Volitelné; pro samotné měření a Dashboard není potřebné. |

Pro seznam knihoven ve Function uzlu musí prostředí umožňovat `functionExternalModules`. Pokud tuto možnost instalace nemá zapnutou, doplňte do stávajícího objektu v `settings.js` položku `functionExternalModules: true` a restartujte Node-RED. Existující nastavení zachovejte. Není tedy správné obecně slibovat provoz bez jakéhokoli zásahu do `settings.js`. Viz [dokumentace Function uzlů](https://nodered.org/docs/user-guide/writing-functions#using-the-functionexternalmodules-option).

Samotný modul nepotřebuje Modbus, MQTT, Victron ani připojení k Nextcloudu. NUT dotazy realizuje přímo, bez dalšího specializovaného NUT uzlu.

## Jak modul pracuje

| Uzel | Úloha a propojení |
| --- | --- |
| `UPS scheduler 1s / NUT 5s` | Po nasazení odešle první impuls po 2 s, dále každou 1 s. Napájí dotazovací uzel i watchdog. |
| `UPS NUT request (CONFIG)` | Čte konfiguraci, běžné dotazy omezuje na jeden za nejméně 5 s, otevře TCP spojení a načte NUT odpověď. Výstup 1 vede do parseru a vypnutého debug uzlu; výstup 2 vrací výsledek testu do konfigurace. |
| `Parser UPS` | Zpracuje řádky `VAR`, aktualizuje globální data. Výstup 1 vede do panelu a historie; výstup 2 nese případnou událost. |
| `Buffer historie` | Uchovává nejvýše 720 bodů ve flow kontextu a posílá celé pole do grafu. |
| `Text notifikace` | Z události sestaví text a případné HTTP hlavičky. |
| `ntfy URL nastavena?` | Propustí zprávu k HTTP uzlu pouze při nastavené ntfy URL. |
| `ntfy POST (UPS config)` | Odešle HTTP požadavek. Odpověď nemá v exportu dalšího příjemce. |
| `UPS CONFIG handler` | Výstup 1: odpověď formuláři; výstup 2: požadavek na uložení; výstup 3: test NUT. |
| `NUT watchdog` | Každou sekundu vyhodnotí stáří poslední úspěšné komunikace. Zprávu odešle jen při změně stavu; jeho výstup není v exportu připojen. |

Běžný cyklus načítá konfiguraci znovu při každém impulsu. Nové uložené hodnoty se proto použijí při následujícím povoleném dotazu bez nového Deploy. Test obchází pětisekundové omezení, ale současně posune čas posledního dotazu.

## Instalace v LINEA

1. Před importem si exportujte zálohu současných flow.
2. Ověřte, zda skupina `MODULE::UPS::EATON` již existuje. V dodaném celkovém projektu už obsažena je, takže není potřeba přidávat další kopii.
3. Při aktualizaci nahraďte odpovídající modul a zkontrolujte připojení, místo spuštění dvou kopií současně.
4. Zachovejte Dashboard skupiny nebo přiřaďte všechny tři šablony do existujícího Dashboardu: `UPS CONFIG`, `UPS Panel`, `GRAF_SVG_2osy`.
5. Zkontrolujte externí propojení `UPS CONFIG -> SAVE config_hacesoft.json` na `SAVE_NEW_CONFIG`. Cílové ID v referenčním exportu je `3c18ac53a47a79d8`; při importu se ID mohou změnit.
6. Ověřte inicializaci a načtení společného `global.config`, potom vyplňte UPS konfiguraci, spusťte test a uložte ji.

### Ukládání v LINEA

Handler využije `global.get("setConfigProperty")`, pokud tato funkce existuje. Jinak upraví `upsConfig` v objektu `global.config` přímo. Na druhém výstupu odešle pouze:

```json
{"topic":"save","payload":"save"}
```

Společný uzel `Create/Save File` přečte skutečný objekt `global.config`, převede jej na JSON a předá do souborového uzlu. V dodané LINEA je výchozí adresář nastaven na `/data/config_FLOW/` a hlavní soubor se jmenuje `config_hacesoft.json`; výsledná cesta závisí na inicializovaném `sValid_Patch` a `filename`.

Tento společný ukládač při běžném požadavku vytváří také výstupy pro konfigurace Shelly a Daikin. Jde tedy o společnou ukládací větev projektu, nikoli izolovaný zapisovač UPS.

Formulář zobrazí potvrzení už po změně kontextu a odeslání požadavku. **Toto potvrzení samo neprokazuje úspěšný zápis na disk.** Ověřte soubor a zachování nastavení po restartu.

### Vazba na API a GridSight

Parser zapisuje `global.lineaApiUpsState`. V dodané LINEA jej čte `Build status API 1.0.0` a v odpovědi endpointu `GET /api/v1/status` vkládá do `ups.data`; vedle něj používá příznak `ups.available`.

Samostatný UPS export neobsahuje žádný `http in` ani `http response`. Samotné vytvoření snapshotu proto nezpřístupní REST API. Externí klient, například GridSight, musí využívat API hostitelského projektu. Archivace v Nextcloudu ani její intervaly nejsou implementovány v tomto modulu a nelze je z těchto souborů potvrdit.

## Samostatné použití

### 1. Import a Dashboard

1. Nainstalujte Dashboard 2.0 a zkontrolujte zpřístupnění `net` ve Function uzlu.
2. Importujte `eaton_flows_19092026_1854.json` do zvolené karty flow. Export je výběrem skupiny a konfigurací, neobsahuje vlastní uzel `tab`.
3. Zkontrolujte importované Dashboard konfigurace. Export zahrnuje základ `/dashboard`, stránky `/FVE` a `/config`, téma a dvě skupiny. Ve stávajícím projektu je můžete nahradit vlastními přiřazeními.
4. `UPS Panel` a graf přiřaďte provozní skupině, `UPS CONFIG` konfigurační skupině. Zkontrolujte šířku a pořadí widgetů.
5. Odpojte nepoužitelné externí propojení na ukládač LINEA a zvolte jednu variantu persistence níže.

Pro běžné samostatné použití stačí fallback handleru bez `setConfigProperty`. Pokud už jiný projekt používá tuto globální funkci, musí mít kompatibilní význam: nastavení hodnoty podle cesty `upsConfig.<položka>`.

### 2A. Uložení do samostatného JSON souboru

Tato varianta ponechává provozní kontext v paměti a ukládá jen nastavení UPS. Následující doplňkové uzly nejsou součástí dodaného exportu.

**Zápis:** Druhý výstup `UPS CONFIG handler` připojte do nového Function uzlu, teprve jeho výstup do `write file`:

```javascript
// Cestu přizpůsobte trvalému úložišti své instalace.
const oConfig = global.get("config") || {};
if (!oConfig.upsConfig) {
    node.error("Konfigurace UPS není dostupná.", msg);
    return null;
}
msg.filename = "/data/ups_config.json";
msg.payload = JSON.stringify({ upsConfig: oConfig.upsConfig }, null, 2);
return msg;
```

V `write file` nastavte název souboru z `msg.filename`, přepsání celého souboru místo připojování a zápis UTF-8. Adresář musí existovat a být zapisovatelný účtem Node-RED. V kontejneru musí být umístěn na trvalém svazku. Původní výstup handleru nepřipojujte přímo: zapsal by text `save`, nikoli konfiguraci.

**Načtení po startu:** Přidejte jednorázový Inject, `read file` pro stejnou cestu s výstupem UTF-8, JSON uzel převádějící text na objekt a Function:

```javascript
const oLoaded = msg.payload;
const oUpsConfig = oLoaded && oLoaded.upsConfig;
if (!oUpsConfig || typeof oUpsConfig !== "object" ||
    Array.isArray(oUpsConfig) || typeof oUpsConfig.host !== "string" ||
    !oUpsConfig.host.trim()) {
    node.error("Soubor neobsahuje platnou konfiguraci upsConfig.", msg);
    return null;
}
const oConfig = global.get("config") || {};
oConfig.upsConfig = oUpsConfig;
global.set("config", oConfig);
return null;
```

Načtení spusťte po případné inicializaci `global.config` ve svém projektu, aby jej další inicializační uzel nepřepsal. Polling bez hostu pouze čeká; po načtení nastavení začne fungovat. Při prvním spuštění soubor ještě nemusí existovat — konfiguraci vytvoříte přes formulář. Chyby čtení, parsování a zápisu zachyťte pomocí Catch a zobrazte v Debug. Po načtení obnovte konfigurační stránku, pokud již byla otevřena.

### 2B. Persistentní výchozí kontext Node-RED

Alternativou je výchozí souborový kontext. Do stávajícího `settings.js` začleňte:

```javascript
contextStorage: {
    default: {
        module: "localfilesystem"
    }
}
```

Po restartu budou nezměněné `global.get/set` i `flow.get/set` používat toto úložiště. Samostatné souborové uzly nejsou potřeba; externí ukládací link může zůstat odpojený.

**Tato změna se týká celého výchozího kontextu instance**, tedy i živých dat, historie a dalších flow. Zápisy jsou standardně odkládané; výchozí interval je 30 s, takže náhlý výpadek může ztratit poslední změny. Pouhé přidání pojmenovaného souborového úložiště při zachování výchozího paměťového nestačí, protože modul název úložiště v `get/set` neuvádí. Viz [Local Filesystem Context Store](https://nodered.org/docs/api/context/store/localfilesystem).

Pro oddělení trvalé konfigurace od provozních dat bez zásahu do modulu použijte variantu 2A. V LINEA zachovejte její vlastní mechanismus ukládání.

### 3. První spuštění

Vyplňte host a skutečný název UPS, spusťte `Test NUT`, uložte konfiguraci a sledujte provozní panel. Bez uložení testované hodnoty nezmění nastavení běžného pollingu. Nakonec ověřte restart podle kontrolního seznamu na konci.

## NUT server a ověření spojení

Modul se připojuje k NUT serveru, nikoli přímo k USB zařízení. Server musí mít funkční ovladač UPS, povolený síťový přístup a správný název UPS. Jméno zadejte přesně podle serveru, včetně velikosti písmen.

| Prostředí | Co připravit |
| --- | --- |
| Synology DSM | Při USB připojení zapněte podporu UPS a síťový UPS server. V povolených zařízeních povolte zdrojovou adresu, ze které přichází Node-RED. Nezaměňujte vlastní síťový server za režim klienta vzdálené UPS. Názvy položek se liší podle DSM; viz [nápověda Synology UPS](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/system_hardware_ups?version=7). |
| pfSense | Použijte balíček NUT; nastavení je v `Services > UPS`. Zajistěte lokální obsluhu UPS a dosažitelnost serveru z Node-RED. Viz [dokumentace Netgate](https://docs.netgate.com/pfsense/en/latest/packages/nut.html). |
| OPNsense | Plugin `os-nut` se instaluje přes `System > Firmware > Plugins`. Pro tuto topologii potřebujete dostupný NUT server; režim `netclient` je klient vzdáleného serveru. Přesné nastavení serverové části ověřte podle instalované verze. [Oficiální návod OPNsense](https://docs.opnsense.org/manual/how-tos/nut.html) popisuje zejména netclient. |
| Linux / jiný NUT server | Použijte správný ovladač a nakonfigurujte naslouchání `upsd` na adrese dostupné pro Node-RED. |

V dodané konfiguraci LINEA je název `ups`, zatímco samostatný formulář má fallback `UPS`. Ani jednu variantu nepovažujte za automatické zjištění názvu.

Pokud máte k dispozici klienta `upsc`, můžete nejprve vypsat jména UPS a potom načíst hodnoty; adresu nahraďte vlastní:

```bash
upsc -l 192.168.1.100:3493
upsc ups@192.168.1.100:3493
```

Syntaxe a význam přepínačů jsou v [manuálu upsc](https://networkupstools.org/docs/man/upsc.html). Test proveďte ze stejného síťového prostředí jako Node-RED; úspěch z PC nemusí potvrzovat dostupnost z kontejneru.

Z Windows lze samostatně ověřit TCP dostupnost:

```powershell
Test-NetConnection -ComputerName 192.168.1.100 -Port 3493
```

Otevřený port ještě nepotvrzuje správný název UPS ani platná data. Flow používá prostý TCP dotaz bez přihlášení a bez TLS. Zpřístupněte server jen potřebným klientům v důvěryhodné síti; pro připojení vyžadující autentizaci nebo TLS nemá formulář nastavení.

## Nastavení v Dashboardu

| Pole | Klíč | Výchozí hodnota / rozsah | Význam |
| --- | --- | --- | --- |
| NUT host | `host` | Prázdný mimo konfiguraci LINEA | IP adresa nebo DNS jméno, bez `http://` a bez cesty. |
| Port | `port` | `3493`; formulář 1–65535 | TCP port NUT serveru. |
| UPS name | `upsName` | `UPS` | Přesný název UPS na serveru. |
| TCP timeout [ms] | `timeoutMs` | `4000`; formulář 500–60000 | Timeout nečinnosti socketu. Není to interval pollingu. |
| ntfy URL | `ntfyUrl` | Prázdná | Úplná URL konkrétního topicu; prázdná vypíná odesílání. |

Příklad části konfigurace, nikoli celý konfigurační soubor LINEA:

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

Test použije právě vyplněné hodnoty bez jejich trvalého uložení. Vrátí výsledek TCP/NUT dotazu do formuláře.

**Úspěšný test současně posílá data do běžného parseru.** Může přepsat provozní stav, přidat bod do historie a vyvolat stavovou notifikaci. Při testování jiné UPS tedy není oddělen od provozního monitoringu. Také neúspěšný test aktualizuje společný stav komunikace.

### Uložit konfiguraci

Upraví konfiguraci v kontextu a vyšle požadavek ukládači. Tlačítko neprovádí automatický test spojení a nečeká na potvrzení zápisu souboru. Samostatný export bez doplněné persistence nastavení po restartu v paměťovém kontextu neuchová.

`enabled`, které se objevuje ve výchozí konfiguraci LINEA, modul nevyhodnocuje. Nastavení `enabled: false` polling nevypne. Pro jeho pozastavení deaktivujte plánovač; ruční test zůstává samostatnou cestou k dotazu.

## Provozní panel a historie

### UPS Panel

Panel zobrazuje hlavní stav s prioritou **nízká baterie → provoz na baterii → síť OK**, doplňkové příznaky, nabití a odhad zbývající výdrže. Dále podle dostupnosti dat ukazuje W, VA, výkonovou rezervu, jmenovitý výkon, vstupní a výstupní napětí, výstupní frekvenci, typ baterie a identifikaci zařízení.

Hodnota výdrže je odhad hlášený UPS při posledním dotazu, převedený na minuty a sekundy. Panel ji mezi dotazy samostatně neodpočítává. Text `vypnuti pri … %` používá `battery.charge.low`; modul tuto hranici nevynucuje a samotná hodnota neurčuje úplnou vypínací politiku hostitelského systému.

Činný a zdánlivý výkon se přímo čtou z NUT. Modul je nedopočítává z procent zátěže. Rezerva je `max(powerNom - realpower, 0)`, pokud jsou obě hodnoty dostupné. Chybějící hodnoty panel zpravidla zobrazí jako pomlčku.

### GRAF_SVG_2osy

Graf zobrazuje příkon ve W (oranžová), zátěž v % (zelená) a vstupní napětí ve V (modrá). Kliknutím na legendu lze řadu skrýt. Najetí myší zobrazí čas a hodnoty vybraného bodu. Šířka se přizpůsobuje kontejneru.

- Buffer pojme **720 bodů**, při pravidelném pětisekundovém vzorkování přibližně hodinu.
- Bod se přidá pouze tehdy, pokud `realpower` není `null`. Bez této veličiny se neukládá ani historie dostupného napětí či zátěže.
- Levá osa je pevná **0–100 pro W i %**. Výkony nad 100 W proto mohou být vykreslené mimo plochu grafu.
- Pravá osa je pevná **200–250 V**. Napětí mimo tento rozsah, včetně 0 V při výpadku, není spolehlivě zobrazeno v ploše grafu.
- Vodorovná poloha odpovídá pořadí vzorků, nikoli jejich skutečným časovým rozestupům. Výpadky komunikace se nepromění v úměrně širokou časovou mezeru.
- Tooltip používá `mousemove` a `mouseleave`; dotykové ovládání není explicitně implementováno.
- Při výchozím paměťovém kontextu restart historii smaže. Modul nemá export ani dlouhodobý archiv historie.

## Události a ntfy

Parser porovnává poslední hlavní stav uložený jako `flow.ups_prevStatus`. Vyšší prioritu má `LB`, potom `OB`, potom `OL`, jinak `?`.

| Přechod | Událost | ntfy priorita |
| --- | --- | --- |
| `OL` → `OB` nebo `LB` | `POWER_LOST` | `high` |
| `OB` nebo `LB` → `OL` | `POWER_RESTORED` | `default` |
| Jiný již známý hlavní stav → `LB`, pokud neplatí první řádek | `BATTERY_LOW` | `urgent` |

První vzorek bez předchozího stavu nevytvoří notifikaci. Opakované vzorky stejného stavu ji také nevytvářejí. Přímý přechod `OL` → `LB` vytvoří pouze `POWER_LOST`, ne současně dvě události. Změna samotného `CHRG`, `DISCHRG`, `OVER`, `RB` nebo `BYPASS` nezakládá samostatný alarm.

Pro ntfy nastavte například `https://ntfy.sh/moje-ups-topic` a odebírejte stejný topic v klientovi. Flow odešle text přes POST s hlavičkami `Title`, `Priority`, `Tags` a `Content-Type`. Používá výhradně `config.upsConfig.ntfyUrl`, nikoli společnou `config.ntfyConfig.url` z LINEA.

Při prázdné URL switch odeslání zablokuje. V exportu není připojen Dashboard toast, přestože jej komentář uzlu zmiňuje. Chcete-li upozornění i v Dashboardu, je nutné doplnit příslušný notifikační uzel. Není implementována fronta, opakování neúspěšného doručení ani kontrola HTTP odpovědi. UI nemá pole pro přihlašovací token ntfy; chráněný topic vyžaduje doplnění autentizace v odesílací větvi.

## Datové rozhraní

### Kontexty a jejich životnost

| Umístění | Význam |
| --- | --- |
| `global.config.upsConfig` | Nastavení spojení a ntfy. |
| `global.ups` | Poslední úspěšně zpracovaný vzorek; včetně sériového čísla a údajů o NUT spojení. |
| `global.upsNutCommState` | Výsledek posledního dokončeného komunikačního pokusu a čas posledního úspěchu. |
| `global.lineaApiUpsState` | Redukovaný provozní snapshot pro další integraci. |
| `flow.ups_history` | Pole nejvýše 720 bodů `{t, w, load, vin}`. |
| `flow.ups_prevStatus` | Předchozí hlavní stav pro detekci událostí. |
| Lokální kontext dotazovacího uzlu | `lastPoll` omezuje četnost běžných dotazů. |
| Lokální kontext watchdogu | `offline` zabraňuje opakovanému odesílání stejného komunikačního stavu. |

Životnost závisí na výchozím context store. Při paměťovém úložišti jde o data v RAM, při souborovém se mohou obnovit také staré provozní hodnoty. Po restartu vždy posuzujte jejich stáří.

### `global.ups` — skutečné názvy polí

| Pole | Obsah / zdroj |
| --- | --- |
| `mfr`, `model`, `serial` | `device.*`, případně odpovídající `ups.*`; chybějící model má fallback `UPS`. |
| `firmware` | `ups.firmware`. |
| `statusRaw`, `statusText`, `statusColor`, `extras` | Původní stav, odvozený popis, barva a doplňkové texty. |
| `online`, `onBattery`, `lowBatt` | Příznaky `OL`, `OB`, `LB`. **`online` znamená síťové napájení, nikoli čerstvá data.** |
| `charging`, `discharging`, `overload`, `replaceBatt`, `bypass` | Příznaky `CHRG`, `DISCHRG`, `OVER`, `RB`, `BYPASS`. |
| `charge`, `chargeLow` | `battery.charge`, `battery.charge.low`, v %. |
| `runtimeSec`, `runtimeMin`, `runtimeTxt` | `battery.runtime` v sekundách, celé minuty a text `m:ss`. |
| `battType`, `battV` | Typ a napětí baterie. |
| `load` | `ups.load`, v %. |
| `realpower`, `powerNom` | `ups.realpower`, `ups.realpower.nominal`, ve W. |
| `powerVA`, `powerNomVA` | `ups.power`, `ups.power.nominal`, ve VA. |
| `headroomW` | Vypočtená nezáporná rezerva činného výkonu. |
| `inputV`, `outputV`, `outputVnom` | Vstupní, výstupní a jmenovité výstupní napětí. |
| `freq`, `freqNom` | Výstupní a jmenovitá výstupní frekvence, v Hz. |
| `beeper` | `ups.beeper.status`. |
| `ts` | Čas zpracování v Node-RED, Unix timestamp v milisekundách. |
| `nutComm` | `{ok, lastOk, host, port, upsName}` platné při vytvoření vzorku. |

Chybějící číselné hodnoty jsou `null`; nečíselný text může přes `parseFloat` skončit jako `NaN`. Konzumenti proto mají kontrolovat platnost čísla. `global.ups` nemá pole `batteryCharge`, `batteryRuntimeMin`, `realPowerW`, `lastUpdate` ani objekt `statusFlags`, které uváděla starší dokumentace.

### Kontrola čerstvosti před použitím

Komunikační stav má pole `ok`, `lastAttempt`, případně `lastOk`, dále `lastError`, `host`, `port`, `upsName`. Časy jsou v milisekundách; `lastAttempt` se zapisuje při dokončení pokusu. `ok: false` říká, že poslední dokončený pokus selhal, zatímco watchdog používá patnáctisekundovou toleranci od posledního úspěchu.

Tento příklad používá stejnou toleranci a navíc kontroluje stáří zpracovaného vzorku:

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
    node.warn("Chybí čerstvá data UPS; stav napájení nelze potvrdit.");
    return null;
}
if (oUps.onBattery) {
    node.warn("UPS běží na baterii.");
}
msg.payload = {
    onBattery: oUps.onBattery,
    charge: Number.isFinite(oUps.charge) ? oUps.charge : null,
    runtimeSec: Number.isFinite(oUps.runtimeSec) ? oUps.runtimeSec : null
};
return msg;
```

V Change uzlu lze pro jednoduché čtení zvolit zdroj typu **global** a cestu `ups.charge`. Samotné přečtení hodnoty neověřuje její aktuálnost.

### `global.lineaApiUpsState`

Snapshot vynechává sériové číslo, IP adresu, port i konfiguraci. Označení „read-only“ popisuje jeho použití pro monitoring; objekt v kontextu není technicky chráněn před změnou jiným flow.

Ilustrační struktura s hodnotami příkladu:

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

Snapshot vzniká po přijatých datech a watchdog jej při výpadku nezneplatní. Jeho stáří vyhodnocujte pomocí `updatedAt`; přítomnost objektu ani `online: true` nepotvrzuje aktuální dostupnost.

## Známá omezení tohoto exportu

Tato část popisuje skutečný stav implementace, nikoli již provedené opravy.

1. **Watchdog není propojen s panelem ani notifikacemi.** Po více než 15 s bez úspěchu zobrazí `NUT OFFLINE` pod uzlem v editoru. Nezmění `global.ups`, snapshot ani stará data v panelu. Bez prvního úspěchu vyhodnocuje stav jako offline.
2. **Úspěšný test vstupuje do provozního zpracování.** Při jiné testované UPS může ovlivnit data a alarmy; současné testy a běžné dotazy nejsou vzájemně uzamčené.
3. **Graf má pevné osy a podmínku přítomného výkonu.** Nelze jej považovat za úplný záznam výpadků napětí nebo univerzální výkonový graf.
4. **Více UPS není odděleno.** Kopie sdílejí globální klíče; na stejné kartě také historii a předchozí stav. Víceinstanční použití vyžaduje změnu jmenných prostorů a konfigurace.
5. **Dostupnost TCP není úplná validace dat.** Dotazovací uzel může přijmout odpověď s rozpoznaným začátkem bloku nebo řádkem `VAR`, kterou parser později odmítne. Test „OK“ nemusí znamenat použitelný provozní vzorek.
6. **Ošetření spojení má mezery.** Chybí ochrana před překrývajícími se dotazy při dlouhém timeoutu a explicitní úklid socketů při zastavení uzlu. Zavření spojení bez dat nemá vlastní chybové dokončení. Timeout používá `socket.end()`, nikoli vynucené zničení socketu; není zárukou úplného ukončení spojení.
7. **Potvrzení uložení a doručení nejsou koncová potvrzení.** UI nečeká na souborový zápis; ntfy větev nevyhodnocuje HTTP výsledek ani neopakuje pokusy.
8. **Samostatné UI upozornění na zastarání není dokončeno.** Panel obsahuje výpočet `stale` s hranicí 2 s, ale nepoužívá jej v šabloně. Tento nepoužitý výpočet neřeší detekci výpadku a jeho hranice navíc neodpovídá běžnému pětisekundovému pollingu.

## Řešení problémů

| Projev | Co zkontrolovat |
| --- | --- |
| `net is not defined` nebo problém knihovny | Přiřazení `net` → `net` a povolení knihoven Function uzlů. |
| `chybi NUT host` | Host ve formuláři a skutečný obsah `global.config.upsConfig`. |
| `ECONNREFUSED`, timeout | Službu NUT, adresu, port, firewall a síťovou cestu z Node-RED/kontejneru. |
| `ERR UNKNOWN-UPS` | Název UPS včetně velikosti písmen; ověřte jej přes `upsc -l`. |
| Jiná `ERR` odpověď | Přesný text chyby a diagnostiku NUT serveru; příčina nemusí být jen název UPS. |
| Test funguje, běžné dotazy ne | Test používá neuložené hodnoty. Uložte konfiguraci a ověřte, že ji jiný inicializační uzel nepřepisuje. |
| UI hlásí uloženo, po restartu vše zmizí | Ukládací link, převod konfigurace na JSON, práva, trvalý svazek a startovní načítání. |
| Panel zůstává zelený po ztrátě komunikace | Jde o poslední vzorek. Kontrolujte watchdog, `upsNutCommState.lastOk` a `ups.ts`. |
| Panel funguje, graf je prázdný | Zda UPS poskytuje `ups.realpower`; bez něj buffer nepřidá bod. |
| Výkon nebo napětí nejsou správně vidět | Pevné rozsahy grafu 0–100 a 200–250. |
| ntfy nic neposílá | UPS URL, skutečný přechod stavu, první vzorek bez alarmu a HTTP odpověď v dočasně připojeném Debug uzlu. |
| Notifikace nepoužívají centrální ntfy LINEA | UPS má vlastní `config.upsConfig.ntfyUrl`. |
| Chybí toast v Dashboardu | Export nemá zapojený toast; je nutné jej doplnit. |
| Dvě UPS si přepisují údaje | Modul používá pevné globální klíče a je určen pro jednu instanci. |

## Kontrola po instalaci

- [ ] Po Deploy nejsou neznámé uzly ani chyba knihovny `net`.
- [ ] Test vrací odpověď správné UPS a provozní panel následně dostává nové vzorky přibližně po 5 s.
- [ ] `global.ups.ts` i `global.upsNutCommState.lastOk` se obnovují.
- [ ] Konfigurace zůstane zachována po restartu Node-RED.
- [ ] Při dočasném zablokování komunikace watchdog přejde po více než 15 s na offline; navazující logika odmítne stará data.
- [ ] Po obnovení komunikace se aktualizují panel i snapshot.
- [ ] Stavové přechody a ntfy jsou ověřeny v kontrolovaném prostředí; samotný `Test NUT` neověřuje doručení ntfy.
- [ ] Integrace používá skutečné názvy polí a samostatně kontroluje jejich stáří.