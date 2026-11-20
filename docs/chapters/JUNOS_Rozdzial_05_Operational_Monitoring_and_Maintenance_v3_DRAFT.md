# Rozdział V. Operational Monitoring and Maintenance — jak rozpoznać problem, zebrać dowody i bezpiecznie utrzymać Junos

**Junos Enterprise Routing & Switching · JNCIA-Junos → JNCIS-ENT → JNCIP-ENT**  
**Blueprint:** 5.1–5.8 [A], 5.9 [LAB]; rozwinięcia diagnostyczne [S/P/EXT].  
**Odbiorca:** inżynier sieci z około rocznym doświadczeniem, znający podstawowe role routera, switcha, adresu IP, VLAN i bramy domyślnej. Nie zakładamy znajomości CCNA, Cisco CLI ani protokołów dynamicznych.  
**Topologia:** [DIAMOND-6R — sześć vSRX](../../labs/reference-diamond-6vsrx/README.md), rozwijana od pojedynczego R1 do pełnej sieci.  
**Status:** dokument dydaktyczny, **PLANNED / NOT RUNTIME VALIDATED**. Przykładowe wyniki są wyraźnie oznaczonymi ilustracjami, nie odczytami z EVE-NG.

## Zaczynamy od pytania: „Co dokładnie nie działa?”

W rozdziale IV uruchomiliśmy podstawowe usługi administracyjne Junos. Wiemy, po co potrzebne są konta, SSH, NTP, syslog, SNMP i kopia konfiguracji. W praktyce inżynier zwykle poznaje ich wartość dopiero podczas incydentu. Monitoring zgłasza przerwę w komunikacji, użytkownik nie może otworzyć aplikacji, a na ekranie routera pojawia się kilka tysięcy linii statystyk.

Pierwszym odruchem nie powinno być wpisanie długiej listy komend. Trzeba ustalić **co nie działa, dla kogo, od kiedy i w którym kierunku**. Jeśli R1 odpowiada na ping, to wiemy jedynie, że na dany test ICMP otrzymaliśmy odpowiedź. Nie wiemy jeszcze, czy R1 przekazuje ruch z R5 do R6, czy konkretny serwer jest osiągalny, ani czy aplikacja może nawiązać połączenie TCP.

Rozdział V nauczy Cię rozróżniać trzy źródła informacji, które często dają pozornie sprzeczne wyniki:

**Konfiguracja** mówi, jak urządzenie zostało zaprogramowane: jakie adresy, trasy, strefy i zasady powinny obowiązywać. **Stan operacyjny** mówi, co działa teraz: czy link jest aktywny, czy trasa została wybrana, czy proces odpowiada, czy licznik błędów rośnie. **Test usługi** sprawdza, czy właściwy użytkownik lub system potrafi naprawdę wykonać wymagane zadanie. Te trzy warstwy należy porównywać, a nie traktować zamiennie.

### Wspólna topologia i nasza zasada dydaktyczna

Nie budujemy nowych, niepowiązanych adresacji. Korzystamy ze stałego DIAMOND-6R:

~~~mermaid
flowchart LR
  R5["R5-vSRX / CE-A<br/>192.0.2.10/24"] ---|"LAN-A 192.0.2.0/24"| R1["R1-vSRX / EDGE-A<br/>192.0.2.1/24"]
  R1 ---|"10.12.0.0/30"| R2["R2-vSRX / CORE-1"]
  R1 ---|"10.13.0.0/30"| R3["R3-vSRX / CORE-2"]
  R2 ---|"10.24.0.0/30"| R4["R4-vSRX / EDGE-B<br/>198.51.100.1/24"]
  R3 ---|"10.34.0.0/30"| R4
  R4 ---|"LAN-B 198.51.100.0/24"| R6["R6-vSRX / CE-B<br/>198.51.100.10/24"]
~~~

*Rysunek 5.1. R5 i R6 są routerami vSRX, a nie komputerami VPCS. Wszystkie połączenia są niezależnymi segmentami L2. Pełną mapę portów znajdziesz w [referencji DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md).*

Na początku użyjemy tylko R1. Następnie dwóch lub trzech węzłów, by zobaczyć, jak symptomy mogą powstawać na różnych etapach. Dopiero w laboratorium zaawansowanym włączymy wszystkie sześć urządzeń i dwie możliwe drogi przez rdzeń.

---

## 5.1. Polecenia `show` — trzy różne pytania do tego samego routera [A]

### Co w ogóle oznacza „stan urządzenia”?

Załóżmy, że R1 ma połączenie do R2 na `ge-0/0/0`. W dokumentacji widnieje opis „UPLINK DO CORE”. Administrator mówi: „Widziałem ten interfejs w konfiguracji, więc link jest poprawny”. To za mało. Wpis konfiguracji nie potwierdza, że kabel lub wirtualny segment faktycznie łączy oba urządzenia.

Zacznij od poleceń, które **niczego nie zmieniają**.

**R1-vSRX — Operational Mode, pierwszy przegląd**

~~~junos
show version
show system uptime
show system alarms
show interfaces terse
show configuration interfaces ge-0/0/0 | display set
~~~

`show version` identyfikuje system i jego wydanie. `show system uptime` odpowiada, kiedy urządzenie uruchomiono i czy mogło niedawno zrestartować się bez wiedzy zespołu. `show system alarms` wyświetla alarmy systemowe, o ile platforma takie udostępnia. `show interfaces terse` to skrócony obraz interfejsów. Ostatnie polecenie pokazuje **zatwierdzoną konfigurację portu**, a nie jego liczniki ani rzeczywistą przepustowość.

### Physical i Logical — dlaczego widzę dwa stany?

W wyjściu `show interfaces terse` spotkasz kolumny odnoszące się do administracyjnego, fizycznego i logicznego stanu. Najpierw wyjaśnijmy pojęcia.

**Admin state** mówi, czy konfiguracja pozwala używać interfejsu. **Link state** informuje, czy urządzenie wykrywa połączenie na danej warstwie. **Logical interface** (np. `ge-0/0/0.0`) reprezentuje logiczną jednostkę, na której może działać IPv4 lub inna rodzina protokołów.

Poniższy fragment jest **ilustracją oczekiwanego formatu**, a nie wynikiem testu:

~~~text
Interface          Admin Link Proto    Local
ge-0/0/0           up    up
ge-0/0/0.0         up    up   inet     10.12.0.1/30
ge-0/0/1           up    up
ge-0/0/1.0         up    up   inet     10.13.0.1/30
~~~

Jeżeli na R1 oba porty pokazują `up/up`, wiemy, że są aktywne w rozumieniu danego sterownika/obrazu. **Nie wiemy**, czy są wolne od błędów, czy R2 jest osiągalny, czy routing do odległej sieci jest poprawny ani czy SRX zezwala na sesję tranzytową.

### Skąd router zna drogę do pakietu?

Przypomnijmy krótko rozdział II. **RIB (Routing Information Base)** to informacje o trasach znanych routerowi. **FIB (Forwarding Information Base)** to widok decyzji używanych do przekazywania pakietów. W Junos szczegółowa realizacja forwarding zależy od platformy i może obejmować sprzętowy lub wirtualny PFE.

Na R1 adres 10.12.0.1/30 daje sieć bezpośrednio połączoną 10.12.0.0/30. Ale trasa do 198.51.100.0/24, sieci R6, musi pochodzić z innego źródła — w pierwszym labie może to być trasa statyczna, w przyszłych rozdziałach protokół dynamiczny.

**R1-vSRX — Operational Mode, trasa do sieci zdalnej i decyzja forwarding**

~~~junos
show route table inet.0
show route 198.51.100.0/24 exact
show route forwarding-table destination 198.51.100.10
show arp no-resolve
~~~

Wyjście `show route` trzeba czytać uważnie. Router może przechowywać więcej niż jedną trasę do jednego prefiksu, ale tylko odpowiednia trasa staje się aktywna i może trafić do decyzji forwarding. **Destinations** to liczba różnych prefiksów docelowych, a **routes** to liczba wpisów tras — te liczby nie muszą być równe.

Znak aktywności trasy, jej next-hop i interfejs wyjściowy mają większą wartość diagnostyczną niż sama liczba tras. `show route forwarding-table` jest ważnym widokiem po stronie RE, lecz nie należy go mylić z niezależnym pomiarem, że pakiet naprawdę wyszedł z portu i dotarł do odbiorcy.

**R1-vSRX — Operational Mode, aktywne i ukryte informacje o trasach**

~~~junos
show route 198.51.100.0/24 exact extensive
show route hidden
~~~

Trasa **hidden** to trasa, której system nie wykorzystuje w zwykłym wyborze aktywnej drogi z określonego powodu. W rozdziałach o BGP i routing policy dowiesz się, jak rozpoznawać te powody. Na tym etapie wystarczy wiedzieć, że brak aktywnego wpisu nie musi oznaczać braku jakiejkolwiek informacji o tej sieci.

Uważaj też na `show route | count`: polecenie liczy **wiersze wyświetlonego tekstu**, nie liczbę prefiksów w sieci.

### Routing Engine też może mieć problem

Gdy router przestaje odpowiadać przez SSH, nie oznacza to automatycznie, że PFE nie przekazuje żadnego pakietu. W Junos rozdzieliliśmy control plane i forwarding plane. Proces `rpd` odpowiada za istotną część funkcji routingu, a `mgd` obsługuje m.in. konfigurację i zarządzanie. System może mieć problem z procesem administracyjnym, a nadal przekazywać już znane pakiety — albo odwrotnie.

**R1-vSRX — Operational Mode, obserwacja obciążenia i alarmów bez ingerencji**

~~~junos
show system processes extensive
show system uptime
show system alarms
show log messages | last 25
~~~

Jednorazowo wysokie CPU nie jest jeszcze diagnozą. Sprawdź, **który proces**, przez jak długi czas i czy pojawiły się zmiany protokołów, opóźnienia CLI lub rzeczywisty wpływ na usługi. Nie restartuj `rpd` dlatego, że liczba wygląda niepokojąco.

### Pierwsza minuta incydentu

Gdy otrzymasz zgłoszenie „sieć R5–R6 nie działa”, zbierz od razu trzy informacje: **który test nie przeszedł**, **jaki był stan portów i tras w tej chwili** oraz **czy niedawno wykonano zmianę**. Odczyt historii commit jest więc częścią diagnostyki, a nie tylko dokumentacji.

**R1-vSRX — Operational Mode, kontekst ostatnich zmian**

~~~junos
show system commit
show configuration interfaces | display set
show log messages | last 50
~~~

Nie wykonuj `rollback 1` na podstawie samego przypuszczenia „to pewnie po ostatniej zmianie”. Ustal, która zmiana mogła mieć związek z symptomem i jakie inne poprawki cofnięcie mogłoby usunąć.

---

## 5.2. Polecenia `monitor` — różnica między zdjęciem a obserwacją zmian [A]

### Dlaczego jeden pomiar często nie wystarcza?

Wyobraź sobie, że użytkownik mówi: „Co kilka minut rozłącza mi aplikację”. Sprawdzasz R1: port `ge-0/0/0` jest `up`. Ale po minucie przestaje działać i po chwili wraca. Jednorazowe `show interfaces terse` może całkowicie minąć się z tym zdarzeniem.

Polecenia `monitor` pomagają obserwować zmiany **w czasie**. Nie są jednak jednym narzędziem o uniwersalnym zastosowaniu: część pokazuje interfejsy, część logi, a część wybrane pakiety widoczne z określonego miejsca systemu.

### `monitor interface` — obserwacja konkretnego portu

**R1-vSRX — Operational Mode, interaktywny podgląd portu do R2**

~~~junos
monitor interface ge-0/0/0
~~~

W odpowiednim wydaniu polecenie może prezentować zmiany liczników, stan i inne informacje. Obsługa wyświetlania, klawiszy oraz sam format zależą od platformy. Niektóre tryby monitorowania Junos OS Evolved działają przez SSH, ale nie przez konsolę lub Telnet. Jeżeli polecenie nie działa w danej sesji, sprawdź dokumentację konkretnego release zamiast od razu podejrzewać uszkodzenie interfejsu.

**R1-vSRX — Operational Mode, alternatywny widok zbiorczy ruchu interfejsów**

~~~junos
monitor interface traffic
~~~

Ten widok może pokazywać zmiany w ruchu na wielu interfejsach. Nie zastępuje pełnej analizy błędów konkretnego portu. Jeśli chcesz ustalić, dlaczego jeden uplink ma problem, wróć do `show interfaces extensive` i liczników — monitorowanie nie jest diagnozą samą w sobie.

### `monitor start` — nowe wiadomości w logu, kiedy problem akurat występuje

W rozdziale IV konfiguracja syslog pozwalała rejestrować zdarzenia. Teraz możesz **śledzić nowe linie dopisywane do pliku**, zamiast co kilka sekund uruchamiać `show log messages`.

**R1-vSRX — Operational Mode, obserwacja nowych zdarzeń w pliku messages**

~~~junos
monitor start messages
~~~

Gdy test się zakończy, wyłącz podgląd:

**R1-vSRX — Operational Mode, zakończenie podglądu logów**

~~~junos
monitor stop messages
~~~

Ważna rzecz: `monitor start` nie włącza magicznie dodatkowego debugowania protokołów. Pokazuje dopisywane linie do istniejącego pliku. Jeśli potrzebujesz bardziej szczegółowych informacji, odpowiednie traceoptions trzeba zaprojektować oddzielnie, z limitem wielkości i planem usunięcia.

### `monitor traffic` — po co nam ostrzeżenie przed fałszywym wnioskiem?

Na komputerze z systemem Linux administrator może uruchomić `tcpdump` i zobaczyć pakiety przechodzące przez wybrany interfejs. W Junos polecenie `monitor traffic` także jest użyteczne do obserwowania pewnych pakietów, ale **nie oznacza automatycznie pełnego packet capture wszystkich pakietów tranzytowych przez PFE**.

Na wielu platformach ruch transit jest przetwarzany w forwarding plane, a `monitor traffic` obserwuje przede wszystkim ruch dostępny z punktu widzenia RE lub określonej ścieżki capture. Wynik zależy od sprzętu, wersji i rodzaju pakietów.

Na R1 możemy ograniczyć obserwację do ICMP przychodzącego na port od R5.

**R1-vSRX — Operational Mode, krótki podgląd ICMP w izolowanym labie**

~~~junos
monitor traffic interface ge-0/0/2 no-resolve matching "icmp" count 10
~~~

Jeżeli R5 potrafi pingować R1, monitor może pokazać pakiety skierowane do lokalnego adresu routera. Jeżeli R5 wysyła pakiety **przez** R1 do R6, brak tych pakietów w capture **nie dowodzi**, że forwarding jest zepsuty. Porównaj wynik z rzeczywistym pingiem, tabelą routingu, licznikami i `show security flow session`.

Na produkcji nie uruchamiaj szerokiego `monitor traffic` bez filtra na obciążonym urządzeniu. Nie rejestruj w ticketach treści mogących zawierać dane użytkowników.

### Wybór narzędzia do pytania

| Pytanie | Dobry pierwszy wybór | Czego sam wynik nie dowodzi |
|---|---|---|
| Czy port jest teraz up? | `show interfaces terse` | Braku błędów i działania usługi |
| Czy port flappuje? | `monitor interface`, logi | Gdzie dokładnie jest wada medium |
| Czy błędy przyrastają? | Dwa odczyty `show interfaces extensive` | Dlaczego pojawiają się błędy |
| Czy pojawił się nowy log? | `monitor start messages` | Że log jest kompletny i przyczyna znana |
| Czy router otrzymał pakiet do własnego IP? | Ograniczony capture + host-inbound checks | Pełnej obserwacji transit przez PFE |
| Czy SRX dopuścił sesję tranzytową? | `show security flow session`, policy | Że aplikacja odległa poprawnie odpowiedziała |

**Dalej:** poznaliśmy narzędzia do obserwacji. Teraz nauczymy się czytać konkretne liczniki i wyprowadzać z nich wnioski, zamiast szukać jednej liczby, która magicznie odpowie na pytanie „czy LAN jest sprawny?”.


---

## 5.3. Statystyki interfejsów — co naprawdę oznaczają błędy, drops i obciążenie? [A]

### Pierwsza zagadka: port jest `up/up`, a użytkownik nadal zgłasza problemy

Załóżmy, że uplink R1–R2 ma status `up/up`. Na jego końcu jest jednak wadliwy przewód albo wkładka optyczna pracująca na granicy dopuszczalnych parametrów. Połączenie może przez długi czas pozostawać aktywne, lecz część ramek zostanie uszkodzona i odrzucona.

Dlatego nie możemy zamykać diagnostyki na `show interfaces terse`. Potrzebujemy **liczników**, czyli wartości, które urządzenie zwiększa, gdy odbiera, wysyła, odrzuca lub błędnie interpretuje pakiety.

**R1-vSRX — Operational Mode, szczegółowy obraz portu w stronę R2**

~~~junos
show interfaces ge-0/0/0 extensive
~~~

Wynik zależy od platformy i typu interfejsu. Szukamy jednak podobnych kategorii informacji: stan portu, szybkość, MTU, liczniki pakietów i bajtów, błędy wejściowe oraz wyjściowe, odrzucenia i ewentualne informacje o kolejkach.

Nie próbuj na początku odczytać każdej linijki. Najpierw ustal, który licznik odpowiada symptomowi.

### Input errors, CRC i drops — trzy różne zjawiska

**Input errors** to kategoria błędów dotyczących odbieranych ramek lub pakietów. Na fizycznym Ethernet może obejmować problemy wykryte podczas przyjmowania ruchu. Jednym z ważnych rodzajów jest **CRC/FCS error**: ramka ma niepoprawną sumę kontrolną, więc odbiornik rozpoznaje, że dane dotarły uszkodzone. Taki błąd może wskazywać na problem fizyczny, ale do ustalenia przyczyny trzeba zbadać medium, moduły, negocjację i oba końce połączenia.

**Drops** oznaczają odrzucone pakiety, lecz nie zawsze z powodu uszkodzenia bitów. Router może odrzucać pakiety przez brak dostępnych buforów, congestion, ograniczenia kolejki lub mechanizmy filtrowania. W zależności od platformy i miejsca pomiaru konkretne liczniki pokazują różne przyczyny.

**Output drops** mogą rosnąć, gdy urządzenie musi wysłać więcej danych, niż może obsłużyć dany interfejs lub kolejka, ale nie wolno każdej takiej wartości automatycznie nazywać „saturacją uplinku”. Wymagane są także informacje o prędkości, natężeniu ruchu, kolejkach, QoS/CoS i czasie występowania.

| Co obserwujesz | Pierwsza hipoteza | Co sprawdzić dalej |
|---|---|---|
| Rosną CRC/FCS po stronie odbiorczej | Jakość sygnału lub medium | Port i moduł po obu stronach, czas narastania, DOM na sprzęcie fizycznym |
| Rosną input errors, bez jasnego typu | Problem odbioru | Szczegółowy rozkład błędów, konfiguracja i platforma |
| Rosną output drops przy dużym ruchu | Congestion lub kolejki | Ruch w czasie, przepustowość, CoS, bursty |
| Dużo pakietów, brak narastania błędów | Ruch może być poprawny | Test rzeczywistej usługi i drogi powrotnej |
| Zero błędów, aplikacja nadal nie działa | Problem wyżej niż badany licznik | Routing, security, DNS, usługa po stronie odbiorcy |

### Najważniejsze: licznik bez czasu nie mówi, czy problem trwa

Załóżmy, że R1 pokazuje 240 błędów CRC. Czy to dużo? Nie umiemy odpowiedzieć, dopóki nie wiemy, czy interfejs działał przez pięć minut, czy przez trzy lata. Wiele liczników ma charakter **skumulowany**: zlicza od momentu inicjalizacji, restartu lub ostatniego ich wyzerowania.

Zróbmy prosty przykład. O 12:00 licznik CRC wynosi 240. O 12:05 wynosi 340. W ciągu pięciu minut przybyło 100 błędnych ramek, czyli średnio **20 na minutę**. Nie musisz od razu znać procentowego BER, żeby zauważyć aktywny problem.

Oto prawidłowy sposób pracy:

**R1-vSRX — Operational Mode, pierwszy odczyt, zapisz godzinę**

~~~junos
show system uptime
show interfaces ge-0/0/0 extensive
~~~

Po kilku minutach, w trakcie reprezentatywnego ruchu, wykonaj drugi odczyt.

**R1-vSRX — Operational Mode, drugi odczyt tego samego portu**

~~~junos
show system uptime
show interfaces ge-0/0/0 extensive
~~~

Porównaj różnice licznika CRC, errors, drops i liczbę przesłanych pakietów. Dopiero wtedy opisz, czy licznik **przyrasta**. Zrzut z dwóch różnych interfejsów, pomiar po restarcie i pomiar wykonany już po wyzerowaniu statystyk nie stanowią spójnej pary.

Dobrą praktyką jest zachowanie również obciążenia portu: 100 błędnych ramek w kilkunastu pakietach to coś innego niż 100 błędnych ramek w miliardzie przesłanych pakietów.

### Czy wolno od razu wyzerować liczniki?

Istnieje polecenie `clear interfaces statistics`, ale to **modyfikacja stanu diagnostycznego**. Jeśli usuniesz historyczny punkt odniesienia, monitoring lub inny inżynier może stracić dowody narastającego problemu. Wspólna sesja może pokazywać inne sumy niż system NMS, a wybrane liczniki sprzętowe mogą zachowywać się specyficznie dla platformy.

Dlatego w tym podręczniku najpierw **robimy dwa odczyty i porównujemy je**. Zerowanie statystyk pozostaje operacją wymagającą uzasadnienia i zgody w odpowiednim procesie change/incident. Nie będzie częścią pierwszego laboratorium.

### A jeśli w logach jest zmiana stanu portu?

Sprawdź logi w tym samym przedziale czasu. Jeżeli interfejs co kilka minut zmienia stan `down/up`, mamy inne zjawisko niż sam powolny przyrost CRC. To nazywamy **flapping**, czyli powtarzające się przechodzenie między stanami.

**R1-vSRX — Operational Mode, korelacja z logiem i alarmami**

~~~junos
show log messages | match "ge-0/0/0|link|LINK"
show system alarms
show interfaces ge-0/0/0 detail
~~~

Zwróć uwagę na ograniczenia wyszukiwania. Samo `match` pokazuje tylko pasujące linie; może ukrywać kontekst. Jeżeli potrzebujesz ustalić kolejność, przejrzyj szerszy zakres logu i skoreluj timestamp z NTP.

### Światłowód: co oznaczają Tx, Rx i dBm?

W części fizycznych urządzeń Juniper można odczytać **DOM (Digital Optical Monitoring)**, czyli parametry wkładek optycznych. Nie jest to funkcja, której możemy wiarygodnie nauczyć się na wirtualnym porcie vSRX — nie ma tam rzeczywistego lasera ani fotodetektora.

Na fizycznym EX/QFX/MX/SRX z obsługiwanym modułem SFP/QSFP spotkasz między innymi:

- **Tx optical power** — moc światła wysyłana przez **lokalny** nadajnik.
- **Rx optical power** — moc światła odbierana przez **lokalny** odbiornik; przychodzi z nadajnika po drugiej stronie.
- Temperaturę i napięcie wkładki, a na wspieranym sprzęcie także inne parametry diagnostyczne.

Jednostką bywa **dBm**, czyli logarytmiczna miara mocy względem 1 mW. Liczby ujemne nie są błędem: na przykład `-10 dBm` oznacza mniejszą moc niż `0 dBm`. **Nie ma jednego „dobrego” poziomu Rx dla wszystkich wkładek.** Odpowiedni zakres zależy od modelu transceivera, odległości, typu światłowodu i progów producenta.

~~~mermaid
flowchart LR
  TXA["Switch A<br/>TX lokalny"] -->|"Światłowód A→B"| RXB["Switch B<br/>RX lokalny"]
  TXB["Switch B<br/>TX lokalny"] -->|"Światłowód B→A"| RXA["Switch A<br/>RX lokalny"]
~~~

*Rysunek 5.2. Rx na jednym urządzeniu mierzy sygnał pochodzący z Tx sąsiada. Dwa kierunki optyczne należy analizować osobno.*

Na wspieranym fizycznym sprzęcie sprawdzisz DOM na przykład tak:

**EX-ACCESS01 — Operational Mode, przykład wyłącznie na fizycznym sprzęcie ze wspieranym modułem**

~~~junos
show interfaces diagnostics optics ge-0/0/0
~~~

Nazwa `EX-ACCESS01` oznacza odrębny, przykładowy switch, **nie jeden z sześciu vSRX**. Tego polecenia nie traktujemy jako testu hardware na R1-vSRX. Jeżeli moduł nie obsługuje DOM, brak wartości nie dowodzi, że link jest wadliwy.

Jeśli Rx na A jest poniżej progu, sprawdź nadajnik B, włókno B→A, złącza i właściwą specyfikację optyki. Nie wymieniaj bez uzasadnienia obu transceiverów tylko dlatego, że odczyt jest liczbą ujemną.

### Czego mamy się nauczyć przed przejściem dalej?

Po sekcji 5.3 powinieneś umieć opisać problem w stylu: „R1 `ge-0/0/0` pozostaje `up/up`, ale w ciągu pięciu minut licznik input CRC wzrósł o 100; na R2 sprawdzimy odpowiadający kierunek oraz ewentualne błędy, a na fizycznym sprzęcie również DOM”. To znacznie wartościowsze niż „port wygląda na OK”.

**Źródła:** [show interfaces extensive](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-interfaces-extensive.html), [Understanding Packet Drops](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/using-show-commands-for-packet-drops.html), [Digital Optical Monitoring](https://www.juniper.net/documentation/us/en/software/junos/chassis/topics/concept/digital-optical-monitoring.html), [clear interfaces statistics](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/clear-interfaces-statistics.html).

---

## 5.4. Ping, traceroute i testy połączeń — co właściwie sprawdzasz? [A]

### Ping nie jest testem „całej sieci”

Załóżmy, że R5 zgłasza brak dostępu do R6. Wpisujesz na R1 `ping 10.12.0.2` i dostajesz odpowiedź. Czy problem rozwiązany? Nie. Sprawdziłeś tylko, że R1 potrafi wygenerować ICMP Echo Request do adresu R2 i otrzymać Echo Reply.

**Ping** jest programem wysyłającym pakiety **ICMP Echo Request** do wybranego adresu i oczekującym **Echo Reply**. Pozwala zbadać pewien rodzaj osiągalności IP i przybliżone opóźnienie odpowiedzi. Jednak nie dowodzi, że dowolna aplikacja TCP działa ani że pakiety użytkownika przechodzą przez firewall z taką samą polityką.

Najpierw nauczmy się świadomie dobierać **source** — adres, z którego wychodzi test.

**R1-vSRX — Operational Mode, test bezpośredniego sąsiada R2**

~~~junos
ping 10.12.0.2 source 10.12.0.1 count 5
~~~

To dobry test łącza R1–R2. Źródło `10.12.0.1` jest adresem R1 na tym samym /30, a cel `10.12.0.2` należy do R2.

Teraz porównaj z innym źródłem — adresem loopback R1 `10.255.255.1`.

**R1-vSRX — Operational Mode, ten sam cel, inne źródło IP**

~~~junos
ping 10.12.0.2 source 10.255.255.1 count 5
~~~

Taki test może się nie udać, mimo że pierwszy działa. Dlaczego? R2 potrzebuje poprawnej drogi **powrotnej do źródła 10.255.255.1**, a ponadto odpowiedź może zależeć od konfiguracji stref i filtrów. R2 zna 10.12.0.0/30 bezpośrednio, ale nie musi automatycznie znać loopbacku R1.

Ten przykład pokazuje fundamentalną zasadę diagnostyki: **zmiana source IP może zmienić wymagany routing powrotny i decyzje bezpieczeństwa**. Nie traktuj jej jako kosmetycznego dodatku do polecenia.

### Cztery testy, które odpowiadają na cztery pytania

Na dwóch bezpośrednio połączonych routerach R1–R2 możesz rozdzielić diagnostykę:

| Test | Pytanie, na które odpowiada |
|---|---|
| R1 → 10.12.0.2, source 10.12.0.1 | Czy bezpośredni sąsiad R2 odpowiada na lokalne ICMP? |
| R1 → 10.12.0.2, source 10.255.255.1 | Czy R2 ma drogę do loopbacku R1 i może odpowiedzieć? |
| R5 → 10.12.0.2, source 192.0.2.10 | Czy R1 przekazuje ICMP tranzytowo i czy R2 odpowiada? |
| R5 → 198.51.100.10, source 192.0.2.10 | Czy działa cała zaprojektowana droga i droga powrotna przez core? |

Pierwsze dwa są ruchem **locally originated** na R1. Trzeci jest tranzytem przez R1 i ruchem skierowanym do samego R2. Czwarty jest locally originated na R5, tranzytowy na R1–R4 i **host-bound** na R6, ponieważ R6 jest routerem, a nie PC. To nie jest dowód ruchu komputer-komputer za CE.

### A co z routing instance?

W rozdziale IV poznaliśmy oddzielną sieć zarządzania. Jeśli router ma specjalną instancję `mgmt_junos`, polecenie ping bez świadomego wyboru tablicy może użyć innej ścieżki niż sesja management.

**R1-vSRX — Operational Mode, przykład tylko przy istniejącej i sprawdzonej instancji mgmt_junos**

~~~junos
show route table mgmt_junos.inet.0
ping 203.0.113.30 routing-instance mgmt_junos count 3
~~~

Nie uruchamiaj tego jako dowodu działania OOB, jeśli wcześniej nie potwierdzono tej instancji i adresacji. Brak `mgmt_junos` nie jest sam w sobie awarią.

### Traceroute: skąd urządzenie wie, przez jakie routery idzie ruch?

**Traceroute** wykorzystuje pakiety z kontrolowaną wartością **TTL (Time To Live)**. TTL jest licznikiem ograniczającym liczbę routerów, przez które pakiet może przejść. Gdy na danym routerze wartość dojdzie do zera, urządzenie zwykle odrzuca pakiet i może wygenerować wiadomość ICMP Time Exceeded. Dzięki kolejnym próbom traceroute próbuje odkryć routery po drodze.

W naszym DIAMOND-6R spodziewana górna ścieżka R5→R6 to R5–R1–R2–R4–R6, o ile routing i filtry tak kierują ruch.

**R5-vSRX — Operational Mode, traceroute z routera CE-A**

~~~junos
traceroute 198.51.100.10 source 192.0.2.10
~~~

Gwiazdka `*` przy jednym hopie oznacza brak oczekiwanej odpowiedzi na konkretną próbę w określonym czasie. **Nie dowodzi automatycznie, że router nie przekazuje innych pakietów.** Router może ograniczać odpowiedzi ICMP, a security policy może blokować wiadomości kontrolne, choć ruch użytkownika przechodzi.

Na SRX trzeba również pamiętać, że traceroute może używać różnych typów pakietów zależnie od platformy i opcji. Jeżeli polityka dopuszcza tylko `junos-ping`, nie musi dopuszczać wszystkich sond traceroute. Wnioski opieraj na testach właściwej aplikacji i stanie tras, nie tylko na „ładnym” traceroute.

### Dlaczego telnet pojawia się w podręczniku, skoro jest niebezpieczny?

**Telnet** jako metoda **administracji** nie szyfruje poprawnie danych logowania i nie powinien być wykorzystywany do zarządzania nowym urządzeniem. Ale w niektórych wersjach Junos klient `telnet` bywa używany jako proste narzędzie sprawdzenia, czy da się zestawić połączenie TCP do określonego portu.

To są dwa różne zastosowania. Nie włączamy telnet server na R1. Jeśli sprawdzamy port aplikacyjny, używamy wyłącznie dozwolonego, testowego serwera i nie przesyłamy sekretów.

**R1-vSRX — Operational Mode, wariant sprawdzenia TCP do własnego kontrolowanego serwera testowego**

~~~junos
telnet 203.0.113.30 port 443
~~~

Ten przykład wymaga naprawdę uruchomionej usługi na `LAB-SERVICES`, osiągalnej z właściwej tablicy routingu. Zestawienie TCP nie oznacza jeszcze, że HTTPS prawidłowo obsługuje certyfikaty i żądania aplikacyjne. Jeśli potrzebujesz testu HTTP/TLS, dobierz odpowiednie narzędzie po stronie klienta.

### SSH to także użyteczny test, ale nie bez ograniczeń

Zamiast Telnet do administracji używaj **SSH**. Udana sesja SSH dowodzi znacznie więcej niż ping: poprawnej drogi dla TCP, dostępności usługi SSH, uzgodnienia połączenia oraz właściwej autoryzacji użytkownika.

Nie dowodzi jednak działania każdego innego portu i każdej polityki. To, że SSH z bastiona do R1 działa, nie znaczy, że sesja do aplikacji w LAN-B będzie dozwolona.

**R5-vSRX — Operational Mode, próba SSH do adresu R1 wyłącznie gdy SSH i host-inbound są dopuszczone w labie**

~~~junos
ssh labadmin@192.0.2.1
~~~

Wynik zależy od konfiguracji konta i strefy na R1. Jeśli SSH nie jest potrzebne na interfejsie danych, nie otwieraj go tylko po to, żeby ten konkretny przykład zakończył się sukcesem. Lepszy może być osobny klient management w izolowanej sieci OOB.

### Jak wygląda dobra sekwencja testów?

Nie zaczynaj od pingowania całego Internetu. Zdefiniuj konkretny cel, np. „R5 musi mieć ICMP do własnego adresu R6”. Sprawdź źródło i jego route, następnie sąsiedni router, potem środkową część drogi, a na końcu trasę powrotną i security. W przypadku TCP spróbuj rzeczywiście zestawić połączenie do odpowiedniego portu.

Nie zawsze trzeba wykonywać wszystkie testy. Najlepszy test to taki, którego wynik pozwala **odróżnić dwie konkretne hipotezy**.

**Źródła:** [ping](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/ping.html), [traceroute](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/traceroute.html), [monitor traffic](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html).


---

## 5.5. Instalowanie i aktualizowanie Junos OS — dlaczego upgrade zaczyna się od projektu, nie od komendy? [A → S/P]

### Co naprawdę aktualizujemy?

Urządzenie sieciowe składa się z wielu warstw: sprzętu lub maszyny wirtualnej, systemu operacyjnego, konfiguracji oraz danych działających procesów. Kiedy wykonujemy **upgrade Junos OS**, zmieniamy oprogramowanie urządzenia. Nie jest to to samo co zmiana jednej linii konfiguracji ani zapisanie nowej kopii `rescue`.

W firmie możesz otrzymać zadanie: „Na tych routerach jest podatność. Trzeba je zaktualizować”. To nie jest wystarczająca informacja, żeby wybrać plik i zacząć instalację.

Najpierw ustal:
- jakiego **dokładnie modelu lub wirtualnej platformy** dotyczy informacja;
- jaka **wersja Junos** jest obecnie zainstalowana;
- czy konkretna podatność i jej warunki dotyczą **tego modelu i konfiguracji**;
- która wersja usuwa problem i czy jest wspierana dla danej platformy;
- jakie są zależności i dopuszczalne **ścieżki aktualizacji** między wersjami;
- jaki będzie wpływ na użytkowników, sąsiedztwa routingu, klastry i system zarządzania;
- jak przywrócić urządzenie, jeśli nowy system nie uruchomi się prawidłowo.

Dlatego dobry proces aktualizacji jest bardziej podobny do planowania operacji technicznej niż do instalowania nowej aplikacji na laptopie.

### Co oznaczają release, maintenance i service release?

W oznaczeniach Junos spotkasz numery wersji oraz symbole literowe, np. `R` czy `S`. Reprezentują one określone rodzaje wydań i poprawki serwisowe. Nie musisz jeszcze znać każdej historycznej konwencji, ale musisz wiedzieć, że **„najnowsza dostępna” nie oznacza zawsze „najlepsza dla tego routera”**.

Producenci publikują zalecane wydania i release notes. Dla części środowisk istotne będzie wybranie wersji rekomendowanej przez JTAC i sprawdzenie wsparcia sprzętu, interfejsów oraz funkcji wykorzystywanych w sieci. Nie podam w podręczniku jednej „zalecanej wersji Junos na listopad 2026”, bo różne rodziny urządzeń mają różne cykle wsparcia i taki numer szybko mógłby stać się nieaktualny.

**Źródłami decyzji** powinny być oficjalne release notes, lista zalecanych wersji dla właściwej platformy, dokumentacja ścieżki upgrade i aktualne advisory. W sytuacji krytycznej należy zweryfikować też możliwość wsparcia przez producenta.

### Najpierw identyfikacja urządzenia

**R1-vSRX — Operational Mode, pre-check modelu i obecnej wersji**

~~~junos
show version
show chassis hardware
show system uptime
show system storage
show system alarms
show system commit
~~~

`show chassis hardware` na vSRX nie pokazuje fizycznych modułów w taki sam sposób jak na EX czy MX. Na urządzeniu fizycznym interesują Cię m.in. rzeczywiste podzespoły, wkładki, Routing Engines i dostępne miejsce w storage. Na wirtualnym vSRX sprawdź również wymagania obrazu co do hypervisora, zasobów RAM, CPU i dysku.

Sam numer Junos z `show version` nie jest kompletną informacją o zgodności package. Musisz znać również **rodzinę i model platformy**.

### Software package, konfiguracja i snapshot — trzy rzeczy, których nie wolno pomylić

**Software package** to obraz lub pakiet Junos przeznaczony dla określonej rodziny urządzeń. **Configuration** to zapis tego, co urządzenie ma robić. **Snapshot** to mechanizm tworzenia odpowiedniej kopii instalacji lub systemu, zależny od platformy.

Wyobraź sobie, że poprawnie cofnąłeś konfigurację do wczorajszej wersji. Czy system operacyjny wrócił do poprzedniego wydania? Nie. `rollback 1` dotyczy konfiguracji. Nie zmienia automatycznie plików systemu operacyjnego.

Podobnie snapshot wykonany na fizycznym urządzeniu lub w środowisku wirtualnym ma określoną rolę. Nie jest zawsze tym samym, co backup konfiguracji na osobnym serwerze. Na pewnych platformach wykonanie nowego **system snapshot** po aktualizacji może nadpisać poprzednią instalację w obszarze zapasowym. Wówczas utracisz prostą możliwość bootowania ze starego obrazu, choć sam snapshot pozornie wygląda na dodatkowe zabezpieczenie.

Dlatego przed poleceniem snapshot sprawdź model, układ partycji i faktyczny efekt opisywany w dokumentacji danej wersji Junos.

### Dlaczego nie wolno brać dowolnego pliku Junos ze strony producenta?

EX, QFX, MX, SRX i vSRX mają różne obrazy, ograniczenia oraz procedury aktualizacji. To nie są wymienne „instalatory Junipera”. Wiele urządzeń ma też specjalne wymagania dla instalacji systemu, Virtual Chassis, redundantnych RE, VMHost czy klastrów bezpieczeństwa.

Nie wykonuj upgrade na obrazie, którego źródło, nazwa, checksum i zgodność z platformą są nieznane. Plik pobrany z niezweryfikowanego forum albo z innego modelu może być niewłaściwy. W produkcji wymagamy **oficjalnego pochodzenia, sprawdzenia integralności i zgodności z planowaną ścieżką**.

### Pre-check: co zapisujemy, zanim coś zepsujemy?

Dla pojedynczego R1 w laboratorium wygląda to jeszcze prosto. W środowisku Enterprise router może być połączony z wieloma sąsiadami, obsługiwać OSPF/BGP, NAT, security policies, tunelowanie i dostęp administracyjny.

Przed upgrade zapisz:
1. Dokładną wersję i model; potwierdź, który image jest wspierany.
2. Uptime, alarmy, storage i stan systemu.
3. Stan wszystkich istotnych interfejsów i tras.
4. Kluczowe sąsiedztwa routing protocols, gdy są uruchomione.
5. Działanie usług zarządzania: SSH, AAA, NTP, syslog, SNMP, backup.
6. Istnienie aktualnego, **niezależnego** backupu konfiguracji i sprawdzony plan przywrócenia.
7. Dostęp do konsoli oraz ustalone okno serwisowe i warunki rollback.
8. Informację, kto sprawdza działanie aplikacji i kto podejmuje decyzję o wycofaniu.

Nie wystarczy zapisać „prechecks OK”. Warto zachować konkretne wyniki, aby później porównać ten sam stan.

**R1-vSRX — Operational Mode, zestaw read-only przed rozważaniem upgrade**

~~~junos
show version
show chassis hardware
show interfaces terse
show system alarms
show system uptime
show system storage
show system processes extensive
show route table inet.0
show security flow session summary
show system commit
~~~

Nie zawsze będziesz potrzebować wszystkich tych poleceń w małym labie. Na urządzeniach niebędących SRX polecenie security flow może nie istnieć. W środowisku z protokołami dynamicznymi dodasz odczyty ich sąsiedztw i tras. Chodzi o **reprezentatywny stan przed zmianą**, nie przypadkową liczbę komend w runbooku.

### Czy niewystarczająca przestrzeń dyskowa jest drobiazgiem?

Instalator potrzebuje odpowiedniej ilości miejsca, a na urządzeniu mogą zalegać stare obrazy, pliki logów lub core dumps. To jednak nie oznacza, że należy usuwać je natychmiast — mogą być potrzebne do diagnozy incydentu, dokumentacji lub powrotu do poprzedniego oprogramowania.

Sprawdź najpierw wykorzystanie storage i listę plików, które system proponuje do cleanup. Wspierane wydania Junos udostępniają **dry-run**, czyli próbę bez faktycznego usuwania.

**R1-vSRX — Operational Mode, bezpieczny przegląd storage**

~~~junos
show system storage
request system storage cleanup dry-run
~~~

`dry-run` nie jest zleceniem faktycznego usunięcia danych. Natomiast właściwe `request system storage cleanup` już zmienia zawartość urządzenia. Nie wykonuj go bez przeczytania listy proponowanych plików i ustalenia, czy mogą być potrzebne do rollback lub support case.

### Dlaczego `no-validate` nie jest wygodnym skrótem?

Normalny proces instalacji może sprawdzać zgodność konfiguracji i platformy. Opcja `no-validate` pomija część takiej weryfikacji. Jeśli instalacja zatrzymuje się z błędem, użycie `no-validate` bez zrozumienia przyczyny nie jest naprawą problemu.

Istnieją **udokumentowane wyjątki** dla określonych modeli i przejść między konkretnymi wersjami Junos. Oficjalna dokumentacja `request system software add` opisuje m.in. wymagania specjalnej składni dla wybranych SRX300-series podczas specyficznego upgrade. To dobry przykład, dlaczego nie wolno uczyć jednej „zawsze poprawnej” kombinacji parametrów. Wyjątek ma zastosowanie tylko do modelu i ścieżki, dla których producent go zaleca.

### Co robi `request system software add`?

To jedna z rodzin komend Junos używanych do instalacji pakietów. Sama nazwa polecenia nie przesądza jeszcze, czy upgrade wymaga osobnego reboot, jak działa walidacja i czy dany wariant jest właściwy dla urządzenia.

Nie podajemy tutaj wykonywalnego polecenia z wymyśloną nazwą pakietu jako „uniwersalnego MOP”. Plik do instalacji musi być konkretny, legalnie uzyskany, zweryfikowany i zgodny z platformą. Dla Junos OS Evolved oraz urządzeń z VMHost lub HA szczegóły mogą się znacznie różnić.

Na poziomie początkującym zrozum **kolejność**: wybór i walidacja pakietu → przygotowanie backupu → pre-check → staging → okno serwisowe → właściwa procedura instalacji → wymagane przełączenie/boot → post-check → decyzja o zamknięciu lub wycofaniu.

### Redundancja to nie gwarancja „zero outage”

W dużych sieciach spotkasz różne mechanizmy wysokiej dostępności:

**Redundant Routing Engines** oznaczają dwa elementy sterujące w pewnych platformach. **Virtual Chassis** łączy wybrane switche w logiczne urządzenie. **SRX chassis cluster** może wykorzystywać redundancję węzłów i obsługi ruchu. Nie są to jednak rozwiązania identyczne, a procedury upgrade nie zawsze są takie same.

Jeżeli urządzenie ma dwa RE, nie oznacza to, że aktualizacja dowolnym poleceniem będzie bezprzerwowa. Sprawdź wsparcie ISSU, kompatybilność wersji, zachowanie protokołów i wymaganą sekwencję. Podobnie w klastrze SRX potrzebujesz właściwego runbooka dla stanu klastra, sesji i przełączania węzłów.

**Nie uczymy się upgrade na wyłączaniu elementów HA w produkcyjnej sieci.** Na początku wystarczy rozpoznać, że sposób utrzymania zależy od architektury.

### Co znaczy „upgrade się udał”?

Urządzenie wróciło przez SSH i pokazuje oczekiwaną wersję Junos. Czy można zamknąć change? Jeszcze nie. Trzeba sprawdzić, czy interfejsy, trasy i usługi zarządzania powróciły do oczekiwanego stanu, a także czy użytkownicy mają dostęp do wymaganych aplikacji.

**R1-vSRX — Operational Mode, read-only post-check po autoryzowanym upgrade**

~~~junos
show version
show system uptime
show system alarms
show interfaces terse
show route table inet.0
show security flow session summary
show log messages | last 50
~~~

Porównaj je z rzeczywistym pre-check. Jeżeli po upgrade zmieniły się modele interfejsów, brak części połączeń albo logi wskazują awarie procesów, nie zamykaj change z samym „version OK”.

### Configuration rollback, software rollback i disaster recovery

To trzy różne pojęcia.

`rollback 1` w trybie konfiguracji ładuje wcześniejszą **konfigurację** do candidate. `request system software rollback`, jeśli wspierane na danej platformie, dotyczy próby powrotu do poprzedniego **oprogramowania**. Natomiast **disaster recovery** może wymagać odtworzenia urządzenia z niezależnego backupu, przywrócenia systemu i testów całej usługi.

Nie każde cofnięcie wersji Junos jest wspierane. Czasem konieczne są specjalne kroki i zachowanie odpowiedniego obrazu. Dlatego plan rollback musi zostać zaakceptowany **przed upgrade**, a nie napisany dopiero po nieudanym uruchomieniu.

### Nasz pierwszy MOP — bez instalacji ani restartu

**MOP (Method of Procedure)** to opis kontrolowanych kroków operacji, w tym weryfikacji i powrotu. Dla poziomu podstawowego przygotuj MOP *na papierze* do aktualizacji klona R1:

| Faza | Co ustalamy? | Dowód |
|---|---|---|
| Scope | Model, Junos release, powód zmiany | Inventory i advisory |
| Compliance | Obraz, checksum, release notes, ścieżka upgrade | Dokument producenta |
| Preparation | Backup, storage, console, snapshots | Pre-check i zewnętrzna kopia |
| Execution | Właściwa procedura dla platformy | Change-approved MOP |
| Validation | System, routing, usługi i test użytkownika | Post-check porównany z baseline |
| Recovery | Cofnięcie software lub odtworzenie | Osobny, wcześniej uzgodniony plan |
| Closure | Wersja, wyniki, anomalie, czas | Raport bez sekretów |

W laboratorium V **nie wykonujemy faktycznej instalacji Junos**. Jeżeli będziesz kiedyś chciał przećwiczyć upgrade na własnym klonie vSRX, potrzebny jest zgodny obraz i osobna instrukcja dla jego wydania.

**Źródła:** [Junos Install & Upgrade Overview](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/software-install-and-upgrade-overview.html), [request system software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html), [Storage Cleanup](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-storage-cleanup.html), [Snapshots](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/backing-up-install-using-snapshot.html), [Software rollback](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-rollback.html).

---

## 5.6. Uruchamianie, restart i bezpieczne wyłączanie urządzeń [A]

### Dlaczego restart jest działaniem, a nie diagnozą?

Czasami aplikacja nie działa, urządzenie zużywa dużo pamięci albo administrator stracił do niego SSH. Może pojawić się pomysł: „Zrestartujmy router, może pomoże”. Restart bywa potrzebny — na przykład po instalacji oprogramowania — ale **sam nie wyjaśnia przyczyny**.

Jeśli R1 przekazuje ruch dla R5, a R6 znajduje się po drugiej stronie core, to restart R1 może przerwać komunikację nie tylko z jednym użytkownikiem. Zależnie od topologii może odciąć cały ruch lokalizacji CE-A. Na switchu access podobna decyzja wpływa na użytkowników podłączonych do jego portów.

Zanim planujesz restart, ustal, jaki ruch przechodzi przez urządzenie, czy istnieje droga zapasowa, kto korzysta z sesji i ile czasu zajmie powrót systemu.

### Różnica między reboot, halt i odcięciem zasilania

**Reboot** oznacza kontrolowane ponowne uruchomienie systemu. **Halt** oznacza kontrolowane zatrzymanie systemu. Nie jest to to samo co natychmiastowe wyciągnięcie przewodu zasilającego.

Nagłe przerwanie zasilania może pozostawić dane w niespójnym stanie, przerwać operację zapisu lub spowodować utratę logów. W systemach z dwoma zasilaczami lub różnymi RE dodatkowo trzeba rozpoznać, **co faktycznie wyłączasz**.

Polecenia `request system reboot` i `request system halt` nie są poleceniami do próbnej diagnostyki. W tym rozdziale **nie umieszczamy ich w blokach przeznaczonych do skopiowania na produkcyjny router**, ponieważ wymagają uzgodnionego okna i szczegółowego MOP.

### Jak rozpoznać, czy restart był niedawno?

**R1-vSRX — Operational Mode, read-only historia i status przed planowaniem restartu**

~~~junos
show system uptime
show system reboot
show log messages | last 50
show system alarms
~~~

`show system reboot` może pomóc sprawdzić informacje o zaplanowanym restarcie na wspieranych platformach. `show system uptime` i logi pomagają zauważyć, czy urządzenie niedawno uruchamiało się ponownie.

Jeżeli uptime wynosi pięć minut, a użytkownik zgłasza przerwę od pięciu minut, jest to ważny trop. Ale nadal trzeba ustalić, **dlaczego** nastąpił restart i czy jego przyczyna została usunięta.

### A co z restartowaniem pojedynczego procesu?

Na niektórych urządzeniach można restartować określone procesy systemowe. Bywa to mniej inwazyjne niż reboot całej platformy, lecz nie jest neutralne. Restart procesu odpowiedzialnego za routing może powodować zmiany sąsiedztw, utratę aktualnych informacji i krótkotrwały wpływ na forwarding, zależnie od architektury i zabezpieczeń.

Nie wykonuj więc „restart routing” tylko dlatego, że nie widzisz trasy w `show route`. Najpierw ustal, skąd router powinien ją znać, jaki jest stan protokołu, czy nie działa filtr lub polityka i czy problem nie leży w warstwie bezpieczeństwa.

### Po restarcie testujemy również to, co nie jest w prompt

Jeżeli R1 wrócił do CLI, sprawdź **wersję systemu, alarmy, porty, trasy, sąsiedztwa i dostęp do rzeczywistych usług**. W środowisku z redundantnymi RE, Virtual Chassis czy SRX cluster trzeba dodatkowo potwierdzić stan wszystkich członków i zależności HA.

Wynik „SSH OK” ma znaczenie, ale nie zastępuje testu end-to-end z R5 do R6. To dokładnie ta sama zasada, której uczyliśmy się od początku rozdziału.

**Źródła:** [request system reboot](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-reboot.html), [request system halt](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-halt.html), [show system reboot](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-reboot.html).


---

## 5.7. Odzyskiwanie hasła root — kiedy procedura jest dostępna, a kiedy celowo zablokowana? [A]

### Najpierw odróżnij brak hasła od braku dostępu

Załóżmy, że nowo przejęty router R1 nie pozwala zalogować się jako `root`. Czy powinieneś od razu wejść do trybu odzyskiwania hasła?

Nie. Najpierw sprawdź, czy problem dotyczy rzeczywiście **hasła root**, czy może innej ścieżki logowania: konta `labadmin`, polityki SSH, centralnego AAA albo adresu management.

Jeśli SSH działa, lecz `root` jest blokowany, może to być poprawny wynik `set system services ssh root-login deny`. Konto root może nadal działać przez konsolę. Jeżeli nie możesz zalogować się przez SSH żadnym kontem, problemem może być również routing, security zone lub serwer uwierzytelniania.

**Root password recovery** to procedura pozwalająca uprawnionemu administratorowi przywrócić dostęp do własnego urządzenia, gdy nie zna hasła root. Zwykle wymaga odpowiedniego fizycznego lub konsolowego dostępu podczas uruchamiania systemu. Nie jest normalnym sposobem codziennej administracji.

### Dlaczego dostęp do konsoli jest tak ważny?

Przez SSH łączysz się z już działającym systemem, który wymaga uwierzytelnienia. Podczas odzyskiwania hasła trzeba zwykle skorzystać z odpowiedniego etapu **boot**, czyli uruchamiania systemu, zanim normalny mechanizm logowania zostanie w pełni uruchomiony. Dlatego konsola musi być dostępna niezależnie od SSH i samej konfiguracji IP.

Na fizycznym routerze może to wymagać obecności technika onsite, terminala szeregowego i zaplanowanej przerwy. Na własnym vSRX w EVE-NG należy mieć dostęp do jego konsoli i **uprawnienia do tej instancji VM**. Sama znajomość adresu IP routera nie wystarcza.

### Nie istnieje jedna procedura dla każdego Junos

W zależności od platformy i wersji spotkasz różne sposoby odzyskiwania. W klasycznym Junos z odpowiednim FreeBSD można trafić na procedurę uruchomienia w trybie pojedynczego użytkownika, np. z etapem `boot -s` i poleceniem `recovery`. W **Junos OS Evolved** dostępna procedura może korzystać z pozycji odzyskiwania w menu boot. Dodatkowo zabezpieczenia takie jak odpowiednie ustawienia konsoli, tryb FIPS czy polityki platformy mogą proces ograniczać lub blokować.

To oznacza, że sam tekst `boot -s` nie jest uniwersalną komendą do wykonania na każdym Juniperze. Jego znaczenie zależy od konkretnego boot loadera. Nie może też zastąpić formalnej autoryzacji do odzyskiwania.

### Jak zaplanować recovery na własnym vSRX?

W oddzielnym klonie szkoleniowym zacznij od odpowiedzi na pytania:

1. Czy to rzeczywiście właściwa maszyna i czy mam upoważnienie do recovery?
2. Czy znam dokładny obraz Junos, model oraz dokumentację tego wydania?
3. Czy mam sprawdzony dostęp konsolowy i mogę obserwować **cały boot**?
4. Czy mam kopię konfiguracji i snapshot lub inną ścieżkę odtworzenia urządzenia?
5. Czy restart maszyny jest dozwolony i nie odetnie innych laboratoriów?
6. Czy ustawienia bezpieczeństwa dopuszczają recovery i czy zespół ma właściwe dane ratunkowe?

Jeżeli odpowiedź na którekolwiek z kluczowych pytań brzmi „nie wiem”, nie rozpoczynaj operacji.

**R1-vSRX — Operational Mode, kontrola stanu na dostępnej autoryzowanej sesji przed planowaną procedurą**

~~~junos
show version
show configuration system root-authentication
show configuration system services ssh
show system uptime
show system commit
~~~

Odczyt `root-authentication` może zawierać informacje wrażliwe o konfiguracji uwierzytelniania, dlatego nie kopiuj jego pełnej treści do publicznego repozytorium. Nie pokaże także prawdziwego hasła w postaci umożliwiającej jego odczytanie.

### Co robisz po przywróceniu dostępu?

Jeżeli zgodna z platformą, autoryzowana procedura recovery zakończyła się sukcesem, skonfiguruj nowe hasło w bezpiecznym interaktywnym dialogu i zweryfikuj dostęp. Nie używaj jawnej wartości w `encrypted-password`, jeśli nie jest ona już poprawnym hashem w formacie obsługiwanym przez Junos.

**R1-vSRX — Configuration Mode [edit private], nowy root password wprowadzany interaktywnie po autoryzowanym recovery**

~~~junos
set system root-authentication plain-text-password
show | compare
~~~

Prawidłowa procedura zatwierdzenia i zakończenia recovery zależy od tego, czy jesteś już w normalnym CLI czy w specjalnym trybie ratunkowym, oraz od konkretnego release. Nie zatwierdzaj ślepo zmian w nieznanym kontekście.

Po odzyskaniu dostępu sprawdź **konta administracyjne, AAA, politykę SSH, logi i przyczynę utraty danych uwierzytelniających**. Jeśli organizacja nie ma kontrolowanego magazynu danych break-glass, problem może się powtórzyć na kolejnym routerze.

### Root recovery a bezpieczeństwo organizacji

Możliwość odzyskiwania root przez konsolę jest przydatna, ale oznacza również, że dostęp do sprzętu i konsoli należy fizycznie chronić. Wysokie wymagania bezpieczeństwa mogą celowo blokować pewne ścieżki recovery. W takim przypadku nie należy obchodzić zabezpieczeń — trzeba skorzystać z zatwierdzonego procesu organizacyjnego lub wsparcia producenta.

Nie przeprowadzamy rzeczywistego resetu hasła w głównym DIAMOND-6R. Jeśli chcesz ćwiczyć ten mechanizm, użyj **oddzielnego, odizolowanego klona R1**, który nie jest potrzebny do pozostałych labów. Status pozostaje NOT TESTED, dopóki nie wykonano prawidłowej procedury dla wskazanej wersji.

**Źródła:** [Recover a Root Password — Junos OS](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/recovering-root-password.html), [Recover a Root Password — Junos OS Evolved](https://www.juniper.net/documentation/us/en/software/junos/user-access-evo/user-access/topics/topic-map/recovering-root-password.html), [Root Authentication](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/user-access-root-password.html).

---

## 5.8. Jak prowadzić diagnostykę i prace utrzymaniowe — od objawu do dowodu naprawy [A → S/P]

### Nie zaczynaj od pytania „jakiej komendy użyć?”

Wracamy do R5 i R6. Załóżmy, że użytkownik zgłasza brak łączności między obiema stronami. Mamy sześć routerów, dwie możliwe drogi przez rdzeń i firewalling SRX na każdym z nich. Gdzie zacząć?

Od dokładnego określenia symptomów. **Nie działa ping z R5 do 198.51.100.10**, czy może nie działa jedna aplikacja TCP za R6? Czy problem pojawił się po zmianie konfiguracji? Czy dotyczy obu kierunków? Czy nadal działa bezpośrednie połączenie R5–R1?

Różne odpowiedzi prowadzą do różnych testów. Jeśli nie możesz pingować sąsiedniego routera na tym samym segmencie, analizowanie BGP na R4 jest przedwczesne. Jeśli bezpośredni ping działa, a ruch przez R1 nie, warto rozważyć **routing, security zones/policies i drogę powrotną**.

### Pierwsza rozmowa podczas incydentu

Dobrze postawione pytania to nie formalność. Poproś o:
- dokładne **źródło** i **cel** połączenia, a jeśli to aplikacja — także protokół i port;
- informację, kiedy problem wystąpił po raz pierwszy i czy jest stały czy sporadyczny;
- liczbę dotkniętych użytkowników lub lokalizacji;
- wynik **tego samego testu**, który wcześniej działał;
- wiadomość, czy w ostatnim czasie wykonywano change, restart albo prace fizyczne.

Jeśli masz ServiceNow lub inny system zgłoszeń, zapisz te dane. Nie wystarczy „network down”, bo taki wpis nie odróżnia awarii sieci danych od awarii zarządzania.

### Hipoteza to nie fakt

**Hipoteza** jest możliwym wyjaśnieniem objawu. Jeśli R5 nie pinguje R6, hipotez może być kilka: nieaktywny link R1–R2, brak trasy, złe rozwiązywanie ARP, niedopuszczona nowa sesja w security policy, brak trasy powrotnej albo brak odpowiedzi samego R6.

Każdą hipotezę można sprawdzić konkretną obserwacją.

| Hipoteza | Test rozstrzygający | Co nie wystarczy? |
|---|---|---|
| Link R1–R2 jest down | Status i logi na obu interfejsach | Sam opis portu w konfiguracji |
| Na R1 brakuje drogi do LAN-B | `show route` i forwarding | Ping do własnego adresu R1 |
| Next-hop nie rozwiązuje się na L2 | ARP oraz stan portu do sąsiada | Sama obecność static route |
| SRX nie dopuścił nowego flow | Security policy, session i counters | Ping z routera do bezpośredniego sąsiada |
| R6 nie zna powrotu do R5 | RIB/FIB i test z R6 | Tylko sprawdzenie R1 |
| Problem jest tylko w aplikacji TCP | Test konkretnego TCP/HTTPS z klienta | Ping ICMP do gateway |

**Nie zmieniaj jednocześnie trzech rzeczy tylko dlatego, że mogą być związane z awarią.** Jeśli poprawisz route, security policy i DNS naraz, nie będziesz wiedział, który mechanizm rzeczywiście spowodował poprawę.

### Odczyt porównujemy z wcześniej dobrym stanem

W rozdziale IV zbieraliśmy baseline po wdrożeniu. To właśnie teraz jest ważne. Jeżeli znasz prawidłowy interfejs, trasę, NTP i stan procesu sprzed awarii, łatwiej rozpoznać zmianę.

Przykład: R1 miał trasę do LAN-B przez R2, a teraz pokazuje next-hop do R3. To może być poprawne przełączenie zapasowe, ale musisz sprawdzić, **dlaczego** nastąpiła zmiana i czy R4 zna poprawną drogę powrotną.

Jedna niezgodność z baseline nie jest zawsze błędem. Dynamiczne protokoły routingu zmieniają tablice, gdy zmienia się topologia. Krytyczne jest ustalenie, czy aktualny stan spełnia wymagania usługi.

### Diagnostyka w warstwach — od sygnału do aplikacji

Przy każdym problemie unikaj mieszania zjawisk z różnych warstw.

Na **warstwie fizycznej** sprawdzasz link, błędy CRC, porty, transceivery i połączenia. Na **warstwie 2** — osiągalność sąsiada we wspólnym segmencie Ethernet, ARP i właściwą domenę przełączania, jeśli występuje. Na **warstwie 3** — adresy, tablice routingu, next-hop, FIB i drogę powrotną. Na poziomie **SRX security** — strefy, polityki, sesje i ewentualne filtry. Na poziomie **usługi** — właściwy protokół aplikacji, port, DNS, odpowiedzi serwera i uprawnienia.

To kolejność rozumowania, a nie sztywny nakaz zaczynania każdego zgłoszenia od sprawdzenia kabla. Jeśli monitoring od razu pokazuje odrzucenie sesji na firewallu, zacznij od tej hipotezy.

~~~mermaid
flowchart TD
  A["Zgłoszenie: test R5 → R6 nie działa"] --> B{"Czy wiemy jaki jest source/destination?"}
  B -->|nie| C["Ustal dokładny test i czas"]
  B -->|tak| D["Odczyt portów i tras na źródle"]
  D --> E{"Jest aktywna użyteczna droga?"}
  E -->|nie| F["RIB/next-hop/ARP, konfiguracja i zdarzenia"]
  E -->|tak| G["Zbadaj kolejne hop-y i security flow"]
  G --> H{"Czy odbiorca zna drogę powrotną?"}
  H -->|nie| I["Napraw świadomie return path"]
  H -->|tak| J["Test właściwej aplikacji i wynik post-check"]
~~~

*Rysunek 5.3. Schemat myślowy diagnostyki, nie literalny pipeline pakietu w każdym modelu Junos.*

### Jak zebrać dane na początku P1?

W przypadku dużej awarii potrzebujesz zestawu niewielkich, wiarygodnych odczytów i czasu ich wykonania.

**R1-vSRX — Operational Mode, pre-check bez zmiany stanu**

~~~junos
show system uptime
show system alarms
show interfaces terse
show route table inet.0
show log messages | last 50
show system commit
~~~

Na R4 i innych hopach powtarzasz odczyty **z odpowiedniego urządzenia**, ale nie wpisujesz ich wszystkich w jednym broadcast. W dokumentacji zawsze zapisuj, z którego routera pochodzi wynik.

Następnie porównaj z prawdziwym testem źródło→cel. Nie wykonuj przypadkowego `clear`, restartu procesu ani globalnego rollback, dopóki nie masz dowodów, że naprawi to konkretną przyczynę.

### Pre-check, post-check i rollback: trzy elementy jednego change

W poprzednich rozdziałach poznaliśmy `commit confirmed`. To zabezpieczenie jest użyteczne również podczas naprawy. Ale trzeba je rozumieć dokładnie: `commit check` wykonany **po** `commit confirmed` może potwierdzić warunkową zmianę, dlatego nie używamy go jako testu w czasie trwania timera.

Poprawny przebieg bezpiecznej zmiany w labie jest następujący:

1. Sprawdź stan **przed** zmianą, zapisz wyniki i znane objawy.
2. Pracuj na właściwym routerze i w odpowiedniej candidate.
3. Zobacz `show | compare` i wykonaj `commit check` **przed aktywacją**.
4. Wykonaj `commit confirmed` z czasem wystarczającym na konkretne testy.
5. Podczas oczekiwania wykonaj **read-only post-check i test usługi**, najlepiej z właściwego źródła.
6. Jeśli wynik jest dobry, świadomie zatwierdź `commit`. Jeśli zły — nie potwierdzaj; wykorzystaj zaplanowany rollback.
7. Porównaj wynik z baseline, odnotuj wpływ, nowe ryzyka i to, co trzeba jeszcze sprawdzić.

Jeśli problem występuje na kilku urządzeniach, nie uruchamiaj sześciu timerów równocześnie tylko dlatego, że w topologii są sześć vSRX. Każda zmiana powinna mieć własny cel i kontrolę skutku.

### Co napisać w worknotes?

Dobre worknotes nie są zbiorem wszystkich wklejonych outputów. Powinny zawierać:
- od kogo i kiedy pochodził symptom;
- jaki source/destination, protokół i kierunek sprawdzono;
- jaki był stan interfejsów, tras i sesji przed zmianą;
- jaka hipoteza została potwierdzona lub odrzucona i na podstawie czego;
- co dokładnie zmieniono oraz jak zabezpieczono rollback;
- jaki test wykazał poprawę i czy usługa działa z perspektywy użytkownika;
- co pozostało nierozwiązane lub wymaga działań innego zespołu.

Nie pisz „root cause: firewall” tylko dlatego, że na SRX pojawia się ruch. Jeśli nie udowodniono konkretnej przyczyny, poprawne jest „suspected” albo „further verification required”.

### Proaktywne utrzymanie: po co, skoro jeszcze nie ma awarii?

Monitoring, właściwe progi i przeglądy pozwalają wykryć narastające problemy, zanim użytkownik je zgłosi. Na fizycznym switchu może to być wzrost CRC albo pogarszający się poziom Rx. Na routerze — spadek miejsca na dysku, częste alarmy procesów, niestabilne sąsiedztwa albo nieskuteczne backupy.

Nie sprowadzaj proaktywnego monitoringu do „wszystkie hosty odpowiadają na ping”. Sprawdza on również **jakość, trendy i możliwość odzyskania usługi**.

---


---

## 5.9. Laboratorium diagnostyczne — od jednego portu do dwóch dróg DIAMOND-6R [LAB]

Naszym celem nie jest pokazanie największej możliwej konfiguracji Junos. Chcemy nauczyć się **przewidywać objaw, znaleźć jego miejsce, przywrócić poprawny stan i udowodnić naprawę**. Dlatego laboratorium ma trzy poziomy i każdą usterkę wprowadzamy osobno.

**Status wszystkich ćwiczeń:** **PLANNED / NOT RUNTIME VALIDATED**. Komendy są propozycją przeznaczoną do wykonania na **własnych, odseparowanych instancjach vSRX**. Nie przeprowadzono tych testów na rzeczywistym EVE-NG. Wszystkie opisane wyniki to **EXPECTED**, a nie obserwacje z routera.

### Zasady wspólne dla wszystkich poziomów

Najpierw upewnij się, że pracujesz na **nowej kopii laboratorium**, bez wpływu na inne projekty, sieć firmową ani środowisko produkcyjne. Posiadaj niezależną konsolę każdego vSRX. W EVE-NG sprawdź mapowanie NIC; oznaczenie `ge-0/0/0` na diagramie jest projektem, a nie gwarancją numeru karty QEMU.

Konfiguracje w tej sekcji są przeznaczone dla **świeżego labu**, w którym nie istnieją już kolidujące strefy, polityki, trasy i adresy. Jeśli rozwijasz gotowy DIAMOND-6R z wcześniejszego rozdziału, najpierw porównaj go z bazą. **Nie wklejaj dwóch konkurencyjnych konfiguracji na ten sam interfejs.** Nie wykonuj `load override`, `zeroize` ani globalnego `delete security`, żeby „dopasować” istniejące urządzenie do przykładu.

**Przed pierwszym commitem:** świeży vSRX musi mieć ważną bazową konfigurację dostępu i `system root-authentication` (rozdział IV). Jeżeli klon tego nie ma, najpierw przeprowadź bezpieczny onboarding — nie próbuj uruchamiać labu diagnostycznego na koncie z nieznanymi uprawnieniami. Od Junos OS 24.2R1 na wspieranych platformach komentarz do commitu może być wymagany; poniższe przykłady używają więc jawnego `comment` zarówno przy `commit confirmed`, jak i przy końcowym `commit`. Przy innym release sprawdź `commit ?` i lokalną politykę commit. Nie dodawaj komentarza do `commit check`. [Dokumentacja Juniper: commit and comment](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html).

Każde zadanie ma następujący porządek:

- **Baseline:** przed zmianą zapisujemy stan interfejsów, tras, ARP, security i testu ICMP.
- **Hypothesis:** przewidujemy, jaki będzie skutek jednej konkretnej usterki.
- **Fault injection:** wprowadzamy jedną zmianę przez `commit confirmed` po wcześniejszym `show | compare` i `commit check`.
- **Observation:** mierzymy skutki z Operational Mode, bez dodatkowego `commit check` podczas timera.
- **Recovery:** nie potwierdzamy błędnej zmiany; po wycofaniu sprawdzamy powrót do baseline.
- **Evidence:** zapisujemy rzeczywisty output i różnicę między przewidywaniem a obserwacją.

### Poziom 1 — tylko R1: naucz się odczytywać urządzenie przed jego konfiguracją

Pierwsze ćwiczenie nie wymaga ani OSPF, ani BGP, ani nawet działającego linku do R2. Potrzebujesz tylko konsoli R1 i urządzenia uruchomionego na odpowiednim obrazie vSRX.

#### Etap A. Rozpoznaj, co masz przed sobą

**R1-vSRX — Operational Mode, odczyt stanu pojedynczego routera**

~~~junos
show version
show interfaces terse
show system uptime
show system alarms
show system processes extensive
show system commit
~~~

Zapisz model, wersję Junos, nazwy interfejsów, uptime i ewentualne alarmy. Nie uznawaj, że interfejs `down` jest uszkodzony, jeśli w EVE-NG nie podłączyłeś go do żadnego segmentu.

#### Etap B. Wybierz jeden interfejs i zrób dwa pomiary

Załóżmy, że `ge-0/0/0` jest rzeczywiście mapowanym portem R1 w stronę przyszłego R2. Sprawdź jego pełne liczniki.

**R1-vSRX — Operational Mode, pierwszy odczyt portu**

~~~junos
show interfaces ge-0/0/0 terse
show interfaces ge-0/0/0 extensive
~~~

Po określonym odstępie (np. pięciu minutach) powtórz odczyt. Porównaj liczniki błędów, drops i pakietów.

**R1-vSRX — Operational Mode, drugi pomiar tego samego portu**

~~~junos
show interfaces ge-0/0/0 extensive
~~~

Jeśli nic nie jest podłączone, nie oczekuj ruchu ani przyrostu liczników. **Nie fałszuj wyniku** przez wpisanie, że „wykryto CRC”, kiedy vSRX nie zgłasza takich błędów. Celem jest nauczenie się metody, nie wygenerowanie alarmu na siłę.

#### Etap C. Zmień tylko opis i przećwicz kontrolę candidate

Przypomnijmy rozdział III: istnieje konfiguracja aktywna i robocza. Zmienimy opis portu i pozwolimy, by zmiana została automatycznie wycofana.

**R1-vSRX — Operational Mode, otwarcie prywatnej konfiguracji**

~~~junos
configure private
~~~

**R1-vSRX — Configuration Mode [edit private], bezpieczna zmiana testowa**

~~~junos
set interfaces ge-0/0/0 description "CH05-TEST-TEMPORARY"
show | compare
commit check
commit confirmed 2 comment "CH05 controlled lab change"
~~~

W osobnej sesji sprawdź, czy opis się pojawił i czy historia commit wskazuje warunkowe zatwierdzenie.

**R1-vSRX — Operational Mode, osobna sesja, odczyt podczas timera**

~~~junos
show configuration interfaces ge-0/0/0 | display set
show system commit
~~~

Nie wykonuj zwykłego `commit` ani `commit check`. Gdy timer wygaśnie, sprawdź, czy poprzedni opis powrócił.

**R1-vSRX — Operational Mode, odczyt po oczekiwanym rollback**

~~~junos
show configuration interfaces ge-0/0/0 | display set
show system commit
~~~

**EXPECTED:** konfiguracja działała warunkowo, po czym wróciła do poprzedniego stanu. Jeżeli nie wróciła, sprawdź, czy inna sesja nie potwierdziła commitu. Nie odnotowuj sukcesu bez rzeczywistego porównania.

**Poziom 1 zaliczony**, gdy potrafisz samodzielnie powiedzieć, jaki jest obecny system i stan portu, czy licznik przyrasta oraz czym różni się kandydacka zmiana od zmiany już aktywnej.

---

### Poziom 2 — R5–R1–R2: pierwszy tranzyt, trzy różne miejsca awarii

Teraz wykorzystamy trzy routery z DIAMOND-6R, ale włączymy jedynie dwa segmenty: **LAN-A 192.0.2.0/24** między R5 i R1 oraz **10.12.0.0/30** między R1 i R2. Nie ma jeszcze R3 ani R4.

| Router | Interfejs | Adres |
|---|---|---|
| R5 | `ge-0/0/0.0` | 192.0.2.10/24 |
| R1 | `ge-0/0/2.0` | 192.0.2.1/24 |
| R1 | `ge-0/0/0.0` | 10.12.0.1/30 |
| R2 | `ge-0/0/0.0` | 10.12.0.2/30 |

Zanim zaczniemy wpisywać trasy, zrozummy, jaki pakiet ma przejść. R5 wyśle ICMP do **10.12.0.2**. Na R5 pakiet będzie **locally originated**. Na R1 będzie **transit**. Na R2 będzie **host-bound**, ponieważ 10.12.0.2 jest własnym adresem R2.

Dla połączenia przez R1 potrzebujemy:
- trasy na R5 do 10.12.0.0/30 przez 192.0.2.1;
- bezpośredniego łącza R1–R2;
- drogi powrotnej na R2 do 192.0.2.0/24 przez 10.12.0.1;
- na SRX odpowiedniej obsługi host-inbound na R2 oraz security policy dopuszczającej nowe ICMP przechodzące między strefami na R1.

#### Etap A. Konfiguracja minimalnego baseline — najpierw R1

Jeżeli korzystasz z gotowej, zgodnej konfiguracji z rozdziału II, nie musisz jej nadpisywać. W świeżym labie przygotuj na R1 dwa porty i strefy.

**R1-vSRX — Operational Mode, wejście do konfiguracji**

~~~junos
configure private
~~~

**R1-vSRX — Configuration Mode [edit private], połączenie do R5 i R2**

~~~junos
set system host-name R1-vSRX
set interfaces ge-0/0/2 unit 0 family inet address 192.0.2.1/24
set interfaces ge-0/0/0 unit 0 family inet address 10.12.0.1/30
set security zones security-zone CE interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security policies from-zone CE to-zone CORE policy CH05-ALLOW-ICMP match source-address any
set security policies from-zone CE to-zone CORE policy CH05-ALLOW-ICMP match destination-address any
set security policies from-zone CE to-zone CORE policy CH05-ALLOW-ICMP match application junos-ping
set security policies from-zone CE to-zone CORE policy CH05-ALLOW-ICMP then permit
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

Co robią te linie? Pierwsze nadają adresy IP. Następne przypisują logiczne interfejsy do **security zones**. Ostatnie cztery linie tworzą jedną policy, która pozwala na **nowe sesje ICMP w kierunku CE→CORE**. Bez niej routing mógłby wyglądać poprawnie, ale firewall mógłby odrzucać tranzyt.

**R1-vSRX — Operational Mode, osobna sesja, lokalna kontrola po aktywacji**

~~~junos
show interfaces terse
show route 192.0.2.0/24 exact
show route 10.12.0.0/30 exact
show security zones
show security policies from-zone CE to-zone CORE
~~~

Jeżeli adresy, interfejsy i policy są zgodne z planem, **potwierdź tylko ten commit na R1**, zanim zaczniesz następne urządzenie.

**R1-vSRX — Configuration Mode [edit private], po PASS**

~~~junos
commit comment "CH05 post-check verified"
~~~

#### Etap B. Konfiguracja R2 — odbiorca pingu i droga powrotna

**R2-vSRX — Operational Mode, otwarcie własnej konfiguracji**

~~~junos
configure private
~~~

**R2-vSRX — Configuration Mode [edit private], adres i powrót do R5**

~~~junos
set system host-name R2-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 10.12.0.2/30
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set routing-options static route 192.0.2.0/24 next-hop 10.12.0.1
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

R2 zna bezpośrednio 10.12.0.0/30, ale **nie zna automatycznie 192.0.2.0/24**, ponieważ to sieć za R1. Dlatego dodajemy trasę powrotną. Jeśli jej zabraknie, pakiet ICMP może dotrzeć do R2, lecz odpowiedź może nie mieć drogi do R5.

**R2-vSRX — Operational Mode, osobna sesja, test połączenia z R1**

~~~junos
show interfaces terse
show route 192.0.2.0/24 exact
ping 10.12.0.1 source 10.12.0.2 count 3
~~~

Jeśli PASS, potwierdź konfigurację R2.

**R2-vSRX — Configuration Mode [edit private], po PASS**

~~~junos
commit comment "CH05 post-check verified"
~~~

#### Etap C. Konfiguracja R5 — nadawca testu

**R5-vSRX — Operational Mode, wejście do konfiguracji CE-A**

~~~junos
configure private
~~~

**R5-vSRX — Configuration Mode [edit private], adres i trasa do R2**

~~~junos
set system host-name R5-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 192.0.2.10/24
set security zones security-zone CE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set routing-options static route 10.12.0.0/30 next-hop 192.0.2.1
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

Nie potrzebujemy na razie BGP ani OSPF. Chcemy po prostu, aby R5 wiedział, że sieć 10.12.0.0/30 leży za R1. R1 jest bezpośrednio osiągalnym next-hopem 192.0.2.1.

**R5-vSRX — Operational Mode, osobna sesja, pierwszy end-to-end test**

~~~junos
ping 192.0.2.1 source 192.0.2.10 count 3
ping 10.12.0.2 source 192.0.2.10 count 5
show route 10.12.0.0/30 exact
~~~

**R1-vSRX — Operational Mode, niezależna obserwacja tranzytu**

~~~junos
show route 10.12.0.0/30 exact
show security policies from-zone CE to-zone CORE
show security flow session source-prefix 192.0.2.10
~~~

**EXPECTED:** po prawidłowej konfiguracji i dostępnych portach ping R5→R1 i ping R5→R2 mogą działać. Uwaga: sesja ICMP na SRX jest krótkotrwała, więc nie zawsze zobaczysz ją w `show security flow session` po zakończeniu kilku pingów. W razie potrzeby wykonaj ponowny, kontrolowany test podczas obserwacji.

Potwierdź R5 dopiero po rzeczywistym PASS. Jeśli nie działa, diagnozuj etapami: ARP R5→R1, Direct na R1, security CE→CORE, host-inbound R2 oraz drogę powrotną na R2. Nie „naprawiaj” wszystkiego naraz.

**R5-vSRX — Configuration Mode [edit private], po PASS**

~~~junos
commit comment "CH05 post-check verified"
~~~

### Trzy kontrolowane awarie, każda z osobną przyczyną

Teraz potrafimy wykonać test, który wcześniej działał. Możemy więc świadomie wprowadzić błąd i sprawdzić, czy potrafimy **przewidzieć objaw, zanim go zobaczymy**.

#### Awaria A — łącze administracyjnie wyłączone

Zdezaktywujemy na R1 port prowadzący do R2. Nie wyciągamy „wirtualnego kabla” w EVE, tylko stosujemy zmianę w konfiguracji. Nie ruszamy portu LAN-A do R5.

**R1-vSRX — Configuration Mode [edit private], fault A tylko na odizolowanym R1**

~~~junos
set interfaces ge-0/0/0 disable
show | compare
commit check
commit confirmed 2 comment "CH05 controlled lab change"
~~~

**R5-vSRX — Operational Mode, obserwacja dwóch różnych testów**

~~~junos
ping 192.0.2.1 source 192.0.2.10 count 3
ping 10.12.0.2 source 192.0.2.10 count 3
~~~

**R1-vSRX — Operational Mode, odczyt podczas warunkowej awarii**

~~~junos
show interfaces ge-0/0/0 terse
show route 10.12.0.0/30 exact
show system commit
~~~

**EXPECTED:** ping R5→R1 może nadal działać, ponieważ LAN-A nie został wyłączony. Ping do R2 może przestać działać, a na R1 zniknie użyteczna bezpośrednia droga przez wyłączony interfejs. Zwróć uwagę, jak jedna zmiana portu wpływa jednocześnie na stan linku i routing.

**Recovery:** nie potwierdzaj błędnego commitu. Po timerze sprawdź, czy `ge-0/0/0` wrócił do stanu bazowego i czy ponowny ping R5→R2 działa. Nie uruchamiaj `commit check` podczas oczekującego `commit confirmed`.

#### Awaria B — linki działają, ale nadawca nie zna drogi

Tym razem nie zmieniamy żadnego portu. Usuniemy tylko statyczną trasę na R5 do sieci R2.

**R5-vSRX — Configuration Mode [edit private], fault B bez zmiany interfejsu**

~~~junos
delete routing-options static route 10.12.0.0/30
show | compare
commit check
commit confirmed 2 comment "CH05 controlled lab change"
~~~

**R5-vSRX — Operational Mode, trasa nadawcy i wynik testów**

~~~junos
show interfaces ge-0/0/0 terse
show route 10.12.0.0/30 exact
ping 192.0.2.1 source 192.0.2.10 count 3
ping 10.12.0.2 source 192.0.2.10 count 3
~~~

**EXPECTED:** jeśli R5 nie ma innej trasy obejmującej 10.12.0.2, ping do R2 przestanie działać, chociaż port LAN-A i ping do R1 pozostaną poprawne. **Sprawdź założenie o braku alternatywnego/default route!** Jeśli istnieje inna użyteczna droga, wynik może być odmienny — i właśnie to trzeba wyjaśnić.

**Recovery:** po automatycznym rollback ponownie sprawdź trasę i ping. Tym razem naprawy szukaliśmy w **RIB nadawcy**, a nie w fizycznym łączu.

#### Awaria C — routing jest dobry, ale SRX nie dopuszcza nowej sesji

Na R1 mamy policy `CH05-ALLOW-ICMP` dopuszczającą ICMP CE→CORE. Gdy ją wyłączymy, routing statyczny i interfejsy pozostaną niezmienione.

**R1-vSRX — Configuration Mode [edit private], fault C na security policy**

~~~junos
deactivate security policies from-zone CE to-zone CORE policy CH05-ALLOW-ICMP
show | compare
commit check
commit confirmed 2 comment "CH05 controlled lab change"
~~~

**R1-vSRX — Operational Mode, kontrola podczas awarii**

~~~junos
show route 10.12.0.0/30 exact
show security policies from-zone CE to-zone CORE
show security flow session
show system commit
~~~

**R5-vSRX — Operational Mode, nowy test po dezaktywacji policy**

~~~junos
ping 192.0.2.1 source 192.0.2.10 count 3
ping 10.12.0.2 source 192.0.2.10 count 3
~~~

**EXPECTED:** host-bound ping do R1 może pozostać dostępny, a nowa próba tranzytowa R5→R2 może zostać zablokowana. Uważaj jednak na istniejące sesje SRX: **już ustanowione flow może nadal działać po zmianie policy**, zależnie od zachowania session table. Aby uczciwie przetestować regułę, użyj nowego flow lub sprawdź, czy poprzednia sesja wygasła. Nie wykonuj globalnego `clear security flow session all` w środowisku współdzielonym.

**Recovery:** po automatycznym rollback odczekaj, aż właściwa policy będzie znów aktywna, i uruchom nowy ping. Nie zmieniaj tras ani interfejsów — one nie były przyczyną.

#### Uwaga dodatkowa: błąd składni to inna kategoria niż błędna logika

Wyobraź sobie, że administrator próbuje przypisać R1 adres `192.0.2.1/33`. Prefiks IPv4 nie może mieć więcej niż 32 bity, więc parser lub walidacja powinny taki wpis odrzucić. **Nie trzeba wykonywać tego jako aktywnego fault**. Wystarczy rozumieć, że to błąd składni lub zakresu, podczas gdy błędna security policy może przejść `commit check`, a mimo to zablokować prawidłowy ruch.

**Podsumowanie poziomu 2:** po każdym fault odpowiedz: *który element zmieniliśmy, które testy powinny nadal działać, które powinny zawieść, co dowodzi przyczyny i co sprawdzamy po powrocie?*


---

### Poziom 3 — pełny DIAMOND-6R: awaria górnej drogi, niejednoznaczny failover i analiza całego ruchu

Teraz uruchomimy wszystkie sześć vSRX. Nie wprowadzamy nowej topologii: nadal są tylko dwie drogi przez rdzeń, **górna R1–R2–R4** i **dolna R1–R3–R4**. R5 i R6 to routery CE, a nie hosty VPCS.

**Cel dydaktyczny:** przeanalizować nie tylko jeden port, lecz również zależności między lokalnym wyborem trasy, forwardingiem kolejnych hopów, polityką SRX oraz ruchem powrotnym.

W podstawowym wariancie tego rozdziału wykorzystamy **adresy interfejsów LAN-A i LAN-B**. Referencyjne adresy loopback `10.255.255.1/32` do `10.255.255.6/32` i dodatkowe prefiksy `172.16.50.0/24`, `172.16.51.0/24`, `172.16.60.0/24`, `172.16.61.0/24` pozostają częścią projektu DIAMOND-6R, ale nie są potrzebne do tego pierwszego eksperymentu. Nie zmieniamy ich ani nie ogłaszamy BGP. Dzięki temu koncentrujemy się na diagnostyce, a nie na pięciu nowych mechanizmach jednocześnie.

#### Przed konfiguracją: poznaj całą adresację

| Połączenie | Interfejs pierwszy | Interfejs drugi |
|---|---|---|
| R5–R1 | R5 `ge-0/0/0.0` 192.0.2.10/24 | R1 `ge-0/0/2.0` 192.0.2.1/24 |
| R1–R2 | R1 `ge-0/0/0.0` 10.12.0.1/30 | R2 `ge-0/0/0.0` 10.12.0.2/30 |
| R1–R3 | R1 `ge-0/0/1.0` 10.13.0.1/30 | R3 `ge-0/0/0.0` 10.13.0.2/30 |
| R2–R4 | R2 `ge-0/0/1.0` 10.24.0.1/30 | R4 `ge-0/0/0.0` 10.24.0.2/30 |
| R3–R4 | R3 `ge-0/0/1.0` 10.34.0.1/30 | R4 `ge-0/0/1.0` 10.34.0.2/30 |
| R4–R6 | R4 `ge-0/0/2.0` 198.51.100.1/24 | R6 `ge-0/0/0.0` 198.51.100.10/24 |

To jest dokładnie sześć segmentów z [mapy referencyjnej](../../labs/reference-diamond-6vsrx/README.md). Nie ma łącza R2–R3 ani R1–R4.

#### Co powinien znać każdy router, żeby ping R5→R6 mógł działać?

Wybieramy **trasy statyczne**, ponieważ czytelnik zna już ich ogólną ideę, a rozbudowane BGP i OSPF pojawią się dopiero w późniejszych rozdziałach.

Na R5 wystarczy wskazać, że sieć LAN-B leży za R1. Na R6 analogicznie, że LAN-A leży za R4. Routery R2 i R3 znają oba końce rdzenia. R1 i R4 mają trasę do drugiego LAN-u przez górny korytarz oraz możliwość wyboru dolnego korytarza jako zapasowego.

**Ważna różnica:** fizyczne istnienie dwóch łączy nie oznacza automatycznie, że Junos przełączy routing. Konfigurując statyczną trasę, musimy jawnie określić next-hop i ewentualną alternatywę.

W tym przykładzie używamy **`qualified-next-hop`**. To sposób nadania odrębnej preferencji zapasowemu następnemu hopowi. Dla zwykłej trasy statycznej preference wynosi typowo 5, a dla naszej alternatywy ustawimy 10. Wartość **niższa** jest lepsza, o ile next-hop jest rozwiązywalny. Nie jest to jednak mechanizm automatycznego monitorowania końcowej aplikacji — statyczny next-hop może nadal wyglądać na osiągalny, gdy odległy fragment drogi jest zepsuty.

#### Konfiguracja dodatkowych routerów — tylko na świeżych klonach

Przygotuj **R3**, następnie **R4**, rozbuduj **R2 i R1**, a na końcu **R6 i R5**. Na każdym urządzeniu osobno wykonaj `configure private`, sprawdź `show | compare`, `commit check` oraz lokalne post-checki. **Nie uruchamiaj timerów `commit confirmed` równocześnie na wszystkich sześciu routerach.** Każde urządzenie musi zostać najpierw poprawnie przygotowane i bezpiecznie zatwierdzone.

**R3-vSRX — Operational Mode, rozpoczęcie konfiguracji nowego routera CORE-2**

~~~junos
configure private
~~~

**R3-vSRX — Configuration Mode [edit private], dolny korytarz i trasy w obu kierunkach**

~~~junos
set system host-name R3-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 10.13.0.2/30
set interfaces ge-0/0/1 unit 0 family inet address 10.34.0.1/30
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone CORE interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP match source-address any
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP match destination-address any
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP match application junos-ping
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP then permit
set routing-options static route 192.0.2.0/24 next-hop 10.13.0.1
set routing-options static route 198.51.100.0/24 next-hop 10.34.0.2
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

Na R3 obie strony połączenia są w tej samej strefie CORE. Tworzymy jawne, ograniczone laboratoryjne zezwolenie **CORE→CORE dla nowych sesji ICMP**. Dzięki temu późniejsza diagnostyka nie będzie oparta na przypadkowych domyślnych regułach intrazone danego obrazu SRX.

Gdy konfiguracja R3 przejdzie lokalne kontrole, zatwierdź ją. Pełen test R3–R4 może być jeszcze niedostępny, ponieważ R4 nie jest gotowy; to nie powód do przedłużania timera w ciemno.

**R3-vSRX — Operational Mode, osobna sesja, post-check**

~~~junos
show configuration interfaces | display set
show configuration security zones | display set
show route table inet.0
show system commit
~~~

**R3-vSRX — Configuration Mode [edit private], po zatwierdzeniu poprawnego stanu lokalnego**

~~~junos
commit comment "CH05 post-check verified"
~~~

Teraz przygotuj R4, który będzie miał dwa interfejsy do rdzenia i jeden do R6.

**R4-vSRX — Operational Mode, wejście do edycji EDGE-B**

~~~junos
configure private
~~~

**R4-vSRX — Configuration Mode [edit private], połączenia CORE-1, CORE-2 i CE-B**

~~~junos
set system host-name R4-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 10.24.0.2/30
set interfaces ge-0/0/1 unit 0 family inet address 10.34.0.2/30
set interfaces ge-0/0/2 unit 0 family inet address 198.51.100.1/24
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone CORE interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone CE interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE match source-address any
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE match destination-address any
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE match application junos-ping
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE then permit
set security policies from-zone CE to-zone CORE policy CH05-CE-TO-CORE match source-address any
set security policies from-zone CE to-zone CORE policy CH05-CE-TO-CORE match destination-address any
set security policies from-zone CE to-zone CORE policy CH05-CE-TO-CORE match application junos-ping
set security policies from-zone CE to-zone CORE policy CH05-CE-TO-CORE then permit
set routing-options static route 192.0.2.0/24 next-hop 10.24.0.1
set routing-options static route 192.0.2.0/24 qualified-next-hop 10.34.0.1 preference 10
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

Na R4 konfigurujemy polityki **w obu kierunkach**, bo w dalszych testach R6 ma móc również inicjować nowy ping do R5. Sesja powrotna dla wcześniej dozwolonego flow jest mechanizmem stateful, ale osobno inicjowana sesja w przeciwnym kierunku może wymagać własnej policy.

Dla LAN-A R4 wybierze next-hop R2 (10.24.0.1) jako podstawowy, a R3 (10.34.0.1) jako alternatywę o preference 10, jeśli jej użycie będzie możliwe. Nie oznacza to monitorowania końcowego R5.

**R4-vSRX — Operational Mode, osobna sesja, lokalny post-check**

~~~junos
show interfaces terse
show route 192.0.2.0/24 exact extensive
show security policies
show system commit
~~~

**R4-vSRX — Configuration Mode [edit private], po pozytywnym lokalnym post-check**

~~~junos
commit comment "CH05 post-check verified"
~~~

R2 ma już interfejs do R1. Dodajemy mu port do R4 i trasę do LAN-B. Na dwóch portach CORE dopuszczamy nowy ICMP między interfejsami w tej samej strefie.

**R2-vSRX — Operational Mode, wejście do edycji CORE-1**

~~~junos
configure private
~~~

**R2-vSRX — Configuration Mode [edit private], drugi port i połączenie do LAN-B**

~~~junos
set interfaces ge-0/0/1 unit 0 family inet address 10.24.0.1/30
set security zones security-zone CORE interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP match source-address any
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP match destination-address any
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP match application junos-ping
set security policies from-zone CORE to-zone CORE policy CH05-CORE-ICMP then permit
set routing-options static route 198.51.100.0/24 next-hop 10.24.0.2
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

**R2-vSRX — Operational Mode, osobna sesja, port i trasa**

~~~junos
show interfaces ge-0/0/1 terse
show route 198.51.100.0/24 exact
ping 10.24.0.2 source 10.24.0.1 count 3
~~~

**R2-vSRX — Configuration Mode [edit private], po PASS**

~~~junos
commit comment "CH05 post-check verified"
~~~

Na R1 dodamy drugą drogę, do R3, oraz trasę do LAN-B. R1 zachowuje swoją istniejącą konfigurację z poziomu 2.

**R1-vSRX — Operational Mode, ponowne wejście do edycji EDGE-A**

~~~junos
configure private
~~~

**R1-vSRX — Configuration Mode [edit private], dolne połączenie i alternatywna trasa**

~~~junos
set interfaces ge-0/0/1 unit 0 family inet address 10.13.0.1/30
set security zones security-zone CORE interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set routing-options static route 198.51.100.0/24 next-hop 10.12.0.2
set routing-options static route 198.51.100.0/24 qualified-next-hop 10.13.0.2 preference 10
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE match source-address any
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE match destination-address any
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE match application junos-ping
set security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE then permit
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

Zwróć uwagę: nie zmieniliśmy numerów portów istniejącego rdzenia. `ge-0/0/0` nadal prowadzi do R2, `ge-0/0/1` do R3, a `ge-0/0/2` do R5. W laboratorium czytelnik ma nauczyć się **zachowywać spójną adresację podczas diagnostyki**, a nie zmieniać topologię z każdą nową lekcją.

**R1-vSRX — Operational Mode, osobna sesja, przegląd nowego układu**

~~~junos
show interfaces terse
show route 198.51.100.0/24 exact extensive
show route forwarding-table destination 198.51.100.10
show security policies
~~~

**R1-vSRX — Configuration Mode [edit private], po pozytywnym post-check**

~~~junos
commit comment "CH05 post-check verified"
~~~

Ostatni router zdalny to R6. Ma on własny adres 198.51.100.10/24 i będzie odbiorcą pingu z R5.

**R6-vSRX — Operational Mode, wejście do edycji CE-B**

~~~junos
configure private
~~~

**R6-vSRX — Configuration Mode [edit private], adres LAN-B i droga powrotna**

~~~junos
set system host-name R6-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.10/24
set security zones security-zone CE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set routing-options static route 192.0.2.0/24 next-hop 198.51.100.1
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

**R6-vSRX — Operational Mode, osobna sesja, reachability R4**

~~~junos
show interfaces ge-0/0/0 terse
show route 192.0.2.0/24 exact
ping 198.51.100.1 source 198.51.100.10 count 3
~~~

**R6-vSRX — Configuration Mode [edit private], po PASS**

~~~junos
commit comment "CH05 post-check verified"
~~~

Na końcu R5 musi poznać sieć LAN-B przez R1. Jego trasa do 10.12.0.0/30 z poziomu 2 nie wystarcza do osiągnięcia 198.51.100.10.

**R5-vSRX — Operational Mode, wejście do edycji CE-A**

~~~junos
configure private
~~~

**R5-vSRX — Configuration Mode [edit private], trasa do LAN-B**

~~~junos
set routing-options static route 198.51.100.0/24 next-hop 192.0.2.1
show | compare
commit check
commit confirmed 5 comment "CH05 controlled lab change"
~~~

**R5-vSRX — Operational Mode, osobna sesja, test docelowy**

~~~junos
show route 198.51.100.0/24 exact
ping 198.51.100.10 source 192.0.2.10 count 5
~~~

**R5-vSRX — Configuration Mode [edit private], po rzeczywistym PASS**

~~~junos
commit comment "CH05 post-check verified"
~~~

W tym momencie **może** działać ping R5→R6 przez górny korytarz, jeśli wszystkie interfejsy, strefy, polityki i drogi powrotne są poprawne. Nie ogłaszamy automatycznego PASS; wynik trzeba potwierdzić na działającym labie. W niektórych wydaniach vSRX może być konieczne dopasowanie szczegółów konfiguracji lokalnie inicjowanego ruchu lub sesji do konkretnej platformy.

### Bazowy test górnej i dolnej ścieżki

Przed fault zapisz aktywną drogę do LAN-B na R1 i drogę powrotną do LAN-A na R4.

**R1-vSRX — Operational Mode, która droga jest podstawowa?**

~~~junos
show route 198.51.100.0/24 exact extensive
show route forwarding-table destination 198.51.100.10
show interfaces ge-0/0/0 terse
show interfaces ge-0/0/1 terse
~~~

**R4-vSRX — Operational Mode, skąd wróci ruch do LAN-A?**

~~~junos
show route 192.0.2.0/24 exact extensive
show route forwarding-table destination 192.0.2.10
show interfaces ge-0/0/0 terse
show interfaces ge-0/0/1 terse
~~~

**R5-vSRX — Operational Mode, wynik ruchu generowanego lokalnie**

~~~junos
ping 198.51.100.10 source 192.0.2.10 count 5
traceroute 198.51.100.10 source 192.0.2.10
~~~

**EXPECTED:** podstawowa droga R1→LAN-B prowadzi przez 10.12.0.2 (R2), a R4→LAN-A przez 10.24.0.1 (R2). Traceroute może pokazywać gwiazdki, jeśli sondy i odpowiedzi są ograniczane przez polityki — nie traktuj tego jako dowodu braku tranzytu.

### Fault D — R1 traci górne łącze, ale R4 nadal preferuje górny powrót

Wyłączymy **tylko** interfejs R1→R2. Nie ruszamy R4 ani R3. To zaawansowane ćwiczenie, ponieważ na pierwszy rzut oka obecność dolnej drogi mogłaby sugerować automatyczny sukces failover.

**R1-vSRX — Configuration Mode [edit private], fault D w izolowanym DIAMOND-6R**

~~~junos
set interfaces ge-0/0/0 disable
show | compare
commit check
commit confirmed 3 comment "CH05 controlled lab change"
~~~

Teraz wykonaj odczyty bez dalszej zmiany konfiguracji.

**R1-vSRX — Operational Mode, warunkowy stan po wyłączeniu portu**

~~~junos
show interfaces ge-0/0/0 terse
show route 198.51.100.0/24 exact extensive
show route forwarding-table destination 198.51.100.10
show system commit
~~~

**R4-vSRX — Operational Mode, niezależnie sprawdź trasę powrotną**

~~~junos
show route 192.0.2.0/24 exact extensive
show route forwarding-table destination 192.0.2.10
~~~

**R5-vSRX — Operational Mode, test usługi podczas awarii**

~~~junos
ping 198.51.100.10 source 192.0.2.10 count 5
~~~

Jaki wynik powinieneś przewidzieć? Na R1 next-hop przez R2 może stać się nieaktywny, a trasa zapasowa przez R3 może zostać użyta — **jeżeli** odzwierciedla to stan połączenia i poprawnie rozwiązuje next-hop. Ale R4 nadal może preferować drogę powrotną przez R2. R2 z kolei może nie mieć już skutecznej drogi do LAN-A przez zerwane R1–R2.

W rezultacie **forward może pójść R1–R3–R4, a return próbować wrócić R4–R2–R1**. Nie mamy więc prawa zagwarantować udanego pingu tylko dlatego, że R1 zobaczył zapasowy next-hop.

To najważniejszy praktyczny wniosek z tego fault: **failover w jednym routerze nie dowodzi konwergencji całej sieci**. Trzeba zbadać obie strony i ewentualne asymetryczne ścieżki. Na urządzeniach SRX asymetria może dodatkowo kolidować z mechanizmem stanowych sesji bezpieczeństwa.

**Recovery:** nie potwierdzaj warunkowego commitu. Po automatycznym rollback sprawdź, że górny link i preferowana trasa wróciły do bazowego stanu, a test R5→R6 ponownie działa. Zapisz realną drogę wybraną przez R1 i R4. Jeśli zachowanie różni się od oczekiwanego, przeanalizuj next-hop, inne trasy w RIB oraz stan logicznych portów.

### Dlaczego nie uruchamiamy na ślepo „awarii całego R2”?

Wyłączenie VM R2 może nie wywołać identycznego stanu `link down` na R1 i R4 w każdym modelu połączeń EVE-NG. Wirtualny most i vNIC mogą nadal pokazywać link aktywny, choć sąsiad już nie odpowiada na ARP. Trasa statyczna oparta tylko na rozwiązywalności next-hopu nie wykrywa każdego rodzaju awarii sieci.

Dlatego bardziej zaawansowany test konwergencji wymaga osobnej analizy mechanizmu wykrywania awarii, protokołu dynamicznego lub świadomego śledzenia reachability (np. rozwiązania RPM/IP monitoring na wspieranej platformie). W rozdziałach 6–11 nauczymy się, jak takie rozwiązania zaprojektować. Nie obiecujemy automatycznego przełączenia na R3 bez określenia mechanizmu.

### Fault E — jedna błędna policy i pozornie poprawny routing

W pełnym DIAMOND-6R wyłącz zezwolenie CORE→CE na **R4**, pozostawiając wszystkie statyczne trasy bez zmian. Dzięki temu zobaczysz, jak awaria na ostatnim routerze może wyglądać z R5 jak „problem routingu”.

**R4-vSRX — Configuration Mode [edit private], fault E tylko na SRX policy**

~~~junos
deactivate security policies from-zone CORE to-zone CE policy CH05-CORE-TO-CE
show | compare
commit check
commit confirmed 3 comment "CH05 controlled lab change"
~~~

**R4-vSRX — Operational Mode, obserwacja policy i sesji**

~~~junos
show route 198.51.100.0/24 exact
show route 192.0.2.0/24 exact
show security policies from-zone CORE to-zone CE
show security flow session
~~~

**R5-vSRX — Operational Mode, nowa próba po zmianie policy**

~~~junos
ping 198.51.100.10 source 192.0.2.10 count 5
~~~

**EXPECTED:** jeżeli nie ma innej policy dopuszczającej ten ruch, **nowa sesja** z rdzenia do R6 może zostać odrzucona na R4, mimo że R1–R3 i routing wyglądają poprawnie. Istniejące sesje mogły powstać wcześniej; należy rozróżnić ich zachowanie od nowej inicjacji.

**Recovery:** po automatycznym rollback przeprowadź nowy test ICMP i porównaj stan policy. W ten sposób naprawa dotyczy dokładnie tego miejsca, w którym zidentyfikowaliśmy problem.

### Fault F — missing route na R6: pakiet dochodzi, ale odpowiedź nie ma dokąd wrócić

Możemy również usunąć na R6 jedyną trasę do LAN-A. Na R5, R1 i R4 nie zmieniamy niczego.

**R6-vSRX — Configuration Mode [edit private], fault F: brak powrotu do R5**

~~~junos
delete routing-options static route 192.0.2.0/24
show | compare
commit check
commit confirmed 3 comment "CH05 controlled lab change"
~~~

**R6-vSRX — Operational Mode, obserwacja drogi powrotnej**

~~~junos
show route 192.0.2.0/24 exact
show route forwarding-table destination 192.0.2.10
show interfaces ge-0/0/0 terse
~~~

**R5-vSRX — Operational Mode, test komunikacji po zmianie tylko R6**

~~~junos
ping 198.51.100.10 source 192.0.2.10 count 5
~~~

**EXPECTED:** bez innej drogi obejmującej 192.0.2.10 R6 nie będzie umiał poprawnie odesłać odpowiedzi do R5. Port R6 do R4 nadal może być `up/up`, a host-bound request może dotrzeć do urządzenia. To brak **return path**, nie brak local link.

Jeżeli na R6 istnieje default route lub BGP z innego laboratorium, wynik może być inny. Zanim uznasz test za PASS, sprawdź pełne dopasowanie routingu.

**Recovery:** nie potwierdzaj commitu; po rollback sprawdź powrót trasy R6 i dwukierunkową łączność. Nie dodawaj default route bez potrzeby, bo zmieniałaby cel ćwiczenia.

### Co zachować po zakończeniu poziomu 3?

Dla każdej awarii zachowaj **rzeczywisty czas i hostname**, wynik odczytu przed zmianą, zmianę w `show | compare`, status po aktywacji, wynik testu z R5/R6 oraz stan po rollback. Jeżeli nie masz zrzutów dla konkretnej wersji Junos, wpisz **NOT TESTED**.

Nie wyczyść z rozpędu całego DIAMOND-6R po jednym ćwiczeniu. Użyj oddzielnych snapshotów lub kontrolowanej procedury cleanup, tak aby rozdziały o OSPF/BGP mogły wykorzystać ten sam, stabilny projekt.

**Źródła do interpretacji:** [Static Route Preferences and Qualified Next Hops](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html), [SRX Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html), [commit confirmed](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html).


---

## 5.TS. Sześć przypadków z pracy inżyniera — jak odróżnić przyczynę od objawu? [EXT]

Po laboratorium warto sprawdzić, czy potrafisz przenieść sposób rozumowania na inne urządzenia, także fizyczne. Poniższe sytuacje są **autorskimi scenariuszami szkoleniowymi**. Nie przedstawiamy ich jako prawdziwych ticketów klienta ani jako pytań z egzaminu.

### Przypadek 1. Switch access jest `up/up`, ale wideokonferencje się zacinają

Na fizycznym EX uplink pracuje z prędkością 1 Gb/s. Zespół monitoringu pokazuje momenty bardzo dużego ruchu, a użytkownicy skarżą się na krótkie przerwy w dźwięku. Liczniki CRC nie rosną, ale przybywa odrzuceń w kolejce wyjściowej.

Nie masz na razie dowodu na wadliwy światłowód. Bardziej prawdopodobna jest hipoteza przeciążenia interfejsu lub konkretnej klasy ruchu. Sprawdź **przyrost liczników z czasem**, obciążenie, parametry kolejek i zasady CoS. Jeśli ruch pojawia się krótkimi seriami, średnie pięciominutowe może nie pokazywać skali chwilowych skoków.

Nie rozwiązuje się takiego problemu automatycznie zwiększeniem MTU. To inny parametr. Nie wyłączaj też funkcji bezpieczeństwa tylko dlatego, że przepływność jest duża.

### Przypadek 2. R1 pokazuje trasę do R6, ale pakiet nie dociera

Na R1 w `show route` widać poprawną sieć 198.51.100.0/24, a operator mówi „routing jest OK”. W rzeczywistości route może wskazywać next-hop, którego warstwa 2 nie rozwiązuje prawidłowo. Może też brakować drogi powrotnej na R6 albo security policy może odrzucać nową sesję.

Właściwa diagnoza to porównanie RIB z forwarding, ARP, stanem interfejsów, rzeczywistą próbą źródło→cel i polityką SRX. Nie każda trasa obecna w tablicy gwarantuje dostarczenie pakietu.

W przypadku połączeń R1–R2 lub R1–R3 warto dodatkowo sprawdzić, **czy w EVE-NG właściwy vNIC jest połączony z właściwym segmentem**. Zgodny adres w konfiguracji nie dowodzi poprawnego wiring.

### Przypadek 3. `monitor traffic` nie pokazuje pakietów, które widzi system bezpieczeństwa

Administrator uruchamia `monitor traffic` na porcie core i nie widzi strumienia, który NMS mierzy jako setki Mb/s. Wnioskuje, że NMS się myli. Ale `monitor traffic` może nie być źródłem pełnego obrazu ruchu PFE transit.

Należy sprawdzić **dokumentowany punkt obserwacji** dla danej platformy, a na SRX wykorzystać security flow, liczniki i test aplikacyjny. Jeśli potrzebny jest pełny packet capture ruchu tranzytowego, dobiera się dedykowaną metodę dla platformy, np. port mirroring w obsługiwanym sprzęcie lub właściwy tryb capture. Nie należy włączać szerokiego debugowania w ciemno.

### Przypadek 4. Po upgrade router odpowiada na SSH, ale monitoring zgłasza go jako niedostępny

System wrócił po planowanej aktualizacji, `show version` wskazuje oczekiwane wydanie, a NMS nadal nie otrzymuje SNMPv3. Nie oznacza to, że cały upgrade zakończył się sukcesem i problem należy do zespołu monitoringu.

Sprawdź na routerze stan usługi, konfigurację SNMPv3, engine ID, dostępność serwera, właściwy source address i routing instance oraz ewentualne zmiany w zachowaniu nowej wersji. W szczególności jeśli zarządzanie ma wydzieloną instancję, działanie SSH z konsoli administracyjnej może nie oznaczać, że router prawidłowo wysyła albo odbiera polling SNMP przez tę samą sieć.

Dobry post-check po upgrade zawiera wynik z **systemu monitoringu**, nie tylko listę komend na routerze.

### Przypadek 5. Po restarcie jeden członek Virtual Chassis nie wrócił

Na fizycznym stacku EX lub Virtual Chassis część portów działa, ale monitoring wykazuje zanik urządzeń przyłączonych do jednego członka. Możliwe, że logiczny system działa na pozostałych elementach, a jeden członek nie dołączył.

Nie wystarczy wykonanie `show version` na masterze. Potrzeba właściwych komend dotyczących **Virtual Chassis**, stanu członków, interfejsów między nimi, kompatybilności software i alarmów. To osobny rodzaj urządzenia niż nasze sześć niezależnych vSRX; nie myl VC ze zwykłymi routerami połączonymi kablami.

Nie restartuj kolejnego członka w celu „wyrównania” stanu, dopóki nie rozpoznasz, które usługi na nim zależą.

### Przypadek 6. Administrator chce ominąć walidację instalacji

Podczas patchowania instalator odrzuca nowy pakiet. Ktoś proponuje użyć `no-validate`. Zanim się zgodzisz, ustal, jaki to dokładnie model, jaki release jest obecnie zainstalowany, na jaki release wykonujesz upgrade i **co mówią release notes oraz CLI reference dla tej pary**.

Niektóre wyjątkowe ścieżki wymagają `no-validate`, ale nie jest to uniwersalne obejście. Jeśli zgodność obrazu jest nieznana, trzeba przerwać operację i zebrać dokumentację. Celem utrzymania nie jest doprowadzenie instalatora do wyświetlenia „complete” za wszelką cenę, lecz utrzymanie wymaganej usługi.

---

## 5.TEST. Sprawdzenie wiedzy — 24 pytania, z wyjaśnieniami [EXT]

Pytania są autorskie i służą powtórce. Ich celem nie jest pamięciowe wyliczenie komend, ale umiejętność wyciągania wniosków z obserwacji.

### Pytania

**1.** R1 ma skonfigurowany adres 10.12.0.1/30. Czy samo `show configuration interfaces` dowodzi, że R1 może przesyłać pakiety do R2?

**2.** Na `show interfaces terse` port jest `up/up`. Czy można wykluczyć problemy z CRC, drops i kolejkami?

**3.** Dlaczego na R1 warto porównać `show route` i `show route forwarding-table`?

**4.** Na routerze widzisz 100 destinations i 145 routes. Czy to wewnętrzna sprzeczność?

**5.** Co oznacza `show route | count` i dlaczego nie powinno być jedynym sposobem liczenia prefiksów?

**6.** `show system processes extensive` pokazuje wysokie CPU jednego procesu przez chwilę. Czy należy od razu restartować `rpd`?

**7.** Dlaczego jedno uruchomienie `show interfaces extensive` nie wystarcza, żeby stwierdzić, że błędy CRC wciąż przyrastają?

**8.** O 12:00 licznik CRC wynosi 240. O 12:05 wynosi 340. Ile błędów przybyło i jakie dodatkowe informacje powinieneś zebrać?

**9.** Na fizycznym switchu Rx optical power wskazuje `-18 dBm`. Czy sama ta wartość oznacza, że światłowód jest uszkodzony?

**10.** Dlaczego Rx na switchu A może być niski, mimo że Tx na switchu A wygląda dobrze?

**11.** Jakie ryzyko wiąże się z wyzerowaniem liczników podczas aktywnego incydentu?

**12.** Dlaczego ping z R1 do 10.12.0.2 nie dowodzi, że pakiet z R5 przejdzie przez R1 do R2?

**13.** R1→R2 ping z source 10.12.0.1 działa, ale z source 10.255.255.1 nie działa. Jaki element musisz sprawdzić na R2?

**14.** Traceroute pokazuje gwiazdkę na R3, lecz aplikacja działa przez ten router. Czy R3 musi odrzucać wszystkie pakiety?

**15.** Dlaczego `monitor traffic` nie zawsze pokaże cały strumień pakietów transmitowany przez PFE?

**16.** Czym `monitor start messages` różni się od włączenia traceoptions konkretnego protokołu?

**17.** Czy jedna paczka instalacyjna Junos pasuje do EX, QFX, MX, SRX i vSRX?

**18.** W której sytuacji `no-validate` jest uzasadnione: gdy instalator zgłosi jakikolwiek błąd czy gdy producent wyraźnie dokumentuje wyjątek dla konkretnej ścieżki?

**19.** Co różni konfigurację rollback `rollback 1` od rollback oprogramowania?

**20.** Dlaczego zrobienie snapshotu po upgrade może zmniejszyć możliwość prostego powrotu do wcześniejszej wersji?

**21.** Co powinieneś ustalić, zanim wykonasz reboot routera obsługującego ruch użytkowników?

**22.** Czy recovery hasła root jest identyczne dla klasycznego Junos FreeBSD, Junos OS Evolved i każdego obrazu vSRX?

**23.** R1 ma zapasową trasę przez R3, a R4 nadal używa R2 do powrotu. Czy możesz zagwarantować łączność R5→R6 po awarii R1–R2?

**24.** Po `commit confirmed` wykonałeś `commit check`. Co może się stać z planowanym automatycznym rollbackiem?

### Odpowiedzi i sposób rozumowania

**1.** Nie. `show configuration` pokazuje ustawienia. Potrzeba odczytu statusu interfejsu, trasy, ARP i odpowiedniego testu z R1, a przy tranzycie również security.

**2.** Nie. `up/up` jest tylko informacją o stanie portu. Szczegółowe błędy i odrzucenia wymagają dodatkowych odczytów, najlepiej porównanych w czasie.

**3.** `show route` pokazuje informacje routingu, a forwarding-table widok decyzji do przekazywania pakietów. Samo istnienie route nie dowodzi dostarczenia pakietu.

**4.** Nie. Jeden prefiks może mieć więcej niż jedną znaną trasę. Destinations i routes nie muszą być równe.

**5.** Liczy linie wyświetlonego wyniku tekstowego, a nie automatycznie prefiksy czy aktywne wpisy RIB. Użyj świadomie właściwych pól statystyk tablic.

**6.** Nie. Najpierw ustal czas trwania, konkretny proces, związane zdarzenia i rzeczywisty wpływ na control/forwarding. Restart procesu może być operacją z impaktem.

**7.** Licznik zwykle pokazuje wartość skumulowaną. Potrzeba drugiego pomiaru w określonym odstępie, aby rozpoznać przyrost.

**8.** Przybyło 100 błędów w pięć minut, średnio 20/min. Sprawdź też ruch, uptime, czy licznik nie był resetowany, port po drugiej stronie i typ medium.

**9.** Nie. Prawidłowy zakres zależy od konkretnego modułu, typu włókna i progów alarmowych. Samo ujemne dBm nie jest awarią.

**10.** Rx na A mierzy sygnał z Tx na **B**, a nie z Tx na A. Trzeba sprawdzić oba końce i odpowiednie włókno.

**11.** Może zniszczyć punkt odniesienia i utrudnić analizę trendów przez innego administratora lub system NMS.

**12.** Ping z R1 jest locally originated, podczas gdy R5→R2 przechodzi przez R1 i może podlegać innej security policy. Potrzeba też poprawnej drogi powrotnej do R5.

**13.** R2 może nie mieć poprawnej trasy powrotnej do loopback R1 10.255.255.1; sprawdź routing oraz ewentualne host-inbound/security.

**14.** Nie. Traceroute może nie dostać oczekiwanej odpowiedzi ICMP od R3, mimo że forwarding innych pakietów działa.

**15.** Ruch transit może być przetwarzany w PFE bez pełnego przekazania jego kopii do ścieżki monitorowanej przez RE.

**16.** `monitor start messages` pokazuje nowo dopisywane linie do istniejącego pliku logu. Traceoptions włączają bardziej szczegółowe rejestrowanie wybranych zdarzeń protokołu/procesu.

**17.** Nie. Wybór obrazu, ścieżki upgrade i procedury zależy od konkretnej rodziny, modelu i release.

**18.** W drugim przypadku, czyli gdy oficjalna dokumentacja dla tej platformy i wersji wymaga wyjątku. `no-validate` nie jest uniwersalnym lekarstwem na nieznane błędy.

**19.** Pierwsze dotyczy starszej konfiguracji Junos, drugie instalacji systemu operacyjnego. Cofnięcie konfiguracji nie cofa wersji Junos.

**20.** Na wybranych urządzeniach snapshot może nadpisać zapasową instalację, przez co poprzedni obraz nie jest już zachowany jako łatwa ścieżka powrotu.

**21.** Znaczenie urządzenia w topologii, możliwy wpływ na usługi, zależności HA, dostęp konsolowy, backup, stan przed zmianą, okno, procedurę restartu i testy powrotu.

**22.** Nie. Mechanizm boot i ograniczenia bezpieczeństwa zależą od platformy, release i polityk. Recovery wymaga autoryzacji i właściwej konsoli.

**23.** Nie. Powrót przez R2 może przestać działać, mimo że R1 wyśle pakiet przez R3. Oba kierunki oraz stanowe sesje SRX wymagają sprawdzenia.

**24.** `commit check` uruchomione podczas oczekiwania może potwierdzić warunkowy commit, wyłączając automatyczny rollback. Dlatego walidacja składni powinna odbyć się **przed** `commit confirmed`, a po aktywacji testujemy read-only.

### Czy po tym teście jesteś gotów do kolejnego rozdziału?

Powinieneś umieć wyjaśnić, jakie dane z R1, R2 i R4 trzeba zebrać, gdy R5 nie może komunikować się z R6; jak wyciągnąć wniosek z licznika błędów oraz dlaczego sam wynik `commit complete`, SSH czy ping nie dowodzi działania całej sieci.

Nie musisz jeszcze znać wszystkich szczegółów OSPF ani BGP. W kolejnych rozdziałach zbudujemy na tym fundamencie statyczne i dynamiczne mechanizmy routingu, wykorzystując te same narzędzia diagnostyczne.


---

## 5.REF. Skrócony przewodnik operacyjny — od pytania do właściwego odczytu [EXT]

Poniższa mapa pomaga wrócić do polecenia po zrozumieniu mechanizmu. Nie jest zastępstwem dla wykładu ani uniwersalnym scenariuszem do wklejenia w każdej awarii.

| Pytanie inżynierskie | Polecenie / mechanizm | Ograniczenie, o którym trzeba pamiętać |
|---|---|---|
| Jaki model i wydanie? | `show version`, `show chassis hardware` | Obraz i funkcje są zależne od modelu |
| Czy urządzenie niedawno restartowano? | `show system uptime`, logi | Niski uptime nie wyjaśnia automatycznie przyczyny |
| Czy system zgłasza alarmy? | `show system alarms` | Brak alarmu nie dowodzi zdrowia usługi |
| Czy port jest obecnie aktywny? | `show interfaces terse` | `up/up` nie wyklucza błędów ani blokad security |
| Czy błędy rosną? | Dwa odczyty `show interfaces ... extensive` | Sprawdź przedział czasu i możliwość resetu liczników |
| Czy fizyczna optyka jest zdrowa? | `show interfaces diagnostics optics` | Wymaga rzeczywistego i wspieranego transceivera |
| Czy router zna trasę? | `show route`, `show route ... exact` | Trasa może być nieaktywna lub hidden |
| Jaki jest next-hop forwarding? | `show route forwarding-table` | Odczyt widoku forwarding ≠ dostarczenie pakietu |
| Czy działa ARP? | `show arp` | Bez poprawnego L2 next-hop może nie być użyteczny |
| Czy SRX dopuścił nową sesję? | `show security policies`, `show security flow session` | Starsze sesje mogą przetrwać zmianę policy |
| Czy proces jest przeciążony? | `show system processes extensive` | Pomiar chwilowy ≠ trwała awaria |
| Czy port zmienia stan w czasie? | `monitor interface`, logi | Platforma i terminal wpływają na dostępność |
| Czy pojawiają się nowe wiadomości? | `monitor start messages` | Nie włącza traceoptions protokołu |
| Czy widzę pakiet do RE? | Ograniczone `monitor traffic` | Nie jest pełnym capture PFE transit |
| Czy bezpośredni sąsiad odpowiada? | `ping ... source ...` | Nie dowodzi komunikacji komputer-komputer |
| Czy znam kolejne hopy? | `traceroute ... source ...` | Gwiazdki nie dowodzą utraty wszystkich pakietów |
| Czy management jest w odrębnym VRF? | `show route table mgmt_junos.inet.0` | Tabela może nie istnieć lub nie być aktywna |
| Czy mamy miejsce na software? | `show system storage` | Nie kasuj plików bez przeglądu |
| Co można usunąć z dysku? | `request system storage cleanup dry-run` | Faktyczny cleanup jest operacją zmieniającą stan |
| Które zmiany ostatnio wykonano? | `show system commit` | Związek czasowy nie zawsze oznacza przyczynę |
| Czy czeka zaplanowany restart? | `show system reboot` | Obecność/wynik zależy od platformy |
| Czy można bezpiecznie cofnąć zmianę? | `show | compare`, `commit confirmed`, rollback plan | `commit check` podczas oczekiwania może potwierdzić commit |
| Czy został zrobiony software upgrade? | `show version`, release notes, porównanie pre/post | Sam nowy numer nie dowodzi zdrowia sieci |

### Jeden operacyjny nawyk wart zachowania

Przed wydaniem polecenia odpowiedz sobie: „**Jaki wynik zmieni moją hipotezę?**”. Jeśli każdy możliwy wynik prowadzi Cię do tej samej przypadkowej naprawy, test prawdopodobnie nie został właściwie zaprojektowany.

---

## Źródła, mapowanie blueprintu i granice walidacji

Rozdział przygotowano na podstawie [unified blueprintu JNCIA-JNCIS-JNCIP](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md), wcześniejszej wersji CH05 [zachowanej w archiwum](../archive/JUNOS_Rozdzial_05_pre_beginner_rewrite_2026-11-20.md), [referencyjnej topologii DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md) oraz [rejestru źródeł i gotchas CH05](../research/CH05_MONITORING_MAINTENANCE_GOTCHAS_I_ZRODLA_2026-11-17.md). Zachowano pełną kolejność punktów 5.1–5.9 i wyjaśniono je bez wymagania wcześniejszej wiedzy o Cisco.

**Mapa pokrycia:**

| Wymaganie blueprintu | Sekcja |
|---|---|
| 5.1 `show` — stan, konfiguracja, routing i interfejsy | §5.1 |
| 5.2 `monitor` — obserwacja w czasie | §5.2 |
| 5.3 Statystyki i błędy interfejsów | §5.3 |
| 5.4 Ping, traceroute, telnet, SSH i inne testy | §5.4 |
| 5.5 Instalacja i aktualizacja Junos | §5.5 |
| 5.6 Boot, reboot, halt i utrzymanie | §5.6 |
| 5.7 Root password recovery | §5.7 |
| 5.8 Monitoring, maintenance, troubleshooting | §5.8 |
| 5.9 [LAB] Interfejs, trasa, konfiguracja | §5.9 — 1 vSRX → 3 vSRX → 6 vSRX |
| Rozszerzenie dydaktyczne | §5.TS, §5.TEST, §5.REF |

**Najważniejsze oficjalne źródła Juniper/HPE:**

1. [Operational Monitoring in Junos CLI](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-operational-mode-commands-monitoring.html).
2. [show route](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html), [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html).
3. [monitor interface](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-interface.html), [monitor start](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-start.html), [monitor traffic](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html).
4. [show interfaces extensive](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-interfaces-extensive.html), [Understanding Packet Drops](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/using-show-commands-for-packet-drops.html), [Digital Optical Monitoring](https://www.juniper.net/documentation/us/en/software/junos/chassis/topics/concept/digital-optical-monitoring.html).
5. [clear interfaces statistics](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/clear-interfaces-statistics.html).
6. [ping](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/ping.html), [traceroute](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/traceroute.html).
7. [Junos Software Installation and Upgrade](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/software-install-and-upgrade-overview.html), [request system software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html), [Storage Cleanup](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-storage-cleanup.html).
8. [Snapshot and Backup Installation](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/backing-up-install-using-snapshot.html), [Software rollback](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-rollback.html).
9. [request system reboot](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-reboot.html), [request system halt](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-halt.html), [show system reboot](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-reboot.html).
10. [Recover root password on Junos OS](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/recovering-root-password.html), [Recover root password on Junos OS Evolved](https://www.juniper.net/documentation/us/en/software/junos/user-access-evo/user-access/topics/topic-map/recovering-root-password.html).
11. [SRX Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html), [Static Route Preferences and Qualified Next Hops](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html).
12. [commit CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html).

**Granice walidacji technicznej:** w tym opracowaniu nie uruchomiono EVE-NG, nie wykonano rzeczywistych pingów ani commitów, nie przetestowano portów, sesji SRX i tras ani nie instalowano Junos. Każda przykładowa konfiguracja pozostaje **PLANNED / NOT RUNTIME VALIDATED** i wymaga dostosowania do realnej wersji obrazu vSRX, mapowania kart, istniejących stref i polityk. Scenariusze fizycznej optyki, Virtual Chassis, upgrade i recovery wymagają odrębnych wspieranych platform lub izolowanych klonów. Nie publikuj produkcyjnych adresów zarządzania, haseł, kluczy prywatnych, obrazów QCOW2 ani niezredagowanych konfiguracji klienta.

### Zakończenie rozdziału

Dobre utrzymanie sieci polega na odróżnianiu **zaobserwowanego objawu** od **udowodnionej przyczyny**. Po przejściu tego materiału potrafisz już zacząć incydent od właściwego pytania, dobrać polecenie, które rzeczywiście coś rozstrzyga, bezpiecznie wprowadzić zmianę i wykazać, czy usługa powróciła.

W rozdziale VI przejdziemy do routingu: poznasz od podstaw, w jaki sposób router wybiera drogę dla pakietu. Narzędzia z obecnego rozdziału będą nam stale towarzyszyć, ale tym razem wykorzystamy je do obserwowania decyzji routingu i ich konsekwencji.

