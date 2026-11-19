# STATUS: DRAFT ARCHIWALNY — NIEKANONICZNY

**Ta wersja alternatywna agenta nie jest bieżącym Rozdziałem 8.** Zawiera historyczną dwurouterową adresację i jest zachowana wyłącznie do porównania. Używaj [aktualnego rozdziału 8](JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters.md) oraz [kanonicznego DIAMOND Mermaid](../../labs/reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md) (4×vSRX+2×VPCS). Nie używaj topologii z tego pliku do nowych ćwiczeń ani promptów dla Codexa.

---

# Rozdział 8 — Routing Policy and Firewall Filters
## Kto może opowiedzieć o trasie, a kto może przepuścić pakiet?

**Podręcznik:** JUNOS Enterprise Routing & Switching — JNCIA-Junos → JNCIS-ENT → JNCIP-ENT  
**Zakres kanoniczny:** 8.1–8.14, [A → S → P], [Unified Blueprint](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md)  
**Wersja redakcyjna:** 1.0 — 19 listopada 2026  
**Status:** GENERATED / SOURCE-REVIEWED. Konfiguracje laboratoryjne są przykładami do odtworzenia; **NOT RUNTIME VALIDATED**. Nie wykonano testów w EVE-NG ani na sprzęcie.  
**Platformy:** Junos OS na routerach, przełącznikach i vSRX, z uwagami o zależnościach od platformy/release; nie utożsamiaj PFE EX/QFX/MX z programowym dataplane vSRX.  
**Zasada:** najpierw przewidywanie decyzji, potem dowód w CLI. Wszystkie wyjścia oznaczone jako przykładowe/symulowane, jeśli nie zaznaczono inaczej.

> **Cel tej lekcji:** po incydencie masz umieć powiedzieć nie tylko „prefix nie został rozgłoszony”, lecz także: *która polityka dostała tę trasę, w którym termie ją zatrzymała, czy route był kwalifikowany do eksportu, czy BGP go ogłosił, a jeżeli ruch i tak nie działa — czy zawinił RIB/FIB, stateless filter, uRPF czy security flow na SRX.*

---

## Prolog. Zielony BGP, martwy ERP — trzy różne bramki

Wyobraź sobie nowy oddział z routerem Juniper na brzegu WAN. Router utrzymuje eBGP z operatorem. Sąsiad jest `Established`, interfejs jest `up/up`, w logach cisza. Mimo to centrala nie widzi podsieci oddziału `10.10.10.0/24`. Operator pyta: „Czy na pewno eksportujecie tę sieć?”. Ty odpowiadasz: „Przecież mam ją w `inet.0`!”. To jeszcze niczego nie dowodzi.

Godzinę później po naprawie ogłoszenia centrala widzi prefiks, a aplikacja nadal nie działa. Na interfejsie LAN routera narasta licznik `BLOCK-ERP`. Ktoś zaczyna edytować BGP export policy. To drugi błąd: **routing policy steruje informacjami o trasach; firewall filter analizuje konkretne pakiety**. Dodatkowo na vSRX ruch może zatrzymać **security policy** po przetwarzaniu stanu sesji.

Pomyśl o trzech kontrolach w budynku:

1. **Routing policy** odpowiada, jakie adresy i instrukcje dotarcia wolno wpisać do firmowego informatora i przekazać partnerowi.
2. **Firewall filter** sprawdza każdą przesyłkę na wejściu lub wyjściu — na podstawie nagłówka, bez pamięci, czy podobna przesyłka przeszła przed chwilą.
3. **SRX security policy** kontroluje *rozmowę*: bierze pod uwagę kierunek inicjacji, strefy i stan sesji; dla nowych połączeń podejmuje decyzję o ustanowieniu sesji.

Analogia kończy się tu: Junos ma określoną kolejność processingu, sprzętowe zasoby filtrów i reguły wyboru ścieżki. „Trzy kontrole” nie są trzema identycznymi ACL-ami wykonywanymi w niezmiennym porządku na wszystkich platformach.

**Przewidź wynik:** jeśli odrzucisz `10.10.10.0/24` w BGP export na R1, czy pakiet PC-A do lokalnej bramy `10.10.10.1` zostanie odrzucony? Nie. Export nie jest filtrem dataplane. Lokalny ruch i dostępność connected prefix nie znikają tylko dlatego, że nie został zareklamowany do BGP. Teraz wyjaśnimy dlaczego.

**Na pamięć:** `policy-options policy-statement`, `term`, `from`, `to`, `then`, `accept`, `reject`, `next term`, `next policy`, `firewall family inet filter`, `count`, `discard`, `rpf-check`.  
**Do zrozumienia:** import/export jako punkty zastosowania polityki; protokołowe default policy; route filtering versus packet filtering; LPM w route-filter versus w FIB; stateless versus stateful.  
**Do przećwiczenia:** BGP Adj-RIB-In/import/RIB/export/advertising; licznik filtra, dopasowanie termów, shadowing, uRPF, rollback.

### Mapa decyzji (oryginalny diagram Mermaid; alt: dwie równoległe ścieżki informacji routingu i pakietu)

~~~mermaid
flowchart TB
  N["Sąsiad BGP / OSPF / IS-IS"] --> RI["Import policy, gdy protokół ją wspiera"]
  RI --> RIB["RIB: inet.0 / inet6.0"]
  RIB --> SEL["Wybór aktywnej ścieżki"]
  SEL --> EXP["Export policy i reguły protokołu"]
  EXP --> ADV["Ogłoszenie do sąsiada"]
  SEL --> FIB["Forwarding table"]
  HOST["Pakiet z hosta"] --> IN["Interface input filter / uRPF"]
  IN --> SEC["SRX flow i security policy, jeśli SRX"]
  SEC --> FIB
  FIB --> OUT["Interface output filter / egress"]
  OUT --> PEER["Następny router / host"]
~~~

**Interpretacja:** górna gałąź mówi o prefiksach; dolna o pakietach. Przejście trasy do RIB nie gwarantuje aktywności ani eksportu. Przejście do FIB nie gwarantuje przepuszczenia pakietu przez SRX. Pozycje bloków pakietowych na diagramie są **logiczne**; konkretna kolejność etapów i obsługa sprzętowa zależą od platformy, funkcji i kierunku. Weryfikuj ją dokumentacją platformową. [S1], [S9], [S14].

---

## 8.1. Default routing policies. [A]

### 8.1.1. „Nie skonfigurowałem polityki” nie znaczy „wszędzie permit”

Administrator Cisco zna pomysł: routing protocols mają zasady rozgłaszania. Może jednak przenieść intuicję „nie ma route-map, więc wszystko pójdzie dalej”. W Junos to zbyt śmiała generalizacja. **Domyślna polityka to reguła zależna od protokołu, kierunku i czasem rodziny platformy**, a nie jeden globalny `permit any`.

Pytanie operatorskie: R1 ma aktywną trasę static `10.10.10.0/24`. Włącza eBGP, ale nie konfiguruje żadnej export policy. Czy trasa statyczna zostanie automatycznie rozgłoszona? Standardowe zachowanie BGP Junos na zwykłych routerach: domyślnie readvertise aktywne trasy *BGP* z respektowaniem ograniczeń protokołu; nie jest to automatyczne „redistribute connected/static”. Żeby wysłać static, potrzebujesz świadomego eksportu.

Kolejne pytanie: czy OSPF bez export policy ogłosi własne sieci interfejsów aktywowanych w OSPF? Tak — przez mechanizm link-state flooding. Domyślna export policy OSPF odrzuca **redistribution z innych źródeł**, a nie normalne rozgłaszanie stanu własnych łączy. **Nie wnioskuj, że „OSPF export reject everything” wyłącza wewnętrzne LSAs.**

Orientacyjna macierz dla klasycznych zastosowań Junos (nie jest uniwersalną tabelą dla każdego PTX/release):

| Źródło | Domyślne import | Domyślne export | Pułapka |
|---|---|---|---|
| BGP | Przyjmuje kwalifikowane trasy od sąsiadów (IPv4/IPv6 do odpowiednich RIB) | Ogłasza aktywne trasy BGP, o ile pozwalają reguły BGP | Nie oznacza automatycznego ogłoszenia static/connected |
| OSPF | Przyjmuje trasy OSPF; import policy dotyczy tras zewnętrznych | Nie redystrybuuje obcych tras bez export policy | Flooding tras wewnętrznych OSPF to inna ścieżka |
| IS-IS | Przyjmuje trasy IS-IS | Nie redystrybuuje tras z innych protokołów bez export | Własne informacje link-state nadal są floodowane |
| Static/direct | Własne trasy trafiają do lokalnego RIB, jeśli są kwalifikowane | Same nie są procesem reklamującym trasy | Eksportuje je *inny* protokół poprzez swoją policy |

**Most z Cisco:** konfiguracja `redistribute static route-map ...` w OSPF/BGP jest dobrą intuicją intencji. Ale w Junos najczęściej przygotujesz `policy-options policy-statement`, a następnie podepniesz ją jako `export` pod wybranym protokołem. Nie zamieniaj `export` w mechanizm reklamacji wszystkich tras automatycznie — protokół nadal egzekwuje własne ograniczenia, aktywność i możliwości address family.

Zobacz też specjalny przypadek PTX: producent jawnie wyłącza go z uproszczonej reguły domyślnej polityki BGP. Dlatego w projekcie produkcyjnym zapisujemy **model + Junos release + family + policy hierarchy**. [S2]

### 8.1.2. Mini-incydent: „BGP działa, ale connected nie widać”

R1 ma connected `10.10.10.0/24` i BGP neighbor Established. Test:

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show bgp summary
show route 10.10.10.0/24 exact
show route advertising-protocol bgp 192.0.2.2
show configuration protocols bgp | display set
~~~

Jeśli prefix jest aktywny, a trasa nie pojawia się w ogłoszeniach, pytasz: czy aktywna trasa była w ogóle eligible do domyślnego BGP export? Czy włączono właściwy `family inet unicast`? Czy export policy explicite akceptuje tę trasę? Sam Established mówi o stanie sesji, nie o zawartości NLRI.

**Do zapamiętania:** default policy to **ostatnia decyzja mechanizmu protokołu**, gdy własne termy i łańcuch polityk nie dały jawnej akceptacji/odrzucenia. Nie jest to implicit deny znane z packet ACL. [S2], [S3]

---

## 8.2. Import policy vs. export policy. [A]

### 8.2.1. Na którym brzegu RIB stoi filtr?

Import i export są intuicyjne dopóki nie zadasz pytania: *import skąd i do czego?* W Junos chodzi o relację między **protokołem** i **tablicą routingu**, nie o kierunek ruchu użytkowników na fizycznym porcie.

**Import:** sąsiad wysyła NLRI/route, protokół odbiera informację, policy może ją odrzucić albo zmienić atrybut, a kwalifikowany wynik jest kandydatem w RIB. Nie jest to pakiet IP przenoszący ruch aplikacji.

**Export:** protokół ogląda kandydatów z RIB, dokonuje doboru tras, które wolno mu reklamować, a policy decyduje, które przejdą dalej i z jakimi atrybutami. Eksport nie kopiuje fizycznie tabeli RIB w całości do neighbor. Dla BGP domyślnie rozważa się aktywne trasy (wyjątki wymagają osobnych funkcji, np. `advertise-inactive`).

~~~mermaid
flowchart LR
  PEER["R2: BGP UPDATE"] --> RX["R1: BGP received routes"]
  RX --> IMP["IMPORT: match / modify / accept-reject"]
  IMP --> RIB["R1: inet.0"]
  RIB --> ELIG["Aktywność + eligibility"]
  ELIG --> EXP["EXPORT: match / modify / accept-reject"]
  EXP --> TX["R3: BGP UPDATE"]
~~~

**Interpretacja:** policy działa na *obiektach routingu*, nie na przechodzącym przez router pakiecie do serwera. Może wpłynąć na pakiety pośrednio: jeśli odrzucisz trasę w import, może zabraknąć drogi do celu albo wygra alternatywna ścieżka. Ale nie dostajesz tym samym stateless ACL dla klientów.

**Most z Cisco:** BGP `neighbor ... route-map ... in/out` oraz `prefix-list in/out` to najbliższy praktyczny mental model. Junos buduje reguły w `policy-options`, potem referencjonuje w `protocols bgp group ... import/export`. Różni się też to, jak zachowuje *rejected* routes: Junos może zachować je jako **hidden**. To diagnostycznie cenne, ale zużywa zasoby; `keep none` zmienia retencję i ma konsekwencje dla reewaluacji. [S1], [S5]

### 8.2.2. Skąd czytać dowód?

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show bgp summary
show route receive-protocol bgp 192.0.2.1
show route receive-protocol bgp 192.0.2.1 hidden
show route protocol bgp
show route 10.10.10.0/24 exact extensive
show route advertising-protocol bgp 192.0.2.1
~~~

Pierwsze polecenie potwierdza sesję i statystyki, następne pokazują perspektywę przyjmowanych i **hidden** ścieżek od konkretnego peera; `show route protocol bgp` to szerszy widok BGP w RIB. Ogłoszenia sprawdza się *na routerze wysyłającym*. Uwaga: output może różnić się według software i family; `advertising-protocol` pokazuje perspektywę eksportu protokołu, nie wiarygodny packet capture na kablu.

**Gotcha:** w BGP odrzucona trasa często nadal jest widoczna w hidden receive; brak widoku w `show route` bez `hidden` nie znaczy, że peer jej nie wysłał. Inaczej wygląda sytuacja z powodu AS-path loop, niedozwolonej family lub gdy peer nie reklamuje route wcale. [S5], [S6]

---

## 8.3. Routing policy flow — kolejność przetwarzania i zakończenie ewaluacji. [A]

### 8.3.1. Term to bramka logiczna, a nie tylko linia konfiguracyjna

Mamy dwie sieci z oddziałów. R1 ma policy z trzema termami:

- `BLOCK-TEST` — odrzuca `198.51.100.0/24`.
- `MARK-BRANCH` — dopasowuje `10.10.10.0/24` i zmienia atrybut.
- `ALLOW-REST` — akceptuje pozostałe.

Routing policy analizuje jedną trasę od **pierwszego** termu. Jeśli match conditions nie pasują, przechodzi do następnego. Jeśli pasują i następuje `accept`, koniec ewaluacji **całego łańcucha**. Dla `reject` również koniec. `next term` kontynuuje od kolejnego termu w tej samej polityce. `next policy` przeskakuje pozostałe termy bieżącej polityki i przechodzi do następnej w łańcuchu.

Jeszcze subtelniej: term, który *dopasował trasę i zmienił atrybut*, ale **nie ma terminating action**, z reguły pozwala ewaluacji iść dalej; zmiana atrybutu może zostać wykorzystana przez następne termy/policies. Nie myl „match occurred” z „route accepted”.

### 8.3.2. Przykładowa analiza krok po kroku

**Urządzenie: R1-vSRX — Configuration Mode (nie podpinaj policy do produkcyjnego peera bez review)**

~~~junos
set policy-options policy-statement EXPORT-WAN term DENY-TEST from route-filter 198.51.100.0/24 exact
set policy-options policy-statement EXPORT-WAN term DENY-TEST then reject
set policy-options policy-statement EXPORT-WAN term PERMIT-BRANCH from protocol direct
set policy-options policy-statement EXPORT-WAN term PERMIT-BRANCH from route-filter 10.10.10.0/24 exact
set policy-options policy-statement EXPORT-WAN term PERMIT-BRANCH then accept
set policy-options policy-statement EXPORT-WAN term DENY-REST then reject
~~~

Rozważ trzy trasy:

| Kandydat RIB | DENY-TEST | PERMIT-BRANCH | DENY-REST | Wynik |
|---|---|---|---|---|
| Direct `10.10.10.0/24` | miss | match i accept | nieosiągnięty | export permit |
| Static `198.51.100.0/24` | match i reject | nieosiągnięty | nieosiągnięty | export deny |
| Direct `10.10.11.0/24` | miss | miss | match i reject | export deny |

W terminie PERMIT-BRANCH dwa warunki `from` — `protocol direct` i `route-filter ... exact` — muszą pasować równocześnie (logiczne AND). Wiele prefixów wewnątrz listy route-filter to zwykle alternatywy (OR), lecz **matching prefix list ma własne reguły longest-match**, do których wrócimy.

Pytanie: co się stanie, jeśli dopiszesz `then accept` do pierwszego termu z `from protocol direct` bez sprawdzenia konkretnego prefiksu? Pozwolisz na eksport **każdej kwalifikowanej trasy direct**, np. sieci OOB lub łącza tranzytowego, jeżeli protokół je rozważa. To jeden z prostszych sposobów ujawnienia adresacji infrastruktury.

### 8.3.3. Łańcuchy: różnica między dwoma rodzajami kontynuacji

**Urządzenie: R1-vSRX — Configuration Mode, przykład dydaktyczny**

~~~junos
set policy-options policy-statement PREPARE term TAG-BRANCH from route-filter 10.10.10.0/24 exact
set policy-options policy-statement PREPARE term TAG-BRANCH then metric 100
set policy-options policy-statement PREPARE term TAG-BRANCH then next policy
set policy-options policy-statement FINAL term APPROVED from route-filter 10.10.10.0/24 exact
set policy-options policy-statement FINAL term APPROVED then accept
set policy-options policy-statement FINAL term ALL-ELSE then reject
set protocols bgp group TO-ISP export [ PREPARE FINAL ]
~~~

Dla `10.10.10.0/24` pierwsza policy zmienia MED (w BGP export `metric` to MED) i każe przejść do kolejnej policy. Druga akceptuje. Dla pozostałych tras brak match w PREPARE powoduje przejście dalej; FINAL odrzuca je w ostatnim termie.

**Gotcha:** jeżeli w PREPARE użyjesz `then accept`, to FINAL nigdy nie zobaczy pasującej trasy. Jeśli w FINAL nie będzie explicit reject i route nie dopasuje żadnego termu, **zadziała protocol-specific default policy**, co może dać inny wynik, niż sądzisz. **Łańcuch nie jest automatycznie koniunkcją kilku whitelist.** [S3], [S4]

~~~mermaid
flowchart TD
  START["Nowy kandydat: route"] --> T1{"Policy PREPARE: term TAG?"}
  T1 -->|match| MOD["Set metric 100; next policy"]
  T1 -->|miss| T2["Koniec PREPARE"]
  MOD --> F{"Policy FINAL: term APPROVED?"}
  T2 --> F
  F -->|match| A["ACCEPT: stop"]
  F -->|miss| R["ALL-ELSE: REJECT, stop"]
~~~

**Interpretacja:** zmiana atrybutu i decyzja o akceptacji to odrębne operacje. W CLI powinieneś umieć wskazać dokładnie term, który zakończył ewaluację.

---

## 8.4. Wpływ policies na trasy i routing tables. [A]

### 8.4.1. Dlaczego „trasa zniknęła z RIB” nie jest pełną diagnozą?

Przypomnij sobie `inet.0` z rozdziału 6: może istnieć wiele kandydatów do jednego prefixu, lecz tylko jeden jest aktywny (pomijamy szczegóły ECMP). Import policy może wykonać co najmniej trzy jakościowo inne rzeczy:

1. **Reject** — dany kandydat nie uczestniczy normalnie w wyborze aktywnej ścieżki, a w BGP może pozostać hidden.
2. **Accept bez modyfikacji** — route staje się kandydatem, ale może przegrać z innym źródłem przez preference/protocol selection.
3. **Accept ze zmianą** — zmieniasz np. `local-preference`, `preference` lub atrybuty BGP; zmieniasz warunki wyboru lub późniejszego eksportu.

Export policy **nie wymazuje** aktywnej trasy z RIB. Umożliwia lub blokuje jej rozgłoszenie przez dany protokół i może modyfikować atrybuty *ogłoszenia*.

### 8.4.2. Ścieżka konkretnego prefiksu

Router R2 otrzymał dwa BGP paths do `172.16.40.0/24`, z ISP-A i ISP-B. Import policy na ISP-A zwiększyła BGP `local-preference` do 200; na ISP-B pozostało 100. Jeśli inne warunki BGP na to pozwalają, path od ISP-A wygrywa BGP selection. Następnie musi być kwalifikowany do bycia aktywnym w Junos RIB. Może jeszcze przegrać z trasą innego protokołu dla **identycznego prefixu**, np. statyczną o lepszej route preference. Więc „wygrywa BGP best path” i „jest aktywna w `inet.0`” to nie zawsze ta sama wypowiedź.

**Most z Cisco:** BGP `weight` to cecha Cisco, nie uniwersalny atrybut używany w Junos; w Junos ważne są m.in. BGP LOCAL_PREF, AS_PATH, MED, a przy rywalizacji różnych protokołów Junos `preference`. Nie zastępuj LOCAL_PREF route preference — jedna reguła działa w procesie BGP, druga w szerszym wyborze routingu.

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show route 172.16.40.0/24 exact extensive
show route protocol bgp hidden
show bgp neighbor 192.0.2.1
show route forwarding-table destination 172.16.40.10
~~~

Tu zbierasz cztery dowody: jakie paths istnieją, co jest hidden, czy neighbor jest prawidłowy i jaki forwarding wynikł z procesu routingu. **Nie wnioskuj o FIB wyłącznie z wyjścia policy.**

### 8.4.3. Gotcha: polityka nie zmienia zawartości LSDB tak jak oczekuje Cisco engineer

OSPF/IS-IS są link-state. Import policy w OSPF dotyczy tras **zewnętrznych**, a nie swobodnego odmówienia dowolnego intra-area LSA. Router musi mieć spójny widok link-state topology; tłumienie wewnętrznych tras w RIB przez dowolną route policy nie jest odpowiednikiem inbound distribute-list z Cisco. W IS-IS również nie należy mylić policy na wybór/instalację tras z filtrowaniem samego flooding. To rozwiniemy w 8.12. [S2], [S12]

---

## 8.5. Struktura polityki: policy statements i terms. [A]

### 8.5.1. Zamiast route-map — wielokrotnego użytku obiekt

W Cisco route-map jest zwykle identyfikowana nazwą i numerami sekwencji. Junos używa hierarchii:

- `policy-options` — globalne definicje polityk, list i communities;
- `policy-statement NAME` — nazwana polityka;
- `term NAME` — kolejna gałąź warunków i działań;
- `from` / `to` — kryteria;
- `then` — actions.

Polityka nie jest aktywna tylko dlatego, że istnieje. Musi zostać **przypisana do punktu użycia**: BGP import/export, OSPF export, static route import context, forwarding-table export, itd. To odpowiednik ACL utworzonego, ale nie dołączonego do interfejsu — istnieje, niczego jeszcze nie zmienia.

~~~mermaid
flowchart TB
  P["policy-options"] --> PS["policy-statement EXPORT-CUSTOMER"]
  PS --> T1["term CUSTOMER"]
  PS --> T2["term ELSE"]
  T1 --> FROM["from protocol direct AND route-filter"]
  T1 --> THEN["then accept"]
  T2 --> REJ["then reject"]
  PS -. "referencja" .-> BGP["protocols bgp group EBGP export"]
~~~

**Interpretacja:** hierarchia oddziela definicję od zastosowania. Dlatego w trakcie awarii oprócz `show configuration policy-options` musisz sprawdzić `show configuration protocols ...` — czy właściwa grupa/peer rzeczywiście referencjonuje politykę.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
set policy-options policy-statement EXPORT-CUSTOMER term CUSTOMER from protocol direct
set policy-options policy-statement EXPORT-CUSTOMER term CUSTOMER from route-filter 10.10.10.0/24 exact
set policy-options policy-statement EXPORT-CUSTOMER term CUSTOMER then accept
set policy-options policy-statement EXPORT-CUSTOMER term ELSE then reject
set protocols bgp group TO-ISP export EXPORT-CUSTOMER
show | compare
commit check
~~~

Na początku **nie dokonuj commit na aktywnym połączeniu bez planu**. W laboratorium, po zapisaniu baseline i posiadaniu konsoli, użyj `commit confirmed`, a następnie sprawdź reklamowane prefiksy. Nie wykonuj `commit check` podczas oczekującego confirmed commit (może zatwierdzić zmianę na niektórych wariantach workflow).

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show configuration policy-options policy-statement EXPORT-CUSTOMER | display set
show configuration protocols bgp group TO-ISP | display set
show route advertising-protocol bgp 192.0.2.2
~~~

### 8.5.2. Pusta gałąź może być bardziej niebezpieczna niż się wydaje

Jeśli term nie ma `from` ani `to`, pasuje do **wszystkich** rozpatrywanych tras. Jeżeli jest na początku i ma `then accept`, wszystkie późniejsze termy są martwe (shadowed). Jeśli term ma warunki, ale brak actions, mechanizm *kontynuuje*, a nie automatycznie odrzuca. Dlatego w książce pokazujemy **jawne zakończenie whitelisty**; nie polegamy na przypadkowych protokołowych default actions. [S3]

---

## 8.6. Match criteria, typy dopasowań i actions w routing policy. [A]

### 8.6.1. Prefix to nie tylko adres; liczy się długość

Znamy `10.50.0.0/16`. Czy pasuje do `10.50.3.0/24`? To zależy od operatora dopasowania. W routing policy rozróżniamy m.in.:

| Match type | Dla `10.50.0.0/16` | Dopasowanie |
|---|---|---|
| `exact` | `10.50.0.0/16` | tylko identyczny prefix i długość |
| `longer` | `10.50.3.0/24`, `10.50.3.1/32` | wszystkie bardziej szczegółowe, ale nie samo /16 |
| `orlonger` | /16 oraz jego bardziej szczegółowe | /16 i dłuższe |
| `upto /24` | /16…/24 w obrębie zakresu | do wskazanej maksymalnej długości |
| `prefix-length-range /24-/28` | wybrane /24…/28 w /16 | zakres długości; składnia/zakres zgodne z CLI Reference |

**Pułapka z maską:** `orlonger` nie oznacza „bardziej szczegółowa sieć wygrywa, gdy router forwarduje pakiet”. To **warunek dopasowania kandydatów routingu do termu policy**, a nie dataplane LPM. W praktyce często potrzebujesz `exact`, by operator nie dostał tysięcy /32 zamiast jednego /24.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
set policy-options policy-statement ACCEPT-LAN term EXACT from route-filter 10.10.10.0/24 exact
set policy-options policy-statement ACCEPT-LAN term EXACT then accept
set policy-options policy-statement ACCEPT-LAN term REST then reject
~~~

Teraz zobacz różnicę między `prefix-list` a `prefix-list-filter`. Zwykła `prefix-list`, użyta jako warunek routingu, dopasowuje enumerowane prefiksy dokładnie; `prefix-list-filter LIST orlonger` pozwala wymagać określonej relacji długości. `route-filter` jest wygodny do pojedynczych prefiksów i kombinacji. [S7]

### 8.6.2. BGP route filtering nie jest ACL na adres źródłowy pakietu

W match criteria routing policy `route-filter 10.50.0.0/16 ...` dotyczy **adresu sieci w obiekcie route**. Nie „źródłowego IP użytkownika”, który wysyła TCP SYN. Jeśli chcesz filtrować pakiety źródłowe od hostów, przejdź do 8.7. Nazwy składni są podobne, obiekty przetwarzania zupełnie różne.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
set policy-options prefix-list ENTERPRISE-PREFIXES 10.10.10.0/24
set policy-options prefix-list ENTERPRISE-PREFIXES 10.10.11.0/24
set policy-options policy-statement EXPORT-ENTERPRISE term LAN from protocol direct
set policy-options policy-statement EXPORT-ENTERPRISE term LAN from prefix-list ENTERPRISE-PREFIXES
set policy-options policy-statement EXPORT-ENTERPRISE term LAN then accept
set policy-options policy-statement EXPORT-ENTERPRISE term REST then reject
~~~

Wszystkie warunki `from` w termie muszą pasować, a listę interpretujemy jako zbiór dopuszczonych prefiksów. Utrzymując whitelist, najpierw sprawdź, czy same prefiksy istnieją w RIB i są aktywne.

### 8.6.3. Najmniej intuicyjny szczegół: „route-filter longest match”

Załóżmy, że w **tym samym termie** umieszczono:

- `10.0.0.0/8 orlonger`;
- `10.10.0.0/16 exact`.

Do policy wchodzi route `10.10.20.0/24`. Inżynier może oczekiwać dopasowania pierwszej reguły `/8 orlonger`. Dokumentacja Juniper opisuje jednak, że parser routingu wybiera w tym termie **najbardziej szczegółowy kandydat route-filter** i sprawdza jego match type. Najbardziej szczegółowy to /16, lecz `exact` nie pasuje do /24. W efekcie term może **nie dopasować**, mimo że szerszy /8 orlonger wygląda na pasujący. Rozwiązaniem jest rozdzielenie wzorców na termy lub rozważenie `walkup` zgodnie z dokumentacją i wsparciem wersji.

~~~mermaid
flowchart TD
  C["Candidate route: 10.10.20.0/24"] --> F["route-filter tree"]
  F --> L["Najdłuższy kandydat: 10.10.0.0/16 exact"]
  L --> N{"Czy /24 to dokładnie /16?"}
  N -->|nie| MISS["Term NO MATCH"]
  MISS --> NEXT["Kolejny term / następna policy / default"]
  F -. "10.0.0.0/8 orlonger nie musi być oceniony" .-> MISS
~~~

**Interpretacja:** to inny longest-match niż forwarding packet lookup, choć także wykorzystuje drzewo prefiksów. Testuj przykłady osobno; prosta „tablica reguł z góry na dół” nie opisuje wszystkich aspektów route-filter. [S8]

### 8.6.4. Actions: modyfikacja i sterowanie przepływem

Oprócz `accept`/`reject` i `next term`/`next policy` działania mogą zmieniać atrybuty tras: `metric` (np. MED w BGP), `local-preference` (BGP), `community add/set/delete`, `as-path-prepend`, `next-hop`, `preference` i inne. Nie każdy action ma sens w każdej rodzinie i kierunku; część jest protokół-specyficzna.

**Urządzenie: R2-vSRX — Configuration Mode (model dla dwóch eBGP peerów, nie do baseline LAB 8.14)**

~~~junos
set policy-options policy-statement FROM-PRIMARY term BRANCH from route-filter 10.10.10.0/24 exact
set policy-options policy-statement FROM-PRIMARY term BRANCH then local-preference 200
set policy-options policy-statement FROM-PRIMARY term BRANCH then accept
set policy-options policy-statement FROM-PRIMARY term REST then reject
~~~

**Dlaczego?** Chcemy preferować trasę przyjętą od jednego źródła w ramach BGP. Nie robimy tego przez zmniejszenie metric OSPF ani przez `then preference 200` — to inne mechanizmy. W przypadku dwóch ścieżek sprawdź best path i active RIB route.

---

## 8.7. Firewall filters — koncepcje i cele. [A]

### 8.7.1. Dwie sytuacje z NOC, ten sam „filter”, inna technologia

W pierwszej sytuacji ISP reklamuje prefiks `0.0.0.0/0`, którego nie chcemy. Modyfikujesz **routing policy**.

W drugiej sytuacji host `10.10.10.50` wysyła nieuprawnione TCP/22 do IP routera. Chcesz ten pakiet zatrzymać zanim trafi do usługi zarządzającej. Modyfikujesz **firewall filter** albo dodatkowy mechanizm ochrony RE/host-inbound, zgodnie z platformą.

Junos firewall filters są zazwyczaj **stateless**: opisują dopasowanie nagłówka każdego pakietu i czynność. Mogą chronić RE, filtrować ruch przez interfejs, zliczać zdarzenia, oznaczać forwarding-class, kierować do specjalnego kontekstu FBF (rozdział 7), pomagać w CoS i ograniczaniu. To znacznie więcej niż „blokowanie portów”.

**Most z Cisco:** to bliższe extended ACL na interfejsie, ACL do CoPP/control-plane albo ACL użytej przez PBR niż do pełnego stanowego Cisco FTD. Ale nawet tu nie ma mapowania 1:1: term-based processing, rodziny `inet/inet6/ethernet-switching` i ograniczenia ASIC różnią się od IOS XE.

### 8.7.2. SRX ma jeszcze jeden etap

Na vSRX stateful flow processing sprawdza bezpieczeństwo nowej sesji na podstawie stref i security policies. Poprawny `then accept` w stateless firewall filter **nie oznacza** „wymuś permit przez security policy”. Podobnie `security policy permit` nie gwarantuje, że wcześniejszy interface filter czy uRPF nie zatrzyma pakietu. RE-bound BGP potrzebuje odpowiedniej konfiguracji `host-inbound-traffic protocols bgp` na strefie, a tranzyt między strefami potrzebuje dopasowanej security policy; te funkcje nie są zamienne. [S9], [S14]

~~~mermaid
flowchart TB
  P["Pakiet z hosta"] --> IF["Ingress family inet firewall filter"]
  IF -->|discard| STOP["Drop; licznik termu"]
  IF -->|accept| NEXT["Pozostałe etapy pipeline"]
  NEXT --> FLOW["SRX flow/session/zone/security policy"]
  FLOW -->|deny| DENY["Drop mimo filter accept"]
  FLOW -->|permit| ROUTE["Forwarding / egress"]
~~~

**Interpretacja:** „accept” w filtrze jest lokalną decyzją filtra, nie nakazem globalnego przepuszczenia niezależnie od reszty systemu. Nie wyciągaj z tego diagramu sztywnej kolejności sprzętowego packet processing dla każdego EX/QFX/MX/SRX.

---

## 8.8. Struktura filtrów: terms, match criteria, actions. [A]

### 8.8.1. Zobaczmy konkretny filtr

W oddziale mamy LAN `10.10.10.0/24`. Chcemy obserwować ICMP z PC-A do PC-B i odrzucać pakiety z testowej, spreparowanej podsieci źródłowej `203.0.113.0/24`; każdy inny pakiet powinien przejść do kolejnych kontroli.

**Urządzenie: R1-vSRX — Configuration Mode, wyłącznie izolowane LAB**

~~~junos
set firewall family inet filter LAN-IN term SPOOF from source-address 203.0.113.0/24
set firewall family inet filter LAN-IN term SPOOF then count SPOOF-DROPS
set firewall family inet filter LAN-IN term SPOOF then discard
set firewall family inet filter LAN-IN term ICMP-OBSERVE from protocol icmp
set firewall family inet filter LAN-IN term ICMP-OBSERVE then count ICMP-SEEN
set firewall family inet filter LAN-IN term ICMP-OBSERVE then accept
set firewall family inet filter LAN-IN term PASS-OTHER then count OTHER-SEEN
set firewall family inet filter LAN-IN term PASS-OTHER then accept
set interfaces ge-0/0/2 unit 0 family inet filter input LAN-IN
~~~

**Dlaczego ten porządek?** Odrzucamy spoofed source w pierwszym termie, zanim dojdziemy do szerokiego match ICMP. Drugi term obserwuje pozostałe ICMP. Ostatni term jest jawnym „allow other”, żeby nie uszkodzić zwykłych testów routingu.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show configuration firewall family inet filter LAN-IN | display set
show configuration interfaces ge-0/0/2 | display set
show firewall filter LAN-IN
show interfaces ge-0/0/2 extensive
~~~

Spodziewamy się osobnych liczników. Na części platform licznik nazywa się inaczej po kompilacji filtra lub ma suffix; najlepszym dowodem jest dostępność filter/counter na konkretnym interfejsie i zmiana wartości w dwóch pomiarach, a nie sama obecność tekstu konfiguracji.

### 8.8.2. Term bez `from` pasuje wszystkim

W termie `PASS-OTHER` nie ma `from`. Jest to celowe: match-all. Jeśli jednak przeniesiesz go **nad SPOOF**, każdy pakiet będzie zaakceptowany w pierwszym dopasowanym termie, a dwa następne staną się martwe. Terminologia z Cisco to **shadowing** — ale w Junos musisz też pamiętać o pozornie niewinnym nonterminating action.

### 8.8.3. Gotcha: samo `then count` nie oznacza „tylko policz i idź dalej”

Dokumentacja Juniper mówi rzecz zaskakującą: w standardowym stateless firewall filter działanie nonterminating `count` lub `log` bez explicit terminating action może oznaczać **niejawne accept** dla pasującego pakietu. Jeśli chcesz liczyć i dalej testować następne termy, użyj wspieranego `then next term` zamiast polegać na domyślności.

**Urządzenie: R1-vSRX — Configuration Mode, eksperyment tylko w odrębnym filtrze testowym**

~~~junos
set firewall family inet filter COUNT-TRAP term FIRST from protocol icmp
set firewall family inet filter COUNT-TRAP term FIRST then count ALL-ICMP
set firewall family inet filter COUNT-TRAP term SECOND from source-address 203.0.113.0/24
set firewall family inet filter COUNT-TRAP term SECOND then discard
set firewall family inet filter COUNT-TRAP term REST then accept
~~~

Pierwszy term może zakończyć się implicit accept i sprawić, że drugi nigdy nie dostanie spreparowanego ICMP. To pułapka szczególnie dla osób, które przynoszą z Cisco myśl, że `log` i `count` są jedynie dodatkowymi atrybutami ACL. **Nie stosuj tego filtra na produkcji.** [S10]

Praktyczna naprawa: jeśli najpierw chcemy zliczyć każdy ICMP, ale sprawdzić spoof później, do termu FIRST dodajemy explicit `then next term`, o ile jest wspierane przez konkretny OS/platformę i jego miejsce w filtrze. Jeszcze czytelniejszy projekt: ustawiaj jawne `accept`/`discard` w każdym finalnym termie i licz w tym samym termie.

---

## 8.9. Wpływ firewall filters na pakiety. [A]

### 8.9.1. Od momentu przyjścia ramki po drop

Router nie ocenia firewall filter jak listy tras w RIB. Jego dataplane ma nagłówek pakietu i wykonuje dopasowania source/destination IP, protocol, port lub innych pól zależnych od rodziny i platformy. Pasujący term może przyjąć, odrzucić cicho (`discard`), odrzucić z odpowiedzią (`reject`; szczegóły zależą od rodzin/protokołów/platformy), policzyć, zalogować lub skierować pakiet w inny sposób.

W typowym stateless filter bez końcowego termu akceptującego obowiązuje **implicit discard**. Jest to *odmienne* od routingu BGP policy, w którym w razie braku jawnego dopasowania zaczynają działać reguły default protocol policy. Wielu Cisco engineers sprowadza obie rzeczy do jednego „implicit deny”; potem tracą godzinę przy BGP.

### 8.9.2. Ingress a egress

Filtr `input` sprawdza pakiet wchodzący na logiczny interfejs/family, `output` pakiet wychodzący. To dwa możliwe miejsca egzekwowania polityki. Filter na `lo0` jest klasycznym narzędziem ochrony RE; nie traktuj go jako pełnego firewall dla wszystkich ruchów tranzytowych przez urządzenie. Ograniczenia i wyjątki dla vSRX, MX, EX/QFX oraz `inet`/`inet6` są platformowo zależne.

**Urządzenie: R1-vSRX — Configuration Mode, filtr IPv6 odrębny od IPv4**

~~~junos
set firewall family inet6 filter V6-LAN-IN term MONITOR-V6 from next-header icmp6
set firewall family inet6 filter V6-LAN-IN term MONITOR-V6 then count V6-ICMP
set firewall family inet6 filter V6-LAN-IN term MONITOR-V6 then accept
set firewall family inet6 filter V6-LAN-IN term REST then accept
set interfaces ge-0/0/2 unit 0 family inet6 filter input V6-LAN-IN
~~~

IPv4 `family inet` nie obejmuje IPv6; prefix w `inet.0` nie przechodzi do `inet6.0`. Dla ICMPv6 i ND sprawdź, czy nie blokujesz komunikatów niezbędnych dla Neighbor Discovery. **W praktyce filtr ochronny IPv6 musi świadomie uwzględnić control/ND traffic**, nie jest mechanicznym copy/paste z IPv4.

### 8.9.3. Kiedy filter permit nie pomaga

R1 ma działającą BGP route do serwera. `show firewall filter LAN-IN` zwiększa `OTHER-SEEN`, więc pakiet nie został odrzucony *przez ten filtr*. Mimo to nie ma połączenia. Na SRX sprawdź:

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show security policies from-zone LAN to-zone WAN
show security flow session source-prefix 10.10.10.10
show security flow session destination-prefix 10.20.20.20
show security zones
show route 10.20.20.0/24 exact
~~~

Jeśli brak sesji, security policy hit albo inną przeszkodę pokaże diagnostyka flow/policy; możliwy jest też brak return route, NAT, application mismatch lub host firewall. Przestrzegaj rozdzielenia hipotez, nie „naprawiaj ACL” na oślep.

---

## 8.10. Unicast Reverse Path Forwarding (uRPF). [A]

### 8.10.1. Dlaczego sprawdzamy drogę **do źródła**?

Router odbiera pakiet ze źródłem `10.10.10.50` na interfejsie WAN, choć `10.10.10.0/24` to sieć LAN. Klasyczny destination routing mógłby wiedzieć, jak dostarczyć pakiet do celu, ale sam adres źródłowy sugeruje spoofing. **uRPF** pyta o route prowadzący *wstecz* do source IP.

- **Strict:** dla źródła pakietu oczekiwany reverse path powinien prowadzić przez interfejs przyjścia (zależnie od ECMP/platformy i aktywnego forwarding). W odgałęzieniach z symetrycznymi ścieżkami jest to świetne zabezpieczenie.
- **Loose:** wystarczy istnienie kwalifikowanego reverse route do źródła, niezależnie od dokładnego interfejsu przyjścia; implementacja i interpretacja default route mogą mieć niuanse platformowe.

**Problem asymetrii:** mamy dwa ISP, z ISP-B pakiet wpada, a najlepsza trasa **do źródła** prowadzi przez ISP-A. Strict uRPF odrzuci nawet legalny ruch. Loose może przepuścić go, ale ma słabsze właściwości anty-spoofingowe. W realnym enterprise HA/WAN asymetria nie jest rzadkością.

~~~mermaid
flowchart TD
  P["Pakiet ingress ISP-B, source S"] --> U["uRPF lookup source S"]
  U --> FOUND{"Czy znamy reverse path?"}
  FOUND -->|nie| DROP["FAIL: drop / fail-filter"]
  FOUND -->|tak| MODE{"Tryb"}
  MODE -->|loose| PASS["PASS: source reachable"]
  MODE -->|strict| SAME{"Czy reverse interface = ISP-B?"}
  SAME -->|tak| PASS
  SAME -->|nie| DROP
~~~

**Interpretacja:** to nie jest sprawdzanie, czy destination route prowadzi do ISP-B. Zobacz, jak bardzo inna jest oś oceny! Jeżeli twój routing opiera się na FBF, routing instances lub ECMP, koniecznie sprawdź szczegółową obsługę uRPF w danym PFE. [S11]

### 8.10.2. Konfiguracja i ograniczenia

**Urządzenie: MX-EDGE-LAB — Configuration Mode, przykład schematu strict uRPF na wspieranej platformie**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet rpf-check
~~~

**Urządzenie: MX-EDGE-LAB — Configuration Mode, przykład loose uRPF, po weryfikacji wsparcia w release**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet rpf-check mode loose
~~~

**Urządzenie: MX-EDGE-LAB — Operational Mode**

~~~junos
show interfaces ge-0/0/0 extensive
show route <SOURCE-IP>
show route forwarding-table destination <SOURCE-IP>
~~~

W powyższym bloku `<SOURCE-IP>` jest placeholderem, nie komendą do wklejenia dosłownie. Dla uRPF zweryfikuj nie tylko konfigurację, ale rzeczywiste wsparcie konkretnego modelu, wydania, rodziny i trybu ECMP. **Domyślnie uRPF jest wyłączony**. Na części urządzeń brak `mode loose` oznacza strict, lecz nie narzucaj identycznego modelu wszystkim EX/QFX/vSRX. Opcjonalny `fail-filter` może zmienić zachowanie po niezdanym check; bez niego domyślnie drop. [S11], [S17]

**Most z Cisco:** `ip verify unicast source reachable-via rx` odpowiada intuicji strict uRPF, `reachable-via any` loose. Jednak obsługa default route, vrf, multicast, ECMP, hardware i exceptions wymaga osobnej weryfikacji.

**Fault injection:** na izolowanym routerze ze znaną powrotną ścieżką zmień tylko to, skąd przychodzi testowy source; nie testuj spoofingu na produkcyjnych łączach. Po awarii najpierw pokaż reverse route i interface, potem uRPF counters, dopiero później zmieniaj mode.

---

## 8.11. Konfiguracja i monitoring podstawowych routing policies i firewall filters. [A]

### 8.11.1. Procedura od wymagania biznesowego do dowodu

Założenie: operator ma otrzymać tylko LAN `10.10.10.0/24` z oddziału. Dodatkowo na wejściu z testowego LAN chcemy odrzucać spoofed source `203.0.113.0/24`. Traktujemy to jako dwa osobne wymagania i dwa mechanizmy:

- Control plane: **BGP export whitelist**.
- Dataplane: **interface input stateless filter**.
- Na vSRX trzeci warunek: **security zones / policies**.

Przed zmianą zapisz stan: active route, sąsiad BGP, lista advertised routes, istniejące interface filters, zones oraz policy counters. W dobrym change request obowiązujący default deny/permit musi być jawny. W przeciwnym razie możesz „naprawić” transmisję poprzez niezamierzone otwarcie nowego ruchu.

### 8.11.2. Minimalny pakiet obserwacji

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show version | match Junos
show interfaces terse
show route 10.10.10.0/24 exact
show bgp summary
show route advertising-protocol bgp 192.0.2.2
show configuration protocols bgp | display set
show configuration policy-options | display set
show configuration firewall | display set
show configuration interfaces ge-0/0/2 | display set
show firewall filter LAN-IN
show security policies
show security flow session summary
~~~

Nie wystarczy jeden `show configuration`. Jeśli filter jest zdefiniowany, ale nie przypięty do interfejsu, nie chroni określonego punktu przepływu. Jeśli policy jest zdefiniowana, ale BGP nie referencjonuje jej w grupie/peerze, nie wpływa na eksport. Jeśli route nie istnieje jako aktywna w RIB, żadna atrakcyjna export policy nie wyczaruje jej znikąd.

### 8.11.3. Kontrola zmiany

**Urządzenie: R1-vSRX — Configuration Mode, izolowany LAB z konsolą**

~~~junos
show | compare
commit check
commit confirmed 5
~~~

W nowej sesji operational (lub po wyjściu z config) sprawdź ruch. Zatwierdź zmianę świadomie **przed upływem** okresu confirmed commit, tylko po PASS.

**Urządzenie: R1-vSRX — Configuration Mode, po pozytywnym postcheck**

~~~junos
commit
~~~

Jeśli testy nie przejdą, nie potwierdzaj zmian i pozwól na automatyczne cofnięcie albo wróć do zapisanej konfiguracji przez udokumentowany rollback. **Gotcha z rozdziału 3:** dodatkowy `commit check` podczas oczekiwania na `commit confirmed` nie jest neutralną komendą diagnostyczną — może skutkować potwierdzeniem oczekującej zmiany.

### 8.11.4. Monitorowanie: dowód dodatni i ujemny

Dla whitelisty BGP dowód dodatni: `10.10.10.0/24` jest w advertised routes R1 i received/active na R2. Dowód ujemny: `198.51.100.0/24` (celowo nieautoryzowany test) **nie** jest reklamowany ani aktywny u sąsiada. Dla firewalla dowód dodatni: ruch legalny przechodzi i `OTHER-SEEN` rośnie; ujemny: spreparowany ruch jest blokowany, `SPOOF-DROPS` rośnie, a odbiorca go nie widzi.

**Uwaga diagnostyczna:** liczniki mogą być zerowane przy zmianach i różnić się na sprzęcie; porównuj dwa odczyty z odstępem, uwzględniaj hardware installation i czas propagacji. „Counter 0” po sekundzie nie dowodzi, że filtr nie działa. [S10], [S16]

---

## 8.12. Routing policy applied to OSPF, IS-IS and BGP. [S]

### 8.12.1. OSPF: export to redistribution, nie automatyczne tłumienie flooding

Rozważ router ABR/ASBR na brzegu LAN. Klient chce rozgłosić w OSPF jedną trasę statyczną `10.70.70.0/24`, ale nie default route. W Junos użyj export policy:

**Urządzenie: R-OSPF-EDGE — Configuration Mode, osobny LAB z już działającym OSPF**

~~~junos
set routing-options static route 10.70.70.0/24 discard
set policy-options policy-statement OSPF-STATIC-ONLY term STATIC-70 from protocol static
set policy-options policy-statement OSPF-STATIC-ONLY term STATIC-70 from route-filter 10.70.70.0/24 exact
set policy-options policy-statement OSPF-STATIC-ONLY term STATIC-70 then metric 50
set policy-options policy-statement OSPF-STATIC-ONLY term STATIC-70 then accept
set policy-options policy-statement OSPF-STATIC-ONLY term REST then reject
set protocols ospf export OSPF-STATIC-ONLY
~~~

Sama `static discard` jest tu **kontrolowanym źródłem prefiksu do nauki redistribution**, nie drogą do serwerów; gdy ktoś spróbuje wysłać pakiet do tego subnetu bez bardziej szczegółowej trasy, zostanie odrzucony. W produkcji najpierw wymagaj rzeczywistej drogi, nie twórz blackhole tylko po to, aby uzyskać LSA.

**Urządzenie: R-OSPF-EDGE — Operational Mode**

~~~junos
show route 10.70.70.0/24 exact
show ospf neighbor
show ospf database external
show configuration protocols ospf | display set
~~~

Oczekujesz aktywnego static i external LSA w odpowiednim OSPF context, jeśli route jest kwalifikowany do redystrybucji. Jeśli nie ma sąsiadów ani area, samo `set protocols ospf export` nic nie dowodzi. Jeśli masz OSPF import policy, **jej zakres obejmuje external routes; nie oczekuj, że odrzuci normalne internal OSPF topology**.

### 8.12.2. IS-IS: ten sam zamiar, inne środowisko

IS-IS jest link-state, więc odpowiednio rozróżniaj wewnętrzne LSP flooding od redistribution. W campus core możesz potrzebować ogłaszać do IS-IS tylko wybrany static summary. Polityka z `from protocol static`, `route-filter ... exact` i `then accept` jest podpinana jako `protocols isis export` — pod warunkiem, że interfejsy/level/family IS-IS są poprawnie skonfigurowane i wspierane.

**Urządzenie: R-ISIS-CORE — Configuration Mode, tylko dodatkowe ćwiczenie z istniejącym IS-IS**

~~~junos
set policy-options policy-statement ISIS-EXPORT term LAB-STATIC from protocol static
set policy-options policy-statement ISIS-EXPORT term LAB-STATIC from route-filter 10.80.80.0/24 exact
set policy-options policy-statement ISIS-EXPORT term LAB-STATIC then accept
set policy-options policy-statement ISIS-EXPORT term REST then reject
set protocols isis export ISIS-EXPORT
~~~

**Urządzenie: R-ISIS-CORE — Operational Mode**

~~~junos
show isis adjacency
show isis database extensive
show route 10.80.80.0/24 exact
show configuration protocols isis | display set
~~~

Rozdział 10 pokaże poziomy L1/L2, metric style, overload i właściwe warunki redistribucji. W rozdziale 8 uczymy się **punktu wywołania policy i diagnostyki akceptacji prefiksu**, nie całego IS-IS.

### 8.12.3. BGP: import i export na właściwym peerze

BGP umie stosować policy na poziomie globalnym, group i peer, z odpowiednim zakresem. W operator/WAN edge nie polegaj na niejawnej akceptacji. Zdefiniuj konkretne listy i zakończ `then reject`. Nie zapominaj, że BGP active-route eligibility oraz iBGP split horizon/route reflection pozostają w mocy.

**Urządzenie: R1-vSRX — Configuration Mode, bazowy przykład eBGP**

~~~junos
set policy-options policy-statement EBGP-OUT term LAN from protocol direct
set policy-options policy-statement EBGP-OUT term LAN from route-filter 10.10.10.0/24 exact
set policy-options policy-statement EBGP-OUT term LAN then accept
set policy-options policy-statement EBGP-OUT term REST then reject
set protocols bgp group TO-ISP export EBGP-OUT
~~~

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show bgp neighbor 192.0.2.2
show route advertising-protocol bgp 192.0.2.2
show route receive-protocol bgp 192.0.2.2 hidden
~~~

Gdy w BGP policy odrzucisz trasę *przychodzącą*, badanie hidden routes jest naturalne. Gdy odrzucasz *wychodzącą*, patrz na advertised routes lokalnie i received routes peer-side. Kierunek diagnostyki ma znaczenie.

### 8.12.4. Gotcha: wartości domyślne są inne dla różnych protokołów

Kontrast, który zapamiętasz na JNCIS: BGP default export może legalnie reklamować aktywne BGP routes, ale OSPF/IS-IS export domyślnie **nie redystrybuuje static/connected**. Gdy w OSPF widzisz local interface subnet w LSDB bez żadnej export policy, nie jest to sprzeczność — to podstawowy mechanizm link-state. [S2], [S12]

---

## 8.13. Zaawansowane zastosowania routing policy w OSPF/IGP i BGP. [P]

### 8.13.1. Od whitelist do traffic engineering

Na poziomie JNCIP nie wystarcza znajomość składni. Musisz umieć odpowiedzieć: czy zmiana jest inbound czy outbound, czy modyfikujemy BGP best path czy multi-protocol route preference, czy społeczność BGP ma znaczenie lokalne czy u sąsiadów i czy backup naprawdę jest backupem?

Scenariusz: Site A ma dwa uplinki WAN (ISP-PRIMARY, ISP-BACKUP), a organizacja chce, aby **ruch wychodzący** preferował Primary, a **ruch przychodzący** w miarę możliwości również wykorzystywał Primary. To dwa różne zadania:

- **Outbound od nas**: import policy z `local-preference 200` dla Primary i 100 dla Backup. LOCAL_PREF jest propagowany w obrębie AS przez iBGP (zależnie od policy), nie służy wprost do wymuszenia decyzji obcego AS.
- **Inbound do nas**: export policy może używać `as-path-prepend` na backupie, społeczności uzgodnionych z operatorem lub MED (o ile odbiorca go uwzględnia). Nie ma gwarancji, że Internet „posłucha” wskazania preferencji.

W Cisco zrobiłbyś route-maps, w Junos są policy-statements. **Zmiana nazwy nie zmienia fizyki polityki między AS.**

### 8.13.2. Przykład BGP community: oznaczamy, niekoniecznie blokujemy

W enterprise możesz chcieć oznaczyć prefix, że pochodzi z oddziału o ograniczonym zaufaniu. W Junos community można zdefiniować jako named policy object i dodać w action. Samo nadanie community nic nie blokuje: to metadana, którą musi zinterpretować inna policy/protokół/peer.

**Urządzenie: R1-vSRX — Configuration Mode, przykład rozszerzający na działającym BGP**

~~~junos
set policy-options community SITE-BRANCH members 64510:100
set policy-options policy-statement MARK-OUT term BRANCH from route-filter 10.10.10.0/24 exact
set policy-options policy-statement MARK-OUT term BRANCH then community add SITE-BRANCH
set policy-options policy-statement MARK-OUT term BRANCH then next policy
set policy-options policy-statement FILTER-OUT term BRANCH from route-filter 10.10.10.0/24 exact
set policy-options policy-statement FILTER-OUT term BRANCH then accept
set policy-options policy-statement FILTER-OUT term REST then reject
set protocols bgp group TO-ISP export [ MARK-OUT FILTER-OUT ]
~~~

Użycie `next policy` jest tu zamierzone: przekazujemy zmodyfikowany route do ostatecznej whitelisty. Pomyłką byłoby zatrzymanie się na `then accept` w MARK-OUT i przekonanie, że FILTER-OUT jeszcze sprawdzi trasę.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route advertising-protocol bgp 192.0.2.2 detail
show configuration policy-options community SITE-BRANCH | display set
~~~

Na routerze odbierającym sprawdź, czy community rzeczywiście przyszło oraz czy jego `import` potrafi ją dopasować. Pamiętaj o rozróżnieniu standardowej community i extended/large communities; format i support zależą od konfiguracji. 

### 8.13.3. Import lokalny z community: nie zamieniaj opisu na decyzję

**Urządzenie: R2-vSRX — Configuration Mode, opcjonalny scenariusz**

~~~junos
set policy-options community SITE-BRANCH members 64510:100
set policy-options policy-statement BRANCH-IN term TRUSTED-TAG from community SITE-BRANCH
set policy-options policy-statement BRANCH-IN term TRUSTED-TAG then local-preference 180
set policy-options policy-statement BRANCH-IN term TRUSTED-TAG then accept
set policy-options policy-statement BRANCH-IN term REST then reject
~~~

To **nie jest automatycznie bezpieczna whitelist**. Neighbor może teoretycznie wysłać inne NLRI z tym samym tagiem. W produkcji łącz warunek community z kontrolą prefiksu i AS path, a w zależności od relacji biznesowej egzekwuj maksymalną liczbę prefiksów. Community dodane samodzielnie nie jest dowodem autoryzacji sieci.

### 8.13.4. AS-PATH i MED — co wygrywa, a co nie

AS-PATH prepend na eBGP export zwiększa postrzeganą długość drogi przez zewnętrznych odbiorców, ale oni mogą stosować LOCAL_PREF, routing policies lub ignorować MED. MED jest wskazówką (często w obrębie połączeń z tym samym sąsiednim AS), a nie twardą dyrektywą. W Junos `then metric` dla BGP to MED, lecz w OSPF może oznaczać metrykę redystrybuowanej trasy — **nazwa action wspólna, semantyka protokołowa różna**.

Warto też rozróżnić `preference` jako Junos route preference (porównanie dróg między źródłami routingu) od BGP `local-preference` (atrybut BGP). Na JNCIP bardzo dużo błędnych zmian polega na poprawnym wpisaniu **niewłaściwego pokrętła**.

### 8.13.5. OSPF/IGP w środowisku wielu domen

Do OSPF eksportuj tylko prefiksy świadomie wprowadzone z innych domen. Zbyt szeroka policy `from protocol static then accept` może wprowadzić default route, testowe discard prefixes lub ścieżki o niepożądanym zasięgu. Projekt powinien określać, czy route ma być OSPF external, jaką metric i external type otrzyma oraz jak będzie filtrowana w innych miejscach.

IS-IS podobnie: import/export policies nie są magicznym substytutem projektowania L1/L2, summarization ani rozumienia LSP flooding. „Nie widzę prefiksu w RIB” nie dowodzi, że sieć nie ma informacji LSDB o drodze.

### 8.13.6. Złożony incident — rozbijamy go na dwa niezależne defekty

**SYMPTOM:** po migracji WAN R2 widzi `10.10.10.0/24` jako hidden, a PC-B nie widzi PC-A. Zespół twierdzi, że zawiódł BGP.

**HIPOTEZA A:** import policy R2 odrzuca poprawny prefiks z powodu `route-filter` typu `exact` versus `orlonger` lub błędnej kolejności termów.

**TEST A — Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show route receive-protocol bgp 192.0.2.1 hidden
show configuration policy-options | display set
show configuration protocols bgp | display set
show route 10.10.10.0/24 exact extensive
~~~

**WYNIK symulowany:** R2 widzi received hidden i adnotację o odrzuceniu; brak aktywnej drogi do `10.10.10.0/24`. **ROOT CAUSE A:** inbound deny (albo route ineligibility potwierdzone w output). **NAPRAWA:** korekta konkretnego termu/prefixu i walidacja.

Po naprawie BGP route jest aktywny, ale ping nadal nie działa. To **nowy fakt**, nie dowód, że poprawka A była bezużyteczna.

**HIPOTEZA B:** ingress firewall filter na R1 odrzuca źródło klienta lub SRX nie przepuszcza sesji.

**TEST B — Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route 10.20.20.0/24 exact
show firewall filter LAN-IN
show security policies from-zone LAN to-zone WAN
show security flow session source-prefix 10.10.10.10
~~~

**WYNIK symulowany:** `SPOOF-DROPS` nie rośnie, lecz nie ma dopasowanej security policy z LAN do WAN. **ROOT CAUSE B:** stateful deny po stronie SRX. **NAPRAWA:** minimalna policy ze zgodnymi strefami i aplikacją, następnie test całego flow i return traffic. Tak właśnie wygląda dojrzały troubleshooting — nie jedna wielka hipoteza „routing”.

---

## 8.14. [LAB] Import/export BGP, filtrowanie prefiksów i weryfikacja ścieżki polityki

### 8.14.1. Cel i granice laboratorium

To laboratorium **nie jest powtórką rozdziału 6**. Routing statyczny służy jedynie stworzeniu testowego prefiksu nieautoryzowanego do export. Koncentrujemy się na: co jest w RIB, co zostaje wybrane do eksportu, co jest widoczne na peerze, co inbound policy zachowa jako hidden oraz jak odróżnić packet filter od security policy na vSRX.

**Status:** PLANNED / NOT RUNTIME VALIDATED. Przed wykonaniem trzeba potwierdzić mapowanie wirtualnych NIC na `ge-*`, dostępność `fxp0`/management, wersję Junos i licencję/funkcje BGP. Nie zakładaj, że `ge-0/0/1` i `ge-0/0/2` wskazują konkretne karty EVE-NG przed `show interfaces terse`. Nie podłączaj laboratorium do produkcji.

**Topologia:** 2× vSRX + 2× Linux/VPCS; dla IPv6 i wygodnych testów spoof Linux jest zalecany. EVE-NG: trzy odizolowane segmenty L2. Bez cloud/Internet bridge.

~~~mermaid
flowchart LR
  PC1["PC-A Linux\n10.10.10.10/24"] --- LAN1["Segment LAN-A"]
  LAN1 --- R1["R1-vSRX\nAS 64510"]
  R1 --- WAN["Segment WAN\n192.0.2.0/30"]
  WAN --- R2["R2-vSRX\nAS 64520"]
  R2 --- LAN2["Segment LAN-B"]
  LAN2 --- PC2["PC-B Linux\n10.20.20.20/24"]
~~~

**Interpretacja:** jeśli BGP nie rozgłosi LAN-A, R2 nie musi znać return route do PC-A. Jeśli BGP działa, pakiet tranzytowy nadal musi przejść przez vSRX security zones i policies. BGP control traffic trafia do RE i wymaga host-inbound allow na interface WAN.

Plan adresacji:

| Element | Interfejs logiczny w Junos (sprawdź mapping!) | IPv4 | Rola |
|---|---|---|---|
| R1 WAN | `ge-0/0/1.0` | `192.0.2.1/30` | eBGP do R2 |
| R2 WAN | `ge-0/0/1.0` | `192.0.2.2/30` | eBGP do R1 |
| R1 LAN | `ge-0/0/2.0` | `10.10.10.1/24` | gateway PC-A |
| R2 LAN | `ge-0/0/2.0` | `10.20.20.1/24` | gateway PC-B |
| PC-A | `eth0` | `10.10.10.10/24` | GW 10.10.10.1 |
| PC-B | `eth0` | `10.20.20.20/24` | GW 10.20.20.1 |
| R1 TEST (nieautoryzowany) | static discard | `198.51.100.0/24` | próba kontrolowanego „leaku” |

RFC 5737 rezerwuje `192.0.2.0/24` oraz `198.51.100.0/24` jako adresy dokumentacyjne; w izolowanej topologii są bezpiecznymi przykładami. Lab nie powinien prowadzić ich do Internetu.

**Przed blokami „Configuration Mode” w laboratorium:** wchodzimy do trybu konfiguracji ze zwykłego promptu operational. Nie wpisujemy `configure private` *wewnątrz* trybu `[edit]`; dlatego polecenie inicjujące sesję jest wydzielone poniżej.

**Urządzenie: R1-vSRX — Operational Mode (wejście do Configuration Mode)**

~~~junos
configure private
~~~

**Urządzenie: R2-vSRX — Operational Mode (wejście do Configuration Mode)**

~~~junos
configure private
~~~

W dalszych blokach z nagłówkiem „Configuration Mode” zakładamy już aktywne `[edit private]`. Po `commit confirmed` można wykonać `run show ...` z konfiguracji albo wrócić do Operational Mode i następnie ponownie wejść do Configuration Mode; zawsze obserwuj prompt. Ta sama uwaga dotyczy przykładów z wcześniejszych sekcji, które pokazują fragmenty konfiguracji do zastosowania w gotowym `[edit]`.

### 8.14.2. Faza 0 — discovery bez zmian

**Cel:** upewnić się, że porty są te, które sądzisz, a config nie ma fabrycznej security policy przeszkadzającej w ćwiczeniach.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show version
show interfaces terse
show chassis hardware
show configuration interfaces | display set
show configuration security zones | display set
show configuration security policies | display set
show configuration protocols bgp | display set
show route
~~~

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show version
show interfaces terse
show chassis hardware
show configuration interfaces | display set
show configuration security zones | display set
show configuration security policies | display set
show route
~~~

**Oczekiwany rezultat:** rozpoznane interfejsy z poprawnym mapping, brak kolizji z istniejącymi adresami. **Jeśli tak nie jest:** zanotuj mapping, zamień nazwy w dalszym labie, nie nadpisuj konfiguracji, nie wykonuj destructive factory reset. Zapisz baseline `show configuration | display set` poza repo bez sekretów.

### 8.14.3. Faza 1 — adresacja, security zones i kontrolowany baseline

**Cel:** pozwolić komputerom na legalne testy tranzytowe, umożliwić control-plane BGP i zachować izolację od innych sieci.

**Urządzenie: R1-vSRX — Configuration Mode, odseparowany lab, po potwierdzeniu mapping**

~~~junos
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/30
set interfaces ge-0/0/2 unit 0 family inet address 10.10.10.1/24
set security zones security-zone WAN interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone WAN interfaces ge-0/0/1.0 host-inbound-traffic protocols bgp
set security zones security-zone LAN interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT match source-address any
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT match destination-address any
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT match application any
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT then permit
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN match source-address any
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN match destination-address any
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN match application any
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN then permit
show | compare
commit check
commit confirmed 5
~~~

**Urządzenie: R2-vSRX — Configuration Mode, po sprawdzeniu własnego mapping**

~~~junos
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.2/30
set interfaces ge-0/0/2 unit 0 family inet address 10.20.20.1/24
set security zones security-zone WAN interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone WAN interfaces ge-0/0/1.0 host-inbound-traffic protocols bgp
set security zones security-zone LAN interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT match source-address any
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT match destination-address any
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT match application any
set security policies from-zone LAN to-zone WAN policy LAB-LAN-OUT then permit
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN match source-address any
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN match destination-address any
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN match application any
set security policies from-zone WAN to-zone LAN policy LAB-WAN-IN then permit
show | compare
commit check
commit confirmed 5
~~~

**Uwaga bezpieczeństwa:** `match application any` oraz obustronne permit są **wyłącznie dla izolowanego LAB**, aby oddzielić routing od stateful policy. To nie jest template produkcyjny SRX. Na produkcji dopuszczamy tylko wymagane aplikacje/kierunki. Sprawdź istniejące security zones: dwa interfejsy nie mogą należeć jednocześnie do różnych stref; nazwa i domyślne strefy zależą od obrazu.

**Urządzenie: PC-A — Linux shell**

~~~bash
ip addr add 10.10.10.10/24 dev eth0
ip route replace default via 10.10.10.1
ping -c 3 10.10.10.1
~~~

**Urządzenie: PC-B — Linux shell**

~~~bash
ip addr add 10.20.20.20/24 dev eth0
ip route replace default via 10.20.20.1
ping -c 3 10.20.20.1
~~~

Jeśli adres istnieje z autokonfiguracji, użyj `ip addr show` i nie dodawaj go ponownie; wymagany jest właściwy adapter Linux. Rezultat: PC ping do własnej bramy, R1↔R2 ping po WAN, ale PC-A→PC-B jeszcze nie musi działać, ponieważ nie mamy BGP route.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
ping 192.0.2.2 source 192.0.2.1 count 3
show route 10.10.10.0/24 exact
show route 192.0.2.0/30 exact
show security zones
~~~

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
ping 192.0.2.1 source 192.0.2.2 count 3
show route 10.20.20.0/24 exact
show route 192.0.2.0/30 exact
~~~

Po wykonaniu postcheck w wymaganym czasie świadomie potwierdź każdy `commit confirmed`; jeżeli test nie przejdzie, pozwól rollback. **Nie wykonuj kolejnego `commit check` w okresie oczekiwania.**

**Urządzenie: R1-vSRX — Configuration Mode, wyłącznie po PASS**

~~~junos
commit
~~~

**Urządzenie: R2-vSRX — Configuration Mode, wyłącznie po PASS**

~~~junos
commit
~~~

### 8.14.4. Faza 2 — eBGP i kontrolowany eksport

**Cel:** zobaczyć różnicę między aktywną trasą direct w RIB, a realnym BGP announcement. Klasyczne ASN z prywatnego zakresu dokumentacyjnego wybrano tylko dla LAB.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
set routing-options autonomous-system 64510
set protocols bgp group TO-R2 type external
set protocols bgp group TO-R2 peer-as 64520
set protocols bgp group TO-R2 family inet unicast
set protocols bgp group TO-R2 neighbor 192.0.2.2
set policy-options policy-statement EXPORT-R1 term LAN-A from protocol direct
set policy-options policy-statement EXPORT-R1 term LAN-A from route-filter 10.10.10.0/24 exact
set policy-options policy-statement EXPORT-R1 term LAN-A then accept
set policy-options policy-statement EXPORT-R1 term REST then reject
set protocols bgp group TO-R2 export EXPORT-R1
show | compare
commit check
commit confirmed 5
~~~

**Urządzenie: R2-vSRX — Configuration Mode**

~~~junos
set routing-options autonomous-system 64520
set protocols bgp group TO-R1 type external
set protocols bgp group TO-R1 peer-as 64510
set protocols bgp group TO-R1 family inet unicast
set protocols bgp group TO-R1 neighbor 192.0.2.1
set policy-options policy-statement EXPORT-R2 term LAN-B from protocol direct
set policy-options policy-statement EXPORT-R2 term LAN-B from route-filter 10.20.20.0/24 exact
set policy-options policy-statement EXPORT-R2 term LAN-B then accept
set policy-options policy-statement EXPORT-R2 term REST then reject
set protocols bgp group TO-R1 export EXPORT-R2
show | compare
commit check
commit confirmed 5
~~~

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show bgp summary
show bgp neighbor 192.0.2.2
show route advertising-protocol bgp 192.0.2.2
show route 10.20.20.0/24 exact
show route forwarding-table destination 10.20.20.20
~~~

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show bgp summary
show route receive-protocol bgp 192.0.2.1
show route advertising-protocol bgp 192.0.2.1
show route 10.10.10.0/24 exact
~~~

**Oczekiwany wynik symulowany:** oba eBGP peers Established; R1 ogłasza tylko LAN-A; R2 ogłasza tylko LAN-B; każda strona otrzymuje cudzą sieć, lokalne direct subnet pozostają direct. PC-A↔PC-B powinny przejść przez obie security zones i dozwolone policy. Jeżeli nie, diagnozuj: interface, BGP NLRI, active route, security zones, return path i host firewall (nie tylko BGP status).

**Urządzenie: PC-A — Linux shell**

~~~bash
ping -c 3 10.20.20.20
traceroute -n 10.20.20.20
~~~

Na niektórych minimalnych Linux images polecenie `traceroute` nie jest preinstalowane. Można użyć `tracepath` lub zweryfikować inny dostępny tool — nie instaluj niczego przez firmową sieć.

Po PASS potwierdź `commit confirmed` na obu urządzeniach.

**Urządzenie: R1-vSRX — Configuration Mode, po PASS**

~~~junos
commit
~~~

**Urządzenie: R2-vSRX — Configuration Mode, po PASS**

~~~junos
commit
~~~

### 8.14.5. Faza 3 — fault injection: nieautoryzowany prefiks próbujący „uciec” do sąsiada

**Cel:** wykazać, że `EXPORT-R1` jest whitelistem, a BGP nie wysyła dowolnej aktywnej static route.

Najpierw R1 tworzy testowy `198.51.100.0/24` jako static discard. To bezpieczny typ dla izolowanego labu (ruch do adresów tej sieci miałby zostać odrzucony). Następnie sprawdzamy, czy po policy pojawia się w `advertising-protocol`.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
set routing-options static route 198.51.100.0/24 discard
show | compare
commit check
commit confirmed 5
~~~

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route 198.51.100.0/24 exact
show route advertising-protocol bgp 192.0.2.2
~~~

**Oczekiwany wynik:** testowa trasa jest aktywna locally; nie jest reklamowana, ponieważ `EXPORT-R1` akceptuje tylko direct LAN-A, a ostatni term odrzuca resztę. Wymagamy dowodu *obu faktów*. Jeśli trasy nie ma w RIB, to nie jest prawidłowy test eksportu.

### 8.14.6. Faza 4 — import whitelist i rejected/hidden routes

**Cel:** sprawdzić inbound policy od strony R2 i dowieść, że route może zostać otrzymany, lecz nie wejść do aktywnej RIB.

Na R1 tymczasowo rozszerzamy eksport o statyczny test prefix. Uwaga: nie zmieniamy broad match-all ani nie otwieramy pełnego BGP table.

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
set policy-options policy-statement EXPORT-R1 term TEST-ONLY from protocol static
set policy-options policy-statement EXPORT-R1 term TEST-ONLY from route-filter 198.51.100.0/24 exact
set policy-options policy-statement EXPORT-R1 term TEST-ONLY then accept
insert policy-options policy-statement EXPORT-R1 term TEST-ONLY before term REST
show | compare
commit check
commit confirmed 5
~~~

**Uwaga kolejności:** `insert` to operacja konfiguracji służąca przeniesieniu termu *przed* końcowy REST. Jeśli edytujesz kolejność inaczej, BGP export może nadal odrzucać TEST-ONLY. Zawsze obejrzyj wynik `show configuration policy-options policy-statement EXPORT-R1` przed zatwierdzeniem.

Na R2 tworzymy inbound whitelist tylko dla LAN-A:

**Urządzenie: R2-vSRX — Configuration Mode**

~~~junos
set policy-options policy-statement IMPORT-R1 term LAN-A from route-filter 10.10.10.0/24 exact
set policy-options policy-statement IMPORT-R1 term LAN-A then accept
set policy-options policy-statement IMPORT-R1 term DENY-OTHER then reject
set protocols bgp group TO-R1 import IMPORT-R1
show | compare
commit check
commit confirmed 5
~~~

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route advertising-protocol bgp 192.0.2.2
~~~

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show route receive-protocol bgp 192.0.2.1
show route receive-protocol bgp 192.0.2.1 hidden
show route 198.51.100.0/24 exact
show route 10.10.10.0/24 exact
~~~

**Oczekiwany wynik symulowany:** R1 wysyła LAN-A **i** TEST-ONLY. R2 przyjmuje LAN-A i odrzuca testową trasę import policy; testowy prefix może być widoczny jako hidden, ale nie jako aktywny route/FIB. **Root cause takiego braku trasy:** inbound policy, nie brak transmisji UPDATE. W zależności od release i `keep` sprawdzaj widok `hidden`.

**Fault injection alternatywny:** umieść końcowe `then reject` na R2 przed `LAN-A`. Przestanie działać cały import i PC-A/PC-B mogą stracić return route. **Naprawa:** przywróć kolejność, nie „podkręcaj” LOCAL_PREF.

### 8.14.7. Faza 5 — test filtrów stateless i SRX policy

**Cel:** odróżnić zliczenie/drop na interfejsie od BGP import/export i stateful policy.

Na R1 zastosuj filtr `LAN-IN` z 8.8 do `ge-0/0/2.0`. W izolowanym Linux PC-A ustaw tymczasowo sekundarny adres testowy i wyślij ICMP ze wskazanym źródłem. **Nie używaj spoofed packets poza labem.**

**Urządzenie: R1-vSRX — Configuration Mode**

~~~junos
set firewall family inet filter LAN-IN term SPOOF from source-address 203.0.113.0/24
set firewall family inet filter LAN-IN term SPOOF then count SPOOF-DROPS
set firewall family inet filter LAN-IN term SPOOF then discard
set firewall family inet filter LAN-IN term ICMP-OBSERVE from protocol icmp
set firewall family inet filter LAN-IN term ICMP-OBSERVE then count ICMP-SEEN
set firewall family inet filter LAN-IN term ICMP-OBSERVE then accept
set firewall family inet filter LAN-IN term PASS-OTHER then accept
set interfaces ge-0/0/2 unit 0 family inet filter input LAN-IN
show | compare
commit check
commit confirmed 5
~~~

**Urządzenie: PC-A — Linux shell**

~~~bash
ip address add 203.0.113.10/32 dev eth0
ping -c 3 -I 203.0.113.10 10.20.20.20
ping -c 3 -I 10.10.10.10 10.20.20.20
~~~

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show firewall filter LAN-IN
show security flow session source-prefix 10.10.10.10
show route 10.20.20.0/24 exact
~~~

**Oczekiwany wynik:** spoofy źródłowy `203.0.113.10` pada na SPOOF i zwiększa licznik, zwykły ICMP z `10.10.10.10` jest akceptowany przez stateless filter. Jeżeli dalej nie działa, sprawdź SRX security policy i return path; nie oskarżaj od razu BGP. Wstępnie test ping do samej bramy może mieć inną ścieżkę processingu niż transit — testuj host-to-host.

**Naprawa:** nie rozluźniaj `SPOOF`, usuń wtórny testowy source z Linux:

**Urządzenie: PC-A — Linux shell**

~~~bash
ip address del 203.0.113.10/32 dev eth0
~~~

### 8.14.8. Faza 6 — małe ćwiczenie wnioskowania: `next term` versus `next policy`

Bez ruszania działającej BGP policy zbuduj osobną `DRY-RUN` z termem ustawiającym `metric 100` i `then next term`. Przewidź, czy kolejny term może odrzucić route i co ujrzysz na peerze, jeśli policy nie jest podpięta. Odpowiedź: bez aplikacji do protokołu **nic nie zmieni w reklamowanych routes**; test policy pozwala przećwiczyć część match logic, ale jego **domyślna akcja to accept**, więc wynik testu nie jest pełną symulacją prawdziwego BGP import/export.

**Urządzenie: R1-vSRX — Configuration Mode (samodzielna, nieaplikowana polityka)**

~~~junos
set policy-options policy-statement DRY-RUN term FIRST from route-filter 10.10.10.0/24 exact
set policy-options policy-statement DRY-RUN term FIRST then metric 100
set policy-options policy-statement DRY-RUN term FIRST then next term
set policy-options policy-statement DRY-RUN term SECOND from route-filter 10.10.10.0/24 exact
set policy-options policy-statement DRY-RUN term SECOND then reject
~~~

**Urządzenie: R1-vSRX — Operational Mode, po commit testowej definicji**

~~~junos
test policy DRY-RUN 10.10.10.0/24
show policy DRY-RUN
~~~

**Wynik:** interpretuj na tle ograniczeń `test policy`; nie zastępuje `advertising-protocol` ani spojrzenia na sąsiedni router. [S13]

### 8.14.9. Faza 7 — kontrolowany rollback i cleanup

**Cel:** wrócić do **dokładnie udokumentowanego baseline**. W laboratorium nie usuwamy globalnej konfiguracji ani nie używamy `load factory-default`.

Przed cleanup potwierdź, że na R1/R2 nie czeka pending `commit confirmed`. Najbezpieczniejsza opcja w nowym, dedykowanym labie: zapisany snapshot baseline i `rollback <index>` po sprawdzeniu różnic; numery historii nie są trwałym identyfikatorem baseline. Jeżeli lab dzielisz z innymi, usuń **tylko** obiekty z tej lekcji. Poniższe usuwanie zakłada, że obiekty utworzono wyłącznie dla tego labu i że nie zależy od nich inny scenariusz.

**Urządzenie: R1-vSRX — Configuration Mode, cleanup tylko obiektów utworzonych w tym rozdziale**

~~~junos
delete interfaces ge-0/0/2 unit 0 family inet filter input LAN-IN
delete firewall family inet filter LAN-IN
delete policy-options policy-statement DRY-RUN
delete policy-options policy-statement EXPORT-R1
delete routing-options static route 198.51.100.0/24
delete protocols bgp group TO-R2
delete routing-options autonomous-system
show | compare
commit check
commit confirmed 5
~~~

**Urządzenie: R2-vSRX — Configuration Mode, cleanup tylko obiektów utworzonych w tym rozdziale**

~~~junos
delete policy-options policy-statement IMPORT-R1
delete policy-options policy-statement EXPORT-R2
delete protocols bgp group TO-R1
delete routing-options autonomous-system
show | compare
commit check
commit confirmed 5
~~~

**Ważna granica:** przywrócenie *sprzed labu* wymaga dodatkowego usunięcia testowych interface IP, security zones i policies **tylko jeżeli nie były częścią wcześniejszej konfiguracji**. Jeśli użyto nowego vSRX i cała topologia została stworzona od zera, możesz usunąć także `ge-0/0/1.0` / `ge-0/0/2.0` i polityki LAB; jeżeli był już baseline, lepiej użyć zweryfikowanego snapshot/rollback. Nie publikuj żadnych danych uwierzytelniających.

**Urządzenie: R1-vSRX — Operational Mode, post-cleanup**

~~~junos
show bgp summary
show route 198.51.100.0/24 exact
show configuration policy-options | display set
show configuration firewall | display set
show configuration protocols bgp | display set
~~~

**Urządzenie: R2-vSRX — Operational Mode, post-cleanup**

~~~junos
show bgp summary
show configuration policy-options | display set
show configuration protocols bgp | display set
~~~

Nie deklaruj PASS bez dowodów uruchomienia. Wynik końcowy powinien być zapisany jako: PLANNED → GENERATED → STATIC REVIEWED (po audycie składni i zależności) → RUNTIME VALIDATED (wyłącznie po EVE i z artefaktami). **Na dzień 19 listopada 2026 status to GENERATED / NOT RUNTIME VALIDATED.**

---

## Warsztat operatorski: pięć dodatkowych testów przewidywania (uzupełnienie 8.3–8.13)

Poniższe historie są autorskimi ćwiczeniami opartymi na udokumentowanych zachowaniach Junos. Nie są pytaniami z prawdziwego egzaminu. Zostały dodane, bo na poziomie JNCIP potrzebujesz nie tylko wiedzieć, **co działa**, ale również przewidzieć **jak router zachowa się po zmianie kolejności lub kontekstu**.

### 1. Dwa termy wyglądają podobnie, lecz jeden chroni firmę, drugi ujawnia management

W sieci office mamy VLAN management `10.99.0.0/24`, VLAN users `10.10.10.0/24` i peering do operatora. Inżynier konfiguruje policy export z termem `from protocol direct` i `then accept`, bo w output widać, że użytkownicy są connected. Test pingów działa. Wydaje mu się, że wdrożenie jest udane.

Zastanów się, czy taką konfigurację zaakceptowałbyś w change review. **Nie.** Direct jest kategorią źródła trasy. Nie odpowiada na pytanie, *która* sieć connected może zostać opublikowana. W tej samej RIB możesz mieć bezpośrednio podłączony WAN transit, management subnet, testowy segment migracyjny i loopback. Jeśli każda taka trasa jest eligible do eksportu, szeroki term może pozwolić na rozgłoszenie więcej niż planowano.

W Cisco analogiczny błąd popełnia się przez zbyt szerokie `redistribute connected` bez route-map/prefix-list. W Junos najpierw definiuj biznesową listę autoryzowanych prefiksów, dopiero potem dokładaj kontrolę `protocol direct`. Zwróć uwagę na kwalifikację do eksportu: policy sama nie tworzy aktywnej trasy, ale może pozwolić na aktywny connected, o którym zapomniałeś.

**Urządzenie: R1-vSRX — Operational Mode, audyt zakresu direct przed export**

~~~junos
show route protocol direct
show route advertising-protocol bgp 192.0.2.2
show configuration policy-options policy-statement EXPORT-R1 | display set
~~~

Prawidłowy postcheck musi zawierać test **ujemny**. Jeżeli na R1 są trzy active direct prefiksy, a klient zamówił reklamę jednej sieci, potwierdź również, że dwóch pozostałych nie ma w advertised routes. To zwykle ważniejsze od samego `BGP Established`.

### 2. Wynik `test policy` wygląda dobrze, ale neighbor nie dostał prefixu

Przed zmianą operator uruchamia `test policy EXPORT-R1 10.10.10.0/24`. Widzi poprawne dopasowanie. Następnie po commit sąsiad nadal nie otrzymuje trasy. Czy Junos ignoruje politykę? Niekoniecznie.

Ten test działa na kandydatach znalezionych w lokalnej RIB i sprawdza ich match logic w odpowiednim kontekście, ale **nie zastępuje rzeczywistej ścieżki protokołu**. Może nie uchwycić stanu BGP neighbor, AFI/SAFI, protokołowych reguł advertisement, kwalifikacji active route czy wcześniejszego chain termu używanego w produkcji. Ma także własne domyślne zachowanie akceptacji, opisane w dokumentacji [S13].

Wybierz zatem właściwy dowód dla właściwego etapu: `test policy` przed zmianą jest testem logiki; `show route advertising-protocol` po zmianie to widok wyjściowy; na sąsiedzie `show route receive-protocol` i active RIB pokazują kolejny etap. Gdy działanie nie zgadza się z przewidywaniem, rozdziel *nieaplikowana polityka*, *niekwalifikowany kandydat*, *brak NLRI z powodu protokołu* i *inbound reject po stronie odbiorcy*.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
test policy EXPORT-R1 10.10.10.0/24
show route 10.10.10.0/24 exact extensive
show route advertising-protocol bgp 192.0.2.2
show bgp neighbor 192.0.2.2
~~~

Jeżeli pierwszy wiersz jest pomyślny, lecz trzeci nie pokazuje prefixu, wciąż masz do zbadania dwa ogniwa: powiązanie policy z grupą/peerem oraz ograniczenia aktywności i eksportu.

### 3. „Zmieniłem filter, więc jego counters powinny rosnąć” — niekoniecznie

Po wprowadzeniu stateless filter pakiety dalej przechodzą; wartości `SPOOF-DROPS` są zerowe. To może oznaczać, że nikt nie wysłał pakietu o źródłowym adresie pasującym do termu, ale równie dobrze: filtr jest zdefiniowany, lecz nie przypięty; jest przypięty do złego kierunku; badany ruch jest IPv6, a filter jest `family inet`; wcześniej działa inny term; filter nie został zainstalowany w PFE; test wykorzystuje RE-originated ping zamiast faktycznego tranzytu.

Dlatego w monitoringu rozdziel trzy pytania. Czy obiekt istnieje w konfiguracji? Czy jest przypisany do **tego interfejsu, tej jednostki i tej family**? Czy urządzenie rzeczywiście wykonało przetwarzanie dla świeżo wywołanego pakietu?

**Urządzenie: EX-CAMPUS-LAB — Operational Mode, przykładowy punkt audytu platformowego**

~~~junos
show configuration firewall family inet | display set
show configuration interfaces | display set | match filter
show firewall
show interfaces terse
~~~

Na EX/QFX dodatkowo sprawdź właściwą dla platformy diagnostykę instalacji/kompilacji filtra. Sam `commit complete` nie oznacza, że w ASIC masz wystarczającą liczbę termów i counters dla wymaganego zachowania na wszystkich kierunkach. Dokładne możliwości trzeba dobrać do PFE, wydania i hardware SKU; nie obiecuj jeden-do-jednego zgodności vSRX z EX.

**Środowisko testowe:** wykonaj dwa odczyty counter przed i po wysłaniu trzech kontrolowanych pakietów. Dopiero zmiana delta pozwala łączyć test z faktycznie dopasowanym termem.

### 4. Działa IPv4, lecz IPv6 „znika” po wdrożeniu nowej polityki

To klasyczny pośpiech przy migracji. Inżynier widzi działające `inet.0`, prawidłowe BGP IPv4 i filtr `family inet`. Zakłada, że przeprowadzono ten sam proces dla IPv6. Tymczasem `inet6.0`, `family inet6 unicast`, `firewall family inet6`, ICMPv6 Neighbor Discovery i host firewalls stanowią odrębne elementy.

Na routingu możesz zbudować oddzielną policy z `route-filter 2001:db8:10::/48 exact`, ale nie wprowadzaj jej do IPv4 BGP family. Na dataplane filtrowanie IPv6 nie musi być włączone razem z IPv4. W dodatku nieumiejętne odrzucenie ICMPv6 może zniszczyć ND i rozpoznawanie sąsiadów, co będzie przypominać błędny routing nawet wtedy, gdy IPv6 route jest aktywna.

**Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route table inet6.0
show bgp summary
show configuration protocols bgp | display set
show configuration firewall family inet6 | display set
show ipv6 neighbors
~~~

**Symulowany wniosek:** BGP IPv4 działa, lecz nie skonfigurowano obsługi inet6 unicast i host nie ma poprawnego Neighbor Discovery. Naprawa polega na ustaleniu **którego etapu brakuje**, a nie na przyznaniu „permit all” do firewall filter w nadziei, że IPv6 ruszy.

### 5. BGP route jest aktywna, lecz filtr polityki „odrzucił coś innego”

Nie każdy błąd ma postać braku prefiksu. R1 reklamuje `10.10.10.0/24`. R2 otrzymał trasę i jest aktywna, ale ruch do serwera przez WAN idzie niezgodnie z oczekiwaniem. Jeśli istnieje alternatywna **bardziej szczegółowa** trasa `10.10.10.128/25`, LPM może skierować ruch do serwerów z tego zakresu inną ścieżką niezależnie od tego, czy BGP policy zaakceptowała /24. Routing policy dla /24 nie „przejmuje” pakietów pasujących /25.

Tutaj koniecznie wykonaj trzy odrębne odczyty: candidate route po BGP import; wszystkie active prefixes obejmujące destination; forwarding entry dla konkretnego destination IP. Dopiero potem sprawdzaj uRPF i filters.

**Urządzenie: R2-vSRX — Operational Mode**

~~~junos
show route 10.10.10.0/24 exact
show route 10.10.10.128/25 exact
show route forwarding-table destination 10.10.10.150
show route receive-protocol bgp 192.0.2.1
~~~

**Root cause symulowany:** specyficzna /25 ma inną drogę; operator ocenił pakiet tylko na podstawie /24. **Naprawa:** popraw projekt lub konfigurację /25, nie zmieniaj bez potrzeby eksportu /24.

---

## Jak wygląda produkcyjny MOP dla zmiany polityki?

Gdybyś projektował nową lokalizację enterprise albo prowadził change na aktywnym WAN edge, powinieneś traktować import/export i interface filters jak zmiany potencjalnie krytyczne dla całej lokalizacji. Dobrą metodą jest zapisanie przed oknem serwisowym: pełnej listy prefiksów, właściciela każdego prefixu, dozwolonego AS, oczekiwanego kierunku, numeru interfejsu, typu policy, istniejącej ścieżki return oraz oczekiwanych security zones. Nie wpisuj w ticket sekretów ani prywatnych danych klienta.

Przy **BGP export** warto policzyć liczbę prefiksów obecnie reklamowanych, przygotować explicit whitelist i udowodnić zarówno wysłanie wymaganych tras, jak i niewysłanie nieautoryzowanych. W czasie zmiany włączaj tylko jeden logiczny etap, porównuj received/advertised routes na obu końcach, a po zmianie wykonaj test transitu z hosta. Przy **import** ostrożność powinna być jeszcze większa: niechcący odrzucona default route może odciąć całą lokalizację, chociaż sam BGP pozostanie Established.

Przy **firewall filter** osobno wpisz test kontrolny allowed traffic, expected denied traffic i obserwowany counter. Nie sprawdzaj tylko IP źródłowego — sprawdź family, protocol, destination, interface unit, input/output i kolejność termów. Szczególnym ryzykiem jest filtrowanie na `lo0`, bo błędny term może odebrać RE BGP, OSPF, NTP lub SSH. Ochrona RE nie powinna być wdrażana bez konsoli/OOB i planu rollback.

Przy **SRX** wymagaj postcheck sesji stateful i właściwego host-inbound; brak transmisji może wynikać z reguł firewall niezależnych od BGP. Ważne jest także rozróżnienie „ping routera do sąsiada” od „aplikacja klienta przez router”: różnią się źródłem i przebywaną ścieżką processingu.

Dopiero na końcu zdecyduj, czy problem należał do control plane (route advertisement/import), route selection (aktywny RIB), dataplane (FIB/ACL/uRPF) czy stateful security flow. Ta decyzja ma wynikać z dowodów, nie z tego, że ostatnio zmieniano BGP.


---

## Dwa pełne przypadki operacyjne (przekrojowe rozwinięcie 8.1–8.14)

### Case A — eBGP Established, ale branch route nie dociera do centrali

**SYMPTOM:** NOC widzi `Established` na R1↔R2, centrala nie ma `10.10.10.0/24`.

**HIPOTEZY:** (1) route w R1 inactive, (2) export policy nieakceptująca, (3) R1 nie reklamuje rodziną inet, (4) R2 import odrzuca route, (5) R2 route jest hidden z innego powodu.

**TEST — Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route 10.10.10.0/24 exact extensive
show bgp summary
show route advertising-protocol bgp 192.0.2.2
show configuration protocols bgp | display set
show configuration policy-options | display set
~~~

**WYNIK symulowany:** R1 ma direct active, ale `advertising-protocol` nie zawiera /24. W `EXPORT-R1` jest błędny filter `10.10.10.0/25 exact`.

**ROOT CAUSE:** export route-filter nigdy nie dopasowuje /24; jawny końcowy reject usuwa prefiks z eksportu.

**NAPRAWA:** popraw match na `10.10.10.0/24 exact`, sprawdź różnice, `commit confirmed` w izolowanym LAB.

**POSTCHECK:** po stronie R1 /24 w advertised, po R2 /24 w received/active, host-to-host ping i route return w normie. Jeśli PC nadal nie działa — osobna hipoteza dataplane/SRX.

### Case B — route aktywna i ping działa „z routera”, ale nie z PC

**SYMPTOM:** R1 potrafi pingować R2 WAN; PC-A nie widzi PC-B.

**HIPOTEZY:** source z RE omija inny fragment dataplane; filtrovan ingress LAN; brak security policy; brak return path; endpoint firewall.

**TEST — Urządzenie: R1-vSRX — Operational Mode**

~~~junos
show route 10.20.20.0/24 exact
show route forwarding-table destination 10.20.20.20
show firewall filter LAN-IN
show security policies from-zone LAN to-zone WAN
show security flow session source-prefix 10.10.10.10
~~~

**WYNIK symulowany:** BGP route aktywna i forwarding wskazuje WAN, lecz po świeżych testach PC-A rośnie filtr `SPOOF-DROPS`; na PC-A ktoś zmienił source IP na `203.0.113.10`.

**ROOT CAUSE:** stateless ingress filter odrzuca pakiet **przed osiągnięciem poprawnego forwarding**. Ping z RE na WAN source nie przechodzi tym samym wejściowym torem i nie dowodzi stanu PC-A transit.

**NAPRAWA:** przywróć właściwe źródło IP hosta; nie usuwaj filtra. Potwierdź zmianę licznika, następnie host-to-host i aplikację. Jeżeli nie ma sesji, sprawdź właściwą security policy w drugim kroku.

---

## Sprawdzian: zatrzymaj się i przewidź

**1.** R1 ma aktywną connected /24, ale brak jawnej BGP export. Czy default BGP policy zawsze ją ogłosi? **Nie.** Domyślny BGP export nie jest „redistribute connected”; jawne export jest bezpiecznym wyborem.

**2.** BGP import odrzuca trasę, ale `show route receive-protocol ... hidden` ją pokazuje. Czy router używa jej do forwarding? **Nie jako normalnej aktywnej trasy.** Hidden to informacja retained diagnostycznie, nie aktywny FIB next-hop.

**3.** Term 1 `then accept`, term 2 `then reject` dla tej samej trasy. Co wygra? **Term 1** — terminating action zatrzymuje politykę.

**4.** Term 1 `then metric 100` i `then next policy`. Czy metric przetrwa? **Może być przeniesiona** do dalszego przetwarzania i wykorzystana przez kolejne policies; następna ma nadal prawo odmówić accept.

**5.** Stateless filter ma pierwszy pasujący term z samym `then count`, a pod nim `then discard`. Czy pakiet na pewno zostanie odrzucony? **Nie.** Może nastąpić implicit accept w pierwszym termie. To cenna pułapka Junos.

**6.** OSPF export domyślnie reject; sąsiad nadal widzi własną sieć OSPF interfejsu R1. Sprzeczność? **Nie.** Link-state flooding niezależnie od external redistribution.

**7.** Strict uRPF odrzuca legalny pakiet na ISP-B, reverse route jest przez ISP-A. Dlaczego? **Asymetria reverse path**. Użycie loose mode powinno wynikać z analizy bezpieczeństwa, nie z przypadkowego wyłączenia ochrony.

**8.** Zmiana export policy pozwala na route, ale dalej brak TCP do serwera przez vSRX. Czy zmiana BGP rozwiązała firewall? **Nie.** Testuj SRX zones/policies/session i host return traffic.

**9.** `10.0.0.0/8 orlonger` i `10.10.0.0/16 exact` w tym samym termie; candidate `10.10.20.0/24`. Czy szersze dopasowanie zawsze zadziała? **Nie.** Route-filter longest-match może wybrać /16 exact, a potem zakończyć match jako false.

**10.** Policy `test policy` pokazuje więcej tras niż produkcyjny BGP export. Czy tool jest błędny? **Nie.** `test policy` używa własnego default accept i nie symuluje wszystkich reguł eligibility oraz protokołu. [S13]

---

## Ścieżka nauki: Associate → Specialist → Professional

**JNCIA-Junos [A]** wymaga rozróżnienia dwóch obiektów — **route** i **packet** — oraz zrozumienia domyślnych zasad, import/export, term/match/then, flow control, podstaw filtrów, uRPF, prostych konfiguracji i diagnostyki.

**JNCIS-ENT [S]** rozwija policy applied to OSPF, IS-IS i BGP. Musisz rozpoznać, *które protokoły rzeczywiście respektują policy import*, jak działa redistribution, dlaczego route jest hidden i jak powiązać BGP Adj-RIB, RIB/FIB i filtrowanie pakietów. To poziom, na którym diagnoza „brak route” przestaje wystarczać.

**JNCIP-ENT [P]** rozszerza to o złożone polityki w IGP/BGP, community, AS_PATH, LOCAL_PREF, MED, policy chaining, multihoming, route leaking/multipath (w osobnych rozdziałach) i troubleshooting. Ten rozdział buduje **fundament logiczny**; nie zastępuje pełnych rozdziałów 9–11 o OSPF/IS-IS/BGP, 13 o multicast, 16 o L2 security ani 20 o CoS.

**Trzy pytania, które powinieneś już sobie zadawać:**

> „Czy problem dotyczy wejścia **trasy do RIB**, wyjścia **trasy do neighbor**, czy przejścia **pakietu przez interfejs i dataplane**?”

> „Jeżeli route została odrzucona: w którym termie, przez jakie match criteria, jakie default policy i jaki protokół?”

> „Jeżeli route istnieje: co mówi FIB, gdzie jest filtr, co mówi uRPF, a na SRX — co mówi security session?”

Jeśli potrafisz odpowiedzieć na te pytania z CLI i przewidzieć efekt zmiany *zanim* ją zatwierdzisz, przestajesz traktować routing policies jako zaklęcia. Zaczynasz świadomie kontrolować informację routingu i ruch klientów.

---

## Źródła producenta i dokumentacja do dalszej nauki

Źródła zostały dobrane do konkretnych mechanizmów, a nie jako ogólna bibliografia do „Junosa”. Wzorce poleceń są dydaktyczne; przed użyciem na konkretnym modelu potwierdź release, rodzinę i Feature Explorer.

- **[S1] Understanding Routing Policies** — model import/export oraz ograniczenia oceny export: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-overview.html
- **[S2] Default Routing Policies** — porównanie BGP, OSPF, IS-IS i wyjątków PTX: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-actions-defaults.html
- **[S3] Summary of Routing Policy Actions** — accept/reject/next term/next policy i modyfikacja atrybutów: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-default-actions-default-policies.html
- **[S4] How a Routing Policy Is Evaluated** — sekwencja termów: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-evaluation-method.html
- **[S5] import (BGP), CLI Reference** — hidden routes, keep all/none: https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-edit-protocols-bgp.html
- **[S6] show route receive-protocol / advertising-protocol**, CLI Reference: https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-receive-protocol.html ; https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-advertising-protocol.html
- **[S7] Routing Policy Match Conditions** — prefix-list/route-filter, AS path, communities: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-match-conditions-in-routing-policy-terms.html
- **[S8] Understanding Route Filters for Routing Policy Match Conditions** — route-filter longest-match/walkup pułapki: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-route-lists-for-use-in-routing-policy-match-conditions.html
- **[S9] Stateless Firewall Filter Overview and Application Points** — gdzie można filtrować pakiety: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-overview.html ; https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-application-points.html
- **[S10] Stateless Firewall Filter Components** — implicit accept po count/log i następne termy: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-components.html
- **[S11] Understanding Unicast RPF (Routers) / rpf-check CLI Reference** — strict/loose, fail-filter i support: https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/interfaces-configuring-unicast-rpf.html ; https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rpf-check-edit-interfaces.html
- **[S12] Protocol Support for Import and Export Policies; OSPF Routing Policy** — które operacje są faktycznie wspierane: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-protocol-support-import-export-policies.html ; https://www.juniper.net/documentation/us/en/software/junos/ospf/topics/topic-map/configuring-ospf-routing-policy.html
- **[S13] Understanding Routing Policy Tests** — `test policy` ma default accept: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-testing-routing-policies.html
- **[S14] SRX Traffic Processing Overview / Security Policies** — stateless filter != stateful flow: https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html ; https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html
- **[S15] Day One: Configuring Routing Policies**, scenariusze BGP import/export: https://www.juniper.net/documentation/en_US/day-one-books/topics/topic-map/configuring-routing-policies.html
- **[S16] Firewall Filter Installation/Monitoring**, counters i ograniczenia hardware: https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/display_firewall_filter_installation_status.html ; https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/firewall-filter-stateless-example-count-accepted-and-rejected-packets.html
- **[S17] Understanding Unicast RPF (Switches)** — różnice sprzętowe: https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/concept/unicast-rpf-ex-series.html
- **[S18] Basic BGP Routing Policies** — praktyczne przykłady import/export BGP: https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html

**Provenance:** sprawdzono publiczne Juniper/HPE CLI Reference, Routing Policies and Firewall Filters, Day One, OSPF/BGP guidance oraz przykłady troubleshootingu. Linki mogą przekierowywać na domenę HPE networkingdocs po zmianach organizacyjnych. Nie korzystano z exam dumpów. Diagramy Mermaid są oryginalne; nie użyto cudzych obrazów. **Dopasowanie CLI do konkretnego obrazu vSRX wymaga testu runtime**.

**Odsyłacze do powiązanych rozdziałów:** [Rozdział 2 — Architektura RE/PFE](JUNOS_Rozdzial_02_Architektura_Junos_OS.md), [Rozdział 3 — CLI i commit](JUNOS_Rozdzial_03_Junos_User_Interfaces_CLI.md), [Rozdział 6 — Routing Fundamentals](JUNOS_Rozdzial_06_Routing_Fundamentals.md), [Rozdział 7 — Protocol-Independent Routing](JUNOS_Rozdzial_07_Protocol_Independent_Routing.md).
