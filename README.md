# vamp

En adaptiv loopmotor för piano. Fyra ackord, fyra takter, och en motor som
håller spelaren precis på gränsen av vad hon klarar — utan att musiken någonsin
stannar för att motorn tänker.

Hela prototypen är **en fil, noll beroenden**. Öppna `index.html` i en webbläsare,
anslut ett MIDI-keyboard och tryck *Spela*. Utan keyboard finns demot i panelen
*Utan keyboard*.

```
git clone https://github.com/paokarlsson/vamp.git
open vamp/index.html     # eller: python3 -m http.server, och surfa dit
```

### Utveckla med hot reload

```
docker compose up
```

Surfa till <http://localhost:3000>. Repot monteras in i containern och
webbläsaren laddas om automatiskt när du sparar `index.html`. Inget att
bygga och ingen Dockerfile — bara `node`-imagen och browser-sync via `npx`.

**Spela/Paus** (eller mellanslag) fryser ljudklockan, så att musik, fallande noter
och motor står still på exakt samma ställe tills du fortsätter. MIDI kräver Web
MIDI (Chrome, Edge, Firefox) och https eller `localhost`.

Spelaren i demot är **simulerad**. Reglaget sätter hennes sanna nivå; motorn ser
den aldrig, utan måste hitta den ur utfallet. Hon har dessutom dolda svagheter
(tvåhandsspel, synkopering, svarta tangenter) som motorn ska upptäcka utan att
någon berättar om dem.

---

## Idén

De flesta övningsappar ställer en fråga, väntar på svar, visar en ruta och
fortsätter. Vamp gör tvärtom: kompet rullar hela tiden, och all anpassning sker
mellan två varv, i övergången, utan avbrott. Nivåhöjningar firas inte,
nivåsänkningar syns inte alls — nästa varv blir bara något som redan sitter i
handen.

Tre påståenden som hela konstruktionen vilar på:

1. **Svårighet är en egenskap hos tonerna, inte hos etiketten.** Motorn prissätter
   material den aldrig sett genom att mäta det. Därför kan biblioteket växa utan
   att motorn byggs om.
2. **Vilan är produkten.** Normaltillståndet efter placeringen är att *inte* höja.
   Att bo på rätt nivå är målet, inte pausen mellan framstegen.
3. **Musiken får aldrig stanna.** Nästa varv beslutas och schemaläggs medan det
   nuvarande fortfarande låter.

---

## Arkitektur

Tolv moduler i `index.html`, i den ordning de bygger på varandra:

| # | Modul | Ansvar |
|---|---|---|
| 1 | Teori | Romerska siffror → tonhöjdsklasser. `bVII`, `vi`, `V7`, `iio`. |
| 2 | Röstföring | Ackord → grepp, via billigaste vägen genom hela progressionen. |
| 3 | Bibliotek | 16 progressioner som text, 5 tonarter. |
| 4 | Texturer | 14 texturer: grepp → faktiska noter, uttryckt i slag. |
| 5 | Svårighet | Noter → formvektor (12 mått) → ett tal. |
| 6 | Spelare | Simulerad hand med dolda styrkor och svagheter. |
| 7 | Motor | Skattning, nio-axlig profil, val av nästa varv. |
| 8 | Ljud | Web Audio. Piano, padda, bas, trummor — inga samplingar. |
| 9 | Varvet | Schemaläggning, beslutsfönster, mjuka landningar. |
| 10 | MIDI-in | Riktiga anslag matchade mot väntade toner. |
| 11 | Bild | Fallande noter och klaviatur på canvas. |
| 12 | Gränssnitt | Spela/paus, demo, paneler, logg. |

### Röstföring är det som gör biblioteket billigt

En progression skrivs som en rad text:

```js
{ id:"andalus", name:"Andalusisk", mood:"dramatisk", mode:"minor",
  numerals:["i","VII","VI","V"] }
```

Ingen arrangerar om något per tonart. Röstföringsmotorn söker via dynamisk
programmering den billigaste vägen genom progressionen, där kostnaden är hur
långt rösterna måste flytta sig — och **stänger loopen** genom att koppla sista
greppet tillbaka till det första. Det är därför handen ligger still och det
låter arrangerat i stället för hoppigt. Varje ny progression kostar en rad.

### Svårighet räknas ur noterna

`shapeOf()` mäter de genererade noterna: anslag per sekund, andel anslag som
faktiskt ändrar något, samtidighet, medelintervall per hand, positionsbyten per
takt, bredaste grepp, handoberoende (0 = en hand, 1 = två i samma rytm, 2 = två i
olika rytm), synkopering, andel svarta tangenter, finaste underdelning.

En detalj som visar hållningen: en åttondel på "och" räknas som *underdelning*,
inte som synkopering. Synkopering börjar först när slaget före är tomt.

`costOf()` viktar ihop formvektorn till ett tal, och tempot multipliceras in med
exponent 1,25. En variant = progression × tonart × textur × tempo × taktart ×
swing, vilket ger 336 varianter per progression och tonart — alla med sin egen
mätta svårighet.

**Vikterna i `W` är handsatta gissningar.** De ska skattas ur riktig speldata
tillsammans med spelarförmågan, som i vilken Elo-liknande modell som helst.
Allt annat i arkitekturen bär oavsett vilka vikterna blir.

### Skattningen är hierarkisk

En allmän nivå plus en avvikelse per axel, där avvikelsen krymps mot noll tills
det finns observationer bakom den. Det är enda sättet att bära nio axlar utan
att drunkna i brus — en blek stapel i profilpanelen betyder "för lite data",
och då lånas den allmänna nivån.

Motorn kör tre faser:

* **Placering** — fyra korta prov i stigande svårighet, samma tonart och tempo.
  Avbryts vid första riktiga misslyckandet; ingen ska plöja igenom något som
  uppenbart är för svårt. Startar sedan **ett steg under** skattningen: ett bra
  första varv avgör om någon stannar kvar, och fel nedåt är billigt.
* **Zoom** — målnivån följer skattningen tätt medan osäkerheten krymper
  (σ ← 0,91 σ per varv).
* **Groove** — motorn bor på nivån. Tre rena varv i rad *och* samlad timing
  (< 42 ms spridning) krävs för +0,45. Under 80 % träffar sänks nivån tyst och
  nästa varv blir en **safe harbour** — ett varv hon redan äger.

Var nionde varv smyger motorn in ett **tyst prov** ~1,1 nivåer över målet. Går
det bra vet den mer; går det dåligt loggas ingenting dramatiskt, och nästa varv
landar mjukt.

Två skyddsmekanismer värda att känna till:

* En axel som spelaren visat sig svag på får inte dra ner den *allmänna*
  skattningen — negativa signaler dämpas med upp till 55 % när svag axel belastas.
* Motorn staplar aldrig två kända svaga axlar i samma varv.

### Beslutsfönstret

`commitLoop()` körs senast en takt före sista tonen i varvet, och räknar bara på
de anslag som redan passerat. Beslutet appliceras på nästa etta. Därför finns
ingen lucka mellan varven — och därför är utvärderingen alltid gjord på
verkligt spelat material, aldrig på en gissning om resten av varvet.

Runt det ligger tre musikaliska knep: **andrum** (trummorna ut ett varv),
**förhandsvisning** (appen spelar vänsterhanden själv ett varv innan spelaren
får ta över den) och lager som byggs upp padda → hi-hat → bas → bastrumma →
virvel allteftersom det går bra.

### MIDI-in: ett anslag letar upp sin ton

Med en riktig spelare är ingenting avgjort i förväg. Varje väntad ton är en
miss tills ett anslag hittar den: samma tonklass, inom ett tidsfönster på
högst 200 ms (smalare vid högt tempo), närmast i tid. Oktaven spelar ingen roll,
så att ett litet keyboard räcker. Ett anslag som inte hittar någon ton är en
felton och kostar en halv miss, eftersom ett grepp med en ton för mycket inte
är samma sak som en ton som aldrig kom.

Tiden mäts mot det som **hörs**, inte mot det som schemalagts
(`getOutputTimestamp()`). Annars skulle ljudkortets fördröjning få varje
spelare att verka släpa. Samma klocka styr bilden, så de fallande noterna når
klaviaturen när tonen faktiskt låter.

Spelarens egna toner går genom appens piano (*Mina toner i appen*). Stäng av
det om keyboardet har egna högtalare.

### Missar låter, de tystnar inte

En missad ton spelas som närmaste skalton, svagare och kortare. Den passerar som
en dissonans i stället för som ett hål. Progressionen roterar dessutom på sin
egen, långsammare klocka — ny harmonik är stor musikalisk variation till nästan
ingen motorisk kostnad, och ska inte konkurrera om samma
en-axel-i-taget-budget som texturen.

---

## Vad som är mätt och vad som är gissat

| | |
|---|---|
| **Riktig** | Röstföring, formvektor, skattningsmatematiken, vallogiken, ljudet, bilden. |
| **Simulerad** (i demot) | Spelaren, anslaget, timingfelen, kaskaden efter en miss. |
| **Gissad** | Alla vikter i `W`, belastningskurvorna i `loadOf()`, trösklarna i motorn, tidsfönstret och feltonskostnaden i MIDI-matchningen. |

## Nästa steg

* Kalibrera `W` och axelbelastningarna mot riktig speldata i stället för mot magkänsla.
* Kalibrera tidsfönstret och feltonskostnaden mot riktiga spelare.
* Skatta spelarförmåga och materialsvårighet i samma modell, inte var för sig.
* Bredda biblioteket — det är, som sagt, en rad text per progression.

## Publicering

Varje push till `main` publicerar sajten via GitHub Pages
(`.github/workflows/pages.yml`). Ingen byggkedja: reporoten laddas upp som den
är, så `index.html` hamnar på rotadressen.

> **Engångssteg innan första deployen går igenom:** slå på Pages under
> *Settings → Pages* och välj **Source: GitHub Actions**. Workflowen kan inte
> göra det åt sig själv — `GITHUB_TOKEN` har `pages: write`, men att *skapa*
> Pages-sajten kräver admin. Kör sedan om workflowen från Actions-fliken
> (den har `workflow_dispatch`).

Sajten hamnar på `https://paokarlsson.github.io/vamp/`.

## Filer

```
index.html                    hela prototypen: teori, motor, ljud, MIDI, bild, gränssnitt
compose.yaml                  utvecklingsserver med live reload (docker compose up)
.github/workflows/pages.yml   deploy till GitHub Pages vid push till main
README.md                     den här filen
```
