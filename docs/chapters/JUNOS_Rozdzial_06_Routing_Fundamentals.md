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



---

## 6.8. Dynamic routing — dlaczego przy trzecim routerze zaczyna się rachunek kosztów administracyjnych? [A]

### 6.8.1. Jedna trasa statyczna jest prosta. Trzysta tras statycznych to projekt utrzymaniowy.

W małym biurze masz R1, R2 i dwie sieci. Statyczne `set routing-options static route` jest przejrzyste, bez wymiany protokołów i bez dodatkowej infrastruktury.

Wyobraź sobie teraz firmę z 24 lokalizacjami. Każda dodaje VLAN, zmienia adresację albo dostaje drugi uplink WAN. W którym momencie administracja statycznymi trasami zamienia się w ryzyko?

Za każdym razem, gdy nowy prefiks pojawi się w oddziale, trzeba zapewnić **jego rozpoznawalność w odpowiednich tablicach, poprawne trasy zwrotne i reakcję na awarie**. Bez automatycznej wymiany informacji możesz mieć dziesiątki zmian do wykonania w różnych miejscach, a jedna zapomniana trasa zwrotna popsuje usługę.

Właśnie dlatego istnieją dynamiczne protokoły routingu. **Nie podejmują decyzji o trasie dla każdego pakietu**, lecz wymieniają informacje służące do budowania RIB.

### 6.8.2. OSPF, IS-IS i BGP nie są trzema zamiennymi nazwami „dynamic route”

**OSPF** to protokół link-state, popularny wewnątrz Enterprise. Routery budują wspólne wyobrażenie topologii w ramach odpowiednich obszarów, liczą SPF i instalują trasy. W Junos zobaczysz `show ospf neighbor`, `show ospf database` i wynik w `show route protocol ospf`. Konfiguracja i detale LSA/area type stanowią rozdział 9.

**IS-IS** jest również link-state i ważny w bardziej rozbudowanych sieciach, częściej SP/core. Ma Level 1 i Level 2 oraz własny model LSDB. Szczegółowy kurs jest w rozdziale 10.

**BGP** to path-vector, przydatny do wymiany prefiksów pomiędzy systemami autonomicznymi, ale również w dużych sieciach Enterprise, data center i WAN. BGP wybiera ścieżki przez atrybuty i policy, a wynik jest porównywany z innymi route sources. Rozdziały BGP rozwiną Local Preference, AS_PATH, MED, communities i route reflection.

**Most z Cisco:** protokoły IP nie zmieniają podstaw dlatego, że pojawia się Junos. Natomiast **hierarchia konfiguracji, default preferences i routing-policy import/export** są inne.

### 6.8.3. IGP kontra EGP — kontekst, nie tylko rozwinięcie skrótów

**IGP** (Interior Gateway Protocol) służy do wymiany wiedzy w granicach kontrolowanej domeny routingu — zwykle OSPF lub IS-IS. **EGP** w praktyce nowoczesnych Enterprise to przede wszystkim BGP do sąsiedniej domeny/AS, chociaż iBGP działa również wewnątrz AS.

Na przykład:
- wewnątrz kampusu: OSPF dla core/distribution;
- z dostawcą Internetu: BGP na WAN edge, jeśli jest odpowiednia potrzeba;
- mały, pojedynczy branch: statyczna default route do SRX;
- duży SD-WAN/cloud: mechanizmy dystrybucji tras zależne od konkretnej architektury i platformy.

Nie wyprowadzaj jednak reguły „każdy kampus musi mieć OSPF”, bo nowoczesne EVPN-VXLAN i routed access zmieniają projekty, a niektóre biura świetnie działają na static routing.

### 6.8.4. Dynamika nie oznacza natychmiastowej konwergencji

Wyobraź sobie awarię primary link w sieci OSPF. Najpierw trzeba wykryć zmianę, zaktualizować relacje/topologię, obliczyć najlepszą drogę i zaprogramować forwarding. To zajmuje czas zależny od platformy, timerów i mechanizmów szybkiego wykrywania awarii.

Statyczna trasa też nie jest „niezniszczalna”: przy fizycznym link down związany connected next-hop może stać się niekwalifikowany i route może zniknąć z FIB. Ale jeśli link do lokalnego routera ISP działa, a problem występuje **dalej** w jego sieci, sama statyczna trasa może wciąż być aktywna i prowadzić do blackhole.

**Gotcha:** „link up” + „static active” nie dowodzi **end-to-end reachability**. To samo dotyczy dynamicznych protokołów, jeżeli błędnie ogłaszają osiągalność lub istnieje problem na warstwie aplikacji.

### 6.8.5. Protokół dynamiczny dodaje własny rodzaj awarii

Zamiast utrzymywania setek statycznych wpisów musisz utrzymywać:
- sąsiedztwa i stany sesji;
- timers i authentication;
- polityki import/export;
- filtrowanie prefiksów;
- LSDB lub BGP Adj-RIB-In/Out;
- zgodność route preference i redistribucji;
- monitoring konwergencji oraz skalowalności RE/PFE.

To jest **trade-off**, nie darmowy bonus. Jeśli nie umiesz przewidzieć zachowania statycznej route i jej next-hopa, konfiguracja BGP nie rozwiąże problemu, tylko go ukryje.

### 6.8.6. Od statycznego labu do protokołów

**Urządzenie: R1-vSRX — Operational Mode, odczyty przydatne także przy routingu dynamicznym**

~~~junos
show route summary
show route protocol static
show route protocol ospf
show route protocol bgp
show interfaces terse
show system processes extensive
~~~

Jeżeli nie skonfigurowaliśmy OSPF i BGP, odpowiednie outputy mogą być puste — **jest to zamierzone**. W tym rozdziale **nie uruchamiamy protokołów dynamicznych tylko po to, by zobaczyć sąsiadów**, bo najpierw trzeba umieć odróżnić route selection, LPM i next-hop resolution.

**Na pamięć:** OSPF/IS-IS to IGP link-state; BGP to path-vector.  
**Do zrozumienia:** protokół wymienia informacje budujące RIB; nie zastępuje PFE ani polityk security.  
**Do przećwiczenia:** wskaż w swoim HLD, gdzie static jest uzasadniona, gdzie dynamic routing ma sens i jaką awarię musi wykrywać.

---

## 6.9. Konfiguracja i monitorowanie podstawowych funkcji routingu [A]

### 6.9.1. Nie zaczynaj od komendy — zacznij od opisu ruchu

Zadanie od architekta: „Z biura A ma działać dostęp do sieci ERP `203.0.113.0/24` przez R2, a sieć serwerowa ma wiedzieć, jak odpowiedzieć do `192.0.2.0/24`. Dodatkowo sprawdź IPv6”.

Jak dobry inżynier odpowiesz? Nie „ustawię route”, tylko rozbijesz wymaganie na siedem warunków:

1. Czy interfejsy R1/R2 są w odpowiednim L3 kontekście?
2. Czy connected route do transit jest aktywna?
3. Czy R1 zna destination ERP?
4. Czy R2 zna return path PC-A (lub ma właściwą default)?
5. Czy next-hop IP jest poprawny?
6. Czy ARP/ND działa w obu kierunkach?
7. Czy ruch aplikacyjny jest dozwolony na SRX (security policy), a host firewall go przyjmuje?

To jest jeden z najważniejszych wzorców troubleshootingu, który przeniesiesz na OSPF i BGP.

### 6.9.2. Komendy weryfikacyjne ułożone jako łańcuch przyczynowy

**Urządzenie: R1-vSRX — Operational Mode, pierwsza warstwa: interfejsy i RIB**

~~~junos
show version
show interfaces terse
show configuration interfaces | display set
show route protocol direct
show route protocol local
show route protocol static
show route 203.0.113.0/24 detail
~~~

**Urządzenie: R1-vSRX — Operational Mode, druga warstwa: forwarding i sąsiad**

~~~junos
show route forwarding-table destination 203.0.113.10
show arp
show ipv6 neighbors
show route table inet6.0
~~~

**Urządzenie: R1-vSRX — Operational Mode, trzecia warstwa: ruch kontrolowany i SRX flow**

~~~junos
ping 198.51.100.2 source 198.51.100.1 count 5
ping 203.0.113.10 source 192.0.2.1 count 5
show security flow session
show security policies
~~~

Jeżeli ping ze `source 192.0.2.1` się nie powiedzie, nie ignoruj go tylko dlatego, że ping z `198.51.100.1` działa. Pierwszy symuluje **inną sieć źródłową i return path**, chociaż nadal nie jest identyczny z ruchem z PC-A: jest generowany przez RE, z innym kontekstem security. **Test końcowy musi pochodzić z klienta.**

### 6.9.3. Niektóre polecenia nie dają tego samego dowodu, choć wyglądają podobnie

**`show configuration routing-options`** odpowiada „co zostało zapisane?”.  
**`show route protocol static`** odpowiada „co rpd zna z static?”.  
**`show route ... detail`** odpowiada „czy to active i jaki next-hop?”.  
**`show route forwarding-table`** odpowiada „jaką forwarding instruction reprezentuje ten host/prefix?”.  
**`show arp`** odpowiada „czy znam MAC sąsiada IPv4?”.  
**`show security flow session`** odpowiada „co SRX wie o flows?”.  
**`ping z PC`** odpowiada „czy klient widzi odpowiedź ICMP?”.  
**`Test-NetConnection -Port 443`** odpowiada „czy klient otwiera TCP do aplikacji?”.

Nie zastępuj ostatnich dwóch pojedynczym `show route`.

### 6.9.4. Typowy błąd: zmiana jest w candidate, a operator diagnozuje active

W Junos `set routing-options static ...` nie działa natychmiast. Jeśli w trybie `configure private` wprowadziłeś nową trasę, ale nie zatwierdziłeś jej, **operational `show route` nie musi nic o niej wiedzieć**. To osobna kategoria problemu niż brak next-hopa.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], przed zmianą**

~~~junos
show | compare
commit check
~~~

Jeżeli diff pokazuje nowe `static route`, a `show route` nie, najpierw ustal, **czy commit rzeczywiście nastąpił**. Nie włączaj BGP, żeby „nauczyć” router trasy, która wciąż leży w candidate.

### 6.9.5. Odczytaj szczegóły, zamiast zakładać, że `show route` zawsze wyświetla tylko active

**Urządzenie: R1-vSRX — Operational Mode, weryfikacja wyboru**

~~~junos
show route table inet.0
show route 203.0.113.0/24 extensive
show route hidden
show route summary
show system commit
~~~

**Wniosek:** destination count może różnić się od routes count; zwracaj uwagę na `active`, `hidden`, next-hop, protocol source, preference i preference2. Podczas incydentu spróbuj odtworzyć **przyczynę decyzji**, a nie tylko stwierdzić „route jest”.

### 6.9.6. Podstawowy runbook RIB/FIB/ARP — objaw „brak łączności”

~~~mermaid
flowchart TD
  FAIL["PC-A nie dociera do ERP"] --> LINK{"Interfejsy up?\nConnected route obecne?"}
  LINK -->|Nie| PHY["L1/L2/port, cabling, VLAN, unit"]
  LINK -->|Tak| RIB{"Active route i właściwa tabela?"}
  RIB -->|Nie| ROUTE["Static/default/instance,\nnext-hop resolve"]
  RIB -->|Tak| FIB{"Forwarding-table i ARP/ND?"}
  FIB -->|Nie| NBR["Link/peer/neighbor adjacency"]
  FIB -->|Tak| FW{"SRX flow/policy i return path?"}
  FW -->|Nie| SEC["Zones/policy/NAT/serwer"]
  FW -->|Tak| APP["End-to-end klient TCP/aplikacja"]
  PHY --> RIB
  ROUTE --> FIB
  NBR --> FW
  SEC --> APP
~~~

**Most z Cisco:** operacyjny przebieg jest podobny do diagnostyki `show ip interface brief` → `show ip route` → `show ip cef` → `show arp` → ACL/CoPP/Firewall. W Junos trzeba jednak pamiętać o `routing-instances` i oddzielnej warstwie SRX flow.

### 6.9.7. Przyporządkowanie wiedzy do ścieżki certyfikacji

**JNCIA [A]**: rozumiesz connected, static, default, preference, LPM, RIB/FIB, polecenia `show` i podstawowe `routing-instances`.

**JNCIS-ENT [S]**: samodzielnie konfigurujesz i troubleshootujesz floating/qualified static, trasy specjalne, next-hop resolve; przygotowujesz się do protocol-independent routing, policies i OSPF/IS-IS/BGP.

**JNCIP-ENT [P]**: wykrywasz skutki konfiguracji w dużej topologii, analizujesz interplay RIB groups/FBF, policies, ECMP, BGP best-path, OSPF external i FIB; sprawdzasz rzeczywisty dataplane oraz projekt failover end-to-end.

Nie twierdzimy, że szczegółowe warianty JNCIP już pojawiają się dosłownie jako pytania w JNCIA. Zamiast tego budujemy **jedną spójną drogę uczenia się**.



---

## 6.10. Static routes — warianty i troubleshooting [S]

Ta część to **pomost do JNCIS-ENT**. Zwykłe `set routing-options static route ... next-hop ...` już znasz. Teraz odpowiemy na trudniejsze pytanie: **co stanie się z trasą, kiedy główny link przestanie działać, ktoś doda bardziej szczegółowy prefiks albo upstream blackholuje ruch?**

### 6.10.1. Dwa next-hopy — skąd Junos wie, który jest primary?

Przed sobą masz R1 połączony dwoma niezależnymi drogami do sieci ERP. Jedna idzie przez ISP-A, druga przez ISP-B. Obie prowadzą do tej samej destynacji `203.0.113.0/24`.

~~~mermaid
flowchart LR
  R1["R1\nbiuro"] -->|"WAN-A\nnext-hop 198.51.100.2"| A["R2 / ISP-A"]
  R1 -->|"WAN-B\nnext-hop 198.51.100.6"| B["R3 / ISP-B"]
  A --> ERP["ERP\n203.0.113.0/24"]
  B --> ERP
~~~

Jeśli tylko dopiszesz dwie równorzędne drogi, możesz uzyskać zachowanie inne niż oczekiwane „primary/backup”. Oficjalna dokumentacja Junipera tłumaczy, że w przypadku kilku next-hopów dla static route bez precyzyjnego ustalenia priorytetu **Junos może wybrać jeden z nich do FIB**, a nie automatycznie stosować deterministyczny primary/standby albo ECMP.

Dla świadomego wyboru właściwym mechanizmem jest **`qualified-next-hop`**, który pozwala nadać każdemu next-hopowi oddzielne parametry, m.in. preference. Źródło: [Static Route Preferences and Qualified Next Hops](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html).

### 6.10.2. `qualified-next-hop` — dwa progi dla jednej drogi

Zakładamy **osobny** laboratorium three-router albo istniejący drugi transit link:
- WAN-A: R1 `198.51.100.1/30` ↔ R2 `198.51.100.2/30`;
- WAN-B: R1 `198.51.100.5/30` ↔ R3 `198.51.100.6/30`;
- za R2/R3 dostępna jest sieć ERP.

**Urządzenie: R1-EDGE — Configuration Mode [edit private], rozszerzony LAB z DWOMA rzeczywistymi next-hopami**

~~~junos
set routing-options static route 203.0.113.0/24 qualified-next-hop 198.51.100.2 preference 5
set routing-options static route 203.0.113.0/24 qualified-next-hop 198.51.100.6 preference 25
show | compare
commit check
~~~

**Co tu się dzieje?** Dla tego samego prefixu preferowany jest kwalifikowany next-hop A o wartości 5, o ile jest eligible. Jeżeli przestanie być kwalifikowany, Junos może wybrać zapasowy B o preference 25. Ale to **nie oznacza**, że test aplikacji ERP automatycznie wykryje awarię gdzieś dalej niż lokalna osiągalność next-hopa.

**Gotcha:** `qualified-next-hop preference 25` to nie to samo co globalna route preference 25 bez przypisania do next-hopa. Możesz ustawić globalną preference dla całej route, a następnie nadpisać ją dla konkretnych next-hopów. Nie mieszaj tych dwóch poziomów.

**Urządzenie: R1-EDGE — Operational Mode, odczyt po zatwierdzeniu w oddzielnym LAB**

~~~junos
show route 203.0.113.0/24 extensive
show route forwarding-table destination 203.0.113.10
show interfaces terse
show arp
~~~

Jeśli na R1 nie ma WAN-B ani interfejsu z reachability do `198.51.100.6`, **nie commituj przykładu jako testu failover**. Skonfigurowanie abstrakcyjnego backup address bez sieci nie jest prawdziwym scenariuszem redundancy.

### 6.10.3. Czy static route potrafi rozpoznać awarię poza bezpośrednim sąsiadem?

Odpowiedź brzmi: nie zawsze. Wyobraź sobie:
- R1 → ISP-A działa fizycznie;
- ISP-A potrafi odpowiedzieć na ARP i ping swojego next-hopa;
- ale za ISP-A uszkodzony jest dalszy odcinek do ERP.

Z punktu widzenia R1 **next-hop nadal jest osiągalny**, więc prosta static route może pozostać aktywna i blackholować traffic.

To dlatego w realnym redundantnym WAN często łączysz static routing z mechanizmami detekcji:
- **BFD** do odpowiedniego peer, jeśli obie platformy i usługa wspierają;
- **RPM/IP monitoring** do testu odleglejszego punktu — ostrożnie, by nie zbudować błędnego failover na podstawie jednego ICMP;
- **SLA tracking**, jeśli wspierany na konkretnym modelu/wydaniu;
- dynamiczny routing (OSPF/BGP) z odpowiednimi timerami lub BFD.

**Gotcha:** BFD do bezpośredniego next-hopa też nie wykrywa magicznie awarii serwera aplikacji 200 km dalej. Każdy mechanizm mierzy **określony zakres** dostępności. Źródło: [BFD for Static Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/bfd-static-routes.html).

### 6.10.4. `resolve` — rekursywny next-hop nie jest domyślnie tym samym co w Cisco

Znasz z Cisco static route, której next-hop nie leży w directly connected subnet, ale router może ją rozwiązać dzięki innym wpisom w RIB. W Junos standardowa static route typowo oczekuje bezpośrednio rozwiązywalnego next-hopa. Gdy wskazujesz **indirect next-hop**, może być potrzebne jawne `resolve`.

Przykład: R1 ma trasę IGP do `198.51.100.200/32` i chce użyć tego adresu jako pośredniego next-hopa dla `203.0.113.0/24`.

**Urządzenie: R1-EDGE — Configuration Mode [edit private], przykład KONCEPCYJNY wymagający istniejącej drogi do 198.51.100.200**

~~~junos
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.200
set routing-options static route 203.0.113.0/24 resolve
show | compare
~~~

Bez prawidłowej ścieżki do indirect next-hop przykład nie tworzy działającego forwardingu. To nie jest komenda do podstawowego labu 6.11, gdzie `198.51.100.2` jest bezpośrednim sąsiadem.

**Niewygodny gotcha:** `resolve` pozwala rozstrzygnąć next-hop przez inne wpisy RIB; jeśli rozwiązywanie bazuje na nieodpowiednim default albo tworzy nieoczekiwany łańcuch zależności, routing może być błędny. Po wdrożeniu zawsze badaj **resolved next-hop i egress**, nie tylko samą obecność statycznej route.

Kolejny oficjalny wyjątek: **`resolve` i `retain` nie mogą być skonfigurowane równocześnie na tej samej static route**. Wynika to z natury resolved next-hop: nie można obiecać zachowania bez rekalkulacji stanu, którego podstawą jest inna trasa. [resolve CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/resolve-edit-routing-options.html), [retain](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/retain-edit-routing-options.html).

### 6.10.5. `discard` i `reject` — czy nadawca ma się dowiedzieć o odrzuceniu?

Czasem potrzebujemy specjalnej trasy, która **świadomie nie kieruje ruchu dalej** — na przykład bezpieczny aggregate blackhole do ochrony przed pętlami.

**Urządzenie: R1-EDGE — Configuration Mode [edit private], izolowany lab, świadomy blackhole**

~~~junos
set routing-options static route 198.18.0.0/15 discard
show | compare
~~~

**`discard`** — pakiety pasujące do trasy są odrzucane bez typowego ICMP unreachable.  
**`reject`** — pakiety są odrzucane, a system wysyła odpowiedni komunikat ICMP/ICMPv6 unreachable, z zastrzeżeniem ograniczeń platformy i filtrowania.

**Urządzenie: R1-EDGE — Configuration Mode [edit private], alternatywa do osobnego eksperymentu, NIE łącz równocześnie z discard dla tego samego prefiksu**

~~~junos
delete routing-options static route 198.18.0.0/15 discard
set routing-options static route 198.18.0.0/15 reject
show | compare
~~~

**Most z Cisco:** droga Null0 przypomina discard (bez analogicznej gwarancji sposobu generowania ICMP na wszystkich platformach). W Junos odróżnij **routing-options static reject** od **routing policy action reject**, ponieważ to dwa zupełnie różne zastosowania słowa „reject”: pierwsze odrzuca **pakiety dopasowane do route**, drugie odrzuca **kandydata na trasę** w procesie polityki.

**Gotcha w praktyce:** bardziej szczegółowy `discard /26` wygra przez LPM z default /0, nawet jeśli default ma świetną preference. Wyjaśniliśmy to w 6.5; wróć tam, jeśli intuicja mówi „ale static powinien przejść defaultem”.

Źródła: [discard statement](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/discard-edit-routing-options.html), [Static Route Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/static-edit-routing-options.html).

### 6.10.6. `no-readvertise` — czego **nie** zrobi sama static route?

Wyobraź sobie sieć z OSPF, która eksportuje wybrane trasy statyczne. Chcesz zachować techniczną trasę lokalnie, ale **nigdy nie rozgłaszać jej do innych routerów**.

**Urządzenie: R1-EDGE — Configuration Mode [edit private], tylko przy świadomym projekcie redystrybucji**

~~~junos
set routing-options static route 198.18.0.0/15 discard
set routing-options static route 198.18.0.0/15 no-readvertise
show | compare
~~~

**Gotcha:** standardowe trasy statyczne **nie zaczynają same z siebie rozgłaszać się przez OSPF lub BGP**. Do tego zwykle potrzebna jest odpowiednia eksportowa routing policy. `no-readvertise` wpływa na kwalifikację tej statycznej trasy do readvertisement; **nie jest poleceniem „wyłącz cały BGP”**. To częsty błąd przy migracji Cisco route-map/redistribution do Junos.

Z drugiej strony `no-readvertise` nie oznacza, że route zostanie usunięta z lokalnej RIB/FIB. Nadal może działać lokalnie.

Źródło: [readvertise/no-readvertise](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/readvertise-edit-routing-options.html).

### 6.10.7. `retain` — czy trasa może przeżyć restart `rpd`?

W niektórych szczególnych sytuacjach chcesz zachować static route w forwarding table, nawet gdy proces routingu `rpd` zostaje zatrzymany. Junos posiada **`retain`**. Nie należy jednak interpretować tego jako „gwarantowanego dostępu po awarii zasilania”.

**Urządzenie: R1-EDGE — Configuration Mode [edit private], pokaz mechanizmu, NIE stosować bez analizy platformy**

~~~junos
set routing-options static route 198.18.0.0/15 discard
set routing-options static route 198.18.0.0/15 retain
show | compare
~~~

Według dokumentacji `retain` dotyczy zachowania informacji w forwarding table podczas pewnych zdarzeń restartu `rpd`/routingu. To **nie** jest mechanizm, który utrzyma fizyczny port up po odłączeniu światłowodu ani nie zastępuje GRES/NSR. Jego dokładna semantyka zależy od platformy, a użycie jest eksperckie.

**Uwaga bezpieczeństwa:** wpis `discard` to trasa blackhole do dokumentacyjnego zakresu; w produkcji z innymi prefiksami mógłby spowodować nieplanowany outage. Pozostaw go wyłącznie w odizolowanym ćwiczeniu.

### 6.10.8. `passive` — pozwól trasie pozostać nawet, gdy next-hop jest nieosiągalny?

Dokumentacja Junos opisuje także `passive` dla statycznych tras. Może utrzymywać wpis route w RIB nawet wtedy, gdy next-hop jest niedostępny, a skutkiem może być **odrzucanie ruchu** do celu.

To zachowanie stoi w sprzeczności z intuicją „trasa jest active, więc na pewno zadziała”. Używaj tego typu ustawień wyłącznie do konkretnych, zrozumianych scenariuszy. Nie dodajemy `passive` do produkcyjnego runbooka „żeby trasa nie znikała”, bo możesz maskować rzeczywisty problem reachability.

**Urządzenie: R1-EDGE — Configuration Mode [edit private], koncepcyjny przykład, nie stosuj na produkcji**

~~~junos
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.2
set routing-options static route 203.0.113.0/24 passive
show | compare
~~~

**Pytanie:** czy po utracie `198.51.100.2` klient ERP na pewno odzyska łączność, bo trasa nadal widnieje w table? **Nie.** Route visibility i realne forwarding reachability są odrębne.

### 6.10.9. `next-table` — kiedy forwarding wchodzi do kolejnej tablicy

Junos wspiera specjalny next-hop typu **`next-table`**. Taki route może kierować lookup do innej tablicy routingu. To już nie jest zwykła static route do adresu sąsiada.

**Gotcha:** przy źle zaprojektowanym `next-table` możesz zbudować nieoczekiwane forwarding loop między tablicami albo przekierować ruch do niewłaściwego tenant context. Nie uczę tutaj gotowej konfiguracji inter-VRF leaking bez pełnego projektu, security enforcement i testów — szczegóły należą do rozdziału 7 o RIB groups i FBF.

Dla JNCIS zapamiętaj tylko, że **next-hop nie musi zawsze oznaczać adresu IPv4**, może być także specjalnym działaniem (`discard`, `reject`, `next-table` itd.), ale **każdy ma inne skutki**.

### 6.10.10. Static route troubleshooting — pięć pytań, zanim użyjesz `delete`

**Scenariusz:** static `203.0.113.0/24` nie pojawia się jako active. Zacznij od pięciu pytań:

1. Czy konfiguruje się ją w `inet.0`, `inet6.0` czy `NAME.inet.0`?
2. Czy istnieje i jest aktywna **connected route do next-hopa** albo poprawna recursive resolution?
3. Czy dla tego samego prefiksu istnieje lepsza route preference?
4. Czy route jest hidden i jaki jest jej **reason**?
5. Czy nawet aktywna route ma prawidłowy ARP/ND i allowed SRX flow?

**Urządzenie: R1-vSRX — Operational Mode, diagnostyka static**

~~~junos
show configuration routing-options static | display set
show route 203.0.113.0/24 extensive
show route hidden
show route forwarding-table destination 203.0.113.10
show route 198.51.100.2
show arp
show security flow session
~~~

**Czego nie robisz:** nie dodajesz losowych statycznych tras do Internetu, nie wykonujesz `clear route` ani `restart routing` bez wskazania problemu, nie zmieniasz całej preference łącząc od razu z backup path.

### 6.10.11. Jedna tabela, pięć wariantów static route — po wykładzie ma już sens

| Wariant | Co zmienia | Najczęstszy błąd |
|---|---|---|
| `next-hop` | Wskazuje sąsiada | Adres niedostępny w kontekście RIB |
| `preference` | Pierwszeństwo route dla danego prefixu | Mylenie z LPM lub OSPF cost |
| `qualified-next-hop` | Parametry indywidualnych next-hopów | Zakładanie niezawodnego failover poza peer |
| `resolve` | Umożliwia indirect next-hop resolution | Nieoczekiwane powiązanie z inną route |
| `discard/reject` | Specjalne działania forwarding | Przypadkowy blackhole przez LPM |
| `no-readvertise` | Ogranicza kwalifikację do eksportu | Przekonanie, że statyczne route domyślnie same się ogłaszają |
| `retain/passive` | Specjalne zachowania retencji/osiągalności | Używanie jako substytutu zdrowego routingu |

**Na pamięć:** qualified preference, `resolve`, `discard`, `reject`, `no-readvertise` i ograniczenie resolve+retain.  
**Do zrozumienia:** wymiana jednej trasy na drugą jest tylko wtedy poprawna, gdy spełnione są warunki routing, next-hop, L2 i end-to-end service.  
**Do przećwiczenia:** osobno test `qualified-next-hop` w rozszerzonym labie, `discard` w izolacji oraz celowo niekwalifikowany next-hop. Żaden z tych eksperymentów nie wymaga dotykania produkcyjnego WAN.



---

## 6.11. [LAB] Konfiguracja IPv4/IPv6, tras statycznych i default routes na dwóch vSRX

### 6.11.1. Dlaczego ten lab ma osobne hosty po obu stronach?

Chcemy przećwiczyć **pełny forwarding end-to-end**, nie tylko ping pomiędzy dwoma adresami tranzytowymi routerów. W rozdziale 2 i 5 dowiedzieliśmy się, że ping generowany przez RE nie jest tym samym testem co ruch klienta. Dlatego instalujemy:
- **R1-vSRX**, **R2-vSRX** — dwa niezależne routery/firewalle Junos;
- **PC-A-LINUX** — lekki host Linux w sieci OFFICE;
- **PC-B-LINUX** — lekki host Linux w sieci ERP;
- **trzy odizolowane segmenty EVE**: OFFICE, TRANSIT, ERP;
- dostęp console EVE do każdego z routerów i hostów, **niezależny od interfejsów danych**.

Dlaczego Linux, a nie koniecznie dwa VPCS? VPCS świetnie nadaje się do prostych testów IPv4, ale w niektórych buildach obsługa **statycznego IPv6 default gateway** jest ograniczona i wymaga SLAAC. Aby uczciwie testować równoległe IPv4/IPv6, Linux z `ip route` i `ip -6 route` daje większą kontrolę.

**Status PLANNED.** Nie wykonałem tego labu w Twoim EVE-NG. Nie ma rzeczywistych outputów potwierdzających PASS, nie zakładam też mapowania NIC w konkretnym obrazie vSRX. Nie modyfikujemy CH02/CH03/CH04/CH05 ani istniejących laboratoriów Cisco.

~~~mermaid
flowchart LR
  subgraph OFFICE["OFFICE — IPv4 192.0.2.0/24\nIPv6 2001:db8:10::/64"]
   PC1["PC-A-LINUX\n192.0.2.10\n2001:db8:10::10"]
  end
  subgraph TRANSIT["TRANSIT — IPv4 198.51.100.0/30\nIPv6 2001:db8:12::/64"]
   R1["R1-vSRX\nge-0/0/1 OFFICE\nge-0/0/0 TRANSIT"]
   R2["R2-vSRX\nge-0/0/0 TRANSIT\nge-0/0/1 ERP"]
  end
  subgraph ERP["ERP — IPv4 203.0.113.0/24\nIPv6 2001:db8:20::/64"]
   PC2["PC-B-LINUX\n203.0.113.10\n2001:db8:20::10"]
  end
  PC1 --- R1
  R1 ---|"198.51.100.1 ↔ .2\n2001:db8:12::1 ↔ ::2"| R2
  R2 --- PC2
  OOB["Konsole EVE\nniezależne od data path"] -.-> R1
  OOB -.-> R2
~~~

### 6.11.2. Plan adresacji i rola każdej trasy

| Węzeł | Port / interface | IPv4 | IPv6 | Gateway / funkcja |
|---|---|---|---|---|
| PC-A-LINUX | eth0* | `192.0.2.10/24` | `2001:db8:10::10/64` | R1 `.1` / `::1` |
| R1-vSRX | ge-0/0/1.0 | `192.0.2.1/24` | `2001:db8:10::1/64` | OFFICE |
| R1-vSRX | ge-0/0/0.0 | `198.51.100.1/30` | `2001:db8:12::1/64` | TRANSIT |
| R2-vSRX | ge-0/0/0.0 | `198.51.100.2/30` | `2001:db8:12::2/64` | TRANSIT |
| R2-vSRX | ge-0/0/1.0 | `203.0.113.1/24` | `2001:db8:20::1/64` | ERP |
| PC-B-LINUX | eth0* | `203.0.113.10/24` | `2001:db8:20::10/64` | R2 `.1` / `::1` |

*Na Linux interface może nazywać się `ens3`, `ens4` itd. **Najpierw sprawdź `ip -br link` i dopiero potem podmień nazwę w komendach.*** Zakładane `ge-0/0/0` i `ge-0/0/1` także potwierdź na obu routerach. W wielu obrazach vSRX istnieje osobne `fxp0` jako management.

**Jeden warunek techniczny:** oba routery mają działać w kontrolowanym, izolowanym środowisku bez produkcyjnego NAT/zewnętrznych łączy. Korzystamy wyłącznie z dokumentacyjnych prefixów IPv4/IPv6 (RFC 5737 i RFC 3849).

### 6.11.3. Preflight — jak wykryć, czy bazowa konfiguracja jest naprawdę świeża

Nie wolno bez zastanowienia wgrywać konfiguracji na urządzenie, które już ma security zones, fabryczny NAT, DHCP server czy przypisane porty. Zacznij od read-only.

**Urządzenie: R1-vSRX — Operational Mode, niezależna konsola**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration interfaces | display set
show configuration routing-options | display set
show configuration security | display set
show system users
show system commit
~~~

**Urządzenie: R2-vSRX — Operational Mode, niezależna konsola**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration interfaces | display set
show configuration routing-options | display set
show configuration security | display set
show system users
show system commit
~~~

Sprawdź, czy porty ge nie są już w strefach typu trust/untrust, czy mają adresy i czy nie występuje fabryczny NAT. Jeśli tak, skoryguj **konkretny obiekt w kopii nowego labu**, nie wykonuj przypadkowego `load factory-default` na istniejącej instancji. Lokalny admin/root i SSH/console zostały omówione w rozdziale 4; to **warunek wcześniejszy**.

### 6.11.4. Etap A — konfigurowanie IPv4/IPv6 interface na R1

Zaczynamy od podłączenia R1 do sieci OFFICE i TRANSIT. W Junos IPv4 jest w `family inet`, IPv6 w `family inet6`, a adres wiążemy z **logical unit 0**.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], TYLKO osobna instancja LAB**

~~~junos
set system host-name R1-vSRX
set interfaces ge-0/0/1 description "CH06-OFFICE-TO-PC-A"
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/24
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:10::1/64
set interfaces ge-0/0/0 description "CH06-TRANSIT-TO-R2"
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/30
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::1/64
show | compare
commit check
~~~

**Dwie ważne rzeczy:** `commit check` potwierdza składnię i część ograniczeń systemu, **nie obecność peer**. Dodanie `2001:db8:12::1/64` nie gwarantuje, że R2 ma już IPv6 ani że ICMPv6 przejdzie policy.

### 6.11.5. Etap B — interfejsy R2

**Urządzenie: R2-vSRX — Configuration Mode [edit private], TYLKO osobna instancja LAB**

~~~junos
set system host-name R2-vSRX
set interfaces ge-0/0/0 description "CH06-TRANSIT-TO-R1"
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.2/30
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::2/64
set interfaces ge-0/0/1 description "CH06-ERP-TO-PC-B"
set interfaces ge-0/0/1 unit 0 family inet address 203.0.113.1/24
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:20::1/64
show | compare
commit check
~~~

Po commit i aktywacji interface spodziewamy się connected IPv4 i IPv6:
- R1: `192.0.2.0/24`, `198.51.100.0/30`, `2001:db8:10::/64`, `2001:db8:12::/64`;
- R2: `198.51.100.0/30`, `203.0.113.0/24`, `2001:db8:12::/64`, `2001:db8:20::/64`.

Pamiętaj o lokalnych host routes /32 i /128.

### 6.11.6. Etap C — statyczna droga R1 do ERP

**Urządzenie: R1-vSRX — Configuration Mode [edit private], statyczne trasy IPv4/IPv6**

~~~junos
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.2
set routing-options rib inet6.0 static route 2001:db8:20::/64 next-hop 2001:db8:12::2
show | compare
commit check
~~~

To dwie *niezależne* trasy, każda w innej rodzinie. **Nie dodawaj default na R1**, bo w podstawowym scenariuszu R1 zna dokładnie docelową sieć za R2. Ograniczamy złożoność do minimum.

### 6.11.7. Etap D — domyślna droga R2 do R1

**Urządzenie: R2-vSRX — Configuration Mode [edit private], IPv4 i IPv6 default**

~~~junos
set routing-options static route 0.0.0.0/0 next-hop 198.51.100.1
set routing-options rib inet6.0 static route ::/0 next-hop 2001:db8:12::1
show | compare
commit check
~~~

**Pytanie:** dlaczego R2 nie potrzebuje osobnego static `192.0.2.0/24`? Bo default przez R1 obejmuje tę sieć, jeżeli nie istnieje bardziej szczegółowa konkurencyjna route. W prawdziwym WAN możesz woleć jawny prefix i filtrowanie, ale na tym poziomie pokażemy mechanikę LPM na default.

### 6.11.8. Etap E — strefy i polityki SRX: nie pozwól, by firewall udawał problem routingu

Dwa vSRX to **firewalle z routingiem**, a nie neutralne routery IP. Bez security policies pakiety klientów mogą być odrzucane niezależnie od prawidłowego RIB/FIB.

Dla jasności konfigurujemy dwie strefy na każdym węźle. W odizolowanym, niedostępnym z zewnątrz labie zezwolimy na **ICMP i ICMPv6** między nimi, żeby testować routing w obie strony. Na produkcji musiałbyś ograniczyć adresy, usługi i właściwe role aplikacji.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], strefy oraz minimalne policy dla LAB ICMP**

~~~junos
set security zones security-zone OFFICE interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone TRANSIT interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD match source-address any
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD match destination-address any
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD match application [ junos-ping junos-icmp6-all ]
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD then permit
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV match source-address any
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV match destination-address any
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV match application [ junos-ping junos-icmp6-all ]
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV then permit
show | compare
commit check
~~~

**Urządzenie: R2-vSRX — Configuration Mode [edit private], strefy oraz polityki LAB**

~~~junos
set security zones security-zone TRANSIT interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone ERP interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD match source-address any
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD match destination-address any
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD match application [ junos-ping junos-icmp6-all ]
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD then permit
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV match source-address any
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV match destination-address any
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV match application [ junos-ping junos-icmp6-all ]
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV then permit
show | compare
commit check
~~~

**Zastrzeżenia przed uruchomieniem:** sprawdź w danym obrazie vSRX istnienie predefiniowanych aplikacji `junos-ping` i `junos-icmp6-all` oraz prawidłowe przypisanie portów do zones. Jeśli parser inaczej implementuje dany parametr, nie zastępuj go od razu `application any` na urządzeniu z siecią produkcyjną. Powyższa struktura jest **planowanym LAB baseline**, nie zweryfikowaną konfiguracją uruchomionej instancji.

### 6.11.9. Commit workflow — czytelny i bezpieczny

Na każdej z obu instancji wykonaj **`show | compare` i `commit check` przed warunkową aktywacją**. Nie zaczynaj automatycznej aktywacji, jeśli nie masz poprawnej konsoli i niezależnego kanału odzyskania.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], aktywacja R1 dopiero po preflight**

~~~junos
commit confirmed 5
~~~

**Urządzenie: R2-vSRX — Configuration Mode [edit private], aktywacja R2 dopiero po preflight**

~~~junos
commit confirmed 5
~~~

Po aktywacji wykonaj **nową sesję administracyjną**, read-only `show` i testy na interfejsach. **Nie wykonuj kolejnego `commit check` w czasie pending commit confirmed**, bo może on potwierdzić zmianę i wyłączyć timer. Jeśli service działa, potwierdź na każdym urządzeniu.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], świadome potwierdzenie po testach**

~~~junos
commit
~~~

**Urządzenie: R2-vSRX — Configuration Mode [edit private], świadome potwierdzenie po testach**

~~~junos
commit
~~~

**Jeśli test się nie powiedzie:** nie potwierdzaj zmian. Ustal przyczynę przez konsolę; pozwól na auto-rollback lub świadomie przywróć poprzednią konfigurację zgodnie z procedurą CH03. Przy tym scenariuszu nie wolno polegać wyłącznie na SSH przez port testowy, który może zostać wyłączony.

### 6.11.10. Etap F — konfiguracja hostów Linux

**Najpierw przeczytaj realną nazwę NIC**. Na lekkim Linuxie w EVE może być `eth0`, `ens3` itd. Wykonaj poniższe polecenia z niezależnej konsoli. Używamy zmiennej `IFACE`, żeby nie zakładać nazwy portu. Wartość dostosuj po odczycie.

**Urządzenie: PC-A-LINUX — Linux shell, read-only discovery**

~~~bash
ip -br link
ip -br address
ip -4 route
ip -6 route
~~~

**Urządzenie: PC-A-LINUX — Linux shell, tylko świeża odizolowana maszyna; po weryfikacji interfejsu**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 address add 192.0.2.10/24 dev "$IFACE"
sudo ip -6 address add 2001:db8:10::10/64 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
sudo ip -6 route replace default via 2001:db8:10::1 dev "$IFACE"
~~~

**Urządzenie: PC-B-LINUX — Linux shell, read-only discovery**

~~~bash
ip -br link
ip -br address
ip -4 route
ip -6 route
~~~

**Urządzenie: PC-B-LINUX — Linux shell, tylko świeża odizolowana maszyna; po weryfikacji interfejsu**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 address add 203.0.113.10/24 dev "$IFACE"
sudo ip -6 address add 2001:db8:20::10/64 dev "$IFACE"
sudo ip -4 route replace default via 203.0.113.1 dev "$IFACE"
sudo ip -6 route replace default via 2001:db8:20::1 dev "$IFACE"
~~~

**Ważny detal Linux:** te polecenia mają charakter **tymczasowy do reboot** w zależności od dystrybucji. Nie zapisujemy ich jako permanentnego network config dla produkcyjnego hosta. Nie używaj `ip addr flush` bez potrzeby — mógłbyś usunąć istniejącą adresację management VM.

### 6.11.11. Etap G — pierwsze pomiary: zanim popsujemy routing, musimy znać stan dobry

Najpierw testujemy bezpośrednią komunikację PC-A ↔ R1 oraz PC-B ↔ R2.

**Urządzenie: PC-A-LINUX — Linux shell, IPv4/IPv6 gateway**

~~~bash
ping -c 5 192.0.2.1
ping -6 -c 5 2001:db8:10::1
ip -4 neigh show
ip -6 neigh show
~~~

**Urządzenie: PC-B-LINUX — Linux shell, IPv4/IPv6 gateway**

~~~bash
ping -c 5 203.0.113.1
ping -6 -c 5 2001:db8:20::1
ip -4 neigh show
ip -6 neigh show
~~~

Dopiero potem test tranzytu między R1 a R2.

**Urządzenie: R1-vSRX — Operational Mode, po baseline commit**

~~~junos
show interfaces terse
show route 198.51.100.2
show route 2001:db8:12::2
ping 198.51.100.2 source 198.51.100.1 count 5
ping 2001:db8:12::2 source 2001:db8:12::1 count 5
show arp
show ipv6 neighbors
~~~

**Urządzenie: R2-vSRX — Operational Mode, po baseline commit**

~~~junos
show route 198.51.100.1
show route 2001:db8:12::1
ping 198.51.100.1 source 198.51.100.2 count 5
ping 2001:db8:12::1 source 2001:db8:12::2 count 5
~~~

Na końcu pełny test end-to-end.

**Urządzenie: PC-A-LINUX — Linux shell, end-to-end dual stack**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
ip -4 route get 203.0.113.10
ip -6 route get 2001:db8:20::10
~~~

**Urządzenie: PC-B-LINUX — Linux shell, kontrola drogi powrotnej oraz nowego flow w odwrotną stronę**

~~~bash
ping -c 5 192.0.2.10
ping -6 -c 5 2001:db8:10::10
ip -4 route get 192.0.2.10
ip -6 route get 2001:db8:10::10
~~~

**Oczekiwany efekt:** IPv4 i IPv6 działają w obie strony. **To prognoza, nie wynik zmierzony w EVE-NG**. Jeżeli którykolwiek ping nie działa, **nie przechodź do fault injection**. Najpierw sprawdź IPv6 ND, host firewall, policies, brakujące adresy/default, port mapping i istniejący fabryczny NAT.

### 6.11.12. Etap H — pokaż rzeczywiste różnice RIB/FIB/ARP/ND

**Urządzenie: R1-vSRX — Operational Mode, snapshot routingu**

~~~junos
show route summary
show route protocol direct
show route protocol local
show route protocol static
show route 203.0.113.0/24 extensive
show route table inet6.0
show route forwarding-table destination 203.0.113.10
show route forwarding-table family inet6 destination 2001:db8:20::10
show arp
show ipv6 neighbors
show security flow session
~~~

**Pytanie do samodzielnego przewidzenia:**
- Czy do ERP IPv4 wygrywa R1 connected czy static? **Static**, bo ERP nie jest bezpośrednio w lokalnej sieci R1.
- Czy na R2 do ERP wygrywa default czy connected? **Connected**, bo /24 jest dłuższy niż /0.
- Czy na R2 do PC-A wygrywa connected czy default? **Default**, bo R2 nie ma bezpośredniego `192.0.2.0/24`.
- Czy R1 zna IPv6 `2001:db8:20::/64`? **Tak**, z `inet6.0` static po commit.
- Czy `show arp` dowodzi stanu IPv6 sąsiadów? **Nie**, do tego jest `show ipv6 neighbors`.

### 6.11.13. LAB-FAULT A — usuwamy trasę R1 do ERP, ale nie uszkadzamy portu

**Cel:** wykazać różnicę między connected interface up a brakującą statyczną RIB route. Przed awarią upewnij się, że **R1 nie ma alternatywnego default** prowadzącego do ERP.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], wyłącznie izolowany LAB, tylko IPv4 static**

~~~junos
delete routing-options static route 203.0.113.0/24
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R1-vSRX — Operational Mode przez niezależną konsolę, po fault, bez kolejnego commit/check**

~~~junos
show interfaces terse
show route 203.0.113.0/24
show route forwarding-table destination 203.0.113.10
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell, podczas fault**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Przewidywany wynik:** IPv4 przestanie działać, ponieważ usunięto route, ale **IPv6 może nadal działać**, bo nie zmieniliśmy `inet6.0`. Właśnie pokazaliśmy, że IPv4/IPv6 to dwa niezależne RIB/FIB.

Po upływie timera bez potwierdzenia wróci poprzednia konfiguracja. **Nie wykonuj `commit check` w trakcie oczekiwania.** Sprawdź w historii commit i powtórz obydwa testy.

### 6.11.14. LAB-FAULT B — poprawna składnia, zły next-hop

**Cel:** zobaczyć różnicę między składnią konfiguracji a faktyczną osiągalnością next-hopa. Najpierw poczekaj, aż LAB-FAULT A zakończy auto-rollback.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], tylko LAB**

~~~junos
delete routing-options static route 203.0.113.0/24 next-hop 198.51.100.2
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.200
show | compare
commit check
commit confirmed 2
~~~

Adres `198.51.100.200` nie jest connected do R1 w naszym labie. Junos może nie zakwalifikować tej static route do active i wskazać **unresolved/hidden** w odpowiednim widoku. Przewidywane zachowanie trzeba potwierdzić **na konkretnym release**, nie zgadywać z pamięci.

**Urządzenie: R1-vSRX — Operational Mode przez konsolę, analiza route reason**

~~~junos
show route 203.0.113.0/24 extensive
show route hidden
show route 198.51.100.200
show interfaces ge-0/0/0 terse
show system commit
~~~

**Pytanie:** dlaczego `commit check` przechodzi, skoro ping nie działa? Bo sprawdza **konfigurację**, nie dowodzi reachability do każdego sąsiada.

### 6.11.15. LAB-FAULT C — zniknęła droga powrotna, więc w jedną stronę pakiet może dojść

**Cel:** nauczyć się nie zapominać o return path. Najpierw upewnij się, że R1 i R2 powróciły do baseline. Z R2 usuń IPv4 default.

**Urządzenie: R2-vSRX — Configuration Mode [edit private], tylko izolowany LAB**

~~~junos
delete routing-options static route 0.0.0.0/0
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R2-vSRX — Operational Mode przez konsolę, odczyt braku return path**

~~~junos
show route 192.0.2.10
show route 0.0.0.0/0
show route 203.0.113.10
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Przewidywany wynik:** IPv4 z PC-A do PC-B przestaje działać, choć R1 może nadal wysłać Echo Request do PC-B. IPv6 może nadal działać, bo `::/0` na R2 nie został usunięty. Nie twierdź, że pakiety IPv4 „na pewno doszły do serwera”, jeśli nie zbierasz dowodu na PC-B, np. krótkim `tcpdump` w izolowanym labie.

**Urządzenie: PC-B-LINUX — Linux shell, opcjonalna obserwacja podczas fault, z odpowiednią autoryzacją**

~~~bash
sudo tcpdump -ni eth0 icmp -c 20
~~~

Zatrzymaj po uzyskaniu dowodu. Sprawdź auto-rollback R2 i ponowny ping.

### 6.11.16. LAB-FAULT D — krótsza trasa default nie powinna wygrać z dłuższą /24

**Cel:** pokazać LPM na R2 w praktyce. R2 ma default do R1, a jednak dla **własnej** sieci ERP `203.0.113.0/24` używa connected. Trasa o preference 5 nie „wygrywa” z connected /24 tylko dlatego, że jest static default.

**Urządzenie: R2-vSRX — Operational Mode, bez wykonywania żadnej zmiany**

~~~junos
show route 0.0.0.0/0
show route 203.0.113.0/24
show route forwarding-table destination 203.0.113.10
show route forwarding-table destination 192.0.2.10
~~~

**Interpretacja:** w R2 `203.0.113.10` pasuje do connected /24, a `192.0.2.10` do default /0. To jest LPM, bez konieczności zmieniania żadnej wartości preference.

### 6.11.17. LAB-FAULT E — IPv6 independently: pozorna poprawność IPv4 nie wystarcza

**Cel:** usuń tylko IPv6 static route R1 do ERP i porównaj wyniki dual-stack. Upewnij się, że wszystkie wcześniejsze confirmed timers są zakończone.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], tylko LAB IPv6**

~~~junos
delete routing-options rib inet6.0 static route 2001:db8:20::/64
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R1-vSRX — Operational Mode z konsoli, podczas fault**

~~~junos
show route table inet6.0
show route 2001:db8:20::10
show ipv6 neighbors
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell, porównaj rodziny**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

Jeżeli IPv4 działa, a IPv6 nie, diagnozuj **`inet6.0`, ND i ICMPv6 policy** zamiast podmieniać kabel. To jest bardzo dobry przykład różnicy między dual-stack działającym na jednym fizycznym porcie a **dwoma niezależnymi control-plane routing contexts**.

### 6.11.18. LAB-FAULT F — `discard` bardziej szczegółowy niż normalna static route

**Cel:** przećwicz LPM na statycznej trasie /32. Mamy normalną R1 static `203.0.113.0/24` przez R2. Dodajemy **host-specific discard** dla jednego IP PC-B.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], tylko izolowany LAB**

~~~junos
set routing-options static route 203.0.113.10/32 discard
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R1-vSRX — Operational Mode przez konsolę, podczas fault**

~~~junos
show route 203.0.113.10/32
show route 203.0.113.0/24
show route forwarding-table destination 203.0.113.10
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell, podczas fault**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Gotcha:** `discard /32` ma wyższy poziom szczegółowości niż normalna `static /24`. IP `203.0.113.10` zostanie blackholowany, mimo że /24 pozostaje aktywna i sprawna dla innych hostów. Preference **nie służy do porównywania /32 z /24**.

Po upływie timera nie wykonuj żadnego commit. Sprawdź, czy /32 zniknęła i IPv4 ping wrócił.

### 6.11.19. LAB-FAULT G — L2/ND nie jest tym samym co RIB

Chcemy pokazać przypadek, w którym connected i static są prawidłowe, ale **adres MAC sąsiada nie jest dostępny**. Najprostszy bezpieczny sposób na izolowanym labie to administracyjnie zatrzymać kartę testową PC-B-LINUX z **jego lokalnej konsoli**, nie z SSH.

**Urządzenie: PC-B-LINUX — Linux shell / konsola, tylko w izolowanym labie, wyłączenie portu danych**

~~~bash
sudo ip link set eth0 down
~~~

**Urządzenie: R2-vSRX — Operational Mode, obserwacja po wyłączeniu hosta**

~~~junos
show route 203.0.113.0/24
show arp
show ipv6 neighbors
show interfaces ge-0/0/1 terse
~~~

Możesz zobaczyć nadal **connected /24 i /64 na R2**, bo routerowy port jest nadal up, ale host `203.0.113.10` nie odpowiada na ARP, a `2001:db8:20::10` nie odpowiada na ND. Sąsiad mógł pozostać chwilowo w cache, więc obserwuj stan i timeout zamiast uznawać pojedynczy odczyt za ostateczny dowód.

**Urządzenie: PC-B-LINUX — Linux shell / konsola, przywrócenie portu po teście**

~~~bash
sudo ip link set eth0 up
~~~

Następnie ponownie przetestuj ARP/ND i ping. Jeśli nazwa NIC jest inna, użyj wcześniej ustalonej prawidłowej nazwy. Nie wykonuj tego polecenia na maszynie, której **jedyny dostęp administracyjny** zależy od wyłączanego interfejsu.

### 6.11.20. Oczekiwane wyniki — nie pomyl tabeli „predykcje” z realnym raportem

| Test | Hipoteza | Oczekiwane zachowanie |
|---|---|---|
| Baseline IPv4/IPv6 | Obie strony znają drogę i return path | Ping z PC-A do PC-B w obu rodzinach PASS |
| A: brak IPv4 route na R1 | Nie ma odpowiedniego LPM do ERP | IPv4 FAIL, IPv6 może działać |
| B: niedostępny next-hop | Static niekwalifikowana lub niewykonalna | Brak active route / route reason; fizyczny port nadal up |
| C: brak default IPv4 na R2 | Odpowiedź nie wraca do OFFICE | IPv4 FAIL, IPv6 może działać |
| D: connected vs default | LPM preferuje /24 nad /0 | Do własnego ERP connected, do OFFICE default |
| E: brak IPv6 static na R1 | Oddzielna rodzina routingowa | IPv6 FAIL, IPv4 PASS |
| F: bardziej szczegółowy discard | /32 wygrywa z /24 | IPv4 PC-B jest blackholed |
| G: PC-B NIC down | RIB nadal może mieć connected | ARP/ND failed i host unreachable |

Wszystkie powyższe to **przewidywane wyniki**. Nie stawiaj przy nich statusu PASS, dopóki nie zobaczysz prawdziwego outputu w EVE-NG.

### 6.11.21. Cleanup i finalny dowód zakończenia

Po każdej próbie:
- sprawdź `show system commit`; nie pozostaw żadnego pending `commit confirmed`;
- potwierdź, że route z baseline jest w właściwej tablicy;
- sprawdź, czy `203.0.113.10/32 discard` już nie istnieje;
- sprawdź status portów i sąsiadów;
- zrób test klienta w obu rodzinach;
- udokumentuj faktyczną godzinę, wersję Junos i obserwację.

**Urządzenie: R1-vSRX — Operational Mode, finalny odczyt**

~~~junos
show system commit
show interfaces terse
show route 203.0.113.0/24
show route 203.0.113.10/32
show route table inet6.0
show route forwarding-table destination 203.0.113.10
show arp
show ipv6 neighbors
show security flow session summary
~~~

**Urządzenie: R2-vSRX — Operational Mode, finalny odczyt**

~~~junos
show system commit
show route 0.0.0.0/0
show route table inet6.0
show interfaces terse
show security flow session summary
~~~

**Urządzenie: PC-A-LINUX — Linux shell, finalny test**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Nie wykonuj `rollback 0` w shared candidate zawierającym potencjalne cudze zmiany.** W izolowanym labie możesz przywrócić baseline do właściwej private candidate po potwierdzeniu, że pending timer zakończył się zgodnie z planem.

### 6.11.22. Co wpisujesz do `VERIFY.md` zamiast „it works”?

Dla każdego eksperymentu zapisz:
- dokładny obraz/vSRX release i ewentualny rodzaj host Linux;
- rzeczywiste mapowanie QEMU NIC → `fxp0/ge`;
- timestamp przed i po;
- baseline `show interfaces`, `show route`, FIB i ARP/ND;
- dokładne `show | compare` i sposób zatwierdzenia zmiany;
- odczyt podczas fault oraz jego interpretację;
- wynik PC-A → PC-B i test zwrotny;
- czy `commit confirmed` automatycznie przywrócił konfigurację;
- co nie zadziałało i jaki był root cause;
- końcowy rezultat PASS / FAIL / NOT TESTED.

**Ważna zasada:** nie publikuj w publicznym repozytorium danych rzeczywistych klientów, kont, haseł, kluczy, tokenów ani całej konfiguracji urządzenia z sekretami.

**Status sekcji 6.11:** **PLANNED / NOT RUNTIME VALIDATED** — czytelnik ma pełny instruktaż, ale potwierdzenie wykonania na konkretnym obrazie jest osobnym zadaniem.

