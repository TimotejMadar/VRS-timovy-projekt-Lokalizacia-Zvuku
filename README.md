# VRS-timovy-projekt-Lokalizacia-Zvuku

# Určovanie smeru zvuku pomocou dvoch mikrofónov (STM32F303K8)

Zariadenie odhaduje smer prichádzajúceho zvuku v rozsahu **−90° až +90°** pomocou dvoch mikrofónov. Mikrokontrolér **STM32F303K8** (doska NUCLEO-F303K8) vzorkuje oba mikrofóny **presne v rovnakom okamihu** (dva ADC v simultánnom režime). Z krížovej korelácie signálov určí časový rozdiel príchodu zvuku (TDOA) a vypočítaný uhol zobrazí na **OLED displeji** ako ručičku na polkruhovej stupnici spolu s číselnou hodnotou.

---

### Mikrofónová zostava

- Dva moduly MAX4466 upevnené na lište (3D tlač alebo drevená lišta) tak, aby **stredy membrán boli presne 10 cm od seba**, v rovnakej výške a natočené rovnakým smerom.
- **Zosilnenie oboch modulov rovnaké** (trimer nastaviť na rovnakú polohu a overiť meraním), aby mali kanály podobnú úroveň.
- **Anti-aliasingový filter** na každom kanáli: RC dolná priepusť **1,5 kΩ + 4,7 nF (C0G)**, f₍c₎ ≈ 22,6 kHz. Oba filtre musia byť z rovnakých hodnôt, aby nevnášali rozdielny fázový posun.
- Vodiče od mikrofónov čo najkratšie, ideálne tienené.

### Zoznam potrebných súčiastok

| # | Súčiastka | Typ | Počet |
|---|---|---|---|
| 1 | Vývojová doska | NUCLEO-F303K8 | 1 |
| 2 | Mikrofónový modul so zosilňovačom | MAX4466 (nastaviteľný zisk) | 2 |
| 3 | OLED displej | SSD1306, 0,96", 128×64, I²C | 1 |
| 4 | Rezistor | 1,5 kΩ | 2 |
| 5 | Kondenzátor (filter) | 4,7 nF, keramický C0G/NP0 | 2 |
| 6 | Blokovacie kondenzátory | 100 nF + 10 µF pri každom mikrofóne | 2 + 2 |
| 7 | Tienený kábel | 2-žilový audio kábel | ~1 m |


---

## Firmvér

Vývojové prostredie: **STM32CubeIDE** (HAL + LL), jazyk C.

### Snímanie

1. Časovač (TIM, 72 MHz / 720 = 100 kHz) generuje TRGO, ktorý spúšťa ADC1.
2. ADC1 a ADC2 sú v režime **dual regular simultaneous**. Oba kanály sa navzorkujú v tom istom okamihu a ich výsledky sa uložia spolu do 32-bitového registra `ADC12_CDR`.
3. DMA v kruhovom režime kopíruje dvojice vzoriek do bufferu s 1024 pozíciami (`uint32_t`, 4 kB).
4. Prerušenie **half-transfer / transfer-complete** oznámi hotovú polovicu (512 vzoriek), ktorá sa spracuje, kým sa druhá polovica plní.

### Spracovanie bloku

1. Rozdelenie dvojíc na kanály L a R, odčítanie strednej hodnoty (DC), prevod na `int16_t`.
2. **Detekcia zvuku:** výpočet energie bloku. Ak je pod prahom, blok sa preskočí a displej ukáže „ticho".
3. **Korelácia** pre k = −32 … +32 a jej pripočítanie do akumulátora.
4. Po 4 blokoch nájdenie maxima, parabolická interpolácia, výpočet `θ = arcsin(c·Δt/d)` s orezaním argumentu na ⟨−1, 1⟩.
5. **Vyhladenie** výsledku (medián z posledných 5 odhadov), aby ručička neskákala.

### Displej

- Ovládač SSD1306 cez I²C1 (400 kHz) s prenosom framebuffera cez **DMA**, aby kreslenie neblokovalo spracovanie zvuku.
- Obrazovka: polkruhová stupnica −90° … +90° s ručičkou, uhol v stupňoch, stĺpec s úrovňou signálu, stav (meranie / ticho).
- Obnovovanie ≈ 10× za sekundu.


## Vyhodnotenie presnosti

### Plán merania

| Parameter | Hodnoty |
|---|---|
| Smer zdroja | −90° až +90° po 15° (13 polôh) |
| Vzdialenosť zdroja | 0,5 m / 1 m / 2 m |
| Hluk | bez hluku / SNR ≈ 20 dB / 10 dB / 0 dB (zdroj hluku v stálej polohe) |
| Typ signálu | biely šum (1 s), tlesknutie, reč |
| Opakovania | 20 odhadov v každom bode |

### Metriky

- priemerná chyba (systematická odchýlka) a **RMSE** pre každý uhol,
- smerodajná odchýlka (rozptyl odhadov),
- percento odhadov s chybou do ±5° a do ±10°,

---

## Členovia tímu a rozdelenie úloh

| Člen | Technická časť projektu |
|---|---|
| Timotej Maďar | Snímanie signálu, Hardvérové zapojenie 
| Sven Schmitz | Algoritmus odhadu smeru 
| Tomáš Slavkov | Displej a vyhodnocovací softvér 

https://trello.com/b/QT1B5NMn/vrs-timovy-projekt-lokaliz%C3%A1cia-zvuku


