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
