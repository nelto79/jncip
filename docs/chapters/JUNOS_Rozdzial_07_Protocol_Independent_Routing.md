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
  L -->|10.60.10.10| A["10.60.10.0/24 → R2"]
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

Tutaj również istnieje contributor. Ale **generated route może przejąć next-hop od primary contributing route**. W naszej prostej sieci będzie nim droga do 10.70.10.0/24 przez 10.12.0.2. Zatem pakiet do 10.70.99.25, choć nie ma własnej /24, może zostać wysłany do R2. Tam upstream może go odrzucić. Sam ping nie wystarczy, żeby powiedzieć, czy R1 wysłał pakiet: trzeba zobaczyć forwarding table albo ruch na interfejsie R2.

~~~mermaid
flowchart LR
  C["Aktywny contributor<br/>10.70.10.0/24 via R2"] --> G["Generated<br/>10.70.0.0/16"]
  G --> N["Next-hop z primary contributor<br/>10.12.0.2"]
  P["Pakiet do 10.70.99.25"] --> G
  N --> W["R2 (górny transit): pakiet może wyjść<br/>mimo braku hosta 10.70.99.25"]
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

Przed przejściem dalej przeanalizuj trzy cele: 10.60.10.10, 10.60.99.25 i 10.70.99.25. Pierwszy wybierze konkretną /24; drugi może trafić w aggregate reject; trzeci może opuścić R1 przez R2 (górny transit) dzięki generated. Wszystkie trzy zachowania mogą być jednocześnie poprawne.

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

Wracamy do naszego DIAMOND. R1 ma dwa istniejące wyjścia: **R2 przez 10.12.0.2** i **R3 przez 10.13.0.2**. Kierownik pyta: „Czy do określonej usługi PC-B możemy wymusić przejazd przez R3, podczas gdy zwykły ruch nadal wybiera R2?” To zadanie **wyboru tabeli routingu**, nie wymiany fizycznego okablowania.

W podstawowym `inet.0` R1 zna LAN-B `198.51.100.0/24` przez R2. Samo dopisanie gorszego backupu przez R3 nie gwarantuje, że *konkretnie dopasowany pakiet* pójdzie do R3. Nie możemy także udawać, że w kanonicznym LAN-A istnieje drugi host: mamy **tylko PC-A 192.0.2.10**. Dlatego w tej lekcji zmienimy kontekst routingu dla ruchu **PC-A→PC-B (destination 198.51.100.10/32)**, a dla innych adresów, np. testowego ping PC-A→R2 10.12.0.2, pozostawimy zwykły lookup. To jest demonstracja destination-based FBF, nie source-based PBR z dwoma klientami.

Możesz potraktować routing instances jak dwie mapy dla tej samej sieci. W `inet.0` LAN-B prowadzi górą przez R2. W `FBF-ALT.inet.0` ten sam cel prowadzi do R3, czyli dolną gałęzią. Zależność od dodatkowej tablicy routingu pokażemy najpierw bez filter, a w późniejszej lekcji dopiero podłączymy ją do ingress R1.

### Czy routing instance to zawsze VRF?

Jeśli znasz Cisco VRF-Lite, pomysł osobnej tablicy jest naturalny. Nie przenoś jednak mechanicznie słowa „VRF” na każdy typ routing instance w Junos. Te konstrukcje mają różne role:

**\`virtual-router\`** jest bliższy klasycznemu VRF-Lite: tworzy oddzielny kontekst z interfejsami i możliwością konfiguracji routingu. W biurowym greenfield mógłby służyć do separacji OT, gości albo wydzielonej sieci klienta — jeśli platforma i projekt security na to pozwalają.

**\`vrf\`** odnosi się w Junos do bardziej wyspecjalizowanego scenariusza, m.in. L3VPN/PE, gdzie pojawiają się RD, RT i mechanizmy import/export tras VPN. Nie jest to po prostu bardziej elegancka nazwa \`virtual-router\`.

**\`forwarding\`** jest szczególnie przydatny przy **Filter-Based Forwarding (FBF)**. Chcemy mieć osobną tablicę do lookup wybranych pakietów, ale nie musimy przenosić całego interfejsu OFFICE do nowego VRF. Wejściowy filtr sam skieruje dopasowane pakiety do tej tablicy.

**\`no-forwarding\`** może przechowywać oddzielny kontekst informacji routingu, ale nie jest skrótem do „zrób jeszcze jedną forwarding table dla PBR”. To inny cel.

Uproszczenie „routing instance = VRF” pomaga rozpocząć naukę, ale przestaje działać, gdy próbujesz skonfigurować FBF. W naszym przypadku potrzebujemy **instance-type forwarding**.

### Najpierw tworzymy alternatywną mapę, jeszcze bez wysyłania do niej pakietów

Zbudujmy instancję **FBF-ALT**. Jej default route ma wskazywać R3 (dolny transit), czyli next-hop 10.13.0.2. W tym samym czasie \`inet.0\` nadal ma default przez R2 (górny transit).

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

Pytanie, które na tym etapie powinno się pojawić, brzmi: **skąd FBF-ALT ma wiedzieć, czym jest 10.13.0.2?** Jej default wskazuje ten next-hop, ale fizyczna sieć 10.13.0.0/30 należy do interfejsu R3 (dolny transit) skonfigurowanego w podstawowym kontekście. Samo zapisanie adresu next-hop w dodatkowej tabeli nie daje automatycznie wiedzy o jego osiągalności.

### Słynna pułapka: „trasa jest w tabeli, ale router nie wie, jak dojść do next-hopa”

Wyobraź sobie nawigację samochodową, która mówi: „Jedź do wjazdu operatora B”. Ale nie zawiera mapy osiedla, w którym znajduje się ten wjazd. Numer adresu jest podany, jednak nie ma sposobu dotarcia do niego.

W routingu taka sytuacja to problem **next-hop resolution**. Dla FBF-ALT możemy mieć zapis \`0.0.0.0/0 via 10.13.0.2\`, ale bez odpowiedniej connected route do 10.13.0.0/30 alternatywna tablica może nie umieć rozwiązać tego nexthopa. Obecność konfiguracji statycznej nie dowodzi, że forwarding będzie możliwy.

~~~mermaid
flowchart LR
  IN["R1: zwykły pakiet OFFICE"] --> M["inet.0<br/>0/0 via 10.12.0.2"]
  SPEC["R1: specjalny pakiet OFFICE"] --> B["FBF-ALT.inet.0<br/>0/0 via 10.13.0.2"]
  M --> A["connected 10.12.0.0/30<br/>R2 (górny transit)"]
  B --> Q{"Czy ta tabela zna<br/>10.13.0.0/30?"}
  Q -->|tak| OK["R3 (dolny transit)"]
  Q -->|nie| FAIL["Unresolved next-hop"]
~~~

*Rysunek 7.4 (autorski). Tekst alternatywny: globalna tablica umie rozwiązać next-hop A, a alternatywna FBF-ALT potrzebuje znajomości sieci connected do B; sam wpis default w drugiej tablicy nie wystarczy.*

To jest prawdziwa przyczyna, dla której w typowym scenariuszu FBF potrzebujemy jeszcze jednego mechanizmu. Nie jest nim kolejna static default i nie jest nim firewall filter. Potrzebujemy możliwości **udostępnienia wybranych informacji routingu między tablicami**. Do tego służą RIB groups w następnym kroku.

### Czy można po prostu przypisać R3 (dolny transit) do FBF-ALT?

W niektórych projektach dałoby się rozważyć przeniesienie interfejsu do innej instancji routingu, ale to nie jest to samo zadanie. Przy klasycznym FBF chcemy zachować fizyczne interfejsy w głównym kontekście, a **dla wybranych pakietów** użyć dodatkowej tabeli. Gdybyśmy przenieśli R3 (dolny transit) do oddzielnego virtual-router, zmienilibyśmy sposób, w jaki całe urządzenie traktuje ten interfejs, oraz jego dostępność dla innych usług.

Pamiętaj również o SRX: routing instance nie jest tym samym co **security zone**. Routing wybiera trasę; strefy i security policies decydują o dopuszczeniu ruchu, a NAT może przekształcić adresację. PBR/FBF na firewallu stateful wymaga weryfikacji współpracy z flow i NAT. Na klasycznym MX albo EX z reguły nie masz identycznego modelu sesji jak na vSRX. To ma ogromne znaczenie przy porównywaniu labu z konfiguracją produkcyjną.

*Źródła: [Routing Instances Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-instances-overview.html), [Filter-Based Forwarding Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-option-filter-based-forwarding-overview.html), [Juniper — Example Configuring FBF](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/example/filter-based-forwarding-example.html).*

---

## 7.4. RIB groups — jak dać drugiej tablicy mapę do bramy, ale nie cały firmowy routing [S]

### Dwie tablice, jedna informacja o bezpośrednim połączeniu

Wróćmy do problemu poprzedniej lekcji. \`inet.0\` zna R3 (dolny transit) jako połączenie bezpośrednie: **10.13.0.0/30 is directly connected**. Tablica \`FBF-ALT.inet.0\` chce wykorzystać next-hop 10.13.0.2, ale nie ma tej samej wiedzy o interfejsie.

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

W tym momencie domyślne interfejsowe prefiksy IPv4 mogą być udostępniane do secondary RIB. To potrzebny element następnego ćwiczenia FBF, ponieważ chcemy, by FBF-ALT potrafiła rozwiązać 10.13.0.2 przez R3 (dolny transit).

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

Powiedzmy, że FBF-ALT potrzebuje wyłącznie connected do R3 (dolny transit). Nie potrzebuje informacji o OFFICE, R2 (górny transit) i OOB. Chcielibyśmy zatem dopuścić do kopii tylko **10.13.0.0/30**.

**R1-vSRX — Configuration Mode [edit], wariant z ograniczeniem importu do secondary RIB**

~~~junos
set policy-options policy-statement RIB-ONLY-P2P13 term MATCH-P2P13 from route-filter 10.13.0.0/30 exact
set policy-options policy-statement RIB-ONLY-P2P13 term MATCH-P2P13 then accept
set policy-options policy-statement RIB-ONLY-P2P13 term REST then reject
set routing-options rib-groups COPY-INTERFACE-ROUTES import-policy RIB-ONLY-P2P13
~~~

Zwróć uwagę na nazwę **import-policy** w kontekście RIB group. To nie jest BGP import policy na sąsiedzie. Tu decydujemy, które wpisy mogą zostać skopiowane do tabel wtórnych. Pierwsza tabela pozostaje podstawowym kontekstem źródła; exact semantics sprawdzaj dla własnego release i typu źródła, a po zmianie koniecznie porównaj obie RIB. **Nie przyjmuj na wiarę, że „reject” usunął route z globalnej inet.0**.

W ogóle nie musisz używać tej dodatkowej policy, aby zrozumieć podstawowy FBF lab. Jest dobrym rozwinięciem dla urządzeń z wieloma interfejsami i wymagającą separacją.

### Dlaczego „kopię” w secondary można stracić, kiedy źródło znika?

Wyobraź sobie, że kabel R3 (dolny transit) zostaje odłączony. Trasa connected do 10.13.0.0/30 przestaje być aktywna w primary. Czy FBF-ALT może sobie powiedzieć: „Nie szkodzi, mam dawną kopię, będę forwardować dalej”? Nie byłoby to rozsądne. **RIB group nie tworzy niezależnej, nieśmiertelnej trasy**. Dokumentacja producenta opisuje, że usunięcie podstawowej trasy powoduje też usunięcie wtórnych wpisów zależnych od niej. Zniknie więc możliwość rozwiązania next-hopa R3 (dolny transit), a default w FBF-ALT może stać się nieużyteczna.

To łączy RIB group z normalnym życiem routingu. Nie rozwiązujemy awarii przez pamiętanie starych informacji. Rozwiązujemy ją przez korzystanie z aktualnych tras, nawet w wielu kontekstach.

**R1-vSRX — Operational Mode, gdy R3 (dolny transit) znika**

~~~junos
show interfaces terse
show route 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 0.0.0.0/0 exact extensive
show route hidden
~~~

Przy tym zestawie nie wystarczy informacja „R3 (dolny transit) is up” — potrzebujesz wiedzieć, czy właściwy logiczny unit ma adres i czy connected jest kwalifikowane w obydwu tabelach. Jeżeli static default istnieje w konfiguracji FBF-ALT, ale nie jest aktywna, problem może być w next-hop resolution, a nie w samym filtrze FBF.

### RIB group to nie „route leaking wszędzie”

To ważna granica analogii z Cisco VRF route leaking. RIB groups są jednym ze sposobów udostępniania informacji o trasach pomiędzy tablicami, ale w zależności od architektury mogą istnieć inne, bardziej odpowiednie rozwiązania: \`instance-import\`, BGP VPN route targets czy osobne protokoły pomiędzy instancjami. Nie mieszaj ich przypadkowo. W szczególności udostępnienie connected interfejsu do FBF-ALT **nie oznacza**, że pełna tablica BGP została skopiowana do nowego kontekstu albo że użytkownicy z dwóch security tenants mogą od tej chwili swobodnie wymieniać pakiety.

Jest też platformowy haczyk. Na części EX/QFX zachowanie RIB groups, typów instancji i kopiowania interfejsów może podlegać ograniczeniom lub innym regułom instalacji niż na MX. Dla produkcyjnego projektu sprawdź konkretne wspierane modele, Junos release oraz Feature Explorer. Ten sam blok konfiguracji nie jest uniwersalną gwarancją działania na dowolnym EX, MX i vSRX.

### Jak wygląda ta zależność na jednej mapie?

~~~mermaid
flowchart TD
  I["Interface route R3 (dolny transit)<br/>10.13.0.0/30"] --> P["Primary RIB: inet.0"]
  I --> RG["RIB group:<br/>COPY-INTERFACE-ROUTES"]
  RG --> S["Secondary: FBF-ALT.inet.0"]
  S --> NH["Rozwiązanie default<br/>via 10.13.0.2"]
  NH --> UP["R3 (dolny transit): prawidłowy next-hop"]
  I -. "awaria interfejsu / brak route" .-> F["Trasa źródłowa znika"]
  F -.-> S
~~~

*Rysunek 7.5 (autorski). Tekst alternatywny: jedna trasa interface R3 (dolny transit) trafia do primary inet.0 i przez RIB group również do FBF-ALT.inet.0; ta kopia umożliwia rozwiązanie default przez 10.13.0.2. Usunięcie źródła może usunąć także wpis wtórny.*

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


## 7.8. Laboratorium DIAMOND — aggregate, generated, RIB groups, FBF i ECMP [LAB]

**Status: PLANNED / NOT RUNTIME VALIDATED.** W rozdziale 7 używamy już wyłącznie kanonicznego DIAMOND: **R1, R2, R3, R4, PC-A i PC-B**. Nie ma Linux R2 ani R3. Wszystkie scenariusze przeprowadzaj na osobnej kopii labu z konsolą, a testy `then routing-instance` wykonuj tylko jeśli konkretny Junos/vSRX tę funkcję obsługuje. Nie wyłączaj flow-mode tylko po to, aby obejść ograniczenia FBF.

### Faza 0. Sprawdź fizyczną mapę

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

LAN-A: PC-A `192.0.2.10/24` ↔ R1 `192.0.2.1/24` na `ge-0/0/2.0`; LAN-B: R4 `198.51.100.1/24` na `ge-0/0/2.0` ↔ PC-B `198.51.100.10/24`. Transit R1–R2 `10.12.0.0/30`, R1–R3 `10.13.0.0/30`, R2–R4 `10.24.0.0/30`, R3–R4 `10.34.0.0/30`. Loopbacki R1–R4: `10.255.255.1/32` do `10.255.255.4/32`.

**R1-vSRX, R2-vSRX, R3-vSRX, R4-vSRX — każde urządzenie osobno, Operational Mode**

~~~junos
show version
show interfaces terse
show configuration interfaces | display set
show configuration routing-options | display set
show configuration routing-instances | display set
show configuration security zones | display set
show configuration firewall | display set
show security flow session
show system commit
~~~

Zweryfikuj faktyczne mapowanie NIC/QEMU, wersję obrazu, konsolę i zasoby EVE. Nie zmieniaj oryginalnego labu wykorzystywanego przez pozostałe rozdziały.

### Faza 1. Przywróć bazowy routing PC-A → R1 → R2 → R4 → PC-B

Na R1 primary do LAN-B prowadzi przez `10.12.0.2`, backup przez `10.13.0.2`. Na R4 return primary do LAN-A przez `10.24.0.1`, backup przez `10.34.0.1`. R2 i R3 znają statycznie oba LAN-y poprzez sąsiadów na swoich odcinkach. Security zones `LAB` i permit dla kontrolowanych testów ICMP muszą być przygotowane tak jak w rozdziale 6.

**R1-vSRX — Configuration Mode [edit private], tylko gdy brakuje baseline**

~~~junos
set routing-options static route 198.51.100.0/24 next-hop 10.12.0.2
set routing-options static route 198.51.100.0/24 qualified-next-hop 10.13.0.2 preference 15
show | compare
commit check
~~~

**R4-vSRX — Configuration Mode [edit private], tylko gdy brakuje baseline**

~~~junos
set routing-options static route 192.0.2.0/24 next-hop 10.24.0.1
set routing-options static route 192.0.2.0/24 qualified-next-hop 10.34.0.1 preference 15
show | compare
commit check
~~~

**PC-A — VPCS console**

~~~text
ip 192.0.2.10/24 192.0.2.1
ping 198.51.100.10
trace 198.51.100.10
~~~

**PC-B — VPCS console**

~~~text
ip 198.51.100.10/24 198.51.100.1
ping 192.0.2.10
~~~

Jeżeli nie ma poprawnego ruchu w obu kierunkach, nie zaczynaj dalszych faz: bez działającego baseline nie odróżnisz błędu RIB group od wcześniejszego braku trasy lub policy.

### Faza 2. Statyczne trasy contributor to nie nowe fizyczne sieci

Zdefiniuj **syntetyczne prefiksy** służące wyłącznie do badań RIB/FIB: `10.60.10.0/24` przez R2 i `10.60.20.0/24` przez R3. Żaden węzeł z naszego diamentu nie ma tych sieci jako direct, zatem **nie oczekuj** rzeczywistego ping do `10.60.10.10` bez osobnej, jawnie nazwanej rozszerzonej wersji ćwiczenia.

**R1-vSRX — Configuration Mode [edit private], kopia LAB**

~~~junos
set routing-options static route 10.60.10.0/24 next-hop 10.12.0.2
set routing-options static route 10.60.20.0/24 next-hop 10.13.0.2
show | compare
commit check
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route 10.60.10.0/24 exact extensive
show route 10.60.20.0/24 exact extensive
show route forwarding-table destination 10.60.10.10
~~~

Dwie trasy mogą być aktywne, bo next-hopy są rozwiązywalne, nawet jeśli żaden serwer pod takim adresem nie istnieje. To dowód route selection, a nie end-to-end reachability.

### Faza 3. Aggregate: podsumowanie i jego domyślny reject

**R1-vSRX — Configuration Mode [edit private]**

~~~junos
set routing-options aggregate route 10.60.0.0/16
show | compare
commit check
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route 10.60.0.0/16 exact extensive
show route 10.60.10.0/24 exact
show route forwarding-table destination 10.60.99.25
~~~

Jeżeli contributor /24 jest aktywny, aggregate /16 może stać się aktywny. Dla adresu z prawdziwej /24 wygrywa specyficzna trasa. Dla nieobsadzonego `10.60.99.25` może wygrać aggregate reject. Nie myl reject w FIB z odrzuceniem BGP import policy.

### Faza 4. Generated: route z next-hop contributor

**R1-vSRX — Configuration Mode [edit private]**

~~~junos
set routing-options static route 10.70.10.0/24 next-hop 10.12.0.2
set routing-options generate route 10.70.0.0/16
show | compare
commit check
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route 10.70.0.0/16 exact extensive
show route 10.70.10.0/24 exact
show route forwarding-table destination 10.70.99.25
~~~

Zamiast reject generated może przekazać pakiet przez next-hop primary contributor R2. Pamiętaj: docelowy host `10.70.99.25` jest hipotetyczny; nie jest nowym urządzeniem EVE.

### Faza 5. FBF-ALT: ta sama sieć docelowa, inna tabela

Zbuduj drugą RIB na R1, gdzie LAN-B wskazuje R3, podczas gdy zwykła `inet.0` nadal wskazuje R2. Dla `FBF-ALT` next-hop `10.13.0.2` musi być **osiągalny w tej samej tabeli**.

**R1-vSRX — Configuration Mode [edit private], jeśli FBF wspierane**

~~~junos
set routing-instances FBF-ALT instance-type forwarding
set routing-instances FBF-ALT routing-options static route 198.51.100.0/24 next-hop 10.13.0.2
show | compare
commit check
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route table inet.0 198.51.100.0/24 exact
show route table FBF-ALT.inet.0 198.51.100.0/24 exact extensive
~~~

Druga tabela jeszcze może mieć unresolved next-hop. Poprawny wpis w konfiguracji nie jest dowodem aktywnego routingu.

### Faza 6. RIB group: celowo udostępnij connected do R3

To jest eksperyment z **rozwiązywaniem next-hop w secondary RIB**, a nie przenoszeniem interfejsu między VRF. Import-policy powinna selektywnie przepuścić connected do P2P-13 `10.13.0.0/30`. Sposób ewaluacji reguł sprawdź na faktycznej platformie; nie zakładaj, że samo commit check dowodzi funkcjonalności.

**R1-vSRX — Configuration Mode [edit private]**

~~~junos
set policy-options policy-statement CH07-P2P13 term P2P-13 from route-filter 10.13.0.0/30 exact
set policy-options policy-statement CH07-P2P13 term P2P-13 then accept
set policy-options policy-statement CH07-P2P13 term REST then reject
set routing-options rib-groups CH07-RIB import-rib [ inet.0 FBF-ALT.inet.0 ]
set routing-options rib-groups CH07-RIB import-policy CH07-P2P13
set routing-options interface-routes rib-group inet CH07-RIB
show | compare
commit check
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route table inet.0 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 198.51.100.0/24 exact extensive
~~~

Jeżeli connected nie dociera do dodatkowej RIB albo default/static pozostaje hidden, wyjaśnij powód na podstawie show route extensive zamiast dodawać przypadkowe next-table.

### Faza 7. Włącz filtr ingress — tylko PC-B przez R3

W sześciowęzłowym DIAMOND nie mamy dwóch klientów w LAN-A. Dlatego uczymy **destination-based FBF**: pakiet PC-A do PC-B `198.51.100.10` trafi do `FBF-ALT`, lecz pakiet PC-A do R2 `10.12.0.2` pozostanie w `inet.0`. Nie przedstawiaj tego jako testu dwóch źródeł PBR.

**R1-vSRX — Configuration Mode [edit private], tylko gdy obraz wspiera FBF**

~~~junos
set firewall family inet filter OFFICE-FBF term SPECIAL from destination-address 198.51.100.10/32
set firewall family inet filter OFFICE-FBF term SPECIAL then routing-instance FBF-ALT
set firewall family inet filter OFFICE-FBF term NORMAL then accept
set interfaces ge-0/0/2 unit 0 family inet filter input OFFICE-FBF
show | compare
commit check
commit confirmed 10
~~~

**PC-A — VPCS console**

~~~text
ping 10.12.0.2
ping 198.51.100.10
trace 198.51.100.10
~~~

**R1-vSRX — Operational Mode**

~~~junos
show configuration interfaces ge-0/0/2 | display set
show firewall filter OFFICE-FBF
show route table FBF-ALT.inet.0 198.51.100.0/24 exact
show security flow session
~~~

Jeśli PC-A dociera do R2, termin NORMAL nie zablokował innych celów. Przekierowanie do R3 potwierdź przez stan FIB/flow i obserwację pakietów, **nie tylko zielony ping**.

### Faza 8. Najważniejsza pułapka: R4 może odesłać przez R2

FBF może wymusić request PC-A → R1 → R3 → R4, ale bazowy R4 nadal preferuje R2 w drodze zwrotnej. SRX jest stateful, więc asymetryczna ścieżka może powodować drop. Dopiero **osobny, świadomy wariant** z tymczasowym return via R3 pozwala sensownie testować end-to-end. Po teście obowiązkowo odtwórz baseline R4.

**R4-vSRX — Configuration Mode [edit private], tylko kopia LAB**

~~~junos
delete routing-options static route 192.0.2.0/24
set routing-options static route 192.0.2.0/24 next-hop 10.34.0.1
show | compare
commit check
commit confirmed 10
~~~

**R4-vSRX — Operational Mode**

~~~junos
show route 192.0.2.0/24 exact extensive
show security flow session
~~~

**PC-A — VPCS console**

~~~text
ping 198.51.100.10
trace 198.51.100.10
~~~

Brak odpowiedzi mimo trasy przez R3 nie musi oznaczać błędnego FBF: sprawdź policies i sesje na R1/R3/R4. Nie potwierdzaj zmiany R4, jeśli test ma się sam wycofać.

### Faza 9. Controlled fault: aggregate bez contributor

**R1-vSRX — Configuration Mode [edit private]**

~~~junos
delete routing-options static route 10.60.10.0/24
delete routing-options static route 10.60.20.0/24
show | compare
commit check
commit confirmed 2
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route 10.60.0.0/16 exact extensive
show configuration routing-options aggregate | display set
show system commit
~~~

Zauważ różnicę między obecnością `set aggregate` w candidate/active a rzeczywistą aktywnością podsumowania w RIB. Po auto-rollback sprawdź, czy contributory wróciły.

### Faza 10. Controlled fault: RIB group traci connected

**R1-vSRX — Configuration Mode [edit private]**

~~~junos
delete routing-options interface-routes rib-group inet CH07-RIB
show | compare
commit check
commit confirmed 2
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route table FBF-ALT.inet.0 10.13.0.0/30 exact
show route table FBF-ALT.inet.0 198.51.100.0/24 exact extensive
~~~

Jeżeli trasa do R3 zniknęła tylko z secondary table, diagnozą jest brak next-hop resolution w FBF-ALT — nie link down w EVE.

### Faza 11. Controlled fault: usuwamy NORMAL z filtra

**R1-vSRX — Configuration Mode [edit private], tylko osobna konsola**

~~~junos
delete firewall family inet filter OFFICE-FBF term NORMAL
show | compare
commit check
commit confirmed 2
~~~

**PC-A — VPCS console**

~~~text
ping 10.12.0.2
ping 198.51.100.10
~~~

Jeżeli ping do R2 przestaje działać, a interfejs R1–R2 jest up, możliwą przyczyną jest implicit discard innych destination po usunięciu końcowego termu. Po auto-rollback sprawdź licznik/konfigurację filtra.

### Faza 12. Generated vs aggregate: forwarding może być inny mimo podobnego /16

**R1-vSRX — Operational Mode**

~~~junos
show route 10.60.0.0/16 exact extensive
show route 10.70.0.0/16 exact extensive
show route forwarding-table destination 10.60.99.25
show route forwarding-table destination 10.70.99.25
~~~

Różne next-hopy mówią, czy router odrzuca prefix aggregate, czy kieruje pakiet zgodnie z generated contributor. Nie wymagaj odpowiedzi od nieistniejących hostów.

### Faza 13. ECMP oddzielnie od FBF

Najpierw wycofaj FBF z ingress. Na R1 przetestuj równorzędne next-hopy do LAN-B — bez jednoczesnej zmiany source policy. W R4 musi nadal istnieć poprawny return path. Jeden ping z PC-A to **jeden flow**, nie statystyka dystrybucji.

**R1-vSRX — Configuration Mode [edit private], osobny wariant ECMP**

~~~junos
delete interfaces ge-0/0/2 unit 0 family inet filter input OFFICE-FBF
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
commit confirmed 2
~~~

**R1-vSRX — Operational Mode**

~~~junos
show route 198.51.100.0/24 exact extensive
show route forwarding-table destination 198.51.100.10
~~~

Jeżeli FIB ma obydwa next-hopy, a ICMP wybiera tylko R2, per-flow hash może być prawidłowy. Dodatkowe flows wymagają odrębnie udokumentowanego generatora — nie zmieniaj kanonicznych VPCS na fikcyjne PC-C.

### Faza 14. Martians, cleanup i evidence

Nie używaj martians jako domyślnego wyjaśnienia dla każdego failed ping. Martian addresses to mechanizm kwalifikacji tras, nie synonim filtracji pakietu przez SRX.

**R1-vSRX — Operational Mode**

~~~junos
show route martians
show route table inet.0
show route table FBF-ALT.inet.0
show configuration firewall | display set
show configuration routing-options | display set
show security flow session
show system commit
~~~

Odtwórz kopię DIAMOND z `R1→LAN-B via R2, backup via R3` oraz `R4→LAN-A via R2, backup via R3`, bez testowych aggregate, generated, FBF i ECMP. Po każdym `commit confirmed` nie wykonuj kolejnego `commit check` przed świadomym potwierdzeniem lub rollbackiem. Zapisz w `VERIFY.md` mapping portów, wersję Junos, RIB/FIB, testy obu VPCS i kontrolowane awarie. Prefiksy `10.60.0.0/16` / `10.70.0.0/16` są wyłącznie dodatkowymi wpisami testowymi; nie zmieniają schematu sześciu połączeń.

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

**Pytanie pierwsze.** R1 ma aktywne 10.60.20.0/24, ale R2 (górny transit) nie działa. Czy 10.60.0.0/16 aggregate może nadal być aktywny? **Tak**, jeśli istnieje choć jeden kwalifikujący się contributor i nie ma dodatkowych ograniczeń. Nie wynika z tego, że wszystkie podsieci wewnątrz /16 są osiągalne.

**Pytanie drugie.** Dlaczego \`show route protocol aggregate\` pokazuje 10.70.0.0/16, chociaż skonfigurowałeś \`generate\`? Ponieważ w Junos typ wyświetlany w RIB i jego domyślna preference dla aggregate/generated mogą być podobne. Rozróżnij je po konfiguracji i next-hop.

**Pytanie trzecie.** \`FBF-ALT.inet.0\` ma skonfigurowaną default przez 198.51.100.6. Czy na pewno przekazuje ruch? **Nie**: sprawdź, czy w tym kontekście istnieje droga do samego next-hopa, czy jest aktywna, a potem czy pakiet został skierowany do tej tabeli przez ingress filter.

**Pytanie czwarte.** Dlaczego po usunięciu \`NORMAL then accept\` nie działa PC-B, mimo że zwykła globalna default jest nadal aktywna? Bo stateless input filter może odrzucić pakiet niedopasowany do żadnego termu, zanim zwykła droga będzie dla niego użyteczna.

**Pytanie piąte.** W FIB widzisz dwa next-hopy, ale jeden transfer TCP korzysta z R2 (górny transit). Czy ECMP jest uszkodzony? **Niekoniecznie**: per-flow hashing może przypisać cały przepływ do jednego next-hopa. Rozkład trzeba badać na większej liczbie niezależnych flows.

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
