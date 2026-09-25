# wawmar-erp-overlay-dist
Rozszerzenie ERP z widokiem uproszczonym
# WawMar ERP – widok uproszczony (rozszerzenie przeglądarki)

Nakładka na kartę zamówienia. **Nic nie wysyła do serwera** – zmienia wyłącznie
wygląd strony u Ciebie w przeglądarce. Kolumny są *ukrywane* (`display:none`),
nigdy usuwane, więc wszystkie pola formularza dalej istnieją i normalnie się
zapisują. Wyłączenie widoku przywraca DOM co do znaku.

To ostatnie jest utrzymywane celowo i sprawdzane po każdej zmianie: atrybuty
robocze (`data-wm-span`, `data-wm-rspan`) są kasowane przy powrocie, a klasy i
zmienne CSS ustawiane przez `setClass()` / `setVar()`, które ruszają atrybut tylko
gdy naprawdę się zmienia i sprzątają puste `class=""` / `style=""`. Bez tego samo
`classList.remove()` dopisywało `class=""` do każdej z ~1000 komórek.

## Instalacja

### Brave / Chrome (zalecane)
1. `brave://extensions`
2. Włącz **Developer mode** (prawy górny róg)
3. **Load unpacked** → wskaż folder `erp-overlay`

Do instalacji na tej maszynie **zip nie jest potrzebny** – wskazujesz katalog.
`erp-overlay-brave.zip` (te same pliki) przydaje się tylko do przeniesienia wtyczki
na inny komputer: rozpakować i dalej „Load unpacked".

Zostaje zainstalowane na stałe, bez podpisywania. Po zmianie w kodzie klikasz
strzałkę „przeładuj” na kafelku rozszerzenia i odświeżasz kartę zamówienia.

### Firefox

**Nie ma osobnej wersji źródeł** – ten sam `content.js`, `overlay.css` i
`manifest.json` działają w obu przeglądarkach. Kod nie używa **żadnego** API
przeglądarki (`chrome.*` / `browser.*`), tylko DOM i `localStorage`, więc nie ma
czego portować. Dla Firefoksa pakujemy te pliki do `.xpi`:

```
python build.py            # -> erp-overlay-firefox.xpi + erp-overlay-brave.zip
python build.py --bump     # najpierw podbija wersję w manifeście, potem buduje
```

`.xpi` to zwykły ZIP, ale **`manifest.json` musi leżeć w korzeniu archiwum**.
`zip -r` z poziomu wyżej wrzuca wszystko do podkatalogu i Firefox taki plik
odrzuca – dlatego jest do tego skrypt, a nie instrukcja „spakuj folder”.

**`--bump` przy każdym wydaniu:** Firefox odmawia instalacji paczki o wersji nie
wyższej niż już zainstalowana. Przy zwykłym przebudowaniu w trakcie pracy podbijać
nie trzeba – stąd flaga, a nie automat.

**Instalacja na próbę (działa od razu, znika po restarcie):**
1. `about:debugging#/runtime/this-firefox`
2. **Load Temporary Add-on…** → wskaż `erp-overlay-firefox.xpi`

**Na stałe** – release Firefox instaluje tylko rozszerzenia **podpisane przez
Mozillę**. Dwie drogi:
- podpisać jako **„unlisted”** na addons.mozilla.org (za darmo, bez publikowania –
  dostajesz podpisany `.xpi` tylko dla siebie), albo
- użyć **Developer Edition / Nightly / ESR** i ustawić
  `xpinstall.signatures.required = false` w `about:config`. Ta droga wymaga, żeby
  rozszerzenie miało własne ID – jest ustawione
  (`browser_specific_settings.gecko.id`).

#### Dwa wpisy w manifeście, które muszą tam być

**`data_collection_permissions`** – bez tego **AMO odrzuca paczkę** przy weryfikacji
(„The 'data_collection_permissions' property is missing”). Wtyczka nie zbiera
żadnych danych, więc deklaracja brzmi:

```json
"data_collection_permissions": { "required": ["none"] }
```

To nie jest formalność do pominięcia – `required` musi być obecne, a `"none"` jest
prawdziwe: nic nie wychodzi poza przeglądarkę. Ustawienia siedzą w `localStorage`,
a „Szukaj” tylko otwiera adres w Twoim własnym ERP-ie, gdy sam klikniesz.

**`strict_min_version: "140.0"`** – `data_collection_permissions` działa od
**Firefoksa 140** (desktop). Niższa wartość nie ma sensu: albo klucz nie zadziała,
albo AMO i tak nie przepuści. Przy okazji załatwia to starszy problem – do FF 126
uprawnienia do hostów w MV3 **nie były nadawane przy instalacji** i wtyczka cicho
nic nie robiła; 140 jest wyraźnie powyżej tej granicy.

`build.py` sprawdza oba wpisy asercją i nie zbuduje paczek, jeśli któryś zniknie.

## Obsługa

Przycisk **Widok uproszczony** w prawym górnym rogu. Po włączeniu dochodzą:

- przycisk **Menu ▾** – odnośniki z karty, jedna kolumna z kategoriami,
- **kolor** wg grubości / wg grupy materiału / bez kolorów,
- **odstęp między grupami** (checkbox),
- **index bez kropek, WIELKIE** (checkbox).

Przycisk **Ilość ▾** pojawia się niezależnie od widoku – **tylko na karcie, którą da
się zapisać** (patrz „Tryb edycji" niżej).

Na samej górze paska jest **licznik do terminu wysyłki** – np. `Wyslac za 2 dni –
2026-09-25`. Bierze się z komórki **„Wysłać:"** w nagłówku karty. Czerwony po terminie
i w dniu terminu, bursztynowy na jeden–dwa dni przed, poza tym spokojny.
Gdy komórka jest pusta (zdarza się, np. karta 40106) – licznik w ogóle się nie pokazuje.

Przycisk **Materiał** pojawia się, gdy uda się sprawdzić stan magazynu (patrz
„Czy jest materiał" niżej). Przy braku materiału robi się czerwony i sam się rozwija.

Przycisk **Powiązane** jest zawsze widoczny i otwiera `/wrs/add` **w nowej karcie**.
Adres bierze się z `location.origin`, czyli z hosta, na którym akurat jesteś –
z firmowej sieci idzie na `http://192.168.0.3/wrs/add`, z zewnątrz na
`http://laser.wawmar.com/wrs/add`. Żadnego przełącznika ani konfiguracji.
Nowa karta, a nie to samo okno, bo inaczej niezapisane zmiany w karcie (np. po
zbiorczej zmianie ilości) przepadłyby bez ostrzeżenia.

Wszystkie ustawienia zapamiętują się w `localStorage` przeglądarki
(klucz `wmErpOverlay`).

## Co robi widok uproszczony

- Zostawia w tabeli głównej 18 kolumn: L.P., Cena w €, Gr., Gatunek, F,
  Index i rysunki, Waga netto, Waga brutto, Cena za 1 kg, Operacje, Ilość [szt],
  Czas laser, Ilość gięć, Czas prasa, Czas ślus, koop, Wartość materiału,
  Możliwości.
- Górną tabelkę (ramka czerwona) przenosi do osobnej tabeli nad układem – inaczej
  jej szerokość rozpychałaby wąską tabelę detali.
- Wszystko, co było pod tabelą główną (ramka zielona), ląduje w przyklejonym
  panelu po prawej.
- Menu odnośników **wyprowadza z karty do paska wtyczki** – przycisk „Menu ▾"
  rozwija jedną kolumnę z tymi samymi nagłówkami (Materiał / Produkcja / Sprzedaż /
  Zlecenie / Edycja). Z panelu po prawej menu znika, więc zostaje w nim miejsce na
  tabele. Odnośniki są **przenoszone, nie kopiowane** – te same `<a>` wracają na
  swoje miejsce po wyłączeniu widoku.
  Nagłówki sekcji dostają własny styl (wersaliki + linia), pozycje stałe wcięcie
  na ikonę – część odnośników ikony nie ma i teksty inaczej by się nie trzymały
  jednej linii. Ikona jest pozycjonowana absolutnie, więc zawinięta nazwa
  wyrównuje się do tekstu, a nie do ikony.
  Pasek jest `flex column` z `max-height: calc(100vh - 14px)` i `border-box`, a lista
  dostaje resztę wysokości i własny scroll – przy 39 odnośnikach nie wychodzi poza
  ekran. Rozwinięte menu przykrywa część panelu po prawej (to zwykły dropdown).
- **Ulubione odnośniki** (lista `FAVS`) dostają bursztynowe tło, ciemniejszy tekst,
  pasek po lewej i gwiazdkę. Pasek jest robiony `box-shadow: inset`, a nie
  `border-left`, więc teksty ulubionych i zwykłych zaczynają się w tej samej linii.
  Kontrast tekstu do tła 6,26:1 (norma AA to 4,5:1).
- Koloruje wiersze wg grubości albo wg grupy materiału (gatunek spoza listy
  dostaje kolor różowy – widać, że czegoś brakuje w mapowaniu).
- **Odstęp między grupami** (checkbox, domyślnie włączony): pierwszy wiersz każdej
  grupy dostaje 7px paddingu i grubszą linię. Granicę wyznacza aktywny tryb
  kolorowania, a przy „bez kolorów” – grubość. Wiersz uwag zostaje przyklejony do
  swojej pozycji. Dlaczego padding, a nie pusty wiersz: ERP ma
  `border-collapse: collapse` i `1px solid black` na każdej komórce, więc prawdziwa
  przerwa przecięłaby pionowe linie siatki.
  **Separatory pokazują zmianę wartości, więc mają sens tylko przy sortowaniu po tym
  samym kluczu** – ERP ma własne radio `sortowanie = grubosc` nad nagłówkiem kolumn.
- **Index bez kropek, WIELKIE** (checkbox, domyślnie wyłączony):
  `1107-01.01.01.18-02` → `1107-01010118-02`, `1301-01.00.03a.01-01` →
  `1301-010003A01-01`. To **wyłącznie wygląd** – oryginały siedzą w `Map`ie (nie
  w atrybutach, żeby nie brudzić DOM-u), a przycisk „Szukaj” i tak szuka pełnej,
  oryginalnej nazwy, bo tak detal nazywa się w systemie.

  Reguła (`normIndex`) jest celowo **bez heurystyk**: kropki i spacje won, wszystkie
  litery duże. Bezpieczeństwo bierze się z *zasięgu*, nie ze zgadywania – zmiana
  dotyka wyłącznie komórek kolumny **Index i rysunki** w wierszach głównych
  (`applyIndexDots` → `tr.children[C_INDEX]`), a tam nie ma wymiarów ani liczb
  dziesiętnych, tylko nazwy detali. Wiersz uwag i wiersz SUM są pomijane.

  Formaty nazw z prawdziwych zamówień – każdy musi przejść:

  | przed | po |
  |---|---|
  | `227-102111` | `227-102111` |
  | `808-0466.0` | `808-04660` |
  | `808-04a-50 2` | `808-04A-502` |
  | `1107-01.01.01.18-02` | `1107-01010118-02` |
  | `1301-01.00.03a.01-01` | `1301-010003A01-01` |
  | `ME-A06581250R01-rozw` | `ME-A06581250R01-ROZW` |

  Spacje lecą **wszystkie**, łącznie z twardą (`&nbsp;` – w JS `\s` ją łapie).

  **To jest świadoma decyzja, nie przeoczenie.** Na karcie 39335 nazwy wyglądają tak:
  `KMC.35.1251L.00v03.1 podstawa SBC` → `KMC351251L00V031PODSTAWASBC`, czyli opis
  skleja się z numerem. Rozważone i **zostawione tak celowo** – rozdzielenie „numer"
  od „opisu" wymagałoby heurystyki po kształcie tekstu, a te w tej funkcji poległy
  już dwa razy na prawdziwych danych (patrz wyżej). Strukturalnie nie da się ich
  rozróżnić: całość siedzi w jednym `<a>`, a `<span class="podobne">` to tylko
  podświetlenie podobnych nazw, nie osobne pole. Dla ERP-a to **jedna nazwa detalu**
  i „Szukaj" słusznie szuka jej w całości.

  Na kartach z opisowymi nazwami po prostu odznacz opcję – jest przełącznikiem.
  **Nie dorabiaj tu wyjątków.**

  Ostatni przypadek jest podwójnie podchwytliwy: zaczyna się literą, a ERP wstawia
  w środek `<span class="podobne">`, przez co tekst siedzi w **dwóch** węzłach
  (`ME-A06581250R01-r` + `ozw`). Dlatego przetwarzamy węzły tekstowe pojedynczo –
  podświetlenie „podobnych" zostaje nietknięte.

  Wcześniejsza wersja miała test kształtu tokena i wymóg „co najmniej dwie kropki"
  (obrona przed zamianą `2.0 mm` na `20 mm`). Poległa na dwóch z trzech prawdziwych
  formatów: `808-0466.0` ma jedną kropkę, a `ME-…` zaczyna się literą. W tej kolumnie
  taka obrona nie była do niczego potrzebna.
- W kolumnie „Możliwości” dodaje przycisk **Szukaj** → otwiera w nowej karcie
  `http://laser.wawmar.com/detals/szukaj?nazwa=<index detalu>`.

## Czy jest materiał na to zamówienie

Przy każdej pozycji w tabeli `#karta-rezerwacja` pojawia się badge:
`✓ 15 ark` / `✗ brakuje 5 z 5` / `? nie sprawdzono`, a w pasku panel **Materiał**
z podsumowaniem. Gdy czegoś brakuje, panel otwiera się sam i przycisk robi się
czerwony — żeby nie dało się tego przeoczyć.

### Dlaczego nie czytamy gotowego przydziału ERP-a

To jest sedno tej funkcji. Strona zapotrzebowania ma pola rezerwacji **wstępnie
wypełnione przez ERP** i kusi, żeby je po prostu zsumować. Nie wolno — ERP przydziela
także materiał, którego fizycznie nie ma.

Zamówienie 38932, pozycja **DD11 4 mm**: ERP przydzielił komplet 5 arkuszy, ale
z wiersza `widmo` z datą **08.10.2026**. Na hali **zero**. Sumowanie przydziału
dałoby „wszystko OK" — dlatego liczymy sami.

### Reguła

```
dostępne_teraz = suma Arkuszy z wierszy, które są:
    • zgodnego gatunku  (szare tło)     — zamienniki NIE liczą się
    • BEZ klasy `odlegla`               — czyli data dziś lub wcześniej
```

`widmo` **samo w sobie nie dyskwalifikuje** – liczy się data. Materiał, który
dojedzie dziś, jest OK; widmo z przyszłą datą i tak odpada przez `odlegla`.
Zamienniki (inny gatunek z tej samej grupy) nie wchodzą do sumy, ale są pokazane
w dymku: *„zamiennik 10 ark OcynkO"*.

### Skąd co się bierze

| | |
|---|---|
| ile trzeba | karta, `#karta-rezerwacja`, kolumna **Ark.** |
| co jest | `GET /zapotrzebowanias/zlisty/<nr>`, `td.magazyny table` |
| gatunek zgodny | `style="background-color:lightGray"` na komórce Materiał |
| data w przyszłości | `class="odlegla"` na komórce Planowana |
| grubość | `<input name="NNNNNNN[grubosc]">` w wierszu formularza `#zapo` |

Klucz łączenia obu stron to `norm(materiał)` + `thicknessKey(gr)` — te same funkcje
co wszędzie indziej, więc `3` i `3.00` to jedno i to samo.

**Dwa sygnały bierzemy wprost od ERP-a zamiast się domyślać:** `odlegla` zamiast
parsowania dat i szare tło zamiast porównywania nazw gatunków. Jeśli ERP zmieni
reguły (np. inny próg „odległości"), wtyczka pojedzie za nim bez zmian w kodzie.

### Co z tym, że to sieciowe

- **Wyłącznie GET.** Zero POST, zero dotykania pól `magazyn[...]`. Potwierdzone
  w teście przez `read_network_requests`.
- Ta sama domena co karta, więc zwykły `credentials: 'same-origin'` — bez CORS
  i bez dodatkowych uprawnień w `manifest.json`.
- Odpowiedź idzie przez `DOMParser`, **nie** przez `innerHTML` — nic się nie wykonuje.
- **Gdy fetch padnie** (404, wylogowanie, sieć), badge mówi `? nie sprawdzono`
  z powodem w dymku. Nigdy „OK" i nigdy „brak materiału" — brak wiedzy to nie to
  samo co brak materiału.
- Pozycja z karty bez odpowiednika w zapotrzebowaniu → `? brak danych`, też nie „brak".

## Licznik do terminu „Wysłać:"

Data siedzi w nagłówku karty jako `<td>Wysłać:</td><td>2026-09-25</td>` – etykieta,
a data w **następnej** komórce. Szukamy po `norm()` etykiety (`"WYSA"`), bo to działa
tak samo przy zepsutym kodowaniu polskich znaków, i jest w karcie jednoznaczne
(sprawdzone: dokładnie jedno trafienie).

Dwie rzeczy, które łatwo tu zepsuć:

- **Różnica musi być w dniach kalendarzowych**, nie w milisekundach. Obie daty są
  sprowadzane do północy przed odjęciem – inaczej pora dnia albo zmiana czasu
  potrafi przesunąć wynik o jeden dzień.
- **Pusta komórka to normalny przypadek**, nie błąd. Karta 40106 nie ma wpisanej
  daty wysyłki. Wtedy licznik się nie pokazuje – zamiast „NaN dni" czy „1970".

## Ostrzeżenie o przekroczonym limicie kredytowym

Pod „Pozostałe uwagi" jest tabelka `#zadluzenia`:

| Należności | W produkcji | Suma | Limit | Różnica | Ta karta brutto |
|---|---|---|---|---|---|
| 21 992 | 141 664 | 163 656 | 300 000 | 136 344 | 2 157 |

**ERP sam nic z tym nie robi** – pokazuje gołe liczby. Jedyne kolorowanie na czerwono
w `zamowienie.js` dotyczy `.kolorowe_numerki` / `.kolorowe_pola` w tabeli detali,
nie tej tabelki. Dlatego wtyczka sprawdza `Suma > Limit` i przy przekroczeniu
oznacza to w **dwóch** miejscach:

- **czerwony baner na górze paska** – pasek jest `position: fixed`, więc widać go
  zawsze, w obu widokach,
- **czerwone komórki Suma i Limit** w samej tabelce.

Dwa miejsca, bo w widoku uproszczonym `#zadluzenia` ląduje w prawym panelu i może
być zescrollowane poza ekran – sam baner w pasku gwarantuje, że nie umknie.

Świadomie **nie** ma tu `alert()` ani modala. Okienko przy każdym wejściu na kartę
przeterminowanego klienta byłoby nie do zniesienia i szybko klikałoby się je odruchowo.

Dwie rzeczy warte uwagi przy dłubaniu w tym kodzie:

- **Tabelka ma odwrotny układ** – najpierw wiersz z wartościami (`<td>`), a `<th>`
  dopiero **pod** nim. Kolumny szukamy po tekście nagłówka (`norm()`), nie po pozycji.
- **Liczby mają twardą spację** jako separator tysięcy (`163&nbsp;656`), więc
  `parseKwota()` wycina wszystko poza cyframi. Zwykłe `parseInt` zwróciłoby `163`.

## Tryb edycji – zbiorcza zmiana Ilość [szt]

ERP jednoznacznie rozróżnia kartę edytowalną od podglądu:

| | edytowalna | tylko podgląd |
|---|---|---|
| `action` formularza | `…/zlecenia/zamowienie/<nr>` | **`/dupa`** (celowo martwy endpoint) |
| `#przyciski` | Zapisz, Kopiuj, Ustaw stawki | pusty |
| ukryte pole | `test_zapisu` | brak |

`isEditable()` sprawdza oba warunki naraz. Gdy karta jest tylko do podglądu,
przycisk **Ilość ▾** w ogóle się nie pokazuje.

Panel: pole z liczbą, **−** i **+** (zmiana na **wszystkich** pozycjach naraz),
**Przywróć** i licznik. Dolna granica to **1** – odejmowanie nigdy nie zejdzie niżej.

**Wtyczka niczego nie wysyła.** Zmienia tylko pola; zapis robi przycisk *Zapisz*
w karcie (w widoku uproszczonym leży w prawym panelu, bo `#przyciski` jest pod
tabelą detali). Zmienione pola dostają bursztynowe tło, więc przed zapisem widać,
co poleci na serwer. **Przywróć** wraca do wartości sprzed *pierwszej* zmiany
i działa też po przełączeniu widoku (wartości `input.value` nie są atrybutami,
więc przenoszenie DOM ich nie rusza).

### Dlaczego pola `readonly` są pomijane

`readonly` **nie blokuje wysyłki** formularza (w odróżnieniu od `disabled`) – gdyby
wtyczka wpisała tam wartość, zapis by ją utrwalił. ERP blokuje te pola świadomie,
więc zostają nietknięte, a licznik mówi wprost: `zmieniono 15, pominięto 5
(zablokowane)`. Komunikat o pominiętych jest pomarańczowy, żeby nie umknął.

Uwaga: **ERP nie przelicza nic przy zmianie ilości.** Jedyne nasłuchy w
`zamowienie.js` są na `.inputyAjax` (pole „Zam. klienta WZ", zapis AJAX-em);
`.sztuki` nie ma żadnego handlera. Sumy i wartości odświeżą się dopiero po zapisie –
to zachowanie ERP-a, nie błąd wtyczki.

## Arkusze PC-CAM (.PCSHT) jednym kliknięciem

Na dole panelu **Materiał** jest przycisk **⬇ Arkusze PC-CAM (.zip)**. Bierze
tabelę `#karta-rezerwacja` i pobiera `arkusze_<nr karty>.zip` z plikiem `.PCSHT`
na każdą kombinację materiał + grubość + format. Zip rozpakowujesz do biblioteki
arkuszy PC-CAM (`C:\KIMLA\Biblioteki\Arkusze\`).

| z karty | do arkusza |
|---|---|
| Materiał | Materiał (przez `MAT_PCCAM`, dziś 1:1 – nazwy ERP-a i PC-CAM są te same) |
| Gr. | Grubość (`2.00` → `2`) |
| Form. min. | wymiar wg `FORMATY`: 1 → 1000×2000, 2 → 1250×2500, 3 → 1500×3000 |
| numer karty | Zewnętrzny ID |
| – | Liczba kopii `PCSHT_KOPIE` = 999, odstęp od krawędzi `PCSHT_ODSTEP` = 3 mm |

Rozpakowywać ręcznie nie trzeba: **`watcher/arkusze_watcher.pyw`** (w tle, z
autostartem) sam wgrywa każdy nowy `arkusze_*.zip` z Pobranych do biblioteki,
kasując stare arkusze – patrz `watcher/README.md`. Wtyczka nic o nim nie wie
i dalej tylko pobiera plik.

Wiersz z nieznanym „Form. min." jest pomijany i wypisany pod przyciskiem na
pomarańczowo. Przycisk działa też, gdy sprawdzenie magazynu padnie
(`? nie sprawdzono`) – potrzebuje tylko tabeli z karty.

**Nic nie idzie do serwera.** Pliki powstają w przeglądarce (`pcsht.js`), zip jest
składany ręcznie (bez kompresji, CRC-32) i pobierany z `Blob` przez `<a download>`.
Żadnych nowych uprawnień w manifeście.

### Format pliku i suma kontrolna

`pcsht.js` to port 1:1 `C:\Users\kasja\Documents\arkusze\pcsht_generator.py`:
szablon (oryginalny `S235, 3 mm, 1500.00x3000.00 mm.PCSHT`, base64 w
`TEMPLATE_B64`) z podmienionymi polami. Najważniejsze:

- **Suma kontrolna = MD4** danych od offsetu zapisanego w pierwszych 8 bajtach
  (u64, 0x173E) do końca pliku, zapisana pod 0x0C (16 B). Bez niej PC-CAM mówi
  „Suma kontrolna pliku jest błędna". Algorytm wyczytany z `PcCam_plus.exe`
  (Indy `TIdHashMessageDigest4`); sprawdzony na wszystkich 12 plikach biblioteki.
- Obrys prostokąta to 2×12 punktów (początek, koniec, środek każdego boku) –
  podmieniane pozycyjnie, z asercją wartości szablonu, bo przy formatach 1:2
  „połowa wysokości" = „szerokość" i zamiana po wartości by je pomyliła.

Test zgodności bajt w bajt: `python -m http.server 8777` w katalogu projektu →
`http://127.0.0.1:8777/test-pcsht/` ma pokazać `WSZYSTKO OK` (oryginały z biblioteki
+ pliki z generatora Pythona). **Po każdej zmianie w `pcsht.js`.**

## Konfiguracja

Wszystko do zmiany jest na górze `content.js`:

| Stała | Do czego |
|---|---|
| `SEARCH_URL` | adres wyszukiwarki detali |
| `FAVS` | ulubione odnośniki wyróżniane w menu |
| `KEEP` | lista kolumn zostawianych w widoku uproszczonym |
| `GROUPS` | grupy materiałów + ich kolory |
| `THICK_COLORS` | kolory dla grubości |
| `FORMATY`, `PCSHT_KOPIE`, `PCSHT_ODSTEP`, `MAT_PCCAM` | arkusze PC-CAM (patrz wyżej) |

Kolumny w `KEEP` są dopasowywane po **tekście nagłówka** (po normalizacji do
`A-Z0-9`, dzięki czemu działa też przy zepsutym kodowaniu polskich znaków),
a `idx` to pozycja zapasowa na wypadek zmiany nazwy kolumny w ERP.

`FAVS` dopasowuje się tak samo – po znormalizowanym tekście odnośnika, więc
wielkość liter, spacje i polskie znaki nie mają znaczenia, ale nazwa musi się
zgadzać w całości (`Zapotrzebowanie lista` ≠ `Zapotrzebowanie 1`). Jeśli któraś
pozycja z `FAVS` nie zostanie znaleziona, wtyczka wypisuje ją raz w konsoli –
zwykle znaczy to, że ERP zmienił podpis odnośnika.

Adresy w `manifest.json` (`192.168.0.3`, `laser.wawmar.com`) dopisz/zmień, jeśli
wchodzisz na ERP pod innym hostem.

## Testowanie bez ERP

Otwórz przez lokalny serwer (`python -m http.server` w folderze wyżej), nie przez
`file://`:

- `../test-podglad.htm` – zapisana karta 39769 z doklejoną wtyczką (1 pozycja).
- `../test-podglad-multi.htm` – ta sama karta rozmnożona do 41 pozycji w 10
  grubościach (20…1.5). Indeksy celowo w **wszystkich** prawdziwych formatach:
  `1107-01.02.01.05-04`, `1301-01.00.06a.01-01` (mała litera w środku),
  `808-0412.0` (**jedna** kropka), `808-0410a-50 2` (spacja w środku) oraz
  `ME-A0658…R01-r<span>ozw</span>` (zaczyna się literą, tekst rozbity na dwa węzły).
  Na wersji z jedną pozycją nie da się sprawdzić ani odstępów między grupami,
  ani normalizacji indeksów.
- `../test-limit.htm` i `../test-limit-ponad.htm` – karta 39335, raz z prawdziwymi
  liczbami (Suma 918 889 < Limit **1 300 000** → bez ostrzeżenia), raz z podmienionymi
  (Suma 1 586 981 > Limit → ostrzeżenie). Żadna prawdziwa karta nie ma przekroczenia,
  więc przypadek dodatni trzeba zrobić samemu. 39335 jest tu lepszą bazą niż 39769,
  bo ma limit **siedmiocyfrowy** – `1&nbsp;300&nbsp;000`, czyli dwa separatory
  w jednej liczbie. Właśnie na tym wyłożyłby się naiwny parser: `parseInt("1 300 000")`
  zwraca `1`, więc limit nigdy nie zostałby przekroczony.
- `../test-edycja.htm` – karta **edytowalna** (z `EDYCJA/Zamówienie 40106.html`),
  20 pozycji, z czego 5 ma `readonly` na `.sztuki` – do sprawdzania pomijania
  i licznika. Uwaga przy przebudowie: zapisana karta 40106 **zawiera już ślady
  wtyczki** (wstrzyknięte `a.wm-find` i cały `#wm-bar`), bo była zapisana z włączonym
  rozszerzeniem – generator je wycina, inaczej po załadowaniu byłyby dwa.

Fixture'y **generuje się skryptami** z `../testy/` (mają w środku ścieżki absolutne):

```
python testy/mkmulti.py     # -> test-podglad-multi.htm  (41 pozycji, podgląd)
python testy/mkedycja.py    # -> test-edycja.htm         (20 pozycji, edycja)
python testy/mklimit.py     # -> test-limit.htm + test-limit-ponad.htm
python testy/mkmagazyn.py   # -> test-magazyn/  (cale drzewo sciezek, patrz nizej)
python testy/mktermin.py    # -> test-termin/   (warianty terminu wzgledem DZISIAJ)
```

Przy dopisywaniu nowego formatu indeksu dorzuć go do gałęzi `elif` w `mkmulti.py`
i przegeneruj – to jest ta regresja, która złapała `808-0466.0` i `ME-…-rozw`.

**Uwaga przy testach:** przeglądarka cache'uje fixture'y. Po przegenerowaniu
otwieraj z doklejonym parametrem (`?v=2`), inaczej zobaczysz starą wersję i będziesz
szukać błędu tam, gdzie go nie ma.

### Fixture terminu liczy daty względem dzisiaj

`testy/mktermin.py` **nie wpisuje dat na sztywno** – liczy je od dzisiejszej daty
(−5, −1, 0, +1, +2, +10 dni i wariant z pustą komórką). Sztywna data z czasem zmienia
znaczenie: to, co dziś jest „jutro", za tydzień byłoby „po terminie", a test cicho
przestałby sprawdzać to, co miał sprawdzać. Po prostu przegeneruj przed testem.

### Fixture stanu materiału jest inny niż reszta

`testy/mkmagazyn.py` nie robi pojedynczego pliku, tylko **drzewo katalogów**
`test-magazyn/`, bo funkcja opiera się na `fetch` pod prawdziwą ścieżką:

```
test-magazyn/
  zlecenia/zamowienie/38932/index.html       karta (DD11 4mm, DD11 3mm, DC01 3mm po 5 ark)
  zapotrzebowanias/zlisty/38932/index.html   zapisana strona GTG
  zlecenia/zamowienie/38933/…                wariant: widmo z DZISIEJSZĄ datą
  zapotrzebowanias/zlisty/38933/…            → DD11 4mm ma wtedy wyjść OK
  zlecenia/zamowienie/99999/…                bez odpowiednika → fetch 404
```

Serwer uruchamiaj **wewnątrz** `test-magazyn`, nie w katalogu projektu:

```
cd test-magazyn && python -m http.server 8777
```

Cztery rzeczy, które ten fixture pilnuje: brak materiału mimo przydziału ERP-a,
niewliczanie zamienników, regułę „przyjedzie dziś = jest" i uczciwe zachowanie
przy padniętym fetchu.

**Przy teście round-trip odczekaj na fetch** (ok. 1,5 s) przed zrobieniem snapshotu.
Inaczej „różnica" w `body.innerHTML` będzie artefaktem testu, a nie błędem.

Uwaga przy pisaniu selektorów: klasa `podswietlanie_wierszy` występuje w ERP **dwa
razy** – w tabeli detali i w `#karta-rezerwacja` na dole strony. Wszystkie regułki
kolorów/odstępów są zawężone do `table.detale`, żeby nie ruszać tej drugiej.

## Gdybyś wrócił do tego za pół roku

Wszystko, co trzeba wiedzieć, jest w tym pliku i w komentarzach w `content.js` –
konwersacja, w której to powstało, nie przetrwa. Najkrótsza droga z powrotem:

1. Załaduj `erp-overlay` w `brave://extensions` (Load unpacked) i wejdź na kartę
   zamówienia – zobaczysz, co wtyczka robi.
2. `python -m http.server` w tym folderze + `test-podglad-multi.htm` i
   `test-edycja.htm` – działa bez dostępu do ERP-a.
3. Cała konfiguracja (adres wyszukiwarki, ulubione, kolumny, grupy materiałów,
   kolory) siedzi na górze `content.js`, w bloku oznaczonym
   `=== do edycji w razie potrzeby ===`.

Dwie rzeczy, które okazały się najmniej oczywiste i najłatwiej je znowu zepsuć:

- **Gwarancja identycznego DOM-u** przy wyjściu z widoku uproszczonego. Trzyma się
  na `setClass()` / `setVar()` i kasowaniu atrybutów roboczych – patrz nagłówek tego
  pliku. Testuj zawsze **od czystego `localStorage`**, bo pierwsze przełączenie jest
  inne niż kolejne.
- **`normIndex` ma być głupi.** Dwie próby sprytnych heurystyk poległy na prawdziwych
  nazwach detali. Bezpieczeństwo daje zasięg (tylko kolumna Index i rysunki), nie
  zgadywanie formatu.
