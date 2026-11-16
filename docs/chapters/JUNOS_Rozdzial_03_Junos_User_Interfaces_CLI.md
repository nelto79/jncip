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



---

## 3.6. Active vs candidate — architektura transakcji, która zmienia styl pracy [A]

### 3.6.1. Dlaczego „brudnopis” nie oznacza „prywatny brudnopis”

W naszej historii z 03:07 opisałem **candidate** jako projekt zmiany. Teraz pojawia się pierwsza pułapka. W typowym **\`configure\`**, gdy dwóch administratorów otworzy konfigurację jednocześnie, korzystają z **wspólnego candidate**. To nie są dwie niezależne kopie.

Wyobraź sobie dwóch inżynierów edytujących jeden wspólny dokument online:
- Ty dodajesz opis portu uplink.
- Kolega zmienia politykę dla BGP.
- Ty wykonujesz \`commit\`.

W zależności od sposobu pracy **zatwierdziłeś obie zmiany**, choć zgłoszenie ServiceNow obejmowało wyłącznie Twój port. To potencjalny problem audytowy, a nie wada jakiejś jednej komendy.

~~~mermaid
flowchart TD
  AA["Administrator A: set interface description"] --> SH["Shared candidate (configure)"]
  BB["Administrator B: set routing policy"] --> SH
  SH --> COM["Commit z dowolnej sesji uprawnionej"]
  COM --> ACT["Nowa active config: obie zmiany"]
  PA["Administrator A: configure private"] --> P1["Private candidate A"]
  PB["Administrator B: configure private"] --> P2["Private candidate B"]
  P1 --> CHECK["Sprawdzenie aktualnego active i konfliktów"]
  P2 --> CHECK
  CHECK --> ACT
~~~

**Do zapamiętania na pamięć:** \`configure\` = zwykle wspólny candidate; \`configure private\` = prywatna kopia robocza; \`configure exclusive\` = blokada edycji/commit shared candidate dla innych administratorów. **Do zrozumienia:** private nie oznacza, że stan sieci w tle przestaje się zmieniać.

### 3.6.2. Który tryb edycji wybrać o 03:07?

Jeśli musisz przygotować niewielką zmianę, zwykły \`configure\` wystarczy **tylko wtedy, gdy świadomie kontrolujesz współdzielony stan konfiguracji**. W większym zespole częściej rozważysz \`configure private\` lub formalnie uzgodniony \`configure exclusive\`. Nie używaj \`exclusive\` bez uzgodnienia z innymi operatorami — może blokować potrzebne zmiany i utrudniać incydent.

**Urządzenie: R1-vSRX — Operational Mode, wariant wspólny**

~~~junos
configure
~~~

**Urządzenie: R1-vSRX — Operational Mode, wariant prywatny**

~~~junos
configure private
~~~

**Urządzenie: R1-vSRX — Operational Mode, wariant wyłączny (tylko przy zgodzie zespołu)**

~~~junos
configure exclusive
~~~

**To warianty alternatywne**, nie trzy następujące po sobie linijki. Każda rozpoczyna inny model współpracy; po eksperymencie kończ konfigurację przed wybraniem następnego wariantu.

### 3.6.3. Gotcha private: kolega zatwierdził zmianę, gdy Ty pracowałeś

Wyobraź sobie, że na początku Twojej prywatnej sesji active ma numer wersji X. W międzyczasie administrator B zmienia hostname lub politykę i wykonuje commit. Teraz Twoja private candidate powstała ze **starszego active**. Junos nie musi automatycznie przyjąć jej jako nowej konfiguracji systemu.

Dokumentacja Juniper mówi o sprawdzaniu zmian dokonanych przez innych; w zależności od sytuacji commit może zgłosić konflikt. Polecenie **\`update\`** synchronizuje prywatną kopię z ostatnio zatwierdzoną konfiguracją, próbując zachować Twoje zmiany. Przy konflikcie trzeba go rozwiązać — \`update\` nie jest magicznym narzędziem „always merge”.

**Urządzenie: R1-vSRX — Configuration Mode [edit private]**

~~~junos
status
show | compare
update
show | compare
commit check
~~~

**Nie wykonuj tego jak mechanicznego skryptu**: najpierw sprawdź, czy rzeczywiście potrzebujesz \`update\` i jaki konflikt wystąpił. Jeśli private candidate zawiera niezależne zmiany, \`update\` może się powieść. Jeśli dotykaliście tych samych obiektów, wymagana będzie ręczna decyzja inżyniera.

**Most z Cisco:** możesz pomyśleć o „configuration lock” w IOS XE i automatyzacji przez controller, ale **Junos configure private** ma własną semantykę candidate i konfliktów. Sam fakt pracy w prywatnym edytorze nie zamraża live routing.

### 3.6.4. Gotcha exclusive: lock nie jest HA

Dokumentacja producenta podaje, że \`configure exclusive\` blokuje shared candidate dla innych operatorów. Jeśli w trakcie takiej sesji wykonasz \`commit confirmed\`, a potem nastąpi automatyczny rollback z powodu braku potwierdzenia, **blokada exclusive może zostać zdjęta**. Możesz zobaczyć komunikat o cofniętym dostępie do lock, mimo że nadal masz sesję.

To ma sens: nie chcemy, żeby opuszczone okno serwisowe trzymało „zamek” przez wieczność.

**Do zrozumienia:** blokada konfiguracji jest zabezpieczeniem organizacyjnym, nie zapewnia redundancji routera ani ochrony przed niepoprawną trasą.

### 3.6.5. Kandydat po zmianach — czy on jeszcze jest prywatny?

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show system users
show system commit
show configuration system host-name
~~~

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
status
show | compare
run show system users
~~~

Zbierz trzy dowody: kto pracuje na urządzeniu, jaki jest stan pending commit oraz czy candidate różni się od active. **Nie traktuj samego promptu \`#\` jako dowodu prywatności sesji.**

Źródła: [Overview of the Configure Command](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configure-command.html), [configure CLI reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/configure.html).

---

## 3.7. Modyfikowanie, usuwanie, zarządzanie i zapisywanie [A]

### 3.7.1. Zanim użyjesz \`set\`, zrozum różnicę między parametrem a gałęzią

W Cisco znasz \`no ip address\`, \`no description\`, \`shutdown\`, \`default interface\`. W Junos konfiguracja jest drzewem, więc możesz usunąć **wartość**, **statement** albo **całą gałąź**. Skutki zależą od miejsca drzewa, nie tylko od słowa \`delete\`.

Rozważmy dwa przykłady:
- Usuń \`description\` z interfejsu — zwykle mały wpływ operacyjny.
- Usuń cały \`interfaces ge-0/0/1\` — możesz stracić adres, L2/L3 i dostęp do urządzenia.

**Urządzenie: R1-vSRX — Configuration Mode [edit], wyłącznie LAB**

~~~junos
set interfaces ge-0/0/1 description "WAN-PRIMARY-LAB"
show interfaces ge-0/0/1
delete interfaces ge-0/0/1 description
show interfaces ge-0/0/1
show | compare
~~~

W przykładzie pokazuję bezpieczny rodzaj zmiany — opis — i sposób jej odwrócenia przed commit. **Nie usuwaj całego portu**, by przećwiczyć \`delete\`.

### 3.7.2. \`deactivate\` nie usuwa konfiguracji: zostawia ją „w szufladzie”

Załóżmy, że masz skonfigurowaną trasę statyczną i chcesz ją tymczasowo wyłączyć, ale nie chcesz zapomnieć next-hopu oraz prefiksu. W Cisco często zrobiłbyś \`no ip route\`, a potem musiał odtwarzać wpis. Junos może oznaczyć dany statement jako **inactive**, aby pozostał w konfiguracji, lecz nie był stosowany po commit.

Na naszym ćwiczeniu wybierzemy bezpieczniejszą gałąź — **description nie jest idealnym przykładem stanu inactive**, więc posłużymy się dedykowaną **testową trasą do dokumentacyjnego prefiksu**, a nie realnym uplinkiem.

**Urządzenie: R1-vSRX — Configuration Mode [edit], izolowany LAB; nigdy nie używaj realnego prefiksu produkcyjnego**

~~~junos
set routing-options static route 203.0.113.0/24 discard
deactivate routing-options static route 203.0.113.0/24
show routing-options | display set
show | compare
~~~

W hierarchicznym widoku przed \`static route\` powinieneś zobaczyć oznaczenie \`inactive:\` (forma outputu zależy od kontekstu). W takim stanie wpis jest zachowany, ale nie powinien ustanawiać używanej trasy w active. Jeśli dopiero przygotowujesz candidate, nie musi jeszcze istnieć żaden aktywny wpis dla tej trasy.

**Urządzenie: R1-vSRX — Configuration Mode [edit], w tym samym izolowanym LAB**

~~~junos
activate routing-options static route 203.0.113.0/24
show | compare
delete routing-options static route 203.0.113.0/24
show | compare
~~~

Widzisz różnicę:
- \`deactivate\`: zaznacz statement jako nieaktywny.
- \`activate\`: przywróć aktywność w candidate.
- \`delete\`: usuń statement z candidate.

**Gotcha:** dezaktywowanie statementu po commit **może mieć realny wpływ na routing, sesje i user traffic**. To nie to samo co ukrycie linijki przed operatorem.

### 3.7.3. \`rename\`, \`copy\`, \`insert\`, \`annotate\` — edytor zamiast przepisywania

Projektujesz 20 identycznych portów access. Zamiast przepisywać wszystkie parametry możesz skopiować odpowiedni fragment i zmienić jego identyfikator. Ale kopiowanie całości portu oznacza również kopiowanie **adresów, filtrów, VLAN-ów i opisów**, dlatego najpierw obejrzyj różnicę.

Poniższy przykład używa **gałęzi \`policy-options prefix-list\`**, żeby nie ryzykować zniszczenia istniejącego interfejsu:

**Urządzenie: R1-vSRX — Configuration Mode [edit], izolowany LAB**

~~~junos
set policy-options prefix-list LAB-SOURCES 192.0.2.0/24
copy policy-options prefix-list LAB-SOURCES to LAB-SOURCES-COPY
show policy-options | display set
delete policy-options prefix-list LAB-SOURCES-COPY
delete policy-options prefix-list LAB-SOURCES
show | compare
~~~

W podobnym stylu możesz \`rename\` wybrane obiekty. **Nie zmieniaj nazwy policy albo security zone bez zbadania wszystkich referencji**, ponieważ Junos może odrzucić commit lub zmienić powiązania, a skutki dla przepływu ruchu zależą od obiektu.

\`insert\` jest ważny przy **uporządkowanych elementach**, np. termach policy lub firewall filters. Kolejność termów bywa merytoryczną częścią decyzji! W Cisco znasz sequence numbers w route-map/ACL; Junos pozwala zmieniać kolejność za pomocą edytora.

**Urządzenie: R1-vSRX — Configuration Mode [edit], izolowany LAB, przykład pozycji termów**

~~~junos
set policy-options policy-statement CH03-ORDER term FIRST then accept
set policy-options policy-statement CH03-ORDER term SECOND then reject
insert policy-options policy-statement CH03-ORDER term SECOND before term FIRST
show policy-options policy-statement CH03-ORDER
delete policy-options policy-statement CH03-ORDER
show | compare
~~~

**Cel:** zobaczyć zmianę **kolejności ewaluacji**. Sama konfiguracja testowej policy nie wpływa na routing, dopóki nie zastosujesz jej w odpowiednim miejscu. Nie wyciągaj z tego wniosku, że każda policy bez \`from\` w produkcji jest nieszkodliwa.

### 3.7.4. \`replace pattern\` — potężne, ale zdradliwe

Wyobraź sobie migrację nazewnictwa 200 portów. Na Cisco często używasz skryptu i sprawdzasz \`show archive config differences\`. Junos pozwala przeszukać i zamienić identyfikatory lub wartości w **candidate configuration** za pomocą \`replace pattern\`, w tym wyrażeń regularnych.

Problem polega na tym, że **jedno zbyt szerokie wyrażenie może zmienić wiele obiektów**. Gdy wrzucisz tam adresację, nazwy interfejsów i ACL, nie możesz ufać samemu „commit complete”.

Bezpieczny test wykonajmy na jednym specjalnie utworzonym opisie:

**Urządzenie: R1-vSRX — Configuration Mode [edit], izolowany LAB**

~~~junos
set interfaces ge-0/0/1 description "CH03-OLD-DESCRIPTION"
replace pattern CH03-OLD-DESCRIPTION with CH03-NEW-DESCRIPTION
show | compare
~~~

**Nie wykonuj globalnego \`replace pattern 10. with 172.\`** w produkcji. Mechanizm regex, escapowanie kropek oraz pełny zakres gałęzi mogą objąć więcej, niż chciałeś. Dokumentacja Juniper wyjaśnia także \`upto n\` — liczba dotyczy **obiektów**, a nie koniecznie pojedynczych wystąpień tekstu w jednym obiekcie.

**Urządzenie: R1-vSRX — Configuration Mode [edit], cofnięcie niezacommitowanej próby — tylko po sprawdzeniu, że nie ma innych zmian do zachowania**

~~~junos
rollback 0
show | compare
~~~

Źródło: [Juniper replace command](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/replace.html).

### 3.7.5. \`save\` — czy zapis pliku oznacza commit? Nie!

To następna pułapka dla inżyniera przyzwyczajonego do \`copy running-config startup-config\`.

W Junos \`save /var/tmp/...\` uruchomione w configuration mode zapisuje wybraną konfigurację **jako tekstowy plik**. Nie oznacza, że zmiany weszły do active. Wykonanie \`commit\` jest inną operacją. Natomiast samo **\`commit\` zapisuje i aktywuje** konfigurację — typowo nie potrzebujesz osobnego odpowiednika \`write memory\`.

**Urządzenie: R1-vSRX — Configuration Mode [edit], izolowany LAB**

~~~junos
top
save /var/tmp/ch03-candidate-backup.conf
show | compare
~~~

**Urządzenie: R1-vSRX — Operational Mode po wyjściu z konfiguracji**

~~~junos
file list /var/tmp/ detail
show configuration system host-name
~~~

Plik w \`/var/tmp\` jest przydatną lokalną kopią roboczą, ale **nie jest trwałym systemem backupu off-device**. Nie trzymaj jedynego ratunku na tym samym dysku routera. Przy upgrade, czyszczeniu storage albo awarii nośnika możesz go stracić.

### 3.7.6. \`annotate\` — dlaczego opis w konfiguracji jest przydatny, a kiedy bywa problemem

Junos pozwala dodawać komentarze do sekcji konfiguracji, m.in. poleceniem \`annotate\`. To pomaga wyjaśnić nietypową politykę i cel zmiany, ale **komentarz nie zastępuje CMDB, dokumentacji HLD ani worknotes w ServiceNow**.

Nie zaczynaj od masowego komentowania konfiguracji produkcyjnej. Ćwiczenie wykonywane na wyizolowanym obiekcie pokażemy w laboratorium 3.LAB; szczegółową składnię sprawdź przez \`annotate ?\` oraz aktualną [CLI Configuration Mode Overview](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html).



---

## 3.8. Przeglądanie, porównywanie i wczytywanie plików konfiguracji [A]

### 3.8.1. Zadanie z rzeczywistej pracy: architekt przysłał 120 linii zmian

Jest środa, 17:30. Architekt przysyła plik z dwudziestoma nowymi VLAN-ami, filtrami i opisami portów dla lokalizacji we Francji. W Cisco zwykle analizujesz skrypt konfiguracyjny, planujesz kolejność, być może korzystasz z \`configure replace\` i archiwizacji. W Junos masz możliwość **wczytania pliku do candidate**, obejrzenia dokładnej różnicy i dopiero potem zatwierdzenia.

Ale tutaj pojawia się większe zagrożenie niż literówka: **zły tryb importu**. Ten sam plik może być nieszkodliwym dopisaniem fragmentu albo zastąpieniem kompletnej konfiguracji urządzenia — zależnie od tego, jakiego \`load\` użyjesz.

Wyobraź sobie edycję dokumentu:
- **merge**: dopisz lub uzupełnij określone akapity;
- **set**: wykonaj sekwencję instrukcji edycyjnych zapisanych jako komendy;
- **replace**: wymień wskazane akapity oznaczone do zastąpienia;
- **override**: **wyrzuć cały dotychczasowy dokument i wczytaj kompletny nowy**;
- **update**: również wczytaj pełny dokument, ale porównuj zmienione części bardziej selektywnie;
- **patch**: zastosuj różnicę/łatkę zamiast pełnego tekstu.

To nie jest sprawa gustu. \`load override\` użyty omyłkowo do pliku z trzema interfejsami może spowodować, że **candidate będzie zawierać tylko te trzy interfejsy i zabraknie reszty konfiguracji**. Po commit tracisz użytkowników, management, BGP i wiele więcej.

### 3.8.2. Zanim coś załadujesz: cztery bezpieczne pytania

Zanim użyjesz \`load\`, sprawdź:
1. Czy plik jest w formacie **hierarchicznym Junos**, zestawie \`set\`, czy patch?
2. Czy chcesz **dodać fragment**, **wymienić oznaczony blok**, czy **zastąpić pełną konfigurację**?
3. Czy znajdujesz się w \`configure private\`, a jeśli tak, czy inni administratorzy już zmienili active?
4. Czy masz konsolę/OOB, kopię ostatniej dobrej konfiguracji i plan odtworzenia dostępu?

Nawet jeśli wszystkie odpowiedzi są dobre, **\`commit check\` nie sprawdzi, czy nowy BGP peer jest osiągalny**. Po zmianie musisz testować usługę.

### 3.8.3. Przeglądanie active i candidate bez zmieniania czegokolwiek

**Urządzenie: R1-vSRX — Operational Mode, odczyt aktywnej konfiguracji**

~~~junos
show configuration
show configuration system host-name
show configuration interfaces ge-0/0/1 | display set
show system commit
show system rollback 1
~~~

**Urządzenie: R1-vSRX — Configuration Mode, odczyt candidate**

~~~junos
top
show
show | display set
show | compare
~~~

Warto od razu nauczyć się jednej różnicy:
- **Operational \`show configuration\`** to odczyt aktualnej, zatwierdzonej konfiguracji.
- **Configuration \`show\`** pokazuje edytowaną candidate.
- **\`show | compare\`** w configuration mode mówi, co candidate zmieniłaby względem active.

Nie kopiuj bezmyślnie konfiguracji z sekcji \`show\` jako „tej, co już działa” — chyba że wiesz, w jakim trybie był operator.

### 3.8.4. \`show | compare\` — czytanie diffu jak inspektor przed zatwierdzeniem

Załóżmy, że dodałeś nowy opis portu i zmieniłeś parametr. Junos pokaże różnicę, na której możesz wychwycić literówkę przed zmianą produkcyjną.

**Urządzenie: R1-vSRX — Configuration Mode [edit], przykład ilustracyjnego diffu**

~~~diff
[edit interfaces ge-0/0/1]
- description "WAN OLD";
+ description "WAN NEW";
~~~

Minus oznacza element usuwany ze starej wersji, plus — dodawany do candidate. Ale to **format porównania**, nie polecenia CLI do uruchomienia.

Haczyk: **dwa plusy w diffie nie gwarantują, że zmiana jest właściwa**. Możesz poprawnie wprowadzić błędną nazwę route-map, złą sieć źródłową w ACL albo wyłączyć port uplink. Poprawność wyniku zależy od HLD/change plan i testów.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
show | compare
~~~

**Urządzenie: R1-vSRX — Operational Mode, porównanie dwóch zapisanych wersji bez ładowania ich do candidate**

~~~junos
show system rollback 0 compare 1
show system rollback 1
show configuration | compare rollback 1
~~~

To druga bardzo ważna różnica. **Nie musisz wykonywać \`rollback 1\`, żeby zobaczyć dawną konfigurację!** Możesz ją odczytać i porównać bez naruszania candidate.

### 3.8.5. \`load set\` — gdy plik wygląda jak gotowe komendy Junos

Załóżmy, że architekt przygotował niewielką, bezpieczną zmianę: opis portu i tekst login banner. Plik zawiera pełne linie \`set\`. To dobry przypadek dla **\`load set\`**, nie dla \`load override\`.

**Plik dla urządzenia R1-vSRX — \`/var/tmp/ch03-change.set\` (przykładowa zawartość)**

~~~junos
set system login message "CH03 LAB - maintenance test"
set interfaces ge-0/0/1 description "CH03-LINK"
~~~

Załóżmy, że plik został wcześniej **bezpiecznie przesłany** do R1 i potwierdzono jego treść. Nie podaję hasła ani rzeczywistego URL serwera produkcyjnego.

**Urządzenie: R1-vSRX — Operational Mode, odczyt pliku przed importem**

~~~junos
file show /var/tmp/ch03-change.set
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], izolowany LAB**

~~~junos
load set /var/tmp/ch03-change.set
show | compare
commit check
~~~

**Przewidź wynik:** \`load set\` dopisze/zmieni wskazane statementy w candidate. Nie powinien usuwać całej konfiguracji BGP, AAA czy pozostałych interfejsów. Jednak jeśli plik zawierałby polecenia \`delete\`, to **mogłyby zostać wykonane podczas ładowania** — sprawdź zawartość, nie tylko rozszerzenie \`.set\`.

Jeśli wolisz nie tworzyć pliku na dysku, możesz użyć \`load set terminal\`. Po wpisaniu lub wklejeniu zawartości terminala kończysz strumień przez **Ctrl+D na nowej linii**.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], izolowany LAB — rozpoczęcie importu z terminala**

~~~junos
load set terminal
~~~

**Dane wklejane dla R1-vSRX po wyświetleniu prośby o wejście — nie osobne polecenia uruchamiane z shell**

~~~junos
set system login message "CH03-LAB FROM TERMINAL"
set interfaces ge-0/0/1 description "CH03-UPLINK"
~~~

Po **Ctrl+D** wykonaj \`show | compare\`. Nie myl końca wejścia terminalowego z \`commit\`: to dwa oddzielne etapy.

### 3.8.6. \`load merge\` — złącz fragment hierarchii

Architekt przesyła plik w postaci drzewa Junos, a nie \`set\`. Do dodania sekcji lub aktualizacji wartości służy \`load merge\`. W praktyce zachowanie przy konflikcie istniejących i przychodzących statementów wymaga uważnego odczytu diffu. Nie przyjmuj, że merge zawsze jest w 100% addytywne.

**Plik dla R1-vSRX — \`/var/tmp/ch03-merge.conf\` (fragment konfiguracji hierarchicznej)**

~~~junos
system {
    login {
        message "CH03-MERGE-LAB";
    }
}
interfaces {
    ge-0/0/1 {
        description "CH03-MERGED";
    }
}
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], izolowany LAB**

~~~junos
load merge /var/tmp/ch03-merge.conf
show | compare
commit check
~~~

Weryfikujesz, że zmienił się tylko banner i opis portu — **zamiast zgadywać na podstawie słowa „merge”**. Jeśli architekt wysłał niepoprawny fragment, loader może zgłosić błąd zanim dojdzie do commit.

### 3.8.7. \`load replace\` — specjalny znacznik zastąpienia gałęzi

To polecenie jest częstą pułapką egzaminacyjną. \`load replace\` **nie oznacza po prostu „zastąp całą konfigurację”**. Szuka specjalnych znaczników **\`replace:\`** we wczytywanej konfiguracji i zastępuje odpowiednie statementy.

Załóżmy, że masz laboratoryjną policy \`CH03-ORDER\` z trzema termami. Chcesz wymienić całą zawartość tej policy, ale nie ruszać innych \`policy-options\`.

**Plik dla R1-vSRX — \`/var/tmp/ch03-replace.conf\` (zastąpienie wyłącznie laboratoryjnej policy)**

~~~junos
policy-options {
    replace: policy-statement CH03-ORDER {
        term ALLOW-LAB {
            then accept;
        }
        term OTHER {
            then reject;
        }
    }
}
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], izolowany LAB**

~~~junos
load replace /var/tmp/ch03-replace.conf
show policy-options policy-statement CH03-ORDER
show | compare
commit check
~~~

**Pułapka:** jeśli \`replace:\` umieścisz o jeden poziom wyżej, możesz niechcący zastąpić dużo większą część konfiguracji. W naszym przykładzie nie zastępujemy \`policy-options\` jako całości, tylko jeden nazwany obiekt.

### 3.8.8. \`load override\` — wielki czerwony przycisk

Przygotowanie do greenfield to jedna sytuacja: nowy EX ma otrzymać **pełną, poprawną konfigurację bazową**. Zastąpienie całej candidate może być zamierzone. Ale przy zmianie kilku portów na produkcji \`override\` jest zwykle **niewłaściwym** narzędziem.

**Polecenie poniżej jest NIEBEZPIECZNE na istniejącym urządzeniu; zapis wyłącznie jako przykład do rozpoznania, NIE URUCHAMIAJ bez odseparowanego labu, pełnego backupu i konsoli.**

**Urządzenie: R1-vSRX — Configuration Mode, tylko koncept do analizy skutku**

~~~junos
load override /var/tmp/FULL-DEVICE-CONFIG.conf
~~~

Po takim load candidate pochodzi z całego pliku, nie jest dopisanym fragmentem. Jeżeli plik zawierał tylko jeden port, **pozostałe elementy kandydackiej konfiguracji mogą zniknąć**, a po commit możesz stracić dostęp do urządzenia.

**Pamiętaj:** sam \`load override\` zmienia candidate; krytyczny moment wejścia zmiany w życie to **commit**. To właśnie daje możliwość zobaczenia diffu i przerwania niebezpiecznej operacji zanim stanie się aktywna.

### 3.8.9. \`load update\` i \`load patch\` — gdy zwykły merge nie wystarcza

Dwa mniej popularne tryby mają praktyczne znaczenie w automatyzacji:
- \`load update\` wczytuje pełną konfigurację, podobnie jak \`override\`, ale oznacza jako zmodyfikowane tylko faktyczne różnice.
- \`load patch\` stosuje różnicę, zamiast zwykłego fragmentu konfiguracji; używa się go m.in. do odtwarzania zmian z diffów.

**Nie jest to równoznaczne z bezpiecznym importem dowolnego pliku.** Na egzaminie Associate trzeba kojarzyć różnicę, a do automatyzacji poziomu Professional — rozumieć semantykę, szczególnie gdy commit scripts i dynamiczne funkcje odnoszą się do zmienionych obiektów.

**Gotcha:** \`load update\` nie jest „update mojej private candidate” — to **inny mechanizm** niż polecenie \`update\` w \`configure private\`. Te same słowa, inne zadanie!

### 3.8.10. Gdzie znajduje się plik i co można z nim zrobić?

Plik dostarczony przez architekta nie pojawia się magicznie w \`/var/tmp\`. Musisz go bezpiecznie przenieść (np. SCP/SFTP lub zatwierdzony system zarządzania konfiguracją), sprawdzić integralność i ścieżkę. Junos udostępnia także polecenia operacyjne do pracy z plikami.

**Urządzenie: R1-vSRX — Operational Mode, odczyt zawartości plików w izolowanym LAB**

~~~junos
file list /var/tmp/
file show /var/tmp/ch03-change.set
~~~

Nie usuwaj przypadkowych plików systemowych „dla miejsca”. Również nie przechowuj w GitHubie realnych kluczy prywatnych, kont, community ani pełnej konfiguracji produkcyjnej.

### 3.8.11. Bezpieczny import według change plan

Wyobraź sobie, że klient zgłasza konkretną zmianę VLAN/port i chcesz mieć audytowalny wynik. Najpierw wykonaj read-only precheck. Później importuj **wyłącznie właściwym trybem**. Następnie porównaj candidate z active, sprawdź poprawność, aktywuj tymczasowo z mechanizmem powrotu, wykonaj test i potwierdź.

~~~mermaid
flowchart TD
  INPUT["Otrzymałeś plik konfiguracyjny"] --> Q{"Jaki format i zakres?"}
  Q -->|"linie set / mały fragment"| SET["load set"]
  Q -->|"fragment hierarchy"| MERGE["load merge"]
  Q -->|"znaczniki replace: w hierarchy"| REPL["load replace"]
  Q -->|"pełny kompletny config"| OVR["load override lub load update — wymaga ostrożności"]
  SET --> COMP["show | compare"]
  MERGE --> COMP
  REPL --> COMP
  OVR --> COMP
  COMP --> OK{"Czy diff jest zgodny z planem?"}
  OK -->|"Nie"| STOP["Przerwij — NIE commituj"]
  OK -->|"Tak"| CHK["commit check"]
  CHK --> CONF["commit confirmed"]
  CONF --> TEST["Sprawdź dostęp, routing, usługi, logi"]
  TEST -->|"PASS"| FIN["commit"]
  TEST -->|"FAIL"| ROL["Rollback / nie potwierdzaj"]
~~~

**Nie daj się oszukać eleganckiemu diagramowi:** \`commit confirmed\` nie jest niezawodnym mechanizmem naprawy każdej usterki, a \`commit check\` w szczególnych okolicznościach może **potwierdzić wcześniejszy pending confirmed commit** — szczegóły w 3.11.

Źródła sekcji: [load CLI reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/load.html), [Loading Configuration Files](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-config-files-loading.html), [save CLI reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/save-cli-config.html).



---

## 3.9. Rollback — powrót do poprzedniej konfiguracji bez zgadywania [A]

### 3.9.1. O 03:18 aplikacja działa gorzej. Co teraz?

Wyobraź sobie, że wdrożyłeś politykę BGP, a użytkownicy zaczynają zgłaszać utratę części tras. Ktoś podpowiada: „Cofnij konfigurację do poprzedniej wersji”. Na Cisco IOS XE sprawdziłbyś archiwum, configuration replace, historię zmian albo przygotowany rollback plan. W Junos **historia kolejnych zatwierdzonych konfiguracji jest częścią codziennego workflow**.

Ale uważaj: **\`rollback\` nie oznacza automatycznego aktywowania starej konfiguracji**. Zazwyczaj ładuje starszą wersję do candidate, którą nadal musisz obejrzeć, zweryfikować i zatwierdzić. To kolejny element świadomego zarządzania ryzykiem.

### 3.9.2. \`rollback 0\` kontra \`rollback 1\`: największa pułapka dla osób po Cisco

Załóżmy, że o 03:00 router działał na konfiguracji A. O 03:05 dodałeś zmiany w candidate B, ale jeszcze nie zrobiłeś commit.

**\`rollback 0\`** przywróci w candidate aktualną **active A** i odrzuci niezacommitowane zmiany B.

Teraz załóżmy, że commit już się odbył i active to B. Chcesz wrócić do **poprzedniej zatwierdzonej wersji A**. Wtedy zwykle użyjesz **\`rollback 1\`**, porównasz candidate z active, a następnie wykonasz commit.

~~~mermaid
flowchart LR
   A["Aktywna wersja A"] -->|"edytujesz candidate"| B["Candidate B"]
   B -->|"rollback 0"| A
   B -->|"commit"| C["Aktywna wersja B"]
   C -->|"rollback 1"| D["Candidate z poprzednią wersją A"]
   D -->|"commit"| E["Nowa aktywna wersja A"]
~~~

**Na pamięć:** \`rollback 0\` to zwykle odrzucenie aktualnych zmian candidate; \`rollback 1\` ładuje poprzedni commit do candidate. **Do zrozumienia:** po wcześniejszym rollbacku 1 i wykonaniu commit powstaje kolejna pozycja historii — numeracja jest względem **aktualnej** historii, nie stałym identyfikatorem „tej dobrej wersji z poniedziałku”.

### 3.9.3. Najpierw obejrzyj, nie cofaj na ślepo

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show system commit
show system rollback 0
show system rollback 1
show system rollback 0 compare 1
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], dopiero po identyfikacji właściwej wersji**

~~~junos
rollback 1
show | compare
commit check
commit confirmed 5
~~~

**Następny krok nie jest automatycznym \`commit\`!** Najpierw sprawdź stan SSH i kontrolę nad urządzeniem, routing, sesje i usługę, której dotyczył incydent. Dopiero przy satysfakcjonującym wyniku potwierdź zmianę. Pamiętaj, że \`commit check\` również potrafi potwierdzić oczekujący \`commit confirmed\` — weryfikacja powinna być zaplanowana świadomie.

### 3.9.4. Historia konfiguracji nie jest nieograniczona

Juniper typowo przechowuje **do 50 wersji**, licząc bieżącą i 49 wcześniejszych (numery \`0\`–\`49\`). Nie opisuj tego jako „50 starych backupów plus current”. Ponadto przechowywane kopie nie są tym samym co **zewnętrzny system backupów** — awaria nośnika, wymiana urządzenia lub problemy z dostępem mogą sprawić, że historia lokalna nie wystarczy.

Źródło: [rollback CLI reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/rollback.html).

### 3.9.5. Rescue configuration — ostatnia pewna przystań

Wyobraź sobie, że wykonałeś wiele commitów, wymieniasz urządzenie lub masz problem z dostępnością historii. Możesz utrzymywać **rescue configuration** — ręcznie zapisaną, sprawdzoną dobrą konfigurację.

**Haczyk:** rescue **nie musi istnieć**, jeśli operator nigdy jej nie zapisał. Nie jest też automatycznym gwarantem odzyskania wszystkich usług: jeśli zapisałeś złą konfigurację jako rescue, ona nie stanie się dobra tylko dlatego, że ma tę nazwę.

**Urządzenie: R1-vSRX — Operational Mode, zapis znanej dobrej konfiguracji po pozytywnej walidacji i zgodzie zmiany**

~~~junos
request system configuration rescue save
show system commit
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit], przywrócenie istniejącej rescue, wyłącznie LAB lub zatwierdzona procedura recovery**

~~~junos
rollback rescue
show | compare
commit check
commit confirmed 5
~~~

Do potwierdzenia i testów stosuj ten sam rygor co przy każdej zmianie. Szczegółowa administracja rescue jest w rozdziale **4.13**, ale tutaj poznajesz jej znaczenie jako części workflow rollback.

Źródło: [Juniper Rescue Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-rescue-configuration.html).

### 3.9.6. Rollback nie przywraca całego świata

Nieoczywisty, ale zawodowo krytyczny fakt: rollback **konfiguracji** nie oznacza cofnięcia stanu całego systemu operacyjnego, tabel sesji i wszystkich zewnętrznych systemów.

Jeżeli zmiana policy zdążyła zerwać sesje BGP albo VPN, powrót konfiguracji **nie zawsze przywróci je natychmiast**; protokoły potrzebują konwergencji. Jeżeli zmieniłeś politykę routingu, sąsiad może potrzebować odświeżenia lub przeliczenia. Jeśli błąd dotyczy fizycznej warstwy, żaden rollback nie naprawi włókna.

**Porównanie Cisco:** \`configure replace\` także nie cofa czasu dla TCP sessions ani nie wskrzesza ruchu na uszkodzonym kablu. Zawsze testuj **service restoration**, a nie tylko „commit complete”.

---

## 3.10. J-Web — dlaczego istnieje GUI i czemu nie zastępuje rozumienia CLI [A]

### 3.10.1. Inżynier CLI patrzy z podejrzliwością na przeglądarkę

Znane uczucie: pracowałeś latami na Cisco IOS XE, więc kiedy ktoś pokazuje GUI do zarządzania urządzeniem, pierwsza myśl brzmi: „Po co mi to, skoro mam konsolę?”. Często CLI będzie szybsze i precyzyjniejsze, ale nie oznacza to, że GUI jest zabawką.

**J-Web** jest przeglądarkowym interfejsem zarządzania dostępnym na określonych rodzinach i wydaniach Junos, przede wszystkim EX i SRX, z funkcjami zależnymi od platformy. Pozwala wykonywać czynności konfiguracji, przeglądania stanu, monitoringu i troubleshootingu.

**Porównanie Cisco:** Cisco Device Manager/GUI dla określonych platform albo Cisco Catalyst Center dają interfejsy wizualne, ale **J-Web jest GUI powiązanym z konkretnym urządzeniem**, a nie automatycznie odpowiednikiem centralnego Catalyst Center. Mist Cloud to jeszcze inna warstwa zarządzania. Nie utożsamiaj tych produktów.

### 3.10.2. Dlaczego pierwsze pytanie brzmi „na jakim urządzeniu i release?”

Jeśli instrukcja mówi „uruchom J-Web na MX, EX, QFX, SRX tak samo”, to za chwilę trafisz na problem dostępności. Juniper publikuje osobne dokumentacje **J-Web for Security Firewalls and Switches** i informacje o wsparciu w Feature Explorer. Nawet w jednej rodzinie funkcje i nawigacja GUI zmieniają się w kolejnych wersjach.

Przed konfiguracją:
1. Sprawdź \`show version\` i model.
2. Zweryfikuj wsparcie J-Web dla modelu i release.
3. Sprawdź, czy usługa web management jest włączona i na którym interfejsie.
4. Oceń, czy powinien to być interfejs zarządzający odseparowany od użytkowników.
5. Zweryfikuj **HTTPS, certyfikat, ograniczenia administracyjne i ewentualne security zone/host-inbound** na SRX.

### 3.10.3. Włączenie J-Web — nie twórzmy niezabezpieczonego panelu w sieci firmowej

W dokumentacji producenta występują zarówno HTTP, jak i HTTPS. **Nie włączaj HTTP na dowolnym interfejsie produkcyjnym tylko po to, by łatwiej zobaczyć GUI.** W praktyce preferujesz HTTPS z właściwym certyfikatem i ograniczonym źródłem dostępu.

Poniżej fragment **ilustracyjny**: zakłada, że odpowiedni certyfikat \`CH03-WEB-CERT\` **już istnieje** i został poprawnie zainstalowany. Nie jest to kompletny gotowy projekt PKI. Interfejs \`fxp0.0\` musi faktycznie istnieć i być przeznaczony do management na danym urządzeniu.

**Urządzenie: EX-ACCESS-01 — Configuration Mode [edit], ilustracja dla wspieranego modelu i wydania**

~~~junos
set system services web-management https local-certificate CH03-WEB-CERT
set system services web-management https interface fxp0.0
show | compare
commit check
~~~

**Nie wykonuj \`commit\` zanim nie zweryfikujesz certyfikatu, interfejsu i ACL/firewall ochrony management.** Na niektórych modelach SRX wymagane są odrębne warunki dostępności usługi i stref. Na części wirtualnych platform i hypervisorów funkcje J-Web mogą być nieobsługiwane.

Źródło: [Secure Web Access for Remote Management](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/secure-web-access-remote-management.html), [web-management CLI reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/web-management-edit-system.html).

### 3.10.4. Jak wygląda praktyczny przegląd J-Web

W systemie EX z odpowiednim pakietem J-Web producent dokumentuje ścieżkę:

**J-Web (EX, odpowiedni pakiet) → Configure → System Properties → Management Access** — do ustawień bezpiecznego dostępu przez HTTPS.

To przykład **konkretnej ścieżki menu dla określonej gałęzi GUI**, nie uniwersalna obietnica dla J-Web na SRX 25.4. J-Web na SRX może mieć osobny Setup Wizard i nawigację zależną od release.

Na wspieranym SRX w nowoczesnym J-Web możesz zobaczyć kreator standalone/HA, konfigurację interfejsów/stref, przegląd policies i monitoring. Jeśli ekran nie zgadza się z przewodnikiem sprzed pięciu lat, sprawdź **wersję J-Web i dokumentację wydania**, zamiast zgadywać nazwy przycisków.

### 3.10.5. Cztery pułapki J-Web

**Pierwsza:** urządzenie ma Junos, ale nie musi mieć J-Web w Twojej konkretnej kombinacji platform/release.

**Druga:** dostęp do GUI wymaga **odpowiedniego management reachability**, a na SRX także zasad security. Sam ping do urządzenia nie dowodzi, że TCP/443 jest dozwolony.

**Trzecia:** GUI korzysta z tego samego świata konfiguracji Junos; nie zakładaj, że równoczesne zmiany w CLI i GUI są zawsze od siebie bezpiecznie odseparowane.

**Czwarta:** „widzę zielony interfejs” nie jest dowodem, że aplikacja klienta działa. Po konfiguracji GUI nadal wykonujesz verification w CLI i na urządzeniu końcowym.

Oficjalne źródła: [J-Web for Firewalls and Switches](https://www.juniper.net/documentation/product/us/en/j-web/), [J-Web Feature Explorer](https://apps.juniper.net/feature-explorer/feature/7987?fn=J-Web+graphical+user+interface), [J-Web EX Management Access](https://www.juniper.net/documentation/us/en/software/jweb-ex/jweb-ex-application-package/topics/task/ex-series-management-access-j-web.html), [SRX J-Web Setup Wizard](https://www.juniper.net/documentation/us/en/software/jweb-srx25.4/jweb-srx/topics/topic-map/j-web-setup-wizard.html).

### 3.10.6. Czy potrzebujemy obrazka GUI?

Tak, **gdy dokładny layout pomaga wykonać zadanie**, ale screenshot z konkretnej wersji J-Web może zdezaktualizować się szybciej niż sam mechanizm. Dlatego dla podręcznika:
- połącz własny **diagram koncepcyjny CLI ↔ J-Web ↔ candidate/commit** z tekstem;
- wskaż [oficjalną stronę dostępu do J-Web](https://www.juniper.net/documentation/us/en/software/jweb-srx24.2/jweb-srx/topics/topic-map/j-web-security-starting.html), gdzie dostępne są zrzuty Setup Wizard;
- nie kopiuj zdjęć i screenshotów producenta do publicznego repo bez zweryfikowanych praw do ponownej publikacji.

~~~mermaid
flowchart LR
   USER["Administrator"] --> CLI["Junos CLI (SSH/Console)"]
   USER --> GUI["J-Web (HTTPS)"]
   CLI --> MGMT["Junos configuration subsystem"]
   GUI --> MGMT
   MGMT --> CAN["Candidate / validation"]
   CAN --> COM["Commit"]
   COM --> ACT["Active configuration i procesy"]
~~~

To nie oznacza, że J-Web i CLI są identycznymi frontendami w każdej funkcji. Oznacza, że **obydwa zarządzają tym samym systemem konfiguracji**, który musisz rozumieć niezależnie od ekranu.



---

## 3.11. [EXT] Praktyczna organizacja pracy: display set, commit check, commit confirmed i porównanie zmian

### 3.11.1. „Zielony commit complete” to dopiero początek

Pamiętasz incydent z nocnej zmiany? Na Cisco ASR dopisałeś filtr, a potem pół sieci straciło reachability. Na Junos masz szansę przetestować konfigurację, zanim stanie się aktywna. Ale oto podchwytliwy fakt: **Junos nie wie, czy Twoja aplikacja ERP będzie działać** tylko dlatego, że przeanalizował konfigurację bez błędu.

Podczas \`commit check\` system sprawdza składnię i wiele zależności oraz wymagań konfiguracji. Może wykryć błędną referencję albo niepoprawną strukturę. Nie musi natomiast wiedzieć, że nowy \`firewall filter\` odcina IP bastion hosta albo że next-hop wciąż odpowiada na ARP, ale nie prowadzi do serwera.

Dla inżyniera Cisco to łatwa analogia: poprawne przyjęcie \`ip access-group\` nie dowodzi, że aplikacja będzie działać. **Configuration valid ≠ service healthy.**

### 3.11.2. Zmiana to proces, nie pięć komend

W profesjonalnym change management rozróżnij **precheck**, **candidate review**, **validation**, **activation**, **service verification** i **confirmation**. Nie przechodź od \`set\` prosto do \`commit\`, bo to skraca najważniejszy etap — świadomą ocenę diffu.

**Urządzenie: R1-vSRX — Operational Mode, read-only precheck przed zmianą**

~~~junos
show version
show system uptime
show system alarms
show chassis alarms
show system users
show system commit
show configuration | display set
~~~

**Urządzenie: R1-vSRX — Operational Mode, wejście do prywatnej sesji zmiany**

~~~junos
configure private
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], zmiana wyłącznie w izolowanym LAB**

~~~junos
set system login message "CH03 - test change control"
show | compare
commit check
~~~

Dopiero kiedy \`show | compare\` pokazało **dokładnie to, co planowałeś**, i \`commit check\` zakończył się poprawnie, włączasz aktywację warunkową:

**Urządzenie: R1-vSRX — Configuration Mode [edit private], izolowany LAB**

~~~junos
commit confirmed 5
~~~

Teraz system pracuje z nową **active configuration**, ale uruchomił się mechanizm **oczekiwania na potwierdzenie**. Na koniec sprawdzasz, czy urządzenie nadal jest osiągalne, czy usługi działają, a change nie spowodował nieplanowanego skutku. Jeżeli jest dobrze, świadomie potwierdzasz.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], potwierdzenie dopiero PO service verification**

~~~junos
commit
~~~

### 3.11.3. Najważniejszy haczyk egzaminacyjny tego rozdziału: \`commit check\` też potwierdza \`commit confirmed\`

Wyobraź sobie, że przed chwilą wydałeś \`commit confirmed 5\`, a następnie powiedziałeś do siebie: „Jeszcze raz zrobię \`commit check\`, bo to przecież tylko sprawdzenie”. I tu Junos może Cię zaskoczyć: **oficjalna dokumentacja potwierdza, że zarówno \`commit\`, jak i \`commit check\` mogą potwierdzić oczekujący \`commit confirmed\`**.

Oznacza to, że wykonanie \`commit check\` już **po** warunkowym wdrożeniu nie musi być neutralnym testem — może zakończyć mechanizm automatycznego powrotu. To jedna z tych rzeczy, które koniecznie warto zapamiętać, bo są podchwytliwe i mają bezpośredni wpływ na produkcję.

**Przed \`commit confirmed\`:** \`commit check\` służy do sprawdzania candidate.  
**Po \`commit confirmed\`, gdy timer oczekuje:** \`commit check\` może potwierdzić zmianę i wyłączyć oczekujący rollback.

Nie „poprawiaj” nawykowo konfiguracji albo nie wykonuj kolejnego commit przed rzeczywistą kontrolą osiągalności urządzenia. Po aktywacji sprawdzaj **stan operacyjny**; jeśli zostajesz w configuration mode, użyj \`run\`.

**Urządzenie: R1-vSRX — Configuration Mode, po \`commit confirmed\`, odczyt stanu bez potwierdzania pending commit**

~~~junos
run show system commit
run show version
run show interfaces terse
~~~

**Urządzenie: R1-vSRX — Operational Mode, odczyt stanu bez modyfikacji**

~~~junos
show system commit
show system alarms
show interfaces terse
~~~

Sprawdź historię — może zawierać informację o **pending rollback in N minutes**. To dowodzi, że mechanizm ochronny jest jeszcze aktywny. Następnie wykonaj **właściwe testy usługi**, nie tylko kontrolę promptu.

Źródło: [Juniper commit CLI reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html).

### 3.11.4. \`commit confirmed 5\` nie jest obietnicą, że po pięciu minutach SSH wróci

Automatyczny rollback przywraca poprzednią konfigurację. Jeśli Twoja awaria wynikała wyłącznie ze złej konfiguracji, może to odzyskać reachability. Ale jeśli link fizyczny padł, upstream odciął Twój prefix albo urządzenie przestało działać, rollback **nie jest narzędziem naprawy hardware**.

Druga pułapka: router mógł przeprowadzić zmianę routingową, której skutki **trwają po rollbacku**, bo sąsiedzi potrzebują konwergencji, a sesje stateful muszą zostać zestawione na nowo.

Trzeci detal: dokumentacja Junipera wskazuje, że timeout \`commit confirmed\` opiera się na **czasie systemowym**. Gdy w trakcie zmieni się zegar, okres oczekiwania może się odpowiednio przesunąć. **Nie planuj eksperymentu NTP w środku pending confirmed commit.**

### 3.11.5. Czy \`commit confirmed\` można uruchomić w \`configure private\`?

Tak — na odpowiednio współczesnych wersjach Junos jest wspierany w private config. Tym bardziej ważne jest rozróżnienie **Twojej prywatnej candidate** od **globalnej aktywnej konfiguracji**. Przed aktywacją i potwierdzeniem sprawdź, czy w międzyczasie ktoś inny wykonał commit i czy Twoja candidate jest aktualna.

**Urządzenie: R1-vSRX — Configuration Mode [edit private]**

~~~junos
show | compare
commit check
commit confirmed 5
~~~

Jeśli dostaniesz konflikt z innym administratorem, **nie zmuszaj systemu do przypadkowego nadpisania zmian**. Oceń \`update\`, porównaj rezultat i uzgodnij zakres zmiany.

### 3.11.6. \`commit comment\` i historia audytu

W dużym środowisku samo „kto i kiedy” może nie wystarczyć. Warto opisać zmianę numerem ticketu, bez danych poufnych. Junos potrafi przechowywać komentarze związane z commit.

**Urządzenie: R1-vSRX — Configuration Mode [edit], przykład dopiero po precheck i zgodzie na change**

~~~junos
commit comment "CH03-LAB controlled configuration example"
~~~

To nie jest alternatywa dla ServiceNow Change Record i nie powinno się wykonywać dodatkowego commit wyłącznie po to, żeby zostawić notatkę. Zastosuj komentarz do **rzeczywistego** planowanego zatwierdzenia.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show system commit
~~~

Wynik pozwala ustalić kolejność commitów, operatorów oraz część informacji o zmianach. **Nie utożsamiaj jednak historii commit z pełnym systemem version control:** Git ma review, PR, odpowiedzialność autorów i audyt zmian między repo, a router przechowuje ograniczoną lokalną historię.

### 3.11.7. Commit na przyszłość — kuszący, lecz ryzykowny

Junos oferuje m.in. **\`commit at\`** do zaplanowanej aktywacji. Brzmi idealnie do nocnego maintenance window: przygotowujesz zmianę o 15:00, a ona sama włącza się o 23:00. Ale jeśli od rana do wieczora zmieni się stan infrastruktury, kandydat może być nieaktualny. Ponadto przy oczekującym zaplanowanym commit część innych zmian może zostać zablokowana.

To już temat na zaawansowany change management. Na poziomie tego rozdziału **wiedza o istnieniu tej funkcji** jest przydatna, ale nie używaj jej do produkcji bez osobnego runbooka, sprawdzonego zegara/timezone, planu anulowania i uzgodnienia okna.

### 3.11.8. Cisco ma mechanizmy bezpiecznego cofania — nie buduj fałszywej opowieści

Mogłoby wyglądać, że Junos wynalazł transakcyjne zarządzanie, a IOS XE nigdy nie posiadał rollbacku. To byłoby nieprawdziwe. IOS XE na odpowiednich platformach ma mechanizmy **configuration replace**, archiwizację i confirmed rollback. Różnica, którą ćwiczysz, jest przede wszystkim w **codziennym, domyślnym modelu pracy CLI Junos**: edytujesz candidate i ją commitujesz.

Dla doświadczonego inżyniera Cisco przejście jest więc prostsze niż wygląda: znasz problemy, które mechanizmy rollback mają rozwiązać. Nowe są **nawyki, nazwy, sekwencja i ich subtelne wyjątki**.

### 3.11.9. Najbardziej praktyczny runbook do zapamiętania

**FAZA 1 — przed zmianą:** sprawdź stan urządzenia, ścieżkę zarządzania, osoby zalogowane, backup, czas systemowy, OOB/console, istniejący pending commit, HLD i plan rollback.

**FAZA 2 — candidate:** najlepiej kontrolowany \`configure private\`; przygotuj tylko potrzebny zakres, \`show | compare\`; zweryfikuj, że w diffie nie ma obcych zmian ani znikającego management.

**FAZA 3 — validation:** \`commit check\` **przed** pierwszym \`commit confirmed\`; kontrola oczekiwanego efektu i dostępności platformy.

**FAZA 4 — warunkowa aktywacja:** \`commit confirmed N\` dobrane do realistycznego czasu potrzebnego do testów, nie przypadkowe „1 minuta”.

**FAZA 5 — service verification:** sprawdź realną usługę w obu kierunkach, BGP/OSPF, management, security, liczniki, monitoring i logi. Kontroluj timer przez \`show system commit\`, nie przez ponowne \`commit check\`.

**FAZA 6 — decyzja:** potwierdź \`commit\` lub pozwól na rollback/wykonaj właściwą procedurę odtworzenia. Uzupełnij worknotes i backup po zmianie.

~~~mermaid
sequenceDiagram
    participant N as Network Engineer
    participant C as Candidate
    participant J as Junos commit logic
    participant A as Active
    N->>C: set / load
    N->>C: show | compare
    N->>J: commit check
    J-->>N: validation PASS (bez aktywacji)
    N->>J: commit confirmed 5
    J->>A: activate candidate
    J-->>N: timer aktywny
    N->>A: service tests (read-only)
    alt Testy przeszły
      N->>J: commit
      J-->>N: confirmed, timer znika
    else Testy nie przeszły / brak potwierdzenia
      J->>A: automatic rollback po timeout
      A-->>N: poprzedni config (konwergencja może zająć czas)
    end
~~~

**Najważniejsze zdanie całego rozdziału:** Junos daje Ci kontrolę nad momentem aktywowania konfiguracji, ale **odpowiedzialność za jakość projektu, kolejność operacji i weryfikację aplikacji pozostaje po stronie inżyniera**.



---

## 3.LAB. [LAB] Pracujemy na prawdziwej candidate: od błędu operatora do bezpiecznego przywrócenia

### 3.LAB.1. Dlaczego ten lab potrzebuje dwóch administratorów, a nie czterech routerów?

W rozdziale 2 ćwiczyliśmy topologię diamond 4×vSRX, bo chodziło o ruch i alternatywne ścieżki. Tutaj chodzi o **zarządzanie stanem jednego urządzenia i konflikty operatorów**. Zatem do większości eksperymentów wystarczy jeden vSRX, dwie osobne sesje SSH i niezależna konsola EVE-NG. To nie jest uproszczenie kosztem jakości. To świadome ograniczenie liczby zmiennych.

~~~mermaid
flowchart TB
    A["Admin-A\nSSH session 1"] -->|shared lub private| R["R1-vSRX\nJunos config database"]
    B["Admin-B\nSSH session 2"] -->|shared lub private| R
    C["EVE-NG Console\nout-of-band recovery"] --> R
    R --> M["Adres zarządzania\nsprawdzony przed testem"]
    R --> D["Interfejs danych — opcjonalny,\nnie jest ścieżką SSH"]
~~~

**Stan laboratorium: PLANNED.** Nie uruchamiano go w EVE-NG; nie podajemy fikcyjnych wyników ani adresów serwera.

#### Warunki wejścia

- EVE-NG ma działający, legalny obraz vSRX i odpowiednie zasoby.
- Masz dostęp do konsoli R1 niezależny od interfejsu, którym działa SSH.
- Potwierdziłeś rzeczywisty adres management, port i drogę SSH. vSRX może wykorzystywać \`fxp0\`, ale nie zakładaj tego bez \`show interfaces terse\`.
- Test przeprowadzasz w **izolowanej sieci laboratoryjnej**, bez połączenia do produkcji/biura.
- Znasz uprawnienia obu kont; jeśli nie masz drugiego konta, utworzenie go należy do rozdziału 4 albo do zatwierdzonego lab startup.
- Wspólna candidate nie ma niezatwierdzonych zmian innych osób; jeśli ma, nie zaczynaj ćwiczenia z shared candidate.
- Zapisujesz precheck, aktualny hostname, opis testowego portu, konfigurację login message i historię commitów.

#### Niezależność od wcześniejszych ćwiczeń

Ten rozdział **nie kasuje adresacji z laboratorium CH02**. Eksperymenty opierają się na \`system login message\` oraz opisach testowego interfejsu **GE-DATA-TEST**. Przed wykonaniem wybierz faktyczny port, który **nie przenosi management**. W przykładach nazwa \`ge-0/0/2\` jest tylko założeniem do adaptacji; jeżeli na R1 jest to port PC-A, wybierz inną bezpieczną gałąź lub wykonaj wyłącznie testy login message.

### 3.LAB.2. Test zerowy: zobacz stan przed zmianą

**Urządzenie: R1-vSRX — Operational Mode (Admin-A lub konsola)**

~~~junos
show version
show system uptime
show system users
show system commit
show interfaces terse
show configuration system login | display set
show configuration interfaces ge-0/0/2 | display set
~~~

**Zadanie ucznia:** zapisz w notatniku, z którego portu zarządzania jesteś połączony. Sprawdź, czy \`ge-0/0/2\` nie jest elementem management. Zapisz istniejącą konfigurację (bez sekretów i danych produkcyjnych). Ustal, czy inne sesje już edytują shared candidate.

**Urządzenie: R1-vSRX — Configuration Mode, tylko po potwierdzeniu, że jesteś jedynym autorem w shared candidate**

~~~junos
configure
status
show | compare
exit
~~~

Jeżeli \`show | compare\` nie jest puste, **nie wykonuj \`rollback 0\`** na cudzej pracy, tylko ustal z zespołem, co znajduje się w candidate. Jeżeli prowadzisz własny lab bez innych operatorów, możesz dopiero wtedy bezpiecznie przywrócić baseline.

### 3.LAB.3. Ćwiczenie A — candidate nie zmienia active

Prowadzący mówi: „Dodaj banner i pokaż mi, że użytkownicy nadal widzą stary banner, bo niczego jeszcze nie commitowałeś”.

**Urządzenie: R1-vSRX — Operational Mode (Admin-A)**

~~~junos
configure private
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-A)**

~~~junos
set system login message "CH03 - CANDIDATE ONLY"
show | compare
run show configuration system login message
~~~

**Oczekiwany rezultat:** \`show | compare\` pokazuje przygotowaną zmianę; operacyjny odczyt active przez \`run show configuration ...\` nie pokazuje nowego banneru, dopóki nie ma commit. Jeżeli różnica jest inna, zatrzymaj ćwiczenie — możliwe, że system miał już taki sam tekst albo w międzyczasie zmienił się active.

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-A), cleanup testu A**

~~~junos
rollback 0
show | compare
exit
~~~

**Nauczony mechanizm:** \`rollback 0\` czyści prywatne niezacommitowane zmiany. W tym ćwiczeniu nie powinno powstać nowe entry historii commit.

### 3.LAB.4. Ćwiczenie B — dwie sesje, jedna shared candidate

To najważniejszy eksperyment związany z \`configure\`. Nie potrzebujesz setek poleceń — wystarczy jeden banner i jedna opisowa linia na bezpiecznym porcie, **jeżeli port jest zweryfikowany jako niezarządzający**.

**Urządzenie: R1-vSRX — Operational Mode (Admin-A)**

~~~junos
configure
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit] (Admin-A)**

~~~junos
set system login message "CH03 - ADMIN A"
show | compare
~~~

W drugiej sesji:

**Urządzenie: R1-vSRX — Operational Mode (Admin-B)**

~~~junos
configure
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit] (Admin-B), wyłącznie potwierdzony port danych**

~~~junos
set interfaces ge-0/0/2 description "CH03-ADMIN-B"
show | compare
status
~~~

Teraz wróć do Admin-A i porównaj candidate:

**Urządzenie: R1-vSRX — Configuration Mode [edit] (Admin-A), nie zatwierdzaj jeszcze**

~~~junos
show | compare
status
~~~

**Przewidywanie:** w shared candidate Admin-A może zobaczyć **oba** niezacommitowane fragmenty. Jeśli teraz wykonałby commit, mógłby zatwierdzić również zmianę Admin-B. To jest sedno problemu. **Nie musisz wykonywać rzeczywistego commit, aby udowodnić mechanikę shared candidate.**

Na końcu, po uzgodnieniu obu sesji:

**Urządzenie: R1-vSRX — Configuration Mode [edit] (Admin-A), czyszczenie wspólnego LAB po zgodzie Admin-B**

~~~junos
rollback 0
show | compare
exit
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit] (Admin-B), zakończenie sesji**

~~~junos
exit
~~~

Upewnij się, że \`show | compare\` nie zawiera cudzych zmian. Jeśli lab był współdzielony z kimś spoza ćwiczenia, nie używaj tu automatycznie \`rollback 0\`.

### 3.LAB.5. Ćwiczenie C — private candidate, a międzyczasie inny administrator commitował

Tym razem Admin-A przygotowuje nowy banner w private, a Admin-B wprowadza zmianę **inną i niezależną** (np. opis portu). Konieczna jest niezależna konsola, choć same operacje mogą być bezpieczne.

**Urządzenie: R1-vSRX — Operational Mode (Admin-A)**

~~~junos
configure private
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-A)**

~~~junos
set system login message "CH03 - PRIVATE-A"
show | compare
~~~

**Urządzenie: R1-vSRX — Operational Mode (Admin-B)**

~~~junos
configure private
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-B), port danych zweryfikowany**

~~~junos
set interfaces ge-0/0/2 description "CH03-PRIVATE-B"
show | compare
commit check
commit confirmed 5
~~~

Admin-B wykonuje test i świadomie potwierdza:

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-B), dopiero po weryfikacji**

~~~junos
commit
exit
~~~

Teraz Admin-A ma private candidate utworzoną przed commit Admin-B. Jaką otrzyma odpowiedź przy próbie własnego commit? Zależy od wydania, konfliktów i aktualnego stanu. **Nie próbuj siłowo commitować**. Sprawdź zmianę globalną i wykonaj \`update\` w private.

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-A)**

~~~junos
show | compare
update
show | compare
commit check
~~~

Oczekiwany koncept: prywatna candidate Admin-A powinna zostać uzgodniona z nowym active w takim zakresie, w jakim nie ma konfliktu. Gdy zobaczysz conflict, rozwiązujesz go, nie zakładasz, że \`update\` wszystko naprawia.

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-A), aktywacja po poprawnym update, tylko LAB**

~~~junos
commit confirmed 5
~~~

Po service verification:

**Urządzenie: R1-vSRX — Configuration Mode [edit private] (Admin-A)**

~~~junos
commit
exit
~~~

Na zakończenie przywróć dokładnie wyjściowy banner i opis na podstawie zapisanego **precheck**, nie przez zgadywanie, że przed testem nie było opisu. Najbezpieczniej przygotować oddzielny, zweryfikowany plan cofnięcia lub rozważyć kontrolowany rollback do zapisanego punktu.

### 3.LAB.6. Ćwiczenie D — wczytaj konfigurację, ale nie zastępuj całego urządzenia

W tym eksperymencie nie potrzebujemy nawet zewnętrznego SCP. Wykorzystaj \`load set terminal\` i \`load merge terminal\`, aby zobaczyć różnicę formatów.

**Urządzenie: R1-vSRX — Operational Mode (Admin-A)**

~~~junos
configure private
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], rozpoczęcie importu set**

~~~junos
load set terminal
~~~

**Dane wprowadzane do R1-vSRX jako zawartość terminalowego \`load set\` (potem Ctrl+D)**

~~~junos
set system login message "CH03 - LOADED-SET"
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], po Ctrl+D**

~~~junos
show | compare
rollback 0
~~~

Teraz drugi wariant:

**Urządzenie: R1-vSRX — Configuration Mode [edit private], rozpoczęcie importu hierarchy**

~~~junos
load merge terminal
~~~

**Dane wprowadzane do R1-vSRX jako hierarchy, potem Ctrl+D**

~~~junos
system {
    login {
        message "CH03 - LOADED-MERGE";
    }
}
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], po Ctrl+D**

~~~junos
show | compare
rollback 0
exit
~~~

**Co się wydarzyło?** Dwa różne formaty importu prowadzą do podobnej zmiany w candidate, ale mechanika parsera i zakres zastąpienia są inne. **Nie uruchamialiśmy \`load override\`**, ponieważ nie jest to potrzebne do osiągnięcia celu szkoleniowego.

### 3.LAB.7. Ćwiczenie E — kontrolowany \`commit confirmed\` z automatycznym rollbackiem

To eksperyment, który naprawdę warto wykonać po drugiej stronie Cisco.

**Cel:** aktywować nieszkodliwy login banner na krótki czas i sprawdzić, że Junos sam wróci do wcześniejszego active, jeśli operator nie potwierdzi zmiany.

**Nie wycinamy SSH i nie usuwamy adresu IP management.** Dla zrozumienia mechanizmu wystarczy bezpieczna zmiana, po której obserwujemy timer.

**Urządzenie: R1-vSRX — Operational Mode (Admin-A)**

~~~junos
show system commit
configure private
~~~

**Urządzenie: R1-vSRX — Configuration Mode [edit private], odseparowany LAB**

~~~junos
set system login message "CH03 - TEMPORARY CONFIRMED CHANGE"
show | compare
commit check
commit confirmed 2
~~~

Teraz bardzo uważnie: **nie wydawaj \`commit check\` ponownie**, bo może potwierdzić oczekujący commit.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], odczyt po aktywacji**

~~~junos
run show system commit
run show configuration system login message
~~~

**Urządzenie: R1-vSRX — Operational Mode (Admin-B albo EVE console), obserwacja niezależna**

~~~junos
show system commit
show configuration system login message
~~~

Przewidywanie: przez chwilę aktywna konfiguracja zawiera nowy login message, a \`show system commit\` informuje o oczekiwanym rollback. Po upływie okresu oczekiwania bez potwierdzenia Junos powinien załadować wcześniejszy active, a historia commitów odnotować zdarzenie. Zapisz wyniki przed i po.

**Jeśli inne procesy lub administratorzy nadal wykonują commity na tym urządzeniu, przerwij ćwiczenie**. Użyj dedykowanego laba. Nie uznawaj, że odzyskanie starego banneru jest dowodem, że w produkcji wszystkie sesje BGP wróciłyby natychmiast.

### 3.LAB.8. Ćwiczenie F — rollback 1, rescue i porównanie dwóch historii

Jeżeli w eksperymencie D nie wykonałeś realnego commit, nie powinien on tworzyć wpisu historii. W eksperymencie E \`commit confirmed\` i auto-rollback już wpływają na historię. To dobra okazja do zrozumienia, **dlaczego numeracja rollback jest względna**.

**Urządzenie: R1-vSRX — Operational Mode (read-only)**

~~~junos
show system commit
show system rollback 0
show system rollback 1
show system rollback 0 compare 1
~~~

Przewidź, która z poprzednich wersji zawiera eksperymentalny banner. **Nie zakładaj, że rollback 1 to zawsze baseline sprzed całego rozdziału.** Jeśli zrobiłeś kilka commitów, indeks może wskazywać inną wersję.

Jeśli jako właściciel laba masz już prawidłową, zweryfikowaną konfigurację bazową i uzgodniłeś zapis rescue:

**Urządzenie: R1-vSRX — Operational Mode, po walidacji baseline, uwaga: nadpisuje istniejący rescue**

~~~junos
request system configuration rescue save
~~~

Na koniec laboratorium **nie zostawiaj w urządzeniu przypadkowych banerów, opisów portów, blokad exclusive ani niepotwierdzonych commitów**. Wykonaj końcowe pre/post porównanie z zapisanym baseline.

### 3.LAB.9. Ćwiczenie G (opcjonalne) — rzeczywista utrata SSH i recovery

To ćwiczenie jest **opcjonalne i bardziej ryzykowne**. Można je przeprowadzić tylko jeśli:
- masz **niezależną konsolę** i fizycznie/logiczną mapę połączenia;
- wiesz, którym interfejsem przechodzi SSH;
- nie korzystasz z management firmy ani chmury klienta;
- masz pełną kopię konfiguracji, rollback plan oraz możliwość odzyskania urządzenia po automatycznym rollback.

Nie podaję gotowej komendy usuwającej adres management, ponieważ bez znajomości mapowania Twojego EVE-NG mogłaby dotyczyć **niewłaściwego interfejsu**. Codex, mając rzeczywisty precheck i odpowiednie uprawnienia, może przygotować wersję tego testu z odłączaną **sztuczną** ścieżką SSH w izolowanej sieci. Minimalny test E powyżej w zupełności wystarcza do zrozumienia \`commit confirmed\` na JNCIA.

### 3.LAB.10. Kiedy lab jest naprawdę ukończony?

Nie dlatego, że wpisałeś wszystkie komendy, tylko gdy potrafisz pokazać **dowody**:
- Kiedy candidate różni się od active, a kiedy obie są takie same?
- Czy w shared candidate Admin-A widzi zmianę Admin-B?
- Czy w private candidate trzeba było użyć \`update\` i dlaczego?
- Czy \`load set\` i \`load merge\` przyniosły oczekiwany diff bez przypadkowego usunięcia reszty konfiguracji?
- Czy \`commit confirmed\` faktycznie zadziałał, gdy **nie** potwierdzono?
- Jak wygląda \`show system commit\` przed i po rollbacku?
- Czy odtworzyłeś początkowy stan interfejsów, login message i routingu?

Prawdziwy wynik weryfikacji zapisz w \`VERIFY.md\` wraz z datą, wersją obrazu Junos, nazwą topologii oraz konkretnymi wynikami. **Nie wpisuj PASS, jeśli nie zrobiłeś testu na działającym urządzeniu.**



---

## 3.TS. Troubleshooting — gdy zmiana Junos nie zachowuje się tak, jak oczekujesz [EXT]

### 3.TS.1. „Commit failed”. Czy router wrócił do poprzedniego stanu?

Pamiętasz scenę z nocnego dyżuru: w IOS XE wklejone polecenia mogą już działać, choć jedna linijka była błędna. W standardowym workflow Junos **nieudany \`commit check\` lub nieudany commit oznacza, że proponowane zmiany nie zostały skutecznie aktywowane jako oczekiwana nowa konfiguracja**. Jednak nie buduj nieprawdziwej pewności, że nic innego w systemie w tym czasie się nie wydarzyło — inni operatorzy, działania operacyjne, stan protokołów i wcześniejsze commity wciąż istnieją.

**Urządzenie: R1-vSRX — Configuration Mode, analiza nieudanej próby**

~~~junos
top
show | compare
commit check
~~~

Najpierw czytaj **pełny komunikat błędu**. Jeśli system mówi o brakującej referencji, poprawiasz referencję. Jeśli o konflikcie zmiany innego administratora, patrzysz na \`configure private\` i \`update\`. Jeśli o błędnym statement, użyj \`?\` dokładnie tam, gdzie parser go odrzucił.

**Nie naprawiaj błędu losowym \`load override\`**. Wtedy zmienisz kategorię problemu z jednej brakującej referencji w potencjalną utratę całej konfiguracji.

### 3.TS.2. „Nie widzę zmiany, którą właśnie wpisałem”

Pomyśl, co tak naprawdę zostało zrobione. Jeśli wykonałeś tylko \`set\`, zmiana może być w candidate. Jeśli jesteś w \`[edit interfaces ...]\`, a patrzysz na inny interfejs, możesz wyciągnąć fałszywy wniosek. Jeśli wchodzi w grę \`apply-groups\`, lokalny widok może nie pokazywać wartości odziedziczonych.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
top
show | compare
show | display set
run show configuration system login message
~~~

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show configuration system login message
show system commit
~~~

**Diagnoza:** odróżnij candidate, active i aktywny stan usługi. Przeczytanie wyłącznie jednego \`show\` bez znajomości trybu CLI jest częstą przyczyną fałszywego zgłoszenia „config nie zapisuje zmian”.

### 3.TS.3. „Mój \`commit\` wgrał zmianę kolegi”

To klasyczny błąd pracy ze shared candidate. Jeśli Admin-A wykonał \`configure\`, Admin-B także, ich zmiany mogą znaleźć się w **tej samej bazie candidate**. Jeden commit potrafi aktywować obie.

**Urządzenie: R1-vSRX — Configuration Mode [edit]**

~~~junos
status
show | compare
~~~

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show system users
show system commit
~~~

Jeżeli błąd już się wydarzył, wróć do historii i change record. Nie cofaj całej konfiguracji w ciemno — możesz cofnąć również zmianę kolegi, która była potrzebna. Na produkcji najpierw uzgodnij zakres przywracania i ewentualnie przygotuj **korektę punktową**, zamiast zbiorczego rollbacku.

### 3.TS.4. „Private config nie chce wejść po tym, jak inny administrator zrobił commit”

Ten problem jest normalną konsekwencją równoczesnych zmian, nie magicznym błędem mgd. Private candidate powstał ze starszego active. Weryfikacja podczas commit może wykryć, że ktoś zmienił konfigurację od tego czasu.

**Urządzenie: R1-vSRX — Configuration Mode [edit private]**

~~~junos
show | compare
update
show | compare
~~~

Jeśli jest konflikt, zatrzymaj się. **Nie zakładaj, że \`update\` nadpisuje wszystko poprawnie.** Odpowiedz: jaki obiekt zmienili obaj administratorzy, jaki był zamierzony efekt, czy możemy rozdzielić zmianę na osobne commity?

### 3.TS.5. „Trasa działa, ale diff po \`load set\` wygląda podejrzanie”

Architekt przesłał plik z konfiguracją „z innej lokalizacji”. Po \`load set\` zobaczyłeś przypadkową zmianę management IP i nazw interfejsów.

Nie ignoruj tego tylko dlatego, że \`commit check\` przeszedł. Wczytanie poprawnej składni nie dowodzi zgodności z Twoim HLD.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], bezpieczna diagnostyka przed commit**

~~~junos
show | compare
show | display set
~~~

Jeżeli nie jesteś pewien, co się zmieniło, **nie commituj**. Jeśli wyłącznie Ty pracujesz w private candidate i nie masz innych potrzebnych niezacommitowanych zmian, możesz ją odrzucić:

**Urządzenie: R1-vSRX — Configuration Mode [edit private], tylko własne niezacommitowane zmiany**

~~~junos
rollback 0
show | compare
~~~

**Następny krok:** popraw plik w systemie wersjonowania lub z architektem. Nie edytuj produkcyjnego routera metodą prób i błędów.

### 3.TS.6. „Po \`commit confirmed\` wszystko działało, a potem się cofnęło”

To nie jest awaria Junos. Jeśli wykonałeś \`commit confirmed 5\`, ale przez pięć minut nie potwierdziłeś zmiany, system powinien automatycznie przywrócić poprzedni config.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show system commit
show configuration system login message
~~~

**Pytanie egzaminacyjne:** czy koniecznie musisz wykonać \`commit\`, by potwierdzić? **Nie — \`commit check\` również może to zrobić.** Nie uruchamiaj więc go po confirmed commit jako „neutralnej próby”, jeśli chcesz zachować rollback timer.

### 3.TS.7. „Po \`rollback 1\` zobaczyłem tylko starą konfigurację w candidate”

To poprawne zachowanie. Załadowanie starej konfiguracji nie jest równoznaczne z jej aktywacją. Przed commit możesz jeszcze obejrzeć różnicę.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], przykład tylko po świadomym wyborze wersji**

~~~junos
rollback 1
show | compare
commit check
~~~

Zanim aktywujesz ją na produkcji, policz zakres strat: czy poprzednia wersja zawiera wszystkie potrzebne zmiany z ostatniego incydentu, w tym pracę innych administratorów? Mechaniczny rollback może być bardziej destrukcyjny od precyzyjnego usunięcia wadliwej linijki.

### 3.TS.8. „J-Web nie działa; SSH i ping działają”

To bardzo dobra okazja, żeby połączyć rozdział 2 z rozdziałem 3. Ping do lokalnego adresu SRX to inny rodzaj ruchu i inna usługa niż HTTPS do J-Web. Jeśli SSH działa, potwierdziłeś jedną usługę, nie cały management.

Sprawdź kolejno: czy platforma obsługuje J-Web; czy serwer HTTPS jest poprawnie skonfigurowany; na jakim interfejsie; czy certyfikat istnieje; czy SRX host-inbound oraz filtr dopuszcza TCP/443; czy routing zwrotny działa; czy to właściwa wersja GUI.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show version
show configuration system services web-management | display set
show configuration security zones | display set
show interfaces terse
show system alarms
~~~

Na różnych wydaniach J-Web są inne modele pakietów i wsparcia. Nie otwieraj portu management na każdym interfejsie ani nie włączaj nieszyfrowanego HTTP jako „szybkiego obejścia”.

### 3.TS.9. „Przez \`show configuration\` widzę coś innego niż po \`show | display inheritance\`”

Znasz potencjalną przyczynę: **configuration groups** i inne mechanizmy dziedziczenia. Lokalny tekst może być krótszy niż wynikowe ustawienia przekazane do procesu. W kontekście CLI to nie błąd — to inny **widok tej samej logicznej konfiguracji**.

**Urządzenie: EX-ACCESS-01 — Operational Mode (jeżeli urządzenie korzysta z groups)**

~~~junos
show configuration interfaces | display inheritance
show configuration interfaces | display set
~~~

Dokładną politykę dziedziczenia i konfigurację groups omówimy w rozdziale 4.

### 3.TS.10. „Nie mogę nic zrobić, bo inny administrator trzyma exclusive”

Nie rozwiązuj tej sytuacji od razu przez zabicie sesji operatora. \`configure exclusive\` jest mechanizmem blokady. Jeśli blokada jest uzasadniona change window, poczekaj na jej zwolnienie zgodnie z procedurą. W rzeczywistym incydencie P1 skontaktuj się z właścicielem lock i zastosuj formalny tryb awaryjny.

Dokumentacja Junipera opisuje administracyjne wylogowanie operatora, ale to może **przerwać cudzą pracę i spowodować utratę niezacommitowanych zmian**. Nie przedstawiamy tego jako rutynowego sposobu rozwiązania konfliktu.

---

## 3.TEST. Sprawdzian rozumowania — 16 sytuacji, nie 16 definicji [EXT]

*Pytania są autorskie, oparte na jawnych mechanizmach producenta. Nie są kopiami pytań z oficjalnego assessmentu JNCIA ani z exam dumpów.*

Spróbuj odpowiedzieć **przed** przeczytaniem wyjaśnienia. Nie chcę, żebyś wybierał literę A/B/C/D na pamięć. Chcę, żebyś przewidział skutek.

**1.** Zalogowałeś się do Junos i widzisz \`user@R1>\`. Czy jesteś pełnym administratorem? Czy możesz wykonać komendę wpływającą na działanie routera?

**2.** Jesteś w \`[edit interfaces ge-0/0/1]\`. Co zrobi \`up\`, co zrobi \`top\`, a co może zrobić \`exit\`?

**3.** Będąc w \`[edit]\`, wykonujesz \`set system host-name R1\`, lecz nie robisz commit. Co zobaczysz w \`show | compare\`, a co w operational \`show configuration system host-name\`?

**4.** Kolega w shared \`configure\` zmienia BGP; Ty w tym samym shared candidate zmieniasz opis portu. Ktoś wykonuje commit. Co może zostać aktywowane?

**5.** Dlaczego \`configure private\` nie gwarantuje, że commit nigdy nie zakończy się konfliktem?

**6.** Co robi \`update\` w private candidate i czym różni się od \`load update\`?

**7.** Dlaczego \`show route | count\` nie daje poprawnej liczby tras w RIB?

**8.** Zamiast \`load set\` dla trzech linijek ktoś wybiera \`load override\`. Co jest największym ryzykiem po commit?

**9.** Jakiego znacznika w hierarchicznym pliku szuka \`load replace\`?

**10.** Co oznacza \`rollback 0\` przy niezacommitowanych zmianach? A co zrobi \`rollback 1\` po wcześniejszym commit?

**11.** Czy \`save /var/tmp/backup.conf\` jest tym samym co \`commit\`?

**12.** Wykonałeś \`commit confirmed 5\`, po czym \`commit check\`. Czy pięciominutowy automatyczny rollback nadal na pewno czeka?

**13.** \`commit check\` zakończył się poprawnie. Czy na pewno Twoja nowa ACL pozwoli utrzymać SSH do urządzenia?

**14.** Dlaczego ping do IP SRX i SSH do tego SRX nie dowodzą, że J-Web działa?

**15.** Gdy \`rollback 1\` zostanie załadowany do candidate, czy połączenie klienta natychmiast zacznie używać poprzednich tras?

**16.** Dlaczego ekran \`show configuration interfaces\` może nie pokazywać wszystkich ustawień działających na interfejsie?

### Rozwiązania — z wyjaśnieniem mechanizmu

**1.** Prompt oznacza operational mode, nie superuser. Uprawnienia wynikają z login class/AAA. W tym trybie można wykonywać również potencjalnie destrukcyjne działania operacyjne, zależnie od uprawnień.

**2.** \`up\` przechodzi poziom wyżej; \`top\` wraca do \`[edit]\`; \`exit\` w zagnieżdżonej hierarchii zwykle przechodzi wyżej, a przy \`[edit]\` wychodzi z configuration mode.

**3.** Candidate będzie inny, więc diff pokaże dodanie/zmianę; operational \`show configuration\` nadal pokazuje active sprzed commit.

**4.** Obie zmiany, ponieważ shared candidate jest wspólny. To ryzyko audytowe i operatorskie.

**5.** Inny administrator może w międzyczasie zmienić active; prywatna candidate bazuje na starszej konfiguracji i może wymagać \`update\` albo ręcznej korekty konfliktu.

**6.** \`update\` w private synchronizuje prywatny stan z aktualnym active. \`load update\` to tryb wczytania **pełnej konfiguracji z pliku** przy selektywnym oznaczaniu zmian.

**7.** \`| count\` liczy linie tekstu wyjścia, a nie rekordy route. Wielowierszowe wpisy i nagłówki zniekształcają wynik.

**8.** \`override\` zastępuje całą candidate. Brak w pliku kompletnych kont, interfejsów i routing mógłby po commit usunąć je z active.

**9.** Znacznika **\`replace:\`** przy konkretnej gałęzi konfiguracji.

**10.** \`rollback 0\` przywraca current active do candidate (odrzuca niezacommitowane zmiany). \`rollback 1\` ładuje poprzednią zatwierdzoną konfigurację do candidate i wymaga dalszego commit, by stała się aktywna.

**11.** Nie. \`save\` zapisuje tekst konfiguracyjny do pliku; \`commit\` zapisuje i aktywuje nowy stan konfiguracji.

**12.** Nie. \`commit check\` potrafi potwierdzić oczekujący \`commit confirmed\`. To bardzo ważny, nieintuicyjny haczyk.

**13.** Nie. Poprawność składni i zależności konfiguracyjnych nie jest testem funkcjonalnym z punktu widzenia zdalnego klienta.

**14.** To różne usługi i porty. HTTPS może nie być włączony, certyfikat może być nieprawidłowy, J-Web niewspierany albo TCP/443 zablokowany.

**15.** Nie. Zmieniła się na razie candidate, nie active. Trzeba porównać i zatwierdzić odpowiednią konfigurację.

**16.** Ponieważ konfiguracja może być dziedziczona z \`apply-groups\`. \`display inheritance\` pomaga zobaczyć wynikowy obraz.

### Kryterium gotowości do kolejnego rozdziału

Uznaj rozdział za opanowany, jeśli potrafisz **bez notatek**:
- pokazać candidate i active oraz ich różnicę;
- odczytać historię rollback i świadomie zdecydować, który snapshot przywrócić;
- wybrać poprawny tryb \`load\` dla fragmentu \`set\` i pełnego config;
- wyjaśnić, co zobaczą Admin-A i Admin-B w shared/private;
- wykonać \`commit confirmed\`, sprawdzić pending timer bez nieświadomego potwierdzania i bezpiecznie wrócić do baseline.

Jeśli nie potrafisz, nie ucz się na pamięć kolejnej tabeli. Wróć do eksperymentów A, B, D i E, zadaj sobie pytanie „**co zostało zatwierdzone, a co jeszcze nie?**”, i sprawdź hipotezę.



---

## 3.REF. Ściąga operatorska — dopiero po zrozumieniu mechanizmów [EXT]

Gdy czytasz tabelę poniżej, nie traktuj jej jak listy do wykucia. To mapa pytań, które pojawiają się przy nocnym change'u: **co obserwuję, co edytuję, co porównuję, co zatwierdzam i jak wracam?** Odpowiedzią na każde pytanie jest konkretne polecenie.

| Pytanie inżyniera | Polecenie/rodzina | Najważniejszy haczyk |
|---|---|---|
| Który tryb CLI? | prompt \`>\` / \`#\` | Uprawnienia nie wynikają z samego promptu |
| Czy inni operatorzy pracują? | \`show system users\`, \`status\` | Shared candidate może zawierać cudze zmiany |
| Jak wejść do edytora? | \`configure\`, \`configure private\`, \`configure exclusive\` | Private nie izoluje od zmian active przez innych |
| Jak poruszać się po konfiguracji? | \`edit\`, \`up\`, \`top\`, \`exit\` | \`exit\` zależy od poziomu |
| Jak sprawdzić operational bez wychodzenia? | \`run show ...\` | Nie każde operational show jest obsługiwane przez \`run\` |
| Jak znaleźć składnię? | \`?\`, \`Tab\`, \`help apropos\` | Zależność od platformy/release |
| Jak skrócić wyjście? | \`match\`, \`except\`, \`find\`, \`last\` | \`find\` pokazuje kontekst po pierwszym trafieniu |
| Ile mam rekordów? | \`pipe count\` | **Liczy linie, nie trasy** |
| Jak zobaczyć konfigurację jako set? | \`display set\` | Dotyczy konfiguracji, nie wszystkich operational show |
| Jak sprawdzić dziedziczenie? | \`display inheritance\` | Widok lokalny może być niekompletny |
| Jak dopisać/wycofać statement? | \`set\`, \`delete\`, \`activate\`, \`deactivate\` | \`delete\` gałęzi może usunąć cały interfejs |
| Jak załadować plik set? | \`load set\` | Nie używaj \`override\` do fragmentów |
| Jak wczytać fragment drzewa? | \`load merge\`, \`load replace\` | \`replace:\` decyduje o zakresie zastąpienia |
| Jak zastąpić cały config? | \`load override\` | Bardzo wysokie ryzyko utraty management |
| Jak porównać candidate? | \`show | compare\` | Plus/minus nie gwarantuje poprawności projektu |
| Jak sprawdzić zapisane wersje? | \`show system commit\`, \`show system rollback\` | \`rollback n\` ładuje candidate |
| Jak sprawdzić config? | \`commit check\` | **Po commit confirmed może potwierdzić zmianę** |
| Jak warunkowo wdrożyć zmianę? | \`commit confirmed N\` | Test usług przed \`commit\` |
| Jak przywrócić stary config? | \`rollback 1\`, \`rollback rescue\` | Zwykle wymaga kolejnego commit |
| Jak zapisać plik roboczy? | \`save /var/tmp/...\` | Save ≠ commit, local file ≠ off-device backup |

**Dodatkowe pojęcie dla poziomu Professional:** \`commit and-quit\` to wygodna forma zatwierdzenia i opuszczenia configuration mode po sukcesie. Z uwagi na to, że w jednej chwili aktywuje zmianę, **nie wybieraj jej zamiast \`commit confirmed\` przy ryzykownym zdalnym change'u**.

**Urządzenie: R1-vSRX — Configuration Mode, przykład polecenia do świadomego użycia po pełnej weryfikacji**

~~~junos
commit and-quit
~~~

---

## 3.SOURCES. Dokumentacja, pytania i ograniczenia materiału

**Wszystkie linki poniżej są oficjalnymi źródłami Juniper/HPE, poza nazwanymi analogiami Cisco i autorskimi scenariuszami.** Dokumentacja potwierdza semantykę mechanizmów; nie oznacza, że każdy fragment jest dostępny we wszystkich rodzinach sprzętu. W szczególności **żadna przykładowa komenda z tej lekcji nie została tutaj zweryfikowana na uruchomionym vSRX**.

1. [CLI Overview](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-overview.html) — rola CLI i tryby.
2. [CLI Configuration Mode Overview](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html) — hierarchia i polecenia edycyjne.
3. [Overview of the Configure Command](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configure-command.html) — shared/private/exclusive, update i konflikty.
4. [CLI Environment Settings](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-environment-settings.html) — screen-length, autocomplete, prompt.
5. [help CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/help-mode-operation.html) — help apropos, topic i reference.
6. [pipe CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/pipe.html) — match, except, find, count, compare.
7. [show | display CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-pipe-display.html) — set/inheritance/XML/JSON.
8. [Configuration Groups](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configuration-groups-usage.html) — widok dziedziczonych statementów.
9. [Modifying Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/modifying-configuration.html) — copy, rename, replace pattern.
10. [load CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/load.html) — set, merge, override, replace, update, patch.
11. [Loading Configuration Files](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-config-files-loading.html) — plik, terminal, relative i replace.
12. [save Configuration Mode](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/save-cli-config.html) — zapisywanie konfiguracji jako ASCII.
13. [Filtering Operational Command Output](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/filtering-operational-command.html) — compare, pipe i kontekst.
14. [rollback CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/rollback.html) — numeracja i zakres.
15. [show system rollback](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-rollback.html) — bezpieczny odczyt i porównania historii.
16. [show system commit](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-commit.html) — historia, pending rollback.
17. [commit CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html) — confirmed, check, at, comment, synchronize.
18. [Commit the Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html) — model candidate/active.
19. [Rescue Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-rescue-configuration.html) — tworzenie i przywracanie.
20. [J-Web for Firewalls and Switches](https://www.juniper.net/documentation/product/us/en/j-web/) — platformy, pakiety i wydania.
21. [J-Web secure access](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/secure-web-access-remote-management.html) — HTTPS, certyfikaty.
22. [J-Web EX Management Access](https://www.juniper.net/documentation/us/en/software/jweb-ex/jweb-ex-application-package/topics/task/ex-series-management-access-j-web.html) — ścieżka GUI.
23. [J-Web SRX Setup Wizard](https://www.juniper.net/documentation/us/en/software/jweb-srx25.4/jweb-srx/topics/topic-map/j-web-setup-wizard.html) — funkcje GUI dla konkretnego wydania.
24. [Day One: Exploring the Junos CLI, Second Edition](https://www.juniper.net/documentation/en_US/day-one-books/ExploreJunosCLI_2ndEd.pdf) — przykład uczenia się CLI i commit confirmed; starsze wydanie, nie źródło aktualnej platform-specific składni.
25. [Day One: Beginner's Guide to Learning Junos](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf) — kontekst szkoleniowy i podstawy rollback.

**Źródła pytań:** zawarte w rozdziale pytania typu „co się stanie?” są **autorskimi zadaniami dydaktycznymi** opartymi na funkcjach potwierdzonych w dokumentacji, nie rzekomo wykradzionymi pytaniami egzaminu. W przypadku JNCIA Practice Exam nie deklaruję odczytu zawartości jego interaktywnego banku. Dla rozdziału 3 źródłem wartościowych gotchas są m.in. szczegółowe opisy \`configure\` i \`commit\`, które zostały przetworzone w scenariusze.

**Ograniczenie ważne dla inżyniera:** przykłady używają standardowego Junos CLI na vSRX jako modelu. Platformy SRX w HA cluster, Virtual Chassis, Junos OS Evolved, wydania z konfiguracją ephemeral/dynamic, FIPS oraz różne pakiety J-Web mogą mieć **inne zachowania albo ograniczenia**. Każdą komendę konfrontuj z właściwym wydaniem i dokumentacją producenta.

---

## 3.END. Końcowa lekcja — czym Junos naprawdę zmienia Twoją pracę?

Wyobraź sobie ponownie godzinę 03:07. Przygotowujesz zmianę w dużej sieci. Masz do wyboru dwa sposoby myślenia.

Pierwszy mówi: „Mam 50 linii do wklejenia. Mam nadzieję, że wszystko zadziała”.

Drugi mówi: „Wiem, jakie **pięć rezultatów** musi być spełnionych, zanim zakończę change: właściwy diff, poprawna walidacja konfiguracji, aktywna sesja zarządzająca, sprawna usługa i udokumentowany rollback”.

Tym drugim sposobem będziemy pracować w Junos.

Nie musisz uczyć się na pamięć kilkuset poleceń. Musisz umieć rozpoznać, **co zmieniasz i czy to już działa**, odróżnić shared candidate od private, sprawdzić wynik przed commit, świadomie użyć \`load\` oraz przewidzieć konsekwencje \`rollback\`.

W następnym rozdziale skonfigurujemy system od początku — konta, uwierzytelnianie, interfejsy management, NTP, syslog i rescue. Teraz masz narzędzia, aby robić to bezpieczniej.

**Blueprint — Rozdział 3: Junos User Interfaces — CLI, konfiguracja i J-Web [A], 3.1–3.10 oraz 3.11 [EXT].**  
**Stan labu:** PLANNED / not runtime validated.  
**Plik kanoniczny:** \`docs/chapters/JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md\`.

