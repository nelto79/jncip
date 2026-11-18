# Rozdział 6 — Routing Fundamentals
## Od decyzji o trasie do faktycznego przekazania pakietu: RIB, FIB, preference, LPM, routing instances i static routing

**Ścieżka:** JNCIA-Junos [A] → JNCIS-ENT [S] → JNCIP-ENT [P].  
**Blueprint:** **6.1–6.9 [A]**, **6.10 [S]**, **6.11 [LAB]** — bez zmiany kolejności kanonicznego spisu treści.  
**Redakcja:** 18 listopada 2026. Baza: jawna dokumentacja Juniper/HPE i autorskie scenariusze.  
**Platformy:** EX/QFX/MX, SRX/vSRX; rozróżniamy sprzętowy forwarding i wirtualny stateful flow.  
**Status laboratorium:** **PLANNED / NOT RUNTIME VALIDATED.** Komendy pokazują zamierzoną konfigurację; nie są odczytami z rzeczywistego EVE-NG.

> **Cel:** nauczyć się przeprowadzać pakiet przez proces **wyboru RIB → aktywny next-hop → FIB → sąsiad L2 → egress** i umieć udowodnić, w którym miejscu przestaje działać. Jesteś inżynierem Cisco i znasz RIB oraz CEF — będziemy wykorzystywać tę wiedzę, a nie uczyć wszystkiego od zera.

---

## Prolog — dlaczego „mam trasę” nie oznacza „mam łączność”?

Jest 07:45, nowy site biurowy. Po dodaniu zapasowego WAN jedna aplikacja przestaje działać. Junior pokazuje w `show route` piękną statyczną trasę do sieci serwera. Senior sprawdza ping z samego SRX — działa. Klient z WLAN nadal nie łączy się po TCP/443.

Co jest źle? Może nie ma drogi zwrotnej. Może ruch użytkownika wchodzi inną routing instance. Może next-hop istnieje w RIB, lecz nie ma adjacency. A na SRX może brakować policy lub sesja jest zablokowana. **Wszystkie te sytuacje mogą wystąpić przy prawidłowym `show route`.**

~~~mermaid
flowchart LR
  USER["PC w biurze"] --> SRC["R1 — jaka instancja,\nsource i destination?"]
  SRC --> LPM["LPM + aktywna trasa"]
  LPM --> FIB["FIB / PFE"]
  FIB --> L2["ARP/ND i MAC rewrite"]
  L2 --> R2["R2 — kolejna\nniezależna decyzja"]
  R2 --> APP["Serwer aplikacji"]
  SRC -.-> SEC["SRX: flow/policy/NAT\nmogą zatrzymać pakiet"]
~~~

**Umowa dydaktyczna:** zaczynamy od jednej paczki, nie od tabel do wykucia. Potem przejdziemy do wyboru tras, instancji, static route i laboratorium. Każdy gotcha ma znaleźć się tam, gdzie rzeczywiście wyjaśniamy mechanizm, a nie w dodatku „20 pułapek”.

**Na pamięć:** `inet.0`, `inet6.0`, `show route`, `show route forwarding-table`, `preference`, `routing-instances`, `static route ... next-hop`.  
**Do zrozumienia:** różnica wybór **aktywnego route dla jednego prefiksu** vs **longest-prefix match dla adresu pakietu**.  
**Do przećwiczenia:** przewidywanie wyników z diagramów, analiza next-hop i dwóch kierunków ruchu, vSRX/EVE-NG z IPv4/IPv6.

---

## 6.1. Podstawowe pojęcia przekazywania pakietów IP [A]

### 6.1.1. Sortownia paczek — po co routerowi next-hop?

Wyobraź sobie sortownię kurierską. Pracownik pierwszego magazynu nie zna miejsca zamieszkania każdego adresata. Ma mapę regionów i informację: „do tej strefy pojedzie transport do magazynu pośredniego”. Następny magazyn podejmie nową decyzję.

Tak działa routing IP. Router analizuje **adres docelowy**, dopasowuje trasę w odpowiedniej tablicy, wybiera next-hop i interfejs wyjściowy. Drugi router nie „realizuje planu pierwszego”; wykonuje własny lookup. To dlatego problem z routingiem powrotnym może zepsuć aplikację, mimo że ruch wychodzący dociera do serwera.

**Gdzie analogia się kończy?** Router ponadto zmniejsza TTL/Hop Limit, w IPv4 aktualizuje checksum nagłówka, stosuje ARP/ND i tworzy nową ramkę L2. Firewall SRX może wykonać NAT i sprawdzić stan sesji.

### 6.1.2. Przejdźmy pakiet PC-A → ERP przez dwa routery

W izolowanym labie użyjemy:
- PC-A `192.0.2.10/24`, gateway `192.0.2.1`;
- R1 `192.0.2.1/24` po stronie klienta, `198.51.100.1/30` po stronie transit;
- R2 `198.51.100.2/30`, sieć ERP `203.0.113.0/24`;
- PC-B/ERP `203.0.113.10/24`, gateway `203.0.113.1`.

**Krok 1:** PC-A widzi, że ERP nie należy do jego /24, więc wysyła pakiet do bramy `192.0.2.1`.  
**Krok 2:** przez ARP ustala MAC R1. **Destination IP pozostaje 203.0.113.10**, ale destination MAC ramki to MAC bramy.  
**Krok 3:** R1 wybiera aktywny lookup w odpowiedniej RIB/FIB i ma next-hop `198.51.100.2`.  
**Krok 4:** R1 ustala MAC sąsiada przez ARP, zmniejsza TTL, przepisuje adresy MAC i wysyła pakiet na transit.  
**Krok 5:** R2 odbiera pakiet, widzi **connected `203.0.113.0/24`**, wykonuje ARP do ERP i przekazuje ruch.  
**Krok 6:** ERP odpowiada przez swoją bramę. **R2 musi mieć trasę zwrotną do `192.0.2.0/24`**, a R1 bezpośrednią sieć PC-A.  
**Krok 7:** Na vSRX potrzebna jest także poprawna polityka przepuszczająca nowe sesje; nie pomyl braku policy z brakiem route.

~~~mermaid
flowchart LR
  A["PC-A\n192.0.2.10"] -->|"MAC bramy"| R1["R1\n192.0.2.1\n198.51.100.1"]
  R1 -->|"nowe MAC, ten sam IP dest"| R2["R2\n198.51.100.2\n203.0.113.1"]
  R2 -->|"MAC serwera"| B["ERP\n203.0.113.10"]
  B -. "odpowiedź / route powrotny" .-> A
~~~

**Most z Cisco:** RIB/CEF + adjacency/rewrite na ISR/Catalyst zaspokaja analogiczne potrzeby jak Junos RE/rpd, KRT/PFE i resolved next-hop, ale implementacje sprzętowe nie są tożsame. Więcej o RE/PFE w rozdziale 2.

### 6.1.3. Host-bound vs transit — pułapka przy pingowaniu bramy

Ping do adresu samego SRX jest **host-bound** i może wymagać `host-inbound-traffic system-services ping`. Ping **przez** SRX do ERP jest ruchem **transit** i podlega między innymi security policy. **Działający ping bramy nie dowodzi routingu do aplikacji.**

**Urządzenie: R1-vSRX — Operational Mode, odczyt stanu**

~~~junos
show interfaces terse
show route 203.0.113.10
show arp
show security flow session summary
~~~

**Urządzenie: R2-vSRX — Operational Mode, odczyt zwrotnej ścieżki**

~~~junos
show route 192.0.2.10
show arp
show security flow session summary
~~~

Jeśli oba urządzenia znają trasy, ale klient nadal nie łączy się do ERP, dopiero wtedy przechodzisz do L2 adjacency, security policy, DNS/TCP i samej aplikacji. Napis „route exists” nie kończy diagnozy.

**Na pamięć:** forwarding jest **hop-by-hop**. **Do zrozumienia:** MAC są per-hop, IP destination pozostaje end-to-end bez NAT. **Do przećwiczenia:** narysuj MAC/IP przed i po R1, przewidź drogę odpowiedzi.

---

## 6.2. Routing tables i ich rola [A]

### 6.2.1. Jedna destynacja, wielu kandydatów

Router może znać drogę do tej samej sieci przez static, OSPF i BGP. Proces `rpd` przechowuje informacje, rozstrzyga, która trasa jest aktywna, i przekazuje wybraną do dalszego programowania forwardingu. Alternatyw nie musisz z miejsca wyrzucać: mogą przydać się po zmianie topologii.

**Most z Cisco:** znajomy podział pomiędzy RIB, bazami OSPF/BGP i FIB/CEF. Na Junos `show route` może pokazywać **kilka kandydatów dla jednego prefiksu** i oznaczać jedną aktywną gwiazdką `*`. Jeden prefiks to niekoniecznie jedna trasa. [Understanding Routing Tables](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-routing-forwarding-table-overview.html).

### 6.2.2. Co oznacza `inet.0`, `inet6.0` i `TENANT.inet.0`?

**`inet.0`** — główna IPv4 unicast RIB.  
**`inet6.0`** — główna IPv6 unicast RIB.  
**`TENANT-A.inet.0`** — oddzielna IPv4 tabela instancji o nazwie `TENANT-A`.  
**`TENANT-A.inet6.0`** — IPv6 w tym samym kontekście.  
**`mgmt_junos.inet.0`** — specjalna management instance, jeśli rzeczywiście została uruchomiona i wspierana.

Nie myl numeru po kropce z indeksem portu fizycznego. `inet.3` może służyć do rozwiązywania next-hopów przez LSP, ale MPLS nie jest tematem fundamentów i wrócimy do niego później.

**Urządzenie: R1-vSRX — Operational Mode, przegląd tablic**

~~~junos
show route summary
show route table inet.0
show route table inet6.0
show route protocol direct
show route protocol static
~~~

**Pytanie:** `show route summary` pokazuje 5 destinations i 7 routes. Czy dwie trasy zniknęły? Nie. Dla niektórych destinations istnieją **alternatywni kandydaci**. Nie używaj `show route | count` do liczenia prefiksów — liczy linie tekstu.

### 6.2.3. Active, inactive, hidden i holddown

**Active:** wybrana droga do prefiksu — zazwyczaj w `show route` oznaczona `*`.  
**Inactive:** znany kandydat, który przegrał z lepszym dla tego samego prefiksu; niekoniecznie błędny.  
**Hidden:** trasa niekwalifikująca się do normalnego wyboru, np. nieosiągalny next-hop lub import policy; ważne jest ustalenie *why hidden*.  
**Holddown:** szczególny stan przejściowy przy zmianach/wycofywaniu trasy, zależny od protokołu.

**Urządzenie: R1-vSRX — Operational Mode, szczegóły i ukryte trasy**

~~~junos
show route 203.0.113.0/24 detail
show route 203.0.113.0/24 extensive
show route hidden
show route summary
~~~

**Gotcha:** inactive ≠ hidden. Pierwsza może być pełnowartościowym standby candidate. Druga może mieć problem z kwalifikacją (np. unresolved next hop). Zanim wykonasz `delete` trasy, dowiedz się, dlaczego jej nie wybrano. [Junos show route](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html).

### 6.2.4. Wiele ścieżek w RIB ≠ automatyczny ECMP

Oficjalny materiał [Day One: Beginner's Guide to Learning Junos](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf) zwraca uwagę na przypadki wielu równokosztowych next-hopów i ich wybór. **Dwie ścieżki w RIB nie gwarantują dwóch instalacji w FIB ani równomiernego per-flow load balancing.** To zależy od źródła route, sposobu budowania next-hopów, forwarding policy i platformy; pełny ECMP rozwiniemy w rozdziale 7.

Na razie zapamiętaj pytanie: „Czy różne next-hopy są tylko znane, czy także aktywne i zainstalowane do forwardingu?”.

### 6.2.5. Routing instances to nie foldery wizualne

Jeśli routing instance `IoT` ma tablicę `IoT.inet.0`, nie oznacza to, że jej trasy są automatycznie w `inet.0`. To **izolacja decyzji routingu**, podobna do Cisco VRF-Lite. Aby traffic między instancjami działał, potrzebna jest jawna architektura route leaking/interconnect/policies — nie wystarczy nazwać dwie tabele.

**Urządzenie: EX-DIST-01 — Operational Mode, wyłącznie jeśli taka instance istnieje**

~~~junos
show route table IoT.inet.0
show configuration routing-instances IoT | display set
~~~

**Do zrozumienia:** kilka routing tables może istnieć na jednym fizycznym urządzeniu, a ruch z danego interfejsu wykorzystuje konkretny routing context.

---

## 6.3. Routing Information Base (RIB) vs forwarding table / FIB [A]

### 6.3.1. Dyspozytor planuje — magazyn realizuje

W firmie logistycznej planista oblicza możliwe trasy i wybiera dostawcę. Magazynowi wystarczy **konkretna instrukcja wysyłkowa**. Planista nie musi konsultować każdej paczki od nowa.

W Junos `rpd` zbiera trasy i utrzymuje RIB na **Routing Engine**. Informacje o wybranych, aktywnych trasach trafiają przez mechanizmy kernel/KRT do struktur **forwarding** i dalej do **PFE**. Hardware EX/QFX/MX ma inną implementację niż wirtualny dataplane vSRX, ale podział logiczny działa podobnie. [Routing and Forwarding Tables](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-routing-forwarding-table-overview.html).

~~~mermaid
flowchart TB
  I["Connected / Static / OSPF / BGP"] --> R["RE: rpd i RIB"]
  R --> A["Wybór aktywnej trasy"]
  A --> K["KRT / kernel forwarding data"]
  K --> P["PFE / forwarding table"]
  PKT["Pakiet IP"] --> P
  P --> N["L2 adjacency / next-hop / egress"]
~~~

**RIB pyta:** „Jakie drogi znam, która jest preferowana i dlaczego?”.  
**FIB pyta:** „Który next-hop i egress dla tego adresu?”.  
**ARP/ND pyta:** „Jaki MAC ma następny sąsiad?”.  
**SRX flow pyta:** „Czy ruch odpowiada dozwolonej polityce i stanie sesji?”.

### 6.3.2. Gotcha: `show route` jest prawidłowe, ale ARP nie istnieje

R1 ma aktywną static route przez `198.51.100.2`. Interfejs do R2 jest `up` i connected `198.51.100.0/30` pozostaje w RIB. R2 jednak nie odpowiada na ARP. Co się stanie?

**Routingowa osiągalność next-hopa** nie jest tym samym co **rozwiązanie MAC na L2**. Static może pozostać active, choć brak sąsiada uniemożliwia realny forwarding. Zatem brak ARP **nie musi** automatycznie tworzyć hidden route.

**Urządzenie: R1-vSRX — Operational Mode, rozdzielenie czterech dowodów**

~~~junos
show route 203.0.113.0/24 detail
show route forwarding-table destination 203.0.113.10
show interfaces ge-0/0/0 terse
show arp
ping 198.51.100.2 count 5
~~~

**Interpretuj w kolejności:** active RIB → forwarding entry → interfejs fizyczny → ARP/ND → SRX security session. To różne mechanizmy, mimo wspólnego symptomu „ping failed”.

### 6.3.3. Gotcha: `show route forwarding-table` to nie zawsze surowa zawartość hardware ASIC

Juniper [Day One Troubleshooting](https://www.juniper.net/documentation/en_US/day-one-books/topics/concept/troubleshooting.html) określa `show route forwarding-table destination` jako widok **wersji tablicy z perspektywy RE**. To cenna informacja, ale nie dowodzi bezwzględnie, że na każdym chipsetcie ASIC/TCAM wszystkie wpisy są identyczne.

**Most z Cisco:** `show ip route` vs `show ip cef` vs platformowe `show platform hardware ...`. Na Junos przy problemie KRT→PFE może być potrzebna diagnostyka specyficzna dla EX/QFX/MX i wsparcie JTAC.

### 6.3.4. Cztery rodzaje dowodu, których nie mieszamy

| Obserwacja | Co udowadnia | Czego nie udowadnia |
|---|---|---|
| `show configuration ... static` | Zamierzony statement w active | Że route wygrała i ma next-hop |
| `show route` z `*[Static/5]` | Wybrana active route | Że ARP i security policy przepuszczą traffic |
| `show route forwarding-table` | Widok forwarding instruction | Pełnego rzeczywistego stanu ASIC na każdej platformie |
| Ping z routera | Reachability ICMP z wybranego source/VRF | TCP aplikacji klienta i return path z innej sieci |

**Do przećwiczenia:** w labie 6.11 oddziel trzy awarie: brak static route, brak ARP po stronie next-hopa i security policy deny. Wszystkie mogą dać ping timeout, ale nie mają tej samej przyczyny.

