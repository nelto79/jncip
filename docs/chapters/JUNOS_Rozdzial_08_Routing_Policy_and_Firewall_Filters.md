
# Rozdział 8. Routing Policy and Firewall Filters — kto decyduje o trasie, a kto zatrzymuje pakiet?

**JUNOS Enterprise Routing & Switching | JNCIA-Junos → JNCIS-ENT → JNCIP-ENT**  
**Zakres:** 8.1–8.14 z [Unified Blueprint](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md)  
**Wersja:** 2.0, 19 listopada 2026 — rozdział napisany od nowa jako prowadzona lekcja.  
**Status:** GENERATED / SOURCE-REVIEWED / NOT RUNTIME VALIDATED. Wszystkie wyniki urządzeń w tym rozdziale są *symulowane*, a konfiguracje to projekty ćwiczeń, nie raporty z działającego laboratorium.  
**Punkt odniesienia:** klasyczny Junos na routerach i switchach oraz vSRX w EVE-NG. Odmienności platform i wersji sygnalizujemy w miejscu, w którym rzeczywiście wpływają na wniosek.

> Przez najbliższe lekcje nie będziemy uczyć się „trzech rodzajów ACL”. Będziemy rozwiązywać jeden problem sieciowy, który początkowo wygląda jak usterka BGP, później jak awaria routingu, a na końcu okaże się także problemem filtrowania pakietów. Tak najłatwiej zobaczyć, dlaczego Junos potrzebuje kilku różnych mechanizmów i dlaczego ich nazwy bywają myląco podobne.

## Przed rozpoczęciem: firma, dwa budynki i telefon z NOC

Jest poniedziałek, 8:15. Firma otworzyła niewielki oddział. W oddziale stoi router **R1**, w centrali router **R2**. Oba działają na Junos i zestawiają eBGP przez połączenie WAN. W oddziale jest sieć użytkowników **10.10.10.0/24**, w centrali — **10.20.20.0/24**. Administrator testuje dostęp do serwera ERP. Sesja BGP jest `Established`, ale połączenie nie działa.

Pierwsza reakcja jest znajoma każdemu inżynierowi Cisco: sprawdzić sąsiada, routing table i ACL. Tyle że tutaj „ACL” może oznaczać trzy bardzo różne rzeczy. Ktoś mógł zabronić **ogłoszenia trasy** z R1 do R2. Ktoś mógł zabronić **przyjęcia trasy** na R2. Albo ścieżka jest prawidłowa, lecz zatrzymywany jest dopiero **pakiet** z komputera użytkownika. Jeśli pracujemy na SRX, dojdzie jeszcze kontrola stanu sesji między strefami bezpieczeństwa.

Nie rozwiązujmy wszystkich tych problemów od razu. Najpierw zobaczmy, jak wygląda sieć, zanim cokolwiek skonfigurujemy.

```mermaid
flowchart LR
    PA["PC-A<br/>10.10.10.10"] --- A["LAN-A<br/>10.10.10.0/24"]
    A --- R1["R1 — Junos<br/>10.10.10.1<br/>AS 64510"]
    R1 --- W["WAN 192.0.2.0/30<br/>eBGP"]
    W --- R2["R2 — Junos<br/>10.20.20.1<br/>AS 64520"]
    R2 --- B["LAN-B<br/>10.20.20.0/24"]
    B --- PB["PC-B<br/>10.20.20.20"]
```

*Rysunek 8.1. Topologia, którą będziemy rozbudowywać. Tekst alternatywny: dwa routery R1 i R2 połączone WAN i eBGP, po jednym LAN oraz komputerze po każdej stronie. Diagram autorski.* WAN: R1 = 192.0.2.1/30, R2 = 192.0.2.2/30. Adresy 192.0.2.0/24 i 198.51.100.0/24 są używane wyłącznie jako przestrzenie dokumentacyjne w izolowanym labie.

Zadaj sobie proste pytanie: **co właściwie musi być prawdą, by PC-A nawiązał połączenie z PC-B?** R1 musi wiedzieć, dokąd wysłać pakiet. R2 musi znać drogę powrotną do PC-A. Oba urządzenia muszą dopuścić ruch na swojej ścieżce forwarding. A komputery muszą mieć właściwe bramy, adresy i lokalne polityki bezpieczeństwa.

Zauważ, jak wiele warunków mieści się za zielonym statusem jednego sąsiedztwa BGP.

---

## 8.1. Default routing policies — co zrobi router, jeśli mu niczego nie nakażesz? [A]

Zacznijmy od sytuacji sprzed pierwszej zmiany. Interfejs LAN R1 jest skonfigurowany, więc w lokalnym RIB — zwykle `inet.0` — pojawia się trasa **direct 10.10.10.0/24**. Router ma także trasę lokalną do własnego adresu interfejsu. R1 może bez żadnego BGP odpowiadać hostowi PC-A z tej podsieci; oczywiście filtrowanie pakietów lub polityka bezpieczeństwa nadal mogą ograniczać konkretne testy.

Konfigurujemy eBGP między R1 i R2. Czy R1 automatycznie zacznie reklamować swoją sieć LAN? Wiele osób odruchowo odpowie „tak, przecież trasa istnieje”. W Junos to błędna intuicja. **Samo posiadanie trasy w RIB nie jest zleceniem rozgłoszenia jej przez dowolny protokół.**

Wyobraź sobie dyspozytora firmy kurierskiej. Na jego wewnętrznej mapie są wszystkie drogi, magazyny, boczne wjazdy i wejścia serwisowe. Nie oznacza to, że cały ten plan trafi do zewnętrznego przewoźnika. Wewnętrzna mapa mówi, gdzie firma *potrafi* dojechać. Zasady publikowania mapy mówią, czego *wolno* dowiedzieć się partnerowi.

W Junos każdy protokół ma własne domyślne zasady importu i eksportu. Dla typowego BGP na klasycznych urządzeniach Junos domyślny eksport dotyczy kwalifikowanych aktywnych tras BGP, z uwzględnieniem zasad BGP. Nie oznacza automatycznej redystrybucji `direct` czy `static`. Aby udostępnić R2 trasę LAN-A pochodzącą z `direct`, będziemy potrzebowali jawnej export policy.

Z OSPF jest inaczej: jeśli włączysz OSPF na interfejsie, protokół rozgłasza własną topologię poprzez LSA. Fakt, że domyślna *export policy* OSPF nie redystrybuuje obcych tras, **nie zatrzymuje tego naturalnego mechanizmu link-state**. W IS-IS obowiązuje podobne rozróżnienie między własną bazą link-state i importowaniem tras z innego źródła. To ważne, ponieważ słowo „export” nie ma identycznych konsekwencji w każdym protokole.

Nie traktuj też default policy jak jednej globalnej zasady `permit any` lub `deny any`. To reguły protokołowe, zależne od kierunku. Wyjątki istnieją również pomiędzy rodzinami urządzeń — dokumentacja Juniper wymienia m.in. odmienne domyślne zachowania BGP na PTX. Na egzaminie najpierw rozpoznaj protokół i kontekst; w produkcji dopisz do analizy model oraz release.

**R1-vSRX — Operational Mode, obserwacja przed wprowadzeniem policy**

~~~junos
show route 10.10.10.0/24 exact
show bgp summary
show route advertising-protocol bgp 192.0.2.2
show configuration protocols bgp | display set
~~~

W wynikach z naszego hipotetycznego oddziału znajdziesz aktywną trasę direct w pierwszym poleceniu, sąsiada `Established` w drugim, a brak LAN-A na liście ogłaszanych prefiksów w trzecim. To **nie jest sprzeczność**: każde polecenie odpowiada na inne pytanie. Pierwsze mówi „czy R1 zna tę sieć?”, drugie „czy rozmawia z R2?”, trzecie „czy opowiada R2 o tej sieci?”.

Jeśli znasz Cisco IOS XE, myśl o tym jak o różnicy między posiadaniem trasy w `show ip route` a celowym `redistribute connected route-map ...`. Analogia dotyczy zamiaru, nie składni ani wszystkich reguł kwalifikacji BGP.

**Przewidź:** gdy R1 nie reklamuje LAN-A, czy PC-A straci ping do 10.10.10.1? Nie z powodu braku eksportu. To problem informacji przekazywanej R2, nie lokalnego połączenia LAN. Gdybyś natomiast odrzucił *trasę potrzebną R1* przez import policy, mógłbyś pośrednio wpłynąć na forwarding. Właśnie na tym polega kolejna lekcja.

*Źródło zachowania: [Juniper — Default Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-actions-defaults.html).*

---

## 8.2. Import i export — dwie strony tej samej rozmowy [A]

Teraz operator pyta: „Skoro mówimy o export, to czy import jest ruchem wchodzącym do interfejsu?”. Nie. To bardzo częsta pułapka. Słowa **import** i **export** w `policy-options` opisują ruch *informacji o trasach* między protokołem a tablicą routingu, a nie kierunek pakietów użytkowników na porcie.

Prześledźmy jedną trasę. R1 wysyła wiadomość BGP UPDATE z prefiksem 10.10.10.0/24. R2 ją odbiera i może zastosować BGP **import policy**. Policy sprawdza, co przyszło, i decyduje, czy informacja może stać się kandydatem w tablicy routingu oraz czy należy zmienić atrybuty trasy. Jeżeli import zaakceptuje kandydaturę, R2 porównuje ją z innymi możliwymi drogami. Tylko odpowiednio wybrana i rozwiązana trasa będzie użyteczna dla forwarding.

W drugą stronę R1 patrzy na trasy kwalifikowane do ogłoszenia i stosuje **export policy**: które wolno wysłać R2 i z jakimi atrybutami? To nie jest kopiowanie całej tablicy `inet.0`; obowiązują reguły protokołu, aktywność trasy, rodzina adresów i inne warunki kwalifikacji.

```mermaid
flowchart LR
    A["R1<br/>RIB: inet.0"] --> B{"BGP export<br/>na R1"}
    B -->|zaakceptowano| C["BGP UPDATE<br/>przez WAN"]
    B -->|odrzucono| X["brak reklamy"]
    C --> D{"BGP import<br/>na R2"}
    D -->|zaakceptowano| E["R2: kandydat<br/>w inet.0"]
    D -->|odrzucono| H["odrzucona/hidden<br/>ścieżka BGP"]
    E --> F["wybór aktywnej trasy<br/>i forwarding"]
```

*Rysunek 8.2. Podróż jednej informacji routingu. Tekst alternatywny: route wychodzi z RIB przez export na R1, BGP przekazuje ją do import na R2; akceptacja daje kandydata w RIB, odrzucenie nie daje aktywnej ścieżki. Diagram autorski. Uproszczenie: BGP może trzymać wiele alternatywnych ścieżek; kolejność szczegółowych etapów zależy od implementacji.*

Przy Cisco użyłbyś podobnej intuicji do `neighbor ... route-map ... in` i `out`. Ale jest różnica, którą warto wykorzystać przy diagnostyce. **Odrzucona przez BGP import trasa może pozostać w Junos jako hidden**. Wtedy użytkownicy jej nie wykorzystają, lecz administrator może zobaczyć, co peer wysłał. Nie myl `hidden` z `inactive`: nieaktywna trasa może po prostu przegrać wybór z lepszą trasą, podczas gdy hidden ma problem z kwalifikacją, polityką lub innym warunkiem użycia. Dokładne powody odczytuj z outputu `extensive`, nie zgaduj.

**R2-vSRX — Operational Mode, sprawdzanie odbioru na R2**

~~~junos
show bgp neighbor 192.0.2.1
show route receive-protocol bgp 192.0.2.1
show route receive-protocol bgp 192.0.2.1 hidden
show route 10.10.10.0/24 exact extensive
show route protocol bgp
~~~

Co tu jest najważniejsze? `show bgp neighbor` dowodzi stanu protokołu, `receive-protocol` pozwala oglądać informacje od konkretnego peera, wariant `hidden` pomaga znaleźć odrzuconych kandydatów, a `show route ... exact extensive` pokazuje stan konkretnego prefiksu w RIB. Ani jedna z tych komend samodzielnie nie dowodzi, że pakiet PC-B wróci do PC-A.

**R1-vSRX — Operational Mode, sprawdzanie reklamy na R1**

~~~junos
show route 10.10.10.0/24 exact extensive
show route advertising-protocol bgp 192.0.2.2
~~~

Teraz możesz prowadzić rozmowę z NOC precyzyjnie: „Trasa jest aktywna na R1, ale nie widzę jej w ogłoszeniach R1” albo „R1 ją reklamuje, R2 widzi ją jako hidden — sprawdzamy import i powód ukrycia”. To dwie różne usterki i dwa różne miejsca naprawy.

Jedno ostrzeżenie na przyszłość: Junos potrafi zachowywać wiele odrzuconych ścieżek BGP. Służy to diagnostyce, ale zużywa pamięć; konfiguracja `keep none` wpływa na retencję. Nie zmieniaj jej „dla porządku” bez zrozumienia konsekwencji ponownej ewaluacji tras po zmianie polityki.

**Most z Cisco:** `show ip bgp neighbors ... received-routes` i route-map inbound/outbound to dobry punkt startowy w myśleniu. Nie zakładaj jednak identycznej dostępności wszystkich widoków, zasad retencji i nazw pól wyjściowych.

*Źródła: [Understanding Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-overview.html), [Basic BGP Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html), [show route receive-protocol](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-receive-protocol.html).*


---

## 8.3. Jak Junos czyta politykę — po co są termy i dlaczego ich kolejność potrafi zepsuć noc? [A]

Wiemy już, gdzie polityka jest stosowana. Teraz popatrzmy, jak pracuje. Mamy konkretne zamówienie z działu bezpieczeństwa: **R1 może reklamować do R2 wyłącznie podsieć użytkowników LAN-A. Nie może ujawniać sieci laboratoryjnych, zarządzających ani tranzytowych.**

Najprostsza myśl brzmi: „stworzę listę dozwolonych podsieci”. Ale Junos nie dostaje jedynie listy. Dostaje **program podejmowania decyzji**. Ten program nazywa się `policy-statement`. Wewnątrz mamy uporządkowane kroki — `term`. Każdy krok może zapytać o właściwości trasy w `from` i wykonać działanie w `then`.

Przyjrzyj się pierwszej polityce, zanim ją wpiszemy. Chcemy sprawdzać prefiks **i** pochodzenie. Jeżeli route do 10.10.10.0/24 jest trasą direct, wolno ją ogłosić. Jeżeli ktoś jutro wprowadzi przypadkiem static z tym adresem, nie chcemy, by sama zgodność numerów automatycznie dała mu przepustkę. Pozostałe prefiksy odrzucamy jawnie.

**R1-vSRX — Configuration Mode, projekt polityki eksportu**

~~~junos
set policy-options policy-statement EXPORT-TO-R2 term USERS from protocol direct
set policy-options policy-statement EXPORT-TO-R2 term USERS from route-filter 10.10.10.0/24 exact
set policy-options policy-statement EXPORT-TO-R2 term USERS then accept
set policy-options policy-statement EXPORT-TO-R2 term DENY-EVERYTHING-ELSE then reject
set protocols bgp group TO-R2 export EXPORT-TO-R2
~~~

Czytasz tę konfigurację jak krótką opowieść:

1. „Jeżeli kandydat jest trasą direct, **i** jego prefix to dokładnie 10.10.10.0/24…”.
2. „…zaakceptuj go do dalszego procesu reklamowania BGP i zakończ ocenę polityki”.
3. „W przeciwnym razie dojdź do ostatniego termu, który odrzuca wszystko inne”.

Warunki różnych rodzajów w `from` współpracują logicznie jak **AND**. Wewnątrz niektórych grup dopasowań istnieją reguły alternatyw, a dla `route-filter` również zasada wybierania najlepszego wzorca. Za chwilę zobaczymy, dlaczego nie należy sprowadzać całego silnika policy do zdania „pierwszy pasujący wiersz ACL”.

Zanim wykonasz commit, sprawdź obiekt i miejsce jego użycia. Samo zdefiniowanie `EXPORT-TO-R2` pod `policy-options` **nie podłącza go do BGP**. Równie dobrze możesz trzymać pięćdziesiąt gotowych polityk, których nie używa żaden protokół.

**R1-vSRX — Configuration Mode, bezpieczna inspekcja zmiany**

~~~junos
show policy-options policy-statement EXPORT-TO-R2
show protocols bgp group TO-R2
show | compare
commit check
~~~

To, że `commit check` przeszedł, oznacza sprawdzenie konfiguracji przez Junos, nie dowód skutecznego eksportu. W odizolowanym laboratorium użyj później `commit confirmed 5`, wykonaj post-check i dopiero po pozytywnej ocenie potwierdź zwykłym `commit`. Uważaj: dodatkowy `commit check` po uruchomieniu `commit confirmed` może potwierdzić oczekującą zmianę; nie rób go mechanicznie w trakcie odliczania.

### Co naprawdę robią accept, reject, next term i next policy?

Wyobraź sobie, że recepcja w biurowcu ma trzy stanowiska. Pierwsze sprawdza identyfikator, drugie może przybić pieczątkę, trzecie podejmuje ostateczną decyzję o wejściu. Nie każda pieczątka kończy wizytę, ale ostateczne „wpuszczam” lub „nie wpuszczam” już tak.

W routing policy `accept` i `reject` są działaniami **kończącymi ocenę łańcucha** w zwykłym scenariuszu. `next term` każe przejść do kolejnego termu *tej samej* polityki. `next policy` pomija resztę bieżącej polityki i przechodzi do *następnej* polityki na liście. Jeśli lista się skończy bez decyzji terminalnej, w grę wchodzi **default policy protokołu**.

To ostatnie zdanie jest szczególnie ważne. „Nic nie dopasowałem” nie musi oznaczać ani akceptacji, ani odrzucenia — liczy się protokół, kierunek i kontekst.

Załóżmy, że chcemy rozdzielić dwa zadania: najpierw oznaczyć trasę community BGP, a następnie sprawdzić, czy należy do whitelisty. Moglibyśmy użyć jednej policy, ale lepiej pokaże to działanie łańcucha.

**R1-vSRX — Configuration Mode, samodzielny przykład rozszerzenia polityki BGP**

~~~junos
set policy-options community BRANCH-ROUTE members 64510:100
set policy-options policy-statement MARK-ROUTE term USERS from route-filter 10.10.10.0/24 exact
set policy-options policy-statement MARK-ROUTE term USERS then community add BRANCH-ROUTE
set policy-options policy-statement MARK-ROUTE term USERS then next policy
set policy-options policy-statement FINAL-CHECK term USERS from protocol direct
set policy-options policy-statement FINAL-CHECK term USERS from route-filter 10.10.10.0/24 exact
set policy-options policy-statement FINAL-CHECK term USERS then accept
set policy-options policy-statement FINAL-CHECK term REST then reject
set protocols bgp group TO-R2 export [ MARK-ROUTE FINAL-CHECK ]
~~~

**Uwaga:** ten przykład **zastępuje**, a nie dodaje do wcześniejszego `EXPORT-TO-R2`, miejsce eksportu BGP. W laboratorium nie wprowadzaj kilku wzajemnie sprzecznych wariantów naraz. `community add` samo w sobie nie przyznaje zgody na reklamę. Dopiero `FINAL-CHECK` mówi ostatecznie „tak”. Gdyby w pierwszej polityce zamiast `next policy` było `accept`, druga policy nie miałaby okazji już niczego sprawdzić.

Można użyć również `next term`, kiedy po dodaniu atrybutu chcemy ocenić kolejny term w **tej samej** polityce. W zależności od akcji i miejsca w hierarchii Junos kontroluje dozwolone kombinacje; na Junos Evolved `next term` nie może być ostatnią akcją termu bez następnego termu, co opisuje dokumentacja producenta.

```mermaid
flowchart TD
    A["Route wchodzi do pierwszej policy"] --> B{"Term 1: match?"}
    B -->|nie| C["Kolejny term"]
    B -->|tak| D{"Działanie"}
    D -->|accept| E["Koniec: akceptacja"]
    D -->|reject| F["Koniec: odrzucenie"]
    D -->|next term| C
    D -->|next policy| G["Następna policy"]
    C --> H["Kolejny match i action"]
    G --> H
    H --> I["Jeśli brak decyzji:<br/>domyślna reguła protokołu"]
```

*Rysunek 8.3. Sterowanie przebiegiem oceny policy. Tekst alternatywny: dopasowanie termu prowadzi do terminalnych accept/reject lub do kontynuacji next term/next policy; gdy nie ma decyzji, działa protokółowa polityka domyślna. Diagram dydaktyczny, nie pełna specyfikacja silnika.*

Sprawdź swoją intuicję. Masz term `MARK` z `then community add`, pod nim term `REST then reject`. Czy sama modyfikacja community oznacza, że trasa została zaakceptowana? Nie. Ocena musi dojść do decyzji. To jedna z podstawowych różnic między **akcją modyfikującą atrybut** a **akcją sterującą przepływem**.

*Źródła: [How a Routing Policy Is Evaluated](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-evaluation-method.html), [Policy Chain Evaluation](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-chain-evaluation-method.html), [Summary of Routing Policy Actions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-default-actions-default-policies.html).*

---

## 8.4. Polityka może zmienić routing — ale nie zawsze tak, jak się spodziewasz [A]

Wróćmy do oddziału. R1 zaczął reklamować LAN-A. R2 odbiera trasę, lecz w centrali działa dodatkowy router R3, który podaje inną drogę do tego samego prefiksu. Na R2 masz teraz więcej niż jednego kandydata. Administrator mówi: „Przecież moja import policy zaakceptowała prefiks, więc na pewno to on jest w FIB”. Nie musi tak być.

W Junos przechodzimy przez kilka różnych etapów: **otrzymanie informacji → ocena przez import policy → obecność kandydata w RIB → wybór aktywnej ścieżki → rozwiązanie next hop → instalacja odpowiedniego forwardingu**. Akceptacja polityki nie jest tym samym co zwycięstwo w wyborze trasy. Decydują m.in. route preference, zasady protokołu i poprawne rozwiązanie next hop. Nawet prawidłowa aktywna trasa nie daje jeszcze gwarancji, że stateless firewall filter albo SRX security policy przepuszczą pakiet.

Wyobraź sobie dwóch kierowców oferujących przewiezienie paczki. Ochrona recepcji zaakceptowała obie przepustki. Ale dyspozytor nadal musi zdecydować, którego kierowcę wysłać. A magazyn może mieć zamkniętą bramę wyjazdową. Właśnie dlatego *accept* w policy nie jest synonimem *packet delivered*.

**R2-vSRX — Operational Mode, trzy kolejne pytania o trasę**

~~~junos
show route 10.10.10.0/24 exact extensive
show route protocol bgp
show route forwarding-table destination 10.10.10.10
~~~

Pierwsze pokazuje stan prefiksu i możliwe konkurujące drogi, drugie pozwala rozpoznać BGP jako źródło kandydatów, a trzecie daje perspektywę forwardingu do konkretnego adresu. RIB i FIB nie muszą prezentować identycznej liczby rekordów ani identycznego poziomu szczegółowości. W prawdziwym incydencie sprawdzisz także tablicę `inet6.0` lub odpowiednią routing instance, jeżeli pracujesz poza podstawowym IPv4 `inet.0`.

### Pozornie paradoksalny przypadek OSPF

Na tej samej zmianie ktoś próbuje użyć import policy OSPF do odrzucenia wewnętrznego prefiksu area. Po zmianie prefiks nadal jest widoczny. „Junos zignorował policy!”, słyszysz.

To niekoniecznie awaria. OSPF jest protokołem link-state: routery muszą zachować spójność obrazu topologii. Junos ogranicza użyteczność import policy OSPF do określonych przypadków, przede wszystkim tras **external**; odrzucenie trasy wewnętrznej zwykłą import policy nie jest uniwersalnym sposobem filtrowania jej z LSDB/RIB. Dokumentacja wprost wyjaśnia, że `reject` zastosowane do nonexternal OSPF route bywa ignorowane. Pełne wyjaśnienie area, LSA i ABR zostawimy do rozdziału 9. Teraz zapamiętaj praktyczny sens: **polityka nie jest obietnicą, że każdy protokół pozwoli ingerować w każdy etap swojej pracy**.

**R-OSPF-EDGE — Operational Mode, rozpoznanie rodzaju trasy**

~~~junos
show ospf neighbor
show ospf database
show route protocol ospf detail
show configuration protocols ospf | display set
~~~

Jeśli widzisz wewnętrzną trasę OSPF, nie próbuj na ślepo „naprawiać” problemu przez kolejne import reject. Ustal, czy problem dotyczy redystrybucji, topologii area, agregacji, czy używania trasy do forwardingu.

*Źródła: [Actions in Routing Policy Terms](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-actions-in-routing-policy-terms.html), [Protocol Support for Import and Export Policies](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-protocol-support-import-export-policies.html).*

---

## 8.5. Policy statement i term — jak budować reguły, których nie trzeba się bać [A]

W laboratorium dwie linie `from` i jedno `then accept` wyglądają niewinnie. Ale wyobraź sobie wielką sieć z trzydziestoma oddziałami, dwiema trasami wyjścia do Internetu, osobną siecią zarządzającą i setkami prefiksów BGP. Nikt nie będzie weryfikował bezpieczeństwa takiej sieci, przeglądając po kolei setki przypadkowych `set`.

Potrzebujemy czytelnej polityki. Nie po to, aby nauczyć się składni, lecz po to, żeby podczas awarii móc w kilka sekund odpowiedzieć: **która reguła jest odpowiedzialna za dany prefix?**

W Junos mamy hierarchię, którą można czytać od zewnątrz do środka:

- `policy-options` jest miejscem, gdzie definiujemy obiekty wielokrotnego użytku;
- `policy-statement` jest nazwanym zestawem zasad;
- `term` jest kolejnym krokiem tej polityki;
- `from` opisuje to, co ma pasować do kandydata;
- `then` mówi, co zrobić z dopasowaną trasą.

Ta struktura ma swoją cenę: bardzo łatwo napisać poprawny składniowo, lecz zbyt szeroki term. Rozważ różnicę między dwiema historiami.

W pierwszej administrator pisze: „Zawsze zezwalaj na reklamy tras direct”. W drugiej: „Zezwalaj wyłącznie na dokładnie zidentyfikowaną sieć użytkowników, a pozostałe odrzucaj”. Pierwsza polityka będzie krótsza. Druga znacznie lepiej zniesie rozbudowę sieci.

**R1-vSRX — Configuration Mode, przykład ryzykownie szerokiej polityki (NIE WDRAŻAJ)**

~~~junos
set policy-options policy-statement TOO-BROAD term DIRECT from protocol direct
set policy-options policy-statement TOO-BROAD term DIRECT then accept
~~~

Co stanie się za pół roku, jeśli R1 dostanie bezpośrednio podłączony segment zarządzający 10.99.99.0/24? Ten sam term może pozwolić BGP ogłosić go na zewnątrz, o ile trasa spełni pozostałe warunki protokołu. Żaden alarm nie musi się pojawić. Z punktu widzenia Junos polityka wykonała poprawnie to, o co ją poproszono.

Właśnie dlatego w produkcyjnych export policies zazwyczaj zaczynam od **świadomej whitelisty**. Nazwa termu powinna mówić, czemu służy: `ALLOW-USER-LAN`, `BLOCK-MGMT`, `DENY-REMAINDER`. To nie poprawia samego algorytmu, ale radykalnie poprawia jakość review i troubleshootingu.

Spójrz jeszcze na dwa przykłady, które pozornie wyglądają podobnie:

**R1-vSRX — Configuration Mode, ilustracja różnicy „brak from”**

~~~junos
set policy-options policy-statement CHECK-A term ALL then accept
set policy-options policy-statement CHECK-B term ALL then reject
~~~

W obu politykach term nie zawiera `from`. To nie znaczy „term nigdy się nie uruchomi”. Taki term **dopasowuje każdą trasę**, która do niego dotrze. W pierwszej polityce otwiera przejście wszystkim kandydatom; w drugiej zamyka je wszystkim. I dlatego końcowe `then reject` jest tak skutecznym mechanizmem domknięcia whitelisty.

W samym Junos możesz łatwo sprawdzić konfigurację w dwóch formatach. `display set` dobrze nadaje się do porównania zmian i wklejenia fragmentów, a hierarchiczny `show` pozwala dostrzec, **co jest wewnątrz czego**. Dobrze jest swobodnie używać obu perspektyw.

**R1-vSRX — Operational Mode, dwie perspektywy tej samej policy**

~~~junos
show configuration policy-options policy-statement EXPORT-TO-R2
show configuration policy-options policy-statement EXPORT-TO-R2 | display set
show configuration protocols bgp | display set
~~~

Najbardziej użyteczne pytanie przy review brzmi nie „czy syntax jest poprawny?”, ale „**jaki dodatkowy prefix mógłby nieoczekiwanie przejść przez ten term?**”. Pomyśl o sieci zarządzającej, interfejsie tranzytowym, loopbacku, default route lub statycznym blackhole.

---

## 8.6. Jak dobrać trasę: exact, longer, orlonger i pułapka najbardziej szczegółowej reguły [A]

Przychodzi nowe polecenie: oddział ma otrzymać nie tylko 10.10.10.0/24, ale również 10.10.11.0/24. Administrator proponuje prostszy filtr „zaakceptuj 10.10.0.0/16”. Brzmi sensownie. Tyle że wtedy musimy zadać pytanie, którego nie da się uniknąć: **czy chodzi o sam prefiks /16, czy o wszystkie sieci znajdujące się w tej przestrzeni?**

To nie jest różnica kosmetyczna. BGP reklamuje *prefiksy* — pary adres/maska. Router nie może zgadnąć, czy `10.10.0.0/16` jest oczekiwanym sumarycznym ogłoszeniem, czy wygodnym zapisem zbioru /24. Jeżeli w RIB są dwie trasy /24, polityka `route-filter 10.10.0.0/16 exact` **nie dopasuje** żadnej z nich.

### Ta sama sieć, kilka zupełnie różnych pytań

Na przykładzie bazy 10.10.0.0/16, zapamiętaj znaczenie operatorów, a nie ich mechaniczne definicje:

| Zapis | Co naprawdę przepuści jako warunek dopasowania? | Kiedy potrzebujesz |
|---|---|---|
| `exact` | Dokładnie 10.10.0.0/16, nie /24 | Gdy reklamujesz konkretny summary |
| `longer` | Prefiksy bardziej szczegółowe niż /16, bez samego /16 | Gdy interesują Cię wyłącznie podsieci |
| `orlonger` | Sam /16 **i** wszystkie bardziej szczegółowe w jego zakresie | Gdy świadomie dopuszczasz deagregację |
| `upto /24` | /16, /17 … aż do /24, lecz nie /25 | Gdy chcesz ograniczyć nadmierną szczegółowość |
| `prefix-length-range /24-/28` | Tylko podsieci /24–/28 wewnątrz /16 | Gdy wymagany jest przedział długości |

Każda z tych reguł mówi o **długości prefiksu wpisu routingu**, a nie o źródłowym adresie pakietu. Jeśli host 10.10.10.50 otwiera HTTPS, jego źródłowe IP nie jest osobnym prefiksem /32 wysyłanym do BGP tylko dlatego, że w nagłówku ma ten adres.

A teraz wykonajmy ćwiczenie bez urządzeń. Mamy w RIB trzy aktywne trasy: 10.10.0.0/16, 10.10.10.0/24 i 10.10.10.128/25. Filtr `10.10.0.0/16 upto /24` będzie mógł dopasować dwie pierwsze. Trzecia, mimo że mieści się adresowo wewnątrz /16, ma długość **/25**, więc zostanie pominięta. **Routing prefix match nie jest zwykłym sprawdzaniem, czy dany adres IP należy do sieci.**

**R1-vSRX — Configuration Mode, tylko dwa konkretne segmenty LAN**

~~~junos
set policy-options policy-statement EXPORT-LANS term USERS from protocol direct
set policy-options policy-statement EXPORT-LANS term USERS from route-filter 10.10.10.0/24 exact
set policy-options policy-statement EXPORT-LANS term USERS from route-filter 10.10.11.0/24 exact
set policy-options policy-statement EXPORT-LANS term USERS then accept
set policy-options policy-statement EXPORT-LANS term REST then reject
~~~

Dwa `route-filter` w jednym termie reprezentują alternatywne wzorce prefiksów, a `from protocol direct` ogranicza typ źródła. Gdyby trzeba było przechowywać dłuższą, wielokrotnie używaną listę, sięgnęlibyśmy po nazwany obiekt `prefix-list` i dopasowanie `from prefix-list`. Zwykłe `prefix-list` dopasowuje wymienione prefiksy dokładnie; `prefix-list-filter ... orlonger` daje inną semantykę długości. Nie traktuj obu poleceń jak synonimów.

**R1-vSRX — Configuration Mode, wariant z nazwanym zbiorem prefiksów**

~~~junos
set policy-options prefix-list BRANCH-LANS 10.10.10.0/24
set policy-options prefix-list BRANCH-LANS 10.10.11.0/24
set policy-options policy-statement EXPORT-LIST term APPROVED from protocol direct
set policy-options policy-statement EXPORT-LIST term APPROVED from prefix-list BRANCH-LANS
set policy-options policy-statement EXPORT-LIST term APPROVED then accept
set policy-options policy-statement EXPORT-LIST term REST then reject
~~~

Te dwa warianty realizują podobną intencję biznesową. Nie wdrażaj ich obu równocześnie bez świadomie zaprojektowanej listy polityk; są alternatywnymi sposobami napisania tej samej lekcji.

### Bardzo ważny niuans: policy też używa longest match, ale to nie jest LPM w FIB

Znasz regułę LPM z Cisco CEF i z rozdziału 6. Pakiet do 10.10.10.130 trafi do najbardziej szczegółowej dostępnej trasy w FIB. W routing policy Junos również potrafi szukać najlepszego prefiksowego dopasowania, lecz **szuka wzorca route-filter**, a dopiero potem ocenia przypisany do niego typ dopasowania.

Załóżmy, że w **jednym termie** są dwa warunki:

- 10.0.0.0/8 `orlonger`;
- 10.10.0.0/16 `exact`.

Przychodzi kandydat **10.10.20.0/24**. Odruchowo chciałoby się powiedzieć: „Reguła /8 orlonger pasuje, więc term pasuje”. W domyślnym zachowaniu Junos najbardziej szczegółowym wzorcem jest jednak /16. Gdy sprawdzi przy nim `exact`, okaże się, że kandydat /24 nie jest /16. Cały term może uzyskać **no match**, mimo że szerszy wzorzec wyglądał na pasujący. To nie jest błędne LPM forwarding — to inny etap i inna semantyka.

```mermaid
flowchart TD
    C["Kandydat 10.10.20.0/24"] --> T["Wybór najdłuższego prefiksu wzorca"]
    T --> X["10.10.0.0/16 exact"]
    X --> Q{"Czy /24 == /16?"}
    Q -->|nie| M["Term: NO MATCH"]
    M --> N["Przejdź do kolejnego termu/policy"]
    T -.-> Z["10.0.0.0/8 orlonger<br/>nie musi zostać sprawdzony"]
```

*Rysunek 8.4. Tekst alternatywny: route-filter najpierw wybiera najbardziej szczegółowy wzorzec /16; dopasowanie exact dla kandydata /24 nie udaje się, więc term może nie dopasować. Diagram autorski, ilustruje domyślną semantykę bez funkcji walkup.*

Jeśli chcesz dopuścić także szersze reguły, możesz rozdzielić je na osobne termy lub rozważyć `walkup`, gdzie funkcja jest dostępna i rozumiesz jej wpływ na politykę. Nie włączaj globalnego `walkup` wyłącznie po to, żeby „naprawić jeden prefix” w działającym środowisku — może zmienić wynik także dla innych polityk.

To jeden z tych niuansów Junos, które pokazują różnicę między nauką komend a rozumieniem narzędzia. Wiesz już, **co testować**: trasy /16, /24 i /25 pod konkretnymi zestawami `route-filter`, a nie tylko jedną wygodną trasę pokazową.

### Sprawdź dopasowanie, zanim dotkniesz sąsiada BGP

Junos ma polecenie `test policy`. Nie udaje ono całego eBGP, ale jest pomocne przy analizie samej policy. Dokumentacja producenta ostrzega, że test używa domyślnego **accept**, więc polityka bez jawnego końcowego `reject` może dać wynik zaskakująco szeroki. Z drugiej strony nawet pomyślny `test policy` nie gwarantuje reklamy: BGP może nie zakwalifikować trasy z powodów niezwiązanych z samym termem.

**R1-vSRX — Operational Mode, test lokalnej policy; składnię potwierdź na swoim release**

~~~junos
test policy EXPORT-LANS 10.10.10.0/24
test policy EXPORT-LANS 10.10.11.0/24
show route advertising-protocol bgp 192.0.2.2
~~~

Czytaj wynik jako dwa różne dowody: pierwszy i drugi testują logikę dopasowania tras w aktualnej tabeli, trzeci dotyczy faktycznego eksportu BGP do peera. Dopiero ich zestawienie może potwierdzić twoją hipotezę.

*Źródła: [Routing Policy Match Conditions](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-configuring-match-conditions-in-routing-policy-terms.html), [Walkup for Route Filters](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/walkup-route-filter-policy-option-overview.html), [Understanding Routing Policy Tests](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-testing-routing-policies.html).*

---

## 8.7. Teraz pakiet naprawdę rusza — skąd nagle firewall filter? [A]

Wyobraź sobie, że naprawiliśmy ogłoszenia BGP. R1 i R2 widzą swoje LAN-y, return path istnieje, a oba routery wybierają aktywne trasy. PC-A nadal nie łączy się z serwerem ERP w centrali. Kolega z drugiej zmiany proponuje: „Zmień export policy, może trzeba dopisać port TCP 443”.

Właśnie w tym miejscu rozdział 8 staje się przydatny w prawdziwej pracy. **Port TCP 443 nie należy do decyzji BGP o reklamowaniu prefiksu.** Jeśli routing jest prawidłowy, a konkretne pakiety są odrzucane, musimy zejść z poziomu informacji routingu na poziom ruchu użytkownika.

Routing policy czyta obiekt typu „sieć 10.20.20.0/24 poznana przez BGP, z takimi atrybutami”. **Firewall filter** czyta konkretne pakiety: źródło 10.10.10.10, cel 10.20.20.20, TCP, port 443, ewentualnie inne pola nagłówków i metadane właściwe rodzinie oraz platformie. To dwa odmienne pytania skierowane do dwóch mechanizmów.

Jeśli pracujesz od lat z Catalystami, odruchowo pomyślisz o IPv4 ACL przypisanej na `in` lub `out`. To dobra intuicja na start. Junos firewall filter jest zwykle **stateless**: ocenia dopasowanie bez konieczności pamiętania, czy dany przepływ został wcześniej zainicjowany. Może jednak pełnić więcej funkcji niż zwykły `permit/deny`: liczyć, logować w odpowiednio wspieranym zakresie, klasyfikować CoS, zastosować policer, chronić Routing Engine, a w specjalnym przypadku skierować ruch do innej instancji routingu (filter-based forwarding, który poznaliśmy w rozdziale 7).

### Jedno słowo „firewall” i dwa różne światy

Na routerze MX, switchu EX/QFX albo klasycznym Junos `firewall family inet filter` to konstrukcja stateless. Na SRX istnieją **dodatkowo** stateful `security policies` powiązane ze strefami i mechanizmem flow. Wygląda to jak zmiana tematu, ale dla kogoś ćwiczącego na vSRX jest absolutnie kluczowe.

Załóżmy, że stateless filter na wejściu R1 przepuszcza SYN z PC-A. Czy to automatycznie dowodzi, że SRX pozwoli mu utworzyć sesję do serwera? Nie. SRX może nadal nie mieć odpowiedniej polityki `from-zone LAN to-zone WAN`. Z drugiej strony dopuszczająca `security policy` nie naprawi pakietu odrzuconego przez skuteczny, wcześniej zastosowany stateless filter. Dokładne etapy przetwarzania zależą od platformy i trybu pracy; przy diagnozie nie wyciągamy wniosku o całej ścieżce na podstawie jednego licznika.

```mermaid
flowchart LR
    P["PC-A<br/>wysyła TCP SYN"] --> I["R1: interfejs LAN"]
    I --> F{"Stateless firewall<br/>filter"}
    F -->|discard| D["Pakiet odrzucony"]
    F -->|accept| S{"SRX: routing, zones<br/>i security flow"}
    S -->|deny| D2["Brak dopuszczonej sesji"]
    S -->|permit| O["R1 przekazuje dalej"]
```

*Rysunek 8.5. Tekst alternatywny: pakiet z hosta może zostać odrzucony przez stateless filter albo przez stateful security policy SRX. Schemat jest logiczną mapą diagnostyczną, nie uniwersalnym porządkiem wykonania w każdym PFE i trybie SRX. Diagram autorski.*

Szczególnie ważna jest jeszcze jedna różnica. Pakiet **do własnego adresu routera** nie jest tym samym co pakiet **tranzytowy przez router**. Na SRX routing protocol, SSH i ping do samego urządzenia mogą wymagać właściwego `host-inbound-traffic`, podczas gdy sesja użytkownika przechodząca między strefami wymaga odpowiedniej polityki bezpieczeństwa. Dlatego ping R1 do R2 po WAN nie dowodzi, że PC-A potrafi rozmawiać z PC-B.

*Źródła: [Comparison of Routing Policies and Firewall Filters](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-firewall-filters-comparison.html), [SRX Security Policies](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html), [SRX Security Zones](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/topic-map/security-zone-configuration.html).*

---

## 8.8. Pierwszy stateless filter — jak zatrzymać podszywający się komputer i nie zablokować całego LAN-u [A]

Spróbujmy zbudować filtr z konkretnym zadaniem, zamiast poznawać od razu listę wszystkich match conditions.

Do interfejsu LAN R1 jest podłączona wyłącznie sieć 10.10.10.0/24. Zespół bezpieczeństwa zauważył, że jeden z testowych komputerów potrafi wysyłać pakiety z nieprawidłowym adresem źródłowym 203.0.113.10. Jeśli taki pakiet pojawi się na wejściu LAN, chcemy go zablokować i policzyć. **Zwykły ruch z LAN-A nadal ma działać.**

Nie zaczynamy od składni. Najpierw ustalamy warunek: jeśli źródło pakietu znajduje się w 203.0.113.0/24, uruchom `discard` i zwiększ licznik `SPOOFED`. Jeśli pakiet nie pasuje, pozwól mu przejść przez filtr, aby dalsze etapy — routing i ewentualna security policy — podjęły swoje decyzje. Ten drugi warunek musi być jawny, bo **domyślny koniec zwykłego stateless firewall filter odrzuca pakiety bez dopasowania**.

**R1-vSRX — Configuration Mode, filtr na interfejsie LAN po potwierdzeniu mapowania NIC**

~~~junos
set firewall family inet filter LAN-IN term BLOCK-SPOOF from source-address 203.0.113.0/24
set firewall family inet filter LAN-IN term BLOCK-SPOOF then count SPOOFED
set firewall family inet filter LAN-IN term BLOCK-SPOOF then discard
set firewall family inet filter LAN-IN term ALLOW-REMAINDER then count OTHER-PACKETS
set firewall family inet filter LAN-IN term ALLOW-REMAINDER then accept
set interfaces ge-0/0/2 unit 0 family inet filter input LAN-IN
~~~

Zauważ różnicę w porównaniu z routing policy. Miejsce definicji to `firewall family inet`, a nie `policy-options`. Dopasowanie to **source-address pakietu**, a nie `route-filter` prefiksu z RIB. I wreszcie filter podpinamy do konkretnego **punktu przepływu**, czyli tutaj `input` na logicznym interfejsie `ge-0/0/2.0`.

Dla EX/QFX/MX konkretne punkty przyłożenia i możliwości sprzętowe zależą od rodziny, ASIC, kierunku oraz `family`. W przykładzie zakładamy vSRX i *zweryfikowane* mapowanie EVE-NG; nie kopiuj nazwy `ge-0/0/2` do swojego labu, jeśli interfejs LAN ma inny numer. Także na vSRX pozostałe zasady security flow nadal obowiązują.

### Dlaczego ostatni term jest tak ważny?

Przewidź, co stałoby się z normalnym pakietem od PC-A, gdybyśmy usunęli `ALLOW-REMAINDER`. Nie pasuje on do `BLOCK-SPOOF`, więc filtr sprawdza następny term. Następnego termu jednak nie ma. Na końcu zwykłego firewall filter jest **implicit discard**. Rezultat: normalny ruch również może zostać utracony.

A teraz druga, mniej intuicyjna sytuacja. Administrator mówi: „Zostawię `then count` w pierwszym termie, żeby tylko zliczać pakiety, a pod spodem dodam `then discard`”. Brzmi rozsądnie, ale **nie jest to bezpieczny sposób tworzenia łańcucha reguł**. W standardowym stateless firewall filter term z `then count`, lecz bez jawnej akcji terminalnej, domyślnie kończy się **accept** dla pasujących pakietów. Taki term może „zasłonić” następny, który nigdy nie zobaczy ruchu.

**R1-vSRX — Configuration Mode, BŁĘDNY przykład wyłącznie do analizy, nie stosować na interfejsie**

~~~junos
set firewall family inet filter BAD-ORDER term COUNT-ALL then count ALL-PACKETS
set firewall family inet filter BAD-ORDER term DENY-BLOCKED from source-address 203.0.113.0/24
set firewall family inet filter BAD-ORDER term DENY-BLOCKED then discard
~~~

Pierwszy term nie ma `from`, więc dopasowuje wszystko. Ma `count`, ale nie ma jawnego `discard`. W typowej semantyce filtra Junos pasujące pakiety zostaną zliczone i przyjęte. Ostatni term nigdy ich nie zatrzyma. Zdziwiony administrator patrzy na rosnący counter `ALL-PACKETS`, uznaje filtr za aktywny i nie zauważa, że stworzył niezamierzone zezwolenie.

Jeśli naprawdę potrzebujesz liczenia i dalszej oceny termów, użyj świadomie wspieranej konstrukcji `next term` dla odpowiedniej rodziny i platformy, zamiast zakładać, że każdy `count` automatycznie oznacza „continue”. Na Junos Evolved obowiązują dodatkowe ograniczenia dotyczące pozycji `next term`.

```mermaid
flowchart TD
  A["Pakiet wpływa do filtra"] --> T1{"Term BLOCK-SPOOF:<br/>source 203.0.113.0/24?"}
  T1 -->|tak| C["count SPOOFED<br/>discard"]
  T1 -->|nie| T2["Term ALLOW-REMAINDER:<br/>count OTHER; accept"]
  C --> D["Koniec: DROP"]
  T2 --> E["Koniec filtra: PASS"]
```

*Rysunek 8.6. Tekst alternatywny: odrzucone źródło trafia do licznika SPOOFED i discard, inne pakiety do licznika OTHER i accept. Diagram autorski.*

### Stateless to nie „pozwól na odpowiedź”

Filter nie pamięta połączenia TCP. Jeżeli dopuścisz SYN w jednym kierunku, samo to nie utworzy automatycznej reguły przepuszczającej SYN/ACK w drugim. Na klasycznym routerze albo switchu projektujesz warunki dla odpowiednich kierunków i miejsc filtra. Na SRX stan odpowiedzi może kontrolować oddzielny mechanizm sesji. Nie przenoś więc intuicji z Cisco FTD na `firewall family inet`, ani odwrotnie.

Jest też ważne zagrożenie praktyczne: filtr na `lo0` może chronić Routing Engine, ale błędny term odcinający BGP, OSPF, SSH lub NTP może pogorszyć awarię, zamiast ją naprawić. Nie wprowadzaj takiego filtra zdalnie bez przetestowanego OOB/console i planu rollback. Wersja `family inet` nie rozwiązuje automatycznie filtrowania IPv6; do tego potrzebujesz odpowiedniej `family inet6` i jej punktu zastosowania.

*Źródła: [Stateless Firewall Filter Components](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-components.html), [Firewall Filter Evaluation for EX](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-ex-series-evaluation-understanding.html), [Guidelines for Configuring Firewall Filters](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-stateless-guidelines-for-configuring.html).*

---

## 8.9. Jak znaleźć miejsce, w którym znika pakiet [A]

W naszym oddziale testowy host wysyła nieprawidłowo zaadresowany pakiet. Router R1 ma już filtr `LAN-IN`. Nie chcemy zgadywać, czy działa. Chcemy zobaczyć dowód, że pakiet dotarł do właściwego interfejsu, trafił do oczekiwanego termu, a następnie został odrzucony.

**R1-vSRX — Operational Mode, odczyt konfiguracji i liczników**

~~~junos
show configuration interfaces ge-0/0/2 | display set
show configuration firewall family inet filter LAN-IN
show firewall filter LAN-IN
show interfaces ge-0/0/2 extensive
~~~

Pierwsze polecenie potwierdza, czy filter jest faktycznie podpięty do logicznego interfejsu i we właściwym kierunku. Drugie pokazuje kolejność termów. Trzecie — o ile dana implementacja wystawia liczniki w tym widoku — pozwala obserwować `SPOOFED` i `OTHER-PACKETS`. Czwarte daje statystyki interfejsu, ale bez wyciągania pochopnych wniosków: licznik błędów L1 nie jest dowodem odrzucenia przez firewall filter.

A teraz próba kontrolna. Najpierw wysyłamy ruch **dozwolony** z PC-A. Licznik `OTHER-PACKETS` powinien rosnąć, a `SPOOFED` nie. Potem, wyłącznie w odizolowanym labie, testujemy źródło 203.0.113.10 i obserwujemy wzrost `SPOOFED`. Nie musimy przy tym doprowadzić do skutecznego połączenia z serwerem: chcemy wykazać lokalne odrzucenie przez filtr.

Dla EX/QFX przy większych filtrach trzeba uwzględnić **zasoby TCAM/PFE**. Nie wszystkie match conditions i action modifiers są jednakowo wspierane na wszystkich modelach; sam fakt, że przykład istnieje w CLI Reference, nie jest gwarancją jego instalacji na każdym układzie forwarding. Jeśli counter nie rośnie, najpierw sprawdź family, interfejs, kierunek i fizyczną ścieżkę pakietu; dopiero później szukaj problemu z samym termem.

### Krótki przypadek: BGP działa, routing działa, lecz użytkownik nadal nie

Wyobraź sobie teraz trzy wyniki:

- `show route 10.20.20.0/24 exact` na R1 pokazuje aktywne BGP;
- `show route forwarding-table destination 10.20.20.20` wskazuje właściwy WAN next hop;
- `show firewall filter LAN-IN` ujawnia rosnący licznik dropów podczas prób PC-A.

Wniosek nie brzmi: „BGP jest zepsuty”. Wniosek brzmi: **mamy dowód, że przynajmniej część pakietów jest odrzucana przed zakończeniem forwarding na R1**. Teraz trzeba potwierdzić, że licznik dotyczy dokładnie testowanego strumienia, a nie równoległego ruchu. W razie potrzeby zbieramy pre/post delta na cichym labie zamiast opierać się na historycznej wartości licznika.

Na vSRX przechodzimy jeszcze jeden poziom. Jeśli filtr dopuszcza pakiet, weryfikujemy `security policies` i stan sesji. Jeśli sesja istnieje i licznik filtrów nie rośnie, sprawdzamy return path, endpoint firewall, NAT oraz przetwarzanie dalszego urządzenia. **Nigdy nie naprawiaj jednego etapu, dopóki nie wiesz, że właśnie w nim występuje usterka.**

**R1-vSRX — Operational Mode, spojrzenie na stateful forwarding**

~~~junos
show security policies from-zone LAN to-zone WAN
show security flow session source-prefix 10.10.10.10
show security flow session summary
show route 10.20.20.0/24 exact
~~~

Działająca security policy to nie uniwersalne „permit” dla wszystkiego: liczą się źródło, cel, strefy, kierunek inicjacji, application i realnie dopasowana reguła. Zwróć też uwagę, że sesje w SRX mogą zachowywać się inaczej niż nowo inicjowany ruch po zmianie konfiguracji. Testuj **nowe połączenie** i aktualną politykę.

### Przenosimy tę lekcję do świata Cisco

Na Catalyście zrobiłbyś `show access-lists`, sprawdził ACL na interfejsie i routing w CEF. Na Junos analogiczne pytania realizujesz przez `show firewall filter`, `show configuration interfaces` i `show route forwarding-table`. Dobrze przenosi się **metodologia**, nie nazwy poleceń. A kiedy pracujesz na SRX, dopisz do tej metodologii osobno `security policies`, `security flow session` i `host-inbound-traffic`.

*Źródła: [Firewall Filters Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-overview.html), [Firewall Filter Installation Status](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/display_firewall_filter_installation_status.html), [Security Policies Overview](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html).*

---

## 8.10. uRPF — kiedy router sprawdza nie dokąd, lecz skąd pakiet przyszedł [A]

Dotąd rozmawialiśmy głównie o adresie docelowym. Router patrzy na destination IP i szuka drogi do następnego hopa. Tymczasem pewnego dnia monitoring pokazuje pakiety przychodzące od operatora z adresem źródłowym należącym do naszej własnej sieci LAN. To wygląda podejrzanie: dlaczego ktoś z Internetu podaje się za komputer z oddziału?

Tu pojawia się **Unicast Reverse Path Forwarding (uRPF)**. Nie jest protokołem routingu ani kolejną BGP policy. To sposób sprawdzenia wiarygodności **źródła pakietu** na podstawie informacji o drodze powrotnej. Router niejako pyta: „Gdybym chciał wysłać coś do tego adresu źródłowego, którędy powinienem to zrobić?”.

W bardzo prostej sieci odpowiedź często zdradza oszustwo. Jeśli pakiet od źródła 10.10.10.50 przyszedł na WAN, ale droga do sieci 10.10.10.0/24 prowadzi przez lokalny LAN, źródło nie pasuje do oczekiwanej strony sieci.

### Strict — „przyszedłeś właściwymi drzwiami?”

W trybie **strict** uRPF sprawdzenie nie kończy się na samym istnieniu route. W klasycznym modelu interfejs, na którym pakiet się pojawił, musi być zgodny z dopuszczalną drogą zwrotną do jego źródła. To mocna ochrona na łączach, gdzie topologia jest symetryczna i dobrze znana.

### Loose — „czy w ogóle wiem, gdzie jest twoja sieć?”

W trybie **loose** nie wymagamy zgodności interfejsu wejściowego z najlepszą drogą powrotną. Wystarcza właściwa osiągalność źródła według zasad danej implementacji. Jest to mniej restrykcyjna kontrola, przydatna zwłaszcza tam, gdzie pakiety mogą przychodzić inną ścieżką niż droga wybierana przez router w przeciwnym kierunku. Zachowanie trasy domyślnej, routing instances oraz kwalifikacja ścieżek mogą zależeć od platformy i ustawień — nie zakładaj jednego uniwersalnego algorytmu na MX, EX, PTX i vSRX.

Teraz ważny scenariusz z sieci Enterprise. Oddział ma dwóch operatorów: **ISP-A** i **ISP-B**. Z polityk BGP wynika, że najlepsza droga *do źródła* konkretnego pakietu prowadzi przez ISP-A. Sam pakiet jednak przyszedł przez ISP-B. Nie musi to oznaczać ataku; czasem jest to wynik asymetrycznego routingu u operatorów albo świadomej polityki traffic engineering.

Jeśli na ISP-B włączysz strict uRPF bez analizy, możesz zacząć odrzucać zupełnie legalny ruch. Administrator widzi wtedy zdrowe BGP i poprawną trasę do destination, a aplikacje nie działają. Często próbuje naprawić to przez zmianę BGP export, choć problem występuje przy **walidacji źródła otrzymanego pakietu**.

```mermaid
flowchart TB
    P["Pakiet od źródła S<br/>przychodzi przez ISP-B"] --> R{"Czy istnieje dopuszczalny<br/>reverse path do S?"}
    R -->|nie| D["uRPF FAIL"]
    R -->|tak| M{"Jaki tryb?"}
    M -->|loose| L["PASS według reguł loose"]
    M -->|strict| Q{"Czy interfejs wejścia<br/>jest dopuszczalnym reverse path?"}
    Q -->|tak| A["PASS"]
    Q -->|nie| D
```

*Rysunek 8.7. Tekst alternatywny: pakiet przychodzi z ISP-B; router sprawdza route powrotną do źródła. Loose dopuszcza istnienie odpowiedniej ścieżki, strict dodatkowo porównuje dopuszczalny interfejs powrotu z ingress. Schemat autorski, celowo pomija platformowe warianty ECMP/feasible paths.*

### Konfiguracja: najpierw sprawdź model, potem składnię

Nie będziemy udawać, że vSRX zawsze zachowa się tak jak MX z konkretnym Trio PFE. W Junos istnieją warianty konfiguracji `rpf-check`, m.in. przy interfejsie logicznym w rodzinie `inet`. Dostępność trybów, akcji na błędzie, współpraca z filter-based forwarding i obsługa ECMP wymagają potwierdzenia w dokumentacji konkretnego modelu i release.

**MX-EDGE-LAB — Configuration Mode, schemat strict uRPF na wspieranym urządzeniu**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet rpf-check
~~~

**MX-EDGE-LAB — Configuration Mode, alternatywny wariant loose, nie stosuj równocześnie z poprzednim**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet rpf-check mode loose
~~~

**MX-EDGE-LAB — Operational Mode, weryfikacja przy problemie z ruchem z 10.30.30.10**

~~~junos
show configuration interfaces ge-0/0/0 | display set
show route 10.30.30.10
show route forwarding-table destination 10.30.30.10
show interfaces ge-0/0/0 extensive
~~~

Po pierwszej komendzie potwierdź, na którym interfejsie włączono ochronę. Po dwóch kolejnych — jak wygląda route do **źródła**, nie tylko do destination. Czwarta pomaga ocenić interfejs i dostępne statystyki. Jeżeli platforma wystawia dedykowane liczniki uRPF, odczytaj je według CLI Reference dla tego urządzenia. Nie oczekuj, że wszędzie ta sama nazwa licznika i to samo wyświetlanie będą dostępne.

Junos ma również warianty korzystające z **active-paths** i **feasible-paths**, dzięki którym ocena dozwolonej drogi powrotnej może uwzględniać inne ścieżki niż pojedyncza aktywna. Nie włączaj ich przez zgadywanie: to decyzja projektowa zależna od topologii i modelu zagrożeń. W niektórych implementacjach interakcja uRPF z FBF i routing instances sprawia, że nawet zaawansowany inżynier musi sprawdzić szczegółowy dokument platformy.

**Most z Cisco:** intuicja `ip verify unicast source reachable-via rx` odpowiada strict, a `reachable-via any` luźniejszej kontroli. Nie są to jednak gwarantowane odpowiedniki jeden do jednego przy ECMP, default route, VRF i szczegółach hardware.

*Źródła: [Understanding Unicast RPF (Routers)](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/topic-map/interfaces-configuring-unicast-rpf.html), [rpf-check CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rpf-check-edit-interfaces.html), [unicast-reverse-path](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/unicast-reverse-path-edit-routing-options.html).*

---

## 8.11. Od pomysłu do bezpiecznej zmiany — cały dzień pracy z policy i filtrami [A]

Nauka Junos nie kończy się wtedy, gdy potrafisz wpisać komendę. Kończy się wtedy, gdy potrafisz wprowadzić zmianę, udowodnić, że zadziałała, i bezpiecznie wrócić do stanu poprzedniego, jeżeli coś pójdzie nie tak.

Wyobraźmy sobie, że dostałeś ServiceNow Change. Wymaganie jest proste: „Do partnera WAN ma być reklamowany tylko LAN-A, a komputery w LAN-A nie mogą wysyłać ruchu ze źródła spoza firmowej adresacji”. Mamy dwa cele. Chcemy kontrolować **informacje o trasach BGP** i **pakiety wchodzące z LAN**. Gdy połączysz je w jedno ogólne zadanie „skonfigurować filtering”, łatwo zapomnisz o osobnych testach.

### Krok pierwszy: zanim dotkniesz konfiguracji

Najpierw pytasz: co dziś działa? Sprawdzasz istniejące BGP, aktywne trasy i reklamy, definicje oraz miejsca przyłożenia filtrów, security zones i policy na SRX. Zapytaj siebie: **które urządzenie ma podjąć decyzję?** Dla export odpowiada R1; dla import odpowiada R2; dla pakietu wchodzącego na LAN R1 — filtr przypięty do odpowiedniego ingress.

**R1-vSRX — Operational Mode, pre-check do zmiany**

~~~junos
show version
show interfaces terse
show bgp summary
show route 10.10.10.0/24 exact extensive
show route advertising-protocol bgp 192.0.2.2
show configuration protocols bgp | display set
show configuration policy-options | display set
show configuration firewall | display set
show configuration interfaces ge-0/0/2 | display set
show security zones
show security policies
~~~

Zanim napiszesz nową policy, obejrzyj też wszystkie **istniejące punkty odniesienia** do obiektów. Nazwa wspólnego `policy-statement` może być używana przez więcej niż jednego peera lub proces. Zmiana tego obiektu „tylko dla R2” może nieoczekiwanie zmodyfikować zachowanie kilku sąsiadów naraz. Dla bezpiecznej zmiany często lepiej stworzyć nową, jasno nazwaną politykę i podpiąć ją tylko tam, gdzie trzeba.

### Krok drugi: przewiduj wynik przed commit

Jeżeli dodasz na końcu policy `then reject`, spodziewasz się konkretnej rzeczy: żadna niezaakceptowana wcześniej kwalifikowana trasa nie będzie wyeksportowana. Jeżeli dodasz filtr `LAN-IN` z `BLOCK-SPOOF`, spodziewasz się wzrostu licznika po celowo nieprawidłowym pakiecie, ale nie po zwykłym ping z PC-A.

Zapisz takie przewidywania w MOP *przed zmianą*. To nie biurokracja: po wdrożeniu będziesz miał jasne kryterium sukcesu i od razu wykryjesz, że testowałeś nie ten kierunek albo nie ten prefix.

### Krok trzeci: wykorzystaj candidate config i bezpieczny rollback

Junos daje nam ogromną przewagę operacyjną: konfigurację kandydacką, porównywanie diff i commit z możliwością automatycznego cofnięcia. Nie traktuj tych narzędzi jak obowiązkowego rytuału, lecz jak części planu diagnostycznego. Na odizolowanym labie, z kontrolą konsoli, zmianę konfiguracji możesz aktywować warunkowo.

**R1-vSRX — Operational Mode, wejście do konfiguracji**

~~~junos
configure private
~~~

**R1-vSRX — Configuration Mode, po wprowadzeniu zaplanowanych zmian**

~~~junos
show | compare
commit check
commit confirmed 5
~~~

Tu zatrzymaj się i wykonaj zaplanowane **post-checki**. Nie uruchamiaj automatycznie kolejnego `commit check`, gdy `commit confirmed` już oczekuje: w Junos taka czynność może potwierdzić oczekującą zmianę. W tym czasie powinny dziać się **testy operacyjne**, nie mechaniczne ponawianie walidacji składni.

**R1-vSRX — Operational Mode, po warunkowym wdrożeniu**

~~~junos
show bgp summary
show route advertising-protocol bgp 192.0.2.2
show route 10.10.10.0/24 exact
show firewall filter LAN-IN
show security flow session summary
~~~

Jeżeli wyniki są zgodne z przewidywaniem i wykonano test ruchu, potwierdź zmianę. Jeśli nie — pozostaw mechanizm cofania lub wróć do znanej konfiguracji zgodnie z procedurą i dostępnością sesji. Nigdy nie opieraj rollbacku zmian w zarządzaniu na SSH, które sam możesz właśnie odciąć.

**R1-vSRX — Configuration Mode, tylko po pomyślnym post-check**

~~~junos
commit
~~~

### Krok czwarty: dowód dodatni i dowód ujemny

NOC często sprawdza wyłącznie, że po zmianie działa ping. To za mało dla whitelisty. Potrzebujemy dwóch rodzajów dowodu:

**Dodatni:** oczekiwany prefiks LAN-A jest reklamowany, R2 go przyjął, właściwa trasa jest aktywna, a ruch zgodny z wymaganiem przechodzi.

**Ujemny:** nieautoryzowany prefiks **nie** jest reklamowany, a spoofed packet powoduje wzrost właściwego licznika odrzucenia. Bez tego druga połowa wymagań biznesowych pozostaje nieprzetestowana.

Należy również rozróżnić „brak w `advertising-protocol`” od „R2 nie odebrał”. Gdy brak na wysyłającym R1, szukasz problemu eksportu albo kwalifikacji trasy. Gdy R1 reklamuje, a R2 nie ma aktywnej trasy, badanie przechodzi na odbiór, import, hidden status i wybór ścieżki. Brak aplikacji mimo poprawnego BGP kieruje nas do dataplane, filtrów, security sessions i return path.

### Krok piąty: pokaż wynik tak, by kolejny dyżurny nie zgadywał

Dobry work note po zmianie nie brzmi „BGP checked, looks fine”. Brzmi: „R1 direct 10.10.10.0/24 active; R1 advertising-protocol to 192.0.2.2 includes /24 and excludes 198.51.100.0/24; R2 route /24 active; PC-A ↔ PC-B traffic passed; spoof source caused LAN-IN SPOOFED counter increase; commit confirmed finalized”. To jest zestaw dowodów, a nie ocena nastroju administratora.

W następnym rozdziale OSPF i późniejszym BGP użyjemy tej samej dyscypliny. Zmieni się protokół, nie metoda myślenia.

*Źródła: [Junos CLI Configuration Workflow](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-cli-configuration.html), [Routing Policies and Firewall Filters Guide](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-routing-policies-firewall-filters-comparison.html).*

---

## 8.12. Ta sama policy, trzy różne protokoły: OSPF, IS-IS i BGP [S]

Do tej pory można było odnieść wrażenie, że polityki routingu dotyczą wyłącznie BGP. To dlatego, że BGP bardzo czytelnie pokazuje granicę między informacją przychodzącą a wychodzącą. W Enterprise równie ważne będą jednak OSPF i IS-IS, szczególnie na brzegu pomiędzy siecią kampusową, centrum danych i WAN.

Wyobraź sobie teraz nowy etap naszego projektu. Oddział ma router **R1**, a wewnątrz firmy działa domena OSPF. Zarząd chce, aby zewnętrzny router partnera dowiedział się o sieci aplikacyjnej 10.70.70.0/24, lecz nie otrzymał wszystkich tras statycznych i zarządzających. Pojawia się naturalne pytanie: **który element sieci jest właścicielem tej informacji i kto ma prawo wprowadzić ją do OSPF?**

### OSPF: nie pomyl rozgłaszania topologii z redystrybucją

Najpierw najprostsza sytuacja. R1 ma aktywny interfejs w OSPF area 0. Sąsiad OSPF może dowiedzieć się o stanie tego łącza przez normalne mechanizmy OSPF. Nie potrzebujesz do tego export policy typu „redistribute direct”, aby OSPF zaczął budować własną LSDB.

Teraz zmieniamy scenariusz. R1 ma **trasę statyczną**, której OSPF nie wywodzi ze swojej topologii. Chcemy ją świadomie wprowadzić jako external route. Tu właśnie przydaje się export policy.

**R-OSPF-EDGE — Configuration Mode, odrębny lab z działającym sąsiedztwem OSPF**

~~~junos
set routing-options static route 10.70.70.0/24 discard
set policy-options policy-statement REDISTRIBUTE-APP term APPLICATION from protocol static
set policy-options policy-statement REDISTRIBUTE-APP term APPLICATION from route-filter 10.70.70.0/24 exact
set policy-options policy-statement REDISTRIBUTE-APP term APPLICATION then metric 50
set policy-options policy-statement REDISTRIBUTE-APP term APPLICATION then accept
set policy-options policy-statement REDISTRIBUTE-APP term REST then reject
set protocols ospf export REDISTRIBUTE-APP
~~~

Zatrzymajmy się przy jednej pozornie niewinnej linijce. `static route ... discard` tworzy w tym przykładzie **kontrolowany prefix testowy**, który router może posiadać w RIB bez rzeczywistej sieci za nim. To wygodne w izolowanym labie, ale bardzo zły pomysł, jeśli nie rozumiesz konsekwencji: pakiety kierowane do tego prefixu zostaną odrzucone, jeżeli nie ma bardziej szczegółowej trasy. W produkcji wprowadzasz *rzeczywistą, działającą* drogę do aplikacji, a nie sztuczną trasę blackhole tylko po to, żeby OSPF wygenerował LSA.

Co zrobić po konfiguracji? Nie sprawdzaj jedynie, czy polecenie `export` istnieje. Zobacz najpierw aktywny static route, potem sąsiadów OSPF i dopiero wówczas ogłoszone external LSAs.

**R-OSPF-EDGE — Operational Mode, weryfikacja redystrybucji**

~~~junos
show route 10.70.70.0/24 exact
show ospf neighbor
show ospf database external
show configuration protocols ospf | display set
~~~

Jeżeli external LSA nie pojawia się, hipotezy dotyczą aktywności static route, poprawności dopasowania policy, procesu OSPF i kontekstu area. Jeżeli sąsiad w ogóle nie istnieje, nie ma sensu zaczynać diagnostyki od zmiany metric w polityce.

#### Dlaczego niektóre import reject w OSPF „nie działają”?

Teraz drugi telefon od administratora: „Chcę usunąć z RIB prefix wewnętrzny OSPF, więc napisałem import policy z `then reject`, ale router ciągle go posiada”.

W normalnym modelu OSPF nie możesz potraktować takiej polityki jak uniwersalnego ACL na dowolną trasę link-state. Junos pozwala wykorzystać OSPF import do pewnych celów, zwłaszcza obsługi **external routes** i priorytetu aktualizacji wybranych prefiksów, ale **reject dla trasy nonexternal może zostać zignorowany**. Producent opisuje to wprost. W ten sposób protokół zachowuje spójność niezbędną dla obliczeń SPF; odmienne decyzje o topologii w różnych routerach mogłyby prowadzić do blackhole.

W tym momencie zaczynasz rozumieć, po co istnieją specjalne mechanizmy między obszarami, agregacja, area types i kontrola external redistribution. Zamiast łamać LSDB import policy, projektujesz właściwy punkt redystrybucji i granice propagacji informacji. Szczegółowa anatomia LSAs i ABR pojawi się w rozdziale 9.

**Most z Cisco:** podobnie jak w Cisco OSPF, redystrybucja `redistribute static` nie jest tym samym co naturalna reklama łączy OSPF. Junos realizuje redystrybucję przez `protocols ospf export POLICY`. Jednak nie zakładaj, że Cisco `distribute-list` ma jeden prosty odpowiednik w dowolnym punkcie OSPF Junos.

*Źródło: [Configuring OSPF Routing Policy](https://www.juniper.net/documentation/us/en/software/junos/ospf/topics/topic-map/configuring-ospf-routing-policy.html).*

### IS-IS: inny protokół link-state, podobny problem z granicami informacji

Przenieśmy się do kampusu, w którym działa IS-IS Level 1/Level 2. Chcemy wprowadzić do domeny IS-IS tylko wybraną statyczną sieć aplikacyjną. Mechanizm jest znajomy: przygotowujemy nazwany `policy-statement`, a pod `protocols isis` przypinamy go jako export.

**R-ISIS-CORE — Configuration Mode, odrębny przykład na działającym IS-IS**

~~~junos
set policy-options policy-statement ISIS-APPS term APP from protocol static
set policy-options policy-statement ISIS-APPS term APP from route-filter 10.80.80.0/24 exact
set policy-options policy-statement ISIS-APPS term APP then accept
set policy-options policy-statement ISIS-APPS term REST then reject
set protocols isis export ISIS-APPS
~~~

Zwróć uwagę, że tu **nie dodajemy** trasy static ani bazowej konfiguracji IS-IS. Zakładamy, że laboratorium z rozdziału 10 ma aktywny prefix 10.80.80.0/24 i poprawne level, NET, rodziny adresowe oraz interfejsy. Policy nie wytworzy trasy, której nie ma, i nie zestawi za nas adjacency.

**R-ISIS-CORE — Operational Mode, gdzie szukać rezultatu**

~~~junos
show isis adjacency
show isis database extensive
show route 10.80.80.0/24 exact
show configuration protocols isis | display set
~~~

Jeśli jesteś zaskoczony obecnością `import` pod IS-IS, to dobrze — w dokumentacji można spotkać starsze opisy, które upraszczały lub negowały praktyczne zastosowania import policy w link-state. **W nowszym Junos istnieje `protocols isis import` (wprowadzony w Junos OS 17.1), z zastosowaniami obejmującymi m.in. kontrolę importu oraz priorytety tras**. Nie oznacza to jednak prawa do dowolnego usuwania tras wewnętrznych z widoku link-state. Producent wprost wskazuje, że `reject` dla non-external IS-IS route może zostać zignorowany. Potwierdź model i wersję zanim użyjesz tej funkcji w produkcji.

To ciekawa różnica między nauką do egzaminu a pracą inżyniera. W pamięci można mieć hasło „IS-IS import/export”. W prawdziwym troubleshooting trzeba wiedzieć **co podlega filtrowaniu, czy instalacja w RIB i spójność LSDB są tym samym oraz jakie ograniczenia ma dane wydanie**.

**Most z Cisco:** podejście z redystrybucją static do IS-IS pozostaje użyteczną analogią; szczegóły Level 1/2, metriki i przechodzenia informacji między poziomami zostawiamy do rozdziału 10.

*Źródła: [import (Protocols IS-IS)](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-edit-protocols-isis.html), [Prioritizing IS-IS Routes](https://www.juniper.net/documentation/us/en/software/junos/is-is/topics/example/example-configuring-routing-policy-to-prioritize-isis-routes.html), [Redistributing OSPF Routes into IS-IS](https://www.juniper.net/documentation/us/en/software/junos/is-is/ospf/topics/example/isis-redistributing-ospf-routes.html).*

### BGP: tu granica import/export staje się narzędziem codziennej pracy

W BGP historia wraca do początku rozdziału. R1 ma aktywną LAN-A, a R2 jest sąsiadem eBGP. Na R1 definiujemy **export whitelist**. Na R2 niezależnie od tego możemy stosować **import whitelist**, która pozwoli przyjmować tylko uzgodnione prefiksy. Dlaczego obie strony mogą chcieć własnych reguł? Bo bezpieczeństwo między systemami autonomicznymi nie powinno polegać na założeniu, że partner nigdy się nie pomyli.

**R1-vSRX — Configuration Mode, eksport z oddziału do centrali**

~~~junos
set policy-options policy-statement WAN-OUT term LAN-A from protocol direct
set policy-options policy-statement WAN-OUT term LAN-A from route-filter 10.10.10.0/24 exact
set policy-options policy-statement WAN-OUT term LAN-A then accept
set policy-options policy-statement WAN-OUT term REST then reject
set protocols bgp group TO-R2 export WAN-OUT
~~~

**R2-vSRX — Configuration Mode, odbiór tylko konkretnego prefiksu z oddziału**

~~~junos
set policy-options policy-statement FROM-R1 term LAN-A from route-filter 10.10.10.0/24 exact
set policy-options policy-statement FROM-R1 term LAN-A then accept
set policy-options policy-statement FROM-R1 term REST then reject
set protocols bgp group TO-R1 import FROM-R1
~~~

Zauważ dwie ważne rzeczy. Po pierwsze, te polityki są **niezależne**, choć opisują ten sam prefix. Po drugie, BGP może przyjmować policy na różnych poziomach konfiguracji (global, group, neighbor). Trzeba sprawdzić faktycznie obowiązujący kontekst dla konkretnego peera, a nie zakładać, że każdy sąsiad dziedziczy i łączy reguły według dowolnej intuicji.

**R1-vSRX — Operational Mode, dowód wyjścia**

~~~junos
show bgp summary
show route advertising-protocol bgp 192.0.2.2
~~~

**R2-vSRX — Operational Mode, dowód odbioru i użycia**

~~~junos
show route receive-protocol bgp 192.0.2.1
show route receive-protocol bgp 192.0.2.1 hidden
show route 10.10.10.0/24 exact extensive
~~~

Mamy więc trzy ważne wnioski z tej lekcji. **OSPF** potrafi stosować import/export, ale nie jest to dowolny filtr wewnętrznej LSDB. **IS-IS** ma zbliżone ograniczenia wynikające z charakteru link-state, z różnicami wsparcia zależnymi od wersji. **BGP** szczególnie naturalnie wykorzystuje polityki na granicach wymiany prefiksów i atrybutów. W każdej sytuacji punktem wyjścia powinno być pytanie: *jaki obiekt trasy oceniamy i na którym etapie protokołu?*

*Źródła: [Protocol Support for Import/Export](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/policy-protocol-support-import-export-policies.html), [Basic BGP Routing Policies](https://www.juniper.net/documentation/us/en/software/junos/bgp/topics/topic-map/basic-routing-policies.html).*
