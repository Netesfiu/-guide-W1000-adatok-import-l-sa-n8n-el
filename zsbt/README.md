# Három részes workflow

Az eredeti workflow szétszedve az alábbiakra:
  - email olvasása (nálam IMAP van)
  - XLSX betöltése, szétszedése
  - adatok HA-ba töltése


A cél, hogy amennyiben változna a táblázat formátuma, csak egy workflow-t kelljen módosítani.
Az email olvasás és HA betöltés marad változatlan n8n-ben, a kredenciák is megmaradnak bennük.

## Telepítés

  1) Importáld a három json fájlt, ezután lesz három független workflow - ha Gmail-t használsz, akkor
persze ne ezt az IMAP-eset :)
  2) Az első két workflow-ba belépve, az utolsó lépést szerkesztve válaszd ki a következő workflow meghívását
  3) Aktiváld ez email olvasás workflow-t.

## Ellenőrzés

Az itt lévő képekkel hasonlítsd össze a sajátodat, hogy nagyjából ugyanúgy nézzenek ki.

