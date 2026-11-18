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



---

## 6.4. Route preference — dlaczego Junos wybrał tę trasę, chociaż druga ma lepszy koszt? [A]

### 6.4.1. Dwie drogi do tego samego prefiksu — tu działa preference

Wyobraź sobie, że R1 zna `203.0.113.0/24` z dwóch źródeł:
- **Static/5** przez `198.51.100.2`;
- **OSPF/10** przez `198.51.100.6`.

Dla **tego samego prefiksu w tej samej tablicy** Junos musi zdecydować, którego kandydata uznać za aktywnego. Domyślnie niższa **route preference** oznacza większe pierwszeństwo. Static z wartością 5 wygrywa więc z OSPF internal 10, zakładając że obie trasy są poprawne i kwalifikowane.

Dobrze znasz pojęcie **Administrative Distance** w Cisco. Ale nie popełnij błędu „skoro liczby wyglądają podobnie, to wszystkie domyślne wartości są takie same”. Nie są.

### 6.4.2. Prawdziwe wartości z dokumentacji producenta

Tabelę warto znać dlatego, że wynika z niej codzienne zachowanie, np. sytuacja, w której OSPF external może przegrać z BGP, ale static ma jeszcze lepsze pierwszeństwo.

| Route source | Domyślna preference Junos | Praktyczna interpretacja |
|---|---:|---|
| Directly connected | 0 | Połączony prefiks jest wysoko preferowany |
| Local / connected host address | zwykle 0 | Host route do własnego adresu urządzenia |
| Static | 5 | Często wygrywa z trasami dynamicznymi dla *tego samego prefixu* |
| OSPF internal | 10 | OSPF intra/inter-area, w obrębie mechaniki protokołu |
| IS-IS level 1 internal | 15 | Niżej niż OSPF internal przy porównywalnym prefixie |
| IS-IS level 2 internal | 18 | Drugi poziom IS-IS |
| RIP | 100 | Zwykle niżej niż OSPF/IS-IS |
| Aggregate | 130 | Preferencja agregacji, nie automatycznie forwarding do konkretnego nexthopa |
| OSPF external | 150 | Różna od OSPF internal |
| BGP | 170 | **Junos ma domyślnie 170 dla BGP**, bez prostego podziału AD 20/200 Cisco |

W tabeli podaję najbardziej użyteczne wartości dla Enterprise; **pełna tabela producenta** obejmuje także LSP, RSVP, LDP, system, router discovery i inne źródła. Źródło: [Understanding Route Preference Values](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/bgp/topics/concept/routing-protocols-default-route-preference-values.html).

**Most z Cisco:**
- Cisco connected = 0, static = 1, OSPF = 110, eBGP = 20, iBGP = 200 (typowe wartości IOS XE).
- Junos direct = 0, static = 5, OSPF internal = 10, BGP = 170.

**Najważniejszy gotcha:** **Junos BGP/170 nie oznacza, że eBGP jest gorszy od iBGP w wewnętrznym procesie wyboru ścieżek BGP**. To porównanie *route source* do innych protokołów przy instalacji w RIB, a nie cała logika BGP best-path. BGP nadal ma własne atrybuty (Local Preference, AS_PATH, MED itd.), które rozwiążemy w rozdziale o BGP.

### 6.4.3. Preference to nie metric ani local-preference

Mamy trzy różne pojęcia:
- **Route preference** porównuje kandydatów do *tego samego prefiksu* w Junos RIB.
- **OSPF metric/cost** wybiera drogę *wewnątrz OSPF*, zanim wynik zostanie porównany z innym source route.
- **BGP Local Preference** jest atrybutem BGP, nie synonymem Junos route preference. Junos w szczegółowym outputcie może używać również pola `Preference2` dla tiebreakerów, a nie do tego samego celu co główna preference.

Praktyczny eksperyment myślowy: do `203.0.113.0/24` dociera OSPF route cost 2 i BGP route LocalPref 500. Która wygra? **Bez dodatkowych zmian polityki globalnej preference Junos porównuje OSPF/10 z BGP/170 i zwykle aktywna będzie OSPF**, nawet jeśli BGP LocalPref jest bardzo wysoki. BGP LocalPref rozstrzyga między drogami *BGP*, nie automatycznie między BGP i OSPF.

Dodatkowo dopuszczalność route ma znaczenie: nawet z preferencją 5 static route nie wygra, jeśli nie może zostać zakwalifikowana do active. **Niższa preference to nie immunitet na unresolved next-hop**.

Źródło: [Route Preferences Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-route-preferences-overview.html).

### 6.4.4. Weryfikacja rzeczywistego wyboru, nie tylko tabelki

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route 203.0.113.0/24 extensive
show route protocol static
show route protocol ospf
show route hidden
~~~

W laboratorium ze statycznymi trasami bez uruchomionego OSPF polecenie `show route protocol ospf` może być puste — **to prawidłowy wynik**, a nie błąd konfiguracji. Dla przykładu z konkurencją OSPF potrzebny jest oddzielny lab lub wcześniej skonfigurowane sąsiedztwo.

**Ilustracja formatu, NIE rzeczywisty output:**

~~~text
203.0.113.0/24   *[Static/5]  ...
                    > to 198.51.100.2 via ge-0/0/0.0
                  [OSPF/10]   ...
                    > to 198.51.100.6 ...
~~~

W rzeczywistym Junos output zależy od platformy i wersji. Sprawdź, gdzie znajduje się **gwiazdka** i jaki jest next-hop aktywnego route. Jeśli widzisz tylko jedną drogę, nie twórz w głowie nieistniejącej alternatywy.

### 6.4.5. Floating static — kontrolowana droga zapasowa

W Cisco używasz statycznej trasy z podwyższonym Administrative Distance, żeby pozostawała nieaktywna, gdy lepsze źródło działa. W Junos robisz to przez **`preference`**.

Załóżmy, że OSPF dostarcza `203.0.113.0/24` z default preference 10. Chcesz static jako backup, aktywny dopiero gdy OSPF route zniknie.

**Urządzenie: R1-vSRX — Configuration Mode [edit], ilustracyjna dodatkowa route backupowa w LAB z istniejącym alternatywnym next-hopem**

~~~junos
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.2
set routing-options static route 203.0.113.0/24 preference 200
show | compare
~~~

To nie tworzy OSPF i nie gwarantuje fizycznie drugiego uplinku. Aby scenariusz miał sens, OSPF route i obydwa next-hopy muszą **naprawdę istnieć**, a statyczna trasa mieć poprawny source i return path.

**Gotcha:** floating static z wyższą preference nie zapewnia automatycznego **end-to-end failover**, jeśli primary gateway jest osiągalny L2, ale upstream blackholuje ruch. Do tego potrzebujesz świadomego mechanizmu wykrywania awarii: BFD, routing protocol, RPM/IP monitoring lub SLA tracking zależnie od platformy/release. Omówimy to dokładniej w punkcie 6.10 i rozdziale 7.

### 6.4.6. Tiebreaker nie jest uniwersalnym „najniższy metric dla wszystkiego”

Jeśli dwie trasy do tej samej sieci mają tę samą preference, wynik może zależeć od porządku tiebreakerów i protokołu. BGP ma własną procedurę best-path. Static route z wieloma next-hopami wymaga osobnego rozumienia `qualified-next-hop`.

Dlatego **nie ucz się fałszywej reguły**: „w Junos zawsze niższy metric wygrywa po niższej preference”. To uproszczenie nie uwzględnia oddzielnych procesów routingu i właściwości kandydatów.

**Na pamięć:** Static 5, OSPF internal 10, OSPF external 150, BGP 170; niższa preference wygrywa *dla tego samego prefiksu*.  
**Do zrozumienia:** preference, metric i BGP Local Preference to różne mechanizmy.  
**Do przećwiczenia:** przewiduj zmianę active route po zwiększeniu static preference, a następnie porównaj `show route` z FIB.

---

## 6.5. Longest Prefix Match — najważniejszy wyjątek od intuicji „niższa preference wygrywa” [A]

### 6.5.1. Adres 203.0.113.74, trzy trasy, jedna odpowiedź

Wyobraź sobie, że R1 zna:
- `0.0.0.0/0`, static preference 5, next-hop ISP-A;
- `203.0.113.0/24`, static preference 5, next-hop WAN-B;
- `203.0.113.64/26`, BGP preference 170, next-hop WAN-C.

Pakiet ma destination **`203.0.113.74`**. Która droga wygra?

Częsta odpowiedź: „static/5, więc WAN-B”. **Błędna**, bo porównujesz prefiksy o różnej długości.

Najpierw **dla każdego prefiksu** wybrano aktywną trasę. Teraz podczas forwarding FIB znajduje **najbardziej szczegółowy pasujący prefix**. `203.0.113.74` należy do `203.0.113.64/26`; jeśli trasa jest aktywna i zainstalowana, pakiet wybierze **WAN-C**, mimo route preference BGP 170. Preference nie porównuje /26 i /24 jako konkurentów „do tego samego prefiksu”.

### 6.5.2. Narysujmy dwa etapy, zamiast uczyć się dwóch haseł

~~~mermaid
flowchart TD
  C["Baza kandydatów RIB"] --> P0["0.0.0.0/0\nwybór active dla /0"]
  C --> P24["203.0.113.0/24\nwybór active dla /24"]
  C --> P26["203.0.113.64/26\nwybór active dla /26"]
  P0 --> F["FIB z aktywnymi prefixami"]
  P24 --> F
  P26 --> F
  D["Pakiet dest 203.0.113.74"] --> F
  F --> LPM["Longest Prefix Match = /26"]
  LPM --> NH["Next-hop / egress odpowiadający /26"]
~~~

**Etap A — RIB route selection:** jeśli dla **`203.0.113.0/24`** masz OSPF/10 i Static/5, wygrywa Static/5 dla /24.

**Etap B — FIB LPM:** dla pakietu `203.0.113.74` /26 wygrywa nad /24, niezależnie od preference, bo **nie są to te same prefiksy**.

W ten sposób możesz nauczyć się bardzo ważnej reguły bez zapamiętywania skomplikowanych algorytmów.

### 6.5.3. Przypadki sprawdzające

| Destination IP | Najdłuższy pasujący prefiks z przykładu | Forwarding |
|---|---|---|
| `203.0.113.74` | `203.0.113.64/26` | WAN-C |
| `203.0.113.120` | `203.0.113.64/26` | WAN-C |
| `203.0.113.130` | `203.0.113.0/24` | WAN-B |
| `198.18.1.10` | `0.0.0.0/0` | ISP-A |

`203.0.113.64/26` obejmuje adresy `203.0.113.64`–`203.0.113.127` (włącznie). To dlatego `.120` pasuje, a `.130` nie.

**Gotcha:** **`0.0.0.0/0` jest również route**. Nie jest magiczną rezerwacją pierwszeństwa. To po prostu najmniej szczegółowy prefix pasujący do każdego IPv4 destination, jeżeli nie ma bardziej szczegółowego aktywnego route.

### 6.5.4. Gotcha: bardziej szczegółowy discard/reject route może zablokować traffic pomimo poprawnego default

Wyobraź sobie, że masz domyślną trasę do Internetu, ale również:

**Urządzenie: R1-vSRX — Configuration Mode [edit], tylko izolowany lab/test, świadomy drop do dokumentacyjnej podsieci**

~~~junos
set routing-options static route 203.0.113.64/26 discard
show | compare
~~~

Co zrobi Junos z `203.0.113.74`? Wybierze /26 przez LPM i **odrzuci ruch**. Nie „spadnie” automatycznie do default `0.0.0.0/0`, bo specjalny route jest bardziej szczegółowy i aktywny.

**Most z Cisco:** `ip route 203.0.113.64 255.255.255.192 Null0` wygrywa nad default w CEF. W Junos `discard` odrzuca bez standardowej odpowiedzi ICMP unreachable, podczas gdy `reject` generuje odpowiednią informację zwrotną zgodnie z implementacją. Czyli różnica semantyki discard vs reject jest **ważna operacyjnie**, nie tylko składniowo. Źródła: [Junos static route discard](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/discard-edit-routing-options.html), [static route statements](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/static-edit-routing-options.html).

Jeśli `discard` służy np. ochronie przed routing loop w agregacji, trzeba to zaprojektować świadomie. Zbyt szeroki discard może spowodować blackhole całego site'u.

### 6.5.5. Jak sprawdzić LPM na realnym Junos?

**Urządzenie: R1-vSRX — Operational Mode, odczyt bez modyfikowania ruchu**

~~~junos
show route 203.0.113.74
show route 203.0.113.64/26
show route 203.0.113.0/24
show route 0.0.0.0/0
show route forwarding-table destination 203.0.113.74
~~~

Używaj zarówno polecenia wyszukującego destination, jak i odczytu konkretnych prefiksów. Wynik `show route` dla **host IP** bywa interpretowany inaczej niż dla **prefiksu**, zależnie od opcji `exact`, `longer` lub `all`. Jeśli nie jesteś pewien, odczytaj tablicę `inet.0` i sprawdź active prefix oraz FIB.

**Eksperyment do przećwiczenia:** zacznij od default; dodaj /24 i zaobserwuj nowy forwarding; dodaj /26 i zaobserwuj zmianę tylko dla adresów z tej podsieci; następnie usuń /26 i sprawdź powrót do /24. W kolejnych ćwiczeniach pokażemy to na R1 i R2 bez niszczenia management.

### 6.5.6. LPM i IPv6 działają analogicznie, lecz z inną rodziną tabel

Dla IPv6 możesz mieć:
- `::/0`;
- `2001:db8::/32`;
- `2001:db8:20::/64`;
- `2001:db8:20::10/128`.

Pakiet do `2001:db8:20::10` dopasuje /128, jeśli jest aktywny. Pamiętaj jednak, że IPv6 korzysta z ND, a nie ARP, i może wymagać innych security policies/ICMPv6.

**Urządzenie: R1-vSRX — Operational Mode, read-only IPv6**

~~~junos
show route table inet6.0
show route 2001:db8:20::10
show route forwarding-table family inet6 destination 2001:db8:20::10
show ipv6 neighbors
~~~

Jeśli dany release nie wspiera wariantu `family inet6` w tej dokładnej pozycji, użyj `show route forwarding-table ?` i dokumentacji modelu, zamiast powtarzać błędny wiersz. Nie nazywaj IPv6 testu PASS, jeśli host nie ma właściwej default route lub ND zawodzi.

**Najważniejsza reguła całego rozdziału:**
1. **Preference** wybiera active candidate do konkretnego prefiksu.
2. **LPM** wybiera prefiks o największej długości dla adresu destination.
3. **Resolved next-hop + forwarding + polityka** decydują, czy pakiet naprawdę dotrze.



---

## 6.6. Routing instances — dlaczego jeden switch może mieć kilka niezależnych map drogowych? [A]

### 6.6.1. Jedno urządzenie, dwóch klientów, ten sam zakres IP

Wyobraź sobie kampus obsługujący dwie firmy połączone wspólną infrastrukturą dystrybucyjną. Obie używają podsieci `10.10.10.0/24`. Jak jedna tablica miałaby wiedzieć, czy pakiet `10.10.10.40` należy wysłać do biura klienta A, czy do OT klienta B?

Jedna globalna `inet.0` nie rozwiąże tego bez dodatkowego kontekstu. Junos pozwala utworzyć kilka **routing instances**, które utrzymują osobne tablice i niezależne decyzje.

**Analogia:** masz dwa identyczne adresy ulicy, ale w dwóch miastach. Sam napis „ulica Leśna 10” nie wystarczy — musisz znać miasto. Routing instance jest właśnie takim *kontekstem miasta*.

**Most z Cisco:** najbliżej jest Ci do **VRF-Lite** na Catalyst 9500 lub ISR oraz do `ip vrf forwarding`. Jednak w Junos pojęcie **routing instance** jest szersze niż jeden typ VRF: może mieć typ `virtual-router`, `vrf` (w kontekście usług VPN), `forwarding`, `virtual-switch`, `evpn` i inne zależnie od platformy. Nie zakładaj, że każdy typ wymaga tego samego zestawu parametrów.

### 6.6.2. `virtual-router` to rozsądny przykład do nauki podstaw

W prostym enterprise VRF-Lite możemy użyć `instance-type virtual-router` i przypisać do niej logiczny interfejs. Nie jest to jeszcze MPLS L3VPN; tutaj nie potrzebujemy automatycznie `route-distinguisher` ani `vrf-target`.

W labie dodatkowym mamy osobny port `ge-0/0/2.0` i sieć `198.18.30.0/24`, **nieużywaną w podstawowej topologii 6.11**. Jest to przykład osobny, którego nie należy doklejać do aktywnej konfiguracji produkcyjnej.

**Urządzenie: LAB-R3-vSRX — Configuration Mode [edit private], dodatkowy izolowany eksperyment virtual-router**

~~~junos
set interfaces ge-0/0/2 unit 0 family inet address 198.18.30.1/24
set routing-instances LAB-IOT instance-type virtual-router
set routing-instances LAB-IOT interface ge-0/0/2.0
show | compare
commit check
~~~

**Co chcemy osiągnąć?** Po commit połączona sieć `198.18.30.0/24` powinna znaleźć się w **`LAB-IOT.inet.0`**, a nie w globalnej `inet.0`. Jeżeli urządzenie jest SRX, przypisanie interfejsu do instancji routingu nie zastępuje konfiguracji **security zone**, policy i host-inbound. Na EX/QFX interfejs może być IRB z odpowiednim VLAN.

**Urządzenie: LAB-R3-vSRX — Operational Mode, po świadomym zatwierdzeniu osobnego LAB**

~~~junos
show route table LAB-IOT.inet.0
show route table inet.0
show configuration routing-instances LAB-IOT | display set
show interfaces ge-0/0/2 terse
~~~

**Podchwytliwe pytanie:** czy samo `set routing-instances LAB-IOT instance-type virtual-router` wystarczy, by pojawiły się trasy do konkretnych interfejsów? **Nie** — trzeba przypisać interfejsy/protokoły/trasy w odpowiednim kontekście. Podobnie jak w Cisco tworzenie pustej VRF nie przenosi do niej wszystkich portów.

Źródło: [Virtual Routing Instances on EX](https://www.juniper.net/documentation/us/en/software/junos/multicast-l2/topics/topic-map/virtual-routing-instances.html), [Routing Instances Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html).

### 6.6.3. Static route w virtual-router nie powinna „przypadkiem” trafić do global

Załóżmy, że LAB-IOT ma własną bramę `198.18.30.254`. Chcemy default **tylko w tej instancji**, bez dotykania globalnej `inet.0`.

**Urządzenie: LAB-R3-vSRX — Configuration Mode [edit private], wyłącznie gdy next-hop 198.18.30.254 faktycznie istnieje**

~~~junos
set routing-instances LAB-IOT routing-options static route 0.0.0.0/0 next-hop 198.18.30.254
show | compare
commit check
~~~

**Urządzenie: LAB-R3-vSRX — Operational Mode, test właściwej tabeli**

~~~junos
show route table LAB-IOT.inet.0
show route table inet.0
ping 198.18.30.254 routing-instance LAB-IOT count 5
~~~

Jeżeli w labie nie ma hosta/bramy `198.18.30.254`, **nie zaliczaj ping jako PASS**. Co więcej, sama aktywna default route w instancji nie daje automatycznie łączności z globalną siecią — do tego potrzebujesz jawnej drogi i odpowiedniego routingu po stronie następnego urządzenia.

**Gotcha:** w Cisco łatwo zapomnieć `vrf` w extended ping. W Junos równie łatwo wykonać `ping` w niewłaściwej instancji. Błędna próba z `inet.0` nie dowodzi awarii VRF.

### 6.6.4. `virtual-router` vs `vrf` vs `forwarding` — nie są synonimami

**Virtual-router** daje niezależny kontekst routingu, odpowiedni np. dla prostej izolacji Enterprise.

**VRF** w terminologii Junos jest również **konkretnym instance-type** często związanym z usługami VPN i wymaganiami RD/import/export, a nie jedynie potocznym synonimem każdej izolowanej tablicy.

**Forwarding instance** ma zastosowania specjalne, m.in. Filter-Based Forwarding. Nie budujemy tu standardowej tablicy klienta jak przy `virtual-router`.

**No-forwarding** może służyć do utrzymywania informacji routingu bez klasycznego przekazywania transit w danej instancji — znów inny model.

**Most z Cisco:** VRF-Lite, MPLS L3VPN i PBR/route-map mogą dotyczyć izolacji i kierowania ruchu, ale na Junos realizuje się je **różnymi konstrukcjami**. Złożone route leaking, RIB groups, FBF i VPN omawiamy w rozdziale 7 oraz kolejnych.

### 6.6.5. Dlaczego niezależna instancja nie oznacza automatycznego firewall segmentation?

Dwie routing instances mogą odizolować **wybór tras**, ale nie zastępują security policy/NAC/firewall rules. Na SRX przepływ jest rozstrzygany również przez strefy i sesje. Na EX instancje nie zapewniają samoczynnie tego samego zestawu zabezpieczeń co firewall.

Przy projektowaniu sieci produkcyjnej rozdziel:
1. Routing table separation.
2. Inter-VRF communication / route leaking.
3. Security enforcement.
4. Management access / observability.

**Szczególnie ważny gotcha:** `system management-instance` i `mgmt_junos` są **specjalnym mechanizmem zarządzania**. Nie traktuj `mgmt_junos` jak dowolnej virtual-router, którą przeniesiesz do innego context jednym poleceniem. Zmiana management routing może jednocześnie odciąć SSH, TACACS+, NTP i syslog — w rozdziale 4 opisaliśmy dlaczego.

~~~mermaid
flowchart TB
  R["Jedno urządzenie Junos"] --> G["Global: inet.0 / inet6.0"]
  R --> A["LAB-IOT: LAB-IOT.inet.0"]
  R --> B["CUSTOMER-B: CUSTOMER-B.inet.0"]
  R --> M["Opcjonalne OOB: mgmt_junos.inet.0"]
  A -. "Brak automatycznego leaking" .-> G
  B -. "Brak automatycznego leaking" .-> A
~~~

**Na pamięć:** `routing-instances NAME instance-type virtual-router`, `interface`, `routing-options`, `show route table NAME.inet.0`.  
**Do zrozumienia:** routing instance jest kontekstem decyzji, a nie samym ACL.  
**Do przećwiczenia:** utwórz osobną tabelę na dodatkowym portcie w odseparowanym labie i pokaż, że jej trasy nie pojawiły się automatycznie w `inet.0`.

---

## 6.7. Connected, static i default routes — trzy rodzaje wiedzy o drodze [A]

### 6.7.1. Connected route nie jest poleceniem `ip route`

Na Cisco, gdy przypisujesz `10.10.10.1/24` do interfejsu L3, system tworzy connected `10.10.10.0/24` i local host route `10.10.10.1/32`. Junos ma analogiczny mechanizm, lecz konfigurujesz **`interfaces ... unit ... family inet address`**.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], izolowany LAB IPv4**

~~~junos
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/24
show | compare
commit check
~~~

Po prawidłowym commit, gdy logical interface spełnia warunki aktywności, powinna pojawić się:
- trasa bezpośredniej sieci `192.0.2.0/24` z source `Direct/0`;
- trasa lokalnego adresu interfejsu `192.0.2.1/32` z `Local/0`.

**Gotcha:** `192.0.2.1/32` local nie jest dynamiczną trasą, którą Junos „otrzymał z sieci”. To lokalny adres hosta routera. Za to destination do `192.0.2.10` może pasować do connected /24 i wymagać ARP.

**Urządzenie: R1-vSRX — Operational Mode, po commit i aktywacji portu**

~~~junos
show interfaces ge-0/0/1 terse
show route protocol direct
show route protocol local
show route 192.0.2.0/24
show route 192.0.2.1/32
show arp
~~~

Zwróć uwagę, że **interfejs administracyjnie down** może uniemożliwić pojawienie się normalnej aktywnej connected route. Sam adres zapisany w config nie gwarantuje działania.

### 6.7.2. Static route to świadome polecenie administratora

Chcemy osiągnąć sieć `203.0.113.0/24` za sąsiednim R2 o adresie `198.51.100.2`.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], prosty IPv4 static**

~~~junos
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.2
show | compare
commit check
~~~

Odpowiednik ideowy Cisco:

**Urządzenie: CISCO-R1 — Cisco IOS XE Global Configuration Mode, porównanie składni**

~~~cisco
ip route 203.0.113.0 255.255.255.0 198.51.100.2
~~~

Na Junos konfiguracja trafia do candidate; na typowym Cisco IOS XE polecenie konfiguracji może wpłynąć na running state od razu. To różnica sposobu zarządzania, nie działania IP forwarding.

**Gotcha:** static route do next-hop, który nie jest osiągalny jako kwalifikowany next-hop, może nie pojawić się jako active. Junos nie musi od razu powiedzieć „syntax error”; składnia IP jest poprawna, ale logiczna droga może być nieosiągalna. [Configure Static Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config_static-routes.html).

### 6.7.3. Default route to route do wszystkiego — gdy nie znam nic bardziej szczegółowego

Wyobraź sobie niewielki oddział, który ma tylko jedno wyjście WAN. Nie musi znać tysiąca prefiksów centrali. Wystarczy default `0.0.0.0/0` do R1.

**Urządzenie: R2-vSRX — Configuration Mode [edit private], IPv4 default przez R1**

~~~junos
set routing-options static route 0.0.0.0/0 next-hop 198.51.100.1
show | compare
commit check
~~~

**Co nie wynika z tego polecenia?** Że R1 zna sieć docelową, że R1 jest Internet gateway albo że R2 będzie miał ruch NATowany. Default route to tylko **instrukcja next-hop** dla prefiksów, które nie mają bardziej szczegółowej trasy. Nie generuje automatycznie security policies ani NAT.

### 6.7.4. Brak default u R1, default u R2 — poprawny i często spotykany projekt

W labie R1 pełni rolę „provider side” i ma statyczną drogę **do konkretnej sieci R2**. R2 pełni rolę „branch” i ma default **do R1**. To asymetria konfiguracji routingu, ale **nie problem asymetrycznego forwardingu**, ponieważ obie strony nadal mają poprawne trasy do siebie.

**R1:** `203.0.113.0/24 via 198.51.100.2`.  
**R2:** `0.0.0.0/0 via 198.51.100.1`.

Pamiętaj, że R2 posiada także własną connected `203.0.113.0/24` i transit `198.51.100.0/30` — dla takich destinations LPM preferuje connected, nie default.

**Urządzenie: R2-vSRX — Operational Mode, oczekiwane konsekwencje**

~~~junos
show route 0.0.0.0/0
show route 192.0.2.10
show route 203.0.113.0/24
~~~

### 6.7.5. Dual-stack: static IPv6 to nie dodatkowa opcja na IPv4 route

Na Junos IPv6 unicast static route konfigurujesz w odpowiedniej RIB, typowo przez **`routing-options rib inet6.0 static`**. To jest ważna różnica składni względem zwykłego IPv4 `routing-options static`.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], interfejs i IPv6 static do sieci R2**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::1/64
set routing-options rib inet6.0 static route 2001:db8:20::/64 next-hop 2001:db8:12::2
show | compare
commit check
~~~

**Urządzenie: R2-vSRX — Configuration Mode [edit private], interfejs i IPv6 default do R1**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::2/64
set routing-options rib inet6.0 static route ::/0 next-hop 2001:db8:12::1
show | compare
commit check
~~~

**Most z Cisco:** `ipv6 route 2001:db8:20::/64 2001:db8:12::2` działa na podobnej zasadzie; w Junos jawnie wskazujesz `rib inet6.0`. [Juniper official IPv4/IPv6 static examples](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config_static-routes.html).

**Gotcha:** ruch IPv6 wymaga także poprawnej **ND (Neighbor Discovery)** i IPv6 security policy. Nie zakładaj, że działający IPv4 ARP/ping dowodzi IPv6 connectivity.

**Urządzenie: R1-vSRX — Operational Mode, po konfiguracji IPv6**

~~~junos
show route table inet6.0
show ipv6 neighbors
ping 2001:db8:12::2 count 5
~~~

### 6.7.6. Weryfikacja static route — co dokładnie sprawdzać?

**Urządzenie: R1-vSRX — Operational Mode, po commit**

~~~junos
show configuration routing-options | display set
show route protocol static
show route 203.0.113.0/24 detail
show route forwarding-table destination 203.0.113.10
show interfaces ge-0/0/0 terse
show arp
~~~

**Przewidywanie wyniku:** jeśli interfejs transit działa i next-hop jest prawidłowy, powinna pojawić się active static route. Ale nie oznacza to jeszcze, że SRX security policy przepuszcza PC-A → PC-B. Stąd dodatkowe `show security flow session` i test z klienta w labie 6.11.

**Na pamięć:** IPv4 static `routing-options static route`, IPv6 static `routing-options rib inet6.0 static route`; default `0.0.0.0/0` i `::/0`.  
**Do zrozumienia:** connected/local są skutkiem aktywnego adresu interfejsu, static — intencją administratora, default — najkrótszym dopasowaniem.  
**Do przećwiczenia:** wyłącz port tranzytowy i porównaj status connected, static, FIB oraz sąsiadów.

