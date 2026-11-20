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

