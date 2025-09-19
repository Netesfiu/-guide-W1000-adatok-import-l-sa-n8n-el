# E.ON W1000 → n8n → Home Assistant (Spook) "integráció"

Ez a repó egy n8n workflow sablont tartalmaz, ami az E.ON portálról e-mailben érkező, **15 perces +A/-A** (import/export) adatokkal és a **1.8.0 / 2.8.0** napi mérőóra-állásokkal dolgozik.
A csatolt **XLSX** fájlból kinyeri a szükséges oszlopokat, órára csoportosítja a negyedórás értékeket, majd:

* frissíti a **Spook/Recorder** statisztikáit `sensor.grid_energy_import` és `sensor.grid_energy_export` ID-k alatt, és
* beállítja az aktuális mérőóra-állást a `input_number.grid_import_meter` és `input_number.grid_export_meter` entitásokon.

> [!NOTE]
> A workflow-t szükséges lehet változtatni, amennyiben az EON által küldött ütemezett exportok kézbesítésében változás történik. A változásokat igyekszek lekövetni. Ha az exportált adatokban változás történik kérem, hogy [jelezd](https://github.com/Netesfiu/EON-W1000-n8n/issues/new)!

## Követelmények

* n8n (felhős vagy self-hosted)
  *   HACS addon innen érhető el: [Rbillon59/home-assistant-addons](https://github.com/Rbillon59/home-assistant-addons)
  *   [n8n hivatalos docker compose sablon](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/#6-create-docker-compose-file)
  *   [n8n egyszerűsített docker compose sablon](main/n8n-docker-compose.yaml)
* Gmail API hitelesítés (OAuth2) **read-only** e-mail hozzáféréssel ahhoz a fiókhoz amelyikre az e-mail érkezik
  * Beállítási útmutató [itt érhető el](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/)
* Home Assistant elérés **Long-Lived Access Token**-nel vagy API kulccsal
  * Beállítási útmutató [itt érhető el](https://docs.n8n.io/integrations/builtin/credentials/homeassistant/)
* Spook integráció
  * a dokumentáció és telepítési útmutató [itt érhető el](https://spook.boo)

## Eon Portál beállítása
1. Hozz létre ütemezett exportálást az EON portálon a következő paraméterekkel
* Távleolvasás menüpont alatt kattints az `+ új ütemezett exportálási beállítás` gombra.
  * `POD azonosító(k) megadása`: azok közül legyen egy kiválasztva, amelyikre lekérdezést szeretnél beállítani.
  * `Mérőváltozó(k) megadása`: A következőket jelöld be:
    * `+A Wattos fogyasztás`
    * `-A Wattos betáplálás`
    * `DP_1-1:1.8.0*0 Wattos fogyasztás napi méróállás`
    * `DP_1-1:2.8.0*0 Wattos betáplálás napi méróállás`
  * `export küldésének gyakorisága`: naponta
  * `Hány napra visszamenőleg szeretné exportálni az adatokat`: 7 napra visszamenőleg javasolt, ezzel pótolhatóak a kimaradt napok.
  * `Kérjük adja meg a küldendő e-mail tárgyát`: alapértelmezetten az `[EON-W1000]` címet javaslom, ha több POD-nak a lekérdezését is a workflow-al szeretnéd feldolgozni javasolt egydi azonosítót adni.

## Home Assistant előkészítés

1. Hozzd létre a következő `input_number` entitásokat config.yaml file-ban, vagy segéd entitásokban:

[![Segéd entitások](https://my.home-assistant.io/badges/helpers.svg)](https://my.home-assistant.io/redirect/helpers/)
```yaml
input_number:
  grid_import_meter:
    name: grid_import_meter
    mode: box
    initial: 0
    min: 0
    max: 9999999999
    step: 0.001
    unit_of_measurement: kWh
  grid_export_meter:
    name: grid_export_meter
    mode: box
    initial: 0
    min: 0
    max: 9999999999
    step: 0.001
    unit_of_measurement: kWh
```
> *Ha másként nevezed el az entitásokat, akkor kövesd le a módosításokat a workflow-ban és a következő lépésben*

2. Hozzd létre a következő `template_sensor` entitásokat config.yaml file-ban, vagy segéd entitásokban:

  [![Segéd entitások](https://my.home-assistant.io/badges/helpers.svg)](https://my.home-assistant.io/redirect/helpers/)

```yaml
template:
  - sensor:
      - name: "grid_energy_import"
        state: "{{ states('input_number.grid_import_meter') | float(0) }}"
        unit_of_measurement: "kWh"
        device_class: energy
        state_class: total_increasing
      - name: "grid_energy_export"
        state: "{{ states('input_number.grid_export_meter') | float(0) }}"
        unit_of_measurement: "kWh"
        device_class: energy
        state_class: total_increasing
```
> *Ha másként nevezed el az entitásokat, akkor kövesd le a módosításokat a workflow-ban*

## n8n import és hitelesítések

1. **Workflow importálása**

   * n8n → *Workflows* → *Import from File/Clipboard* → illeszd be a repóban található JSON-t.

2. **n8n hitelesítések beállítása**
   A hitelesítési beállítások a homeassistant és a Gmail-node okban szükséges beállítani. A beállítás menetét a [#követelmények](#követelmények) pontnál találod

## Működés röviden

### **indítási feltétel és E-mail letöltés**:

<img width="926" height="310" alt="image" src="https://github.com/user-attachments/assets/d3253977-291d-498d-8a2f-1508fc0e3b69" />

* A workflow 2 féle képpen fut le:
  1. **időzítés alapján**: Minden nap a node-ban megadott órában
  2. **Gmail alapján**: A Gmail node elindítja a lefuttatást, ha a `noreply@eon.com` címtől érkezik levél a postafiókba.
* Az ezt követő szakaszban a node megvizsgálja, hogy a levél tárgyában szerepel-e a portálon megadott tárgy (alapértelmezetten: `[EON-W1000]`)
* Amennyiben a levél a tárgy szerint leolvasást tartalmaz letölti a benne található `.xlsx` fájlt további feldolgozásra.

### **Csatolmány feldolgozás**:
<img width="1015" height="491" alt="image" src="https://github.com/user-attachments/assets/96be0e5e-ea86-4ca4-a973-068bf70c83c6" />

* Az `.xlsx` táblázat a következőképpen tartalmazza a szükséges adatokat:

pod | Időbélyeg | Változó | Érték | Mértékegység | Változó | Érték | Mértékegység | Változó | Érték | Mértékegység | Változó | Érték | Mértékegység |  
-- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | --
HU000120-11-| 45905 | +A | 0.222 | kWh | -A | 0 | kWh | DP_1-1:1.8.0*0 | 32673.404 | kWh | DP_1-1:2.8.0*0 | 39435.881 | kWh |  
HU000120-11-| 45905.010416666664 | +A | 0.418 | kWh | -A | 0 | kWh | DP_1-1:1.8.0*0 | [undefined] | [undefined] | DP_1-1:2.8.0*0 | [undefined] | [undefined] |  
HU000120-11-| 45905.020833333336 | +A | 0.246 | kWh | -A | 0 | kWh | DP_1-1:1.8.0*0 | [undefined] | [undefined] | DP_1-1:2.8.0*0 | 

Mivel a lekérdezésben ismétlődő oszlopnevek szerepelnek a workflow a következőképpen nevezi át az oszlopokat:
| pod | Időbélyeg | Változó | Érték | Mértékegység | Változó_1 | Érték_1 | Mértékegység_1 | Változó_2 | Érték_2 | Mértékegység_2 | Változó_3 | Érték_3 | Mértékegység_3 |  
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |

* a workflow az utolsó `merge` node-nál a következőképpen alakítja át az adatokat:

| start | AP | AM | 1_8_0 | 2_8_0 |
| -- | -- | -- | -- | -- |
| 45912 | 0.135 | 0 | 32749.288 | 39627.868	|
| 45912.010416666664 | 0.207 | 0 | [undefined] | [undefined] |
| 45912.020833333336 |	0.153 |	0 |	[undefined] |	[undefined]	|
| ... | ... | ... | ... | ... |

* ezt követően az `start` oszlopba található időpontok átkonvertálásra kerülnek ISO időbélyegre (az excelben a "0" értékű dátummezők "1900 január 0."-nak felelnek meg) és kerekíti az időpontokat egész órára.
* `code {}` node összesíti a `+A` és `-A` 15 perces adatait az adott órában és kiszámolja ennek megfelelően az `1_8_0` és `2_8_0` órás adatait az sorok `[undefined]` mezőiben.

### `stats` adattömb generálása

<img width="785" height="334" alt="image" src="https://github.com/user-attachments/assets/168215a5-3c2f-4e83-aa1b-8c2a0e3e70ec" />

* A workflow itt az órásra kiszámított `1_8_0` és `2_8_0` értékekhez tartozó oszlopokat átnevezi a Spook `recorder.import_statistics` szolgáltatása által elfogadott nevekre. Ezt követően egy listába rakja az oszlop értékeit és a kapott listával meghívja a `recorder.import_statistics` szolgáltatást és frissíti a korábban elkészített `input_number` entitásokat az `1_8_0` és `2_8_0` listák utolsó elemével (utolsó ismert óraállás)

## Időzítés

* Az E.ON jellemzően **10:00** után küldi a legfrissebb adatokat.
* Ajánlott időzítés (Schedule Trigger): **10:10** vagy később (példában 14:00).

## Fontos node-ok és kifejezések

* **Gmail → Get last message**: `filters.sender = noreply@eon.com`, `limit = 1` (példa)
* **IF (Subject szűrés)**: bal érték `{{$json.Subject}}`, jobb érték `"[EON-W1000]"`
* **Attachment**: `attachment_0` (ha több csatolmány érkezhet, érdemes ellenőrizni a MIME-típusokat / nevet)
* **Excel idő konverzió** (`Convert Excel time2`):

  * Módszer: `addToDate` az **1899-12-30** bázisnaphoz (Excel epoch)
  * `duration: {{$json['start'] + 0.00000001}}` – apró offset a float hibák ellen
* **Dátum formátum a Spookhoz**: `yyyy-MM-dd HH:00:00ZZ`
* **Recorder import payload** (`service: recorder.import_statistics`):

  * `statistic_id`: pl. `sensor.grid_energy_import`
  * `unit_of_measurement`: `kWh`
  * `has_mean: {{false}}`, `has_sum: {{true}}`
  * `stats`: az `Aggregate` node által készített `data` tömb (elemei: `{ start, state, sum }`)

Remélem a leírásom segített és hasznosnak találtad. Ha szeretnéd a munkámat támogatni, akkor egy kávéval megtehedet :coffee:

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/H2H06KNT5)
