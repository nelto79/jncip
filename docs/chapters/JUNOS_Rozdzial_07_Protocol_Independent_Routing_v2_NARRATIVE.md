# Rozdział 7. Protocol-Independent Routing — kiedy zwykła tablica tras przestaje wystarczać

*JUNOS Enterprise Routing & Switching — JNCIS-ENT, z mostami do JNCIA i JNCIP*

**Wersja redakcyjna:** 2.0 — 19 listopada 2026. **Zakres:** 7.1–7.8 z [Unified Blueprint](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md), z praktycznymi rozszerzeniami projektowymi.  
**Status:** materiał opracowany i zweryfikowany redakcyjnie względem źródeł producenta; **NOT RUNTIME VALIDATED**. Przewidywane wyniki CLI i testów to *oczekiwania*, nie wyniki uruchomionego EVE-NG.  
**Odbiorca:** inżynier, który zna Cisco IOS/IOS XE, RIB/FIB, routing statyczny, SVI/VRF i PBR, a chce naprawdę zrozumieć Junos.  
**Platformy:** klasyczny Junos, ze wskazaniem różnic EX/QFX/MX oraz vSRX. Lab z FBF jest **warunkowy**: konkretne możliwości obrazu i ścieżkę flow/security trzeba sprawdzić przed wykonaniem.

---

## Prolog. Poniedziałek, 8:17 — dwóch operatorów, a tylko jedna działająca droga

Nowe biuro firmy jest już niemal gotowe. Są przełączniki dostępowe, firewall/router przy wyjściu do WAN, dwa łącza operatorskie i zespół pracowników, który za dwa dni ma tu rozpocząć pracę. Z punktu widzenia projektu wszystko wygląda dobrze. Operator A jest głównym wyjściem do usług, operator B ma przejmować wybrane połączenia i pełnić rolę alternatywy.

Rano dostajesz telefon z NOC:

> „R1 ma dwa uplinki up/up, default route jest w tabeli, a mimo to część użytkowników nie może dotrzeć do aplikacji. Gdy zmieniamy default na operatora B, zaczynają działać jedni, a przestają drudzy. Czy naprawdę musimy budować dwa osobne routery?”

Nie musimy. Ale musimy nauczyć się odróżniać kilka pytań, które zwykle traktujemy jak jedno. **Jaką trasę router zna?** To pierwsze. **Czy ta trasa powinna istnieć tylko wtedy, gdy istnieje coś bardziej szczegółowego?** Drugie. **Którą tablicę należy wykorzystać dla danego pakietu?** Trzecie. **Czy obie równorzędne drogi faktycznie znalazły się w forwarding table?** Czwarte.

W rozdziale 6 nauczyliśmy się, jak R1 wybiera trasę na podstawie route preference i longest prefix match. Teraz budujemy nad tym bardziej zaawansowane mechanizmy. Żaden z nich nie zastępuje zwykłego routingu. Każdy rozwiązuje inny kłopot, który pojawia się dopiero wtedy, gdy sieć staje się bardziej złożona.

Przez cały rozdział będziemy wracać do jednej, niewielkiej topologii. Nie będziemy losowo przeskakiwać między pięcioma różnymi firmami. Na początku mamy tylko dwa wyjścia WAN. Później pojawi się sieć użytkowników, drugi klient o innych wymaganiach, agregacja prefiksów, druga tablica routingu i filtr wybierający tę tablicę.

~~~mermaid
flowchart LR
  subgraph OFFICE["Biuro: OFFICE — 192.0.2.0/24"]
    A["PC-A: 192.0.2.10"]
    B["PC-B: 192.0.2.11"]
  end
  A --- R["R1 — Junos<br/>brama 192.0.2.1"]
  B --- R
  R ---|"WAN-A: 198.51.100.1/30"| UA["UP-A — Linux<br/>198.51.100.2"]
  R ---|"WAN-B: 198.51.100.5/30"| UB["UP-B — Linux<br/>198.51.100.6"]
  UA --> APP["Wspólny testowy cel:<br/>203.0.113.10"]
  UB --> APP
~~~

*Rysunek 7.1 (autorski). Tekst alternatywny: dwóch klientów PC-A i PC-B łączy się z routerem R1; R1 ma osobny WAN-A do upstream A i WAN-B do upstream B; testową usługę da się osiągnąć przez oba upstreamy. W laboratorium oba upstreamy mają własny loopback 203.0.113.10/32, ale są w rozłącznych segmentach L2. Schemat pokazuje ideę, a nie gotowy EVE-NG.*

Przyjęliśmy specjalnie **adresy dokumentacyjne** RFC 5737 dla OFFICE, transitów i usługi. To nie są przykładowe adresy do skopiowania do firmowego WAN. Całe ćwiczenie ma działać bez wyjścia do Internetu. Jeśli obraz vSRX nie obsłuży klasycznego FBF dokładnie tak jak MX, wyjaśnimy ograniczenie zamiast udawać, że coś zadziałało.

Na końcu sam wywołasz błędy: zniknięcie trasy podsumowującej, unresolved next-hop w dodatkowej tablicy, zablokowanie zwykłych klientów przez filtr i pozornie nieskuteczny ECMP. Zanim do tego dojdziemy, musimy zbudować intuicję.

---

## 7.1. Static, aggregate i generated — trzy sposoby powiedzenia routerowi, dokąd prowadzi droga [S]

### Najpierw zwykła static: nie trzeba odkrywać sieci, wystarczy znać następnego hopa

Wyobraź sobie, że PC-B wysyła pakiet do 10.60.10.10 — usługi, która znajduje się za UP-A. R1 ma bezpośrednie połączenie WAN-A z 198.51.100.2. Jeśli administrator doda statyczną trasę, router nie musi negocjować żadnego BGP ani OSPF. Wystarczy, że potrafi rozwiązać next-hop i zainstalować poprawny forwarding.

**R1-vSRX — Configuration Mode [edit], pierwszy przykład statycznej drogi**

~~~junos
set routing-options static route 10.60.10.0/24 next-hop 198.51.100.2
~~~

Ta linijka nie jest „pakietem” ani reklamą do sąsiada. Jest intencją administratora zapisaną w konfiguracji. Jeśli 198.51.100.2 jest osiągalny jako next-hop, R1 może użyć trasy. Jeśli interfejs WAN-A zostanie odłączony, statyczna trasa może przestać być aktywna; nie pomaga jej to, że konfiguracja nadal zawiera tekst \`set routing-options static route\`.

To dobry moment na porównanie z Cisco: \`ip route 10.60.10.0 255.255.255.0 198.51.100.2\` rozwiązuje podobny problem. Ale na Junos osobno obserwujemy status w RIB, kwalifikację next-hop i stan FIB. **Konfiguracja, RIB i rzeczywiste forwarding nie są trzema nazwami na to samo.**

**R1-vSRX — Operational Mode, skąd wiemy, że trasa działa**

~~~junos
show route 10.60.10.0/24 exact extensive
show route forwarding-table destination 10.60.10.10
show arp no-resolve
~~~

Jeśli trasa ma aktywny next-hop przez WAN-A, forwarding do 10.60.10.10 może być na niego skierowany. Ale samo to nie dowodzi, że serwer istnieje ani że SRX przepuści daną sesję. Ping PC-B do serwera jest osobnym dowodem.

### Drugi telefon: „Nie chcemy reklamować setek /24. Nie da się ich zamknąć w jednym /16?”

Sieć się rozrasta. Mamy podsieć 10.60.10.0/24 za UP-A i 10.60.20.0/24 za UP-B. W centrali ktoś chce reklamować partnerowi jeden czytelny prefiks 10.60.0.0/16. Po co? Żeby partner nie musiał znać każdego szczegółu firmowej adresacji i nie dostawał kolejnej zmiany BGP, gdy pojedynczy dział przeprowadza się do innego budynku.

To jest moment, w którym pojawia się **aggregate route**. Nie jest zwykłą statyczną trasą do konkretnego operatora. Jest podsumowaniem tego, co R1 zna *pod spodem*.

Najpierw w RIB muszą istnieć co najmniej jedna **aktywna, bardziej szczegółowa trasa** w zakresie agregatu. Te trasy nazywamy *contributors*. Jeśli mamy 10.60.10.0/24 oraz 10.60.20.0/24, obie mogą przyczyniać się do aktywności 10.60.0.0/16.

**R1-vSRX — Configuration Mode [edit], dokładamy drugą drogę i agregat**

~~~junos
set routing-options static route 10.60.20.0/24 next-hop 198.51.100.6
set routing-options aggregate route 10.60.0.0/16
~~~

Od razu zapytajmy o pakiet do 10.60.10.10. Czy po dodaniu /16 zacznie iść jakąś „drogą agregatową”? Nie. W RIB nadal jest /24 i LPM wybiera **bardziej szczegółową trasę**. Pakiet dotrze do UP-A tak samo jak wcześniej, jeśli ścieżka działa.

A co z 10.60.99.25? To adres, który mieści się w podsumowaniu /16, ale nie jest objęty żadną z naszych prawdziwych podsieci /24. Tu zaczyna działać głęboki sens agregatu: **domyślny next-hop aggregate to reject**. R1 nie przekazuje pakietu na ślepo do przypadkowego upstreama. Zwraca brak osiągalności w sposób zależny od platformy i warunków filtrowania.

~~~mermaid
flowchart TD
  P["Pakiet do sieci 10.60.x.x"] --> L{"Najbardziej szczegółowa trasa?"}
  L -->|10.60.10.10| A["10.60.10.0/24 → UP-A"]
  L -->|10.60.20.10| B["10.60.20.0/24 → UP-B"]
  L -->|10.60.99.25| S["10.60.0.0/16 → reject"]
  S --> X["Brak realnego /24: pakiet nie idzie do ISP"]
~~~

*Rysunek 7.2 (autorski). Tekst alternatywny: pakiety do rzeczywistych /24 idą przez odpowiednie upstreamy, a adres istniejący tylko w podsumowaniu /16 trafia do reject next-hop. Wniosek zależy od aktywności agregatu i brakujących bardziej szczegółowych tras.*

To wygląda jak Cisco static summary do \`Null0\`, ale analogia kończy się w ważnym miejscu. Junos \`aggregate\` **zwykle zależy od contributors**. Nie jest z definicji niezmienną, bezwarunkową trasą /16. Można modyfikować jej zachowanie opcjami, takimi jak \`passive\` czy policy contributorów, więc zawsze sprawdzaj konkretną konfigurację.

**R1-vSRX — Operational Mode, najpierw contributors, potem aggregate**

~~~junos
show route 10.60.10.0/24 exact
show route 10.60.20.0/24 exact
show route 10.60.0.0/16 exact extensive
show route forwarding-table destination 10.60.99.25
show configuration routing-options aggregate | display set
~~~

Jeśli aggregate jest aktywny, ale w RIB nie ma spodziewanych contributorów, nie zakładaj natychmiast błędu Junos. Sprawdź, czy istnieją inne trasy bardziej szczegółowe, czy skonfigurowano policy contributorów albo \`passive\`. Jeśli /16 nie jest aktywne mimo poprawnego \`commit\`, również nie mamy sprzeczności: commit zatwierdza konfigurację, ale nie tworzy kwalifikującego się contributora.

### Gdy zostaje jedna /24, czy podsumowanie mówi prawdę?

Załóżmy, że WAN-A przestaje działać i trasa 10.60.10.0/24 znika, ale 10.60.20.0/24 przez B nadal jest aktywna. Agregat /16 może **pozostać aktywny**. To może zaskoczyć: przecież połowa usług w firmie nie jest osiągalna.

Z punktu widzenia Junos wystarczy contributor spełniający warunki aktywności agregatu. Z punktu widzenia partnera nadal może istnieć reklama całego /16, jeśli dodatkowo skonfigurowaliśmy odpowiednią **routing export policy**. Wtedy ruch do niedostępnych części zakresu zostanie przyciągnięty do R1 i odrzucony przez aggregate reject, jeśli brak bardziej szczegółowej trasy.

Nie znaczy to, że agregacja jest zła. Znaczy to, że podsumowanie **świadomie ukrywa szczegóły**. Jeśli umowa SLA wymaga przerwania reklamy summary dopiero po utracie określonego zestawu usług, musimy zaprojektować policy contributorów i warunek dostępności znacznie staranniej niż „jedna /24 wystarczy”.

Także oszczędność tablic działa nie tam, gdzie wielu początkujących szuka: agregacja sama w sobie **nie usuwa contributorów z lokalnej RIB**. R1 nadal potrzebuje szczegółowych /24. Rozmiar tabel u *sąsiadów* może się zmniejszyć, gdy świadomie reklamujemy /16 zamiast kilku /24. Samo \`set aggregate\` jeszcze niczego nie reklamuje. To temat export policy z rozdziału 8.

### Trzecia konstrukcja: generated route. Podobna nazwa, zupełnie inna odpowiedź na pakiet

Teraz potrzebujemy czegoś innego. Chcemy, aby R1 tworzył trasę 10.70.0.0/16 **tylko wtedy**, gdy istnieje aktywna droga do 10.70.10.0/24. Ale zamiast zatrzymywać pakiet do innej części /16, chcemy wysłać go przez następny hop tej istotnej /24. Wtedy rozważamy **generated route**.

**R1-vSRX — Configuration Mode [edit], oddzielny przykład generated**

~~~junos
set routing-options static route 10.70.10.0/24 next-hop 198.51.100.2
set routing-options generate route 10.70.0.0/16
~~~

Tutaj również istnieje contributor. Ale **generated route może przejąć next-hop od primary contributing route**. W naszej prostej sieci będzie nim droga do 10.70.10.0/24 przez 198.51.100.2. Zatem pakiet do 10.70.99.25, choć nie ma własnej /24, może zostać wysłany do UP-A. Tam upstream może go odrzucić. Sam ping nie wystarczy, żeby powiedzieć, czy R1 wysłał pakiet: trzeba zobaczyć forwarding table albo ruch na interfejsie UP-A.

~~~mermaid
flowchart LR
  C["Aktywny contributor<br/>10.70.10.0/24 via UP-A"] --> G["Generated<br/>10.70.0.0/16"]
  G --> N["Next-hop z primary contributor<br/>198.51.100.2"]
  P["Pakiet do 10.70.99.25"] --> G
  N --> W["WAN-A: pakiet może wyjść<br/>mimo braku hosta 10.70.99.25"]
~~~

*Rysunek 7.3 (autorski). Tekst alternatywny: generated /16 powstaje na podstawie aktywnego, bardziej szczegółowego contributora i może odziedziczyć jego next-hop. W przeciwieństwie do aggregate reject kieruje pakiety do wyjścia.*

Jeśli contributorów jest kilku, w grę wchodzą reguły wyznaczania **primary contributor**, a nie proste „wybieramy bardziej szczegółową trasę dla każdej wysyłanej paczki”. Wybór contributorów, ich preferencje i dodatkowe policy mogą zmienić wynik. Również funkcje \`active\`/\`passive\`, \`next-table\` i pozostałe warianty wymagają osobnej analizy; nie buduj produkcyjnej usługi na uproszczeniu „generate = aggregate bez reject”.

**R1-vSRX — Operational Mode, dwa pytania: czy jest route i dokąd faktycznie kieruje**

~~~junos
show configuration routing-options generate | display set
show route 10.70.0.0/16 exact extensive
show route 10.70.10.0/24 exact
show route forwarding-table destination 10.70.99.25
show route protocol aggregate
~~~

Ostatnie polecenie może wydawać się dziwne: dlaczego przy \`generate\` sprawdzamy \`aggregate\`? Ponieważ **obie rodziny mogą być prezentowane w Junos jako \`[Aggregate/130]\`**. Ich domyślna preference wynosi zazwyczaj 130, podczas gdy standardowa static ma preference 5. Nie wnioskuj z etykiety protokołu wyłącznie na podstawie jednej linii \`show route\`. Sprawdź konfigurację oraz next-hop.

To też oznacza, że **martian, static, aggregate i generated nie są czterema równoważnymi sposobami filtrowania routingu**. Najpierw zdecyduj, czy potrzebujesz realnej stałej trasy, warunkowego bezpiecznego summary czy warunkowej trasy przekazującej ruch dalej.

Przed przejściem dalej przeanalizuj trzy cele: 10.60.10.10, 10.60.99.25 i 10.70.99.25. Pierwszy wybierze konkretną /24; drugi może trafić w aggregate reject; trzeci może opuścić R1 przez WAN-A dzięki generated. Wszystkie trzy zachowania mogą być jednocześnie poprawne.

*Źródła techniczne: [Juniper — Configuring Route Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html), [Understanding Conditionally Generated Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/concept/policy-generated-routes.html), [Static Route Preferences](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html), [Route Aggregation CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/aggregate-edit-routing-options.html).*


---

## 7.2. Martian addresses — dlaczego router czasem odmawia nawet rozważenia trasy [S]

### „Partner wysyła prefix, a Junos udaje, że go nie zna”

Zaczęliśmy układać własne trasy, ale prawdziwy WAN ma jeszcze jedną komplikację: nie wszystkie informacje, które router otrzymuje, należy traktować jako sensowne. Pewnego dnia na sesji BGP widzisz próbę ogłoszenia **127.10.20.0/24**. Operator mówi: „Przecież to poprawnie zapisany prefiks IPv4. Dlaczego nie znalazł się w normalnej tablicy?”

Na poziomie samej matematyki adresowania rzeczywiście wygląda poprawnie. Ale zakres **127.0.0.0/8** jest specjalny — przeznaczony dla ruchu loopback. To nie powinna być zwykła trasa do zdalnej lokalizacji. Junos ma mechanizm **martian routes**, który pozwala rozpoznawać prefiksy niedopuszczalne lub specjalne w informacjach routingu.

Ważne, żeby nie przenosić na niego zbyt wielu skojarzeń. Martian route **nie jest filtrem TCP/443**, nie jest ACL na wejściu do interfejsu ani zamiennikiem uRPF. To mechanizm kwalifikacji *informacji o trasach*. Jeżeli pakiet do jakiejś sieci nie przechodzi, nie wystarczy stwierdzić „może martian”, bo równie dobrze może nie być next-hopa, aktywnej trasy, security policy albo return path.

Przyjrzyjmy się dwóm prefiksom. **10.60.0.0/16** to normalny fragment prywatnej przestrzeni RFC1918; budujemy na nim własne trasy aggregate. **127.10.20.0/24** należy do zakresu loopback. Gdybyś po prostu oznaczył wszystkie sieci niepubliczne jako „martians”, odciąłbyś znaczną część korporacyjnego routingu.

**R1-vSRX — Operational Mode, sprawdzamy rzeczywiste zasady, nie zgadujemy**

~~~junos
show route martians
show route martians table inet.0
show configuration routing-options martians | display set
~~~

W pierwszym poleceniu widzisz nie tylko prefiksy, ale również to, czy są **allowed** czy **disallowed**. Drugie zawęża odczyt do konkretnej tabeli, a trzecie pozwala odróżnić domyślną listę Junos od zmian administratora. To szczególnie ważne, gdy urządzenie odziedziczyło niestandardową konfigurację z poprzedniego projektu.

### Dlaczego default route jest na liście martians — i wcale nie musi być zabroniona?

W dokumentacji Juniper można zobaczyć wpisy podobne do:

~~~text
0.0.0.0/0 exact       -- allowed
0.0.0.0/8 orlonger    -- disallowed
127.0.0.0/8 orlonger  -- disallowed
240.0.0.0/4 orlonger  -- disallowed
~~~

*Fragment ilustruje przykładową postać outputu z dokumentacji producenta, **nie** rzeczywisty wynik naszego R1. Pełna lista jest zależna od rodziny adresów, wydania i konfiguracji.*

Na pierwszy rzut oka może pojawić się myśl: „Skoro 0.0.0.0/0 jest na liście martian, to nie powinienem mieć default route!”. Wystarczy jednak doczytać do końca: przy **0.0.0.0/0 exact** jest **allowed**. A kolejna reguła **0.0.0.0/8 orlonger** dotyczy zupełnie innego zbioru prefiksów; nie dotyczy default /0.

To jest ta sama precyzja, której nauczyłeś się na Cisco prefix-list. Różnica pomiędzy \`0.0.0.0/0\` a \`0.0.0.0/0 le 32\` jest ogromna. W Junos \`exact\` obejmuje podany prefiks i jego długość, a \`orlonger\` obejmuje również bardziej szczegółowe prefiksy mieszczące się w danym zakresie. Dlatego wpis dla 127.0.0.0/8 **orlonger** obejmie np. 127.10.20.0/24.

### A jeśli chcę świadomie zmienić listę?

Można dodać regułę, ale najpierw trzeba określić, *jaką informację chcemy zakazać*. Załóżmy, że w osobnym ćwiczeniu chcemy potraktować prefix testowy **198.18.0.0/15** jako niedozwolony w kwalifikacji tras. Uwaga: ta przestrzeń jest używana do testów benchmarkowych i nasza polityka byłaby świadomym ograniczeniem; nie zakładaj, że jest domyślnie martian.

**R1-MARTIAN-LAB — Configuration Mode [edit], osobny izolowany eksperyment, NIE na R1 produkcyjnym**

~~~junos
set routing-options martians 198.18.0.0/15 orlonger
show | compare
commit check
~~~

Dlaczego nie robimy tego na głównym R1? Bo modyfikowanie globalnych reguł routingu podczas nauki FBF i agregacji popsułoby nam wartość kolejnych eksperymentów. Nie widzielibyśmy już czystej różnicy między brakiem trasy, niekwalifikowanym prefiksem a odrzuceniem pakietu.

W dokumentacji Junos występuje również możliwość \`allow\` dla jawnego wyjątku od reguł martian. Producent pokazuje np. sposób dopuszczania przestrzeni **240.0.0.0/4**, ale potraktujmy go jako **pokaz składni**, nie jako receptę na firmowy problem.

**R1-MARTIAN-LAB — Configuration Mode [edit], TYLKO ilustracja wyjątku; nie wdrażać jako poprawki**

~~~junos
set routing-options martians 240.0.0.0/4 orlonger allow
~~~

Router przyjmujący prefiks to jedno; systemy końcowe, firewalle, urządzenia pośrednie i inni operatorzy muszą jeszcze rozumieć jego znaczenie. Samo \`allow\` nie daje interoperacyjności z całą siecią ani nie zmienia zasad filtrowania pakietów na SRX.

### Historia z nocnej zmiany: „odrzucenie BGP to chyba policy import?”

Jeśli R1 nie instaluje nietypowego prefiksu od BGP sąsiada, kolega może przez godzinę szukać \`reject\` w \`policy-options\`. Tymczasem przyczyną jest reguła martian. Zamiast losowo zmieniać import policy, sprawdź kwalifikację prefiksu, domyślne wpisy martian, a dopiero potem zachowanie protokołu. Gdyby route okazała się ukryta, zobacz jej **hidden reason** w odpowiednim widoku — nie każda hidden route jest odrzucona z tej samej przyczyny.

**R1-MARTIAN-LAB — Operational Mode, inspekcja informacji o trasach**

~~~junos
show route martians
show route hidden
show route protocol bgp
~~~

Jeżeli nie masz sesji BGP, ostatnie polecenie oczywiście nie będzie dowodem awarii sąsiedztwa. W głównym labie z rozdziału 7 celowo **nie konstruujemy BGP** tylko po to, żeby „coś pokazać”: będziemy czytać martians, a protokołowy scenariusz odrzucenia trasy wróci w rozdziale 8 i 11.

Tym sposobem masz już trzy różne odpowiedzi na trzy problemy. **Aggregate** może odrzucić pakiet do nieobsługiwanego fragmentu sumarycznego prefiksu. **Martian** może zakwestionować samą informację routingu. **Firewall filter/security policy** może odrzucić konkretny pakiet lub sesję. Podobny efekt widziany przez użytkownika („nie działa”) nie oznacza wspólnej przyczyny.

*Źródła: [Recognize Martian Addresses for Routing](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/recognize-martian-addr-routing.html), [show route martians](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-martians.html), [martians CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/martians-edit-routing-options.html).*

---

## 7.3. Routing instances — skąd router wie, że dwa identyczne pakiety mają pojechać inną drogą? [S]

### Z jedną tablicą wszystko jest proste, dopóki firma nie chce dwóch zasad

Wracamy do naszego biura. Dwie firmy operatorskie, dwa wyjścia. Kierownik IT zgłasza wymaganie: „Komputery zwykłych pracowników mają korzystać z WAN-A, ale wybrane stanowisko laboratoryjne PC-A powinno zawsze wychodzić przez WAN-B, bo tam jest dodatkowa kontrola lub prywatna usługa”. Cel nadal jest identyczny: **203.0.113.10**. Chodzi o różnicę w drodze, nie w adresie docelowym.

W zwykłym routingu R1 sprawdzi destination IP w \`inet.0\`. Jeśli dla 203.0.113.10 jedyną najlepszą drogą jest default przez WAN-A, oba pakiety pójdą przez A. Wpisanie drugiej default route z gorszą preference nie rozwiąże wymagania biznesowego: nadal wygrywa jedna droga dla danego celu. Wpisanie dwóch equal-cost defaults również nie zapewni, że **właśnie PC-A** będzie korzystał z B, a PC-B z A.

Potrzebujemy czegoś innego: możliwości wyboru **kontekstu routingu** na podstawie tego, jaki pakiet przyszedł. W Junos każdy kontekst może mieć własną tablicę tras — to świat **routing instances**.

Możesz potraktować je jak dwie niezależne mapy. Na mapie A droga do usługi prowadzi przez WAN-A. Na mapie B — przez WAN-B. Problem nie polega na tym, że jedno z miast jest położone gdzie indziej. Problem polega na tym, że na dwóch mapach **inaczej wyznaczamy drogę**. Router może trzymać obie mapy jednocześnie.

### Czy routing instance to zawsze VRF?

Jeśli znasz Cisco VRF-Lite, pomysł osobnej tablicy jest naturalny. Nie przenoś jednak mechanicznie słowa „VRF” na każdy typ routing instance w Junos. Te konstrukcje mają różne role:

**\`virtual-router\`** jest bliższy klasycznemu VRF-Lite: tworzy oddzielny kontekst z interfejsami i możliwością konfiguracji routingu. W biurowym greenfield mógłby służyć do separacji OT, gości albo wydzielonej sieci klienta — jeśli platforma i projekt security na to pozwalają.

**\`vrf\`** odnosi się w Junos do bardziej wyspecjalizowanego scenariusza, m.in. L3VPN/PE, gdzie pojawiają się RD, RT i mechanizmy import/export tras VPN. Nie jest to po prostu bardziej elegancka nazwa \`virtual-router\`.

**\`forwarding\`** jest szczególnie przydatny przy **Filter-Based Forwarding (FBF)**. Chcemy mieć osobną tablicę do lookup wybranych pakietów, ale nie musimy przenosić całego interfejsu OFFICE do nowego VRF. Wejściowy filtr sam skieruje dopasowane pakiety do tej tablicy.

**\`no-forwarding\`** może przechowywać oddzielny kontekst informacji routingu, ale nie jest skrótem do „zrób jeszcze jedną forwarding table dla PBR”. To inny cel.

Uproszczenie „routing instance = VRF” pomaga rozpocząć naukę, ale przestaje działać, gdy próbujesz skonfigurować FBF. W naszym przypadku potrzebujemy **instance-type forwarding**.

### Najpierw tworzymy alternatywną mapę, jeszcze bez wysyłania do niej pakietów

Zbudujmy instancję **FBF-ALT**. Jej default route ma wskazywać WAN-B, czyli next-hop 198.51.100.6. W tym samym czasie \`inet.0\` nadal ma default przez WAN-A.

**R1-vSRX — Configuration Mode [edit], przygotowanie odrębnej tablicy dla ruchu specjalnego**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 0.0.0.0/0 next-hop 198.51.100.6
~~~

Nie podpinamy jeszcze filtra do interfejsu. Nie zmieniamy domyślnej tablicy. Tworzymy tylko alternatywną strukturę, którą później wykorzysta FBF. Po świadomym commit i na wspieranej platformie będziemy mogli porównać obie tablice.

**R1-vSRX — Operational Mode, odczyt dwóch różnych kontekstów**

~~~junos
show route table inet.0
show route table FBF-ALT.inet.0
show configuration routing-instances FBF-ALT | display set
~~~

Pytanie, które na tym etapie powinno się pojawić, brzmi: **skąd FBF-ALT ma wiedzieć, czym jest 198.51.100.6?** Jej default wskazuje ten next-hop, ale fizyczna sieć 198.51.100.4/30 należy do interfejsu WAN-B skonfigurowanego w podstawowym kontekście. Samo zapisanie adresu next-hop w dodatkowej tabeli nie daje automatycznie wiedzy o jego osiągalności.

### Słynna pułapka: „trasa jest w tabeli, ale router nie wie, jak dojść do next-hopa”

Wyobraź sobie nawigację samochodową, która mówi: „Jedź do wjazdu operatora B”. Ale nie zawiera mapy osiedla, w którym znajduje się ten wjazd. Numer adresu jest podany, jednak nie ma sposobu dotarcia do niego.

W routingu taka sytuacja to problem **next-hop resolution**. Dla FBF-ALT możemy mieć zapis \`0.0.0.0/0 via 198.51.100.6\`, ale bez odpowiedniej connected route do 198.51.100.4/30 alternatywna tablica może nie umieć rozwiązać tego nexthopa. Obecność konfiguracji statycznej nie dowodzi, że forwarding będzie możliwy.

~~~mermaid
flowchart LR
  IN["R1: zwykły pakiet OFFICE"] --> M["inet.0<br/>0/0 via 198.51.100.2"]
  SPEC["R1: specjalny pakiet OFFICE"] --> B["FBF-ALT.inet.0<br/>0/0 via 198.51.100.6"]
  M --> A["connected 198.51.100.0/30<br/>WAN-A"]
  B --> Q{"Czy ta tabela zna<br/>198.51.100.4/30?"}
  Q -->|tak| OK["WAN-B"]
  Q -->|nie| FAIL["Unresolved next-hop"]
~~~

*Rysunek 7.4 (autorski). Tekst alternatywny: globalna tablica umie rozwiązać next-hop A, a alternatywna FBF-ALT potrzebuje znajomości sieci connected do B; sam wpis default w drugiej tablicy nie wystarczy.*

To jest prawdziwa przyczyna, dla której w typowym scenariuszu FBF potrzebujemy jeszcze jednego mechanizmu. Nie jest nim kolejna static default i nie jest nim firewall filter. Potrzebujemy możliwości **udostępnienia wybranych informacji routingu między tablicami**. Do tego służą RIB groups w następnym kroku.

### Czy można po prostu przypisać WAN-B do FBF-ALT?

W niektórych projektach dałoby się rozważyć przeniesienie interfejsu do innej instancji routingu, ale to nie jest to samo zadanie. Przy klasycznym FBF chcemy zachować fizyczne interfejsy w głównym kontekście, a **dla wybranych pakietów** użyć dodatkowej tabeli. Gdybyśmy przenieśli WAN-B do oddzielnego virtual-router, zmienilibyśmy sposób, w jaki całe urządzenie traktuje ten interfejs, oraz jego dostępność dla innych usług.

Pamiętaj również o SRX: routing instance nie jest tym samym co **security zone**. Routing wybiera trasę; strefy i security policies decydują o dopuszczeniu ruchu, a NAT może przekształcić adresację. PBR/FBF na firewallu stateful wymaga weryfikacji współpracy z flow i NAT. Na klasycznym MX albo EX z reguły nie masz identycznego modelu sesji jak na vSRX. To ma ogromne znaczenie przy porównywaniu labu z konfiguracją produkcyjną.

*Źródła: [Routing Instances Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html), [Filter-Based Forwarding Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html), [Juniper — Example Configuring FBF](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html).*

---

## 7.4. RIB groups — jak dać drugiej tablicy mapę do bramy, ale nie cały firmowy routing [S]

### Dwie tablice, jedna informacja o bezpośrednim połączeniu

Wróćmy do problemu poprzedniej lekcji. \`inet.0\` zna WAN-B jako połączenie bezpośrednie: **198.51.100.4/30 is directly connected**. Tablica \`FBF-ALT.inet.0\` chce wykorzystać next-hop 198.51.100.6, ale nie ma tej samej wiedzy o interfejsie.

Nie musimy kopiować całej \`inet.0\`. Nie potrzebujemy przenosić wszystkich BGP, OSPF i static routes ani mieszać ze sobą wszystkich tenantów. Chcemy udostępnić **informacje pochodzące z interfejsów**. Właśnie tu naturalnie pojawia się **RIB group**.

RIB group to mechanizm pozwalający umieszczać wybrane informacje routingu w więcej niż jednej tabeli. Pierwsza tabela w \`import-rib\` jest **primary**, kolejne to **secondary**. Nie jest to niezależny, jednorazowy eksport wszystkich wpisów z pierwszej tablicy; grupa musi być **podpięta do konkretnego źródła tras**, a kopie w tabelach wtórnych są powiązane z życiem źródłowych wpisów.

### Najpierw definiujemy grupę, potem pokazujemy jej źródło

Nasza grupa będzie nazywała się \`COPY-INTERFACE-ROUTES\`. Pierwszym wpisem umieszczamy \`inet.0\`, bo to podstawowa tabela dla interfejsów R1. Drugim — \`FBF-ALT.inet.0\`, która potrzebuje tych tras do rozwiązywania next hopów.

**R1-vSRX — Configuration Mode [edit], grupa tablic IPv4**

~~~junos
set routing-options rib-groups COPY-INTERFACE-ROUTES import-rib [ inet.0 FBF-ALT.inet.0 ]
~~~

Na tym etapie jeszcze niczego nie udowodniliśmy. Stworzyliśmy definicję, ale Junos nie zgadnie, których tras ma ona dotyczyć. To dokładnie ten sam błąd logiczny co przy nazwanej routing policy, której nigdzie nie przypięto.

Dopiero teraz wskazujemy, że grupa ma być używana dla **IPv4 interface routes**.

**R1-vSRX — Configuration Mode [edit], aktywacja RIB group dla tras z interfejsów**

~~~junos
set routing-options interface-routes rib-group inet COPY-INTERFACE-ROUTES
~~~

W tym momencie domyślne interfejsowe prefiksy IPv4 mogą być udostępniane do secondary RIB. To potrzebny element następnego ćwiczenia FBF, ponieważ chcemy, by FBF-ALT potrafiła rozwiązać 198.51.100.6 przez WAN-B.

**R1-vSRX — Operational Mode, czy connected jest rzeczywiście w dodatkowej tabeli?**

~~~junos
show configuration routing-options rib-groups | display set
show configuration routing-options interface-routes | display set
show route table inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
~~~

Dwa ostatnie polecenia są kluczowe: pierwsze sprawdza „czy FBF-ALT zna drogę do next-hopa?”, drugie „czy default przez 198.51.100.6 może być rozwiązana?”. Nie uznawaj testu za wykonany tylko dlatego, że \`show configuration rib-groups\` ładnie wygląda.

### Co, jeśli skopiowaliśmy za dużo?

W naszej małej topologii zwykłe \`interface-routes rib-group\` może wystarczyć. Ale w dużej sieci firma ma również interfejsy management, uplinki do różnych security zones i segmenty, których nie chcesz widzieć w danej tabeli. Tu pojawia się drugi powód użycia RIB group: możliwość **ograniczenia, które trasy trafią do secondary** przez odpowiednią import policy.

Powiedzmy, że FBF-ALT potrzebuje wyłącznie connected do WAN-B. Nie potrzebuje informacji o OFFICE, WAN-A i OOB. Chcielibyśmy zatem dopuścić do kopii tylko **198.51.100.4/30**.

**R1-vSRX — Configuration Mode [edit], wariant z ograniczeniem importu do secondary RIB**

~~~junos
set policy-options policy-statement RIB-ONLY-WAN-B term MATCH-WAN-B from route-filter 198.51.100.4/30 exact
set policy-options policy-statement RIB-ONLY-WAN-B term MATCH-WAN-B then accept
set policy-options policy-statement RIB-ONLY-WAN-B term REST then reject
set routing-options rib-groups COPY-INTERFACE-ROUTES import-policy RIB-ONLY-WAN-B
~~~

Zwróć uwagę na nazwę **import-policy** w kontekście RIB group. To nie jest BGP import policy na sąsiedzie. Tu decydujemy, które wpisy mogą zostać skopiowane do tabel wtórnych. Pierwsza tabela pozostaje podstawowym kontekstem źródła; exact semantics sprawdzaj dla własnego release i typu źródła, a po zmianie koniecznie porównaj obie RIB. **Nie przyjmuj na wiarę, że „reject” usunął route z globalnej inet.0**.

W ogóle nie musisz używać tej dodatkowej policy, aby zrozumieć podstawowy FBF lab. Jest dobrym rozwinięciem dla urządzeń z wieloma interfejsami i wymagającą separacją.

### Dlaczego „kopię” w secondary można stracić, kiedy źródło znika?

Wyobraź sobie, że kabel WAN-B zostaje odłączony. Trasa connected do 198.51.100.4/30 przestaje być aktywna w primary. Czy FBF-ALT może sobie powiedzieć: „Nie szkodzi, mam dawną kopię, będę forwardować dalej”? Nie byłoby to rozsądne. **RIB group nie tworzy niezależnej, nieśmiertelnej trasy**. Dokumentacja producenta opisuje, że usunięcie podstawowej trasy powoduje też usunięcie wtórnych wpisów zależnych od niej. Zniknie więc możliwość rozwiązania next-hopa WAN-B, a default w FBF-ALT może stać się nieużyteczna.

To łączy RIB group z normalnym życiem routingu. Nie rozwiązujemy awarii przez pamiętanie starych informacji. Rozwiązujemy ją przez korzystanie z aktualnych tras, nawet w wielu kontekstach.

**R1-vSRX — Operational Mode, gdy WAN-B znika**

~~~junos
show interfaces terse
show route 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
show route hidden
~~~

Przy tym zestawie nie wystarczy informacja „WAN-B is up” — potrzebujesz wiedzieć, czy właściwy logiczny unit ma adres i czy connected jest kwalifikowane w obydwu tabelach. Jeżeli static default istnieje w konfiguracji FBF-ALT, ale nie jest aktywna, problem może być w next-hop resolution, a nie w samym filtrze FBF.

### RIB group to nie „route leaking wszędzie”

To ważna granica analogii z Cisco VRF route leaking. RIB groups są jednym ze sposobów udostępniania informacji o trasach pomiędzy tablicami, ale w zależności od architektury mogą istnieć inne, bardziej odpowiednie rozwiązania: \`instance-import\`, BGP VPN route targets czy osobne protokoły pomiędzy instancjami. Nie mieszaj ich przypadkowo. W szczególności udostępnienie connected interfejsu do FBF-ALT **nie oznacza**, że pełna tablica BGP została skopiowana do nowego kontekstu albo że użytkownicy z dwóch security tenants mogą od tej chwili swobodnie wymieniać pakiety.

Jest też platformowy haczyk. Na części EX/QFX zachowanie RIB groups, typów instancji i kopiowania interfejsów może podlegać ograniczeniom lub innym regułom instalacji niż na MX. Dla produkcyjnego projektu sprawdź konkretne wspierane modele, Junos release oraz Feature Explorer. Ten sam blok konfiguracji nie jest uniwersalną gwarancją działania na dowolnym EX, MX i vSRX.

### Jak wygląda ta zależność na jednej mapie?

~~~mermaid
flowchart TD
  I["Interface route WAN-B<br/>198.51.100.4/30"] --> P["Primary RIB: inet.0"]
  I --> RG["RIB group:<br/>COPY-INTERFACE-ROUTES"]
  RG --> S["Secondary: FBF-ALT.inet.0"]
  S --> NH["Rozwiązanie default<br/>via 198.51.100.6"]
  NH --> UP["WAN-B: prawidłowy next-hop"]
  I -. "awaria interfejsu / brak route" .-> F["Trasa źródłowa znika"]
  F -.-> S
~~~

*Rysunek 7.5 (autorski). Tekst alternatywny: jedna trasa interface WAN-B trafia do primary inet.0 i przez RIB group również do FBF-ALT.inet.0; ta kopia umożliwia rozwiązanie default przez 198.51.100.6. Usunięcie źródła może usunąć także wpis wtórny.*

W tym momencie drugi dyspozytor w naszej analogii ma już mapę do bramy operatora B. Ale nadal żaden klient nie został mu przypisany. **Routing instances i RIB groups przygotowują alternatywną drogę; dopiero FBF zdecyduje, który pakiet ją wykorzysta.**

*Źródła: [RIB Groups CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html), [import-rib CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html), [interface-routes CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/interface-routes-edit-routing-options.html), [Przykład FBF z RIB group](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html).*
