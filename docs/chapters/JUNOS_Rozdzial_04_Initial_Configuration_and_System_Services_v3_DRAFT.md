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

## 4.3. Lokalne konta — kto ma prawo zalogować się do R1? [A]

### Pierwsza osoba, która ma własny login

Wyobraź sobie, że R1 został uruchomiony i przez konsolę można go konfigurować jako \`root\`. Kolejny krok nie powinien polegać na przesłaniu wszystkim administratorom tego samego hasła. Jeśli pięć osób używa jednego loginu, w logach trudno będzie rozpoznać, kto wykonał zmianę.

Lokalne konto w Junos tworzymy w gałęzi \`system login user\`. Każdy użytkownik ma nazwę, opcjonalny opis, sposób uwierzytelnienia i przypisaną **login class** — czyli zestaw uprawnień, który omówimy dokładniej za chwilę.

Zaczniemy od dwóch kont. \`labadmin\` będzie administratorem testowym, a \`nocview\` będzie przeznaczony wyłącznie do odczytu wybranych informacji. Tworzymy je na R1, nie na wszystkich sześciu routerach naraz.

**R1-vSRX — Configuration Mode [edit private], tworzenie kont bez wpisywania sekretów do pliku**

~~~junos
set system login user labadmin full-name "Administrator DIAMOND-6R"
set system login user labadmin class super-user
set system login user labadmin authentication plain-text-password
~~~

Ostatnie polecenie **nie zawiera hasła**. Urządzenie powinno uruchomić interaktywny dialog, w którym podasz i potwierdzisz hasło. Nie wpisuj swojego prawdziwego hasła do materiałów szkoleniowych, terminala rejestrującego sesje ani repozytorium GitHub.

\`super-user\` jest bardzo szeroką klasą uprawnień, więc nadajemy ją wyłącznie kontu używanemu w odizolowanym labie. W firmie zakres przywilejów powinien wynikać z polityki bezpieczeństwa.

**R1-vSRX — Configuration Mode [edit private], konto tylko do odczytu**

~~~junos
set system login user nocview full-name "Podglad stanu urzadzenia"
set system login user nocview class read-only
set system login user nocview authentication plain-text-password
show | compare
commit check
commit confirmed 5
~~~

To ważne miejsce, aby przypomnieć rozróżnienie z rozdziału III: dopiero zatwierdzenie konfiguracji powoduje, że nowy użytkownik staje się częścią działającego systemu. Po \`commit confirmed\` przetestuj oba loginy z **nowej**, niezależnej sesji. Nie wykonuj kolejnego \`commit check\` w czasie timera.

**R1-vSRX — Operational Mode, odczyt konfiguracji kont i historii**

~~~junos
show configuration system login | display set
show system users
show system commit
~~~

Nie polegaj wyłącznie na \`show configuration\`: to dowód istnienia kont w konfiguracji, ale nie dowód, że nowy login może się faktycznie zalogować. Potrzebujesz przynajmniej jednego udanego testu z osobnej sesji konsoli lub SSH.

### Dlaczego login i prawo do poleceń są różnymi sprawami?

Pomyśl o wejściu do budynku i dostępie do konkretnych pomieszczeń. To, że strażnik rozpoznał Twoją kartę, nie znaczy, że możesz wejść do serwerowni. Podobnie w Junos:

- **Authentication** odpowiada na pytanie: czy użytkownik przedstawił właściwy dowód tożsamości?
- **Authorization**: co użytkownikowi wolno robić po zalogowaniu?
- **Accounting**: jakie informacje o jego aktywności i zmianach można później odtworzyć?

Te trzy funkcje składają się na pojęcie **AAA**. W kolejnych sekcjach dowiemy się, jak Junos łączy konta, login classes i serwery centralne.

### SSH keys — dlaczego administrator może nie potrzebować hasła za każdym razem?

W SSH można korzystać z pary kluczy: **prywatnego** i **publicznego**. Klucz prywatny zostaje u administratora i nigdy nie powinien trafić do konfiguracji routera ani repozytorium. Klucz publiczny można dodać do konta na Junos, aby umożliwić uwierzytelnienie za pomocą odpowiadającego mu klucza prywatnego.

Nie oznacza to, że „klucz publiczny jest hasłem”. Klucz prywatny nadal trzeba chronić, najlepiej zabezpieczyć hasłem i przechowywać zgodnie z polityką organizacji.

Na wspieranych wydaniach można używać nowoczesnych typów kluczy, np. Ed25519. Dostępność algorytmu i składnię potwierdź jednak dla **rzeczywistej wersji Junos**. Nie wpisujemy do podręcznika fikcyjnego ciągu \`AAAA...\` udającego działający klucz. Konfiguracja jest kompletna dopiero, gdy masz prawdziwy, zatwierdzony klucz publiczny.

### Kiedy wolno zablokować logowanie root przez SSH?

Dopiero po udanym teście indywidualnego konta, poprawnej roli i niezależnej konsoli. Wtedy można ograniczyć dostęp \`root\` przez sieć, nie usuwając automatycznie możliwości pracy ratunkowej przez konsolę.

**R1-vSRX — Configuration Mode [edit private], dopiero po udanym teście labadmin z nowego SSH**

~~~junos
set system services ssh root-login deny
show | compare
commit check
commit confirmed 5
~~~

Po tej zmianie z nowej sesji sprawdź, czy \`labadmin\` nadal może wejść na router i czy \`nocview\` nie uzyskał szerszych uprawnień. Potwierdź zmianę dopiero po pozytywnych testach. Nie wyłączaj zdalnego root na urządzeniu, do którego nie masz innej pewnej ścieżki administracyjnej.

**R1-vSRX — Operational Mode, read-only kontrola polityki SSH**

~~~junos
show configuration system services ssh | display set
show system users
show system commit
~~~

**Dokumentacja:** [User Access Privileges](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-access-privileges.html), [Remote Access Overview](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-software-remote-access-overview.html).

---

## 4.4. Login classes — dlaczego dwie zalogowane osoby widzą inne możliwości? [A]

Po utworzeniu kont \`labadmin\` i \`nocview\` mamy dwóch użytkowników na tym samym R1. Mogą logować się przez ten sam interfejs, ale nie powinni mieć identycznych praw. Osoba odpowiedzialna za monitoring musi widzieć stan portów i alarmy; nie musi mieć prawa kasować konfiguracji czy restartować routera.

W Junos zakres uprawnień opisuje **login class**. Nie jest ona tożsama z hasłem ani z samą możliwością otwarcia sesji. Konto wskazuje klasę; klasa określa, które grupy poleceń i konfiguracji są dostępne.

Na początek wystarczą nam cztery nazwy klas wbudowanych:

| Klasa | Ogólna rola | O czym pamiętać |
|---|---|---|
| \`super-user\` | Bardzo szerokie uprawnienia | Dla zaufanych administratorów |
| \`read-only\` | Odczyt wielu informacji bez prawa edycji | Nie wszystkie polecenia operacyjne muszą być dostępne |
| \`operator\` | Obsługa operacyjna urządzenia | To **nie** jest synonim read-only |
| \`unauthorized\` | Minimalne uprawnienia | Celowo bardzo ograniczona klasa |

Nie oceniaj klas tylko na podstawie nazwy. Najlepiej przetestować konkretne polecenia na rzeczywistym koncie. \`operator\` może pozwalać na czynności zmieniające stan systemu i nie powinna być stosowana jako „bezpieczny obserwator” bez weryfikacji.

### Tworzymy własną klasę

Załóżmy, że do nocnego dyżuru NOC potrzebujesz tylko podstawowego podglądu, a nie pełnych uprawnień. Możesz utworzyć klasę \`CH04-NOC\` z odpowiednimi prawami. Na początek dajemy jej minimalne uprawnienia do odczytu i sprawdzamy, czy wystarczają do wymaganych poleceń.

**R1-vSRX — Configuration Mode [edit private], własna klasa laboratoryjna**

~~~junos
set system login class CH04-NOC permissions view
set system login class CH04-NOC idle-timeout 15
set system login user nocview class CH04-NOC
show | compare
commit check
commit confirmed 5
~~~

\`idle-timeout 15\` oznacza limit bezczynności sesji przewidziany przez tę konfigurację. Wymagany zakres i realne działanie zależą od wydania i typu sesji. \`permissions view\` jest przykładem bardzo ograniczonego dostępu do informacji, ale nie zakładaj, że pozwala czytać wszystkie gałęzie konfiguracji.

Jeżeli NOC ma mieć prawo oglądania konfiguracji, trzeba rozważyć również odpowiednie uprawnienie do widoku konfiguracji, np. \`view-configuration\`, i przetestować tę decyzję. Więcej praw należy nadawać **dopiero wtedy, gdy istnieje konkretna potrzeba**.

**R1-vSRX — Operational Mode, zalogowany jako nocview w nowej sesji**

~~~junos
show cli authorization
show interfaces terse
show system uptime
~~~

Sprawdź, które polecenia są dostępne, a które odrzucane. To pozwoli dowiedzieć się znacznie więcej niż samo przeczytanie opisu klasy. Nie próbuj zgadywać wyniku testu, jeśli nie uruchamiałeś go w konkretnej wersji Junos.

### Co z bardzo szczegółowymi rolami?

W bardziej zaawansowanych środowiskach klasy mogą używać dodatkowych ograniczeń poleceń i hierarchii, np. \`allow-commands\`, \`deny-commands\` lub innych ustawień dostępu. Ważne jednak, że te reguły **współpracują z bazowymi permissions** i nie powinny być traktowane jako sposób bezwarunkowego obejścia każdej kontroli.

Jeżeli NOC ma móc wykonać wyłącznie jeden konkretny \`show\`, najpierw projektujesz rolę i testujesz ją na wydzielonym koncie. Nie kopiujesz przypadkowego regexu \`allow-commands\` z bloga, bo zbyt szeroki wzorzec może dopuścić więcej poleceń niż planowano.

**Rozumienie na poziomie JNCIP** polega tu nie na zapamiętaniu wszystkich nazw permissions, lecz na umiejętności odróżnienia: udanego uwierzytelnienia, wyboru login class oraz konkretnej decyzji o dopuszczeniu polecenia.

**Dokumentacja:** [Junos Access Privileges](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-access-privileges.html).

---

## 4.5. AAA — co się dzieje, gdy serwer uwierzytelniania nie odpowiada? [A]

### Po co firmie centralny serwer logowania?

Na jednym routerze dwie lokalne nazwy użytkowników są proste w utrzymaniu. Ale firma może mieć setki urządzeń w kilkudziesięciu lokalizacjach. Zakładanie i wyłączanie kont na każdym z nich osobno szybko staje się problemem. Dlatego wiele organizacji korzysta z **centralnego AAA**, najczęściej przez **TACACS+** albo **RADIUS**.

Kiedy administrator próbuje wejść na R1, router może wysłać zapytanie do serwera AAA. Serwer sprawdza przedstawione dane i zwraca odpowiedź. W odpowiednio skonfigurowanym rozwiązaniu może też uczestniczyć w przypisaniu uprawnień oraz rejestrowaniu działań.

Nie potrzebujesz na razie znać formatu protokołów TACACS+ czy RADIUS. Kluczowe są trzy pytania: **kto potwierdza tożsamość**, **co się stanie, jeśli ten system nie działa** i **co się stanie, jeśli świadomie odmówi dostępu**.

### Timeout nie jest tym samym co reject

Wyobraź sobie dwa scenariusze.

W pierwszym R1 wysyła zapytanie do serwera TACACS+, ale żaden pakiet z odpowiedzią nie wraca. Może nie działać droga management, serwer lub usługa. To **timeout** albo niedostępność metody.

W drugim R1 wysyła zapytanie i otrzymuje poprawną odpowiedź „ten login nie powinien być zaakceptowany”. To **reject**. Serwer działa, tylko świadomie nie autoryzuje danego logowania.

Na pierwszy rzut oka użytkownik w obu sytuacjach widzi nieudane logowanie. Ale dla zasad fallback to różne warunki.

W klasycznym Junos zachowanie **\`authentication-order\`** zależy od metod znajdujących się w liście. Dokumentacja Juniper wskazuje m.in., że kiedy serwery zdalne **nie odpowiadają**, lokalne uwierzytelnianie hasłem może być stosowane jako ostatnia szansa, również gdy nie zostało jawnie wymienione w kolejności. Natomiast po **jawnej odmowie** z serwera system może przejść do następnej metody tylko wtedy, gdy ta metoda jest w skonfigurowanej sekwencji. Nie wyciągaj z tego wniosku, że local fallback jest identyczny dla wszystkich konfiguracji.

Przykład:

| Kolejność | Serwer TACACS+ milczy | Serwer odpowiada reject |
|---|---|---|
| \`tacplus\` | W typowym Junos możliwa lokalna próba awaryjna | Lokalna próba nie wynika automatycznie z samego reject |
| \`[ tacplus password ]\` | Po niedostępnej metodzie możliwy local | Po odmowie można przejść do jawnie wymienionego \`password\` |
| \`password\` | Zdalny TACACS+ nie bierze udziału | Zdalny TACACS+ nie bierze udziału |

To uproszczenie mechaniki dla **logowania administratorów** w klasycznym Junos. Dokładny wynik zależy też od istnienia lokalnego konta, jego hasła, przypisania class, reguł zdalnego serwera i wydania. Na sesji SSH z kluczem publicznym może istnieć osobna wcześniejsza metoda uwierzytelnienia.

### Bezpieczna kolejność wdrożenia AAA

Zanim ustawisz TACACS+, upewnij się, że masz poprawne lokalne konto administracyjne, działa niezależna konsola i serwer jest osiągalny z właściwej **tablicy routingu** oraz source IP. Wtedy dopiero rozważamy dołączenie zdalnego AAA.

Nie tworzymy fikcyjnego działającego serwera. Poniżej jest przykład **samej polityki kolejności**, nie kompletny projekt TACACS+.

**R1-vSRX — Configuration Mode [edit private], przykład centralnego AAA z jawnym local fallback**

~~~junos
set system authentication-order [ tacplus password ]
show | compare
~~~

To nie jest jeszcze gotowa konfiguracja. Brakuje wpisu serwera TACACS+, prawidłowego source-address, jego osiągalności i **shared secret**. Nie dodawaj \`commit\`, dopóki tych warunków nie spełniono i nie wykonano osobnych testów dostępu.

### Gdzie skonfigurować serwer TACACS+?

W klasycznym Junos serwery administracyjnego TACACS+ konfiguruje się pod \`system tacplus-server\`. Potrzebne jest IP serwera oraz sekret współdzielony z systemem AAA. Serwer musi wiedzieć, z jakiego adresu źródłowego router będzie nadawał zapytania. Przy odseparowanym OOB/VRF niewłaściwy source IP albo routing mogą uniemożliwić logowanie mimo poprawnego sekretu.

Załóżmy wyłącznie jako **wariant laboratoryjny**, że \`LAB-SERVICES\` posiada prawidłowo uruchomiony serwer TACACS+ pod 203.0.113.30, a R1 ma faktycznie skonfigurowany i osiągalny adres zarządzania **203.0.113.10** w odpowiednim kontekście. Dopiero po weryfikacji tych faktów można rozważyć:

**R1-vSRX — Configuration Mode [edit private], szablon TACACS+ bez sekretu — NIEKOMPLETNY**

~~~junos
set system tacplus-server 203.0.113.30 source-address 203.0.113.10
set system authentication-order [ tacplus password ]
show | compare
~~~

Celowo nie ma tu polecenia z hasłem TACACS+, bo nie wolno umieszczać prawdziwego sekretu w repozytorium. Sekret wprowadza się odpowiednim wspieranym mechanizmem lub z kontrolowanego vault, bez jego ujawniania w historii sesji. **Powyższy blok nie powinien być samodzielnie zatwierdzany jako pełny onboarding AAA.**

### RADIUS — podobny cel, inny sposób działania

RADIUS jest również protokołem używanym do AAA. Spotkasz go w systemach Wi-Fi, 802.1X i dostępie do urządzeń. Nie oznacza to jednak, że konfiguracja RADIUS i TACACS+ jest po prostu zmianą jednego słowa. W zależności od zastosowania różnią się możliwości zarządzania uprawnieniami oraz obsługa atrybutów.

W wariancie urządzeń Junos można użyć \`system radius-server\` i wskazać kolejność \`[ radius password ]\`, pod warunkiem przygotowania serwera, source IP i zgodnego sekretu. **Nie włączaj jednocześnie kilku metod zdalnych bez zaprojektowania oczekiwanej kolejności**.

### Jak udowodnić, że AAA naprawdę działa?

Trzeba wykonać co najmniej trzy różne testy: udane logowanie z centralnym serwerem, kontrolowaną odmowę oraz niedostępność serwera z zachowaniem awaryjnego dostępu lokalnego. Jeśli nie masz prawdziwego serwera TACACS+/RADIUS, nie jesteś w stanie uczciwie oznaczyć tego eksperymentu jako PASS.

**R1-vSRX — Operational Mode, pre-check do testu AAA**

~~~junos
show configuration system authentication-order
show configuration system tacplus-server
show route 203.0.113.30
show log messages | match "tac|radius|sshd|login"
show system users
~~~

Te odczyty nie dowodzą jeszcze logowania. Potrzebne są logi **z obu stron**: Junos i serwera AAA, plus rzeczywista nowa sesja. Ważne jest też, jaką login class otrzymał użytkownik. Udane hasło nie oznacza automatycznie \`super-user\`.

**Dokumentacja:** [Authentication Order for RADIUS/TACACS+/Local Password](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-authentication-order.html), [TACACS+ Authentication](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/user-access-tacacs-authentication.html).

---

## 4.6. Interfejsy Junos — co tak naprawdę konfigurujesz, gdy wpisujesz adres IP? [A]

### Port Ethernet, jednostka logiczna i protokół to trzy różne rzeczy

W rozdziale II poznaliśmy różnicę między siecią bezpośrednio połączoną a trasą statyczną. W rozdziale III widzieliśmy już gałąź \`interfaces\`. Teraz możemy połączyć te wiadomości.

Na R1 port \`ge-0/0/0\` prowadzi do R2. Z nazwy wiemy, że to fizyczny port Ethernet pewnej klasy. Ale w Junos adres IPv4 nie musi znajdować się bezpośrednio na poziomie fizycznego portu. Zwykle konfigurujemy **jednostkę logiczną \`unit 0\`**, a dopiero w niej rodzinę protokołów **\`family inet\`**.

W naszym laboratorium adres R1 na tym łączu to 10.12.0.1/30. Pełna ścieżka wygląda więc tak:

~~~text
interfaces
└── ge-0/0/0           ← interfejs fizyczny
    └── unit 0         ← jednostka logiczna (.0)
        └── family inet ← rodzina IPv4
            └── address 10.12.0.1/30
~~~

Nie musisz dziś znać wszystkich rodzin protokołów. \`inet\` to IPv4, \`inet6\` jest związane z IPv6, a \`ethernet-switching\` dotyczy trybu przełączania na wspieranych platformach. Istnieją też interfejsy agregowane \`ae\`, loopback \`lo0\` i interfejsy VLAN L3, np. \`irb\`. Ich dostępność zależy od modelu i odpowiedniego trybu pracy.

### Konkretny przykład na R1

**R1-vSRX — Configuration Mode [edit private], tylko jeśli adres nie jest już ustawiony w bazowym labie**

~~~junos
set interfaces ge-0/0/0 description "DIAMOND-6R to R2"
set interfaces ge-0/0/0 unit 0 family inet address 10.12.0.1/30
show interfaces ge-0/0/0
show | compare
~~~

Nie zatwierdzaj ślepo tej konfiguracji, jeśli interfejs już ma adres. W referencyjnym DIAMOND-6R port powinien być znany; celem ćwiczenia jest **zrozumienie hierarchii**, a nie duplikowanie parametrów czy zmiana istniejącej sieci.

Po aktywowaniu interfejsu i połączeniu z R2 router może mieć bezpośrednią trasę \`Direct 10.12.0.0/30\` i lokalną \`Local 10.12.0.1/32\`, jak wyjaśniliśmy w rozdziale II. To dobry przykład konsekwencji prostej konfiguracji: wpisanie adresu zmienia nie tylko opis portu, ale również informacje routingu.

**R1-vSRX — Operational Mode, weryfikacja powiązania portu i tras**

~~~junos
show interfaces ge-0/0/0 terse
show interfaces ge-0/0/0 extensive
show route 10.12.0.0/30 exact
show route 10.12.0.1/32 exact
~~~

### Czy interfejs fizyczny „up” oznacza, że IP działa?

Nie. \`Physical link is Up\` oznacza, że port zgłasza działającą warstwę fizyczną lub emulowany link. Jednostka logiczna może jednak być nieaktywna, może mieć inny adres, złą rodzinę protokołów albo wpływ może mieć security zone. Na vSRX ruch do własnego adresu dodatkowo zależy od reguł host-inbound, a ruch tranzytowy od security policies.

To znaczy, że test \`show interfaces terse\` jest początkiem diagnostyki, a nie jej końcem.

### Czym różnią się \`fxp0\`, \`me0\`, \`vme\`, \`irb\`, \`ae\` i \`lo0\`?

\`fxp0\` lub \`me0\` spotkasz w kontekście zarządzania, zależnie od platformy. \`vme\` może pełnić funkcję wirtualnego interfejsu zarządzania w wybranych rozwiązaniach przełącznikowych. \`irb\` pozwala powiązać routing warstwy trzeciej z odpowiednimi domenami przełączania, gdy urządzenie wspiera ten mechanizm. \`ae\` to interfejs agregujący kilka fizycznych połączeń. \`lo0\` to interfejs logiczny, często wykorzystywany do stabilnych adresów routera i mechanizmów ochrony control plane.

Nie ma sensu konfigurować wszystkich tych typów na vSRX tylko po to, by zapamiętać ich nazwy. Przełącznik EX ma inne możliwości niż wirtualny firewall, a niektóre funkcje zależą od hardware i trybu pracy. Do ćwiczeń Layer 2 w późniejszych rozdziałach mogą być potrzebne EX/vEX lub inny wspierany obraz.

### A co z adresacją management?

W 4.2 ustaliliśmy, że rozróżniamy **sieć danych**, **port zarządzania** i **tablicę routingu, z której korzystają usługi systemowe**. Na R1 adres 192.0.2.1/24 należy do LAN-A, a 10.12.0.1/30 do łącza rdzeniowego. Nie chcemy przeznaczyć żadnego z tych portów na fikcyjny management.

Jeśli masz prawdziwy izolowany interfejs OOB o zweryfikowanej nazwie, można go skonfigurować według jego instrukcji. Dopiero wtedy sprawdzisz, czy serwery AAA, NTP i syslog są osiągalne w odpowiedniej tablicy. Nie przenoś \`default route\` do \`mgmt_junos.inet.0\` bez zrozumienia skutku.

**R1-vSRX — Operational Mode, dwa widoki tras, jeśli wspierana jest mgmt_junos**

~~~junos
show route table inet.0
show route table mgmt_junos.inet.0
show interfaces terse
~~~

**Wniosek:** dobrze skonfigurowany adres IP jest dopiero elementem procesu. Port, logiczna jednostka, rodzina, security zone i właściwa tablica routingu muszą pasować do rzeczywistego celu.

---

