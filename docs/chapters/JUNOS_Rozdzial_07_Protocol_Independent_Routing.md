# Rozdział 7 — Protocol-Independent Routing
## Gdy sama „najlepsza trasa” nie wystarcza: agregacja, generowanie tras, martians, RIB groups, load balancing i Filter-Based Forwarding

**Blueprint:** rozdział **7.1–7.8 [S]**, według [kanonicznego planu](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md).  
**Ścieżka:** JNCIA-Junos → **JNCIS-ENT** → JNCIP-ENT (kontekst troubleshootingowy).  
**Wydanie redakcyjne:** 18 listopada 2026; odniesienia do publicznej dokumentacji Juniper/HPE i inżynierskich pytań z Juniper Community.  
**Platformy:** MX/EX/QFX/SRX, zależnie od wsparcia modelu i release. **FBF, RIB groups i load balancing nie muszą działać identycznie na różnych PFE.**  
**Laboratorium:** projekt izolowanego EVE-NG, **PLANNED / NOT RUNTIME VALIDATED**. Przykłady konfiguracji to materiał dydaktyczny, nie zatwierdzone konfiguracje produkcyjne.

> **Umowa z czytelnikiem:** nie zaczniemy od suchej listy opcji `aggregate`, `generate` i `rib-group`. Najpierw postawimy Cię w sytuacji inżyniera Cisco, który musi zaprojektować i naprawić kilka realnych ścieżek ruchu na jednym Juniperze. Przy każdej technologii ustalimy: **dlaczego powstała → co zmienia w RIB/FIB → jak skonfigurować → jak udowodnić działanie → jaką ma pułapkę → jak bezpiecznie wywołać awarię w labie**.

---

## Prolog — dwa łącza WAN, dwa rodzaje ruchu i nieoczekiwany blackhole

Oddział biurowy ma zwykły Internet przez operatora A i dedykowany tunel do centrali przez operatora B. Sieć ma 200 prefiksów, które można ogłosić partnerowi w jednym podsumowaniu. Pracownicy biurowi powinni domyślnie korzystać z A, a ruch laboratorium testowego z B. W międzyczasie ktoś informuje, że „sieć 10.60.99.0/24 przestała działać po konfiguracji aggregate” i że na routerze znajdują się **te same connected routes w kilku RIB**.

Znasz Cisco. Możesz pomyśleć o:
- statycznej Null0 summary oraz BGP `aggregate-address`;
- PBR z route-map i `set ip next-hop`;
- VRF-Lite i route leaking;
- CEF ECMP;
- ACL/blokowaniu bogonów.

W Junos te problemy można rozwiązać, ale konstrukcje są inne. `aggregate route` nie zachowuje się identycznie jak `generated route`. `RIB group` nie jest poleceniem „skopiuj wszystko ze wszystkich VRF”. `load-balance per-packet` w wielu platformach **historycznie oznacza podział per-flow**, a nie losowe przeplatanie każdego kolejnego pakietu. `filter-based forwarding` zmienia **kontekst wyszukiwania trasy**, a nie jedynie pakietowi wpisuje adres bramy.

~~~mermaid
flowchart LR
  PC["Klienci biura"] --> ING["EX/MX/SRX — wejściowy L3 interface"]
  ING --> DEC{"Klasyfikacja FBF?"}
  DEC -->|"normalny ruch"| G["Główna tablica inet.0"]
  DEC -->|"wybrany ruch"| T["FBF-WAN-B.inet.0"]
  G --> A["Operator A"]
  T --> B["Operator B"]
  RIB["RIB group: udostępnienie\nwybranych connected"] -. "rozwiązanie next-hop" .-> T
  SUM["Aggregate/generate routes"] -. "informacje o sieciach" .-> G
~~~

Z tego diagramu wynika od razu ważne rozróżnienie: **agregacja i RIB groups dotyczą tworzenia/umieszczania tras**, a **FBF i load balancing dotyczą wyboru sposobu przekazywania pakietów**. To nie cztery różne nazwy jednej funkcji.

**Na pamięć:** `routing-options aggregate`, `routing-options generate`, `show route martians`, `rib-groups import-rib`, `forwarding-table export`, `then routing-instance`.  
**Do zrozumienia:** contributing route, pierwotny route vs duplicate w secondary RIB, resolved next-hop, hash per-flow, FBF forwarding instance.  
**Do przećwiczenia:** najpierw niezależnie aggregate, RIB group i FBF, dopiero potem ich połączenie w laboratorium 7.8.

---

## 7.1. Static, aggregate i generated routes [S]

### 7.1.1. Trzy sposoby zapisania intencji routingu

Załóżmy, że budujesz sieć dla firmy z dziesięcioma lokalizacjami. Każda ma podsieci `10.60.10.0/24`, `10.60.20.0/24` itd. Czy router w centrali ma ogłaszać partnerowi każdy prefiks oddzielnie, czy może jeden `10.60.0.0/16`?

Nie chodzi tylko o piękniejszą tabelę. Ogłaszanie setek szczegółów może zwiększyć rozmiar tablic u sąsiadów i liczbę zmian rozgłaszanych podczas awarii. **Summarization** może ukrywać szczegóły topologii przed resztą sieci.

W Junos trzy konstrukcje mają inne własności:

**Static route** — administrator bezwarunkowo zapisuje intencję `route destination → next-hop/discard/reject`. Route może jednak przestać być kwalifikowana, jeśli next-hop nie da się poprawnie rozwiązać. Sama static nie jest protokołem wymiany tras.

**Aggregate route** — tworzy trasę podsumowującą, której **aktywność domyślnie zależy od istnienia co najmniej jednej aktywnej, bardziej szczegółowej contributing route**. Typowym next-hop aggregate jest **reject**: dla adresu w podsumowaniu, który nie pasuje do bardziej szczegółowej realnej trasy, nie będziemy ślepo forwardować gdzieś dalej.

**Generated route** — także jest zależna od aktywnego contributing route, ale **może odziedziczyć użyteczny next-hop z primary contributing route**. Nadaje się m.in. do warunkowego generowania default/summary kierującego pakiety, a nie wyłącznie do tworzenia bezpiecznej granicy podsumowania.

Dokumentacja: [Configuring Route Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html), [Understanding Conditionally Generated Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/concept/policy-generated-routes.html).

### 7.1.2. Dlaczego aggregate nie ma „normalnego” next-hopa do serwera?

Przyjmijmy, że R1 ma:

- `10.60.10.0/24` → R2;
- `10.60.20.0/24` → R3;
- aggregate `10.60.0.0/16`.

Dla pakietu do `10.60.10.25` istnieje bardziej szczegółowa /24 i **LPM** kieruje go do R2. Dla pakietu do `10.60.99.25` nie ma żadnej /24. FIB dopasowuje więc /16. **Domyślny aggregate reject** zatrzymuje pakiet, zamiast puścić go na ślepo do operatora. To nie jest automatycznie awaria — często to właśnie pożądany mechanizm zapobiegania pętlom.

~~~mermaid
flowchart TD
  P["Pakiet docelowy"] --> LPM{"Czy istnieje bardziej szczegółowa trasa?"}
  LPM -->|10.60.10.25 pasuje /24| R2["R2 — realny next-hop"]
  LPM -->|10.60.20.25 pasuje /24| R3["R3 — realny next-hop"]
  LPM -->|10.60.99.25 pasuje tylko /16| REJ["Aggregate /16 — reject next-hop"]
  REJ --> ICMP["Zwykle ICMP unreachable\nzgodnie z platformą i polityką"]
~~~

**Most z Cisco:** przypomina świadomie ustawioną statyczną trasę podsumowującą do **Null0** i bardziej szczegółowe trasy do realnych next-hopów. Ale konfiguracja Junos `aggregate` jest **warunkowa od contributorów**, nie jest prostą kopią `ip route ... Null0`.

### 7.1.3. Pierwszy eksperyment: aggregate do 10.60.0.0/16

W osobnym laboratorium R1 zna aktywną trasę do `10.60.10.0/24`, np. static przez R2, i do `10.60.20.0/24` przez R3. Dopiero wtedy dodajemy aggregate.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], osobny izolowany LAB z działającymi contributorami**

~~~junos
set routing-options aggregate route 10.60.0.0/16
show | compare
commit check
~~~

**Urządzenie: R1-vSRX — Operational Mode, po zatwierdzeniu w odseparowanym LAB**

~~~junos
show route 10.60.0.0/16 extensive
show route 10.60.10.0/24
show route 10.60.20.0/24
show route protocol aggregate
show route forwarding-table destination 10.60.99.25
~~~

**Przewidywany mechanizm:** w miarę istnienia aktywnych route /24 aggregate /16 może być active. Jeśli contributorzy znikną, aggregate zwykle stanie się inactive / nie będzie instalowany, chyba że świadomie skonfigurujesz specjalne zachowanie. To sprawdzimy eksperymentem z usunięciem contributorów, bez zgadywania z jednego outputu.

**Gotcha #1:** samo `set routing-options aggregate route 10.60.0.0/16` **nie gwarantuje aktywnej trasy**, bo potrzebuje eligible contributor; `commit check` może być poprawny mimo braku contributor.

### 7.1.4. Kiedy aggregate przestaje być aktywny?

Wyobraź sobie awarię połączenia do R2, ale R3 nadal ma aktywną trasę /24. Aggregate /16 może pozostać aktywny dzięki R3 — i nadal będzie ogłaszany, jeśli export policy tego wymaga.

To prowadzi do trudnego pytania: **czy istnienie jednego contributor naprawdę dowodzi, że wszystkie /24 wewnątrz /16 są dostępne?** Nie. Reklamowany summary może powodować przyciąganie pakietów dla sieci, które nie istnieją, a aggregate reject odrzuci je lokalnie. Dlatego summarization **zmniejsza liczbę wpisów u sąsiadów, ale nie magicznie naprawia reachability wszystkich adresów w obszarze summary**.

Uwaga z dokumentacji producenta: sama agregacja **nie zmniejsza rozmiaru RIB na routerze, który ją wykonuje**. Ten router nadal musi znać contributors; oszczędność zyskują **sąsiedzi**, jeśli export policy ogłasza aggregate zamiast szczegółów. <https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html>.

### 7.1.5. Gotcha #2: aggregate vs generated mają ten sam protocol tag „Aggregate”

To jeden z prawdziwych dylematów, które pojawiają się w [dyskusji Juniper Community „Aggregate VS Generate routes”](https://community.juniper.net/discussion/aggregate-vs-generate-routes). Czytelnik widzi wygenerowaną trasę i oczekuje protokołu „Generated” w outputcie. Tymczasem Junos może pokazywać **oba rodzaje jako `[Aggregate/130]`**. Domyślna preference dla aggregate i generated wynosi **130**. Nie rozstrzygniesz typu wyłącznie po etykiecie route protocol.

**Urządzenie: R1-vSRX — Operational Mode, weryfikacja źródła konfiguracji i szczegółów**

~~~junos
show configuration routing-options aggregate | display set
show configuration routing-options generate | display set
show route protocol aggregate
show route 10.60.0.0/16 extensive
~~~

**Pytanie instruktora:** „Czy `[Aggregate/130]` dowodzi, że ktoś wpisał `routing-options aggregate`?” **Nie.** Sprawdź, czy istnieje `generate`, a następnie przeanalizuj next-hop i contributing routes. Oficjalna [preference CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/preference-edit-routing-options.html) potwierdza domyślne 130 dla obu rodzin.

### 7.1.6. Generated route: warunkowa droga, która potrafi nieść next-hop

Generated route ma również contributors, ale w domyślnym mechanizmie jej **next-hop jest wybierany na podstawie primary contributing route**. To zasadnicza różnica względem aggregate reject.

Wyobraź sobie, że R1 ma aktywną trasę:
- `10.70.10.0/24` przez `198.51.100.2`.

Chcesz utworzyć conditional generated `10.70.0.0/16`. Jeżeli contributor istnieje i jest kwalifikowany, generated może przejąć jego next-hop. Jeśli contributor zniknie, generated route nie będzie już kwalifikowana w zwykłym trybie.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], po przygotowaniu aktywnego contributor w LAB**

~~~junos
set routing-options generate route 10.70.0.0/16
show | compare
commit check
~~~

**Urządzenie: R1-vSRX — Operational Mode, po świadomej aktywacji w LAB**

~~~junos
show configuration routing-options generate | display set
show route 10.70.0.0/16 extensive
show route 10.70.10.0/24
show route forwarding-table destination 10.70.99.25
~~~

**Gotcha #3:** wygenerowane /16 może przekazywać pakiet do adresu **bez bardziej szczegółowej realnej trasy /24**, wykorzystując next-hop primary contributor. Nie myl więc generated z bezpiecznym blackhole summary. A jeżeli contributorów jest wielu, wybór **primary contributor** nie jest tym samym co FIB LPM dla pakietu; dokumentacja opisuje własny ranking contributorów (route preference, preference2 i porównania prefiksów). <https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/concept/policy-generated-routes.html>.

### 7.1.7. Contributor musi mieć znaczenie, a nie tylko pasować numerycznie

Przyjmijmy, że konfigurujesz `10.60.0.0/16`. Jako contributor może działać aktywna i zaakceptowana `10.60.20.0/24`. Ale `10.0.0.0/8` **nie** jest bardziej szczegółowym prefiksem pod /16, a `10.60.0.0/16` ma **taką samą długość jak aggregate**. Żaden z tych dwóch nie jest właściwym bardziej szczegółowym contributorem.

**Gotcha #4:** contributor musi być **aktywny i dłuższy** od aggregate/generate oraz mieścić się w jego zakresie. Policy może ograniczyć jego kwalifikację. Z reguły jeden route contributing przypisuje się do jednego aggregate na danym etapie; aktywny aggregate może z kolei być contributorem kolejnego, szerszego aggregate. Nie buduj pętli logicznych bez zrozumienia hierarchii.

**Urządzenie: R1-vSRX — Operational Mode, szukanie contributorów**

~~~junos
show route table inet.0 10.60.0.0/16 longer
show route 10.60.0.0/16 extensive
show route 10.60.10.0/24
show route 10.60.20.0/24
~~~

Jeżeli wariant `longer` nie pasuje do konkretnego wydania CLI, użyj `show route ?` lub pełnego odczytu RIB zamiast zgadywać.

### 7.1.8. Jak export policy zmienia sens całego przedsięwzięcia?

Powiedzmy, że masz summary `10.60.0.0/16` lokalnie. Czy BGP natychmiast ogłosi to światu? **Nie.** Samo stworzenie aggregate/generate w RIB nie jest równoznaczne z redystrybucją do protokołu. Potrzebujesz odpowiedniej **export policy** w BGP/OSPF/IS-IS lub właściwej konfiguracji.

Konfigurację policy rozwiniemy w rozdziale 8, ale już teraz możesz zrozumieć przyczynę:
- route istnieje lokalnie;
- może być active;
- policy decyduje, czy powstanie ogłoszenie;
- peer może je przyjąć, odrzucić lub preferować inne.

**Urządzenie: R1-vSRX — Operational Mode, tylko jeśli istnieje BGP neighbor do testowej sieci**

~~~junos
show route 10.60.0.0/16
show route advertising-protocol bgp 198.51.100.2
~~~

Jeżeli BGP nie jest uruchomiony, wynik drugiego polecenia **nie jest testem poprawności aggregate**. W osobnym BGP labie będziemy porównywać ogłoszone prefiksy przed i po export policy.

### 7.1.9. Aggregate, generated i static — po objaśnieniu różnice stają się proste

| Własność | Static | Aggregate | Generated |
|---|---|---|---|
| Typowy powód użycia | Jawna ścieżka do next-hop | Podsumowanie wielu bardziej szczegółowych route | Warunkowe utworzenie route z next-hop opartym o contributor |
| Wymaga aktywnego contributor | Nie | Tak, w typowym trybie | Tak, w typowym trybie |
| Domyślna preference | 5 | 130 | 130 |
| Typowy forwarding action | next-hop / discard / reject | **reject** | **next-hop primary contributor** |
| Sama ogłasza się do BGP | Nie | Nie | Nie |
| Ryzyko | stale/unresolved next-hop | przyciąganie ruchu do nieobsługiwanych podprefiksów | niezamierzony forwarding na podstawie primary contributor |

**Na pamięć:** `aggregate` i `generate` są warunkowe, obie z preference 130; aggregate zwykle reject, generated dziedziczy next-hop.  
**Do zrozumienia:** summarization działa dla sąsiadów dopiero po polityce eksportu.  
**Do przećwiczenia:** obserwuj stan aggregate/generate z jednym contributorem, drugim, żadnym i niekwalifikowanym next-hop.

---

## 7.2. Martian addresses — co Junos uznaje za nierealne lub niepoprawne informacje o trasach? [S]

### 7.2.1. Nie wszystko, co wygląda jak IPv4, jest sensowną trasą

Operator BGP dzwoni: „Od peer dostajemy prefix `127.10.20.0/24`. Dlaczego Junos ignoruje tę sieć, skoro filtr importu go nie blokuje?”. Odpowiedź jest głębsza niż konfiguracja policy.

Niektóre zakresy są z definicji specjalne: loopback, multicast, eksperymentalne, część zakresów specjalnego zastosowania. Junos utrzymuje mechanizm **martian routes** — adresów/prefiksów, których routing information jest domyślnie ignorowana/niekwalifikowana zgodnie z regułami.

To **nie jest** ta sama funkcja co:
- firewall filter blokujący pakiety w dataplane;
- BGP prefix-list odrzucający ogłoszenia sąsiada;
- lista prywatnych adresów RFC1918;
- narzędzie antyspoofingowe uRPF;
- security policy SRX.

### 7.2.2. Cisco ↔ Junos: „bogon” nie jest jednym przyciskiem bezpieczeństwa

Cisco może filtrować bogon prefixes przez prefix-list + route-map i ograniczać spoofing przez uRPF lub ACL. Junos martian route recognition jest **kontrolą sensowności informacji routingu**, ale **nie zastępuje ochrony ingress traffic**.

**Gotcha #5:** `10.0.0.0/8`, `172.16.0.0/12` i `192.168.0.0/16` **nie są automatycznie martian tylko dlatego, że są prywatne**. W Enterprise prywatne adresy są standardową podstawą routingu. Natomiast `127.0.0.0/8` to loopback — włączenie routingu do niego byłoby wyjątkowo podejrzane.

### 7.2.3. Co naprawdę zawiera lista domyślnych martians?

Nie zgadujemy; Juniper udostępnia komendę:

**Urządzenie: R1-vSRX — Operational Mode, odczyt wyłącznie informacyjny**

~~~junos
show route martians
show route martians table inet.0
show configuration routing-options martians | display set
~~~

W dokumentacji znajdziesz przykłady takich reguł jak:
- `0.0.0.0/0 exact — allowed`;
- `0.0.0.0/8 orlonger — disallowed`;
- `127.0.0.0/8 orlonger — disallowed`;
- `224.0.0.0/4 exact — disallowed`;
- `240.0.0.0/4 orlonger — disallowed`;
- dodatkowe zakresy i IPv6 special-use, zależnie od platformy i wydania.

Źródło producenta: [Recognize Martian Addresses for Routing](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/recognize-martian-addr-routing.html), [show route martians](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-martians.html).

**Gotcha #6 — autentyczne pytanie z Juniper Community:** [„Why is the default route in the default Martian table?”](https://community.juniper.net/discussion/why-is-the-default-route-in-the-default-martian-table). To logiczne, że student widzi w martians wpis `0.0.0.0/0 exact` i myśli: „default route jest zakazana!”. **Nie.** Trzeba odczytać również **`allowed`**, a reguła `0.0.0.0/8 orlonger disallowed` nie obejmuje `/0` — to inna długość prefiksu. Default route jest dopuszczalna. Takie podchwytliwe pytania uczą precyzji czytania CLI.

### 7.2.4. Exact vs orlonger — jeden parametr decyduje o znaczeniu

Znasz Cisco prefix-list `le` / `ge`. W Junos słowa `exact` i `orlonger` opisują **zbiór prefiksów**, do których stosuje się reguła.

- `127.0.0.0/8 exact` dotyczyłoby tylko tego prefiksu /8.
- `127.0.0.0/8 orlonger` obejmuje /8 oraz wszystkie dłuższe prefiksy wewnątrz /8, np. `127.1.2.0/24`.
- `0.0.0.0/0 exact allow` dopuszcza default /0, ale **nie** oznacza dopuszczenia każdej bardziej szczegółowej trasy świata.

**Most z Cisco:** różnica `ip prefix-list permit 0.0.0.0/0` vs `ip prefix-list permit 0.0.0.0/0 le 32`. To błędy podobnej kategorii: drobna różnica w dopasowaniu prefiksu zmienia bardzo dużo.

### 7.2.5. Czy możemy dodać własny martian?

Można. Jednak zanim administracyjnie zablokujesz routing informacji do jakiegoś prefiksu, upewnij się, że **nie jest to aktywna sieć firmy** ani nie będzie potrzebna do tunelu, management czy BGP. Zmiana martians to **kontrola nad kwalifikacją tras**, a nie filtr pojedynczej usługi.

Przykład **czysto demonstracyjny**, bez wykonywania na produkcyjnym routerze: w odizolowanym labie możemy jawnie zablokować przyjmowanie informacji routingu do prefiksu dokumentacyjnego `198.18.0.0/15` na poziomie martians.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], SZABLON izolowanego eksperymentu martian**

~~~junos
set routing-options martians 198.18.0.0/15 orlonger
show | compare
commit check
~~~

**Gotcha:** istnieją modele, wydania i sytuacje, w których lokalne/direct trasy specjalnego przeznaczenia i importowane dynamicznie prefiksy zachowują się inaczej. **Nie zakładaj, że ta jedna reguła zablokuje każdy transit packet do tego IP.** Dowód obejmuje obserwację RIB i protokołu, a filtracja traffic to osobny mechanizm firewall filter/security policy.

Nie dodajemy takiego polecenia do głównej topologii 7.8, aby nie zepsuć sąsiedztw i testów FBF.

### 7.2.6. Wyjątek `allow`: dlaczego nie jest bezpiecznym „napraw problem z prefixem” 

Dokumentacja Junipera opisuje możliwe odblokowanie pewnych zakresów np. Class E przez:

**Urządzenie: R1-vSRX — Configuration Mode [edit private], WYŁĄCZNIE ilustracja SKŁADNI, NIE wykonywać w sieci Enterprise**

~~~junos
set routing-options martians 240.0.0.0/4 orlonger allow
~~~

**Nie polecam tego jako operacyjnego fix**. To, że Junos pozwala technicznie zmienić kwalifikację martian, nie oznacza, że Internet, systemy operacyjne hostów, middleboxy i firmware zaakceptują ten zakres. A otwarcie nieoczekiwanego routingu może mieć konsekwencje bezpieczeństwa i kompatybilności.

**Na pamięć:** `show route martians`; exact vs orlonger; allowed vs disallowed.  
**Do zrozumienia:** martian route ≠ firewall filter ≠ RFC1918 bogon.  
**Do przećwiczenia:** przeczytaj wynik `show route martians` i wyjaśnij, dlaczego `0.0.0.0/0` jest dozwolona, ale `127.2.0.0/16` jest specjalna.

