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

