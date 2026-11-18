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



---

## 7.3. Routing instances — rozwinięcie konfiguracji i diagnostyki [S]

### 7.3.1. Routing instance to oddzielna mapa dróg. A FBF to przekazanie paczki do innego dyspozytora.

Rozdział 6 nauczył nas, że tablice `inet.0` i `LAB-IOT.inet.0` nie są jednym wielkim katalogiem tras. Każda ma osobny wybór active route, własny lookup i własną ścieżkę rozwiązywania next-hop.

Wyobraź sobie wielkie centrum dystrybucyjne z dwoma dyspozytorami:
- dyspozytor A prowadzi standardowy transport przez operatora A;
- dyspozytor B ma kontrakt na transport specjalny przez operatora B.

Kurier przywozi paczkę i na wejściu ktoś stempluje ją „obsłuż u dyspozytora B”. To właśnie intuicja **Filter-Based Forwarding**: najpierw klasyfikujesz pakiet, potem wybierasz **inną tablicę**, w której znajduje się trasa przez właściwego operatora.

Żeby jednak dyspozytor B umiał znaleźć **swoją lokalną bramę**, musi mieć wiedzę, gdzie znajduje się jego next-hop. Tą wiedzą zajmiemy się w 7.4: **RIB groups**.

### 7.3.2. Cztery rodzaje routing instances, które łatwo pomylić

W Junos istnieje wiele `instance-type`. Najważniejsze dla tego rozdziału:

**`virtual-router`:** niezależna instancja routingowa, z własnymi interfejsami i protokołami. Bliska Cisco VRF-Lite. Jeśli chcesz zrobić osobną sieć routowaną dla OT, to jest naturalny punkt startowy.

**`vrf`:** konkretny typ instancji często wykorzystywany w L3VPN/PE, z route distinguisher, route targets i eksportem/importem tras. **Nie** jest słowem wymiennym z każdym Junos virtual-router.

**`forwarding`:** specjalna instancja do wyboru ścieżki przekazywania, popularna przy FBF/PBR. Często nie przypisujesz do niej klasycznego fizycznego ingress interface; pakiety są **kierowane tam przez firewall filter action**. Jest osobnym kontekstem forwarding, ale nie projektuj jej jak pełnego tenant VRF z wszystkimi funkcjami dowolnego protokołu.

**`no-forwarding`:** stosowana, gdy potrzebujesz odrębnego kontekstu informacji routingowej, ale nie standardowego forwarding dataplane. Nie będzie automatycznym zamiennikiem FBF.

Źródło: [Routing Instances Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html).

### 7.3.3. FBF-ALT: routing instance, której nie przypiszemy zwykłym `interface`

W głównym labie 7.8 chcemy, aby standardowy ruch miał default przez operatora A, a pakiety od wybranego klienta przez operatora B. Dla uproszczenia:
- R1 → operator A: `198.51.100.2` przez `ge-0/0/0`;
- R1 → operator B: `198.51.100.6` przez `ge-0/0/2`.

Przygotujemy specjalną forwarding instance:

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], izolowany lab, po zweryfikowaniu wersji i NIC**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 0.0.0.0/0 next-hop 198.51.100.6
show | compare
commit check
~~~

Na tym etapie **nie ma jeszcze FBF filter**. Nie twierdź, że klient wysyła ruch przez B. Stworzyliśmy tylko alternatywną **tabelę decyzyjną**. Dopiero wejściowy filter w 7.6 skieruje wybrane pakiety do `FBF-ALT.inet.0`.

**Urządzenie: LAB-R1-vSRX — Operational Mode, po aktywacji na nowym LAB**

~~~junos
show route table FBF-ALT.inet.0
show configuration routing-instances FBF-ALT | display set
show route table inet.0
~~~

**Pytanie instruktora:** czy `FBF-ALT.inet.0` musi od razu zawierać connected route do `198.51.100.4/30`, w której leży next-hop `198.51.100.6`? **Nie.** Samo utworzenie forwarding instance i statycznej default nie musi skopiować do niej połączonych tras z `inet.0`. To kluczowy brakujący element, który naprawimy RIB group.

### 7.3.4. Problem next-hop resolution — tabela wie dokąd, ale nie wie, jak tam dojść

Jeśli forwarding instance zna:
- `0.0.0.0/0 → 198.51.100.6`,

ale nie ma:
- `198.51.100.4/30 → ge-0/0/2.0`,

to nie ma pełnej instrukcji egress do lokalnego next-hop. To trochę jak dyspozytor wiedzący „oddaj do przewoźnika B”, lecz nieznający jego rampy załadunkowej.

**Gotcha #7:** obecność default w forwarding instance **nie dowodzi**, że jej next-hop jest poprawnie rozwiązany. Kiedy zwykły `inet.0` ma connected do ISP-B, nie oznacza to automatycznie, że drugi kontekst również go widzi. Przy rozwiązywaniu tej pułapki często potrzebujesz **`rib-group` dla interface routes**. Oficjalny [FBF Example](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html) pokazuje dokładnie ten związek.

### 7.3.5. Czy forwarding instance musi mieć swój interfejs L3?

W typowym przykładzie FBF **nie przypisujesz wejściowego interfejsu do FBF-ALT**. Zostaje on w podstawowym kontekście, a stateless filter na jego **ingress** wybiera dla określonego ruchu alternatywną tablicę. Egress next-hop może być przez interfejs należący do głównej RIB, dlatego przekazywanie connected interface routes do FBF tabeli ma sens.

W przeciwieństwie do tego w `virtual-router`/VRF zwykle świadomie przypisujesz interfejsy do instancji, aby każdy ruch z tej sieci korzystał z jej tabeli domyślnie.

~~~mermaid
flowchart TD
  P["Pakiet z ge-0/0/1.0"] --> FILTER{"Input firewall filter\nmatch source"}
  FILTER -->|"192.0.2.10/32"| ALT["FBF-ALT.inet.0\n0/0 via B"]
  FILTER -->|"pozostałe źródła"| BASE["inet.0\n0/0 via A"]
  ALT --> RESOLVE["Czy FBF-ALT zna connected\n198.51.100.4/30?"]
  RESOLVE -->|"tak"| B["ge-0/0/2 → operator B"]
  RESOLVE -->|"nie"| BAD["unresolved next-hop"]
  BASE --> A["ge-0/0/0 → operator A"]
~~~

### 7.3.6. Routing context a security zone: uważaj na SRX

Na SRX możesz jednocześnie mieć:
- routing instance: **wybór trasy**;
- firewall filter: **stateless classification**;
- security zones/policies: **stateful permit/deny**;
- NAT: **zmiana adresacji**.

To nie są cztery warianty jednej reguły. Stworzenie FBF nie zwalnia z security policy. Jeżeli testujesz FBF na vSRX, weryfikuj **czy dana platforma/wersja obsługuje konkretny typ PBR/FBF i jego interakcję z flow/NAT**. Nie przenoś bez testu przykładu z MX/QFX na SRX.

**Most z Cisco:** PBR route-map na L3 Catalyst i PBR na firewallu stateful nie muszą zachowywać się identycznie z powodu NAT/state/zone classification. Podobnie Junos. [FBF Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html).

### 7.3.7. Diagnoza „routing-instance istnieje, ale ruch nie idzie”

**Urządzenie: LAB-R1-vSRX — Operational Mode, dwie tabele i ścieżka do ISP-B**

~~~junos
show configuration routing-instances FBF-ALT | display set
show route table FBF-ALT.inet.0
show route table inet.0
show route 198.51.100.6
show arp
show security flow session
~~~

Najpierw sprawdzasz, czy route jest w odpowiedniej table i czy ma resolved next-hop. Dopiero potem filter, policy i host traffic. Jeżeli route jest hidden, **nie proponuj jeszcze globalnego restartu routingu**.

**Na pamięć:** `virtual-router` oddziela routing interfejsów, `forwarding` jest specjalną instancją do FBF.  
**Do zrozumienia:** klasyfikacja pakietu wybiera tablicę; same tablice nie przekazują sobie informacji automatycznie.  
**Do przećwiczenia:** stwórz `FBF-ALT.inet.0` bez RIB group, zobacz stan default i next-hop, potem uzupełnij interface routes.

---

## 7.4. RIB groups — precyzyjne udostępnianie tras do wielu tabel [S]

### 7.4.1. Trzy różne sposoby na „route leaking”, tylko jeden omawiamy teraz

Architekt mówi: „Chcę, żeby FBF-ALT wiedziała, że 198.51.100.4/30 jest bezpośrednio podłączona do R1”. Junior proponuje: „Skopiujmy całe `inet.0` do tej tabeli”. To mało precyzyjne i potencjalnie niebezpieczne.

W zależności od celu możesz:
1. użyć **RIB group** przy imporcie wybranych tras (np. connected);
2. użyć statycznych tras/odpowiedniej instancji i ręcznego route leaking;
3. użyć polityk import/export, instance-import, VPN mechanisms lub FBF, zależnie od architektury.

W tym rozdziale skupiamy się na **RIB group**. To nie jest automatyczna dwukierunkowa „synchronizacja wszystkich tabel” i nie należy traktować jej jak zwykłej kopii pliku.

### 7.4.2. Dwie tablice, jedno źródło trasy

Wyobraź sobie dwie tablice magazynowe. Pierwsza jest główna i przechowuje informacje z konkretnego źródła. Decydujesz, że część z nich będzie równocześnie widoczna w drugiej. **Pierwsza tabela pozostaje primary**, a druga otrzymuje dodatkową kopię przy procesie importu tras.

W Junos:
- **`rib-groups NAME import-rib [ PRIMARY SECONDARY ]`** określa zestaw tabel;
- **pierwsza RIB na liście jest primary**;
- kolejne to secondary;
- musisz **przypiąć RIB group do źródła informacji**, np. `routing-options interface-routes rib-group inet NAME` dla IPv4 direct/interface routes;
- opcjonalna **import-policy** ogranicza, które routes faktycznie trafią do secondary.

Nie jest to to samo co routing-policy `export` do BGP neighbor. Tutaj słowo „import” oznacza **umieszczenie informacji z procesu routingu w RIB**, nie otrzymanie dowolnego pakietu od sąsiada.

Źródła: [Introduction to RIB Groups — Juniper Technical Note](https://www.juniper.net/documentation/us/en/software/nce/rg-understanding-tn/rg-understanding-tn.pdf), [rib-groups CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html), [import-rib](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html).

### 7.4.3. Konfiguracja: udostępniamy connected routes do FBF-ALT

W naszym labie primary IPv4 RIB to `inet.0`, a secondary `FBF-ALT.inet.0`.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], wyłącznie izolowany LAB**

~~~junos
set routing-options rib-groups COPY-DIRECT import-rib [ inet.0 FBF-ALT.inet.0 ]
set routing-options interface-routes rib-group inet COPY-DIRECT
show | compare
commit check
~~~

Dwa wiersze mają różne zadania:
- pierwszy **definiuje** RIB group i jej kolejność;
- drugi **wiąże** ją z IPv4 **interface routes**.

**Gotcha #8:** samo stworzenie `rib-groups COPY-DIRECT` nie powoduje przenoszenia wszystkich tras. Bez przypisania do appropriate routing source grupa może pozostawać niewykorzystana.

**Gotcha #9:** kolejność **`inet.0` jako pierwsza** ma znaczenie. Przenoszenie primary na losową secondary nie jest kosmetyczne — niektóre konfiguracje naruszają warunki family/primary route i nie będą działać.

### 7.4.4. Dlaczego kopiujemy connected, a nie od razu cały Internet?

Do rozwiązania next-hopa `198.51.100.6` potrzebujemy przede wszystkim informacji:
- `198.51.100.4/30` jest connected;
- egress = `ge-0/0/2.0`.

Nie potrzebujemy na początek:
- setek tras BGP;
- osobistych host routes do interfejsów management;
- informacji o wszystkich VPN;
- dynamicznych routes z innych tenantów.

**RIB group jest mechanizmem o potencjalnie szerokim skutku.** Może zmienić dostępność prefiksów w innych instancjach, a tym samym semantykę izolacji i security. Dlatego w produkcji stosuj świadomy zakres oraz import-policy, gdy nie możesz bezpiecznie udostępnić wszystkich interface routes.

### 7.4.5. Import-policy — gdy chcesz kopiować tylko jeden uplink

Przykład wyjaśnia **sam pomysł polityki**. Uwaga: matching candidate routes i obsługa import-policy do RIB group mogą być zależne od platformy; test musi pokazać, że wybrana implementacja rzeczywiście skopiowała oczekiwany prefix.

Chcemy udostępnić tylko `198.51.100.4/30`, nie całe `inet.0`. Potrzebujemy routing policy, która akceptuje właściwy route i odrzuca pozostałe secondary imports.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], OPCJONALNY eksperyment z ograniczeniem leaking**

~~~junos
set policy-options policy-statement RG-ONLY-WAN-B term ALLOW from route-filter 198.51.100.4/30 exact
set policy-options policy-statement RG-ONLY-WAN-B term ALLOW then accept
set policy-options policy-statement RG-ONLY-WAN-B term OTHER then reject
set routing-options rib-groups COPY-DIRECT import-policy RG-ONLY-WAN-B
show | compare
commit check
~~~

**Szczególny gotcha:** import-policy przy RIB group **nie może bezmyślnie wyeliminować primary routes**, od których zależy podstawowy forwarding. Dokumentacja Junipera opisuje, jak i gdzie działa filtr importu, a różne platformy mają dodatkowe ograniczenia. W produkcji nie stosuj tej policy bez weryfikacji, czy normalny `inet.0` pozostał nietknięty. W labie najpierw porównaj oba RIB i test klienta, dopiero uznaj eksperyment za poprawny.

### 7.4.6. Jak weryfikować RIB group?

**Urządzenie: LAB-R1-vSRX — Operational Mode, read-only porównanie przed/po**

~~~junos
show configuration routing-options rib-groups | display set
show configuration routing-options interface-routes | display set
show route table inet.0
show route table FBF-ALT.inet.0
show route table FBF-ALT.inet.0 198.51.100.4/30
show route table FBF-ALT.inet.0 198.51.100.6
~~~

**Przewidywany rezultat:** po przypięciu `COPY-DIRECT` tabela `FBF-ALT.inet.0` powinna mieć potrzebną informację o sieci connected do operatora B, umożliwiając rozwiązanie next-hopa dla swojej default route. **Nie oznacza to**, że policja FBF jest aktywna, że SRX akceptuje sesje ani że provider B ma drogę zwrotną.

### 7.4.7. Gotcha platformowy: RIB group na EX nie zawsze kopiuje to, co kopiuje na MX/vSRX

CLI Reference zawiera istotną uwagę: **na niektórych EX Series RIB groups pomiędzy tablicami mogą być ograniczone do tras dynamicznych**. W praktyce może to uniemożliwić kopiowanie konkretnego rodzaju routes tak samo jak na MX.

W tej części **nie obiecujemy jednolitego efektu na wszystkich EX/QFX/MX/SRX**. Zawsze sprawdzaj [Feature Explorer](https://apps.juniper.net/feature-explorer/) i dokumentację danego modelu. Nasz lab planujemy na **vSRX**, a i tam trzeba potwierdzić działanie w konkretnym release.

Źródło: [import-rib](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html) — uwaga o EX Series.

### 7.4.8. Gotcha z prawdziwej Technical Note: route w secondary nie jest niezależną kopią

Jeżeli primary route zostaje usunięta, odpowiednia secondary route importowana przez RIB group może także zostać usunięta. **RIB group nie tworzy dwóch autonomicznych źródeł prawdy**, tylko pozwala temu samemu procesowi wprowadzać odpowiednie routes do kilku tablic.

To praktycznie oznacza, że jeśli `ge-0/0/2` straci connected `198.51.100.4/30`, **FBF-ALT** może utracić resolution next-hopa, nawet jeśli jej statyczna default route pozostaje zapisana w config. Nie „naprawisz” takiej awarii dopisaniem kolejnej kopii do secondary bez odtworzenia rzeczywistego next-hopa.

### 7.4.9. RIB group vs normalny route leaking do security tenantów

Jeśli klient IoT musi rozmawiać z jedną usługą management, nie kopiuj wszystkich tras `inet.0` do jego virtual-router. Łatwo wtedy nieumyślnie umożliwić komunikację wbrew HLD.

Wybierz:
- **co ma być osiągalne** (konkretne prefiksy);
- **w którą stronę**;
- **jak zostanie rozstrzygnięta droga powrotna**;
- **co z kontrolą ruchu** (firewall filter, SRX security policy, ACL/NAC);
- **jak zweryfikujesz, że nie ma niepożądanego leaking**.

To rozróżnienie ma ogromne znaczenie w produkcyjnej infrastrukturze typu Campus LAN/OT.

### 7.4.10. Mini-egzamin podczas wykładu

**Pytanie 1.** Utworzyłeś `rib-groups COPY-DIRECT import-rib [ inet.0 FBF-ALT.inet.0 ]`, ale w secondary nic się nie zmieniło. Co sprawdzasz najpierw?  
**Odpowiedź:** czy grupa została **przypięta do źródła importu**, np. `interface-routes rib-group inet COPY-DIRECT`; czy w ogóle jest aktywna connected route; czy platforma wspiera dany typ route i czy import-policy jej nie wyklucza.

**Pytanie 2.** W `FBF-ALT.inet.0` widzisz default via `198.51.100.6`, lecz brak connected do jego /30. Co oznacza to dla dataplane?  
**Odpowiedź:** nie możesz zakładać rozwiązania next-hop. Najpierw napraw interface route visibility, potem sprawdź FIB i forwarding.

**Pytanie 3.** Czy RIB group sama ogłasza route do OSPF/BGP sąsiada?  
**Odpowiedź:** nie. Zmiana tego, **w jakiej tablicy występuje route**, nie jest automatycznie ogłoszeniem na sesji protokołu; export policy/protocol config to osobna decyzja.

**Na pamięć:** `import-rib [ PRIMARY SECONDARY ]` i przypięcie do właściwego routing source.  
**Do zrozumienia:** RIB group to import do kilku tabel, nie pełny bi-directional VRF bridge.  
**Do przećwiczenia:** porównaj connected routes w `inet.0` i `FBF-ALT.inet.0` oraz rozwiązywanie `198.51.100.6` przed i po RIB group.



---

## 7.5. Load balancing — kiedy dwie równe drogi nie oznaczają dwóch użytych łączy [S]

### 7.5.1. RIB zna dwa next-hopy, ale na jednym łączu widzisz zero ruchu

W firmie uruchomiono dwa równorzędne uplinki. Monitoring pokazuje, że ISP-A ma 800 Mb/s, a ISP-B jedynie kilka Mb/s. Inżynier mówi: „Jest ECMP, powinniśmy mieć 50/50. Juniper coś źle liczy”.

Ale co dokładnie mierzyliśmy?
- Czy **oba next-hopy** należą do aktywnej drogi?
- Czy **oba** są zainstalowane w forwarding table?
- Czy mamy wystarczającą liczbę **różnych przepływów**, żeby hash mógł rozłożyć obciążenie?
- Czy to jeden backup po BFD, czy prawdziwy ECMP?
- Czy mamy podobny capacity obu łączy i czy nie działa policy przekierowująca część ruchu?

W Cisco rozróżniasz RIB, CEF load sharing i EtherChannel hashing. Junos też ma kilka warstw decyzji. **Samo „dwa łącza up” nie oznacza ECMP.**

### 7.5.2. Co to jest ECMP, a co nim nie jest?

**Equal-Cost Multi-Path (ECMP)** dotyczy sytuacji, w której **kilka kwalifikowanych next-hopów** może prowadzić do tej samej docelowej trasy przy równoważnym koszcie. Dopiero wtedy forwarding może wykorzystywać więcej niż jeden path.

Nie jest to:
- floating static primary/backup, gdzie tylko jedna droga ma pierwszeństwo;
- dwa różne prefiksy o innej długości (tam najpierw LPM);
- dwa uplinki Ethernet połączone w jeden LAG/ae (oddzielny mechanizm L2/L3 hashing);
- dwa BGP route candidates, które nie zostały dopuszczone do multipath;
- automatycznie równy podział bajtów 50/50.

**Most z Cisco:** Catalyst CEF ECMP i Junos PFE multipath są analogiczne w celu, ale nie w ustawieniach domyślnych, hash fields i możliwej liczbie ścieżek.

### 7.5.3. Niezwykły gotcha #10: `load-balance per-packet` w Junos to często per-flow

Z nazwy wynika, że Juniper będzie przerzucać pakiet 1 przez ISP-A, pakiet 2 przez ISP-B, pakiet 3 przez A. Gdyby rzeczywiście tak działał standardowy balans, jeden TCP flow mógłby mocno cierpieć na **packet reordering**, bo drogi mają różne opóźnienia.

W oficjalnej dokumentacji [Configuring Per-Packet Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-packet-load-balancing.html) wyjaśniono, że historyczna akcja **`load-balance per-packet`** w wielu typowych konfiguracjach realizuje **per-flow load sharing** przez hash. Dokumentacja [Per-Flow Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-flow-load-balancing.html) opisuje także `load-balance per-flow`, w odpowiednich wydaniach funkcjonalnie zbliżone/identyczne z legacy per-packet.

**Dlatego nie ucz się „per-packet = pakiet po pakiecie” bez sprawdzenia trybu, platformy i release.** Istnieją inne mechanizmy, np. random spray / packet spraying lub dynamic load balancing, ale to **osobne funkcje** z innym wpływem na kolejność pakietów.

### 7.5.4. Jak Junos wybiera drogę dla przepływu?

Wyobraź sobie tysiące klientów i tylko jeden wspólny ERP. Router wylicza hash z pól pakietu — w zależności od platformy może używać IP source/destination, protokołu i portów L4, a czasem również danych z enkapsulacji.

W wielu typowych implementacjach **ten sam flow dostaje tę samą drogę**, co redukuje packet reordering. Nowy flow może dostać inną drogę.

~~~mermaid
flowchart TD
  F["ECMP: dwa aktywne next-hopy"] --> HASH["Hash fields:\nsource/dest IP, czasem L4 ports"]
  HASH --> A["Flow 1 → WAN-A"]
  HASH --> B["Flow 2 → WAN-B"]
  HASH --> C["Flow 3 → WAN-A"]
  HASH --> D["Flow 4 → WAN-B"]
  ONE["Jeden duży TCP flow"] -.-> HASH
  ONE -. "nie musi zostać\nrozłożony na obydwa łącza" .-> A
~~~

**Gotcha #11:** jeden ogromny transfer TCP może zajmować tylko jeden link. Ocenianie ECMP po jednym ping albo jednym `iperf3` flow nie jest dobrym testem rozkładu hash. Potrzebujesz wielu niezależnych flow z różnymi adresami/portami.

### 7.5.5. Co domyślnie trafia z RIB do FIB?

Junos dokumentuje przypadki, w których pomimo kilku równokosztowych next-hopów dla active route **domyślnie wybiera jeden next-hop do forwarding table**, a dopiero policy `load-balance per-packet` / `per-flow` umożliwia instalację wielu do forwarding table. Ta zasada ma wyjątki platformowe i protokołowe.

Nie mów więc: „W RIB są dwie drogi, więc ASIC na pewno ECMP”. Najpierw sprawdź szczegóły active route i FIB.

Źródło: [Per-Flow Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-flow-load-balancing.html).

### 7.5.6. Forwarding-table export policy — to nie jest export do BGP

W Junos policy, która modyfikuje sposób instalacji route w forwarding table, przypina się w **`routing-options forwarding-table export`**. To bardzo podchwytliwa nazwa dla osoby, która słowo „export” kojarzy wyłącznie z reklamowaniem prefiksów do sąsiada BGP.

Są dwa różne eksporty:
- **BGP/OSPF export policy** — czy i jak reklamujemy route na zewnątrz;
- **forwarding-table export policy** — jak dana route jest reprezentowana na potrzeby FIB/PFE.

**Gotcha #12:** dodanie `then load-balance per-packet` do nieużywanej policy **niczego nie zmienia**, dopóki nie przypniesz jej do odpowiedniego forwarding-table export. To ten sam wzorzec „obiekt istnieje, ale nigdzie go nie zastosowano”, który poznaliśmy przy `apply-groups` i `rib-group`.

### 7.5.7. Konfiguracja: ograniczmy multipath do wybranego prefiksu

W rozszerzonym, odseparowanym LAB R1 ma **dwa rzeczywiście osiągalne next-hopy**:
- WAN-A `198.51.100.2`;
- WAN-B `198.51.100.6`.

Na obu Linux gateways istnieje testowy adres loopback `203.0.113.10/32`, aby można było zaobserwować, którą drogą rzeczywiście idzie pakiet. To **kontrolowany przykład anycast w labie**, nie wzorzec do uruchamiania publicznej usługi bez projektu HA.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], izolowany LAB, oba next-hopy rzeczywiście istnieją**

~~~junos
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.2
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.6
show | compare
commit check
~~~

Ta konfiguracja tworzy wielu kandydatów/next-hopów do jednej static route. Nie zakładaj jeszcze, że obie drogi są instalowane do FIB. Sprawdź read-only route details.

**Urządzenie: LAB-R1-vSRX — Operational Mode, przed forwarding policy**

~~~junos
show route 203.0.113.10/32 extensive
show route forwarding-table destination 203.0.113.10
show arp
~~~

Dopiero następnie w osobnym kroku dodajemy świadomą policy.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], konfiguracja tylko dla wybranego prefiksu**

~~~junos
set policy-options policy-statement CH07-ECMP term SERVICE from route-filter 203.0.113.10/32 exact
set policy-options policy-statement CH07-ECMP term SERVICE then load-balance per-packet
set policy-options policy-statement CH07-ECMP term SERVICE then accept
set policy-options policy-statement CH07-ECMP term REST then accept
set routing-options forwarding-table export CH07-ECMP
show | compare
commit check
~~~

**Uwaga produkcyjna:** `routing-options forwarding-table export` ma znaczenie globalne dla odpowiedniego kontekstu. Jeżeli już istnieje forwarding policy, **nie zastępuj jej** nową. Trzeba połączyć logikę z istniejącym policy chain i przetestować, czy inne trasy nadal trafiają do FIB. Dlatego w labie używamy nowego, izolowanego routera, a nie istniejącego MX core.

**Urządzenie: LAB-R1-vSRX — Operational Mode, po zatwierdzeniu w nowym LAB**

~~~junos
show configuration routing-options forwarding-table | display set
show configuration policy-options policy-statement CH07-ECMP | display set
show route 203.0.113.10/32 extensive
show route forwarding-table destination 203.0.113.10
~~~

Wspierane wersje i platformy mogą różnić się w reprezentacji multipath i algorytmie hasha; nie traktuj przykładowego outputu jako gwarancji ECMP bez testu na vSRX.

### 7.5.8. Gotcha: sama policy nie wytworzy dwóch equal-cost next-hopów

Jeśli RIB ma tylko jeden prawidłowy next-hop, forwarding policy nie stworzy drugiego. Podobnie jeśli drugi link jest admin-down albo route do niego jest unqualified. ECMP wymaga **rzeczywistej multipath eligibility**.

Pytanie instruktora: „Konfiguracja `then load-balance per-packet` jest poprawna, lecz w FIB widzę tylko jedną drogę”. Co sprawdzasz?  
**Odpowiedź:** źródło trasy, eligible next-hops, ich koszt/weight, resolution, interfejsy, matching policy, platform support i czy route naprawdę kwalifikowała się do multipath.

### 7.5.9. Jak zmienić fields używane w hashu?

Nie każdy PFE bierze pod uwagę te same wartości. Jeśli hash uwzględnia tylko L3, tysiące połączeń pomiędzy **jedną parą adresów** mogą wykorzystywać ten sam egress mimo różnych portów TCP.

Na niektórych rodzinach można ustawić hash-key z polami L4.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], WYŁĄCZNIE opcjonalna składnia do walidacji per platform**

~~~junos
set forwarding-options hash-key family inet layer-4
show | compare
commit check
~~~

Dokładne wspierane pola i warianty są **silnie platform-dependent**. Na MX/QFX/EX oraz vSRX możesz uzyskać różne efekty; zasięgnij [hash-key CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/hash-key-edit-forwarding-options.html). Nie kopiuj opcji na produkcję tylko dlatego, że polepszyła rozkład w syntetycznym teście.

**Gotcha #13:** lepsza równowaga *liczby flow* nie gwarantuje równości **przepływności w Mb/s**. Jeśli jedno połączenie przesyła 20 GB, a pozostałe dziesięć po 1 MB, nawet idealny hash może dać nierówny bandwidth.

### 7.5.10. Load balancing vs failover vs LAG

| Mechanizm | Co wygrywa / decyduje? | Podstawowe ograniczenie |
|---|---|---|
| ECMP | Wiele równorzędnych next-hopów w L3 | Jeden flow może wykorzystać jeden next-hop |
| Floating static | Preferowany primary, backup po utracie eligibility | Nie wykrywa awarii aplikacji za aktywnym peer |
| LACP/ae | Hash przez fizyczne members agregacji | Jedno logical interface, niekoniecznie kilka L3 next-hop |
| BGP multipath | Dodatkowe warunki BGP dla wielu ścieżek | Nie każde dwie ścieżki BGP kwalifikują się |
| FBF | Wybór innej tablicy na podstawie match | Nie jest load balancing, jeżeli trafia zawsze do jednej drogi |
| Packet spray | Potencjalnie pakiet-po-pakiecie w dedykowanych funkcjach | Możliwy reorder, wymaga właściwego modelu/dataplane |

### 7.5.11. Mini-lab: nie testuj ECMP jednym ping

**Urządzenie: LAB-UP-A-LINUX — Linux shell, krótki capture na WAN-A**

~~~bash
sudo tcpdump -ni eth0 host 203.0.113.10 -c 25
~~~

**Urządzenie: LAB-UP-B-LINUX — Linux shell, krótki capture na WAN-B**

~~~bash
sudo tcpdump -ni eth0 host 203.0.113.10 -c 25
~~~

**Urządzenie: LAB-PC-A-LINUX — Linux shell, ICMP i kilka prób z różnych źródeł w izolowanym labie**

~~~bash
ping -c 5 203.0.113.10
~~~

Jeden ICMP stream może trafić tylko na A — to **nie** oznacza, że ECMP jest zepsuty. Do sensownego testu trzeba generować wiele niezależnych flow, np. kilka adresów source lub TCP z różnymi source-port, i potwierdzić **rzeczywistą dystrybucję** po obu gatewayach. Do automatyzacji można użyć `iperf3`/`curl` z hostów na testowej sieci, ale najpierw uruchom usługę docelową, a nie zakładaj, że serwer istnieje.

**Na pamięć:** `load-balance per-packet` może być per-flow; `forwarding-table export` ≠ BGP export.  
**Do zrozumienia:** RIB multiple next-hops → FIB eligibility → hash fields → wiele realnych przepływów.  
**Do przećwiczenia:** zmierz liczbę przepływów na obu uplinkach, nie tylko aggregate throughput.

---

## 7.6. Filter-Based Forwarding (FBF) — podstawy i zastosowania [S]

### 7.6.1. Dlaczego sam destination IP nie wystarcza w każdym projekcie?

W standardowym IP routing R1 wybiera drogę na podstawie **destination** i właściwej RIB. Ale w campusie zdarzają się sytuacje, gdy chcesz inaczej potraktować:
- pakiety z konkretnej podsieci gościnnej;
- serwery backupowe generujące duży ruch;
- ruch IoT do systemów przemysłowych;
- tylko HTTPS do partnera;
- wybrane klasy usług o konkretnym DSCP;
- ruch kierowany do dodatkowego firewalla/inspection service.

Przy zwykłym LPM wszystkie te pakiety do tego samego destination mogą otrzymać identyczną ścieżkę. **FBF** daje możliwość klasyfikacji na wejściu na podstawie **nagłówków pakietu**, a następnie wybrania innej **routing instance** i jej forwarding decision.

**Most z Cisco:** koncepcyjnie najbliżej jest **PBR route-map przypięta do interfejsu wejściowego**. Ale Junos FBF zwykle nie robi po prostu `set ip next-hop X`; jego typowym działaniem jest przekierowanie lookup do **innej tabeli**, która sama ma swoje trasy i next-hop resolution.

### 7.6.2. FBF składa się z trzech elementów, nie jednego

1. **Routing instance** typu `forwarding` z wybraną trasą, np. default via operator B.
2. **Stateless firewall filter** z match criteria i akcją `then routing-instance FBF-ALT`.
3. **Przypięcie filtra na ingress** oraz, często, **RIB group** dostarczająca connected routes do rozwiązania next-hopa w FBF table.

~~~mermaid
flowchart TD
  PKT["Pakiet: source 192.0.2.10,\ndest 203.0.113.10"] --> IN["Ingress ge-0/0/1.0"]
  IN --> FILT["Input firewall filter CH07-PBR"]
  FILT --> MATCH{"Source pasuje do /32?"}
  MATCH -->|TAK| FBF["FBF-ALT.inet.0\n0/0 via WAN-B"]
  MATCH -->|NIE| MAIN["inet.0\n0/0 via WAN-A"]
  RIB["RIB group COPY-DIRECT"] -. "connected uplinks" .-> FBF
  FBF --> B["WAN-B: 198.51.100.6"]
  MAIN --> A["WAN-A: 198.51.100.2"]
~~~

**Gotcha #14:** bez działania w pkt 3 filter może istnieć w config, ale nie być stosowany do interfejsu. Bez RIB group route FBF może mieć unresolved next-hop. Bez security policy na SRX nawet poprawnie przekierowany pakiet może nie przejść.

### 7.6.3. Konfigurujemy klasyfikację jednego klienta

Przykład ma sens **tylko** w nowym, izolowanym labie:
- klient specjalny = `192.0.2.10/32`;
- pozostali użytkownicy = reszta sieci OFFICE;
- forwarding instance `FBF-ALT` już istnieje i ma default via `198.51.100.6`;
- RIB group już udostępnia connected do WAN-B.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], stateless FBF tylko dla 192.0.2.10**

~~~junos
set firewall family inet filter CH07-PBR term SPECIAL from source-address 192.0.2.10/32
set firewall family inet filter CH07-PBR term SPECIAL then routing-instance FBF-ALT
set firewall family inet filter CH07-PBR term NORMAL then accept
set interfaces ge-0/0/1 unit 0 family inet filter input CH07-PBR
show | compare
commit check
~~~

**Co powinno się stać?**
- pakiet **przychodzący** przez `ge-0/0/1.0` ze źródła `192.0.2.10` pasuje do term SPECIAL;
- jego lookup zostaje skierowany do `FBF-ALT.inet.0`;
- tabela FBF ma default przez WAN-B;
- inne pakiety trafiają w term NORMAL, który **pozwala kontynuować normalną ścieżkę**, czyli zwykły `inet.0`.

**Nie oznacza to:** że firewall filter jest stanową security policy, że BGP automatycznie ogłosi cokolwiek, że NAT samo się zmieni albo że klient będzie miał poprawny return path.

### 7.6.4. Dlaczego term NORMAL z `then accept` jest konieczny?

W standardowym stateless firewall filter, jeśli pakiet nie pasuje do żadnego term, obowiązuje domyślne zachowanie filtra (w typowym scenariuszu **implicit discard**). To byłaby katastrofa na produkcyjnym uplinku, gdybyś chciał przekierować jeden host, a przypadkowo odciął wszystkie inne.

Dlatego jawnie dodajemy:

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], term zabezpieczający pozostały ruch**

~~~junos
set firewall family inet filter CH07-PBR term NORMAL then accept
~~~

**Gotcha #15:** w FBF słowo **`accept`** oznacza przyjęcie pakietu przez **stateless filter** i kontynuowanie przetwarzania — nie jest automatycznym permit w SRX **stateful security policy**. To dwie odrębne warstwy.

### 7.6.5. Nie wszystkie akcje można łączyć z `then routing-instance`

Masz pokusę, by w jednym term SPECIAL dodać:
- `count`;
- `log`;
- `routing-instance`.

W zależności od platformy i wydania **łączenie `routing-instance` z innymi akcjami może być ograniczone**. Na niektórych rodzinach (np. udokumentowany wariant ACX) lista niezgodnych akcji obejmuje `count`, `log`, `syslog`, `policer` i `port-mirror`. Nie mów więc, że każdy filter FBF może automatycznie liczyć pakiety na tym samym term.

Sprawdź [Filter-Based Forwarding for Routing Instances](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/filter-based-forwarding-acx-series.html) i Feature Explorer dla **konkretnego modelu**. Jeśli potrzebujesz licznika, wybierz wspierany sposób, np. **oddzielny filtr/term lub telemetry** zależnie od platformy, zamiast na siłę dodawać `then count` do term z akcją routing-instance.

### 7.6.6. FBF jest stateless, a SRX firewall jest stateful

To jest operacyjnie najważniejsza różnica.

Wyobraź sobie ruch:
- PC-A → ERP przechodzi przez FBF-ALT/WAN-B;
- odpowiedź wraca przez WAN-A, bo druga strona ma inny routing;
- SRX ma stateful session utworzoną z innym ingress/egress zone context.

**Gotcha #16:** prawidłowe dopasowanie FBF nie gwarantuje, że **asymetryczny return path** zostanie zaakceptowany przez SRX. Trzeba rozumieć NAT, security zones, flow sessions i projekt symetrii, szczególnie przy stateful firewall i HA.

**Urządzenie: LAB-R1-vSRX — Operational Mode, diagnostyka SRX flow po testowym pakiecie**

~~~junos
show security zones
show security policies
show security flow session
show route table FBF-ALT.inet.0
show route table inet.0
~~~

Jeżeli na vSRX pojawia się błąd parsera przy FBF albo ruch nie jest zgodny z oczekiwaniami, **nie oszukuj się**, że konfiguracja z MX musi działać tak samo. Na SRX mogą istnieć również dedykowane **Advanced Policy-Based Routing** mechanizmy, z inną kolejnością przetwarzania. Sprawdź [Advanced PBR for SRX](https://www.juniper.net/documentation/us/en/software/junos/application-identification/topics/topic-map/security-application-advanced-policy-based-routing.html) oraz model/wersję.

### 7.6.7. FBF vs routing policy vs security policy — trzy różne pytania

| Mechanizm | Na czym pracuje? | Jakie pytanie rozstrzyga? |
|---|---|---|
| Routing policy | **Na trasach** w RIB/protokołach | Czy i jak route zostanie zaakceptowana/ogłoszona |
| Firewall filter FBF | **Na pakietach** przy wejściu do dataplane | Która routing table ma obsłużyć forwarding |
| SRX security policy | **Na sesjach/ruchu** z uwzględnieniem stref i usług | Czy nowy flow ma być dopuszczony |
| Forwarding-table export policy | **Na instalacji route do FIB** | Czy i jak używać multipath / next-hop w FIB |

**Gotcha #17:** w Junos wszędzie może wystąpić słowo `policy`. **Nazwa nie mówi, czy dotykasz pakietów, tras, stateful sessions czy FIB.** Dopiero hierarchia CLI ujawnia warstwę, na której działasz.

### 7.6.8. Jak udowodnić, że FBF naprawdę przekierował pakiet?

Samo `show configuration firewall` dowodzi wyłącznie intencji. Zbieramy cztery typy dowodu:

1. **Configuration:** czy filter istnieje i jest przypięty na właściwym ingress?
2. **Routing:** czy FBF instance ma default i resolved next-hop do WAN-B?
3. **Forwarding/flow:** czy platforma pokazuje dane zgodne z routing-instance selection, a SRX flow/policy nie odrzuca ruchu?
4. **External observation:** czy na gateway WAN-B rzeczywiście pojawia się pakiet testowego klienta? Czy pakiet z innego klienta idzie WAN-A?

**Urządzenie: LAB-R1-vSRX — Operational Mode, read-only check**

~~~junos
show configuration firewall family inet filter CH07-PBR | display set
show configuration interfaces ge-0/0/1 | display set
show configuration routing-instances FBF-ALT | display set
show route table FBF-ALT.inet.0
show route table inet.0
show firewall
show security flow session
~~~

**Urządzenie: LAB-UP-A-LINUX — Linux shell, niezależny punkt obserwacyjny WAN-A**

~~~bash
sudo tcpdump -ni eth0 host 203.0.113.10 -c 20
~~~

**Urządzenie: LAB-UP-B-LINUX — Linux shell, niezależny punkt obserwacyjny WAN-B**

~~~bash
sudo tcpdump -ni eth0 host 203.0.113.10 -c 20
~~~

Gdy rzeczywiście widzisz na WAN-B pakiety PC-A ze source `192.0.2.10`, a drugi klient trafia na WAN-A, masz znacznie mocniejszy dowód niż tylko poprawny `commit check`. **Uwaga:** w zależności od NAT/flow processing IP source może zostać przepisany; dlatego w naszym izolowanym labie NAT jest jawnie wyłączony/brak konfiguracji NAT, a test z realną sesją musi to potwierdzić.

### 7.6.9. FBF a IPv6 — nie zakładaj parytetu 1:1

Dla IPv6 istnieje również `firewall family inet6` i możliwość skierowania ruchu do odpowiedniego kontekstu routingu, ale **nie wszystkie match criteria L4 oraz platformy obsługują to identycznie jak IPv4**. Przykładowo niektóre platformy ograniczają L4 source-port/destination-port przy IPv6 FBF.

Nie kopiujemy IPv4 FBF wprost do `inet6` jako „na pewno działającego” labu. W przyszłym rozszerzeniu trzeba zweryfikować Feature Explorer, rodzaj PFE i dokumentację Junos release. Zgodnie z kursem JNCIS najpierw musisz **przewidzieć kierunek IPv4**, a dopiero potem dual-stack FBF.

### 7.6.10. FBF jest ingerencją w architekturę routingu, nie kosmetyczną optymalizacją

Każdy wyjątek PBR/FBF jest lokalną regułą, którą trzeba uwzględnić przy:
- failover i utracie WAN-B;
- rollback change;
- routingu powrotnym;
- planie firewall/NAT;
- testach NMS i archiwizacji;
- dokumentacji HLD/LLD;
- przeglądzie security i threat model.

**Most z Cisco:** PBR jest skutecznym narzędziem, ale duża liczba route-map/next-hop exceptions potrafi uczynić sieć trudną w utrzymaniu. Junos FBF daje strukturę odseparowanych RIB, lecz **nie usuwa kosztu operacyjnego wyjątku**. Jeśli tego samego celu można bezpiecznie dokonać dobrym dynamic routing, zwykle warto porównać obie opcje.

**Na pamięć:** `firewall family inet filter ... then routing-instance NAME`, `routing-instances NAME instance-type forwarding`, `interface-routes rib-group`.  
**Do zrozumienia:** FBF zmienia **tablicę**; nie uruchamia samodzielnie NAT, security policy ani default route.  
**Do przećwiczenia:** dwa źródła do tego samego destination → dwa różne egress, potwierdzone capture na dwóch gatewayach.



---

## 7.7. Konfiguracja, monitoring i troubleshooting: aggregate, generated, load balancing i FBF [S]

### 7.7.1. „Po zmianie routingu jedno VLAN działa, a drugie nie” — jaki jest dobry pierwszy ruch?

Wyobraź sobie incydent w kampusie. W nocy ktoś wdrożył FBF dla podsieci gościnnej. Rano Service Desk zgłasza: „VPN dla kontrahentów działa, ale ERP w biurze nie odpowiada, choć globalna `inet.0` ma route”. Operator uruchamia `show route` i stwierdza: „W tablicy jest default, więc winne jest FW”.

Właśnie tu wchodzi wiedza z tego rozdziału. Możliwe przyczyny:
- filtr FBF został przypięty **nie na tym interfejsie**;
- pakiet trafił do **innej routing instance** niż globalna;
- w tej instancji istnieje default, ale nie ma **connected next-hop**;
- stateless filter nie ma końcowego `then accept`, więc odrzuca niezaznaczony ruch;
- ruch po FBF wybiera egress WAN-B, ale na SRX nie ma policy z właściwego to-zone;
- return path przechodzi innym firewall;
- aggregate /16 z `reject` zatrzymuje nieistniejący podprefiks;
- `show route forwarding-table` nie pokazuje wszystkich ścieżek, które założyłeś na podstawie RIB.

**Nie szukamy jednej cudownej komendy. Szukamy miejsca, w którym zmieniono decyzję pakietu.**

### 7.7.2. Pięć warstw śledzenia jednego pakietu

~~~mermaid
flowchart TD
  PC["Klient zgłasza błąd"] --> ING["1. Ingress / source / VLAN\nshow interfaces, LLDP"]
  ING --> CLASS["2. Stateless FBF filter\ni właściwa routing-instance"]
  CLASS --> RIB["3. RIB / active route / aggregate"]
  RIB --> NH["4. Forwarding-table i\nconnected next-hop/ARP"]
  NH --> SEC["5. SRX policy / stateful flow /\nreturn path"]
  SEC --> CLIENT["Dowód z hosta i testu usługi"]
~~~

**Ważne:** kolejność to **model wyjaśnienia**, nie literalny pipeline dla każdego PFE. SRX flow processing, filter stage i NAT mogą mieć szczególne uwarunkowania. Właśnie dlatego w produkcji potrzebujesz dokumentacji wydania i testu, zamiast zakładać, że wszystkie modele zachowują się identycznie.

### 7.7.3. Zbierz najpierw aktywną konfigurację i kontekst ingress

**Urządzenie: LAB-R1-vSRX — Operational Mode, read-only first response**

~~~junos
show version
show interfaces terse
show interfaces descriptions
show configuration interfaces ge-0/0/1 | display set
show configuration firewall family inet filter CH07-PBR | display set
show configuration routing-instances FBF-ALT | display set
show system commit
~~~

Pytasz: który interface otrzymał pakiet? Czy input filter działa **na tym logical unit i dla właściwej family**? Czy match source jest prawidłowy? Czy zmianę naprawdę zatwierdzono?

**Gotcha #18:** administrator może widzieć poprawną definicję firewall filter, ale jeśli nie jest podpięta do odpowiedniej family input, **nie klasyfikuje ruchu klienta**. Tak samo routing policy wpisana w `policy-options` nie działa, jeśli nie ma odniesienia w odpowiednim protokole/forwarding table.

### 7.7.4. Teraz sprawdź dwie RIB i next-hop

**Urządzenie: LAB-R1-vSRX — Operational Mode, diagnostyka tablic**

~~~junos
show route table inet.0
show route table FBF-ALT.inet.0
show route table FBF-ALT.inet.0 0.0.0.0/0
show route table FBF-ALT.inet.0 198.51.100.4/30
show route hidden
show arp
~~~

Jeśli w `FBF-ALT.inet.0` brakuje connected do WAN-B, przejdź do RIB group i jej przypięcia:

**Urządzenie: LAB-R1-vSRX — Operational Mode, read-only RIB group**

~~~junos
show configuration routing-options rib-groups | display set
show configuration routing-options interface-routes | display set
show configuration policy-options policy-statement RG-ONLY-WAN-B | display set
~~~

Jeżeli opcjonalna import-policy nie jest skonfigurowana, ostatnie polecenie może nic nie zwrócić — to **normalny wynik**. Nie diagnozuj przez przypadek „missing policy” jako awarii, jeśli projekt dopuszcza import connected bez filtra.

### 7.7.5. Rozdziel RIB i FIB oraz routing od security

**Urządzenie: LAB-R1-vSRX — Operational Mode, FIB/flow**

~~~junos
show route forwarding-table destination 203.0.113.10
show route 203.0.113.10
show security policies
show security flow session
~~~

**Szczególny gotcha:** `show route forwarding-table destination 203.0.113.10` bez wskazania kontekstu może dać **widok inny niż faktyczne FBF lookup**, bo musisz zbadać właściwą forwarding table/instance i możliwości CLI platformy. Nie dowodź działania FBF samym odczytem global FIB.

Dowód ostateczny to najlepiej **rzeczywiste pakiety** na obu upstream gatewayach. W labie użyj krótkiego `tcpdump`; w produkcji przy fizycznym EX/QFX/MX dobierz bezpieczny analyzer/telemetrię zgodnie z prawami klienta i obciążeniem.

### 7.7.6. Trouble ticket A — aggregate reklamu przyciąga ruch, którego nie ma

**Objaw:** partner BGP otrzymał summary `10.60.0.0/16`. Może dotrzeć do `10.60.10.25`, ale `10.60.99.25` nie odpowiada.

**Błędny wniosek:** skoro BGP ogłosił /16, wszystkie 256 możliwych podsieci /24 muszą być fizycznie dostępne.

**Wyjaśnienie:** aggregate wymaga co najmniej jednego contributor. Dla adresów bez bardziej szczegółowej route domyślny `reject` na aggregate jest legalną i często pożądaną obroną przed routing loop.

**Urządzenie: LAB-R1-vSRX — Operational Mode**

~~~junos
show route 10.60.0.0/16 extensive
show route 10.60.10.0/24
show route 10.60.99.0/24
show route forwarding-table destination 10.60.99.25
~~~

**Rozwiązanie:** albo uzupełnij rzeczywistą reachability do brakującego subnet, albo **nie ogłaszaj tak szerokiego summary**, jeśli HLD wymaga komunikacji do wszystkich części /16. Nie usuwaj na ślepo reject, bo możesz stworzyć pętlę przez default.

### 7.7.7. Trouble ticket B — po dodaniu FBF cała sieć OFFICE straciła dostęp

**Objaw:** filtrowałeś source `192.0.2.10/32`, a pozostałe komputery także straciły Internet.

**Hipoteza:** brak jawnego końcowego `term NORMAL then accept` i pakiety, których nie dopasowano do SPECIAL, zostały odrzucone przez domyślną akcję filtra.

**Urządzenie: LAB-R1-vSRX — Operational Mode, odczyt**

~~~junos
show configuration firewall family inet filter CH07-PBR | display set
show configuration interfaces ge-0/0/1 | display set
show firewall
~~~

**Rozwiązanie:** przywróć ostatnią dobrą konfigurację przez `commit confirmed` timer/rollback albo dodaj bezpieczny końcowy term — po dokładnym porównaniu diff. **Nie zakładaj**, że `then routing-instance` ma też ukryte `accept rest`.

### 7.7.8. Trouble ticket C — ECMP jest w RIB, ale 98% traffic idzie WAN-A

**Objaw:** dwie ścieżki mają ten sam koszt, obie interfaces są up, lecz z telemetrii wynika silna nierównowaga.

**Hipotezy:** jeden elephant flow, hash tylko po L3, niesymetryczna liczba niezależnych flow, nierówny capacity, FBF przenoszący niektóre flows poza global route, albo w FIB jest tylko jeden next-hop.

**Urządzenie: LAB-R1-vSRX — Operational Mode**

~~~junos
show route 203.0.113.10/32 extensive
show route forwarding-table destination 203.0.113.10
show configuration routing-options forwarding-table | display set
show configuration policy-options policy-statement CH07-ECMP | display set
~~~

**Rozwiązanie:** udowodnij multipath eligibility i faktyczną dystrybucję **wielu flows**, a nie oceniaj hash na podstawie jednego `ping`. Nie włączaj packet spray na produkcji bez analizy reorder.

### 7.7.9. Trouble ticket D — martian trasa nie pojawia się po BGP import

**Objaw:** peer reklamuje `127.22.0.0/16`, router nie instaluje go jako zwykłej active route.

**Pytanie:** czy samo użycie innej BGP preference rozwiąże problem? Nie. Jeśli prefix jest objęty martian rule, nie zachowuje się jak zwykły kandydat routingu.

**Urządzenie: LAB-R1-vSRX — Operational Mode**

~~~junos
show route martians
show route hidden
show route protocol bgp
~~~

**Ważne:** `show route protocol bgp` będzie puste, jeśli nie masz BGP lab. Nie twórz pozornego incydentu z pustego output. Na rzeczywistym peer wykorzystaj odpowiednie receive-protocol i policy diagnostics.

### 7.7.10. Trouble ticket E — generated route została active, ale pakiety idą do nieoczekiwanego WAN

**Objaw:** generated `10.70.0.0/16` działa, lecz jej next-hop wskazuje ISP-A, a operator myślał, że wybierze ISP-B.

**Przyczyna możliwa:** generated route przejmuje next-hop od **primary contributing route**. Wybór primary contributors zależy od preferencji i reguł, a nie od Twojej intencji „drugi podprefix powinien być primary”.

**Urządzenie: LAB-R1-vSRX — Operational Mode**

~~~junos
show route 10.70.0.0/16 extensive
show route table inet.0 10.70.0.0/16 longer
show configuration routing-options generate | display set
show route forwarding-table destination 10.70.99.25
~~~

**Rozwiązanie:** jeśli celem jest bezpieczne podsumowanie, rozważ aggregate/reject; jeśli celem jest conditional forwarding, dobierz contributors/policy i świadomie zweryfikuj next-hop. Nie zamieniaj `aggregate` na `generate` tylko dlatego, że nazwy podobnie brzmią.

### 7.7.11. Change workflow dla FBF/RIB group — jak zrobić to bez P1?

Przygotowując change na EX/MX/SRX, najpierw zrób read-only snapshot:
- istniejących interface assignments i routing instances;
- aktywnych routing policies, forwarding-table export i firewall filters;
- istniejących connected / static / dynamic routes;
- security zones, sessions i NAT (na SRX);
- bieżącego traffic i obciążenia uplinków;
- testu aplikacji z dwóch klientów o różnych źródłach.

Następnie:
1. utwórz docelową forwarding table i sprawdź resolution;
2. jeśli niezbędne, dodaj RIB group z **jak najmniejszym zakresem**;
3. dopiero potem przypnij FBF filter do wejścia;
4. wykonaj `show | compare` i `commit check` **przed** `commit confirmed`;
5. po aktywacji nie uruchamiaj ponownie `commit check` w czasie pending timer;
6. wykonaj odrębne testy ruchu zwykłego i przekierowanego;
7. potwierdź commit dopiero, gdy obie grupy klientów i monitoring są zgodne z baseline.

**Gotcha #19:** nawet jeśli PC-A korzystający z FBF działa, **PC-B korzystający z normalnego routingu może być uszkodzony**. Test jednego hosta nie wystarczy do zamknięcia change.

### 7.7.12. Case study: Security pyta, dlaczego guest Wi-Fi omija centralny firewall

W projekcie nowej lokalizacji masz WLAN corporate i guest. Po dołożeniu FBF tylko część guest traffic idzie przez firewall. Analizując to, nie możesz mówić „Wi-Fi działa” na podstawie ping do AP lub WLC.

Sprawdź **rzeczywisty source IP / VLAN / IRB**, bo firewall filter może dopasowywać się do adresacji NATowanej wcześniej, a nie do tego, co myślałeś. Zobacz, **na którym fizycznym/logical ingress filtr został przypięty**. Sprawdź forwarding instance oraz **czy nie istnieje bardziej szczegółowy route w FBF RIB**, która zmienia zamierzony WAN-B.

**Most z Cisco:** tak samo PBR route-map na SVI może działać inaczej niż statyczna default, a polityka i NAT na bramie mogą przetwarzać ruch w innym miejscu niż wydawało się w HLD. Tu nie chodzi o „komendy Junos są dziwne”, tylko o świadome zaprojektowanie punktu kontroli ruchu.

### 7.7.13. Worknotes i RCA — napisz, dlaczego, nie tylko że

Po teście do ServiceNow nie wpisuj tylko „configured PBR, traffic OK”. W dobrym podsumowaniu znajdują się:
- dokładny affected source subnet i destination;
- rola interfejsu i source of truth topologii;
- active RIB/FIB w normalnym oraz FBF context;
- zastosowany match i expected/actual action;
- read-only counters/flow i dowód ze zdalnego endpoint;
- zdarzenie fault i test recovery;
- wynik nieobjętych polityką hostów;
- czas, software release, numer change i brak pozostawionego pending commit.

**Zasada Tier 3:** nie przejmujemy kontroli nad ruchem wyłącznie deklaracją w config. **Udowadniamy, gdzie pakiety rzeczywiście wyszły.**



---

## 7.8. [LAB] Aggregate route + FBF + RIB group — weryfikacja decyzji routingu

### 7.8.1. Dlaczego potrzebujemy dwóch upstreamów, a nie tylko dwóch adresów na jednym porcie?

Jeżeli chcemy pokazać, że ruch **rzeczywiście wybrał WAN-B**, potrzebujemy fizycznie/logicznym odrębnego next-hop i punktu obserwacji. `show route` nie wystarczy. Dlatego budujemy:
- **LAB-R1-vSRX** — jeden router/firewall Junos z trzema portami danych;
- **LAB-PC-A** i **LAB-PC-B** — dwa niezależne lekkie Linux hosty w sieci OFFICE;
- **LAB-UP-A** i **LAB-UP-B** — dwa lekkie Linux gateways symulujące niezależnych operatorów;
- wspólny segment OFFICE oraz **dwa osobne segmenty transit**;
- niezależną konsolę EVE do wszystkich urządzeń.

W ten sposób nie potrzebujemy trzech ciężkich vSRX do pokazania 7.1–7.8; za routing między uproszczonymi sieciami testowymi odpowiada R1, a Linux gateways tworzą prawdziwe, obserwowalne next-hopy.

**Status: PLANNED / NOT RUNTIME VALIDATED.** Nie wygenerowałem ani nie uruchomiłem w tej rozmowie pliku `.unl`, nie wiem jeszcze, jak rzeczywista wersja vSRX mapuje porty QEMU, i nie twierdzę, że FBF jest wspierane w Twoim konkretnym obrazie. Przed wykonaniem trzeba sprawdzić Feature Explorer, CLI `?`, release i zasoby EVE. Jeśli vSRX nie obsługuje wymaganego FBF, użyj odpowiedniego vMX/EX/QFX lab w osobnej, wspieranej platformie, **nie udawaj PASS**.

~~~mermaid
flowchart LR
  subgraph OFFICE["OFFICE\n192.0.2.0/24"]
    A["LAB-PC-A\n192.0.2.10"]
    B["LAB-PC-B\n192.0.2.11"]
  end
  R1["LAB-R1-vSRX\nge-0/0/1 CLIENT\nge-0/0/0 WAN-A\nge-0/0/2 WAN-B"]
  subgraph WANA["WAN-A 198.51.100.0/30"]
    UA["LAB-UP-A\n198.51.100.2"]
  end
  subgraph WANB["WAN-B 198.51.100.4/30"]
    UB["LAB-UP-B\n198.51.100.6"]
  end
  A --- R1
  B --- R1
  R1 ---|"198.51.100.1/30"| UA
  R1 ---|"198.51.100.5/30"| UB
  UA -. "Loopback testowy\n203.0.113.10/32" .-> APP["Wspólny adres testowy\nanycast w odizolowanym labie"]
  UB -. "Loopback testowy\n203.0.113.10/32" .-> APP
~~~

Dwa upstream Linux mają **ten sam adres `203.0.113.10/32` na loopback** wyłącznie po to, żeby dało się wysyłać ruch do **identycznego destination** i udowodnić, że R1 zmienił *drogę*, a nie cel. Jest to uproszczony test anycast — nie modeluj na tej podstawie wieloregionowej usługi DNS bez niezależnego projektu.

### 7.8.2. Adresacja i intencja routingu

| Node | Port / interfejs (do potwierdzenia) | IPv4 | Funkcja |
|---|---|---|---|
| LAB-PC-A | eth0 | 192.0.2.10/24 | Klient do przekierowania FBF |
| LAB-PC-B | eth0 | 192.0.2.11/24 | Klient zwykłego routingu |
| LAB-R1-vSRX | ge-0/0/1.0 | 192.0.2.1/24 | OFFICE ingress |
| LAB-R1-vSRX | ge-0/0/0.0 | 198.51.100.1/30 | WAN-A |
| LAB-UP-A | eth0 | 198.51.100.2/30 | Pierwszy upstream |
| LAB-R1-vSRX | ge-0/0/2.0 | 198.51.100.5/30 | WAN-B |
| LAB-UP-B | eth0 | 198.51.100.6/30 | Drugi upstream |
| LAB-UP-A / LAB-UP-B | lo | 203.0.113.10/32 | Ten sam cel testowy, dwa egress |
| LAB-UP-A | lo | 10.60.10.10/32, 10.70.10.10/32 | Contributors A |
| LAB-UP-B | lo | 10.60.20.10/32 | Contributor B |

Wykorzystujemy adresy **RFC 5737 TEST-NET** oraz prywatne prefixy laboratoryjne. Nie łączymy ich z Internetem/klientem i nie dopuszczamy automatycznego NAT z fabrycznej konfiguracji SRX.

**Routing plan R1 (global `inet.0`):**
- default `0.0.0.0/0 → 198.51.100.2` (WAN-A);
- static `10.60.10.0/24 → 198.51.100.2`;
- static `10.60.20.0/24 → 198.51.100.6`;
- aggregate `10.60.0.0/16` z dwoma contributors;
- static `10.70.10.0/24 → 198.51.100.2`;
- generated `10.70.0.0/16` z primary contributor WAN-A.

**FBF context `FBF-ALT.inet.0`:** default `0.0.0.0/0 → 198.51.100.6` (WAN-B), z connected interface routes udostępnionymi przez RIB group.

**FBF filter:** PC-A source `192.0.2.10/32` → FBF-ALT; PC-B → zwykła globalna tablica.

### 7.8.3. Preflight, który chroni inne laboratoria i prawdziwą sieć

Najpierw przejrzyj:
- katalogi EVE i istniejące laby CH02–CH06 oraz Cisco 300-410;
- wersję EVE i dostępne legalne obrazy vSRX/Linux;
- RAM/CPU, vNIC mapping, nazwę interfejsów;
- działanie niezależnej konsoli;
- istniejące factory NAT/security/AAA na vSRX;
- czy R1 ma wystarczającą liczbę portów danych.

**Nie nadpisuj żadnego istniejącego `.unl`, QEMU image, bridge, lab UUID, konfiguracji ani globalnego EVE setup.**

**Urządzenie: LAB-R1-vSRX — Operational Mode przez console EVE, read-only discovery**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration interfaces | display set
show configuration routing-options | display set
show configuration security | display set
show system commit
~~~

**Urządzenie: LAB-UP-A-LINUX — Linux shell, read-only discovery**

~~~bash
ip -br link
ip -br address
ip -4 route
~~~

**Urządzenie: LAB-UP-B-LINUX — Linux shell, read-only discovery**

~~~bash
ip -br link
ip -br address
ip -4 route
~~~

Uwaga: port może nazywać się `ens3`, a nie `eth0`. Podmień nazwę przed wykonaniem dowolnej zmiany. Nie uruchamiaj `ip address flush` na produkcyjnym bastionie.

### 7.8.4. Etap A — włączamy trzy L3 segmenty na R1

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], tylko świeży izolowany LAB**

~~~junos
set system host-name LAB-R1
set interfaces ge-0/0/1 description "CH07-OFFICE"
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/24
set interfaces ge-0/0/0 description "CH07-WAN-A"
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/30
set interfaces ge-0/0/2 description "CH07-WAN-B"
set interfaces ge-0/0/2 unit 0 family inet address 198.51.100.5/30
show | compare
commit check
~~~

To jest **konfiguracja warstwy L3**, nie complete SRX commissioning. Hasła i lokalne konta z rozdziału 4 już powinny działać. Mapowanie portów musi zgadzać się z realnym EVE-NG.

### 7.8.5. Etap B — Linux clients i upstream gateways

Przed każdą konfiguracją potwierdź nazwę NIC. Poniżej `IFACE=eth0` to **placeholder**, nie obietnica, że tak nazywa się karta.

**Urządzenie: LAB-PC-A-LINUX — Linux shell, tylko nowa testowa maszyna**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 192.0.2.10/24 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
ip -4 route
~~~

**Urządzenie: LAB-PC-B-LINUX — Linux shell, tylko nowa testowa maszyna**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 192.0.2.11/24 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
ip -4 route
~~~

**Urządzenie: LAB-UP-A-LINUX — Linux shell, tylko nowa maszyna reprezentująca WAN-A**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 198.51.100.2/30 dev "$IFACE"
sudo ip -4 addr add 203.0.113.10/32 dev lo
sudo ip -4 addr add 10.60.10.10/32 dev lo
sudo ip -4 addr add 10.70.10.10/32 dev lo
sudo ip -4 route replace 192.0.2.0/24 via 198.51.100.1 dev "$IFACE"
ip -4 route
~~~

**Urządzenie: LAB-UP-B-LINUX — Linux shell, tylko nowa maszyna reprezentująca WAN-B**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 198.51.100.6/30 dev "$IFACE"
sudo ip -4 addr add 203.0.113.10/32 dev lo
sudo ip -4 addr add 10.60.20.10/32 dev lo
sudo ip -4 route replace 192.0.2.0/24 via 198.51.100.5 dev "$IFACE"
ip -4 route
~~~

**Gotcha Linux:** host może mieć uruchomiony NetworkManager/systemd-networkd lub firewall. Komendy `ip` są z reguły nietrwałe po restarcie; nie stanowią gotowej konfiguracji dla produkcyjnego systemu. Dwa hosty z tym samym loopback IP są izolowane w **dwóch odrębnych Ethernet transit**, więc nie konkurują ARP-em na tym samym segmencie.

### 7.8.6. Etap C — SRX security zones i kontrolowane ICMP

Bez policy vSRX może nie przepuścić ruchu transit nawet wtedy, gdy RIB/FIB jest doskonały. W labie dopuszczamy jedynie ICMP z OFFICE do dwóch WAN zones. Ruch powrotny dla zarejestrowanej sesji nie wymaga osobnej polityki dopuszczającej nowe połączenie w przeciwną stronę.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], tylko izolowany LAB**

~~~junos
set security zones security-zone CLIENT interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone WAN-A interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone WAN-B interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone CLIENT to-zone WAN-A policy CH07-ICMP-A match source-address any
set security policies from-zone CLIENT to-zone WAN-A policy CH07-ICMP-A match destination-address any
set security policies from-zone CLIENT to-zone WAN-A policy CH07-ICMP-A match application junos-ping
set security policies from-zone CLIENT to-zone WAN-A policy CH07-ICMP-A then permit
set security policies from-zone CLIENT to-zone WAN-B policy CH07-ICMP-B match source-address any
set security policies from-zone CLIENT to-zone WAN-B policy CH07-ICMP-B match destination-address any
set security policies from-zone CLIENT to-zone WAN-B policy CH07-ICMP-B match application junos-ping
set security policies from-zone CLIENT to-zone WAN-B policy CH07-ICMP-B then permit
show | compare
commit check
~~~

**Uwagi:** na danym vSRX sprawdź istnienie predefiniowanej aplikacji `junos-ping` oraz działanie stref. **Nie konfiguruj globalnego NAT** i potwierdź, że nie odziedziczyłeś fabrycznej source NAT policy. To nie jest produkcyjna polityka dla guest Wi-Fi.

### 7.8.7. Etap D — najpierw zwykły default i działający ruch przez WAN-A

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], statyczna globalna default**

~~~junos
set routing-options static route 0.0.0.0/0 next-hop 198.51.100.2
show | compare
commit check
~~~

Po zatwierdzeniu i uruchomieniu bazowych adresów sprawdź, czy **oba clients** docierają do usługi `203.0.113.10` przez **WAN-A**. Bez aktywnego FBF obaj mają ten sam destination i ten sam default. To warunek wyjściowy — jeśli nie działa, nie dodawaj FBF.

**Urządzenie: LAB-PC-A-LINUX — Linux shell, baseline przed FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**Urządzenie: LAB-PC-B-LINUX — Linux shell, baseline przed FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**Urządzenie: LAB-UP-A-LINUX — Linux shell, obserwacja rzeczywistego ruchu przy testach baseline**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**Urządzenie: LAB-UP-B-LINUX — Linux shell, kontrola, że WAN-B nie jest jeszcze używany dla tego celu**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**Przewidywanie:** oba testy klienta mogą przechodzić przez WAN-A, B nie powinien widzieć ich request. **To nie jest wynik runtime**; jeśli R1 ma bardziej szczegółowy route, NAT, inne policy albo hardware zachowuje się inaczej, trzeba to wyjaśnić.

### 7.8.8. Etap E — budujemy aggregate i generated routes

Następnie dodajemy **bardziej szczegółowe aktywne routes** (contributors) oraz ich summary. Nie mieszamy ich z FBF — to niezależne mechanizmy routingu.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], contributorzy, aggregate i generated**

~~~junos
set routing-options static route 10.60.10.0/24 next-hop 198.51.100.2
set routing-options static route 10.60.20.0/24 next-hop 198.51.100.6
set routing-options aggregate route 10.60.0.0/16
set routing-options static route 10.70.10.0/24 next-hop 198.51.100.2
set routing-options generate route 10.70.0.0/16
show | compare
commit check
~~~

**Urządzenie: LAB-R1-vSRX — Operational Mode, odczyt po zatwierdzeniu na LAB**

~~~junos
show route 10.60.0.0/16 extensive
show route 10.60.10.0/24
show route 10.60.20.0/24
show route 10.70.0.0/16 extensive
show route 10.70.10.0/24
show route protocol aggregate
~~~

**Oczekiwanie:** aggregate `10.60.0.0/16` powinien mieć typowo **reject next-hop** i działać dzięki active /24, a generated `10.70.0.0/16` może odziedziczyć next-hop z `10.70.10.0/24` przez WAN-A. Obie rodziny mogą być pokazane jako `[Aggregate/130]`. Zweryfikuj szczegóły.

### 7.8.9. Etap F — testujemy blackhole nieistniejącej części summary

**Urządzenie: LAB-PC-B-LINUX — Linux shell, kontrolowany test trzech celów**

~~~bash
ping -c 5 10.60.10.10
ping -c 5 10.60.20.10
ping -c 5 10.60.99.25
~~~

**Oczekiwanie:**
- `10.60.10.10` idzie przez specific /24 do WAN-A;
- `10.60.20.10` idzie przez specific /24 do WAN-B;
- `10.60.99.25` pasuje do /16 aggregate **reject**, bo brak specific /24; może pojawić się ICMP unreachable zależnie od platformy.

**Pytanie:** czy `10.60.99.25` jest unreachable, bo `martian`? **Nie.** To prefix private, a w tym scenariuszu brak szczegółowej trasy spowodował dopasowanie aggregate reject. Nie myl filtracji prefixów z działaniem LPM.

### 7.8.10. Etap G — wygenerowana trasa może skierować ruch do primary contributor

**Urządzenie: LAB-PC-B-LINUX — Linux shell, test istnienia i braku adresu w generated summary**

~~~bash
ping -c 5 10.70.10.10
ping -c 5 10.70.99.25
~~~

**Urządzenie: LAB-UP-A-LINUX — Linux shell, przechwycenie nieobsługiwanego destination podczas generated forwarding**

~~~bash
sudo tcpdump -ni eth0 'host 10.70.99.25' -c 10
~~~

**Przewidywany mechanizm:** do `10.70.99.25` R1 nie ma bardziej szczegółowej trasy niż generated /16. Ponieważ generated może przejąć WAN-A next-hop z primary contributor, pakiety **mogą zostać wysłane do UP-A**, mimo że na jego loopback nie istnieje adres `10.70.99.25`. Ping może timeoutować, lecz **tcpdump widzący pakiet na UP-A** jest dowodem odróżniającym generated od aggregate reject.

Nie twierdź, że jedna nieudana odpowiedź ICMP dowodzi forwarding. Sprawdź capture i FIB.

### 7.8.11. Etap H — FBF forwarding instance i RIB group

Teraz robimy to, co oficjalny przykład Juniper FBF wskazuje jako konieczne do **next-hop resolution** w alternatywnej tablicy: najpierw tworzymy forwarding instance, a potem udostępniamy interface routes.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], alternatywna tablica WAN-B**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 0.0.0.0/0 next-hop 198.51.100.6
show | compare
commit check
~~~

**Urządzenie: LAB-R1-vSRX — Operational Mode, sprawdzenie forwarding instance przed RIB group**

~~~junos
show route table FBF-ALT.inet.0
show route table inet.0
~~~

Dopiero potem:

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], COPY-DIRECT dla IPv4 connected**

~~~junos
set routing-options rib-groups COPY-DIRECT import-rib [ inet.0 FBF-ALT.inet.0 ]
set routing-options interface-routes rib-group inet COPY-DIRECT
show | compare
commit check
~~~

**Urządzenie: LAB-R1-vSRX — Operational Mode, porównanie RIB po zatwierdzeniu**

~~~junos
show route table FBF-ALT.inet.0
show route table FBF-ALT.inet.0 198.51.100.4/30
show route table inet.0
show configuration routing-options rib-groups | display set
~~~

**Oczekiwanie:** connected `198.51.100.4/30` będzie dostępne w FBF-ALT, aby jej default mogła rozwiązać `198.51.100.6`. **Nie jest to automatyczne route leaking wszystkich statycznych tras globalnych**.

### 7.8.12. Etap I — włączamy FBF tylko dla PC-A

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], source-based FBF**

~~~junos
set firewall family inet filter CH07-PBR term SPECIAL from source-address 192.0.2.10/32
set firewall family inet filter CH07-PBR term SPECIAL then routing-instance FBF-ALT
set firewall family inet filter CH07-PBR term NORMAL then accept
set interfaces ge-0/0/1 unit 0 family inet filter input CH07-PBR
show | compare
commit check
~~~

**Jeszcze nie commituj w prawdziwym environment, jeśli nie masz niezależnej konsoli i poprawnych RIB/FIB.** W odizolowanym labie użyj procedury:

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], tylko po precheck i z działającą konsolą**

~~~junos
commit confirmed 5
~~~

Nie wykonuj `commit check` w trakcie pending confirmed commit; może nieświadomie potwierdzić zmianę. Wykonaj wyłącznie read-only i testy klientów; potwierdź później, jeśli **oba** są prawidłowe.

**Urządzenie: LAB-PC-A-LINUX — Linux shell, klient specjalny → ma wybrać WAN-B**

~~~bash
ping -c 5 203.0.113.10
~~~

**Urządzenie: LAB-PC-B-LINUX — Linux shell, klient zwykły → ma pozostać na WAN-A**

~~~bash
ping -c 5 203.0.113.10
~~~

**Urządzenie: LAB-UP-A-LINUX — Linux shell, obserwacja pakietów z klienta zwykłego**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 25
~~~

**Urządzenie: LAB-UP-B-LINUX — Linux shell, obserwacja pakietów z klienta FBF**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 25
~~~

**Oczekiwanie:** pakiety source `192.0.2.10` trafiają WAN-B, a source `192.0.2.11` do WAN-A. Zwróć uwagę na **kierunek pakietu i source** przy analizie tcpdump, żeby nie policzyć odpowiedzi jako request. Jeśli vSRX blokuje policy/flow, **nie potwierdzaj**, tylko zdiagnozuj; status labu pozostaje FAIL/NOT TESTED, nie PASS.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], dopiero po potwierdzonych wynikach**

~~~junos
commit
~~~

### 7.8.13. LAB-FAULT A — co, jeśli usuniemy jedynego contributora?

Najpierw przywróć normalny baseline i zakończ wszystkie confirmed timers. Następnie wyłączamy **oba** szczegółowe contributor prefixes do aggregate.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], tylko izolowany LAB, fault injection**

~~~junos
delete routing-options static route 10.60.10.0/24
delete routing-options static route 10.60.20.0/24
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: LAB-R1-vSRX — Operational Mode przez niezależną konsolę, read-only podczas pending timer**

~~~junos
show route 10.60.0.0/16 extensive
show route protocol aggregate
show system commit
~~~

**Pytanie:** czy aggregate dalej jest aktywny, gdy nie ma żadnego contributor? W normalnym trybie **nie powinien być**. Ale jeśli masz konfigurację `passive` lub inne contributing routes, wynik będzie inny. To dlatego przed testem sprawdziliśmy całą tablicę.

Po upływie timera bez potwierdzenia trasom /24 oraz aggregate powinien wrócić wcześniejszy stan. Nie wykonuj kolejnego commit ani commit check w trakcie oczekiwania.

### 7.8.14. LAB-FAULT B — jest FBF, ale RIB group zniknęła

**Cel:** udowodnić, że connected route w secondary RIB nie jest ozdobą. Tylko w odizolowanym labie usuwamy przypięcie interface routes do grupy.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], fault dotyczący RIB group**

~~~junos
delete routing-options interface-routes rib-group inet COPY-DIRECT
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: LAB-R1-vSRX — Operational Mode, bez dodatkowych commit/check**

~~~junos
show route table FBF-ALT.inet.0
show route table FBF-ALT.inet.0 198.51.100.4/30
show route table inet.0
show system commit
~~~

**Przewidywanie:** secondary RIB może utracić connected resolution do WAN-B i FBF-ALT default przestanie być użyteczna. Ale nie wpisuj „FBF failed” automatycznie: dokładne zachowanie zależy od tego, czy tabela miała inną drogę do next-hopa i jak vSRX przetwarza lookup. Sprawdź też tcpdump WAN-A/WAN-B i test PC-A.

Po rollbacku powinny wrócić prawidłowe connected routes, jeśli nic innego nie zostało zmienione.

### 7.8.15. LAB-FAULT C — zapomnieliśmy `NORMAL then accept`

**Cel:** pokazać, dlaczego błąd jednego końcowego term może odciąć wszystkich niepasujących klientów.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], świadomie błędny filter tylko w LAB**

~~~junos
delete firewall family inet filter CH07-PBR term NORMAL
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: LAB-PC-A-LINUX — Linux shell, klient pasujący do SPECIAL**

~~~bash
ping -c 5 203.0.113.10
~~~

**Urządzenie: LAB-PC-B-LINUX — Linux shell, klient NIEpasujący do SPECIAL**

~~~bash
ping -c 5 203.0.113.10
~~~

**Przewidywany rezultat:** PC-A może nadal działać przez FBF-ALT, ale PC-B może być **dropowany przez implicit firewall filter discard**. To jest realnie groźny haczyk z Junos: nawet jeśli miało się zmienić zachowanie jednego hosta, brak końcowego accept może uszkodzić pozostały site.

Nie potwierdzaj tego commita. Poczekaj na timer i sprawdź, czy PC-B wrócił.

### 7.8.16. LAB-FAULT D — generated route może forwardować do nieistniejącej części sieci

Wprowadzenie `generate route 10.70.0.0/16` było poprawne. W tym eksperymencie chcemy pokazać, dlaczego nie jest to zamiennik aggregate reject.

**Urządzenie: LAB-PC-B-LINUX — Linux shell, dwa adresy w obrębie generated summary**

~~~bash
ping -c 5 10.70.10.10
ping -c 5 10.70.99.25
~~~

**Urządzenie: LAB-UP-A-LINUX — Linux shell, potwierdzenie przekazania na upstream przy braku hosta docelowego**

~~~bash
sudo tcpdump -ni eth0 'host 10.70.99.25' -c 10
~~~

**Co powinieneś zaobserwować?** Przy wspieranej i aktywnej generated route R1 może forwardować do primary contributor next-hop, choć UP-A nie ma hosta `10.70.99.25`. Nie utożsamiaj więc nieudanego ping z brakiem forwardingu po stronie Junos. **Brak odpowiedzi to objaw; capture pokazuje, gdzie dotarł request.**

### 7.8.17. LAB-FAULT E — ECMP, jeden flow i fałszywy alarm „nierówny podział”

Dopiero po ukończeniu testów FBF przygotuj **osobny wariant baseline**. Tymczasowo wyłącz input FBF, żeby normalny routing stosował global `inet.0`, i utwórz dwa równorzędne static next-hops do identycznego anycast `203.0.113.10/32`.

**Urządzenie: LAB-R1-vSRX — Configuration Mode [edit private], osobny eksperyment ECMP**

~~~junos
delete interfaces ge-0/0/1 unit 0 family inet filter input CH07-PBR
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.2
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.6
set policy-options policy-statement CH07-ECMP term SERVICE from route-filter 203.0.113.10/32 exact
set policy-options policy-statement CH07-ECMP term SERVICE then load-balance per-packet
set policy-options policy-statement CH07-ECMP term SERVICE then accept
set policy-options policy-statement CH07-ECMP term REST then accept
set routing-options forwarding-table export CH07-ECMP
show | compare
commit check
~~~

**Urządzenie: LAB-R1-vSRX — Operational Mode, po osobnym commit**

~~~junos
show route 203.0.113.10/32 extensive
show route forwarding-table destination 203.0.113.10
show configuration routing-options forwarding-table | display set
~~~

**Urządzenie: LAB-PC-A-LINUX — Linux shell, jeden strumień ICMP**

~~~bash
ping -c 20 203.0.113.10
~~~

Oczekiwanie: jeden flow może cały czas korzystać z **jednej** drogi. Do dowodu ECMP potrzebujesz wielu flows, odpowiednich hash fields i odczytu obu upstreams. **Nie nazywaj braku dokładnego 50/50 błędem**.

**Uwaga:** na SRX/vSRX ECMP i stateful flows mają dodatkowe zależności, a hash-key options mogą różnić się od MX/QFX. Jeśli dana wersja vSRX nie wspiera przewidywanej składni, oznacz test **NOT TESTED** i zachowaj pełny zapis problemu.

### 7.8.18. LAB-FAULT F — martian nie jest tym samym co utrata odpowiedzi

W izolowanym labie porównaj domyślne entries martian oraz to, czy prefix `10.60.0.0/16` jest zwykłym route. **Nie zmieniaj polityki martians w podstawowym labie**, ponieważ może to wpłynąć na test contributorów i routing.

**Urządzenie: LAB-R1-vSRX — Operational Mode, tylko odczyt**

~~~junos
show route martians
show route 10.60.0.0/16
show route 127.0.0.0/8
~~~

Wniosek do egzaminu: `127.0.0.0/8` jest loopback/special i Junos ma odpowiednią martian logic; `10.60.0.0/16` nie jest zabronione automatycznie jako RFC1918.

### 7.8.19. Co trzeba porównać przed i po fault injection?

| Obszar | Baseline | W trakcie fault | Po recovery |
|---|---|---|---|
| Connected route do WAN-B | obecna w global i secondary | może zniknąć z secondary | przywrócona |
| Aggregate /16 | aktywny z contributorami | brak qualifying contributors → inactive | znów active |
| Generated /16 | next-hop primary contributor | może utracić eligibility | odtworzona |
| FBF special client | PC-A do WAN-B | bez connected nie rozwiąże next-hopa | WAN-B znów obsługuje PC-A |
| Normal traffic | PC-B do WAN-A | brak NORMAL term może dropować | WAN-A |
| ECMP eksperyment | osobny wariant bez FBF | single flow może być na jednym WAN | wariant testowy usunięty |
| Martians | zgodne z OS defaults | tylko odczyt | bez zmian |

**To są oczekiwania dydaktyczne**, nie dowody rzeczywistych outputów na EVE.

### 7.8.20. Cleanup, który przywraca stan i nie zostawia „ukrytej” zmiany

Po każdym fault:
- upewnij się, że **nie ma pending confirmed commit**;
- sprawdź, czy **obie** tablice zawierają odpowiednie connected routes;
- sprawdź, czy normalny firewall filter ma końcowy `NORMAL then accept`;
- potwierdź, że PC-A idzie WAN-B, a PC-B WAN-A;
- jeśli wykonałeś wariant ECMP, przywróć bazową FBF policy i usuń dodatkowy statyczny host route/forwarding-table export wyłącznie po porównaniu diff;
- zbierz faktyczny test z każdego Linux upstream.

**Urządzenie: LAB-R1-vSRX — Operational Mode, read-only final checklist**

~~~junos
show system commit
show configuration firewall family inet filter CH07-PBR | display set
show configuration routing-options forwarding-table | display set
show route 10.60.0.0/16 extensive
show route 10.70.0.0/16 extensive
show route table FBF-ALT.inet.0
show route table inet.0
show security flow session
show interfaces terse
~~~

**Urządzenie: LAB-PC-A-LINUX — Linux shell, końcowy test FBF klienta**

~~~bash
ping -c 5 203.0.113.10
~~~

**Urządzenie: LAB-PC-B-LINUX — Linux shell, końcowy test zwykłego klienta**

~~~bash
ping -c 5 203.0.113.10
~~~

**Uwaga:** jeżeli przypadkowo potwierdzisz zmianę z fault injection, nie zakładaj, że licznik rollback 1 zawsze wskazuje tę samą wersję co przed testem. Odczytaj historię, porównaj dokładny diff i przywróć świadomie baseline. Nie wykonuj destrukcyjnego `load override` ani globalnego `clear sessions all` jako „cleanup”.

### 7.8.21. Evidence package: co musi znaleźć się w raportach LAB

W pliku `VERIFY.md` wymagaj dla każdego ćwiczenia:
1. Image/version R1 i nazwy NIC Linux.
2. Segment ID, konfiguracja adresacji, efektywna tablica routingu.
3. Realny precheck `show route`, `show route martians`, `show configuration` (z sekretami zredagowanymi).
4. Output `show | compare` i wynik `commit check` **przed** aktywacją.
5. Faktyczny odczyt podczas fault i czas.
6. Capture z Linux WAN-A i WAN-B oraz source/destination.
7. Czego nauczyliśmy się o RIB vs FIB vs filter vs policy.
8. Czy auto-rollback się wykonał, czy nadal jest pending timer.
9. Ponowny test obu klientów po recovery.
10. Status **PASS / FAIL / NOT TESTED** oraz post-mortem bez prywatnych danych.

**Dopóki nie ma realnego odczytu EVE, nie nadawaj testowi PASS.**



---

## 7.EXT. Warsztat projektowy — jak wykorzystać te mechanizmy przy budowie nowego site'u Enterprise?

### 7.EXT.1. Nie dobieraj technologii, zanim nie rozpoznasz wymagania

Wyobraź sobie warsztaty HLD dla nowego biura na 800 użytkowników. Architekt pokazuje dwa uplinki, cztery sieci VLAN, jedno połączenie VPN do centrali i żądanie „rozłóżmy traffic między łączami”.

Żeby nie zaprojektować rozwiązania, które dobrze wygląda wyłącznie w prezentacji, zadaj pytania:
- Czy wymagane jest **load sharing**, czy wyłącznie **redundancy/failover**?
- Czy drugi ISP musi obsługiwać specyficzny ruch, np. backup/M365/guest, czy dowolny ruch?
- Czy mają pozostać **stałe source IP** i symetryczny egress dla stateful firewalla?
- Czy oba łącza mają to samo SLA, latency, QoS i przepustowość?
- Czy policy musi działać dla IPv6?
- Czy urządzenie wybrane w BOM jest **EX/QFX/MX/SRX** i na którym release ma wspierany FBF, RIB group i hash?
- Czy po awarii ISP-B ruch powinien **automatycznie wrócić na A**, czy zostać zablokowany z powodów polityki bezpieczeństwa?

Rozdział 1 omawiał portfolio access/distribution/core/security/WAN. Tutaj wiesz już, że **wybór feature musi być poparty macierzą model/release i możliwościami PFE**, nie tylko napisem „Junos” na pudełku.

### 7.EXT.2. Trzy architektury mogą wyglądać podobnie na diagramie, ale działają inaczej

**Wariant A — zwykły default plus backup:** jedna preferowana default via ISP-A, druga z wyższą route preference via ISP-B. To jest prosty failover, a nie active/active load balancing. Musisz zadbać o wykrywanie awarii poza lokalnym next-hop.

**Wariant B — ECMP:** dwie równorzędne drogi do tego samego destination. Jeżeli źródło tras i platforma wspierają multipath, a odpowiednia forwarding policy instaluje kilka next-hopów, ruch może być dzielony przez hash. Ale **jeden duży flow** może pozostać na jednym łączu.

**Wariant C — FBF:** filtr klasyfikuje źródło/protokół/DSCP i kieruje pakiety do odrębnego routing context, np. guest na ISP-B, office na ISP-A. Nie jest to automatyczny load balancing i wymaga starannie zaprojektowanej obsługi utraty ISP-B, policy/NAT i return path.

W praktyce te warianty mogą się łączyć, ale każda kolejna warstwa komplikuje troubleshooting.

~~~mermaid
flowchart TD
  GOAL["HLD: dwa WAN"] --> Q{"Co chcemy osiągnąć?"}
  Q -->|Primary/backup| FLOAT["Floating static / dynamic protocol\nplus failure detection"]
  Q -->|Wiele równorzędnych flows| ECMP["ECMP + forwarding-table policy\nhash fields + test wielu flows"]
  Q -->|Konkretny rodzaj/source ruchu| FBF["FBF + forwarding instance\nRIB group + security path"]
  FLOAT --> POST["Pre/post: oba kierunki ruchu\nweryfikacja klienta"]
  ECMP --> POST
  FBF --> POST
~~~

### 7.EXT.3. FBF bez failover planu może obniżyć dostępność

Często pojawia się na spotkaniach architektonicznych zdanie: „Nie chcemy, żeby aplikacje backupowe zapychały WAN-A. Przekierujmy je przez WAN-B”. Brzmi sensownie. A co się stanie, gdy WAN-B przestanie działać, ale jego interfejs nadal będzie fizycznie `up`?

Jeśli FBF instance ma tylko default via B, ruch backupowy będzie próbował tam wychodzić. **RIB group z connected routes nie tworzy automatycznie health-check do zdalnej aplikacji.** Może być potrzebny BFD, routing dynamiczny, mechanizm RPM/monitorowania i dodatkowa fallback route w kontekście FBF, zależnie od release oraz ryzyka.

Zanim zatwierdzisz projekt, odpowiedz:
- czy awaria WAN-B ma powodować **failover** na WAN-A, czy świadome **fail-closed**?
- czy w obu stanach ruch nie omija inspekcji?
- czy operator B musi mieć osobny source NAT publiczny?
- czy klient akceptuje przerwę sesji po failover?

**Most z Cisco:** dokładnie taki sam problem może wystąpić z PBR route-map bez poprawnego tracking. Reguła „przekieruj wybrane pakiety” nie zapewnia automatycznie obsługi wszystkich awarii.

### 7.EXT.4. Nie traktuj aggregate jako zastępstwa pełnej dokumentacji reachability

Załóżmy, że w nowym kampusie masz:
- Building-A: `10.60.10.0/24`;
- Building-B: `10.60.20.0/24`;
- przyszłą rezerwację pod Building-C: `10.60.30.0/24`;
- summary `10.60.0.0/16`.

Dzięki aggregate możesz reklamować /16, ale jeżeli Building-C **jeszcze nie istnieje**, pakiet do `10.60.30.20` prawdopodobnie trafi w aggregate reject. To jest poprawny, przewidywalny efekt. Nie przesuwaj ticketu do „server team” bez sprawdzenia, czy subnet naprawdę został wdrożony jako contributor.

Przy HLD i change tracking przechowuj:
- wszystkie podsieci składowe;
- ich rzeczywistą lokalizację i rolę;
- next-hop / routing source;
- status ogłoszenia summary;
- impact po utracie jednego contributor;
- politykę drop dla brakujących podprefiksów.

### 7.EXT.5. Bezpieczeństwo danych i szczególne pułapki przy kopiowaniu konfiguracji

RIB groups i FBF mogą dotknąć **izolacji klientów**, dlatego powinny być przeglądane razem z zespołem Security/Architecture. Jeśli skopiujesz do `FBF-ALT.inet.0` wszystkie connected, możesz ujawnić w tej tablicy także subnets management/OT. Samo widzenie route nie zawsze daje pełną komunikację, ale zmienia możliwość rozwiązywania next-hop i warunki późniejszego forwardingu.

**Najlepsza praktyka:** na poziomie blueprintu nie traktuj „routing-table leaking” jako wyłącznie ćwiczenia z syntax. To **zmiana modelu trust boundaries**, którą trzeba pokazać na schemacie i zatwierdzić.

---

## 7.FIELD. Mapa poleceń do pracy w incydencie — już po pełnej lekcji

| Gdy pytasz... | Odczyt Junos | Co może zmylić |
|---|---|---|
| Czy summary naprawdę istnieje? | `show route 10.60.0.0/16 extensive` | Nie każda skonfigurowana aggregate jest active |
| Jakie trasy stanowią contributors? | `show route table inet.0`, specific longer routes | Contributor musi być aktywny, bardziej szczegółowy |
| Czy generated ma forwarding next-hop? | `show route 10.70.0.0/16 extensive` | Obie rodziny mogą pokazywać `Aggregate/130` |
| Dlaczego prefix wygląda na niemożliwy? | `show route martians`, `show route hidden` | Martian ≠ firewall packet filter |
| Co wie global? | `show route table inet.0` | FBF może korzystać z innej RIB |
| Co wie FBF? | `show route table FBF-ALT.inet.0` | Default bez connected resolution to za mało |
| Czy RIB group działa? | `show configuration routing-options rib-groups` | Sama definicja bez attachment nie działa |
| Jakie są connected w secondary? | `show route table FBF-ALT.inet.0` | Wsparcie i route-types zależą od platformy |
| Czy FBF filter jest przypięty? | `show configuration interfaces ge-0/0/1` | Filter bez ingress attachment nie ma wpływu |
| Czy pozostałe pakiety są przyjęte? | `show configuration firewall family inet filter CH07-PBR` | Brak końcowego `accept` może dropować |
| Czy FIB zna ECMP? | `show route forwarding-table destination IP` | Multiple candidates w RIB ≠ multi-next-hop w PFE |
| Czy ECMP rzeczywiście dzieli ruch? | Capture/counters na obu egress | Jedno ICMP flow to zły test |
| Czy SRX policy przepuszcza? | `show security flow session`, `show security policies` | Stateless accept ≠ stateful permit |
| Czy po rollbacku działa klient? | Ping/TCP z dwóch odrębnych PC | Ping z RE nie zastępuje testu end-to-end |

---

## 7.SOURCES. Weryfikacja w dokumentacji producenta i pochodzenie pytań

**Oficjalne źródła Juniper/HPE:**

1. [Route Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html) — contributing routes, aggregate default reject, summary export.
2. [Conditionally Generated Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/concept/policy-generated-routes.html) — generated next-hop based on primary contributor.
3. [Aggregate/Generated Preference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/preference-edit-routing-options.html) — default 130.
4. [Martian Address Recognition](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/recognize-martian-addr-routing.html) i [show route martians](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-martians.html) — permitted/disallowed scopes i format exact/orlonger.
5. [Routing Instances Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html) — virtual-router, forwarding, vrf i inne typy.
6. [RIB Groups CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html), [import-rib](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html), [interface-routes](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/interface-routes-edit-routing-options.html) — primary/secondary, attachment, EX limitations.
7. [Juniper Introduction to RIB Groups, Technical Note](https://www.juniper.net/documentation/us/en/software/nce/rg-understanding-tn/rg-understanding-tn.pdf) — import/export model oraz wykorzystanie przy FBF.
8. [Per-Flow Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-flow-load-balancing.html), [Per-Packet Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-packet-load-balancing.html) — nazwy opcji a realny per-flow hash.
9. [hash-key](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/hash-key-edit-forwarding-options.html) — fields do hashing i ograniczenia PFE.
10. [Filter-Based Forwarding — konfiguracja](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html), [FBF Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html), [FBF by source](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/firewall-filter-option-filter-based-forwarding-example.html).
11. [FBF limitations on ACX](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/filter-based-forwarding-acx-series.html) — niedozwolone kombinacje term actions i IPv6 L4 restrictions, jako **platform-specific przykład**, nie ogólna zasada dla wszystkich.
12. [Understanding FBF on EX](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/filter-based-forwarding-ex-series.html) oraz [Feature Explorer](https://apps.juniper.net/feature-explorer/).
13. [Advanced Policy-Based Routing for SRX](https://www.juniper.net/documentation/us/en/software/junos/application-identification/topics/topic-map/security-application-advanced-policy-based-routing.html) — osobna ścieżka przy określonych family/release.
14. [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html) — ograniczenia interpretacji FIB i next-hop.
15. [show route advertising-protocol](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-advertising-protocol.html) — rozróżnienie RIB route i rzeczywistego protokolarnego export.

**Rzeczywiście opublikowane pytania/dyskusje Juniper Community:**
- [Aggregate VS Generate routes](https://community.juniper.net/discussion/aggregate-vs-generate-routes) — autentyczne pytanie o podobieństwa, pref 130 i identyczną etykietę Aggregate.
- [Aggregate Routes vs Generated Routes](https://community.juniper.net/discussion/aggregate-routes-vs-generated-routes?hlmlt=VT) — analiza domyślnego next-hop i struktury output.
- [Why is the default route in the default Martian table?](https://community.juniper.net/discussion/why-is-the-default-route-in-the-default-martian-table) — autentyczna wątpliwość o `0.0.0.0/0 exact allow` i `0/8 orlonger disallow`.
- [FBF on SRX](https://community.juniper.net/discussion/filter-based-forwarding-on-srx) — inspiracja do jawnego oznaczania platform-specific flow caveats; nie traktowana jako gwarancja wspierania na każdym obrazie vSRX.

**Klasyfikacja pochodzenia:** źródłowe pytania Community wskazano wyżej; pozostałe narracje i wszystkie fault injections są **autorskimi konstrukcjami dydaktycznymi na podstawie dokumentacji producenta**. Nie cytujemy poufnych materiałów ani exam dumpów. Dokumentacja i Feature Explorer mają pierwszeństwo nad niezweryfikowaną odpowiedzią z forum.

**Media:** wszystkie schematy Mermaid są oryginalne. Nie wykorzystano zewnętrznych zdjęć routerów tylko dla ozdoby, bo w rozdziale o tablicach routingu i kierunku pakietów własne diagramy są bardziej precyzyjne. Nie należy deklarować istnienia lokalnych JPG/PNG, których nie dodano do repo.

**Ograniczenie laboratoryjne:** nie wykonano testu na realnym EVE-NG, nie pobrano żadnych obrazów Junos i nie potwierdzono na konkretnym vSRX wsparcia FBF, ECMP policy ani RIB group. Dla EX/QFX/MX, SRX/VS, Junos OS Evolved i odmiennych PFE mogą występować różnice. Wszystkie konfiguracje wymagają odpowiedniej walidacji i weryfikacji runtime.

---

## 7.END. Epilog — wiesz już, że Junos ma więcej niż jedną mapę drogową

Wracamy do porannego incydentu w nowym biurze. R1 ma dwie drogi WAN, operator B obsługuje specjalny ruch, a partner dostaje summary `10.60.0.0/16`. Teraz zamiast ogólnego „na routerze jest trasa” potrafisz zapytać:

- **Czy to static, aggregate czy generated?** Czy potrzebny jest contributor? Co robi route, gdy destination nie ma specific?
- **Czy prefix jest martian?** Czy rozumiem exact/orlonger i allow/disallow?
- **Która routing instance odpowiada za ten pakiet?** Czy został przekierowany przez FBF?
- **Czy RIB group udostępniła potrzebne connected do secondary?** Czy next-hop jest rozwiązywany?
- **Czy FIB ma jeden czy kilka realnych next-hopów?** Czy używany jest per-flow hash?
- **Czy stateful security, NAT i return path zgadzają się z projektem?**
- **Jakim testem z klienta i upstreamu udowodnię rzeczywistą drogę?**

To poziom **JNCIS-ENT**: nie tylko umiesz skonfigurować polecenie, ale **potrafisz przewidzieć skutek jego interakcji z innymi mechanizmami**.

W rozdziale 8 przejdziemy głębiej do routing policy i firewall filters: term evaluation, default actions, import vs export, match criteria, termination i uRPF. Tam nauczysz się sterować tym, **które route i pakiety są dopuszczone**, a nie tylko którą tablicę wykorzystują.

**Blueprint: 7.1–7.8 [S].**  
**Status:** wykład i specyfikacja LAB opracowane; **LAB PLANNED / NOT RUNTIME VALIDATED**.

