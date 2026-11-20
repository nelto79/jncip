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

