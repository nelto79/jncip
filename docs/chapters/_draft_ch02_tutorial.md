# Rozdział 2. Junos OS Fundamentals — od pierwszego pakietu do architektury RE/PFE

**Podręcznik:** JUNOS Enterprise Routing & Switching — JNCIA-Junos → JNCIS-ENT → JNCIP-ENT (JN0-650)  
**Wersja redakcyjna:** 2.0 — lekcja prowadzona, 15 listopada 2026  
**Oficjalny zakres w naszym Unified Blueprint:** 2.1–2.6 [A]  
**Dla kogo:** inżynier Cisco IOS/IOS XE, który chce zrozumieć Junos, a nie tylko nauczyć się kolejnej listy komend  
**Platforma praktyczna:** vSRX; uwagi o routerach MX i switchach EX/QFX podane tam, gdzie mają znaczenie  
**Zasada:** najpierw „dlaczego?” i „co się stanie?”, dopiero później CLI, skrócona ściąga i pytania kontrolne.

> **Umowa na ten rozdział:** nie będę Cię zasypywać tabelami z definicjami. Zaczniemy od problemu, pokażemy, co naprawdę robi urządzenie, następnie przewidzisz wynik, sprawdzisz go w Junos i wreszcie sam wywołasz kontrolowaną awarię. Gdy pojawi się termin egzaminacyjny, od razu dowiesz się, czy trzeba go **znać na pamięć**, **rozumieć**, czy **umieć sprawdzić**.

## Zanim zaczniemy — pewien incydent o 02:17

Wyobraź sobie nocną zmianę w NOC. Na ekranie monitoringu wszystko wygląda spokojnie: Twój router w oddziale odpowiada na ping, SSH działa, a w tablicy routingu widzisz prawidłową trasę do centrum danych. W tym samym czasie użytkownicy krzyczą, że aplikacja ERP nie działa.

— „Przecież router żyje!” — mówi kolega.

To prawda. Ale co tak naprawdę udowodnił? Jedynie to, że **jakiś rodzaj ruchu do routera** dotarł i otrzymał odpowiedź. Nie udowodnił, że router potrafi przepuścić **ruch przez siebie**.

Godzinę później spotykasz odwrotną sytuację: klienci pracują normalnie, ale ping do bramy nie odpowiada. Czy router jest zepsuty? Niekoniecznie. Mógł przestać przyjmować ping do własnego adresu, a jednocześnie bezbłędnie przekazywać pakiety klientów.

Te dwie pozornie sprzeczne sytuacje są drzwiami do architektury Junos. Przez cały rozdział będziemy szukać odpowiedzi na trzy pytania:

1. **Kto wymyśla drogę dla pakietu?**
2. **Kto naprawdę przekazuje pakiet?**
3. **Co się dzieje, gdy pakiet chce rozmawiać z samym routerem, zamiast przejść przez router?**

Jeśli po rozdziale będziesz umiał rozdzielić te trzy pytania w czasie awarii, podstawowa architektura Junos przestanie być teorią egzaminacyjną — zacznie działać w Twojej głowie jak mapa diagnostyczna.

### Jak będziemy pracować

W tym rozdziale zachowujemy dokładnie sześć podpunktów blueprintu:

- **2.1.** Architektura oprogramowania Junos OS.
- **2.2.** Rozdzielenie control plane i forwarding plane.
- **2.3.** Routing Engine (RE) — rola i funkcje.
- **2.4.** Packet Forwarding Engine (PFE) — rola i funkcje.
- **2.5.** Przetwarzanie ruchu tranzytowego (transit traffic).
- **2.6.** Exception traffic i relacja do control plane.

Po nich przejdziesz większe laboratorium **4 × vSRX + 2 × VPCS**, przewidziane specjalnie po to, by można było obserwować ścieżkę podstawową, zapasową i skutki różnych usterek. Dodatkowe sekcje LAB, TROUBLESHOOTING i TEST są autorskimi rozszerzeniami dydaktycznymi, a nie osobnymi oficjalnymi objectives.

### Trzy rodzaje wiedzy, których będę od Ciebie wymagać

**Na pamięć:** co oznaczają RE, PFE, RIB i FIB; funkcje rpd oraz mgd; rozróżnienie transit, host-bound i locally originated.

**Musisz zrozumieć:** jak informacja od protokołu routingu trafia do mechanizmu forwarding; dlaczego prawidłowa trasa nie gwarantuje sukcesu aplikacji; dlaczego ICMP nie jest automatycznie „exception traffic”.

**Musisz umieć wykonać:** znaleźć wersję systemu, sprawdzić procesy, interfejsy, trasę, forwarding table i — na SRX — sesję bezpieczeństwa; opisać, na którym etapie pakiet przestał iść dalej.

---

## 2.1. Architektura oprogramowania Junos OS [A]

### 2.1.1. Problem sprzed rozdzielenia ról: nie każ każdemu samochodowi dzwonić do dyspozytora

Wyobraź sobie ogromną firmę kurierską. Z sortowni wyjeżdżają tysiące paczek na minutę. Mamy dwa zadania:

- Ustalić plan: gdzie są miasta, jakie drogi istnieją i która trasa będzie najlepsza.
- Dla każdej paczki błyskawicznie odczytać etykietę i skierować ją na właściwy taśmociąg.

Gdyby sorter musiał za każdym razem zadzwonić do centralnego biura i zapytać „czy paczka do Łodzi ma dziś jechać przez Poznań?”, firma natychmiast by się zatkała. Przy tysiącach paczek na sekundę nawet świetny dyspozytor nie nadąży.

**Sieci mają dokładnie ten sam problem.** Routing dynamiczny, konfiguracja, sąsiedztwa OSPF i BGP to działalność inteligentna, ale stosunkowo kosztowna obliczeniowo. Przekazanie kolejnego pakietu na znaną już trasę powinno być szybkie i przewidywalne.

Stąd logiczny podział Junos:

- **Routing Engine (RE)** — dyspozytornia. Poznaje sieć, przelicza trasy, utrzymuje konfigurację i odpowiada na polecenia operatora.
- **Packet Forwarding Engine (PFE)** — automatyczna sortownia. Patrzy na pakiet i realizuje wcześniej przygotowaną decyzję przekazywania.

To bardzo dobra analogia, ale pamiętaj o jej granicach: na fizycznym MX lub EX/QFX wiele operacji danych wykonują wyspecjalizowane układy, a na vSRX funkcje forwarding i kontroli sesji realizuje programowe środowisko wirtualne. Nie zawsze są to dwa oddzielne fizyczne procesory.

**Zatrzymaj się na dziesięć sekund.** Jeśli do routera wpadnie milion podobnych pakietów w ciągu sekundy, czy milion razy trzeba uruchomić OSPF, żeby znaleźć drogę? Nie. OSPF pomaga przygotować tablice, a forwarding korzysta z przygotowanego wyniku.

### 2.1.2. Podróż informacji: od BGP do kabla

Załóżmy, że R1 otrzymał od sąsiada BGP informację: sieć 10.50.0.0/16 jest osiągalna przez 10.12.0.2.

~~~text
  R2 informuje R1: "znam drogę do 10.50.0.0/16"
                 |
                 v
         rpd analizuje BGP
                 |
                 v
         RIB — tablica tras
          (np. inet.0)
                 |
                 v
       wybór aktywnej ścieżki
                 |
                 v
    master forwarding table na RE
                 |
                 v
     instalacja/synchronizacja
      tablicy forwarding do PFE
                 |
                 v
          PAKIET PRZYCHODZI
   destination IP = 10.50.8.17
                 |
                 v
       PFE wybiera next hop
        i interfejs wyjściowy
                 |
                 v
       Pakiet wychodzi z R1
~~~

Kiedy później na tym samym routerze zrobisz **show route**, sprawdzasz perspektywę routingu. Kiedy wykonasz **show route forwarding-table**, przyglądasz się master forwarding table po stronie RE. To nie jest automatycznie bezpośredni zrzut sprzętowej tablicy z PFE.

I właśnie tu rodzi się pierwszy egzaminacyjny haczyk: **informacja o istnieniu trasy to nie to samo co dowód, że pakiet został poprawnie wysłany**.

### 2.1.3. Junos to nie jeden ogromny proces

W Cisco spotkałeś różne procesy i podsystemy IOS XE. Junos również składa się z wyspecjalizowanych usług, często określanych jako **daemons**. Nie musisz znać wszystkich. Na początku poznaj cztery postacie naszej historii.

**rpd (routing protocol daemon)** jest analitykiem transportu. Utrzymuje informacje protokołów takich jak OSPF, BGP czy IS-IS i uczestniczy w budowaniu tablic routingu. Kiedy tracisz sąsiedztwo BGP, rpd należy do pierwszych elementów, których stan chcesz rozumieć.

**mgd (management daemon)** jest kierownikiem przyjmującym zmiany. Kiedy wpisujesz polecenia konfiguracyjne i wykonujesz commit, zarządzanie konfiguracją wiąże się z mgd. Jeśli występuje problem z zatwierdzaniem zmian, nie oznacza to automatycznie, że forwarding przestał działać.

**chassisd** dogląda platformy. Zależnie od modelu zna stan kart, modułów, alarmów i komponentów chassis.

**Komponenty obsługi interfejsów, np. dcd**, dbają o odpowiednie aspekty konfiguracji i stanu interfejsów. Dokładny zestaw procesów oraz ich nazwy mogą zależeć od rodziny Junos i wydania.

A gdzie w tym wszystkim kernel? To fundament systemu operacyjnego, zapewniający procesom i mechanizmom systemowym wspólne usługi. Nie jest „piątym protokołem routingu”. W dużym uproszczeniu procesy i kernel współpracują w aktualizacji stanu potrzebnego do forwarding.

### 2.1.4. Co się stanie, jeśli jeden z tych procesów będzie mieć problem?

Wyobraź sobie, że na chwilę zabrakło analityka tras. Sortownia wciąż ma wydrukowaną ostatnią mapę dróg, więc **część ruchu może przez pewien czas działać**. Jednak gdy zamkną autostradę, nie ma komu przeliczyć poprawnej drogi.

To tłumaczy, dlaczego:
- problem z rpd może uderzyć w sąsiedztwa lub aktualizację tras, choć część istniejących ścieżek nadal przekazuje pakiety;
- problem z mgd może przeszkodzić operatorowi w konfiguracji, lecz nie dowodzi natychmiastowej utraty forwarding;
- awaria krytycznego komponentu lub synchronizacji może jednak dotknąć obu płaszczyzn.

Nie wyciągaj z analogii zbyt daleko idącej gwarancji. **RE/PFE separation nie zastępuje mechanizmów HA** — GRES, NSR czy redundantne RE mają osobne wymagania.

### 2.1.5. Sprawdźmy, z czym naprawdę rozmawiamy

Nie zakładamy, że wirtualny SRX zachowuje się identycznie jak MX czy EX. Najpierw identyfikujemy platformę i wersję.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show version
show system uptime
show chassis hardware
show system alarms
show chassis alarms
~~~

Co chcesz zobaczyć? Model i wersję, czas od ostatniego startu, wykryte elementy platformy i aktualne alarmy.

Potem zajrzyjmy „na zaplecze” i sprawdźmy procesy.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show system processes extensive
show system processes extensive | match "rpd|mgd|chassisd|dcd"
~~~

Pierwsza komenda pokazuje pełny kontekst. Druga jest tylko ułatwiającym filtrem tekstowym. **Brak danej nazwy w przefiltrowanym wyjściu nie dowodzi automatycznie awarii**. Proces może nie występować w danej postaci, nie mieścić się w wyświetlanym fragmencie albo być inaczej opisany.

**Przewidź wynik, zanim sprawdzisz:** gdy SSH działa, ale operator nie może zatwierdzić żadnej zmiany, który obszar obejrzysz najpierw? Management/configuration — mgd, logi, stan config, zasoby. Nie zaczynasz od wyłączania interfejsów.

### 2.1.6. Klasyczny Junos, Junos OS Evolved i vSRX

Junos OS to wspólna rodzina systemowa, ale **nie wszystkie urządzenia mają identyczne wnętrze**. Klasyczny Junos wywodzi się z FreeBSD. Junos OS Evolved używa jądra Linux i ma odmienną architekturę wdrożeniową. vSRX to wirtualna platforma security, na której dodatkowo istotna jest obsługa **flow-based**.

Dla poziomu Associate zapamiętaj logikę RE/PFE. Dla poziomów Specialist i Professional dodaj nawyk pytania: „**na jakiej platformie i wersji działa ten przykład?**”.

Jeśli kiedyś zobaczysz komendę działającą na MX, a niedostępną na vSRX, nie zakładaj błędu ucznia. Możesz po prostu mierzyć inną implementację.

### Mała powtórka po 2.1

Wyjaśnij własnymi słowami: dlaczego zwykły pakiet tranzytowy nie powinien pytać procesu rpd o trasę? Jeśli odpowiadasz „bo rpd przygotował informacje wcześniej, a forwarding korzysta z gotowej tablicy”, możemy iść dalej.

---

## 2.2. Control plane i forwarding plane — dwa różne światy [A]

### 2.2.1. Dlaczego router może być jednocześnie „żywy” i „martwy”?

Wracamy do nocnego incydentu. Router odpowiada na ping, ale aplikacja nie działa. To brzmi jak sprzeczność tylko wtedy, gdy w głowie mamy model „router to jedno pudełko, które albo działa, albo nie działa”.

Spróbujmy innego modelu. Na lotnisku jest **wieża kontroli lotów** oraz **pasy startowe z systemami obsługi ruchu**. Wieża może rozmawiać z pilotami, podczas gdy konkretny pas jest zamknięty. Samoloty mogą też wykonywać zaplanowane wcześniej operacje, choć wieża ma chwilowy problem z aktualizacjami harmonogramu. Mamy jeden port lotniczy, ale różne funkcje.

W Junos rozdzielamy:

**Control plane** — „skąd urządzenie wie, którędy prowadzi droga?”. Tutaj rodzą się relacje OSPF, sesje BGP, obliczenia tras i informacje dla routing table.

**Forwarding plane / data plane** — „co urządzenie robi z konkretnym pakietem?”. Tutaj pracują mechanizmy przełączania i routingu pakietu, kolejkowanie oraz funkcje właściwe platformie.

**Management plane** — „jak operator rozmawia z urządzeniem?”. SSH, CLI, NETCONF, logowanie i monitoring. Na klasycznych urządzeniach zarządzanie jest w znacznym stopniu związane z RE. Nie oznacza to automatycznie trzeciego fizycznego układu.

**Do zapamiętania na pamięć:** control plane przygotowuje decyzję; forwarding plane ją stosuje; management plane pozwala zarządzać i obserwować.

### 2.2.2. Co naprawdę dowodzi ping?

Nie mówmy „ping działa”. Zawsze dopytujmy: **skąd, dokąd, przez co?**

Wyobraź sobie trzy testy:

**Test A.** Laptop operatora ping → adres interfejsu R1. Pakiet chce dotrzeć do R1. To **host-bound**, a nie dowód tranzytu.

**Test B.** Operator uruchamia ping z CLI R1 → serwer w innym VLAN-ie. To **locally originated**: pakiet powstaje na urządzeniu, które testujesz.

**Test C.** PC-A ping → PC-B, przy czym ruch przechodzi przez R1. To **transit**: R1 ma go przepuścić dalej.

Wszystkie trzy wykorzystują ICMP. **Sam protokół ICMP nie mówi, z którą płaszczyzną masz do czynienia.** Decyduje przede wszystkim rola urządzenia w ścieżce pakietu oraz jego lokalne przeznaczenie.

Prawidłowa odpowiedź w troubleshootingu nie brzmi „ping przechodzi”, lecz np. „PC-A osiąga IP R1, ale nie osiąga PC-B; host-bound działa, transit nie został jeszcze potwierdzony”.

### 2.2.3. Wspólna historia jednego pakietu

Zróbmy z tego krótki film.

**Godzina 10:00.** R1 ma trasę do 198.51.100.0/24 przez R2. Informacja pojawiła się w routing table, a odpowiednie struktury forwarding zostały przygotowane.

**Godzina 10:01.** Z PC-A przychodzi pierwszy pakiet do 198.51.100.10.

**W control plane:** nie ma potrzeby od nowa negocjować OSPF dla tego pakietu.

**W data plane:** router rozpoznaje interfejs wejściowy, poszukuje pasującej trasy w strukturach forwarding, rozwiązuje sąsiedztwo L2/next-hop i przekazuje dalej. Na SRX dodatkowo sprawdzany jest odpowiedni stan sesji i security policy.

**Godzina 10:02.** R2 znika. Jeśli protokół routingu lub inny mechanizm wykryje zmianę, control plane musi przeliczyć wybór, a forwarding może otrzymać nowe wpisy. Jeśli użyłeś zwykłej trasy statycznej do next-hopu, który wciąż wydaje się dostępny, sam odległy zanik łącza nie musi uruchomić zmiany.

Widzisz związek przyczynowo-skutkowy? **To nie pakiety wymagają osobnego „odpytywania” RE, lecz RE aktualizuje ich wspólną instrukcję, kiedy sieć się zmienia.**

### 2.2.4. Trzy warstwy dowodów, nie trzy magiczne komendy

W czasie awarii zbierasz dowody stopniowo, zamiast wykonać jedną komendę i ogłosić zwycięstwo.

Najpierw pytanie: czy urządzenie **zna** trasę?

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show route table inet.0
show route 198.51.100.10
~~~

Jeśli widzisz poprawny prefiks, to dopiero początek. Teraz pytasz: czy przygotowano wpis forwarding?

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show route forwarding-table family inet
show route forwarding-table destination 198.51.100.10
~~~

Następnie pytasz o realną drogę i ewentualne zatrzymanie ruchu.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show interfaces terse
show interfaces ge-0/0/0 extensive
show arp
show security flow status
show security flow session summary
~~~

Tutaj interfejs i ARP wskazują, czy ścieżka do sąsiada ma sens. Komendy security flow dotyczą SRX, ponieważ nawet perfekcyjny routing na firewallu nie stanowi gwarancji zezwolenia na sesję.

**Pamiętaj:** forwarding table RE nie jest zawsze bezpośrednim odczytem PFE. Komenda **show pfe route** może udostępniać taką perspektywę na wspieranych platformach, ale na vSRX może jej nie być. Do diagnostyki dodaj obserwację faktycznych sesji i liczników.

### 2.2.5. Miniscenariusz: wysoki CPU RE

Na monitoringu widzisz 90% CPU Routing Engine. Jaki wniosek?

Kuszący: „Router nie da rady przekazywać ruchu”. To zbyt szybkie. Przyczyna może leżeć w rpd, licznych zmianach routingu, burzy ruchu host-bound lub innym procesie. Forwarding może w tym samym czasie nadal działać, choć sytuacja jest niepokojąca.

Odwrotność też jest możliwa: CPU RE jest niewielki, lecz port wyjściowy ma błędy lub security policy blokuje sesje.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show chassis routing-engine
show system processes extensive
show log messages | last 40
show interfaces terse
~~~

Szukaj zależności w czasie: kiedy wzrosło CPU, jaki proces obciąża system, czy zmieniły się sąsiedztwa, czy pojawiły się alarmy lub dropy? Nie restartuj rpd tylko dlatego, że „routing jest podejrzany”.

### Zrób pauzę i przewidź odpowiedź

Na R1 działa SSH. Nie działa TCP z PC-A do PC-B. Czy SSH potwierdziło kontrolę nad urządzeniem? Tak. Czy potwierdziło forwarding PC-A → PC-B? Nie. Pierwszy test użył management/host-bound, drugi wymagał przejścia przez data plane i — na vSRX — zaakceptowania sesji.

To rozróżnienie jest **wymagane do rozumienia JNCIA** i pozostaje ważne przez cały JNCIP.

---

## 2.3. Routing Engine — „mózg”, ale nie operator każdej paczki [A]

### 2.3.1. Po co w ogóle RE?

Przypomnij sobie GPS. Urządzenie może mieć mapę i potrafić obliczyć trasę. Jednak samochód nie przesyła każdej decyzji o skręcie do centralnego serwera. Kierowca korzysta ze wskazówek już wyliczonych. Kiedy zamknięta zostanie droga, mapa i trasa wymagają aktualizacji.

RE wykonuje tę inteligentną pracę: uruchamia i nadzoruje odpowiednie procesy, przechowuje konfigurację, buduje relacje protokołów, utrzymuje routing table, przygotowuje dane przekazywania i przyjmuje ruch skierowany do samego urządzenia.

W modelu egzaminacyjnym potraktuj RE jako **control + znaczną część management plane**, a PFE jako głównego wykonawcę forwarding.

### 2.3.2. RIB, active route, FIB — wytłumaczę to na trzech nawigacjach

Załóżmy, że masz trzy aplikacje do planowania trasy z Łodzi do Gdańska. Pierwsza proponuje autostradę, druga drogę krajową, trzecia przejazd przez Toruń. Lista dostępnych kandydatów nie oznacza, że wszystkie są jednocześnie realizowane.

- **RIB (Routing Information Base)** — zbiór tras znanych routerowi. W Junos najczęściej zaczynasz od IPv4 **inet.0** albo IPv6 **inet6.0**. Różne protokoły mogą dostarczać kandydatów dla tego samego prefiksu.
- **Active route** — kandydat wybrany do użytku zgodnie z zasadami selekcji i preferencji.
- **FIB (Forwarding Information Base)** — struktura potrzebna do przekazywania ruchu na podstawie wyników selekcji, odpowiednich next-hopów i interfejsów.

Nie myl dwóch różnych pytań: „którą z kandydackich tras wybrano do danego **prefiksu**?” i „który **prefiks** pasuje najlepiej do adresu docelowego?”. Pierwsze dotyczy selekcji trasy; drugie to **longest prefix match** w lookup forwarding.

Spróbujmy przykładu. R1 zna 198.51.100.0/24 przez R2, ale ma także default route 0.0.0.0/0 przez ISP. Pakiet do 198.51.100.10 pasuje do obu; wygra /24, bo jest bardziej szczegółowy. Jeśli istnieją dwie alternatywne trasy **do tego samego /24**, wtedy porównywane są odpowiednie preferencje/protokół i atrybuty.

### 2.3.3. Czytamy prawdziwy typ wyniku, a nie uczymy się kolumn

Zobaczmy uproszczony, **ilustracyjny**, niepochodzący z działającego laba, fragment outputu.

**Urządzenie: R1-vSRX — przykładowy output (illustrative)**

~~~text
user@R1> show route 198.51.100.10

inet.0: 12 destinations, 14 routes (12 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

198.51.100.0/24  *[Static/5] 00:01:22
                   > to 10.12.0.2 via ge-0/0/0.0
~~~

Nie próbuj zapamiętywać całego wyniku. Wypatrz **cztery rzeczy**, które opowiadają historię:

1. **inet.0** — jesteś w tabeli routingu IPv4.
2. **198.51.100.0/24** — do jakiej sieci odnosi się wpis.
3. **Static/5** — źródło trasy i jej preferencja.
4. **to 10.12.0.2 via ge-0/0/0.0** — plan wyjścia do następnego urządzenia.

Gwiazdka oznacza aktywną trasę według legendy. Ale odpowiedz teraz na pytanie: czy to dowodzi, że PC-B odbiera ping? **Nie!** Nie widzieliśmy jeszcze ARP, portu, security zones, security policies ani ruchu powrotnego.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show route 198.51.100.10
show route 198.51.100.0/24 detail
show route forwarding-table family inet
~~~

Pierwsza komenda znajduje pasującą trasę, druga pomaga przyjrzeć się prefiksowi, trzecia pokazuje master forwarding table po stronie RE. Na platformach obsługujących **show pfe route** można dodatkowo sprawdzić perspektywę PFE. Na vSRX nie opieramy całego laboratorium na tej komendzie.

**Pułapka egzaminacyjna:** obecność wpisu w RIB nie dowodzi ani poprawnej instalacji forwarding, ani zgodności polityk bezpieczeństwa, ani sprawności odległego hosta.

### 2.3.4. Czy Routing Engine także zajmuje się konfiguracją?

Tak. To drugi powód, dla którego RE jest ważny poza samym routingiem.

W Junos typowo przygotowujesz **candidate configuration**, sprawdzasz zmiany i dopiero poleceniem **commit** aktywujesz konfigurację. Przykład w IOS XE to bardziej bezpośrednie wprowadzanie zmian w running configuration, chociaż konkretne platformy Cisco również oferują mechanizmy transakcyjne.

**Zanim wykonasz zmianę**, odpowiadasz na dwa pytania: co dokładnie zmieniam i jak wrócę do stanu wcześniejszego?

**Urządzenie: R1-vSRX — tryb operational → configuration, tylko LAB**

~~~junos
configure
set system host-name R1-vSRX
show | compare
commit check
commit confirmed 5
~~~

**show | compare** pokazuje różnicę candidate względem aktywnej konfiguracji; **commit check** sprawdza poprawność konfiguracji, ale nie działanie aplikacji; **commit confirmed 5** aktywuje zmianę ze zwrotem po czasie, jeżeli nie potwierdzisz jej ponownie.

Gdy zmiana jest dobra:

**Urządzenie: R1-vSRX — tryb configuration, tylko LAB**

~~~junos
commit
exit
~~~

W razie błędu w konfiguracji — jeśli nie potwierdzisz — działa mechanizm wycofania; nie traktuj go jako zastępstwa testów. Na produkcji uwzględnij też równoległych operatorów i zasady change management. Mechanikę candidate/commit/rollback nauczymy się szerzej w rozdziale 3.

### 2.3.5. Kiedy RE pracuje ciężko i co wtedy robisz?

Wyobraź sobie, że dyspozytor dostaje co sekundę tysiące zgłoszeń o zamykanych drogach. Ciągle przelicza mapę. Tak może wyglądać niestabilny routing albo burza update'ów.

Możliwe przyczyny wysokiego CPU RE: flapping sąsiedztwa, ogromne zmiany RIB, intensywne logowanie, ruch wyjątkowy do control plane, problem usług zarządzania, awaria procesu.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show chassis routing-engine
show system processes extensive
show system uptime
show log messages | match "rpd|mgd|chassisd|dcd"
~~~

Zadaj sobie pytania: jaki proces najbardziej obciąża CPU? Czy problem narasta? Czy usterka rozpoczęła się po zmianie konfiguracji? Czy sąsiedztwa dynamiczne się zrywają? Czy forwarding host-host naprawdę ma straty?

**Nie restartuj procesu bez diagnozy.** Reset rpd na produkcyjnym routerze może być znacznie groźniejszy niż sam alarm CPU.

### 2.3.6. Dwa RE to nie to samo co dwa sprawne połączenia

Na większych platformach możesz mieć primary i backup Routing Engine. Brzmi pięknie: „mam drugą dyspozytornię”. Ale czy masz też drugi pas startowy? Czy dane routingowe są synchronizowane? Czy protokoły przetrwają failover bez restartu sąsiedztw?

Do tego służą dodatkowe technologie HA, np. GRES/NSR, nie samo fizyczne istnienie dwóch RE. Szczegóły będą w rozdziale 17.

### Ćwiczenie bez kabla: diagnoza w trzech zdaniach

**Objaw:** show route pokazuje trasę, SSH działa, aplikacja nie odpowiada.

Odpowiedź: „Znamy co najmniej prawidłowy dostęp administracyjny i istnienie wpisu w RIB. Nie potwierdziliśmy forwarding end-to-end. Następny etap: interfejsy, forwarding table, next-hop/ARP, security policy i sesje SRX oraz droga powrotna”.

Jeśli potrafisz to powiedzieć bez patrzenia w notatki, rozumiesz sens RE. Nie tylko jego rozwinięcie z trzech liter.

---

## 2.4. Packet Forwarding Engine — co się dzieje z pakietem po podjęciu decyzji? [A]

### 2.4.1. Kierowca nie musi dzwonić do nawigacji przy każdym skrzyżowaniu

Wiesz już, że RE pomaga urządzeniu zdecydować, dokąd wysłać ruch. Teraz wróćmy na chwilę do sortowni paczek. W budynku wisi kompletna mapa połączeń, ale sam plan nie przesunie ani jednej paczki. Potrzeba pasów, skanerów, zwrotnic i obsługi wyjazdu.

**PFE to właśnie ta wykonawcza część Junos.**

Na wybranych routerach i switchach Junipera wiele czynności wykonują układy ASIC i sprzętowe mechanizmy forwarding. Na vSRX wirtualne przetwarzanie pakietów działa programowo i jest powiązane ze stanową analizą bezpieczeństwa. Dlatego słowo „PFE” opisuje **funkcję logiczną**, nie zawsze jeden identyczny fizyczny moduł.

Kiedy pakiet przychodzi na interfejs, urządzenie musi ustalić:
- jaki to interfejs i do jakiego kontekstu należy;
- czy pakiet jest do lokalnego urządzenia, czy ma zostać przekazany dalej;
- jaka droga i next-hop są właściwe;
- jakie reguły filtering, QoS lub security mają zastosowanie;
- jak przygotować wyjściową ramkę i na który interfejs ją wysłać.

Jeżeli przyjrzysz się temu uważnie, zobaczysz ważny szczegół: PFE nie „zna świata” w sposób, w jaki znają go OSPF lub BGP. **PFE stosuje przygotowane instrukcje forwarding.**

### 2.4.2. Jeszcze jeden problem: „routing działa”, a firewall odmawia

Tu pojawia się ważna różnica między typowym routerem a **vSRX**, którego użyjemy do ćwiczeń.

Wyobraź sobie most prowadzący do miasta. Inżynier drogowy narysował mapę i jest pewien, że most istnieje. Jednak na moście stoi punkt kontroli bezpieczeństwa. Sam fakt, że **droga istnieje**, nie znaczy, że każdy samochód **dostanie pozwolenie na przejazd**.

Na SRX tym punktem kontrolnym są m.in. security zones, policies, NAT i **stateful flow processing**.

Gdy pojawia się pierwszy pakiet nowego strumienia, SRX musi ustalić, czy ruch jest dozwolony, jakie działania zastosować i czy utworzyć sesję. Kolejne pakiety tej samej sesji mogą korzystać z już zapisanego stanu. Mechanika jest bogatsza niż samo lookup w FIB.

**Nie myl dwóch rodzajów pracy:**
- rpd **nie** jest po prostu procesem analizującym każdy nowy strumień;
- analiza sesji security odbywa się w odpowiednim datapath SRX, a nie dlatego, że każdy pakiet trzeba wysłać do routingu na RE.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show security flow status
show security flow session summary
show security flow session
show security flow statistics
~~~

Gdy zobaczysz tryb **flow based**, spodziewaj się konieczności badania sesji, stref i polityk. Jeśli rzeczywisty system działa inaczej, nie zmieniaj globalnego trybu forwarding „żeby zgadzał się z podręcznikiem”. Zidentyfikuj wersję i zamierzony sposób działania labu.

### 2.4.3. Najdłuższy prefiks — dlaczego 140 nie pojedzie „dowolną drogą”

Ustalmy sytuację.

Router ma takie wpisy forwarding:

- default 0.0.0.0/0 przez ISP;
- 198.51.100.0/24 przez R2;
- 198.51.100.128/25 przez R3.

Do routera przychodzi pakiet z docelowym IP **198.51.100.140**.

Wyobraź sobie adresowanie przesyłek:
- „Polska” to bardzo ogólna etykieta.
- „Województwo łódzkie” jest bardziej precyzyjne.
- „Łódź, konkretny rejon” wskazuje jeszcze węższy obszar.

Dopasowanie do **/25** jest najdokładniejsze. Mimo istnienia /24 i /0 router wybiera /25. W skrócie to **Longest Prefix Match (LPM)**.

Teraz uważaj na typowy haczyk: jeśli masz **dwie trasy do tego samego 198.51.100.128/25**, nie „wygrywa dłuższy prefiks”, bo obie są równie długie. Musisz rozpatrzyć selekcję tras i stosowne preferencje. LPM służy do dopasowania docelowego adresu do prefiksu, a nie do wybierania między kandydatami do identycznej sieci.

**Pytanie kontrolne:** pakiet do 198.51.100.50 — którą drogą? Przez R2, ponieważ 50 nie należy do 198.51.100.128/25, ale należy do /24.

### 2.4.4. Co naprawdę mówią „route”, „forwarding table” i „PFE”?

To temat, przy którym nietrudno nauczyć się złej diagnozy. Dlatego zamiast tabeli wyobraź sobie trzy kartki trzymane przez trzy osoby:

**Kartka dyspozytora:** „znam dwie drogi, wybrałem tę przez R2” — tak myślisz o RIB.

**Kartka przy sorterze:** „dla prefiksu 198.51.100.0/24 użyj next-hop R2” — to master forwarding table przygotowana po stronie RE.

**Kartka w maszynie wykonawczej:** lokalnie zainstalowane informacje przekazywania, których faktycznie używa PFE.

Jeśli te kartki się rozjadą, możesz zobaczyć trasę w RIB, ale nadal mieć problem z forwarding. A nawet jeśli kartki są identyczne, pakiet może utknąć na wyjściu przez problem z ARP, interfejsem, MTU, filtrem albo policy.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show route table inet.0
show route forwarding-table family inet
show route forwarding-table destination 198.51.100.10
~~~

Na sprzętowych platformach, które to wspierają, polecenie **show pfe route** może dać bezpośrednią perspektywę PFE. Na vSRX sprawdź jego dostępność, zanim zbudujesz na nim cały proces troubleshootingu.

**Urządzenie: wybrana platforma Junos obsługująca polecenie — tryb operational**

~~~junos
show pfe route
~~~

**Najważniejsze do zapamiętania:** **show route forwarding-table przedstawia master forwarding table na RE, nie gwarantuje pełnego odczytu hardware PFE.** Gdy jesteś na vSRX, realny przesył pakietów potwierdzają również testy end-to-end i **show security flow session**.

### 2.4.5. Gdy dataplane ma problem, nie musi umrzeć cały router

Przypadek z produkcji, który może się zdarzyć na platformie sprzętowej: dostęp do RE jest normalny, sąsiedztwa routingu są stabilne, a ruch przez porty określonego modułu nie przechodzi. Nie ma sensu udawać, że sam fakt działającego SSH wyklucza problem z kartą forwarding.

**Urządzenie: fizyczny MX/EX/QFX (jeśli polecenia dostępne) — tryb operational**

~~~junos
show chassis fpc
show chassis fpc pic-status
show interfaces terse
show interfaces extensive
show chassis alarms
show log messages | match "fpc|pfe|chassisd"
~~~

Na vSRX zamiast szukać nieistniejącego ASIC analizuj **wirtualne interfejsy, route/FIB, stan sesji i statystyki flow**.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show interfaces terse
show interfaces ge-0/0/0 extensive
show route forwarding-table family inet
show security flow session summary
show security flow statistics
~~~

**Rzecz do zrozumienia:** dostęp administracyjny, protokół routingu, forwarding i stateful security to różne etapy. Diagnoza jest dobra wtedy, gdy umiesz wskazać, **który etap masz udowodniony, a którego jeszcze nie**.

---

## 2.5. Transit traffic — idziemy z pakietem od źródła do celu [A]

### 2.5.1. Kiedy pakiet jest tranzytem?

Załóżmy, że stoi przed Tobą R1 z dwoma sieciami:

- PC-A: 192.0.2.10/24, gateway 192.0.2.1;
- PC-B: 198.51.100.10/24, gateway 198.51.100.1;
- R1 łączy obie sieci.

PC-A chce porozmawiać z PC-B. Adres docelowy **nie należy do R1**. Router powinien przeprowadzić pakiet przez siebie — to **transit traffic**.

To brzmi prosto, ale teraz prześledzimy dokładnie, **co musi się zgadzać**, zanim ping zadziała. W ten sposób zrozumiesz, dlaczego w rzeczywistej awarii „routingu” wina często leży w zupełnie innym miejscu.

### 2.5.2. Film w siedmiu scenach: jedna wiadomość ICMP

**Scena 1 — PC-A wybiera bramę.** Host widzi, że adres 198.51.100.10 nie jest w jego /24. Nie szuka MAC docelowego PC-B w swojej sieci. Wysyła pakiet do **default gateway**.

**Scena 2 — ARP.** PC-A potrzebuje MAC interfejsu R1 w swojej sieci. Jeśli wpisu jeszcze nie ma, wykonuje ARP. Dopóki nie zna odpowiedniego MAC, nie potrafi poprawnie zaadresować wyjściowej ramki Ethernet.

**Scena 3 — R1 odbiera ramkę.** Docelowy MAC to MAC interfejsu R1. Ale **destination IP** pakietu to 198.51.100.10 — host za routerem.

**Scena 4 — decyzja forwarding.** R1 odnajduje odpowiedni wpis w tablicy forwarding i określa next-hop oraz interfejs wyjściowy. Na vSRX do tej ścieżki dochodzą mechanizmy session/security.

**Scena 5 — kontrola bezpieczeństwa.** Jeżeli nie ma odpowiedniego zezwolenia w security policy, pakiet może zostać odrzucony nawet przy idealnym route lookup.

**Scena 6 — zmiana ramki.** W routingu L3 przejście do kolejnego segmentu zwykle oznacza zmianę **źródłowego i docelowego MAC**. Adresy IP nadawcy i odbiorcy na ogół pozostają takie same, o ile nie ma NAT lub innej transformacji. IPv4 TTL zostaje zmniejszony, a w IPv6 zmniejsza się Hop Limit.

**Scena 7 — odpowiedź.** PC-B musi mieć **drogę powrotną** do PC-A. Jeśli request dotarł, ale reply nie potrafi wrócić, ping również się nie powiedzie.

Możesz to zilustrować tak:

~~~text
PC-A                  R1-vSRX                   PC-B
192.0.2.10       ge-0/0/1  ge-0/0/2        198.51.100.10
GW 192.0.2.1     192.0.2.1  198.51.100.1 GW 198.51.100.1
   |                  |           |                |
   |  L2 DST = R1     |           |                |
   |  IP DST = PC-B   |           |                |
   +----------------->|           |                |
                      |  routing  |                |
                      |  policy   |                |
                      |  flow     |                |
                      +-----------+--------------->|
                           L2 DST = PC-B            |
                           IP DST = PC-B            |
~~~

Nawet prosty ping PC-A → PC-B jest więc małym łańcuchem zależności. **Powód i skutek** są ważniejsze niż mechaniczne wpisywanie komend.

### 2.5.3. Dlaczego Ethernet się zmienia, a IP często nie?

Wyobraź sobie przesyłkę wysyłaną przez kilka sortowni. Na paczce nadal widnieje ostateczny adres odbiorcy. Zmienia się jednak etykieta lokalnego transportu: z którego magazynu do której następnej sortowni ma trafić.

W Ethernet taką **lokalną etykietą** są adresy MAC na danym hopie. W IP adresem docelowym paczki jest IP końcowego hosta.

Na wejściu R1:

- L2 destination MAC — interfejs R1 od strony PC-A;
- IP source — 192.0.2.10;
- IP destination — 198.51.100.10.

Na wyjściu z R1 do sieci PC-B:

- L2 source MAC — wyjściowy interfejs R1;
- L2 destination MAC — MAC PC-B;
- IP source — nadal 192.0.2.10;
- IP destination — nadal 198.51.100.10.

Jest istotny wyjątek: **NAT** może zmienić IP, dlatego mówimy „zwykle”, a nie „zawsze”.

### 2.5.4. Transit na SRX: jeszcze jeden warunek

Pamiętasz kontrolę na moście z sekcji 2.4? Właśnie tu ona działa.

Na vSRX możesz mieć:
- poprawne adresowanie obu interfejsów;
- prawidłową trasę connected;
- aktywne porty;
- a mimo to brak pingu PC-A → PC-B.

Dlaczego? Bo **security policy nie zezwala na sesję między odpowiednimi strefami**.

W tym miejscu myślenie wyłącznie kategoriami routera potrafi wprowadzić inżyniera Cisco w błąd. SRX to nie tylko „Junos jako router”; jego stateful security jest pełnoprawną częścią datapath.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show security zones
show security policies
show security flow status
show security flow session source-prefix 192.0.2.10/32
show security flow statistics
~~~

Co mówi wynik? **Zone** wskazuje logiczny kontekst wejścia i wyjścia. **Policy** decyduje, czy nowy ruch jest dopuszczony. **Session** mówi, czy dla obserwowanego ruchu powstał stan; liczniki mogą pomóc ustalić, czy pakiety rzeczywiście wędrują.

Jeżeli sesji nie ma, nie wyciągaj od razu wniosku „policy deny”. Pakiet mógł jeszcze nie dotrzeć do SRX, trafić do złej sieci, mieć zły gateway lub zostać odrzucony wcześniej.

### 2.5.5. Dlaczego routing z R1 może działać, a PC-A → PC-B nie?

Przykładowo wykonujesz ping **z CLI R1** do PC-B. Wynik jest poprawny. Następnie PC-A próbuje tego samego i nie ma odpowiedzi.

Czy R1 oszukuje? Nie. Te próby sprawdzają inne ścieżki:

- ping z R1 jest generowany lokalnie i może używać innego adresu źródłowego, interfejsu oraz ścieżki przetwarzania;
- ping z PC-A jest ruchem tranzytowym, wymaga poprawnego gateway, routingu, stref, policy, sesji i drogi zwrotnej.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
ping 198.51.100.10 source 198.51.100.1 count 5
~~~

**Urządzenie: PC-A — konsola VPCS**

~~~text
ping 198.51.100.10
~~~

Nie zastępuj drugim wynikiem pierwszego ani odwrotnie. Dobre testy nie tylko mówią, że „działa” — mówią, **co zostało przetestowane**.

### 2.5.6. Wnioski, które zapamiętasz bez tabeli

Jeżeli test H1 → H2 nie działa, przejdź kolejno: IP/maska/gateway H1 → ARP do bramy → ingress na R1 → routing/next-hop → zone/policy/session SRX → egress → host H2 → droga powrotna.

To jest praktyczny model przyczyna → skutek, który przyda Ci się w każdej technologii od VLAN po BGP.

---

## 2.6. Exception traffic — kiedy pakiet puka do drzwi routera? [A]

### 2.6.1. Dwa podobne auta, dwa zupełnie różne zadania

Wracamy do lotniska. Jeden samochód jedzie **przez teren lotniska** na drugą stronę miasta. Inny dojeżdża **pod wieżę kontroli**, bo ma dokumenty dla personelu. Oba przejechały przez tę samą bramę. Ale jeden jest ruchem tranzytowym, drugi ma zostać obsłużony przez samą administrację lotniska.

W sieci podobnie:

- **Transit**: docelowy adres IP należy do innego hosta, urządzenie ma pakiet przesłać dalej.
- **Host-bound / local-destination**: docelowy adres IP należy do samego urządzenia albo pakiet jest przeznaczony dla jego usług, np. SSH, BGP, OSPF czy ping do interfejsu.
- **Locally originated**: urządzenie samo tworzy pakiet, np. z CLI uruchamiasz ping, protokół wysyła update lub system wysyła syslog.

**Exception traffic** to szersze określenie ruchu wymagającego specjalnej obsługi, często w związku z control plane. Może obejmować ruch host-bound, wybrane pakiety protokołów i sytuacje nietypowe dla normalnego forwarding, np. wyczerpanie TTL. Nie każdy wyjątek musi być realizowany identycznie na każdym PFE, a część reakcji może być obsługiwana specjalizowanymi mechanizmami platformy.

**Nie zapamiętuj błędnej reguły „ICMP = exception”.** ICMP Echo Request z PC-A do PC-B, przechodzący przez R1, to normalny ruch tranzytowy. ICMP Echo Request do adresu R1 jest natomiast ruchem do lokalnego urządzenia.

### 2.6.2. Skąd się bierze packet punt?

Wyobraź sobie zautomatyzowaną sortownię. Większość paczek idzie taśmociągiem. Ale raz na jakiś czas skaner zobaczy szczególny przypadek: paczkę, której system automatyczny nie może obsłużyć zwykłą regułą. Odkłada ją do stanowiska operatora.

W sieciach analogiczna operacja może nazywać się **punt** — skierowanie wybranego pakietu do specjalnej obsługi, często do control plane. To nie jest jednak obietnica, że **każdy** pakiet szczególnego typu w każdej wersji Junos zawsze trafia fizycznie do CPU RE.

Przykłady problemów i wyjątków:
- pakiet przeznaczony do lokalnej usługi SSH/BGP/OSPF;
- pakiet wymagający reakcji przy TTL/Hop Limit równym zero;
- niektóre zdarzenia kontroli lub obsługi sąsiedztwa, zależnie od platformy;
- pakiet, którego nie można przetworzyć zwykłą ścieżką forwarding, zależnie od sprzętu i funkcji.

Zadanie inżyniera: rozpoznać **dlaczego taki pakiet może trafić do innej ścieżki** i jaki będzie efekt dla działania sieci.

### 2.6.3. Dlaczego ping do bramy może nie działać, choć router doskonale routuje

Pamiętasz dwa scenariusze z początku rozdziału? Teraz potrafisz je rozdzielić.

Na SRX możesz mieć politykę pozwalającą PC-A rozmawiać z PC-B, ale nie zezwolić PC-A na pingowanie **samego SRX**. SRX ma do tego osobne ustawienia **host-inbound-traffic**. Polityka transit **nie** jest automatycznie pozwoleniem na dostęp do usług routera.

To ma sens również z punktu widzenia bezpieczeństwa. Możesz chcieć, aby klienci mogli przejść przez sieć, ale nie skanowali interfejsów i usług zarządzania.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show security zones
show configuration security zones | display set
show security policies
~~~

Szukasz dwustopniowej historii:
1. Czy interfejs wejściowy ma strefę i czy zezwolono na **host-inbound system-services ping** do samego SRX?
2. Czy osobna policy pozwala na **transit** do innej sieci?

Te dwie decyzje można zmienić niezależnie, co pokażemy w laboratorium.

### 2.6.4. BGP to też rozmowa z samym routerem

Masz kolegę, który skonfigurował adresy i IP connectivity, ale BGP nie przechodzi z Active do Established. Mówi:

— „Przecież transit policy pozwala na wszystko!”

Problem polega na tym, że BGP na adresie R1 nie jest zwykłym przekazywaniem pakietów host-host. To ruch do **lokalnego procesu routingu**. Na SRX trzeba sprawdzić obsługę protokołu w host-inbound oraz samą konfigurację BGP.

**Urządzenie: R1-vSRX — tryb operational**

~~~junos
show bgp summary
show configuration protocols bgp | display set
show configuration security zones | display set
show log messages | match "rpd|bgp"
~~~

Nie budujemy tutaj pełnego BGP. Chcę jedynie, abyś przestał utożsamiać security policy dla transit z możliwością komunikowania się z control plane SRX.

### 2.6.5. Traceroute pokazuje gwiazdki — czy to przerwa w sieci?

Wyobraź sobie, że traceroute pokazuje odpowiedzi z R1, potem kilka gwiazdek, a następnie docelowy serwer odpowiada prawidłowo.

Wzrok ciągnie Cię do gwiazdek. Ale **traceroute bada również zachowanie wyjątków**: TTL na danym hopie wygasa, a urządzenie może wygenerować lub ograniczyć odpowiedź ICMP Time Exceeded. Brak odpowiedzi z pośredniego routera nie dowodzi automatycznie, że normalny transit przez ten router nie działa.

Tak samo router może ograniczać pakiety do RE, ale przekazywać ruch użytkownika normalnie.

Zrób sobie małą zasadę diagnostyczną: **zawsze rozdziel odpowiedzi generowane przez router od pakietów, które router tylko przekazuje**.

### 2.6.6. Jak oglądać ruch do RE bez udawania, że widzimy cały dataplane?

W Junos masz narzędzie **monitor traffic**, ale traktuj je rozsądnie. W typowych zastosowaniach obserwujesz pakiety widoczne z perspektywy RE/host stack. Nie jest to uniwersalny sniffer każdej ramki przechodzącej przez sprzętowy PFE.

**Urządzenie: R1-vSRX — tryb operational, tylko izolowany LAB**

~~~junos
monitor traffic interface ge-0/0/2 no-resolve matching "icmp" count 10
~~~

Uruchom tę komendę podczas pingowania **lokalnego IP R1** z PC-A. Potem uruchom ping PC-A → PC-B. Jeśli monitor traffic nie pokaże ruchu transit, a test end-to-end oraz sesja SRX pokazują pakiety, nie traktuj tego jako sprzeczności.

**Dobrze zapamiętaj:** brak pakietu w zrzucie z RE **nie jest dowodem**, że nie dotarł do urządzenia.

### 2.6.7. Jak chroni się control plane

Dlaczego w ogóle ograniczamy ping/SSH do interfejsów routera? Bo RE musi mieć zasoby na właściwe zadania — protokoły, zarządzanie i obsługę wyjątków. Nie chcesz, by tysiące losowych żądań odebrały mu zdolność aktualizacji tras.

Na wybranych platformach stosuje się filtry na interfejsie loopback (np. **lo0**), mechanizmy ochrony DDoS/policery control-plane. Na SRX dodatkowo istnieje **host-inbound-traffic** ograniczające lokalnie dostępne usługi i protokoły. To różne mechanizmy — nie przepisuj bez zastanowienia filtra MX na SRX.

**Urządzenie: R1-vSRX — tryb configuration, tylko izolowany LAB**

~~~junos
configure
set security zones security-zone LAB interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
commit check
commit confirmed 5
~~~

Przykład dotyczy interfejsu **ge-0/0/2.0** dopiero po potwierdzeniu mapowania EVE-NG. Jeśli interfejs już należy do innej strefy, musisz świadomie dostosować przykład, a nie dopisywać go do dwóch stref.

### Tu sprawdzamy, czy naprawdę rozumiesz rozdział

PC-A może pingować PC-B przez R1, ale nie może pingować IP R1. Co musi być prawdą?

Nie wiemy jeszcze, **dlaczego** nie pingujesz R1. Wiemy natomiast, że **funkcja transit jest co najmniej w jakimś zakresie sprawna**, a host-bound wymaga osobnej diagnostyki. Na SRX sprawdzisz host-inbound ping i stan interfejsu. Na innych platformach możesz szukać np. filtrów control plane lub policerów.

Właśnie o to chodzi w dobrej analizie — oddzielić udowodnione fakty od przypuszczeń.

---

## 2.LAB. Laboratorium prowadzone — zbudujemy własną małą sieć [EXT]

### 2.LAB.1. Dlaczego aż cztery vSRX, skoro do rozdziału 2 wystarczy jeden?

Do pokazania różnicy między host-bound a transit **jeden SRX jest wystarczający**. Ale chcemy zbudować coś, co będzie towarzyszyło Ci również przy następnych rozdziałach. Dzięki czterem routerom zobaczysz nie tylko pakiet na jednym urządzeniu, lecz także **dwie alternatywne drogi przez sieć**.

Wyobraź sobie dwie autostrady łączące miasta. Jeśli autostrada północna zostanie zamknięta, chcesz wiedzieć, kiedy i dlaczego ruch pojedzie południową. To doskonała okazja do rozróżnienia wiedzy routingu, tablic forwarding i realnych pakietów.

~~~text
                 R2-vSRX
                /       \
               /         \
PC-A --- R1-vSRX         R4-vSRX --- PC-B
               \         /
                \       /
                 R3-vSRX

Podstawowa trasa: PC-A → R1 → R2 → R4 → PC-B
Alternatywna:     PC-A → R1 → R3 → R4 → PC-B
~~~

**Ważne:** cztery vSRX po 4 GiB RAM to orientacyjnie **16 GiB przydzielonej pamięci**, bez EVE-NG i systemu hosta. Nie uruchamiaj ich jednocześnie, jeśli Twój host nie ma wystarczających zasobów. Jeżeli jeszcze nie masz legalnie pobranego obrazu QCOW2, specyfikację można przygotować bez uruchamiania urządzeń. Rozdział nie udaje, że topologia została już przetestowana.

**Konfiguracja jest przykładem do LAB, nie gotowym szablonem produkcyjnym**. Używamy czytelnych adresów i szerokiej polityki allow w izolowanym środowisku wyłącznie po to, by badać działanie datapath. Nie kopiuj tego do firmowego SRX.

### 2.LAB.2. Rozrysuj adresy, zanim wpiszesz pierwszą komendę

Najpierw opowiedzmy sobie trasę od PC-A do PC-B. PC-A ma sieć **192.0.2.0/24** i bramę 192.0.2.1 na R1. PC-B jest w **198.51.100.0/24** i ma bramę 198.51.100.1 na R4. R1 do R4 może dojść przez R2 albo R3.

Wybieramy następujące łącza:

- R1 ↔ R2: 10.12.0.0/30, R1 .1, R2 .2.
- R1 ↔ R3: 10.13.0.0/30, R1 .1, R3 .2.
- R2 ↔ R4: 10.24.0.0/30, R2 .1, R4 .2.
- R3 ↔ R4: 10.34.0.0/30, R3 .1, R4 .2.
- PC-A ↔ R1: 192.0.2.0/24.
- R4 ↔ PC-B: 198.51.100.0/24.

Adresy 192.0.2.0/24 i 198.51.100.0/24 należą do zakresów dokumentacyjnych RFC 5737. Świadomie izolujemy lab i **nie** projektujemy routingu tych sieci do publicznego Internetu.

Dopiero teraz krótka mapa portów, aby uniknąć zgadywania:

| Urządzenie | Port 0 | Port 1 | Port 2 |
|---|---|---|---|
| R1 | ge-0/0/0 → R2 | ge-0/0/1 → R3 | ge-0/0/2 → PC-A |
| R2 | ge-0/0/0 → R1 | ge-0/0/1 → R4 | — |
| R3 | ge-0/0/0 → R1 | ge-0/0/1 → R4 | — |
| R4 | ge-0/0/0 → R2 | ge-0/0/1 → R3 | ge-0/0/2 → PC-B |

To **plan mapowania**, nie gwarancja dla konkretnego obrazu. vSRX bywa wyposażony w osobny interfejs zarządzający **fxp0**, a kolejność kart NIC w EVE może wpływać na widoczne porty. Przed wklejeniem konfiguracji **koniecznie** zweryfikuj:

**Urządzenia: R1-vSRX, R2-vSRX, R3-vSRX i R4-vSRX — każde osobno, tryb operational**

~~~junos
show version
show interfaces terse
show security flow status
show security zones
~~~

Jeśli portów brakuje albo mają inne nazwy, zatrzymujesz się i poprawiasz mapowanie. Nie „naprawiasz” problemu przypadkowym usunięciem fabrycznej konfiguracji management.

### 2.LAB.3. Najpierw plan działania, potem uruchomienie

Wykonaj ćwiczenia w następującej kolejności:

**Etap A:** skonfiguruj adresy interfejsów i sprawdź linki oraz ARP między sąsiadami.

**Etap B:** dodaj trasy statyczne tak, aby ścieżka przez R2 była podstawowa, a przez R3 alternatywna.

**Etap C:** skonfiguruj logiczne strefy bezpieczeństwa na vSRX i zezwól na laboratoryjny transit oraz konkretne host-inbound ping.

**Etap D:** sprawdź ruch host-host, trasę, sesje oraz jej powrót.

**Etap E:** kolejno zasymuluj: zablokowany host-inbound ping, zablokowany transit oraz awarię podstawowego łącza.

Dlaczego w tej kolejności? Bo inaczej bardzo łatwo pomylić trzy niezależne przyczyny awarii: brak route, brak policy oraz problem host-bound. Dobry lab powinien uczyć **izolowania zmiennej**, nie losowego zmieniania pięciu rzeczy naraz.

