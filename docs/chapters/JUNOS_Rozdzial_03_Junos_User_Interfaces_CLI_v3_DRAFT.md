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

