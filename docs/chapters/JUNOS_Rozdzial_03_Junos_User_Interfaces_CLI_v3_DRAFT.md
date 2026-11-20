# Rozdział III. Junos User Interfaces — od pierwszego logowania do bezpiecznego zatwierdzania zmian

**Podręcznik Junos Enterprise Routing & Switching • JNCIA-Junos → JNCIS-ENT → JNCIP-ENT**  
**Blueprint:** 3.1–3.10 [A] i 3.11 [EXT]  
**Odbiorca:** Network Engineer z około rocznym doświadczeniem. Zna znaczenie routera, adresu IP, VLAN i bramy domyślnej, ale nie musi znać Cisco, CCNA, JNCIA ani wcześniej konfigurować Junos.  
**Wersja redakcyjna:** 3.0, 20.11.2026  
**Topologia:** [DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md), stopniowo od jednego routera do całego układu.  
**Status laboratoriów:** **PLANNED / NOT RUNTIME VALIDATED** — przykłady są projektem do wykonania, a nie outputem uzyskanym z uruchomionego vSRX.

---

## Zanim wydasz pierwszą komendę

W poprzednim rozdziale poznaliśmy dwa ważne elementy Junos. Routing Engine (RE) zajmuje się m.in. informacjami o trasach i administracją, a Packet Forwarding Engine (PFE) wykorzystuje przygotowane informacje do przekazywania ruchu. Teraz pora nauczyć się rozmawiać z urządzeniem: sprawdzać jego stan i zmieniać konfigurację bez narażania całej sieci.

Zacznijmy od bardzo zwyczajnego zadania. W firmie masz router R1 i otrzymujesz prośbę o wpisanie opisu jednego łącza: „Połączenie do R2 — górny korytarz”. Taka zmiana nie powinna przełączyć routingu, odciąć użytkowników ani zmienić adresu IP. Mimo to warto wiedzieć, jak dokładnie trafi do działającej konfiguracji.

W wielu systemach, gdy zmienisz ustawienie, zaczyna ono działać od razu. **Junos rozdziela przygotowanie konfiguracji od jej aktywowania.** Administrator tworzy nową wersję roboczą, może obejrzeć różnice i dopiero potem świadomie ją zatwierdza.

To główna idea całego rozdziału. Każda nowa komenda będzie miała sens, jeśli będziesz potrafił odpowiedzieć: *czy właśnie tylko oglądam urządzenie, czy edytuję wersję roboczą, czy już zmieniłem działający system?*

### Nasza wspólna topologia

Nie budujemy nowej sieci tylko dla rozdziału o CLI. Obowiązuje ten sam układ **DIAMOND-6R**:

~~~mermaid
flowchart LR
  R5["R5 CE-A<br/>192.0.2.10/24"] ---|"192.0.2.0/24"| R1["R1 EDGE-A<br/>192.0.2.1/24"]
  R1 ---|"10.12.0.0/30"| R2["R2 CORE-1"]
  R1 ---|"10.13.0.0/30"| R3["R3 CORE-2"]
  R2 ---|"10.24.0.0/30"| R4["R4 EDGE-B<br/>198.51.100.1/24"]
  R3 ---|"10.34.0.0/30"| R4
  R4 ---|"198.51.100.0/24"| R6["R6 CE-B<br/>198.51.100.10/24"]
~~~

*Rysunek 3.1. Do nauki samego CLI wystarczy R1. R5 posłuży później jako drugi router do kontrolowanego testu konfiguracji, a R1–R6 jako kontekst dużej zmiany. Wszystkie adresy końców łączy i nazwy portów są w [referencji DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md).*

R1 ma `ge-0/0/0.0` do R2 (10.12.0.1/30), `ge-0/0/1.0` do R3 (10.13.0.1/30) i `ge-0/0/2.0` do R5 (192.0.2.1/24). W tym rozdziale **nie będziemy zmieniać adresacji ani protokołów routingu**. Ćwiczymy narzędzia administracyjne, nie złożony routing. Nie ma powodu, aby początkujący uczeń uruchamiał sześć routerów tylko po to, by nauczyć się polecenia `show`.

---

## 3.1. Czym jest CLI i jak rozmawia się z Junos? [A]

**CLI** oznacza *Command-Line Interface* — interfejs tekstowy, w którym wydajesz polecenia urządzeniu. Komendami można odczytywać informacje, wykonywać testy, zmieniać ustawienia i sprawdzać wyniki.

Do CLI Junos najczęściej trafisz przez **konsolę** albo przez **SSH**. Konsola jest bezpośrednim kanałem dostępu do urządzenia lub jego maszyny wirtualnej. SSH to szyfrowana sesja sieciowa, wymagająca działającej drogi do adresu zarządzania oraz odpowiedniej usługi. W EVE-NG konsola jest szczególnie przydatna, ponieważ może pozostać dostępna nawet wtedy, gdy sam przypadkowo popsujesz konfigurację sieciową.

Po zalogowaniu możesz zobaczyć coś takiego:

~~~text
operator@R1-vSRX>
~~~

To **prompt**, czyli znak zachęty. Pokazuje, że system czeka na kolejne polecenie. Nazwa po znaku `@` oznacza urządzenie, do którego jesteś zalogowany. W przykładach stosujemy nazwy zgodne z topologią; Twój realny vSRX może początkowo pokazywać inną nazwę.

**R1-vSRX — Operational Mode: pierwszy, bezpieczny odczyt**

~~~junos
show version
show system uptime
show interfaces terse
show system users
~~~

Co mówią te polecenia? `show version` pokazuje model i wersję systemu. `show system uptime` mówi, jak długo działa urządzenie. `show interfaces terse` wyświetla krótką listę interfejsów i ich stanu. `show system users` pomaga sprawdzić, kto jeszcze jest zalogowany.

Pamiętaj o jednej rzeczy: **tryb operational nie oznacza automatycznie trybu tylko do odczytu**. Można w nim także wykonywać działania operacyjne o poważnych skutkach — na przykład restart lub czyszczenie określonego stanu. Na początku ograniczamy się do poleceń kontrolnych.

### CLI, Configuration Mode i shell nie są tym samym

Junos udostępnia dwa główne obszary pracy administratora: tryb wykonywania poleceń operacyjnych oraz tryb edycji konfiguracji. W tle działa również system operacyjny, a na wybranych platformach można wejść do jego **shella**, czyli powłoki systemowej.

Shell nie jest „wyższym poziomem CLI” ani skrótem do konfiguracji routera. Daje dostęp do innej warstwy systemu i nie obejmuje go zwykły model edycji candidate/commit. Dlatego nie będziemy używać go do zmieniania plików konfiguracji. Dla początkującego inżyniera bezpieczniejszym wyborem jest standardowy Junos CLI.

**R1-vSRX — Operational Mode: rozpoznanie własnego środowiska**

~~~junos
show cli
show version
help apropos commit
~~~

Polecenie `help apropos` pozwala szukać pomocy po słowie kluczowym. Nie musisz znać składni wszystkich komend; ważne, żeby wiedzieć, jak bezpiecznie ją odnaleźć.

---

## 3.2. Operational Mode i Configuration Mode — co właściwie zmienia znak promptu? [A]

Po zalogowaniu zobaczyłeś znak **`>`**. Zwykle oznacza **Operational Mode**, w którym sprawdzasz stan i wykonujesz polecenia operacyjne. Aby zmienić konfigurację, musisz przejść do osobnego trybu.

**R1-vSRX — Operational Mode: przejście do edycji**

~~~junos
configure
~~~

Teraz prompt może wyglądać tak:

~~~text
[edit]
operator@R1-vSRX#
~~~

Znak **`#`** oznacza, że znajdujesz się w **Configuration Mode**. Wiersz `[edit]` informuje, że oglądasz najwyższy poziom drzewa konfiguracji.

Nie chodzi tylko o inny znak. W Configuration Mode pracujesz z **candidate configuration** — konfiguracją przygotowywaną do zatwierdzenia. Samo wpisanie nowej wartości nie musi jeszcze zmienić zachowania routera.

**R1-vSRX — Configuration Mode [edit]: pierwsze polecenia**

~~~junos
show system
top
run show version
~~~

`show system` pokazuje gałąź **konfiguracji**, nie bieżący stan CPU ani interfejsów. `top` wraca do najwyższego poziomu hierarchii. `run show version` pozwala wykonać polecenie operacyjne, nie opuszczając edytora.

Słowo `run` jest przydatne, gdy przeglądasz konfigurację i chcesz szybko sprawdzić np. stan portu. Nie wszystkie działania dostępne w Operational Mode da się jednak uruchamiać w każdym kontekście przez `run`; to zależy od polecenia i wydania Junos.

**R1-vSRX — Configuration Mode [edit]: powrót do trybu operacyjnego**

~~~junos
exit
~~~

Gdy na urządzeniu są niezapisane zmiany w candidate, system może poprosić o decyzję przy wyjściu. Nie klikaj ani nie zatwierdzaj komunikatu mechanicznie — sprawdź, czy w edytorze nie ma potrzebnych zmian.

### Pierwsze rozróżnienie, które warto zapamiętać

`show interfaces terse` w Operational Mode to odczyt stanu interfejsów. Natomiast `show interfaces` w Configuration Mode to odczyt części konfiguracji, którą przygotowujesz do aktywacji.

Podobne słowa, lecz dwa różne pytania. Dlatego przy każdym bloku CLI w podręczniku będziemy wskazywać **hostname i tryb pracy**. Jeśli kopiujesz polecenie, najpierw zobacz, czy router pokazuje `>`, czy `#`.

**R1-vSRX — Operational Mode: odczyt aktywnej konfiguracji bez przechodzenia do edytora**

~~~junos
show configuration interfaces ge-0/0/0
~~~

To ważny most między trybami: w Operational Mode można wyświetlić aktualnie **zatwierdzoną konfigurację**, nawet gdy nie chcesz niczego zmieniać.

---

## 3.3. Hierarchia konfiguracji — dlaczego drzewo jest lepsze niż przypadkowa lista linii? [A]

W Junos konfiguracja ma budowę **hierarchiczną**. To znaczy, że ustawienia znajdują się w odpowiednich gałęziach drzewa, podobnie jak pliki w katalogach na komputerze.

Wyobraź sobie, że opisujesz port R1 do R2. Informacja nie istnieje w próżni. Należy do konkretnego interfejsu, który z kolei należy do gałęzi `interfaces`.

~~~text
interfaces
└── ge-0/0/0
    ├── description
    └── unit 0
        └── family inet
            └── address 10.12.0.1/30
~~~

To uproszczony obraz jednej małej części konfiguracji R1. `ge-0/0/0` to nazwa interfejsu. `unit 0` oznacza jednostkę logiczną, a `family inet` — rodzinę protokołów IPv4. Dokładniej omówimy interfejsy i jednostki logiczne w rozdziale IV. Teraz ważne jest to, **gdzie** wpisujesz polecenie.

**R1-vSRX — Operational Mode: wejście do edycji**

~~~junos
configure
~~~

**R1-vSRX — Configuration Mode [edit]: nawigacja do konkretnej gałęzi**

~~~junos
edit interfaces ge-0/0/0
show
up
top
show interfaces ge-0/0/0
~~~

Gdy jesteś w `[edit interfaces ge-0/0/0]`, `show` dotyczy właśnie tego interfejsu. `up` przechodzi poziom wyżej, a `top` wraca do początku drzewa. To tak, jakbyś przechodził do katalogu, wracał do katalogu nadrzędnego albo wracał do korzenia.

Możesz też używać pełnej ścieżki bez wchodzenia do każdej gałęzi. To wygodne podczas automatyzacji i dla osób, które na początku łatwo gubią się w hierarchii.

**R1-vSRX — Configuration Mode [edit]: pełna ścieżka do ustawienia**

~~~junos
show interfaces ge-0/0/0 | display set
~~~

Wynik w formacie `set` pokazuje hierarchię spłaszczoną do pojedynczych poleceń. Ułatwia to odczyt i kopiowanie **konfiguracji**, ale nie oznacza, że system przechowuje dwie różne jej wersje.

### Co robi `delete`, gdy jesteś w niewłaściwej gałęzi?

Konfiguracja hierarchiczna jest wygodna, ale wymaga ostrożności. Usunięcie jednej wartości, np. opisu portu, ma zupełnie inne konsekwencje niż usunięcie całej gałęzi `interfaces ge-0/0/0`, która może zawierać adresy i inne parametry.

W kolejnej sekcji nauczysz się używać `set` i `delete` na nieszkodliwym opisie. Na razie zapamiętaj: **przed usunięciem czegokolwiek sprawdź aktualną hierarchię**, a po zmianie zobacz `show | compare`.

---

## 3.4. Pomoc kontekstowa — nie musisz znać całego CLI na pamięć [A]

Kiedy pierwszy raz otworzysz Configuration Mode, liczba możliwych poleceń może wydawać się ogromna. Nie ma sensu uczyć się wszystkich hierarchii od pierwszego dnia. Ważniejsze jest zrozumienie, **jak Junos podpowiada dostępne możliwości**.

Wyobraź sobie, że chcesz sprawdzić składnię konfiguracji opisu interfejsu. Znasz gałąź `interfaces`, ale nie pamiętasz reszty. W Junos znak **`?`** pokazuje pomoc w miejscu, w którym się znajdujesz. W tym przypadku wpisz polecenie częściowo, a potem znak zapytania.

**R1-vSRX — Configuration Mode [edit]: pomoc w gałęzi interfejsu**

~~~junos
set interfaces ge-0/0/0 ?
~~~

To nie jest próba zatwierdzenia niepełnej konfiguracji. System ma pokazać, jakie słowa i argumenty są możliwe w tym miejscu.

Drugi przydatny mechanizm to **uzupełnianie poleceń** klawiszem `Tab`. Gdy wpiszesz początek jednoznacznej komendy, CLI może uzupełnić resztę. Jeśli jest kilka możliwości, poproś o listę. Część zachowań zależy od ustawień sesji, wersji Junos i liczby pasujących słów.

**R1-vSRX — Operational Mode: pomoc opisowa i ustawienia sesji**

~~~junos
help apropos interface
help apropos rollback
show cli
~~~

Dwie pierwsze linie wyszukują informacje po słowie. Ostatnia pozwala sprawdzić ustawienia terminala. Zobacz, dlaczego to wygodniejsze od losowego wklejania komend znalezionych w Internecie: CLI odpowiada z uwzględnieniem tego, czego **Twoja platforma i wydanie** rzeczywiście potrafią oczekiwać.

### Długa odpowiedź z routera nie musi zasypać terminala

Przy niektórych poleceniach wynik ma kilkaset linii. Junos może zatrzymywać ekran i czekać na dalsze przewijanie — to nazywamy **paginacją**. Jeśli potrzebujesz skopiować cały output, możesz ją czasowo wyłączyć w swojej sesji.

**R1-vSRX — Operational Mode: ustawienia czytelności terminala**

~~~junos
show cli
set cli screen-length 0
set cli screen-width 160
show cli
~~~

`screen-length 0` wyłącza pauzy między ekranami. Ustawienie dotyczy sesji CLI, a nie działania portów routera. Przy bardzo obszernych wynikach brak paginacji może być uciążliwy, więc warto przywrócić normalną długość ekranu.

**R1-vSRX — Operational Mode: ponowne włączenie paginacji**

~~~junos
set cli screen-length 24
~~~

Jeśli polecenie nie istnieje na danym vSRX, nie zakładaj od razu, że urządzenie jest zepsute. Sprawdź `show version`, użyj `?` i odpowiedniej dokumentacji. W Juniper podobna funkcja może mieć inną implementację na EX, MX, QFX i SRX.

---

## 3.5. Filtrowanie wyników — znajduj potrzebne informacje zamiast czytać każdą linię [A]

W rozdziale II wykonywałeś `show interfaces terse` i `show route`. W małym labie ich wyniki nie są szczególnie długie. W dużej firmie router może mieć setki interfejsów, wiele wpisów logów i ogromną tablicę tras. Szybko zobaczysz, że samo `show` nie wystarcza.

Junos pozwala dopisać do polecenia znak **`|`**, nazywany *pipe*. Po nim umieszczasz sposób przetworzenia wyniku. Najpierw nauczmy się czterech najprostszych.

**R1-vSRX — Operational Mode: ograniczenie liczby linii**

~~~junos
show interfaces terse | match ge-0/0
show interfaces terse | except lo0
show log messages | last 20
show interfaces extensive | find ge-0/0/0
~~~

`match` pokazuje tylko linie zawierające dopasowanie do wzorca. `except` odrzuca pasujące linie. `last` pomaga zobaczyć końcówkę outputu. `find` zaczyna wyświetlanie od znalezionego miejsca, często zachowując dalszy kontekst.

Przy pierwszym czytaniu wyników `show interfaces extensive` łatwo pomylić `find` z `match`. Pierwsze polecenie zwykle pokaże również tekst **po** odnalezionym miejscu, podczas gdy `match` ograniczy wynik do dopasowanych linii. Jeśli szukasz całego bloku liczników portu, `find` może być właściwsze; jeśli potrzebujesz tylko komunikatów zawierających określone słowo, użyj `match`.

### A co, jeśli musisz znaleźć dwie informacje jednocześnie?

W filtrach można używać dopasowań do wzorców tekstowych. W wielu przypadkach zamiast dwóch oddzielnych `match` wygodniej zastosować wyrażenie z alternatywą, np. `"error|down"`. To znaczy „linia zawierająca error **lub** down”.

**R1-vSRX — Operational Mode: prosty wzorzec alternatywny**

~~~junos
show log messages | match "error|down"
show interfaces terse | match "ge-0/0/0|ge-0/0/1"
~~~

Jeżeli dopiszesz następny `| match` **po pierwszym**, drugie filtrowanie dotyczy tylko wyników, które przeszły przez pierwsze. W efekcie możesz uzyskać inne znaczenie niż „albo to, albo tamto”.

### Uważaj na `count`

`| count` nie jest uniwersalnym licznikiem interfejsów czy tras. **Liczy linie tekstowego wyniku**. Jeśli jedno logiczne zdarzenie zajmuje kilka wierszy, wynik może być większy niż liczba zdarzeń.

**R1-vSRX — Operational Mode: liczba wyświetlonych wierszy, nie liczba portów**

~~~junos
show interfaces terse | match ge- | count
~~~

Jeżeli zobaczysz liczbę 15, nie pisz w raporcie „router ma 15 aktywnych interfejsów”. Policzyłeś wiersze, które spełniały filtr. Liczba interfejsów administracyjnie lub fizycznie aktywnych wymaga osobnego kryterium i świadomej interpretacji.

### Dlaczego konfiguracja może wyglądać inaczej w dwóch widokach?

Junos przechowuje konfigurację hierarchicznie, ale może pokazać ją jako zestaw poleceń `set`. To szczególnie pomocne, gdy porównujesz dwa routery.

**R1-vSRX — Operational Mode: dwa widoki tej samej konfiguracji**

~~~junos
show configuration interfaces ge-0/0/0
show configuration interfaces ge-0/0/0 | display set
show configuration system | display set
~~~

Pierwszy wynik ma strukturę z nawiasami klamrowymi. Drugi prezentuje pojedyncze linie rozpoczynające się od `set`. Oba opisują tę samą aktywną konfigurację; `display set` nie tworzy jej kopii ani nie zapisuje pliku.

W dalszych rozdziałach spotkasz **configuration groups**. To mechanizm, dzięki któremu różne fragmenty konfiguracji mogą dziedziczyć ustawienia z wspólnego szablonu. Gdy parametr nie jest widoczny bezpośrednio w interfejsie, pomocne jest `display inheritance`.

**R1-vSRX — Operational Mode: odczyt dziedziczonych ustawień, jeśli istnieją**

~~~junos
show configuration interfaces ge-0/0/0 | display inheritance
~~~

Szczegółowe `apply-groups` poznasz później. Tutaj zapamiętaj, że wynik `show configuration interfaces` nie musi zawierać wszystkich ustawień wynikających z dziedziczenia. To ważne podczas diagnostyki.

### Filtr nie zawsze daje pełny obraz

Jeśli w logu wyszukasz tylko `error`, możesz przeoczyć wcześniejszy komunikat, który wyjaśnia przyczynę awarii. Jeśli wyślesz do ticketu tylko jedną linię `show interfaces`, zgubisz informację o licznikach i czasie ostatniej zmiany stanu. Dlatego podczas diagnozy zaczynamy od pełnego odczytu właściwego obszaru, a dopiero potem ograniczamy go filtrami.

Przy udostępnianiu konfiguracji pamiętaj również o bezpieczeństwie: całe `show configuration` może zawierać hasła w postaci zaszyfrowanej, klucze, dane management oraz informacje o architekturze klienta. **Nigdy nie publikuj pełnego zrzutu w publicznym repozytorium bez weryfikacji i anonimizacji.**

---

## 3.6. Active i candidate — najważniejszy mechanizm konfiguracji Junos [A]

Teraz przejdziemy do zagadnienia, bez którego wcześniejsze polecenia `set` byłyby tylko słowami w terminalu.

Wyobraź sobie dwie wersje jednego dokumentu. Pierwsza jest opublikowana i wszyscy z niej korzystają. Druga to wersja robocza, w której poprawiasz treść. Możesz ją edytować, porównywać i wycofywać zmiany bez ruszania dokumentu opublikowanego. Dopiero świadome zatwierdzenie zastępuje opublikowaną wersję nową.

Tak Junos oddziela **active configuration** od **candidate configuration**.

**Active** to konfiguracja aktualnie zatwierdzona i wykorzystywana przez system. **Candidate** to wersja, którą przygotowujesz w Configuration Mode. Candidate może różnić się od active, ale samo wpisanie do niej polecenia jeszcze nie aktywuje nowej wartości.

### Pierwszy eksperyment na nieszkodliwym opisie

R1 ma port `ge-0/0/0` prowadzący do R2. Dodamy opis tylko w candidate. Na początku nie wykonamy `commit`.

**R1-vSRX — Operational Mode: wejście do edytora**

~~~junos
configure
~~~

**R1-vSRX — Configuration Mode [edit]: przygotowanie zmiany**

~~~junos
set interfaces ge-0/0/0 description "CH03-LAB-DO-R2"
show interfaces ge-0/0/0
show | compare
~~~

Pierwsza linia dodaje opis do wersji roboczej. Druga pokazuje ten fragment candidate. Trzecia odpowiada na bardzo praktyczne pytanie: **co dokładnie zmieniłem względem działającej konfiguracji?**

Jeżeli opis nie był wcześniej ustawiony, diff powinien pokazywać dodaną linię. Jeśli istnieje inny opis, zobaczysz zastąpienie starej wartości nową. W żadnym przypadku nie jest to jeszcze dowód, że router przestał korzystać z poprzedniej aktywnej konfiguracji.

**R1-vSRX — Operational Mode, osobna druga sesja lub po wyjściu z edytora bez porzucania zmian: odczyt active**

~~~junos
show configuration interfaces ge-0/0/0
~~~

To odczyt konfiguracji zatwierdzonej. Jeżeli jeszcze nie wykonałeś commit, nowy opis może być widoczny w candidate, ale nie w active. Właśnie to chcieliśmy zobaczyć.

### Co rzeczywiście robi `commit`?

`commit` każe systemowi sprawdzić przygotowaną konfigurację i — jeśli zostanie przyjęta — aktywować ją jako nową. To nie jest tylko „zapisanie tekstu w pliku”.

**R1-vSRX — Configuration Mode [edit]: zatwierdzenie tylko w osobnym, bezpiecznym labie**

~~~junos
show | compare
commit check
commit
~~~

`commit check` jest kontrolą konfiguracji przed aktywacją. Nie zastępuje testu działającego pingu, aplikacji ani utrzymania dostępu przez SSH. Poprawny commit może aktywować logicznie błędną konfigurację — na przykład trasę przez niewłaściwego operatora lub filtr blokujący użytkowników.

W tej prostej próbie zmieniamy wyłącznie opis portu danych, więc ryzyko jest niewielkie. Przy zmianach adresacji, stref, routingu czy managementu zastosujemy później **`commit confirmed`**.

### A jeśli dwie osoby jednocześnie edytują konfigurację?

Do tej pory pracowaliśmy jako jedyny administrator. W prawdziwej firmie urządzenie może mieć dwóch operatorów w tym samym czasie. Tu pojawia się niuans: zwykłe **`configure` prowadzi do wspólnej candidate**, jeśli kilku administratorów korzysta z tego trybu.

Załóżmy, że administrator A zmienia opis portu R1→R2, a administrator B przygotowuje zmianę opisu portu R1→R3. Oboje są w zwykłym `configure`. Jeśli A wykona `commit`, może zatwierdzić również zmiany B. To nie musi być błąd systemu; to konsekwencja **wspólnej wersji roboczej**.

Junos pozwala jednak pracować inaczej.

**R1-vSRX — Operational Mode, trzy alternatywne sposoby wejścia do edytora**

~~~junos
configure
~~~

Pierwszy wariant daje współdzielony candidate. Pozostałych nie wpisuj po nim w tej samej sekwencji; to **alternatywne wejścia**, wybierane z Operational Mode po zakończeniu poprzedniej sesji edycji.

**R1-vSRX — Operational Mode, alternatywa: prywatna wersja robocza**

~~~junos
configure private
~~~

W `configure private` tworzysz własną candidate. Inni operatorzy mogą równolegle mieć własne prywatne zmiany. Nie oznacza to jednak, że konfiguracja aktywna w tle przestaje się zmieniać. Jeśli inny operator zatwierdzi konfigurację wcześniej, Twoja prywatna wersja może wymagać aktualizacji i rozstrzygnięcia konfliktu.

**R1-vSRX — Operational Mode, alternatywa: wyłączna blokada edycji**

~~~junos
configure exclusive
~~~

`configure exclusive` służy do zajęcia globalnego edytora na wyłączność. Może blokować zatwierdzanie zmian przez innych operatorów, dlatego nie jest czymś, co włącza się bez uzgodnienia w dużej firmie. To mechanizm organizacji pracy, a nie rozwiązanie wysokiej dostępności routera.

### Co, jeśli moja prywatna konfiguracja powstała przed zmianą kolegi?

Wyobraź sobie, że administrator A otworzył `configure private` i zaczął pracować nad opisem portu. Tymczasem B zmienił inny parametr i wykonał commit. Wersja robocza A jest oparta na wcześniejszym stanie active.

Junos udostępnia polecenie **`update`**, które w private mode pozwala spróbować zsynchronizować prywatną kandydacką konfigurację z nowszym active, zachowując własne zmiany, jeśli nie ma nieusuwalnego konfliktu. **Nie działa to jak magiczny automatyczny merge wszystkiego**.

**R1-vSRX — Configuration Mode [edit private], tylko po stwierdzeniu zmiany active przez innego operatora**

~~~junos
status
show | compare
update
show | compare
~~~

Przed i po `update` sprawdź różnicę. Jeśli dwie osoby modyfikowały te same obiekty, potrzebna może być ręczna decyzja. Nie wciskaj potem `commit` tylko dlatego, że polecenie `update` zakończyło się bez komunikatu o błędzie.

### Czemu nawet w trybie operational sprawdzamy innych administratorów?

Dwie przydatne komendy to `show system users` i `show system commit`. Pierwsza pozwala zobaczyć zalogowanych użytkowników. Druga zawiera historię zatwierdzania i informację o niektórych oczekujących operacjach, w tym o niepotwierdzonym `commit confirmed`.

**R1-vSRX — Operational Mode: zanim rozpoczniesz zmianę w zespole**

~~~junos
show system users
show system commit
~~~

Zanim uruchomisz `configure` w produkcyjnym change window, ustal, czy ktoś inny właśnie nie pracuje nad tym samym routerem. Czas poświęcony na to pytanie jest mniejszy niż koszt wyjaśniania, dlaczego Twój commit przypadkowo aktywował cudzą konfigurację.

~~~mermaid
flowchart TD
  A["Administrator wpisuje configure"] --> SH["Shared candidate"]
  B["Drugi administrator w configure"] --> SH
  SH --> C["Commit może zatwierdzić<br/>wspólne zmiany"]
  D["Administrator A: configure private"] --> PA["Private candidate A"]
  E["Administrator B: configure private"] --> PB["Private candidate B"]
  PA --> X["Sprawdzenie aktualnego active<br/>oraz konfliktów przy commit"]
  PB --> X
  X --> ACT["Active configuration"]
~~~

*Rysunek 3.2. Wspólny i prywatny model edycji. W prywatnym modelu nadal trzeba uwzględniać zmiany zatwierdzane przez innych administratorów.*

---

## 3.7. Zmiana konfiguracji: `set`, `delete`, `activate`, `deactivate` i zapisywanie [A]

Skoro znamy candidate, wróćmy do podstawowych działań edytora. W Junos masz kilka komend, których działanie warto odróżniać już na początku.

**`set`** tworzy albo zmienia wskazany fragment konfiguracji. **`delete`** usuwa wskazany fragment z candidate. **`deactivate`** pozostawia dany element w konfiguracji, lecz oznacza go jako nieaktywny. **`activate`** przywraca go do użycia. Wszystkie te zmiany wpływają na działającą konfigurację dopiero po odpowiednim commit.

### Najpierw dodanie i usunięcie jednego opisu

**R1-vSRX — Configuration Mode [edit], izolowany lab**

~~~junos
set interfaces ge-0/0/0 description "CH03-TEST"
show interfaces ge-0/0/0 | display set
delete interfaces ge-0/0/0 description
show | compare
~~~

W tym eksperymencie nic nie musisz aktywować. Chodzi o to, żeby zobaczyć, że `set` i `delete` działają na wybranym fragmencie hierarchii. Uważaj na różnicę między **`delete interfaces ge-0/0/0 description`** a **`delete interfaces ge-0/0/0`**. To drugie usuwa całą gałąź portu i może pozbawić go adresacji lub innych ustawień. Nie wpisuj tej szerszej formy do urządzenia, na którym zależy Ci na działaniu łącza.

### `deactivate` nie oznacza „usuń”

Doświadczony administrator może chcieć tymczasowo wyłączyć element konfiguracji, ale zachować go do późniejszego użycia. Wyobraź sobie trasę statyczną przygotowaną jako awaryjna. Zamiast usuwać całą jej składnię, można ją oznaczyć jako inactive.

Na razie wykonamy eksperyment na izolowanym prefiksie dokumentacyjnym **203.0.113.0/24**, nie na prawdziwej sieci klienta.

**R1-vSRX — Configuration Mode [edit], wyłącznie w labie**

~~~junos
set routing-options static route 203.0.113.0/24 discard
deactivate routing-options static route 203.0.113.0/24
show routing-options | display set
show | compare
~~~

Polecenie `discard` w tej demonstracji oznaczałoby odrzucanie pakietów do wskazanej sieci, gdyby trasa była aktywna. Dzięki `deactivate` element pozostaje zapisany w candidate, ale nie powinien działać jako aktywna trasa po commit. Nie używaj tego mechanizmu na produkcyjnym prefiksie bez analizy wpływu.

**R1-vSRX — Configuration Mode [edit], usunięcie własnej próby**

~~~junos
activate routing-options static route 203.0.113.0/24
delete routing-options static route 203.0.113.0/24
show | compare
~~~

Nie zapamiętuj tego jako „pięć linii do wklejenia”. Zapamiętaj różnicę: **deactivate zachowuje ustawienie, delete je usuwa**.

### Zmiana nazwy, kopiowanie i kolejność

Gdy będziesz pracować z większą liczbą portów czy polityk, spotkasz `rename`, `copy`, `insert` i `annotate`. Przydają się do przenoszenia konfiguracji albo zmiany kolejności obiektów, dla których kolejność ma znaczenie — np. termów routing policy lub firewall filter.

Na początku nie używaj tych poleceń globalnie. Kopiowanie portu może również skopiować jego adres IP i inne ustawienia. `insert` nie służy do przesuwania dowolnej linijki jak w zwykłym pliku tekstowym; działa w odpowiednich uporządkowanych listach konfiguracji.

Zobacz bezpieczny przykład kopiowania **tymczasowej prefix-list**, która nie jest nigdzie używana.

**R1-vSRX — Configuration Mode [edit], kopia nieprzypiętej listy**

~~~junos
set policy-options prefix-list CH03-LIST 203.0.113.0/24
copy policy-options prefix-list CH03-LIST to CH03-LIST-COPY
show policy-options | display set
delete policy-options prefix-list CH03-LIST-COPY
delete policy-options prefix-list CH03-LIST
show | compare
~~~

Ta lista nie filtruje żadnych tras ani pakietów, dopóki nie zostanie powiązana z odpowiednią polityką. Dlatego nadaje się do nauki edycji, ale nie jest jeszcze ćwiczeniem z realnego filtrowania. To zagadnienie rozdziału VIII.

### Czy `save` to samo co `commit`?

Nie — to bardzo ważna różnica.

`save` w Configuration Mode zapisuje tekst konfiguracji do wskazanego pliku, np. lokalnego pliku w `/var/tmp`. **Nie aktywuje** wersji roboczej. `commit` natomiast aktywuje zaakceptowaną konfigurację na urządzeniu.

**R1-vSRX — Configuration Mode [edit], tylko w odizolowanym labie, po sprawdzeniu zawartości**

~~~junos
save /var/tmp/ch03-candidate.conf
~~~

To może być przydatne przed eksperymentem, ale nie traktuj lokalnego pliku jako niezależnego backupu: jest na tym samym urządzeniu lub VM. Jeśli stracisz maszynę, plik również może zniknąć. Nie zapisuj w repozytorium pełnych konfiguracji, które zawierają hasła, certyfikaty albo adresy management.

### Czy można przypadkowo zmienić wiele elementów jedną komendą?

Tak. `replace pattern` pozwala zastępować pasujące fragmenty konfiguracji. Na dużym urządzeniu może to być bardzo wygodne, ale zbyt szeroki wzorzec może dotknąć więcej pól, niż zamierzałeś. Dlatego uczymy się go na jednej kontrolowanej wartości.

**R1-vSRX — Configuration Mode [edit], demonstracja w izolowanym labie**

~~~junos
set interfaces ge-0/0/0 description "CH03-OLD-TEXT"
replace pattern CH03-OLD-TEXT with CH03-NEW-TEXT
show | compare
~~~

Nie używaj podobnego polecenia do masowej podmiany adresów, nazw VRF czy policy bez dokładnego diffu i planu rollback. **Składniowo poprawna zmiana może być projektowo błędna.**

---

## 3.8. Odczyt, porównywanie i wczytywanie konfiguracji — jak uniknąć przypadkowej podmiany całego urządzenia? [A]

Dotychczas modyfikowaliśmy pojedyncze linie. Co, jeśli architekt przesłał Ci plik z większą zmianą? W pracy często otrzymasz kilkanaście poleceń `set`, fragment konfiguracji hierarchicznej albo pełną konfigurację przygotowaną na innym routerze. Junos potrafi je wczytywać, ale **tryb wczytywania decyduje o tym, co stanie się z istniejącą candidate**.

Zanim wkleisz plik, zadaj sobie dwa pytania: czy dostałem **fragment**, który chcę dołożyć do obecnej konfiguracji, czy **całą konfigurację**, która ma zastąpić dotychczasową? I czy plik dotyczy naprawdę tego urządzenia, a nie innego modelu, adresacji lub lokalizacji?

### Jak porównać active i candidate?

Najprostsze polecenie poznaliśmy już wcześniej. Teraz nauczymy się czytać jego wynik świadomie.

**R1-vSRX — Configuration Mode [edit], po przygotowaniu wyłącznie własnej zmiany**

~~~junos
show | compare
~~~

W wyniku możesz zobaczyć linie z **`+`** i **`-`**. Plus oznacza dodany element względem porównywanej wersji, minus — usunięty. Jeśli zmienisz opis portu z `OLD` na `NEW`, porównanie może pokazać jedną linię usuniętą i jedną dodaną.

Nie traktuj poprawnego diffu jako gwarancji działania. Diff mówi, **co się zmieni**, ale nie odpowiada automatycznie, czy klient nadal będzie mógł korzystać z aplikacji.

**R1-vSRX — Operational Mode: zatwierdzona konfiguracja przed zmianą**

~~~junos
show configuration interfaces ge-0/0/0 | display set
show system commit
~~~

Zapisz najważniejsze elementy stanu odniesienia przed change. Nie zawsze trzeba zachowywać pełny dump. Przy zmianie opisu portu wystarczy aktualny opis i identyfikacja interfejsu. Przy zmianie BGP trzeba natomiast zgromadzić odpowiednie informacje o sąsiedztwie, trasach i politykach.

### `load set`: gdy plik zawiera linie `set`

Wyobraź sobie, że masz dwie przygotowane linie do R1: opis połączenia do R2 i komunikat dla operatora przy logowaniu. Możesz wpisać je ręcznie albo wczytać je z formatu `set`.

**R1-vSRX — Configuration Mode [edit], początek interaktywnego importu**

~~~junos
load set terminal
~~~

Po tej komendzie CLI przejdzie w tryb przyjmowania tekstu. Wklej tylko następujące dwie linie, dokładnie w tym formacie:

**R1-vSRX — Configuration Mode, tekst wczytywany po `load set terminal` (to dane wejściowe, nie nowy prompt)**

~~~text
set interfaces ge-0/0/0 description "CH03-IMPORTED-R2"
set system login message "CH03: Lab access only"
~~~

Zakończ wprowadzanie danych klawiszem **Ctrl+D**, zgodnie z komunikatem CLI. Następnie wróć do analizy candidate.

**R1-vSRX — Configuration Mode [edit], kontrola importu przed commit**

~~~junos
show | compare
show interfaces ge-0/0/0 | display set
~~~

Obie linie zostały **załadowane do candidate**, a nie od razu aktywowane. To właśnie zaleta tego modelu: możesz obejrzeć ich rzeczywisty wpływ przed zmianą działającego routera.

Jeśli wolisz wczytać istniejący plik lokalny, odpowiednikiem jest `load set /var/tmp/plik.set`, o ile plik naprawdę znajduje się na urządzeniu. W tym rozdziale nie tworzymy pliku przez shell — wystarczy import z terminala. Rozdział o automatyzacji może później wykorzystać pliki w repozytorium, ale bez sekretów i danych produkcyjnych.

### `load merge`: gdy wczytujesz konfigurację w formie drzewa

Junos obsługuje również zapis hierarchiczny, z gałęziami i nawiasami klamrowymi. Przykładowo komunikat logowania w takim zapisie mógłby wyglądać następująco:

~~~text
system {
    login {
        message "CH03: Lab access only";
    }
}
~~~

`load merge` łączy wczytane fragmenty z candidate. Nie jest to to samo co `load set`: oczekiwany format danych jest inny. Gdy nowy fragment zawiera wartość konfliktującą z dotychczasową, może ją zastąpić w odpowiednim miejscu drzewa, dlatego także tu sprawdzamy diff.

**R1-vSRX — Configuration Mode [edit], alternatywny import hierarchiczny**

~~~junos
load merge terminal
~~~

Następnie wklej pokazany fragment hierarchiczny i zakończ **Ctrl+D**. Nie wykonuj tego natychmiast po poprzednim przykładzie, jeśli chcesz zachować dwa niezależne eksperymenty — najpierw wycofaj własną candidate albo pracuj na świeżym snapshotcie labu.

**R1-vSRX — Configuration Mode [edit], kontrola efektu**

~~~junos
show | compare
~~~

### `load replace`: zastępujemy konkretną gałąź — ale tylko gdy oznaczymy ją właściwie

Załóżmy, że w jednym fragmencie masz kilka ustawień, ale nie chcesz dołożyć kolejnych do starego zestawu. Chcesz zastąpić wskazaną gałąź nową definicją.

`load replace` może to zrobić, lecz dokumentacja Juniper przewiduje specjalny znacznik **`replace:`** w wczytywanej konfiguracji. Bez niego operacja może zachowywać się jak merge, a nie jak oczekiwane pełne zastąpienie gałęzi. To bardzo łatwa pułapka.

Przykładowy fragment pokazujący samą ideę:

~~~text
replace: system {
    login {
        message "CH03: Replacement example";
    }
}
~~~

**Uwaga:** powyższy fragment jest demonstracją działania `replace:`, nie poleceniem do bezrefleksyjnego uruchomienia. Zastąpienie całego `system` byłoby niebezpieczne, bo mogłoby usunąć inne ustawienia. W prawdziwym labie znacznik umieszcza się przy **najwęższej właściwej gałęzi**, po sprawdzeniu dokumentacji i pełnego diffu. Na początku wystarczy rozumieć różnicę między *merge* i *replace*, bez aktywowania ryzykownego przykładu.

### `load override`: dlaczego to inna kategoria ryzyka?

`load override` zastępuje **całą candidate configuration** wczytywanym plikiem. Jeśli otrzymałeś tylko dwa polecenia opisujące port i użyjesz override, nowa candidate może nie mieć kont, adresów zarządzania, interfejsów, security policies ani wielu innych ustawień. Po commit skutki byłyby poważne.

**Nie wykonujemy `load override` w tym szkoleniu.** Czytelnik powinien znać jego znaczenie, ale nie musi ryzykować utraty dostępu, by się o nim przekonać.

Pozostałe tryby, takie jak `load update` i `load patch`, są przydatne w bardziej zaawansowanych scenariuszach migracji i automatyzacji. Ważne, żeby nie pomylić `load update` z samym `update` w `configure private`: pierwsze jest sposobem wczytywania konfiguracji z pliku, drugie pomaga synchronizować prywatną candidate z aktualnym stanem.

| Cel | Właściwy mechanizm | Główne ryzyko |
|---|---|---|
| Wkleić kilka linii `set` | `load set terminal` | Nieoczekiwane nadpisanie pojedynczych wartości |
| Dołożyć fragment hierarchiczny | `load merge` | Konflikty z istniejącymi wartościami |
| Zastąpić oznaczony fragment drzewa | `load replace` i `replace:` | Zbyt szeroki zakres zastępowanej gałęzi |
| Zastąpić całą candidate | `load override` | Utrata ustawień nieobecnych w pliku |
| Zmienić tylko różnice według pełnej konfiguracji | `load update` | Mylenie znaczenia z aktualizacją private candidate |
| Wczytać patch w odpowiednim formacie | `load patch` | Nieprawidłowy format lub kontekst patcha |

**Reguła przed każdym importem:** sprawdź format wejścia, zakres zmiany, właściwe urządzenie, różnicę candidate/active oraz plan rollback. `commit check` nie zastępuje oceny, czy fragment pochodzi z właściwej lokalizacji.

---

## 3.9. Rollback — powrót do poprzedniej konfiguracji [A]

Nie każda poprawnie zatwierdzona zmiana daje dobry rezultat. Czasami nowa konfiguracja przechodzi kontrolę składni, ale użytkownicy tracą dostęp do usługi. Wtedy potrzebujesz sposobu, by wrócić do wcześniejszego, znanego stanu.

Junos przechowuje historię zatwierdzonych konfiguracji. Możesz **obejrzeć** starsze wersje, **załadować** wybraną wersję do candidate i dopiero potem ją aktywować. Ten ostatni szczegół jest najważniejszy: **`rollback 1` nie oznacza natychmiastowego cofnięcia działającego routera.**

### Co oznaczają numery rollback?

`rollback 0` odnosi się do aktualnej aktywnej konfiguracji jako punktu odniesienia. Gdy jesteś w Configuration Mode i masz niezacommitowane zmiany, `rollback 0` może odrzucić je, przywracając candidate do active.

`rollback 1` ładuje do candidate poprzednią zatwierdzoną konfigurację. Po czymś takim możesz wykonać `show | compare` i zobaczyć, co zmieniłoby się względem obecnego stanu, gdybyś ten rollback zatwierdził.

Późniejsze numery oznaczają starsze wersje — ale **nie zapamiętuj numeru „1” jako magicznego backupu sprzed całej serii zmian**. Po każdym kolejnym commit historia przesuwa się. Zawsze sprawdź datę i treść konkretnej wersji.

**R1-vSRX — Operational Mode: odczyt historii bez ingerencji w candidate**

~~~junos
show system commit
show system rollback 1
show system rollback compare 0 1
~~~

Pierwsze polecenie pokazuje historię commit. Drugie wyświetla starszą konfigurację. Trzecie pozwala porównać dwie wersje. Na wybranych wydaniach forma widoku może się różnić — sprawdź dostępność w `?`.

### Najprostszy rollback: niezatwierdzona pomyłka

Jesteś w Configuration Mode, dopisałeś błędny opis, ale **nie zrobiłeś commit**. Wtedy nie musisz odtwarzać historii całego urządzenia.

**R1-vSRX — Configuration Mode [edit], tylko gdy jesteś jedynym autorem zmian w candidate**

~~~junos
show | compare
rollback 0
show | compare
~~~

Po `rollback 0` diff powinien być pusty, jeśli kandydat wrócił do active. **Uważaj na zwykły współdzielony `configure`:** jeśli w candidate znajdują się niezacommitowane zmiany innego administratora, `rollback 0` może je również odrzucić. Nie używaj go bez uprzedniego ustalenia, do kogo należą zmiany.

### Gdy błędna konfiguracja już została zatwierdzona

Załóżmy, że zmiana opisu portu została zatwierdzona, a chcesz wrócić do poprzedniej wersji. Najpierw odczytujesz historię. Następnie wczytujesz starszą konfigurację do candidate, sprawdzasz różnicę i dopiero potem decydujesz o jej aktywacji.

**R1-vSRX — Configuration Mode [edit private], wyłącznie po wybraniu właściwego snapshotu**

~~~junos
rollback 1
show | compare
commit check
~~~

Nie dodajemy tu od razu `commit` — najpierw trzeba upewnić się, że historyczna konfiguracja nie usuwa innych, potrzebnych zmian dokonanych od tamtego czasu. W produkcji taki rollback mógłby przypadkowo usunąć cudze poprawki, jeśli nie zrozumiesz, z jakiego momentu pochodzi snapshot.

### Rescue configuration — po co osobna wersja ratunkowa?

Oprócz historii rollback Junos udostępnia specjalną **rescue configuration**. To konfiguracja ratunkowa, którą administrator może zapisać jako znany, użyteczny punkt odtworzenia. Nie jest zapisywana automatycznie po każdym commit.

Ważne, żeby rozróżnić dwie operacje. Najpierw administrator świadomie zapisuje dobrą konfigurację jako rescue. Później, gdy jest potrzeba, może ją wczytać do candidate i zatwierdzić, po weryfikacji.

**R1-vSRX — Operational Mode, zapis rescue wyłącznie po zatwierdzeniu baseline i zgodzie właściciela labu**

~~~junos
request system configuration rescue save
~~~

**R1-vSRX — Configuration Mode [edit], przywracanie rescue jako candidate — nie wykonywać bez potrzeby**

~~~junos
rollback rescue
show | compare
~~~

Rescue jest bardzo pomocna, ale nie oznacza pełnego backupu sprzętu, plików użytkowników, licencji czy zewnętrznych systemów. Nie zastępuje też niezależnego dostępu konsolowego, bo błędna konfiguracja może odciąć SSH zanim zdążysz jej użyć.

### Rollback nie jest cofnięciem czasu dla wszystkich elementów urządzenia

Załóżmy, że zmieniono konfigurację i wykonano upgrade oprogramowania. Wczytanie starszej konfiguracji nie przywraca automatycznie starszego obrazu Junos. Podobnie rollback nie zawsze odtwarza stan dynamicznych sesji bezpieczeństwa, liczników czy procesów.

Dlatego w procedurach operacyjnych odróżniamy **rollback konfiguracji** od pełnego planu przywrócenia usługi. Ten ostatni może wymagać także testów protokołów, ścieżek, stanu systemu i pracy aplikacji.

---

## 3.10. J-Web — administracja przez przeglądarkę [A]

Nie każda osoba zarządzająca Junos pracuje wyłącznie przez terminal. Juniper udostępnia również **J-Web** — graficzny interfejs dostępny przez przeglądarkę na wspieranych urządzeniach i wydaniach. Możesz w nim przeglądać informacje o systemie, monitorować wybrane elementy i przygotowywać konfigurację.

J-Web jest szczególnie przydatny, kiedy uczysz się układu urządzenia lub chcesz pokazać mniej doświadczonemu administratorowi stan interfejsów. W niektórych sytuacjach prostszy będzie jednak CLI, zwłaszcza podczas analizy wielu urządzeń, diffów i logów.

Nie zakładaj, że każdy model Juniper ma identyczny wygląd J-Web. **SRX i EX mogą mieć inne widoki, a dostępność funkcji zależy od Junos release oraz zainstalowanego pakietu J-Web.** Ekran opisany w instrukcji dla innej rodziny urządzeń nie musi odpowiadać Twojemu vSRX.

### Jak bezpiecznie korzystać z GUI?

Połączenie z J-Web powinno być szyfrowane przez **HTTPS** i dopuszczone tylko ze znanego segmentu zarządzania. Do działania może być potrzebna poprawna konfiguracja usługi `web-management`, certyfikatu oraz reguł dopuszczających dostęp do urządzenia. Na SRX dochodzą kwestie `host-inbound-traffic`.

Nie otwieraj J-Web przez zwykłe HTTP tylko dlatego, że HTTPS wymaga dodatkowej konfiguracji. Nie wystawiaj też panelu na wszystkie interfejsy w sieci ćwiczeniowej lub, co gorsza, do Internetu.

**R1-vSRX — Operational Mode: sprawdź najpierw, czy J-Web w ogóle jest skonfigurowany**

~~~junos
show version
show configuration system services web-management | display set
show configuration security zones | display set
show interfaces terse
~~~

Jeżeli pierwsze polecenie potwierdza wspierany release, a drugie pokazuje konfigurację HTTPS, masz część potrzebnych informacji. Nadal nie jest to dowód, że panel działa: sprawdź certyfikat, właściwą strefę, adres management i dostępność TCP na wybranym porcie.

### Dlaczego nie podajemy jednej uniwersalnej ścieżki menu J-Web?

Interfejs J-Web zmieniał się między wersjami i rodzinami sprzętu. Nie byłoby uczciwe podać jako uniwersalnej ścieżki `Configure → Interfaces`, jeśli na Twoim obrazie funkcja znajduje się w innym miejscu albo nie jest zainstalowana.

W odizolowanym labie zastosuj następującą procedurę: otwórz stronę HTTPS pod **zweryfikowanym adresem zarządzania**, zaloguj się uprawnionym kontem, znajdź sekcję monitorowania interfejsów i porównaj stan portu `ge-0/0/0` z `show interfaces terse` w CLI. Jeśli w danym release dostępne jest porównanie zmian konfiguracji, sprawdź, czy GUI prezentuje ten sam zakres zmiany, który widzisz w CLI przez `show | compare`.

**Nie włączamy J-Web jako obowiązkowej części bazowej konfiguracji DIAMOND-6R.** Ten rozdział dotyczy przede wszystkim zasad CLI; GUI pozostaje wariantem opcjonalnym na wspieranej platformie. Dla konkretnych kliknięć korzystaj z dokumentacji producenta właściwej dla zidentyfikowanego release: [J-Web for SRX and EX](https://www.juniper.net/documentation/product/us/en/j-web/), [Secure Web Access](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/secure-web-access-remote-management.html).

---

## 3.11. Bezpieczny sposób wprowadzania zmian: diff, check, confirmed, test i dopiero zatwierdzenie [EXT]

Do tego momentu znasz już elementy potrzebne do wykonania zmiany. Teraz połączymy je w jeden proces, który może stać się Twoim codziennym nawykiem.

Przedstawmy zmianę trochę ważniejszą niż opis portu. Administrator chce zmodyfikować adresowanie lub filtry w drodze do serwera. Jeśli się pomyli, może stracić połączenie SSH do routera. **Nie wystarczy sprawdzić, że konfiguracja jest poprawna składniowo.** Trzeba zaplanować sposób przywrócenia dostępu, jeśli test po aktywacji zakończy się niepowodzeniem.

### Co oznacza `commit confirmed`?

Zwykły `commit` aktywuje zmianę i pozostawia ją w konfiguracji, chyba że później wykonasz kolejne działania. **`commit confirmed`** aktywuje ją warunkowo: daje określony czas na potwierdzenie, że wszystko działa. Jeśli nie otrzyma potwierdzenia, Junos powinien automatycznie przywrócić poprzednią konfigurację.

Wyobraź sobie, że wykonujesz `commit confirmed 5`. System ma pięć minut na otrzymanie potwierdzenia. To czas na sprawdzenie, czy router pozostał osiągalny i czy usługa działa. Jeśli popełniłeś błąd i straciłeś SSH, mechanizm rollback może pomóc odzyskać poprzednią konfigurację.

Nie należy jednak traktować tego jak gwarancji, że w każdym przypadku wszystko „naprawi się samo”. Dostępność konsoli, stan systemu, sposób zapisu i specyfika platformy nadal mają znaczenie. Nie przeprowadzaj pierwszego takiego testu na routerze produkcyjnym.

### Jedna szczególnie ważna pułapka: `commit check` po `commit confirmed`

Zanim aktywujesz zmianę, `commit check` jest rozsądnym krokiem. Ale **po wykonaniu `commit confirmed` jego rola może być inna**. Zgodnie z dokumentacją Juniper, również `commit check` wykonany podczas oczekiwania może **potwierdzić warunkowy commit**, zatrzymując automatyczny rollback.

Właśnie dlatego kolejność działań ma znaczenie:

1. Przejrzyj `show | compare`.
2. Uruchom `commit check` **przed** aktywacją.
3. Wykonaj `commit confirmed 5`.
4. Sprawdź **działanie**, korzystając z poleceń operacyjnych, konsoli lub niezależnego testu klienta — bez kolejnego `commit check`.
5. Dopiero po sukcesie zatwierdź nową konfigurację poleceniem `commit`.

Pierwsze dwa kroki sprawdzają poprawność projektu zmian przed aktywacją. Czwarty odpowiada na pytanie, czy system działa poprawnie po aktywacji. Tych dwóch rodzajów weryfikacji nie wolno zastępować jednym testem.

~~~mermaid
flowchart TD
  START["Przygotuj zmianę w candidate"] --> DIFF["show | compare"]
  DIFF --> CHECK["commit check — PRZED aktywacją"]
  CHECK --> CC["commit confirmed 5"]
  CC --> TEST{"Czy działa dostęp i usługa?"}
  TEST -->|Tak| COMMIT["commit — potwierdzenie"]
  TEST -->|Nie| WAIT["Nie potwierdzaj; rollback po timerze<br/>lub kontrolowany powrót przez konsolę"]
  COMMIT --> END["Zapisz wynik i dowody"]
  WAIT --> END
~~~

*Rysunek 3.3. Bezpieczny model zmiany zdalnej. Podczas oczekiwania po `commit confirmed` nie uruchamiaj kolejnego `commit check`, jeśli chcesz zachować możliwość automatycznego cofnięcia.*

### Jak wygląda to na prostym przykładzie?

Użyjmy bezpiecznego ustawienia — komunikatu wyświetlanego użytkownikowi przy logowaniu. To nie jest zmiana routingu ani polityki bezpieczeństwa, ale pozwoli przećwiczyć mechanizm.

**R1-vSRX — Operational Mode: wejdź do prywatnej konfiguracji**

~~~junos
configure private
~~~

**R1-vSRX — Configuration Mode [edit private]: przygotowanie i kontrola przed aktywacją**

~~~junos
set system login message "CH03: Test warunkowego commit"
show | compare
commit check
commit confirmed 5
~~~

Teraz **nie wpisuj niczego, co mogłoby potwierdzić zmianę**. Przejdź do odczytu operacyjnego z osobnej sesji, której nie używasz do edycji.

**R1-vSRX — Operational Mode, osobna sesja: post-check podczas timera**

~~~junos
show system commit
show configuration system login message
show system uptime
~~~

Jeśli potwierdzisz, że zmiana jest zgodna z planem i nie ma wpływu na działanie labu, możesz ją zachować.

**R1-vSRX — Configuration Mode [edit private]: potwierdzenie dopiero po PASS**

~~~junos
commit
~~~

Nie wykonuj tej ostatniej komendy, jeśli celem konkretnego eksperymentu jest obserwacja automatycznego rollback. W takim przypadku pozwól timerowi upłynąć i potwierdź powrót poprzedniej wersji.

### Co, jeśli w mojej wersji Junos wymagany jest komentarz do commitu?

W niektórych nowszych wydaniach Junos zasady commit mogą wymagać komentarza do zmiany. W takim przypadku składnia musi być dostosowana do konkretnego release. Przykładowy opis jest przydatny również ze względów operacyjnych: następny administrator widzi, dlaczego wprowadzono dany stan.

**R1-vSRX — Configuration Mode [edit]: przykład jawnego komentarza przy finalnym zatwierdzeniu**

~~~junos
commit comment "CH03: validated login message"
~~~

Nie przedstawiamy jednak jednolitego obowiązku komentarza jako reguły każdego starego vSRX. Przed wykonaniem ćwiczenia sprawdź `show version` i właściwą dokumentację polecenia `commit`.

### Co wpisujemy do worknotes po zmianie?

W profesjonalnym change nie wystarczy informacja „commit complete”. Zapisz, **który router i element konfiguracji zostały zmienione**, co pokazał diff, czy walidacja przeszła, czy management pozostał osiągalny, jakie testy zostały wykonane i czy warunkowy commit został potwierdzony.

Możesz opisać również plan cofnięcia: wykorzystanie `commit confirmed`, wskazanie snapshotu z historii lub punktowy powrót do stanu pierwotnego. Ale nie twierdź, że wykonano rollback, jeśli był tylko przygotowany jako zabezpieczenie.

### `commit` to nie to samo co test aplikacji

Nawet najlepszy workflow konfiguracji nie zastąpi testu działania. Jeśli po zmianie filtra użytkownik nie może otworzyć strony, udany commit niczego nie naprawia. Wtedy należy szukać przyczyny i wracać do stanu bezpiecznego zgodnie z planem. To samo dotyczy zmiany interfejsu w górnym korytarzu DIAMOND-6R — warto sprawdzić nie tylko R1, lecz także R2, R4 i R6, jeżeli zmiana miała wpływ na ich ruch.

**Źródła Juniper:** [Commit the Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html), [commit CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html), [Configure Command Overview](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configure-command.html).

---

## 3.LAB. Laboratorium w trzech poziomach — nie ucz się change management na produkcyjnym routerze [EXT]

Wiedzę o CLI najłatwiej przyswoić przez małe, bezpieczne eksperymenty, w których widać skutek każdej zmiany. Laboratorium podzielimy na trzy poziomy. W pierwszym potrzebujesz tylko jednego vSRX. W drugim sprawdzisz współpracę administratorów i drugie urządzenie. W trzecim wykorzystasz cały DIAMOND-6R, ale bez dokładania nowej adresacji i skomplikowanych protokołów.

**Warunek bezpieczeństwa:** pracujemy na nowym, odizolowanym labie. Nie modyfikujemy istniejących plików `.unl`, połączeń management, globalnych bridge EVE-NG ani produkcyjnych urządzeń. Nie zakładamy, że każdy obraz vSRX obsługuje te same opcje. **RUNTIME VALIDATED** można wpisać dopiero po rzeczywistym wykonaniu testów na znanym release, ze zrzutami wyników.

### Preflight — co sprawdzić jeszcze przed edycją?

Zacznij od R1. Powinieneś mieć niezależny dostęp konsolowy i, jeśli to możliwe, drugą sesję administracyjną. Nie używaj `commit confirmed` w scenariuszu utraty dostępu, dopóki nie sprawdzisz konsoli w praktyce.

**R1-vSRX — Operational Mode: odczyt przed laboratorium**

~~~junos
show version
show system users
show system commit
show configuration system login message
show configuration interfaces ge-0/0/0 | display set
show interfaces terse
~~~

Zapisz zanonimizowane wyniki: wersję Junos, stan interfejsów i obecny tekst login message. Jeśli wcześniej skonfigurowano inne elementy, które kolidują z naszym ćwiczeniem, nie kasuj ich globalnie. Pracuj na nowej kopii R1 lub dostosuj eksperyment po rozpoznaniu realnego stanu.

### Poziom 1. Jeden vSRX: zobacz różnicę między candidate i active

W pierwszej fazie przygotujemy opis portu i nie zatwierdzimy go. Jeśli wszystko zrobisz poprawnie, `show | compare` pokaże nowy opis, a `show configuration` w Operational Mode nadal pokaże starą konfigurację.

**R1-vSRX — Operational Mode: wejdź do edytora**

~~~junos
configure
~~~

**R1-vSRX — Configuration Mode [edit]: tylko wersja robocza**

~~~junos
set interfaces ge-0/0/0 description "CH03-LEVEL1"
show interfaces ge-0/0/0
show | compare
~~~

**R1-vSRX — Operational Mode, druga sesja: aktywna konfiguracja**

~~~junos
show configuration interfaces ge-0/0/0 | display set
~~~

Jeśli to, co widzisz, zgadza się z przewidywaniem, możesz zatwierdzić wyłącznie tę zmianę — pod warunkiem, że candidate nie zawiera cudzego diffu.

**R1-vSRX — Configuration Mode [edit], tylko po przejrzeniu diffu**

~~~junos
commit check
commit
~~~

Teraz porównaj aktywny opis z poprzednim odczytem. Następnie przygotuj jego usunięcie, ale **nie wykonuj commit**. Zamiast tego sprawdź działanie `rollback 0`.

**R1-vSRX — Configuration Mode [edit]: cofnięcie tylko własnej niezacommitowanej próby**

~~~junos
delete interfaces ge-0/0/0 description
show | compare
rollback 0
show | compare
~~~

Jeżeli nie ma cudzych zmian, diff powinien zniknąć. Router nadal zachowuje zatwierdzony wcześniej opis `CH03-LEVEL1`. Uczysz się w ten sposób, że `rollback 0` **nie cofa aktywnej zmiany z wcześniejszego commit**; odrzuca aktualną różnicę candidate.

### Poziom 1, dodatkowe ćwiczenie: automatyczny rollback bez ryzyka utraty sieci

W tym samym R1 zmienimy komunikat logowania i **celowo nie potwierdzimy** warunkowego commit.

**R1-vSRX — Configuration Mode [edit], przygotowanie na czystej candidate**

~~~junos
set system login message "CH03: AUTO-ROLLBACK-TEST"
show | compare
commit check
commit confirmed 2
~~~

Teraz nie wykonuj `commit` ani `commit check`. Odczytaj bieżącą konfigurację z osobnej sesji i sprawdź historię. Odczekaj, aż warunkowa zmiana powinna zostać automatycznie cofnięta, a potem ponownie sprawdź stan.

**R1-vSRX — Operational Mode, druga sesja: obserwacja przed i po timerze**

~~~junos
show system commit
show configuration system login message
~~~

Dopiero gdy widzisz, że poprzedni komunikat powrócił, możesz odnotować rezultat testu. Jeśli oczekiwane cofnięcie nie nastąpiło, nie nazywaj go PASS — sprawdź, czy przypadkiem ktoś nie potwierdził commitu, czy timer jeszcze trwa albo czy w tej wersji systemu wystąpił problem.

To ćwiczenie jest ważniejsze niż zapamiętanie samej składni `commit confirmed 2`. Uczysz się **różnicy między aktywacją warunkową a trwałym zatwierdzeniem**.

