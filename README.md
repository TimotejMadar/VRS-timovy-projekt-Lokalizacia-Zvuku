# VRS-timovy-projekt-Lokalizacia-Zvuku

# Určovanie smeru zvuku v rozsahu 360° pomocou štyroch mikrofónov (STM32F303K8)

Zariadenie odhaduje smer prichádzajúceho zvuku v **celom kruhu 0° až 360°** pomocou **štyroch mikrofónov** vo vrcholoch štvorca. Mikrokontrolér **STM32F303K8** (doska NUCLEO-F303K8) vzorkuje mikrofóny dvoma ADC v simultánnom režime. Z krížových korelácií signálov určí časové rozdiely príchodu zvuku (TDOA), vypočíta z nich uhol a zobrazí ho na **OLED displeji** ako ručičku kompasu spolu s číselnou hodnotou.

Dva mikrofóny dokážu určiť smer len v rozsahu −90° až +90°, pretože nerozlíšia zvuk spredu a zozadu. Štyri mikrofóny túto nejednoznačnosť odstránia a dávajú rovnakú presnosť v celom kruhu.

---

## Hardvér

### Mikrofónová zostava

- Štyri moduly MAX4466 vo vrcholoch **štvorca so stranou 10 cm**, v rovnakej výške a natočené nahor.
- **Rovnaké zosilnenie všetkých modulov** (trimer nastaviť rovnako a overiť meraním).
- **Anti-aliasingový filter** na každom kanáli: RC dolná priepusť **1,5 kΩ + 4,7 nF (C0G)**, f₍c₎ ≈ 22,6 kHz. Všetky filtre musia mať rovnaké hodnoty.
- Vodiče od mikrofónov čo najkratšie, ideálne tienené.

### Zapojenie

| Signál | Pin MCU | Pin Nucleo | Periféria |
|---|---|---|---|
| Mikrofón M1 | PA0 | A0 | ADC1_IN1 |
| Mikrofón M3 | PA1 | A1 | ADC1_IN2 |
| Mikrofón M2 | PA4 | A3 | ADC2_IN1 |
| Mikrofón M4 | PA7 | A6 | ADC2_IN4 |
| OLED SCL / SDA | PB6 / PB7 | D5 / D4 | I²C1 |
| Výstup do PC | PA2 / PA15 | (ST-LINK VCP) | USART2 |

### Zoznam potrebných súčiastok

| # | Súčiastka | Typ | Počet |
|---|---|---|---|
| 1 | Vývojová doska | NUCLEO-F303K8 | 1 |
| 2 | Mikrofónový modul so zosilňovačom | MAX4466 | 4 |
| 3 | OLED displej | SSD1306, 0,96", 128×64, I²C | 1 |
| 4 | Rezistor | 1,5 kΩ | 4 |
| 5 | Kondenzátor (filter) | 4,7 nF, keramický C0G/NP0 | 4 |
| 6 | Blokovacie kondenzátory | 100 nF + 10 µF pri každom mikrofóne | 4 + 4 |
| 7 | Tienený kábel | 2-žilový audio kábel | ~2 m |
| 8 | Držiak mikrofónov | štvorec 10 cm (3D tlač / preglejka) | 1 |

---

## Firmvér

Vývojové prostredie: **STM32CubeIDE** (HAL + LL), jazyk C.

### Snímanie

1. Časovač (TIM, 72 MHz / 720 = 100 kHz) generuje TRGO, ktorý spúšťa ADC1.
2. ADC1 a ADC2 sú v režime **dual regular simultaneous**, každý so sekvenciou dvoch kanálov: najprv sa súčasne navzorkujú **M1 a M2**, hneď potom **M3 a M4**. Malé konštantné oneskorenie druhej dvojice sa vo výpočte odčíta.
3. DMA v kruhovom režime kopíruje vzorky do bufferu s 1024 pozíciami (`uint32_t`, 4 kB).
4. Prerušenie **half-transfer / transfer-complete** oznámi hotovú polovicu bufferu (256 vzoriek z každého mikrofónu), ktorá sa spracuje, kým sa druhá polovica plní.

### Spracovanie bloku

1. Rozdelenie dát na kanály M1 … M4, odčítanie strednej hodnoty (DC), prevod na `int16_t`.
2. **Detekcia zvuku:** ak je energia bloku pod prahom, blok sa preskočí a displej ukáže „ticho“.
3. **Korelácia** pre všetkých 6 dvojíc mikrofónov (k = −44 … +44) a jej pripočítanie do akumulátorov.
4. Po 8 blokoch nájdenie maxima každej korelácie a parabolická interpolácia, čím sa získajú časové rozdiely.
5. **Výpočet uhla** zo všetkých časových rozdielov naraz pomocou `atan2`, výsledok v rozsahu 0° … 360°.
6. **Vyhladenie** výsledku (priemer z posledných 5 odhadov), aby ručička neskákala.

### Displej

- Ovládač SSD1306 cez I²C1 (400 kHz) s prenosom cez **DMA**, aby kreslenie neblokovalo spracovanie zvuku.
- Obrazovka: kruhová stupnica (kompas) s ručičkou, uhol v stupňoch, stĺpec s úrovňou signálu, stav (meranie / ticho).
- Obnovovanie ≈ 10× za sekundu.
- Odhady sa zároveň posielajú cez UART do PC na vyhodnotenie.

---

## Vyhodnotenie presnosti

### Plán merania

| Parameter | Hodnoty |
|---|---|
| Smer zdroja | 0° až 345° po 15° (24 polôh) |
| Vzdialenosť zdroja | 0,5 m / 1 m / 2 m |
| Hluk | bez hluku / SNR ≈ 20 dB / 10 dB / 0 dB |
| Typ signálu | biely šum (1 s), tlesknutie, reč |
| Opakovania | 20 odhadov v každom bode |

### Metriky

- priemerná chyba (systematická odchýlka) a **RMSE** pre každý uhol,
- smerodajná odchýlka (rozptyl odhadov),
- percento odhadov s chybou do ±5° a do ±10°,
- percento zámen zvuku spredu a zozadu.

---

## Členovia tímu a rozdelenie úloh

| Člen | Zodpovednosť | Konkrétne úlohy |
|---|---|---|
| **Timotej Maďar** | Hardvér a snímanie signálu | stavba mikrofónovej zostavy, zapojenie mikrofónov a filtrov, nastavenie zosilnenia, konfigurácia časovača, ADC a DMA | integrácia firmvéru |
| **Sven Schmitz** | Algoritmus odhadu smeru | spracovanie signálu, korelácia, výpočet a vyhladenie uhla, overenie algoritmu v simulácii | meranie presnosti |
| **Tomáš Slavkov** | Displej a vyhodnotenie | ovládač displeja a grafika, posielanie dát do PC, skript na vyhodnotenie meraní a grafy | dokumentácia a prezentácia |

Trello: https://trello.com/b/QT1B5NMn/vrs-timovy-projekt-lokaliz%C3%A1cia-zvuku
