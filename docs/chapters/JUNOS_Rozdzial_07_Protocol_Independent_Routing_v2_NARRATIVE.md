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
