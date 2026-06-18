# COC: Sistema de Detecció de MIDI Obsolets a Cubase 15

**Cubase14_ObsoleteMidi_Canary** — Vigilancia passiva de missatges MIDI que ja no s'usen

---

## Índex de continguts

1. [Què és COC](#què-és-coc)
2. [Motivació: Per què existeix COC](#motivació-per-què-existeix-coc)
3. [Arquitectura: Dos Translators, Dos Ports](#arquitectura-dos-translators-dos-ports)
4. [Estructura de cada Translator a BMTP](#estructura-de-cada-translator-a-bmtp)
5. [Camps del fitxer .txt i estats possibles](#camps-del-fitxer-txt-i-estats-possibles)
6. [Què fer quan COC salta](#què-fer-quan-coc-salta)
7. [Manteniment: Sincronització Manual](#manteniment-sincronització-manual)
8. [Documentació ampliada](#documentació-ampliada)

---

## Què és COC

### Definició tècnica

**COC** (Cubase14_ObsoleteMidi_Canary) és un sistema de **detecció passiva** i **extern** a aquest script que resideix a:

- **Ubicació**: Bome MIDI Translator Pro (BMTP)
- **Secció**: Global
- **Preset**: Canary Hub

COC **vigila missatges MIDI** que estaven configurats abans de Cubase 15 i que ja no s'usen a Cubase 15. Aquests missatges es consideren **OBSOLETS**.

### Concepte de missatge obsolet

Un missatge MIDI es considera **OBSOLET** quan compleix aquestes condicions:

| Condició | Significat |
|----------|-----------|
| Estava configurat | En algun dels sistemes de mapatge MIDI anteriors a Cubase 15 (Dispositivo Generico, MIDI Remote JSON o MIDI Remote JS) |
| Ja no està lligat | Després de la migració a Cubase 15, no està connectat a cap element actiu que el reculli |
| El missatge "existeix" | Alguna font del setup podria continuar emetent-lo |
| El missatge "no fa res" | Cap element del MIDI Remote actual hi respon |

### Raons per les quals un missatge és obsolet

Un missatge obsolet pot derivar de tres situacions:

1. **ELEMENT ELIMINAT** — L'element que el feia servir va ser eliminat completament
2. **ELEMENT DESACTIVAT** — L'element existeix però està desactivat i no escolta res
3. **ELEMENT ACTIU AMB BINDING CANVIAT** — L'element segueix actiu però escolta un binding MIDI diferent (el binding antic ha quedat orfe)

### Àmbit de vigilancia

> ⚠️ **CRÍTIC**: COC vigila el **PORT D'ENTRADA** de Cubase (missatges que arriben cap a Cubase), **MAI** el de sortida.
> 
> El feedback que Cubase envia cap a fora (p. ex., cap al Stream Deck) no es vigila mai perquè no entra: només surt.

### Comportament quan es detecta un missatge obsolet

Quan un missatge obsolet arriba a un dels ports d'entrada vigilats, COC executa aquesta seqüència:

1. **Crea** un fitxer `.txt` al Desktop
2. **Mostra** una notificació de macOS
3. **Obre** el fitxer automàticament a TextEdit

El fitxer documenta:
- El missatge rebut (canal, CC, valor)
- Timestamp amb milisegons
- Rol del control
- Estat del control
- Origen del mapping
- Informació addicional (si existeix)
- Footer explicatiu

---

## Motivació: Per què existeix COC

### Context: Migració de Cubase 14 a Cubase 15

En migrar de **Cubase 14** a **Cubase 15**, es van unificar tres sistemes de mapatge MIDI en un únic script JavaScript de MIDI Remote API:

| Sistema antic | Format | Estat a C15 |
|---|---|---|
| Dispositivo Generico (Generic Remote) | `.xml` (Legacy des de C12) | Reemplaçat |
| MIDI Remote Editor (MIDI Remote) | `.json` | Reemplaçat |
| MIDI Remote Manual (API) | `.js` | Eliminat |

Durant aquest procés de unificació:
- Molts missatges MIDI es van considerar obsolets per desús
- Alguns van ser eliminats directament
- Altres van ser reconfigurats amb un binding MIDI diferent

### El problema: Incertesa

**"Es creu que un missatge ja no s'usa" no és el mateix que "està confirmat que ja no s'usa".**

Una font del setup podria continuar emetent un d'aquests missatges sense que el desenvolupador se n'adoni:
- Una drecera del Stream Deck
- Una regla de BMTP
- Un trigger de BTT o Keyboard Maestro
- Un AppleScript
- Qualsevol altre mapatge MIDI antic

### La solució: COC com a "canari a la mina"

COC cobreix aquesta incertesa fent de **canari a la mina de carbó**:

```
Si COC no salta
  ↓
Confirmació que els missatges obsolets no s'emeten més

Si COC salta
  ↓
Avís que un missatge donat per mort s'està emetent dins del setup
  ↓
El fitxer .txt indica exactament quin és el missatge, d'on venia i què fer-hi
```

---

## Arquitectura: Dos Translators, Dos Ports

Els missatges obsolets poden arribar per **dos ports d'entrada MIDI** diferents de l'IAC Driver de macOS. Cada port té el seu propi translator a BMTP.

> 🔹 Els dos translators són **totalment independents** (mai es solapen), encara que un mateix número de canal+CC aparegui als dos.

### Translator 1: Port ACTIU 'IAC Everybody > CubaseMidi'

**Nom complet del translator a BMTP:**
```
COC - Cubase14_ObsoleteMidi_Canary 
  [TranslatorDetectorDeCCsEspecificsPelPortActiuDeMidiRemoteActual]
```

#### Característiques

| Aspecte | Descripció |
|---------|-----------|
| **Port** | `IAC Everybody > CubaseMidi` (ACTIU) |
| **Propòsit** | Port que fa servir el MIDI Remote **ACTUAL** (aquest script) |
| **Contingut** | Entra tant: missatges vius (controladors actius) + missatges obsolets |
| **Estratègia** | **FILTRATGE**: només documenta missatges obsolets **específics i coneguts** |
| **Raó del filtratge** | El port s'usa activament; COC NO pot documentar-ho tot |

#### Funcionament del filtratge

Les Rules del translator **filtren per canal + CC**:
- Si el parell `canal+CC` està a la **llista blanca**, executa la Outgoing Action
- Si **NO** està a la llista, atura el fluxe sense fer res

#### Missatges vigilats: 51 CCs en total

**Origen dels missatges:**
- `C14_XML`: Dispositivo Generico → 42 missatges
- `C14_JSON`: MIDI Remote JSON → 9 missatges

**Distribució per canal:**

| Canal | Nombre de CCs | CCs vigilats |
|-------|---|---|
| **1** | 43 | 35, 36, 39, 40, 41, 42, 43, 52, 53, 57, 60, 61, 62, 63, 64, 65, 66, 67, 68, 69, 78, 80, 84, 85, 88, 94, 97, 98, 105, 106, 107, 109, 110, 111, 112, 113, 115, 116, 117, 118, 119, 120, 121 |
| **11** | 2 | 120, 125 |
| **14** | 2 | 4, 5 |
| **15** | 4 | 1, 2, 3, 4 |
| **TOTAL** | **51** | — |

---

### Translator 2: Port OBSOLET 'IAC Everybody > CubaseMidiApi'

**Nom complet del translator a BMTP:**
```
COC - Cubase14_ObsoleteMidi_Canary 
  [TranslatorDetectorDeTotsElsCCsDelPortObsoletDelMidiRemoteJSEliminat]
```

#### Característiques

| Aspecte | Descripció |
|---------|-----------|
| **Port** | `IAC Everybody > CubaseMidiApi` (OBSOLET) |
| **Propòsit** | Port creat **exclusivament** per al MIDI Remote `.js` (C14_JS) |
| **Estat del port** | **MORT**: l'script JS va ser eliminat quan es va instal·lar Cubase 15 |
| **Expectativa** | Res ni ningu hauria d'enviar-hi res |
| **Realitat** | Qualsevol missatge que hi entri és, per definició, **sospitós** |
| **Estratègia** | **MONITORATGE TOTAL**: documenta **tots els CCs** que arriben |

#### Funcionament del monitoratge total

Les Rules del translator **NO filtren res**:
- Qualsevol CC que entri pel port crea sempre un fitxer `.txt`
- Si el CC està a la llista de 22 missatges documentats → mostra rol, estat i infoExtra
- Si el CC **NO** està a la llista → mostra "No documentat" (però crea el fitxer igual)

#### Missatges documentats: 22 CCs en total

**Origen dels missatges:**
- `C14_JS`: MIDI Remote JS (l'únic origen possible per aquest port)

**Distribució per canal:**

| Canal | Nombre de CCs | CCs vigilats |
|-------|---|---|
| **1** | 15 | 0, 1, 2, 3, 4, 5, 6, 7, 8, 11, 12, 13, 14, 15, 16 |
| **11** | 7 | 0, 11, 12, 13, 14, 15, 16 |
| **TOTAL** | **22** | — |

---

## Estructura de cada Translator a BMTP

Cada translator a la secció 'Global', preset 'Canary Hub' té **tres peces encadenades**:

### 1️⃣ INCOMING ACTION

**Funció:** Captura qualsevol CC que entri pels 16 canals del port

**Accions:**
- Desa el **canal** a la variable `qq`
- Desa el **número de CC** a la variable `xx`
- Desa el **valor** a la variable `vv`

```
CC arriba per MIDI
  ↓
Emmagatzema: qq (canal), xx (CC), vv (valor)
  ↓
Passa a RULES
```

### 2️⃣ RULES

**Funció:** Converteix el canal de format 0-based a 1-based i decideix si s'executa la Outgoing Action

**Conversió de canal:**
```javascript
qq = qq + 1  // Convierte 0-15 a 1-16
```

#### Lògica de RULES: Translator 1 (Filtratge)

```
qq (canal) + xx (CC) està a la llista blanca?
  ├─ SÍ → Executa Outgoing Action + Deixa traça a consola BMTP
  └─ NO → Atura el fluxe (no fa res) + Deixa traça a consola BMTP
```

#### Lògica de RULES: Translator 2 (Monitoratge total)

```
Sense filtratge
  ↓
Deixa traça a consola BMTP del CC rebut
  ↓
Executa SEMPRE automaticament la Outgoing Action
```

### 3️⃣ OUTGOING ACTION

**Funció:** Executa un AppleScript que crea el fitxer `.txt`

**Procediment:**

1. **Busca** el parell `canal+CC` rebut dins del mapa de missatges
   - Si el **troba** → agafa rol, estat, origen i infoExtra
   - Si **NO** el troba → assigna valors per defecte

2. **Escriu** el fitxer al Desktop amb:
   - Els camps extrets (o per defecte)
   - Canal, CC i valor rebuts
   - Timestamp amb milisegons
   - Footer explicatiu adaptat a l'origen

3. **Mostra** notificació de macOS

4. **Obre** el fitxer a TextEdit

---

## Camps del fitxer .txt i estats possibles

### Structure del fitxer .txt

Cada fitxer generat per COC contiene els següents camps al header:

```
DETECCIO DE MISSATGE MIDI OBSOLET
=================================

Canal:          [X]
CC:             [YY]
Valor:          [VVV]
Timestamp:      [YYYY-MM-DD HH:MM:SS.mmm]

Rol:            [rol del control]
Estat:          [estat actual]
Origen:         [sistema de mapatge antic]
Info Extra:     [detalls específics]

---
[FOOTER EXPLICATIU ADAPTAT A L'ORIGEN]
```

### Els 6 camps de cada fila del mapa

| Camp | Tipus | Descripció |
|------|-------|-----------|
| `canal` | Número | 1-16 |
| `cc` | Número | 0-127 |
| `rol` | Text | Descripció del control (p. ex., "Fader Transport") |
| `estat` | Enumeració | ELIMINAT \| DESACTIVAT (Ta/Tb/Ub) \| ACTIU AMB MIDI BINDING CANVIAT \| NO DOCUMENTAT |
| `origen` | Text | C14_XML \| C14_JSON \| C14_JS |
| `infoExtra` (elemExtraInfo) | Text lliure | Detalls específics de cada missatge |

### Els quatre estats possibles

#### 🔴 ELIMINAT

L'element **ja no existeix** al MIDI Remote actual.

- **Significat**: El seu missatge MIDI no està lligat a res
- **Acció recomanada**: Veure secció [Què fer quan COC salta](#què-fer-quan-coc-salta)

#### 🟡 DESACTIVAT (Ta / Tb / Ub)

L'element **existeix** al codi d'aquest script però **està desactivat** i es pot reactivar.

- **Lletra indicadora**: Indica a quina regió del script està el codi
  - `Ta` = Desactivat (codi comentat a zona **Transport Advanced**)
  - `Tb` = Desactivat (a l'array no usat **TB_CONTROLS_MAP_OBSOLETS_I_NO_USATS**, Transport Basic)
  - `Ub` = Desactivat (a l'array no usat **UB_CONTROLS_MAP_OBSOLETS_I_NO_USATS**, Utility Basic)

#### 🟢 ACTIU AMB MIDI BINDING CANVIAT

L'element **segueix actiu i funcional** al MIDI Remote actual, però respon a un binding **diferent** del que ha arribat.

- **Situació**: 
  - Element funciona correctament
  - Missatge que ha arribat = binding **ANTIC** (orfe)
  - `infoExtra` indica el binding MIDI que está viu i válido

#### ⚪ NO DOCUMENTAT

Estat assignat quan el missatge rebut **NO** està a la llista del mapa del translator.

- **Situació**: COC ha detectat un missatge que **no tenia catalogat**
- **Camp `rol`**: Mostra "No documentat"
- **Camp `estat`**: Mostra "No documentat"
- **Camp `infoExtra`**: No existeix
- **Resta de dades**: Es generen normalment (canal, CC, valor, timestamp, origen, footer)

### El camp ORIGEN

Indica de quin dels **tres mapatges MIDI antics** provenia l'element:

| Origen | Sistema | Format | Llegat |
|--------|---------|--------|--------|
| `C14_XML` | Dispositivo Generico (Generic Remote) | `.xml` | Legacy des de Cubase 12 |
| `C14_JSON` | MIDI Remote (Editor de MIDI Remote) | `.json` | Cubase 14 |
| `C14_JS` | MIDI Remote (Codi manual de API) | `.js` | Cubase 12-14 |

#### Comportament d'origen per translator

**Translator 1 (port CubaseMidi):**
- Vigila **DOS origens**: C14_XML i C14_JSON
- Ambdós mapatges enviaven per aquest port actiu
- Cada fila del mapa documenta quin origen prove l'element
- El footer del fitxer `.txt` s'adapta a l'origen

**Translator 2 (port CubaseMidiApi):**
- Vigila **UN SOL origen**: C14_JS
- Port usat exclusivamente per l'script JS
- AppleScript **fixa l'origen a C14_JS** abans fins i tot de mirar el mapa
- Fins i tot un CC no documentat que arribi per aquest port = C14_JS amb certesa

### El footer del fitxer .txt

El footer:
- Es redacta segons l'**origen** del missatge
- Explica en **llenguatge planer** per què s'ha creat el document
- Ofereix les **opcions per resoldre-ho**
- Referencia el document ampli de documentació

---

## Què fer quan COC salta

Quan aparegui un fitxer al Desktop, significa que un missatge donat per obsolet s'ha rebut. El propi fitxer `.txt` conté la guia detallada al footer.

### Procediment general

1. **Obrir el fitxer `.txt`** que ha creat COC
2. **Mirar el camp `ESTAT`**
3. **Decidir si el rol de l'element fa falta**
4. **Actuar segons l'estat i la necessitat**

### Els quatre escenaris d'actuació

#### ❌ L'element JA NO fa falta (sigui quin sigui l'estat)

**Accions:**
1. Buscar la **font que emet el missatge**
2. **Eliminar-la** o **editar-la** perquè deixi d'emetre'l

**Fonts possibles a revisar:**
- Dreceres del Stream Deck
- Regles de BMTP
- Triggers de BTT o Keyboard Maestro
- AppleScripts
- Qualsevol altre mapatge MIDI antic

**Nota sobre BMTP:**
- No cal tocar res de BMTP
- Deixant la vigilancia activa, COC seguirà protegint per si el missatge reapareix

---

#### ✅ L'element SI que fa falta i està DESACTIVAT

**Opcions de reactivació:**

**Opció A (Recomanada): Assignar un binding MIDI DIFERENT**

| Avantatge | Detall |
|-----------|--------|
| El missatge antic segueix vigilat | COC continua detectant si apareix |
| Detecta altres fonts mortes | Si alguna altra font el continua emetent |
| No cal tocar BMTP | Es queda tot com està |

**Procediment:**
- Descomentar el codi a l'script (si està a zona Ta)
- O treure'l de l'array d'obsolets (si està a Tb/Ub)
- Assignar-li un midi binding DIFERENT

**Opció B: Reutilitzar el mateix midi binding**

> ⚠️ **OBLIGATORI si es reutilitza el binding antic:**
> 
> S'ha d'eliminar aquest missatge de:
> 1. Les **RULES** del translator de BMTP corresponent
> 2. El **mapa** de l'AppleScript de la Outgoing Action
> 
> Raó: Ja no serà obsolet, així que COC no hauria de vigilar-lo més

**Procediment:**
- Reactivar l'element a l'script
- Assignar-li el **MATEIX midi binding** que ha arribat
- Anar a BMTP i fer els canvis anteriors

---

#### ✅ L'element SI que fa falta i està ELIMINAT

**Necessitat:** Crear-lo de nou al codi d'aquest script

**Opcions de creació:**

**Opció A (Recomanada): Assignar un binding MIDI DIFERENT**

| Avantatge | Detall |
|-----------|--------|
| El missatge antic segueix vigilat | COC continua detectant si apareix |
| Detecta altres fonts mortes | Si alguna altra font el continua emetent |
| No cal tocar BMTP | Es queda tot com està |

**Procediment:**
- Crear l'element de nou al codi d'aquest script
- Assignar-li un midi binding DIFERENT del que ha arribat

**Opció B: Reutilitzar el mateix midi binding**

> ⚠️ **OBLIGATORI si es reutilitza el binding antic:**
> 
> S'ha d'eliminar aquest missatge de:
> 1. Les **RULES** del translator de BMTP corresponent
> 2. El **mapa** de l'AppleScript de la Outgoing Action
> 
> Raó: Ja no serà obsolet, així que COC no hauria de vigilar-lo més

**Procediment:**
- Crear l'element de nou al codi d'aquest script
- Assignar-li el **MATEIX midi binding** que ha arribat
- Anar a BMTP i fer els canvis anteriors

---

#### ✅ L'element SI que fa falta i està ACTIU AMB MIDI BINDING CANVIAT

**Situació:** L'element ja funciona, però amb un binding MIDI diferent del que ha arribat.

**Informació útil:**
- `infoExtra` del fitxer indica **quin és el midi binding viu i valid actual**

**Acció:** Editar la **font que emet el missatge**

**Procediment:**
1. Buscar la font que continua emetent el binding obsolet (stream Deck, BTT, AppleScript, etc.)
2. Editar-la perquè emeti el **midi binding viu actual** en comptes del missatge obsolet

**Informació crítica:**
- El missatge antic **seguirà sent obsolet**
- S'ha de **MANTENIR vigilat a COC**
- **NO s'ha de tocar** ni les Rules ni el mapa del translator de BMTP
- Només cal editar la font de l'emissió

---

## Manteniment: Sincronització Manual

### ⚠️ CRÍTIC: No hi ha font de dades compartida

> **COC i aquest script NO comparteixen cap font de dades.**
> 
> La correspondència entre els botons d'aquest script i els missatges vigilats per COC és **totalment MANUAL**.

### Obligació de sincronització

Si en el futur es fa alguna de les següents accions relacionada amb un missatge vigilat:
- ✏️ **Activa** un element
- ❌ **Desactiva** un element
- ➕ **Crea** un element nou
- 🗑️ **Elimina** un element

**Cal actualitzar EN PARAL·LEL**, dins del translator de BMTP corresponent:

| Element a actualizar | Ubicació | Accció |
|---|---|---|
| **RULES** | Translator de BMTP | Afegir/eliminar la condició de canal+CC |
| **Mapa de l'AppleScript** | Outgoing Action | Afegir/eliminar la fila del mapa |

### Conseqüència de no sincronitzar

Si només es toca un dels dos llocs (script o BMTP), **COC quedarà incoherent**:

```
❌ Només es toca l'script (no BMTP)
   ↓
   COC segueix vigilant un element que ja no és obsolet
   ↓
   COC salta quan NO hauria de fer-ho

❌ Només es toca BMTP (no l'script)
   ↓
   COC deixa de vigilar un element que segueix sent obsolet
   ↓
   COC deixa de saltar quan hauria de fer-ho
```

### Backups de referència

Els backups de les Rules i dels AppleScripts dels dos translators es guarden com a fitxers de referència:

| Port | Fitxer de backup |
|------|---|
| CubaseMidi | `COC_Port_CubaseMidi_*.txt` |
| CubaseMidiApi | `COC_Port_CubaseMidiApi_*.txt` |

**Ús:** Serveixen com a referència per restaurar o auditar els translators.

---

## Documentació ampliada

Tota la informació ampliada del procés de migració dels tres mapatges MIDI de Cubase 14 a aquest únic MIDI Remote de Cubase 15, així com els fitxers de referència, enllaços i materials relacionats amb COC, es troba en:

```
~/KH/Varis/Processos boqod/
  Cubase - Migracio de varis dispositius midi de C14 a un unic Midi Remote a C15/
    READ ME.txt
```

**Nota:** Aquest directori és el que referencien els footers dels fitxers `.txt` que genera COC quan no se sap què fer davant d'un missatge obsolet detectat.

---

## Apèndix: Glossari de termes

| Terme | Definició |
|-------|-----------|
| **BMTP** | Bome MIDI Translator Pro — software de traducció i redirecció de MIDI |
| **CC** | Control Change — missatge MIDI estàndard (0-127) |
| **Binding MIDI** | Mapatge entre un control físic (CC) i una acció de software |
| **IAC Driver** | Inter-App Communication Driver — port MIDI virtual de macOS |
| **AppleScript** | Llenguatge de scripting per a automatització a macOS |
| **Dispositivo Generico** | Sistema legacy de mapatge MIDI de Cubase 12-14 (format `.xml`) |
| **MIDI Remote** | Sistema de mapatge MIDI basat en editor visual (format `.json`) |
| **MIDI Remote API** | Sistema de mapatge MIDI basat en codi JavaScript (format `.js`) |
| **Translator** | Regla individual de BMTP que processa missatges MIDI |
| **Obsolet** | Missatge MIDI que estava configurat però ja no s'usa |
| **Port d'entrada** | Connexió per la qual arriben missatges MIDI a Cubase |
| **Port de sortida** | Connexió per la qual surt feedback de Cubase |
| **Monitoratge total** | Vigilancia sense filtratge de tots els missatges |
| **Filtratge** | Vigilancia selectiva de missatges específics |

---

**Darrera actualització:** 2026  
**Sistema:** COC (Cubase14_ObsoleteMidi_Canary)  
**Versió de documentació:** 1.0
