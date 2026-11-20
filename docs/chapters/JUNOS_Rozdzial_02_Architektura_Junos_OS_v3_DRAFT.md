# Rozdział II. Jak działa Junos OS — od pierwszego pakietu do Routing Engine i Packet Forwarding Engine

**JUNOS Enterprise Routing & Switching | Rozdział 2: Junos OS Fundamentals | zakres blueprintu 2.1–2.6 [A]**

**Wydanie redakcyjne:** 3.0 — 20 listopada 2026  
**Odbiorca:** Network Engineer z około rocznym doświadczeniem, znający podstawowe pojęcia IPv4, routera, switcha i bramy domyślnej. **Nie zakładamy ukończenia CCNA, JNCIA ani znajomości Cisco.**  
**Cel:** przygotować zrozumienie Junos od zera, tak aby późniejsze rozdziały o CLI, routingu, OSPF, IS-IS, BGP i filtrach miały solidne podstawy.  
**Laboratorium:** [DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md) — rozpoczniemy od dwóch, następnie trzech routerów, a dopiero na końcu wykorzystamy wszystkie sześć. **Status: PLANNED / NOT RUNTIME VALIDATED.** Konfiguracje i wyniki są materiałem do samodzielnego wykonania, nie zrzutem działających urządzeń.

---

## Zanim zajmiemy się systemem operacyjnym: co robi router?

Wyobraź sobie, że masz dwa biura połączone routerem. Komputer w pierwszym biurze chce otworzyć stronę znajdującą się w drugim. Komputer nie musi znać całej drogi. Powinien jednak wiedzieć, czy adres odbiorcy należy do jego własnej sieci. Jeżeli nie, wysyła pakiet do **bramy domyślnej** — adresu routera, który zna dalszą drogę.

Na jednym kablu do routera mogą przychodzić pakiety od setek komputerów. Router ma szybko odpowiedzieć na bardzo proste pytanie: **dokąd przekazać ten konkretny pakiet?** Jednocześnie musi robić rzeczy mniej oczywiste. Administrator zmienia konfigurację, połączenie przestaje działać, protokół routingu zgłasza nową drogę, interfejs traci sygnał. Router musi zauważyć te zdarzenia, zaktualizować swoje informacje i dopiero wtedy przekazywać następne pakiety odpowiednią trasą.

Właśnie dlatego warto rozumieć architekturę Junos. Gdy wszystko działa, jej elementy są prawie niewidoczne. Podczas awarii rozróżnienie „router zna drogę” od „router faktycznie przekazuje pakiet” decyduje o tym, czy naprawisz problem w dziesięć minut, czy spędzisz godzinę na sprawdzaniu nie tej funkcji.

Zanim przeczytasz dalej, przyjmijmy trzy określenia:

- **Trasa (route)** to informacja, do jakiej sieci można dotrzeć i jaki następny krok jest potrzebny.
- **Pakiet (packet)** to konkretna porcja danych z adresem źródłowym i docelowym, którą router ma przetworzyć.
- **Interfejs (interface)** to logiczny lub fizyczny punkt wejścia i wyjścia ruchu — na przykład port Ethernet routera.

Nie musisz jeszcze wiedzieć, jak działa OSPF czy BGP. Nauczymy się tego później. Dzisiaj chcemy zobaczyć, **jak Junos organizuje pracę urządzenia**, zanim zaczniemy go szczegółowo konfigurować.

### Nasze laboratorium nie zmienia się między rozdziałami

Docelowo mamy sześć routerów w układzie DIAMOND-6R. R5 i R6 to routery na krańcach sieci, R1 i R4 są na jej brzegach, a R2 i R3 oferują dwie alternatywne drogi przez rdzeń.

~~~mermaid
flowchart LR
  R5["R5 · CE-A<br/>192.0.2.10"] ---|"LAN-A<br/>192.0.2.0/24"| R1["R1 · EDGE-A<br/>192.0.2.1"]
  R1 ---|"10.12.0.0/30"| R2["R2 · CORE-1"]
  R1 ---|"10.13.0.0/30"| R3["R3 · CORE-2"]
  R2 ---|"10.24.0.0/30"| R4["R4 · EDGE-B<br/>198.51.100.1"]
  R3 ---|"10.34.0.0/30"| R4
  R4 ---|"LAN-B<br/>198.51.100.0/24"| R6["R6 · CE-B<br/>198.51.100.10"]
~~~

*Rysunek 2.1. Pełna topologia DIAMOND-6R. Górna droga: R5–R1–R2–R4–R6. Dolna: R5–R1–R3–R4–R6. Między R2 i R3 oraz R1 i R4 nie ma połączeń. Numeracja konkretnych portów i oba adresy każdego segmentu są w [referencji laboratorium](../../labs/reference-diamond-6vsrx/README.md).*

Na początku wykorzystamy tylko połączenie **R5–R1** do nauki ruchu skierowanego do samego routera. Później dołożymy **R2**, aby zobaczyć prawdziwy tranzyt przez R1. Dopiero gdy to zrozumiesz, uruchomimy dwa korytarze całego diamentu. Sześć urządzeń od początku nie powinno oznaczać sześciu niezrozumiałych konfiguracji.

---

## 2.1. Architektura oprogramowania Junos OS [A]

### Dlaczego router nie może za każdym razem „pytać systemu o drogę”?

Załóżmy, że R1 dostał pakiet do 10.12.0.2 — adresu R2, z którym jest bezpośrednio połączony. Żeby przekazać ten pakiet, R1 musi sprawdzić, czy zna sieć docelową i którym portem ma wysłać dane.

Teraz wyobraź sobie dziesięć tysięcy takich pakietów. Czy za każdym razem router powinien od nowa sprawdzać wszystkie protokoły routingu, odpytywać sąsiadów i zastanawiać się nad najlepszą drogą? Nie. Gdyby tak było, samo podejmowanie decyzji zajmowałoby więcej czasu niż przesyłanie danych.

Junos rozdziela zatem dwie prace: **ustalanie, co router wie o sieci**, oraz **wykonywanie przygotowanych decyzji dla nadchodzących pakietów**. W architekturze urządzeń Juniper nadajemy im dwie nazwy.

**Routing Engine (RE)** utrzymuje informacje potrzebne do działania routera: konfigurację, protokoły routingu, kontrolę systemu i tablice tras. Na nim pracują procesy, które dowiadują się, jakie sieci istnieją i jak najlepiej do nich dotrzeć.

**Packet Forwarding Engine (PFE)** realizuje przekazywanie pakietów zgodnie z przygotowaną informacją forwarding. Na fizycznym routerze lub switchu mogą to robić wyspecjalizowane układy. W vSRX korzystamy z wirtualnego, programowego środowiska, więc nie powinniśmy wyobrażać sobie osobnej, fizycznej karty PFE wewnątrz wirtualnej maszyny.

Najprostszy sens tego podziału jest taki: RE **uczy się i utrzymuje decyzje**, a PFE **stosuje je do zwykłego ruchu**. Oczywiście podczas rzeczywistego forwardingu mogą zachodzić dodatkowe operacje, takie jak rozwiązywanie adresów warstwy drugiej czy filtrowanie. Dla SRX dochodzi także kontrola stanu sesji. Ale najpierw opanujmy sam podział zadań.

~~~mermaid
flowchart TB
   A["Administrator oraz informacje od sąsiadów sieciowych"] --> RE
   subgraph RE["Routing Engine — control plane i zarządzanie"]
      RPD["rpd: wiedza o trasach"]
      MGD["mgd: obsługa konfiguracji"]
      RIB["RIB: dostępne drogi do sieci"]
      RPD --> RIB
   end
   RIB --> K["Systemowa tablica forwarding<br/>i synchronizacja"]
   K --> PFE["PFE: przekazywanie ruchu"]
   PFE --> I["Interfejs wyjściowy"]
   P["Pakiety przychodzące z sieci"] --> PFE
~~~

*Rysunek 2.2. Uproszczona architektura logiczna. Nie każdy pakiet przechodzi przez RE. RE przygotowuje wiedzę sterującą, a PFE wykorzystuje przygotowaną tablicę do większości ruchu tranzytowego. Schemat nie przedstawia wszystkich etapów przetwarzania SRX security flow.*

### Jak wygląda ta praca w normalnym życiu urządzenia?

Najpierw uruchamiasz router. Junos rozpoznaje interfejsy, wczytuje zatwierdzoną konfigurację i uruchamia potrzebne usługi systemowe. Jeśli interfejs R1 ma adres 10.12.0.1/30 i jest aktywny, router potrafi stwierdzić, że sieć **10.12.0.0/30** jest bezpośrednio połączona. Do zdobycia tej wiedzy nie potrzebuje BGP ani OSPF.

Ta informacja trafia do tablicy routingu. System ustala, jakie szczegóły są niezbędne do przekazania pakietu do R2. Gdy forwarding ma już właściwą informację o porcie i kolejnym kroku, kolejne pakiety do tego celu mogą być przekazywane bez ponownego uruchamiania całej logiki routingu.

Gdy kabel R1–R2 zostanie odłączony, sytuacja zmienia się. Urządzenie musi zauważyć utratę interfejsu, a informacje o trasach mogą zostać zmienione lub usunięte. Wtedy odpowiednie aktualizacje muszą dotrzeć również do forwardingu. Rozdzielenie RE i PFE nie oznacza, że oba elementy żyją niezależnie. **Wymieniają informacje**, aby pakiety korzystały z aktualnych decyzji.

### Junos to zbiór współpracujących procesów

W systemie Windows lub Linux różne programy zajmują się różnymi zadaniami. Podobnie działa Junos. Wiele jego funkcji realizują wyspecjalizowane procesy, często nazywane *daemons*. Na początku warto poznać tylko te, które będziemy spotykać podczas diagnostyki.

**\`rpd\` — routing protocol process.** Utrzymuje informacje związane z routingiem i jego protokołami. Kiedy za kilka rozdziałów uruchomimy OSPF i BGP, będziemy wiedzieli, że nie są to niezależne programy obsługujące każdy pakiet, lecz mechanizmy dostarczające informacji dla routingu.

**\`mgd\` — management process.** Odpowiada za obsługę poleceń administratora i konfiguracji. Junos ma ważną cechę: zanim wprowadzisz wiele zmian do działającej konfiguracji, przygotowujesz je w wersji kandydackiej i świadomie zatwierdzasz. Szczegółowo opanujemy to w rozdziale III.

**\`chassisd\` — chassis daemon.** Zajmuje się elementami platformy, na przykład stanem części sprzętowych i powiązanymi alarmami. Na wirtualnym vSRX jego funkcjonalność i widoczne obiekty mogą być inne niż na fizycznym MX czy EX.

**\`dcd\` — device control daemon.** Jest związany z zarządzaniem interfejsami. W różnych rodzinach i wydaniach Junos zestaw usług oraz ich podział mogą się różnić, ale dla początkującego najważniejsze pozostaje to, że zarządzanie interfejsami nie jest tożsame z liczeniem tras.

**Kernel** to jądro systemu operacyjnego — część pośrednicząca m.in. pomiędzy procesami, pamięcią, urządzeniami i systemowymi strukturami forwardingu. W klasycznym Junos historycznie podstawą jest FreeBSD. **Junos OS Evolved** wykorzystuje jądro Linux i inną architekturę uruchamiania aplikacji. Nie oznacza to jednak, że wszystkie polecenia Junos i funkcje systemu są zupełnie odmienne.

Nie ucz się tych nazw jako losowej listy. Powiąż każdą z pytaniem: kto utrzymuje informacje o trasach? — rpd. Kto obsługuje konfigurację? — mgd. Kto kontroluje istotne aspekty chassis? — chassisd. Dzięki temu pierwsze polecenia diagnostyczne zaczną mieć sens.

### Co zmienia fakt, że uczymy się na vSRX?

W laboratorium wszystkie R1–R6 są **vSRX**, czyli wirtualnymi zaporami sieciowymi Juniper, które potrafią także routować pakiety. To bardzo wygodne do nauki CLI, tras i protokołów, ale niesie dodatkową cechę: SRX w typowym trybie *flow-based* kontroluje również **sesje ruchu**, security zones i security policies. Dlatego sam poprawny routing nie musi oznaczać, że ping między routerami przejdzie.

Na fizycznym MX skupiamy się na RE i sprzętowym forwarding. Na EX/QFX RE współpracuje z odpowiednim układem switching/forwarding. Na vSRX funkcje te są realizowane programowo i dochodzi security flow. **Wspólny jest model logiczny; sposób wykonania zależy od platformy.** Nie używaj jednej komendy diagnostycznej PFE z fizycznego MX jako gwarantowanej komendy dla każdej wersji vSRX.

Jeżeli dopiero zaczynasz Junos, nie musisz teraz znać wszystkich modeli Juniper. Na tym etapie wystarczy zrozumieć, że Junos występuje na urządzeniach o bardzo różnych rolach, ale jego zasady administracji i routingu pozostają w dużej mierze wspólne.

**Pierwsze pytanie kontrolne.** R1 potrafi wymienić pakiety z R2, ale nie można zatwierdzić nowej konfiguracji. Czy mamy wystarczający dowód, że PFE jest uszkodzony? Nie. Problemy z obsługą konfiguracji i problemy z przekazywaniem pakietów należą do różnych obszarów działania. Trzeba zbadać je osobno.

**Źródła Juniper:** [Junos OS Architecture Overview](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-architecture.html), [Routing Engine Components and Processes](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-components-and-processes.html), [Junos OS Evolved Differences](https://www.juniper.net/documentation/us/en/software/junos/overview-evo/topics/concept/evo-top-differences.html).


---

## 2.2. Control plane i forwarding plane — dwie różne odpowiedzi na problem sieciowy [A]

### Skąd router wie, gdzie jest sieć, a skąd wie, jak wysłać następny pakiet?

Wróćmy do R1 i R2. Łączy je sieć **10.12.0.0/30**. R1 ma adres 10.12.0.1, a R2 — 10.12.0.2. To jedno fizyczne połączenie między dwoma urządzeniami.

Kiedy administrator nada R1 adres 10.12.0.1/30, router wie, że adres 10.12.0.2 znajduje się na tej samej podsieci. Później, gdy R5 wyśle do R2 pakiet przez R1, R1 będzie mógł wybrać port prowadzący do R2.

Zauważ, że występują tu dwa różne momenty. W pierwszym **powstaje wiedza o drodze**. W drugim **przychodzi pakiet i trzeba tej wiedzy użyć**. To właśnie granica między control plane a forwarding plane.

**Control plane** to funkcje sterujące routerem, czyli między innymi uczenie się tras, ustalanie sąsiedztw, wykrywanie zmian topologii i przygotowywanie odpowiednich tablic. W prostym labie wystarczy adres interfejsu. W dużej sieci informacje mogą pochodzić z OSPF lub BGP.

**Forwarding plane**, nazywany również *data plane*, odpowiada za obsługę pakietów wchodzących na porty, wyszukiwanie drogi w odpowiedniej tablicy oraz ich przekazywanie na port wyjściowy. Na SRX towarzyszą temu operacje bezpieczeństwa i obsługa sesji, których dokładnej kolejności nie musisz jeszcze znać.

Możemy wydzielić także **management plane**: dostęp SSH, konfigurację, logowanie administratorów, monitoring, zarządzanie oprogramowaniem. To funkcjonalnie odrębny obszar, choć na wielu platformach jest wykonywany przez procesy RE. Nie traktuj go jako trzeciego, identycznego sprzętowo silnika obok RE i PFE.

### Dlaczego ping do routera nie dowodzi, że użytkownicy mogą przez niego pracować?

Załóżmy, że R5 potrafi wykonać ping do R1, na adres **192.0.2.1**. To dobra wiadomość, ale bardzo ograniczona. Ping dotarł do **samego R1** i ten odpowiedział.

Jeśli R5 próbuje pingować **10.12.0.2**, czyli R2, sytuacja jest inna. Pakiet najpierw trafia na R1, ale jego celem **nie jest R1**. R1 musi przekazać go dalej. Nawet gdy posiada prawidłową trasę, SRX może odmówić przekazania, jeżeli dla tej komunikacji nie ma odpowiedniej polityki bezpieczeństwa.

Zatem dwa bardzo podobne testy odpowiadają na dwa różne pytania:

| Test | O czym mówi? | Czego jeszcze nie dowodzi? |
|---|---|---|
| R5 → 192.0.2.1 | R1 odpowiada na ICMP do własnego adresu, z danej ścieżki | Że R1 przekazuje ruch do R2 |
| R5 → 10.12.0.2 | R1 może przekazać pakiet do R2, a odpowiedź może wrócić | Że wszystkie inne sieci i aplikacje działają |
| SSH z administratora do R1 | Działa konkretna ścieżka zarządzania oraz odpowiednia usługa | Że forwarding ruchu użytkowników jest sprawny |

Uczymy się więc ważnej zasady: **wynik testu musi odpowiadać dokładnie na pytanie, które zadałeś**. Jeżeli testowałeś tylko dostęp do własnego adresu routera, nie potwierdziłeś tym ruchu tranzytowego.

### A co, jeśli Routing Engine pracuje z wysokim obciążeniem?

Może się zdarzyć, że proces \`rpd\` zużywa dużo CPU, ponieważ w sieci zachodzą liczne zmiany tras. Nie oznacza to automatycznie, że każdy już przekazywany pakiet zatrzymuje się i czeka na obliczenia RE. Na fizycznych urządzeniach z odrębnym PFE część forwarding może działać dalej z wcześniej przygotowaną tablicą.

Ale jest ważna granica tego wniosku. Jeśli RE nie nadąża z przeliczaniem nowych dróg lub aktualizowaniem forwardingu, sieć może korzystać z **nieaktualnych informacji**, a nowe sesje i sytuacje wyjątkowe mogą zachowywać się inaczej. Na vSRX, gdzie wiele funkcji działa programowo w ramach jednego środowiska, wysokie użycie CPU przez maszynę wirtualną może też bezpośrednio pogorszyć forwarding.

Dlatego zdanie „control plane i forwarding plane są rozdzielone” nie jest gwarancją, że usterka jednego nigdy nie wpłynie na drugi. To model architektoniczny, który pomaga rozpoznać rodzaj problemu.

### Czy proces sterujący uczestniczy w każdej wymianie danych?

Nie. Router nie pyta \`rpd\` o każdą następną porcję danych TCP. Protokół routingu i RIB dostarczają informację, która jest używana w przygotowanej tablicy forwardingu. Dla zwykłego, znanego ruchu szybka obsługa odbywa się poza takim pełnym cyklem obliczania tras.

Natomiast pakiet skierowany do **samego routera** — na przykład wiadomość do lokalnej usługi SSH albo właściwy pakiet sterujący OSPF — musi trafić do części systemu, która potrafi go obsłużyć. Również część pakietów wyjątkowych może wymagać specjalnej obsługi przez RE. Nie utożsamiaj tego z wysyłaniem każdego pakietu do CPU.

~~~mermaid
flowchart LR
    IN["Pakiet wchodzi na R1"] --> D{"Jaki jest cel pakietu?"}
    D -->|"Cel jest dalej w sieci"| F["Forwarding plane:<br/>szukanie drogi i dalsze przetwarzanie"]
    D -->|"Cel to R1 lub wyjątek"| C["Lokalna obsługa:<br/>RE / właściwy proces"]
    F --> OUT["Interfejs w stronę R2"]
    C --> S["Odpowiedź lokalna, aktualizacja stanu<br/>lub odrzucenie"]
~~~

*Rysunek 2.3. Dwie logicznie różne drogi obsługi. Na vSRX przed końcowym rezultatem może zadziałać również security flow. Nie jest to ścisły diagram kolejności operacji dla każdego PFE.*

**Krótka sytuacja do rozważenia.** Ping R5 do 192.0.2.1 działa, ale ping R5 do 10.12.0.2 nie. Czy od razu restartujesz proces \`rpd\` na R1? Nie. Najpierw sprawdzasz, czy R5 ma trasę przez R1, czy R1 ma trasę do R2, czy R2 zna drogę powrotną do R5 i czy security policy R1 dopuszcza ruch tranzytowy. Każdy z tych warunków można sprawdzić bez restartu usługi.

**Źródła Juniper:** [Junos OS Architecture Overview](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-architecture.html), [Router Data Flow Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-router-data-flow-overview.html).

---

## 2.3. Routing Engine — miejsce, w którym router poznaje drogi i zarządza sobą [A]

### Zacznijmy od pytania: skąd R1 bierze trasę do R2?

Widzieliśmy już, że R1 ma bezpośrednie połączenie z R2. Z adresu interfejsu **10.12.0.1/30** Junos wyprowadza informację, że sieć **10.12.0.0/30** jest dostępna bezpośrednio. Tę informację przechowuje w **Routing Information Base (RIB)**, czyli bazie informacji o trasach. Na początku możesz myśleć o RIB po prostu jako o tablicy, w której router zapisuje znane drogi do sieci.

W Junos domyślna tablica routingu dla IPv4 unicast nazywa się **\`inet.0\`**. To nazwa, którą bardzo często zobaczysz po wpisaniu \`show route\`.

Przyjrzyjmy się trzem podstawowym rodzajom tras, które możemy spotkać już na starcie.

**Direct** oznacza sieć, z którą router jest bezpośrednio połączony przez odpowiednio skonfigurowany interfejs. Dla R1 to na przykład 10.12.0.0/30.

**Local** oznacza adres przypisany do samego routera. Na R1 istnieje lokalny adres 10.12.0.1/32; pakiet zaadresowany dokładnie do tego IP nie powinien być potraktowany jak zwykły pakiet do R2. Mamy więc osobno informację o całej podsieci i o własnym adresie urządzenia.

**Static** oznacza trasę, którą administrator skonfigurował ręcznie, na przykład „do sieci 172.16.60.0/24 idź do następnego routera 10.12.0.2”. W przeciwieństwie do direct Junos nie wyprowadzi jej automatycznie z adresu interfejsu.

Kiedy później uruchomimy OSPF czy BGP, pojawią się kolejne źródła informacji. Ale mechanizm odczytywania tablic pozostanie znajomy.

### Jak odczytać pierwszą trasę, jeśli nigdy nie pracowałeś na Junos?

W Junos są dwa podstawowe tryby interfejsu tekstowego. **Operational Mode** służy do odczytywania stanu i wykonywania diagnostyki; typowy prompt kończy się znakiem \`>\`. **Configuration Mode** służy do przygotowywania zmian i zwykle ma prompt kończący się \`#\`. Pełną naukę obu trybów dostaniesz w rozdziale III. Na razie wszystkie polecenia \`show\` poniżej wykonujesz w Operational Mode.

**R1-vSRX — Operational Mode, podstawowy odczyt tras**

~~~junos
show route
show route table inet.0
show route 10.12.0.0/30 exact
show route 10.12.0.1/32 exact
show interfaces terse
~~~

Pierwsze polecenie wyświetla informacje routingu. Drugie wskazuje konkretną tabelę IPv4. Trzecie szuka dokładnie sieci 10.12.0.0/30, a czwarte sprawdza lokalny adres R1 jako /32. Ostatnie pokazuje stan interfejsów, abyśmy mogli powiązać trasę z rzeczywiście skonfigurowanym portem.

Zwróć uwagę na słowo \`exact\`. W dokumentacji będziemy czasem pytać o konkretny prefix, nie o dowolny pasujący wpis. Dzięki \`exact\` wynik nie zostanie pomylony z inną, mniej lub bardziej szczegółową trasą.

Na poprawnie skonfigurowanym R1 **oczekujemy** informacji podobnej do poniższej:

~~~text
inet.0:
10.12.0.0/30       *[Direct/0] ... via ge-0/0/0.0
10.12.0.1/32       *[Local/0] ... Local via ge-0/0/0.0
~~~

*Przykład ilustruje najważniejsze pola; celowo pominięto część metadanych, czasu i szczegółów next-hop. **To nie jest rzeczywisty output z EVE-NG.** Format i dodatkowe kolumny zależą od wydania Junos.*

Jak to czytać? \`Direct\` albo \`Local\` mówi, skąd pochodzi wpis. Liczba po ukośniku w \`[Direct/0]\` to **route preference** używana przez Junos przy wyborze źródła trasy. Gwiazdka \`*\` przy trasie oznacza, że jest ona aktywna w danym kontekście RIB. Nie musisz znać na pamięć wszystkich preferencji — ich znaczenie wyjaśnimy na przykładzie.

### Co, jeśli R1 zna dwie drogi do tej samej sieci?

Wyobraź sobie, że R1 dowiaduje się o sieci 172.16.60.0/24 dwiema metodami. Pierwsza to statyczna trasa wprowadzona przez administratora. Druga przychodzi z protokołu routingu. System nie powinien losowo przełączać się między nimi.

Junos ma reguły wyboru aktywnej trasy. Jednym z najważniejszych parametrów jest **route preference**: w standardowym porównaniu tras z różnych źródeł **niższa liczba jest lepsza**. Przykładowo typowa statyczna trasa ma domyślną preference 5, a trasa BGP — 170. Nie oznacza to jednak, że statyczna „zawsze wygrywa”. Musi być poprawna, mieć rozwiązywalny next-hop i odnosić się do właściwej tabeli. W bardziej zaawansowanych scenariuszach stosowane są dodatkowe zasady protokołów.

Jest też inna reguła: **Longest Prefix Match (LPM)**. Kiedy router już przekazuje pakiet, wybiera najbardziej szczegółowy pasujący prefix w forwarding table. Jeśli istnieje 172.16.0.0/16 i 172.16.60.0/24, pakiet do 172.16.60.1 pasuje do obu, ale zostanie wybrana bardziej szczegółowa /24. To nie jest bezpośredni pojedynek preference 5 z preference 170: najpierw trzeba odróżnić wybór *trasy do tego samego prefixu* od wyszukania *najbardziej szczegółowego prefixu dla adresu docelowego*.

Na tym etapie wystarczą nam dwa wnioski: **RIB może znać kilka dróg**, a **forwarding musi otrzymać użyteczną decyzję**. Nie każda znana trasa stanie się aktywna i nie każda aktywna trasa gwarantuje dostarczenie pakietu do odległej aplikacji.

### RIB a FIB — dwa skróty, które będziesz stale spotykać

**RIB — Routing Information Base** to miejsce przechowywania informacji o trasach, ich źródłach, next-hopach i statusach. Router potrzebuje tej bazy do podejmowania decyzji, która droga jest najlepsza.

**FIB — Forwarding Information Base** to informacja przygotowana do samego wyszukiwania i przekazywania pakietów. Nie musi mieć identycznej postaci jak pełna RIB. W klasycznym modelu Junos kernel na RE utrzymuje **master forwarding table**, a potrzebne dane są synchronizowane do Packet Forwarding Engine.

Dlatego nie wystarczy otworzyć \`show route\` i powiedzieć: „Wszystko działa”. Polecenie to przede wszystkim pokazuje wiedzę routingu. Aby zobaczyć, jaki forwarding jest przygotowany do konkretnego destination, wykonujemy drugi odczyt.

**R1-vSRX — Operational Mode, od RIB do tabeli forwarding**

~~~junos
show route 10.12.0.0/30 exact extensive
show route forwarding-table destination 10.12.0.2
~~~

Pierwsze polecenie pozwala przeanalizować konkretną trasę w RIB. Drugie pokazuje widok forwardingu dla adresu 10.12.0.2. **\`show route forwarding-table\` nie jest automatycznie bezpośrednim odczytem z pamięci układu PFE.** Juniper opisuje go jako widok master forwarding table po stronie RE; na wspieranych platformach istnieją odrębne polecenia, takie jak \`show pfe route\`, do inspekcji tablic zainstalowanych w PFE. Nie zakładaj ich dostępności na każdym vSRX.

~~~mermaid
flowchart TD
  A["Adres R1: 10.12.0.1/30"] --> D["RIB: inet.0<br/>10.12.0.0/30 Direct"]
  B["Trasy od administratora<br/>lub protokołu"] --> D
  D --> W["Wybór aktywnej trasy<br/>i rozwiązanie next-hop"]
  W --> M["Master forwarding table na RE"]
  M --> P["Synchronizacja do forwarding/PFE"]
  P --> N["Pakiet do 10.12.0.2<br/>wyjście do R2"]
~~~

*Rysunek 2.4. Przepływ informacji routingu do forwardingu. Nie jest to ścieżka, którą każdy pakiet musi fizycznie przejść przez procesy RE.*

### Active, inactive, hidden — co oznacza, że trasa „jest”, ale nie działa?

Kiedy router poznaje tę samą sieć z kilku źródeł, część kandydatów może przegrać wybór. Taką trasę możesz zobaczyć jako **inactive**: istnieje jako informacja, ale nie jest aktywną drogą do celu w danym RIB.

Inna sytuacja to **hidden**. Trasa może być ukryta, ponieważ nie spełnia warunków użycia, na przykład jej next-hop nie daje się rozwiązać albo została odrzucona przez odpowiednią politykę. Nie utożsamiaj \`hidden\` z „drugą najlepszą drogą”, bo przyczyny mogą być zupełnie inne.

**R1-vSRX — Operational Mode, rozszerzona diagnoza**

~~~junos
show route 172.16.60.0/24 exact extensive
show route hidden
show route protocol static
~~~

Te trzy polecenia mają sens dopiero, gdy takie trasy rzeczywiście występują. Jeśli na początku ćwiczenia nie masz jeszcze statycznej ani dynamicznej drogi do 172.16.60.0/24, pusty wynik nie oznacza awarii. Oznacza, że nie skonfigurowałeś jeszcze odpowiedniego źródła informacji.

### Routing Engine nie służy tylko do routingu

RE obsługuje również wiele funkcji administracyjnych. To tutaj działają procesy odpowiedzialne za konfigurację i dostęp operatora, zbierane są wybrane alarmy oraz pracują usługi wymagające udziału systemu operacyjnego.

Przyjrzyj się spokojnie kilku read-only poleceniom.

**R1-vSRX — Operational Mode, stan systemu i procesów**

~~~junos
show version
show system uptime
show system processes extensive
show system alarms
show chassis alarms
~~~

\`show version\` mówi, z jaką wersją systemu pracujesz. \`show system uptime\` pozwala sprawdzić czas działania. \`show system processes extensive\` pokazuje aktywność procesów i obciążenie systemu; przy dużej liczbie procesów możesz wyszukać nazwy \`rpd\` oraz \`mgd\`. Alarmy mogą wskazywać problemy z elementami platformy i nie zawsze występują w każdym wirtualnym labie.

Początkujący administrator nie powinien odruchowo restartować \`rpd\` tylko dlatego, że wynik \`show system processes\` wygląda nietypowo. Najpierw trzeba potwierdzić objaw, ustalić wpływ na routing i forwarding oraz zebrać dane. Restart procesów sterujących może spowodować utratę sąsiedztw i zakłócenia w sieci.

### A jeśli fizyczne urządzenie ma dwa Routing Engines?

W większych routerach Juniper mogą występować redundantne Routing Engines. Jeden jest aktywny, a drugi pełni rolę zapasową, zależnie od modelu i konfiguracji. Funkcje wysokiej dostępności, takie jak GRES czy nonstop routing, ograniczają skutki pewnych przełączeń, ale **samo posiadanie dwóch RE nie zapewnia bezprzerwowego działania wszystkich protokołów i wszystkich przepływów**.

Nasz vSRX w EVE-NG nie jest laboratoryjnym odpowiednikiem modułowego MX z dwiema osobnymi kartami Routing Engine. Możemy zrozumieć architekturę logiczną, ale nie będziemy udawać, że przeprowadziliśmy sprzętowy test GRES.

**Pytanie do samodzielnego rozwiązania.** W \`show route\` R1 widać aktywną trasę, ale użytkownik nie dostaje odpowiedzi. Czy to znaczy, że RIB jest uszkodzona? Nie. Trasa może być poprawna, a problem występuje podczas instalacji/realizacji forwardingu, w security policy, na dalszym routerze, na drodze powrotnej lub na samym urządzeniu docelowym. W kolejnej części zobaczymy, jak w praktyce oddzielać te możliwości.

**Źródła Juniper:** [Routing Databases Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-routing-databases-overview.html), [Junos Routing and Forwarding Tables](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-routing-forwarding-table-overview.html), [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html).


---

## 2.4. Packet Forwarding Engine — jak router przekazuje pakiet, gdy zna już drogę? [A]

### Co PFE robi po otrzymaniu pakietu?

R1 ma przygotowaną informację, że sieć 10.12.0.0/30 znajduje się na porcie w kierunku R2. Gdy R5 wyśle pakiet z adresem docelowym 10.12.0.2, R1 nie musi najpierw uruchamiać OSPF ani BGP. Może skorzystać z odpowiedniej pozycji w tablicy forwarding.

To jednak nie oznacza, że przekazanie pakietu sprowadza się do mechanicznego „wyślij na port 0”. Router musi odczytać właściwe pola nagłówka, odnaleźć pasującą trasę, sprawdzić wymagane warunki przetwarzania i przygotować nową ramkę do wysłania.

Zatrzymajmy się przy słowie **ramka**. Na Ethernet pakiet IP nie jest przesyłany po kablu samodzielnie. Umieszczamy go wewnątrz ramki Ethernet, która ma między innymi adresy MAC źródła i odbiorcy. Adresy IP pozwalają routerom ustalać drogę pomiędzy sieciami. Adresy MAC pozwalają dostarczyć ramkę do następnego urządzenia w obrębie konkretnego segmentu Ethernet.

Kiedy R5 wysyła do R2 przez R1, dzieje się więc coś ważnego: **adres IP docelowy nadal oznacza R2, ale adres MAC odbiorcy zmienia się na każdym odcinku Ethernet**.

### Prześledźmy R5 → R1 → R2 bez skrótów

R5 ma adres 192.0.2.10/24 na porcie do R1. R1 ma po tej stronie adres 192.0.2.1/24, a na łączu do R2 adres 10.12.0.1/30. R2 ma 10.12.0.2/30.

R5 chce wysłać ICMP Echo Request do **10.12.0.2**. Adres ten nie należy do jego lokalnej sieci 192.0.2.0/24. Dlatego R5 potrzebuje trasy kierującej pakiet do **192.0.2.1**, czyli R1. Może być to trasa do konkretnej sieci albo, w bardziej ogólnym scenariuszu, default route.

R5 przygotowuje pakiet IP:

- Source IP: 192.0.2.10.
- Destination IP: 10.12.0.2.
- Protokół: ICMP.

Następnie umieszcza go w ramce Ethernet przeznaczonej dla MAC interfejsu R1 w LAN-A. Jeśli R5 nie zna jeszcze tego MAC, może użyć **ARP (Address Resolution Protocol)**, aby dowiedzieć się, jaki adres MAC odpowiada 192.0.2.1.

Gdy R1 otrzymuje ramkę, odczytuje pakiet IP. Destination IP to 10.12.0.2, czyli **nie jego własny adres 192.0.2.1 ani 10.12.0.1**. R1 sprawdza forwarding i znajduje wyjście na interfejs do R2.

Na wyjściu R1 musi przygotować **nową ramkę Ethernet**. Jej source MAC będzie związany z portem R1 do R2, a destination MAC z portem R2. Pakiet IP nadal ma destination **10.12.0.2**. W typowym routingu bez NAT source IP też pozostaje **192.0.2.10**. Zmienia się natomiast m.in. **TTL**, ponieważ pakiet przechodzi przez router IP.

~~~mermaid
flowchart LR
  R5["R5<br/>IP źródła 192.0.2.10"] -->|"Ramka 1<br/>MAC: R5 → R1<br/>IP: .10 → 10.12.0.2"| R1["R1<br/>przekazuje pakiet"]
  R1 -->|"Ramka 2<br/>MAC: R1 → R2<br/>IP: .10 → 10.12.0.2"| R2["R2<br/>IP celu 10.12.0.2"]
~~~

*Rysunek 2.5. R1 przekazuje pakiet IP, ale tworzy nową ramkę Ethernet na następnym odcinku. W rzeczywistym systemie adresy MAC trzeba odczytać z interfejsów i ARP; nie wpisujemy ich na sztywno do referencyjnej topologii.*

Wyobraź sobie teraz, że R1 ma poprawną trasę do 10.12.0.2, lecz nie może ustalić adresu MAC R2. Samo posiadanie wpisu w RIB nie wystarczy do skutecznej transmisji Ethernet. Trzeba jeszcze rozwiązać informację warstwy drugiej potrzebną do wyjścia.

**R1-vSRX — Operational Mode, połączenie routingu, interfejsu i ARP**

~~~junos
show route 10.12.0.0/30 exact
show route forwarding-table destination 10.12.0.2
show interfaces ge-0/0/0 terse
show arp no-resolve
~~~

W \`show arp\` po udanym ruchu możemy spodziewać się adresu IP 10.12.0.2 powiązanego z MAC R2 na odpowiednim interfejsie. Jeśli go nie ma, to jeszcze nie dowód awarii — być może nie było ruchu wymagającego ARP, wpis wygasł albo platforma inaczej pokazuje sąsiedztwo. Dopiero połączenie stanu interfejsu, tablicy forwardingu i testu pakietowego pozwala rozpoznać problem.

### Longest Prefix Match: skąd router wie, który wpis wybrać?

W rozdziale I poznałeś pojęcie prefiksu. Przypomnijmy je na konkretnym przykładzie. Wpis **172.16.0.0/16** obejmuje większą przestrzeń adresów niż **172.16.60.0/24**. Jeśli obie trasy są użyteczne, pakiet do **172.16.60.1** pasuje do obu, ale bardziej szczegółowa /24 wskazuje precyzyjniejszą drogę.

To nazywamy **Longest Prefix Match (LPM)**. „Longest” oznacza tu najdłuższą maskę/prefix — więcej bitów opisujących sieć — a nie najdłuższą fizycznie trasę.

W naszym DIAMOND-6R sieć **172.16.60.0/24** może należeć do R6. Gdy R1 będzie ją znał, pakiet do 172.16.60.1 powinien zostać skierowany drogą do R6. Jeśli R1 miałby również default route 0.0.0.0/0, nie wygra ona z /24 dla tego celu. Default ma najkrótszy możliwy prefix i służy jako droga ogólna, gdy nie ma bardziej szczegółowego dopasowania.

Nie myl tego z wcześniejszą **route preference**. LPM porównuje dopasowanie różnych prefiksów do adresu docelowego. Route preference pomaga wybierać między konkurującymi trasami do **tego samego** prefiksu. Te mechanizmy często występują obok siebie, ale odpowiadają na inne pytania.

### Co dzieje się z TTL i dlaczego nie każdy ping znika tak samo?

**TTL — Time To Live** to pole w pakiecie IPv4, które ogranicza liczbę przejść przez routery. W typowym routingu router zmniejsza TTL o 1. Jeśli wskutek tego TTL nie pozwala przekazać pakietu dalej, pakiet jest odrzucany, a urządzenie może wygenerować informację **ICMP Time Exceeded** do nadawcy, zależnie od zachowania platformy i filtrów.

To mechanizm chroniący sieć przed pakietami krążącymi bez końca w pętli routingu. Jest także jednym z mechanizmów wykorzystywanych przez **traceroute**.

Ważne: nie trzeba uruchamiać procesu OSPF po każdym zmniejszeniu TTL. W nowoczesnym forwarding odpowiednie operacje mogą zachodzić w układzie PFE. Nie każdy pakiet ICMP musi więc być obsługiwany w całości przez RE. Zagadnienie wyjątkowych pakietów wróci w sekcji 2.6.

### A co, jeśli router zna drogę, ale świadomie odrzuca pakiet?

W sieci istnieją też decyzje typu **discard** i **reject**. To nie są synonimy „brak trasy”. Router może mieć skonfigurowaną trasę, której zadaniem jest właśnie odrzucanie ruchu.

Najprostsza ilustracja to laboratoryjna statyczna trasa do nieobsługiwanej sieci. W Junos można utworzyć route typu \`discard\` albo \`reject\`; ich semantyka odpowiedzi na odrzucenie różni się. W szczególnych platformach i wariantach komunikat ICMP może podlegać dodatkowym ograniczeniom. Gdy pakiet jest odrzucony świadomie, **brak odpowiedzi nie oznacza automatycznie awarii interfejsu**.

Nie konfiguruj takich tras w działającym środowisku tylko po to, żeby zobaczyć odpowiedź ping. W dalszych laboratoriach będziemy je wprowadzać na wyraźnie oznaczonych, izolowanych prefiksach, a następnie usuwać.

### Czy PFE na vSRX jest tym samym co ASIC w switchu?

Nie. **PFE jest nazwą roli w architekturze Junos**, natomiast implementacja zależy od sprzętu i systemu. W wielu fizycznych urządzeniach forwarding realizują wyspecjalizowane układy. W wirtualnym vSRX ruch jest przetwarzany przez programowy dataplane, w którym bardzo ważny jest również mechanizm security flow.

Dla nas oznacza to dwie rzeczy. Po pierwsze, nauka rozdzielenia RE/FIB/forwarding pozostaje użyteczna. Po drugie, nie możemy z wyników vSRX wyciągać wniosków o wydajności sprzętowego PFE w MX lub EX ani deklarować testów TCAM czy ASIC na podstawie maszyny wirtualnej.

**Pytanie kontrolne.** \`show route\` ma właściwą trasę, a \`show route forwarding-table\` również wskazuje sensowny next-hop. Czy możesz uznać, że pakiet dotarł do R2? Jeszcze nie. Trzeba sprawdzić łącze Ethernet, osiągalność MAC/ARP, ewentualne filtry i security policies, drogę powrotną oraz faktyczny test.

**Źródła Juniper:** [Router Data Flow Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-router-data-flow-overview.html), [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html), [show pfe route](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-pfe-route.html).

---

## 2.5. Transit traffic — pakiet, którego celem nie jest router po drodze [A]

### Jeden pakiet, trzy urządzenia i cztery warunki powodzenia

Przyjmijmy, że podłączyliśmy R5–R1–R2 i skonfigurowaliśmy potrzebne adresy. R5 chce wysłać ICMP Echo Request do R2. Interesuje nas teraz **rola środkowego routera R1**.

R1 nie jest nadawcą i nie jest końcowym odbiorcą. Jest urządzeniem, przez które pakiet musi przejść. Taki ruch nazywamy **transit traffic**.

Wielu początkujących inżynierów kojarzy router z czymś, co „po prostu przepuszcza IP”. Ale w praktyce, aby R1 poprawnie przekazał pakiet, muszą być spełnione kolejne warunki:

1. **Pakiet musi dotrzeć na właściwy interfejs wejściowy R1.** Do tego potrzebujemy poprawnego L2 i adresowania po stronie R5.
2. **R1 musi mieć użyteczną drogę do adresu docelowego.** W naszym przypadku 10.12.0.2 jest w sieci bezpośrednio połączonej.
3. **Forwarding i wymagane zabezpieczenia muszą dopuścić ruch.** Na vSRX sama poprawna trasa nie omija stref i security policies.
4. **Odpowiedź musi mieć drogę powrotną.** R2 powinien wiedzieć, jak dotrzeć do źródła 192.0.2.10, a R1 musi odpowiedź poprawnie przekazać.

Nie trzeba znać jeszcze wszystkich poleceń, aby zrozumieć powyższy ciąg. Wyjaśnia on, dlaczego ping w jedną stronę może nie działać pomimo poprawnych tras na routerze środkowym.

### Jak R1 widzi pakiet R5 → R2?

R5 wysyła ICMP Echo Request z source 192.0.2.10 do destination 10.12.0.2. Ramka trafia na R1 przez LAN-A. Router rozpoznaje, że destination 10.12.0.2 nie jest lokalnym adresem R1. To jest potencjalny **tranzyt**.

Na R1 istnieje connected 10.12.0.0/30. Dzięki temu router zna właściwy interfejs wyjściowy. Musi jednak sprawdzić, czy pakiet może zostać przekazany.

W typowej konfiguracji **vSRX w trybie flow-based** sprawdza ruch zgodnie z obowiązującymi security zones i security policies. Dla pakietu R5→R2 potrzebna jest odpowiednia kontrola ruchu w kierunku od strefy przypisanej interfejsowi LAN-A do strefy po stronie R2. To jest **osobny mechanizm od routing table**.

Jeśli ruch zostanie dopuszczony i zostanie utworzona odpowiednia sesja, pakiety należące do tej komunikacji mogą następnie korzystać z informacji zapisanych w stanie flow. Dokładna kolejność przetwarzania pakietów, NAT i filtrów na SRX zależy od trybu i konfiguracji. W rozdziale II nie musimy jej odtwarzać co do pojedynczego etapu wewnętrznego dataplane.

R2 otrzymuje pakiet z destination 10.12.0.2. Jest to już **ruch do samego R2** — w tym miejscu nie jest tranzytem. I to bardzo ważny wniosek: **klasyfikacja pakietu jako tranzytowego lub lokalnego zależy od routera, na którym go obserwujesz**.

### Co z odpowiedzią?

R2 ma wysłać ICMP Echo Reply do 192.0.2.10. Ten adres należy do LAN-A R1, a nie do bezpośredniej sieci R2. R2 potrzebuje zatem trasy powrotnej przez **10.12.0.1**.

Jeśli zapomnisz o tej drodze, R1 może poprawnie dostarczyć Echo Request do R2, lecz odpowiedź nie znajdzie powrotu do R5. Właśnie dlatego diagnoza połączenia powinna zawsze brać pod uwagę **request i reply**.

W większej topologii R5→R6 problem jest jeszcze ciekawszy. W górnym korytarzu przekazywanie mogą wykonywać R1, R2 i R4. Ale droga powrotna z R6 do R5 może przebiegać inaczej, jeżeli trasy są skonfigurowane asymetrycznie. Na firewallach stateful taki wybór może mieć dodatkowe konsekwencje.

### Jak odróżnić awarię routingu od odmowy security policy?

Powiedzmy, że na R1 widzisz aktywną connected do R2 oraz poprawną trasę powrotną do R5. R5 nadal nie może pingować R2. Na vSRX sprawdź nie tylko routing, ale również interfejsy, strefy oraz polityki bezpieczeństwa.

**R1-vSRX — Operational Mode, weryfikacja ruchu tranzytowego R5 → R2**

~~~junos
show interfaces terse
show route 10.12.0.0/30 exact
show route 192.0.2.0/24 exact
show security zones
show security policies
show security flow session source-prefix 192.0.2.10
~~~

Co dostajemy? Pierwsze trzy polecenia pozwalają ustalić stan portów i tras. Odczyt security zones mówi, do jakich stref przypisane są interfejsy. Security policies pokazują zasady decydujące o dopuszczaniu sesji. Ostatnie polecenie pozwala szukać stanu sesji z danym źródłem.

Nie jest błędem, jeśli nowa sesja **nie powstała**, bo ruch został wcześniej odrzucony. Właśnie dlatego sprawdzamy jednocześnie politykę i routing, a nie tylko obecność wpisu sesji.

### „Ping z routera działa” nie znaczy „pakiet klienta przejdzie”

Załóżmy, że z samego R1 wykonasz polecenie \`ping 10.12.0.2\`. Router będzie wówczas **nadawcą** pakietu. Nie będzie to identyczna komunikacja jak ping R5→R2.

W pierwszym przypadku źródłem może być adres interfejsu R1 10.12.0.1. W drugim źródłem jest 192.0.2.10. Na SRX te dwa pakiety mogą przechodzić różne drogi logiczne i podlegać innym regułom niż nowa sesja tranzytowa między strefami.

To częsta pułapka diagnostyczna. Administrator wpisuje ping na routerze, dostaje odpowiedź i stwierdza: „sieć działa”. W rzeczywistości przetestował tylko ruch wygenerowany lokalnie przez router, a nie zachowanie klienta.

**R1-vSRX — Operational Mode, przykład ruchu wygenerowanego lokalnie**

~~~junos
ping 10.12.0.2 source 10.12.0.1 count 3
~~~

Wniosek po tym teście jest wąski, ale przydatny: **R1 może skomunikować się z R2 z podanego source**. Nie wyciągamy z niego dalej idących wniosków o tranzycie.

### Co dokładnie zmienia Ethernet, a co IP?

W typowym routingu bez NAT R1 **nie zmienia** source IP 192.0.2.10 ani destination IP 10.12.0.2. Zmniejsza TTL i przygotowuje odpowiednie nagłówki potrzebne do dalszego przesyłania, w tym nowy nagłówek Ethernet dla segmentu do R2.

Zmiana MAC jest czymś normalnym i oczekiwanym. Router kończy jedną domenę L2 i zaczyna kolejną. Nie wysyła dalej tej samej ramki bez żadnych zmian, tak jak robiłby to zwykły switch przełączający Ethernet w ramach tej samej domeny.

~~~mermaid
flowchart TD
  A["R5 generuje Echo Request<br/>SRC 192.0.2.10<br/>DST 10.12.0.2"] --> B["R1 odbiera na LAN-A"]
  B --> C{"Czy celem jest własny adres R1?"}
  C -->|nie| D["Lookup trasy i kontrola SRX flow"]
  D --> E{"Czy można przekazać?"}
  E -->|tak| F["Nowa ramka Ethernet<br/>do R2; TTL -1"]
  E -->|nie| X["Odrzucenie / brak sesji"]
  F --> G["R2 odbiera pakiet<br/>do własnego IP 10.12.0.2"]
  G --> H["Echo Reply do 192.0.2.10<br/>potrzebna trasa powrotna"]
~~~

*Rysunek 2.6. Jeden pakiet jest tranzytowy na R1, ale kierowany do samego R2. To logiczny przebieg komunikacji, nie uniwersalna sekwencja funkcji bezpieczeństwa w dataplane.*

**Pytanie kontrolne.** Czy dwa routery z poprawną tablicą routingu zawsze będą mogły pingować się przez trzeci router SRX? Nie. Oprócz tras trzeba uwzględnić strefy, polityki, obsługę lokalnego ICMP na odbiorcy, rozwiązywanie MAC i drogę powrotną. Dopiero pełny test pozwala stwierdzić, co działa.

**Źródła Juniper:** [Traffic Processing on SRX Series](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html), [Packet-Based Forwarding](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-packet-based-forwarding.html), [SRX Security Policies Overview](https://www.juniper.net/documentation/us/en/software/junos/security-policies/topics/concept/policy-overview.html).


---

## 2.6. Exception traffic — kiedy pakiet wymaga uwagi samego routera? [A]

### Nie każdy pakiet przechodzący przez port jest pakietem tranzytowym

Poznaliśmy ruch R5→R2, który na R1 był tranzytem. Teraz zmieńmy tylko **adres docelowy** pakietu. R5 nadal jest źródłem, nadal korzysta z tej samej sieci 192.0.2.0/24, a ramka nadal trafia do R1. Tym razem jednak destination IP to **192.0.2.1**, czyli własny adres R1.

R1 nie powinien wysyłać tego pakietu dalej do R2. Pakiet jest skierowany **do samego R1**. Junos musi przekazać go do odpowiedniego mechanizmu lokalnej obsługi, jeśli bezpieczeństwo i konfiguracja zezwalają na taki ruch.

Taki ruch często nazywamy **host-bound** lub **local traffic** — w odróżnieniu od pakietów tranzytowych. W architekturze spotkasz również termin **exception traffic**: obejmuje ruch wymagający szczególnego przetwarzania zamiast zwykłego szybkiego forwardingu. W zależności od platformy może być to komunikacja do RE, pakiet wymagający specjalnej obsługi, zdarzenie kontrolne lub inny wyjątek w dataplane.

Dla początkującego najważniejsze jest rozpoznanie, że **nie każdy pakiet ma być przekazany dalej**, a część ruchu jest skierowana do usług samego routera.

### Trzy rodzaje ruchu, których nie warto mylić

Załóżmy, że obserwujesz R1.

**Transit traffic:** R5 wysyła do 10.12.0.2. R1 ma przekazać pakiet do R2.

**Host-bound traffic:** R5 wysyła do 192.0.2.1. R1 sam jest odbiorcą i ma zdecydować, czy lokalnie obsłużyć ping.

**Locally originated traffic:** Administrator wykonuje na R1 \`ping 10.12.0.2\`. To R1 sam generuje pakiet. Nie przeszedł on najpierw przez interfejs LAN-A jako ruch z R5.

Takie rozróżnienie bardzo pomaga w awariach. Możesz mieć poprawnie działający **locally originated ping** z routera, a jednocześnie zablokowany **transit** z klienta. Możesz też mieć sprawny transit, a zablokowaną odpowiedź na **host-bound ping** do interfejsu routera.

| Co obserwujemy na R1? | Kto jest źródłem? | Kto jest odbiorcą? | Kategoria na R1 |
|---|---|---|---|
| ICMP R5 → R2 | R5 | R2 | Transit |
| ICMP R5 → R1 | R5 | R1 | Host-bound |
| ICMP R1 → R2 | R1 | R2 | Locally originated |
| SSH administratora do R1 | Administrator | R1 | Host-bound |
| BGP UPDATE od R2 do R1, gdy działa peering | R2 | R1 | Local/control-plane |
| Pakiet TTL expired wymagający obsługi wyjątkowej | Zależnie od scenariusza | Zależnie od scenariusza | Exception handling |

Zwróć uwagę na ostatni wiersz. Pakiet, którego TTL się wyczerpał, **nie musi mieć jako destination IP adresu routera**. Może mimo to wymagać szczególnego przetwarzania, na przykład wygenerowania ICMP Time Exceeded. Sposób realizacji zależy od platformy; nie każde takie zdarzenie jest pełnym przetwarzaniem w RE.

### Dlaczego na SRX nie wystarczy „interfejs ma adres, więc ping musi działać”?

vSRX jest nie tylko routerem, ale również firewallem. Junos SRX używa pojęcia **security zone**. Strefa grupuje interfejsy pod względem zasad bezpieczeństwa. Ruch przechodzący między strefami może wymagać odpowiedniej **security policy**, ale ruch do samego urządzenia ma jeszcze osobne reguły **host-inbound-traffic**.

Zostańmy przy R1. Interfejs \`ge-0/0/2.0\` prowadzi do R5. Załóżmy, że przypisaliśmy go do strefy \`CE\`. Chcemy pozwolić R5 pingować własny adres R1 **192.0.2.1**. Na SRX możemy zezwolić na odpowiednią usługę host-inbound dla tego interfejsu.

**R1-vSRX — Configuration Mode, przykład wyłącznie dla izolowanego laboratorium**

~~~junos
set security zones security-zone CE interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
~~~

To polecenie nie jest równoważne „zezwól wszystkim komputerom CE na cały ruch przez R1”. Dopuszcza określony rodzaj komunikacji **do samego urządzenia**, zgodnie z innymi warunkami konkretnej platformy i konfiguracji.

Natomiast jeżeli R5 wysyła pakiet **przez** R1 do R2, potrzebujemy analizy odpowiednich security policies związanych z ruchem między strefami, np. \`CE → CORE\`. Samo host-inbound \`ping\` na R1 nie pozwala automatycznie na transit.

To jeden z najważniejszych praktycznych wniosków całego rozdziału: **te same pakiety ICMP mogą podlegać innym zasadom w zależności od tego, czy ich odbiorcą jest router, czy odległe urządzenie.**

### Co z SSH, BGP i innymi usługami routera?

Przy SSH mamy połączenie administracyjne. Jeśli administrator łączy się z R1, próbuje dotrzeć do konkretnej usługi uruchomionej na R1. Nie jest to tranzyt, nawet jeśli pakiety przeszły wcześniej przez inne routery.

Przy BGP i OSPF sytuacja jest podobna pod względem odbiorcy wiadomości sterujących. Wiadomość protokołu routingu dociera do routera, który ma ją przetworzyć w swoim systemie sterującym. To nie jest zwykły pakiet użytkownika, który zawsze ma przejść dalej bez lokalnej obsługi.

W laboratorium DIAMOND-6R, gdy później uruchomimy BGP pomiędzy R1 i R2, do zestawienia sesji może być potrzebne odpowiednie \`host-inbound-traffic protocols bgp\` na właściwych interfejsach/strefach SRX. Samo ustawienie adresów IP i wpisanie konfiguracji protokołu nie zawsze wystarczy.

Nie konfiguruj jednak w ciemno \`host-inbound-traffic system-services all\` na wszystkich strefach. To wygodne w niektórych krótkich ćwiczeniach, ale nie powinno być nawykiem inżyniera: lepiej świadomie dopuścić tylko usługi potrzebne do danego testu.

### Dlaczego traceroute może pokazywać gwiazdki, choć aplikacja działa?

Traceroute wysyła pakiety tak, aby po kolejnych routerach stopniowo wyczerpywał się ich TTL. Dzięki temu może uzyskać odpowiedzi ICMP informujące o routerach po drodze.

Jeżeli router nie generuje takiej odpowiedzi albo filtruje ją jego konfiguracja, traceroute może pokazać \`*\`. Nie znaczy to automatycznie, że router przestał przekazywać ruch użytkownika. Może poprawnie obsługiwać transit, a nie odpowiadać na pewien rodzaj ruchu kontrolnego.

Ta obserwacja ma znaczenie w diagnostyce Enterprise. Ostatni hop traceroute jest widoczny, aplikacja działa, ale jeden środkowy router pokazuje gwiazdki. Zamiast natychmiast ogłaszać awarię rdzenia, weryfikujesz rzeczywisty ruch aplikacyjny, stan interfejsów i odpowiednie trasy.

### Czy \`monitor traffic\` widzi wszystko, co przechodzi przez router?

Nie. W Junos polecenie \`monitor traffic\` jest przydatne przy obserwacji ruchu dostępnego z perspektywy systemu, zwłaszcza ruchu przychodzącego do lub wychodzącego z RE. Na wielu urządzeniach **zwykły ruch tranzytowy jest obsługiwany w PFE i nie musi pojawić się w tym widoku**.

To częsty błąd początkującego administratora. Próbuje wykonać capture na interfejsie routera, nie widzi pakietów użytkownika i uznaje, że pakiet nigdy nie dotarł. Tymczasem wybrane narzędzie może w ogóle nie służyć do obserwowania całego ruchu transit w danym modelu.

**R1-vSRX — Operational Mode, ostrożny przykład obserwacji pakietów do routera**

~~~junos
monitor traffic interface ge-0/0/2 no-resolve matching "icmp"
~~~

Takie polecenie stosuj tylko na izolowanym labie z niewielkim ruchem i po sprawdzeniu składni konkretnego wydania. Przerwij monitorowanie po zebraniu potrzebnych danych. Wynik może zależeć od tego, które pakiety są kierowane do systemu i jaki mechanizm monitoringu wspiera platforma.

W rzeczywistej diagnostyce dataplane mogą być potrzebne narzędzia platformowe, port mirroring, dedykowany packet capture lub odczyt liczników, a na SRX także **\`show security flow session\`**. Nie zakładaj, że \`monitor traffic\` na vSRX zachowuje się identycznie jak capture na fizycznym MX.

### Ochrona control plane: dlaczego router nie powinien odpowiadać na wszystko?

R1 obsługuje routing i konfigurację. Jego zasoby sterujące nie są nieograniczone. Jeśli ktoś wysyła ogromną liczbę pakietów do własnych adresów routera, może obciążyć lokalne procesy lub mechanizmy obsługi wyjątków.

Dlatego w Junos stosujemy kontrolę dostępu do usług systemowych, filtry, policery i rozwiązania ochrony Routing Engine. W wielu fizycznych urządzeniach ważną rolę odgrywa **stateless firewall filter na \`lo0\`**, ale jego wpływ i interakcja z zarządzaniem oraz SRX security zależą od platformy. Nie jest to to samo co zwykła security policy dla ruchu tranzytowego.

Nie konfigurujemy jeszcze filtra ochrony RE na sześciu routerach. Wymaga to znajomości potrzebnych protokołów i odpowiedniego planu awaryjnego, bo błędny filtr może odciąć SSH, monitoring lub sąsiedztwa routingu. W rozdziale 8 zajmiemy się tym krok po kroku, korzystając z \`commit confirmed\` i niezależnej konsoli.

### Management przez \`fxp0\` nie oznacza automatycznie oddzielnej sieci routingu

Na wielu urządzeniach Juniper spotkasz interfejs zarządzający, często oznaczony \`fxp0\`. Może służyć do niezależnego dostępu administracyjnego, ale jego obecność nie oznacza automatycznie, że każde urządzenie ma dedykowaną, izolowaną instancję routingu management.

Nie wszystkie obrazy vSRX mapują management w ten sam sposób. Zanim wpiszesz komendy albo spróbujesz wykorzystać OOB do bezpiecznego rollback, sprawdź \`show interfaces terse\`, dokumentację template EVE-NG i aktualną konfigurację. Nie wolno zgadywać, który wirtualny NIC jest managementem.

**R1-vSRX — Operational Mode, rozpoznanie dostępu i interfejsów**

~~~junos
show interfaces terse
show configuration interfaces | display set
show configuration security zones | display set
show system services
~~~

Jeżeli ostatnia komenda nie jest dostępna w Twoim release, sięgnij do zwykłego \`show configuration system services\`. Ważniejsza od zapamiętania jednego widoku jest umiejętność powiązania adresu zarządzania z rzeczywistym interfejsem i ochroną dostępu.

### Wracamy do trzech pytań z początku rozdziału

Kto poznaje trasy? **RE i jego procesy**, które utrzymują control plane.

Kto przekazuje zwykły pakiet tranzytowy? **Forwarding plane/PFE**, korzystając z przygotowanych informacji i właściwego przetwarzania bezpieczeństwa.

Kto obsługuje pakiet skierowany do samego routera? **Lokalne mechanizmy systemu**, z udziałem odpowiednich procesów i zabezpieczeń. Nie jest to zwykły tranzyt, nawet jeśli pakiet fizycznie przychodzi tym samym portem.

Teraz możesz zrozumieć dwie pozorne sprzeczności: „ping do bramy działa, aplikacja nie działa” oraz „aplikacja działa, ping do bramy nie działa”. Nie są to sprzeczności — testują inne ścieżki w architekturze.

**Źródła Juniper:** [Router Data Flow Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-router-data-flow-overview.html), [monitor traffic CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html), [Day One: Beginner's Guide to Learning Junos](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf), [SRX Packet Processing](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html).

