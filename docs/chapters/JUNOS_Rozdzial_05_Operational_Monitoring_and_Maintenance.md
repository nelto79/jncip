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



---

## 5.5. Instalacja i aktualizacja Junos OS — change, który może skończyć się utratą całego site'u [A]

### 5.5.1. „Mamy CVE. Aktualizujcie wszystkie Junipery do najnowszego” — dlaczego nie zaczynamy od `software add`

Piątek, 16:30. Zespół Security publikuje pilne advisory. Kilkadziesiąt EX, SRX i MX wymaga oceny. W środowisku Cisco znasz matrix IOS XE, model, ROMMON, boot mode, stack/HA i okna serwisowe. W Juniperze postępujesz **dokładnie tak samo świadomie**, ale masz dodatkowe różnice między klasycznym Junos, Junos OS Evolved, poszczególnymi family image i mechanizmami dual-root, snapshot i switchover.

Nie ma **jednego pakietu Junos do wszystkiego**. Nie ma też jednej magicznej komendy, której użyjesz identycznie na EX4400, MX204, QFX5k, SRX300 i nowym urządzeniu Evolved. To jest najważniejsze zdanie o patchowaniu, którego często brakuje w podręcznikach Associate.

### 5.5.2. Co jest właściwym celem patchowania?

Celem nie jest „mieć najwyższy numer wersji”, lecz:
- usunąć rzeczywistą podatność lub błąd;
- zachować kompatybilność hardware i feature;
- przeprowadzić **wspieraną ścieżkę upgrade**;
- utrzymać możliwość przywrócenia poprzedniego stanu;
- zminimalizować wpływ na użytkownika i dostarczyć powtarzalne dowody.

**Most z Cisco:** przy Catalyst 9300 weryfikujesz wersję IOS XE, boot variables, INSTALL mode, stack members i TAC advisories. W Junos sprawdzasz model, obecny/planowany release, obraz właściwej rodziny, boot media, storage, RE redundancy, HA cluster/Virtual Chassis i software installation guide.

Zawsze szukaj **JTAC Suggested Releases (KB21476)**, bieżących Security Advisories (JSA), Release Notes, Feature Explorer, End-of-Engineering/End-of-Support, exact supported path. **Nie zapisujemy jednego release 2026 jako dobrego dla wszystkich urządzeń**, bo byłaby to fałszywa informacja projektowa.

### 5.5.3. Rozdziel trzy rzeczy: konfigurację, system operacyjny i obraz zapasowy

W rozdziale 3 `rollback 1` cofał **konfigurację** w candidate. Teraz musisz znać co najmniej trzy odmienne rodzaje przywracania:

1. **Configuration rollback:** `rollback 1`, `rollback rescue` → candidate → commit.
2. **Software rollback:** procedura przywracająca **poprzedni Junos image**, zależna od platformy i dostępnego backupu.
3. **Root/boot snapshot:** kopia systemu na wspieranym alternatywnym nośniku/partycji; jej dokładna forma jest platform-dependent.

~~~mermaid
flowchart TD
  REQ["Need to revert after incident"] --> Q{"Co zostało zmienione?"}
  Q -->|config/ACL/hostname| CFG["rollback n / rescue\npotem compare + commit"]
  Q -->|Junos release| SW["Software rollback\nplatform + version matrix"]
  Q -->|boot media corruption| BOOT["Recovery snapshot / alternate partition\nconsole + recovery procedure"]
  CFG --> TEST["Service verification"]
  SW --> TEST
  BOOT --> TEST
~~~

**Podchwytliwe pytanie:** czy `commit confirmed 5` ochroni Cię przed nieudaną instalacją Junos? **Nie.** Confirmed commit zarządza **konfiguracją**, a upgrade wymienia **system software**. Możesz wykonać poprawny commit i nadal mieć niebootujący obraz po upgrade. Recovery wymaga właściwego snapshotu, obrazu i dostępu console.

### 5.5.4. Zidentyfikuj wszystko, co determinuje procedurę

**Urządzenie: EX-ACCESS-01 — Operational Mode, precheck przed ustaleniem upgrade path**

~~~junos
show version
show chassis hardware
show chassis alarms
show system alarms
show system uptime
show system storage
show system processes extensive
show system users
show system commit
show configuration | display set
~~~

Dopiero teraz pytasz o:
- aktualne pełne wydanie, np. R/S i rodzinę obrazu;
- Model/PID, RE/FPC/members;
- czy VC/cluster ma kilka członków i jak wygląda redundancja;
- czy boot i root partitions mają miejsce;
- czy obecny software obsługuje `request system software add` i jaki plik;
- czy istnieje aktywna niezatwierdzona zmiana;
- czy zespół utrzymaniowy ma **OOB console**, gdy management przez SSH przestanie działać.

**Urządzenie: EX-ACCESS-01 — Operational Mode, dodatkowa weryfikacja sprzętowego VC, jeśli taka topologia istnieje**

~~~junos
show virtual-chassis
show virtual-chassis status
~~~

**Nie każde EX pracuje jako Virtual Chassis.** Jeśli CLI zwraca brak funkcji, nie jest to automatycznie błąd urządzenia. Dla SRX chassis cluster potrzebna będzie odrębna procedura `show chassis cluster status` i często inny mechanizm upgrade; nie używaj na ślepo komend VC.

### 5.5.5. Precheck powinien zawierać plan biznesowy, a nie tylko RAM

Nie rozpoczniesz upgrade bez uzgodnionego maintenance window, klasyfikacji ryzyka i rollback planu. Zanim przejdziesz dalej, zadaj pytania:

**Co stracimy, jeśli urządzenie przestanie działać?** Jeden edge switch, piętro biurowca, core kampusu czy centralną bramę Internetu? Czy są redundantne łącza i czy aplikacja przeżyje konwergencję?

**Jak wrócimy?** Czy stara wersja jest dostępna jako obraz, na alternatywnej partycji, przez snapshot albo na zatwierdzonym repozytorium? Czy downgrade jest wspierany? Czy potrzebna będzie reinstalacja zamiast rollback?

**Jak zweryfikujemy sukces?** Nie wystarczy nowy `show version`. Sprawdzasz wszystkie znaczące funkcje: VLAN/IRB, routing, HA, monitoring, AAA, NTP, syslog, policery/firewall, optykę i aplikację klienta.

**Co z firmware i bootloaderem?** Niektóre ścieżki software install mogą również wpływać na firmware/bootrom lub wymagać określonego upgrade path. Nie zakładaj, że jest to tylko podmiana pakietu `.tgz`.

### 5.5.6. Przygotuj miejsce na dysku — nie wymazuj potrzebnych plików

**Urządzenie: EX-ACCESS-01 — Operational Mode, kontrola wolnego miejsca i symulacja cleanup**

~~~junos
show system storage
file list /var/tmp/ detail
request system storage cleanup dry-run
~~~

`dry-run` pokazuje propozycję plików do wyczyszczenia, **nie wykonuje cleanup**. To właściwy nawyk: najpierw obejrzyj listę, oceń, czy potrzebujesz logów do incydentu lub poprzedniego package, dopiero potem rozważ działanie administracyjne.

**Urządzenie: EX-ACCESS-01 — Operational Mode, działanie zmieniające storage; NIE wykonuj bez oceny dry-run i zgody**

~~~junos
request system storage cleanup
~~~

**Haczyk:** `request system storage cleanup` to nie banalny odpowiednik `delete temporary files`; może usunąć użyteczne logi/ślad diagnostyczny. Na Junos Evolved zarządzanie obrazami z poziomu `show system software list` i poleceń software różni się od klasycznego Junos. Zawsze przeczytaj [storage cleanup documentation](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-storage-cleanup.html).

### 5.5.7. Plik image — skąd go masz i skąd wiesz, że jest właściwy?

Pakiet pobierasz z oficjalnego Support Portal na podstawie modelu i zalecanego release. **Sprawdź sumę kontrolną** podaną przez producenta i cyfrowy podpis/mechanizm weryfikacji, jeśli dany format i platforma go wspierają. Nie pobieraj nieznanej binarki z forum, GitHuba czy przypadkowego mirroru.

Na EX/QFX producent zaleca umieszczenie image w **`/var/tmp`**, a w części rodzin jest to jedyna obsługiwana lokalizacja przy `software add`.

**Urządzenie: EX-ACCESS-01 — Operational Mode, tylko lokalny read-only check uprzednio dostarczonego pliku**

~~~junos
file list /var/tmp/ detail
show system storage
~~~

**Urządzenie: EX-ACCESS-01 — Operational Mode, koncept pobierania z zaufanego serwera SCP (wymaga istniejącego serwera i poprawnej ścieżki)**

~~~text
file copy scp://upgrade-user@192.0.2.50/approved-image.tgz /var/tmp/
~~~

Wiersz jest **szablonem transportu**. Nie zawiera rzeczywistego pakietu, nie zakłada istnienia serwera `192.0.2.50` i nie jest do wklejenia bez przygotowania infrastruktury.

**Gotcha z dokumentacji:** w części procedur nie należy podawać URL SCP **bezpośrednio** do `request system software add`; najpierw używa się `file copy` na `/var/tmp`, dopiero potem lokalnego install. Dokumentacja producenta wyraźnie to rozróżnia. [request system software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html).

### 5.5.8. Sam proces `request system software add`: dlaczego jest *wzorem*, a nie gotowym skryptem dla wszystkich?

**Urządzenie: EX-ACCESS-01 — Operational Mode, WYŁĄCZNIE SZABLON polecenia do zatwierdzonego change**

~~~text
request system software add /var/tmp/APPROVED-PACKAGE-FOR-THIS-MODEL.tgz
~~~

Zamiast wklejać losowy pakiet, sprawdzasz:
1. czy działa właściwe command family dla modelu;
2. czy plik faktycznie istnieje i ma zgodną sumę;
3. czy release notes potwierdzają **direct upgrade path**;
4. czy wymagane jest oddzielne `reboot`, ISSU, obraz host OS albo VMHost;
5. czy konfiguracja została zweryfikowana względem nowego release;
6. czy masz plan powrotu do starej wersji.

**Uwaga:** `request system software add` zwykle może uruchamiać walidację zgodności konfiguracji. **`no-validate`** pomija ten krok — nie powinno być domyślnym nawykiem „bo szybciej”. Ale istnieją **uzasadnione wyjątki platformowe i międzywydaniowe**, np. przy zmianie bazowego FreeBSD, w których dokumentacja wymaga `no-validate` albo innej ścieżki walidacji.

**Konkretny haczyk producenta:** dla SRX300/SRX320/SRX340/SRX345/SRX380 przy upgrade **z 23.4R2-S3 lub 24.2R2 do 24.4R1** dokumentacja wymienia specjalny wariant instalacji `no-copy no-validate reboot`. To **nie jest** ogólna recepta dla każdego SRX i każdego release. [Oficjalna składnia i wyjątki](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html).

### 5.5.9. Snapshot — ubezpieczenie, które może również skasować Twoją szansę cofnięcia

W wielu modelach klasycznego Junos można tworzyć snapshoty boot media/alternate partitions. Zależnie od platformy i wieku systemu możesz spotkać m.in. `request system snapshot`, `request system snapshot slice alternate`, `request system snapshot recovery`. Nie są zamienne.

Dobra intuicja: **przed upgrade** zachowaj znaną dobrą wersję w sposób obsługiwany przez platformę. **Po upgrade** nie nadpisuj starej kopii, dopóki nie zaakceptujesz nowego obrazu jako stabilnego. Dlaczego? Bo na niektórych mechanizmach snapshotu zapis nowego obrazu do zapasowej partycji **nadpisuje właśnie tę starą wersję, do której chciałeś wrócić**.

**Urządzenie: EX-ACCESS-01 — Operational Mode, read-only kontrola warunków snapshotu**

~~~junos
show version
show system storage
show system alarms
~~~

**Urządzenie: EX-ACCESS-01 — Operational Mode, ilustracja składni historycznie obsługiwanej na wybranych EX/SRX; NIE uruchamiaj bez potwierdzenia modelu/release**

~~~text
request system snapshot slice alternate
~~~

Nie obiecuję, że to zadziała na EX4400 ani konkretnym vSRX — na nowszych systemach i wydaniach występują inne warianty, a na Evolved odmienna architektura. [Snapshots on Junos](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/backing-up-install-using-snapshot.html), [Junos OS Evolved snapshots](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade-evo/topics/topic-map/evo-backup-snapshot.html).

**Most z Cisco:** przypomina relację boot partition/rollback package na Catalyst, ale nie zakładaj identycznego dual-root i install packages. Zaprojektuj i przetestuj software rollback **oddzielnie** od configuration rollback.

### 5.5.10. HA/VC: „chcę zrobić upgrade bez outage” — najdroższe niedopowiedzenie

Zespół architektury mówi: „Mamy dwie sztuki w HA, więc patch będzie non-impacting”. Zanim potwierdzisz, zapytaj, o jaki mechanizm chodzi:
- SRX **chassis cluster** ze stateful failover;
- EX **Virtual Chassis**;
- MX z redundantnymi **Routing Engines**;
- EVPN multihoming;
- Junos Evolved i dedykowane procedury package upgrade.

Każdy ma inne wymagania i ryzyka. **Dwie fizyczne skrzynki ≠ zero-impact upgrade.**

Dokumentacja `request system software add` zwraca uwagę, że gdy na określonych urządzeniach włączono **GRES**, zwykłe `software add` może być niedostępne, a wspierana procedura wymaga **ISSU** lub przygotowania zgodnie z instrukcją. Nie wyłączaj GRES na produkcji tylko dlatego, że instalator odmówił pracy. Wybrana ścieżka musi pochodzić z dokumentacji **konkretnej platformy/release/topologii HA**.

**Urządzenie: SRX-CLUSTER-01 — Operational Mode, tylko jeśli rzeczywiście jest SRX chassis cluster**

~~~junos
show chassis cluster status
show chassis cluster interfaces
show version
~~~

**Urządzenie: MX-CORE-01 — Operational Mode, tylko system z odpowiednią redundancją RE**

~~~junos
show chassis routing-engine
show system switchover
show version
~~~

**Urządzenie: EX-VC-01 — Operational Mode, tylko Virtual Chassis**

~~~junos
show virtual-chassis status
show version
~~~

Weryfikuj rzeczywistą redundancję i **stan obu stron**, nie wyłącznie ogólny `show version`.

### 5.5.11. Software rollback ≠ `rollback 1` (drugi ważny haczyk)

Załóżmy, że upgrade do nowego Junos zakończył się, ale routing feature przestał działać. Czy wpisanie `rollback 1` przywróci stary obraz? **Nie.** Przywróci poprzednią **konfigurację**, a nadal będziesz uruchamiać nowy OS.

Dla klasycznego Junos możesz potrzebować `request system software rollback` albo zgodnej procedury reinstalacji poprzedniego pakietu. **Nie każdy downgrade jest wspierany w obie strony**; niektóre przejścia powodują nieodwracalne migracje plików lub wymagają recovery image. Na Junos OS Evolved istnieje osobny wariant software rollback i komendy `show system software list` określające obrazy dostępne do powrotu.

**Urządzenie: EX-ACCESS-01 — Operational Mode, read-only sprawdzenie po zmianie**

~~~junos
show version
show system commit
show system storage
show system alarms
~~~

Nie podaję ślepego `request system software rollback` jako polecenia do wykonania na wszystkich modelach. Najpierw potwierdź support matrix i dostępność rollback image. [Software rollback on Junos](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-rollback.html), [Software rollback on Junos Evolved](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-rollback-evo.html).

### 5.5.12. Postcheck: co znaczy „upgrade zakończył się sukcesem”?

Urządzenie odpowiada przez SSH i pokazuje nową wersję. To **warunek konieczny, ale niewystarczający**. Dla Enterprise campus sprawdź minimum:

**Urządzenie: EX-ACCESS-01 — Operational Mode, po aktualizacji i przywróceniu dostępu**

~~~junos
show version
show system uptime
show chassis hardware
show chassis alarms
show system alarms
show interfaces terse
show interfaces descriptions
show route summary
show lldp neighbors
show system commit
show log messages | last 100
~~~

Potem w zależności od roli:
- na EX: VC, VLAN/IRB, porty access, PoE i LLDP, STP/LACP, uplinki, wireless AP;
- na SRX: routing, security zones, session table, IPsec/VPN/HA;
- na MX: peerings i forwarding, RE/PFE, dynamic routing, interfejsy, MPLS (jeśli używasz);
- na wszystkich: SSH/AAA, NTP, syslog, SNMPv3, archiwizacja oraz **test aplikacji z klienta**.

**Precheck i postcheck powinny używać porównywalnych poleceń, czasu i interfejsów**, inaczej nie będziesz wiedzieć, co właściwie zmieniło się po patchu.

### 5.5.13. Etapowy runbook patchowania — od advisory do closure

~~~mermaid
flowchart TD
  ADV["CVE / bug / compliance"] --> INVENTORY["Model, Junos, features, topology HA"]
  INVENTORY --> MATRIX["Support matrix / JTAC release / release notes"]
  MATRIX --> APPROVE["Change approval + impact + rollback plan"]
  APPROVE --> PRE["Precheck + off-device backup + OOB"]
  PRE --> PKG["Image: exact family + checksum + storage"]
  PKG --> METHOD{"Upgrade mode?"}
  METHOD -->|single RE| ONE["Platform-specific software add"]
  METHOD -->|VC/cluster/redundant RE| HA["Vendor documented HA/ISSU procedure"]
  ONE --> VERIFY["Reboot/activation + postcheck"]
  HA --> VERIFY
  VERIFY --> PASS{"Services PASS?"}
  PASS -->|TAK| CLOSE["Handover + monitoring + closure"]
  PASS -->|NIE| RECOVER["Supported software recovery / rollback"]
  RECOVER --> VERIFY
~~~

**Czytelnik ma wiedzieć przed egzaminem:** `request system software add` służy instalacji oprogramowania, ale w prawdziwej pracy **najważniejsza jest decyzja, czy można użyć tej ścieżki na danym modelu**. Nie oceniaj wydania tylko po największej liczbie.

**Źródła rozdziału:** [Software Installation and Upgrade Overview](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/software-install-and-upgrade-overview.html), [Prepare for Software Installation](https://www.juniper.net/documentation/us/en/software/junos/junos-install-upgrade/topics/topic-map/prepare-sotware-install-and-upgrade.html), [request system software add](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add.html), [Evolved installation](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-software-add-evo.html).



---

## 5.6. Uruchamianie, reboot i bezpieczne wyłączenie urządzenia [A]

### 5.6.1. „Zrestartuję switch, może pomoże” — dlaczego reboot nie jest diagnozą

Wyobraź sobie P1 w zakładzie produkcyjnym. Po godzinie użytkownicy nie mają dostępu do aplikacji, inżynier widzi pojedynczy alarm i ktoś proponuje reboot core switcha. To kuszące, bo może „wyczyścić stan”. Ale w ten sposób można:
- utracić dowody bieżącej usterki i liczniki;
- wywołać outage większy niż pierwotny;
- utrudnić root-cause analysis;
- wywołać niekontrolowany failover HA;
- doprowadzić do problemu boot, jeśli obraz lub storage są uszkodzone.

**Most z Cisco:** `reload` na Catalyst/ISR robi dokładnie tego typu zmianę w dostępności. Junos `request system reboot` nie jest magicznie bezpieczniejszy; wymaga change window i właściwego wpływu na zależne site'y.

### 5.6.2. Rozróżnij `reboot`, `halt` i fizyczne odcięcie prądu

**Reboot** uruchamia kontrolowane zatrzymanie usług i ponowne uruchomienie oprogramowania/urządzenia zgodnie z platformą.

**Halt** zatrzymuje system operacyjny. Nie oznacza to, że każda platforma sama odłączy zasilanie w identyczny sposób. Przy wyłączeniu hardware trzeba stosować odpowiedni Hardware Guide i procedury zasilania.

**Power cycle** to fizyczne odcięcie i przywrócenie zasilania. Nie jest domyślnym sposobem utrzymania, bo niesie większe ryzyko uszkodzenia filesystem niż graceful shutdown.

**Urządzenie: LAB-SRX01 — Operational Mode, odczyt przed każdym planowanym reboot**

~~~junos
show system uptime
show system users
show system commit
show system alarms
show chassis alarms
show system storage
show log messages | last 40
~~~

Sprawdź także aktywne sesje klientów, stan chassis cluster, Virtual Chassis, routing peers i dependency. W prologu działający SSH nie dowodzi, że możesz bezpiecznie wyłączyć WAN gateway dla całej lokalizacji.

### 5.6.3. Komenda, którą trzeba znać, lecz nie wolno wykonać bez planu

**Urządzenie: LAB-SRX01 — Operational Mode, SYNTAX ONLY — restart całego systemu, potencjalnie destrukcyjny**

~~~junos
request system reboot
~~~

Na wielu urządzeniach CLI prosi o potwierdzenie. **Nie traktuj tego jako gwarancji**, że operator przypadkowo niczego nie przerwie — polecenie może mieć warianty z natychmiastowym restartem. Na EX/QFX w pewnych wersjach `request system reboot` uruchamia domyślnie niewielkie opóźnienie, a `at now` restartuje od razu. [request system reboot](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-reboot.html).

**Urządzenie: LAB-SRX01 — Operational Mode, SYNTAX ONLY — zaplanowany restart w środowisku laboratoryjnym**

~~~junos
request system reboot in 15
~~~

Nie wykonuj tego w produkcji bez akceptacji change. Jeżeli zaplanowano reboot, dowiedz się **jak go zobaczyć i odwołać**, nie polegaj na pamięci.

**Urządzenie: LAB-SRX01 — Operational Mode, read-only sprawdzenie zaplanowanego reboot**

~~~junos
show system reboot
~~~

**Urządzenie: LAB-SRX01 — Operational Mode, SYNTAX ONLY — odwołanie zaplanowanego reboot na wspieranym Junos**

~~~junos
clear system reboot
~~~

Polecenie `clear system reboot` **nie jest kasowaniem konfiguracji**; dotyczy oczekującego restartu. Jeśli nie masz zaplanowanego reboot, wynik może być odpowiednią informacją „no scheduled reboot”. Sprawdź obsługę na danym release. [show system reboot](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-system-reboot.html).

### 5.6.4. Gotcha: reboot na routerze z redundantnymi Routing Engines

Masz MX z dwoma RE, jedno Master i jedno Backup. Na Cisco przypomina to redundancy supervisor/SSO. Czy `request system reboot` musi oznaczać „cała platforma restartuje się bez możliwości failover”? **To zależy od hardware, opcji polecenia, topologii i aktualnego stanu**. Dokumentacja wymienia warianty `other-routing-engine`, `both-routing-engines`, a na Virtual Chassis `member` i `all-members`.

**Urządzenie: MX-CORE-01 — Operational Mode, read-only ustalenie ról przed jakąkolwiek akcją**

~~~junos
show chassis routing-engine
show system switchover
show system uptime
show version
~~~

Nie próbuj zgadywać, który RE restartuje się po wydaniu komendy z aktualnej sesji. W produkcji przyjmij procedurę vendor dla **tego modelu i konfiguracji GRES/NSR/ISSU**. W przypadku SRX cluster restart węzła może oznaczać failover i utratę sesji zależnie od aktualnego stanu RG.

**Urządzenie: SRX-CLUSTER-01 — Operational Mode, tylko urządzenie rzeczywiście będące SRX chassis cluster**

~~~junos
show chassis cluster status
show chassis cluster interfaces
~~~

Dwie redundantne jednostki nie tworzą automatycznie bezprzerwowego rebootu. **Przed restartem sprawdź, czy backup jest healthy**, a nie tylko czy istnieje.

### 5.6.5. Bezpieczne wyłączanie, gdy remote hands demontuje sprzęt

W rzeczywistym offboardingu użytkownik pyta: „Możemy wyjąć kabel zasilający?”. Inżynier powinien znać nie tylko `request system halt`, ale też stan ruchu, redundancy, plan migracji, storage oraz procedurę fizyczną.

**Urządzenie: EX-ACCESS-01 — Operational Mode, WYŁĄCZNIE po odłączeniu użytkowników i zatwierdzonym offboardingu**

~~~junos
request system halt
~~~

Po uruchomieniu `halt` system bezpiecznie zatrzymuje procesy/OS na wspieranej platformie. **Zanim remote hands fizycznie odłączy zasilanie, musi zobaczyć odpowiedni komunikat na konsoli i zastosować wskazany przez producenta okres oczekiwania**. Niektóre QFX wymagają np. odczekania po „The operating system has halted.”.

**Gotcha:** jeśli wykonasz halt z sesji SSH, stracisz możliwość obserwacji końca procedury po przerwaniu sieci. Dlatego producent rekomenduje **bezpośrednią konsolę** przy czynnościach zatrzymywania oprogramowania. Nie myl `halt` z odpowiednim `power-off` na platformach, które mają osobny mechanizm wyłączenia zasilania. [request system halt](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-halt.html), [Operational Commands](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-operational-mode-commands-monitoring.html).

### 5.6.6. Dlaczego nie warto od razu restartować daemonów?

Ktoś w zespole proponuje „zrestartuj rpd, bo jest problem z BGP”. Ale `rpd` obsługuje dużo więcej niż jednego sąsiada. Restart procesu może zerwać sesje routingowe, wpłynąć na RIB, protokoły i forwarding w sposób zależny od platformy. Nie jest równoważny pojedynczemu `clear bgp neighbor`.

**Urządzenie: R1-vSRX — Operational Mode, read-only obserwacja procesu**

~~~junos
show system processes extensive
show route summary
show log messages | match "rpd|mgd"
~~~

Oficjalna dokumentacja ostrzega przed restartowaniem software daemons bez konkretnej rekomendacji wsparcia Juniper. W tym rozdziale **nie uczymy `restart routing` jako domyślnego rozwiązania problemu**.

### 5.6.7. Powrót po restarcie — sprawdź więcej niż uptime

Po reboot:
- Czy nowa sesja SSH działa z właściwego bastiona?
- Czy interfejsy wróciły admin/link up?
- Czy wszystkie membery/RE są online i w oczekiwanych rolach?
- Czy routing neighbors oraz sesje firewall wróciły?
- Czy monitoring, NTP, syslog i archiwizacja działają?
- Czy logi zawierają nieoczekiwane kernel panic/alarms?
- Czy aplikacja klienta przeszła test **ze strony użytkownika**?

**Urządzenie: LAB-SRX01 — Operational Mode, read-only postcheck**

~~~junos
show system uptime
show version
show system alarms
show chassis alarms
show interfaces terse
show route summary
show system commit
show log messages | last 100
~~~

**Na pamięć:** `request system reboot` vs `request system halt`, `show system reboot`, `clear system reboot`.  
**Do zrozumienia:** RE redundancy/VC/cluster zmieniają skutki restartu.  
**Do przećwiczenia:** w EVE-NG tylko na oddzielnej kopii labu wykonaj kontrolowany reboot i porównaj stan przed i po — bez ingerencji w poprzednie laboratoria.

---

## 5.7. Odzyskiwanie hasła root — mechanizm ratunkowy i odpowiedzialność administratora [A]

### 5.7.1. Jeśli zapomniałeś hasła root, co jeszcze musisz mieć?

Nowy inżynier przejmuje starszy EX. Lokalny administrator odszedł, centralne AAA nie działa, hasło break-glass nie zostało poprawnie przekazane. Czy wystarczy wejść przez SSH i poprosić urządzenie o „reset root password”? Nie — taka operacja byłaby niebezpieczna z punktu widzenia bezpieczeństwa.

**Recovery root password** jest specjalną procedurą serwisową, zwykle wymagającą dostępu do **konsoli/bootloadera** i możliwości zrestartowania urządzenia. W odróżnieniu od normalnej zmiany hasła przez uprawnionego użytkownika może oznaczać **przerwę w działaniu** oraz zależeć od ustawień ochrony konsoli i boot.

Zadaj trzy pytania:
1. Czy masz formalne uprawnienie do odzyskania dostępu do tego urządzenia?
2. Czy masz **fizyczną lub równoważną konsolę hypervisora/EVE**, a nie tylko sesję SSH?
3. Czy możesz bezpiecznie wykonać reboot urządzenia i masz okno serwisowe?

### 5.7.2. Trzy sytuacje, których nie wolno mieszać

**Sytuacja A: masz już legalny dostęp super-user i chcesz zmienić root password.** To normalna operacja na konfiguracji, bez bootloader recovery.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], uprawniony administrator, hasło podawane interaktywnie**

~~~junos
set system root-authentication plain-text-password
show | compare
commit check
~~~

Po tym wykonasz świadomy commit i nowy test dostępu zgodnie z runbookiem. **Nie wpisuj hasła w plaintext do publicznego `.set`**; `plain-text-password` uruchamia interaktywny prompt.

**Sytuacja B: nie znasz root password, lecz masz uprawniony dostęp do konsoli/bootloadera klasycznego Junos.** Możesz użyć **procedury password recovery**, zależnej od FreeBSD generation i platformy.

**Sytuacja C: system ma hardened console, FIPS albo Junos OS Evolved.** Procedura może wyglądać inaczej albo być zablokowana. Nie zakładaj, że bootloader bezwarunkowo umożliwia zmianę root password.

### 5.7.3. Recovery na klasycznym Junos z upgraded FreeBSD — skąd biorą się `boot -s` i `recovery`?

Oficjalny przewodnik Juniper opisuje scenariusz:
- fizyczna konsola i kontrolowany reboot;
- w odpowiednim momencie zatrzymanie sekwencji boot, wejście do loadera;
- **`boot -s`** w celu uruchomienia trybu single-user;
- wybór **`recovery`** w odpowiednim prompt;
- przejście do Junos CLI, wejście w configuration mode;
- ustawienie nowego root password i commit;
- zakończenie recovery/reboot oraz weryfikacja dostępu.

To **procedura dla odpowiedniej gałęzi klasycznego Junos**, nie ślepy skrypt dla EX4400, każdej generacji MX ani platformy Evolved.

**Urządzenie: LAB-EX-LEGACY — BOOT LOADER, wyłącznie autoryzowany, odizolowany lab i odpowiadająca mu generacja FreeBSD**

~~~text
loader> boot -s
~~~

**Urządzenie: LAB-EX-LEGACY — prompt recovery w single-user mode, tylko jeśli oficjalna procedura danej platformy wyświetli taki prompt**

~~~text
Enter full pathname of shell or 'recovery' ...: recovery
~~~

**Urządzenie: LAB-EX-LEGACY — Junos Configuration Mode po udanym recovery**

~~~junos
set system root-authentication plain-text-password
commit
~~~

**Ostrzeżenie:** to materiały szkoleniowe do własnego urządzenia. Nie wykonuj na produkcyjnym switchu bez zatwierdzonego change i konsoli. W niektórych wariantach recovery CLI wymaga określonej sekwencji `cli`/`configure`; użyj dokładnej instrukcji producenta dla wykrytego bootloadera.

### 5.7.4. Najważniejszy gotcha: recovery może być celowo zablokowane

Juniper dokumentuje szczególny przypadek: jeśli console port jest skonfigurowany jako **insecure** (odpowiedni hardening), próba przejścia przez single-user mode może **żądać hasła root**, zamiast umożliwiać reset. Mechanizmy FIPS i Evolved także mogą zmieniać przebieg.

To jest ochrona przed nieuprawnioną osobą mającą fizyczny dostęp do szafy. Nie nazywaj tego „błędem recovery”, jeśli **urządzenie wykonuje zaprojektowaną politykę bezpieczeństwa**.

**Most z Cisco:** podobnie jak password recovery disable/ROMMON security oraz fizyczne zabezpieczenia urządzeń Cisco. Uprawniony recovery i możliwość bootowania alternatywnych obrazów zależą od konfiguracji sprzętu i security policy.

Źródła: [Recover a Root Password (Junos)](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/recovering-root-password.html), [Recover a Root Password (Junos OS Evolved)](https://www.juniper.net/documentation/us/en/software/junos/user-access-evo/user-access/topics/topic-map/recovering-root-password.html).

### 5.7.5. Gdy `encrypted-password` zaszkodzi bardziej niż brak hasła

Wyobraź sobie, że administrator wpisuje:
- `set system root-authentication encrypted-password "MojeHaslo123"`

I robi commit, zakładając, że Junos je zaszyfruje. To błąd. **`encrypted-password` oczekuje już prawidłowej zaszyfrowanej reprezentacji**, a nie zwykłego hasła. Producent ostrzega, że użycie plaintext w tym miejscu może uniemożliwić późniejsze zalogowanie się jako root i wymusić recovery.

**Urządzenie: LAB-SRX01 — Configuration Mode, bezpieczna forma interaktywna (nie wpisujemy hasła w dokumencie)**

~~~junos
set system root-authentication plain-text-password
~~~

**To jedna z najważniejszych różnic semantycznych podczas migracji Cisco → Junos.** `plain-text-password` odnosi się do **sposobu wejścia sekretu**, a nie koniecznie do przechowywania go jawnie w active configuration.

### 5.7.6. Po recovery trzeba naprawić przyczynę organizacyjną

Jeżeli do odzyskania dostępu potrzebowałeś single-user mode, sama nowa wartość hasła nie kończy problemu. Trzeba także:
- odtworzyć inwentaryzację kont i role;
- przywrócić i przetestować TACACS+/RADIUS;
- poprawnie zapisać/rotować break-glass credentials w vault;
- sprawdzić kto ma dostęp do OOB/console i jak jest audytowany;
- potwierdzić NTP, syslog, SSH i logging działań;
- zaktualizować rescue i off-device backup dopiero po poprawnej walidacji;
- udokumentować wpływ reboot i zachować event timeline.

**Na pamięć:** uprawniony recovery klasycznego Junos może wymagać console + single-user + `recovery`; nowe hasło przez `plain-text-password`.  
**Do zrozumienia:** secure/insecure console, FIPS i Evolved mogą blokować/zmieniać procedurę.  
**Do przećwiczenia:** na osobnej instancji vSRX, której odzyskanie jest dozwolone, prześledź wyłącznie ścieżkę potwierdzoną dla tej wersji, bez naruszania aktywnych labów.



---

## 5.8. Procedury monitorowania, konserwacji i diagnozowania — od zgłoszenia P1 do zamknięcia ticketu [A]

### 5.8.1. Inżynier Tier 3 nie wykonuje najwięcej komend. Zadaje najlepsze pytania.

Wyobraź sobie dwóch administratorów podczas tego samego incydentu. Pierwszy wkleja trzy strony `show` i mówi: „Nie widzę błędu”. Drugi uruchamia pięć poleceń, ale po każdym zawęża hipotezę. Po dziesięciu minutach wie, że pakiety dochodzą do SRX z właściwym source IP, że nie pojawia się odpowiednia sesja, a na wyjściowym interfejsie nie ma ramek. **Drugi administrator zebrał dowód przyczynowy**, nie tylko dużo danych.

Dobra procedura diagnostyczna nie jest katalogiem komend. Jest **maszyną do eliminowania hipotez**.

~~~mermaid
flowchart TD
   INCIDENT["INC: użytkownik traci ERP"] --> SCOPE["Scope: kto, kiedy, z jakiego VLAN / SSID?"]
   SCOPE --> L1["L1: state, errors, peer link"]
   L1 --> L2["L2: VLAN, MAC, LLDP, LAG, STP"]
   L2 --> L3["L3: source, routing table, ARP/NDP, return path"]
   L3 --> SEC["Security: SRX zones, policies, sessions, ACL/filters"]
   SEC --> APP["DNS/TCP/HTTPS/aplikacja"]
   APP --> ACTION["Najmniejsza naprawa i rollback plan"]
   ACTION --> PROOF["Pre/post: CLI + klient + monitoring"]
~~~

**To nie jest obowiązkowa kolejność wykonywania wszystkich komend.** Jeżeli od razu wiesz, że port flappuje i rosną CRC, nie potrzebujesz analizować DNS przez 40 minut. Diagram to **model hipotez**, nie „odczytaj wszystkie warstwy zanim pomyślisz”.

### 5.8.2. Pierwsze pięć minut: ustal impact i ostatnią zmianę

W środowisku wielolokalizacyjnym najważniejsze jest, aby uniknąć ogólnego pytania „czy Internet działa?”.

Ustal:
- **dokładną lokalizację** i affected CI;
- VLAN, adres IP klienta i jego default gateway;
- czy symptom jest tylko WLAN, tylko LAN, czy oba;
- czy dotyczy jednej aplikacji czy całego TCP/IP;
- kiedy się zaczął i kiedy był ostatni working state;
- czy w tym czasie był change, reboot, flap, zanik zasilania, utrata WAN;
- czy masz klienta testowego oraz osobę po stronie aplikacji;
- jakie są **trzy warunki rozwiązania** problemu uzgodnione z klientem.

**Urządzenie: EX-ACCESS-01 — Operational Mode, read-only pierwsza minuta**

~~~junos
show system uptime
show system commit
show system alarms
show interfaces terse
show lldp neighbors
show log messages | last 60
~~~

**Most z Cisco:** `show logging`, `show version`, `show interfaces status`, `show cdp neighbors` + Change History. W Junos także łączysz **alarm z topologią i chwilą zmiany**.

**Gotcha:** `show system commit` nie jest pełną historią wszystkich zdarzeń. Interfejs mógł opaść po odłączeniu włókna bez żadnego commita. Monitorowanie NMS i syslog muszą dostarczyć tę część dowodów.

### 5.8.3. Mapa fizyczna: nie ufaj samemu `description`

Wyobraź sobie, że port opisano `UPLINK-CORE-1`, ale lokalny technik podczas wcześniejszej migracji przepiął go do innego switcha. `show interfaces descriptions` pokaże **tekst zapisany przez człowieka**, nie faktycznego peer.

**Urządzenie: EX-ACCESS-01 — Operational Mode**

~~~junos
show interfaces descriptions
show lldp neighbors
show lldp neighbors detail
show interfaces terse
~~~

Porównaj LLDP local port → remote system → remote port. Jeśli są agregacje, pamiętaj, że LLDP może raportować **fizyczne members**, a konfiguracja uplinku działa na `ae`. Nie nazywaj `ge-0/0/47` samotnym uplinkiem, gdy jest członkiem `ae0`.

Na Cisco masz CDP i LLDP. W Junos standardowym odpowiednikiem do discovery jest LLDP. Nie zakładaj, że każdy neighbor ogłasza system name czy remote port — zależy od peer i konfiguracji.

### 5.8.4. Gdzie ginie pakiet? Pięć odrębnych miejsc

W naszym uproszczonym scenariuszu klient A próbuje uzyskać dostęp do serwera B.

**Miejsce 1 — klient:** zły VLAN, DNS, gateway, Wi-Fi policy, proxy.

**Miejsce 2 — switch access:** physical errors, VLAN membership, storm control, MAC learning, LACP/STP.

**Miejsce 3 — routing:** brak aktywnej trasy, zły VRF, nierozwiązany ARP, asymetria.

**Miejsce 4 — firewall:** brak policy, sesji, NAT, UTM, host-inbound (gdy celem jest sam firewall).

**Miejsce 5 — serwer:** listener TCP, host firewall, TLS, application pool i ACL.

Pamiętasz z Cisco rozróżnienie CoPP/traffic to device vs transit? Na SRX szczególnie uważaj: **ping do samej bramy** trafia do `host-inbound`; **ping przez SRX** jest rozpatrywany jako ruch tranzytowy przez security flow/policies. Sukces pierwszego nie udowadnia drugiego.

**Urządzenie: R1-vSRX — Operational Mode, route/ARP i flow dla ruchu testowego**

~~~junos
show interfaces terse
show route 203.0.113.10
show arp
show security flow session summary
show security flow session
~~~

**Nie wyciągaj wniosku „firewall blokuje” tylko dlatego, że nie widzisz sesji.** To może oznaczać, że pakiet nie dotarł do SRX, policy go odrzuciła przed utworzeniem sesji, ruch poszedł inną drogą albo sesja wygasła. Zweryfikuj fizyczny ingress, security zone i konkretne flow diagnostics zgodnie z dokumentacją modelu.

### 5.8.5. Zły „fix” bywa większą awarią niż pierwotny symptom

Podczas P1 ktoś proponuje:
- `clear interfaces statistics` — tracisz dowód wzrostu liczników;
- `clear security flow session all` — usuniesz aktywne sesje klientów;
- `restart routing` — możesz przerwać wiele sesji routingowych;
- `request system reboot` — odetniesz cały site;
- `load override` — możesz usunąć aktywną konfigurację.

Każde z tych poleceń ma określony kontekst, ale żadne **nie jest domyślną „naprawą” bez root cause**. W razie konieczności rozważ minimalny i odwracalny zakres. Dla przykładu, jeśli masz błędną konfigurację jednego statementu, popraw go punktowo z `show | compare`, a nie zastępuj całego configu.

### 5.8.6. Monitoring proaktywny kontra reagowanie na alarm

**Monitoring proaktywny** to NMS zbierający baseline interface utilization, optical DOM, errors, CPU, RE memory, NTP status, syslog i HA. **Incident response** to analiza zmiany stanu w konkretnym czasie.

Zaprojektuj progi, ale nie wierz bezkrytycznie jednemu procentowi. Na przykład „80% link utilization” może oznaczać:
- przez 10 sekund incydentowy burst;
- przez całą godzinę stały traffic;
- średnią z pięciu minut, która ukrywa microbursts;
- niedopasowanie sampling interval.

**Most z Cisco:** w Catalyst możesz mieć egress drops mimo średnio niskiego bandwidth na porcie. ASIC buffer i microbursts nie muszą być widoczne w zwykłym pięciominutowym average. W Junos podobnie: analizuj queue drops i model PFE/CoS, a nie tylko Gbps na interfejsie.

### 5.8.7. Protokół wykonania change — standard, który obniża liczbę incydentów

Zanim zmienisz konfig:
1. Przeczytaj HLD/LLD, wymaganą ścieżkę ruchu, affected CI i zależności.
2. Zanotuj precheck, normalny baseline, aktualny `show system commit` i użytkowników.
3. Sprawdź console/OOB, route management i backup.
4. W `configure private` przygotuj **minimalną** zmianę.
5. Wykonaj `show | compare` i `commit check` **przed** `commit confirmed`.
6. Po `commit confirmed` zbierz testy **bez dodatkowego `commit check`** — bo może potwierdzić zmianę.
7. Potwierdź commit dopiero po testach dostępności i usług.
8. Zapisz postcheck, wynik klienta i historyczny stan w ticket.

**Pamiętaj:** jeśli zmiana jest software upgrade, confirmed commit nie zastępuje procedury software rollback; gdy to restart, nie potrzebujesz candidate change, ale nadal potrzebujesz change window i planu odzyskania.

### 5.8.8. Evidence-based handover — co należy do worknotes?

W dobrym worknotes znajdują się:
- symptom i exact affected scope;
- identyfikacja urządzenia, portu i jego roli;
- data/czas i źródło timestamp;
- komendy diagnostyczne i **wniosek z ich wyników**;
- działania, kto je wykonał i czy zmieniły stan;
- potwierdzony root cause lub **hipoteza z poziomem pewności**;
- wynik postcheck, test klienta i status;
- kto przejmuje następny krok, jeśli problem nie jest w network domain.

To różnica między „checked switch all fine” i informacją, która umożliwia kolejnej zmianie Tier 3 kontynuowanie pracy.

### 5.8.9. Jeden scenariusz — trzy hipotezy

**Symptom:** wybrane stacje na WLAN nie otwierają ERP, ale ping do default gateway jest dobry. Stawiasz hipotezy:
- VLAN WLAN ma złą trasę do serwera;
- SRX security policy blokuje TCP/443 dla source subnet WLAN;
- serwer nie ma poprawnej drogi powrotnej do podsieci WLAN.

**Urządzenie: R1-vSRX — Operational Mode, przyjmujemy, że SRX jest na ścieżce i mamy zgodę na diagnostykę**

~~~junos
show route 203.0.113.20
show security policies
show security flow session summary
show interfaces terse
~~~

**Urządzenie: PC-WLAN — Windows PowerShell, test z rzeczywistej karty WLAN**

~~~powershell
ipconfig /all
route print -4
Test-NetConnection 203.0.113.20 -Port 443 -InformationLevel Detailed
~~~

Dopiero zestawienie wyników pozwala powiedzieć, czy problem był po stronie network, FW, return path, DNS czy aplikacji. Nie kończ incydentu wyłącznie na podstawie testu z RE.

**Na pamięć:** `show`, `monitor`, `ping`, `traceroute` to narzędzia, nie diagnozy.  
**Do zrozumienia:** minimalny test rozstrzygający ogranicza ryzyko.  
**Do przećwiczenia:** wykonaj przynajmniej trzy różne scenariusze z 5.9 i napisz worknotes oparte na dowodach.

