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

---

## 8.3. Jak Junos wykonuje routing policy — termy, kolejność i akcje [A]

W tej lekcji warto oddzielić dwie rzeczy. **Kolejność termów w polityce ma znaczenie.** Natomiast **kolejność wierszy \`route-filter\` wewnątrz jednego termu nie jest zwykłą kolejnością ACL**. Druga zasada będzie ważna w sekcji 8.6.

Najpierw poznajmy podstawowy mechanizm. Załóżmy, że R1 otrzymuje od R2 prefiks 172.16.60.0/24. Dla tego prefiksu ma obowiązywać następująca polityka: wybrane trasy od R6 oznaczamy, potem sprawdzamy whitelistę, a na końcu podejmujemy ostateczną decyzję.

### Policy statement to program, a term to pojedyncza decyzja

Każdy \`policy-statement\` ma nazwę. Wewnątrz może zawierać kolejne \`term\`. Każdy term zawiera opcjonalne warunki \`from\` i akcje \`then\`.

Czytaj je dosłownie:

> Jeżeli trasa spełnia warunki \`from\`, wykonaj \`then\`. Jeżeli nie spełnia, sprawdź następny term.

Kiedy trasa pasuje do termu, trzeba jeszcze sprawdzić **jakiego rodzaju akcję** wykonujemy. Na tym etapie Junos różni się od intuicji wyrobionej przez standardową Cisco route-map.

| Działanie | Co robi | Czy kończy ocenę? |
|---|---|---|
| \`then accept\` | Zezwala na import/eksport | **Tak**, nie sprawdza dalszych termów ani polityk |
| \`then reject\` | Odrzuca trasę | **Tak** |
| \`then next term\` | Przechodzi do kolejnego termu tej samej polityki | **Nie** |
| \`then next policy\` | Pomija resztę bieżącej polityki i przechodzi do kolejnej w łańcuchu | **Nie** |
| \`then local-preference 200\` | Zmienia atrybut BGP w ocenianej kopii trasy | **Nie**, sama modyfikacja nie jest akceptacją |
| \`then community add ...\` | Dodaje community do ocenianej trasy BGP | **Nie** |
| brak decyzji terminalnej na końcu wszystkich polityk | Przekazuje decyzję do polityki domyślnej protokołu | Zależy od protokołu i kierunku |

Załóżmy, że polityka \`SET-PREFERENCE\` zawiera jeden term: \`from route-filter 172.16.60.0/24 exact\` oraz \`then local-preference 200\`. Nie ma żadnego innego termu ani żadnej kolejnej polityki w łańcuchu. Co się stanie na imporcie BGP? Pasująca trasa otrzyma LOCAL_PREF 200, a następnie przejdzie do domyślnej polityki importu BGP, która zwykle ją zaakceptuje. **Pozostałe poprawne trasy mogą również zostać zaakceptowane**, choć nie otrzymają nowej wartości LOCAL_PREF.

Jeśli jednak pod pierwszym termem dopiszesz \`term REJECT-REST then reject\`, zmienisz wynik: pasująca trasa z ustawionym LOCAL_PREF, ale **bez \`accept\`**, przejdzie do termu REJECT-REST i zostanie odrzucona. Zmiana atrybutu nie była decyzją „permit”.

### Prześledźmy konkretny łańcuch na R1

Chcemy zastosować dwie polityki importu do peera R2. Pierwsza odrzuca testowy 172.16.62.0/24, druga preferuje 172.16.60.0/24 i akceptuje tylko dwie właściwe sieci R6.

**R1-vSRX — Configuration Mode, obiekty do analizy mechanizmu**

~~~junos
set policy-options policy-statement FILTER-CE-B term BLOCK-TEST from route-filter 172.16.62.0/24 exact
set policy-options policy-statement FILTER-CE-B term BLOCK-TEST then reject
set policy-options policy-statement FILTER-CE-B term OTHERWISE then next policy
set policy-options policy-statement PREFER-R2 term BUSINESS from route-filter 172.16.60.0/24 exact
set policy-options policy-statement PREFER-R2 term BUSINESS then local-preference 200
set policy-options policy-statement PREFER-R2 term BUSINESS then accept
set policy-options policy-statement PREFER-R2 term SERVICES from route-filter 172.16.61.0/24 exact
set policy-options policy-statement PREFER-R2 term SERVICES then local-preference 100
set policy-options policy-statement PREFER-R2 term SERVICES then accept
set policy-options policy-statement PREFER-R2 term REJECT-OTHER then reject
set protocols bgp group CORE neighbor 10.12.0.2 import [ FILTER-CE-B PREFER-R2 ]
~~~

Tu nie trzeba domyślać się żadnej ukrytej logiki. Dla testowej sieci 172.16.62.0/24 pierwsza polityka mówi \`reject\` i kończy pracę. Dla biznesowej 172.16.60.0/24 pierwszy term nie pasuje, \`OTHERWISE\` wykonuje \`next policy\`, a dopiero \`PREFER-R2\` nadaje LOCAL_PREF 200 i akceptuje trasę. Dla 172.16.61.0/24 wynik jest taki sam, ale LOCAL_PREF wynosi 100.

Kolejność ma znaczenie także w samej polityce \`PREFER-R2\`. Gdybyś przeniósł \`REJECT-OTHER\` na początek, ten term — bez warunków \`from\` — **dopasowałby wszystkie trasy**, które do niego dotrą. Kolejne, bardziej szczegółowe termy już nie miałyby czego sprawdzać.

~~~mermaid
flowchart TD
    A["172.16.60.0/24 z R2"] --> B["FILTER-CE-B"]
    B --> C{"Czy /62 exact?"}
    C -->|"nie"| D["OTHERWISE → next policy"]
    C -->|"tak"| E["reject · koniec"]
    D --> F["PREFER-R2"]
    F --> G["BUSINESS /60 exact"]
    G --> H["local-preference 200"]
    H --> I["accept · koniec"]
~~~

*Rysunek 8.3. Trasa jest oceniana przez dwie polityki w podanej kolejności. Odrzucenie kończy całą ocenę; \`next policy\` przekazuje dalszą decyzję do kolejnego policy-statement.*

### Co się stanie, jeśli napiszę „next term” zamiast „next policy”?

\`next term\` nie przenosi trasy do nowej polityki. Przechodzi tylko do następnego termu **w tej samej** policy. W powyższym przykładzie \`FILTER-CE-B\` nie zawiera już innego termu po \`OTHERWISE\`. W produkcyjnej konfiguracji korzystaj z \`next policy\`, jeśli intencją jest wywołanie następnego elementu łańcucha — to jest jednoznaczne dla czytającego i dla review.

Możesz również umieścić całą logikę w jednej polityce, w kolejnych termach \`BLOCK-TEST\`, \`PREFER-BUSINESS\`, \`ALLOW-SERVICES\`, \`REJECT-OTHER\`. Łańcuchowanie daje korzyść głównie wtedy, gdy wspólne zasady, np. odrzucenie określonych prefiksów, mają obowiązywać na kilku różnych peerach.

### Gdy łańcuch się kończy

Jeśli trasa nie pasuje do żadnego termu i nie spotka \`accept\` ani \`reject\`, Junos zastosuje odpowiednią **default policy**. Właśnie dlatego nie można powiedzieć: „wszystko, czego nie wypisałem w whitelist, jest zablokowane”. To prawda dopiero wtedy, gdy polityka **rzeczywiście ma terminalny reject** albo określony protokół w danym kontekście zapewnia taki rezultat.

To również wyjaśnia jedną częstą pomyłkę z Cisco: końcowe \`deny any\` w route-map nie jest dobrym modelem mentalnym dla Junos routing policy. Na Junos końcowe odrzucenie należy napisać świadomie.

*Dokumentacja: [Evaluation Method](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-evaluation-method.html), [Policy Chain](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-chain-evaluation-method.html), [Actions in Routing Policy Terms](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-actions-in-routing-policy-terms.html).*

---

## 8.4. Co dzieje się z otrzymaną trasą — BGP UPDATE, import, RIB, export [A]

Zobaczmy to już nie jako język konfiguracyjny, lecz jako ciąg rzeczywistych zdarzeń w R1. R6 ogłasza 172.16.60.0/24. R4 przesyła tę trasę w górnym i dolnym kierunku. R2 reklamuje ją R1 z AS_PATH \`65002 65004 65106\`, a R3 z AS_PATH \`65003 65004 65106\`.

R1 otrzymuje dwa BGP UPDATE dotyczące **tego samego prefiksu**, lecz opisujące różne ścieżki. Mimo to do zwykłego forwardingu najczęściej wybierze jedną.

### Pięć etapów, które trzeba rozdzielać podczas diagnozy

**Pierwszy: odebranie informacji BGP.** Router ma relację BGP z R2 i R3. W \`show bgp summary\` widzimy status sesji i liczniki. Status \`Established\` dowodzi, że sesja działa. Nie dowodzi, że konkretny prefiks rzeczywiście dotarł, został zaakceptowany ani znalazł się w FIB.

**Drugi: import policy.** R1 bada otrzymaną trasę i stosuje dopasowanie, np. whitelist prefiksów lub ustawienie LOCAL_PREF. Odrzucona informacja nie zostanie wykorzystana jako zwykła aktywna trasa.

**Trzeci: wybór trasy w RIB.** Jeśli oba BGP paths są poprawne, R1 porównuje atrybuty BGP. Po dokonaniu wyboru BGP trzeba również uwzględnić inne protokoły w tej samej tablicy. W Junos podstawowe route preference **OSPF internal 10**, **OSPF external 150** i **BGP 170** powodują, że do tego samego prefiksu aktywna droga OSPF może wygrać z BGP. LOCAL_PREF 200 nie jest tym samym co route preference 200.

**Czwarty: instalacja forwardingu.** Zwycięska droga ma użyteczny next-hop tylko wtedy, gdy router potrafi go rozwiązać. Właśnie dlatego patrzymy osobno na \`show route forwarding-table\`, a nie zakładamy, że gwiazdka w RIB wyjaśnia los pakietu.

**Piąty: export policy.** R1 może przekazać aktywną trasę do kolejnego peera. Jeżeli działa whitelist eksportu, sama obecność trasy w \`inet.0\` niczego nie gwarantuje. Trasa może być dozwolona do użycia przez R1, a jednocześnie **zabroniona do reklamowania** innemu AS.

~~~mermaid
flowchart LR
  UP["R2/R3<br/>BGP UPDATE"] --> IN["Import policy"]
  IN -->|reject| H["Hidden / odrzucona<br/>nieaktywna do forwarding"]
  IN -->|accept| R["R1 inet.0<br/>BGP best path + RIB selection"]
  R --> F["Next-hop resolution<br/>FIB / forwarding"]
  R --> OUT["Export policy<br/>do następnego peera"]
  OUT --> ADV["Reklamowany NLRI"]
~~~

*Rysunek 8.4. Dwie niezależne konsekwencje zaakceptowanej trasy: lokalny forwarding i możliwość dalszej reklamy. Routing policy nie jest filtrem pakietów.*

### Hidden i inactive to dwa różne powody, aby nie używać drogi

Zobaczmy dwa przypadki z naszej topologii.

- R1 akceptuje od R2 i R3 172.16.60.0/24, lecz preferuje ścieżkę przez R2. Droga z R3 może pozostać w RIB jako **inactive**: nie jest wybranym zwycięzcą, mimo że może być prawidłowym kandydatem.
- R1 otrzymuje 172.16.62.0/24, a import policy wykonuje \`reject\`. Taka trasa nie może być używana do zwykłego forwardingu. W zależności od retencji i kontekstu BGP może być widoczna jako **hidden**, z informacją o przyczynie.

Słowo \`hidden\` nie oznacza automatycznie „route została odrzucona przez import policy”. Może też oznaczać np. **unresolved next-hop** albo inne ograniczenie kwalifikacji. W każdym przypadku sprawdź przyczynę, a nie tylko stan.

### Korekta ważnego nieporozumienia: \`keep all\`

Warto tu skorygować popularne uproszczenie. W klasycznym Junos BGP **domyślnie zachowuje większość otrzymanych tras odrzuconych na imporcie** i może prezentować je jako hidden. \`keep all\` rozszerza retencję m.in. na trasy z problematycznym AS_PATH zawierającym lokalny AS. \`keep none\` natomiast umożliwia odrzucanie niektórych niekwalifikowanych tras bez ich przechowywania w pamięci.

Zatem nie jest prawdą, że trzeba zawsze wpisać \`keep all\`, aby zobaczyć każdy prefiks odrzucony przez import policy. W labie **nie zmieniamy domyślnego ustawienia keep bez potrzeby**. Gdybyśmy ćwiczyli retencję, najpierw sprawdzilibyśmy wersję, dostępne RAM i potencjalny restart sesji przy zmianie tych opcji.

**R1-vSRX — Operational Mode, dowody z obu peerów**

~~~junos
show bgp summary
show route receive-protocol bgp 10.12.0.2
show route receive-protocol bgp 10.13.0.2
show route receive-protocol bgp 10.12.0.2 hidden
show route hidden detail
show route 172.16.60.0/24 exact extensive
show route forwarding-table destination 172.16.60.1
~~~

Nie traktuj \`receive-protocol\` jak uniwersalnego, surowego zrzutu wszystkich bajtów Adj-RIB-In niezależnie od \`keep\`, procesu BGP czy wersji. Łącz te widoki z informacjami o policy, licznikami peerów i aktualnym RIB.

### Nadpisywanie polityk w hierarchii BGP

Na R1 grupa CORE ma dwóch peerów: R2 i R3. Chcemy wspólnego filtra bezpieczeństwa oraz oddzielnych reguł LOCAL_PREF. Możesz utworzyć politykę na poziomie \`group\` i nadpisać ją na poziomie \`neighbor\`. **Nie zakładaj jednak, że policy z poziomu neighbor zostanie automatycznie dodana do polityki grupowej.**

W praktyce trzeba świadomie zbudować cały łańcuch obowiązujący dla konkretnego peera. Przykładowo, po ustawieniu na grupie \`import FILTER-CE-B\`, konfiguracja pod sąsiadem \`import PREFER-R2\` może **zastąpić odziedziczony import**, zostawiając tego sąsiada bez filtra. Poprawny zamierzony łańcuch to \`import [ FILTER-CE-B PREFER-R2 ]\`.

To nie kwestia kosmetyki. Taki błąd sprawia, że jedna sesja BGP jest zabezpieczona, a druga — na tym samym routerze — nagle akceptuje prefiks, który powinien być odrzucony.

*Dokumentacja: [BGP keep](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/keep-edit-protocols-bgp.html), [BGP import](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-edit-protocols-bgp.html), [Basic BGP Policies](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html).*

---

## 8.5. Z czego składa się policy-statement i jak ją czytać? [A]

Wiemy już, czym routing policy zajmuje się w RIB oraz jak przebiega jej ocena. Teraz można pokazać składnię bez ryzyka, że zamieni się ona w zbiór przypadkowych \`set\`.

Poniższy przykład dotyczy rzeczywistego celu z topologii: **R1 chce dopuścić tylko 172.16.60.0/24 i 172.16.61.0/24 od R2, a wszystkie inne prefiksy R6 odrzucić**.

**R1-vSRX — Configuration Mode, polityka w formie \`set\`**

~~~junos
set policy-options policy-statement R2-IMPORT term BUSINESS from route-filter 172.16.60.0/24 exact
set policy-options policy-statement R2-IMPORT term BUSINESS then local-preference 200
set policy-options policy-statement R2-IMPORT term BUSINESS then accept
set policy-options policy-statement R2-IMPORT term SERVICES from route-filter 172.16.61.0/24 exact
set policy-options policy-statement R2-IMPORT term SERVICES then local-preference 100
set policy-options policy-statement R2-IMPORT term SERVICES then accept
set policy-options policy-statement R2-IMPORT term DENY-OTHER then reject
~~~

Ta sama polityka w czytelnym formacie hierarchicznym wygląda tak:

~~~text
policy-options {
    policy-statement R2-IMPORT {
        term BUSINESS {
            from route-filter 172.16.60.0/24 exact;
            then {
                local-preference 200;
                accept;
            }
        }
        term SERVICES {
            from route-filter 172.16.61.0/24 exact;
            then {
                local-preference 100;
                accept;
            }
        }
        term DENY-OTHER {
            then reject;
        }
    }
}
~~~

*Listing poglądowy w formacie hierarchicznym; to nie jest oddzielna konfiguracja do dodania drugi raz.*

Warto zobaczyć, jak niewiele tutaj trzeba zapamiętywać. Każdy term jest po prostu warunkiem \`IF\` i akcją \`THEN\`. Nazwy termów \`BUSINESS\`, \`SERVICES\` i \`DENY-OTHER\` odpowiadają temu, o czym rozmawiamy na review.

### Wiele warunków \`from\` — kiedy AND, kiedy OR?

Jeżeli term zawiera **różne kategorie dopasowania**, takie jak \`from protocol bgp\` oraz \`from route-filter 172.16.60.0/24 exact\`, obie muszą być spełnione: logika **AND**. To chroni przed sytuacją, w której podobny prefix pojawi się jako \`Static\` i trafi do polityki przeznaczonej dla BGP.

Jeżeli pod jednym typem kryterium występuje więcej wartości, często traktujemy je jako alternatywy. Na przykład \`from protocol [ direct static ]\` oznacza **Direct LUB Static**. Wiele \`route-filter\` w jednym termie pozwala opisać zbiór prefiksów, ale dokładne dopasowanie odbywa się według ich semantyki i reguł wyboru najbardziej szczegółowego wzorca, nie po prostu „pierwszy z góry”.

Term bez \`from\` oznacza **dopasuj każdą trasę, która do niego dotrze**. Dlatego \`DENY-OTHER then reject\` skutecznie domyka whitelistę. To również powód, dla którego nie umieszczamy go na początku.

### Policy musi być przypięta do miejsca jej użycia

Samo zdefiniowanie \`R2-IMPORT\` nie uruchamia żadnego filtra. Aby dotyczyła peera R2, trzeba ją wskazać w konfiguracji BGP, na właściwym poziomie:

**R1-vSRX — Configuration Mode, przypisanie polityki do sąsiada R2**

~~~junos
set protocols bgp group CORE neighbor 10.12.0.2 import R2-IMPORT
~~~

W dalszym laboratorium użyjemy **łańcucha wspólnego filtra i policy preferencji**, zamiast tej uproszczonej pojedynczej polityki. Traktuj dwa warianty jako **alternatywne projekty**, a nie kolejne komendy do dokładania bez sprawdzenia konfiguracji.

### Dlaczego nazywamy termy?

W Junos można spotkać bardzo krótkie polityki z nienazwanym termem. W dokumentacji na poziomie Enterprise lepiej jednak używać nazw. Kiedy za kilka miesięcy trzeba będzie wstawić dodatkowy wyjątek, \`insert ... before term ...\` pozwoli jednoznacznie wskazać jego miejsce. Łatwiej też zrobić code review, bo komentarz „przesuń BLOCK-TEST nad BUSINESS” ma sens od razu.

**R1-vSRX — Operational Mode, szybkie czytanie polityki i jej użycia**

~~~junos
show configuration policy-options policy-statement R2-IMPORT
show configuration policy-options policy-statement R2-IMPORT | display set
show configuration protocols bgp group CORE | display set
~~~

To ćwiczenie daje dobrą zasadę pracy: **najpierw odczytaj politykę, potem znajdź jej punkt przyłożenia, a na końcu sprawdź wynik na trasach**. Konfiguracja, która nie jest podpięta, nie może być przyczyną filtrowania peera. Konfiguracja podpięta na niewłaściwym poziomie może natomiast działać na inną grupę sąsiadów, niż zakładasz.

*Dokumentacja: [Policy Match Conditions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-match-conditions-in-routing-policy-terms.html), [Routing Policy Structure](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-overview.html).*

---

## 8.6. Route filters, prefix lists, AS_PATH i community — co dokładnie dopasowujemy? [A → S]

Skoro policy wykonuje decyzję na podstawie \`from\`, jej jakość zależy przede wszystkim od **precyzji warunku**. Można mieć bardzo czytelny \`then reject\` i mimo to dopuścić złą trasę, jeżeli użyliśmy zbyt szerokiego dopasowania.

Na R1 chcemy akceptować dwie sieci R6: 172.16.60.0/24 i 172.16.61.0/24. Niedozwolona jest 172.16.62.0/24. Dlaczego więc nie wpisać po prostu \`172.16.0.0/16 orlonger\`? Bo wówczas zaakceptowalibyśmy znacznie więcej niż potrzeba — każdy bardziej szczegółowy prefix należący do tego /16, włącznie z testowym /62.

### Sześć modyfikatorów \`route-filter\`

W Cisco używałeś \`ip prefix-list\` z operatorami \`ge\` oraz \`le\`. Junos ma własny, bardzo czytelny zestaw modyfikatorów. Najlepiej nauczyć się ich na jednym prefiksie bazowym, np. **172.16.0.0/16**.

| Modyfikator | Co dopasuje | Czego nie dopasuje |
|---|---|---|
| \`exact\` | Wyłącznie \`172.16.0.0/16\` | \`172.16.60.0/24\` |
| \`longer\` | Wszystkie bardziej szczegółowe prefiksy zawarte w /16 | Samego \`172.16.0.0/16\` |
| \`orlonger\` | Bazowe /16 i jego bardziej szczegółowe podsieci | Niespokrewnionych zakresów |
| \`upto /24\` | /16, /17 … /24 w zakresie bazowym | /25, /26 itd. |
| \`prefix-length-range /24-/28\` | Tylko /24–/28 należące do bazowego /16 | /16–/23 i /29–/32 |
| \`through\` | Prefiksy na określonej ścieżce od pierwszego prefiksu do wskazanego bardziej szczegółowego | Innych gałęzi drzewa prefiksów |

Nie jest to dopasowanie **adresu źródłowego pakietu**, lecz **prefiksu wpisu routingu**. Jeśli PC albo router wysyła ping do 172.16.60.1, BGP nie ocenia jego source/destination portów ani długości maski komputera. Ocenia ogłaszany route \`172.16.60.0/24\`.

### Szczególny przypadek: Longest Match wewnątrz jednego termu

To jeden z ważniejszych niuansów Junos. Wyobraź sobie pojedynczy term, który mówi:

- akceptuj **172.16.0.0/16 orlonger**;
- ale dla konkretnego **172.16.62.0/24 exact** wykonaj lokalną akcję \`reject\`.

**R1-vSRX — Configuration Mode, osobny wariant ilustrujący akcję przy \`route-filter\`**

~~~junos
set policy-options policy-statement PREFIX-EXAMPLE term TEN from route-filter 172.16.0.0/16 orlonger
set policy-options policy-statement PREFIX-EXAMPLE term TEN from route-filter 172.16.62.0/24 exact reject
set policy-options policy-statement PREFIX-EXAMPLE term TEN then accept
set policy-options policy-statement PREFIX-EXAMPLE term REST then reject
~~~

Co zrobi term dla **172.16.62.0/24**? W obrębie jednego termu Junos wyszukuje **najbardziej szczegółowy wzorzec prefiksu**. Wygra dopasowanie \`172.16.62.0/24 exact\` i jego lokalna akcja \`reject\`, nie wcześniejszy, szeroki \`172.16.0.0/16 orlonger\`. Kolejność dwóch wierszy wewnątrz termu nie jest kolejnością wykonywania jak w zwykłej ACL.

Jest jednak druga, subtelniejsza sytuacja. Załóżmy, że drugi wpis to \`172.16.60.0/24 exact\`, bez lokalnej akcji \`reject\`, a oceniana trasa to **172.16.60.0/25**. W domyślnej semantyce Junos najpierw może zostać wybrany bardziej szczegółowy wzorzec /24, a późniejszy test \`exact\` dla kandydata /25 zakończy się niepowodzeniem. Mimo że szersze /16 \`orlonger\` wygląda na pasujące, silnik nie musi automatycznie do niego wracać. Taką możliwość zmienia funkcja \`walkup\`, której dostępność i zakres należy sprawdzić na swoim release.

Dlatego w konfiguracji przeznaczonej dla zespołu operacyjnego często wolę **osobne, nazwane termy**. Nie dlatego, że Junos działa źle, ale dlatego, że w nocy łatwiej zrozumieć \`BLOCK-TEST\` nad \`ALLOW-BUSINESS\` niż analizować działania lokalne przy najbardziej szczegółowym wzorcu radix tree.

~~~mermaid
flowchart TD
  P["Kandydat: 172.16.62.0/24"] --> W["Wzorce w tym samym termie"]
  W --> A["172.16.0.0/16 orlonger"]
  W --> B["172.16.62.0/24 exact reject"]
  A --> C{"Najbardziej szczegółowy?"}
  B --> C
  C --> D["/24 exact → lokalny reject"]
~~~

*Rysunek 8.5. \`route-filter\` porównuje wzorce prefiksów, a nie przegląda ich jako wierszy ACL. Na egzaminie i w analizie produkcyjnej ma to znaczenie, gdy nakładają się zakresy.*

### \`prefix-list\` — lista nazwanych sieci używana w wielu miejscach

Zamiast wpisywać dwie sieci R6 osobno w każdej polityce, możemy stworzyć jedną listę.

**R1-vSRX — Configuration Mode, lista zaakceptowanych prefiksów CE-B**

~~~junos
set policy-options prefix-list CE-B-ALLOWED 172.16.60.0/24
set policy-options prefix-list CE-B-ALLOWED 172.16.61.0/24
set policy-options policy-statement ALLOW-CE-B term MATCH from prefix-list CE-B-ALLOWED
set policy-options policy-statement ALLOW-CE-B term MATCH then accept
set policy-options policy-statement ALLOW-CE-B term REST then reject
~~~

Zwykłe \`from prefix-list CE-B-ALLOWED\` dopasowuje wpisane prefiksy **dokładnie**. Jeśli potrzebujesz zezwalać na bardziej szczegółowe sieci w ramach tych bloków, użyj odpowiedniej składni \`prefix-list-filter CE-B-ALLOWED orlonger\`. Ale nie traktuj jej jak niegroźnej kosmetyki: przypadkowe \`orlonger\` może dopuścić deagregację /25–/32, której nie planowałeś.

Nazwa \`prefix-list\` jest również użyteczna w bezstanowych firewall filters, np. przy \`source-prefix-list\`. Trzeba jednak pamiętać o różnicy: w routing policy sprawdzasz **wpis routingowy**, a w firewall filter **adres IP pojedynczego pakietu**.

### \`protocol\` — ten sam prefix, inne źródło

Jeżeli R1 ma do 172.16.60.0/24 trasę \`BGP\` oraz z jakiegoś powodu trasę \`Static\`, sama zgodność prefixu nie wystarczy do wybrania zamierzonego wariantu. Możemy dopisać \`from protocol bgp\` albo \`from protocol static\`.

**R1-vSRX — Configuration Mode, oba warunki muszą być spełnione**

~~~junos
set policy-options policy-statement BGP-ONLY term SERVICE from protocol bgp
set policy-options policy-statement BGP-ONLY term SERVICE from route-filter 172.16.60.0/24 exact
set policy-options policy-statement BGP-ONLY term SERVICE then accept
set policy-options policy-statement BGP-ONLY term REST then reject
~~~

To przykład operatora **AND**: trasa musi pochodzić z BGP **i** mieć dokładnie wskazany prefix. Gdy w jednej kategorii wpiszemy kilka wartości \`protocol [ static direct ]\`, akceptujemy jedną **lub** drugą kategorię pochodzenia.

### BGP AS_PATH — skąd naprawdę przyszła trasa?

W naszym diamencie na R1 droga do R6 przez R2 może mieć AS_PATH **65002 65004 65106**, a przez R3 — **65003 65004 65106**. Obie pochodzą ostatecznie od R6, ale mają inny pierwszy AS. Właśnie dlatego atrybut AS_PATH nadaje się do rozpoznawania sąsiadujących domen i do polityk międzyoperatorskich.

W Junos regular expressions dla AS_PATH **nie są wprost składnią Cisco**, w której \`_\` często oznacza granicę numeru AS. Junos interpretuje sekwencje **tokenów AS**. Nie przepisuj wzorca Cisco z podkreślnikami bez zrozumienia różnicy.

**R1-vSRX — Configuration Mode, przykład dopasowania ścieżek przez R2**

~~~junos
set policy-options as-path THROUGH-R2 "65002 .*"
set policy-options policy-statement AS-PATH-EXAMPLE term UPPER from as-path THROUGH-R2
set policy-options policy-statement AS-PATH-EXAMPLE term UPPER then accept
set policy-options policy-statement AS-PATH-EXAMPLE term REST then reject
~~~

To jest przykład do czytania i testowania składni dla konkretnej wersji, **nie** kompletna ochrona antyloop. Bez kontroli prefiksów, sąsiada i pozostałych atrybutów zgoda oparta wyłącznie na AS_PATH byłaby zbyt szeroka. W produkcji używaj polityk adresowych i AS-path jednocześnie, jeśli wymagają tego reguły między AS.

### Communities — etykieta, a nie samoistne zezwolenie

Oprócz AS_PATH do BGP możemy dodawać **communities**, czyli znaczniki opisujące klasę trasy. Załóżmy, że wszystkie prefiksy usług R6 powinny otrzymać \`65106:100\`. R1 może potem dopasować tę community i przypisać LOCAL_PREF. To pomaga oddzielić **identyfikację rodzaju trasy** od **decyzji o jej preferowaniu**.

**R6-vSRX — Configuration Mode, community dla usług CE-B**

~~~junos
set policy-options community CE-B-SERVICES members 65106:100
set policy-options policy-statement TAG-CE-B term SERVICE from route-filter 172.16.61.0/24 exact
set policy-options policy-statement TAG-CE-B term SERVICE then community add CE-B-SERVICES
set policy-options policy-statement TAG-CE-B term SERVICE then next policy
~~~

W łańcuchu \`export [ TAG-CE-B CE-B-OUT ]\` pierwsza polityka **oznacza**, druga wykonuje ostateczną whitelistę. Samo \`community add\` nie powinno być traktowane jak końcowe „zaakceptuj wszystkie trasy”. Użyliśmy jawnego \`next policy\`, bo właśnie taki przebieg zamierzamy.

Rozróżnij trzy działania:

| Akcja | Znaczenie |
|---|---|
| \`community add NAME\` | Dodaj znaczniki, zachowując dotychczasowe |
| \`community set NAME\` | Zastąp zestaw community zdefiniowanym zbiorem |
| \`community delete NAME\` | Usuń wskazane znaczniki |

Jest to niezmiernie ważne przy pracy z operatorem. Przypadkowe \`community set\` może usunąć znaczniki, które inny router wykorzystuje do \`no-export\`, blackholingu lub sterowania trasami. Nie zakładaj też, że sama obecność community „uwierzytelnia” prefix: sąsiad może przesłać zarówno NLRI, jak i community; właściwa import whitelist pozostaje potrzebna.

Dla dużych organizacji istnieją również **Large Communities** z formatem \`large:global-admin:local-data1:local-data2\`, przydatne m.in. przy 4-bajtowych numerach AS. Zasada jest ta sama: znacznik nabiera znaczenia dopiero wtedy, gdy jakaś polityka go interpretuje.

*Dokumentacja: [Routing Policy Match Conditions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-match-conditions-in-routing-policy-terms.html), [Walkup Route Filter](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/walkup-route-filter-policy-option-overview.html), [AS Path Regular Expressions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-as-path-regular-expressions-to-use-as-routing-policy-match-conditions.html).*

---

## 8.7. Routing Policy, Firewall Filter i Security Policy SRX — trzy różne decyzje [A]

Do tej pory pracowaliśmy z prefiksami 172.16.60.0/24 i 172.16.61.0/24. Zajmowaliśmy się tym, **czy R1 ma prawo przyjąć tę informację**, jak ją zmodyfikować i czy przekazać ją dalej. Teraz wysyłamy pierwszy pakiet od R5 do adresu \`172.16.60.1\` na R6.

Jeśli wszystkie trasy są prawidłowe, R5 wyśle pakiet do R1, R1 wybierze ścieżkę przez R2, następnie przejdzie on przez R4 i dotrze do R6. Ale poprawne BGP to dopiero część historii. Na każdym vSRX działa mechanizm ochrony ruchu. Dlatego trzeba odróżnić trzy operacje:

1. **Routing Policy** sprawdza **trasę**. Przykładowo: czy R1 może zaimportować \`172.16.60.0/24\` od R2 i nadać jej LOCAL_PREF 200?
2. **Stateless Firewall Filter** sprawdza **pakiet**, np. source 172.16.50.1, destination 172.16.60.1, ICMP, kierunek input na R1. Nie ma tabeli sesji, która automatycznie przyznałaby zgodę na odpowiedź.
3. **Stateful Security Policy SRX** sprawdza **nową sesję** w określonym kierunku między security zones. Dopasowuje m.in. adresy, aplikację i strefy; dla dopuszczonej sesji utrzymuje stan ruchu powrotnego.

Na klasycznym routerze MX czy przełączniku EX/QFX najczęściej interesują Cię pierwsze dwa mechanizmy. Na SRX musisz uwzględnić również trzeci. To dlatego laboratorium na vSRX wymaga osobnego przygotowania stref oraz polityk bezpieczeństwa.

~~~mermaid
flowchart LR
    UP["BGP UPDATE od R2/R3"] --> RP["Routing policy R1"]
    RP --> RIB["RIB: 172.16.60.0/24"]
    P["Pakiet z R5"] --> FF["Stateless filter<br/>R1 ingress"]
    FF --> SRX["SRX flow / security policy<br/>dla nowej sesji"]
    RIB --> F["Lookup / next-hop"]
    SRX --> F
    F --> WAN["R2 lub R3 → R4 → R6"]
~~~

*Rysunek 8.6. Informacja routingu i ruch użytkownika mają różne ścieżki oceny. Schemat przedstawia zależności diagnostyczne, a nie uniwersalną kolejność sprzętowego processingu na każdym wydaniu SRX.*

### Przykład, w którym routing jest całkowicie poprawny, a aplikacja nadal nie działa

Załóżmy, że na R1:

- \`show route 172.16.60.0/24 exact\` pokazuje aktywną trasę BGP przez 10.12.0.2;
- \`show route forwarding-table destination 172.16.60.1\` wskazuje właściwy next-hop;
- na wejściu LAN-A pracuje filtr, który odrzuca ICMP z R5.

W takim stanie zmiana BGP LOCAL_PREF niczego nie naprawi. Nawet gdy R1 wybierze inny korytarz, pakiet może zostać zatrzymany **wcześniej**, na interfejsie wejściowym.

Rozważmy też przypadek odwrotny. Filtr stateless na R1 mówi \`accept\`, więc pakiet przechodzi przez jego ocenę. Czy to oznacza, że SRX na pewno pozwoli mu przejść przez router? **Nie.** Jeśli stateful policy \`from-zone LAB to-zone LAB\` lub odpowiednia polityka między innymi strefami nie dopuszcza nowej sesji, nadal możemy zobaczyć drop.

W rozdziale 8 nie wyłączamy globalnie security flow, aby „routing zaczął działać”. Pozostawimy vSRX w normalnym trybie stanowym, a do demonstracji skonfigurujemy ściśle laboratoryjną politykę przepuszczającą potrzebne testy ICMP.

### Ruch tranzytowy i ruch do własnego routera to nie to samo

Gdy R5 pinguje **192.0.2.1**, pakiet jest skierowany do adresu R1. To ruch **host-bound**. Na SRX trzeba uwzględnić \`host-inbound-traffic\` i ewentualny filtr ochronny RE. Natomiast ping R5 → **172.16.60.1** ma charakter **tranzytowy przez R1**, choć ostatecznie kończy się na adresie własnym R6.

To oznacza, że możesz uzyskać następujący zestaw wyników:

- R5 pinguje bramę R1: PASS;
- R1 pinguje R2 po transit: PASS;
- R5 nie pinguje 172.16.60.1 na R6: FAIL.

Takie wyniki **nie dowodzą**, że BGP jest zepsuty. Dwa pierwsze testy nie są testem całej ścieżki tranzytowej i obu security policies. Właściwa diagnostyka obejmuje RIB, next-hop, stateless filter, security session, ruch powrotny i lokalne host-inbound na R6.

*Dokumentacja: [Comparison of Routing Policies and Firewall Filters](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-firewall-filters-comparison.html), [SRX Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html), [Security Zones](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/security-zone-configuration.html).*

---

## 8.8. Jak działa stateless firewall filter — i dlaczego \`count\` może przypadkowo przepuścić ruch [A]

Stateless firewall filter ma znajomą strukturę: \`term\`, \`from\`, \`then\`. Zewnętrznie przypomina routing policy. To podobieństwo jest przydatne przy czytaniu konfiguracji, ale **nie wolno przenosić na niego reguł zakończenia oceny z routing policy**.

Najważniejsza różnica brzmi:

**Jeżeli pakiet przejdzie przez wszystkie termy standardowego firewall filter i żaden go nie dopasuje, zostanie odrzucony przez implicit discard.**

To jest zachowanie bardzo podobne do końca Cisco ACL. Oznacza jednak, że filtr składający się tylko z jednej reguły \`BLOCK-X then discard\` nie „blokuje wyłącznie X”. Dla wszystkich pozostałych pakietów również zabraknie zezwolenia.

### Filtr pozwalający wszystkim oprócz jednego źródła

W naszym diamencie R5 ma prawidłowe adresy źródłowe 172.16.50.1 i 172.16.51.1. W oddzielnym teście dodamy mu tymczasowy adres \`203.0.113.9/32\`, którego R1 nie powinien traktować jako normalnego źródła CE-A.

Najprostszy filtr mógłby zablokować ten konkretny adres, a całą resztę przepuścić do kolejnych etapów.

**R1-vSRX — Configuration Mode, filtr na ruch wchodzący od R5**

~~~junos
set firewall family inet filter CE-A-IN term BLOCK-TEST-SOURCE from source-address 203.0.113.9/32
set firewall family inet filter CE-A-IN term BLOCK-TEST-SOURCE then count BLOCKED-TEST
set firewall family inet filter CE-A-IN term BLOCK-TEST-SOURCE then discard
set firewall family inet filter CE-A-IN term PERMIT-REMAINDER then count OTHER-TRAFFIC
set firewall family inet filter CE-A-IN term PERMIT-REMAINDER then accept
set interfaces ge-0/0/2 unit 0 family inet filter input CE-A-IN
~~~

Czytamy go tak: pakiet przychodzi od R5 do R1 na interfejs \`ge-0/0/2.0\`. Jeśli ma testowy source, zwiększamy licznik i odrzucamy. Każdy inny pakiet pasuje do termu końcowego i przechodzi dalej — do zwykłego routingu i ewentualnie do security flow.

To jest przykład **stateless filter**, nie stateful security policy. Nie założyliśmy nim żadnej sesji TCP ani nie przyznaliśmy automatycznej zgody na return traffic.

### Dlaczego samo \`then count\` nie oznacza „sprawdź następny term”?

Teraz ważniejszy przypadek. Załóżmy, że ktoś chce zliczać wszystkie pakiety ICMP na wejściu R1, ale niżej zostawić szczegółowe reguły blokujące. Pisze:

**R1-vSRX — Configuration Mode, BŁĘDNY przykład do analizy, nie przypinać do interfejsu**

~~~junos
set firewall family inet filter BAD-COUNT term COUNT-ICMP from protocol icmp
set firewall family inet filter BAD-COUNT term COUNT-ICMP then count ICMP-TOTAL
set firewall family inet filter BAD-COUNT term BLOCK-TEST from source-address 203.0.113.9/32
set firewall family inet filter BAD-COUNT term BLOCK-TEST then discard
~~~

Administrator myśli: „Pierwszy term tylko policzy, a potem drugi zablokuje nieprawidłowe źródło”. W standardowym Junos firewall filter jest inaczej: **pasujący term z akcją nieterminującą, taką jak \`count\`, ale bez jawnej akcji sterującej, dostaje domyślną akcję terminalną \`accept\`**. Pakiet ICMP ze źródła 203.0.113.9 pasuje już do pierwszego termu, zostaje zliczony i zaakceptowany. Drugi term nie zobaczy go wcale.

Jeśli w danym typie standardowego filtra musisz **policzyć i kontynuować**, odpowiednią konstrukcją jest \`then next term\`. Trzeba jednak sprawdzić, czy konkretny ASIC/model/rodzina i wydanie Junos wspierają wymaganą kombinację akcji.

**R1-vSRX — Configuration Mode, wersja poprawiona do analizy**

~~~junos
set firewall family inet filter COUNT-THEN-CHECK term COUNT-ICMP from protocol icmp
set firewall family inet filter COUNT-THEN-CHECK term COUNT-ICMP then count ICMP-TOTAL
set firewall family inet filter COUNT-THEN-CHECK term COUNT-ICMP then next term
set firewall family inet filter COUNT-THEN-CHECK term BLOCK-TEST from source-address 203.0.113.9/32
set firewall family inet filter COUNT-THEN-CHECK term BLOCK-TEST then discard
set firewall family inet filter COUNT-THEN-CHECK term PERMIT-REST then accept
~~~

Zauważ różnicę wobec routing policy. W routing policy \`then local-preference 200\` bez \`accept\` zwykle nie kończy oceny. W standardowym firewall filter \`then count\` bez \`next term\` może zakończyć ocenę **akceptacją pakietu**.

To jeden z najważniejszych mechanizmów całego rozdziału, bo składnia wygląda podobnie, lecz końcowe zachowanie jest odmienne.

### Jakie kryteria i akcje warto rozpoznawać od razu?

| Obszar | Przykłady | Do czego używamy |
|---|---|---|
| Adresacja | \`source-address\`, \`destination-address\`, \`source-prefix-list\` | Filtrowanie ruchu z określonych sieci |
| Protokół | \`protocol tcp\`, \`udp\`, \`icmp\`, \`ospf\` | Dopasowanie IP protocol / L4 |
| Port | \`source-port\`, \`destination-port\`, \`port\` | SSH, BGP, DNS, HTTPS itd. |
| Pola dodatkowe | \`tcp-established\`, \`icmp-type\`, \`dscp\` | Szczegółowa kontrola i klasyfikacja |
| Akcje terminalne | \`accept\`, \`discard\`, \`reject\` | Zgoda lub odrzucenie |
| Monitorowanie | \`count\`, \`log\`, \`syslog\` | Liczniki i diagnostyka |
| Przepływ | \`next term\`, \`routing-instance\` | Kontynuacja oceny lub FBF |
| Policing / CoS | \`policer\`, \`forwarding-class\` | Rate limiting, klasyfikacja |

\`port bgp\` w stateless filter dopasowuje port po odpowiedniej stronie pakietu (source lub destination), co jest przydatne dla sesji TCP/179; \`destination-port bgp\` nie jest równoważne dla obu kierunków komunikacji. Podobnie \`tcp-established\` nie jest pamięcią sesji — zwykle opiera się na flagach TCP, a nie na pełnym stanie trójstronnego handshake.

Na platformach EX/QFX/MX różnią się możliwości ASIC i dostępne zasoby do instalacji filtrów, a na vSRX dochodzi ścieżka przetwarzania sesji. Nie wszystkie akcje można dowolnie połączyć we wszystkich rodzinach interfejsów; nie zakładaj, że przykład znaleziony dla MX będzie bez zmian działał na każdym EX czy vSRX.

~~~mermaid
flowchart TD
  IN["Pakiet od R5 wchodzi na R1"] --> T1{"source 203.0.113.9?"}
  T1 -->|"tak"| D["count BLOCKED-TEST<br/>discard"]
  T1 -->|"nie"| T2["PERMIT-REMAINDER<br/>count OTHER<br/>accept"]
  T2 --> OK["Dalszy routing / security"]
~~~

*Rysunek 8.7. Odrzucenie przez stateless filter nie usuwa tras BGP z RIB. Po odrzuceniu pakietu żadna zmiana BGP export nie będzie jego naprawą.*

*Dokumentacja: [Stateless Firewall Filter Components](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-components.html), [Guidelines for Configuring Firewall Filters](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-stateless-guidelines-for-configuring.html).*

---

## 8.9. Filtr na \`lo0.0\` — ochrona Routing Engine przed skanowaniem i przeciążeniem [A]

Na Cisco znasz ideę CoPP: chronić CPU/control plane przed ruchem, który nie musi być przekazywany do procesów routera. Junos oferuje ochronę ruchu przeznaczonego dla Routing Engine między innymi przez **stateless firewall filter na \`lo0.0\`**.

W naszym diamencie R1 utrzymuje eBGP do trzech sąsiadów: **R5 192.0.2.10, R2 10.12.0.2, R3 10.13.0.2**. R1 może również potrzebować SSH z sieci OOB, synchronizacji NTP i odpowiedzi DNS/SNMP. Jeśli dołożymy filtr na \`lo0.0\` zezwalający tylko na ICMP, ale zapomnimy o BGP, możemy utracić wszystkie trzy sesje po czasie potrzebnym na wygaśnięcie timera.

Dlatego ochrony RE nie projektujemy przez skopiowanie kilku reguł z internetu. Najpierw spisujemy **kto ma prawo łączyć się z routerem**, jakimi protokołami i skąd. To są wymagania dla filtra, a nie odwrotnie.

### \`apply-path\` — prefix-list tworzona z rzeczywistych sąsiadów BGP

W klasycznej konfiguracji miałbyś w filtrze wpisy dla R5, R2 i R3. Problem pojawia się za kilka miesięcy, gdy dodajesz kolejnego peera BGP, ale zapominasz zmienić ochronę RE. Junos może wygenerować listę adresów peerów bezpośrednio z konfiguracji BGP przez \`apply-path\`.

**R1-vSRX — Configuration Mode, dynamiczna lista sąsiadów BGP**

~~~junos
set policy-options prefix-list BGP-PEERS apply-path "protocols bgp group <*> neighbor <*>"
~~~

Na R1 rozwinięcie tej listy powinno zawierać sąsiadów ze wszystkich właściwych grup BGP, m.in. 192.0.2.10, 10.12.0.2 i 10.13.0.2. To nie jest lista stworzona przez scanning sieci: \`apply-path\` czyta **aktualną konfigurację**, a nie odkrywa aktywnych hostów.

**R1-vSRX — Operational Mode, sprawdzenie rozwinięcia \`apply-path\`**

~~~junos
show configuration policy-options prefix-list BGP-PEERS | display inheritance
show configuration protocols bgp | display set
~~~

Jeżeli nowy peer nie pojawia się w rozwinięciu, sprawdź najpierw konfigurację BGP i ścieżkę wzorca. Nie zakładaj, że wszystkie adresy BGP w całym urządzeniu leżą pod tym samym \`protocols bgp\`: routing instances i logical systems wymagają uwzględnienia odpowiedniej hierarchii.

### Prosty filtr ochronny — konstrukcja do review, nie gotowy baseline produkcyjny

Pokażmy samą ideę. Mamy zezwolić na BGP wyłącznie od sąsiadów, na ICMP do sprawdzenia łączności, a wszystko inne odrzucać. **Nie przypinamy tego wzorca automatycznie na działającym R1**, bo nie znamy jeszcze adresacji OOB/SSH, NTP, DNS, monitoringu, a także wszystkich procesów host-bound i service permissions SRX.

**R1-vSRX — Configuration Mode, SZKIC zasad do weryfikacji przed przyłożeniem**

~~~junos
set firewall family inet filter PROTECT-RE term ALLOW-BGP from source-prefix-list BGP-PEERS
set firewall family inet filter PROTECT-RE term ALLOW-BGP from protocol tcp
set firewall family inet filter PROTECT-RE term ALLOW-BGP from port bgp
set firewall family inet filter PROTECT-RE term ALLOW-BGP then count RE-BGP-OK
set firewall family inet filter PROTECT-RE term ALLOW-BGP then accept
set firewall family inet filter PROTECT-RE term ALLOW-ICMP from protocol icmp
set firewall family inet filter PROTECT-RE term ALLOW-ICMP then count RE-ICMP-OK
set firewall family inet filter PROTECT-RE term ALLOW-ICMP then accept
set firewall family inet filter PROTECT-RE term DENY-OTHER then count RE-DROPPED
set firewall family inet filter PROTECT-RE term DENY-OTHER then discard
~~~

Nie ma tu reguły \`ALLOW-SSH\` ani odpowiedzi na pytania dotyczące NTP, DNS i pozostałych usług. Dlatego przykład jest **celowo niekompletny do wdrożenia na porcie zarządzanym zdalnie**. W głównym labie będziemy go analizować i ewentualnie testować tylko po zidentyfikowaniu wszystkich niezbędnych wyjątków, z konsolą/OOB i \`commit confirmed\`.

Na vSRX obowiązuje dodatkowo \`security zones ... host-inbound-traffic\` — pozwolenie w filtrze \`lo0\` **nie zastępuje** odpowiednich uprawnień host-bound w security zones. Sam fakt, że firewall filter ma słowo \`accept\`, nie oznacza, że proces SSH lub BGP automatycznie otrzyma pakiet w każdej konfiguracji SRX.

### Czego nie pomijać na liście dozwolonych usług?

Przed review filtra trzeba sprawdzić działające usługi routera. Jeżeli R1 ma SSH z OOB, BGP do R5/R2/R3, OSPF na innych segmentach, NTP, SNMP, DNS, TACACS+/RADIUS albo BFD, każda funkcja może wymagać odpowiednich reguł zależnych od kierunku. Nie da się stworzyć „uniwersalnego filtra produkcyjnego” bez takiej inwentaryzacji.

Policing ICMP może być bardzo przydatny, ale w sieci warto zostawić niezbędne komunikaty błędów, w tym istotne dla MTU/path diagnostics. Zbyt szerokie \`discard\` na RE może powodować awarie, których objawy wyglądają jak problemy routingu.

**R1-vSRX — Operational Mode, monitoring ochrony RE po poprawnym, bezpiecznym wdrożeniu**

~~~junos
show firewall filter PROTECT-RE
show firewall
show bgp summary
show security zones
show log messages | last 30
~~~

Jeżeli stosujesz \`then log\` albo \`syslog\`, również monitoruj wpływ ilości logów na urządzenie. \`log\` i \`syslog\` nie muszą mieć identycznej implementacji na wszystkich platformach, a niekontrolowane logowanie intensywnego ruchu host-bound może samo stać się problemem operacyjnym.

*Dokumentacja: [Applying Firewall Filters](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-stateless-guidelines-for-applying.html), [Firewall Filter Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-overview.html), [SRX Security Zones](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/security-zone-configuration.html).*

---

## 8.10. uRPF — router sprawdza, czy źródło pakietu ma sens [A]

Routing zwykle kojarzy się z destination IP. Router sprawdza, dokąd wysłać pakiet. W sieciach wielooperatorskich dochodzi jednak inne pytanie: **czy źródło pakietu wygląda wiarygodnie z perspektywy interfejsu, na którym go otrzymaliśmy?**

Do takiej oceny służy **Unicast Reverse Path Forwarding (uRPF)**. Router korzysta z informacji o drodze **do źródła**, a nie do celu pakietu.

W naszej topologii R1 przyjmuje pakiet od R5 na LAN-A. Normalny ruch R5 może mieć source \`172.16.50.1\`. Jeśli R1 zna trasę do \`172.16.50.0/24\` przez tego właśnie R5, source wygląda spójnie z topologią. Ale jeżeli R5 wygeneruje izolowany testowy pakiet ze źródłem \`203.0.113.9\`, którego R1 w ogóle nie zna lub zna przez inną stronę sieci, może zostać odrzucony przez uRPF.

### Strict: czy pakiet przyszedł właściwym wejściem?

W **strict uRPF** test polega na sprawdzeniu, czy kwalifikująca się droga powrotna do source IP wskazuje ten sam interfejs, na którym pakiet przyszedł (z uwzględnieniem szczegółów ECMP i wariantu platformowego). W nieskomplikowanej sieci klienckiej typu single-homed jest to silny mechanizm antyspoofingowy.

**R1-vSRX — Configuration Mode, wariant strict na ingress od R5 — tylko przy potwierdzonym wsparciu**

~~~junos
set interfaces ge-0/0/2 unit 0 family inet rpf-check
~~~

W naszym diamencie działałoby to sensownie dla prefiksów CE-A, których droga powrotna prowadzi przez R5. Nie znaczy to jednak, że strict można bez analizy włączyć na wszystkich sześciu interfejsach.

### Kiedy strict może zatrzymać legalny ruch?

Załóżmy, że R1 preferuje drogę do **172.16.61.0/24** przez dolny R3. R1 otrzymuje jednak legalny pakiet ze źródłem **172.16.61.1** na górnym interfejsie od R2. Jeżeli strict uRPF ocenia reverse path tylko przez najlepszą aktywną drogę, to do source 172.16.61.1 wskazuje **R3**, nie R2. Pakiet może zostać odrzucony pomimo tego, że pochodzi z rzeczywistej sieci R6.

To jest typowe zjawisko w topologiach z routingiem asymetrycznym. Z punktu widzenia biznesu droga pakietu jest poprawna. Z punktu widzenia strict uRPF nie odpowiada jednak najlepszej drodze powrotnej.

### Loose: czy w ogóle istnieje droga do źródła?

W **loose uRPF** nie wymagamy zgodności interfejsu wejściowego z reverse path. Wystarczy, że istnieje odpowiednia dopuszczalna droga do źródła według zasad danej implementacji. To może pomagać na łączach wielooperatorskich, ale nie jest równoważne ścisłej ochronie na pojedynczym interfejsie klienta.

**R1-vSRX — Configuration Mode, alternatywa loose — nie wprowadzać razem ze strict jako dwóch niezależnych ustawień**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet rpf-check mode loose
~~~

Jest jeszcze istotny niuans: jeśli router ma aktywną default route \`0.0.0.0/0\`, różne modele i wydania mogą inaczej traktować jej przydatność dla loose uRPF. Nie wyciągaj z samej obecności default wniosku, że mechanizm automatycznie przyjmie wszystkie możliwe źródła. Sprawdź zachowanie sprzętu i konkretną dokumentację.

### Feasible paths: kiedy sama aktywna droga nie wystarcza

Na poziomie Specialist/Professional spotkasz również **feasible-paths**. To wariant, w którym kontrola reverse path może uwzględnić inne wykonalne ścieżki niż pojedyncza aktywna droga z FIB. Jest przydatny w pewnych asymetrycznych sieciach, lecz nie jest uniwersalnym lekarstwem na wszystkie problemy strict uRPF.

**R1-vSRX — Configuration Mode, schemat funkcji do weryfikacji modelu i wersji**

~~~junos
set routing-options forwarding-table unicast-reverse-path feasible-paths
~~~

Zanim włączysz tę opcję, sprawdź zgodność z routing instances, FBF i ECMP. Szczególnie na vSRX nie można bez testu zakładać, że zachowanie będzie identyczne z fizycznym MX.

### Fail-filter i wyjątki, np. DHCP

Stateless \`fail-filter\` może w niektórych implementacjach pomóc obsłużyć pakiety, które nie zdały kontroli uRPF, ale z konkretnego powodu trzeba je przepuścić. Klasyczny przykład to DHCPDISCOVER ze source \`0.0.0.0\`: klient nie ma jeszcze poprawnego adresu ani normalnej drogi powrotnej.

Nie będziemy automatycznie łączyć pełnego filtra DHCP z FBF i BGP na wszystkich R1–R6. To osobne ćwiczenie. Przy produkcyjnym projekcie należy zweryfikować obsługę \`fail-filter\` i dokładną składnię na swojej platformie, a także to, czy DHCP relay/host-bound traffic jest faktycznie obecny w danym punkcie sieci.

~~~mermaid
flowchart TD
  IN["Pakiet na ingress, source = X"] --> M{"Tryb uRPF?"}
  M -->|"strict"| S{"Reverse path do X<br/>akceptuje ingress?"}
  M -->|"loose"| L{"Istnieje dopuszczalna<br/>droga do X?"}
  S -->|"tak"| OK["Dalszy processing"]
  L -->|"tak"| OK
  S -->|"nie"| F["uRPF failure / opcjonalny fail-filter"]
  L -->|"nie"| F
~~~

*Rysunek 8.8. Strict porównuje drogę powrotną z interfejsem, loose koncentruje się na osiągalności źródła. Schemat nie modeluje każdej implementacji ECMP, FBF i PFE.*

**R1-vSRX — Operational Mode, co sprawdzić, gdy uRPF odrzuca ruch z 172.16.61.1**

~~~junos
show configuration interfaces ge-0/0/0 | display set
show route 172.16.61.1
show route forwarding-table destination 172.16.61.1
show interfaces ge-0/0/0 extensive
show configuration routing-options forwarding-table | display set
~~~

Pamiętaj, że tutaj pytasz o drogę **do źródła 172.16.61.1**, nie o drogę do destination IP konkretnej aplikacji. To odróżnia uRPF od zwykłego routingu i od BGP export policy.

*Dokumentacja: [Understanding Unicast RPF](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/interfaces-configuring-unicast-rpf.html), [rpf-check CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rpf-check-edit-interfaces.html).*

---

## 8.11. Konfiguracja i troubleshooting — jak udowodnić, że policy zrobiła to, czego oczekiwaliśmy? [A]

W pracy operacyjnej sama znajomość składni nie wystarczy. Najważniejsze jest umieć **umiejscowić decyzję** w odpowiednim etapie i zebrać dowody. W naszym diamencie prefiks \`172.16.60.0/24\` ma dotrzeć od R6 do R5. Jeśli R5 go nie widzi, nie można od razu założyć błędu export na R6. R4 mógł go otrzymać, lecz nie reklamować; R1 mógł odrzucić import; ścieżka R2 lub R3 mogła być ukryta; R5 mógł zainstalować nieodpowiednią trasę z innego źródła.

Przejdźmy więc przez prosty workflow.

### Najpierw ustal, kto naprawdę nie ma prefiksu

Jeśli R5 nie ma 172.16.60.0/24, sprawdzamy od początku ścieżki reklamy:

**R6-vSRX — Operational Mode, początek łańcucha**

~~~junos
show route 172.16.60.0/24 exact extensive
show bgp summary
show route advertising-protocol bgp 198.51.100.1
show configuration protocols bgp | display set
~~~

Jeżeli R6 nie ma własnej aktywnej route, nie ma sensu poprawiać R1 import. Jeśli R6 ma trasę, ale nie reklamuje jej R4, analizujemy **R6 export** i dopasowanie \`from protocol direct\` lub \`route-filter\`.

**R4-vSRX — Operational Mode, czy R4 rzeczywiście otrzymał i wybrał drogę od CE-B?**

~~~junos
show route receive-protocol bgp 198.51.100.10
show route 172.16.60.0/24 exact extensive
show route advertising-protocol bgp 10.24.0.1
show route advertising-protocol bgp 10.34.0.1
~~~

Jeśli R4 otrzymał route, ale nie reklamuje do R2/R3, trzeba sprawdzić jego **CORE export**. Jeśli wysłał ją do obu, przechodzimy na R1.

**R1-vSRX — Operational Mode, podwójny odbiór i wybór drogi**

~~~junos
show bgp summary
show route receive-protocol bgp 10.12.0.2
show route receive-protocol bgp 10.13.0.2
show route 172.16.60.0/24 exact extensive
show route 172.16.61.0/24 exact extensive
show route hidden
show route advertising-protocol bgp 192.0.2.10
~~~

Ostatnia komenda pokazuje perspektywę R5 — czy R1 dalej reklamuje zaakceptowane prefiksy. Jeśli na R1 jest aktywna \`172.16.60.0/24\`, ale reklama do R5 jest pusta, zajmujemy się eksportem z R1 do CE-A. Jeśli R1 nie ma trasy, zajmujemy się jego importem albo problemem next-hop.

Ważne: **\`show route receive-protocol\` i \`show route advertising-protocol\` nie są tym samym widokiem**. Pierwszy pomaga badać otrzymaną informację, drugi rzeczywisty kierunek reklamy po export policy. Wyniki tych poleceń należy interpretować w kontekście opcji BGP \`keep\`, konkretnego peer’a i wydania systemu.

### \`test policy\` — sprawdź logikę na trasach już obecnych w RIB

Junos udostępnia polecenie \`test policy\`. Jest ono bardzo użyteczne przy sprawdzaniu, czy zbudowana policy dopasowuje prefixy zgodnie z założeniami. Nie jest jednak pełnym symulatorem BGP UPDATE pomiędzy dwoma routerami.

**R1-vSRX — Operational Mode, test na prefiksach obecnych w lokalnej tablicy**

~~~junos
test policy ALLOW-CE-B 172.16.60.0/24
test policy ALLOW-CE-B 172.16.61.0/24
test policy ALLOW-CE-B 172.16.62.0/24
~~~

Polecenie wykorzystuje trasy obecne w danym kontekście routingu, nie tworzy prefiksu z powietrza. Dodatkowo posiada własne zasady dotyczące domyślnego wyniku i nie odtwarza wszystkich kontekstowych ograniczeń BGP. Jeśli prefix jest odrzucony na imporcie i nie występuje jako odpowiednia trasa testowa, \`test policy\` nie może magicznie wykazać zachowania rzeczywiście odebranego UPDATE.

Dlatego w naszym laboratorium zrobimy dwa rodzaje testów. Najpierw użyjemy \`test policy\` do weryfikacji **logiki dopasowania**. Następnie rzeczywistego \`advertising-protocol\` i \`receive-protocol\` do potwierdzenia działania BGP na poszczególnych łączach.

Istotna rzecz: w teście polityki bez jawnego końcowego \`reject\` można uzyskać pozornie korzystny wynik, ponieważ test posługuje się własnym domyślnym \`accept\`. Właśnie dlatego politykę bezpieczeństwa oceniamy zarówno na dozwolonych, jak i zabronionych prefiksach.

### Jak testować filtr pakietów bez zgadywania?

Dla firewall filter bardziej użyteczne są liczniki i pomiar **różnicy** przed oraz po znanym strumieniu testowym. Jeśli licznik \`BLOCKED-TEST\` wynosi 10 000, nie wiesz, czy wzrósł właśnie od Twojej próby, czy od godzin pracy routera. Zmierz wartość przed testem i po nim.

**R1-vSRX — Operational Mode, przypięcie filtra i liczniki**

~~~junos
show configuration interfaces ge-0/0/2 | display set
show configuration firewall family inet filter CE-A-IN
show firewall filter CE-A-IN
show interfaces ge-0/0/2 extensive
~~~

Jeżeli licznik rośnie, ale nie masz pewności, czy dotyczy właściwego pakietu, w odizolowanym labie wykonaj krótką kontrolowaną próbę z jednoznacznym source/destination. Unikaj globalnego czyszczenia liczników i sesji na urządzeniu produkcyjnym tylko po to, aby uzyskać wygodny output.

Na vSRX po dopuszczeniu pakietu przez stateless filter nadal sprawdzamy stateful security.

**R1-vSRX — Operational Mode, przekazywanie przez strefy i nowa sesja**

~~~junos
show security zones
show security policies
show security flow session
show route 172.16.60.0/24 exact
~~~

Jeśli \`security flow session\` nie pokazuje spodziewanego flow, sprawdzamy kolejność testowania, źródło pakietu, policy hit count (jeśli obsługiwane), a także to, czy nie obserwujemy jedynie pingu do własnego adresu routera. Na SRX „mam poprawną trasę” i „mogę utworzyć sesję” to odrębne twierdzenia.

### Zmiana produkcyjna: pre-check, przewidywanie, commit, post-check

Załóżmy, że klient prosi o zablokowanie jednego prefiksu od CE-B. Dobrze przygotowana zmiana powinna mieć dokładnie opisany rezultat: **po zmianie 172.16.62.0/24 ma nie być używany przez R1, ale 172.16.60.0/24 i 172.16.61.0/24 mają pozostać dostępne**.

Nie zaczynaj od \`set\`. Najpierw zachowaj pre-check: wersję, istniejący łańcuch BGP import/export, aktywne trasy, sessions i strefy. Następnie zapisz wynik, którego oczekujesz na R1 i R5. Dopiero wtedy wprowadź minimalną zmianę.

**R1-vSRX — Operational Mode, wejście do konfiguracji**

~~~junos
configure private
~~~

**R1-vSRX — Configuration Mode, po przygotowaniu minimalnej zmiany**

~~~junos
show | compare
commit check
commit confirmed 5
~~~

Po \`commit confirmed\` wykonujemy **operacyjne post-checki**, nie kolejne \`commit check\`. W Junos dodatkowy \`commit check\` podczas oczekującego confirmed commit może potwierdzić zmianę; nie uruchamiaj go mechanicznie. Jeśli możesz odciąć własny SSH, użyj niezależnej konsoli/OOB. Nie polegaj na „rollback przez to samo SSH”, które sam właśnie filtrujesz.

**R1-vSRX — Operational Mode, tylko testy po aktywacji**

~~~junos
show bgp summary
show route 172.16.60.0/24 exact extensive
show route 172.16.61.0/24 exact extensive
show route 172.16.62.0/24 exact extensive
show route hidden
show route advertising-protocol bgp 192.0.2.10
~~~

Jeśli kryteria są spełnione, potwierdzamy zmianę. Jeśli nie, pozwalamy automatycznemu rollbackowi zadziałać albo cofamy zgodnie z przygotowanym planem.

**R1-vSRX — Configuration Mode, wyłącznie po pozytywnym post-check**

~~~junos
commit
~~~

Wynikiem profesjonalnej zmiany jest **dowód pozytywny i negatywny**: dozwolone prefiksy nadal działają, a zabroniony został skutecznie odrzucony. Komentarz „BGP checked, all OK” nie jest równoważnym dowodem.

*Dokumentacja: [Testing Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-testing-routing-policies.html), [test policy CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/test-policy.html), [Junos CLI Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-cli-configuration.html).*

---

## 8.12. Te same polityki w OSPF, IS-IS i BGP — dlaczego protokół ma znaczenie? [S]

Wszystkie poznane mechanizmy \`policy-options\` stosują podobny język: term, from, then. Ale nie oznacza to, że każdy protokół pozwala w identyczny sposób zmieniać każdy rodzaj trasy.

Żeby to zrozumieć, weźmy **tę samą fizyczną topologię DIAMOND-6R** i wyobraźmy sobie trzy warianty konfiguracji control plane. W pierwszym rdzeń R1–R4 działa na OSPF, w drugim na IS-IS, a w trzecim — tak jak w podstawowym laboratorium rozdziału — wszystkie sześć połączeń jest eBGP.

Nie uruchamiamy tych wariantów **jednocześnie** bez wyraźnego planu redystrybucji. Chodzi o pokazanie, jak zmieni się znaczenie \`import\` i \`export\` dla poszczególnych protokołów.

### 8.12.1. OSPF: własne LSA są rozpowszechniane niezależnie od export policy

Załóżmy, że R1, R2, R3 i R4 tworzą wspólny OSPF Area 0. R1 ma interfejs \`ge-0/0/0.0\` do R2 w sieci 10.12.0.0/30. Kiedy włączysz OSPF na tym interfejsie, R1 i R2 wymienią informacje link-state i poznają swoją topologię.

**R1-vSRX — Configuration Mode, oddzielny wariant OSPF rdzenia**

~~~junos
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0
~~~

To nie jest kompletna konfiguracja OSPF całego diamantu. R2, R3 i R4 potrzebują odpowiednich interfejsów i wspólnego area, a vSRX wymaga dopuszczenia protokołu OSPF jako host-inbound na danych strefach. W tym wariancie **nie** pozostawiamy równolegle nieprzemyślanego import/export BGP z bazowego labu.

Skąd router dowiaduje się o 10.12.0.0/30? Z mechanizmu OSPF i jego LSA, nie z \`protocols ospf export\` robiącego redystrybucję \`Direct\`.

Teraz inny przypadek: R1 ma **statyczną trasę** do testowej sieci 172.31.10.0/24. Ta trasa nie jest wewnętrznym linkiem OSPF. Jeśli chcemy ją rozpowszechnić jako OSPF external, właśnie wtedy użyjemy export policy.

**R1-vSRX — Configuration Mode, osobny wariant redystrybucji statycznej trasy testowej**

~~~junos
set routing-options static route 172.31.10.0/24 discard
set policy-options policy-statement OSPF-EXPORT-TEST term ONLY-TEST from protocol static
set policy-options policy-statement OSPF-EXPORT-TEST term ONLY-TEST from route-filter 172.31.10.0/24 exact
set policy-options policy-statement OSPF-EXPORT-TEST term ONLY-TEST then metric 20
set policy-options policy-statement OSPF-EXPORT-TEST term ONLY-TEST then external type 1
set policy-options policy-statement OSPF-EXPORT-TEST term ONLY-TEST then tag 65106
set policy-options policy-statement OSPF-EXPORT-TEST term ONLY-TEST then accept
set policy-options policy-statement OSPF-EXPORT-TEST term REST then reject
set protocols ospf export OSPF-EXPORT-TEST
~~~

Dlaczego pokazujemy \`discard\`? Tylko dlatego, że jest to bezpieczny **sztuczny prefix do oceny redystrybucji**, a nie działająca aplikacja. Router posiada aktywny route do tego prefiksu, ale pakiety do nieobjętych inną trasą adresów będą odrzucane. Nie wykorzystuj tego wzorca jako gotowego sposobu „publikowania usługi” w produkcji.

Po wdrożeniu takiej policy router OSPF może utworzyć external LSA reprezentujący prefix. To odrębny proces od natywnego rozgłaszania interfejsów OSPF.

**R1-vSRX — Operational Mode, rozdzielenie trasy lokalnej i external LSA**

~~~junos
show route 172.31.10.0/24 exact
show ospf neighbor
show ospf database external
show configuration protocols ospf | display set
~~~

Na drugim routerze OSPF sprawdzamy, czy external LSA dotarł i czy route została rzeczywiście zainstalowana. Nie wystarczy zobaczyć tylko \`show ospf neighbor Full\`.

#### Dlaczego import \`reject\` nie usuwa wewnętrznego OSPF?

Teraz załóżmy, że R3 nie chce używać wewnętrznej trasy OSPF do 10.12.0.0/30. Administrator pisze \`protocols ospf import\` z termem \`then reject\` i oczekuje, że route zniknie.

W Junos **nie należy traktować OSPF import jak dowolnego filtra wszystkich LSA**. Zwykłe odrzucenie trasy nonexternal w imporcie OSPF jest ignorowane: link-state wymaga wspólnego obrazu topologii, a lokalna polityka nie powinna dowolnie usuwać z niego elementów. OSPF import ma zastosowania, przede wszystkim związane z **external routes** i określonymi atrybutami, lecz nie jest mechanizmem usuwania natywnych wewnętrznych prefiksów z LSDB.

Co zatem robić, jeśli nie chcemy, by pewien prefix był widoczny w całym enterprise? Zastanowić się nad projektem obszarów, sumaryzacją w odpowiednim punkcie, kontrolą external redistribution lub innym sposobem oddzielenia domen routingu. To są zagadnienia dla rozdziału OSPF, a nie próba wygrania walki z LSDB jedną linią \`reject\`.

**Praktyczne porównanie Cisco → Junos:** Cisco \`redistribute static subnets\` i Junos \`protocols ospf export POLICY\` mogą służyć podobnemu celowi. Ale samo dodanie OSPF-enabled interface nie jest redystrybucją \`Direct\` do external LSA. Jeśli chcesz ogłosić lokalną sieć LAN jako natywny prefix OSPF, zwykle konfigurujesz właściwy interfejs w OSPF (często passive dla sieci bez sąsiadów), a nie sztuczną policy export direct.

*Dokumentacja: [OSPF Routing Policy](https://www.juniper.net/documentation/us/en/software/junos/ospf/topics/topic-map/configuring-ospf-routing-policy.html), [Protocol Support](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-protocol-support-import-export-policies.html).*

### 8.12.2. IS-IS: zbliżona logika link-state, ale inny model poziomów

Drugi wariant: ten sam R1–R4 działa na **IS-IS**. Nie zmieniamy adresów 10.12, 10.13, 10.24, 10.34. Zmieniamy protokół odpowiedzialny za budowę topologii.

IS-IS również rozsyła własne informacje link-state. Domyślna polityka export nie oznacza, że routery nie wysyłają LSP. Eksport w policy framework służy między innymi do redystrybucji tras innych protokołów oraz — w odpowiedniej architekturze — do kontrolowanego **route leaking pomiędzy poziomami**.

Czym różni się to od OSPF? IS-IS posiada **Level 1** i **Level 2**. Router L1 jest zwykle zainteresowany informacjami lokalnego obszaru, a routery L2 odpowiadają za szerszy ruch pomiędzy obszarami. Gdy istnieje więcej niż jedno wyjście z obszaru L1, czasami warto wprowadzić wybrane bardziej szczegółowe prefiksy L2 do L1, aby ruch wybrał optymalną drogę zamiast tylko najbliższego routera L1/L2.

W Junos można projektować do tego polityki eksportu IS-IS. Warunki \`from protocol isis\`, \`from level 2\` i \`to level 1\` pomagają sprecyzować, które wpisy są źródłem oraz dokąd powinny trafić. To znacznie bardziej określone zastosowanie niż ogólne \`then accept\` dla całej tabeli.

W tym wariancie nie przygotowujemy gotowego „copy/paste” IS-IS do sześciu vSRX, ponieważ potrzebowałby NET, konfiguracji Levels, ISO family na interfejsach i odrębnego planu sąsiedztwa. To zakres rozdziału 10. Teraz najważniejsze jest zrozumienie **do jakiego punktu protokołu podpinana jest routing policy**.

**R2-vSRX — Operational Mode, gdy osobny wariant IS-IS został zbudowany**

~~~junos
show isis adjacency
show isis database extensive
show route protocol isis
show configuration protocols isis | display set
~~~

A co z \`protocols isis import\`? W nowszych wydaniach Junos taka funkcja istnieje, lecz nie oznacza swobodnego odrzucania wszystkich informacji nonexternal z LSDB. Producent opisuje ograniczenie, że \`reject\` dla niektórych tras nonexternal może zostać zignorowane. Stare materiały mogą dodatkowo opisywać IS-IS import jako niedostępny — dlatego zawsze wiąż temat z **konkretną wersją Junos** i aktualną CLI Reference.

*Dokumentacja: [IS-IS import](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-edit-protocols-isis.html), [IS-IS Route Prioritization](https://www.juniper.net/documentation/us/en/software/junos/is-is/topics/example/example-configuring-routing-policy-to-prioritize-isis-routes.html).*

### 8.12.3. BGP: tutaj import i export stają się podstawowym narzędziem sterowania

Wracamy do naszej standardowej topologii **eBGP na sześciu łączach**.

Na R6 najpierw eksportujemy local \`Direct 172.16.60.0/24\` do R4. R4 otrzymuje ją przez BGP, a następnie rozgłasza do R2 i R3. R1 otrzymuje dwa BGP paths, nakłada import policy, wybiera aktywną drogę i może ogłosić tę sieć do R5.

Tutaj polityki mają naturalne zastosowanie na **każdej granicy między AS**:

- R6 kontroluje, co ujawnia R4;
- R4 kontroluje, co przekazuje przez dwa korytarze;
- R1 kontroluje, które prefiksy zaakceptuje z R2 i R3;
- R1 kontroluje, co wolno przekazać do CE-A na R5.

BGP jest szczególnie elastyczny w pracy z prefixami i atrybutami. Jeśli R1 chce preferować 172.16.60.0/24 przez R2, import policy może nadać tej ścieżce LOCAL_PREF 200. Jeśli 172.16.61.0/24 ma iść dolnym korytarzem, inna import policy może dać LOCAL_PREF 200 ścieżce od R3.

**R1-vSRX — Operational Mode, porównanie dróg do dwóch niezależnych usług R6**

~~~junos
show route 172.16.60.0/24 exact extensive
show route 172.16.61.0/24 exact extensive
show bgp neighbor 10.12.0.2
show bgp neighbor 10.13.0.2
~~~

W prawidłowo skonfigurowanym labie oba prefiksy mają **po dwie dostępne BGP paths**, ale mogą wybierać przeciwny korytarz. To świadoma polityka sterowania ruchem, a nie konsekwencja „szybszego interfejsu” czy różnicy metryk OSPF.

### Jedno spojrzenie porównawcze

| Pytanie | OSPF | IS-IS | BGP |
|---|---|---|---|
| Czy własna informacja link-state jest rozgłaszana bez policy export? | Tak | Tak | Nie dotyczy tego mechanizmu |
| Czy policy export służy do redystrybucji? | Tak | Tak | Tak |
| Czy import policy pozwala dowolnie odrzucać wewnętrzne trasy protokołu? | Nie | Nie w zwykłym ujęciu nonexternal | Zwykle duża swoboda, z ograniczeniami BGP |
| Jak możemy sterować wyborem korytarza? | Topologia, metric, areas | Metric, levels, polityki w odpowiednich miejscach | Import, LOCAL_PREF, AS_PATH i pozostałe atrybuty |
| Co jest najważniejszą pułapką? | \`export reject\` nie zatrzymuje LSA flooding | Ograniczenia import i route leaking L1/L2 | Domyślny eksport aktywnych tras BGP i ryzyko route leak |

Wszystkie trzy protokoły używają słów \`import\` i \`export\`, ale oceniają trasy **w innych punktach swojego życia**. To ważniejsza wiedza niż zapamiętanie samych komend.

---

## 8.13. Zaawansowane polityki — modularność, wybór ścieżki i zapobieganie pętlom [P]

Na poziomie JNCIP nie wystarczy napisać polityki, która akceptuje jedną sieć i odrzuca pozostałe. W praktyce trzeba połączyć kilka wymagań bez sytuacji, w której nowa reguła niszczy starą.

W naszym diamencie mamy już czytelny zestaw potrzeb:

- \`172.16.60.0/24\` od R6 ma być preferowane **górą**, R6 → R4 → R2 → R1;
- \`172.16.61.0/24\` ma być preferowane **dołem**, R6 → R4 → R3 → R1;
- \`172.16.62.0/24\` nie może zostać zaakceptowane jako poprawna droga do R5;
- do R2 i R3 R1 ma reklamować tylko prefiksy od **własnego CE-A R5**, nie może przypadkiem stać się tranzytem pomiędzy górnym i dolnym korytarzem.

To nie jest problem jednej magicznej komendy. Potrzebujemy podzielić odpowiedzialność między polityki.

### 8.13.1. Modularny łańcuch: wspólne zasady, indywidualne preferencje

Najpierw stwórz politykę wspólną \`FILTER-CE-B\`, która odrzuca testowy prefix /62. Potem użyj dwóch odrębnych polityk preferencji dla R2 i R3.

**R1-vSRX — Configuration Mode, wspólny filtr importu**

~~~junos
set policy-options policy-statement FILTER-CE-B term DROP-TEST from route-filter 172.16.62.0/24 exact
set policy-options policy-statement FILTER-CE-B term DROP-TEST then reject
set policy-options policy-statement FILTER-CE-B term CONTINUE then next policy
~~~

**R1-vSRX — Configuration Mode, preferencja górnego korytarza dla 172.16.60.0/24**

~~~junos
set policy-options policy-statement PREFER-UPPER term BUSINESS from route-filter 172.16.60.0/24 exact
set policy-options policy-statement PREFER-UPPER term BUSINESS then local-preference 200
set policy-options policy-statement PREFER-UPPER term BUSINESS then accept
set policy-options policy-statement PREFER-UPPER term SERVICES from route-filter 172.16.61.0/24 exact
set policy-options policy-statement PREFER-UPPER term SERVICES then local-preference 100
set policy-options policy-statement PREFER-UPPER term SERVICES then accept
set policy-options policy-statement PREFER-UPPER term REST then reject
~~~

**R1-vSRX — Configuration Mode, preferencja dolnego korytarza dla 172.16.61.0/24**

~~~junos
set policy-options policy-statement PREFER-LOWER term BUSINESS from route-filter 172.16.60.0/24 exact
set policy-options policy-statement PREFER-LOWER term BUSINESS then local-preference 100
set policy-options policy-statement PREFER-LOWER term BUSINESS then accept
set policy-options policy-statement PREFER-LOWER term SERVICES from route-filter 172.16.61.0/24 exact
set policy-options policy-statement PREFER-LOWER term SERVICES then local-preference 200
set policy-options policy-statement PREFER-LOWER term SERVICES then accept
set policy-options policy-statement PREFER-LOWER term REST then reject
~~~

**R1-vSRX — Configuration Mode, dwa kompletne łańcuchy importu pod dwoma sąsiadami**

~~~junos
set protocols bgp group CORE neighbor 10.12.0.2 import [ FILTER-CE-B PREFER-UPPER ]
set protocols bgp group CORE neighbor 10.13.0.2 import [ FILTER-CE-B PREFER-LOWER ]
~~~

Teraz dopiero zobaczysz, po co całe to rozdzielenie. \`FILTER-CE-B\` jest wspólne dla obu peerów i zawsze wykonuje się jako pierwsze. \`PREFER-UPPER\` i \`PREFER-LOWER\` sterują natomiast różnymi atrybutami dla dwóch różnych korytarzy. Każda ma własny końcowy reject, więc nie dopuszczamy innych przypadkowych prefiksów tylko dlatego, że ominęły pierwszy term.

### 8.13.2. LOCAL_PREF wpływa na wyjście z R1 — nie steruje samodzielnie całym Internetem

Warto zrozumieć, że **LOCAL_PREF** dotyczy własnego wyboru BGP. Wyższy jest zwykle preferowany przez BGP; nasza policy nadaje 200 dla wybranej ścieżki, a 100 dla alternatywnej.

Nie należy mylić tego z Junos **route preference**. Lokalna route preference mówi, który *protokół* wygrywa w RIB dla tego samego prefixu. LOCAL_PREF rozróżnia *BGP paths*, zanim najlepsza droga BGP zostanie porównana z innymi protokołami.

Jeżeli R1 ma aktywną OSPF/10 do 172.16.60.0/24, może ona wygrać w Junos RIB, nawet gdy ścieżka BGP ma LOCAL_PREF 200. To nie znaczy, że BGP źle policzył swoje atrybuty. Oznacza, że finalna selekcja w RIB jest innym etapem.

### 8.13.3. AS_PATH prepend, MED i communities — kogo możemy przekonać, a kogo nie?

Na R4 mamy dwie sesje eBGP do rdzenia: R2 i R3. Gdyby R4 chciał *zasugerować*, aby R1 preferował określoną trasę przez R2, może zastosować różne atrybuty eksportu, np. dodać AS_PATH prepend tylko na połączeniu do R3. Ale to zadziała tylko wtedy, gdy R1 oraz inne routery nie mają silniejszej polityki — np. właśnie skonfigurowanego LOCAL_PREF.

**R4-vSRX — Configuration Mode, koncepcyjny wariant AS_PATH prepend dla R3**

~~~junos
set policy-options policy-statement TO-LOWER term CE-B from protocol bgp
set policy-options policy-statement TO-LOWER term CE-B from route-filter 172.16.60.0/24 exact
set policy-options policy-statement TO-LOWER term CE-B then as-path-prepend "65004 65004"
set policy-options policy-statement TO-LOWER term CE-B then accept
set policy-options policy-statement TO-LOWER term REST then reject
~~~

**Uwaga projektowa:** nie podpinaj tej polityki jako całego eksportu R4 do R3 w podstawowym labie, bo ograniczyłaby pozostałe prefiksy, w tym 172.16.61.0/24. To osobny eksperyment pokazujący działanie atrybutu; do produkcji należałoby zbudować pełny, bezpieczny łańcuch obejmujący wszystkie uzgodnione prefiksy.

\`as-path-prepend\` może sprawić, że ścieżka wygląda na dłuższą dla obcych AS. Nie daje gwarancji wyboru R2, jeśli odbiorca preferuje inną drogę za pomocą LOCAL_PREF.

\`metric\` w polityce BGP może oznaczać atrybut MED. W klasycznym przypadku jest on porównywany według zasad BGP, często istotny pomiędzy trasami od tego samego sąsiedniego AS; jego porównywanie i wpływ na wybór drogi zależy od konfiguracji. MED nie jest uniwersalnym przyciskiem „ta droga ma pierwszeństwo”.

BGP communities pozwalają uzgodnić z operatorem reguły sterowania: np. operator może odczytać community i odpowiednio ustawić własne LOCAL_PREF. Ale to **operator** musi naprawdę realizować taką politykę. Bez umowy o znaczeniu community są po prostu dodatkowymi metadanymi.

### 8.13.4. Anti-loop przy redystrybucji pomiędzy OSPF i BGP

Na poziomie Professional pojawia się kolejny rodzaj problemu: **dwa punkty redystrybucji między protokołami**. Wyobraź sobie alternatywny wariant DIAMOND-6R, w którym R1–R4 tworzą domenę OSPF, a R1 i R4 są jednocześnie granicami do sąsiadów BGP.

R4 może odebrać od R6 \`172.16.60.0/24\` jako BGP i wprowadzić ją do OSPF jako external. R1 widzi wtedy ten prefix jako OSPF external. Jeżeli R1 bez ograniczeń wprowadzi każdą OSPF route z powrotem do BGP, może przekazać informację, która pierwotnie pochodziła z BGP, do niewłaściwej domeny. Jeśli w sieci jest drugi punkt wzajemnej redystrybucji, mogą powstać niepożądane informacje zwrotne, hairpinning i pętle.

Szczególnie ważna jest różnica wartości preferencji Junos: **OSPF External 150** jest domyślnie preferowane względem **BGP 170** dla tego samego prefixu. To może prowadzić do zaskakującego wyboru OSPF, choć operator sądził, że „bezpośrednio odebrana eBGP będzie lepsza”, tak jak bywa w Cisco IOS z AD 20 dla eBGP i 110 dla OSPF.

Typową strategią ochrony jest **tagging**. Gdy R4 eksportuje trasę BGP do OSPF, dodaje określony route tag. Gdy R1 później przygotowuje OSPF→BGP, odrzuca trasy zawierające ten tag. To nie zwalnia z osobnego filtrowania prefiksów ani dokładnego zaplanowania obu granic.

**R4-vSRX — Configuration Mode, przykład oznaczania tras BGP w eksporcie do OSPF**

~~~junos
set policy-options policy-statement BGP-TO-OSPF term CE-B from protocol bgp
set policy-options policy-statement BGP-TO-OSPF term CE-B from route-filter 172.16.60.0/24 exact
set policy-options policy-statement BGP-TO-OSPF term CE-B then tag 65106
set policy-options policy-statement BGP-TO-OSPF term CE-B then accept
set policy-options policy-statement BGP-TO-OSPF term REST then reject
~~~

**R1-vSRX — Configuration Mode, ochrona eksportu OSPF → BGP przed powrotem oznaczonej trasy**

~~~junos
set policy-options policy-statement OSPF-TO-BGP term BLOCK-RETURN from protocol ospf
set policy-options policy-statement OSPF-TO-BGP term BLOCK-RETURN from tag 65106
set policy-options policy-statement OSPF-TO-BGP term BLOCK-RETURN then reject
set policy-options policy-statement OSPF-TO-BGP term ALLOW-LOCAL from protocol ospf
set policy-options policy-statement OSPF-TO-BGP term ALLOW-LOCAL from route-filter 172.16.50.0/24 exact
set policy-options policy-statement OSPF-TO-BGP term ALLOW-LOCAL then accept
set policy-options policy-statement OSPF-TO-BGP term REST then reject
~~~

Nie podpinamy tych dwóch przykładów do aktywnego labu eBGP rozdziału 8: są ilustracją odrębnego wariantu OSPF/BGP. Zanim ich użyjesz, trzeba sprawdzić, czy R1 rzeczywiście posiada \`172.16.50.0/24\` jako aktywną OSPF route, czy eksport ma sens dla adresacji klienta i czy nie rozgłaszamy prefiksu własnego AS w nieodpowiednim kierunku.

### 8.13.5. Policy subroutines i policy expressions

Gdy polityki stają się bardziej rozbudowane, czasami musimy użyć jednej polityki jako **warunku** w innej. Junos obsługuje subroutines oraz policy expressions, które pozwalają budować bardziej zaawansowane warunki logiczne.

Nie są one jednak zamiennikiem dobrze zaprojektowanego łańcucha \`[ FILTER TAG ALLOW ]\`. Wywołując policy jako subroutine, trzeba rozumieć, jaki wynik logiczny zwraca \`accept\` lub \`reject\`, jak interpretowane są nieterminujące akcje i czy modyfikacje atrybutów wpływają na dalszą ocenę. To dużo łatwiej przetestować na małej, izolowanej tabeli niż wdrażać od razu przy wielu peerach.

W praktyce na R1 najprostszy i najbardziej czytelny model dla naszych trzech prefiksów to nadal **jawna whitelist + lokalne preferencje + końcowy reject**. Nie stosujemy wielopoziomowej logiki tylko dlatego, że Junos ją umożliwia.

*Dokumentacja: [Policy Chain Evaluation](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-chain-evaluation-method.html), [Policy Expressions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-applying-policy-expressions-to-routes-exported-from-routing-tables.html), [Actions in Routing Policy Terms](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-actions-in-routing-policy-terms.html).*
