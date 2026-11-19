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

