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

**`rpd` — routing protocol process.** Utrzymuje informacje związane z routingiem i jego protokołami. Kiedy za kilka rozdziałów uruchomimy OSPF i BGP, będziemy wiedzieli, że nie są to niezależne programy obsługujące każdy pakiet, lecz mechanizmy dostarczające informacji dla routingu.

**`mgd` — management process.** Odpowiada za obsługę poleceń administratora i konfiguracji. Junos ma ważną cechę: zanim wprowadzisz wiele zmian do działającej konfiguracji, przygotowujesz je w wersji kandydackiej i świadomie zatwierdzasz. Szczegółowo opanujemy to w rozdziale III.

**`chassisd` — chassis daemon.** Zajmuje się elementami platformy, na przykład stanem części sprzętowych i powiązanymi alarmami. Na wirtualnym vSRX jego funkcjonalność i widoczne obiekty mogą być inne niż na fizycznym MX czy EX.

**`dcd` — device control daemon.** Jest związany z zarządzaniem interfejsami. W różnych rodzinach i wydaniach Junos zestaw usług oraz ich podział mogą się różnić, ale dla początkującego najważniejsze pozostaje to, że zarządzanie interfejsami nie jest tożsame z liczeniem tras.

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

Może się zdarzyć, że proces `rpd` zużywa dużo CPU, ponieważ w sieci zachodzą liczne zmiany tras. Nie oznacza to automatycznie, że każdy już przekazywany pakiet zatrzymuje się i czeka na obliczenia RE. Na fizycznych urządzeniach z odrębnym PFE część forwarding może działać dalej z wcześniej przygotowaną tablicą.

Ale jest ważna granica tego wniosku. Jeśli RE nie nadąża z przeliczaniem nowych dróg lub aktualizowaniem forwardingu, sieć może korzystać z **nieaktualnych informacji**, a nowe sesje i sytuacje wyjątkowe mogą zachowywać się inaczej. Na vSRX, gdzie wiele funkcji działa programowo w ramach jednego środowiska, wysokie użycie CPU przez maszynę wirtualną może też bezpośrednio pogorszyć forwarding.

Dlatego zdanie „control plane i forwarding plane są rozdzielone” nie jest gwarancją, że usterka jednego nigdy nie wpłynie na drugi. To model architektoniczny, który pomaga rozpoznać rodzaj problemu.

### Czy proces sterujący uczestniczy w każdej wymianie danych?

Nie. Router nie pyta `rpd` o każdą następną porcję danych TCP. Protokół routingu i RIB dostarczają informację, która jest używana w przygotowanej tablicy forwardingu. Dla zwykłego, znanego ruchu szybka obsługa odbywa się poza takim pełnym cyklem obliczania tras.

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

**Krótka sytuacja do rozważenia.** Ping R5 do 192.0.2.1 działa, ale ping R5 do 10.12.0.2 nie. Czy od razu restartujesz proces `rpd` na R1? Nie. Najpierw sprawdzasz, czy R5 ma trasę przez R1, czy R1 ma trasę do R2, czy R2 zna drogę powrotną do R5 i czy security policy R1 dopuszcza ruch tranzytowy. Każdy z tych warunków można sprawdzić bez restartu usługi.

**Źródła Juniper:** [Junos OS Architecture Overview](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-architecture.html), [Router Data Flow Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-router-data-flow-overview.html).

---

## 2.3. Routing Engine — miejsce, w którym router poznaje drogi i zarządza sobą [A]

### Zacznijmy od pytania: skąd R1 bierze trasę do R2?

Widzieliśmy już, że R1 ma bezpośrednie połączenie z R2. Z adresu interfejsu **10.12.0.1/30** Junos wyprowadza informację, że sieć **10.12.0.0/30** jest dostępna bezpośrednio. Tę informację przechowuje w **Routing Information Base (RIB)**, czyli bazie informacji o trasach. Na początku możesz myśleć o RIB po prostu jako o tablicy, w której router zapisuje znane drogi do sieci.

W Junos domyślna tablica routingu dla IPv4 unicast nazywa się **`inet.0`**. To nazwa, którą bardzo często zobaczysz po wpisaniu `show route`.

Przyjrzyjmy się trzem podstawowym rodzajom tras, które możemy spotkać już na starcie.

**Direct** oznacza sieć, z którą router jest bezpośrednio połączony przez odpowiednio skonfigurowany interfejs. Dla R1 to na przykład 10.12.0.0/30.

**Local** oznacza adres przypisany do samego routera. Na R1 istnieje lokalny adres 10.12.0.1/32; pakiet zaadresowany dokładnie do tego IP nie powinien być potraktowany jak zwykły pakiet do R2. Mamy więc osobno informację o całej podsieci i o własnym adresie urządzenia.

**Static** oznacza trasę, którą administrator skonfigurował ręcznie, na przykład „do sieci 172.16.60.0/24 idź do następnego routera 10.12.0.2”. W przeciwieństwie do direct Junos nie wyprowadzi jej automatycznie z adresu interfejsu.

Kiedy później uruchomimy OSPF czy BGP, pojawią się kolejne źródła informacji. Ale mechanizm odczytywania tablic pozostanie znajomy.

### Jak odczytać pierwszą trasę, jeśli nigdy nie pracowałeś na Junos?

W Junos są dwa podstawowe tryby interfejsu tekstowego. **Operational Mode** służy do odczytywania stanu i wykonywania diagnostyki; typowy prompt kończy się znakiem `>`. **Configuration Mode** służy do przygotowywania zmian i zwykle ma prompt kończący się `#`. Pełną naukę obu trybów dostaniesz w rozdziale III. Na razie wszystkie polecenia `show` poniżej wykonujesz w Operational Mode.

**R1-vSRX — Operational Mode, podstawowy odczyt tras**

~~~junos
show route
show route table inet.0
show route 10.12.0.0/30 exact
show route 10.12.0.1/32 exact
show interfaces terse
~~~

Pierwsze polecenie wyświetla informacje routingu. Drugie wskazuje konkretną tabelę IPv4. Trzecie szuka dokładnie sieci 10.12.0.0/30, a czwarte sprawdza lokalny adres R1 jako /32. Ostatnie pokazuje stan interfejsów, abyśmy mogli powiązać trasę z rzeczywiście skonfigurowanym portem.

Zwróć uwagę na słowo `exact`. W dokumentacji będziemy czasem pytać o konkretny prefix, nie o dowolny pasujący wpis. Dzięki `exact` wynik nie zostanie pomylony z inną, mniej lub bardziej szczegółową trasą.

Na poprawnie skonfigurowanym R1 **oczekujemy** informacji podobnej do poniższej:

~~~text
inet.0:
10.12.0.0/30       *[Direct/0] ... via ge-0/0/0.0
10.12.0.1/32       *[Local/0] ... Local via ge-0/0/0.0
~~~

*Przykład ilustruje najważniejsze pola; celowo pominięto część metadanych, czasu i szczegółów next-hop. **To nie jest rzeczywisty output z EVE-NG.** Format i dodatkowe kolumny zależą od wydania Junos.*

Jak to czytać? `Direct` albo `Local` mówi, skąd pochodzi wpis. Liczba po ukośniku w `[Direct/0]` to **route preference** używana przez Junos przy wyborze źródła trasy. Gwiazdka `*` przy trasie oznacza, że jest ona aktywna w danym kontekście RIB. Nie musisz znać na pamięć wszystkich preferencji — ich znaczenie wyjaśnimy na przykładzie.

### Co, jeśli R1 zna dwie drogi do tej samej sieci?

Wyobraź sobie, że R1 dowiaduje się o sieci 172.16.60.0/24 dwiema metodami. Pierwsza to statyczna trasa wprowadzona przez administratora. Druga przychodzi z protokołu routingu. System nie powinien losowo przełączać się między nimi.

Junos ma reguły wyboru aktywnej trasy. Jednym z najważniejszych parametrów jest **route preference**: w standardowym porównaniu tras z różnych źródeł **niższa liczba jest lepsza**. Przykładowo typowa statyczna trasa ma domyślną preference 5, a trasa BGP — 170. Nie oznacza to jednak, że statyczna „zawsze wygrywa”. Musi być poprawna, mieć rozwiązywalny next-hop i odnosić się do właściwej tabeli. W bardziej zaawansowanych scenariuszach stosowane są dodatkowe zasady protokołów.

Jest też inna reguła: **Longest Prefix Match (LPM)**. Kiedy router już przekazuje pakiet, wybiera najbardziej szczegółowy pasujący prefix w forwarding table. Jeśli istnieje 172.16.0.0/16 i 172.16.60.0/24, pakiet do 172.16.60.1 pasuje do obu, ale zostanie wybrana bardziej szczegółowa /24. To nie jest bezpośredni pojedynek preference 5 z preference 170: najpierw trzeba odróżnić wybór *trasy do tego samego prefixu* od wyszukania *najbardziej szczegółowego prefixu dla adresu docelowego*.

Na tym etapie wystarczą nam dwa wnioski: **RIB może znać kilka dróg**, a **forwarding musi otrzymać użyteczną decyzję**. Nie każda znana trasa stanie się aktywna i nie każda aktywna trasa gwarantuje dostarczenie pakietu do odległej aplikacji.

### RIB a FIB — dwa skróty, które będziesz stale spotykać

**RIB — Routing Information Base** to miejsce przechowywania informacji o trasach, ich źródłach, next-hopach i statusach. Router potrzebuje tej bazy do podejmowania decyzji, która droga jest najlepsza.

**FIB — Forwarding Information Base** to informacja przygotowana do samego wyszukiwania i przekazywania pakietów. Nie musi mieć identycznej postaci jak pełna RIB. W klasycznym modelu Junos kernel na RE utrzymuje **master forwarding table**, a potrzebne dane są synchronizowane do Packet Forwarding Engine.

Dlatego nie wystarczy otworzyć `show route` i powiedzieć: „Wszystko działa”. Polecenie to przede wszystkim pokazuje wiedzę routingu. Aby zobaczyć, jaki forwarding jest przygotowany do konkretnego destination, wykonujemy drugi odczyt.

**R1-vSRX — Operational Mode, od RIB do tabeli forwarding**

~~~junos
show route 10.12.0.0/30 exact extensive
show route forwarding-table destination 10.12.0.2
~~~

Pierwsze polecenie pozwala przeanalizować konkretną trasę w RIB. Drugie pokazuje widok forwardingu dla adresu 10.12.0.2. **`show route forwarding-table` nie jest automatycznie bezpośrednim odczytem z pamięci układu PFE.** Juniper opisuje go jako widok master forwarding table po stronie RE; na wspieranych platformach istnieją odrębne polecenia, takie jak `show pfe route`, do inspekcji tablic zainstalowanych w PFE. Nie zakładaj ich dostępności na każdym vSRX.

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

Inna sytuacja to **hidden**. Trasa może być ukryta, ponieważ nie spełnia warunków użycia, na przykład jej next-hop nie daje się rozwiązać albo została odrzucona przez odpowiednią politykę. Nie utożsamiaj `hidden` z „drugą najlepszą drogą”, bo przyczyny mogą być zupełnie inne.

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

`show version` mówi, z jaką wersją systemu pracujesz. `show system uptime` pozwala sprawdzić czas działania. `show system processes extensive` pokazuje aktywność procesów i obciążenie systemu; przy dużej liczbie procesów możesz wyszukać nazwy `rpd` oraz `mgd`. Alarmy mogą wskazywać problemy z elementami platformy i nie zawsze występują w każdym wirtualnym labie.

Początkujący administrator nie powinien odruchowo restartować `rpd` tylko dlatego, że wynik `show system processes` wygląda nietypowo. Najpierw trzeba potwierdzić objaw, ustalić wpływ na routing i forwarding oraz zebrać dane. Restart procesów sterujących może spowodować utratę sąsiedztw i zakłócenia w sieci.

### A jeśli fizyczne urządzenie ma dwa Routing Engines?

W większych routerach Juniper mogą występować redundantne Routing Engines. Jeden jest aktywny, a drugi pełni rolę zapasową, zależnie od modelu i konfiguracji. Funkcje wysokiej dostępności, takie jak GRES czy nonstop routing, ograniczają skutki pewnych przełączeń, ale **samo posiadanie dwóch RE nie zapewnia bezprzerwowego działania wszystkich protokołów i wszystkich przepływów**.

Nasz vSRX w EVE-NG nie jest laboratoryjnym odpowiednikiem modułowego MX z dwiema osobnymi kartami Routing Engine. Możemy zrozumieć architekturę logiczną, ale nie będziemy udawać, że przeprowadziliśmy sprzętowy test GRES.

**Pytanie do samodzielnego rozwiązania.** W `show route` R1 widać aktywną trasę, ale użytkownik nie dostaje odpowiedzi. Czy to znaczy, że RIB jest uszkodzona? Nie. Trasa może być poprawna, a problem występuje podczas instalacji/realizacji forwardingu, w security policy, na dalszym routerze, na drodze powrotnej lub na samym urządzeniu docelowym. W kolejnej części zobaczymy, jak w praktyce oddzielać te możliwości.

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

W `show arp` po udanym ruchu możemy spodziewać się adresu IP 10.12.0.2 powiązanego z MAC R2 na odpowiednim interfejsie. Jeśli go nie ma, to jeszcze nie dowód awarii — być może nie było ruchu wymagającego ARP, wpis wygasł albo platforma inaczej pokazuje sąsiedztwo. Dopiero połączenie stanu interfejsu, tablicy forwardingu i testu pakietowego pozwala rozpoznać problem.

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

Najprostsza ilustracja to laboratoryjna statyczna trasa do nieobsługiwanej sieci. W Junos można utworzyć route typu `discard` albo `reject`; ich semantyka odpowiedzi na odrzucenie różni się. W szczególnych platformach i wariantach komunikat ICMP może podlegać dodatkowym ograniczeniom. Gdy pakiet jest odrzucony świadomie, **brak odpowiedzi nie oznacza automatycznie awarii interfejsu**.

Nie konfiguruj takich tras w działającym środowisku tylko po to, żeby zobaczyć odpowiedź ping. W dalszych laboratoriach będziemy je wprowadzać na wyraźnie oznaczonych, izolowanych prefiksach, a następnie usuwać.

### Czy PFE na vSRX jest tym samym co ASIC w switchu?

Nie. **PFE jest nazwą roli w architekturze Junos**, natomiast implementacja zależy od sprzętu i systemu. W wielu fizycznych urządzeniach forwarding realizują wyspecjalizowane układy. W wirtualnym vSRX ruch jest przetwarzany przez programowy dataplane, w którym bardzo ważny jest również mechanizm security flow.

Dla nas oznacza to dwie rzeczy. Po pierwsze, nauka rozdzielenia RE/FIB/forwarding pozostaje użyteczna. Po drugie, nie możemy z wyników vSRX wyciągać wniosków o wydajności sprzętowego PFE w MX lub EX ani deklarować testów TCAM czy ASIC na podstawie maszyny wirtualnej.

**Pytanie kontrolne.** `show route` ma właściwą trasę, a `show route forwarding-table` również wskazuje sensowny next-hop. Czy możesz uznać, że pakiet dotarł do R2? Jeszcze nie. Trzeba sprawdzić łącze Ethernet, osiągalność MAC/ARP, ewentualne filtry i security policies, drogę powrotną oraz faktyczny test.

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

Załóżmy, że z samego R1 wykonasz polecenie `ping 10.12.0.2`. Router będzie wówczas **nadawcą** pakietu. Nie będzie to identyczna komunikacja jak ping R5→R2.

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

**Locally originated traffic:** Administrator wykonuje na R1 `ping 10.12.0.2`. To R1 sam generuje pakiet. Nie przeszedł on najpierw przez interfejs LAN-A jako ruch z R5.

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

Zostańmy przy R1. Interfejs `ge-0/0/2.0` prowadzi do R5. Załóżmy, że przypisaliśmy go do strefy `CE`. Chcemy pozwolić R5 pingować własny adres R1 **192.0.2.1**. Na SRX możemy zezwolić na odpowiednią usługę host-inbound dla tego interfejsu.

**R1-vSRX — Configuration Mode, przykład wyłącznie dla izolowanego laboratorium**

~~~junos
set security zones security-zone CE interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
~~~

To polecenie nie jest równoważne „zezwól wszystkim komputerom CE na cały ruch przez R1”. Dopuszcza określony rodzaj komunikacji **do samego urządzenia**, zgodnie z innymi warunkami konkretnej platformy i konfiguracji.

Natomiast jeżeli R5 wysyła pakiet **przez** R1 do R2, potrzebujemy analizy odpowiednich security policies związanych z ruchem między strefami, np. `CE → CORE`. Samo host-inbound `ping` na R1 nie pozwala automatycznie na transit.

To jeden z najważniejszych praktycznych wniosków całego rozdziału: **te same pakiety ICMP mogą podlegać innym zasadom w zależności od tego, czy ich odbiorcą jest router, czy odległe urządzenie.**

### Co z SSH, BGP i innymi usługami routera?

Przy SSH mamy połączenie administracyjne. Jeśli administrator łączy się z R1, próbuje dotrzeć do konkretnej usługi uruchomionej na R1. Nie jest to tranzyt, nawet jeśli pakiety przeszły wcześniej przez inne routery.

Przy BGP i OSPF sytuacja jest podobna pod względem odbiorcy wiadomości sterujących. Wiadomość protokołu routingu dociera do routera, który ma ją przetworzyć w swoim systemie sterującym. To nie jest zwykły pakiet użytkownika, który zawsze ma przejść dalej bez lokalnej obsługi.

W laboratorium DIAMOND-6R, gdy później uruchomimy BGP pomiędzy R1 i R2, do zestawienia sesji może być potrzebne odpowiednie `host-inbound-traffic protocols bgp` na właściwych interfejsach/strefach SRX. Samo ustawienie adresów IP i wpisanie konfiguracji protokołu nie zawsze wystarczy.

Nie konfiguruj jednak w ciemno `host-inbound-traffic system-services all` na wszystkich strefach. To wygodne w niektórych krótkich ćwiczeniach, ale nie powinno być nawykiem inżyniera: lepiej świadomie dopuścić tylko usługi potrzebne do danego testu.

### Dlaczego traceroute może pokazywać gwiazdki, choć aplikacja działa?

Traceroute wysyła pakiety tak, aby po kolejnych routerach stopniowo wyczerpywał się ich TTL. Dzięki temu może uzyskać odpowiedzi ICMP informujące o routerach po drodze.

Jeżeli router nie generuje takiej odpowiedzi albo filtruje ją jego konfiguracja, traceroute może pokazać `*`. Nie znaczy to automatycznie, że router przestał przekazywać ruch użytkownika. Może poprawnie obsługiwać transit, a nie odpowiadać na pewien rodzaj ruchu kontrolnego.

Ta obserwacja ma znaczenie w diagnostyce Enterprise. Ostatni hop traceroute jest widoczny, aplikacja działa, ale jeden środkowy router pokazuje gwiazdki. Zamiast natychmiast ogłaszać awarię rdzenia, weryfikujesz rzeczywisty ruch aplikacyjny, stan interfejsów i odpowiednie trasy.

### Czy `monitor traffic` widzi wszystko, co przechodzi przez router?

Nie. W Junos polecenie `monitor traffic` jest przydatne przy obserwacji ruchu dostępnego z perspektywy systemu, zwłaszcza ruchu przychodzącego do lub wychodzącego z RE. Na wielu urządzeniach **zwykły ruch tranzytowy jest obsługiwany w PFE i nie musi pojawić się w tym widoku**.

To częsty błąd początkującego administratora. Próbuje wykonać capture na interfejsie routera, nie widzi pakietów użytkownika i uznaje, że pakiet nigdy nie dotarł. Tymczasem wybrane narzędzie może w ogóle nie służyć do obserwowania całego ruchu transit w danym modelu.

**R1-vSRX — Operational Mode, ostrożny przykład obserwacji pakietów do routera**

~~~junos
monitor traffic interface ge-0/0/2 no-resolve matching "icmp"
~~~

Takie polecenie stosuj tylko na izolowanym labie z niewielkim ruchem i po sprawdzeniu składni konkretnego wydania. Przerwij monitorowanie po zebraniu potrzebnych danych. Wynik może zależeć od tego, które pakiety są kierowane do systemu i jaki mechanizm monitoringu wspiera platforma.

W rzeczywistej diagnostyce dataplane mogą być potrzebne narzędzia platformowe, port mirroring, dedykowany packet capture lub odczyt liczników, a na SRX także **`show security flow session`**. Nie zakładaj, że `monitor traffic` na vSRX zachowuje się identycznie jak capture na fizycznym MX.

### Ochrona control plane: dlaczego router nie powinien odpowiadać na wszystko?

R1 obsługuje routing i konfigurację. Jego zasoby sterujące nie są nieograniczone. Jeśli ktoś wysyła ogromną liczbę pakietów do własnych adresów routera, może obciążyć lokalne procesy lub mechanizmy obsługi wyjątków.

Dlatego w Junos stosujemy kontrolę dostępu do usług systemowych, filtry, policery i rozwiązania ochrony Routing Engine. W wielu fizycznych urządzeniach ważną rolę odgrywa **stateless firewall filter na `lo0`**, ale jego wpływ i interakcja z zarządzaniem oraz SRX security zależą od platformy. Nie jest to to samo co zwykła security policy dla ruchu tranzytowego.

Nie konfigurujemy jeszcze filtra ochrony RE na sześciu routerach. Wymaga to znajomości potrzebnych protokołów i odpowiedniego planu awaryjnego, bo błędny filtr może odciąć SSH, monitoring lub sąsiedztwa routingu. W rozdziale 8 zajmiemy się tym krok po kroku, korzystając z `commit confirmed` i niezależnej konsoli.

### Management przez `fxp0` nie oznacza automatycznie oddzielnej sieci routingu

Na wielu urządzeniach Juniper spotkasz interfejs zarządzający, często oznaczony `fxp0`. Może służyć do niezależnego dostępu administracyjnego, ale jego obecność nie oznacza automatycznie, że każde urządzenie ma dedykowaną, izolowaną instancję routingu management.

Nie wszystkie obrazy vSRX mapują management w ten sam sposób. Zanim wpiszesz komendy albo spróbujesz wykorzystać OOB do bezpiecznego rollback, sprawdź `show interfaces terse`, dokumentację template EVE-NG i aktualną konfigurację. Nie wolno zgadywać, który wirtualny NIC jest managementem.

**R1-vSRX — Operational Mode, rozpoznanie dostępu i interfejsów**

~~~junos
show interfaces terse
show configuration interfaces | display set
show configuration security zones | display set
show configuration system services
~~~

To ostatnie polecenie pokazuje skonfigurowane usługi systemowe. Niektóre z nich mogą mieć dodatkowe ograniczenia w strefach SRX lub filtrach ochronnych. Ważniejsza od zapamiętania jednego widoku jest umiejętność powiązania adresu zarządzania z rzeczywistym interfejsem i ochroną dostępu.

### Wracamy do trzech pytań z początku rozdziału

Kto poznaje trasy? **RE i jego procesy**, które utrzymują control plane.

Kto przekazuje zwykły pakiet tranzytowy? **Forwarding plane/PFE**, korzystając z przygotowanych informacji i właściwego przetwarzania bezpieczeństwa.

Kto obsługuje pakiet skierowany do samego routera? **Lokalne mechanizmy systemu**, z udziałem odpowiednich procesów i zabezpieczeń. Nie jest to zwykły tranzyt, nawet jeśli pakiet fizycznie przychodzi tym samym portem.

Teraz możesz zrozumieć dwie pozorne sprzeczności: „ping do bramy działa, aplikacja nie działa” oraz „aplikacja działa, ping do bramy nie działa”. Nie są to sprzeczności — testują inne ścieżki w architekturze.

**Źródła Juniper:** [Router Data Flow Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-router-data-flow-overview.html), [monitor traffic CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html), [Day One: Beginner's Guide to Learning Junos](https://www.juniper.net/documentation/en_US/day-one-books/junos-beginners-guide.pdf), [SRX Packet Processing](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html).


---

## 2.LAB. Laboratorium prowadzone — trzy etapy, od dwóch routerów do DIAMOND-6R [EXT]

W tym rozdziale **nie egzaminujemy Cię ze znajomości BGP ani OSPF**. Chcemy zobaczyć, co router robi z pakietem. Potrzebujesz do tego trzech coraz większych ćwiczeń. Każde korzysta z tej samej adresacji i tych samych węzłów, dzięki czemu wykonane wcześniej konfiguracje mają sens również później.

**Status wszystkich ćwiczeń: PLANNED / NOT RUNTIME VALIDATED.** Nie uruchomiono ich na konkretnym obrazie Junos. Wartości, warianty security flow i wyniki opisane jako oczekiwane trzeba potwierdzić w EVE-NG. Zapis `commit check` świadczy o walidacji składni na urządzeniu, nie o dostarczeniu pakietu.

### Zanim wystartujesz — bezpieczne przygotowanie

Pracujemy na **osobnych, izolowanych wirtualnych maszynach vSRX**, bez łączenia interfejsów danych z firmową siecią, Internetem, istniejącymi labami Cisco czy managementem hypervisora. Każdy router musi mieć dostęp konsolowy, a wirtualne karty trzeba rzeczywiście zmapować do portów Junos. Domyślne interfejsy EVE-NG nie zawsze mają taki numer jak na diagramie.

Ten rozdział zakłada **nowe instancje vSRX przeznaczone wyłącznie do naszego kursu**. Jeśli masz już konfigurację stref, NAT, static routes lub host-name, nie doklejaj kolejnych linii bez review — możesz stworzyć konflikty. Zamiast tego użyj świeżej maszyny albo świadomie dostosuj procedurę do stanu istniejącego labu. Nie wykonuj `load override`, `zeroize` ani zbiorczego `delete security` tylko po to, by dopasować urządzenie do ćwiczenia.

Na każdym uruchamianym routerze najpierw wykonaj odczyty.

**R5-vSRX — Operational Mode, preflight**

~~~junos
show version
show interfaces terse
show configuration interfaces | display set
show configuration security zones | display set
show configuration routing-options | display set
show system commit
~~~

**R1-vSRX — Operational Mode, preflight**

~~~junos
show version
show interfaces terse
show configuration interfaces | display set
show configuration security zones | display set
show configuration routing-options | display set
show system commit
~~~

**R2-vSRX — Operational Mode, uruchomienie dopiero w poziomie średnim**

~~~junos
show version
show interfaces terse
show configuration interfaces | display set
show configuration security zones | display set
~~~

Przed każdą zmianą potwierdź nazwę portu fizycznego połączonego z właściwym segmentem. Osobno sprawdź interfejs management/OOB, aby `commit confirmed` był rzeczywiście bezpieczny. W szkoleniowej maszynie z niezależną konsolą możesz przećwiczyć prywatny tryb konfiguracji.

**R1-vSRX — Operational Mode, wejście do konfiguracji**

~~~junos
configure private
~~~

W Configuration Mode wprowadzisz polecenia `set` i przejrzysz różnice. Przed aktywowaniem wykonasz `commit check`, a następnie `commit confirmed 5`. Ten ostatni zatwierdza zmianę na pięć minut z automatycznym powrotem, jeśli nie potwierdzisz jej na stałe. **Podczas oczekującego `commit confirmed` nie wykonuj ponownie `commit check`**, bo w Junos może on potwierdzić oczekującą zmianę. Zamiast tego przejdź do poleceń operacyjnych i testów, a po sukcesie wróć do Configuration Mode i wykonaj zwykłe `commit`.

Zawsze kontroluj, czy użytkownik rzeczywiście znajduje się w odpowiednim trybie. Rozdział III szczegółowo omówi `configure private`, `commit confirmed`, `rollback` i aktywną oraz kandydacką konfigurację.

---

### Poziom 1 — dwa routery R5 i R1: kto jest odbiorcą pakietu? [A]

**Cel:** odróżnić trasę `Direct` od `Local` i zobaczyć pakiet, którego odbiorcą jest sam router.

Włącz R5 i R1 oraz tylko jeden segment L2 **LAN-A 192.0.2.0/24**. Pozostałych czterech routerów na razie nie potrzebujemy.

**R5:** `ge-0/0/0.0` = 192.0.2.10/24.  
**R1:** `ge-0/0/2.0` = 192.0.2.1/24.

W pierwszym ćwiczeniu nie konfigurujemy BGP, OSPF ani zdalnych tras statycznych. Oba adresy należą do tej samej podsieci, więc Junos po aktywowaniu interfejsów automatycznie będzie znał sieć 192.0.2.0/24 jako bezpośrednio połączoną.

**R5-vSRX — Configuration Mode, przygotowanie pierwszego urządzenia**

~~~junos
set system host-name R5-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 192.0.2.10/24
set interfaces lo0 unit 0 family inet address 10.255.255.5/32
set security zones security-zone CE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
show | compare
commit check
commit confirmed 5
~~~

**R1-vSRX — Configuration Mode, drugi koniec LAN-A**

~~~junos
set system host-name R1-vSRX
set interfaces ge-0/0/2 unit 0 family inet address 192.0.2.1/24
set interfaces lo0 unit 0 family inet address 10.255.255.1/32
set security zones security-zone CE interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
show | compare
commit check
commit confirmed 5
~~~

Dlaczego przypisujemy interfejsy do strefy `CE` i dopuszczamy `ping` jako host-inbound? Ponieważ vSRX to firewall. Chcemy, aby **pakiet skierowany do adresu interfejsu drugiego vSRX** mógł być lokalnie obsłużony. Nie potrzebujemy jeszcze polityki między strefami, ponieważ nie ma urządzenia trzeciego i nie testujemy ruchu tranzytowego.

**R5-vSRX — Operational Mode, pierwszy ping z podanym źródłem**

~~~junos
ping 192.0.2.1 source 192.0.2.10 count 3
show arp no-resolve
show route 192.0.2.0/24 exact
show route 192.0.2.10/32 exact
~~~

**R1-vSRX — Operational Mode, rozpoznanie otrzymanego pakietu**

~~~junos
show route 192.0.2.0/24 exact
show route 192.0.2.1/32 exact
show interfaces ge-0/0/2 terse
show security zones
~~~

**Oczekujemy:** R5 potrafi pingować 192.0.2.1, jeśli oba porty, L2 i host-inbound są poprawne. Na obu routerach widać trasę `Direct` 192.0.2.0/24 oraz lokalny /32 własnego adresu interfejsu. W obu urządzeniach możemy znaleźć sąsiednie adresy MAC w tablicy ARP po prawidłowej komunikacji.

**Teraz ważne pytanie:** czy pakiet R5→192.0.2.1 był tranzytowy przez R1? **Nie.** R1 był końcowym odbiorcą. Wynik pingu nie dowodzi jeszcze, że R1 poprawnie przekazuje ruch do jakiejkolwiek innej sieci.

Po pozytywnym sprawdzeniu potwierdź konfiguracje obu maszyn. Jeśli utraciłeś dostęp lub interfejsy mają złe mapowanie, skorzystaj z niezależnej konsoli i oczekiwanego automatycznego rollback zamiast wpisywać dalsze zmiany w ciemno.

**R5-vSRX — Configuration Mode, tylko po PASS**

~~~junos
commit
~~~

**R1-vSRX — Configuration Mode, tylko po PASS**

~~~junos
commit
~~~

#### Pierwsza kontrolowana nieprawidłowość: R1 nie odpowiada na ping, choć ma poprawną trasę

Zanim przejdziesz do R2, możesz sprawdzić mechanizm host-inbound. **Tylko w izolowanym labie** czasowo dezaktywuj zezwolenie `ping` na wejściu `CE` R1. Nie usuwaj adresu interfejsu i nie zmieniaj trasy. Chcemy sprawdzić, czy sama konfiguracja lokalnej usługi może zmienić wynik testu.

**R1-vSRX — Configuration Mode, kontrolowana dezaktywacja host-inbound**

~~~junos
deactivate security zones security-zone CE interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
show | compare
commit check
commit confirmed 3
~~~

**R5-vSRX — Operational Mode, ponowny test**

~~~junos
ping 192.0.2.1 source 192.0.2.10 count 3
~~~

**R1-vSRX — Operational Mode, odczyt bez ingerencji w konfigurację**

~~~junos
show route 192.0.2.0/24 exact
show route 192.0.2.1/32 exact
show configuration security zones security-zone CE | display set
show system commit
~~~

**Oczekiwany rezultat:** route `Direct` i `Local` nadal istnieją, lecz nowy ping host-bound do R1 może przestać otrzymywać odpowiedź, ponieważ jego lokalna obsługa nie jest dozwolona. Pamiętaj, że wpływają na to również inne istniejące konfiguracje i wersja vSRX. Jeżeli wynik nie jest zgodny z założeniem, ustal rzeczywisty punkt kontroli, zamiast deklarować, że funkcja działa.

**Naprawa:** nie potwierdzaj warunkowej zmiany. Poczekaj na automatyczne przywrócenie poprzedniej konfiguracji i ponownie wykonaj ping. W ten sposób uczysz się rollback bez polegania na SSH, które sam mógłbyś odciąć. Nie uruchamiaj `commit check` w trakcie oczekującego confirmed commit.

**Czego nauczył pierwszy poziom?** Router może posiadać poprawną trasę oraz działać na poziomie interfejsu, a jednocześnie nie odpowiadać na konkretną usługę skierowaną do jego własnego adresu.

---

### Poziom 2 — trzy routery R5–R1–R2: pierwszy prawdziwy tranzyt [S — przygotowanie]

**Cel:** zobaczyć komunikację, w której R1 jest urządzeniem pośrednim, a R2 końcowym odbiorcą. To wciąż nie jest zaawansowany routing dynamiczny: używamy tras bezpośrednich i jednej jawnej drogi powrotnej.

Do działającej topologii R5–R1 dołącz R2 i segment **10.12.0.0/30**:

- R1 `ge-0/0/0.0` = 10.12.0.1/30.
- R2 `ge-0/0/0.0` = 10.12.0.2/30.

R5 nie jest bezpośrednio połączony z 10.12.0.0/30. Musimy powiedzieć mu, że do tej sieci ma użyć następnego routera **192.0.2.1**. R2 z kolei musi wiedzieć, że do **192.0.2.0/24** wraca przez **10.12.0.1**.

**R5-vSRX — Configuration Mode, droga do sieci R2**

~~~junos
set routing-options static route 10.12.0.0/30 next-hop 192.0.2.1
show | compare
commit check
commit confirmed 5
~~~

**R1-vSRX — Configuration Mode, drugi interfejs i polityka tranzystu ICMP**

~~~junos
set interfaces ge-0/0/0 unit 0 family inet address 10.12.0.1/30
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security policies from-zone CE to-zone CORE policy ALLOW-TRAINING-PING match source-address any
set security policies from-zone CE to-zone CORE policy ALLOW-TRAINING-PING match destination-address any
set security policies from-zone CE to-zone CORE policy ALLOW-TRAINING-PING match application junos-ping
set security policies from-zone CE to-zone CORE policy ALLOW-TRAINING-PING then permit
show | compare
commit check
commit confirmed 5
~~~

Zwróć uwagę na różnicę względem poziomu pierwszego. Oprócz adresu interfejsu R1 ma teraz **security policy CE → CORE**. To zezwolenie na nowe połączenie ICMP przechodzące **przez R1** pomiędzy dwoma strefami, a nie tylko na ping do własnego adresu R1.

**R2-vSRX — Configuration Mode, drugi koniec tranzytu i droga powrotna**

~~~junos
set system host-name R2-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 10.12.0.2/30
set interfaces lo0 unit 0 family inet address 10.255.255.2/32
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set routing-options static route 192.0.2.0/24 next-hop 10.12.0.1
show | compare
commit check
commit confirmed 5
~~~

Przed wysłaniem ruchu od R5 warto upewnić się, że R1 i R2 widzą się jako bezpośredni sąsiedzi Ethernet.

**R1-vSRX — Operational Mode, pre-check połączenia do R2**

~~~junos
show interfaces ge-0/0/0 terse
show route 10.12.0.0/30 exact
ping 10.12.0.2 source 10.12.0.1 count 3
~~~

**R2-vSRX — Operational Mode, czy znasz drogę powrotną do R5?**

~~~junos
show route 192.0.2.0/24 exact
show route forwarding-table destination 192.0.2.10
~~~

Dopiero po tych kontrolach uruchamiamy właściwy eksperyment.

**R5-vSRX — Operational Mode, pakiet przechodzący przez R1**

~~~junos
ping 10.12.0.2 source 192.0.2.10 count 5
show route 10.12.0.0/30 exact
~~~

**R1-vSRX — Operational Mode, co zrobił router pośredni?**

~~~junos
show route 10.12.0.0/30 exact
show route forwarding-table destination 10.12.0.2
show security policies from-zone CE to-zone CORE
show security flow session source-prefix 192.0.2.10
~~~

**Oczekujemy:** ICMP Echo Request z R5 przechodzi przez R1, dociera do własnego adresu R2 10.12.0.2, a R2 odsyła odpowiedź do 192.0.2.10 przez R1. Na R1 może pojawić się odpowiadająca temu sesja security flow. Jeśli sesji nie widać, weź pod uwagę czas życia ICMP, aktualną security policy i dokładny tryb SRX.

Teraz popatrzmy na ten sam ping z perspektywy trzech urządzeń.

| Router | Rola w teście R5 → 10.12.0.2 |
|---|---|
| R5 | Nadawca (locally originated) |
| R1 | Router pośredni (transit) |
| R2 | Odbiorca (host-bound) |

To najważniejsza rzecz, którą powinieneś umieć samodzielnie wyjaśnić po poziomie drugim. **Ten sam pakiet ma inną rolę na każdym urządzeniu.**

Po pomyślnych testach potwierdź wszystkie trzy konfiguracje. Jeśli coś nie działa, nie wykonuj natychmiast zmian na trzech routerach naraz. Sprawdź w kolejności: trasę R5 do 10.12.0.0/30, connected na R1, security policy CE→CORE, host-inbound ping na R2, a na końcu trasę R2 do 192.0.2.0/24.

**R5-vSRX — Configuration Mode, tylko po PASS**

~~~junos
commit
~~~

**R1-vSRX — Configuration Mode, tylko po PASS**

~~~junos
commit
~~~

**R2-vSRX — Configuration Mode, tylko po PASS**

~~~junos
commit
~~~


---

### Poziom 3 — cały DIAMOND-6R: jeden cel, dwa korytarze, sześć routerów [P — rozszerzenie dydaktyczne]

Na tym poziomie nie uczymy jeszcze konfiguracji OSPF i BGP. Chcemy wyłącznie **zobaczyć, co dzieje się z pakietem, gdy ma przejść przez wiele routerów**, oraz porównać dwa możliwe kierunki. Dlatego wykorzystamy kontrolowane trasy statyczne, które już potrafisz rozpoznać w CLI. Dynamicznego wyboru drogi nauczysz się w późniejszych rozdziałach.

Ważna różnica względem dwóch poprzednich poziomów: **R5 i R6 to routery, nie zwykłe komputery**. Test z `ping ... source ...` jest ruchem wygenerowanym lokalnie na R5, tranzytowym na R1, R2/R3 i R4, a host-bound na R6. Nie jest to pełny dowód komunikacji dwóch komputerów za routerami. Jeśli będziemy kiedyś potrzebować testów ruchu użytkowników z prawdziwych hostów, dodamy odpowiednie węzły końcowe jako osobny wariant laboratorium.

#### Jakie adresy wykorzystamy?

Dla pełnego obrazu wróćmy do stałej mapy interfejsów DIAMOND-6R. Pierwsze dwa łącza po stronie R5/R1/R2 zostały skonfigurowane w poprzednich poziomach.

| Połączenie | Pierwszy koniec | Drugi koniec |
|---|---|---|
| R5–R1 | R5 `ge-0/0/0.0` 192.0.2.10/24 | R1 `ge-0/0/2.0` 192.0.2.1/24 |
| R1–R2 | R1 `ge-0/0/0.0` 10.12.0.1/30 | R2 `ge-0/0/0.0` 10.12.0.2/30 |
| R1–R3 | R1 `ge-0/0/1.0` 10.13.0.1/30 | R3 `ge-0/0/0.0` 10.13.0.2/30 |
| R2–R4 | R2 `ge-0/0/1.0` 10.24.0.1/30 | R4 `ge-0/0/0.0` 10.24.0.2/30 |
| R3–R4 | R3 `ge-0/0/1.0` 10.34.0.1/30 | R4 `ge-0/0/1.0` 10.34.0.2/30 |
| R4–R6 | R4 `ge-0/0/2.0` 198.51.100.1/24 | R6 `ge-0/0/0.0` 198.51.100.10/24 |

Na końcowych routerach dodamy dwa adresy usługowe na `lo0.0`:

- R5: **172.16.50.1/24** (Business-A) oraz **172.16.51.1/24** (Management-A).
- R6: **172.16.60.1/24** (Business-B) oraz **172.16.61.1/24** (Services-B).

Dlaczego używamy loopback? Adres loopback jest logicznym adresem samego urządzenia, niezależnym od konkretnego portu Ethernet. Przydaje się do testów routingu oraz jako stabilny adres w wielu projektach sieciowych. Zwróć jednak uwagę, że **ping do loopbacku R6 pozostaje host-bound na R6**, choć po drodze jest transit na kilku innych routerach.

#### Krok A — uzupełniamy brakujące porty i strefy

Konfigurujemy tylko rzeczy, których jeszcze nie ma. Jeżeli wcześniejsze etapy zmieniły nazwy lub statusy portów, najpierw sprawdź mapping EVE-NG. Nie wykonuj tego na gotowych urządzeniach klienta.

**R1-vSRX — Configuration Mode, dolny port do R3**

~~~junos
set interfaces ge-0/0/1 unit 0 family inet address 10.13.0.1/30
set security zones security-zone CORE interfaces ge-0/0/1.0
show | compare
commit check
commit confirmed 5
~~~

**R2-vSRX — Configuration Mode, górne łącze do R4**

~~~junos
set interfaces ge-0/0/1 unit 0 family inet address 10.24.0.1/30
set security zones security-zone CORE-RIGHT interfaces ge-0/0/1.0
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP match source-address any
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP match destination-address any
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP match application junos-ping
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP then permit
show | compare
commit check
commit confirmed 5
~~~

**R3-vSRX — Configuration Mode, dolna ścieżka**

~~~junos
set system host-name R3-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 10.13.0.2/30
set interfaces ge-0/0/1 unit 0 family inet address 10.34.0.1/30
set interfaces lo0 unit 0 family inet address 10.255.255.3/32
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone CORE-RIGHT interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP match source-address any
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP match destination-address any
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP match application junos-ping
set security policies from-zone CORE to-zone CORE-RIGHT policy PASS-LAB-ICMP then permit
show | compare
commit check
commit confirmed 5
~~~

**R4-vSRX — Configuration Mode, dwa uplinki rdzenia oraz łącze do R6**

~~~junos
set system host-name R4-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 10.24.0.2/30
set interfaces ge-0/0/1 unit 0 family inet address 10.34.0.2/30
set interfaces ge-0/0/2 unit 0 family inet address 198.51.100.1/24
set interfaces lo0 unit 0 family inet address 10.255.255.4/32
set security zones security-zone CORE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone CORE interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone CE interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone CORE to-zone CE policy PASS-LAB-ICMP match source-address any
set security policies from-zone CORE to-zone CE policy PASS-LAB-ICMP match destination-address any
set security policies from-zone CORE to-zone CE policy PASS-LAB-ICMP match application junos-ping
set security policies from-zone CORE to-zone CE policy PASS-LAB-ICMP then permit
show | compare
commit check
commit confirmed 5
~~~

**R6-vSRX — Configuration Mode, ostatni router**

~~~junos
set system host-name R6-vSRX
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.10/24
set interfaces lo0 unit 0 family inet address 10.255.255.6/32
set interfaces lo0 unit 0 family inet address 172.16.60.1/24
set interfaces lo0 unit 0 family inet address 172.16.61.1/24
set security zones security-zone CE interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
show | compare
commit check
commit confirmed 5
~~~

**R5-vSRX — Configuration Mode, dwa adresy testowe na loopback**

~~~junos
set interfaces lo0 unit 0 family inet address 172.16.50.1/24
set interfaces lo0 unit 0 family inet address 172.16.51.1/24
show | compare
commit check
commit confirmed 5
~~~

W tym momencie powinny działać bezpośrednie połączenia między sąsiadami, o ile porty, strefy i mapowanie są prawidłowe. Nie powinieneś jeszcze oczekiwać pingu pomiędzy zdalnymi loopbackami R5 i R6, ponieważ brakuje tras przez cały diament.

**R3-vSRX — Operational Mode, kontrola dolnej ścieżki**

~~~junos
show interfaces terse
show route 10.13.0.0/30 exact
show route 10.34.0.0/30 exact
ping 10.34.0.2 source 10.34.0.1 count 3
~~~

**R4-vSRX — Operational Mode, kontrola połączeń**

~~~junos
show interfaces terse
show route 10.24.0.0/30 exact
show route 10.34.0.0/30 exact
show route 198.51.100.0/24 exact
~~~

Nie zatwierdzaj kolejnego zestawu zmian na błędnej topologii. Po pozytywnych pre-checkach potwierdź oczekujące `commit confirmed` na każdym zmienionym routerze, zawsze z konsoli odpowiedniego urządzenia.

#### Krok B — najpierw przygotowujemy górny korytarz

Wybieramy dwa adresy, które będziemy testować:

**Źródło:** R5 `172.16.50.1`  
**Cel:** R6 `172.16.60.1`

Chcemy, aby Echo Request przeszedł drogą **R5 → R1 → R2 → R4 → R6**, a odpowiedź wróciła odwrotnie. Do każdej strony potrzebujemy tras. R5 zna własne 172.16.50.0/24 jako Direct. R6 zna własne 172.16.60.0/24 jako Direct. Pozostałym routerom musimy pokazać next-hop.

Najpierw popatrz na plan dla **drogi tam**:

| Router | Trasa do 172.16.60.0/24 | Następny hop |
|---|---|---|
| R5 | Static | 192.0.2.1 — R1 |
| R1 | Static | 10.12.0.2 — R2 |
| R2 | Static | 10.24.0.2 — R4 |
| R4 | Static | 198.51.100.10 — R6 |
| R6 | Direct | Lokalny loopback |

A teraz **droga powrotna** do 172.16.50.0/24:

| Router | Trasa do 172.16.50.0/24 | Następny hop |
|---|---|---|
| R6 | Static | 198.51.100.1 — R4 |
| R4 | Static | 10.24.0.1 — R2 |
| R2 | Static | 10.12.0.1 — R1 |
| R1 | Static | 192.0.2.10 — R5 |
| R5 | Direct | Lokalny loopback |

To wystarczy, żeby zobaczyć, dlaczego cztery routery po drodze nie „zgadują” tras powrotnych. Każdy posiada własną decyzję o kolejnym hopie.

**R5-vSRX — Configuration Mode, wyjście w stronę R6**

~~~junos
set routing-options static route 172.16.60.0/24 next-hop 192.0.2.1
show | compare
commit check
commit confirmed 5
~~~

**R1-vSRX — Configuration Mode, do R6 przez R2 i powrót do R5**

~~~junos
set routing-options static route 172.16.60.0/24 next-hop 10.12.0.2
set routing-options static route 172.16.50.0/24 next-hop 192.0.2.10
show | compare
commit check
commit confirmed 5
~~~

**R2-vSRX — Configuration Mode, górne pośrednictwo w obu kierunkach**

~~~junos
set routing-options static route 172.16.60.0/24 next-hop 10.24.0.2
set routing-options static route 172.16.50.0/24 next-hop 10.12.0.1
show | compare
commit check
commit confirmed 5
~~~

**R4-vSRX — Configuration Mode, do R6 bezpośrednio, do R5 przez R2**

~~~junos
set routing-options static route 172.16.60.0/24 next-hop 198.51.100.10
set routing-options static route 172.16.50.0/24 next-hop 10.24.0.1
show | compare
commit check
commit confirmed 5
~~~

**R6-vSRX — Configuration Mode, droga powrotna do sieci R5**

~~~junos
set routing-options static route 172.16.50.0/24 next-hop 198.51.100.1
show | compare
commit check
commit confirmed 5
~~~

Gdy już połączysz routing z security policies w odpowiednich strefach, rozpocznij pierwszy test.

**R5-vSRX — Operational Mode, test górną ścieżką**

~~~junos
ping 172.16.60.1 source 172.16.50.1 count 5
show route 172.16.60.0/24 exact
~~~

**R1-vSRX — Operational Mode, początek rdzenia**

~~~junos
show route 172.16.60.0/24 exact
show route forwarding-table destination 172.16.60.1
show security flow session source-prefix 172.16.50.1
~~~

**R2-vSRX — Operational Mode, router pośredni górnego korytarza**

~~~junos
show route 172.16.60.0/24 exact
show route 172.16.50.0/24 exact
show security flow session source-prefix 172.16.50.1
~~~

**R4-vSRX — Operational Mode, wyjście z rdzenia**

~~~junos
show route 172.16.60.0/24 exact
show route 172.16.50.0/24 exact
show security policies from-zone CORE to-zone CE
~~~

**Oczekiwany wynik:** nowe Echo Requesty R5→R6 mogą dotrzeć górnym korytarzem, a Echo Reply wrócić przez R4/R2/R1, jeżeli właściwe policy SRX, host-inbound oraz rozwiązywanie next-hop działają prawidłowo. Gdy ping nie działa, **nie dokładaj automatycznie następnej trasy**: sprawdź, czy dotychczasowe route są aktywne, czy każdy hop zna drogę powrotną i czy na vSRX jest dopuszczona odpowiednia sesja.

Dokładne zachowanie lokalnego ruchu do `lo0` na R6 oraz sesji flow wymaga weryfikacji z danym release. Nie można ogłosić sukcesu na podstawie przygotowanej konfiguracji.

#### Krok C — dolny korytarz i drugi prefiks usługowy

Teraz wprowadzimy **osobny zestaw sieci**, żeby oba korytarze mogły działać równocześnie bez przesuwania tych samych static routes.

**Źródło:** R5 `172.16.51.1`  
**Cel:** R6 `172.16.61.1`

Oczekujemy drogi **R5 → R1 → R3 → R4 → R6** i powrotu przez R3. To dobry przykład, że router może mieć różne następne hopy dla dwóch różnych sieci docelowych.

| Router | Trasa do 172.16.61.0/24 | Trasa do 172.16.51.0/24 |
|---|---|---|
| R5 | przez R1 192.0.2.1 | Direct |
| R1 | przez R3 10.13.0.2 | przez R5 192.0.2.10 |
| R3 | przez R4 10.34.0.2 | przez R1 10.13.0.1 |
| R4 | przez R6 198.51.100.10 | przez R3 10.34.0.1 |
| R6 | Direct | przez R4 198.51.100.1 |

**R5-vSRX — Configuration Mode, Services-B przez R1**

~~~junos
set routing-options static route 172.16.61.0/24 next-hop 192.0.2.1
show | compare
commit check
commit confirmed 5
~~~

**R1-vSRX — Configuration Mode, dolna ścieżka przez R3**

~~~junos
set routing-options static route 172.16.61.0/24 next-hop 10.13.0.2
set routing-options static route 172.16.51.0/24 next-hop 192.0.2.10
show | compare
commit check
commit confirmed 5
~~~

**R3-vSRX — Configuration Mode, przekazywanie między R1 i R4**

~~~junos
set routing-options static route 172.16.61.0/24 next-hop 10.34.0.2
set routing-options static route 172.16.51.0/24 next-hop 10.13.0.1
show | compare
commit check
commit confirmed 5
~~~

**R4-vSRX — Configuration Mode, usługa R6 i droga powrotna przez R3**

~~~junos
set routing-options static route 172.16.61.0/24 next-hop 198.51.100.10
set routing-options static route 172.16.51.0/24 next-hop 10.34.0.1
show | compare
commit check
commit confirmed 5
~~~

**R6-vSRX — Configuration Mode, powrót do Management-A**

~~~junos
set routing-options static route 172.16.51.0/24 next-hop 198.51.100.1
show | compare
commit check
commit confirmed 5
~~~

**R5-vSRX — Operational Mode, porównanie obu dróg**

~~~junos
ping 172.16.60.1 source 172.16.50.1 count 3
ping 172.16.61.1 source 172.16.51.1 count 3
show route 172.16.60.0/24 exact
show route 172.16.61.0/24 exact
~~~

**R1-vSRX — Operational Mode, gdzie router wyśle dwa rodzaje pakietów?**

~~~junos
show route 172.16.60.0/24 exact
show route 172.16.61.0/24 exact
show route forwarding-table destination 172.16.60.1
show route forwarding-table destination 172.16.61.1
~~~

Na R1 powinieneś zobaczyć, że dla 172.16.60.0/24 next-hop jest **10.12.0.2 (R2)**, a dla 172.16.61.0/24 next-hop **10.13.0.2 (R3)**. To rezultat **dwóch różnych statycznych tras do dwóch różnych prefiksów**, a nie automatycznego load balancingu czy decyzji BGP.

Jeżeli testy pokazują oczekiwane ścieżki i komunikacja działa, potwierdź odpowiednie commity po pozytywnych kontrolach. W późniejszych rozdziałach nauczymy się zastępować ręczne static routes protokołami dynamicznymi i politykami wyboru drogi.


#### Krok D — dwie kontrolowane awarie, dzięki którym naprawdę zrozumiesz architekturę

Kiedy obie drogi działają, możemy świadomie zepsuć pojedynczy element. Bez tego laboratorium byłoby tylko ćwiczeniem przepisywania konfiguracji.

**Awaria 1: R1 jest dostępny, ale nie przepuszcza ruchu z CE do CORE.**

Na R1 tymczasowo wyłączamy laboratoryjną security policy `ALLOW-TRAINING-PING`. Nie ruszamy adresacji, tras ani portów. Oczekujemy, że R5 nadal będzie mógł pingować **własny adres R1 192.0.2.1**, ponieważ host-inbound działa osobno. Natomiast **nowe próby komunikacji** R5→R2/R6 przechodzące przez R1 mogą zostać odrzucone przez brak odpowiedniej policy tranzytowej.

**R1-vSRX — Configuration Mode, fault injection wyłącznie na izolowanym labie**

~~~junos
deactivate security policies from-zone CE to-zone CORE policy ALLOW-TRAINING-PING
show | compare
commit check
commit confirmed 3
~~~

**R5-vSRX — Operational Mode, dwie różne kategorie testu**

~~~junos
ping 192.0.2.1 source 192.0.2.10 count 3
ping 10.12.0.2 source 192.0.2.10 count 3
ping 172.16.60.1 source 172.16.50.1 count 3
~~~

**R1-vSRX — Operational Mode, osobne dowody routingu i bezpieczeństwa**

~~~junos
show route 10.12.0.0/30 exact
show security policies from-zone CE to-zone CORE
show security flow session
show system commit
~~~

**Oczekiwane zachowanie:** adres i trasy na R1 nadal istnieją, ping host-bound do R1 może działać, ale inicjacja nowej sesji tranzytowej od CE do CORE jest zablokowana. **Istniejące sesje mogą jeszcze działać** — nie myl ich z nową próbą po zmianie policy. Dla jednoznacznego testu wykorzystaj nowy flow, zaczekaj na wygaśnięcie testowych sesji albo zastosuj celowane czyszczenie tylko własnego labowego ruchu zgodnie ze składnią danej wersji Junos. Nie wykonuj globalnego `clear security flow session all` na urządzeniu współdzielonym z innymi użytkownikami.

**Naprawa:** nie potwierdzaj zmiany. Pozwól zadziałać automatycznemu rollbackowi z `commit confirmed`, następnie sprawdź, że policy znów jest aktywna i **nowa sesja** R5→R2 przechodzi. Nie potrzebujesz resetu `rpd` ani zmiany static routes — ich nie dotykaliśmy.

**Awaria 2: górny korytarz nie ma działającego łącza R2–R4.**

Teraz chcemy sprawdzić, czy konfiguracja sieci z dwoma korytarzami oznacza automatyczny failover. Odpowiedź brzmi: **nie**, dopóki nie zaprojektujesz odpowiedniego mechanizmu wyboru alternatywnej drogi.

Gdy dezaktywujesz na R2 interfejs do R4, trasa 172.16.60.0/24 przez 10.24.0.2 może utracić rozwiązywalny next-hop. R1 nadal może znać własną statyczną drogę do 172.16.60.0/24 *via R2*, ale to nie oznacza, że R2 ma dalej drogę do R6. Natomiast sieć 172.16.61.0/24, którą prowadziliśmy przez R3, powinna nadal móc działać, jeśli jej własna ścieżka i security są poprawne.

**R2-vSRX — Configuration Mode, kontrolowane wyłączenie łącza do R4**

~~~junos
deactivate interfaces ge-0/0/1
show | compare
commit check
commit confirmed 3
~~~

**R2-vSRX — Operational Mode, sprawdzenie skutku dla forwardingu**

~~~junos
show interfaces ge-0/0/1 terse
show route 10.24.0.0/30 exact
show route 172.16.60.0/24 exact extensive
show route forwarding-table destination 172.16.60.1
show system commit
~~~

**R5-vSRX — Operational Mode, dwie usługi przez różne korytarze**

~~~junos
ping 172.16.60.1 source 172.16.50.1 count 3
ping 172.16.61.1 source 172.16.51.1 count 3
~~~

**Oczekiwane zachowanie:** górna usługa może przestać odpowiadać, podczas gdy dolna nadal działa. Utrata jednego interfejsu na R2 nie usuwa całego routera ani nie oznacza, że wszystkie prefiksy diamentu muszą być niedostępne.

Pomyśl teraz, co byłoby potrzebne do automatycznego przełączenia górnej usługi na R3. R1 musiałby mieć alternatywną użyteczną drogę do **tego samego** prefiksu, a informacja o awarii R2–R4 musiałaby wpłynąć na jej wybór. Potrzebne byłyby też spójne trasy powrotne i bezpieczeństwo. To właśnie zadanie protokołów dynamicznych i zaawansowanych mechanizmów routingu, które opanujesz w rozdziałach 6–11.

**Naprawa:** pozwól automatycznemu rollbackowi przywrócić interfejs R2 do R4. Potwierdź stan `up/up`, ponowną obecność 10.24.0.0/30 oraz prawidłowy forwarding do 172.16.60.1. Nie deklaruj automatycznej konwergencji całego DIAMOND-6R, bo w tym ćwiczeniu używaliśmy jawnych tras statycznych.

#### Krok E — co zachowujemy po ćwiczeniu?

Zachowaj działającą, bezpieczną konfigurację jako **baseline** do dalszych rozdziałów, jeśli zamierzasz ponownie wykorzystywać laboratorium. Wpisz do notatek realne `show version` dla sześciu routerów, mapowanie portów EVE-NG i wyniki pre/post-check. Zrób snapshot zgodny z mechanizmami platformy wirtualizacyjnej albo wyeksportuj do oddzielnych, bezpiecznych plików **sanityzowane** konfiguracje bez haseł i kluczy.

Nie jest dobrym pomysłem zbiorcze kasowanie `interfaces` i `security` po każdej lekcji: mogłoby to wyczyścić nie tylko ćwiczenie, ale też dostęp administracyjny i stan bazowy potrzebny do rozdziału III.

Przykładowe dowody sukcesu:

| Sprawdzany mechanizm | Jaki dowód zachować? |
|---|---|
| Direct / Local | `show route` na R5 i R1 dla LAN-A |
| Transit R5→R2 | Ping z R5 + route, forwarding i security session na R1 |
| Górna droga do R6 | `show route` na R1/R2/R4 + ping source 172.16.50.1 |
| Dolna droga do R6 | `show route` na R1/R3/R4 + ping source 172.16.51.1 |
| Brak host-inbound | Trasa istnieje, ale ping do własnego adresu R1 nie jest dopuszczony |
| Brak policy tranzytowej | R5→R1 może działać, a nowe R5→R2 nie |
| Zerwana droga R2–R4 | Upper może przestać działać, lower może pozostać sprawny |
| Cofnięcie zmiany | Historyczny `show system commit` oraz wyniki po automatycznym rollback |

Nie podawaj w README wpisu **RUNTIME VALIDATED**, dopóki sam nie uzyskasz realnych dowodów z konsoli. Dokument napisany przez model językowy, nawet bardzo szczegółowy, pozostaje projektem testów.

---

## 2.TS. Troubleshooting — jak znaleźć miejsce usterki zamiast zmieniać wszystko naraz [EXT]

Przed pierwszym samodzielnym dyżurem nie musisz znać na pamięć stu komend. Potrzebujesz umieć zadać poprawne pytania i dobrać do nich narzędzia.

### Sytuacja A: „Router odpowiada na ping, ale użytkownicy nie mają sieci”

Nie zaczynaj od stwierdzenia, że routing na pewno działa. Zapytaj, **kto wysyłał ping i do jakiego adresu**. Jeśli administrator pingował R1 z innego routera, przetestował tylko daną ścieżkę do lokalnego adresu R1. Nie przetestował ruchu użytkownika przez R1.

Jeśli problem dotyczy połączenia R5→R2, przejdź po logicznych etapach: adresacja R5, route R5 do destination, interfejs wejściowy R1, aktywna trasa R1, security policy R1 CE→CORE, interfejs R1→R2, host-inbound R2 i powrót R2→R5.

**R1-vSRX — Operational Mode, pierwsza kontrola bez zmieniania konfiguracji**

~~~junos
show interfaces terse
show route 10.12.0.0/30 exact
show route forwarding-table destination 10.12.0.2
show security zones
show security policies
show security flow session source-prefix 192.0.2.10
~~~

Każda komenda odpowiada na inne pytanie. Nie szukaj po całym output wszystkiego naraz. Najpierw potwierdź, czy interfejs jest dostępny, potem czy route istnieje, a dopiero następnie czy security pozwala na właściwy flow.

### Sytuacja B: „W show route jest trasa, ale nie ma komunikacji”

To częsty objaw, którego przyczyny bywają bardzo różne. RIB może przechowywać aktywną trasę, lecz next-hop może mieć problem z rozwiązywaniem L2. Security flow może nie pozwalać na nową sesję. Odległy router może nie mieć drogi powrotnej.

Dlatego dla konkretnego celu porównaj **RIB z widokiem forwardingu**, a następnie sprawdź rzeczywistą ścieżkę. Pamiętaj przy tym, że `show route forwarding-table` nie jest gwarantowanym bezpośrednim odczytem stanu sprzętowego PFE ani dowodem dostarczenia pakietu.

**R1-vSRX — Operational Mode, trzy perspektywy tej samej sieci**

~~~junos
show route 172.16.60.0/24 exact extensive
show route forwarding-table destination 172.16.60.1
show interfaces extensive
~~~

Jeśli widzisz poprawny next-hop, ale nadal nie ma odpowiedzi, kolejne pytania dotyczą security, ARP, ruchu na dalszym interfejsie i odpowiedzi w przeciwnym kierunku.

### Sytuacja C: „Traceroute pokazuje gwiazdki na R3, ale aplikacja działa”

Gwiazdka oznacza, że nie otrzymałeś oczekiwanej odpowiedzi traceroute w określonym momencie. Nie musi oznaczać, że R3 nie forwarduje pakietów. Obserwuj rzeczywistą aplikację, route i stany interfejsów. Wysyłanie odpowiedzi ICMP Time Exceeded i przekazywanie innego ruchu nie są identycznym zadaniem.

### Sytuacja D: „Proces rpd używa dużo CPU”

`rpd` jest procesem związanym z routingiem. Zwiększone zużycie zasobów może towarzyszyć zmianom topologii, problemom protokołów albo dużej liczbie tras. Nie wiesz jednak jeszcze, czy to błąd, czy chwilowe normalne obciążenie.

Sprawdź stabilność czasu działania, alarmy, procesy i stan tras. Jeżeli problem jest produkcyjny, zbieraj obserwacje oraz odpowiednie logi. Nie restartuj `rpd` bez planu, ponieważ może to mieć wpływ na sesje i dostępność sieci.

**R1-vSRX — Operational Mode, read-only analiza obciążenia**

~~~junos
show system uptime
show system processes extensive
show system alarms
show log messages | last 30
~~~

### Sytuacja E: „BGP jest Established, ale nie mogę pingować loopbacku R6”

To przykład problemu z przyszłych rozdziałów. `Established` świadczy o stanie sąsiedztwa BGP, nie o tym, że konkretny prefix loopbacku R6 jest reklamowany, zaakceptowany, aktywny, poprawnie zainstalowany i dopuszczony przez SRX.

Najpierw odpowiedz na dwa podstawowe pytania: czy router ma użyteczną drogę **do celu** oraz czy odbiorca ma drogę **z powrotem do źródła**. Później przejdziesz do szczegółowego debugowania BGP import/export w rozdziale 8. Nie musisz go znać już teraz, ale umiesz rozpoznać, czego sam status sesji jeszcze nie dowodzi.

### Prosta metoda diagnostyczna, którą warto zachować

~~~mermaid
flowchart TD
  A["Usługa nie odpowiada"] --> B{"Czy źródło zna adres i next-hop?"}
  B -->|nie| C["Adresacja / trasa na źródle"]
  B -->|tak| D{"Czy router pośredni ma aktywną trasę i interface?"}
  D -->|nie| E["RIB, interfejs i next-hop"]
  D -->|tak| F{"Czy ruch przechodzi przez security/forwarding?"}
  F -->|nie| G["Security zone/policy, ARP, filtry, FIB"]
  F -->|tak| H{"Czy cel otrzymuje pakiet i zna powrót?"}
  H -->|nie| I["Odbiorca, lokalna usługa, return path"]
  H -->|tak| J["Sprawdź aplikację i odpowiedź"]
~~~

*Rysunek 2.7. Prosty schemat pracy od źródła do celu. Nie jest to bezwzględna kolejność wewnętrznych etapów przetwarzania wszystkich urządzeń.*

---

## 2.TEST. Sprawdź, czy rozumiesz — zanim przejdziesz do CLI i konfiguracji [EXT]

Spróbuj odpowiedzieć bez zaglądania do definicji. Nie chodzi o zapamiętywanie słów, lecz o ustalenie, jak zachowa się urządzenie.

**Pytanie 1.** R1 ma adres 192.0.2.1/24. R5 wysyła ping do 192.0.2.1. Czy pakiet jest tranzytowy przez R1?

**Pytanie 2.** R5 wysyła ping do 10.12.0.2 przez R1. Jaką rolę pełni pakiet na R5, R1 i R2?

**Pytanie 3.** R1 pokazuje aktywną trasę `Direct` do 10.12.0.0/30, ale nie odpowiada na ping do własnego 10.12.0.1. Czy sama obecność `Direct` gwarantuje lokalną odpowiedź ICMP?

**Pytanie 4.** W tablicy R1 znajdują się dwa prefiksy: 172.16.0.0/16 oraz 172.16.60.0/24. Który z nich będzie bardziej szczegółowym dopasowaniem do 172.16.60.1?

**Pytanie 5.** R1 widzi w RIB właściwą trasę. Czy można zakończyć troubleshooting bez sprawdzania forwardingu, security i drogi powrotnej?

**Pytanie 6.** Co dokładnie pokazuje `show route forwarding-table` w klasycznym modelu Junos? Czy oznacza to automatycznie odczyt z fizycznego układu PFE?

**Pytanie 7.** R5 ma dwie drogi do R6 przez R2 i R3. Czy samo istnienie obu połączeń oznacza automatyczny failover, jeśli routing jest wyłącznie statyczny?

**Pytanie 8.** Dlaczego `ping 10.12.0.2` uruchomiony na R1 nie potwierdza, że ruch tranzytowy od R5 do R2 jest dozwolony?

**Pytanie 9.** R3 nie odpowiada na traceroute, ale ruch aplikacyjny przez R3 działa. Czy musi to oznaczać awarię?

**Pytanie 10.** Co sprawdzisz najpierw, gdy `rpd` ma wysokie CPU: zrestartujesz proces czy zbierzesz dane o stanie, obciążeniu i wpływie na routing?

### Odpowiedzi z wyjaśnieniem

**1.** Nie. R1 jest odbiorcą pingu, dlatego jest to host-bound na R1.

**2.** R5 jest nadawcą, R1 przekazuje pakiet jako tranzyt, a R2 jest jego odbiorcą. Ten sam pakiet pełni różne role na różnych urządzeniach.

**3.** Nie. Trasa mówi o osiągalności/powiązaniu adresu z interfejsem. Odpowiedź ICMP wymaga jeszcze odpowiedniej lokalnej obsługi oraz dopuszczenia przez zabezpieczenia.

**4.** Prefiks 172.16.60.0/24. Zasada longest prefix match preferuje bardziej szczegółowy pasujący prefix, jeśli trasa jest użyteczna.

**5.** Nie. RIB to tylko jeden element procesu. Trzeba zweryfikować next-hop, forwarding, interfejsy, zabezpieczenia i odpowiedź z drugiej strony.

**6.** Pokazuje widok master forwarding table po stronie RE w klasycznym Junos. Nie jest sam w sobie gwarancją odczytu sprzętowego PFE; szczegóły poleceń PFE zależą od platformy.

**7.** Nie. Przy statycznych trasach musimy świadomie zaprojektować mechanizm wycofania niewłaściwej drogi i wybór alternatywy oraz zapewnić spójny powrót.

**8.** Ping z R1 jest ruchem wygenerowanym lokalnie. Pakiet R5→R2 przechodzi przez R1 z innym źródłem, na innym ingress i może wymagać security policy CE→CORE.

**9.** Nie. Odpowiedzi traceroute podlegają osobnym ograniczeniom i filtrom; forwarding użytkowników może nadal działać.

**10.** Najpierw zbiorę dane read-only, ocenię wpływ na routing i ustalę przyczynę. Restart procesu sterującego bez diagnozy może wywołać kolejne problemy.

### Po czym poznasz, że rozdział był dla Ciebie przydatny?

Nie potrzebujesz dziś pamiętać nazw wszystkich procesów. Powinieneś jednak potrafić stanąć przed konsolą nowego vSRX i odpowiedzieć własnymi słowami na następujące pytania: skąd router wziął daną trasę, czy ma przygotowaną decyzję forwarding, czy dany pakiet ma przejść **przez** router, czy **do** routera, oraz jak odróżnić brak routingu od blokady w SRX security flow.

Jeśli potrafisz to zrobić na R5–R1–R2, a potem wskazać górną i dolną drogę DIAMOND-6R, masz fundament potrzebny do kolejnych rozdziałów. Teraz nauczymy się **sprawnie wydawać polecenia i bezpiecznie zatwierdzać konfigurację w CLI**. To temat rozdziału III.

---

## Źródła, zakres i status techniczny

Ten rozdział opracowano od nowa, korzystając z wcześniejszej wersji rozdziału II, zunifikowanego blueprintu oraz oficjalnej dokumentacji Juniper. Zachowano pełne cele **2.1 architektura**, **2.2 control/forwarding**, **2.3 RE**, **2.4 PFE**, **2.5 transit**, **2.6 exception**. Laboratorium, scenariusze awarii i pytania kontrolne są rozszerzeniem dydaktycznym, nie dodatkowymi oficjalnymi punktami egzaminu.

Obowiązkowe referencje repozytorium:

- [Unified Blueprint](../JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md)
- [DIAMOND-6R — pełna adresacja i role](../../labs/reference-diamond-6vsrx/README.md)
- [DIAMOND-6R — schemat Mermaid](../../labs/reference-diamond-6vsrx/DIAMOND_TOPOLOGY.md)
- [Archiwum poprzedniej wersji rozdziału II](../archive/JUNOS_Rozdzial_02_pre_novice_rewrite_2026-11-20.md)
- [Rozdział VIII — wzorzec narracji i dalszy rozwój routing policies](JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters.md)

Podstawowe materiały producenta:

- [Junos OS Architecture Overview](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-architecture.html)
- [Routing Engine Components and Processes](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-components-and-processes.html)
- [Routing and Forwarding Tables](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-routing-forwarding-table-overview.html)
- [Routing Databases Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-overview/topics/concept/routing-protocols-routing-databases-overview.html)
- [Router Data Flow Overview](https://www.juniper.net/documentation/us/en/software/junos/routing-policy/topics/concept/firewall-filter-router-data-flow-overview.html)
- [show route forwarding-table](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html)
- [Junos OS Evolved Differences](https://www.juniper.net/documentation/us/en/software/junos/overview-evo/topics/concept/evo-top-differences.html)
- [Traffic Processing on SRX Series](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html)
- [monitor traffic CLI](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html)

**Status walidacji: SOURCE REVIEWED / STATIC EDITORIAL REVIEW / NOT RUNTIME VALIDATED.** Wszystkie topologie, konfiguracje i spodziewane wyniki są projektem ćwiczenia. Nie wykonano rzeczywistych testów `commit check`, `ping`, sesji SRX ani fizycznego odczytu PFE na urządzeniach. Rysunki Mermaid są autorskie. Przy realizacji należy podać model oraz release Junos, sprawdzić mapowanie NIC, istniejące konfiguracje stref, wsparcie konkretnych poleceń i faktyczne ścieżki pakietów.

*Po tym rozdziale nie powinieneś już używać zdania „router działa” bez dopowiedzenia, **co dokładnie zostało przetestowane**. To prosta zmiana w myśleniu, ale jedna z najważniejszych w pracy inżyniera sieci.*
