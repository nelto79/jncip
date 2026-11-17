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



---

## 5.3. Statystyki interfejsów — jak rozpoznać awarię kabla, przeciążenie i problem z optyką [A]

### 5.3.1. „Interfejs up/up, więc to nie LAN” — błąd, który kosztuje godziny

Telefon z fabryki. „Aplikacja działa powoli tylko w budynku B. Port uplink jest up/up, więc proszę eskalować do serwera”. To klasyczny błąd myślowy. Interfejs Ethernet może mieć poprawny link, a mimo to gubić ramki z powodu uszkodzonego światłowodu, wadliwej wkładki SFP, przeciążenia egress queue lub niedopasowanej konfiguracji.

Musimy oddzielić cztery rodzaje sygnału:
1. **Stan linku** — czy urządzenie widzi carrier i administracyjne włączenie?
2. **Błędy transmisji** — czy ramki dochodzą uszkodzone?
3. **Odrzucenia i kolejki** — czy prawidłowe pakiety są gubione z powodu buforów/polityki?
4. **Trendy** — czy problem **narasta teraz**, czy licznik jest stary?

~~~mermaid
flowchart TD
  SYM["Wolna aplikacja, uplink UP"] --> PHY{"CRC/FCS lub carrier transitions rosną?"}
  PHY -->|Tak| L1["Podejrzenie L1:\npatch, SFP, światłowód, peer"]
  PHY -->|Nie| Q{"Drops/egress queue rosną?"}
  Q -->|Tak| CONG["Congestion, oversubscription, CoS"]
  Q -->|Nie| PATH["Sprawdź VLAN/L3,\nSRX policy, serwer i aplikację"]
  L1 --> CHANGE["Potwierdź także licznik na drugim końcu"]
  CONG --> CHANGE
  PATH --> CHANGE
  CHANGE --> TEST["Re-test klienta + trend counters"]
~~~

**Most z Cisco:** `show interfaces Gi1/0/24` i liczniki input errors/CRC/output drops to znane narzędzia. Na Junos zrobisz podobnie, ale interpretacja nazw liczników może zależeć od **rodziny PFE, platformy i portu**. Oficjalny przewodnik producenta wyraźnie rozróżnia input errors, resource errors, carrier transitions i drop statistics. [Understanding Dropped Packets](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/using-show-commands-for-packet-drops.html).

### 5.3.2. Stosujesz zawężanie problemu od fizyki do ruchu aplikacyjnego

**Urządzenie: EX-ACCESS-01 — Operational Mode, start diagnozy uplinku**

~~~junos
show interfaces terse | match ge-0/0/47
show interfaces descriptions | match ge-0/0/47
show interfaces ge-0/0/47 extensive
show log messages | match "ge-0/0/47"
~~~

Pomyśl, co tak naprawdę czytasz. Jeśli na `ge-0/0/47` masz kilka tysięcy input errors, nie możesz jeszcze stwierdzić, czy pojawiły się w czasie dzisiejszego incydentu. Zrób pierwszy pomiar wraz z czasem, potem drugi po np. 60 sekundach **normalnego obciążenia**; policz zmianę liczników. Pamiętaj o możliwości resetów liczników po reboot, flapie, zmianie karty lub `clear interfaces statistics`.

**Przykład ilustracyjny — NIE rzeczywisty output Junos:**

| Pomiar | CRC / input errors | Egress drops |
|---|---:|---:|
| 10:00:00 | 12 400 | 1 500 |
| 10:01:00 | 12 580 | 1 500 |
| Przyrost przez 60 s | **+180** | 0 |

Interpretacja: w tej minucie rosną błędy input. Nie ma dowodu, że egress congestion jest główną przyczyną. To uzasadnia weryfikację L1 i licznika **na porcie peer**.

**Gotcha:** nawet szybki przyrost CRC **nie pozwala automatycznie wskazać winnego komponentu**. Uszkodzenie może być w kablu, złączu, nadajniku zdalnej strony, odbiorniku lokalnej strony albo w torze optycznym. Potrzebujesz testu krzyżowego lub porównania z peer.

### 5.3.3. Liczniki, które musisz rozumieć, nie tylko kojarzyć po nazwie

**Input errors / CRC / FCS:** uszkodzenie odebranych ramek. Uwaga: konkretne kategorie błędów i sposób sumowania zależą od platformy; nie każdy licznik jest prostą sumą pozostałych.

**Carrier transitions:** zmiany stanu carrier. Wzrost razem z flap events wzmacnia hipotezę niestabilnej warstwy fizycznej, ale sam licznik bywa kumulowany od startu.

**Output drops:** pakiety odrzucone przy transmisji. Mogą wskazywać problem kolejkowania, ale **nie zawsze** są dowodem przeciążenia fizycznego portu. Niektóre liczniki obejmują specyficzne dla PFE odrzucenia i wymagają analizy egress queues / CoS.

**Framing/MTU:** duże pakiety, drops związane z MTU, błędy nadmiarowej długości. Przy tunelach/VPN asymetria MTU potrafi zepsuć wybrane aplikacje, choć ping 64 bajty działa.

**FEC/optical errors:** na wyższych prędkościach i transceiverach możesz spotkać dodatkowe pomiary BER, FEC i sygnału optycznego. Nie interpretuj wszystkich liczników jednakowo na ge/xe/et i w wirtualnym vSRX.

### 5.3.4. Digital Optical Monitoring (DOM) — co naprawdę oznacza -18 dBm?

Wyobraź sobie link 10G przez dwie szafy i patch panel. Na Juniperze odczytujesz **Rx optical power -18 dBm**. Czy to dobry sygnał? Nie odpowiadaj bez informacji o module i jego dopuszczalnym Rx range. Inny transceiver może mieć progi warning/alarm w zupełnie innym zakresie.

**Urządzenie: EX-DIST-01 — Operational Mode, wyłącznie sprzęt z optyką obsługującą DOM**

~~~junos
show chassis hardware
show interfaces diagnostics optics xe-0/0/0
show interfaces xe-0/0/0 extensive
~~~

DOM zazwyczaj pokazuje temperaturę, napięcie, Tx laser output power, Rx optical power oraz alarmy i warning thresholds. **Nie myl Tx i Rx**: Rx mówi, jaki sygnał dociera od nadajnika po drugiej stronie; Tx mówi, co lokalny moduł wysyła.

**Gotcha:** `show interfaces diagnostics optics` może nie być dostępne na danym portcie lub module; wirtualny vSRX **nie ma fizycznego lasera**, więc ten lab wymaga prawdziwego EX/QFX/SRX z odpowiednią wkładką lub analizy udokumentowanego, wyraźnie oznaczonego outputu przykładowego. [Digital Optical Monitoring](https://www.juniper.net/documentation/us/en/software/junos/chassis/topics/concept/digital-optical-monitoring.html).

### 5.3.5. Dwa końce światłowodu — eksperyment myślowy

**Kierunek A → B:** nadajnik **A Tx** → włókno → odbiornik **B Rx**.  
**Kierunek B → A:** nadajnik **B Tx** → drugie włókno → odbiornik **A Rx**.

~~~mermaid
flowchart LR
  subgraph A["Switch A"]
    ATX["A Tx"]
    ARX["A Rx"]
  end
  subgraph B["Switch B"]
    BTX["B Tx"]
    BRX["B Rx"]
  end
  ATX -->|"światłowód A→B"| BRX
  BTX -->|"światłowód B→A"| ARX
~~~

Jeśli **A Rx** widzi bardzo słaby sygnał, to nie jest bezpośrednia informacja o **A Tx**. Weryfikuj nadajnik B, tor B→A, budżet optyczny i zabrudzone złącza. Przy modułach BiDi/WDM może być **jedno włókno i różne długości fali** — diagram klasycznej pary włókien nie opisuje wtedy całej architektury.

Właśnie dlatego technik onsite powinien przekazać **model SFP, typ medium, długość linku i DOM z obu stron**, a nie tylko stwierdzenie „sygnał jest niski”.

### 5.3.6. `clear interfaces statistics` — test, który może zniszczyć dowód incydentu

Kiedy licznik jest stary, możesz chcieć go wyzerować, żeby zobaczyć przyrost od zera. To wygodne, ale w środowisku produkcyjnym **zmieniasz bazę odniesienia także dla innych operatorów, telemetry lub raportu**. Nie wykonuj `clear` w P1 bez wcześniejszego snapshotu i uzgodnienia.

**Urządzenie: EX-LAB-01 — Operational Mode, wyłącznie izolowany LAB po zapisaniu counters**

~~~junos
clear interfaces statistics ge-0/0/1
~~~

Preferowana metoda w produkcji to **dwa odczyty i delta**, bez kasowania liczników. Juniper dokumentuje też szczególne zachowania clear przy failover RE i na różnych FPC — nie zakładaj, że reset w jednej jednostce zeruje wszystkie kopie counterów. [clear interfaces statistics](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/clear-interfaces-statistics.html).

### 5.3.7. Jakie dowody w SNOW/ServiceNow, a jakie do LLD?

Dla incydentu z linkiem:
- Oba hostnames i interfejsy wraz z opisami.
- Timestamp **w tej samej strefie czasowej** (NTP).
- Status admin/link i czas ostatniej zmiany.
- Delta CRC/drops/carrier transitions dla interwału.
- DOM obu stron, model i typ wkładki.
- Fizyczny przebieg (patchpanel/ODF), wyniki testu onsite.
- Rekomendacja: wymiana patchcordu/SFP, przeniesienie portu, kontrola uplinku lub zmiana QoS — **dopiero po dowodach**.

**Na pamięć:** `show interfaces ... extensive`, `diagnostics optics`, `monitor interface`.  
**Do zrozumienia:** CRC ≠ output drops, stan `up` ≠ brak problemu, Rx ≠ Tx.  
**Do przećwiczenia:** zrób dwa odczyty licznika, policz deltę i przedstaw hipotezę bez niepotrzebnego `clear`.

---

## 5.4. Narzędzia sieciowe — ping, traceroute, telnet i SSH jako eksperymenty diagnostyczne [A]

### 5.4.1. „Ping działa” to nie pełna definicja osiągalności

Klient zgłasza: tylko użytkownicy na WLAN nie mogą otworzyć serwera ERP, a ten sam serwer odpowiada z LAN. Operator uruchamia ping **z routera** i ogłasza „sieć działa”. Ale pakiet z routera ma inny **source IP, routing instance, politykę bezpieczeństwa, drogę w sieci i czasem nawet inny typ ruchu** niż pakiet z klienta WLAN.

To nie problem Junos. To problem metodologii diagnostycznej. Dobry inżynier testuje **hipotezę, a nie tylko narzędzie**.

**Most z Cisco:** `ping vrf MGMT source Loopback0` i extended ping to mechanizm znany z IOS XE. W Junos użyj `source`, `routing-instance` i odpowiednich opcji ping. Zwróć uwagę na to, czy test jest dostępny na danej platformie i czy source istnieje w danej tablicy.

### 5.4.2. Cztery różne testy ICMP, cztery różne pytania

**Urządzenie: R1-vSRX — Operational Mode, odizolowany LAB z istniejącą trasą do 198.51.100.2**

~~~junos
ping 198.51.100.2 count 5
ping 198.51.100.2 source 198.51.100.1 count 5
ping 198.51.100.2 size 1400 count 5
show route 198.51.100.2
~~~

1. **Zwykły ping:** czy RE może wysłać/odebrać ICMP do celu używając wybranej drogi i source?
2. **Ping z source:** czy działa **konkretny adres źródłowy**, który chcesz zbadać?
3. **Duższy ping:** czy większe datagramy też dochodzą? Nie jest to automatycznie pełny test PMTUD/MTU — bez odpowiedniego DF pakiety IPv4 mogą być fragmentowane.
4. **Routing:** czy masz trasę i jaki jest next-hop w właściwej tablicy?

**Haczyk:** dla IPv4 w dokumentacji `size` odnosi się do rozmiaru danych ICMP (zgodnie z daną implementacją CLI), a nie całkowitej długości ramki Ethernet. Jeśli testujesz MTU, zweryfikuj **jak Junos liczy size dla IPv4/IPv6** i czy ustawiono `do-not-fragment` na wspieranej platformie. [ping CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/ping.html).

### 5.4.3. Nie mieszaj `mgmt_junos` z `inet.0`

Załóżmy, że router ma dedykowany port management i tablicę `mgmt_junos.inet.0`. Zwykły ping do `192.0.2.30` może podjąć próbę w `inet.0`, gdzie drogi do serwera nie ma. Nie oznacza to automatycznie awarii kabla OOB.

**Urządzenie: LAB-SRX01 — Operational Mode, tylko jeśli `mgmt_junos` istnieje**

~~~junos
show route table mgmt_junos.inet.0
ping 192.0.2.30 routing-instance mgmt_junos count 5
~~~

Jeśli instancji nie ma, polecenie zwróci błąd — **nie twórz jej dla samego ćwiczenia bez projektu management**. Pamiętaj, że `routing-instance` jest wyborem tablicy routingu, a `source` — lokalnego adresu pakietu. To dwa różne pytania.

### 5.4.4. Traceroute: brak odpowiedzi jednego hopa nie dowodzi, że ruch tam ginie

Każdy zna traceroute z gwiazdkami:
- Hop 1 odpowiada.
- Hop 2 pokazuje `*`.
- Hop 3 odpowiada i aplikacja działa.

Czy hop 2 „przestał forwardować”? Nie. Urządzenie mogło ograniczyć ICMP Time Exceeded, zablokować odpowiedzi z control plane, a sam **forwarding transit** działał. To klasyczny przypadek RE/PFE separation z rozdziału 2.

**Urządzenie: R1-vSRX — Operational Mode, lab z trasą do zdalnego hosta**

~~~junos
traceroute 203.0.113.2
traceroute 203.0.113.2 no-resolve
show route 203.0.113.2
~~~

Traktuj traceroute jako **wskazówkę o drodze i odpowiedziach urządzeń**, nie ostateczny dowód ścieżki biznesowej. Różnice per-hop mogą wynikać z ECMP, asymetrycznej trasy powrotnej, MPLS/tuneli, filtrów lub rodzaju probes.

**Most z Cisco:** gwiazdka `*` w `traceroute` na IOS XE nie znaczy „ruter dropuje każdą ramkę użytkownika”. W Junos dokładnie tak samo. [traceroute CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/traceroute.html).

### 5.4.5. Telnet nie jest zalecaną metodą administracji, ale narzędziem diagnostycznym bywa przydatny

Blueprint wymienia **telnet i SSH** jako narzędzia. Rozróżnijmy dwie rzeczy:
- **Telnet do administrowania siecią** wysyła dane uwierzytelniające bez ochrony porównywalnej z SSH i nie powinien być domyślnie aktywną usługą management nowego site.
- **Wywołanie klienta `telnet host port`** może być prostym testem zestawienia TCP do określonego portu, o ile CLI i release wspierają taką składnię.

**Urządzenie: LAB-SRX01 — Operational Mode, wyłącznie autoryzowany test do laboratoryjnego listenera TCP/443**

~~~junos
telnet 192.0.2.30 port 443
~~~

To sprawdza możliwość **zestawienia TCP**, nie dowodzi poprawności TLS, certyfikatu ani działania aplikacji HTTP. Zwykły `telnet` na port SSH może dać banner, ale nie jest klientem bezpiecznej sesji SSH. **Nie wpisuj tam haseł do usług produkcyjnych.**

Na niektórych platformach dostępność `telnet` klienta może być ograniczona. Wtedy użyj zatwierdzonego narzędzia z klienta Linux/Windows. Nie włączaj serwera Telnet na Junos tylko dlatego, że chcesz wykonać test TCP do innego hosta.

### 5.4.6. SSH jako test, a nie tylko kanał administracyjny

Jeżeli możesz zalogować się z bastiona na EX, to dowód, że masz **transport TCP/22 + SSH negotiation + authorization użytkownika** z tego miejsca. Ale jeśli ten sam EX nie może zalogować się na serwer kopii zapasowych, może to wskazywać inne source IP, VRF, ACL i host keys.

**Urządzenie: LAB-SRX01 — Operational Mode, test inicjowanego przez urządzenie SSH do laboratoryjnego serwera**

~~~junos
ssh labuser@192.0.2.30
~~~

Nie wpisuj w publiczny ticket hasła ani fingerprintu wyciągniętego z niezaufanego źródła. Połączenie SSH do hosta musi przejść prawidłową weryfikację jego tożsamości.

**Urządzenie: LAB-ADMIN — Linux shell, niezależny test SSH W KIERUNKU urządzenia**

~~~bash
ssh labadmin@192.0.2.10
~~~

**Pytanie:** dlaczego oba testy są potrzebne? Bo **kierunki i źródła połączeń są inne**. Firewall lub routing może przepuszczać jeden, a blokować drugi.

### 5.4.7. Test z klienta: Windows i Linux

Czasem urządzenie Junos jest tylko częścią ścieżki. Musisz sprawdzić, czy usterka dotyczy jednego NIC, DNS, WLAN czy IP routing.

**Urządzenie: PC-CLIENT-WIN — Windows PowerShell**

~~~powershell
ipconfig /all
route print -4
Test-NetConnection 203.0.113.20 -Port 443 -InformationLevel Detailed
tracert -d 203.0.113.20
~~~

**Urządzenie: PC-CLIENT-LINUX — Linux shell**

~~~bash
ip -br address
ip route get 203.0.113.20
traceroute -n 203.0.113.20
~~~

To narzędzia klienta, nie komendy Junos. Kluczowy jest **porównywalny zakres**: jeśli problem dotyczy WLAN, wykonaj test ze stacji na WLAN, z rzeczywistym source IP, nie wyłącznie z routera management.

### 5.4.8. Szkic decyzyjny: czy brak ping jest problemem L3?

~~~mermaid
flowchart TD
  FAIL["Ping do celu FAIL"] --> SRC{"Czy source i routing-instance\nodpowiadają klientowi?"}
  SRC -->|Nie| RETEST["Popraw test z właściwego źródła"]
  SRC -->|Tak| RT{"Czy aktywna trasa i next-hop istnieją?"}
  RT -->|Nie| ROUTE["Sprawdź RIB/FIB, routing i ARP"]
  RT -->|Tak| FILTER{"Czy ICMP może być blokowany?"}
  FILTER -->|Tak| TCP["Sprawdź usługę TCP / policy / sesję"]
  FILTER -->|Nie| PATH["L1/L2, return path, serwer, capture"]
  RETEST --> VERIFY["Test ponowny i wynik klienta"]
  ROUTE --> VERIFY
  TCP --> VERIFY
  PATH --> VERIFY
~~~

**Na pamięć:** `ping ... source`, `routing-instance`, `traceroute`, `ssh`, `telnet ... port`.  
**Do zrozumienia:** działający ICMP z RE nie potwierdza aplikacji ani drogi klienta.  
**Do przećwiczenia:** uruchom dwa ping z różnymi source i porównaj z pomiarem klient–serwer w izolowanym labie.

