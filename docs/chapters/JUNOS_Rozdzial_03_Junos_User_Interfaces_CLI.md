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

