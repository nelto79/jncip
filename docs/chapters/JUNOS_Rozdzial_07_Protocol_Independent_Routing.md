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

