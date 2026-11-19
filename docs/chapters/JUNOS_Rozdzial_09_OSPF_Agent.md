# Rozdział 9 — OSPF: od protokołu link-state do zaawansowanego IGP

## Dlaczego OSPF działa, chociaż każdy router widzi tylko część sieci?

**Podręcznik:** JUNOS Enterprise Routing & Switching — JNCIA-Junos → JNCIS-ENT → JNCIP-ENT  
**Kanoniczny zakres:** sekcje 9.1–9.16 [S → P], [Unified Blueprint](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md)  
**Autorstwo:** niezależne opracowanie edukacyjne (Agent), 19 listopada 2026  
**Status:** GENERATED / SOURCE-REVIEWED; przykładowe konfiguracje **NOT RUNTIME VALIDATED**. Nie uruchamiano vSRX ani EVE-NG.  
**Docelowa platforma LAB:** cztery izolowane vSRX, wirtualny switch EVE i opcjonalnie dwa hosty Linux/VPCS. Składnia może wymagać dostosowania do release i numeracji NIC.  
**Nota o wynikach:** wszystkie fragmenty outputu są **SYMULOWANE**, chyba że tekst wprost stanowi inaczej. Polecenia muszą zostać sprawdzone na danym modelu/release przed wdrożeniem produkcyjnym.

**O co walczymy:** po tym rozdziale nie wystarczy powiedzieć „OSPF ma Full”. Masz umieć odtworzyć łańcuch: **Hello → neighbor → adjacency → wymiana LSDB → SPF → route selection → RIB/FIB → ruch użytkownika**; następnie znaleźć moment, w którym łańcuch się urwał.

> **Granica tego rozdziału.** Nie powtarzamy wszystkich podstaw RIB/FIB z rozdziału 6 ani ogólnego silnika routing policy z rozdziału 8. Wracamy do nich wyłącznie tam, gdzie tłumaczą faktyczne zachowanie OSPF. OSPFv2 i OSPFv3 mają osobne omówienie, nie są tym samym protokołem z innym adresem.

---

## Prolog — „Full” po obu stronach, a oddział nadal nie działa

Wyobraź sobie dwie części dużej firmy. W centrali znajdują się główne serwery aplikacyjne i zespół IT. W oddziale jest kilkaset stanowisk, serwery OT i lokalne VLAN-y. Operator zapewnia transport IP między budynkami. Inżynier wdraża dynamiczny routing, żeby przestać prowadzić ręcznie kilkadziesiąt tras statycznych. Na R2 widzi sąsiada **Full**. Po chwili test ping do oddziału nadal nie działa.

Niedoświadczona odpowiedź: „OSPF nie działa”. Inna, równie niebezpieczna: „OSPF jest Full, więc routing na pewno jest dobry”. Obie są nieprecyzyjne.

W OSPF sukces wymaga kilku odrębnych faktów. Routery muszą najpierw **odkryć się** na wspólnym łączu, uzgodnić parametry, zsynchronizować informacje topologiczne, przeliczyć drzewo najkrótszych ścieżek, wybrać drogę do właściwego prefiksu, zainstalować trasę i rozwiązać next-hop. Dopiero potem pojawia się osobny problem: czy ruch może opuścić interfejs i czy druga strona ma trasę powrotną? Na SRX dochodzi jeszcze **security flow processing**.

Tak wygląda nasz plan myślowy:

~~~mermaid
flowchart LR
  A["Interfejs: link/IP/area"] --> B["Hello i wykrycie sąsiada"]
  B --> C["Adjacency i synchronizacja LSDB"]
  C --> D["SPF: graf → najtańsze ścieżki"]
  D --> E["Route selection w RIB"]
  E --> F["FIB / next-hop"]
  F --> G["Pakiet i polityka bezpieczeństwa SRX"]
  G --> H["Droga powrotna i odpowiedź"]
~~~

**Interpretacja:** to mapa diagnostyczna, a nie obietnica, że każdy pakiet wywołuje OSPF. OSPF działa w control plane (proces **rpd** na Junos); dataplane korzysta z już zainstalowanych decyzji forwardingowych. Nie naprawisz błędu security policy przez zmianę kosztu OSPF. Nie naprawisz rozbieżnej LSDB przez dodanie default static route.

**Most z Cisco:** w Catalyst 9500/ISR znasz **show ip ospf neighbor**, **show ip ospf database**, **show ip route ospf** i CEF. W Junos odpowiednikiem diagnostycznym są **show ospf neighbor**, **show ospf database**, **show ospf route**, **show route protocol ospf** i **show route forwarding-table**. Polecenia nie są kopiami 1:1: szczególnie **show ospf route** pokazuje perspektywę protokołu, a **show route** pokazuje RIB oraz jego decyzję.

**Na pamięć:** rodzaje pięciu pakietów OSPF, stany neighbor, typy LSA v2 1/2/3/4/5/7, role DR/BDR/ABR/ASBR, area 0, router ID, podstawowe show.  
**Do zrozumienia:** kiedy tworzy się adjacency; dlaczego wszystkie routery w area mają logicznie zgodną LSDB; jak Dijkstra wybiera ścieżkę; skąd biorą się trasy IA/E1/E2 i co zmienia granica area.  
**Do przećwiczenia:** nadanie router ID i kosztu, OSPFv2/v3, passive, stub/NSSA, export static, interpretacja LSDB, zatrzymanie adjacency, policy filtering i bezpieczny rollback.

---

## 9.1. Link-state database (LSDB). [S]

### 9.1.1. Nie pytaj sąsiada o gotową trasę. Poproś go o mapę.

Przypomnij sobie zalety i wady protokołów distance-vector. Sąsiad może powiedzieć: „sieć X jest ode mnie trzy kroki”. OSPF wybiera inną metodę. Router opisuje własne łącza, sąsiadów, koszty i wybrane osiągalne sieci w **Link-State Advertisements (LSA)**. To coś jak uzgodniona mapa dróg, a nie pojedyncza wskazówka na skrzyżowaniu.

OSPF jest **link-state IGP**. Po zbudowaniu adjacency routery wymieniają informacje potrzebne do zrekonstruowania topologii **obszaru**. LSDB w danym obszarze ma być spójna na sąsiadujących, prawidłowo zsynchronizowanych routerach — nie znaczy to, że każda maszyna w całym autonomicznym systemie ma identyczne LSDB dla wszystkich obszarów. **Area 10 nie dostaje pełnego grafu Area 20.** Dostaje zagregowane/międzyobszarowe informacje od ABR. W OSPFv3 niektóre LSA mają scope link-local, a nie area-wide.

Wyobraź sobie trójkąt R1–R2–R3: koszt R1→R2 = 10, R2→R3 = 10, R1→R3 = 50. R1 nie musi zapytać R2, którą drogą dojść do R3. Buduje graf i liczy 10+10=20, mniej niż 50. Ale pamiętaj: sumujemy **koszty kierunkowe interfejsów**, a nie „długość kabla”. Koszt może być ustawiony administracyjnie i być asymetryczny.

~~~mermaid
flowchart TB
  R1["R1, RID 10.255.0.1"] -->|cost 10| R2["R2, RID 10.255.0.2"]
  R2 -->|cost 10| R3["R3, RID 10.255.0.3"]
  R1 -->|cost 50| R3
  R3 --> P["10.90.30.0/24"]
~~~

**Interpretacja:** R1 wybierze dojście do R3 przez R2, jeśli wszystkie krawędzie i prefiks są prawidłowo opisane w LSDB, a żaden inny warunek wyboru trasy nie zmieni wyniku. Wyliczenie SPF to nie to samo co route preference porównująca OSPF z BGP/static w Junos.

### 9.1.2. Co naprawdę przechowuje LSDB?

LSDB to baza **reklam stanu łączy**, a nie tabela „Destination → Next Hop”. LSA ma m.in. typ, identyfikator LSA, Advertising Router, sequence number, wiek i zawartość określoną przez typ. Router używa tych danych do obliczeń SPF i inter-area/external processing.

Dla uproszczonego obszaru z pięcioma routerami możemy dostać LSA Router od pięciu routerów, Network LSA od wybranego DR na sieci wielodostępowej, a także Summary LSA od ABR. Ta sama informacja może dotyczyć sieci, której nie ma jako aktywnej trasy w globalnej tabeli **inet.0**. Dlaczego? Ponieważ SPF/policy/protokół dopiero przetwarza LSDB, a route selection może ostatecznie wybrać inne źródło.

To krytyczna różnica:

~~~mermaid
flowchart LR
  L["LSA flooding / LSDB obszaru"] --> S["SPF + obliczenia OSPF"]
  S --> O["OSPF candidate routes"]
  O --> R["RIB inet.0 / inet6.0"]
  X["static / BGP / direct"] --> R
  R --> W["Active route / FIB"]
~~~

**Interpretacja:** brak prefiksu w **show route protocol ospf** ma inne znaczenie niż brak LSA w **show ospf database**. Jeżeli LSA jest widoczna, a trasa nie, szukaj różnicy między topologią a wyliczeniem, filtrowaniem, klasą trasy, dostępnością next-hopa i aktywnością.

### 9.1.3. Życie jednej zmiany

Po zerwaniu łącza R2–R3 nie trzeba czekać, aż upłynie jakiś „globalny update interval” jak w klasycznych okresowych distance-vector. R2 i R3 aktualizują informacje o stanie łącza, LSA są rozgłaszane w ramach właściwego flooding scope, routery aktualizują LSDB, uruchamiają przeliczenia i instalują nowe drogi. W praktyce prędkość reakcji zależy od wykrycia awarii, timerów, throttlingu SPF/LSA, kolejek, wydajności rpd i instalacji FIB. **„Link down” to inna ścieżka wykrycia niż milczący, wciąż up/up peer.**

Sequence number pozwala rozstrzygnąć, która reklama jest nowsza. Wiek i MaxAge chronią przed trzymaniem nieaktualnej topologii w nieskończoność. Junos posiada także mechanizmy LSA refresh, w tym udokumentowany domyślny okres odświeżania własnych LSA **50 minut**; nie mieszaj go z **Dead interval** ani z czasem konwergencji. [S1], [S2]

**Urządzenie: R2-vSRX — Operational Mode (po zbudowaniu LAB)**

~~~junos
show ospf overview
show ospf database
show ospf database extensive
show ospf route
show route protocol ospf
~~~

Pierwszy odczyt służy do poznania stanu procesu i obszarów. Drugi pokazuje LSA, trzeci ich szczegóły, czwarty perspektywę obliczonych tras OSPF, piąty widok kandydatów OSPF w głównym RIB. W przypadku wielu instancji doprecyzuj **instance** albo odpowiednią tablicę.

**Przewidź:** R1 ma adjacency Full z R2. R2 nie pokazuje żadnego route z R1 w **show route protocol ospf**. Czy to dowodzi błędu adjacency? Nie. Może R1 nie reklamuje żadnego nowego prefiksu albo konkretna trasa ma inną, już bardziej preferowaną reprezentację; należy porównać LSDB, OSPF route i RIB. Zobacz [S1], [S3].

---

## 9.2. Rodzaje pakietów OSPF. [S]

### 9.2.1. Pięć wiadomości protokołu — pięć różnych potrzeb

Kiedy dwie osoby chcą zsynchronizować dwie wielkie książki telefoniczne, nie zaczynają od wysłania tysięcy kompletnych stron na ślepo. Najpierw sprawdzają, czy się słyszą, porównują spis treści, żądają brakujących stron i potwierdzają ich odbiór. OSPF postępuje podobnie.

| Typ pakietu | Numer | Co robi naprawdę | Typowy moment |
|---|---:|---|---|
| Hello | 1 | Discover, keepalive, parametry sąsiedztwa i lista widzianych peerów | Discover, 2-Way, utrzymanie |
| Database Description (DBD/DD) | 2 | Streszcza nagłówki LSA i negocjuje master/slave/sequence DD | ExStart, Exchange |
| Link-State Request (LSR) | 3 | Żąda konkretnych brakujących lub starszych LSA | Loading |
| Link-State Update (LSU) | 4 | Niesie jedną lub wiele pełnych LSA | Synchronizacja i flooding zmian |
| Link-State Acknowledgment (LSAck) | 5 | Potwierdza otrzymanie LSA | Niezawodny flooding |

**To typy pakietów OSPF, nie typy LSA.** To częsta pułapka na styku poziomu JNCIS/CCNP. Type 1 *packet* to Hello, a Type 1 *LSA w OSPFv2* to Router LSA. Mieszanie tych numeracji prowadzi do absurdalnych diagnoz.

~~~mermaid
sequenceDiagram
  participant R1 as R1
  participant R2 as R2
  R1->>R2: Hello: RID, area, timers, options
  R2->>R1: Hello: zawiera RID R1
  Note over R1,R2: 2-Way; decyzja o pełnym adjacency
  R1->>R2: DBD: opis mojej bazy
  R2->>R1: DBD: opis mojej bazy
  R1->>R2: LSR: potrzebuję LSA X
  R2->>R1: LSU: pełna LSA X
  R1->>R2: LSAck: potwierdzam
  Note over R1,R2: Full po synchronizacji
~~~

**Interpretacja:** Hello nie przenosi całej LSDB. DBD nie jest pełną kopią LSA. LSR to żądanie; LSU dostarcza dane, a LSAck dotyczy niezawodnego flooding. Dzięki temu szukanie winnego można powiązać z etapem: brak Hello, problem w ExStart/Exchange, albo brak potrzebnych LSA w Loading.

### 9.2.2. W jakim IP płyną te pakiety?

OSPFv2 działa bezpośrednio w IPv4 jako **IP protocol 89**, a nie na TCP/UDP port 89. Na typowym broadcast segment używa multicast **224.0.0.5** (AllSPFRouters) i **224.0.0.6** (AllDRouters) w stosownych etapach. OSPFv3 również ma IP protocol/Next Header **89**, lecz jest przenoszony przez IPv6, zwykle z link-local source i multicast **ff02::5 / ff02::6**. OSPF nie używa TCP sesji, więc „port UDP 89 zamknięty” jest błędnym opisem problemu.

W Cisco analizowałbyś ACL i CoPP; na vSRX musisz odróżnić ruch **host-bound do samego routera** od tranzytowego. Zgoda na OSPF w host-inbound strefy jest warunkiem udziału w protokole, lecz nie zastępuje security policy dla pakietów użytkownika. [S4], [S5]

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show ospf interface detail
show ospf neighbor detail
show log messages | match "OSPF|ospf|RPD_OSPF"
~~~

W interfejsie sprawdzisz m.in. typ sieci, koszty i timery. Neighbor pokaże postęp wymiany baz. Logi zawierają zdarzenia, ale brak wpisów nie dowodzi, że żaden Hello nie dotarł; do głębszego rozpoznania służą **traceoptions** omówione w 9.12. Zrzut pakietów na kontrolowanym interfejsie testowym może być pomocny, jednak nie zakładaj, że **monitor traffic** na każdej platformie przechwytuje cały ruch tranzytowy w hardware.

---

## 9.3. Router ID. [S]

### 9.3.1. Adres identyfikujący uczestnika, nie punkt końcowy transportu

OSPF musi wiedzieć, kto stworzył LSA. Sam adres interfejsu nie wystarczy: router może mieć dziesięć portów, zmienić uplink lub adresację tranzytową. Stąd **Router ID**, wartość 32-bitowa zapisana w formacie dotted decimal. RID identyfikuje router w domenie OSPF. **Nie jest sam w sobie trasą ani obowiązkowo adresem IP przypisanym do interfejsu.** OSPFv3 mimo używania IPv6 nadal posługuje się 32-bitowym RID.

Na Cisco można znać heurystykę „highest loopback, highest physical IP”. Nie przenoś jej bez sprawdzenia: wybór domyślnego router ID w Junos ma własne, udokumentowane reguły zależne od dostępnych adresów i wersji. Niezawodnym podejściem projektowym jest **jawne** ustawienie RID.

Jeżeli R2 i R3 użyją tej samej wartości, informacje w LSDB mogą identyfikować różne urządzenia jako jedno. Nie jest to kosmetyczny warning. Router ID musi być unikalny w odpowiednim kontekście routingu/protokołu. Zmiana RID na działającym routerze może wymagać restartu protokołu i ma skutki dla sąsiedztw; nie traktuj jej jako bezpiecznego live rename.

**Urządzenie: R2-vSRX — Configuration Mode**

~~~junos
set routing-options router-id 10.255.0.2
set interfaces lo0 unit 0 family inet address 10.255.0.2/32
commit check
show | compare
commit confirmed 5
~~~

Najpierw nadajemy identyfikator **pod routing-options**, potem stabilny adres loopback. Można wybrać ten sam numer dla czytelności; to dwie odrębne rzeczy. Loopback nie jest potrzebny do samego istnienia Router ID, ale ma sens jako stabilny punkt monitorowania, źródło testów i ewentualny prefiks OSPF. W laboratorium kolejne kroki muszą potwierdzić **commit** przed wygaśnięciem confirmed, a testy powinny iść przez konsolę.

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show configuration routing-options | display set
show interfaces terse lo0
show ospf overview
show ospf database router
~~~

Sprawdź, czy RID w OSPF i w polu Advertising Router LSA jest zgodny z założeniami. Pamiętaj, że **lo0** nie reklamuje się sam w OSPF wyłącznie dlatego, że istnieje: trzeba go umieścić w odpowiedniej area jako **passive** albo świadomie redystrybuować, co daje inny typ trasy.

**Gotcha: dwa identyczne RID w różnych laboratoriach** nie muszą stanowić problemu, jeśli domeny są izolowane. Dwa identyczne RID we wspólnym obszarze to błąd. Sprawdzaj identyfikator po obu stronach linku, nie tylko na maszynie z alarmem. [S6]

---

## 9.4. Neighbors i adjacencies. [S]

### 9.4.1. „Widzę sąsiada” i „mam wspólną mapę” to inne stany

Na sieci Ethernet router R1 może odebrać Hello od R2, ale jeszcze nie mieć pełnej synchronizacji LSDB. To naturalne: discovery i adjacency są osobnymi etapami. W OSPF sąsiedztwo (neighbor relationship) oznacza, że routery się rozpoznają. **Adjacency** oznacza właściwy poziom synchronizacji i wymiany link-state. Nie każda para neighbors na broadcast ma osiągnąć Full: **DROther ↔ DROther** może celowo pozostać **2-Way**.

Kolejne etapy to:

- **Down:** brak aktywnego stanu rozmowy.
- **Attempt:** specyficzny dla nonbroadcast, próby kontaktu z ręcznie określonym peerem.
- **Init:** Hello otrzymano, ale w jego liście nie znaleziono jeszcze naszego Router ID.
- **2-Way:** sąsiad wymienił z nami informację wzajemnego widzenia.
- **ExStart:** uzgadnianie master/slave i numeracji sekwencji Database Description.
- **Exchange:** wymiana streszczeń LSDB (DD/DBD).
- **Loading:** żądanie brakujących lub nowszych LSA i otrzymywanie LSU.
- **Full:** adjacency zsynchronizowana.

~~~mermaid
flowchart LR
  D["Down"] --> I["Init"]
  I --> TW["2-Way"]
  TW -->|p2p lub adjacency wybrana| EX["ExStart"]
  EX --> E["Exchange"]
  E --> L["Loading"]
  L --> F["Full"]
  TW -->|DROther do DROther| H["2-Way: stan prawidłowy"]
~~~

**Interpretacja:** śledząc „utknął w ExStart”, pytamy o DBD, master/slave, MTU i zgodność opcji. Gdy jest tylko Init, sprawdzamy ruch Hellos w obie strony. Gdy dwa DROther są w 2-Way na segmencie multiaccess, nie naprawiamy stanu, który jest poprawny.

### 9.4.2. Co musi być zgodne po obu stronach?

Na tym samym łączu routery muszą rozmawiać w tym samym kontekście OSPF: wersja, area, zgodna interpretacja obszaru (stub/NSSA/options), odpowiednie hello/dead timers, uwierzytelnianie (gdy skonfigurowane), właściwy typ sieci i osiągalność na lokalnym medium. W OSPFv2 broadcast dochodzi sensowna zgodność podsieci/maski; dla OSPFv3 identyfikatory i adresacja są traktowane inaczej. Weryfikuj też unikalność RID.

**Nie każda rozbieżność** daje ten sam objaw. Mismatch hello/dead często uniemożliwia neighbor; mismatch MTU częściej zatrzymuje osiąganie Full podczas DBD, w zależności od implementacji i konfiguracji. Błąd host-inbound na SRX może sprawić, że konfiguracja OSPF wygląda prawidłowo, lecz ruch protokołu nigdy nie dojdzie do rpd.

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show ospf interface detail
show ospf neighbor extensive
show interfaces terse
show configuration protocols ospf | display set
show security zones | display set
~~~

**Urządzenie: R3-vSRX — Operational Mode**

~~~junos
show ospf interface detail
show ospf neighbor extensive
show interfaces terse
show configuration protocols ospf | display set
show security zones | display set
~~~

Odczyty wykonuj **po obu stronach**. Nie zgaduj o parametrach peera ze statusu jednej strony. Podczas diagnozy zacznij od pytania: czy **show ospf interface** widzi port i właściwe area? Następnie: czy w **show ospf neighbor** peer w ogóle istnieje? Jeśli tak, w jakim stanie i od jak dawna? Dopiero potem analizuj LSDB.

### 9.4.3. Krótki incydent: tylko Init

**Symptom:** R2 pokazuje R3 w Init; R3 w ogóle nie widzi R2.  
**Hipoteza:** Hello z R3 dochodzi do R2, ale w przeciwnym kierunku nie dochodzi lub jest odrzucany.  
**Test:** sprawdzenie interface, host-inbound, filtrów protokołu IP 89, multicast i realnego mapowania portów vSRX.  
**Wynik:** na R3 zapomniano dodać **ospf** do host-inbound na interfejsie tranzytowym.  
**Root cause:** protokół nie otrzymywał pakietów, mimo sprawnego IPv4 linku.  
**Naprawa:** dopuszczenie OSPF wyłącznie na interfejsie, gdzie ma działać adjacency.  
**Postcheck:** Full, zsynchronizowane odpowiednie LSA i instalacja oczekiwanego prefiksu.

**Urządzenie: R3-vSRX — Configuration Mode (przykład naprawy)**

~~~junos
set security zones security-zone LAB interfaces ge-0/0/1.0 host-inbound-traffic protocols ospf
commit check
commit confirmed 5
~~~

To nie jest zamiennik **security policies** dla tranzytu. Nie rozszerzaj host-inbound na wszystkie interfejsy produkcyjne „bo działa”. [S4], [S7]

### 9.4.4. Przewidź wynik

R2 i R3 są połączeni dedykowanym Ethernetem. Konfigurujesz na obu stronach **interface-type p2p**; połączenie nie ma innych urządzeń L3. Czy powinniśmy zobaczyć DR i BDR? Nie — na OSPF point-to-point nie przeprowadza się wyboru DR/BDR. Czy dwa routery mają osiągnąć Full? Tak, po prawidłowej synchronizacji. OSPF po Ethernet nie musi oznaczać OSPF **broadcast network type**.

**Most z Cisco:** odpowiada to wymuszeniu **ip ospf network point-to-point** na interfejsie Ethernet. Junos: **set protocols ospf area ... interface ... interface-type p2p**. To dobra praktyka na rzeczywistym łączu dwóch routerów, ale na łączu z trzema aktywnymi routerami byłaby niewłaściwą abstrakcją.

---

## 9.5. Designated Router (DR) i Backup Designated Router (BDR). [S]

### 9.5.1. Dlaczego OSPF nie chce synchronizować każdego z każdym?

Wyobraź sobie firmową sieć tranzytową z sześcioma routerami na jednym VLAN-ie. Gdyby każdy router utrzymywał pełną adjacency ze wszystkimi pozostałymi, liczba potencjalnych par wynosiłaby **n(n−1)/2**, czyli dla sześciu routerów 15. Dla dziesięciu już 45. OSPF ma problem nie tyle z jednorazowym Hello, co z liczbą synchronizacji, floodingu i operacyjnego zamieszania przy zmianie grafu. Na sieciach **broadcast multiaccess** rozwiązuje go przez wyznaczenie DR i BDR.

DR jest przedstawicielem segmentu dla wymiany i tworzenia **Network LSA Type 2** w OSPFv2; BDR stoi w gotowości. Pozostałe routery (DROther) budują pełną adjacency z DR i BDR, a między sobą mogą pozostawać 2-Way. **Nie myl DR ze „sprzętowym routerem domyślnym” czy VRRP/HSRP.** DR nie jest next-hopem wszystkich pakietów użytkowników na tym VLAN-ie. FIB nadal może wskazywać fizycznie innego next-hopa.

~~~mermaid
flowchart TB
  DR["R2: DR"] --- A["R1: DROther"]
  DR --- B["R3: DROther"]
  DR --- C["R4: DROther"]
  BDR["R5: BDR"] --- A
  BDR --- B
  BDR --- C
  DR --- BDR
  A -. "2-Way, nie Full" .- B
  B -. "2-Way, nie Full" .- C
~~~

**Interpretacja:** linie pełne to adjacencies do DR/BDR; przerywane wskazują relację, która może poprawnie kończyć się na 2-Way. Ten graf ma sens tylko na wspólnym **broadcast** medium OSPF, nie na p2p między dwoma routerami.

### 9.5.2. Wybór, priorytet i pułapka „najwyższy RID”

W election liczy się **OSPF interface priority**, a przy jednakowym priorytecie Router ID. Priority **0** wyklucza router z kandydowania do DR/BDR. Rola DR/BDR jest **per broadcast segment**, nie „jeden DR na całą area”. Co ważniejsze, wybór jest niepreempcjny: router o wyższym priority, podłączony później, nie musi wywłaszczyć istniejącego DR. W środowisku Cisco znasz to z **ip ospf priority**. To samo podejście trzeba zachować przy migracji do Junos.

Trzy routery uruchamiają się kolejno: R1 priority 1 zostaje DR, R2 priority 10 BDR, a później R3 priority 200 dołącza do działającego segmentu. Czy R3 natychmiast zostanie DR? **Nie jest to prawidłowa przewidywalna konsekwencja samego priority.** Trzeba ocenić aktualny DR/BDR i moment election. Nie wykonuj na produkcji **clear ospf neighbor** tylko po to, by wymusić „estetyczny” DR: zrywasz control plane i możesz spowodować transient loss.

**Urządzenie: R3-vSRX — Configuration Mode (opcjonalny broadcast LAB z trzema routerami)**

~~~junos
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 priority 200
commit check
commit confirmed 5
~~~

**Urządzenie: R3-vSRX — Operational Mode**

~~~junos
show ospf interface detail
show ospf neighbor detail
show ospf database network
~~~

Pole DR ID/BDR ID i stan interfejsu wyjaśniają rzeczywistą rolę. Network LSA tworzy DR, opisując routery uczestniczące w segmencie. W laboratorium zasadniczym 9.16 łącza tranzytowe pracują jako **p2p**, więc nie należy oczekiwać tam DR/BDR. Aby zrobić osobny test election, potrzeba co najmniej trzech routerów w **jednym** segmencie broadcast; wariant opcjonalny opisano w LAB.

**Gotcha:** router o priority 0 nadal może być uczestnikiem OSPF i osiągnąć Full z DR/BDR. Wykluczenie z election nie wyłącza protokołu. [S8]

---

## 9.6. Area types i router types. [S]

### 9.6.1. Po co obszary, skoro link-state miał rozwiązać problem skalowalności?

Do pewnego rozmiaru jedna LSDB jest zaletą. Gdy firma ma 150 oddziałów, dziesiątki tysięcy prefiksów i mnóstwo flapujących połączeń, jedna wspólna topologia może stać się kosztowna. Każda istotna zmiana wymusza dystrybucję LSA i przeliczenia. Podział na **areas** ogranicza szczegółowość topologii do obszaru, zmniejsza zakres flooding dla intra-area LSA i daje ABR możliwość kontrolowania informacji między obszarami.

Najważniejsze rozróżnienie: **OSPF area nie jest VRF**. Wszystkie obszary w jednej instancji OSPF współtworzą jeden domenowy routing. ABR może rozgłaszać osiągalność prefiksów między nimi. VRF / Junos routing instance izoluje tablice i kontekst routingu; samo przejście z Area 10 do Area 20 nie wymaga route leaking.

W standardowym wieloobszarowym OSPF centralny **Area 0.0.0.0** jest backbone. Inne obszary są połączone z nim przez **Area Border Routers (ABR)**. Router może mieć kilka interfejsów w różnych obszarach; na każdym łączu OSPF obie strony muszą zgadzać się co do jego area. ABR utrzymuje osobny zbiór informacji topologicznej dla obszarów, do których należy, i generuje odpowiednie inter-area reklamy.

~~~mermaid
flowchart LR
 subgraph A10["Area 10: biuro"]
   R1["R1: internal"]
 end
 subgraph A0["Area 0: backbone"]
   R2["R2: ABR"] --- R3["R3: ABR"]
 end
 subgraph A20["Area 20: fabryka"]
   R4["R4: internal"]
 end
 R1 --- R2
 R3 --- R4
~~~

**Interpretacja:** R1 widzi szczegółowo Area 10, ale nie odtwarza wszystkich Router LSA z Area 20. Osiągalność R4 dostaje poprzez reklamy inter-area od ABR. Jeśli R2–R3 przestaje działać, nie wystarczy, że R1 ma nadal Full z R2: odcięta zostaje droga przez backbone.

### 9.6.2. Rodzaje routerów — nazwy opisują zadania, nie modele sprzętu

- **Internal Router:** wszystkie aktywne interfejsy OSPF należą do jednego obszaru (może być Area 0). R1 w naszej topologii.
- **Backbone Router:** uczestniczy w Area 0, lecz niekoniecznie jest ABR. Router z wszystkimi interfejsami OSPF w Area 0 jest backbone i internal jednocześnie.
- **Area Border Router (ABR):** łączy co najmniej dwa obszary, w tym logiczne połączenie z backbone zgodnie z modelem multi-area. W naszym LAB R2 i R3.
- **Autonomous System Boundary Router (ASBR):** wprowadza do domeny OSPF informacje z **innego źródła routingu**, np. static lub BGP. ASBR może być również ABR lub routerem wewnętrznym, zależnie od miejsca i konfiguracji. R3 stanie się ASBR, gdy świadomie wyeksportujemy jedną trasę statyczną.

Kiedy sieć ma połączenie bezpośrednie do operatorem BGP, nie oznacza to automatycznie, że router jest ASBR. Musi rzeczywiście **redystrybuować** trasy do OSPF. Sama konfiguracja interfejsu OSPF albo sama trasa default static nie czyni z routera ASBR.

### 9.6.3. Normal, stub, totally stubby, NSSA — problem z trasami zewnętrznymi

Załóżmy, że oddział Area 20 ma jedno wyjście do centrali. Czy potrzebuje stu tysięcy szczegółowych tras zewnętrznych od dostawców, żeby dostać się do serwera w Internecie? Prawdopodobnie nie. Może wystarczyć default route od ABR. OSPF pozwala ograniczać obieg określonych typów LSA:

| Typ area | Intra-area | Inter-area summaries | AS-external Type 5 | Lokalne wprowadzanie external |
|---|---|---|---|---|
| Normal | tak | tak | tak | tak, przez ASBR |
| Stub | tak | tak | **nie** | **nie** przez Type 5 |
| Totally stubby (stub + no-summaries) | tak | zasadniczo default zamiast szczegółowych summaries | **nie** | **nie** |
| NSSA | tak | zwykle tak | **nie** (z zewnątrz area) | **tak**, Type 7 |
| Totally NSSA (NSSA + no-summaries) | tak | zasadniczo default zamiast szczegółowych summaries | **nie** | **tak**, Type 7 w obszarze |

Tabela jest uproszczonym modelem działania; generowanie default, Type 3 i Type 7 oraz zachowanie ABR zależy od odpowiednich opcji, które musisz sprawdzić w rzeczywistej konfiguracji. Nie należy interpretować „stub” jako całkowitego zakazu komunikacji z sieciami zewnętrznymi. To **kontrola informacji routingu**, a ruch może przechodzić przez ABR po default route.

**Kluczowy gotcha Junos:** konfiguracja stub na interfejsach nie musi automatycznie dać działającą trasę domyślną. W dokumentacji konfiguracji obszarów Juniper wyraźnie opisano, że na ABR należy jawnie włączyć generowanie default przez **default-metric** dla określonych scenariuszy. Zawsze sprawdź faktyczne LSDB i RIB; nie polegaj na skrócie „stub zawsze dostaje default”. [S9]

**Urządzenie: R3-vSRX — Configuration Mode (wariant demonstracyjny, NIE baseline LAB)**

~~~junos
set protocols ospf area 0.0.0.20 stub default-metric 10
commit check
show | compare
~~~

**Urządzenie: R4-vSRX — Configuration Mode (ten sam typ area po stronie wewnętrznej)**

~~~junos
set protocols ospf area 0.0.0.20 stub
commit check
show | compare
~~~

Zgodność flag area między neighbor jest krytyczna. Ten wariant zostanie użyty dopiero po sprawdzeniu bazowego OSPF. Na R4 oczekujemy międzyobszarowej informacji o default, jeśli ABR naprawdę ją generuje. Nie nadpisuj tej konfiguracji, jeśli w Area 20 w tej samej chwili ćwiczysz lokalnego ASBR — **stub** i redystrybucja external w tym samym obszarze to sprzeczne cele.

**Urządzenie: R4-vSRX — Operational Mode**

~~~junos
show ospf database netsummary
show route 0.0.0.0/0 exact
show ospf route
~~~

**Problem do przewidzenia:** jeżeli R4 ma już konkretną trasę do 10.9.101.0/24 i dostał również default, gdzie trafi ruch do 10.9.101.5? Zawsze stosuje się **Longest Prefix Match** na aktywnej tablicy forwarding, zatem bardziej specyficzny /24, niezależnie od tego, czy default ma ładną metrykę.

### 9.6.4. NSSA: kiedy za oddziałem jest jeszcze własny operator

Wyobraź sobie Area 20 jako zamknięty obszar fabryczny, ale w fabryce działa zewnętrzna sieć serwisowa reprezentowana przez static. Chcemy, by informacje o niej wydostały się do backbone, ale nie chcemy wlewać całej zewnętrznej bazy Type 5 do fabryki. **NSSA** rozwiązuje właśnie tę sprzeczność. ASBR w NSSA generuje **Type 7 LSA**, a ABR może przetłumaczyć kwalifikujące się Type 7 na **Type 5** i rozgłosić je dalej w AS. Nie każdy Type 7 musi zostać bezwarunkowo przetłumaczony; obowiązują reguły P-bit/wyboru translatora i konfiguracji.

**Urządzenie: R3-vSRX — Configuration Mode (wariant NSSA, najpierw usuń ewentualne stub)**

~~~junos
delete protocols ospf area 0.0.0.20 stub
set protocols ospf area 0.0.0.20 nssa default-lsa default-metric 10
commit check
show | compare
~~~

**Urządzenie: R4-vSRX — Configuration Mode**

~~~junos
delete protocols ospf area 0.0.0.20 stub
set protocols ospf area 0.0.0.20 nssa
commit check
show | compare
~~~

**Uwaga o składni:** ścieżka **nssa default-lsa default-metric** jest wzorowana na oficjalnej dokumentacji obszarów; konkretny wariant generowania Type 3 vs Type 7 default i wymagania kompatybilności zależą od ustawień **no-summaries**, **type-7** i release. W labie zawsze odczytaj właściwy LSA, nie zakładaj wyniku z nazwy opcji. Wstawienie **delete** jest poprawne tylko wtedy, gdy poprzedni eksperyment rzeczywiście dodał stub. [S9], [S10]

### 9.6.5. Backbone discontinuity i virtual link

Często spotykany błąd projektowy: Area 20 ma kontakt tylko z Area 10; żadna z nich nie jest połączona z Area 0. Czy routery są połączone IP? Owszem. Czy to poprawny multi-area OSPF design? **Nie.** Topologia inter-area zakłada backbone. Istnieje **virtual link** przez kwalifikowany transit area, ale traktuj go jak narzędzie naprawy/transformacji architektury, nie jak normalny zamiennik porządnej Area 0. Virtual link wymaga właściwych końców, skonfigurowanego transit area i zgodności topologii; nie można bezmyślnie zestawić go przez stub. [S11]

**Most z Cisco:** OSPF area 0, ABR, stub i NSSA są standardowymi pojęciami, lecz składnia i zachowanie domyślne Junos różnią się od IOS XE. Nie zamieniaj automatycznie **area 20 stub no-summary** na jeden dowolny **set ... stub** bez obejrzenia LSA i default.

---

## 9.7. Realms. [S]

### 9.7.1. Realm to nie area — i nie routing instance

W Junos słowo **realm** pojawia się w konfiguracji **OSPFv3**. OSPFv3 zrodził się jako protokół dla IPv6, ale późniejsze rozszerzenia pozwalają używać go także dla innych address families. Junos grupuje te dodatkowe rodziny w osobne realms. Mówiąc prościej: **area** jest podziałem topologii wewnątrz instancji OSPF, zaś **realm** rozdziela przetwarzanie określonej rodziny adresowej OSPFv3 — z własnym zestawem neighbors i LSDB.

Domyślny OSPFv3 obsługuje **IPv6 unicast** i korzysta z **inet6.0**. Dodatkowe realms mogą dotyczyć **ipv4-unicast** (typowo inet.0), **ipv4-multicast** (inet.2) oraz **ipv6-multicast** (inet6.2). Włączenie realm nie stanowi automatycznego route leaking ani nie zamienia IPv4 w tunel IPv6. Używa mechanizmów protokołu dla danej AF, a wynik nadal trafia do odpowiedniej tablicy. OSPFv3 niesie swoje pakiety kontrolne po IPv6 na łączu, niezależnie od tego, czy reklamuje IPv4-unicast. [S12]

~~~mermaid
flowchart TB
 O["OSPFv3 / Junos rpd"] --> V6["Domyślne IPv6 unicast"]
 O --> V4["Realm ipv4-unicast"]
 O --> M4["Realm ipv4-multicast"]
 V6 --> R6["inet6.0"]
 V4 --> R4["inet.0"]
 M4 --> RM["inet.2"]
~~~

**Interpretacja:** to diagram kontekstów protokołu i docelowych rodzin routingu, nie rysunek połączeń fizycznych. Na tym samym łączu OSPFv3 może używać rozdzielnych instancji identyfikowanych przez Instance ID. W każdym realm konfiguracja area/interface jest niezależna.

**Urządzenie: R2-vSRX — Configuration Mode (oddzielny eksperyment, nie baseline)**

~~~junos
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/2.0 interface-type p2p
commit check
show protocols ospf3
~~~

Tę samą rodzinę musi poprawnie obsługiwać sąsiad i platforma. Nie włączaj realm jako „przy okazji” migracji OSPFv2 → OSPFv3. W pierwszym przebiegu LAB 9.16 użyj równolegle OSPFv2 dla IPv4 i **domyślnego OSPFv3 dla IPv6**, bo najłatwiej wtedy oddzielić dwie tablice i dwie bazy.

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show ospf3 overview
show ospf3 neighbor
show ospf3 database realm ipv4-unicast
show ospf3 route realm ipv4-unicast
~~~

Przy braku włączonego realm te polecenia mogą nie mieć użytecznych danych: to oczekiwany brak instancji, nie dowód uszkodzenia OSPFv3. Istotne: „realms” Junos OSPFv3 to **nie** „OSPF routing instance/VRF” i nie należy ich traktować jak osobnych firewall zones.

**Pytanie:** czy włączenie ipv4-unicast realm oznacza, że R2 może reklamować IPv4 po interfejsie bez funkcjonującego IPv6 link-local? Nie. Sam mechanizm transportu OSPFv3 bazuje na IPv6; testuj capability i link-local sąsiadów zgodnie z dokumentacją swojej wersji. [S12]

---

## 9.8. Rodzaje LSA. [S]

### 9.8.1. LSA to wiadomości o topologii, a nie typy pakietów OSPF

Jeżeli zobaczysz w LSDB **Router LSA**, wiesz, że dowiedziałeś się czegoś o połączeniach w obszarze. Jeśli widzisz **Summary LSA**, pytasz o ABR i dostęp między obszarami. Jeśli widzisz **AS-External**, analizujesz ASBR, redystrybucję i możliwy wpływ zewnętrznego kosztu. Umiejętność czytania typu LSA jest jak umiejętność rozpoznawania, czy dokument jest mapą dróg, streszczeniem regionu czy informacją o wyjeździe za granicę.

**OSPFv2 — najważniejsze typy:**

| LSA | Nazwa / autor | Flooding scope | Praktyczna interpretacja |
|---|---|---|---|
| 1 | Router LSA, każdy router | Area | Router, łącza i koszty w obszarze |
| 2 | Network LSA, DR | Area | Zestaw routerów na segmencie transit multiaccess |
| 3 | Summary / Network Summary, ABR | Area docelowa | Prefix osiągalny w innym obszarze |
| 4 | ASBR Summary, ABR | Area docelowa | Jak osiągnąć ASBR generujący external LSA |
| 5 | AS External, ASBR | AS z wyjątkami (np. stub/NSSA) | Prefix redystrybuowany spoza OSPF |
| 7 | NSSA External, ASBR w NSSA | NSSA | Zewnętrzna trasa wewnątrz NSSA, potencjalna translacja do Type 5 |

**Type 6** (historyczny MOSPF) nie jest częścią codziennej praktyki Junos Enterprise. **Type 9/10/11 w OSPFv2** to rodzina *Opaque LSAs* o odmiennym znaczeniu i scope od numerów 9/10/11 w OSPFv3; nie utrwalaj „typ 9 zawsze znaczy dokładnie jedno”. W dokumentacji i outputach Junos mogą pojawiać się także rozszerzenia dla traffic engineering i nowych funkcji, zależnie od wydania.

~~~mermaid
flowchart LR
 R["R1 — Area 10"] -->|Type 1: mój link| A10["LSDB Area 10"]
 D["DR sieci LAN"] -->|Type 2: segment| A10
 ABR["R2 — ABR"] -->|Type 3: prefiks Area 20| A10
 ABR -->|Type 4: droga do ASBR, gdy potrzebna| A10
 ASBR["R4 — ASBR"] -->|Type 7 w NSSA| N["LSDB NSSA"]
 N -->|kwalifikowana translacja ABR| A5["Type 5 w pozostałej domenie"]
~~~

**Interpretacja:** to mapa odpowiedzialności za informację, nie jeden obligatoryjny strumień pakietów. Type 4 ma sens przy zewnętrznych trasach i potrzebie wskazania drogi do ASBR; nie każde urządzenie musi mieć Type 4 w każdej area. Przez poprawny flooding scope nie wszystkie LSA docierają wszędzie.

### 9.8.2. Gdy ASBR ogłasza static: E1 kontra E2

Załóżmy, że R3 eksportuje 172.31.90.0/24 ze statycznej tabeli. W OSPF stanowi to external. **E1** uwzględnia metrykę zewnętrzną i wewnętrzny koszt dojścia do ASBR; **E2** porównuje przede wszystkim metrykę zewnętrzną, a koszt wewnętrzny jest wykorzystywany w tie-break (przy dalszych zasadach wyboru). Junos domyślnie używa external Type 2 dla ogłaszanych external routes; wartość można kontrolować w export policy. Dokumentacja wskazuje także, że przy rozpatrywaniu dwóch wersji E1/E2 tego samego celu E1 ma pierwszeństwo zgodnie z zasadami OSPF, co jest **regułą wewnętrznego wyboru OSPF**, a nie route preference między protokołami. [S1], [S13]

Znamy z Cisco metrykę E2. W Junos gotcha jest głębszy: **show route** może pokazywać **[OSPF/150]** dla external, zamiast **[OSPF/10]** jak dla internal. To **route preference**, nie E2 metric! Junos standardowo ustawia preference dla wewnętrznego OSPF na **10**, a dla OSPF external na **150**. Porównanie z Cisco AD **110** dla OSPF nie jest translacją liczby. [S14]

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show ospf database external extensive
show ospf route
show route protocol ospf
show route 172.31.90.0/24 exact extensive
~~~

Najpierw sprawdzasz LSA zewnętrzne. Potem wynik obliczeń OSPF, a dopiero na końcu aktywną lub nieaktywną trasę w RIB. External LSA nie gwarantuje wygrania route selection, jeśli np. istnieje lepsza static, direct lub alternatywna trasa.

### 9.8.3. OSPFv3: te same cele, inne rozdzielenie informacji

OSPFv3 rozdziela opis topologii od opisów prefiksów IPv6 inaczej niż OSPFv2. W OSPFv3 pojawiają się **Link-LSA** (Type 8) o link-local flooding scope oraz **Intra-Area-Prefix-LSA** (Type 9) łącząca prefix z określoną topologią/obiektem. OSPFv3 ma ponadto Router LSA, Network LSA, Inter-Area-Prefix-LSA, Inter-Area-Router-LSA, AS-External-LSA i NSSA LSA — nazwy dobrze oddają rolę, lecz numeracja, format i zakres mogą się różnić od OSPFv2. OSPFv3 Router LSA nie służy jako ten sam „nośnik IPv4 sieci” co w prostych opisach OSPFv2.

~~~mermaid
flowchart LR
 L["OSPFv3 Link-LSA, Type 8"] --> LL["Informacje link-local"]
 R["OSPFv3 Router/Network LSAs"] --> T["Graf topologii obszaru"]
 P["OSPFv3 Intra-Area-Prefix LSA, Type 9"] --> X["Prefiksy przypięte do topologii"]
 T --> SPF["SPF"]
 X --> SPF
 SPF --> F["inet6.0: trasy IPv6"]
~~~

**Interpretacja:** OSPFv3 może mieć zdrowy graf, a jednocześnie brakować oczekiwanych prefiksów. W takim przypadku nie kończ na **show ospf3 neighbor Full**; sprawdź IntraAreaPrefix-LSA i wynik OSPFv3 route.

**Urządzenie: R3-vSRX — Operational Mode**

~~~junos
show ospf3 neighbor
show ospf3 database
show ospf3 database extensive
show ospf3 route
show route table inet6.0 protocol ospf3
~~~

**Przewidź:** jeśli IPv4 OSPFv2 jest Full, ale IPv6 host nie odpowiada, czy OSPFv3 musi być sprawny? Nie. OSPFv2 i OSPFv3 mają różne bazy i sesje protokołu; host-inbound i IPv6 interface family są osobne, podobnie jak routing inet.0 i inet6.0. [S1], [S15]

---

