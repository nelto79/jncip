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

