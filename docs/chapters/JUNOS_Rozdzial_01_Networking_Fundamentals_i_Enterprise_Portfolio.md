# Rozdział 1. Networking Fundamentals i portfolio Juniper w sieci Enterprise

**Podręcznik:** JUNOS Enterprise Routing & Switching — JNCIA-Junos (Associate) → JNCIS-ENT (Specialist) → JNCIP-ENT (Professional)  
**Wersja:** pierwsza pełna redakcja szkoleniowa, 16 listopada 2026  
**Zakres blueprintu:** 1.1–1.11 [A] i **1.12 [EXT] — praktyczny przewodnik po portfolio Campus LAN/Branch/WLAN i przykładowym greenfield/BOM**.  
**Odbiorca:** inżynier z doświadczeniem Cisco Enterprise (Catalyst, ISR/ASR, WLC/9800), który uczy się Junos i potrzebuje umieć także rozmawiać z architektem, handlowcem oraz klientem o doborze sprzętu.  
**Źródła:** oficjalne HPE Juniper Networking / Juniper (datasheets, validated designs, Hardware Explorer, JTAC KB21476).  
**Uwaga:** części dotyczące produktów/wersji to **zdjęcie stanu na 16.11.2026**, wymagające weryfikacji przed ofertowaniem. Wersje w KB to **sugerowane gałęzie do przetestowania, nie polecenie automatycznego upgrade**. W szczególności końcówka „-Sx” nie oznacza jednego konkretnego pliku do wdrożenia.

> **Zanim pojawią się modele:** zobaczmy prawdziwą sytuację. Klient przychodzi i mówi: „Potrzebujemy sieci w nowym biurze na trzy piętra: 320 stanowisk, Wi-Fi 7, 35 telefonów IP, 28 kamer, WAN redundantny i oddzielny guest Internet. Wyślij mi BOM”. W Cisco odruchowo rozważasz Catalyst 9200/9300 na access, Catalyst 9500 na core, Firewall i 9800 WLC. **Co w takim projekcie wybierzesz z Junipera?** Gdzie kończy się EX, zaczyna QFX, po co SRX, czym różni się Mist od Junos i czy do Wi-Fi musisz postawić tradycyjny WLC? To właśnie praktyczny cel rozszerzenia 1.12.

## 1.0. Mapa rozdziału: na pamięć, do zrozumienia, do użycia przy projekcie

**Na pamięć:** przełącznik L2 vs L3, domena broadcastowa, ARP/ND, longest-prefix match, TCP vs UDP, CoS; rozpoznawanie rodzin **EX / QFX / SRX / MX / SSR / Mist AP** oraz tego, które urządzenia uruchamiają Junos.

**Do zrozumienia:** hierarchia access–distribution–core, kiedy tworzy się collapsed core, dlaczego wielogigabitowe AP wymusza inne porty i PoE, a EVPN-VXLAN jest alternatywą architektoniczną, a nie obowiązkowym „nowszym VLAN-em”.

**Do przećwiczenia:** rozpoznawanie modeli i systemu przez CLI, przygotowanie HLD/LLD, port count, PoE budget, uplink bandwidth, redundancja, optyka, subskrypcje, dobór release i lista warunków do potwierdzenia u producenta/dystrybutora.

Nie traktuj tabel poniżej jak cennika. **Żaden opis rodziny nie zastępuje sprawdzenia konkretnego SKU, karty datasheet, licencji, Hardware Compatibility Tool, Release Notes, statusu lifecycle i portów fizycznych.**

---

## 1.1. Router czy switch? Zacznij od roli, a nie od obudowy [A]

Na rysunku małego biura masz 48 laptopów, drukarki, AP i telefony. Chcesz, żeby wszystkie urządzenia w jednym VLAN-ie mogły przekazywać ramki bez pytania o trasę IP przy każdym hopie — to przełączanie **Layer 2** na podstawie MAC. Gdy laptop w VLAN 20 chce połączyć się z serwerem w VLAN 30, potrzebujesz decyzji **Layer 3**: bramy, trasy, a potem odpowiedniego local next-hop.

**Most z Cisco:** Catalyst 9300 jest fizycznie switchem, ale robi L3 routing przez SVI. Juniper EX4400 również może pełnić funkcje L3. Sama nazwa „switch” nie oznacza „nie potrafi routować”. Z drugiej strony SRX może mieć porty switchingowe, a jego główną rolą jest jednak **stateful security/WAN edge**, nie szafa z 48 portami użytkowników.

Dla praktyka to pytanie powinno brzmieć: „**Gdzie będzie gateway VLAN, kto trzyma policy i którędy wychodzimy z budynku?**”. Dopiero później dobierasz model.

**Urządzenie: EX-ACCESS-01 — Operational mode (przykład Junos, sprawdź model/release)**

~~~junos
show interfaces terse
show vlans
show ethernet-switching table
show route table inet.0
~~~

Pierwsze trzy polecenia pomagają poznać warstwę L2, czwarte L3. Nie wymagaj, by każdy EX miał identyczny zakres przełączania i routingu — funkcje zależą od konkretnej rodziny, wydania i licencji.

## 1.2. Ethernet — pakiet przechodzi przez więcej niż jeden świat [A]

Komputer wysyła pakiet do serwera w innym VLAN-ie. Najpierw buduje ramkę do MAC bramy, a nie do MAC odległego serwera. Switch access uczy się adresów MAC i przekazuje ramki zgodnie ze swoim VLAN-em. Urządzenie L3 wykonuje lookup IP i wysyła pakiet dalej z nowym nagłówkiem Ethernet na kolejnym segmencie. **To dlatego diagnoza „ping bramy działa, serwer nie” nie kończy się na interfejsie access switcha.**

Na Catalyst zaglądasz w \`show mac address-table\`; na Junos w \`show ethernet-switching table\`. Obie komendy odpowiadają na pytanie „gdzie przełącznik nauczył się danego MAC?”, ale struktura outputu i kontekst VLAN/IRB są inne.

**Urządzenie: Cisco-C9300 — privileged EXEC**

~~~cisco
show mac address-table dynamic
show interfaces trunk
~~~

**Urządzenie: EX-ACCESS-01 — Operational mode**

~~~junos
show ethernet-switching table
show interfaces terse
show vlans
~~~

## 1.3. Broadcast domain vs collision domain [A]

Jeden VLAN zwykle tworzy jedną domenę rozgłoszeniową L2. Przy dużym campuse nie projektujesz „VLAN 10 dla wszystkich pięter, bo tak jest wygodnie”: to utrudnia izolację, awarie, analizę broadcastów i zmianę architektury. Każdy full-duplex switch port jest odrębnym linkiem; dawne collision domains pamiętamy głównie po to, żeby rozumieć historię hubów i Ethernet.

**Gotcha:** Virtual Chassis nie jest jedną olbrzymią domeną broadcastową samą z siebie. To sposób połączenia kilku przełączników w **jedno logiczne urządzenie**. Domeny VLAN nadal zależą od konfiguracji.

## 1.4. ARP i IPv6 ND — jak IP znajdzie kolejny MAC? [A]

Wyobraź sobie adres pocztowy budynku i numer rejestracyjny samochodu kurierskiego. IP to adres celu, MAC to lokalny sposób dostarczenia ramki. Dla zdalnego IP host rozwiązuje **MAC lokalnej bramy**, nie MAC serwera w innym mieście.

W IPv4 dzieje się to przez ARP; w IPv6 przy pomocy Neighbor Discovery (ND). Kiedy w rozdziale 2 uczymy się o RIB/FIB/PFE, to właśnie brak poprawnego ARP/ND może sprawić, że trasa w tabeli wygląda dobrze, lecz pakiet nadal nie rusza.

**Urządzenie: EX-DIST-01 — Operational mode**

~~~junos
show arp
show ipv6 neighbors
show interfaces terse
~~~

Nie zakładaj, że składnia i zakres widoku są identyczne na SRX, EX, MX i różnych wersjach Junos.

## 1.5. IP i maski — ile sieci tak naprawdę potrzebuje nowy biurowiec? [A]

Nie musisz uczyć się od początku /24 ani /26. Zadaj sobie jednak pytanie projektowe: czy 320 użytkowników, 35 telefonów i 28 kamer powinno trafić do jednej sieci, bo „wszyscy są na tym samym piętrze”? Najczęściej to zła przesłanka. Oddzielasz **funkcję i politykę dostępu**: corporate, voice, cameras, IoT, guest, infrastruktura i OOB management. Następnie dopiero nadajesz odpowiedni prefiks. Nie kopiuj adresacji z poprzedniego site bez planu agregacji i tras na WAN.

**Most z Cisco:** planowanie SVI/VLAN i VRF jest podobne, ale na Junos bramy L3 w switching często reprezentuje **IRB** (Integrated Routing and Bridging), a ograniczanie routingu instancjami odbywa się z użyciem \`routing-instances\`. Szczegóły IRB i konfiguracji poznasz w rozdziale 14.

## 1.6. IPv4 kontra IPv6 — nie wystarczy „dołożyć AAAA w DNS” [A]

W IPv4 potrzebujesz ARP do rozwiązania lokalnego MAC. W IPv6 do tego służy Neighbor Discovery, a do poprawnej pracy hostów często także Router Advertisements oraz ICMPv6. **Nie blokuj bezmyślnie całego ICMPv6**, bo możesz uszkodzić wykrywanie sąsiadów, Path MTU Discovery i inne fundamentalne funkcje. To praktyczny problem migracyjny również na Cisco.

**Urządzenie: EX-DIST-01 — operational mode (jeśli obsługiwane przez model/release)**

~~~junos
show route table inet6.0
show ipv6 neighbors
~~~

W Greenfield HLD zapisz jasno, które segmenty mają dual-stack, gdzie działają RA/SLAAC/DHCPv6 i jak NAC/firewall traktują IPv6. **IPv6 wymaga osobnego planu bezpieczeństwa**, nawet jeśli aplikacje używają jeszcze tylko IPv4.

## 1.7. Subnetting i supernetting — oszczędność IP to nie jedyny cel [A]

Wiesz, że /30 daje klasycznie dwa użyteczne adresy w L3 point-to-point, a /31 pozwala oszczędzić przestrzeń na wspieranych platformach. W nowym campuse projektuj jednak adresację tak, by dało się **agregować prefiksy per budynek/site**, ograniczać ilość tras w WAN i stosować czytelne ACL/policies. Dwieście rozsianych po całej korporacji /24 utrudnia późniejszą agregację.

**Pytanie projektowe:** czy wszystkie lokalne VLAN-y biurowca mieszczą się w zarezerwowanym bloku adresacji dla danego site? Jeśli nie, masz problem projektowy jeszcze przed wprowadzeniem pierwszej komendy \`set interfaces irb\`.

## 1.8. Binary, decimal i wildcard — wiedza Cisco, którą można zastosować bez powtórnego wkuwania [A]

Znajomość bitów przydaje się, gdy liczysz wielkość podsieci, adresy hostów, rozbicie prefiksów, adresy multicast i selekcję tras. Z Cisco znasz **wildcard masks** w ACL oraz maski podsieci; nie przepisuj ich mechanicznie do Junos jako identycznej składni. W Junos używa się m.in. prefiksów w formacie CIDR oraz różnych mechanizmów dopasowania w **policy-statement**, **firewall filter** i **route-filter**, które mają własną składnię.

**Do przećwiczenia:** weź \`10.80.0.0/21\`, rozbij na /24 dla VLAN-ów, następnie sprawdź, czy da się utrzymać sensowną agregację do core/WAN. To bardziej przydatne przy greenfield niż kolejne dziesięć sztucznych zadań „10010011 na dziesiętny”.

**Przykład greenfield:** access na piętrze 2 ma VLAN użytkowników, voice i AP management. Czy jedno /24 „na cały site” jest wystarczające? Problem nie jest tylko w liczbie hostów. Jest w planie segmentacji, DHCP, bramach, NAC, routing instances i możliwości rozbudowy.

## 1.9. Longest Prefix Match — nie mylić z preferencją protokołu [A]

Załóżmy, że do 10.50.40.11 pasuje 0/0, 10.0.0.0/8 oraz 10.50.40.0/24. Pakiet użyje **/24**, nawet jeśli default route jest „bardziej preferowana” jako trasa we własnej klasie. To dwie różne decyzje: wybór aktywnego kandydata **dla jednego prefiksu** i późniejsze znalezienie **najdłuższego pasującego prefiksu** dla pakietu.

**Most z Cisco:** Cisco \`show ip route\` vs \`show ip cef\`; Junos \`show route\` vs \`show route forwarding-table\`. W rozdziale 2 omawiamy subtelny haczyk, że dla **tego samego prefiksu** Cisco i Junos mogą wybrać inny protokół ze względu na domyślne AD/preference.

## 1.10. TCP vs UDP — ważniejsze dla architekta niż same numery portów [A]

Nie projektuj „sieci LAN do Teams” na podstawie samego słowa UDP. Zrozum, że aplikacje czasu rzeczywistego są wrażliwe na jitter i straty, a usługi typu BGP/SSH zależą od zestawiania i utrzymania sesji TCP. W greenfield potrzebujesz klasyfikacji ruchu, odpowiedniej ścieżki WAN, polityk security i obserwowalności, a nie tylko listy portów firewall.

## 1.11. Class of Service — dlaczego biuro z wideokonferencjami nie może opierać się wyłącznie na „szybkim uplinku”? [A]

Wyobraź sobie poniedziałek 9:00. Trwa wideokonferencja zarządu, backup przenosi setki gigabajtów, a na tym samym łączu wysyłają się nagrania z kamer. Bez kontroli kolejek nawet port 10G może w chwilach obciążenia dawać straty/jitter dla voice.

W Cisco myślisz o DSCP, queueing i policy maps. W Junos o **CoS**, forwarding classes, schedulers, rewrite rules i kolejkach właściwych platformie. Zasady będziemy rozwijać w rozdziale 20. Teraz zapamiętaj regułę projektową: **na access rozpoznaj i oznacz ruch, na uplinkach zachowaj zgodną politykę, a na przeciążonych punktach pilnuj kolejek**.

---

## 1.12. [EXT] Portfolio Juniper/HPE — od szafy IDF do redundantnego kampusu

### 1.12.1. Rozmowa z klientem: „dobierz mi modele” to dopiero finał, nie początek

W dobrym projekcie nie zaczynasz od „EX4100 czy EX4400?”. Zaczynasz od **warunków fizycznych i funkcjonalnych**, ponieważ jeden zły wybór portu albo zasilacza może unieważnić całe zamówienie.

Wyobraź sobie, że klient chce „48-portowy PoE switch do AP Wi-Fi 7”. Dobierasz 48-portowy switch z PoE+ i 1GbE. Karta wygląda dobrze, marka się zgadza, ale AP potrzebuje **mGig** i odpowiedniej mocy PoE, by pracować z zamierzonym zestawem radiowym i funkcji. Uruchomi się w trybie ograniczonym albo nie spełni założeń wydajnościowych. Na papierze BOM się zgadza, w sali konferencyjnej ludzie narzekają.

**To największa lekcja:** dobór SKU jest wynikiem wymagań, nie ich zamiennikiem.

Dla nowego biura ustal przed BOM:
1. Liczba stanowisk przewodowych, VoIP, kamer, AP, IoT — aktualna i trzyletni wzrost.
2. Rozkład na piętra i szafy IDF, ilość kabli, kategorie okablowania i trasy światłowodowe.
3. Port speed (1G/2.5G/5G/10G), budżet PoE na port i **łącznie** dla chassis, opcje PSU.
4. Liczba i prędkość uplinków, redundancja fizyczna, odmienne trasy światłowodowe.
5. L2/L3 boundary: gdzie jest IRB/default gateway, które VLAN-y potrzebują routingu, jakie VRF i mikrosegmentacja.
6. SLA operacyjne: awaria jednego urządzenia, upgrade, logi/telemetria, integracja NAC, zarządzanie.
7. WLAN: RF survey, rodzaj AP, 6 GHz, roaming, lokalny vs tunelowany datapath, uplinki mGig i PoE.
8. Internet/WAN: operatorzy, przepustowość **z włączonym IPS/UTM**, IPsec, failover, cloud/SD-WAN.
9. Preferencja zarządzania: Junos CLI/automatyzacja czy Mist/Wired Assurance, wymagania subskrypcji i operacyjne.
10. Lifecycle, wersje Junos sugerowane przez JTAC, transceivery, wsparcie, dostawy i kompatybilność.

Juniper w [oficjalnym przewodniku greenfield wired/wireless](https://www.juniper.net/documentation/us/en/software/nce/juniper-ai-driven-wired-wireless-network-deployment-guide/juniper-ai-driven-wired-wireless-network-deployment-guide.pdf) rozpoczyna HLD od network assessment, portów, PoE, WLAN survey, WAN i wymagań security — nie od modelu na pierwszej stronie.

### 1.12.2. Najpierw poznaj rodziny, zanim zapamiętasz modele

**EX** to podstawowa rodzina switchy do biura/campus. Może pełnić rolę access, a wybrane modele — distribution lub core. Myśl o niej jak o rodzinie Catalyst 9200/9300/9500, ale **nie** jako dokładnej translacji SKU.

**QFX** to rodzina wysokowydajnych przełączników znanych również z data center. W campusie wybrane QFX są dobrym wyborem na aggregation/core/EVPN fabric, szczególnie gdy potrzebujesz większej liczby portów 25/100/400G. Nie każde QFX nadaje się do szafy access z telefonami IP — w wielu wariantach nie masz portów miedzianych PoE.

**SRX** to rodzina firewalli/security gateways z Junos. W biurze zwykle pełni funkcję **WAN edge, Internet security, VPN, segmentation** albo połączenia do zewnętrznych usług. Porównuj ją do ASA/FTD lub Cisco Secure Firewall, a nie automatycznie do routera ISR bez security.

**MX** to routing edge/aggregation o wyższym poziomie skalowania — historycznie szeroko stosowany w operatorach i dużych enterprise. W zwykłym trzydziestoosobowym biurze to często nadmiar. W większym kampusie lub interconnect/WAN może być uzasadniony.

**SSR / Session Smart Router** to odrębna linia SD-WAN, z **Session Smart Routing software**, a nie Junos OS. To ważne, gdy handlowiec mówi „Juniper router” — nie możesz zakładać, że ma ten sam \`configure\`/Junos CLI co SRX.

**Mist AP** to punkty dostępowe Wi-Fi, zarządzane przez Juniper Mist cloud i własne AP firmware. **Nie działają pod Junos**. Mist Cloud to **platforma zarządzania/assurance**, nie zamiennik fizycznego EX albo tradycyjny kontroler danych pracujący w każdym biurze.

~~~mermaid
flowchart TB
   ISP["WAN / Internet / MPLS / SD-WAN"] --> FW["WAN Edge / Security\nSRX lub SSR (inna rodzina OS)"]
   FW --> CORE["Core / Distribution\nEX4400-24X, EX4650, EX9200, wybrane QFX"]
   CORE --> ACC1["IDF / Access\nEX4000 / EX4100 / EX4400"]
   CORE --> ACC2["IDF / Access\nEX4000 / EX4100 / EX4400"]
   ACC1 --> USERS["PC · Voice · Kamera · IoT"]
   ACC2 --> APS["Mist AP34 / AP36 / AP37 / AP47"]
   CLOUD["Mist Cloud\nWired / Wi-Fi / WAN Assurance"] -. "management/telemetry" .-> FW
   CLOUD -. "management/telemetry" .-> CORE
   CLOUD -. "management/telemetry" .-> ACC1
   CLOUD -. "management/telemetry" .-> APS
~~~

**Nie pomyl linii przerywanych:** Mist Cloud to control/management i telemetry. Typowy ruch klientów Wi-Fi może być **bridgowany lokalnie**, a nie wysyłany do chmury. W określonych architekturach możesz dodać **Mist Edge** do tunelowania WLAN. To rozróżnienie jest podobne do różnicy Cisco AP local switching/FlexConnect vs central switching, ale nie jest identyczne co do implementacji.

### 1.12.3. Access layer — EX4000, EX4100-F, EX4100, EX4400

Najpierw spójrz na trzy typy szaf.

**Mały oddział / płytka szafa / brak HA sprzętowego.** Tu ważne są kompaktowość, cena platformy, chłodzenie, 10G uplink i PoE. **EX4000** ma wersje 8/12/24/48-portowe, w tym małe fanless i warianty 24/48 portów. Przykładowo **EX4000-12T** jest fanless i nie oferuje PoE, a **EX4000-24P** to 24-portowy wariant PoE. Nie zakładaj, że wszystkie porty oznaczone jako „MP” są 10G — w tej rodzinie mGig obejmuje konkretne porty i szybkości opisane w datasheet.

**Standardowy access w przedsiębiorstwie.** **EX4100-24P/48P** oferuje porty 1G PoE, uplinki 1/10G i dodatkowe 10/25G do uplink/Virtual Chassis. **EX4100-F** to wariant z bardziej uproszczoną budową PSU/fan i konfiguracjami 12/24/48 portów. **EX4100-H** to inny przypadek: ruggedized/harsh environment, nie ta sama obudowa co zwykły EX4100. Jeżeli masz przemysłowe, niewygodne miejsce montażu, dobrze zweryfikuj zakres temperatur i wymogi zasilania zamiast zamawiać „cokolwiek EX4100”.

**Biuro wymagające większej niezawodności i Wi-Fi 6E/7.** **EX4400-24P/48P** oferuje tradycyjne porty 1G PoE, ale gdy AP lub stacja potrzebuje >1G, patrz na **EX4400-24MP / EX4400-48MP** lub odpowiednie nowe warianty XP/MXP. **EX4400-24MP** ma 24 porty mGig do 10GbE. **EX4400-48MP** ma mieszankę 12 portów do 10G i 36 portów do 2.5G. Właśnie z tego powodu **„48MP” nie znaczy „48 × 10G mGig”**!

PoE+/PoE++ to nie samo „może zasilić AP”. Na BOM podaj wymagany standard i moc na porcie. Dodatkowo policz **łączny budżet PoE** względem zasilaczy i redundancji. 48 × 30 W = 1440 W tylko dla urządzeń pobierających pełne 30 W; przy wyższych potrzebach nie wystarczy założenie „każdy port PoE działa niezależnie od PSU”.

**Mini zestawienie, dopiero po wyjaśnieniu roli:**

| Rodzina / wariant przykładowy | Typowy scenariusz | Cecha, którą naprawdę trzeba sprawdzić |
|---|---|---|
| EX4000-12T / EX4000-12P | Mikrooddział, mała szafa | fanless, 12 portów, z/bez PoE |
| EX4000-24P / EX4000-48P | Ekonomiczny access | 1G, budżet PoE, fixed PSU/fans |
| EX4000-24MP / 48MP | Access z częścią mGig | mGig tylko na wybranych portach; PoE++ do określonej mocy |
| EX4100-F-24P / 48P | Standardowy access przy prostszej platformie | fixed power/fans i port map |
| EX4100-24P/48P | Access o podwyższonej odporności | redundantne PSU, Virtual Chassis, EVPN-VXLAN |
| EX4100-H | Trudne warunki środowiskowe | ruggedized, porty i chłodzenie właściwe SKU |
| EX4400-24P/48P | Rozbudowane piętro enterprise | Virtual Chassis, PoE i uplink options |
| EX4400-24MP / 48MP | Wi-Fi 6E/7, większa liczba mGig | 24MP: do 10G; 48MP: **12 do 10G + 36 do 2.5G** |

Źródła modeli i parametrów: [EX4000 datasheet](https://www.juniper.net/gb/en/products/switches/ex-series/ex4000-line-of-ethernet-switches-datasheet.html), [EX4100 specifications](https://www.juniper.net/us/en/products/switches/ex-series/ex4100-ethernet-switch/specs.html), [EX4100-F models](https://www.juniper.net/documentation/us/en/hardware/ex4100-f/topics/concept/EX4100-f-models.html), [EX4400 models](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/concept/ex4400-models.html), [EX4400 multigig specs](https://www.juniper.net/us/en/products/switches/ex-series/ex4400-multigigabit-enterprise-switch/specs.html).

### 1.12.4. Access features i odpowiedniki Cisco — co naprawdę kupujesz?

**Virtual Chassis** pozwala połączyć wspierane EX w jedno urządzenie logiczne z jednym management/control view. Dla inżyniera Cisco odpowiednikiem koncepcyjnym jest **Catalyst StackWise**, lecz szczegółowe limity, porty VC, role RE, wersje oprogramowania i failover są inne. **Nie łącz różnych modeli „bo należą do EX”** bez sprawdzenia tabel kompatybilności VC.

**802.1X/MAB/NAC** pomaga kontrolować urządzenia. Cisco ISE/ClearPass to typowy RADIUS w znanym Ci campusie; po stronie Juniper możesz używać istniejącego NAC lub rozważyć **Mist Access Assurance**. Nie zakładaj identycznych funkcji/licencji i flow change-of-authorization bez weryfikacji.

**LLDP/LLDP-MED, Voice VLAN, PoE, storm control, BPDU protections, MACsec** to funkcje, których oczekujesz w enterprise access. Dany EX może je obsługiwać, lecz **poziom wsparcia (np. MACsec na konkretnych portach i release) trzeba sprawdzić per SKU/feature explorer**, a nie dla samego brandu EX.

**EVPN-VXLAN / Group-Based Policy (GBP)** w campusie Juniper to opcje architektoniczne, a nie powód do automatycznego eliminowania VLAN/STP z każdego site. Jeśli operator nie ma kompetencji utrzymania fabric, prostszy L2 access + redundantny collapsed core nadal może być odpowiednim projektem.

**Wired Assurance** w Mist daje telemetryczne informacje o użytkownikach, AP/switch portach i Service Level Expectations (SLE). Z Cisco skojarz to z operacyjnym monitoringiem Catalyst Center i Cisco DNA/Catalyst Assurance, ale architektura chmury i wymogi licencyjne są inne.



### 1.12.5. Distribution i core — kiedy EX4400, a kiedy EX4650 / QFX / EX9200?

W Cisco szybko postawiłbyś granicę: 9300 na access, 9500 na distribution/core, a może 9600 jako modularny core. W Juniperze podobny proces projektowy prowadzi do innych rodzin i wariantów, ale **nie do idealnego mapowania 9300 ↔ EX4400**.

**Małe i średnie biuro — collapsed core.** Masz dwa redundantne urządzenia distribution/core, a do nich połączone access switche. Nie musisz koniecznie budować trzech odrębnych warstw tylko dlatego, że w szkoleniu jest rysunek „3-tier”. Właśnie w takim przypadku **EX4400-24X** ma sens: oferuje **24 × 1/10GbE (SFP/SFP+) oraz 2 × 40/100GbE uplink**, plus obsługiwane moduły uplink — to switch do agregacji/rdzenia, nie ten sam produkt co 24 porty miedziane EX4400-24P. Na większe wymagania 25/100G rozważ **EX4650**.

**Średni lub większy campus — łącza 10/25/100G.** **EX4650-48Y** w 1U oferuje **48 × 10/25GbE oraz 8 × 40/100GbE**. Gdy wiesz, że wiele szaf IDF będzie oddawać 10/25GbE i ma być rezerwa na wzrost, EX4650 jest naturalnym kandydatem do sprawdzenia, ale musisz policzyć pojemność portów i uplinków oraz wymagany model HA. Nie kupujesz 1U „bo jest szybki”; sprawdzasz także, czy potrzebujesz odpowiedniej liczby 1G, 25G czy 100G.

**Duży campus lub fabric z dużą gęstością uplinków.** **QFX5120** ma warianty 25/100G, które oficjalne projekty Junipera wykorzystują w campus core/distribution/services block. **QFX5130** zapewnia wyższą gęstość 100/400G i uruchamia **Junos OS Evolved** — nie kupuj go jak prostego zamiennika EX4650 do biura liczącego 30 osób. **QFX5700** jest również platformą Junos OS Evolved dla wymagających fabric. **EX9200** to rodzina modularna, dla której znaczenie mają chassis, karty, sloty, zasilanie oraz dobór właściwego hardware i wersji — ekonomicznie i operacyjnie inna liga niż fixed EX.

**Dygresja z życia:** klienci często pytają „czy potrzebuję EX9200, bo główny core ma być future proof?”. Odpowiedź nie zależy od nazwy core. Jeśli dwa EX4650 zapewniają wszystkie porty, właściwy failover i wymaganą skalę, modularny chassis może być niepotrzebną komplikacją. Jeśli natomiast sieć wymaga wielu kart liniowych, gęstości 100/400G i skali routingu/fabric, fixed 1U może przestać wystarczać.

| Poziom projektu | Kandydat do rozważenia | Weryfikuj przed BOM |
|---|---|---|
| Collapsed core mały/średni site | EX4400-24X | 24×1/10G, 2×40/100G, moduły, model VC/fabric |
| Distribution/core ze światłowodami 25/100G | EX4650-48Y | 48×10/25G, 8×40/100G, HA/EVPN, PoE **nie dotyczy** |
| Większy campus services block/aggregation | QFX5120 (odpowiedni wariant) | 10/25/100G, EX/QFX interoperability, EVPN, VC vs ESI-LAG |
| Wysoka gęstość 100/400G | QFX5130 / QFX5700 | **Junos OS Evolved**, porty i optyka, wymagania fabric |
| Modularny campus core | EX9200 | chassis i line cards, PFE/RE, redundantne PSU, lifecycle |

Źródła: [EX4400-24X](https://www.juniper.net/us/en/products/switches/ex-series/ex4400-24x-ethernet-switch/specs.html), [EX4650](https://www.juniper.net/us/en/products/switches/ex-series/ex4650-campus-aggregation-and-core-switch/specs.html), [oficjalna lista platform dla Campus Fabric](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/concept/choose-campus-fabric-topology.html), [QFX5130 Junos OS Evolved](https://www.juniper.net/documentation/us/en/hardware/qfx5130/topics/topic-map/qfx5130-system-overview.html).

**Gotcha z Cisco:** *„QFX zawsze = Data Center, EX zawsze = tylko access”* jest nieprawdą. Oficjalna architektura Campus Fabric umieszcza wybrane EX4400 i EX4650 w core/distribution, a QFX5120/QFX5130 również w campusie. Patrz na **konkretny SKU i potrzebne porty**, nie sam prefiks nazwy.

### 1.12.6. Jakie architektury naprawdę proponuje Juniper?

Dla początkującego w Junos dwie rzeczy mogą wyglądać podobnie na diagramie: *dwa core switche i cztery access switche*. Jednak sposób połączenia i konwergencji może być zupełnie inny.

**Tradycyjny campus z access, distribution, core** to coś, co znasz z Cisco. OSPF lub BGP w core, klasyczne VLAN-y i bramy, redundancja przez właściwe protokoły oraz kontrola STP. Jest to nadal użyteczny model organizacyjny i może być odpowiednim HLD.

**Collapsed core / EVPN multihoming** usuwa potrzebę osobnej fizycznej warstwy distribution w mniejszym campuse. Juniper dokumentuje **EVPN multihoming** z redundantnymi uplinkami, które pozwalają ograniczyć zależność od klasycznego STP w szkielecie. Nie oznacza to „STP już nie istnieje” — urządzenia brzegowe i dowolne połączenia L2 nadal wymagają właściwej ochrony przed pętlami. W Cisco intuicyjnym punktem porównania jest redundantny L2 access do pary core/distribution przez MC-LAG/StackWise Virtual/VSS, ale **mechanika EVPN ESI-LAG nie jest tym samym co StackWise**.

**Campus Fabric Core-Distribution (CRB/ERB).** Gdy masz wiele budynków i chcesz operacyjnie kontrolować, gdzie znajdują się bramy VLAN/VRF, Juniper proponuje fabric z osobnymi rolami core, distribution i access. **CRB** (centrally routed bridging) i **ERB** (edge routed bridging) różnią się m.in. miejscem realizacji routingu i gateway. To są decyzje HLD — nie wpisujesz „EVPN” do BOM bez rozstrzygnięcia, gdzie ma kończyć się L2.

**Campus Fabric IP Clos.** Dla większych wdrożeń używa routed underlay i EVPN-VXLAN overlay, ECMP, możliwości segmentacji oraz podejścia wielostopniowego. W zamian wymaga kompetencji BGP/EVPN, monitoringu underlay, planowania VTEP/loopback i zrozumienia sieci overlay. Dla 2 szaf na jednym piętrze taka złożoność może być zbędna.

~~~mermaid
flowchart LR
 subgraph TRAD["Tradycyjne 3 warstwy"]
    C1["Core"] --- D1["Distribution"]
    D1 --- A1["Access"] 
 end
 subgraph COLL["Collapsed core / EVPN multihoming"]
    C2["Core+Distribution pair"] --- A2["Access VC / ESI-LAG"]
 end
 subgraph FAB["Campus IP Clos"]
    S1["Spine / Core"] --- L1["Leaf / Distribution"]
    L1 --- A3["Access / Endpoint"]
 end
~~~

**Pytanie do architekta:** „Czy potrzebujemy L2 extension między budynkami i host mobility? Kto będzie gateway w każdym VLAN-ie? Jak działa failover, jeśli fizycznie utracimy jeden core?” Bez tych odpowiedzi nazwa „Campus Fabric” nie daje pełnego projektu.

Oficjalne projekty: [topologie Campus Fabric](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/concept/choose-campus-fabric-topology.html), [EVPN multihoming](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/task/campus-fabric-evpn-multi-homing.html), [Core-Distribution](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/task/campus-fabric-core-distribution-configuration.html), [migracja tradycyjnego campus do fabric](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/concept/migrate-to-campus-fabric.html).

### 1.12.7. WLAN: Juniper Mist AP — gdzie jest odpowiednik Cisco WLC 9800?

Tutaj najłatwiej o bardzo kosztowne nieporozumienie. Znasz Cisco 9800: kontroler, profile/tagi WLAN, CAPWAP, central/flex switching, HA SSO. **Nie próbuj szukać „Juniper WLC 9800” w każdej ofercie.** Juniper Mist stosuje inną architekturę zarządzania.

**Mist Cloud** dostarcza zarządzanie, telemetrykę, Wi-Fi Assurance, RF automation, SLE i analitykę. **Mist AP** uruchamia odrębne firmware, a nie Junos. **Ruch klientów może być przekazywany lokalnie** przez AP i LAN zgodnie z SSID/VLAN. Nie musi za każdym razem przechodzić przez chmurę. Jeśli projekt wymaga tunelowania centralnego trafficu do DC/DMZ, stosuje się **Mist Edge**, odpowiednio zaprojektowany i licencjonowany.

**Most z Cisco:** myśl o Cisco 9800 Local/FlexConnect i Catalyst Center/Assurance, ale pamiętaj, że umiejscowienie control/management/data plane oraz operacyjne funkcje Mist Edge mają inny model niż tradycyjny WLC.

Gdy projektujesz nowe biuro, dobór AP zaczynasz od **RF site survey**, a nie od rachunku „1 AP na 70 m²”. O ścianach, źródłach zakłóceń, roamingu i liczbie klientów nie opowie Ci folder sprzedażowy.

Przykładowe dostępne modele:
- **AP24 / AP34 / AP45(E)**: rodziny **Wi-Fi 6E**; AP24/34 typowo 2×2, AP45 klasa 4×4 i scenariusze wyższej gęstości.
- **AP36 / AP37 / AP47**: rodziny **Wi-Fi 7**; AP36 i AP37 oferują różne warianty anten i funkcji, AP47 to wysokowydajny 4×4:4 z dedykowanym radiem scanning.
- **AP64**: Wi-Fi 6E dla odpowiednich instalacji zewnętrznych.
- **AP66**: Wi-Fi 7 do trudnych warunków indoor/outdoor.

Zanim wpiszesz AP47 do oferty, sprawdź **konkretny wariant antenowy** (np. internal/directional/external), radio 6 GHz w danym regionie, wymagany 802.3bt/PoE, uplink mGig/10G, redundancję uplinków i licencję Mist Wi-Fi Assurance.

Źródła: [oficjalne porównanie AP i radia](https://www.juniper.net/documentation/us/en/software/nce/juniper-ai-driven-wired-wireless-network-deployment-guide/wireless_network_deployment_juniper_aps.html) i [portfolio Juniper AP](https://www.juniper.net/us/en/products/access-points.html), [local vs tunneled datapath / Mist Edge](https://www.juniper.net/documentation/us/en/software/nce/juniper-ai-driven-wired-wireless-network-deployment-guide/wireless_network_deployment_juniper_aps.html).

**Gotcha:** **AP47 nie oznacza „AP na Junos 24.4”**. AP ma firmware zarządzane w Mist, dlatego w matrycy software nie wpisujemy mu Junos OS. W projekcie notujesz osobno **firmware AP**, licencję Mist i release Junos na switchu EX.

### 1.12.8. SRX — security/WAN edge od małego oddziału po duży campus

Wyobraź sobie trzy obiekty:
- **punkt sprzedaży / mały oddział**, 20 osób, jeden uplink ISP i backup LTE;
- **średni regionalny site**, 200 osób, dwa ISP i IPS/SSL/segmentation;
- **duży HQ**, wiele VLAN/VRF, wysokie przepustowości firewall/IPS i wymóg HA.

Nie wybieraj SRX na podstawie samego maksymalnego firewall throughput. To tak, jak wybrać silnik samochodu po prędkości maksymalnej, ignorując spalanie, obciążenie i rodzaj drogi.

**SRX300/SRX320** to małe urządzenia branch (desktop); **SRX340/SRX345** są typowym krokiem wyżej dla większych branch offices; **SRX380** jest wydajniejszym branch/security WAN gateway. W campus/regional HQ masz m.in. **SRX1500, SRX1600, SRX2300, SRX4300**, z zupełnie innymi skalami ruchu, liczby sesji, portów i redundancji.

Kluczowa rzecz przy bidowaniu: datasheet podaje różne wielkości **Firewall / IPS / VPN throughput**. Dla przykładu SRX1600 w oficjalnym zestawieniu osiąga nominalnie **24 Gb/s firewall, 21 Gb/s IPS, 18 Gb/s VPN**; SRX2300 odpowiednio **39 / 35 / 36 Gb/s**. To nie gwarancja identycznej przepustowości przy wszystkich usługach równocześnie i w konkretnym miksie małych pakietów. **Oferta 5 Gb/s ISP przy włączonym IPS musi uwzględniać wynik IPS, nie marketingową liczbę firewall.**

Przykładowe rodziny:
| Klasa/rola | Modele do porównania | Co zdecydować przed BOM |
|---|---|---|
| Mały branch/retail | SRX300 / SRX320 | 1/2 WAN, porty lokalne, VPN, liczba sesji, lifecycle |
| Średni branch | SRX340 / SRX345 / SRX380 | IPS/VPN, HA, porty WAN, wymagania licencji i zasilania |
| Regionalny HQ / campus perimeter | SRX1500 / SRX1600 | porty 10/25G, NGFW, redundantne edge |
| Większy campus | SRX2300 / SRX4300 | wyższy throughput, IPS, porty, skala sesji/HA |
| Bardzo duży DC/perimeter | SRX4700 / SRX5800 | tylko jeśli uzasadniają to pojemność, interfejsy, model licencji |

**Most z Cisco:** porównujesz do Cisco Secure Firewall / FTD, a nie do Catalyst 9500. Na SRX masz **security zones**, policies, stateful sessions i możliwość routingu. **Routing reachability ≠ security permit** — to utrwalaliśmy w rozdziale 2.

Źródła i parametry: [oficjalna rodzina SRX](https://www.juniper.net/us/en/products/security/srx-series.html/), [WAN Assurance obsługiwane SRX](https://www.juniper.net/us/en/products/cloud-services/juniper-wan-assurance-datasheet.html).

### 1.12.9. A co ze sprzętowym WAN routerem, SSR i MX?

Nie każdy nowy enterprise site potrzebuje osobnego routera WAN przed SRX. Czasem firewall jest też BGP WAN edge. Czasem potrzebujesz **SSR**, aby realizować rozproszony SD-WAN oparty na sesjach. Czasem masz dużą sieć MPLS/interconnect i wybierasz **MX**.

**Juniper SSR** to osobna linia **Session Smart Routing software**, a nie Junos. Przykładowo aktualne dokumenty wymieniają **SSR120 (small branch)**, **SSR130 (medium branch)**, **SSR400/440 (branch-in-a-box)** oraz **SSR1200/1300/1400/1500** do większych lokalizacji/centrów danych. Architektura SSR wykorzystuje inny model routingu i zarządzania niż typowy SRX z Junos. Nie obiecuj klientowi, że inżynier zaloguje się do SSR i zobaczy identyczne \`show route\` jak na SRX.

**MX204/MX304** i większe MX to kierunek na rozbudowane routing edge, enterprise/carrier aggregation, gdzie istotne są protokoły, skalowanie i zaawansowane funkcje WAN. W zwykłym oddziale SRX albo SSR często odpowie lepiej na wymagania biznesowe. Nie przypisuj MX roli access switcha.

**Mist WAN Assurance** może obejmować zarówno SSR, jak i wspierane SRX. Ale to nie oznacza identycznego systemu operacyjnego na obu platformach.

Źródła: [Session Smart Router](https://www.juniper.net/us/en/products/routers/session-smart-router.html), [SSR modele, throughput i pozycjonowanie](https://www.juniper.net/us/en/products/routers/session-smart-router/session-smart-networking-datasheet.html), [Juniper Router portfolio](https://www.juniper.net/us/en/products/routers.html), [WAN Assurance](https://www.juniper.net/us/en/products/cloud-services/juniper-wan-assurance-datasheet.html).

### 1.12.10. Junos, Junos OS Evolved, Mist AP firmware, SSR — cztery różne światy

Można kupić sprzęt jednej firmy i mieć **cztery różne procesy zarządzania oprogramowaniem**.

**Junos OS:** typowo EX4000/EX4100/EX4400/EX4650/EX9200, wiele QFX5120, SRX i MX. Znasz CLI \`show version\`, \`commit\`, \`show route\`.

**Junos OS Evolved:** np. QFX5130 i QFX5700, z własnymi obrazami **-EVO**. Interfejs CLI jest znajomy, ale nie jest to ten sam pakiet systemowy co na EX4400. Nie próbuj ładować obrazu Evolved na klasyczne EX.

**Mist AP firmware:** oprogramowanie punktów dostępowych AP24/AP37/AP47 itd. aktualizowane zgodnie z mechaniką Mist; **nie Junos OS**.

**Session Smart Routing software:** oprogramowanie SSR, którego cykl wydań jest osobny od Junos.

**Mist Cloud, Wired/Wi-Fi/WAN Assurance, Access Assurance** to usługi platformy chmurowej i subskrypcje — nie wymienia się ich „wersji Junos” jak w routerze.

Dla oferty ważny jest więc **rozdział OS/firmware i licencji per rodzina**.



### 1.12.11. Wersje systemu w 2026 — jak dobrać release i nie narobić sobie problemów przy patchowaniu?

Do tej pory, pracując z Cisco, nie wpisywałeś przecież „najnowszy IOS XE” do Change Request bez sprawdzenia rekomendacji, fixów i zgodności platformy. Potrzebujesz tego samego na Junos — tylko dokładniejszego rozróżnienia.

Juniper prowadzi oficjalny artykuł **JTAC KB21476: [Junos Software Versions — Suggested Releases to Consider and Evaluate](https://supportportal.juniper.net/s/article/Junos-Software-Versions-Suggested-Releases-to-Consider-and-Evaluate)**, ostatnio zaktualizowany **4 listopada 2026**. To punkt wyjścia do zakupu i testów regresyjnych, a **nie nakaz natychmiastowego upgrade wszystkich modeli**. Producent każe czytać release notes, znać ograniczenia danej platformy i zweryfikować kod we własnym wdrożeniu.

**Najważniejsza zasada dla architekta:** „Junos 26.2 istnieje” **nie oznacza**, że 26.2 jest właściwy do EX4650, SRX380, EX9200 albo QFX5130. Sugerowany stabilny target może być starszy niż najnowszy dostępny release.

#### Obecny punkt startowy do planowania — klasyczne EX i SRX (snapshot 16.11.2026)

Dopiero teraz, kiedy wiesz, **po co** jest wersjonowanie per platform, warto zobaczyć zwięzłą tabelę.

| Platforma | Sugerowana linia Junos wg JTAC KB21476 | Kiedy wpiszesz w projekt |
|---|---|---|
| EX4000 | **24.4R2-Sx** | compact/branch access |
| EX4100, EX4100-F, EX4100-H | **24.4R2-Sx** | access w campus/branch |
| EX4100-H-12T | **25.4R1-Sx** | wariant z osobnym targetem |
| EX4400, EX4400-24X, EX4400-48XP/MXP | **24.4R2-Sx** | access lub dystrybucja zgodnie z SKU |
| EX4650 | **24.4R2-Sx** | distribution/core |
| EX9200 | **23.4R2-Sx** | modularny core |
| EX4300-MP (istniejące wdrożenia) | **24.4R2-Sx** | legacy/istniejące środowiska |
| SRX300/320/340/345/380 | **24.4R2-Sx** | branch perimeter |
| SRX1500/1600/2300 | **23.4R2-Sx** | HQ/campus perimeter |
| SRX4300 | **24.4R2-Sx** | większy perimeter |
| vSRX 3.0 | **23.4R2-Sx** | lab/virtual firewall |
| MX204 | **24.4R2-Sx** | routing edge |
| MX304 | **24.4R2-Sx (latest SR)** | routing edge/aggr |

**\`-Sx\` = rodzina service releases, nie rzeczywisty plik do pobrania.** Przed rolloutem wybierz konkretną wersję \`24.4R2-S<number>\` zgodnie z Software Downloads, platformą, Feature Explorer, Release Notes i kompatybilnością Mist.

**Szczególny haczyk SRX300-series:** przy przejściu na **Junos 24.4R1 lub późniejszy** oficjalna tabela odsyła do **TSB92728**, czyli dodatkowych wymagań i ścieżki aktualizacji. Nie uruchamiaj masowego upgrade bez poznania tej procedury. To bardzo dobry przykład pytania, o które można potknąć się na produkcji, mimo doskonałej znajomości routingu.

#### QFX — nie wszystkie używają tego samego Junos

| Platforma | System operacyjny | Wskazana linia wg KB21476 |
|---|---|---|
| QFX5120-48Y / 48T / 32C | **Junos OS** | **25.2R2-Sx** |
| QFX5130 (warianty 32CD, 48C/CM) | **Junos OS Evolved** | **25.2X100-D21-EVO** |
| QFX5700 / QFX5700E | **Junos OS Evolved** | **25.2X100-D35-EVO** |
| QFX5110 | **Junos OS** | **25.2R2-Sx** |

Dla QFX z Junos Evolved widać zupełnie inną notację pakietu. **Nie dobieraj release dla core „z jednej tabeli dla QFX”**. W naszym przykładzie **QFX5130 to Evolved**, EX4650 i QFX5120 to klasyczny Junos.

Źródło systemu QFX5130: [QFX5130 System Overview](https://www.juniper.net/documentation/us/en/hardware/qfx5130/topics/topic-map/qfx5130-system-overview.html); wersje per platforma: [JTAC KB21476](https://supportportal.juniper.net/s/article/Junos-Software-Versions-Suggested-Releases-to-Consider-and-Evaluate).

#### WLAN i SSR — inne numery, inne kalendarze

**Mist AP:** nie używa Junos; firmware AP dobierasz w Mist zgodnie z obsługiwanym modelem, regionem, funkcjami RF i guidance producenta. Nie wymyślaj wersji AP37 „24.4R2”.

**SSR:** ma **oddzielny release train**, a nie Junos. W [oficjalnym kalendarzu SSR](https://support.juniper.net/support/eol/software/ssr/) wymieniony jest **SSR software 7.2** z datą first release 1 lipca 2026 i odrębnymi datami wsparcia. To **informacja o wydaniu**, nie rekomendacja, że 7.2 jest właściwy dla każdego SSR120/130.

**Junos release lifetime:** sprawdzaj [Junos OS Dates & Milestones](https://support.juniper.net/support/eol/software/junos/), EoE/EoS i support dla konkretnego hardware. Przykładowo sama dostępność wersji 25.4/26.2 nie oznacza, że trafiła na każdą serię switchy.

**Praktyczna checklista do CR/patchingu:**
- Zapisz dokładny PID/serial, model Virtual Chassis (i wszystkie członki), licencje, aktualną wersję z \`show version\` i \`show chassis hardware\`.
- Sprawdź czy obecny boot image, storage i upgrade path spełniają wymagania.
- Wybierz target per platforma ze wsparciem producenta, sprawdź suggested release, known issues i wymagane security advisories.
- Zweryfikuj zgodność z Mist/Wired Assurance i danymi typami optyki/PoE/portów.
- Ustal rollback, management access, maintenance window i wpływ na VC/HA/EVPN.
- Po upgrade przetestuj VLAN/IRB, uplink, routing adjacency, PoE, AP, NAC, logi i monitoring.

**Urządzenie: EX-CORE-01 — tryb operational**

~~~junos
show version
show chassis hardware
show system storage
show system alarms
show chassis alarms
~~~

**Urządzenie: EX-ACCESS-VC01 — tryb operational (jeśli Virtual Chassis obsługiwany)**

~~~junos
show virtual-chassis status
show version
show chassis hardware
~~~

To pozwala zebrać podstawę pod decyzję, ale **nie zastępuje** pełnego precheck dla konkretnego modelu.

### 1.12.12. Trzy wzorce greenfield: jak rozmawiać z klientem

#### Scenariusz A — mały branch, 25 pracowników

Klient ma jedną szafę, kilka AP, dwie drukarki i około 20 komputerów. Chce awaryjny ISP/LTE, prostą politykę guest/corp i nie chce osobnego kontrolera WLAN.

**Koncepcja, nie oferta:** 1–2 przełączniki access z odpowiednimi portami PoE (np. EX4000-24P/48P lub EX4100-F), dobrane po policzeniu portów i zasilania, SRX300/320 lub SSR120 zależnie od modelu WAN/security, kilka Mist AP dobranych po RF survey, usługi Mist zgodnie z wymaganiami operacyjnymi.

**Haczyk:** **jeden** switch access plus jeden SRX to nadal pojedyncze punkty awarii. Jeżeli klient wymaga wysokiej dostępności, minimalny BOM jest niewłaściwy. Zamówienie nie może wyprzedzać uzgodnienia SLA.

#### Scenariusz B — średni site, ~120 użytkowników na dwa piętra

Klient chce VLAN Voice, 802.1X, guest Wi-Fi, kamery i 10G uplinki.

**Koncepcja:** kilka EX4100-48P/EX4400-48P (według wymaganej liczby portów i PoE), 1–2 EX4400-24X na collapsed core (jeżeli spełnia skalę/HA) albo EX4650, odpowiednia para SRX z planem HA/WAN, punkty AP34/36/37 w zależności od wymagań radio i dostępności, Mist Wired/Wi-Fi Assurance oraz NAC (istniejący ClearPass/ISE lub Mist Access Assurance po zaplanowaniu migracji).

**Haczyk:** 120 użytkowników to **nie** 120 portów. Dodaj drukarki, AP, telefony, kamery, urządzenia sal konferencyjnych, helpdesk, dostęp administracyjny i spare. Następnie **oddziel porty 1G od portów mGig**, nie sumuj ich w jednej anonimowej kolumnie.

#### Scenariusz C — HQ/campus na 3 piętrach, 320 stanowisk (case study do BOM)

Na początku rozdziału klient powiedział: 320 stanowisk, 35 telefonów IP, 28 kamer i Wi-Fi 7. Załóżmy **18 AP jako wyłącznie wartość ilustracyjną**, którą RF designer ma później zastąpić z pomiarów. **Nie jest to rekomendowana liczba AP na trzy piętra bez survey.**

Najpierw liczymy **fizyczne porty Ethernet** (przyjmujemy niezależne kable dla urządzeń; jeśli telefon ma PC passthrough, wynik może być inny):
- 320 stanowisk PC,
- 35 telefonów IP,
- 28 kamer,
- 18 AP (assumption),
- **401 końcówek** przed zapasem i przed portami infrastruktury.

Dopiero później wybieramy klasy portów:

**1G copper:** PC + voice + kamery = 383.  
**mGig/PoE dla AP:** 18 portów jako założenie; prędkość i potrzebną moc każdej końcówki uzgadniamy według modelu AP.

**Przykład nieoptymalizowanego kandydata do analizy HLD:** 9 × EX4100-48P daje 432 porty 1G, a 3 × EX4400-24MP daje 72 porty mGig (po jednym na piętro). To oznacza 49 wolnych portów 1G, czyli około 11% zapasu w tej klasie, oraz aż 54 wolne mGig — **bardzo nierówną rezerwę**. Jest to materiał do dyskusji, a nie gotowy do zakupu BOM. Wariant może być zbyt drogi mGig i mieć zbyt małą rezerwę 1G, zależnie od polityki klienta.

Jeśli klient wymaga np. **20% wolnych portów 1G na dzień odbioru**, 383/0.8 ≈ 479, więc trzeba zaplanować ok. 480 portów **1G** z odpowiednią dystrybucją po piętrach (np. 10 × 48-portowy 1G access, dopiero po rozbiciu na IDF). To **nie** oznacza, że wszystkie 480 portów muszą wspierać PoE++; podział na PoE/non-PoE rozstrzyga osobny rachunek urządzeń.

Dla AP potrzebujemy co najmniej 18 odpowiednich mGig portów **we właściwych szafach** — nie centralnie na jednym piętrze. Zależnie od wymagań i cen można rozważyć mniejsze wyspy mGig/PoE, ale dopiero po sprawdzeniu port-mix EX4000/EX4100/EX4400.

**Przykładowy szkic HLD (nie zlecenie zakupowe):**

~~~mermaid
flowchart TB
 WAN1["ISP-A"] --> FW1["SRX pair / HA\nthroughput wg IPS/VPN"]
 WAN2["ISP-B"] --> FW1
 FW1 --> CORE1["Core-A\nEX4650 / inny model po capacity"]
 FW1 --> CORE2["Core-B\nEX4650 / inny model po capacity"]
 CORE1 --- CORE2
 CORE1 --> F1["Floor 1 — IDF\nEX4100 1G + EX4400 mGig"]
 CORE2 --> F1
 CORE1 --> F2["Floor 2 — IDF\nEX4100 1G + EX4400 mGig"]
 CORE2 --> F2
 CORE1 --> F3["Floor 3 — IDF\nEX4100 1G + EX4400 mGig"]
 CORE2 --> F3
 F1 --> USERS1["PC / phones / AP / cameras"]
 F2 --> USERS2["PC / phones / AP / cameras"]
 F3 --> USERS3["PC / phones / AP / cameras"]
~~~

**Co jeszcze policzyć zanim oferta zostanie podpisana?** Uplinki 10/25/100G i liczba transceiverów, optyka (SM/MM, trasy, tłumienie), patchcordy, rack space, zasilanie A/B, liczba PSU, redundancy, AP mount kits, Site Survey, NAC i licencje, Mist subscription terms, obsługa firmware, okres support 3/5 lat, spares 1–2 sztuki, access to management cloud, bieżące RMA lead times.

### 1.12.13. Od HLD do BOM — przykładowe kolumny i zapytania do producenta

Do negocjacji z dostawcą nie wystarczy „EX4400 × 10”. **Musisz odróżnić logiczny model urządzenia od konkretnego orderable SKU**. Ten sam EX4400 może mieć inną konfigurację portów, typ zasilania, airflow, wariant optyki i wsparcie PoE.

Praktyczny BOM powinien mieć co najmniej:
- **Role / layer** (access floor 1, collapsed core A/B, WAN FW A/B, AP Wi-Fi);
- **Model family + pełny hardware PID/SKU**;
- **Quantity + spare quantity**;
- **Port profile** (copper 1G, 2.5/5/10G, SFP+/SFP28, QSFP28) z uzasadnieniem;
- **PoE** (zasilanie pojedynczego portu, łączny budżet przy redundancji);
- **Uplinks** (liczba, prędkość, typ transceivera, długość/sztuki);
- **Power** (PSU, AC/DC, redundancy, power cords, airflow);
- **License/subscription** (Mist Wired/Wi-Fi/WAN/Access Assurance, security subscriptions, termin i sposób liczenia);
- **Support & software** (data rozpoczęcia, okres, warunki RMA, sugerowany release, JTs/bugs i testy);
- **Integration** (NAC, TACACS/RADIUS, DHCP/DNS, Syslog/SIEM, backup, ZTP, cloud egress);
- **Source & assumptions** (link do specyfikacji, data weryfikacji, status „to confirm”, wycena dystrybutora).

**Nie wymyślaj cen ani zamienników SKU.** Ceny, bundle i dostępność wymagają aktualnego quote od autoryzowanego kanału HPE Juniper Networking. Dopiero wtedy przygotowujesz *price-to-performance* warianty: minimalny, recommended, HA/expandable.

**Pytania do architekta przed podpisaniem HLD:** Czy core ma być L3 boundary czy EVPN border? Czy potrzebujemy „dual-homed” access? Czy cloud-managed switching jest wymaganiem? Czy istniejące Cisco ISE/ClearPass ma zostać? Czy AP tuneluje do Mist Edge? Czy chcemy NAC per klient czy per AP? Czy są wymagania lokalizacji danych i dostępu z chmury? Czy planowane są późniejsze 25/100G? Czy dwa ISP są naprawdę niezależne fizycznie? Czy utrata szafy IDF lub zasilania A eliminuje cały site?

### 1.12.14. Gotchas dla inżyniera Cisco i dla autora oferty — z życia

**Haczyk 1 — EX4100-F ≠ EX4100.** „F” może oznaczać różnice w PSU/fans i możliwościach. Jeśli projekt wymaga redundantnych hot-swappable PSU na access, nie traktuj obu linii identycznie. [EX4100-F Models](https://www.juniper.net/documentation/us/en/hardware/ex4100-f/topics/concept/EX4100-f-models.html).

**Haczyk 2 — EX4400-24X ≠ EX4400-24P.** Jedna rodzina, inne porty i role. Do optical aggregation model „X” to sensowny wybór; do 24 komputerów z PoE sam EX4400-24X może być błędem. [EX4400-24X](https://www.juniper.net/us/en/products/switches/ex-series/ex4400-24x-ethernet-switch/specs.html).

**Haczyk 3 — VC porty mogą być fabrycznie zajęte dla Virtual Chassis.** Uplink QSFP z datasheet nie zawsze jest wolnym zwykłym uplinkiem bez świadomej konfiguracji. W EX4400 porty rear QSFP28 bywają domyślnie VCP, a wariant 24X wymaga określonego trybu. [EX4400 System Overview](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/topic-map/ex4400-system-overview.html).

**Haczyk 4 — 48 portów PoE nie oznacza 48 × 90 W.** Liczy się **PoE budget konkretnego PSU**, a także zapas przy awarii jednego zasilacza. [EX4000 datasheet](https://www.juniper.net/gb/en/products/switches/ex-series/ex4000-line-of-ethernet-switches-datasheet.html).

**Haczyk 5 — Mist ≠ Junos na AP.** AP ma osobne firmware; Mist Cloud jest usługą zarządzania, a ruch klientów może mieć local breakout. [Juniper wireless deployment](https://www.juniper.net/documentation/us/en/software/nce/juniper-ai-driven-wired-wireless-network-deployment-guide/wireless_network_deployment_juniper_aps.html).

**Haczyk 6 — Juniper SSR ≠ Junos SRX.** W jednej ofercie na WAN mogą być dwie linie produktów z zupełnie innym systemem i procedurami. [SSR datasheet](https://www.juniper.net/us/en/products/routers/session-smart-router/session-smart-networking-datasheet.html).

**Haczyk 7 — „nowy Junos” ≠ „sugerowany Junos”.** Gdy EX4400 ma suggested 24.4R2-Sx, nie wybieraj automatycznie 26.2 tylko dlatego, że 26.2 istnieje w kalendarzu wydań. [JTAC KB21476](https://supportportal.juniper.net/s/article/Junos-Software-Versions-Suggested-Releases-to-Consider-and-Evaluate).

**Haczyk 8 — Evolved to inny software.** QFX5130 w campusie może wymagać obrazów Junos OS Evolved, podczas gdy EX4650 klasycznych Junos. [QFX5130](https://www.juniper.net/documentation/us/en/hardware/qfx5130/topics/topic-map/qfx5130-system-overview.html).

**Haczyk 9 — porty access w BOM nie są zamienne 1:1.** Port 1G non-PoE nie zasili nowego AP Wi-Fi 7, a szybki uplink 100G nie musi być dostępny jako normalny port w obecnym trybie Virtual Chassis. Podziel port count na klasy.

**Haczyk 10 — Cloud assurance to nie dowód licencji NAC.** Wired Assurance monitoruje działanie, a **Mist Access Assurance** jest oddzielną usługą NAC/licencjonowania klientów. [Access Assurance](https://www.juniper.net/documentation/us/en/software/mist/mist-access/mist-access.pdf).

### 1.12.15. Zestaw dokumentów, z których korzystasz przy każdym greenfield

Najpierw **Validated Designs** dla architektury kampusowej (jak połączyć urządzenia), potem **Datasheet + Hardware Explorer** dla SKU/portów, następnie **Feature Explorer** dla funkcji w konkretnej wersji, wreszcie **JTAC Suggested Releases i Release Notes** dla wdrożenia.

- [Juniper Campus Fabric design & supported models](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/concept/choose-campus-fabric-topology.html)
- [Juniper Campus migration HLD](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/concept/migrate-to-campus-fabric.html)
- [Juniper AI-Driven Wired/Wireless Deployment Guide](https://www.juniper.net/documentation/us/en/software/nce/juniper-ai-driven-wired-wireless-network-deployment-guide/juniper-ai-driven-wired-wireless-network-deployment-guide.pdf)
- [EX4000 datasheet](https://www.juniper.net/gb/en/products/switches/ex-series/ex4000-line-of-ethernet-switches-datasheet.html)
- [EX4100/EX4400 models](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/concept/ex4400-models.html)
- [EX4650 specs](https://www.juniper.net/us/en/products/switches/ex-series/ex4650-campus-aggregation-and-core-switch/specs.html)
- [QFX5130 OS and ports](https://www.juniper.net/documentation/us/en/hardware/qfx5130/topics/topic-map/qfx5130-system-overview.html)
- [Juniper AP portfolio](https://www.juniper.net/us/en/products/access-points.html)
- [SRX portfolio](https://www.juniper.net/us/en/products/security/srx-series.html/)
- [SSR datasheet](https://www.juniper.net/us/en/products/routers/session-smart-router/session-smart-networking-datasheet.html)
- [JTAC KB21476 (recommended to consider/evaluate)](https://supportportal.juniper.net/s/article/Junos-Software-Versions-Suggested-Releases-to-Consider-and-Evaluate)
- [Junos OS EoE/EoS](https://support.juniper.net/support/eol/software/junos/)
- [SSR software milestones](https://support.juniper.net/support/eol/software/ssr/)
- [Juniper Hardware Compatibility Tool](https://apps.juniper.net/hct/)
- [Juniper Feature Explorer](https://apps.juniper.net/feature-explorer/)

### 1.12.16. Najważniejsza umiejętność po tym podrozdziale

Nie musisz pamiętać na pamięć, czy EX4100 ma 4 czy 6 konkretnych uplinków w każdym wariancie. Masz wiedzieć, **której rodziny i modelu poszukać, jakie pytanie zadać przed wyborem i gdzie sprawdzić szczegóły**, oraz rozpoznać typowe pułapki:

- Access EX4000/EX4100/EX4400 i mGig PoE dla AP.
- Collapsed core EX4400-24X albo EX4650 po capacity/HA.
- Campus core QFX/EX4650/EX9200 zgodnie ze skalą.
- Perimeter SRX, SD-WAN SSR (nie Junos), WAN routing MX tam, gdzie to uzasadnione.
- Wi-Fi Mist AP + cloud assurance, opcjonalny Mist Edge zależnie od przepływu.
- Aktualny rekomendowany release **per model** z oficjalnej KB, nie „najnowszy Junos”.

**Ćwiczenie projektowe:** na podstawie case study 320 użytkowników przygotuj dwa HLD i dwa warianty orientacyjnego BOM: (A) traditional/collapsed core, (B) EVPN multihoming. Dla każdego pokaż port/PoE distribution per IDF, uplinks, dwa scenariusze failover, policy/ZTP/Mist, rozliczenie support i oddzielne listy ryzyk. **Nie dobieraj finalnych SKU i cen bez RF survey i wywiadu z klientem.**

---

## 1.RECAP. Miejsce tego rozdziału w pełnym podręczniku

Punkty **1.1–1.11** to powtórka fundamentów JNCIA-Junos. **1.12 [EXT] to dodatkowy warsztat portfolio i greenfield**, który pomaga inżynierowi pracować przy rzeczywistych projektach i ofertach. Nie jest literalnym podpunktem blueprintu JNCIA, JNCIS ani JNCIP.

Dalej: **rozdział 2** w tym repozytorium rozkłada na części działanie Junos OS, RE/PFE, transit i exception traffic. Po jego przejściu łatwiej zrozumieć, dlaczego EX i SRX mogą obydwa uruchamiać Junos, a jednak odpowiadać za inne funkcje architektury. Rozdziały **14–21** rozwijają switching, HA, 802.1X, CoS i EVPN.

**Granica poprawności:** tu uczymy się shortlistingu i projektowania. Przed rzeczywistym przetargiem trzeba uzyskać **autoryzowany current BOM/quote** i potwierdzić kompatybilność funkcji, licencji, dostępność hardware i wersji Junos.

**Blueprint:** 1.1–1.11 [A] oraz 1.12 [EXT] — praktyczne portfolio producenta i greenfield networking.
