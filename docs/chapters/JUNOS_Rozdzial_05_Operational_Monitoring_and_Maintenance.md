# Rozdział 5. Operational Monitoring and Maintenance — obserwacja, diagnostyka i utrzymanie Junos

**Podręcznik:** Junos Enterprise Routing & Switching · JNCIA-Junos [A] → JNCIS-ENT → JNCIP-ENT  
**Blueprint (kanoniczny):** 5.1–5.8 [A], 5.9 [LAB] — wszystkie punkty zachowane w oryginalnej kolejności.  
**Wersja redakcyjna:** 17 listopada 2026. Źródła: publiczna dokumentacja Juniper/HPE, wyjaśnienia i zadania autorskie.  
**Platformy:** EX, QFX, MX i SRX; gdzie to ważne — klasyczny Junos vs Junos OS Evolved; laboratorium referencyjne **vSRX/EVE-NG**.  
**Status laboratoryjny:** **PLANNED / NOT RUNTIME VALIDATED.** Przykładowe outputy są oznaczane jako ilustracje. Nie potwierdzono działania komend na konkretnej instalacji w Twoim EVE-NG.

> **Umowa dydaktyczna:** jesteś doświadczonym inżynierem Cisco. Nie będę tłumaczył od nowa, czym jest ICMP ani że interfejs ma liczniki. Zbudujemy natomiast nowy odruch: **objaw → hipoteza → test rozstrzygający → przyczyna → ograniczona naprawa → pomiar wyniku**. Gdy zrozumiesz, *co* konkretna komenda mierzy i *czego nie mierzy*, Junos CLI przestanie być obcą listą słów.

## Prolog: 02:41 — monitoring świeci na czerwono, a telefon dzwoni trzeci raz

W dużym zakładzie produkcyjnym monitoring melduje, że piętro biurowca traci dostęp do aplikacji centralnych. Uplink na switchu EX jest `up`. Ping do gateway działa. Router brzegowy SRX odpowiada przez SSH. Service Desk pyta: „Czy to problem LAN, WAN czy firewalla? Możemy zamknąć incydent, skoro ping działa?”.

W Cisco wykonałbyś `show interface`, `show interfaces status`, `show ip route`, `show logging`, ewentualnie `show platform` i `traceroute`. W Junos będziemy szukać tych samych odpowiedzi, ale **mamy trzy oddzielne warstwy prawdy**:

1. **Configuration:** czy urządzenie *powinno* mieć określony adres, port czy trasę?
2. **Operational state:** czy interfejs, sąsiedztwo, trasa i RE *rzeczywiście* są w danym stanie?
3. **Service result:** czy ruch klienta do konkretnej aplikacji przechodzi *w obie strony*, z odpowiedniego źródła i w odpowiedniej instancji?

~~~mermaid
flowchart TD
  ALARM["Alarm NMS: aplikacja nie działa"] --> P["Jakiej usługi i lokalizacji dotyczy symptom?"]
  P --> CFG["Konfiguracja: show configuration"]
  P --> OPS["Stan: show interfaces / route / alarms / log"]
  P --> TEST["Test klienta: source IP / VRF / TCP / DNS"]
  CFG --> H["Hipoteza: L1, L2, L3, security, usługa?"]
  OPS --> H
  TEST --> H
  H --> FIX["Najmniej inwazyjny test i naprawa"]
  FIX --> VERIFY["Postcheck i dokumentacja"]
~~~

**Zapamiętaj od początku:** `show interfaces terse` może pokazać `up/up`, lecz to jeszcze nie dowód braku CRC, strat w kolejce, błędnego VLAN-u lub blokującej security policy. Podobnie `ping` z RE nie jest substytutem testu aplikacji z komputera pracownika.

### Kompas tego rozdziału

**Na pamięć:** `show version`, `show chassis hardware`, `show interfaces terse/extensive`, `show route`, `show route forwarding-table`, `show system alarms`, `monitor interface`, `monitor start/stop`, `monitor traffic`, `ping`, `traceroute`, `show system storage`, `request system software add`, `request system reboot/halt`.

**Do zrozumienia:** liczniki skumulowane vs szybkość wzrostu, RE capture vs PFE forwarding, routing-instance i source address przy testach, różnica software rollback i configuration rollback, konsekwencje reboot na HA/Virtual Chassis, zależność odzyskania hasła od console/bootloader/security controls.

**Do przećwiczenia:** śledzenie alertu NMS aż do fizycznego portu i usługi, bezpieczny `monitor`, interpretacja transceiver Rx/Tx, logi podczas flapu, rozpoznanie brakującej trasy, przed/po upgrade checks, opis wykonania maintenance i trzy kontrolowane fault injections w EVE-NG.

---

## 5.1. Polecenia `show` — zobacz rzeczywisty stan urządzenia, zanim postawisz diagnozę [A]

### 5.1.1. Dlaczego „show” nie jest jedną funkcją?

Wyobraź sobie samochód stojący przy drodze. Dowód rejestracyjny mówi, jaki ma silnik. Komputer pokładowy mówi, jakie są obroty i błędy. Kierowca mówi, że samochód nie przyspiesza. Wszystkie trzy informacje są potrzebne, ale żadna osobno nie jest diagnozą.

W Junos:
- `show configuration` to intencja zapisania parametrów, **active configuration**;
- `show version` oraz `show chassis hardware` identyfikują rzeczywisty hardware/software;
- `show interfaces`, `show route`, `show system` pokazują stan operacyjny;
- test z użytkownika odzwierciedla usługę spoza RE.

**Most z Cisco:** `show running-config` nie mówi Ci, czy BGP neighbor jest Established. `show ip interface brief` nie mówi, czy kolejka uplinku nie upuszcza pakietów. W Junos ta sama zasada jest szczególnie ważna, bo konfiguracja i operacja mają różne konteksty CLI.

### 5.1.2. Trzy pytania z pierwszej minuty incydentu

**Pierwsze:** co to za urządzenie i kiedy się restartowało?  
**Drugie:** czy system zgłasza alarm hardware/software?  
**Trzecie:** czy incydent dotyczy jednej usługi czy całego site'u?

**Urządzenie: EX-ACCESS-01 — Operational Mode, read-only precheck**

~~~junos
show version
show chassis hardware
show system uptime
show system alarms
show chassis alarms
show system users
show system commit
~~~

Wyobraź sobie, że `show system uptime` pokazuje 4 minuty, choć użytkownicy zgłaszali problem przez godzinę. To może być dowód **restartu w trakcie incydentu**, a nie jego rozwiązania. Skoreluj czas z NTP i logami.

**Gotcha:** `show chassis alarms` oraz `show system alarms` nie muszą pokazywać identycznych przyczyn. Chassis może alarmować o zasilaniu i wentylatorach, system o stanie usług czy rescue; model/release decyduje o dostępności i treści. Brak alarmu nie dowodzi sprawności aplikacji.

### 5.1.3. Interfejs ma dwie warstwy stanu

Na Cisco `show ip interface brief` często sprowadza status do `Status` i `Protocol`. Junos rozdziela stan fizycznego interfejsu oraz logicznego unit. Możesz mieć fizyczny `ge-0/0/1` `up`, ale logiczną jednostkę `ge-0/0/1.0` z innym statusem lub błędną rodziną protokołów.

**Urządzenie: EX-ACCESS-01 — Operational Mode**

~~~junos
show interfaces terse
show interfaces descriptions
show interfaces ge-0/0/1 detail
show configuration interfaces ge-0/0/1 | display set
~~~

**Czytanie wyniku:** najpierw znajdź **fizyczne** `ge-0/0/1`, potem `ge-0/0/1.0`; sprawdź stan `Admin/Link`, opis, właściwą family oraz konfigurację. Nie myl chwilowego `up` z długoterminową niezawodnością.

**Dwa bardzo różne przypadki:**
- `Admin down`: port mógł być administracyjnie wyłączony lub zdezaktywowany; nie ma sensu od razu wymieniać wkładki.
- `Admin up / Link down`: sprawdzamy zasilanie peer, patchcord, wkładkę, autonegocjację i warstwę fizyczną.

### 5.1.4. Routing table to nie forwarding table

W rozdziale 2 oddzielaliśmy RE i PFE; teraz wreszcie rozumiesz, po co. Na RE `rpd` może znać trasę, ale forwarding zależy od jej aktywności, rozwiązania next-hopa, odpowiedniej tabeli i instalacji FIB.

**Urządzenie: R1-vSRX — Operational Mode, read-only routing check**

~~~junos
show route summary
show route table inet.0
show route 198.51.100.2 detail
show route forwarding-table
show arp
~~~

**Gotcha Cisco → Junos:** `show route` może pokazywać wiele tras do jednego prefiksu. Nagłówek `destinations` ≠ `routes`, a gwiazdka `*` na pozycji active ma znaczenie. **`show route | count` liczy linie outputu, a nie prefiksy** — ten błąd wyjaśniliśmy w rozdziale 3. `show route hidden` pozwala zobaczyć trasy ukryte, np. wskutek policy, a nie tylko zwycięzcę procesu wyboru. Dokumentacja: [show route](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html), [show route hidden](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-hidden.html).

**Pytanie instruktora:** masz trasę `198.51.100.0/30` w `inet.0`, a ruch nie dochodzi. Co zrobisz? Sprawdzisz, czy jest aktywna, jak rozwiązany jest next-hop, czy poprawna jest tablica (np. `mgmt_junos.inet.0`), a na SRX dodatkowo sesję i security policy. **Nie zatrzymuj się na „route exists”.**

### 5.1.5. RE, procesy i pamięć: czy CPU 90% musi znaczyć DDoS?

Nie. Możesz mieć dużą liczbę logów, timeout AAA, routing reconvergence, proces `rpd` pracujący po zmianie BGP, RE zarządzający CLI albo zwyczajnie krótkotrwałe obciążenie.

**Urządzenie: R1-vSRX — Operational Mode, obciążenie i health**

~~~junos
show chassis routing-engine
show system processes extensive
show system memory
show system storage
show log messages | last 40
~~~

Nie każda platforma wspiera identyczny podzbiór poleceń. Na urządzeniu z wieloma RE/VC trzeba sprawdzić **którego membera / RE** dotyczą pomiary. Na Junos Evolved narzędzia i wartości CPU/memory mają własną semantykę.

**Most z Cisco:** analogia do `show processes cpu sorted` i `show platform resources`. Ale 90% CPU RE nie oznacza automatycznie, że PFE przestał przekazywać ruch — zależy od architektury platformy i rodzaju zdarzenia. Nie buduj incydentu P1 z jednego odczytu CPU.

### 5.1.6. Zbieraj dowody, nie tylko wielki dump

Załóżmy, że masz 12 000 linii `show interfaces extensive`. Wklejenie ich do ServiceNow to nie diagnostyka. Zapisz **czas, port, licznik, wartość, poprzednią wartość i hipotezę**.

**Urządzenie: EX-ACCESS-01 — Operational Mode, wybrane krótkie odczyty**

~~~junos
show interfaces ge-0/0/1 extensive
show interfaces ge-0/0/1 extensive | match "Physical interface|Link-level|Input errors|Output errors|CRC|Drops|Carrier transitions"
show log messages | match "ge-0/0/1"
~~~

Nie opieraj pełnej diagnozy wyłącznie na `match`, bo możesz wyrzucić **kontekst**, np. to, czy licznik jest `input` czy `output`, czy dotyczy fizycznego portu czy logical unit. Najpierw przeczytaj całość; dopiero później filtruj do worknotes.

**Źródła techniczne:** [Junos operational monitoring](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-operational-mode-commands-monitoring.html), [show interfaces extensive](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-interfaces-extensive.html), [show system processes](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-processes.html).

---

## 5.2. Polecenia `monitor` — obserwuj zjawisko w czasie, a nie tylko fotografię [A]

### 5.2.1. Dlaczego snapshot nie odpowiada na pytanie „czy problem trwa teraz?”

Wczoraj port miał 1234 błędy CRC. Dziś `show interfaces extensive` nadal pokazuje 1234. Czy link wciąż jest uszkodzony? **Nie wiesz.** Licznik mógł narastać dwa tygodnie temu i od tego czasu się nie zmieniać. Potrzebujesz dwóch pomiarów z odstępem, tempa wzrostu i korelacji z ruchem.

Pomyśl o liczniku kilometrów samochodu. Wskazanie 150 000 km nie dowodzi, że samochód teraz jedzie 150 km/h. Statystyki kumulacyjne mówią „ile dotychczas”, a odczyty z czasem pozwalają policzyć „jak szybko obecnie”.

W Junos `monitor interface` oraz inne funkcje `monitor` potrafią pokazywać zmiany w czasie bez wielokrotnego uruchamiania `show`.

### 5.2.2. `monitor interface` vs `monitor interface traffic`

**Urządzenie: EX-ACCESS-01 — Operational Mode, obserwacja portu**

~~~junos
monitor interface ge-0/0/1
~~~

W odpowiedniej wersji i na wspieranym terminalu polecenie prezentuje aktualizowany obraz parametrów portu; sposób wyjścia i skróty klawiaturowe są wyświetlane na ekranie lub opisane w CLI Reference. Monitorowanie interaktywne zależy od trybu sesji i platformy.

**Urządzenie: EX-ACCESS-01 — Operational Mode, obserwacja przepływności wielu portów**

~~~junos
monitor interface traffic
~~~

**Most z Cisco:** można to porównać do cyklicznego `show interfaces counters`, narzędzia telemetry/monitoring albo `watch` w Linuxie. Ale pamiętaj, że **odświeżony ekran nie jest wiecznie odłożoną historią**. Do długoterminowego trendu używaj NMS/SNMP/telemetrii z rozdziału 4, a nie terminala pozostawionego otwartego przez tydzień.

**Gotcha platformowa:** Juniper dokumentuje, że na **Junos OS Evolved** `monitor interface` można używać w sesjach SSH, lecz pewne tryby terminala console/Telnet nie są wspierane. To pokazuje, dlaczego nie uczymy się jedynie nazwy komendy, lecz także platformy. [monitor interface](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-interface.html).

### 5.2.3. `monitor start` — log na żywo, gdy problem występuje tylko przez 10 sekund

Załóżmy, że interfejs flappuje raz na godzinę. Możesz wykonywać `show log messages | last 20` co 30 sekund. Wygodniej obserwować dopisywane wpisy logu na żywo.

**Urządzenie: R1-vSRX — Operational Mode, monitoring logu**

~~~junos
monitor start messages
~~~

W tym czasie wykonuj inne odczyty CLI lub obserwuj zdarzenia. Po zakończeniu **zatrzymaj monitor**, aby nie zalać sesji:

**Urządzenie: R1-vSRX — Operational Mode, zamknięcie podglądu live**

~~~junos
monitor stop messages
~~~

Pamiętaj, że polecenie `monitor start` wyświetla dane z **istniejącego pliku logu**; to nie gwarancja, że system loguje akurat interesujące Cię zdarzenie. Dla konkretnego protokołu może być potrzebna dedykowana konfiguracja logowania/traceoptions — zgodnie z 4.10 i 4.12.

**Gotcha:** `monitor start messages` i `show log messages` to dwie różne czynności: pierwsza śledzi dopisywane logi, druga czyta zgromadzony plik. **`monitor stop` nie usuwa logów**; kończy wyświetlanie.

### 5.2.4. `monitor traffic` — największa pułapka dla kogoś, kto zna SPAN i tcpdump

W Cisco możesz kojarzyć `monitor session` (SPAN) jako kopię ruchu z fizycznych portów. W Junos polecenie **`monitor traffic interface ...`** wygląda podobnie do tcpdump, ale **standardowo działa z perspektywy Routing Engine i widzi ruch dostępny dla tego mechanizmu**, niekoniecznie każdy pakiet tranzytowy przetwarzany sprzętowo przez PFE.

Wyobraź sobie magazyn. Pracownik w recepcji widzi ludzi wchodzących do biura, ale nie widzi każdej palety przechodzącej przenośnikiem na hali. Jeśli recepcja nie widziała palety, nie znaczy to, że paleta nie jechała. Analogicznie RE-capture nie zastępuje pełnego SPAN/analyzera dataplane.

~~~mermaid
flowchart LR
  IN["Ruch z klienta"] --> PFE["PFE / dataplane"]
  PFE -->|transit forwarded| OUT["Ruch do serwera"]
  PFE -->|host-bound / wyjątki| RE["RE i control plane"]
  RE --> CAP["monitor traffic\nwidok RE"]
  PFE -. "nie wszystkie pakiety transit" .-> CAP
~~~

**Urządzenie: R1-vSRX — Operational Mode, ograniczony capture w izolowanym LAB**

~~~junos
monitor traffic interface ge-0/0/1 no-resolve count 20
~~~

Opcja `count 20` ogranicza liczbę pakietów, a `no-resolve` ogranicza rozwiązywanie nazw. Bardziej szczegółowe filtry i składnia `matching` wymagają dobrania do wydania. Na vSRX pakiety w wirtualnym dataplane i host-bound mają osobne uwarunkowania — przed wyciągnięciem wniosku **sprawdź dokumentację danego obrazu**.

**Pułapka egzaminacyjna:** jeśli `monitor traffic` nie pokazuje ruchu klient–serwer, **nie możesz powiedzieć, że Junos nie forwarduje tego ruchu**. Na SRX dodatkowo sprawdzasz `show security flow session`; na EX/QFX fizyczny SPAN/analyzer i telemetrię, zależnie od zadania. Źródło: [monitor traffic](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html).

### 5.2.5. Kiedy monitorowanie jest ryzykowne?

Szerokie `monitor traffic` bez filtra, odczyty co ułamek sekundy, `traceoptions flag all detail` i ogromne logi mogą zwiększać obciążenie RE oraz zapełniać terminal lub storage. **Na produkcyjnym core nie rozpoczynaj szerokiego zrzutu bez wiedzy, jakie zasoby obciąży.**

Najpierw uzgodnij **co jest celem** (konkretny klient, uplink, port, protokół), następnie wybierz **najmniejszy zakres** i limit czasu/danych. Zapisz w worknotes, co monitorowałeś i kiedy.

**Na pamięć:** `monitor interface` ≠ `monitor traffic`; `monitor start/stop` dotyczy śledzenia logu.  
**Do zrozumienia:** RE capture ≠ ruch transit PFE.  
**Do przećwiczenia:** porównaj w labie ping do IP samego R1 z ruchem pomiędzy dwoma VPCS; sprawdź, co `monitor traffic` rzeczywiście widzi.

