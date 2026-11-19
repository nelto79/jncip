# ARCHIWUM — poprzednia wersja labu 7.8

**ARCHIVE ONLY, NIE jest to kanoniczny DIAMOND.** Historyczny wariant R1 + cztery Linux (dwa hosty i dwa uplinki) zachowany do analiz dydaktycznych. Bieżący rozdział 7 ma bazować na `labs/reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md`.

## 7.8. Laboratorium — od jednej działającej trasy do FBF i ECMP [LAB]

Zrobimy to w taki sposób, jak prowadziłbym warsztat przy tablicy i konsoli. Najpierw zobaczysz sieć bez żadnej „magii”. Użytkownicy dotrą do testowej usługi po zwykłej default route. Potem dodamy aggregate i generated, a na końcu alternatywną tablicę oraz FBF. Będziemy wiedzieć, **co właśnie zmieniliśmy i co powinno zmienić się w ruchu**, zamiast wklejać 200 linii konfiguracji i potem zgadywać.

**Status laboratorium: PLANNED / NOT RUNTIME VALIDATED.** Ten scenariusz to projekt do uruchomienia, a nie raport z uruchomionej maszyny. Nie mamy tu wyniku \`show version\` konkretnego vSRX, mapowania NIC, zachowania security flow ani potwierdzenia wsparcia \`then routing-instance\` przez konkretny obraz. Przed wykonaniem zweryfikuj te elementy w EVE-NG i Juniper Feature Explorer. Jeśli klasyczny FBF jest niewspierany na danej wersji vSRX, przenieś tę fazę na wspierany EX/MX/QFX albo zaprojektuj osobny wariant APBR z właściwą dokumentacją SRX. **Nie deklaruj runtime PASS na podstawie samego commit albo wygenerowanego .unl.**

### Co musimy mieć w EVE-NG?

Nie potrzebujemy czterech ciężkich vSRX dla tego konkretnego mechanizmu. Wystarczy jeden router **LAB-R1-vSRX** i cztery lekkie maszyny Linux: dwóch klientów OFFICE oraz dwóch operatorów. R1 ma trzy porty danych: OFFICE, WAN-A, WAN-B. Segmenty mają być całkowicie izolowane. Nie łączymy cloud bridge, Internetu ani sieci zarządzania GCP z ruchem testowym.

~~~mermaid
flowchart LR
  subgraph OFF["OFFICE 192.0.2.0/24"]
    PCA["LAB-PC-A<br/>192.0.2.10/24"]
    PCB["LAB-PC-B<br/>192.0.2.11/24"]
  end
  PCA --- R1["LAB-R1-vSRX<br/>OFFICE 192.0.2.1"]
  PCB --- R1
  R1 ---|"WAN-A 198.51.100.1/30"| A["LAB-UP-A<br/>198.51.100.2/30<br/>lo: 203.0.113.10/32"]
  R1 ---|"WAN-B 198.51.100.5/30"| B["LAB-UP-B<br/>198.51.100.6/30<br/>lo: 203.0.113.10/32"]
~~~

*Rysunek 7.8 (autorski). Tekst alternatywny: PC-A i PC-B współdzielą OFFICE oraz bramę R1; dwa różne upstream Linux reprezentują WAN-A i WAN-B; oba mają identyczny testowy loopback, ale nie współdzielą segmentu Ethernet, więc ten sam cel pozwala udowodnić różny wybór ścieżki.*

Dlaczego używamy **tego samego 203.0.113.10/32 na dwóch Linux upstream**? Chodzi o kontrolowany eksperyment podobny do lokalnego anycast. Gdy PC-A i PC-B wysyłają do dokładnie tego samego destination, ale pakiety pojawiają się na dwóch różnych upstreamach, mamy dowód, że R1 wybrał **inną drogę**, a nie że jeden klient trafił do innego adresu. To izolowany test, nie pełny projekt usługi anycast z routingiem dynamicznym.

| Maszyna | Interfejs logiczny do weryfikacji | Adres | Rola |
|---|---|---|---|
| LAB-R1 | \`ge-0/0/1.0\` | 192.0.2.1/24 | OFFICE, ingress FBF |
| LAB-PC-A | \`eth0\` lub rzeczywisty NIC | 192.0.2.10/24 | Źródło „special” |
| LAB-PC-B | \`eth0\` lub rzeczywisty NIC | 192.0.2.11/24 | Źródło „normal” |
| LAB-R1 | \`ge-0/0/0.0\` | 198.51.100.1/30 | WAN-A |
| LAB-UP-A | \`eth0\` | 198.51.100.2/30 | Operator A |
| LAB-R1 | \`ge-0/0/2.0\` | 198.51.100.5/30 | WAN-B |
| LAB-UP-B | \`eth0\` | 198.51.100.6/30 | Operator B |
| Oba upstream Linux | \`lo\` | 203.0.113.10/32 | Wspólny testowy cel |
| LAB-UP-A | \`lo\` | 10.60.10.10/32 i 10.70.10.10/32 | Cele do tras contributor |
| LAB-UP-B | \`lo\` | 10.60.20.10/32 | Cel do drugiego contributor |

Wszystkie wartości IP są laboratoryjne. Zwróć uwagę, że **IP interfejsów EVE-NG nie jest automatycznie zgodne z numeracją \`ge-0/0/x\` w obrazie vSRX**. Dla konkretnej wersji mogą istnieć inne interfejsy danych i osobny management \`fxp0\`. Mapowanie jest pierwszą rzeczą do potwierdzenia.

### Faza 0. Zanim cokolwiek zepsujemy, sprawdźmy, co już jest na wirtualnych maszynach

Wchodzisz na R1, widzisz prompt \`>\`, ale nie wykonujesz jeszcze \`set\`. Najpierw sprawdzamy wersję, interfejsy, ewentualne istniejące strefy, NAT i konfigurację zarządzania. Jeśli maszyna była już używana, nie próbuj doprowadzić jej do stanu „czystego” komendą \`delete security\` czy przypadkowym factory reset. W takim przypadku lepiej stworzyć odrębną instancję labową.

**LAB-R1-vSRX — Operational Mode, read-only discovery**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration interfaces | display set
show configuration security zones | display set
show configuration security policies | display set
show configuration security nat | display set
show configuration routing-options | display set
show configuration routing-instances | display set
show configuration firewall | display set
show system commit
~~~

Na Linux hostach sprawdzamy nazwy interfejsów, adresację i systemowy firewall. Nie zakładaj, że masz \`eth0\`: współczesny Linux może pokazać \`ens3\`, \`enp1s0\` albo coś innego. Jeśli \`tcpdump\` nie jest zainstalowany, przygotuj obraz z tym narzędziem **przed** odizolowaniem labu, a nie instaluj go później przez produkcyjną sieć.

**LAB-PC-A — Linux shell, discovery**

~~~bash
ip -br link
ip -4 addr
ip -4 route
~~~

**LAB-UP-A — Linux shell, discovery**

~~~bash
ip -br link
ip -4 addr
ip -4 route
command -v tcpdump
~~~

Na końcu tej fazy powinieneś wiedzieć, które NIC tworzą OFFICE, WAN-A i WAN-B; czy nie ma konfliktujących adresów; czy security zone jest rzeczywiście pusta; i czy możesz bez ryzyka użyć \`commit confirmed\` mając console. Dopiero wtedy przystąp do konfiguracji.

### Faza 1. Trzy interfejsy R1 — ale jeszcze bez alternatywnej tablicy

R1 ma po prostu znać swoje trzy segmenty. Niczego jeszcze nie dzielimy między instances. Potrzebujemy najpierw stabilnego podłoża.

**LAB-R1-vSRX — Operational Mode, wejście do konfiguracji**

~~~junos
configure private
~~~

**LAB-R1-vSRX — Configuration Mode [edit private], wyłącznie po potwierdzeniu mapowania NIC**

~~~junos
set system host-name LAB-R1
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/24
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/30
set interfaces ge-0/0/2 unit 0 family inet address 198.51.100.5/30
set interfaces ge-0/0/1 description "CH07-OFFICE"
set interfaces ge-0/0/0 description "CH07-WAN-A"
set interfaces ge-0/0/2 description "CH07-WAN-B"
show | compare
commit check
commit confirmed 10
~~~

Dlaczego \`commit confirmed\`? Ponieważ popełnienie błędu w mapowaniu interfejsów może odciąć dostęp. W odizolowanym labie i z niezależną konsolą ta funkcja pozwala bezpiecznie cofnąć zmianę po upływie czasu, jeśli niczego nie potwierdzimy. **Po uruchomieniu \`commit confirmed\` nie wykonuj ponownie \`commit check\`**. Post-check ma być operacyjny, a nie kolejnym commitowym rytuałem.

### Faza 2. Konfiguracja hostów i droga powrotna

Przygotowujemy PC-A i PC-B. Oba są w tej samej sieci i mają tę samą bramę. Każdy z nich będzie potem wysyłał do 203.0.113.10. Pamiętaj, że poniżej \`IFACE=eth0\` to przykład; podstaw poprawną nazwę interfejsu. Jeśli obraz ma już skonfigurowany adres, nie dodawaj go drugi raz.

**LAB-PC-A — Linux shell, klient do przekierowania**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 192.0.2.10/24 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
ip -4 route
ping -c 3 192.0.2.1
~~~

**LAB-PC-B — Linux shell, klient korzystający ze zwykłego routingu**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 192.0.2.11/24 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
ip -4 route
ping -c 3 192.0.2.1
~~~

Na obu upstream Linux chcemy mieć **adres transit, testowy loopback oraz drogę powrotną do OFFICE**. To ostatnie łatwo przeoczyć. Wtedy zobaczysz echo request na UP-A, ale odpowiedź nie wróci do PC-A i wyciągniesz fałszywy wniosek „FBF nie działa”. Uzupełnimy drogę powrotną jawnie.

**LAB-UP-A — Linux shell, operator A**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 198.51.100.2/30 dev "$IFACE"
sudo ip -4 addr add 203.0.113.10/32 dev lo
sudo ip -4 addr add 10.60.10.10/32 dev lo
sudo ip -4 addr add 10.70.10.10/32 dev lo
sudo ip -4 route replace 192.0.2.0/24 via 198.51.100.1 dev "$IFACE"
ip -4 route
~~~

**LAB-UP-B — Linux shell, operator B**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 198.51.100.6/30 dev "$IFACE"
sudo ip -4 addr add 203.0.113.10/32 dev lo
sudo ip -4 addr add 10.60.20.10/32 dev lo
sudo ip -4 route replace 192.0.2.0/24 via 198.51.100.5 dev "$IFACE"
ip -4 route
~~~

Te ustawienia \`ip\` są zwykle nietrwałe po restarcie Linux. To zamierzone w laboratorium, ale przy tworzeniu reprodukowalnego .unl warto je umieścić w skrypcie bootstrap dedykowanym do testowego obrazu. **Nie commituj haseł ani produkcyjnych konfiguracji.**

R1 może już sprawdzić reachability operatorów po transit. To jeszcze **nie** jest test FBF ani końcowej aplikacji, ale pozwala wykryć zamienione WAN-A z WAN-B.

**LAB-R1-vSRX — Operational Mode, test transitów**

~~~junos
ping 198.51.100.2 source 198.51.100.1 count 3
ping 198.51.100.6 source 198.51.100.5 count 3
show route 198.51.100.0/30 exact
show route 198.51.100.4/30 exact
~~~

Zanim potwierdzisz fazę interfejsów, upewnij się, że post-check przeszedł. Jeśli nie, pozwól automatycznemu rollbackowi lub napraw mapowanie przez konsolę, w sposób zgodny z planem.

**LAB-R1-vSRX — Configuration Mode [edit private], wyłącznie po PASS fazy 1–2**

~~~junos
commit
~~~

### Faza 3. SRX musi dopuścić testy — inaczej każda lekcja routingu skończy się fałszywą diagnozą

Na vSRX potrzebujemy security zones. Nasze testy mają korzystać z ICMP od klientów w OFFICE do obu upstreamów. Z punktu widzenia zasad bezpieczeństwa stworzymy tylko laboratoryjne polityki **CLIENT → WAN-A** i **CLIENT → WAN-B** dla \`junos-ping\`. Na firewallu produkcyjnym użyłbyś precyzyjnych adresów i aplikacji; nie jest to rekomendacja bezwarunkowego zezwalania na cały ruch.

**LAB-R1-vSRX — Configuration Mode [edit private], po sprawdzeniu ewentualnych konfliktów stref**

~~~junos
set security zones security-zone CLIENT interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone WAN-A interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone WAN-B interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A match source-address any
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A match destination-address any
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A match application junos-ping
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A then permit
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B match source-address any
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B match destination-address any
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B match application junos-ping
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B then permit
show | compare
commit check
commit confirmed 10
~~~

Dlaczego nie wprowadzamy reguł pozwalających na wszystko? Po pierwsze, nie potrzebujemy ich. Po drugie, chcemy utrzymać różnicę między **routowaniem pakietu** a **zezwoleniem na jego sesję**. Do testów wystarczy ICMP, jeśli konkretna wersja SRX ma predefiniowaną aplikację \`junos-ping\`. Sprawdź też, czy nie działa dziedziczone source NAT, które mogłoby zmienić adres widziany po drugiej stronie i zepsuć naszą obserwację source-based FBF.

**LAB-R1-vSRX — Operational Mode, kontrola stref i polityk**

~~~junos
show security zones
show security policies from-zone CLIENT to-zone WAN-A
show security policies from-zone CLIENT to-zone WAN-B
show configuration security nat | display set
~~~

Jeżeli output odpowiada założeniom i podstawowe testy działają, potwierdź warunkowy commit.

**LAB-R1-vSRX — Configuration Mode [edit private], po PASS security baseline**

~~~junos
commit
~~~

### Faza 4. Najpierw musi działać zwykły default via WAN-A

Zanim zaczniemy manipulować tablicami, ustawiamy klasyczną default route. Zwykły router z jedną tablicą powinien teraz kierować obu klientów do 203.0.113.10 przez WAN-A. Tak wygląda **baseline**, którego nie możemy pominąć.

**LAB-R1-vSRX — Configuration Mode [edit private], default w globalnej inet.0**

~~~junos
set routing-options static route 0.0.0.0/0 next-hop 198.51.100.2
show | compare
commit check
commit confirmed 10
~~~

**LAB-PC-A — Linux shell, przed FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**LAB-PC-B — Linux shell, przed FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

Teraz odpalamy jedną z najbardziej wartościowych obserwacji całego kursu. Użytkownicy nie zobaczą, którym WAN poszedł pakiet — obie strony mają ten sam testowy loopback. Ale na upstreamach możemy przechwycić pakiety. \`tcpdump\` jest tutaj znacznie mocniejszym dowodem niż \`show route\` R1.

**LAB-UP-A — Linux shell, obserwacja prób do testowego loopback**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**LAB-UP-B — Linux shell, kontrola czy ruch nie trafił na łącze zapasowe**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**Przewidywany wynik (NIEZWERYFIKOWANY RUNTIME):** requesty z PC-A i PC-B pojawią się na UP-A; UP-B nie zobaczy tych pakietów. Jeśli obydwa komputery nie mogą dostać odpowiedzi, oddzielnie sprawdź Linux return route, firewall, SRX security policy i NAT. Jeśli UP-B widzi requesty mimo braku FBF, znaczy to, że baseline różni się od zakładanego — nie przechodź do kolejnej fazy bez wyjaśnienia.

**LAB-R1-vSRX — Operational Mode, RIB i FIB baseline**

~~~junos
show route 0.0.0.0/0 exact extensive
show route forwarding-table destination 203.0.113.10
show security flow session
~~~

Po pozytywnym post-check potwierdź commit. W tym momencie obie klasy klientów mają działać przez WAN-A. Dopiero z tego stabilnego stanu będziemy budować bardziej zaawansowane mechanizmy.

**LAB-R1-vSRX — Configuration Mode [edit private], po PASS default**

~~~junos
commit
~~~


### Faza 5. Trzy statyczne drogi, dwa różne rodzaje summary

Mamy już prosty działający ruch. Teraz dokładamy trasy do sieci aplikacyjnych, aby nie mylić kolejnych mechanizmów z problemami fizycznego WAN. Na UP-A znajduje się testowy loopback 10.60.10.10 i 10.70.10.10, na UP-B — 10.60.20.10. R1 dostanie dwa bardziej szczegółowe prefixy wewnątrz **10.60.0.0/16** oraz jeden wewnątrz **10.70.0.0/16**.

Z pierwszej grupy stworzymy **aggregate reject**, z drugiej **generated** odziedziczony z contributora. Nie wprowadzamy jeszcze żadnego BGP export: to czysta obserwacja RIB/FIB.

**LAB-R1-vSRX — Configuration Mode [edit private], statyczne contributors i dwa summary**

~~~junos
set routing-options static route 10.60.10.0/24 next-hop 198.51.100.2
set routing-options static route 10.60.20.0/24 next-hop 198.51.100.6
set routing-options aggregate route 10.60.0.0/16
set routing-options static route 10.70.10.0/24 next-hop 198.51.100.2
set routing-options generate route 10.70.0.0/16
show | compare
commit check
commit confirmed 10
~~~

**LAB-R1-vSRX — Operational Mode, czy contributorzy są aktywni?**

~~~junos
show route 10.60.10.0/24 exact extensive
show route 10.60.20.0/24 exact extensive
show route 10.60.0.0/16 exact extensive
show route 10.70.10.0/24 exact extensive
show route 10.70.0.0/16 exact extensive
~~~

Są tu dwa rodzaje oczekiwań. **Oba /24** powinny być aktywne jako static przy działających WAN i rozwiązanych next-hopach. **Aggregate /16** powinien móc powstać dzięki aktywnym bardziej szczegółowym trasom, domyślnie z reject next-hop. **Generated /16** powinien móc powstać dzięki 10.70.10.0/24 i skierować nieobjęte bardziej szczegółowym prefiksem pakiety przez next-hop primary contributor.

Teraz trzy testy na PC-B. Zauważ, że wybieramy **różne cele w obrębie jednego summary**, żeby udowodnić, że działa LPM i że summary nie „zastępuje” realnych /24.

**LAB-PC-B — Linux shell, dwa realne hosty i nieistniejący fragment aggregate**

~~~bash
ping -c 3 10.60.10.10
ping -c 3 10.60.20.10
ping -c 3 10.60.99.25
~~~

Dwa pierwsze powinny przejść właściwymi upstreamami, jeśli security i return path są poprawne. Trzeci powinien zostać zatrzymany na R1 przez aggregate /16, ponieważ brak bardziej szczegółowej trasy do 10.60.99.25. Otrzymanie ICMP unreachable albo timeout zależy od platformy i filtrowania odpowiedzi; dlatego sprawdzamy FIB i położenie pakietu, a nie opieramy całego wniosku na ping.

**LAB-R1-vSRX — Operational Mode, forwarding do adresu nieistniejącego pod aggregate**

~~~junos
show route forwarding-table destination 10.60.99.25
show route 10.60.0.0/16 exact extensive
~~~

**LAB-PC-B — Linux shell, drugi eksperyment — generated**

~~~bash
ping -c 3 10.70.10.10
ping -c 3 10.70.99.25
~~~

Dla 10.70.99.25 oczekujemy innego zachowania niż przy aggregate: generated /16 może skierować pakiet przez **WAN-A**, pomimo braku hosta na upstream. Ping może się nie udać, ale jeśli zobaczymy request na UP-A, mamy dowód, że R1 przekazał go dalej.

**LAB-UP-A — Linux shell, obserwacja nieistniejącego celu wewnątrz generated summary**

~~~bash
sudo tcpdump -ni eth0 'host 10.70.99.25' -c 10
~~~

**Przewidywany wynik (NIEZWERYFIKOWANY):** UP-A zobaczy pakiety przeznaczone do 10.70.99.25, jeśli generated jest aktywna i forwarduje via next-hop primary contributor. Nie twierdzimy, że sam TCP/IP stack UP-A odpowie na taki pakiet. To test **forwardingu R1**, nie sukcesu usługi na nieistniejącym host.

**LAB-R1-vSRX — Configuration Mode [edit private], po pozytywnym sprawdzeniu obu summary**

~~~junos
commit
~~~

### Faza 6. Budujemy osobną tablicę, ale na razie nie kierujemy do niej klientów

R1 wciąż ma globalną default przez WAN-A. To ma pozostać bez zmian. Obok tworzymy \`FBF-ALT\` typu forwarding z default przez WAN-B. Ważne: do tej pory PC-A i PC-B nadal korzystają z \`inet.0\`, bo nigdzie nie podpięliśmy filtra wejściowego.

**LAB-R1-vSRX — Configuration Mode [edit private], tworzymy forwarding instance**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 0.0.0.0/0 next-hop 198.51.100.6
show | compare
commit check
commit confirmed 10
~~~

**LAB-R1-vSRX — Operational Mode, co druga tabela wie przed RIB group?**

~~~junos
show configuration routing-instances FBF-ALT | display set
show route table FBF-ALT.inet.0
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
~~~

Jeśli connected route 198.51.100.4/30 nie jest w secondary, nie traktuj tego jako awarii: to punkt dydaktyczny, który chcieliśmy zobaczyć. Sama static default w nowej instancji nie daje automatycznie dostępu do connected z globalnej tabeli. Zrób notatkę, czy default jest active/inactive i co mówi output. Ten etap nie musi powodować sukcesu forwarding — ponieważ FBF nie jest jeszcze aktywny, nie uszkadza normalnego ruchu klientów.

**LAB-R1-vSRX — Configuration Mode [edit private], po zapisaniu obserwacji**

~~~junos
commit
~~~

Teraz definiujemy RIB group \`COPY-INTERFACE-ROUTES\` z primary \`inet.0\` i secondary \`FBF-ALT.inet.0\`, a następnie podpinamy ją do interface routes. To powinno udostępnić w FBF-ALT podstawowe informacje do rozwiązania next-hop 198.51.100.6.

**LAB-R1-vSRX — Configuration Mode [edit private], grupa dla interface routes IPv4**

~~~junos
set routing-options rib-groups COPY-INTERFACE-ROUTES import-rib [ inet.0 FBF-ALT.inet.0 ]
set routing-options interface-routes rib-group inet COPY-INTERFACE-ROUTES
show | compare
commit check
commit confirmed 10
~~~

**LAB-R1-vSRX — Operational Mode, obserwacja po podpięciu RIB group**

~~~junos
show configuration routing-options rib-groups | display set
show configuration routing-options interface-routes | display set
show route table inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
~~~

**Oczekiwany wynik (NIEZWERYFIKOWANY):** FBF-ALT zna connected do WAN-B i jej default może zostać poprawnie rozwiązana. Jeśli na tej platformie RIB group nie udostępnia oczekiwanych tras, zatrzymaj lab i sprawdź wsparcie konkretnego Junos, zamiast dorzucać przypadkowe \`next-table\` albo przenosić interfejsy pomiędzy instancjami.

**LAB-R1-vSRX — Configuration Mode [edit private], dopiero po PASS RIB group**

~~~junos
commit
~~~

Właśnie przeprowadziliśmy ważny eksperyment: przygotowaliśmy alternatywną trasę bez jeszcze jednej zmiany routingu klientów. I to jest dobra praktyka produkcyjna — najpierw przygotuj nowy kontekst i sprawdź jego next-hop, dopiero potem decyduj, jakie pakiety mają zacząć z niego korzystać.

### Faza 7. Włączamy FBF dla PC-A, ale PC-B ma nie zauważyć zmiany

Mamy \`inet.0\` dla WAN-A i \`FBF-ALT.inet.0\` dla WAN-B. Teraz wejściowy stateless filter wybierze alternatywną tablicę dla jednego źródła. Nie włączaj go, jeśli FBF nie jest wspierany na danym vSRX lub jeśli FBF-ALT default nadal jest unresolved. Jeśli to SRX, przed zmianą potwierdź również \`CLIENT → WAN-B\` security policy.

**LAB-R1-vSRX — Configuration Mode [edit private], FBF na ingress OFFICE**

~~~junos
set firewall family inet filter OFFICE-FBF term SPECIAL from source-address 192.0.2.10/32
set firewall family inet filter OFFICE-FBF term SPECIAL then routing-instance FBF-ALT
set firewall family inet filter OFFICE-FBF term NORMAL then accept
set interfaces ge-0/0/1 unit 0 family inet filter input OFFICE-FBF
show | compare
commit check
commit confirmed 10
~~~

Teraz jeden z najważniejszych testów w całym laboratorium. **PC-A i PC-B wysyłają do tego samego 203.0.113.10**. Oczekujemy, że pierwszy zostanie skierowany przez WAN-B, drugi pozostanie na WAN-A. Uważnie rozróżnij requesty od odpowiedzi, bo na uplinkach możesz zobaczyć oba kierunki.

**LAB-PC-A — Linux shell, klient objęty FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**LAB-PC-B — Linux shell, klient nieobjęty FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**LAB-UP-A — Linux shell, obserwacja requestów klienta zwykłego**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**LAB-UP-B — Linux shell, obserwacja requestów klienta specjalnego**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

Spodziewamy się, że na UP-B pojawi się źródło **192.0.2.10**, a na UP-A źródło **192.0.2.11**. Jeśli ruch pakietowy trafia na poprawny upstream, ale ping nie ma odpowiedzi, przyjrzyj się drodze powrotnej, strefom oraz stateful session na SRX. Sama klasyfikacja FBF nie daje automatycznej permisji security.

**LAB-R1-vSRX — Operational Mode, status po zmianie**

~~~junos
show configuration interfaces ge-0/0/1 | display set
show configuration firewall family inet filter OFFICE-FBF | display set
show route table inet.0 0.0.0.0/0 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact
show security flow session
~~~

**Oczekiwany wynik (NIEZWERYFIKOWANY):** działa host-to-host test z obu PC; requesty PC-A pojawiają się tylko na WAN-B, a PC-B tylko na WAN-A, bez niezamierzonych dodatkowych tras. Ostateczna obserwacja ma pochodzić z faktycznego \`tcpdump\` oraz session/route output — nie tylko z faktu, że parser przyjął \`then routing-instance\`.

Jeśli dwa niezależne testy przeszły, potwierdź \`commit\`. W przeciwnym razie wykorzystaj rollback, nie usuwaj całego filtrowania.

**LAB-R1-vSRX — Configuration Mode [edit private], dopiero po PASS obu klientów**

~~~junos
commit
~~~

Teraz mamy pierwszy kompletny punkt odniesienia: globalny ruch korzysta z WAN-A, a jedno źródło z WAN-B. W dalszej części celowo będziemy go psuć i zbierać dowody, dlaczego.


### Faza 8. Pierwsza awaria: aggregate przestaje istnieć, choć jego konfiguracja została

Zaczniemy od bezpiecznego eksperymentu, który nie powinien odciąć nam całej sieci OFFICE. Wycofujemy oba static routes /24 tworzące 10.60.0.0/16, ale nie usuwamy \`aggregate route 10.60.0.0/16\` z konfiguracji. Co przewidujesz?

Jeżeli nie ma innych aktywnych contributorów ani specjalnego \`passive\`, aggregate straci podstawę do aktywności. Właśnie dlatego \`commit check\` nie wystarczy jako test routingu: konfiguracja /16 będzie nadal poprawna składniowo, ale route może przestać być aktywna.

**LAB-R1-vSRX — Configuration Mode [edit private], kontrolowany fault tylko w odizolowanym labie**

~~~junos
delete routing-options static route 10.60.10.0/24
delete routing-options static route 10.60.20.0/24
show | compare
commit check
commit confirmed 2
~~~

**LAB-R1-vSRX — Operational Mode, podczas odliczania — bez kolejnego commit/check**

~~~junos
show route 10.60.0.0/16 exact extensive
show route protocol aggregate
show configuration routing-options aggregate | display set
show system commit
~~~

**Oczekiwanie (NIEZWERYFIKOWANE):** brak aktywnego aggregate 10.60.0.0/16, ale konfiguracja \`aggregate route\` nadal istnieje. Wyjątkiem byłby inny kwalifikujący się contributor lub celowo ustawiona opcja zmieniająca zachowanie trasy.

**Naprawa:** nie zatwierdzaj tej zmiany. Pozwól automatycznemu rollbackowi po \`commit confirmed 2\` przywrócić contributor routes. Po odtworzeniu wykonaj read-only \`show route\` i test hostów. Jeśli rollback nie nastąpi, zatrzymaj test, sprawdź historię i zastosuj zaplanowane ręczne przywrócenie baseline przez konsolę. Nie traktuj \`rollback 1\` jak stałego aliasu do „naszej wersji sprzed pięciu faz”, bo historia zmienia się po commit.

### Faza 9. Druga awaria: FBF istnieje, ale WAN-B nie daje się rozwiązać w secondary RIB

Teraz celowo odpinamy RIB group od źródła tras interfejsowych. Sama definicja \`COPY-INTERFACE-ROUTES\` zostaje w konfiguracji, podobnie jak \`FBF-ALT\` i input filter na OFFICE. To ma pokazać różnicę między *istniejącym obiektem*, *używaną grupą* i *faktyczną trasą connected w secondary table*.

Najpierw przewidywanie: jeśli FBF-ALT nie ma innej drogi do 198.51.100.6, a connected route przestanie być dostępna, jej default przez B może stać się unresolved. PC-A powinien wtedy utracić prawidłową alternatywną ścieżkę, podczas gdy PC-B nadal może korzystać z normalnej \`inet.0\` przez A. Dokładny efekt na danym vSRX musimy zaobserwować, a nie zakładać.

**LAB-R1-vSRX — Configuration Mode [edit private], kontrolowane usunięcie powiązania RIB group**

~~~junos
delete routing-options interface-routes rib-group inet COPY-INTERFACE-ROUTES
show | compare
commit check
commit confirmed 2
~~~

**LAB-R1-vSRX — Operational Mode, rozpoznanie problemu podczas pending rollback**

~~~junos
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
show route table inet.0 198.51.100.4/30 exact
show configuration routing-options interface-routes | display set
show system commit
~~~

**LAB-PC-A — Linux shell, klient SPECIAL w czasie awarii**

~~~bash
ping -c 3 203.0.113.10
~~~

**LAB-PC-B — Linux shell, klient NORMAL w czasie awarii**

~~~bash
ping -c 3 203.0.113.10
~~~

**Naprawa:** pozwól wygaśnięciu \`commit confirmed\` automatycznie przywrócić powiązanie \`interface-routes\`, a potem sprawdź connected w FBF-ALT i właściwy upstream dla PC-A. Jeśli secondary nadal nie ma potrzebnej trasy, nie dodawaj przypadkowej kolejnej default; sprawdź wsparcie RIB group i istniejące import-policy.

### Faza 10. Trzecia awaria: włączyliśmy FBF tylko dla jednego hosta, ale odcięliśmy resztę biura

W naszej konfiguracji filtr \`OFFICE-FBF\` ma dwa termy: SPECIAL i NORMAL. Do tej pory NORMAL był trochę niewidocznym bohaterem całej historii. Zobaczysz jego znaczenie, gdy usuniemy go na dwie minuty. FBF dla PC-A pozostanie skonfigurowany, ale inne źródła nie dopasują już żadnej reguły. Standardowy stateless filter ma na końcu **implicit discard**.

**LAB-R1-vSRX — Configuration Mode [edit private], kontrolowany błąd wyłącznie w laboratorium**

~~~junos
delete firewall family inet filter OFFICE-FBF term NORMAL
show | compare
commit check
commit confirmed 2
~~~

**LAB-PC-A — Linux shell, dopasowany do SPECIAL**

~~~bash
ping -c 3 203.0.113.10
~~~

**LAB-PC-B — Linux shell, niepasujący do SPECIAL**

~~~bash
ping -c 3 203.0.113.10
~~~

**Oczekiwany wynik:** PC-A może nadal korzystać z WAN-B, podczas gdy PC-B jest odrzucany przez brak termu kończącego filtrowanie \`accept\`. Trasa default PC-B w \`inet.0\` nadal jest aktywna; to nie routing, tylko stateless filter zatrzymał pakiet przed jego zwykłą ścieżką. Jeśli PC-B nadal działa, sprawdź, czy właściwy filter jest aktywny i czy pakiet faktycznie przechodzi przez ingress, który testujemy.

**Naprawa:** nie zatwierdzaj zmiany. Pozwól automatycznemu rollbackowi przywrócić NORMAL, a następnie wykonaj dwa testy post-check. Nie usuwaj filtra w całości, bo celem nie jest wyłączenie FBF dla PC-A, tylko przywrócenie ruchu, który powinien pozostać nietknięty.

### Faza 11. Czwarta lekcja: generated route nie jest bezpiecznym reject summary

To nie musi być nowa konfiguracja fault injection. Po prostu wróć do tras 10.70.0.0/16 i do obserwacji UP-A. Zwykłe \`aggregate 10.60.0.0/16\` chroni nieistniejący fragment summary przez reject. \`generate 10.70.0.0/16\` może wysłać pakiet do nieistniejącego fragmentu /16 przez next-hop contributora.

**LAB-PC-B — Linux shell, dwa cele bez hostów w odpowiednich summary**

~~~bash
ping -c 3 10.60.99.25
ping -c 3 10.70.99.25
~~~

**LAB-UP-A — Linux shell, obserwacja nieobsługiwanego fragmentu generated**

~~~bash
sudo tcpdump -ni eth0 'host 10.70.99.25 or host 10.60.99.25' -c 10
~~~

**Oczekiwanie:** pakiet do 10.70.99.25 może opuścić R1 przez A, podczas gdy pakiet do 10.60.99.25 powinien trafić w reject w R1, jeśli aggregate jest aktywny i nie ma konkretnej trasy. **Niedziałający ping do obu adresów nie oznacza tego samego mechanizmu**. Różnicę rozpoznajemy po RIB/FIB i rzeczywistym capture na UP-A.

### Faza 12. Piąta lekcja — ECMP poza FBF, aby nie mieszać dwóch mechanizmów

Dopiero gdy zakończysz fault injection FBF, rozpocznij osobny eksperyment na dwóch równorzędnych next-hopach. W przeciwnym razie nie będziesz wiedzieć, czy dany pakiet wybrał WAN-B na podstawie FBF czy hasha ECMP.

Tymczasowo zdejmujemy filtr \`OFFICE-FBF\` **z interfejsu**, ale go nie kasujemy z konfiguracji. Następnie tworzymy do tego samego 203.0.113.10/32 dwa static next-hopy. Stosujemy forwarding-table export policy \`ECMP-TO-SERVICE\`, której zadaniem jest umożliwienie wykorzystania multipath w FIB.

**LAB-R1-vSRX — Configuration Mode [edit private], ODRĘBNY WARIANT ECMP**

~~~junos
delete interfaces ge-0/0/1 unit 0 family inet filter input OFFICE-FBF
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.2
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.6
set policy-options policy-statement ECMP-TO-SERVICE term SERVICE from route-filter 203.0.113.10/32 exact
set policy-options policy-statement ECMP-TO-SERVICE term SERVICE then load-balance per-packet
set policy-options policy-statement ECMP-TO-SERVICE term SERVICE then accept
set policy-options policy-statement ECMP-TO-SERVICE term REST then accept
set routing-options forwarding-table export ECMP-TO-SERVICE
show | compare
commit check
commit confirmed 10
~~~

W tym eksperymencie **nie potwierdzamy od razu** nowego stanu. Chcemy porównać RIB i FIB, a potem pozwolić na bezpieczny rollback do bazowego FBF. W produkcji nigdy nie zastępuj istniejącej globalnej forwarding-table policy bez review — tutaj bazowy router miał nie mieć żadnej.

**LAB-R1-vSRX — Operational Mode, weryfikacja kwalifikacji multipath**

~~~junos
show route 203.0.113.10/32 exact extensive
show route forwarding-table destination 203.0.113.10
show configuration routing-options forwarding-table | display set
~~~

Przypomnij sobie pułapkę nazwy: \`load-balance per-packet\` na wielu Junos prowadzi w praktyce do **per-flow balancing**. Uruchomienie jednego ping z PC-A nie musi dać równomiernego podziału. Warto wykonać wiele **niezależnych flows**, różniących się source/destination lub portami, jeśli konkretne narzędzia i hash platformy pozwalają to obserwować.

**LAB-PC-A — Linux shell, podstawowy test jednego źródła**

~~~bash
ping -c 20 203.0.113.10
~~~

**LAB-PC-B — Linux shell, drugi niezależny source**

~~~bash
ping -c 20 203.0.113.10
~~~

**LAB-UP-A — Linux shell, ruch w wariancie ECMP**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 30
~~~

**LAB-UP-B — Linux shell, ruch w wariancie ECMP**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 30
~~~

Tylko dwa hosty i ICMP to **ograniczona próba**, nie statystyczny dowód równomiernego ECMP. Jeżeli każdy ping wybrał ten sam WAN, ale FIB ma oba next-hopy, eksperyment wymaga większej liczby niezależnych flows, a nie natychmiastowej zmiany konfiguracji. Możesz użyć przygotowanego wcześniej offline narzędzia generującego połączenia z różnymi portami, o ile platforma hash rzeczywiście bierze je pod uwagę. Również 70/30 w liczbie bajtów może być poprawne przy per-flow, jeśli rozmiary strumieni są nierówne.

**Naprawa/przywrócenie FBF baseline:** pozwól \`commit confirmed\` wycofać wariant ECMP i sprawdź, że filter \`OFFICE-FBF\` znowu jest przypięty do OFFICE, a forwarding-table export wróciło do wcześniejszego stanu. Po tym teście PC-A ma znów korzystać z WAN-B, a PC-B z WAN-A — inaczej laboratorium nie wróciło do właściwego punktu odniesienia.

### Faza 13. Martians pozostawiamy jako read-only

To świadoma decyzja. Zmienianie martian list na tym samym R1 mogłoby utrudnić interpretację wcześniejszych tras oraz zachowania FBF. Dlatego w tym głównym labie wykonujemy tylko obserwację.

**LAB-R1-vSRX — Operational Mode, domyślne martian prefixes i zwykłe trasy firmy**

~~~junos
show route martians
show route 10.60.0.0/16 exact
show route 127.0.0.0/8
~~~

Nie wyciągaj wniosku, że istnieje trasa 127.0.0.0/8 tylko dlatego, że jest reguła martian. I nie dodawaj \`allow\` dla zakresu specjalnego „aby móc przetestować ping”. Oddzielny eksperyment martian można zrealizować na osobnym węźle z kontrolowanym prefiksem i zapisanymi stanami, bez mieszania go z usługą OFFICE.

### Faza 14. Cleanup i evidence package

Często bardziej wartościowe niż sam wynik ping jest to, co pozostawiasz po zakończeniu szkolenia. Masz dwie sensowne drogi:

**Wariant zachowania labu:** pozostaw działający baseline FBF jako przyszły punkt odniesienia. Usuń tylko tymczasowe eksperymenty i potwierdź, że nie ma \`commit confirmed\` w toku, PC-A korzysta z WAN-B, PC-B z WAN-A, RIB group działa, a ECMP-only wariant został wycofany. Oznacz lab jako **PLANNED**, dopóki nie uruchomisz go i nie zbierzesz rzeczywistych wyników.

**Wariant pełnego przywrócenia:** użyj izolowanego snapshotu maszyny sprzed ćwiczenia, jeśli taki snapshot jest rzeczywiście sprawdzony i obejmuje odpowiednie dyski/konfigurację. Sam restart wirtualnej maszyny nie jest automatycznym rollback. Jeśli nie masz snapshotu, wykonaj **punktowy cleanup** wyłącznie swoich obiektów i tylko po przejrzeniu \`show | compare\`. Nie kasuj cudzych \`routing-options\`, \`security\` czy \`interfaces\` globalnie.

Wariant punktowy wciąż wymaga ostrożności. Poniższe polecenia usuwają tylko obiekty nazwane w tym rozdziale i zakładają **świeżego, izolowanego** vSRX bez obcych referencji. Najpierw sprawdź dokładnie aktualną konfigurację. Jeśli coś zostało użyte także przez inny lab, nie wykonuj cleanup automatycznie.

**LAB-R1-vSRX — Configuration Mode [edit private], przykład punktowego usunięcia obiektów zaawansowanych**

~~~junos
delete interfaces ge-0/0/1 unit 0 family inet filter input OFFICE-FBF
delete firewall family inet filter OFFICE-FBF
delete routing-options forwarding-table export ECMP-TO-SERVICE
delete policy-options policy-statement ECMP-TO-SERVICE
delete routing-options static route 203.0.113.10/32
delete routing-options interface-routes rib-group inet COPY-INTERFACE-ROUTES
delete routing-options rib-groups COPY-INTERFACE-ROUTES
delete routing-instances FBF-ALT
delete routing-options aggregate route 10.60.0.0/16
delete routing-options generate route 10.70.0.0/16
delete routing-options static route 10.60.10.0/24
delete routing-options static route 10.60.20.0/24
delete routing-options static route 10.70.10.0/24
show | compare
commit check
commit confirmed 10
~~~

Jeśli nie wykonywałeś wariantu ECMP i policy nie istnieje, polecenie \`delete\` może zgłosić brak obiektu — to nie dowodzi uszkodzenia routera. W takim przypadku pomiń odpowiednią linię po przejrzeniu aktualnego stanu, zamiast wklejać blok bezmyślnie.

W zależności od planowanego końcowego stanu możesz zachować podstawową default, adresy interfejsów i reguły \`CH07-PING-A/B\` jako bazę innych labów albo usunąć je osobno, punktowo, odtwarzając snapshot. Nie podaję masowego \`delete security\`, ponieważ mogłoby przypadkowo usunąć management i inne potrzebne reguły. Przy takim cleanup **brak host-to-host reachability po usunięciu static/FBF może być oczekiwany**; test końcowy ma potwierdzać docelowy stan, a nie poprzedni.

**LAB-R1-vSRX — Operational Mode, końcowe sprawdzenie stanu**

~~~junos
show system commit
show configuration firewall | display set
show configuration routing-instances | display set
show configuration routing-options | display set
show interfaces terse
show route
~~~

**LAB-R1-vSRX — Configuration Mode [edit private], po zatwierdzonych testach cleanup**

~~~junos
commit
~~~

Jeśli chcesz mieć z tego pełnowartościowy dowód laboratoryjny, zapisz w \`VERIFY.md\` rzeczywisty release, mapowanie NIC, konfigurację baseline, liczbę aktywnych tras, \`show route\` do obu summary, zawartość obu RIB, wynik FBF na obu upstreamach, ścieżkę rollback każdego fault i status osobno dla każdego testu. Nie wpisuj „RUNTIME VALIDATED”, jeśli jedynym dowodem jest poprawny \`commit\` albo statyczna kontrola pliku .unl.

### Czego nauczyliśmy się naprawdę?

Najlepszą częścią ćwiczenia nie jest to, że potrafisz wpisać \`set routing-instances FBF-ALT\`. Najważniejsze jest rozumienie **dlaczego**. Zbudowaliśmy dwie alternatywne drogi, ale to nie wystarczyło do rozwiązania problemu. Musieliśmy nauczyć drugą tablicę, jak dotrzeć do własnego next-hopa, a potem wskazać wyłącznie wybrane pakiety do tej tablicy. Równolegle zobaczyliśmy, że aggregate i generated inaczej traktują pakiety do nieistniejących fragmentów summary, a ECMP nie obiecuje dokładnie 50/50 dla pojedynczego połączenia.

To już nie jest „więcej komend routingu”. To jest świadome projektowanie **punktów podejmowania decyzji w Junos**.


---

