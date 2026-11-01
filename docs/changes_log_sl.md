# Dnevnik sprememb: kaj smo spremenili in kaj je to prineslo

*Zajema 30. 10. 2026: zvezke `04_poi`, `05_streets`, `06_baseline_clustering`, `07_radius_counts`.*
*Prejšnje delo (01–03) je razloženo v `progress_explained_sl.md`.*
*(Angleška različica: `changes_log.md`.)*

Vsak vnos: **sprememba → zakaj → rezultat**. Številke so iz zagonov 30. 10. 2026.

---

## A. Odločitve o tem, katere podatke uporabimo

### A1. Urban Atlas opuščen, za »funkcijo« uporabimo OSM POI
- **Zakaj:** razredi Urban Atlasa so vnaprej določene kategorije rabe prostora. Raziskovalno vprašanje išče tipe, ki se pojavijo *brez vnaprej določenih oznak*, gruče na podlagi Urban Atlasa pa bi lahko le ponovile njegove razrede.
  (Opomba: z njim metoda ne bi postala »nadzorovana«; deleži razredov bi bili le vhodne spremenljivke. Argument je, da ne vgradimo tujih kategorij.)
- **Rezultat:** funkcijo opisujejo števila točk interesa (POI) (04_poi). Mentoričin prvotni seznam je vseboval Urban Atlas, zato je to treba pokazati Špeli.

### A2. Velikost gospodinjstva opuščena
- **Zakaj:** je ni v SURS datoteki, ki jo imamo; za osnovni model ni nujna.
- **Rezultat:** sloj »ljudje« ima en kazalnik za gručenje (gostota prebivalstva). Starostna deleža (0–14, 65+) se uporabljata samo za profiliranje, nikoli kot vhod (pravilo krožnosti).

### A3. Dostopnost JPP izpuščena iz osnovnega modela
- **Zakaj:** prekriva se s prebivalstvom, POI in gostoto ulic, večinoma pa je gradient center–rob, ki bi lahko prevladal v gručah. Po načelu »en sloj naenkrat« jo lahko dodamo kasneje kot samostojen sloj in izmerimo njen učinek (ablacija).
- **Rezultat:** prenos GTFS zaenkrat ni potreben.

### A4. Relief (DEM) prestavljen na kasneje
- **Zakaj:** topografija je samo kontrolna spremenljivka pri vrednotenju z LST, nikoli vhod.
- **Rezultat:** potrebujemo ga šele pri koraku LST; za mrežo 100 m zadošča 30 m DEM (Copernicus GLO-30).

---

## B. Pridobivanje podatkov OSM

### B1. Spletni prenos (osmnx / Overpass API) → ni uspel
- **Kaj se je zgodilo:** strežnik je najprej zavrnil mejo MOL kot poizvedbeni poligon (413, preveč oglišč). Po poenostavitvi sta bila glavni strežnik in njegovo zrcalo preobremenjena (502 / časovne omejitve).
- **Rezultat:** opuščeno; nič ni bilo shranjeno.

### B2. Ročni prenos z Geofabrika (shapefile za Slovenijo, 30. 10. 2026)
- **Zakaj:** ena datoteka, brez poizvedb na strežnik, ponovljivo (datum zapisan v `data/raw/OSM/PREBERI.txt`).
- **Rezultat, s stranskimi učinki, ki so oblikovali kodo:**
  - POI že imajo tip (`fclass`), zato so kategorije preprosta preslikava.
  - Brezplačni izvleček **nima pisarn** → delovna mesta so v `poi_services` podzastopana.
  - Ceste nimajo podtipa `service` in oznake za pločnike → parkirnih prehodov ne moremo ločiti od dovoznih cest, pločnikov pa ne od peš poti (glej C2).

---

## C. Izdelava kazalnikov

### C1. Kategorije POI (04_poi)
- **Sprememba:** 6 kategorij (hrana, trgovina, izobraževanje, zdravstvo, storitve, prosti čas). Izpuščeno: urbana oprema (klopi, koši, kamere), spomeniki/umetnine, parki, igrišča za šport in otroška igrišča (zelenje je svoj sloj; na poljih je na stotine igrišč), postaje souporabe avtomobilov/koles (mobilnost ni v osnovnem modelu).
- **Rezultat:** obdržanih 3.181 od 10.988 objektov OSM; 3.172 znotraj MOL. Celice z največ POI: staro mestno jedro in nakupovalni centri (Šiška, Rudnik).
  **Ugotovljena težava:** samo 1.523 od 27.505 celic (5,5 %) ima vsaj en POI. → vodi do D5.

### C2. Ulično omrežje (05_streets)
- **Sprememba 1:** gozdne/poljske poti izključene → **rezultat:** gozdovi ne izgledajo, kot da imajo ulice.
- **Sprememba 2:** ceste `service` izključene → **rezultat:** ni na tisoče lažnih križišč/slepih ulic na parkiriščih. Stranski učinek: ulica, ki se nadaljuje kot servisna cesta, šteje kot slepa ulica.
- **Sprememba 3:** peš in kolesarske poti se ne uporabljajo za križišča, samo za `str_path_share` → **rezultat:** prehodi za pešce in ločeno narisani pločniki v centru ne ustvarjajo lažnih križišč.
- **Sprememba 4:** vozlišča križišč, oddaljena manj kot 10 m, združena → **rezultat:** 5.033 vozlišč → 4.677 križišč (eno križišče šteje enkrat). 1.952 slepih ulic.

### C3. Popravek napake: izgubljena križišča
- **Kaj se je zgodilo:** po združevanju bližnjih vozlišč so imela vsa križišča enak indeks, korak »odstrani podvojena ujemanja« pa je skoraj vsa izbrisal (največ 1 križišče na celico).
- **Popravek:** ponastavitev indeksa.
- **Rezultat:** največ 13 križišč na celico; celica z največ križišči je staro mestno jedro (mediana odseka 24 m), kot pričakovano.

### C4. Sesutje jedra Jupyter
- **Kaj se je zgodilo:** jedro `python3` kliče golo ukazno vrstico `python`, ki je brez aktiviranega okolja sistemski Python → jedro se je sesulo.
- **Popravek:** Jupyter / nbconvert zaganjamo z aktiviranim okoljem `urban`.

---

## D. Osnovno gručenje

### D1. Izbira 6 kazalnikov, po en na sloj
- **Zakaj:** z vsemi ~26 kazalniki bi sloji z veliko stolpci (POI 8, stavbe 6) prevladali nad sloji z enim (prebivalstvo).
  Standardizacija uravnoteži *kazalnike*, ne *slojev*.
- **Kako:** Spearmanove korelacije med 8.207 celicami s stavbami (čez vse celice številne prazne celice napihnejo vse korelacije).

| sloj | izbrano | ni izbrano, ker |
|---|---|---|
| ljudje | `pop` | – |
| površina tal | `imp_share` | – |
| stavbe | `bld_height` | pozidanost in FAR ponavljata neprepustnost (ρ 0,81 / 0,80); višina ne (0,59) |
| zelenje | `can_cover` | NDVI je skoraj zrcalna slika neprepustnosti (ρ −0,91); krošnje so skoraj neodvisne (−0,30) in ločijo drevesa od trave/polj |
| funkcija | `poi_count` | raznolikost je prazna v 94 % celic |
| ulice | `str_intersections` | dolžina ulic se z njo prekriva (0,68) in bolj korelira z drugimi sloji |

- **Rezultat:** noben par med izbranimi 6 nima |ρ| nad približno 0,6.

### D2. Katere celice: možnost A (samo obseg podatkov o stavbah)
- **Zakaj:** EUBUCCO pokriva naselje Ljubljana, ne celotne MOL; zunaj tega višina stavb ni znana (ni pa nič).
- **Rezultat:** 16.369 celic − 18 vodnih celic (`ndvi_neg_share > 0.5`) = **16.351 celic** z 290.960 od 300.443 prebivalcev.
  885 poseljenih robnih celic (9.481 prebivalcev) je izpuščenih → Špelo prosimo za izvleček za MOL.

### D3. Transformacije
- **Celica študentskih domov omejena** na drugo največjo vrednost prebivalstva → ostane »zelo gosta«, ne tvori pa svoje gruče.
- **log(1 + x)** za prebivalstvo, POI, križišča → redke ekstremne celice ne stisnejo več vseh običajnih celic skupaj.
- **z-vrednost** za vseh 6 → enaka teža v razdalji k-means (sicer odloča kazalnik z največjimi številkami).

### D4. Izbira k (06)

| k | silhueta | Davies–Bouldin | ARI med semeni, povprečje (min) | najmanjša gruča |
|---|---|---|---|---|
| 4 | 0,44 | 1,10 | 0,68 (0,42) | 1.379 |
| 5 | 0,47 | 0,98 | 0,77 (0,41) | 1.270 |
| **6** | **0,49** | **0,95** | **0,95 (0,80)** | 710 |
| 7 | 0,50 | 1,05 | 0,89 (0,80) | 398 |
| 9 | 0,51 | 1,06 | 0,87 (0,76) | 171 |

- **Rezultat:** **k = 6**: najboljša stabilnost med semeni in najboljši Davies–Bouldin; silhueta potem le počasi raste, gruče pa postajajo majhne.

### D5. Rezultat osnovnega modela (06, k = 6)
| gruča | branje |
|---|---|
| 0 | gozd, gozdnati griči |
| 1 | odprt prostor: polja, travniki, Barje |
| 2 | stanovanjsko, malo križišč v celici |
| 3 | stanovanjsko, križišča v celici |
| 4 | nestanovanjske pozidane površine: industrija, BTC, logistika |
| 5 | center in mešana raba |

- **Ugotovljena težava:** gruči 2 in 3 se razlikujeta skoraj samo po tem, ali v 100 m celici slučajno leži križišče (večinoma 0 ali 1) → vzorec »sol in poper« na stanovanjskih območjih.

---

## E. Štetje v radiju (07)

### E1. POI in križišča štejemo v radiju 300 m namesto znotraj celice
- **Sprememba:** za vsako celico seštejemo POI in križišča v vseh celicah, katerih središče je oddaljeno največ 300 m (29 celic), in delimo s številom celic mreže v tem oknu → na hektar v soseski. Postopek je sicer enak, enak k = 6.
- **Rezultat:**

| | osnovni model (v celici) | radij 300 m |
|---|---|---|
| celice z vsaj enim POI | 1.523 | 9.352 |
| celice z vsaj enim križiščem | 2.921 | 14.924 |
| ujemanje s sosedi (»sol in poper«; višje = bolj strnjeno) | 0,63 | **0,71** |
| ARI glede na osnovni model | – | 0,81 |
| silhueta, k = 6 | 0,49 | 0,42 (ni primerljivo, glej spodaj) |
| ARI med semeni, k = 6 | 0,95 | 0,86 |
| najbolj stabilen k | 6 | **5** (ARI 0,997) |

- **Kaj se je spremenilo v gručah:**
  - Gozd (ostane 99 % celic), odprt prostor (94 %) in nestanovanjske površine (88 %) se malo spremenijo.
  - Delitev stanovanjskih območij zdaj nekaj pomeni: **stanovanjsko nizke gostote** (mediana 20 prebivalcev/ha, višina 7,6 m) proti **gostemu mestnemu stanovanjskemu** (65 prebivalcev/ha, 9,5 m, trgovine in križišča v bližini; 182.000 prebivalcev). Stara delitev po slučajnih križiščih se pomeša v obe.
  - Gruča centra se skrči s 710 na 458 celic in postane strnjena (staro mestno jedro + nekaj sekundarnih središč).
- **Opozorila:**
  - Silhueta ni primerljiva med različnimi nabori kazalnikov: štetja v celici so bila večinoma točne ničle, zaradi česar gruče izgledajo umetno kompaktne.
  - k = 5 združi obe stanovanjski gruči. Če radijska različica postane osnovni model, je treba k izbrati znova (stabilnost proti smiselni delitvi).

---

## F. Občutljivost na radij (08)

### F1. Enak postopek z 200 m in 500 m
- **Sprememba:** podatki ostanejo enaki; 07 ponovljen z radijem 200 m (okno 13 celic) in 500 m (81 celic), za primerjavo še različica v celici. Izhod: `radius_sensitivity.csv`, `08_scores_by_radius.png`, `08_maps_by_radius.png`.
- **Rezultat:**

| | v celici | 200 m | 300 m | 500 m |
|---|---|---|---|---|
| ujemanje s sosedi, k = 5 | 0,63 | 0,74 | 0,75 | 0,75 |
| ujemanje s sosedi, k = 6 | 0,63 | 0,68 | 0,71 | 0,73 |
| ARI med semeni, k = 5 | 0,77 | **0,999** | **0,997** | 0,89 |
| ARI med semeni, k = 6 | 0,95 | 0,92 | 0,86 | 0,89 |
| najbolj stabilen k | 6 | 5 | 5 | 8 |
| ARI glede na 300 m, k = 5 | 0,75 | 0,93 | – | 0,92 |
| ARI glede na 300 m, k = 6 | 0,81 | 0,81 | – | 0,91 |

- **Kaj to pomeni:**
  - Vsak radij odstrani večino vzorca »sol in poper«; nad 300 m je izboljšanje majhno (0,71 → 0,73 pri k = 6, pri k = 5 nič).
  - Pri k = 5 radij skoraj ni pomemben (ARI 0,88–0,93 med vsemi radiji).
  - Pri k = 6 dasta 300 m in 500 m enako strukturo: stanovanjsko nizke gostote (20–23 prebivalcev/ha) proti gostemu mestnemu stanovanjskemu (65–72 prebivalcev/ha). Gruča centra se z radijem krči (458 → 270 celic).
  - Pri 200 m je delitev pri k = 6 drugačna: majhna redko poseljena obrobna gruča (7 prebivalcev/ha, 26.000 prebivalcev) proti eni veliki stanovanjski gruči (197.000) in večji center (617 celic). 200 m je premajhna soseska, da bi ločila oba stanovanjska tipa.
- **Odločitev (predlog):** obdržati radij **300 m**: rezultati se ujemajo s 500 m, k = 5 je zelo stabilen, center ostane strnjen in se ne zgladi. Izbira k (5 ali 6) ostaja odprta.

---

## G. Podatki o stavbah za celotno MOL (02a, ponovni zagon 02 in 06–08)

### G1. EUBUCCO, obrezan na MOL, namesto Špelinega izvlečka za naselje
- **Sprememba:** nov zvezek `02a_eubucco_mol_extract`: datoteka EUBUCCO v0.2 za zahodno Slovenijo (`SI04.gpkg`, 524.000 stavb, prenesena 1. 11. 2026), obrezana na mejo MOL (`intersects`, stavbe na meji ostanejo cele) → `data/processed/eubucco_mol.gpkg`, stolpci preimenovani v Špeline kratke nazive. 02 zdaj bere to datoteko.
- **Preverjanje:** 82.495 stavb (+7.435); 75.060 od Špelinih 75.061 je vključenih s 100 % enakimi atributi (manjkajoča leži 4 m zunaj MOL). Na starem območju ima 99,8 % celic enake kazalnike stavb.
- **Rezultat:** celice s podatki o stavbah 16.369 → **27.483** od 27.505; poseljene celice brez podatkov o stavbah 885 → 3 (15 prebivalcev).
- **Opozorilo:** zunaj naselja so tlorisi pogosteje iz Microsofta (13 % proti 8 %), višine pa pogosteje ocenjene (29 % proti 20 %).
- Preverjeni ostali sloji: vsi rastri (neprepustnost, NDVI, višina krošenj) pokrivajo 100 % MOL; POI in ulice so iz izvlečka OSM za vso Slovenijo.
- Stari izhodi shranjeni kot `*_settlement.csv` v `data/processed/`.

### G2. Ponovni zagon 06–08 na 27.465 celicah (prej 16.351)

| | prej (naselje) | zdaj (MOL) |
|---|---|---|
| razvrščene celice | 16.351 | 27.465 |
| ARI med semeni k = 6, v celici / 300 m | 0,95 / 0,86 | 0,94 / 0,94 |
| najbolj stabilen k, v celici / 300 m | 6 / 5 | 9 / **4** |
| ujemanje s sosedi k = 6, v celici / 300 m | 0,63 / 0,71 | 0,69 / 0,73 |
| ARI 300 m proti 200 m / 500 m, k = 6 | 0,81 / 0,91 | 0,95 / 0,91 |

- **Istih šest gruč** v obeh različicah: gozd (zdaj tudi gozdnati griči na vzhodu), odprt prostor, dve stanovanjski, center/mešana raba, nestanovanjske površine.
- **Delitev stanovanjskih območij spremeni pomen** (300 m, k = 6): z dodanimi vasmi vzhodne MOL je zdaj **redko poseljeno / vasi** (7 prebivalcev/ha, 34.000 prebivalcev) proti **mestnemu stanovanjskemu** (53/ha, 204.000), namesto nizke (20/ha) proti visoki gostoti (65/ha). Vsi trije radiji se zdaj ujemajo v tej strukturi.
- **Številke gruč so se spremenile** (urejene so po povprečni neprepustnosti): v 06 center = 4, nestanovanjsko = 5; v 07 (300 m) nestanovanjsko = 4, center = 5.
- 300 m ostaja smiseln radij. **Izbira k je spet odprta:** k = 4 (ARI 0,999, ena stanovanjska gruča) ali k = 6 (ARI 0,94, delitev stanovanjskih območij, ločen center).

---

## H. Prispevek slojev (09)

### H1. Osnovni model določen: kazalniki v radiju 300 m, k = 6 (odločeno 1. 11. 2026)

### H2. Odstranjevanje enega kazalnika naenkrat
- **Metoda:** ponovno razvrščanje brez vsakega kazalnika (k = 6, isto seme), primerjava s polnim razvrščanjem z ARI na vseh celicah in samo na pozidanih celicah (polne gruče 2–5), ker gozd in odprt prostor predstavljata 72 % celic in prevladujeta v ARI. Šum semena: ARI 0,994 (vse), 0,983 (pozidano).
- **Rezultat:**

| odstranjen sloj | ARI vse | ARI pozidano | glavni učinek |
|---|---|---|---|
| zelenje | 0,45 | 0,47 | gozd in odprt prostor se združita |
| prebivalci | 0,85 | 0,49 | nestanovanjsko se združi z mestnim stanovanjskim |
| stavbe (višina) | 0,89 | 0,49 | vasi se delno pridružijo odprtemu prostoru |
| neprepustnost | 0,91 | 0,48 | zmerno premešanje, nobena gruča ne izgine |
| funkcija (POI) | 0,93 | 0,70 | center razpade |
| ulice | 0,93 | 0,71 | najmanjša sprememba |

- **Sklep:** znotraj pozidanega območja so štirje sloji oblike (prebivalci, stavbe, neprepustnost, zelenje) približno enako pomembni; noben ni odveč. POI je edini sloj, ki določa center. Ulice prinesejo najmanj nove informacije.
- **Glede k:** 6 → 5 razdeli redko poseljeno/vasi med mestno stanovanjsko in odprt prostor; 6 → 7 razdeli mestno stanovanjsko in center (ARI 0,86 v obeh primerih). Redko poseljena gruča je najmanj stabilna (prehodno območje).

---

## Odprta vprašanja
1. ~~Radijska različica kot osnovni model, k~~ → odločeno: 300 m, k = 6 (H1).
2. Za Špelo: zamenjava Urban Atlas → POI, izbira kazalnikov, karta k = 6; prositi za sloj UHI.
