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

Przez cały rozdział będziemy wracać do **tej samej referencyjnej sieci**, która pojawiła się już w poprzednich rozdziałach. PC-A jest w LAN-A za R1; PC-B w LAN-B za R4. R1 i R4 mają dwie możliwości tranzytu: górną przez R2 i dolną przez R3. Zamiast wymyślać dwóch operatorów Linux i inny plan IP, użyjemy tych czterech routerów do nauki aggregate, generated, dodatkowych tabel RIB, FBF i ECMP.

~~~mermaid
flowchart LR
  PCA["PC-A / VPC1<br/>VPCS<br/>Gateway: 192.0.2.1"]
  R1["R1-vSRX<br/>lo0.0: 10.255.255.1/32"]
  R2["R2-vSRX<br/>lo0.0: 10.255.255.2/32"]
  R3["R3-vSRX<br/>lo0.0: 10.255.255.3/32"]
  R4["R4-vSRX<br/>lo0.0: 10.255.255.4/32"]
  PCB["PC-B / VPC2<br/>VPCS<br/>Gateway: 198.51.100.1"]

  PCA ---|"LAN-A · 192.0.2.0/24<br/>PC-A eth0: 192.0.2.10/24<br/>R1 ge-0/0/2.0: 192.0.2.1/24"| R1
  R1 ---|"P2P-12 · 10.12.0.0/30<br/>R1 ge-0/0/0.0: 10.12.0.1/30<br/>R2 ge-0/0/0.0: 10.12.0.2/30"| R2
  R1 ---|"P2P-13 · 10.13.0.0/30<br/>R1 ge-0/0/1.0: 10.13.0.1/30<br/>R3 ge-0/0/0.0: 10.13.0.2/30"| R3
  R2 ---|"P2P-24 · 10.24.0.0/30<br/>R2 ge-0/0/1.0: 10.24.0.1/30<br/>R4 ge-0/0/0.0: 10.24.0.2/30"| R4
  R3 ---|"P2P-34 · 10.34.0.0/30<br/>R3 ge-0/0/1.0: 10.34.0.1/30<br/>R4 ge-0/0/1.0: 10.34.0.2/30"| R4
  R4 ---|"LAN-B · 198.51.100.0/24<br/>R4 ge-0/0/2.0: 198.51.100.1/24<br/>PC-B eth0: 198.51.100.10/24"| PCB

  classDef vsrx fill:#e6f1fc,stroke:#1565a6,stroke-width:2px,color:#13344f;
  classDef vpcs fill:#e8f5e9,stroke:#2f7d4a,stroke-width:1.5px,color:#17422a;
  class R1,R2,R3,R4 vsrx;
  class PCA,PCB vpcs;
~~~

*Rysunek 7.1. Kanoniczna topologia DIAMOND. Każda etykieta zawiera adresy obu końców, a każdy vSRX ma własny loopback /32. [Źródłowa wersja Mermaid](../../labs/reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md).*

To **sześć węzłów i sześć segmentów**, bez połączenia R2–R3. Dzięki temu eksperymenty w rozdziale 7 nie zmienią adresów i portów poznanych w rozdziałach 2, 5 i 6. Część dodatkowych prefiksów (np. `10.60.0.0/16` i `10.70.0.0/16`) będziemy traktować jako *syntetyczne wpisy routingu* służące do pokazania podsumowań, a **nie nowe sieci dołączone do diamentu**. Do ich pingowania trzeba by jawnie zbudować odrębny wariant z dodatkowymi usługami — same trasy nie tworzą hostów.

Nie utożsamiaj vSRX z EX/QFX/MX: wsparcie klasycznego FBF `then routing-instance` i oddziaływanie na stateful flow zależy od obrazu/wersji. Jeśli funkcja nie jest wspierana, nie przedstawiaj przykładowej konfiguracji jako działającej — wybierz sprzęt/platformę wspieraną przez producenta albo opracuj świadomy wariant APBR dla SRX.

---

## 7.1. Static, aggregate i generated — trzy sposoby powiedzenia routerowi, dokąd prowadzi droga [S]

### Najpierw zwykła static: nie trzeba odkrywać sieci, wystarczy znać następnego hopa

Wyobraź sobie, że PC-A wysyła pakiet do 10.60.10.10 — **hipotetycznej usługi testowej** reprezentowanej przez route contributor za R2. Ta sieć NIE jest dodatkową fizyczną podsiecią kanonicznego diamentu. R1 ma bezpośredni uplink P2P-12 do R2 o adresie 10.12.0.2. Jeśli administrator doda statyczną trasę, router nie musi negocjować żadnego BGP ani OSPF. Wystarczy, że potrafi rozwiązać next-hop i zainstalować poprawny forwarding.

**R1-vSRX — Configuration Mode [edit], pierwszy przykład statycznej drogi**

~~~junos
set routing-options static route 10.60.10.0/24 next-hop 10.12.0.2
~~~

Ta linijka nie jest „pakietem” ani reklamą do sąsiada. Jest intencją administratora zapisaną w konfiguracji. Jeśli 10.12.0.2 jest osiągalny jako next-hop, R1 może użyć trasy. Jeśli interfejs P2P-12 / R2 zostanie odłączony, statyczna trasa może przestać być aktywna; nie pomaga jej to, że konfiguracja nadal zawiera tekst \`set routing-options static route\`.

To dobry moment na porównanie z Cisco: \`ip route 10.60.10.0 255.255.255.0 10.12.0.2\` rozwiązuje podobny problem. Ale na Junos osobno obserwujemy status w RIB, kwalifikację next-hop i stan FIB. **Konfiguracja, RIB i rzeczywiste forwarding nie są trzema nazwami na to samo.**

**R1-vSRX — Operational Mode, skąd wiemy, że trasa działa**

~~~junos
show route 10.60.10.0/24 exact extensive
show route forwarding-table destination 10.60.10.10
show arp no-resolve
~~~

Jeśli trasa ma aktywny next-hop przez R2, forwarding do 10.60.10.10 może być na niego skierowany. Ale samo to nie dowodzi, że serwer istnieje ani że SRX przepuści daną sesję. Ping z PC-A do usługi — **wyłącznie jeśli dodamy ją jako osobny, jawnie oznaczony wariant usługi na R2** — jest osobnym dowodem. Bez takiej usługi test jest wyłącznie rozumowaniem o RIB/FIB.

### Drugi telefon: „Nie chcemy reklamować setek /24. Nie da się ich zamknąć w jednym /16?”

Sieć się rozrasta. W tablicy R1 wprowadzimy **symulowane trasy contributor** `10.60.10.0/24` via R2 i `10.60.20.0/24` via R3; te prefiksy nie są fizycznymi LAN-ami, dopóki nie zbudujemy oddzielnego wariantu. W centrali ktoś chce reklamować partnerowi jeden czytelny prefiks 10.60.0.0/16. Po co? Żeby partner nie musiał znać każdego szczegółu firmowej adresacji i nie dostawał kolejnej zmiany BGP, gdy pojedynczy dział przeprowadza się do innego budynku.

To jest moment, w którym pojawia się **aggregate route**. Nie jest zwykłą statyczną trasą do konkretnego operatora. Jest podsumowaniem tego, co R1 zna *pod spodem*.

Najpierw w RIB muszą istnieć co najmniej jedna **aktywna, bardziej szczegółowa trasa** w zakresie agregatu. Te trasy nazywamy *contributors*. Jeśli mamy 10.60.10.0/24 oraz 10.60.20.0/24, obie mogą przyczyniać się do aktywności 10.60.0.0/16.

**R1-vSRX — Configuration Mode [edit], dokładamy drugą drogę i agregat**

~~~junos
set routing-options static route 10.60.20.0/24 next-hop 10.13.0.2
set routing-options aggregate route 10.60.0.0/16
~~~

Od razu zapytajmy o pakiet do 10.60.10.10. Czy po dodaniu /16 zacznie iść jakąś „drogą agregatową”? Nie. W RIB nadal jest /24 i LPM wybiera **bardziej szczegółową trasę**. Pakiet dotrze do R2 tak samo jak wcześniej, jeśli ścieżka działa.

A co z 10.60.99.25? To adres, który mieści się w podsumowaniu /16, ale nie jest objęty żadną z naszych prawdziwych podsieci /24. Tu zaczyna działać głęboki sens agregatu: **domyślny next-hop aggregate to reject**. R1 nie przekazuje pakietu na ślepo do przypadkowego upstreama. Zwraca brak osiągalności w sposób zależny od platformy i warunków filtrowania.

~~~mermaid
flowchart TD
  P["Pakiet do sieci 10.60.x.x"] --> L{"Najbardziej szczegółowa trasa?"}
  L -->|10.60.10.10| A["10.60.10.0/24 → UP-A"]
  L -->|10.60.20.10| B["10.60.20.0/24 → R3"]
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

Załóżmy, że łącze R1–R2 przestaje działać i trasa 10.60.10.0/24 znika, ale 10.60.20.0/24 przez B nadal jest aktywna. Agregat /16 może **pozostać aktywny**. To może zaskoczyć: przecież połowa usług w firmie nie jest osiągalna.

Z punktu widzenia Junos wystarczy contributor spełniający warunki aktywności agregatu. Z punktu widzenia partnera nadal może istnieć reklama całego /16, jeśli dodatkowo skonfigurowaliśmy odpowiednią **routing export policy**. Wtedy ruch do niedostępnych części zakresu zostanie przyciągnięty do R1 i odrzucony przez aggregate reject, jeśli brak bardziej szczegółowej trasy.

Nie znaczy to, że agregacja jest zła. Znaczy to, że podsumowanie **świadomie ukrywa szczegóły**. Jeśli umowa SLA wymaga przerwania reklamy summary dopiero po utracie określonego zestawu usług, musimy zaprojektować policy contributorów i warunek dostępności znacznie staranniej niż „jedna /24 wystarczy”.

Także oszczędność tablic działa nie tam, gdzie wielu początkujących szuka: agregacja sama w sobie **nie usuwa contributorów z lokalnej RIB**. R1 nadal potrzebuje szczegółowych /24. Rozmiar tabel u *sąsiadów* może się zmniejszyć, gdy świadomie reklamujemy /16 zamiast kilku /24. Samo \`set aggregate\` jeszcze niczego nie reklamuje. To temat export policy z rozdziału 8.

### Trzecia konstrukcja: generated route. Podobna nazwa, zupełnie inna odpowiedź na pakiet

Teraz potrzebujemy czegoś innego. Chcemy, aby R1 tworzył trasę 10.70.0.0/16 **tylko wtedy**, gdy istnieje aktywna droga do 10.70.10.0/24. Ale zamiast zatrzymywać pakiet do innej części /16, chcemy wysłać go przez następny hop tej istotnej /24. Wtedy rozważamy **generated route**.

**R1-vSRX — Configuration Mode [edit], oddzielny przykład generated**

~~~junos
set routing-options static route 10.70.10.0/24 next-hop 10.12.0.2
set routing-options generate route 10.70.0.0/16
~~~

Tutaj również istnieje contributor. Ale **generated route może przejąć next-hop od primary contributing route**. W naszej prostej sieci będzie nim droga do 10.70.10.0/24 przez 10.12.0.2. Zatem pakiet do 10.70.99.25, choć nie ma własnej /24, może zostać wysłany do UP-A. Tam upstream może go odrzucić. Sam ping nie wystarczy, żeby powiedzieć, czy R1 wysłał pakiet: trzeba zobaczyć forwarding table albo ruch na interfejsie UP-A.

~~~mermaid
flowchart LR
  C["Aktywny contributor<br/>10.70.10.0/24 via UP-A"] --> G["Generated<br/>10.70.0.0/16"]
  G --> N["Next-hop z primary contributor<br/>10.12.0.2"]
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

Wracamy do naszego biura. Dwie firmy operatorskie, dwa tranzyty R2 oraz R3. Kierownik IT zgłasza wymaganie: „Komputery zwykłych pracowników mają korzystać z WAN-A, ale wybrane stanowisko laboratoryjne PC-A powinno zawsze wychodzić przez P2P-13 / R3, bo tam jest dodatkowa kontrola lub prywatna usługa”. Cel nadal jest identyczny: **198.51.100.10**. Chodzi o różnicę w drodze, nie w adresie docelowym.

W zwykłym routingu R1 sprawdzi destination IP w \`inet.0\`. Jeśli dla 198.51.100.10 jedyną najlepszą drogą jest default przez WAN-A, oba pakiety pójdą przez A. Wpisanie drugiej default route z gorszą preference nie rozwiąże wymagania biznesowego: nadal wygrywa jedna droga dla danego celu. Wpisanie dwóch equal-cost defaults również nie zapewni, że **właśnie PC-A** będzie korzystał z B, a PC-B z A.

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

Zbudujmy instancję **FBF-ALT**. Jej default route ma wskazywać WAN-B, czyli next-hop 10.13.0.2. W tym samym czasie \`inet.0\` nadal ma default przez WAN-A.

**R1-vSRX — Configuration Mode [edit], przygotowanie odrębnej tablicy dla ruchu specjalnego**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 0.0.0.0/0 next-hop 10.13.0.2
~~~

Nie podpinamy jeszcze filtra do interfejsu. Nie zmieniamy domyślnej tablicy. Tworzymy tylko alternatywną strukturę, którą później wykorzysta FBF. Po świadomym commit i na wspieranej platformie będziemy mogli porównać obie tablice.

**R1-vSRX — Operational Mode, odczyt dwóch różnych kontekstów**

~~~junos
show route table inet.0
show route table FBF-ALT.inet.0
show configuration routing-instances FBF-ALT | display set
~~~

Pytanie, które na tym etapie powinno się pojawić, brzmi: **skąd FBF-ALT ma wiedzieć, czym jest 10.13.0.2?** Jej default wskazuje ten next-hop, ale fizyczna sieć 10.13.0.0/30 należy do interfejsu WAN-B skonfigurowanego w podstawowym kontekście. Samo zapisanie adresu next-hop w dodatkowej tabeli nie daje automatycznie wiedzy o jego osiągalności.

### Słynna pułapka: „trasa jest w tabeli, ale router nie wie, jak dojść do next-hopa”

Wyobraź sobie nawigację samochodową, która mówi: „Jedź do wjazdu operatora B”. Ale nie zawiera mapy osiedla, w którym znajduje się ten wjazd. Numer adresu jest podany, jednak nie ma sposobu dotarcia do niego.

W routingu taka sytuacja to problem **next-hop resolution**. Dla FBF-ALT możemy mieć zapis \`0.0.0.0/0 via 10.13.0.2\`, ale bez odpowiedniej connected route do 10.13.0.0/30 alternatywna tablica może nie umieć rozwiązać tego nexthopa. Obecność konfiguracji statycznej nie dowodzi, że forwarding będzie możliwy.

~~~mermaid
flowchart LR
  IN["R1: zwykły pakiet OFFICE"] --> M["inet.0<br/>0/0 via 10.12.0.2"]
  SPEC["R1: specjalny pakiet OFFICE"] --> B["FBF-ALT.inet.0<br/>0/0 via 10.13.0.2"]
  M --> A["connected 10.12.0.0/30<br/>WAN-A"]
  B --> Q{"Czy ta tabela zna<br/>10.13.0.0/30?"}
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

Wróćmy do problemu poprzedniej lekcji. \`inet.0\` zna WAN-B jako połączenie bezpośrednie: **10.13.0.0/30 is directly connected**. Tablica \`FBF-ALT.inet.0\` chce wykorzystać next-hop 10.13.0.2, ale nie ma tej samej wiedzy o interfejsie.

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

W tym momencie domyślne interfejsowe prefiksy IPv4 mogą być udostępniane do secondary RIB. To potrzebny element następnego ćwiczenia FBF, ponieważ chcemy, by FBF-ALT potrafiła rozwiązać 10.13.0.2 przez WAN-B.

**R1-vSRX — Operational Mode, czy connected jest rzeczywiście w dodatkowej tabeli?**

~~~junos
show configuration routing-options rib-groups | display set
show configuration routing-options interface-routes | display set
show route table inet.0 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
~~~

Dwa ostatnie polecenia są kluczowe: pierwsze sprawdza „czy FBF-ALT zna drogę do next-hopa?”, drugie „czy default przez 10.13.0.2 może być rozwiązana?”. Nie uznawaj testu za wykonany tylko dlatego, że \`show configuration rib-groups\` ładnie wygląda.

### Co, jeśli skopiowaliśmy za dużo?

W naszej małej topologii zwykłe \`interface-routes rib-group\` może wystarczyć. Ale w dużej sieci firma ma również interfejsy management, uplinki do różnych security zones i segmenty, których nie chcesz widzieć w danej tabeli. Tu pojawia się drugi powód użycia RIB group: możliwość **ograniczenia, które trasy trafią do secondary** przez odpowiednią import policy.

Powiedzmy, że FBF-ALT potrzebuje wyłącznie connected do WAN-B. Nie potrzebuje informacji o OFFICE, WAN-A i OOB. Chcielibyśmy zatem dopuścić do kopii tylko **10.13.0.0/30**.

**R1-vSRX — Configuration Mode [edit], wariant z ograniczeniem importu do secondary RIB**

~~~junos
set policy-options policy-statement RIB-ONLY-WAN-B term MATCH-WAN-B from route-filter 10.13.0.0/30 exact
set policy-options policy-statement RIB-ONLY-WAN-B term MATCH-WAN-B then accept
set policy-options policy-statement RIB-ONLY-WAN-B term REST then reject
set routing-options rib-groups COPY-INTERFACE-ROUTES import-policy RIB-ONLY-WAN-B
~~~

Zwróć uwagę na nazwę **import-policy** w kontekście RIB group. To nie jest BGP import policy na sąsiedzie. Tu decydujemy, które wpisy mogą zostać skopiowane do tabel wtórnych. Pierwsza tabela pozostaje podstawowym kontekstem źródła; exact semantics sprawdzaj dla własnego release i typu źródła, a po zmianie koniecznie porównaj obie RIB. **Nie przyjmuj na wiarę, że „reject” usunął route z globalnej inet.0**.

W ogóle nie musisz używać tej dodatkowej policy, aby zrozumieć podstawowy FBF lab. Jest dobrym rozwinięciem dla urządzeń z wieloma interfejsami i wymagającą separacją.

### Dlaczego „kopię” w secondary można stracić, kiedy źródło znika?

Wyobraź sobie, że kabel WAN-B zostaje odłączony. Trasa connected do 10.13.0.0/30 przestaje być aktywna w primary. Czy FBF-ALT może sobie powiedzieć: „Nie szkodzi, mam dawną kopię, będę forwardować dalej”? Nie byłoby to rozsądne. **RIB group nie tworzy niezależnej, nieśmiertelnej trasy**. Dokumentacja producenta opisuje, że usunięcie podstawowej trasy powoduje też usunięcie wtórnych wpisów zależnych od niej. Zniknie więc możliwość rozwiązania next-hopa WAN-B, a default w FBF-ALT może stać się nieużyteczna.

To łączy RIB group z normalnym życiem routingu. Nie rozwiązujemy awarii przez pamiętanie starych informacji. Rozwiązujemy ją przez korzystanie z aktualnych tras, nawet w wielu kontekstach.

**R1-vSRX — Operational Mode, gdy WAN-B znika**

~~~junos
show interfaces terse
show route 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 10.13.0.0/30 exact
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
  I["Interface route WAN-B<br/>10.13.0.0/30"] --> P["Primary RIB: inet.0"]
  I --> RG["RIB group:<br/>COPY-INTERFACE-ROUTES"]
  RG --> S["Secondary: FBF-ALT.inet.0"]
  S --> NH["Rozwiązanie default<br/>via 10.13.0.2"]
  NH --> UP["WAN-B: prawidłowy next-hop"]
  I -. "awaria interfejsu / brak route" .-> F["Trasa źródłowa znika"]
  F -.-> S
~~~

*Rysunek 7.5 (autorski). Tekst alternatywny: jedna trasa interface WAN-B trafia do primary inet.0 i przez RIB group również do FBF-ALT.inet.0; ta kopia umożliwia rozwiązanie default przez 10.13.0.2. Usunięcie źródła może usunąć także wpis wtórny.*

W tym momencie drugi dyspozytor w naszej analogii ma już mapę do bramy operatora B. Ale nadal żaden klient nie został mu przypisany. **Routing instances i RIB groups przygotowują alternatywną drogę; dopiero FBF zdecyduje, który pakiet ją wykorzysta.**

*Źródła: [RIB Groups CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html), [import-rib CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html), [interface-routes CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/interface-routes-edit-routing-options.html), [Przykład FBF z RIB group](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html).*

---


## 7.5. Load balancing — dlaczego dwa uplinki nie muszą dzielić ruchu po połowie? [S]

### Dwa dobre szlaki i tylko jeden używany

Mamy w DIAMOND dwa korytarze R1 → R4: górny przez R2, dolny przez R3. R1 zna docelowy LAN-B `198.51.100.0/24`; w zwykłym statycznym baseline preferuje `10.12.0.2`, a `10.13.0.2` jest kwalifikowanym backupem. Taki układ to **failover**, a nie ECMP.

Kierownik NOC pyta: „Skoro mamy dwa łącza, dlaczego graf pokazuje ruch głównie przez R2?” Bo router nie musi instalować obu dróg jako aktywne next-hopy dla tego samego prefiksu. Trzeba odróżnić kilka różnych decyzji: (1) czy oba sąsiednie next-hopy są osiągalne, (2) czy istnieje jeden najlepszy route, (3) czy FIB zawiera multipath oraz (4) jak platforma hashuje **różne flows**.

**R1-vSRX — Operational Mode, przed jakąkolwiek zmianą**

~~~junos
show route 198.51.100.0/24 exact extensive
show route forwarding-table destination 198.51.100.10
show interfaces terse
~~~

### RIB nie jest obietnicą dwóch ścieżek w forwarding

Aby przećwiczyć ECMP, pracuj na osobnej kopii DIAMOND i **świadomie usuń dotychczasowe rozróżnienie primary/qualified backup**, następnie skonfiguruj równorzędne next-hopy do tego samego celu. Nie mieszaj w tym samym momencie FBF z ECMP, bo nie będziesz wiedzieć, który mechanizm wybrał R3.

**R1-vSRX — Configuration Mode [edit private], wyłącznie wariant ECMP**

~~~junos
delete routing-options static route 198.51.100.0/24
set routing-options static route 198.51.100.0/24 next-hop 10.12.0.2
set routing-options static route 198.51.100.0/24 next-hop 10.13.0.2
set policy-options policy-statement CH07-ECMP term LANB from route-filter 198.51.100.0/24 exact
set policy-options policy-statement CH07-ECMP term LANB then load-balance per-packet
set policy-options policy-statement CH07-ECMP term LANB then accept
set policy-options policy-statement CH07-ECMP term REST then accept
set routing-options forwarding-table export CH07-ECMP
show | compare
commit check
~~~

Zwróć uwagę na nieintuicyjną nazwę `load-balance per-packet`: w licznych implementacjach Junos nie oznacza to ścisłego przełączania *każdego* kolejnego pakietu w round-robin. Możliwy jest podział **per-flow**, w zależności od platformy i konfiguracji. Jeden ping z PC-A nie jest miarodajnym testem równomiernego rozkładu.

**R1-vSRX — Operational Mode, po bezpiecznej warunkowej aktywacji**

~~~junos
show route 198.51.100.0/24 exact extensive
show route forwarding-table destination 198.51.100.10
show configuration routing-options forwarding-table | display set
~~~

Nie obiecuj, że same dwie trasy będą bezproblemowo działać na vSRX: stateful security i asymetryczny ruch zwrotny mogą zmienić wynik. Na R4 nadal musi istnieć poprawna droga do LAN-A, a flow-session nie może zostać potraktowana jako zastępstwo routingu. Jeśli chcesz **statystycznie** ocenić ECMP, wygeneruj wiele odrębnych strumieni w jawnie oznaczonym wariancie generatora ruchu, zamiast opierać się na jednym VPCS.

### Load balancing to nie LAG, nie NAT i nie mechanizm detekcji awarii

LAG agreguje porty L2, podczas gdy ECMP wybiera alternatywne ścieżki L3. NAT zmienia adresy, a nie tworzy automatycznie dwóch tras. Pojedynczy link down potrafi usunąć lokalny next-hop, ale nie zapewnia rozpoznania awarii odległego odcinka R2–R4, jeżeli R1–R2 wciąż działa. Ten sam fizyczny DIAMOND pozwala więc odtworzyć różne przyczyny pozornie podobnej „awarii redundantnego WAN”.

---

## 7.6. Filter-Based Forwarding — ten sam pakiet do PC-B, ale inna tablica routingu [S]

### Zaczynamy od pytania, którego nie rozwiąże zwykła trasa globalna

Mamy R1 i docelowy host PC-B `198.51.100.10`. Normalny `inet.0` wybiera **R2** next-hop `10.12.0.2`, ale dla wybranych testów chcemy przeprowadzić ruch przez **R3** next-hop `10.13.0.2`. Nie zmieniamy adresów hostów, nie przenosimy R4 i nie dokładamy drugiego hosta w LAN-A.

Klasyczne FBF działa na **ruchu wchodzącym przez interfejs**. To ważna różnica względem pakietu wygenerowanego przez R1 (router-originated): ten ostatni może nie przechodzić przez filtr ingress LAN-A. Do testu „termin specjalny vs pozostałe pakiety” wykorzystamy zatem **różne adresy docelowe** przy tym samym źródle PC-A. ICMP do PC-B ma trafić do alternatywnej tabeli, natomiast ICMP do interfejsu R2 `10.12.0.2` ma pozostać w zwykłej tabeli. Taka demonstracja pokazuje **wybór tablicy po destination**, nie klasyczne source-based PBR — przy zaledwie jednym PC-A w LAN-A nie należy udawać, że przetestowaliśmy dwóch niezależnych klientów.

### Trzy mechanizmy, które muszą być prawidłowe jednocześnie

**Pierwszy:** instancja `FBF-ALT` typu `forwarding`, z trasą do LAN-B przez R3.

**Drugi:** drugi RIB musi umieć rozwiązać next-hop `10.13.0.2`. Sam wpis next-hop nie przenosi automatycznie connected `10.13.0.0/30` z `inet.0` — wykorzystamy ostrożnie RIB group.

**Trzeci:** filtr wejściowy na R1 `ge-0/0/2.0` musi skierować **tylko** ruch do PC-B do alternatywnej tabeli, ale zostawić jawne `NORMAL then accept` dla innych pakietów. Ważne: konkretna wersja **vSRX może nie obsługiwać klasycznego FBF**. Sprawdź Feature Explorer i dokumentację obrazu zanim rozpoczniesz konfigurację.

**R1-vSRX — Configuration Mode [edit private], izolowany wariant FBF**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 198.51.100.0/24 next-hop 10.13.0.2
set routing-options rib-groups CH07-RIB import-rib [ inet.0 FBF-ALT.inet.0 ]
set routing-options interface-routes rib-group inet CH07-RIB
set firewall family inet filter OFFICE-FBF term SPECIAL from destination-address 198.51.100.10/32
set firewall family inet filter OFFICE-FBF term SPECIAL then routing-instance FBF-ALT
set firewall family inet filter OFFICE-FBF term NORMAL then accept
set interfaces ge-0/0/2 unit 0 family inet filter input OFFICE-FBF
show | compare
commit check
~~~

To schemat do weryfikacji, a nie gotowy wynik walidacji na vSRX. Uwaga szczególna: RIB group może kopiować również **niepotrzebne** connected; jeśli izolacja ma znaczenie, dołóż selektywny import-policy przepuszczający tylko `10.13.0.0/30`, a nie wszystkie trasy zarządzające. Nie przypisuj portu R1–R3 do dwóch różnych routing instances naraz.

### Ścieżka pakietu — od ingress do alternatywnej RIB

~~~mermaid
flowchart TD
  A["PC-A 192.0.2.10<br/>ingress R1 ge-0/0/2.0"] --> F{"Cel pakietu =<br/>198.51.100.10/32?"}
  F -->|"tak"| ALT["FBF-ALT.inet.0<br/>via R3: 10.13.0.2"]
  F -->|"nie"| MAIN["inet.0<br/>dla 10.12.0.2: direct via R2"]
  ALT --> R3["R3 → R4 → PC-B"]
  MAIN --> R2["R2-vSRX 10.12.0.2"]
~~~

A gdzie wraca odpowiedź? To kluczowe. R4 domyślnie wybiera górną trasę przez R2. Jeśli wymusisz **PC-A → R3 → R4** i powrót **R4 → R2 → R1**, otrzymujesz asymetrię, którą stanowe SRX mogą obsługiwać inaczej niż zwykłe routery. Dlatego w ćwiczeniu wymagającym pełnej sesji ICMP **kontrolowanie przełącz drogę zwrotną na R4 przez R3**, bez ingerencji w bazowy dokument referencyjny. Zmianę cofnij podczas cleanup. Nawet wtedy kontroluj stan sesji i zgodność FBF z danym obrazem.

### Jak odróżnisz prawdziwy dowód od błędnej interpretacji?

**R1-vSRX — Operational Mode, po aktywacji FBF, jeśli wspierane**

~~~junos
show configuration interfaces ge-0/0/2 | display set
show configuration firewall family inet filter OFFICE-FBF | display set
show route table FBF-ALT.inet.0 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 198.51.100.0/24 exact
show route forwarding-table destination 198.51.100.10
show security flow session
~~~

**PC-A — VPCS console**

~~~text
ping 10.12.0.2
ping 198.51.100.10
trace 198.51.100.10
~~~

Ping R1 → R3 nie potwierdza FBF, bo jest ruchem host-bound generowanym przez router. Ping PC-A → R2 przechodzący przez termin NORMAL pokazuje, że **nie wszystko zostało przekierowane**. Ping PC-A → PC-B pokazuje dopasowanie SPECIAL, ale sam nie dowodzi jeszcze, którym korytarzem pakiet popłynął — sprawdź FIB, liczniki filtra, sesje i ewentualny capture na wspieranej platformie.

### FBF a SRX: filtrowanie routingu nie jest firewall permit

`then routing-instance` zmienia kontekst lookup, **nie** gwarantuje dopuszczenia nowej sesji przez SRX security policy. Filtr stateless i security flow są różnymi mechanizmami. Jeśli w buildzie vSRX nie ma właściwego wsparcia FBF, nie przenoś tego 1:1 z EX/MX; opisz konkretny blocker i zrealizuj demonstrację na zgodnym obrazie, zachowując logiczną adresację diamentu w wariancie.

---

## 7.7. Troubleshooting — znajduje się zły pakiet, trasa czy tablica? [S]

### Incydent: po zmianie FBF PC-A widzi R2, ale nie widzi PC-B

Pierwsza myśl operatora: „R3 się zepsuł”. Ale to jedna z kilku możliwości. Zacznij od **rzeczywistego wejścia pakietu**. PC-A ma `192.0.2.10/24`, bramę R1 `192.0.2.1` i ingress w R1 `ge-0/0/2.0`. Czy filter `OFFICE-FBF` jest właśnie na tym logical unit? Czy SPECIAL pasuje do destination `198.51.100.10/32`, a NORMAL pozwala przejść do innych celów?

**R1-vSRX — Operational Mode, read-only**

~~~junos
show interfaces terse
show configuration interfaces ge-0/0/2 | display set
show configuration firewall family inet filter OFFICE-FBF | display set
show firewall filter OFFICE-FBF
show route table inet.0 198.51.100.0/24 exact
show route table FBF-ALT.inet.0 198.51.100.0/24 exact
show route table FBF-ALT.inet.0 10.13.0.0/30 exact
show security flow session
~~~

### Druga RIB widzi default, ale nie ma jak dotrzeć do next-hopa

Pamiętasz rozdział 7.4: `interface-routes rib-group` pozwala przepisać connected do additional RIB. Jeżeli RIB group pomija `10.13.0.0/30`, sam static route do LAN-B via `10.13.0.2` w `FBF-ALT` może być nierozwiązywalny. Diagnozą nie jest reset BGP ani wstawienie fake default przez management. Napraw source route i import-policy, a potem sprawdź dokładny next-hop.

### A jeśli routing wygląda dobrze, lecz pakiety nie wracają?

R1 wysłał PC-A przez R3, ale R4 nadal preferuje R2. Na R3 odpowiedź może nie pasować do istniejącej sesji, bo wraca inną drogą. Zamiast zgadywać, sprawdź obie RIB (R1 i R4), powiązane security zones/policies, stateful sessions i test nowego flow po kontrolowanej zmianie drogi powrotnej R4.

**R4-vSRX — Operational Mode, wyłącznie read-only**

~~~junos
show route 192.0.2.0/24 exact extensive
show route forwarding-table destination 192.0.2.10
show security flow session
~~~

### Trzy historie, które wyglądają identycznie z laptopa

**Pierwsza:** `aggregate 10.60.0.0/16` jest aktywny dzięki contributors /24, ale testowy destination `10.60.99.25` nie ma specyficznej trasy. R1 może odrzucać go przez aggregate reject. Nie rozwiązuj tego zmianą security policy, jeśli FIB mówi reject.

**Druga:** `FBF-ALT.inet.0` utraciła direct `10.13.0.0/30` po usunięciu konfiguracji RIB group. Next-hop `10.13.0.2` nie rozwiązuje się w secondary RIB. Restore RIB group / policy, a nie przypadkowo restart rpd.

**Trzecia:** R1 widzi dwa równorzędne next-hopy do LAN-B po aktywacji ECMP, ale jeden długi ICMP flow korzysta tylko z R2. To może być poprawny per-flow hash, nie awaria R3.

### Jak przeprowadzić zmianę, by nie zostawić po sobie P1?

Spisz przed zmianą: wersję obrazu vSRX, mapping portów, stan obu tras R1→R4, drogę zwrotną R4, ingress filter, additional RIB, security zones, stan sesji, aktualne `show system commit`. Następnie użyj `show | compare`, `commit check`, `commit confirmed` na kopii laba i sprawdź oba kierunki z PC-A i PC-B. Bez OOB/console nie wyłączaj przypadkowego interfejsu na produkcyjnym SRX.

**Przykładowa notatka do maintenance:** „In isolated JNCIP DIAMOND, R1 ingress ge-0/0/2.0 FBF route lookup via R3 was reviewed against FBF-ALT and R4 return path. Lab outcome pending runtime verification; no production change performed.”

## 7.8. Laboratorium — od jednej działającej trasy do FBF i ECMP [LAB]

Zrobimy to w taki sposób, jak prowadziłbym warsztat przy tablicy i konsoli. Najpierw zobaczysz sieć bez żadnej „magii”. Użytkownicy dotrą do testowej usługi po zwykłej default route. Potem dodamy aggregate i generated, a na końcu alternatywną tablicę oraz FBF. Będziemy wiedzieć, **co właśnie zmieniliśmy i co powinno zmienić się w ruchu**, zamiast wklejać 200 linii konfiguracji i potem zgadywać.

**Status laboratorium: PLANNED / NOT RUNTIME VALIDATED.** Ten scenariusz to projekt do uruchomienia, a nie raport z uruchomionej maszyny. Nie mamy tu wyniku \`show version\` konkretnego vSRX, mapowania NIC, zachowania security flow ani potwierdzenia wsparcia \`then routing-instance\` przez konkretny obraz. Przed wykonaniem zweryfikuj te elementy w EVE-NG i Juniper Feature Explorer. Jeśli klasyczny FBF jest niewspierany na danej wersji vSRX, przenieś tę fazę na wspierany EX/MX/QFX albo zaprojektuj osobny wariant APBR z właściwą dokumentacją SRX. **Nie deklaruj runtime PASS na podstawie samego commit albo wygenerowanego .unl.**

### Co musimy mieć w EVE-NG?

Nie potrzebujemy czterech ciężkich vSRX dla tego konkretnego mechanizmu. Wystarczy jeden router **LAB-R1-vSRX** i cztery lekkie maszyny Linux: dwóch klientów OFFICE oraz dwóch operatorów. R1 ma trzy porty danych: OFFICE, WAN-A, WAN-B. Segmenty mają być całkowicie izolowane. Nie łączymy cloud bridge, Internetu ani sieci zarządzania GCP z ruchem testowym.

~~~mermaid
flowchart LR
  subgraph OFF["OFFICE 192.0.2.0/24"]
    PCA["LAB-PC-A<br/>192.0.2.10/24"]
    PCB["LAB-PC-B<br/>192.0.2.11/24"]
  end
  PCA --- R1["LAB-R1-vSRX<br/>OFFICE 192.0.2.1"]
  PCB --- R1
  R1 ---|"WAN-A 198.51.100.1/30"| A["LAB-UP-A<br/>198.51.100.2/30<br/>lo: 203.0.113.10/32"]
  R1 ---|"WAN-B 198.51.100.5/30"| B["LAB-UP-B<br/>198.51.100.6/30<br/>lo: 203.0.113.10/32"]
~~~

*Rysunek 7.8 (autorski). Tekst alternatywny: PC-A i PC-B współdzielą OFFICE oraz bramę R1; dwa różne upstream Linux reprezentują WAN-A i WAN-B; oba mają identyczny testowy loopback, ale nie współdzielą segmentu Ethernet, więc ten sam cel pozwala udowodnić różny wybór ścieżki.*

Dlaczego używamy **tego samego 203.0.113.10/32 na dwóch Linux upstream**? Chodzi o kontrolowany eksperyment podobny do lokalnego anycast. Gdy PC-A i PC-B wysyłają do dokładnie tego samego destination, ale pakiety pojawiają się na dwóch różnych upstreamach, mamy dowód, że R1 wybrał **inną drogę**, a nie że jeden klient trafił do innego adresu. To izolowany test, nie pełny projekt usługi anycast z routingiem dynamicznym.

| Maszyna | Interfejs logiczny do weryfikacji | Adres | Rola |
|---|---|---|---|
| LAB-R1 | \`ge-0/0/1.0\` | 192.0.2.1/24 | OFFICE, ingress FBF |
| LAB-PC-A | \`eth0\` lub rzeczywisty NIC | 192.0.2.10/24 | Źródło „special” |
| LAB-PC-B | \`eth0\` lub rzeczywisty NIC | 192.0.2.11/24 | Źródło „normal” |
| LAB-R1 | \`ge-0/0/0.0\` | 198.51.100.1/30 | WAN-A |
| LAB-UP-A | \`eth0\` | 198.51.100.2/30 | Operator A |
| LAB-R1 | \`ge-0/0/2.0\` | 198.51.100.5/30 | WAN-B |
| LAB-UP-B | \`eth0\` | 198.51.100.6/30 | Operator B |
| Oba upstream Linux | \`lo\` | 203.0.113.10/32 | Wspólny testowy cel |
| LAB-UP-A | \`lo\` | 10.60.10.10/32 i 10.70.10.10/32 | Cele do tras contributor |
| LAB-UP-B | \`lo\` | 10.60.20.10/32 | Cel do drugiego contributor |

Wszystkie wartości IP są laboratoryjne. Zwróć uwagę, że **IP interfejsów EVE-NG nie jest automatycznie zgodne z numeracją \`ge-0/0/x\` w obrazie vSRX**. Dla konkretnej wersji mogą istnieć inne interfejsy danych i osobny management \`fxp0\`. Mapowanie jest pierwszą rzeczą do potwierdzenia.

### Faza 0. Zanim cokolwiek zepsujemy, sprawdźmy, co już jest na wirtualnych maszynach

Wchodzisz na R1, widzisz prompt \`>\`, ale nie wykonujesz jeszcze \`set\`. Najpierw sprawdzamy wersję, interfejsy, ewentualne istniejące strefy, NAT i konfigurację zarządzania. Jeśli maszyna była już używana, nie próbuj doprowadzić jej do stanu „czystego” komendą \`delete security\` czy przypadkowym factory reset. W takim przypadku lepiej stworzyć odrębną instancję labową.

**LAB-R1-vSRX — Operational Mode, read-only discovery**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration interfaces | display set
show configuration security zones | display set
show configuration security policies | display set
show configuration security nat | display set
show configuration routing-options | display set
show configuration routing-instances | display set
show configuration firewall | display set
show system commit
~~~

Na Linux hostach sprawdzamy nazwy interfejsów, adresację i systemowy firewall. Nie zakładaj, że masz \`eth0\`: współczesny Linux może pokazać \`ens3\`, \`enp1s0\` albo coś innego. Jeśli \`tcpdump\` nie jest zainstalowany, przygotuj obraz z tym narzędziem **przed** odizolowaniem labu, a nie instaluj go później przez produkcyjną sieć.

**LAB-PC-A — Linux shell, discovery**

~~~bash
ip -br link
ip -4 addr
ip -4 route
~~~

**LAB-UP-A — Linux shell, discovery**

~~~bash
ip -br link
ip -4 addr
ip -4 route
command -v tcpdump
~~~

Na końcu tej fazy powinieneś wiedzieć, które NIC tworzą OFFICE, WAN-A i WAN-B; czy nie ma konfliktujących adresów; czy security zone jest rzeczywiście pusta; i czy możesz bez ryzyka użyć \`commit confirmed\` mając console. Dopiero wtedy przystąp do konfiguracji.

### Faza 1. Trzy interfejsy R1 — ale jeszcze bez alternatywnej tablicy

R1 ma po prostu znać swoje trzy segmenty. Niczego jeszcze nie dzielimy między instances. Potrzebujemy najpierw stabilnego podłoża.

**LAB-R1-vSRX — Operational Mode, wejście do konfiguracji**

~~~junos
configure private
~~~

**LAB-R1-vSRX — Configuration Mode [edit private], wyłącznie po potwierdzeniu mapowania NIC**

~~~junos
set system host-name LAB-R1
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/24
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/30
set interfaces ge-0/0/2 unit 0 family inet address 198.51.100.5/30
set interfaces ge-0/0/1 description "CH07-OFFICE"
set interfaces ge-0/0/0 description "CH07-WAN-A"
set interfaces ge-0/0/2 description "CH07-WAN-B"
show | compare
commit check
commit confirmed 10
~~~

Dlaczego \`commit confirmed\`? Ponieważ popełnienie błędu w mapowaniu interfejsów może odciąć dostęp. W odizolowanym labie i z niezależną konsolą ta funkcja pozwala bezpiecznie cofnąć zmianę po upływie czasu, jeśli niczego nie potwierdzimy. **Po uruchomieniu \`commit confirmed\` nie wykonuj ponownie \`commit check\`**. Post-check ma być operacyjny, a nie kolejnym commitowym rytuałem.

### Faza 2. Konfiguracja hostów i droga powrotna

Przygotowujemy PC-A i PC-B. Oba są w tej samej sieci i mają tę samą bramę. Każdy z nich będzie potem wysyłał do 203.0.113.10. Pamiętaj, że poniżej \`IFACE=eth0\` to przykład; podstaw poprawną nazwę interfejsu. Jeśli obraz ma już skonfigurowany adres, nie dodawaj go drugi raz.

**LAB-PC-A — Linux shell, klient do przekierowania**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 192.0.2.10/24 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
ip -4 route
ping -c 3 192.0.2.1
~~~

**LAB-PC-B — Linux shell, klient korzystający ze zwykłego routingu**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 192.0.2.11/24 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
ip -4 route
ping -c 3 192.0.2.1
~~~

Na obu upstream Linux chcemy mieć **adres transit, testowy loopback oraz drogę powrotną do OFFICE**. To ostatnie łatwo przeoczyć. Wtedy zobaczysz echo request na UP-A, ale odpowiedź nie wróci do PC-A i wyciągniesz fałszywy wniosek „FBF nie działa”. Uzupełnimy drogę powrotną jawnie.

**LAB-UP-A — Linux shell, operator A**

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

**LAB-UP-B — Linux shell, operator B**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 addr add 198.51.100.6/30 dev "$IFACE"
sudo ip -4 addr add 203.0.113.10/32 dev lo
sudo ip -4 addr add 10.60.20.10/32 dev lo
sudo ip -4 route replace 192.0.2.0/24 via 198.51.100.5 dev "$IFACE"
ip -4 route
~~~

Te ustawienia \`ip\` są zwykle nietrwałe po restarcie Linux. To zamierzone w laboratorium, ale przy tworzeniu reprodukowalnego .unl warto je umieścić w skrypcie bootstrap dedykowanym do testowego obrazu. **Nie commituj haseł ani produkcyjnych konfiguracji.**

R1 może już sprawdzić reachability operatorów po transit. To jeszcze **nie** jest test FBF ani końcowej aplikacji, ale pozwala wykryć zamienione WAN-A z WAN-B.

**LAB-R1-vSRX — Operational Mode, test transitów**

~~~junos
ping 198.51.100.2 source 198.51.100.1 count 3
ping 198.51.100.6 source 198.51.100.5 count 3
show route 198.51.100.0/30 exact
show route 198.51.100.4/30 exact
~~~

Zanim potwierdzisz fazę interfejsów, upewnij się, że post-check przeszedł. Jeśli nie, pozwól automatycznemu rollbackowi lub napraw mapowanie przez konsolę, w sposób zgodny z planem.

**LAB-R1-vSRX — Configuration Mode [edit private], wyłącznie po PASS fazy 1–2**

~~~junos
commit
~~~

### Faza 3. SRX musi dopuścić testy — inaczej każda lekcja routingu skończy się fałszywą diagnozą

Na vSRX potrzebujemy security zones. Nasze testy mają korzystać z ICMP od klientów w OFFICE do obu upstreamów. Z punktu widzenia zasad bezpieczeństwa stworzymy tylko laboratoryjne polityki **CLIENT → WAN-A** i **CLIENT → WAN-B** dla \`junos-ping\`. Na firewallu produkcyjnym użyłbyś precyzyjnych adresów i aplikacji; nie jest to rekomendacja bezwarunkowego zezwalania na cały ruch.

**LAB-R1-vSRX — Configuration Mode [edit private], po sprawdzeniu ewentualnych konfliktów stref**

~~~junos
set security zones security-zone CLIENT interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone WAN-A interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone WAN-B interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A match source-address any
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A match destination-address any
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A match application junos-ping
set security policies from-zone CLIENT to-zone WAN-A policy CH07-PING-A then permit
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B match source-address any
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B match destination-address any
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B match application junos-ping
set security policies from-zone CLIENT to-zone WAN-B policy CH07-PING-B then permit
show | compare
commit check
commit confirmed 10
~~~

Dlaczego nie wprowadzamy reguł pozwalających na wszystko? Po pierwsze, nie potrzebujemy ich. Po drugie, chcemy utrzymać różnicę między **routowaniem pakietu** a **zezwoleniem na jego sesję**. Do testów wystarczy ICMP, jeśli konkretna wersja SRX ma predefiniowaną aplikację \`junos-ping\`. Sprawdź też, czy nie działa dziedziczone source NAT, które mogłoby zmienić adres widziany po drugiej stronie i zepsuć naszą obserwację source-based FBF.

**LAB-R1-vSRX — Operational Mode, kontrola stref i polityk**

~~~junos
show security zones
show security policies from-zone CLIENT to-zone WAN-A
show security policies from-zone CLIENT to-zone WAN-B
show configuration security nat | display set
~~~

Jeżeli output odpowiada założeniom i podstawowe testy działają, potwierdź warunkowy commit.

**LAB-R1-vSRX — Configuration Mode [edit private], po PASS security baseline**

~~~junos
commit
~~~

### Faza 4. Najpierw musi działać zwykły default via WAN-A

Zanim zaczniemy manipulować tablicami, ustawiamy klasyczną default route. Zwykły router z jedną tablicą powinien teraz kierować obu klientów do 203.0.113.10 przez WAN-A. Tak wygląda **baseline**, którego nie możemy pominąć.

**LAB-R1-vSRX — Configuration Mode [edit private], default w globalnej inet.0**

~~~junos
set routing-options static route 0.0.0.0/0 next-hop 198.51.100.2
show | compare
commit check
commit confirmed 10
~~~

**LAB-PC-A — Linux shell, przed FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**LAB-PC-B — Linux shell, przed FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

Teraz odpalamy jedną z najbardziej wartościowych obserwacji całego kursu. Użytkownicy nie zobaczą, którym WAN poszedł pakiet — obie strony mają ten sam testowy loopback. Ale na upstreamach możemy przechwycić pakiety. \`tcpdump\` jest tutaj znacznie mocniejszym dowodem niż \`show route\` R1.

**LAB-UP-A — Linux shell, obserwacja prób do testowego loopback**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**LAB-UP-B — Linux shell, kontrola czy ruch nie trafił na łącze zapasowe**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**Przewidywany wynik (NIEZWERYFIKOWANY RUNTIME):** requesty z PC-A i PC-B pojawią się na UP-A; UP-B nie zobaczy tych pakietów. Jeśli obydwa komputery nie mogą dostać odpowiedzi, oddzielnie sprawdź Linux return route, firewall, SRX security policy i NAT. Jeśli UP-B widzi requesty mimo braku FBF, znaczy to, że baseline różni się od zakładanego — nie przechodź do kolejnej fazy bez wyjaśnienia.

**LAB-R1-vSRX — Operational Mode, RIB i FIB baseline**

~~~junos
show route 0.0.0.0/0 exact extensive
show route forwarding-table destination 203.0.113.10
show security flow session
~~~

Po pozytywnym post-check potwierdź commit. W tym momencie obie klasy klientów mają działać przez WAN-A. Dopiero z tego stabilnego stanu będziemy budować bardziej zaawansowane mechanizmy.

**LAB-R1-vSRX — Configuration Mode [edit private], po PASS default**

~~~junos
commit
~~~


### Faza 5. Trzy statyczne drogi, dwa różne rodzaje summary

Mamy już prosty działający ruch. Teraz dokładamy trasy do sieci aplikacyjnych, aby nie mylić kolejnych mechanizmów z problemami fizycznego WAN. Na UP-A znajduje się testowy loopback 10.60.10.10 i 10.70.10.10, na UP-B — 10.60.20.10. R1 dostanie dwa bardziej szczegółowe prefixy wewnątrz **10.60.0.0/16** oraz jeden wewnątrz **10.70.0.0/16**.

Z pierwszej grupy stworzymy **aggregate reject**, z drugiej **generated** odziedziczony z contributora. Nie wprowadzamy jeszcze żadnego BGP export: to czysta obserwacja RIB/FIB.

**LAB-R1-vSRX — Configuration Mode [edit private], statyczne contributors i dwa summary**

~~~junos
set routing-options static route 10.60.10.0/24 next-hop 198.51.100.2
set routing-options static route 10.60.20.0/24 next-hop 198.51.100.6
set routing-options aggregate route 10.60.0.0/16
set routing-options static route 10.70.10.0/24 next-hop 198.51.100.2
set routing-options generate route 10.70.0.0/16
show | compare
commit check
commit confirmed 10
~~~

**LAB-R1-vSRX — Operational Mode, czy contributorzy są aktywni?**

~~~junos
show route 10.60.10.0/24 exact extensive
show route 10.60.20.0/24 exact extensive
show route 10.60.0.0/16 exact extensive
show route 10.70.10.0/24 exact extensive
show route 10.70.0.0/16 exact extensive
~~~

Są tu dwa rodzaje oczekiwań. **Oba /24** powinny być aktywne jako static przy działających WAN i rozwiązanych next-hopach. **Aggregate /16** powinien móc powstać dzięki aktywnym bardziej szczegółowym trasom, domyślnie z reject next-hop. **Generated /16** powinien móc powstać dzięki 10.70.10.0/24 i skierować nieobjęte bardziej szczegółowym prefiksem pakiety przez next-hop primary contributor.

Teraz trzy testy na PC-B. Zauważ, że wybieramy **różne cele w obrębie jednego summary**, żeby udowodnić, że działa LPM i że summary nie „zastępuje” realnych /24.

**LAB-PC-B — Linux shell, dwa realne hosty i nieistniejący fragment aggregate**

~~~bash
ping -c 3 10.60.10.10
ping -c 3 10.60.20.10
ping -c 3 10.60.99.25
~~~

Dwa pierwsze powinny przejść właściwymi upstreamami, jeśli security i return path są poprawne. Trzeci powinien zostać zatrzymany na R1 przez aggregate /16, ponieważ brak bardziej szczegółowej trasy do 10.60.99.25. Otrzymanie ICMP unreachable albo timeout zależy od platformy i filtrowania odpowiedzi; dlatego sprawdzamy FIB i położenie pakietu, a nie opieramy całego wniosku na ping.

**LAB-R1-vSRX — Operational Mode, forwarding do adresu nieistniejącego pod aggregate**

~~~junos
show route forwarding-table destination 10.60.99.25
show route 10.60.0.0/16 exact extensive
~~~

**LAB-PC-B — Linux shell, drugi eksperyment — generated**

~~~bash
ping -c 3 10.70.10.10
ping -c 3 10.70.99.25
~~~

Dla 10.70.99.25 oczekujemy innego zachowania niż przy aggregate: generated /16 może skierować pakiet przez **WAN-A**, pomimo braku hosta na upstream. Ping może się nie udać, ale jeśli zobaczymy request na UP-A, mamy dowód, że R1 przekazał go dalej.

**LAB-UP-A — Linux shell, obserwacja nieistniejącego celu wewnątrz generated summary**

~~~bash
sudo tcpdump -ni eth0 'host 10.70.99.25' -c 10
~~~

**Przewidywany wynik (NIEZWERYFIKOWANY):** UP-A zobaczy pakiety przeznaczone do 10.70.99.25, jeśli generated jest aktywna i forwarduje via next-hop primary contributor. Nie twierdzimy, że sam TCP/IP stack UP-A odpowie na taki pakiet. To test **forwardingu R1**, nie sukcesu usługi na nieistniejącym host.

**LAB-R1-vSRX — Configuration Mode [edit private], po pozytywnym sprawdzeniu obu summary**

~~~junos
commit
~~~

### Faza 6. Budujemy osobną tablicę, ale na razie nie kierujemy do niej klientów

R1 wciąż ma globalną default przez WAN-A. To ma pozostać bez zmian. Obok tworzymy \`FBF-ALT\` typu forwarding z default przez WAN-B. Ważne: do tej pory PC-A i PC-B nadal korzystają z \`inet.0\`, bo nigdzie nie podpięliśmy filtra wejściowego.

**LAB-R1-vSRX — Configuration Mode [edit private], tworzymy forwarding instance**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 0.0.0.0/0 next-hop 198.51.100.6
show | compare
commit check
commit confirmed 10
~~~

**LAB-R1-vSRX — Operational Mode, co druga tabela wie przed RIB group?**

~~~junos
show configuration routing-instances FBF-ALT | display set
show route table FBF-ALT.inet.0
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
~~~

Jeśli connected route 198.51.100.4/30 nie jest w secondary, nie traktuj tego jako awarii: to punkt dydaktyczny, który chcieliśmy zobaczyć. Sama static default w nowej instancji nie daje automatycznie dostępu do connected z globalnej tabeli. Zrób notatkę, czy default jest active/inactive i co mówi output. Ten etap nie musi powodować sukcesu forwarding — ponieważ FBF nie jest jeszcze aktywny, nie uszkadza normalnego ruchu klientów.

**LAB-R1-vSRX — Configuration Mode [edit private], po zapisaniu obserwacji**

~~~junos
commit
~~~

Teraz definiujemy RIB group \`COPY-INTERFACE-ROUTES\` z primary \`inet.0\` i secondary \`FBF-ALT.inet.0\`, a następnie podpinamy ją do interface routes. To powinno udostępnić w FBF-ALT podstawowe informacje do rozwiązania next-hop 198.51.100.6.

**LAB-R1-vSRX — Configuration Mode [edit private], grupa dla interface routes IPv4**

~~~junos
set routing-options rib-groups COPY-INTERFACE-ROUTES import-rib [ inet.0 FBF-ALT.inet.0 ]
set routing-options interface-routes rib-group inet COPY-INTERFACE-ROUTES
show | compare
commit check
commit confirmed 10
~~~

**LAB-R1-vSRX — Operational Mode, obserwacja po podpięciu RIB group**

~~~junos
show configuration routing-options rib-groups | display set
show configuration routing-options interface-routes | display set
show route table inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
~~~

**Oczekiwany wynik (NIEZWERYFIKOWANY):** FBF-ALT zna connected do WAN-B i jej default może zostać poprawnie rozwiązana. Jeśli na tej platformie RIB group nie udostępnia oczekiwanych tras, zatrzymaj lab i sprawdź wsparcie konkretnego Junos, zamiast dorzucać przypadkowe \`next-table\` albo przenosić interfejsy pomiędzy instancjami.

**LAB-R1-vSRX — Configuration Mode [edit private], dopiero po PASS RIB group**

~~~junos
commit
~~~

Właśnie przeprowadziliśmy ważny eksperyment: przygotowaliśmy alternatywną trasę bez jeszcze jednej zmiany routingu klientów. I to jest dobra praktyka produkcyjna — najpierw przygotuj nowy kontekst i sprawdź jego next-hop, dopiero potem decyduj, jakie pakiety mają zacząć z niego korzystać.

### Faza 7. Włączamy FBF dla PC-A, ale PC-B ma nie zauważyć zmiany

Mamy \`inet.0\` dla WAN-A i \`FBF-ALT.inet.0\` dla WAN-B. Teraz wejściowy stateless filter wybierze alternatywną tablicę dla jednego źródła. Nie włączaj go, jeśli FBF nie jest wspierany na danym vSRX lub jeśli FBF-ALT default nadal jest unresolved. Jeśli to SRX, przed zmianą potwierdź również \`CLIENT → WAN-B\` security policy.

**LAB-R1-vSRX — Configuration Mode [edit private], FBF na ingress OFFICE**

~~~junos
set firewall family inet filter OFFICE-FBF term SPECIAL from source-address 192.0.2.10/32
set firewall family inet filter OFFICE-FBF term SPECIAL then routing-instance FBF-ALT
set firewall family inet filter OFFICE-FBF term NORMAL then accept
set interfaces ge-0/0/1 unit 0 family inet filter input OFFICE-FBF
show | compare
commit check
commit confirmed 10
~~~

Teraz jeden z najważniejszych testów w całym laboratorium. **PC-A i PC-B wysyłają do tego samego 203.0.113.10**. Oczekujemy, że pierwszy zostanie skierowany przez WAN-B, drugi pozostanie na WAN-A. Uważnie rozróżnij requesty od odpowiedzi, bo na uplinkach możesz zobaczyć oba kierunki.

**LAB-PC-A — Linux shell, klient objęty FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**LAB-PC-B — Linux shell, klient nieobjęty FBF**

~~~bash
ping -c 5 203.0.113.10
~~~

**LAB-UP-A — Linux shell, obserwacja requestów klienta zwykłego**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

**LAB-UP-B — Linux shell, obserwacja requestów klienta specjalnego**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 20
~~~

Spodziewamy się, że na UP-B pojawi się źródło **192.0.2.10**, a na UP-A źródło **192.0.2.11**. Jeśli ruch pakietowy trafia na poprawny upstream, ale ping nie ma odpowiedzi, przyjrzyj się drodze powrotnej, strefom oraz stateful session na SRX. Sama klasyfikacja FBF nie daje automatycznej permisji security.

**LAB-R1-vSRX — Operational Mode, status po zmianie**

~~~junos
show configuration interfaces ge-0/0/1 | display set
show configuration firewall family inet filter OFFICE-FBF | display set
show route table inet.0 0.0.0.0/0 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact
show security flow session
~~~

**Oczekiwany wynik (NIEZWERYFIKOWANY):** działa host-to-host test z obu PC; requesty PC-A pojawiają się tylko na WAN-B, a PC-B tylko na WAN-A, bez niezamierzonych dodatkowych tras. Ostateczna obserwacja ma pochodzić z faktycznego \`tcpdump\` oraz session/route output — nie tylko z faktu, że parser przyjął \`then routing-instance\`.

Jeśli dwa niezależne testy przeszły, potwierdź \`commit\`. W przeciwnym razie wykorzystaj rollback, nie usuwaj całego filtrowania.

**LAB-R1-vSRX — Configuration Mode [edit private], dopiero po PASS obu klientów**

~~~junos
commit
~~~

Teraz mamy pierwszy kompletny punkt odniesienia: globalny ruch korzysta z WAN-A, a jedno źródło z WAN-B. W dalszej części celowo będziemy go psuć i zbierać dowody, dlaczego.


### Faza 8. Pierwsza awaria: aggregate przestaje istnieć, choć jego konfiguracja została

Zaczniemy od bezpiecznego eksperymentu, który nie powinien odciąć nam całej sieci OFFICE. Wycofujemy oba static routes /24 tworzące 10.60.0.0/16, ale nie usuwamy \`aggregate route 10.60.0.0/16\` z konfiguracji. Co przewidujesz?

Jeżeli nie ma innych aktywnych contributorów ani specjalnego \`passive\`, aggregate straci podstawę do aktywności. Właśnie dlatego \`commit check\` nie wystarczy jako test routingu: konfiguracja /16 będzie nadal poprawna składniowo, ale route może przestać być aktywna.

**LAB-R1-vSRX — Configuration Mode [edit private], kontrolowany fault tylko w odizolowanym labie**

~~~junos
delete routing-options static route 10.60.10.0/24
delete routing-options static route 10.60.20.0/24
show | compare
commit check
commit confirmed 2
~~~

**LAB-R1-vSRX — Operational Mode, podczas odliczania — bez kolejnego commit/check**

~~~junos
show route 10.60.0.0/16 exact extensive
show route protocol aggregate
show configuration routing-options aggregate | display set
show system commit
~~~

**Oczekiwanie (NIEZWERYFIKOWANE):** brak aktywnego aggregate 10.60.0.0/16, ale konfiguracja \`aggregate route\` nadal istnieje. Wyjątkiem byłby inny kwalifikujący się contributor lub celowo ustawiona opcja zmieniająca zachowanie trasy.

**Naprawa:** nie zatwierdzaj tej zmiany. Pozwól automatycznemu rollbackowi po \`commit confirmed 2\` przywrócić contributor routes. Po odtworzeniu wykonaj read-only \`show route\` i test hostów. Jeśli rollback nie nastąpi, zatrzymaj test, sprawdź historię i zastosuj zaplanowane ręczne przywrócenie baseline przez konsolę. Nie traktuj \`rollback 1\` jak stałego aliasu do „naszej wersji sprzed pięciu faz”, bo historia zmienia się po commit.

### Faza 9. Druga awaria: FBF istnieje, ale WAN-B nie daje się rozwiązać w secondary RIB

Teraz celowo odpinamy RIB group od źródła tras interfejsowych. Sama definicja \`COPY-INTERFACE-ROUTES\` zostaje w konfiguracji, podobnie jak \`FBF-ALT\` i input filter na OFFICE. To ma pokazać różnicę między *istniejącym obiektem*, *używaną grupą* i *faktyczną trasą connected w secondary table*.

Najpierw przewidywanie: jeśli FBF-ALT nie ma innej drogi do 198.51.100.6, a connected route przestanie być dostępna, jej default przez B może stać się unresolved. PC-A powinien wtedy utracić prawidłową alternatywną ścieżkę, podczas gdy PC-B nadal może korzystać z normalnej \`inet.0\` przez A. Dokładny efekt na danym vSRX musimy zaobserwować, a nie zakładać.

**LAB-R1-vSRX — Configuration Mode [edit private], kontrolowane usunięcie powiązania RIB group**

~~~junos
delete routing-options interface-routes rib-group inet COPY-INTERFACE-ROUTES
show | compare
commit check
commit confirmed 2
~~~

**LAB-R1-vSRX — Operational Mode, rozpoznanie problemu podczas pending rollback**

~~~junos
show route table FBF-ALT.inet.0 198.51.100.4/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
show route table inet.0 198.51.100.4/30 exact
show configuration routing-options interface-routes | display set
show system commit
~~~

**LAB-PC-A — Linux shell, klient SPECIAL w czasie awarii**

~~~bash
ping -c 3 203.0.113.10
~~~

**LAB-PC-B — Linux shell, klient NORMAL w czasie awarii**

~~~bash
ping -c 3 203.0.113.10
~~~

**Naprawa:** pozwól wygaśnięciu \`commit confirmed\` automatycznie przywrócić powiązanie \`interface-routes\`, a potem sprawdź connected w FBF-ALT i właściwy upstream dla PC-A. Jeśli secondary nadal nie ma potrzebnej trasy, nie dodawaj przypadkowej kolejnej default; sprawdź wsparcie RIB group i istniejące import-policy.

### Faza 10. Trzecia awaria: włączyliśmy FBF tylko dla jednego hosta, ale odcięliśmy resztę biura

W naszej konfiguracji filtr \`OFFICE-FBF\` ma dwa termy: SPECIAL i NORMAL. Do tej pory NORMAL był trochę niewidocznym bohaterem całej historii. Zobaczysz jego znaczenie, gdy usuniemy go na dwie minuty. FBF dla PC-A pozostanie skonfigurowany, ale inne źródła nie dopasują już żadnej reguły. Standardowy stateless filter ma na końcu **implicit discard**.

**LAB-R1-vSRX — Configuration Mode [edit private], kontrolowany błąd wyłącznie w laboratorium**

~~~junos
delete firewall family inet filter OFFICE-FBF term NORMAL
show | compare
commit check
commit confirmed 2
~~~

**LAB-PC-A — Linux shell, dopasowany do SPECIAL**

~~~bash
ping -c 3 203.0.113.10
~~~

**LAB-PC-B — Linux shell, niepasujący do SPECIAL**

~~~bash
ping -c 3 203.0.113.10
~~~

**Oczekiwany wynik:** PC-A może nadal korzystać z WAN-B, podczas gdy PC-B jest odrzucany przez brak termu kończącego filtrowanie \`accept\`. Trasa default PC-B w \`inet.0\` nadal jest aktywna; to nie routing, tylko stateless filter zatrzymał pakiet przed jego zwykłą ścieżką. Jeśli PC-B nadal działa, sprawdź, czy właściwy filter jest aktywny i czy pakiet faktycznie przechodzi przez ingress, który testujemy.

**Naprawa:** nie zatwierdzaj zmiany. Pozwól automatycznemu rollbackowi przywrócić NORMAL, a następnie wykonaj dwa testy post-check. Nie usuwaj filtra w całości, bo celem nie jest wyłączenie FBF dla PC-A, tylko przywrócenie ruchu, który powinien pozostać nietknięty.

### Faza 11. Czwarta lekcja: generated route nie jest bezpiecznym reject summary

To nie musi być nowa konfiguracja fault injection. Po prostu wróć do tras 10.70.0.0/16 i do obserwacji UP-A. Zwykłe \`aggregate 10.60.0.0/16\` chroni nieistniejący fragment summary przez reject. \`generate 10.70.0.0/16\` może wysłać pakiet do nieistniejącego fragmentu /16 przez next-hop contributora.

**LAB-PC-B — Linux shell, dwa cele bez hostów w odpowiednich summary**

~~~bash
ping -c 3 10.60.99.25
ping -c 3 10.70.99.25
~~~

**LAB-UP-A — Linux shell, obserwacja nieobsługiwanego fragmentu generated**

~~~bash
sudo tcpdump -ni eth0 'host 10.70.99.25 or host 10.60.99.25' -c 10
~~~

**Oczekiwanie:** pakiet do 10.70.99.25 może opuścić R1 przez A, podczas gdy pakiet do 10.60.99.25 powinien trafić w reject w R1, jeśli aggregate jest aktywny i nie ma konkretnej trasy. **Niedziałający ping do obu adresów nie oznacza tego samego mechanizmu**. Różnicę rozpoznajemy po RIB/FIB i rzeczywistym capture na UP-A.

### Faza 12. Piąta lekcja — ECMP poza FBF, aby nie mieszać dwóch mechanizmów

Dopiero gdy zakończysz fault injection FBF, rozpocznij osobny eksperyment na dwóch równorzędnych next-hopach. W przeciwnym razie nie będziesz wiedzieć, czy dany pakiet wybrał WAN-B na podstawie FBF czy hasha ECMP.

Tymczasowo zdejmujemy filtr \`OFFICE-FBF\` **z interfejsu**, ale go nie kasujemy z konfiguracji. Następnie tworzymy do tego samego 203.0.113.10/32 dwa static next-hopy. Stosujemy forwarding-table export policy \`ECMP-TO-SERVICE\`, której zadaniem jest umożliwienie wykorzystania multipath w FIB.

**LAB-R1-vSRX — Configuration Mode [edit private], ODRĘBNY WARIANT ECMP**

~~~junos
delete interfaces ge-0/0/1 unit 0 family inet filter input OFFICE-FBF
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.2
set routing-options static route 203.0.113.10/32 next-hop 198.51.100.6
set policy-options policy-statement ECMP-TO-SERVICE term SERVICE from route-filter 203.0.113.10/32 exact
set policy-options policy-statement ECMP-TO-SERVICE term SERVICE then load-balance per-packet
set policy-options policy-statement ECMP-TO-SERVICE term SERVICE then accept
set policy-options policy-statement ECMP-TO-SERVICE term REST then accept
set routing-options forwarding-table export ECMP-TO-SERVICE
show | compare
commit check
commit confirmed 10
~~~

W tym eksperymencie **nie potwierdzamy od razu** nowego stanu. Chcemy porównać RIB i FIB, a potem pozwolić na bezpieczny rollback do bazowego FBF. W produkcji nigdy nie zastępuj istniejącej globalnej forwarding-table policy bez review — tutaj bazowy router miał nie mieć żadnej.

**LAB-R1-vSRX — Operational Mode, weryfikacja kwalifikacji multipath**

~~~junos
show route 203.0.113.10/32 exact extensive
show route forwarding-table destination 203.0.113.10
show configuration routing-options forwarding-table | display set
~~~

Przypomnij sobie pułapkę nazwy: \`load-balance per-packet\` na wielu Junos prowadzi w praktyce do **per-flow balancing**. Uruchomienie jednego ping z PC-A nie musi dać równomiernego podziału. Warto wykonać wiele **niezależnych flows**, różniących się source/destination lub portami, jeśli konkretne narzędzia i hash platformy pozwalają to obserwować.

**LAB-PC-A — Linux shell, podstawowy test jednego źródła**

~~~bash
ping -c 20 203.0.113.10
~~~

**LAB-PC-B — Linux shell, drugi niezależny source**

~~~bash
ping -c 20 203.0.113.10
~~~

**LAB-UP-A — Linux shell, ruch w wariancie ECMP**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 30
~~~

**LAB-UP-B — Linux shell, ruch w wariancie ECMP**

~~~bash
sudo tcpdump -ni eth0 'icmp and host 203.0.113.10' -c 30
~~~

Tylko dwa hosty i ICMP to **ograniczona próba**, nie statystyczny dowód równomiernego ECMP. Jeżeli każdy ping wybrał ten sam WAN, ale FIB ma oba next-hopy, eksperyment wymaga większej liczby niezależnych flows, a nie natychmiastowej zmiany konfiguracji. Możesz użyć przygotowanego wcześniej offline narzędzia generującego połączenia z różnymi portami, o ile platforma hash rzeczywiście bierze je pod uwagę. Również 70/30 w liczbie bajtów może być poprawne przy per-flow, jeśli rozmiary strumieni są nierówne.

**Naprawa/przywrócenie FBF baseline:** pozwól \`commit confirmed\` wycofać wariant ECMP i sprawdź, że filter \`OFFICE-FBF\` znowu jest przypięty do OFFICE, a forwarding-table export wróciło do wcześniejszego stanu. Po tym teście PC-A ma znów korzystać z WAN-B, a PC-B z WAN-A — inaczej laboratorium nie wróciło do właściwego punktu odniesienia.

### Faza 13. Martians pozostawiamy jako read-only

To świadoma decyzja. Zmienianie martian list na tym samym R1 mogłoby utrudnić interpretację wcześniejszych tras oraz zachowania FBF. Dlatego w tym głównym labie wykonujemy tylko obserwację.

**LAB-R1-vSRX — Operational Mode, domyślne martian prefixes i zwykłe trasy firmy**

~~~junos
show route martians
show route 10.60.0.0/16 exact
show route 127.0.0.0/8
~~~

Nie wyciągaj wniosku, że istnieje trasa 127.0.0.0/8 tylko dlatego, że jest reguła martian. I nie dodawaj \`allow\` dla zakresu specjalnego „aby móc przetestować ping”. Oddzielny eksperyment martian można zrealizować na osobnym węźle z kontrolowanym prefiksem i zapisanymi stanami, bez mieszania go z usługą OFFICE.

### Faza 14. Cleanup i evidence package

Często bardziej wartościowe niż sam wynik ping jest to, co pozostawiasz po zakończeniu szkolenia. Masz dwie sensowne drogi:

**Wariant zachowania labu:** pozostaw działający baseline FBF jako przyszły punkt odniesienia. Usuń tylko tymczasowe eksperymenty i potwierdź, że nie ma \`commit confirmed\` w toku, PC-A korzysta z WAN-B, PC-B z WAN-A, RIB group działa, a ECMP-only wariant został wycofany. Oznacz lab jako **PLANNED**, dopóki nie uruchomisz go i nie zbierzesz rzeczywistych wyników.

**Wariant pełnego przywrócenia:** użyj izolowanego snapshotu maszyny sprzed ćwiczenia, jeśli taki snapshot jest rzeczywiście sprawdzony i obejmuje odpowiednie dyski/konfigurację. Sam restart wirtualnej maszyny nie jest automatycznym rollback. Jeśli nie masz snapshotu, wykonaj **punktowy cleanup** wyłącznie swoich obiektów i tylko po przejrzeniu \`show | compare\`. Nie kasuj cudzych \`routing-options\`, \`security\` czy \`interfaces\` globalnie.

Wariant punktowy wciąż wymaga ostrożności. Poniższe polecenia usuwają tylko obiekty nazwane w tym rozdziale i zakładają **świeżego, izolowanego** vSRX bez obcych referencji. Najpierw sprawdź dokładnie aktualną konfigurację. Jeśli coś zostało użyte także przez inny lab, nie wykonuj cleanup automatycznie.

**LAB-R1-vSRX — Configuration Mode [edit private], przykład punktowego usunięcia obiektów zaawansowanych**

~~~junos
delete interfaces ge-0/0/1 unit 0 family inet filter input OFFICE-FBF
delete firewall family inet filter OFFICE-FBF
delete routing-options forwarding-table export ECMP-TO-SERVICE
delete policy-options policy-statement ECMP-TO-SERVICE
delete routing-options static route 203.0.113.10/32
delete routing-options interface-routes rib-group inet COPY-INTERFACE-ROUTES
delete routing-options rib-groups COPY-INTERFACE-ROUTES
delete routing-instances FBF-ALT
delete routing-options aggregate route 10.60.0.0/16
delete routing-options generate route 10.70.0.0/16
delete routing-options static route 10.60.10.0/24
delete routing-options static route 10.60.20.0/24
delete routing-options static route 10.70.10.0/24
show | compare
commit check
commit confirmed 10
~~~

Jeśli nie wykonywałeś wariantu ECMP i policy nie istnieje, polecenie \`delete\` może zgłosić brak obiektu — to nie dowodzi uszkodzenia routera. W takim przypadku pomiń odpowiednią linię po przejrzeniu aktualnego stanu, zamiast wklejać blok bezmyślnie.

W zależności od planowanego końcowego stanu możesz zachować podstawową default, adresy interfejsów i reguły \`CH07-PING-A/B\` jako bazę innych labów albo usunąć je osobno, punktowo, odtwarzając snapshot. Nie podaję masowego \`delete security\`, ponieważ mogłoby przypadkowo usunąć management i inne potrzebne reguły. Przy takim cleanup **brak host-to-host reachability po usunięciu static/FBF może być oczekiwany**; test końcowy ma potwierdzać docelowy stan, a nie poprzedni.

**LAB-R1-vSRX — Operational Mode, końcowe sprawdzenie stanu**

~~~junos
show system commit
show configuration firewall | display set
show configuration routing-instances | display set
show configuration routing-options | display set
show interfaces terse
show route
~~~

**LAB-R1-vSRX — Configuration Mode [edit private], po zatwierdzonych testach cleanup**

~~~junos
commit
~~~

Jeśli chcesz mieć z tego pełnowartościowy dowód laboratoryjny, zapisz w \`VERIFY.md\` rzeczywisty release, mapowanie NIC, konfigurację baseline, liczbę aktywnych tras, \`show route\` do obu summary, zawartość obu RIB, wynik FBF na obu upstreamach, ścieżkę rollback każdego fault i status osobno dla każdego testu. Nie wpisuj „RUNTIME VALIDATED”, jeśli jedynym dowodem jest poprawny \`commit\` albo statyczna kontrola pliku .unl.

### Czego nauczyliśmy się naprawdę?

Najlepszą częścią ćwiczenia nie jest to, że potrafisz wpisać \`set routing-instances FBF-ALT\`. Najważniejsze jest rozumienie **dlaczego**. Zbudowaliśmy dwie alternatywne drogi, ale to nie wystarczyło do rozwiązania problemu. Musieliśmy nauczyć drugą tablicę, jak dotrzeć do własnego next-hopa, a potem wskazać wyłącznie wybrane pakiety do tej tablicy. Równolegle zobaczyliśmy, że aggregate i generated inaczej traktują pakiety do nieistniejących fragmentów summary, a ECMP nie obiecuje dokładnie 50/50 dla pojedynczego połączenia.

To już nie jest „więcej komend routingu”. To jest świadome projektowanie **punktów podejmowania decyzji w Junos**.


---

## Warsztat projektowy. Jak wykorzystać tę wiedzę przy budowaniu prawdziwego oddziału Enterprise?

Wyobraź sobie spotkanie HLD z architektem, bezpieczeństwem i dostawcą łączy. Ktoś pokazuje slajd „dual-WAN + guest traffic via backup + summary routes” i oczekuje, że zamienisz go na konkretną decyzję techniczną. Po tym rozdziale nie powinieneś odpowiadać: „Da się, Junos ma FBF”. Powinieneś zapytać, **jakie zachowanie musi być gwarantowane, a jakie jest tylko preferencją**.

Jeśli wymaganie brzmi „cały oddział ma mieć internet, a w razie awarii głównego operatora automatycznie przełączyć się na backup”, najważniejsze będą **detekcja awarii, route preference i kontrola drogi powrotnej**. Być może nawet nie potrzebujesz FBF. Dwie równorzędne static defaults bez monitorowania realnej osiągalności aplikacji nie są pełną strategią HA, bo sam link up nie oznacza działającego upstreama.

Jeśli wymaganie brzmi „ruch guest Wi-Fi ma zawsze przejść przez centralny firewall”, trzeba zdecydować, **gdzie fizycznie i logicznie klasyfikujesz ruch**: na EX/QFX, IRB, routerze WAN, czy na SRX. Czy widzisz prawdziwy source IP gościa przed NAT? Czy filter jest na właściwym logical unit? Czy provider B zna return path? Czy ruch nie może ominąć centralnego firewalla przez bardziej szczegółową trasę albo inną routing instance? Jeśli te pytania pozostają bez odpowiedzi, sama konfiguracja FBF nie spełnia wymagań bezpieczeństwa.

Jeśli wymaganie brzmi „zredukujmy ilość tras wymienianych z operatorem”, warto rozważyć **aggregate**, ale zadać trudniejsze pytanie: co stanie się, kiedy aktywna zostanie tylko jedna contributing /24? Czy nadal chcemy reklamować całe /16, wiedząc, że niektóre fragmenty będą trafiały do reject? To wybór między uproszczeniem informacji routingu a precyzją reachability. Generated route jest innym narzędziem i nie powinno zastępować aggregate tylko dlatego, że obie mogą wyglądać w RIB jak \`[Aggregate/130]\`.

Jeżeli klient oczekuje „sumy łącz 1 Gb/s + 1 Gb/s dla jednego transferu”, nie obiecuj takiego rezultatu na podstawie ECMP. Per-flow hashing zwykle pozwoli rozkładać **wiele niezależnych przepływów**, a nie złoży pojedynczego flow TCP w 2 Gb/s. Zanim zaprojektujesz dwa WAN-y jako aktywne, zapytaj o profile ruchu, NAT/statefulness, asymetrię, przepustowość poszczególnych łączy, SLA operatorów i możliwości obserwacji rzeczywistych flows.

Wreszcie, wybór sprzętu. To, że dany model ma Junos, nie oznacza automatycznie identycznego FBF, ECMP hash-key i RIB groups. Przy EX/QFX liczą się możliwości konkretnego ASIC i Junos release. Przy MX — dostępne PFE i warianty forwarding. Przy SRX/vSRX — security flow, NAT i obsługiwany typ PBR/APBR. Zanim taki projekt trafi do BOM, trzeba sprawdzić Feature Explorer, hardware platform guide i sugerowany release JTAC dla konkretnego SKU. **Nie wymyślaj w ofercie jednej „wersji Junosa dla całej rodziny” ani nie zaznaczaj feature jako supported na podstawie samego przykładu z innej platformy.**

To są właśnie informacje, których często brakuje w kursach egzaminacyjnych. Na egzaminie pytanie może brzmieć: „Co robi RIB group?”. W pracy pytanie od architekta brzmi: „Czy klient z sieci OT może przypadkiem uzyskać route do management, gdy udostępnimy interface routes do forwarding instance?”. Dobra odpowiedź wymaga nie tylko składni, lecz świadomego określenia **zakresu informacji**, która przechodzi pomiędzy tablicami.

---

## Epilog. Nie ma jednej mapy dla wszystkich pakietów

Wracamy do pierwszego telefonu z NOC. Dwa uplinki są zielone, ale jeden użytkownik nie działa. Wtedy łatwo powiedzieć: „Problem z routingiem”, i stracić godzinę na czytanie \`show route\`.

Teraz jesteśmy dużo bardziej precyzyjni. **Static route** opisuje świadomie zadaną drogę. **Aggregate** może stworzyć summary z domyślnym reject next-hop, jeśli są odpowiedni contributors. **Generated** może stworzyć warunkową trasę, która odziedziczy next-hop. **Martians** opisują, jakie informacje routingu mogą być problematyczne niezależnie od BGP import. **Routing instances** dają osobne tablice, a **RIB groups** udostępniają im potrzebne informacje, np. do next-hop resolution. **ECMP i forwarding-table export** wpływają na to, które równorzędne next-hopy trafią do FIB. **FBF** pozwala wybrać tablicę na podstawie pakietu — bez magicznego zmieniania wszystkich tras w \`inet.0\`.

To nie jest lista funkcji do wykucia. To kolejne decyzje w historii jednego pakietu.

Warto zakończyć rozdział kilkoma pytaniami. Nie musisz zapamiętać całych outputów, ale powinieneś umieć je rozstrzygnąć z CLI.

**Pytanie pierwsze.** R1 ma aktywne 10.60.20.0/24, ale WAN-A nie działa. Czy 10.60.0.0/16 aggregate może nadal być aktywny? **Tak**, jeśli istnieje choć jeden kwalifikujący się contributor i nie ma dodatkowych ograniczeń. Nie wynika z tego, że wszystkie podsieci wewnątrz /16 są osiągalne.

**Pytanie drugie.** Dlaczego \`show route protocol aggregate\` pokazuje 10.70.0.0/16, chociaż skonfigurowałeś \`generate\`? Ponieważ w Junos typ wyświetlany w RIB i jego domyślna preference dla aggregate/generated mogą być podobne. Rozróżnij je po konfiguracji i next-hop.

**Pytanie trzecie.** \`FBF-ALT.inet.0\` ma skonfigurowaną default przez 198.51.100.6. Czy na pewno przekazuje ruch? **Nie**: sprawdź, czy w tym kontekście istnieje droga do samego next-hopa, czy jest aktywna, a potem czy pakiet został skierowany do tej tabeli przez ingress filter.

**Pytanie czwarte.** Dlaczego po usunięciu \`NORMAL then accept\` nie działa PC-B, mimo że zwykła globalna default jest nadal aktywna? Bo stateless input filter może odrzucić pakiet niedopasowany do żadnego termu, zanim zwykła droga będzie dla niego użyteczna.

**Pytanie piąte.** W FIB widzisz dwa next-hopy, ale jeden transfer TCP korzysta z WAN-A. Czy ECMP jest uszkodzony? **Niekoniecznie**: per-flow hashing może przypisać cały przepływ do jednego next-hopa. Rozkład trzeba badać na większej liczbie niezależnych flows.

**Pytanie szóste.** Czy FBF zastępuje security policy na vSRX? **Nie**. To odrębny wybór kontekstu routingu; stateful flow, strefy, NAT i droga powrotna muszą być poprawne niezależnie od FBF.

Jeśli potrafisz wyjaśnić te sytuacje bez patrzenia do ściągi, jesteś gotowy na następny rozdział. Tam spotkamy bardziej rozbudowane polityki import/export i firewall filters, ale nie będą już wyglądały jak obce narzędzie. Znasz bowiem najważniejszy fundament: **oddzielasz informację o trasach, wybór forwarding table i los konkretnego pakietu.**

---

## Źródła, archiwum i granice weryfikacji

Ta wersja została przebudowana z oryginalnego rozdziału 7 na podstawie [Unified Blueprint](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md), z uwzględnieniem poprzednich ustaleń stylistycznych projektu. Poprzednia wersja pozostaje w archiwum: [JUNOS_Rozdzial_07_Protocol_Independent_Routing_v1_ARCHIVE.md](JUNOS_Rozdzial_07_Protocol_Independent_Routing_v1_ARCHIVE.md). Zachowano pełny zakres **7.1–7.8**: static/aggregate/generated, martians, routing instances, RIB groups, load balancing, FBF, troubleshooting i laboratorium. Przywrócono też kontekst operatorski greenfield, ale nie zamieniono go w osobny katalog sprzętowy.

Najważniejsze materiały producenta użyte do sprawdzenia faktów technicznych:

1. [Route Aggregation](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config-route-aggregation.html) i [Generated Routes](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/concept/policy-generated-routes.html): contributors, aktywność i semantyka next-hop.
2. [Static Routes / Qualified Next Hop](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/static-route-prefer-qualified-next-hop.html): kwalifikacja statycznych tras.
3. [Martian Addresses](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/recognize-martian-addr-routing.html) i [show route martians CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-martians.html): prefix rules i allowed/disallowed.
4. [Routing Instances](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html): oddzielne konteksty tablic.
5. [RIB Groups](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/rib-groups-edit-routing-options.html), [import-rib](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/import-rib-edit-routing-options.html) i [interface-routes](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/interface-routes-edit-routing-options.html): primary/secondary i udostępnianie tras.
6. [Per-Flow Load Balancing](https://www.juniper.net/documentation/us/en/software/junos/sampling-forwarding-monitoring/topics/concept/policy-configuring-per-flow-load-balancing.html): historyczne \`load-balance per-packet\` i rzeczywiste per-flow zachowanie.
7. [Filter-Based Forwarding Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html) i [przykład konfiguracji FBF](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html): forwarding instance, match filter, RIB group.
8. [Advanced PBR on SRX](https://www.juniper.net/documentation/us/en/software/junos/application-identification/topics/topic-map/security-application-advanced-policy-based-routing.html): ostrzeżenie przed bezrefleksyjnym przenoszeniem przykładów MX/EX na flow-based SRX.

Wszystkie diagramy Mermaid są autorskie; nie pobierano cudzych ilustracji. Linki do Juniper mogą przekierowywać do dokumentacji HPE po zmianach organizacyjnych. Przykłady CLI nie są podpisane jako \`RUNTIME VALIDATED\`, ponieważ nie wykonano ich na konkretnym urządzeniu. Dla różnych rodzin PFE i wersji Junos należy zweryfikować wsparcie FBF, import RIB groups, sposób hashowania ECMP, kolejność przetwarzania SRX i format wyjść diagnostycznych.

**Mapa pokrycia blueprintu:** 7.1 static/aggregate/generated; 7.2 martians; 7.3 routing instances; 7.4 RIB groups; 7.5 load balancing; 7.6 FBF; 7.7 konfiguracja, monitoring i troubleshooting; 7.8 laboratorium. Wyróżnienia egzaminacyjne są wplecione w przebieg lekcji, a nie tworzą osobnej listy ściągawek.

*Ostatnia rada z warsztatu: jeśli nowa funkcja Junos wydaje się trudna, nie zaczynaj od wkuwania hierarchii \`set\`. Narysuj drogę jednego pakietu, wypisz tablicę, z której wykonuje lookup, i wskaż, skąd ta tablica dostała informację o next-hopie. Wtedy komendy zaczną wynikać z rozwiązania problemu.*
