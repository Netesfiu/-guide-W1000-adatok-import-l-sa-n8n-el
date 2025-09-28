# E.ON W1000 → n8n → Home Assistant (Spook) "integráció"

Ez a repó n8n workflow sablonokat tartalmaz tartalmaz, ami az E.ON portálról e-mailben érkező, **15 perces +A/-A** (import/export) adatokkal és a **1.8.0 / 2.8.0** napi mérőóra-állásokkállásokat tudja az ütemezett exportok alapján importálni a Homeassistant-be.
A csatolt **XLSX** fájlból kinyeri a szükséges oszlopokat, órára csoportosítja a negyedórás értékeket, majd:

* frissíti a **Spook/Recorder** statisztikáit `sensor.grid_energy_import` és `sensor.grid_energy_export` ID-k alatt, és
* beállítja az aktuális mérőóra-állást a `input_number.grid_import_meter` és `input_number.grid_export_meter` entitásokon.

> [!NOTE]
> A workflowkat szükséges lehet változtatni, amennyiben az EON által küldött ütemezett exportok kézbesítésében változás történik. Ha az exportált adatokban változás történik kérem, hogy [jelezd](https://github.com/Netesfiu/EON-W1000-n8n/issues/new)!

## Workflow sablonok
### [\[Netesfiu\] Alap workflow](/netesfiu/HA-W1000-workflow.json)
Ez a sablon egy workflow-ban kezeli a teljes folyamatot.

## Közreműködés
Ha szeretnéd a saját workflow-dat megosztani azt PR formájában megteheted a következő módon:
* Készíts egy új mappát a felhasználóneveddel
* A mappában mentsd le az általad készített workflow-t `.json`-formátumba
  * n8n-en belül a workflow nézetben: Ctrl-A, majd Ctrl-C
  * a `.json` file-ba pedig Ctrl-V
* Ha a beállításhoz szükséges készíts a mappában egy `readme` file-t.

* frissítsd ezt a `readme`-t a workflow sablonoknál:

```markdown
### [\[<FELHASZNÁLÓNÉV>\] <WORKFLOW_NEVE>](/<FELHASZNÁLÓNÉV>/<WORKFLOW NEVE>.json)
<workflow leírása>
```

> [!IMPORTANT]
> Győződj meg róla, hogy ha HTTP node-ot használsz nincsenek benne a saját hitelesítő adataid , mint pl. Bearer token (a Gmail, és Homeassistant node-ok nem osztják meg.)

## Követelmények

* n8n (felhős vagy self-hosted)
  *   HACS addon innen érhető el: [Rbillon59/home-assistant-addons](https://github.com/Rbillon59/home-assistant-addons)
  *   [n8n hivatalos docker compose sablon](https://docs.n8n.io/hosting/installation/server-setups/docker-compose/#6-create-docker-compose-file)
  *   [n8n egyszerűsített docker compose sablon](main/n8n-docker-compose.yaml)
* (Gmail esetén) Gmail API hitelesítés (OAuth2) **read-only** e-mail hozzáféréssel ahhoz a fiókhoz amelyikre az e-mail érkezik
  * Beállítási útmutató [itt érhető el](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/)
* (IMAP esetén) IMAP szolgáltató hitelesítési adatok
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

> Remélem a leírásom segített és hasznosnak találtad. Ha szeretnéd a munkámat támogatni, akkor egy kávéval megtehedet :coffee:

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/H2H06KNT5)
