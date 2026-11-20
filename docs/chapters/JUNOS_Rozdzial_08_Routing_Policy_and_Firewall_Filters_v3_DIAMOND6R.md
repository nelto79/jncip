# Rozdział 8. Routing Policy and Firewall Filters
## Które trasy wolno reklamować, którą drogę wybrać i gdzie zatrzymać niepożądany pakiet?

**JUNOS Enterprise Routing & Switching · JNCIA-Junos [A] → JNCIS-ENT [S] → JNCIP-ENT [P]**

**Zakres:** cele 8.1–8.14 zunifikowanego blueprintu. **Wariant laboratorium:** [DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md), sześć routerów vSRX.  
**Status:** pełny materiał dydaktyczny; lab jest **PLANNED — NOT RUNTIME VALIDATED**. Przykłady opisują oczekiwane zachowanie, a nie wyniki z uruchomionej maszyny. Nazwy interfejsów i wsparcie poszczególnych akcji należy sprawdzić na posiadanym Junos/vSRX.

---

## Dlaczego w ogóle potrzebujemy polityk?

W poprzednich rozdziałach nauczyłeś się sprawdzać, **czy router zna trasę** i jak wybiera najlepszą drogę do celu. Teraz pojawia się pytanie bardziej praktyczne: **czy każdą znaną trasę należy przekazać innemu routerowi?**

W naszej sieci istnieją dwa niezależne połączenia pomiędzy oddziałami. R5 to router klienta po lewej stronie, R6 — router klienta po prawej. Pomiędzy nimi znajdują się cztery routery rdzenia R1–R4. Górny korytarz prowadzi przez R2, dolny przez R3. Wszystkie sześć urządzeń pracuje na Junos, a na potrzeby tego rozdziału zestawimy sześć relacji eBGP.

Na początku wszystko jest proste: R6 ma sieć aplikacyjną **172.16.60.0/24**. Powinna być osiągalna z R5 przez oba korytarze. R6 ma również sieć usług **172.16.61.0/24**, dla której preferujemy dolny korytarz przez R3, oraz testową sieć **172.16.62.0/24**, której R5 nie powinien ani instalować, ani używać.

W pewnym momencie R1 poznaje trzy prefiksy od R4 — pośrednio przez R2 i R3. Jedne powinien zaakceptować, inne odrzucić. Dwie poprawne trasy powinien wybierać w różny sposób. Potem pojawia się kolejny wymóg: nawet jeśli routing działa, nie każdy pakiet z R5 powinien być wpuszczony do rdzenia.

Właśnie dlatego Junos ma **dwa różne języki filtracji**. Pierwszy, routing policy, decyduje o informacjach o trasach. Drugi, stateless firewall filter, podejmuje decyzje dotyczące rzeczywistych pakietów. Na vSRX działa ponadto trzeci mechanizm — **stateful security policy**, który może dopuścić lub odrzucić nową sesję.

Nie będziemy mieszać tych mechanizmów. Zaczniemy od routing policy, ponieważ najpierw sieć musi wiedzieć, **dokąd prowadzi droga**. Do packet filtering przejdziemy dopiero wtedy, gdy droga już istnieje.

### Nasza wspólna topologia DIAMOND-6R

~~~mermaid
flowchart LR
    R5["R5 · CE-A<br/>AS 65105<br/>172.16.50.0/24"]
    R1["R1 · EDGE-A<br/>AS 65001"]
    R2["R2 · CORE-1<br/>AS 65002"]
    R3["R3 · CORE-2<br/>AS 65003"]
    R4["R4 · EDGE-B<br/>AS 65004"]
    R6["R6 · CE-B<br/>AS 65106<br/>172.16.60/61/62.0/24"]
    R5 ---|"LAN-A 192.0.2.0/24"| R1
    R1 ---|"10.12.0.0/30"| R2
    R1 ---|"10.13.0.0/30"| R3
    R2 ---|"10.24.0.0/30"| R4
    R3 ---|"10.34.0.0/30"| R4
    R4 ---|"LAN-B 198.51.100.0/24"| R6
~~~

*Rysunek 8.1. Dwie ścieżki między CE-A i CE-B. R1–R4 zachowują adresację z wcześniejszych laboratoriów. R5 zastępuje PC-A, a R6 zastępuje PC-B; są teraz routerami vSRX, nie komputerami.*

| Odcinek | Pierwszy koniec | Drugi koniec |
|---|---|---|
| LAN-A | R5: 192.0.2.10/24 | R1: 192.0.2.1/24 |
| R1–R2 | R1: 10.12.0.1/30 | R2: 10.12.0.2/30 |
| R1–R3 | R1: 10.13.0.1/30 | R3: 10.13.0.2/30 |
| R2–R4 | R2: 10.24.0.1/30 | R4: 10.24.0.2/30 |
| R3–R4 | R3: 10.34.0.1/30 | R4: 10.34.0.2/30 |
| LAN-B | R4: 198.51.100.1/24 | R6: 198.51.100.10/24 |

Router ID i podstawowe loopbacki pozostają \`10.255.255.1/32\`…\`10.255.255.6/32\` odpowiednio dla R1–R6. Na R5 dodamy biznesowe \`172.16.50.1/24\` i \`172.16.51.1/24\`, a na R6 — \`172.16.60.1/24\`, \`172.16.61.1/24\` oraz izolowaną trasę testową \`172.16.62.0/24 discard\`. Te dodatkowe prefiksy służą nauce selekcji i nie zmieniają istniejących czterech łączy rdzenia.

W tym rozdziale **adres CE-A 192.0.2.10 nie jest adresem komputera PC-A**. Router R5 sam może generować testowe pingi ze swojego loopbacku, ale taki test nie jest identyczny z ruchem tranzytowym od fizycznego hosta za CE. To rozróżnienie okaże się ważne przy analizie filtrów i sesji SRX.

---

## 8.1. Default routing policies — co Junos zrobi, zanim napiszesz pierwszy term? [A]

Zacznijmy od czegoś, co początkowo wygląda na drobiazg, a w BGP potrafi całkowicie zmienić zachowanie sieci.

Załóżmy, że R4 nauczył się od R6 prefiksu 172.16.60.0/24. R4 ma sąsiadów eBGP R2 i R3. Nie skonfigurowałeś jeszcze żadnej polityki eksportu. Czy R4 przekaże otrzymaną trasę dalej?

**Tak — w typowym klasycznym Junos, jeśli trasa BGP jest aktywna, kwalifikuje się do reklamy i nie zabraniają tego reguły samego BGP.** Domyślna polityka eksportu BGP pozwala na dalsze reklamowanie aktywnych tras BGP. To właśnie w naszym diamencie umożliwia drogę R6 → R4 → R2/R3 → R1, zanim wprowadzimy własne ograniczenia.

Ale na R6 jest także bezpośrednio podłączona sieć 172.16.60.0/24. Czy samo zestawienie eBGP do R4 spowoduje, że R6 automatycznie wyśle własną trasę **Direct**? **Nie.** Domyślny eksport BGP dotyczy aktywnych tras BGP, a nie każdej trasy \`Direct\` czy \`Static\`. Dla pierwszej reklamy własnego LAN potrzebna jest jawna policy eksportu.

To rozróżnienie jest fundamentalne:

- **R6 → R4:** lokalna trasa \`Direct\` musi zostać świadomie wprowadzona do BGP przez \`export\`.
- **R4 → R2/R3:** trasa poznana już przez BGP może być domyślnie dalej reklamowana, jeśli jest aktywna i kwalifikuje się protokołowo.

Nie ma tu sprzeczności. Router raz pełni rolę **źródła nowej informacji BGP**, a raz **pośrednika przekazującego informację BGP**. W pierwszym przypadku domyślna polityka nie zakłada, że chcesz ujawniać wszystkie lokalne sieci. W drugim zakłada wymianę aktywnych tras BGP, o ile nie wprowadzisz ograniczeń.

### Dlaczego OSPF i IS-IS zachowują się inaczej?

BGP wymienia informacje o ścieżkach do prefiksów. OSPF i IS-IS są protokołami *link-state*: najpierw rozpowszechniają informacje o topologii przez LSA/LSP flooding, a dopiero na tej podstawie obliczają trasy.

Kiedy dołączysz interfejs R1–R2 do OSPF, informacje o tym łączu pojawią się w domenie OSPF. **Nie trzeba** tworzyć \`protocols ospf export ACCEPT-ALL\`, aby OSPF zaczął opisywać własne interfejsy. Export policy OSPF służy przede wszystkim do **redystrybucji** innych tras do domeny OSPF.

Dlatego OSPF i IS-IS mają domyślny **export reject** w języku routing policies. Nie oznacza to zablokowania natywnego flooding LSA/LSP. Oznacza: „nie wprowadzaj automatycznie do tego protokołu tras obcego pochodzenia”.

| Protokół | Domyślny import | Domyślny export | Wniosek operacyjny |
|---|---|---|---|
| **BGP** | Akceptuj kwalifikujące się trasy BGP | Reklamuj aktywne trasy BGP zgodnie z regułami protokołu | Ryzyko tranzytu / route leak, jeśli nie filtrujesz |
| **OSPF** | Akceptuj wewnętrzne informacje OSPF; kontrola zewnętrznych ma ograniczony zakres | Nie redystrybuuj automatycznie obcych tras | Natywny flooding nadal działa |
| **IS-IS** | Akceptuj informacje IS-IS, z platformowo i wersyjnie ograniczonym import | Nie redystrybuuj automatycznie obcych tras | Natywne LSP nadal są rozgłaszane |
| **RIP** | Akceptuj trasy RIP | Brak automatycznego eksportu bez polityki | Do reklam zwykle potrzeba jawnej policy |

Jest tu jeszcze szczegół dla poziomu Professional: zachowania domyślne mogą zależeć od platformy, np. PTX, i mogą zostać zmienione przez funkcje bezpieczeństwa eBGP lub polityki nadane wyżej w konfiguracji. Nie wolno też ignorować zasad takich jak AS_PATH loop detection, iBGP split horizon czy reachability BGP next-hop. Słowa „domyślnie reklamuj” nie oznaczają „wyślij wszystko, niezależnie od protokołu”.

**Pierwszy wniosek rozdziału:** jeśli w Junos chcesz być pewny, *co* zostanie wysłane do sąsiada BGP, nie polegaj na niedopasowaniu jako mechanizmie odrzucenia. Ustal jasną whitelistę i zakończ ją \`then reject\`. Podobnie jak w Cisco route-map, ale z ważną różnicą: w Junos **routing policy nie ma automatycznego „deny any” na końcu**.

*Dokumentacja: [Default Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-actions-defaults.html), [Basic BGP Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html).*

---

## 8.2. Import i export — określ kierunek względem tablicy, nie portu [A]

W Cisco przyzwyczailiśmy się do \`route-map ... in\` lub \`out\` pod sąsiadem BGP. To działa intuicyjnie, dopóki nie połączymy BGP z redystrybucją OSPF, \`network\`, \`redistribute\` i innymi mechanizmami.

W Junos wygodniej przyjąć jedno konsekwentne miejsce obserwacji: **Routing Information Base**, czyli \`inet.0\`.

**Import** to kierunek **od protokołu w stronę tablicy tras**. Na R1 otrzymujemy od R2 i R3 BGP UPDATE zawierające 172.16.60.0/24. Polityka importu może odrzucić tę trasę albo zmienić jej atrybut LOCAL_PREF, zanim R1 dokona ostatecznego wyboru drogi.

**Export** to kierunek **z tablicy tras do protokołu**. Na R6 mamy lokalny \`Direct 172.16.60.0/24\`. Export policy decyduje, czy BGP może ogłosić tę informację R4. Podobnie export OSPF decyduje, czy do OSPF wprowadzimy statyczną trasę zewnętrzną.

~~~mermaid
flowchart LR
    B["R2/R3<br/>BGP UPDATE"] -->|"IMPORT policy R1"| RIB["R1 inet.0<br/>kandydaci i aktywne trasy"]
    RIB -->|"EXPORT policy R1"| A["R5 lub inny BGP peer<br/>reklamowane prefiksy"]
    RIB -->|"forwarding-table export<br/>inny przypadek użycia"| FIB["Forwarding table / FIB"]
~~~

*Rysunek 8.2. Perspektywa RIB. Na imporcie kontrolujemy otrzymane informacje; na eksporcie — to, co router może rozgłosić. Eksport do FIB służy innym celom, np. ECMP, i nie jest tym samym co eksport do BGP.*

### Praktyczny przykład: R6 ma sieć, R4 powinien ją poznać

Na R6 mamy trzy wpisy routingu: \`Direct 172.16.60.0/24\`, \`Direct 172.16.61.0/24\` i \`Static 172.16.62.0/24 discard\`. Chcemy reklamować je **tylko w izolowanym laboratorium** do R4, aby później R1 mógł przetestować import whitelist i odrzucić jedną z nich. Samo skonfigurowanie BGP nie wystarczy, bo Direct/Static nie są automatycznie redystrybuowane.

**R6-vSRX — Configuration Mode, selektywna reklama trzech prefiksów testowych**

~~~junos
set policy-options policy-statement CE-B-OUT term BUSINESS from protocol direct
set policy-options policy-statement CE-B-OUT term BUSINESS from route-filter 172.16.60.0/24 exact
set policy-options policy-statement CE-B-OUT term BUSINESS from route-filter 172.16.61.0/24 exact
set policy-options policy-statement CE-B-OUT term BUSINESS then accept
set policy-options policy-statement CE-B-OUT term TEST from protocol static
set policy-options policy-statement CE-B-OUT term TEST from route-filter 172.16.62.0/24 exact
set policy-options policy-statement CE-B-OUT term TEST then accept
set policy-options policy-statement CE-B-OUT term REJECT-REST then reject
set protocols bgp group TO-R4 export CE-B-OUT
~~~

To przykład **kompletnej intencji**, nie komendy do bezrefleksyjnego wdrożenia. Zakładamy, że masz już trzy wymienione trasy w RIB, prawidłowy eBGP peer R4 oraz potwierdzony mapping interfejsów. Trasa \`172.16.62.0/24 discard\` nie prowadzi do żadnej rzeczywistej usługi — służy do testu filtrowania otrzymanych prefiksów, a nie komunikacji host-to-host.

Czytając tę konfigurację, widzisz od razu: są dwa termy zezwalające na konkretne prefiksy i jeden term końcowy. Później sprawdzimy, **co dokładnie wysłał R6**, zamiast uznawać sam \`commit\` za dowód.

**R6-vSRX — Operational Mode, po uruchomieniu sesji BGP**

~~~junos
show route 172.16.60.0/24 exact
show route 172.16.61.0/24 exact
show route 172.16.62.0/24 exact
show route advertising-protocol bgp 198.51.100.1
~~~

Ostatnia komenda ma pokazać trzy prefiksy po policy eksportu. To, że \`172.16.62.0/24\` jest typu discard, nie czyni z niej reachability do hosta; nie testujemy jej przez ping. Sprawdzamy tylko, że NLRI może być otrzymane przez BGP i następnie świadomie odrzucone na R1.

### Dlaczego RIB nie zawsze znaczy „to jest aktywna droga”?

Na R1 172.16.60.0/24 przychodzi dwiema ścieżkami. Jedna via R2, druga via R3. Import policy może zaakceptować obie, lecz tylko jedna ścieżka zwycięży w wyborze najlepszego BGP route (jeżeli nie wdrożymy multipath). Później RIB musi jeszcze wybrać aktywną trasę względem ewentualnych tras innych protokołów.

Domyślna BGP export policy zwykle ocenia **aktywne trasy**, a nie każdą kandydującą. Jeśli ten sam prefiks istnieje jako aktywna OSPF/10 i nieaktywna BGP/170, nie wystarczy \`from protocol bgp\`, aby wyeksportować tę nieaktywną ścieżkę z jej oryginalnymi atrybutami. Istnieje funkcja \`advertise-inactive\`, lecz wymaga odpowiedniej konfiguracji i nie jest domyślnym obejściem każdego ograniczenia. Nie myl jej z ręczną redystrybucją OSPF do BGP — to dwa inne sposoby pozyskania informacji do reklamy.

Najważniejszy zestaw dowodów ma zatem trzy części: **co sąsiad wysłał**, **co R1 przyjął i wybrał**, **co R1 następnie ogłosił**. W sekcji 8.4 zobaczymy odpowiednie polecenia i wyjątki.

*Dokumentacja: [Understanding Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-overview.html), [BGP Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html).*
