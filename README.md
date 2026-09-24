# vamp

En adaptiv loopmotor för piano. Fyra ackord, fyra takter, och en motor som
håller spelaren precis på gränsen av vad hon klarar — utan att musiken någonsin
stannar för att motorn tänker.

Hela prototypen är **en fil, noll beroenden**. Öppna `index.html` i en webbläsare,
anslut ett MIDI-keyboard och tryck *Spela*. Utan MIDI går det att spela på datorns
tangentbord.

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

**Datorns tangentbord** spelar som i en tracker: nedre bokstavsraden `Z`–`M` är
en oktav från C3 (svarta tangenter på `S D G H J`), övre raden `Q`–`P` fortsätter
från C4 (svarta på `2 3 5 6 7 9 0`). Utan MIDI-keyboard står bokstaven på varje
tangent i bilden. Många tangentbord registrerar bara två eller tre tangenter
samtidigt i vissa kombinationer, så större grepp kan tappa toner.

**Klaviaturen i bilden följer ditt keyboard.** Web MIDI säger inte hur många
tangenter det har, så appen räknar fram det i tre steg, där varje steg går före
det förra:

1. **Namnet.** "Keystation 49" eller "Launchkey 25" ger en standardstorlek
   (25, 32, 37, 49, 61, 76 eller 88).
2. **Det du spelar.** Klaviaturen växer till närmaste standardstorlek som rymmer
   lägsta och högsta ton du spelat, också efter en oktavknapp.
3. **Mät keyboardet.** Tryck på lägsta och högsta tangenten, så är storleken exakt.

Det inlärda och det mätta sparas per enhetsnamn. Toner utanför keyboardet flyttas
i hela oktaver in på det. När vita tangenter annars skulle bli smalare än 16 px
bryter scenen sig ur kolumnen, upp till hela fönsterbredden. Ryms det ändå inte
(88 tangenter på en telefon) blir tangenterna smalare, och delen som varvet
använder markeras.

**Helskärm** (knappen längst ned till höger i scenen, eller `F`) visar bara
noterna och klaviaturen, med spela/paus i raden under. Esc eller `F` igen
lämnar. På iPhone går bara video i helskärm, där syns ingen knapp.

**Skärmen tänd** (bredvid Spela) hindrar skärmen från att slockna medan du
spelar, via Screen Wake Lock. Webbläsaren släpper låset när fliken döljs; appen
tar det igen när du kommer tillbaka. Valet sparas, och knappen syns bara i
webbläsare som stöder det.

### Grundton, skala och spelsätt

Tre val under Spela-knappen. *Mix* betyder att motorn väljer själv, som förut.
Valen är filter på det motorn väljer bland: anpassningen fungerar som vanligt,
bara inom det man valt. De sparas i webbläsaren och kan ändras medan musiken
går. Då gäller de från nästa varv som inte redan är lagt.

* **Grundton**: vilken som helst av de tolv. Svarta tangenter mäts redan som
  svårighet, så en tonart med många förtecken syns i nivån.
* **Skala**: dur, moll, harmonisk moll, dorisk, mixolydisk, pentatonisk dur
  och moll, blues. Skalan väljer progressionerna (dorisk i–IV, mixolydisk
  I–bVII, harmonisk moll med V som durackord) och avgör vart en miss faller.
  Pentatonerna lånar dur- och mollprogressionerna, eftersom ackorden under
  är desamma.
* **Spelsätt**: *Mix*, *Ackord och rytm* eller *Brutna ackord*. Varje
  spelsätt har egna placeringsprov.

**Ackord och rytm är en trappa.** Stegen kommer i ordning, och tre bra varv
låser upp nästa. Två varv under 80 % i rad tar en tillbaka, tyst, och ett
under 60 % räcker.

| Steg | Vänster | Höger |
|---|---|---|
| 1 Blockackord | — | Blockackord |
| 2 Bas och grundton | Bas, hel not | Grundtonen, hel not |
| 3 Bas och ackord | Bas, hel not | Blockackord |
| 4 Öppet grepp | Bas, hel not | Mittentonen upp en oktav: C-E-G blir C-G-E' |
| 5 Växelvis | Bas på de tunga slagen | De två övre tonerna på de lätta |
| 6 Växelvis med driv | Kvinten på trean | Föregriper nästa ackord på sista "och" |
| 7 Delat grepp i en hand | Bas, hel not | Undre tonen på tunga slag, de övre på lätta |

Ordningen följer den uppmätta svårigheten. Att dela greppet i en hand mäts
som svårast, eftersom handen måste flytta sig en decima varje slag. Steg 2
finns för att andra handen annars kommer in med ett helt ackord på en gång,
det största hoppet i trappan: där får den i stället en ton i varje hand. Ett nytt
steg kommer i det tempo målnivån räcker till, så det börjar lugnt.

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
| 3 | Bibliotek | 25 progressioner som text, åtta skalor, alla tolv grundtoner. |
| 4 | Texturer | 21 texturer: grepp → faktiska noter, uttryckt i slag. Tre spelsätt. |
| 5 | Svårighet | Noter → formvektor (12 mått) → ett tal. |
| 6 | Axlar | Vad ett varv belastar, axel för axel. |
| 7 | Motor | Skattning, nio-axlig profil, val av nästa varv. |
| 8 | Ljud | Web Audio. Piano, padda, bas, trummor — inga samplingar. |
| 9 | Varvet | Schemaläggning, beslutsfönster, mjuka landningar. |
| 10 | MIDI-in | Riktiga anslag, från MIDI eller datorns tangentbord, matchade mot väntade toner. |
| 11 | Bild | Fallande noter och klaviatur på canvas. |
| 12 | Gränssnitt | Spela/paus, paneler, logg. |

### Röstföring är det som gör biblioteket billigt

En progression skrivs som en rad text:

```js
{ id:"andalus", name:"Andalusisk", mood:"dramatisk", scale:"harmonic",
  numerals:["i","VII","VI","V"] }
```

Ingen arrangerar om något per tonart. Röstföringsmotorn söker via dynamisk
programmering den billigaste vägen genom progressionen, där kostnaden är hur
långt rösterna måste flytta sig — och **stänger loopen** genom att koppla sista
greppet tillbaka till det första. Det är därför handen ligger still och det
låter arrangerat i stället för hoppigt. Varje ny progression kostar en rad.

Basen röstförs också som en sluten loop, men med **en oktav per grundton**: samma
ackord ligger på samma ton varje gång det kommer i varvet. En kortaste väg ger
inte det — Canon fick C på både C2 och C3. Registret 36–52 ger högst två lägen
per grundton, så alla kombinationer prövas.

**Basgången** har grundton på ettan, kvint och oktav däremellan, och slaget
före ett ackordbyte leder in i nästa grundton med ett halvtonssteg, från det
håll basen redan står. Tre lägen: lugn när basen just kommit in, driv med
åttondelar när hela kompet är inne, och walking (grundton, ters, kvint, ledton)
när varvet går i swing. Basen är knäppt: filtret öppnar i anslaget och stänger
sig, med en sinus en oktav under som botten.

Paddan och basen följer **ackorden, inte takterna**. Canon har två ackord per
takt och får två anslag per takt; ett ackord som varar två takter slås an igen
vid taktstrecket.

### Svårighet räknas ur noterna

`shapeOf()` mäter de genererade noterna: anslag per sekund, andel anslag som
faktiskt ändrar något, samtidighet, medelintervall per hand, positionsbyten per
takt, bredaste grepp, handoberoende (0 = en hand, 1 = två i samma rytm, 2 = två i
olika rytm), synkopering, andel svarta tangenter, finaste underdelning.

En detalj som visar hållningen: en åttondel på "och" räknas som *underdelning*,
inte som synkopering. Synkopering börjar först när slaget före är tomt.

`costOf()` viktar ihop formvektorn till ett tal, och tempot multipliceras in med
exponent 1,25. En variant = progression × tonart × textur × tempo × taktart ×
swing, vilket ger 588 varianter per progression och tonart — alla med sin egen
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
  uppenbart är för svårt. En enda miss är inget riktigt misslyckande — provens
  första tre takter har ibland bara tre toner — så det krävs två. Med riktig
  spelare föregås första provet av en takts inräkning, annars når första tonen
  tangenterna innan den hunnit synas. Ett prov räknas som klarat först vid
  90 % — 82 % är ett prov man kämpat sig igenom, inte ett man äger.
  Skattningen läggs strax över det senast klarade, och första varvet **ett steg
  under** den: ett bra första varv avgör om någon stannar kvar, och fel nedåt
  är billigt.
* **Zoom** — målnivån följer skattningen tätt medan osäkerheten krymper
  (σ ← 0,91 σ per varv). Ett felfritt varv säger bara att nivån räcker, inte var
  gränsen går: det driver skattningen uppåt men krymper inte σ. Den som spelar
  felfritt fortsätter alltså uppåt tills gränsen syns.
* **Groove** — motorn bor på nivån. Tre rena varv i rad *och* samlad timing
  (< 60 ms spridning, som datorns tangentbord också klarar) krävs för +0,45 —
  eller mer, om skattningen sprungit ifrån under de rena varven, men högst
  +0,8 åt gången. Under 80 % träffar sänks nivån tyst och nästa varv blir en
  **safe harbour** — ett varv hon redan äger, och som ligger under det nya
  målet.

**Sänkningen går efter det som spelades.** Ett varv under 80 % säger att
gränsen ligger under det varvet, så nästa mål läggs under dess nivå, mer ju
fler missar: 0,3 + 2,5 × (0,8 − träffar). Är två av de tre senaste varven
dåliga följer skattningen med ned direkt, annars drar den upp målet igen.
I simuleringen, där spelaren tappar två nivåer mitt i, tog det tidigare
12–15 varv att hitta tillbaka, och nu tar det ungefär 5.

Var nionde varv smyger motorn in ett **tyst prov** ~1,1 nivåer över målet. Går
det bra (≥ 85 %) är målet för lågt och höjs; går det dåligt loggas ingenting
dramatiskt, och nästa varv landar mjukt.

Två skyddsmekanismer värda att känna till:

* En axel som spelaren visat sig svag på får inte dra ner den *allmänna*
  skattningen — negativa signaler dämpas med upp till 55 % när svag axel belastas.
* Motorn staplar aldrig två kända svaga axlar i samma varv.

### Paddan slås an på slaget

Paddan svällde förut upp exponentiellt under en halv sekund. En sådan kurva
ligger nära noll större delen av tiden, så paddan hördes först 350 ms efter
ettan och lät som att den kom in för sent. Nu har den en rak attack på 20 ms,
sjunker sedan mjukt till en jämn nivå och klingar ut en liten bit in i nästa
ackord, så att inget glapp uppstår.

### Ljudet slår aldrig i taket

Kompet ensamt når nästan fullt utslag, och spelarens egna toner läggs ovanpå.
Därför går allt genom lägre master, kompressor, en hård begränsare och sist en
mjuk klippning som är rak upp till 0,7. Spelarens toner klingar av som en
pianoton även om tangenten hålls, och tystnar helt efter tio sekunder, så ett
tappat note-off lämnar ingen ton kvar som låter för alltid.

### Beslutsfönstret

`commitLoop()` körs senast en takt före sista tonen i varvet, och räknar bara på
de anslag som redan passerat. Beslutet appliceras på nästa etta. Därför finns
ingen lucka mellan varven — och därför är utvärderingen alltid gjord på
verkligt spelat material, aldrig på en gissning om resten av varvet.

Runt det ligger tre musikaliska knep: **andrum** (trummorna ut ett varv),
**förhandsvisning** (appen spelar vänsterhanden själv ett varv innan spelaren
får ta över den) och lager som byggs upp padda → hi-hat → bas → bastrumma →
virvel, ett per bra varv (≥ 90 %) från första provet. Fullt komp efter fyra bra
varv; ett dåligt varv tar inget ut.

### MIDI-in: ett anslag letar upp sin ton

Med en riktig spelare är ingenting avgjort i förväg. Varje väntad ton är en
miss tills ett anslag hittar den: samma tonklass, inom ett tidsfönster på
högst 200 ms (smalare vid högt tempo), närmast i tid. Oktaven spelar ingen roll,
så att ett litet keyboard räcker. Ett anslag som inte hittar någon ton är en
felton och kostar en halv miss, eftersom ett grepp med en ton för mycket inte
är samma sak som en ton som aldrig kom.

Tiden mäts mot det som **hörs**, inte mot det som schemalagts. Annars skulle
ljudkortets fördröjning få varje spelare att verka släpa. Klockan är
`ctx.currentTime` minus fördröjningen ut till högtalaren, hållen som en jämn
förskjutning mot `performance.now()`: den glider mot rätt värde, högst 3 ms per
bildruta, och står still under paus. Webbläsarens rapporterade fördröjning
(`outputLatency`, som även `getOutputTimestamp()` bygger på) kan hoppa hundratals
millisekunder mellan två bildrutor, och följd rakt av får det bilden att flimra.
Samma klocka styr bilden och tidsstämplar anslagen, så de fallande noterna når
klaviaturen när tonen faktiskt låter, och tidsspridningen som motorn mäter är
spelarens, inte ljudkortets.

**Fördröjning som ingen rapporterar lär sig appen.** Webbläsaren vet ofta
inte hur lång tid ljudet tar genom Bluetooth-hörlurar eller en extern
ljudenhet, och keyboardet har sin egen. Allt sådant syns som att varje anslag
kommer lika mycket för sent. Efter varje varv tas medianen av avvikelserna,
och tidsfönstret flyttas dit; kvar blir spridningen, som är spelarens. Långt
ifrån tas stora steg, så en okänd fördröjning hittas på ett varv eller två.
För att en stor fördröjning alls ska synas räknas närmaste ton upp till 350 ms
bort, också utanför fönstret. Flyttades fördröjningen mycket under ett
placeringsprov görs provet om — det mätte klockan, inte spelaren. Värdet sparas
per keyboard och visas bredvid det, tillsammans med den fördröjning
webbläsaren själv rapporterar (*ljud ut*). Priset: den som jämnt släpar efter
tolkas som fördröjning, inte som ett timingfel.

I ett test i webbläsaren med en spelare som alltid slår an 280 ms efter
tonen blev förut varje varv 0 %. Nu görs första provet om och resten går
igenom.

Spelarens egna toner går genom appens piano (*Mina toner i appen*). De kommer
alltid så mycket efter tangenten som ljudet ut tar, och det går inte att
räkna bort. Har keyboardet egna högtalare, stäng av det: då hörs tonen direkt.
Appen föreslår det vid start när ljudet ut är över 40 ms, och valet sparas
per keyboard.

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
