# Rozdział IV. Pierwsze uruchomienie Junos — od dostępu przez konsolę do bezpiecznie zarządzanego routera

**JUNOS Enterprise Routing & Switching • JNCIA-Junos → JNCIS-ENT → JNCIP-ENT**  
**Unified Blueprint:** 4.1–4.14 [A] + 4.15 [LAB]  
**Dla kogo:** Network Engineer z około rocznym doświadczeniem, który zna podstawy IP, routera, switcha i VLAN, ale nie musi znać Cisco ani żadnego wcześniejszego egzaminu sieciowego.  
**Wersja:** 3.0, 20.11.2026  
**Lab referencyjny:** [DIAMOND-6R](../../labs/reference-diamond-6vsrx/README.md). Do ćwiczeń zarządzania dodamy *opcjonalnie* odizolowany segment administracyjny, bez zmiany sześciu istniejących połączeń danych.  
**Status:** **PLANNED / NOT RUNTIME VALIDATED.** Komendy są przykładami do sprawdzenia na konkretnym vSRX i release; nie stanowią wyników wykonanych testów.

## Co musi działać, zanim nowy router oddamy administratorowi?

Postawiono nowy router R1. Na ekranie konsoli widać nazwę urządzenia i znak zachęty Junos. Router może mieć aktywne interfejsy i poprawną tablicę routingu. Czy jest gotowy do pracy w firmie?

Jeszcze nie. Jeżeli tylko jedna osoba zna hasło administracyjne, serwer monitoringu nie widzi urządzenia, jego zegar wskazuje niepoprawną godzinę, a konfiguracja istnieje tylko na lokalnym dysku, to pierwsza poważniejsza awaria może być bardzo trudna do naprawienia. Niewiele pomoże znajomość OSPF czy BGP, jeśli nie możesz bezpiecznie zalogować się na router albo ustalić, co wydarzyło się kilka minut wcześniej.

Ten rozdział dotyczy właśnie tej części pracy sieciowej, którą często wykonuje się *przed* uruchomieniem właściwych usług. Będziemy konfigurować urządzenie tak, żeby dało się nim zarządzać, ograniczyć dostęp do uprawnionych osób, otrzymywać informacje o problemach i odzyskać działanie po pomyłce. Nie zaczniemy jednak od dwudziestu komend wklejonych do terminala. Najpierw zrozumiemy, **co każda usługa wnosi do działania sieci**.

Z poprzednich dwóch rozdziałów znasz już trzy rzeczy: Junos ma osobny obszar konfiguracji roboczej (*candidate*), zmiany aktywujesz przez *commit*, a sprzętowy lub wirtualny router może przekazywać pakiety niezależnie od tego, czy usługa administracyjna działa prawidłowo. Teraz połączymy te wiadomości w procedurę uruchomienia urządzenia.

### Jedna topologia, kilka różnych ról

W tej książce używamy DIAMOND-6R, a więc sześciu vSRX w stałym układzie. R5 jest routerem przy sieciach A, R6 przy sieciach B, R1/R4 są brzegami rdzenia, a R2/R3 dają dwie alternatywne drogi.

~~~mermaid
flowchart LR
  R5["R5 CE-A<br/>192.0.2.10/24"] ---|"LAN-A<br/>192.0.2.0/24"| R1["R1 EDGE-A<br/>192.0.2.1/24"]
  R1 ---|"10.12.0.0/30"| R2["R2 CORE-1"]
  R1 ---|"10.13.0.0/30"| R3["R3 CORE-2"]
  R2 ---|"10.24.0.0/30"| R4["R4 EDGE-B<br/>198.51.100.1/24"]
  R3 ---|"10.34.0.0/30"| R4
  R4 ---|"LAN-B<br/>198.51.100.0/24"| R6["R6 CE-B<br/>198.51.100.10/24"]
~~~

*Rysunek 4.1. Nie zmieniamy żadnego z czterech łączy rdzeniowych. R1–R5 i R4–R6 nadal należą do referencyjnych LAN-A/LAN-B. Pełne adresy interfejsów, loopbacki i nazwy portów: [DIAMOND-6R README](../../labs/reference-diamond-6vsrx/README.md).*

Do samej nauki kont, SSH i rescue wystarczy jedna izolowana instancja R1. Do sprawdzenia logowania sieciowego przyda się R5 jako drugi router lub niewielka Linux VM. Do NTP, syslog, SNMPv3 i archiwizacji potrzebujesz **prawdziwych usług po drugiej stronie**, np. Linux VM jako \`LAB-SERVICES\`. Sam router nie stworzy nam serwera NTP ani odbiornika syslog tylko dlatego, że go wpisaliśmy do konfiguracji.

Jeżeli chcesz zbudować **oddzielną sieć administracyjną OOB**, można użyć izolowanego segmentu ćwiczebnego **203.0.113.0/24**, z serwerami \`LAB-ADMIN 203.0.113.20\` i \`LAB-SERVICES 203.0.113.30\`. To **opcjonalne rozszerzenie zarządzania**, a nie siódme łącze między routerami DIAMOND-6R. Adres zarządzania R1, numer jego interfejsu OOB i sposób podłączenia do EVE-NG ustalisz **dopiero po sprawdzeniu konkretnego obrazu vSRX**. Nie przypisuj z góry \`fxp0\` do losowej karty QEMU. Oddzielny segment jest w pełni odizolowany od Internetu i środowiska firmowego.

---

## 4.1. Factory-default state — dlaczego dwa nowe Junipery mogą wyglądać zupełnie inaczej? [A]

### Urządzenie „fabryczne” nie zawsze oznacza pusty system

Jeżeli rozpakujesz nowy router i switch tego samego producenta, możesz oczekiwać podobnego menu. Ale na jednym urządzeniu producent mógł przygotować uproszczoną konfigurację na potrzeby pierwszego połączenia, a na drugim jedynie dostęp konsolowy. Dotyczy to również rodziny Juniper: EX, MX, SRX i vSRX służą do różnych zadań.

**Factory-default configuration** to ustawienia przygotowane jako punkt startowy. Nie należy jej mylić z przypadkową konfiguracją po poprzednim właścicielu ani z szablonem wirtualnej maszyny EVE-NG, który mógł zostać już wcześniej zmodyfikowany.

Przykładowo niektóre fizyczne SRX mają w stanie fabrycznym przygotowane strefy \`trust\`/\`untrust\`, DHCP i inne elementy ułatwiające pierwsze uruchomienie. Switch EX może mieć odmienny sposób uruchamiania portu management. Nasz vSRX może nie posiadać tych samych fabrycznych połączeń ani mapowania interfejsów.

Dlatego pierwsze uruchomienie zaczyna się od **rozpoznania rzeczywistego stanu**, nie od założenia „wszystko jest puste”.

**R1-vSRX — Operational Mode, bezpieczna inspekcja przed pierwszą zmianą**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration | display set
show system commit
show system alarms
~~~

\`show version\` pomaga zidentyfikować wersję Junos i model. \`show chassis hardware\` pokazuje dostępne informacje o platformie (na vSRX są to informacje wirtualne). \`show interfaces terse\` pozwala sprawdzić faktyczne nazwy portów. Odczyt konfiguracji odpowiada na pytanie, co zostało już zapisane. Historia commit pomoże zauważyć, czy urządzenie było wcześniej zmieniane.

Nie publikuj całego \`show configuration\` w publicznym repozytorium. Mogą się w nim znajdować adresy management, zakodowane hasła lub inne dane wrażliwe.

### Co z kontem root?

W wielu scenariuszach pierwsze uruchomienie wymaga skonfigurowania uwierzytelniania użytkownika \`root\`. Jest to konto o najwyższych uprawnieniach systemowych. W zależności od sprzętu i wersji sposób pierwszego logowania może być różny; nie zakładaj uniwersalnego pustego hasła dla każdego urządzenia Juniper.

Pierwsze ważne rozróżnienie to **root przez konsolę** i **root przez SSH**. Są to różne ścieżki dostępu, dla których mogą obowiązywać odmienne zasady. To, że możesz wejść do urządzenia jako root przez konsolę, nie oznacza, że jego SSH powinno dopuszczać ten sam login.

W rozdziale 4.3 utworzymy osobne konto administratora. Dopiero kiedy przetestujemy je przez nową sesję, rozważymy ograniczenie logowania root przez SSH.

### Czy powrót do factory default i „wymazanie” urządzenia to to samo?

Nie. \`load factory-default\` jest mechanizmem wczytania fabrycznej konfiguracji do edytora; wymaga dalszego przygotowania i zatwierdzenia. \`request system zeroize\` to osobna, znacznie bardziej destrukcyjna operacja związana z czyszczeniem danych i konfiguracji. Takie działania mają różne konsekwencje na urządzeniu i nie powinny być traktowane jako synonimy.

**Nie wykonujemy żadnej z tych operacji na istniejącym labie ani na urządzeniu firmowym.** Do nauki nowego systemu lepiej przygotować świeżą maszynę wirtualną i zrobić jej zewnętrzny snapshot.

### A jeśli urządzenie próbuje samo pobrać konfigurację?

Niektóre modele i wdrożenia obsługują **Zero-Touch Provisioning (ZTP)**. To proces automatycznego pobrania ustawień lub wykonania pierwszych etapów konfiguracji przez odpowiednią infrastrukturę, np. z usług sieciowych. ZTP nie oznacza „braku konfiguracji”; może oznaczać wręcz, że urządzenie rozpoczęło automatyczne przygotowanie.

Nie podłączaj nowego lub resetowanego sprzętu do niezweryfikowanej sieci DHCP/ZTP, jeżeli nie wiesz, jakie opcje dostanie i co pobierze. W naszym labie używamy kontrolowanego, izolowanego środowiska.

**Wniosek z pierwszej lekcji:** zanim wprowadzisz pierwsze \`set\`, musisz znać urządzenie, wersję, aktywną konfigurację, interfejsy i dostęp ratunkowy. Dopiero wtedy wiesz, od którego miejsca zaczynasz.

**Dokumentacja:** [SRX initial configuration](https://www.juniper.net/documentation/us/en/hardware/srx300/topics/topic-map/srx300-configuring-junos.html), [EX4400 initial configuration](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/topic-map/ex4400-configure-junos-os.html).

---

## 4.2. Initial configuration — jak nadać routerowi tożsamość i nie stracić dostępu? [A]

### Hostname nie przesyła pakietów, ale jest ważny dla ludzi

Na naszej topologii wszystkie urządzenia mają konkretne role: R1 jest EDGE-A, R2 górnym routerem rdzenia, R3 dolnym. W środowisku klienta nazwa hosta zwykle zawiera również lokalizację i funkcję. Bez niej łatwo pomylić urządzenia, szczególnie przy pracy przez kilka okien SSH.

W Junos nazwę określa ustawienie \`system host-name\`. Zróbmy najpierw tylko tę zmianę na izolowanym R1.

**R1-vSRX — Operational Mode, wejście do konfiguracji**

~~~junos
configure private
~~~

**R1-vSRX — Configuration Mode [edit private], ustawienie nazwy**

~~~junos
set system host-name R1-vSRX
show | compare
commit check
commit confirmed 5
~~~

Teraz sprawdź z osobnej sesji, czy system wyświetla nową nazwę. Dopiero potem potwierdź zmianę w edytorze przez \`commit\`. Pamiętasz z rozdziału III, że **nie uruchamiamy kolejnego \`commit check\` podczas oczekującego \`commit confirmed\`**, ponieważ może to potwierdzić warunkową zmianę.

**R1-vSRX — Operational Mode, osobna sesja: post-check**

~~~junos
show configuration system host-name
show system uptime
show system commit
~~~

**R1-vSRX — Configuration Mode [edit private], tylko po pozytywnym teście**

~~~junos
commit
~~~

To mała zmiana, lecz ustanawia nawyk: **każdy commit ma cel, pre-check i post-check**, nawet kiedy jego ryzyko jest niewielkie.

### Jaka jest różnica między hostname, domain-name a DNS server?

\`host-name\` to nazwa samego urządzenia. \`domain-name\` to domena, np. \`lab.example\`, która może być używana do budowania pełnych nazw systemowych. \`name-server\` wskazuje serwer DNS, od którego router może uzyskać odpowiedzi na zapytania o nazwy.

Wiele początkujących osób traktuje te trzy pola jako odmiany jednego ustawienia. Ale wyobraź sobie, że ustawiłeś hostname R1 i domenę lab.example, a zapomniałeś skonfigurować serwer DNS albo nie ma do niego routingu. Router będzie miał poprawną nazwę, ale może nie umieć rozwiązać nazwy \`ntp.lab.example\` na adres IP.

**R1-vSRX — Configuration Mode [edit private], przykład tożsamości laboratorium**

~~~junos
set system domain-name lab.example
set system time-zone Europe/Warsaw
show | compare
~~~

Ta konfiguracja nie tworzy serwera DNS. Jeżeli w labie istnieje rzeczywisty, osiągalny serwer nazw, dodajemy go świadomie po ustaleniu IP i tablicy routingu. W podręczniku nie będziemy wymyślać działającego DNS tylko po to, aby wstawić polecenie do listy.

Ustawienie strefy czasowej też nie synchronizuje zegara. Oznacza sposób lokalnej prezentacji czasu; synchronizacją zajmie się NTP w sekcji 4.8.

### Co to właściwie jest management?

Do routera możesz dostać się przez **konsolę**, przez połączenie sieciowe SSH albo przez inny wspierany interfejs zarządzania. W Enterprise często projektuje się oddzielny segment **OOB (out-of-band management)**. Jego celem jest zapewnienie dostępu administracyjnego nawet wtedy, gdy główna sieć transmisji danych jest niedostępna.

W praktyce OOB zwykle wymaga osobnego połączenia, adresowania i urządzeń zapewniających dostęp. Nie wystarczy nazwać portu „management”. Jeżeli zarówno SSH, jak i routing użytkowników zależą od tego samego jedynego uplinku, utrata uplinku odcina oba.

Na fizycznych Juniperach interfejs zarządzania może mieć nazwę \`fxp0\`, \`me0\`, \`vme\` lub inną, zależnie od rodziny sprzętu. **Nie zakładamy, że vSRX w EVE-NG na pewno ma \`fxp0\` oraz że jest to zawsze pierwszy NIC.**

### Management port a management routing instance

To rozróżnienie jest szczególnie ważne. **Interfejs management** to miejsce podłączenia sieci administracyjnej. **Management routing instance** to odrębny kontekst tablic routingu, w którym usługi systemowe mogą szukać dróg do serwerów zarządzania.

W Junos niektóre platformy udostępniają instancję \`mgmt_junos\`; nie jest ona jednak automatycznie aktywna na każdej maszynie tylko dlatego, że port nazywa się \`fxp0\`. Jeżeli ją włączysz lub odziedziczysz z szablonu, musisz kontrolować właściwą tablicę routingu, obsługę DNS/NTP/syslog/AAA i możliwość powrotu do administratora.

**R1-vSRX — Operational Mode, pre-check modelu zarządzania**

~~~junos
show interfaces terse
show configuration system management-instance
show route table inet.0
show route table mgmt_junos.inet.0
~~~

Ostatnia komenda może nie pokazać wpisów albo instancja może nie istnieć w Twoim wariancie. To informacja diagnostyczna, nie dowód awarii.

Nie podajemy na tym etapie gotowego \`set interfaces fxp0 unit 0...\`, ponieważ **nie znamy jeszcze rzeczywistego mapowania management**. W ćwiczeniu będzie osobny warunek: najpierw odczytaj realną nazwę i połączenie w EVE-NG, dopiero potem skonfiguruj adres z izolowanej sieci administracyjnej. Nie zmieniaj adresów R1–R4 na łączach DIAMOND-6R, aby uzyskać management.

### Pierwsze SSH: bezpieczny, ale nie samodzielnie wystarczający krok

SSH to szyfrowany dostęp tekstowy do routera. Aby zadziałał, potrzebujesz nie tylko włączonej usługi, ale również drogi IP, konta, odpowiednich uprawnień i ewentualnego dopuszczenia ruchu **do samego SRX**.

**R1-vSRX — Configuration Mode [edit private], włączenie usługi SSH bez ruszania portów**

~~~junos
set system services ssh
show | compare
commit check
commit confirmed 5
~~~

Na SRX ruch SSH do lokalnego adresu może dodatkowo zależeć od \`host-inbound-traffic\` i strefy, a dla niektórych połączeń OOB od specyfiki management. Samo \`set system services ssh\` nie otwiera uniwersalnie dostępu ze wszystkich portów.

**Nie ograniczaj jeszcze root SSH**, dopóki nie utworzymy i nie przetestujemy nowego konta administracyjnego w następnej sekcji. Jeśli router ma już dostęp do SSH, nie zmieniaj jego konfiguracji bez sprawdzenia obecnego stanu.

**R1-vSRX — Operational Mode, osobna sesja: weryfikacja usługi**

~~~junos
show configuration system services ssh
show interfaces terse
show system users
show system commit
~~~

Nie traktuj tych odczytów jako testu końcowego. Aby potwierdzić SSH, trzeba naprawdę otworzyć **nowe połączenie z osobnego klienta**. Jeśli nie masz tego klienta, zaznacz wynik jako NOT TESTED, nie jako PASS.

**Dokumentacja:** [Remote Access Overview](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-software-remote-access-overview.html), [Management Interface in Nondefault Instance](https://www.juniper.net/documentation/us/en/software/junos/junos-getting-started/topics/topic-map/management-interface-in-non-default-instance.html).

---

