# kb

Kunskapsbas: data som motorn kan luta sig mot. Används inte av `index.html` än.

## `ackordtrad_c_dur_4ackord.json`

Ackordträd för C-dur, fyra ackord djupt. Varje nivå visar vilka ackord som
följer efter vägen dit, med sannolikhet — t.ex. `tree.C.next.F.next.C.next`
svarar på "vad kommer efter C → F → C?".

- **Källa:** McGill Billboard Project 2.0 (CC0), Burgoyne, Wild & Fujinaga,
  ISMIR 2011. Nyckelrelativ version från
  [corpusmusic/bb-cluster](https://github.com/corpusmusic/bb-cluster).
- **Urval:** 677 låtar, 1958–1991.
- **Regler:**
  - Alla låtar transponerade till C; bara ackord ur C-durskalan
    (C, Dm, Em, F, G, Am, Bdim).
  - Sjuackord och utvidgningar räknas som grundackordet (G7 = G).
  - Upprepningar slås ihop (C C G = C G). En följd bryts vid ackord utanför skalan.
  - `probability` = `count` / summan av `count` på samma nivå. `count` = antal
    ackordbyten, `songs` = antal olika låtar.
  - `next` finns bara om minst 3 olika låtar går den vägen, och högst 4 nivåer ner.
  - Toppnivån visar hur vanligt varje ackord är totalt.

Varje nod har formen:

```json
{ "probability": 0.4525, "count": 1649, "songs": 356, "next": { "C": { … } } }
```
