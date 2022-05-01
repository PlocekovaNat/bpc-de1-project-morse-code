# Morse code receiver

### Členové týmu

* David Pěčonka (responsible for xxx)
* Natália Pločeková (responsible for xxx)
* Petra Slotová (responsible for xxx)
* Filip Stryk (responsible for xxx)

### Obsah

* [Project objectives](#objectives)
* [Hardware description](#hardware)
* [VHDL modules description and simulations](#modules)
* [TOP module description and simulations](#top)
* [Video](#video)
* [References](#references)

<a name="objectives"></a>

## Projekt

Write your text here.

<a name="hardware"></a>

## Popis HW

Write your text here.

<a name="modules"></a>

## Popis a simulace VHDL modulů

### btn_to_morse ([kód](morse-code-receiver/morse-code-receiver.srcs/sources_1/new/btn_to_morse.vhd))

Modul `btn_to_morse` slouží k zadání tečky nebo čárky pomocí jediného vstupního signálu `btn_i`. Tečka a čárka jsou rozlišeny podle doby, po kterou je signál ve vysoké úrovni. Časové intervaly pro tečku/čárku je možné nastavit pomocí hodnot `g_DOT_MIN`, `g_DOT_MAX`, `g_DASH_MIN`, `g_DASH_MAX`, kde každá hodnota značí počet cyklů interního hodinového signálu s frekvencí 25 ms, který je odvozen ze základního 100MHz hodinového signálu pomocí entity `clock_enable` s hodnotou `g_MAX` nastavenou na 2 500 000 cyklů.

Kromě již zmíněného vstupu `btn_i` obsahuje entita dále vstupy `clk` a `rst` pro hodinový signál a reset. Výstupy pak jsou `dot_o` a `dash_o`, které slouží jako hlavní výstup, a `led_dot_o` a `led_dash_o`, které slouží k indikaci toho, jaký symbol by byl zadán, kdyby v danou chvíli přešel vstupní signál do logické nuly. Posledním výstupem je `cnt_o` s aktuální hodnotou interního čítače.

Entita funguje jako stavový automat se 4 stavy - `INIT`, `PUSHED`, `DOT` a `DASH`. Výchozím stavem je `INIT`, ve kterém jsou oba výstupy `dot_o` a `dash_o` nulové, stejně jako hodnota interního čítače `s_cnt`. Ze stavu `INIT` je možný přechod pouze do stavu `PUSHED`, a to pokud má `btn_i` hodnotu logické jedničky. V tomto stavu dochází k pravidelné inkrementaci čítače `s_cnt`, který tak měří dobu trvání vysoké úrovně. Ze stavu `PUSHED` je při změně `btn_i` na 0 možný přechod jak zpět do stavu `INIT`, tak i do stavů `DOT` a `DASH`. Rozhodujícím faktorem je hodnota `s_cnt`. Pokud se nachází v jednom z intervalů pro tečku/čárku, tak automat přejde do stavu `DOT`, resp. `DASH`. V případě, kdy je hodnota mimo oba intervaly, přejde automat zpět do výchozího stavu. Ve stavech `DOT` a `DASH` je pak aktivní odpovídající výstup `dot_o`, resp. `dash_o`. Z těchto dvou stavů je možný přechod pouze do výchozí stavu při vysoké úrovni `btn_i`.

#### Diagram stavového automatu
![btn_to_morse fsm diagram](images/btn_to_morse_diagram.png)

#### Průběhy signálů při simulaci
![btn_to_morse waveforms](images/tb/btn_to_morse.png)

#### Průběhy signálů při simulaci (detail resetu)
![btn_to_morse waveforms](images/tb/btn_to_morse_reset_detail.png)

<a name="edge_detector"></a>

### edge_detector ([kód](morse-code-receiver/morse-code-receiver.srcs/sources_1/new/edge_detector.vhd))

Tato entity slouží k detekci náběžné a sestupné hrany signálu jako náhrada za funkce `rising_edge` a `falling_edge`, které by se měly používat pouze pro hodinové signály.

Vstupem entity je hodinový signál `clk` a signál, na kterém se detekují hrany `sig_i`. Výtupy pak jsou `rise_o` a `fall_o`, které jsou aktivní při detekování náběžné, resp. sestupné hrany. Při každé náběžné hraně hodinového signálu je do interních signálů uložena aktuální a předcházející hodnota `sig_i`. Jednoduchými logickými [výrazy](morse-code-receiver/morse-code-receiver.srcs/sources_1/new/edge_detector.vhd#L32-L33) je pak detekována změna z nízké na vysokou úroveň a naopak.

#### Průběhy signálů při simulaci
![edge_detector waveforms](images/tb/edge_detector.png)


### morse_to_bin ([kód](morse-code-receiver/morse-code-receiver.srcs/sources_1/new/morse_to_bin.vhd))

Entita `morse_to_bin` tvoří jádro celého projektu. Jejím úkolem je převést vstupní sekvenci teček a čárek, které reprezentují znaky v Morseově abecedě, na [ASCII](https://en.wikipedia.org/wiki/ASCII) kód.

Vystupy entity jsou opět `clk` a `rst` pro hodinový signál a reset, `dot_i` a `dash_i` pro zadání tečky, resp. čárky, a `enter_i`, který slouží pro potvrzení zadávání znaku. Výstupem pak je osmibitový vektor `bin_o` s ASCII kódem zadaného znaku, pětibitový (maximální počet symbolů pro jeden znak v Morseově abecedě je 5) vektor `morse_o` s aktuální zadanou sekvencí, ve které je tečka reprezentována nulou a čárka jedničkou. Posledním výstupem je `shift_o`, který slouží k indikaci přeložení nového znaku pro následující blok - posuvný registr.

Hlavní část entity tvoří synchronní proces `p_morse_to_bin` spouštěný při náběžné hraně hodinového signálu. Pokud je aktivní vstup `rst`, tak dojde k vynulování zadané sekvence (`s_morse_local`) a její délky (`s_len`). V [opačném případě](morse-code-receiver/morse-code-receiver.srcs/sources_1/new/morse_to_bin.vhd#L65-L180) dojde při náběžné hraně vstupu `s_dot`/`s_dash` k přidání dalšího symbolu do sekvence a k inkrementaci signálu s její aktuální délkou. K detekci náběžných hran je použit [detektor hran](#edge_detector). Při náběžné  hraně vstupu `s_enter` je přeložena sekvence uložená v `s_morse_local` na ASCII kód odpovídající danému znaku z množiny `A-Z` a `0-9`. Zárověň je také vyslán jeden impulz (pouze při zadání platného znaku) na výstupu `s_shift` a jsou vynulovány signály `s_morse_local` a `s_cnt`.

#### ASCII tabulka
![ascii table](images/ASCII.png)

#### Morseova abaceda
![morse code](images/morse_code.png)

#### Průběhy signálů při simulaci (A-I)
![morse_to_bin waveforms](images/tb/morse_to_bin_A-I.png)

#### Průběhy signálů při simulaci (I-R)
![morse_to_bin waveforms](images/tb/morse_to_bin_I-R.png)

#### Průběhy signálů při simulaci (R-Z)
![morse_to_bin waveforms](images/tb/morse_to_bin_R-Z.png)

#### Průběhy signálů při simulaci (Z-6)
![morse_to_bin waveforms](images/tb/morse_to_bin_Z-6.png)

#### Průběhy signálů při simulaci (3-9)
![morse_to_bin waveforms](images/tb/morse_to_bin_3-9.png)

#### Průběhy signálů při simulaci (reset)
![morse_to_bin waveforms](images/tb/morse_to_bin_reset.png)

#### Průběhy signálů při simulaci (zadání neplatného znaku - není vyslán impulz na výstupu `shift_o`)
![morse_to_bin waveforms](images/tb/morse_to_bin_invalid.png)

<a name="top"></a>

## Popis a simulace TOP modulu

Write your text here.

<a name="video"></a>

## Video

Write your text here

<a name="references"></a>

## References

1. Write your text here.