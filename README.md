# Twister Control — Részletes Kezelői Kézikönyv

Ez a program a fonalsodró gép teljes vezérlőszoftvere érintőképernyőre (`768×1200` álló kijelzőre) és normál egér + billentyűzet használatra tervezve.  
A PC egyetlen RS-485 soros kábelen keresztül (`COM3`, `19200 bps`, `8N1`) egyszerre 4 eszközt irányít:
1. **`0x01` — Kinco FM léptetőmotor-hajtás** (a fonálvezető szán fel-le mozgatása)
2. **`0x02` — Waveshare 8 csatornás I/O relémodul** (fizikai Start/Stop gombok olvasása + jelzőlámpák és főkontaktor kapcsolása)
3. **`0x03` — Delta VFD frekvenciaváltó** (alsó gyűjtőorsó villanymotor)
4. **`0x04` — Delta VFD frekvenciaváltó** (felső fonaladagoló henger villanymotor)

---

## 1. Alapvető használat (Indítás, mentés, számok átírása)

### Indítás és ablakkijelzés
- **Program indítása:** Kattints duplán a `dist\TwisterControl.exe` fájlra. Semmit sem kell telepíteni.
- **Teljes képernyő ↔ Ablak váltás:** Alapértelmezetten teljes képernyőn indul. Az **`ESC`** billentyűvel (vagy a jobb felső sarokban lévő `ESC` / `FULL` gombbal) bármikor válthatsz ablakos és teljes képernyős mód között *(kivéve ha épp egy szövegdobozba írsz — akkor az első `ESC` kilép a dobozból, a második `ESC` vált ablakmódot)*.
- **Automatikus mentés és napló:**
  - Minden módosítás automatikusan elmentődik a program melletti **`twister_config.ini`** fájlba. Ha újraindítod a programot, minden beállításod pontosan úgy marad, ahogy hagytad.
  - Ha van a régi C# programból megmaradt `config.txt` fájlod az `.exe` mellett, az első indításkor a program automatikusan átolvassa belőle a régi adatokat.
  - Minden működési lépés és hiba bekerül a **`debug.txt`** fájlba.

### Hogyan kell értékeket (számokat és kódokat) módosítani?
Nincs felugró képernyő-billentyűzet — közvetlenül a rendes billentyűzetről tudsz írni:
1. **Kattints bele abba a dobozba**, amit át akarsz írni. A doboz kerete **világoskékre vált**, a régi szám kijelölődik, és megjelenik a végén a `|` kurzor.
2. **Írd be az új értéket:**
   - Ha rögtön elkezdesz gépelni, az első leütés felülírja a régi számot.
   - Ha a **`Backspace`**-t nyomod meg (vagy még egyszer belekattintasz az aktív dobozba), karakterenként törölhetsz és javíthatsz.
3. **Mentéshez két lehetőséged van:**
   - Nyomd meg az **`Enter`** billentyűt, **VAGY**
   - **Kattints bárhová máshová a képernyőn** (egy másik dobozra, másik fülre vagy az üres háttérre).
   - Mindkét esetben a program azonnal elmenti az új értéket a fájlba, újraszámolja a függő értékeket és kilép a szerkesztésből.
4. **Mégse (`ESC`):** Ha írás közben **`ESC`**-et nyomsz, elveti a félbehagyott gépelést, és visszaáll a korábbi érték.

---

## 2. A felső állapotsor (minden oldalon ugyanott látható)

| Elem | Mit mutat / Mit csinál ha rákattintasz? |
| :--- | :--- |
| **`TWISTER`** | A program neve. |
| **`COM3`** *(port gomb)* | Zöld, ha a soros port sikeresen csatlakozva van; piros, ha nem érhető el. Rákattintva rögtön a **`BEÁLLÍTÁSOK`** oldalra ugrik, ahol portot választhatsz. |
| **Állapotjelző doboz** | A gép pillanatnyi állapota:<br>• **`OFFLINE`** (piros): nincs nyitott soros port<br>• **`KÉSZENLÉT`** (sárga): áll a gép, várja az indítást<br>• **`INDÍTÁS...`** (sárga): relék kapcsolása, frekiváltók bebootolása és felpörgése<br>• **`ÜZEMEL`** (zöld): forognak a motorok és megy a léptetőmotor-ciklus<br>• **`LEÁLLÍTÁS`** (piros): motorok leállítása és léptető Home-ba küldése<br>• **`HIBA`** (piros): hiba történt (pl. port vagy meghajtó hiba — részletek a `DIAGNOSZTIKA` naplóban) |
| **`ESC` / `FULL` gomb** | Vált teljes képernyős és ablakos mód között (ugyanazt csinálja, mint az `ESC` billentyű). |
| **4 fő oldalválasztó fül** | **`VEZÉRLÉS`** • **`MODBUS`** • **`DIAGNOSZTIKA`** • **`BEÁLLÍTÁSOK`** — rákattintva azonnal oldalt vált (ha épp írtál egy mezőbe, előtte elmenti). |

---

## 3. Hogyan indul és áll le a gép a fizikai gombokkal?

A program folyamatosan figyeli a gép **`0x02`** I/O modulján lévő fizikai Start és Stop gombokat:

### Zöld gomb (`Start`) megnyomása:
1. Kikapcsolja a piros jelzőlámpát (`0x0000`), felkapcsolja a zöld jelzőlámpát (`0x0001`).
2. Áramot ad a frekiváltók főkontaktorára (`0x0003`).
3. **Megvárja, míg a két Delta frekiváltó feléled** és hibamentes készenlétet jelez (`0x2100/0x2101`).
4. Beírja a beállított felfutási és lefutási időket (ha `> 0 ms`), elküldi a kiszámolt célfrekvenciákat (`0x2001`), majd kiadja az indítás parancsot (`0x2000 = 0x0012`).
5. **Megvárja, míg mindkét motor ténylegesen felpörög** a célfrekvenciára (`0x2103`).
6. Csak ezután indítja el a Kinco léptetőmotor ciklusát (`0`-tól `4400` felé).

### Piros gomb (`Stop`) megnyomása:
1. Azonnal kiadja a leállítás parancsot mindkét Delta frekiváltónak (`0x2000 = 0x0001`).
2. Ha a **`Home leállításkor: BE`** be van kapcsolva, visszaküldi a léptetőmotort a kiinduló pontba (**Home**) a megadott **`HOME SEBESSÉG`** értékkel, és megvárja, míg odaér. Ezután letiltja a léptető hajtást.
3. **Megvárja, míg mindkét Delta frekiváltó ténylegesen megáll (`0 Hz`, `Stop` állapot)** — így a zöld lámpa mindaddig világít, amíg az orsók ténylegesen forognak.
4. Ha minden megállt: zöld lámpa KI, piros lámpa BE, a gép **`KÉSZENLÉT`**-be áll.

---

## 4. Oldalankénti részletes leírás — Minden gomb és mező

---

### 1. oldal: `VEZÉRLÉS` *(Főoldal napi gyártáshoz)*

Ez az oldal szándékosan letisztult, nagy feliratokkal és kevés gombbal, hogy üzemi környezetben is tévedésmentesen lehessen kezelni.

#### A) Számítási mód váltó (legfelső két gomb)
A három fonalparaméter közül kettőt te adsz meg, a harmadikat a gép áttételeiből automatikusan számolja a program:
* **`Sodrat + Orsó → Henger`** *(aktív = piros)*:
  - Te írod be az **`ALSÓ GYŰJTŐORSÓ`** fordulatszámát és a **`SODRATSZÁM`**-ot.
  - A gép kiszámolja a **`FELSŐ ADAGOLÓ HENGER (SZÁMOLT)`** értékét:
    $$\text{Henger RPM} = \text{Orsó RPM} / \text{Sodratszám}$$ *(egész osztás, pl. `2500 / 120 = 20 RPM`)*
* **`Orsó + Henger → Sodrat`**:
  - Te írod be az **`ALSÓ GYŰJTŐORSÓ`** és a **`FELSŐ ADAGOLÓ HENGER`** fordulatszámát.
  - A gép kiszámolja a **`SODRAT (SZÁMOLT)`** értékét:
    $$\text{Sodratszám} = \text{Orsó RPM} / \text{Henger RPM}$$

#### B) A 3 nagy paraméterdoboz
* **`ALSÓ GYŰJTŐORSÓ` (`RPM`):** Mindig szerkeszthető. Ebből számolja a frekiváltó parancsot is: `((Orsó / 118) × 100)` (`0,01 Hz`-es egységben, pl. `2500 → 2100 = 21,00 Hz`).
* **`SODRATSZÁM` (`sodrat`)** *(vagy `SODRAT (SZÁMOLT)`)*: Ha a bal oldali mód aktív, rózsaszín keretes és kattintva szerkeszthető; ha a jobb oldali mód aktív, világoskék keretes (csak kijelző, a program számolja).
* **`FELSŐ ADAGOLÓ HENGER` (`RPM`)** *(vagy `(SZÁMOLT)`)*: Jobb oldali módban szerkeszthető; bal oldali módban automatikusan számolt (`Henger × 0,96 × 100`, pl. `20 → 1920 = 19,20 Hz`).

#### C) A két folyamatjelző sáv (középen)
* **`Ciklus` sáv** (`pl. 0 / 23  •  01:18`):
  - A szám azt mutatja: **hány lépés teljesült / hány lépésből áll az aktuális menet**.
  - Mellette az **`MM:SS`** formátumú idő a **várható hátralévő idő** (indításnál is látod előre, mennyit fog menni a program, futás közben pedig folyamatosan, egyenletesen csökkenve pontosodik a mért léptetési sebességhez — sosem ugrál össze-vissza!).
* **`Léptetőmotor pozíció` sáv** (`pl. 0 / 4400`):
  - A szán aktuális helyzetét mutatja a **`0`** és a **`FELSŐ HATÁR`** (`4400`) között, **mínusz előjel nélkül** *(a háttérben továbbra is negatív számmal számol a Kinco vezérlőnek, de a kijelzőn mindig tisztán, előjel nélkül jelenik meg)*.
  - Ha `0x3700` valós pozíció-kiolvasás válaszol, pontosan a motor tényleges elmozdulását követi; visszafelé tekercselésnél pedig lefelé csökken a sáv.

#### D) A két óriás biztonsági gomb (alul)
* **`VÉSZSTOP`** *(óriás piros gomb)*:
  - Bármikor azonnal megszakít minden folyamatot:
    1. Egy paranccsal **lekapcsolja az összes relét** (`02 05 00 FF 00 00`) — elmegy a kontaktor táp és minden lámpa.
    2. **Helyben megállítja** a léptetőmotort (`0x3100 = 0x000F`) — **NEM indítja el Home-ba** (nehogy visszafelé mozdulva kárt tegyen!).
    3. Leállítja mindkét frekiváltót (`0x2000 = 0x0001`).
* **`PC KI`** *(alsó sötétpiros gomb)*:
  - Megnyit egy megerősítő ablakot (`GÉP ÉS PC LEÁLLÍTÁSA`), hogy véletlen érintésre ne kapcsoljon ki:
    - **`MÉGSE (ESC)`:** Bezárja az ablakot, nem történik semmi.
    - **`IGEN, LEÁLLÍTÁS`:** Lefuttatja a Vészstop relé-lekapcsolást, vár 1 másodpercet, majd azonnal leállítja a Windows számítógépet (`shutdown /s /t 0`).

---

### 2. oldal: `MODBUS` *(Eszközök, címek és kézi parancsteszt)*

Itt van összegyűjtve a gép összes Modbus-kódja **HEX** (hexadecimális) formában. Bármelyik mezőre kattintva átírhatod (pl. `4000` vagy `0x4000` is működik, `Enter` vagy mellékattintás menti).

#### 1. kártya: `Modbus Slave Címek (Busz ID • HEX)`
* **`LÉPTETŐ ID`** (alap: `0x01`): A Kinco FM léptetőmotor-hajtás buszcíme.
* **`I/O MODUL ID`** (alap: `0x02`): A Waveshare 8 csatornás gomb/relé modul buszcíme.
* **`ORSÓ VFD ID`** (alap: `0x03`): Az alsó gyűjtőorsót hajtó Delta frekiváltó címe.
* **`HENGER VFD ID`** (alap: `0x04`): A felső adagoló hengert hajtó Delta frekiváltó címe.
* **`Gyári Modbus alap` gomb** *(jobb felső narancs gomb)*: Egy kattintással visszaállítja az összes Modbus címet és kódot az eredeti alapértékre.

#### 2. kártya: `Kinco FM Léptetőmotor Regiszterek`
* **`POZÍCIÓ (32B)`** (`0x4000`): Ide írja a célpozíciót (32 bites szám).
* **`AKTUÁLIS POZ`** (`0x3700`): Innen olvassa ki a motor tényleges, valós idejű pozícióját mozgás közben.
* **`VEZÉRLŐ REG`** (`0x3100`): Vezérlőregiszter (indítás / megállítás).
* **`ÁLLAPOT REG`** (`0x3200`): Statusword (ebből tudja a program, hogy a motor még mozog, célba ért, vagy hibát jelez).
* **`ÜZEMMÓD`** (`0x3500`): Pozíció mód (`1`) és Homing mód (`6`) közti váltás.
* **`SEBESSÉG`** (`0x4A00`): A munka közbeni mozgási sebesség regisztere.
* **`HOME SEB.`** (`0x5010`): A Home-ra (végállás-keresésre) állított sebesség regisztere (CiA-402 módban).
* **`ENABLE`** (`0x001F`): Parancsszó, ami elindítja a beírt pozícióra a mozgást.
* **`STOP / DIS`** (`0x000F`): Parancsszó, ami leállítja/nyugtázza a léptetés indítóját.

#### 3. kártya: `Delta VFD (Orsó & Henger)`
* **Gyorsválasztó gombok a fejlécben** *(kiválasztott = piros keret)*:
  - **`VFD-E / EL`**: Beállítja az `ACCEL REG = 0x0109` és `DECEL REG = 0x010A` címeket (`Pr.01.09 / 01.10`).
  - **`VFD-M`**: Beállítja az `ACCEL REG = 0x010A` és `DECEL REG = 0x010B` címeket (`Pr.10 / Pr.11`).
  - **`MS300 / C`**: Beállítja az `ACCEL REG = 0x010C` és `DECEL REG = 0x010D` címeket (`Pr.01-12 / 01-13`).
  *(Válaszd ki azt, amelyik széria a gépedben van, hogy a fel- és lefutás a jó helyre íródjon!)*
* **`VFD VEZÉRLŐ`** (`0x2000`): Start/Stop vezérlőregiszter.
* **`FREKVENCIA REG`** (`0x2001`): Ide írja a kiszámolt célfrekvenciát (`0,01 Hz`-es léptékben).
* **`START PARANCS`** (`0x0012`): Előre forgás indítása.
* **`STOP PARANCS`** (`0x0001`): Lassított megállítás parancs.
* **`HIBA REG`** (`0x2100`): Hibakód olvasása (`0` = hibamentes).
* **`ÁLLAPOT`** (`0x2101`): Üzemállapot olvasása (áll, lassul, készenlét, üzemel).
* **`KIMENŐ HZ`** (`0x2103`): A motorra kiadott tényleges kimeneti frekvencia olvasása.
* **`ACCEL REG`** / **`DECEL REG`**: A felfutási és lefutási idő paraméter-regisztere (`0,1 mp` egységben).

#### 4. kártya: `Waveshare I/O Modul Bemenetek & Relé Tekercsek`
* **`BEMENET CÍM`** (`0x0000`): A digitális bemenetek (nyomógombok) kezdőcíme.
* **`ZÖLD START BIT`** (`0x03`): Ez a bemeneti érték indítja el a gépet (`IN0 + IN1` aktív).
* **`PIROS STOP BIT`** (`0x00`): Ez a bemeneti érték jelenti a Stop gombot (vagy relé bontását).
* **`PIROS LÁMPA`** (`0x0000`): Piros jelzőlámpa relé kimenete.
* **`ZÖLD LÁMPA`** (`0x0001`): Zöld jelzőlámpa relé kimenete.
* **`VFD TÁP RELÉ`** (`0x0003`): A két frekiváltó tápkontaktorát kapcsoló relé.
* **`ÖSSZES RELÉ`** (`0x00FF`): Az összes kimeneti relé együttes lekapcsolása (Vészstophoz).

#### 5. kártya: `Egyedi Modbus RTU Parancs Küldése`
Szervizhez, próbaküldéshez:
* **`HEX BÁJTOK` mező:** Kattints bele és írj be szóközökkel (vagy kötőjellel) tagolt bájtokat, pl. `01 06 31 00 00 1F`.
* **`Auto CRC` / `Kézi CRC` gomb:** Ha `BE`, automatikusan hozzászámolja a 2 bájtos Modbus CRC-16 ellenőrző összeget küldéskor; ha `KI`, pontosan azt küldi ki, amit beírtál.
* **`KÜLDÉS` (`RTU TX`) gomb:** Kiküldi a parancsot a soros portra, a **`Válasz:`** sorban pedig azonnal megjelenik a nyers válasz vagy hibaüzenet.
* **4 minta-gomb alul:** **`Léptető En`**, **`Léptető St`**, **`Orsó VFD St`**, **`Relék KI`** — egy kattintással beírják a megfelelő parancsot a felső mezőbe (az aktuális Slave ID-kkel!). Innen a `KÜLDÉS`-sel rögtön tesztelheted.

---

### 3. oldal: `DIAGNOSZTIKA` *(Műszerfal, tesztek és élő napló)*

#### Felső állapotsorok
* **1. sor:** `TX` (elküldött parancsok száma) | `RX` (sikeresen vett válaszok) | `Hiba` (sérült CRC / hibás válaszok) | `Újrapr` (automatikusan újrapróbált üzenetek) | `I/O` (aktuális bemenet bájt hexában) | `0x3200` (léptető utolsó státusza: `Célba ért`, `Mozog`, `HIBA!`).
* **2. sor:** Mindkét Delta VFD élő kijelzése: **`Orsó VFD: XX.XX Hz (állapot, err=...)`** és **`Henger VFD: XX.XX Hz (állapot, err=...)`**.  
  Az állapotok jelentése: `RUN` = megy, `KÉSZ` = táp alatt áll, `LEFUT` = lassul megállásig, `STOP` = áll, `HIBA` = frekiváltó hibajelzés, `--` = kikapcsolt táp / nem válaszol.

#### `IN0` – `IN7` bemenet-visszajelzők
A 8 optocsatolt bemenet állapota valós időben (zöld = bemenet aktív/zárt, sötét = nyitott).

#### A 6 tesztgomb (külön-külön kipróbáláshoz)
* **`Poz +200` (`Küldés`):** Elküldi a `+200`-as célpozíciót a léptetőnek (önmagában még nem indítja a motort, vagy előkészíti a pozíciót).
* **`Poz 0` (`Küldés`):** Elküldi a `0` célpozíciót a léptetőnek.
* **`Enable` (`0x001F`):** Kiadja a léptetőmotornak a mozgásindító parancsot (`0x3100 = 0x001F`).
* **`0x3200` (`Léptető`):** Azonnal lekérdezi a léptetőmotor állapotát és aktuális pozícióját (`0x3700`).
* **`0x2100` (`VFD Áll.`):** Azonnal lekérdezi mindkét Delta frekiváltó hibakódját, üzemállapotát és kimeneti frekvenciáját.
* **`Ciklus` (`Teszt`):** Elindít **csak egy** léptetőmotoros előremenetet (`0 → 4400`) a frekiváltók bekapcsolása nélkül — szán- és végállás-próbához.

#### Élő Modbus napló (alul)
* Színkódolt élő lista minden történésről (ujjal húzva vagy egérgörgővel görgethető):
  - `[INFO]` (szürke): rendszeresemények, fázisváltások
  - `[TX]` (kék): PC által küldött Modbus üzenetek
  - `[RX]` (zöld): eszközöktől visszaérkezett válaszok
  - `[WARN]` (narancs): újrapróbálás, tartalék módra váltás, vészstop
  - `[HIBA]` (piros): kommunikációs vagy hajtás hiba
* **`Napló törlése` gomb:** Kiüríti a képernyőn látható listát (a `debug.txt` fájl megmarad!).

---

### 4. oldal: `BEÁLLÍTÁSOK` *(Minden paraméter és működési opció)*

#### Felső kártya: `Soros Port & RS-485 Ütközésvédelem`
* **`Portok friss.` gomb:** Újra beolvassa a Windows-ban jelenleg csatlakoztatott COM portokat.
* **`Újracsatl.` gomb:** Lezárja és újranyitja a kiválasztott soros portot.
* **COM port gombok (`COM3`, `COM1`, ...):** Kattintással kiválasztható a használt RS-485 átalakító portja (`Aktív` = zöld, ha él a kapcsolat).
* **`Léptető (0x4000) sorrend:` `Kinco CDAB` vs `Big-Endian ABCD`:**
  - A 32-bites pozíciószám két regiszterének sorrendje. **Kinco FM**-hez a **`Kinco CDAB` (Low-Word First)** a gyári beállítás (ez megegyezik az eredeti C# program sorrendjével).
* **RS-485 busz védelmi mezők (ütközések ellen):**
  - **`RS-485 SZÜNET` (`15 ms`):** Ennyi kötelező csendet tart két Modbus üzenet között, hogy az egyik eszköz adása garantáltan befejeződjön, mielőtt a következő elkezdődik.
  - **`VÁLASZ TIMEOUT` (`120 ms`):** Legfeljebb ennyi ideig vár egy eszköz válaszára, mielőtt továbbmegy vagy újrapróbálja.
  - **`ÚJRAPRÓBÁLKOZÁS` (`2 db`):** Ha zavar miatt kimarad vagy hibás CRC-vel érkezik vissza egy válasz, ennyiszer küldi újra automatikusan.

#### Középső kártya: `Léptető, Delta VFD & Állapotgép beállítások`
* **`Gyári alapértékek` gomb** *(jobb felső narancs gomb)*: Visszaállítja az alábbi összes gépbeállítást a biztos, bevált alapokra (a `MODBUS` oldal címeit békén hagyja!).

##### Paraméterdobozok:
1. sor:
* **`LÉPTETŐ LÉPÉS` (`400 offset`):** Minden ciklusban ennyit lép a motor a cél (`4400` felső határ) felé. Mindig nagyobbnak kell lennie, mint a `FEL-LÉPÉS (+)`!
* **`LÉPÉS SEBESSÉG` (`0 = alap`):** A léptetőmotor munkasebessége (`0x4A00` regiszter). Ha `0`, nem írja felül a hajtásban tárolt belső sebességet (ugyanúgy, mint a régi programban). Ha `> 0`, elküldi az új sebességet.
* **`FELSŐ HATÁR` (`4400 poz`):** Meddig menjen fel a léptetőmotor (`0`-tól `4400`-ig; háttérben `-4400`-ként kezeli).

2. sor:
* **`FEL-LÉPÉS (+)` (`200 +`):** Ennyit mozdul vissza ellentétes irányba minden lépés elején. A nettó haladás ciklusonként = `LÉPTETŐ LÉPÉS - FEL-LÉPÉS (+)` (tehát `400 - 200 = 200`, így lesz `4400 / 200 + 1 = 23` lépés).
* **`HOME SEBESSÉG` (`0 = alap`):** Külön sebesség beállítás a Home-ra (visszaállásra). `0` esetén nem módosítja a sebességet; `> 0` esetén `CiA-402` módban a `0x5010` homing regiszterbe, `Pozíció 0` módban a `0x4A00`-ba írja a visszaállás idejére.
* **`MAX LÉPÉSIDŐ` (`15000 ms`)** *(vagy `FIX VÁRAKOZÁS` ha az állapotgép ki van kapcsolva)*: Legfeljebb ennyit vár egyetlen lépés befejezésére állapotgép módban, mielőtt továbbmegy.
* **`HOME TARTALÉK` (`6000 ms`):** Max/tartalék várakozási idő a Home mozgás befejezésére.

3. sor (Delta Frekiváltók fel- és lefutása):
* **`ORSÓ FELFUTÁS` (`2000 ms`):** Mennyi idő alatt pörögjön fel az alsó gyűjtőorsó `0`-ról a célfordulatra (Delta `0,1 mp` egységbe átszámolva, pl. `2000 ms = 2,0 mp`; ha `0`, marad a frekiváltón beállított idő).
* **`ORSÓ LEFUTÁS` (`2000 ms`):** Mennyi idő alatt álljon meg az alsó gyűjtőorsó (`0 = VFD alap`).
* **`HENGER FELFUTÁS` (`2000 ms`):** Mennyi idő alatt pörögjön fel a felső adagoló henger (`0 = VFD alap`).
* **`HENGER LEFUTÁS` (`2000 ms`):** Mennyi idő alatt álljon meg a felső adagoló henger (`0 = VFD alap`).

##### Fő kapcsolók:
* **`Léptető állapotgép (0x3200): BE / KI`:**
  - **`BE` (ajánlott):** Folyamatosan kérdezi a Kinco meghajtótól a `0x3200` (célba érés) és `0x3700` (tényleges pozíció) regisztert, és amint odaért a motor, azonnal folytatja (nincs felesleges várakozás). Ha nem kap választ a meghajtótól, automatikusan átvált a `Tartalék lépésidő`-re.
  - **`KI`:** Fixen kivárja a beállított időzítést (`FIX VÁRAKOZÁS`).
* **`VFD állapotgép (0x2101): BE / KI`:**
  - **`BE` (ajánlott):** Tápbekapcsolás után megvárja a frekiváltók tényleges beindulását, Start után ellenőrzi, hogy elérték-e a célfordulatot, leállításkor pedig megvárja, míg `0 Hz`-re lelassulnak.
  - **`KI`:** Fix időzítésekkel vár a kontaktor és a Start után.
* **`Home leállításkor: BE / KI`:** Ha `BE`, a piros Stop gomb (vagy a ciklus végi automatikus leállás) előbb visszaküldi a léptetőt Home-ba, és csak utána tiltja le.
* **`Home mód: Pozíció 0` vs `CiA-402 (0x3500)`:**
  - **`Pozíció 0` (biztonságos alapértelmezett):** Egyszerűen visszavezényli a motort a `0` pozícióra (`0x4000 = 0`).
  - **`CiA-402 (0x3500)`:** Átkapcsolja a Kinco drivert gyári Homing módba (`0x3500 = 6`), megkeresteti a végálláskapcsolót (`0x5010` sebességgel), majd visszavált Pozíció módba (`1`). *(Csak akkor válaszd ezt, ha a hajtásban a végálláskapcsoló és Homing Method fel van tanítva!)*
* **`Teljes képernyőn indul: BE / KI`:** Eldönti, hogy a program indításakor egyből teljes képernyős legyen-e.
* **`Tartalék lépésidő: 3000 ms`:** Rákattintva közvetlenül átírható. Akkor használja a léptetővezérlő tartalékként, ha a `0x3200` állapotjelzés nem érkezne meg a hajtásról.
* **`Modbus Kódok & Kereső →` gomb:** Gyors átugrás a `MODBUS` oldalra.

##### `Ciklus végén` és a több algombos visszafutás-választó:
* **`Ciklus végén: Leáll (1:1)` / `Újraindul`:**
  - **`Leáll (1:1)`** *(zöld)*: Ha a szán eléri a `4400`-as határt, leállítja a gépet, Home-ba küldi a motort és visszavált készenlétbe. *(Az alatta lévő két algombsor ilyenkor inaktív.)*
  - **`Újraindul`** *(sárga)*: Nem áll le a `4400`-nál, hanem folyamatosan tovább dolgozik a beállított mód szerint, amíg meg nem nyomod a Stop-ot vagy Vészstop-ot! Ekkor aktiválódik alatta a **2 soros algomb-választó**:
    1. **`Újraindulás módja:`**
       - **`Nulláról indul`:** Eléri a `4400`-at, majd rögtön visszatér és újra `0`-tól `4400`-ig fut végig.
       - **`Visszafelé indul`:** Amikor elérte a `4400`-at, **nem ugrik vissza**, hanem szépen lépésről lépésre visszafelé tekercsel lefelé!
    2. **`Visszafutás célja:`** *(akkor aktív, ha `Visszafelé indul` van kiválasztva)*:
       - **`Teljesen nulláig (0)`:** `0 → 4400 → 0 → 4400 ...` folyamatos teljes oda-vissza ciklus.
       - **`Középpontig (2200)`:** Első menetben felmegy `0 → 4400`-ig, utána visszafelé **csak a felső határ feléig (`2200`-ig)** megy vissza, és innentől **`2200 ↔ 4400`** között dolgozik oda-vissza folyamatosan!

---

#### Alsó kártya: `Képletek & Aktív Busz Paraméterek`
Csak tájékoztató áttekintő doboz: egyetlen pillantással látszanak benne az aktuális `MODBUS` fülről származó főbb címek (`Orsó`, `Henger`, `Kinco Poz/Akt/HomeSeb`, `Delta VFD Acc/Dec`), valamint az aktív COM port.

---

## 5. Hogyan van megoldva a háttérben? (Stabilitás és pontosság)

- **Ütközésmentes egyetlen soros busz-kezelő:**
  A régi rendszerekben a külön háttérben futó gomb-lekérdezés könnyen beleütközhetett egy éppen elküldött motorparancsba az RS-485 kábelen. Itt egyetlen szál végzi sorban az összes kommunikációt: minden küldés előtt kivárja az `RS-485 SZÜNET` (`15 ms`) csendet, kiüríti a régi zajt, majd minden írásnál (`0x05`, `0x06`, `0x10`) és olvasásnál (`0x02`, `0x03`) megvárja a CRC-16-tal ellenőrzött választ. Ha egy csomag megsérül, az `ÚJRAPRÓBÁLKOZÁS` azonnal újraküldi.
- **Valódi eszköz-állapotgép + biztonsági tartalék:**
  - Nem vak várakozási időkkel lép tovább a program: tápbekapcsoláskor a Delta `0x2100/0x2101` készenlétet figyeli, Startnál a `0x2103` kimenő frekvencia elérését, meneteléskor a Kinco `0x3200` (célba érés + hibafigyelés) és `0x3700` (valós pozíció) adatait, leállításkor pedig a frekiváltók `0 Hz`-es lefékezését.
  - Közben menet közben is felváltva ellenőrzi a fizikai Stop gombot és a két Delta VFD hibakódját (`0x2100`), hogy hiba esetén azonnal leálljon.
  - A főoldalon megjelenő hátralévő idő (`MM:SS`) futás közben a mért valós léptetési időkből simított átlaggal számolódik, ezért nem ugrál össze-vissza.

---

## 6. Fejlesztői fordítás

Ha módosítasz a C++ forráskódon (`src\` mappa), a `build-exe.bat` lefuttatásával:
1. Lefordítja és futtatja mind a 8 automatikus egységtesztet (`build\ProtocolTests.exe`).
2. Elkészíti az új, önálló futtatható programot a `dist\TwisterControl.exe` fájlba.
