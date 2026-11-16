# Rozdział 3. Junos User Interfaces — CLI, konfiguracja, bezpieczny change i J-Web

**Podręcznik:** JUNOS Enterprise Routing & Switching · JNCIA-Junos → JNCIS-ENT → JNCIP-ENT  
**Zakres Unified Blueprint:** **3.1–3.10 [A]** oraz **3.11 [EXT]**  
**Redakcja:** 16 listopada 2026 · materiał wykładowy, Cisco IOS/IOS XE → Junos OS  
**Platforma labowa:** vSRX w EVE-NG; odpowiedniki Cisco IOS XE oraz uwagi dla EX/QFX/MX i Junos OS Evolved  
**Status weryfikacji:** dokumentacja producenta sprawdzona; przykłady konfiguracji i laboratorium są **PLANNED / NOT RUNTIME VALIDATED** (nie wykonano ich na żywym vSRX).  
**Następna lekcja:** Rozdział 4 — initial configuration, użytkownicy, AAA, usługi i rescue configuration.

> **Umowa z uczniem:** nie będziemy zapamiętywać przypadkowych komend. Każda komenda odpowiada na konkretne pytanie: **co chcę zmienić, co system przygotował, co rzeczywiście działa, co może się nie udać i jak odzyskać dostęp?** Masz doświadczenie Cisco, więc wykorzystamy je jak trampolinę. Nie będziemy od nowa uczyć się routingu, tylko innego sposobu prowadzenia zmiany.

## 03:07 — incydent, od którego zaczyna się ta lekcja

W środku nocnego maintenance window masz przygotować nowe peeringi BGP, dopisać trzy prefiksy i zmodyfikować filtr wejściowy. Na Cisco IOS XE wydajesz kolejne polecenia \`configure terminal\`. Większość z nich działa od razu na bieżącej konfiguracji. Ktoś pomylił kolejność, zastosował błędny filtr lub niechcący odciął SSH. Teraz dowiadujesz się, jak dobra była Twoja ścieżka OOB i czy uruchomiłeś mechanizm bezpiecznego cofania zmian.

Na Junos także możesz odciąć sobie sieć. **To nie jest system magicznie wolny od błędów operatora.** Ale filozofia pracy zmienia się radykalnie: konfiguracja przygotowywana jest jako **candidate**, a dopiero commit aktywuje ją jako nową **active configuration**. Zanim naciśniesz przycisk, możesz zobaczyć różnicę, zweryfikować poprawność i ustawić automatyczne przywrócenie poprzedniej wersji.

Wyobraź sobie pilota samolotu. Pilot nie zmienia ustawień nawigacji całego lotniska za każdym razem, gdy edytuje plan lotu na ekranie. Najpierw przygotowuje i sprawdza plan, potem go aktywuje. **Candidate** jest planem przygotowanym; **active** — planem, według którego system aktualnie pracuje. Ale ważne: nawet plan zatwierdzony przez komputer może poprowadzić Cię na niewłaściwe lotnisko, jeśli podałeś niewłaściwy cel. To ograniczenie \`commit check\`.

### Zanim przejdziemy do CLI — trzy rodzaje wiedzy

**Na pamięć:** znaki promptu \`>\` i \`#\`; operational vs configuration; \`set/delete/activate/deactivate\`; \`show | compare\`; \`commit\`, \`commit confirmed\`, \`rollback 0/1\`; \`configure private\`; różnice \`load merge/replace/override/set\`.

**Do zrozumienia:** shared vs private candidate; dlaczego \`commit check\` nie jest testem reachability; co oznacza „zapisano plik” vs „zatwierdzono konfigurację”; czemu utrata sesji po \`commit confirmed\` nie zawsze oznacza, że SSH samo odżyje.

**Do przećwiczenia:** wejście do hierarchii, szybkie \`?\` i \`Tab\`, filtrowanie outputu, porównanie dwóch konfiguracji, bezpieczny import fragmentu, dwa równoległe logowania administracyjne, celowy rollback i weryfikacja historii zmian.

~~~mermaid
flowchart LR
  subgraph IOS["Cisco IOS/IOS XE — typowy ręczny workflow"]
    A1["conf t"] --> A2["polecenie 1, 2, 3..."]
    A2 --> A3["zmiany zwykle od razu w running-config"]
    A3 --> A4["test / ewentualnie rollback mechanizmami Cisco"]
  end
  subgraph JUN["Junos OS — workflow transakcyjny"]
    B1["configure"] --> B2["candidate: set/delete/load"]
    B2 --> B3["show | compare"]
    B3 --> B4["commit check"]
    B4 --> B5["commit confirmed"]
    B5 --> B6["test produkcyjny"]
    B6 --> B7["commit lub automatyczny rollback"]
  end
~~~

*Diagram dydaktyczny: na wybranych platformach IOS XE istnieją również archiwizacja, configuration replace i confirmed rollback. Porównanie przedstawia typowy styl pracy CLI, a nie twierdzenie, że Cisco nie ma mechanizmów transakcyjnych.*

---

## 3.1. Funkcje CLI i zasady interakcji z systemem [A]

### 3.1.1. Dlaczego Junos CLI nie przypomina „samego routera z komendami”

W poprzednim rozdziale poznaliśmy RE i daemony. CLI nie jest osobną magiczną instancją routingu: jest interfejsem człowieka do stanu urządzenia i systemu konfiguracji, powiązanym z usługami zarządzania (m.in. \`mgd\`). Możesz zarządzać urządzeniem przez konsolę, SSH, protokoły automatyzacyjne, a na wspieranych platformach także J-Web.

Przy pierwszym logowaniu do Junos zwykle widzisz prompt podobny do \`user@R1>\`. Pierwsza ważna różnica względem Cisco: **nie używasz polecenia \`enable\`**, by podnieść uprawnienia. Klasa użytkownika / autoryzacja określa, jakie polecenia możesz wykonywać. Sam prompt \`>\` **nie oznacza**, że masz uprawnienia superuser.

**Most z Cisco:** w IOS/IOS XE przyzwyczaiłeś się do user EXEC, privileged EXEC i global configuration. W Junos rozdział opiera się na **trybie operational i configuration**, a poziom uprawnień wynika z konta/klasy oraz konfiguracji uwierzytelniania i autoryzacji. Pełne login classes i TACACS/RADIUS pojawią się w rozdziale 4.

### 3.1.2. Trzy środowiska, z których nie wszystkie są „trybem CLI”

**Operational CLI** to interfejs diagnostyczny/zarządczy. **Configuration CLI** to edytor hierarchii. **System shell** to środowisko leżącego pod Junos systemu operacyjnego: zwykle powłoka FreeBSD w klasycznym Junos, a na Junos OS Evolved — środowisko Linux.

Nie nazywaj każdego promptu z \`%\` „trybem jądra”. To powłoka **użytkownika systemu operacyjnego**, nie jądro. I nie zakładaj, że na każdej platformie shell musi wyglądać identycznie.

**Urządzenie: R1-vSRX — Operational Mode (przykładowa sesja, ilustracyjna)**

~~~text
user@R1-vSRX> show version
user@R1-vSRX> show cli
user@R1-vSRX> show system users
~~~

\`show cli\` odpowiada na pytanie o ustawienia sesji (terminal, screen length/width, completion). \`show system users\` pomaga zobaczyć, kto jeszcze jest zalogowany. Właśnie od tego zaczyna się dobry precheck do nocnego change'u.

**Pytanie kontrolne:** czy \`>\` oznacza „bezpieczny read-only”? **Nie.** W operational mode możesz także wykonać \`clear\`, \`request\`, wyłączenie urządzenia i inne działania zmieniające stan systemu, zależnie od uprawnień. Polecenia **nie edytują candidate configuration**, ale to nie znaczy, że nie mogą wywołać przestoju.

### 3.1.3. Shell i gotcha z bezpieczeństwem

**Urządzenie: R1-vSRX — Operational Mode, tylko w LAB z odpowiednimi uprawnieniami**

~~~junos
start shell
~~~

**Urządzenie: R1-vSRX — shell systemowy, przykład klasycznego Junos**

~~~sh
pwd
exit
~~~

Nie używaj shella do rutynowego monitoringu, skoro istnieją komendy CLI. Nie edytuj \`juniper.conf\` bezpośrednio edytorem systemowym ani nie traktuj dostępu do shella jako odpowiednika \`configure\`. Ewentualne problemy z plikami i recovery wymagają jawnej procedury oraz zgodności z wersją.

**Gotcha Cisco:** \`start shell\` nie jest „bezpieczniejszym enable”. To przejście do innej warstwy systemu, gdzie błędy nie są objęte zwykłą transakcją \`candidate/commit\`.

---

## 3.2. Operational mode vs configuration mode [A]

### 3.2.1. Co naprawdę zmienia znak promptu?

W Junos zwykle zobaczysz:

- \`user@host>\`: **operational** — obserwacja i działania operacyjne.
- \`[edit]\` nad \`user@host#\`: **configuration** — budowanie/zmiana candidate.

Bardzo ważne: na Cisco \`#\` kojarzy Ci się z uprzywilejowanym EXEC. Na Junos jest znakiem promptu konfiguracji. Nie próbuj bezmyślnie kopiować poleceń Cisco z \`#\` do Junos \`#\`.

**Urządzenie: R1-vSRX — Operational → Configuration Mode**

~~~junos
configure
~~~

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
show
top
run show version
exit
~~~

Co właśnie zrobiłeś? **\`show\` w configuration mode** pokazuje kontekst konfiguracji, a **\`run show version\`** pozwala wykonać polecenie operacyjne bez opuszczania edytora. \`top\` przenosi do korzenia hierarchii. \`exit\` na głównym poziomie wraca do operational; jeśli w candidate pozostały niezatwierdzone zmiany, system może poprosić o potwierdzenie opuszczenia trybu.

**Most z Cisco:** w \`(config)#\` do komend operacyjnych często używasz \`do show ...\`. W Junos najbliższym funkcjonalnym odpowiednikiem jest \`run show ...\`.

### 3.2.2. Istotny wyjątek: nie wszystkie polecenia operational wolno poprzedzić run

Przykładowo dokumentacja polecenia \`show system rollback\` zaznacza, że **nie działa ono przez \`run\` z configuration mode**. Dlatego zasada „wystarczy dać \`run\` przed dowolnym operational show” byłaby zbyt prosta.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show system rollback 1
show system commit
~~~

**Zapamiętaj:** \`run\` rozwiązuje większość codziennych potrzeb, ale CLI ma ograniczenia poleceń. W razie problemu sprawdź dokumentację konkretnej komendy, zamiast szukać „błędu Junos”.

### 3.2.3. Jak nie pomylić edycji z natychmiastową zmianą sieci

Wpisanie \`set interfaces ...\` w configuration mode zazwyczaj zmienia **candidate**, nie bieżący forwarding. Dopóki nie wykonasz skutecznego commit, Active pozostaje bez zmian. To nie znaczy, że wszystkie polecenia w configuration mode są nieszkodliwe: **\`commit\`, \`load override\`, \`rollback\` i polecenia masowej edycji mogą mieć poważne skutki po zatwierdzeniu**.

Urządzenie i jego wersja mogą także posiadać mechanizmy dynamicznej/ephemeral configuration. Na poziomie tego rozdziału uczymy się **standardowej bazy konfiguracji Junos**, z którą najczęściej pracujesz w CLI.

### 3.2.4. Prosta próba z własną hipotezą

**Urządzenie: R1-vSRX — Configuration Mode, wyłącznie izolowany LAB**

~~~junos
set system login message "LAB CH03 - test candidate"
show | compare
rollback 0
show | compare
~~~

Przewidywanie: po \`set\` diff będzie zawierał dodany login message; po \`rollback 0\` różnica z active zniknie (zakładając brak innych zmian). **Żaden commit nie został wykonany**, więc aktywna konfiguracja nie została zmieniona.

Jeśli nie zniknie — sprawdź, czy edytujesz wspólną konfigurację równocześnie z inną sesją, czy pozostajesz w innym edit level, czy w międzyczasie ktoś wykonał commit.



---

## 3.3. Nawigacja po hierarchii konfiguracji i poleceniach CLI [A]

### 3.3.1. Dlaczego hierarchia jest wielką zaletą — i pierwszą pułapką inżyniera Cisco?

Na Cisco znasz takie podejście: \`interface GigabitEthernet1/0/12\`, potem \`description\`, \`switchport mode access\`, następnie \`exit\` i kolejna sekcja. W Junos prawie cała konfiguracja jest **drzewem**, a nie jedynie sekwencją poleceń. Zamiast zapamiętać plik od góry do dołu, możesz odnaleźć właściwą **gałąź**: \`system\`, \`interfaces\`, \`routing-options\`, \`protocols\`, \`policy-options\`, \`security\`.

Wyobraź sobie wielki biurowiec, a w nim pokoje. Jeśli przebywasz w pokoju \`[edit interfaces ge-0/0/1]\` i dopisujesz \`set description "UPLINK"\`, to edytujesz **opis tylko tego interfejsu**. Jeżeli z tego miejsca zaczniesz wpisywać \`set system host-name...\`, polecenie nie odwoła się automatycznie do root. W Junos **kontekst ma znaczenie**.

Zobacz najpierw jeden fragment hierarchii. To **przykład struktury**, a nie wynik odczytany z Twojego vSRX.

**Urządzenie: R1-vSRX — przykład widoku Configuration Mode [edit interfaces]**

~~~junos
interfaces {
    ge-0/0/1 {
        description "UPLINK-TO-R2";
        unit 0 {
            family inet {
                address 10.12.0.1/30;
            }
        }
    }
}
~~~

\`ge-0/0/1\` to port fizyczny, \`unit 0\` to logiczny interfejs, \`family inet\` określa rodzaj adresowania, a \`address\` konkretną wartość. Dla Cisco intuicyjnie możesz to porównać do interfejsu z adresem L3, ale **Juniper rozdziela interfejs fizyczny, logiczną jednostkę i family**.

Teraz to samo zobaczymy jako polecenia **set**:

**Urządzenie: R1-vSRX — Configuration Mode [edit], izolowany LAB**

~~~junos
set interfaces ge-0/0/1 description "UPLINK-TO-R2"
set interfaces ge-0/0/1 unit 0 family inet address 10.12.0.1/30
show interfaces ge-0/0/1
show | compare
~~~

Jeśli R1 ma już taką adresację od wcześniejszego laboratorium, traktuj te linie jako **odczyt/rekonstrukcję istniejącej konfiguracji**, nie jako bezrefleksyjną zmianę. Prawidłowy workflow zaczyna się od \`show interfaces ge-0/0/1\` oraz sprawdzenia rzeczywistej roli portu.

### 3.3.2. Poruszamy się po drzewie — praktyka zamiast listy pojęć

Wolisz wejść do pokoju i pisać krócej? Tak, ale pamiętaj, gdzie jesteś.

**Urządzenie: R1-vSRX — Configuration Mode, demonstracja na istniejącym interfejsie**

~~~junos
top
edit interfaces ge-0/0/1
show
up
top
show interfaces ge-0/0/1
~~~

Po \`edit interfaces ge-0/0/1\` prompt powinien wskazywać \`[edit interfaces ge-0/0/1]\`; \`up\` wraca do \`[edit interfaces]\`; \`top\` przenosi do \`[edit]\`. Nie musisz powtarzać pełnej ścieżki, gdy jesteś głęboko zagnieżdżony.

**Haczyk:** \`exit\` **nie zawsze wychodzi z configuration mode**. W zagnieżdżonym miejscu zwykle przechodzi poziom wyżej; dopiero przy \`[edit]\` kończy tryb konfiguracji. Jest też jawne \`exit configuration-mode\`.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
top
edit interfaces ge-0/0/1 unit 0
up
exit
top
exit
~~~

Ostatnie \`exit\` kończy tryb configuration, ale przy niezapisanych zmianach może pojawić się pytanie. **Nie ucz się mechanicznie polecenia \`exit\` bez obserwacji \`[edit ...]\` nad promptem.**

### 3.3.3. \`top\`, \`up\`, \`edit\`, \`run\` — mapa piętra

~~~mermaid
flowchart TB
  ROOT["[edit]"] --> SYS["[edit system]"]
  ROOT --> IF["[edit interfaces]"]
  IF --> GE["[edit interfaces ge-0/0/1]"]
  GE --> UNIT["[edit interfaces ge-0/0/1 unit 0]"]
  UNIT --> IP["[edit interfaces ge-0/0/1 unit 0 family inet]"]
  IP -. "up" .-> UNIT
  IP -. "top" .-> ROOT
  GE -. "edit unit 0" .-> UNIT
  GE -. "run show interfaces terse" .-> OP["Operational command"]
~~~

Przygodne pytanie instruktora: stoisz w \`[edit interfaces ge-0/0/1 unit 0 family inet]\`. Chcesz sprawdzić, czy routing table pokazuje trasę connected. Czy musisz wyjść pięć razy przez \`exit\`? **Nie.** Używasz \`run show route\`.

**Urządzenie: R1-vSRX — Configuration Mode [edit interfaces ge-0/0/1 unit 0 family inet]**

~~~junos
run show route
top
~~~

### 3.3.4. Wybieranie dokładnej ścieżki i pułapka z \`delete\`

Załóżmy, że w hierarchii portu znajduje się opis, MTU i adres. Chcesz usunąć sam opis. Na Cisco to \`no description\`. W Junos to \`delete interfaces ge-0/0/1 description\` z root konfiguracji albo po wejściu do interfejsu po prostu \`delete description\`.

**Urządzenie: R1-vSRX — Configuration Mode [edit], izolowany LAB**

~~~junos
delete interfaces ge-0/0/1 description
show | compare
~~~

**Gotcha:** \`delete interfaces ge-0/0/1\` usuwa **całą gałąź**, w tym adres, unit i inne parametry. To fundamentalna różnica między odłączeniem jednego liścia a wycięciem całego drzewa. Dlatego zanim wykonasz \`delete\`, przeczytaj ścieżkę i sprawdź \`show | compare\`.

Jeżeli komenda była jedynie ćwiczeniem, cofnij niezatwierdzoną zmianę:

**Urządzenie: R1-vSRX — Configuration Mode [edit], tylko jeśli nie chcesz zachować żadnych niezacommitowanych zmian**

~~~junos
rollback 0
show | compare
~~~

**Nie rób \`rollback 0\` w shared candidate, jeśli inni administratorzy mają tam własne niezatwierdzone poprawki.**

---

## 3.4. Pomoc kontekstowa i uzupełnianie poleceń [A]

### 3.4.1. Najbardziej niedoceniane narzędzie inżyniera: nie Google, tylko \`?\`

W Cisco możesz wpisywać \`?\` i wybierać możliwe opcje. Junos również oferuje pomoc kontekstową, często bardzo bogatą. Podczas egzaminu, laboratorium i change'u **nie musisz pamiętać pełnej składni każdej komendy**. Musisz wiedzieć, gdzie znaleźć dalszy element drzewa.

Powiedzmy, że nigdy wcześniej nie konfigurowałeś konkretnie \`system login message\`. Nie googluj przypadkowego wpisu z forum. Zapytaj CLI o możliwe następne parametry.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show ?
show interfaces ?
help apropos "commit"
help topic interfaces
~~~

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
top
set system ?
set interfaces ?
help reference system
~~~

**Uwaga:** znaki \`?\` oznaczają interaktywny mechanizm pomocy. Jeżeli wgrywasz plik poleceń, nie traktuj tych przykładów jako linijek konfiguracji do \`load set\`. Dostępność konkretnych tematów \`help topic\`/\`help reference\` zależy od wydania i obecności pakietu dokumentacji.

### 3.4.2. Tab, Space i jednoznaczność

\`Tab\` pomaga dokończyć rozpoznawalne słowo kluczowe i nazwy. Junos może również wykorzystywać spację jako autocomplete, zgodnie z ustawieniem \`CLI complete-on-space\`. Jeśli wpisałeś zbyt krótki, niejednoznaczny fragment, CLI nie powinno zgadywać, czy chodzi Ci o \`interfaces\`, \`interface-range\` czy inną opcję.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show cli
set cli complete-on-space on
show cli
~~~

**To ustawienie wpływa na Twoją interaktywną sesję CLI**, nie jest zmianą konfiguracji wszystkich interfejsów urządzenia. Sprawdź wersję przed użyciem konkretnej formy polecenia.

**Ćwiczenie:** wpisz \`show int\`, naciśnij \`Tab\`, a następnie porównaj zachowanie przy \`show interfaces te\`. Klucz do skutecznego używania autocomplete to znalezienie **wystarczająco jednoznacznego prefiksu**, a nie liczenie na magię.

### 3.4.3. Ustawienia terminala, które ratują długą noc

Gdy analizujesz długi output, stronicowanie \`---(more)---\` pomaga go czytać. Przy automatyzacji może przeszkadzać. Junos pozwala sterować liczbą wierszy i szerokością terminala.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show cli
set cli screen-length 0
set cli screen-width 160
show cli
~~~

\`screen-length 0\` oznacza brak pauz po każdym ekranie. Jest wygodne dla skryptów i kopiowania outputu, ale przy dużej tablicy BGP terminal może zalać Cię tysiącami linii. Wtedy zamiast zwiększać scrollback, użyj \`| match\`, \`| find\` albo \`| no-more\` tylko dla konkretnej komendy.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
set cli screen-length 24
show cli
~~~

**Cisco-moment:** \`terminal length 0\` w Cisco i \`set cli screen-length 0\` w Junos rozwiązują bardzo podobny problem z paginacją.

### 3.4.4. Co, jeśli CLI nie zna komendy?

Podczas migracji Cisco → Junos łatwo wpaść w nawyk „komenda powinna istnieć, bo jest logiczna”. Ale CLI Junos jest **zależny od platformy i wydania**. Polecenie PFE z MX nie musi działać na vSRX, a funkcje J-Web dostępne na EX nie muszą działać na innym urządzeniu.

Rzetelna reakcja:
1. Wpisz \`?\` w miejscu, gdzie spodziewasz się argumentu.
2. Sprawdź \`show version\`.
3. Sprawdź [Junos CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/).
4. Zweryfikuj rodzaj platformy (EX, MX, QFX, SRX, Junos Evolved).
5. Dopiero potem twierdź, że polecenie jest niedostępne lub ma inną nazwę.

Nie ucz się diagnozy „router zepsuty” na podstawie \`syntax error\` po wklejeniu cudzej komendy.

---

## 3.5. Filtrowanie wyników poleceń — twoja lupa do CLI [A]

### 3.5.1. Nie odczytujesz całej książki telefonicznej, żeby znaleźć jeden numer

Na C9500 znasz \`show run | section\`, \`show ... | include\`, \`exclude\` i \`begin\`. W Junos filtrujesz wyniki przez znak pipe \`|\`, ale masz też narzędzia typowe dla wyświetlania **konfiguracji jako danych strukturalnych**.

**Najważniejszy błąd, którego chcę Cię oduczyć:** \`show route | count\` **nie liczy tras**. Zlicza **wiersze tekstu powstałego z danego polecenia**. Jeśli wiersz route zajmuje trzy linie, wynik count nie odpowiada liczbie prefiksów.

### 3.5.2. Szukasz jednej wiadomości, nie tysiąca linii

Przypuśćmy, że monitorujesz incydent z interfejsem \`ge-0/0/1\` i BGP. W logu jest kilkadziesiąt tysięcy wpisów. Nie kopiuj wszystkiego do ticketu.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show interfaces terse | match ge-0/0/1
show log messages | match "rpd|bgp|ge-0/0/1"
show log messages | except "SNMP"
show log messages | last 30
show interfaces extensive | find "ge-0/0/1"
~~~

- \`match\` — wybiera linie pasujące do wzorca.
- \`except\` — wyklucza linie pasujące do wzorca.
- \`last\` — pokazuje końcową część wyjścia (w pewnych warunkach co najmniej rozmiar ekranu).
- \`find\` — zaczyna wyświetlanie od pierwszego trafienia i pokazuje także **dalszy kontekst**, nie tylko pasujące linie.

**Gotcha:** \`find\` i \`match\` **nie są zamienne**. Gdy szukasz statusu portu wraz z kolejnymi licznikami, \`find\` może dać potrzebny kontekst. Gdy chcesz jedynie wpisy ze słowem „error”, \`match\` da krótszy wynik.

### 3.5.3. Dwa filtry mogą naraz zmienić sens wyniku

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show log messages | match "rpd" | match "BGP"
show interfaces terse | match "ge-" | count
show log messages | no-more
~~~

Przy dwóch \`match\` wynik to linie spełniające **oba** warunki, a nie „rpd lub BGP”. Wzorzec z \`|\` **wewnątrz cytowanego regex** oznacza alternatywę. To zachowanie przydaje się przy analizie określonego procesu.

W przypadku \`| count\` zatrzymaj się i zapytaj: **co właśnie policzyłem?** Jeżeli wcześniej filtrowałeś linie z \`ge-\`, policzyłeś linie tekstu pasujące do wzorca. Nie porty „up”, nie trasy „active”, chyba że wynik był specjalnie przetworzony w taki sposób, by każda linia odpowiadała dokładnie jednej pozycji.

### 3.5.4. \`display set\` — najlepszy pomost do modelu „linia po linii”

Wyobraź sobie, że musisz przekazać koledze konfigurację pojedynczego interfejsu. W Cisco wysyłasz fragment \`show run interface\`. W Junos możesz wybrać czytelny **widok hierarchiczny** albo **linie set**. To nie są dwie niezależne konfiguracje. To dwa sposoby oglądania tej samej konfiguracji.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show configuration interfaces ge-0/0/1
show configuration interfaces ge-0/0/1 | display set
show configuration system | display set
~~~

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
top
show interfaces ge-0/0/1
show interfaces ge-0/0/1 | display set
show | display inheritance
~~~

\`display set\` pokazuje konfigurację jako polecenia możliwe do odtworzenia. **Nie zmienia** bieżącej konfiguracji. I nie używaj go jako rzekomego „tłumacza” dowolnych wyników \`show route\` na komendy set — dotyczy **struktury konfiguracji**.

### 3.5.5. Właściwy obraz konfiguracji bywa ukryty przez dziedziczenie

Na Junos istnieją **configuration groups**. Interfejs może dziedziczyć parametry, których **nie zobaczysz wprost** w jego bezpośredniej sekcji. To podobne do sytuacji, gdy w Cisco przyzwyczaiłeś się do jednego miejsca na konfigurację, a część stanu pochodzi z template, policy lub domyślnego profilu.

**Urządzenie: EX-ACCESS-01 — Operational Mode (w przypadku używania groups)**

~~~junos
show configuration interfaces ge-0/0/1
show configuration interfaces ge-0/0/1 | display inheritance
show configuration interfaces ge-0/0/1 | display set
~~~

**Praktyczne pytanie:** dlaczego port ma określony MTU, choć nie widzę go w zwykłym \`show configuration interfaces ge-0/0/1\`? Jednym z tropów jest **inheritance**. Szczegółowa konfiguracja \`apply-groups\` należy do rozdziału 4, lecz już teraz musisz wiedzieć, że brak linii w widoku lokalnym **nie musi oznaczać braku efektu**.

### 3.5.6. Widoki do archiwizacji i automatyzacji

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show configuration | display set
show configuration | display xml
show configuration | display json
~~~

Wyświetlanie XML/JSON zależy od rodzaju danych i wydania; ma znaczenie przy automatyzacji oraz integracjach NETCONF. Nie uczmy się tutaj całego YANG/NETCONF — zapamiętaj, że hierarchiczny Junos **dobrze nadaje się do strukturalnego przetwarzania konfiguracji**, a nie tylko do tekstowych zrzutów.

**Nie umieszczaj w publicznym repozytorium bez filtrowania pełnych \`show configuration\`: mogą zawierać adresy zarządzania, informacje o kontach, community, sekrety albo dane infrastruktury.**

Źródła sekcji: [CLI pipe command](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/pipe.html), [Filtering Operational Command Output](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/filtering-operational-command.html), [configuration groups](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configuration-groups-usage.html).

