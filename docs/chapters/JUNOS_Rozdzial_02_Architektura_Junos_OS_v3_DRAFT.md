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

