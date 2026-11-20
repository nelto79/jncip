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

## 4.7. Configuration groups — jak uniknąć ręcznego przepisywania tych samych ustawień? [A]

W małym laboratorium masz sześć routerów, a w prawdziwej firmie może być ich kilkaset. Wyobraź sobie, że na każdym urządzeniu obowiązuje ta sama firmowa zasada: standardowy komunikat logowania, pewne ustawienia systemowe, konwencja opisów i wybrane parametry monitoringu.

Możesz ręcznie wpisywać identyczne linie na każdym routerze. To zadziała, lecz przy kolejnej zmianie standardu musisz poprawić wszystkie miejsca. W Junos istnieje mechanizm **configuration groups**, który pozwala wielokrotnie wykorzystywać fragmenty konfiguracji i dziedziczyć je w odpowiednich miejscach hierarchii.

### Najpierw wyjaśnijmy samo dziedziczenie

Gdy konfiguracja lokalna używa grupy, Junos może pobrać z niej wartości, które mają obowiązywać w danej części drzewa. Sama grupa to jednak **definicja szablonu**, a nie jego natychmiastowe zastosowanie. Potrzebne jest wskazanie \`apply-groups\`.

Przyjmijmy prosty przykład. Na R1 chcemy zastosować standardowy komunikat logowania.

**R1-vSRX — Configuration Mode [edit private], definicja grupy**

~~~junos
set groups CH04-BASE system login message "CH04: Lab device"
show groups CH04-BASE
~~~

Zwróć uwagę: właśnie utworzyłeś grupę w candidate. Ale dopóki nie została zastosowana, nie musi wpływać na wynikową konfigurację systemu.

**R1-vSRX — Configuration Mode [edit private], zastosowanie grupy**

~~~junos
set apply-groups CH04-BASE
show | compare
show system login | display inheritance
~~~

\`apply-groups\` na poziomie korzenia wskazuje, że system ma uwzględniać tę grupę w odpowiednich miejscach drzewa konfiguracji. \`display inheritance\` pokazuje wartości uwzględniające dziedziczenie. To ważne: bez tego polecenia inżynier może nie znaleźć ustawienia w miejscu, w którym go oczekiwał.

### Co, gdy lokalnie ustawisz inną wartość?

Załóżmy, że R1 potrzebuje innego komunikatu niż pozostałe urządzenia. Nie musisz od razu usuwać całej grupy. W typowym mechanizmie dziedziczenia **jawnie ustawiona wartość lokalna może mieć pierwszeństwo** nad wartością dziedziczoną.

**R1-vSRX — Configuration Mode [edit private], lokalny wyjątek od grupy**

~~~junos
set system login message "CH04: R1 local exception"
show system login | display inheritance
show | compare
~~~

Sprawdź, która wartość pojawiła się w konfiguracji wynikowej. Dzięki temu nie uczysz się tylko komend, ale mechanizmu: najpierw przychodzi szablon, potem może go nadpisać ustawienie zdefiniowane bliżej konkretnego urządzenia albo odpowiedniego obiektu.

Przy większej liczbie grup **kolejność \`apply-groups\` ma znaczenie** dla wyboru wartości. Nie próbuj zgadywać, że ostatnia wymieniona zawsze wygrywa w każdej sytuacji — sprawdź dokumentację i faktyczne \`display inheritance\`.

### A jeśli chcę wyłączyć dziedziczenie tylko na jednym porcie?

Junos posiada również **\`apply-groups-except\`**, które pozwala wyłączyć dziedziczenie wskazanej grupy w odpowiednim poddrzewie. To przydatne, kiedy setki interfejsów korzystają ze standardu, a pojedynczy wymaga wyjątku.

Nie będziemy konfigurować fikcyjnych 120 portów na vSRX. Wystarczy zrozumieć, że grupa i jej wyłączenie są częścią **hierarchii**, a nie globalnymi tekstowymi „makrami”, które zawsze działają identycznie na każdym miejscu.

W rozbudowanych konfiguracjach mogą występować także symbole dopasowujące, np. wildcards, pozwalające stosować wspólny fragment do większej liczby obiektów. To temat, przy którym warto szczególnie uważać na zakres dopasowania: zbyt szeroki wzorzec może objąć także interfejsy zarządzające lub porty z inną rolą.

### Czy \`apply-groups\` wysyła konfigurację automatycznie na R2 i R3?

Nie. Grupa w konfiguracji R1 jest lokalnym mechanizmem dziedziczenia na R1. Nie jest centralnym systemem zarządzania sześcioma routerami. Jeżeli chcesz zastosować standard na R2 i R3, musisz go świadomie dostarczyć i zatwierdzić na tych urządzeniach, ewentualnie korzystając z zewnętrznej automatyzacji.

Właśnie dlatego configuration groups pomagają w **powtarzaniu konfiguracji**, ale nie zastępują repozytorium, backupu, orkiestracji ani kontroli zmian.

### Jak bezpiecznie wycofać laboratoryjny eksperyment?

Jeśli grupa i wyjątek nie zostały jeszcze zatwierdzone, a w prywatnej candidate masz wyłącznie własne testy, możesz usunąć ich elementy i porównać stan z active.

**R1-vSRX — Configuration Mode [edit private], wycofanie własnego eksperymentu**

~~~junos
delete system login message
delete apply-groups CH04-BASE
delete groups CH04-BASE
show | compare
~~~

Nie zakładaj pustego diffu, jeśli wcześniej istniał już jakiś \`system login message\` — w takim przypadku trzeba przywrócić poprzednią wartość, a nie usuwać ją na ślepo. W tym rozdziale wielokrotnie wracamy do zasady: **baseline ma większą wartość niż mechaniczny cleanup**.

**Dokumentacja:** [Configuration Groups Usage](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configuration-groups-usage.html), [apply-groups](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/apply-groups.html).

---

## 4.8. NTP — dlaczego poprawna godzina jest częścią działania sieci? [A]

### Dwie awarie i trzy różne zegary

R1 pokazuje awarię portu o 10:04, R2 o 10:07, a serwer aplikacji zapisuje błąd o 10:03. Czy najpierw padło połączenie, a dopiero potem aplikacja? Jeśli zegary różnią się o kilka minut, nie możesz tego wiarygodnie ustalić.

W środowisku Enterprise logi routerów, switchy, firewalli, AAA i serwerów trafiają często do jednego systemu. Żeby ustalić kolejność zdarzeń, urządzenia potrzebują możliwie wspólnego czasu. Do tego służy **NTP — Network Time Protocol**.

NTP pozwala urządzeniu porównywać swój czas z wybranymi serwerami i stopniowo korygować zegar. Nie jest to tylko „pobranie aktualnej godziny raz na zawsze”. System utrzymuje informację o źródłach czasu i ich jakości; może też uznać serwer za niewiarygodny lub nieosiągalny.

### Stratum, association i synchronized — co oznaczają?

**NTP server** to urządzenie lub usługa udostępniająca czas innym. **Stratum** opisuje pozycję danego źródła w hierarchii NTP, ale niższa liczba nie zawsze oznacza, że ten serwer jest najlepszym wyborem dla konkretnego klienta. Ważne są także stabilność, osiągalność i inne parametry.

**Association** to relacja klienta NTP z danym źródłem. Sama widoczność takiej relacji nie znaczy jeszcze, że system jest zsynchronizowany. Polecenia \`show ntp associations\` i \`show ntp status\` pozwalają rozróżnić serwer wpisany w konfigurację od serwera rzeczywiście używanego i stan synchronizacji urządzenia.

### Konfigurujemy NTP dopiero po przygotowaniu drogi do serwera

W wariancie dodatkowej sieci management załóżmy, że Linux VM \`LAB-SERVICES\` ma IP **203.0.113.30** i rzeczywiście uruchomiony daemon NTP. R1 musi mieć z niego osiągalną odpowiedź na odpowiednie pakiety, zwykle UDP/123.

**R1-vSRX — Configuration Mode [edit private], serwer NTP w opcjonalnym izolowanym OOB**

~~~junos
set system ntp server 203.0.113.30
set system time-zone Europe/Warsaw
show | compare
~~~

To **szablon zależny od wariantu labu**. Jeśli nie utworzyłeś sieci management i nie uruchomiłeś usługi NTP na Linux, nie konfiguruj nieistniejącego serwera i nie oznaczaj testu jako PASS.

\`time-zone\` odpowiada za prezentację czasu lokalnego. Nie synchronizuje zegara. NTP może prawidłowo utrzymywać czas UTC, a administrator zobaczy go w lokalnej strefie; to dwie różne sprawy.

### Dlaczego \`source-address\` ma znaczenie?

Router może mieć kilka adresów IP. Jeśli wysyła zapytanie do NTP z niewłaściwego source, serwer może nie mieć drogi powrotnej, filtrować zapytania albo traktować router jako nieznanego klienta.

W rozbudowanym środowisku ważna jest także **routing instance** — z której tablicy router szuka drogi do serwera. W NTP istnieją specyficzne zależności od \`mgmt_junos\`, platformy i wydania, więc nie można po prostu skopiować \`source-address\` z cudzej konfiguracji i zakładać, że będzie poprawny.

**R1-vSRX — Operational Mode, pre-check trasy do serwera czasu**

~~~junos
show interfaces terse
show route 203.0.113.30
show configuration system ntp | display set
~~~

Jeżeli korzystasz z wydzielonego management routing instance, sprawdź także odpowiednią tabelę, a nie tylko \`inet.0\`.

### Jak udowodnić, że NTP działa?

**R1-vSRX — Operational Mode, po rzeczywistym uruchomieniu NTP server**

~~~junos
show ntp associations
show ntp status
show system uptime
~~~

Po konfiguracji i udanym commit system może potrzebować pewnego czasu, aby wybrać i zsynchronizować źródło. Nie interpretuj braku natychmiastowego \`synchronized\` jako jednoznacznej awarii.

Przy diagnozie przejdź kolejno przez następujący związek przyczynowy: **adres serwera → droga IP → source-address/VRF → dostęp UDP/123 → odpowiedź serwera → rzeczywista synchronizacja**.

Ping do 203.0.113.30 potwierdza tylko pewien rodzaj osiągalności ICMP. Nie testuje NTP. Możesz mieć poprawny ping, ale zablokowany UDP/123 lub brak działającej usługi.

**Dokumentacja:** [NTP Configuration](https://www.juniper.net/documentation/us/en/software/junos/time-mgmt/topics/example/ntp-configuration.html), [NTP source-address and management instance](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/ntp-edit-system.html).

---

## 4.9. SNMP — jak system monitoringu dowiaduje się, co dzieje się na routerze? [A]

### Nie każdy administrator stale patrzy na konsolę

Wyobraź sobie, że R1 działa w biurze oddalonym o kilkaset kilometrów. Zespół sieciowy nie będzie co minutę logował się przez SSH i wykonywał \`show interfaces\`. Potrzebuje systemu, który regularnie sprawdza stan urządzeń, zapisuje pomiary i powiadamia o problemach.

Jednym ze sposobów zbierania takich informacji jest **SNMP — Simple Network Management Protocol**. Router udostępnia pewne dane przez SNMP agent, a system monitoringu — często nazywany **NMS (Network Management System)** — może je odczytywać.

Wśród typowych danych znajdują się informacje o czasie działania, interfejsach i licznikach. Ale odczyt informacji nie jest tym samym co wykrywanie każdego rodzaju awarii. Nie każda niedostępność aplikacji jest widoczna jako awaria interfejsu, a poprawne odpowiedzi SNMP nie dowodzą, że użytkownicy mają działający Internet.

### Polling, trap i inform — trzy sytuacje

Najczęściej monitoring działa jako **polling**: NMS sam pyta router o wybrane informacje, zwykle przez UDP/161. Wyobraź sobie, że co minutę sprawdza, czy port R1 do R2 jest w stanie \`up\`.

**Trap** działa w odwrotnym kierunku: router wysyła do systemu monitoringu powiadomienie o zdarzeniu, np. zmianie stanu interfejsu. W typowej konfiguracji SNMP służy do tego UDP/162. **Inform** jest podobnym rodzajem powiadomienia, ale przewiduje potwierdzenie odbioru przez system docelowy.

Te mechanizmy nie zastępują się nawzajem. Polling może wykryć aktualny stan, a zdarzeniowa notyfikacja pomaga zauważyć zmianę szybciej. Każdy wymaga poprawnej konfiguracji i ścieżki sieciowej.

### Dlaczego w nowej sieci wybieramy SNMPv3?

W starszych środowiskach często spotkasz **SNMPv2c**, w którym dostęp opiera się na community string. Ta wartość nie zapewnia nowoczesnego szyfrowania komunikacji i nie powinna być domyślnym wyborem dla nowego projektu.

**SNMPv3** pozwala używać uwierzytelniania i ochrony poufności danych. Poziom **authPriv** oznacza, że komunikacja używa zarówno uwierzytelniania, jak i szyfrowania. W Junos występują tu dwa istotne mechanizmy:

**USM (User-based Security Model)** odpowiada za bezpieczeństwo komunikatów związane z użytkownikiem SNMP, np. uwierzytelnianie i szyfrowanie. **VACM (View-based Access Control Model)** określa, jaki zakres danych dany użytkownik lub grupa może odczytywać.

To podobne rozróżnienie do konta administracyjnego i jego uprawnień. Nie wystarczy znać hasło; trzeba jeszcze wiedzieć, **które informacje wolno odczytać**.

### Czym jest engine ID?

SNMPv3 korzysta z identyfikatora engine, który wiąże się z bezpieczeństwem i użytkownikami USM. W pewnych konfiguracjach zmiana engine ID po skonfigurowaniu kluczy powoduje problemy z uwierzytelnianiem i może wymagać ponownego skonfigurowania użytkowników/kluczy.

Dlatego w produkcyjnym wdrożeniu najpierw ustala się właściwy, stabilny identyfikator, a dopiero później finalną konfigurację użytkowników USM. Nie zmieniaj go arbitralnie podczas troubleshootingu.

### Projektujemy ograniczony widok do odczytu

Na początku pokażmy część SNMP, która nie wymaga żadnego sekretu: zdefiniowanie widoku danych i powiązanie go z grupą uprawnień. W tym przykładzie nazwa \`CH04-RO\` oznacza *read-only*.

**R1-vSRX — Configuration Mode [edit private], fragment modelu dostępu SNMPv3**

~~~junos
set snmp view CH04-RO oid .1 include
set snmp v3 vacm security-to-group security-model usm security-name CH04-NMS group CH04-MON
set snmp v3 vacm access group CH04-MON default-context-prefix security-model usm security-level privacy read-view CH04-RO
show | compare
~~~

W tej chwili **nie masz jeszcze działającego SNMPv3 authPriv**. Powstała tylko część dotycząca widoku i przypisania grupy. Brakuje prawdziwego użytkownika USM i jego bezpiecznie skonfigurowanych parametrów uwierzytelniania oraz szyfrowania.

Wybrany OID \`.1\` pozwala w tej demonstracji zbudować szeroki widok danych SNMP. W produkcji najlepiej ograniczyć widok do informacji niezbędnych dla NMS — nie zakładaj, że „read-only” oznacza brak ryzyka ujawnienia informacji o infrastrukturze.

### Dlaczego nie wklejamy sekretów do dokumentacji?

Do skonfigurowania authPriv potrzebne są m.in. hasła lub klucze uwierzytelniania i prywatności. W dokumentacji producenta znajdziesz składnię z \`authentication-sha\` i \`privacy-aes128\`, ale nie wolno umieszczać prawdziwych danych w GitHub ani w zapisywanej sesji terminala.

Dlatego zamiast udawać kompletną konfigurację wpisem \`<SECRET>\`, traktujemy poniższy opis jako **niedokończony etap wdrożenia**. Administrator musi:

1. Wybrać obsługiwany algorytm dla konkretnego Junos i NMS.
2. Ustalić engine ID.
3. Utworzyć użytkownika USM oraz jego parametry bezpieczeństwa z bezpiecznego źródła.
4. Zweryfikować VACM i ewentualną listę klientów/filtry sieciowe.
5. Uruchomić rzeczywisty test SNMPv3 GET z serwera monitoringu.

W nowych wydaniach dostępne mogą być mocniejsze algorytmy niż historyczne SHA-1/AES128, ale ich wybór powinien wynikać z **wspólnego wsparcia urządzenia i klienta SNMP**. Nie zakładaj automatycznie, że składnia z dokumentacji innego modelu działa na naszym obrazie vSRX.

### Skąd wiadomo, że SNMPv3 jest skonfigurowany poprawnie?

Samo \`show configuration snmp\` oznacza tylko, że router przechowuje ustawienia. Potrzebny jest rzeczywisty **SNMPv3 authPriv GET**, np. odczyt \`sysUpTime.0\`, wykonany z przygotowanego serwera \`LAB-SERVICES\`. Odpowiedź ma też być dozwolona przez właściwe filtry oraz ścieżkę management.

**R1-vSRX — Operational Mode, podstawowa diagnostyka SNMP**

~~~junos
show configuration snmp | display set
show snmp statistics
show snmp v3
show log messages | match "snmp|SNMP"
~~~

\`show snmp statistics\` pomoże zauważyć błędy obsługi, ale do rozstrzygnięcia problemu najlepiej zbierać wyniki po obu stronach. Jeśli router odpowiada na ping, lecz klient SNMP dostaje timeout, zbadaj także UDP/161, source IP, engine ID, użytkownika, algorytmy i VACM.

Pamiętaj, że **trap/inform UDP/162** to inny test niż **polling UDP/161**. Udany GET nie dowodzi jeszcze dostarczania powiadomień.

**Dokumentacja:** [Configure SNMPv3](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/configure-snmpv3.html), [Configure SNMP in Junos OS](https://www.juniper.net/documentation/us/en/software/junos/network-mgmt/topics/topic-map/configure-snmp-in-junos-os.html).

---

## 4.10. Syslog — jak router opowiada historię swojej awarii? [A]

### Bez logów wiesz tylko, co dzieje się teraz

Widzisz, że port R1 do R2 ma stan \`up\`. Ale użytkownicy zgłosili przerwę 15 minut temu. Czy port był wtedy \`down\`? Czy ktoś zmienił konfigurację? Czy urządzenie zostało zrestartowane? Bieżący stan nie daje pełnej odpowiedzi.

**Syslog** to mechanizm zapisywania zdarzeń. Router może przechowywać logi lokalnie, a także wysyłać je do centralnego serwera, aby ich nie stracić przy poważniejszej awarii.

Na początek musimy odróżnić **treść zdarzenia**, **czas** i **miejsce przechowywania**. Jeśli zegar routera jest błędny, kolejność zdarzeń będzie trudna do odtworzenia, nawet gdy sam log zawiera poprawną informację. Dlatego NTP pojawia się w tym rozdziale przed syslog.

### Severity i facility — dwa sposoby opisywania jednej wiadomości

W syslog wiadomość może należeć do określonej kategorii, zwanej **facility**, np. związanej z uwierzytelnianiem lub systemem. Ma też poziom ważności, czyli **severity**.

Nie musisz zapamiętywać wszystkich numerów severity na początku. Wystarczy rozumieć, że \`notice\` i \`info\` nie są tym samym co nagły alarm krytyczny. Konfiguracja powinna zapisywać potrzebne zdarzenia bez zasypywania systemu i terminali.

### Lokalny log

**R1-vSRX — Configuration Mode [edit private], plik lokalny z systemowymi zdarzeniami**

~~~junos
set system syslog file messages any notice
set system syslog file messages authorization info
show | compare
~~~

Pierwsza linia określa, jakie zdarzenia trafią do lokalnego pliku \`messages\` w odpowiednim zakresie ważności. Druga pozwala uwzględnić informacje o autoryzacji. W rzeczywistej konfiguracji upewnij się, że nie dublujesz niepotrzebnie istniejących reguł ani nie ustawiasz zbyt dużego wolumenu logów.

**R1-vSRX — Operational Mode, odczyt lokalnych zdarzeń**

~~~junos
show log messages | last 30
show log messages | match "mgd|sshd|UI_COMMIT"
~~~

Jeśli po commit pojawi się nowa wiadomość, to dobry początek analizy. Nie znaczy to jeszcze, że centralny serwer logowania też ją odebrał.

### Zdalny syslog: drugi koniec musi działać

Załóżmy, że \`LAB-SERVICES\` rzeczywiście odbiera syslog pod adresem **203.0.113.30** w izolowanej sieci management. Wtedy R1 może wysyłać do niego odpowiednie wiadomości.

**R1-vSRX — Configuration Mode [edit private], zdalny syslog w opcjonalnym OOB**

~~~junos
set system syslog host 203.0.113.30 any notice
set system syslog host 203.0.113.30 authorization info
show | compare
~~~

To nie uruchamia syslog receiver. Jeśli nie masz serwera, zdalna strona testu pozostaje **NOT TESTED**. Dla produkcyjnego rozwiązania trzeba także rozważyć bezpieczny transport, segmentację management, odporność odbiornika i retencję logów.

### Dlaczego adres źródłowy routera ma znaczenie?

Serwer logów zwykle rozpoznaje urządzenie między innymi po IP, hostname i metadanych zdarzenia. Jeśli R1 wysyła wiadomości z innego źródła niż oczekiwane, alert może zostać źle przypisany albo filtr go odrzuci. Gdy urządzenie ma \`mgmt_junos\`, trzeba jeszcze sprawdzić właściwą trasę do odbiornika.

Dlatego docelowy projekt zawiera **spójną nazwę urządzenia, poprawny czas, odpowiedni source-address i prawidłową drogę do kolektora**. To cztery oddzielne warunki. Sam \`set system syslog host ...\` ich nie zapewnia.

### System syslog a logowanie sesji SRX

Na SRX istnieją również mechanizmy związane z **security flow logs**, dokumentujące decyzje i sesje bezpieczeństwa. Nie są one tym samym co zwykły systemowy syslog o błędach interfejsu czy commitach. W zależności od trybu i platformy konfiguracja ruchu flow może być odrębna.

W rozdziale IV uczymy się podstaw zdarzeń systemowych. Nie będziemy konfigurować pełnego SIEM ani szczegółowego eksportu każdej sesji firewall, ponieważ to oddzielny temat.

### Uwaga na zalanie konsoli

Zbyt szerokie wysyłanie wszystkich informacji do wszystkich zalogowanych użytkowników może spowodować **log storm** — lawinę wiadomości, utrudniającą administrację. Nie konfiguruj globalnego wysyłania wszystkiego na terminale jako szybkiego rozwiązania braku monitoringu.

### Jak sprawdzić syslog end-to-end?

Po testowej zmianie nazwy, opisu portu lub logowaniu znajdź zdarzenie lokalnie na R1 i w odpowiednim czasie na \`LAB-SERVICES\`. Porównaj jego timestamp i identyfikator urządzenia. Zapisane ustawienie docelowego IP nie dowodzi, że pakiet został dostarczony.

**R1-vSRX — Operational Mode, odczyt przed raportem do NOC**

~~~junos
show configuration system syslog | display set
show log messages | last 40
show system uptime
~~~

**Dokumentacja:** [System Syslog Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/syslog-edit-system.html), [System Logging Source Address](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/security-source-address-edit-system-ntp-radius-syslog-tacacs.html).

---

## 4.11. Configuration archival — dlaczego historia rollback na routerze to jeszcze nie backup? [A]

### Gdyby jutro trzeba było wymienić R1

Wyobraź sobie, że R1 przestaje działać i nie da się już uruchomić jego systemu operacyjnego. Technicy mają w magazynie nowy vSRX lub fizyczny router zastępczy. Ktoś pyta: „Gdzie jest ostatnia zatwierdzona konfiguracja?”.

Z rozdziału III znasz lokalną historię commit oraz \`rollback 1\`. To bardzo wygodne, kiedy sam router działa. Ale jeśli nie można już odczytać jego pamięci, lokalna historia może być niedostępna razem z urządzeniem.

Dlatego administracja Enterprise potrzebuje **niezależnej kopii konfiguracji**, przechowywanej poza routerem. W Junos jedną z opcji jest **configuration archival**, czyli automatyczne przesyłanie zatwierdzonej konfiguracji na zewnętrzny serwer.

### Trzy różne formy zabezpieczenia konfiguracji

Nie mieszajmy trzech mechanizmów.

**Rollback history** pozwala przywrócić wcześniejsze wersje zatwierdzonej konfiguracji na urządzeniu, o ile te wersje są dostępne.

**Rescue configuration**, którą poznamy w 4.13, jest specjalnie zapisaną konfiguracją ratunkową. Administrator decyduje, kiedy jest wystarczająco dobra, aby stać się punktem powrotu.

**Off-device archive** to kopia wysłana poza urządzenie, np. na odpowiednio zabezpieczony serwer. Ten mechanizm pomaga, kiedy lokalna pamięć urządzenia lub sam router zostaną utracone.

Dobry projekt często wykorzystuje wszystkie trzy, ponieważ rozwiązują różne problemy.

### Co oznacza archiwizacja po każdym commit?

W Junos można zaplanować przesyłanie konfiguracji do archiwum, np. przy każdej zatwierdzonej zmianie. To nazywa się **\`transfer-on-commit\`**.

**R1-vSRX — Configuration Mode [edit private], włączenie samego mechanizmu archival**

~~~junos
set system archival configuration transfer-on-commit
show | compare
~~~

Nie mamy jeszcze podanego docelowego serwera i nie powinniśmy zatwierdzać tej części jako kompletnego rozwiązania backupowego. Konfiguracja archiwum potrzebuje rzeczywistego miejsca docelowego, działającej ścieżki sieciowej oraz autoryzacji do zapisu plików.

### SCP/SFTP i dlaczego nie podajemy hasła serwera

W zależności od modelu i release Junos może przekazywać archiwa przez wybrane protokoły, w tym **SCP** lub **SFTP**. Oba korzystają z SSH, ale sposób transferu i wymagania implementacyjne mogą się różnić. W nowym środowisku unikamy nieszyfrowanego przesyłania konfiguracji, ponieważ jej zawartość może ujawniać ważne informacje o sieci.

Dla przykładu załóżmy, że \`LAB-SERVICES\` ma odpowiednio przygotowany katalog archiwum, konto do przyjmowania backupów i serwer SSH. Wówczas można przygotować **szablon**:

**R1-vSRX — Configuration Mode [edit private], szablon kierunku SCP — NIEKOMPLETNY**

~~~junos
set system archival configuration archive-sites "scp://junos-backup@203.0.113.30/backup/junos/"
show | compare
~~~

W tym przykładzie **nie podajemy ani hasła, ani prywatnego klucza SSH**. Istnienie samego URL nie oznacza, że transfer się uda. Trzeba skonfigurować bezpieczny mechanizm uwierzytelniania, zweryfikować host key/fingerprint odbiornika i upewnić się, że docelowy katalog jest zapisywalny. Dokładny sposób ustawienia kluczy, wymiany danych i obsługi \`mgmt_junos\` zależy od platformy, wydania i polityki organizacji.

Nie próbuj „rozwiązać” problemu przez ustawianie hasła w jawnym URL i zapisanie go na GitHub. Tak samo nie akceptuj automatycznie dowolnego host key bez weryfikacji — oznaczałoby to ryzyko przesłania konfiguracji nieautoryzowanemu serwerowi.

### Czy \`transfer-on-commit\` gwarantuje, że plik dotarł?

Nie. Mechanizm może próbować wysłać konfigurację po commit, ale serwer może być nieosiągalny, konto zablokowane, katalog pełny albo połączenie odrzucone. Samo \`commit complete\` nie jest potwierdzeniem backupu.

Prawidłowy test to:

1. Sprawdzenie konfiguracji archiwizacji i routingu do serwera.
2. Wykonanie kontrolowanego commit na bezpiecznej zmianie.
3. Weryfikacja logów transferu na urządzeniu.
4. **Potwierdzenie istnienia właściwego pliku po stronie serwera.**
5. W bardziej zaawansowanym teście — bezpieczna próba wczytania jego kopii na izolowanym routerze i porównania zawartości.

Na niektórych platformach z odrębną management instance istnieją dodatkowe ograniczenia narzędzi transferowych. Dlatego projekt archiwizacji musi uwzględniać faktyczną wersję Junos, nie tylko adres serwera.

**R1-vSRX — Operational Mode, odczyt konfiguracji i wiadomości**

~~~junos
show configuration system archival | display set
show log messages | match "archive|transfer|scp|sftp"
show system commit
~~~

Gdy na serwerze nie ma pliku, nie ogłaszaj PASS mimo prawidłowej konfiguracji na routerze. Sprawdź po kolei: routing, nazwy/klucze, konto, prawa katalogu, dostępny dysk i logi obu stron.

**Dokumentacja:** [Back Up Configurations to an Archive Site](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/cli/topics/task/junos-software-system-management-router-configuration-archiving.html), [archival statement](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/archival-edit-system.html).

---

## 4.12. Logging i tracing — jak znaleźć przyczynę, gdy zwykły syslog nie wystarcza? [A]

### Syslog mówi, że coś się stało. Tracing pomaga dowiedzieć się dlaczego

W sekcji 4.10 nauczyliśmy się odczytywać zdarzenia systemowe. Wyobraź sobie teraz, że w późniejszym laboratorium OSPF sąsiedztwo R1–R2 co kilka minut zmienia stan. W logu widzisz informację o jego zerwaniu, ale nie znasz przyczyny. Czy routery nie uzgadniają parametrów? Czy zanika interfejs? Czy istnieje błąd komunikacji?

W tej sytuacji potrzebujesz bardziej szczegółowych danych. Junos udostępnia **traceoptions** — możliwość rejestrowania wybranych zdarzeń związanych z protokołem lub procesem. Nie jest to jednak synonim „zapisz wszystko, co dzieje się w całym systemie”.

### Trzy pytania, trzy różne rodzaje dowodów

Zwykły \`show log messages\` odpowiada na pytanie o wybrane **zdarzenia systemowe**.

Polecenia \`show\` dotyczące protokołów, np. \`show ospf neighbor\`, pokażą **bieżący stan danego mechanizmu**.

Natomiast \`traceoptions\` mogą pomóc zrozumieć **szczegółowe zdarzenia procesu** w określonym czasie.

Jeśli zaufasz tylko jednemu źródłu, łatwo wyciągniesz błędny wniosek. Na przykład \`show ospf neighbor\` może aktualnie pokazywać pełne sąsiedztwo, mimo że w logu kilka minut wcześniej nastąpiła zmiana stanu.

### Dlaczego tracing może być niebezpieczny?

Szerokie zapisywanie wszystkich informacji przez dłuższy czas może obciążać urządzenie i zapełniać pliki. W środowisku produkcyjnym niewłaściwe traceoptions uruchomione na protokole mającym tysiące sąsiedztw mogłyby nawet pogorszyć sytuację, którą próbujesz diagnozować.

Dlatego tracing powinien mieć **jasny cel, ograniczony zakres, przewidywany rozmiar logów i procedurę wyłączenia**.

W przyszłym labie OSPF można wykorzystać ograniczony tracing błędów. Nie uruchamiamy tutaj OSPF na pełnym DIAMOND-6R tylko po to, żeby mieć co logować.

**R1-vSRX — Configuration Mode [edit private], osobny wariant OSPF LAB, nieaktywowany w bazowym CH04**

~~~junos
set protocols ospf traceoptions file CH04-OSPF size 1m files 3 no-world-readable
set protocols ospf traceoptions flag error
show | compare
~~~

Jeśli OSPF w ogóle nie jest uruchomiony w tej wersji laboratorium, nie oczekuj użytecznych komunikatów o sąsiedztwie. To przygotowana składnia do ćwiczeń w rozdziale OSPF, po sprawdzeniu jej obsługi na konkretnym release. Nie wykonuj jej przypadkowo na rzeczywistym routerze core.

**R1-vSRX — Operational Mode, gdy tracing rzeczywiście uruchomiono w wariancie OSPF**

~~~junos
show log CH04-OSPF | last 30
show configuration protocols ospf traceoptions | display set
~~~

### Co zrobić po zakończeniu diagnostyki?

Ograniczone traceoptions powinny zostać usunięte albo dezaktywowane zgodnie z planem. Nie zostawiaj ich jako stale włączonego „narzędzia na przyszłość”, jeśli nie mają określonego celu operacyjnego.

**R1-vSRX — Configuration Mode [edit private], cleanup tylko własnego eksperymentu**

~~~junos
delete protocols ospf traceoptions
show | compare
~~~

Przed commit sprawdź, czy \`traceoptions\` nie zostały wcześniej świadomie skonfigurowane przez kogoś innego. W takim przypadku usuń tylko **własne zmiany**, zamiast kasować całą gałąź.

### A co z packet capture?

Nie myl rejestrowania zdarzeń protokołu z przechwytywaniem wszystkich pakietów. \`monitor traffic\` w Junos może pokazywać przede wszystkim ruch widoczny z perspektywy systemu, a nie każdy pakiet przekazywany sprzętowo lub programowo przez PFE.

Na SRX do diagnozy ruchu użytkowników mogą być potrzebne odpowiednie **security flow session**, dedykowane opcje packet capture albo inne narzędzia zależne od platformy. Nie wyciągaj wniosku „pakiet nie dotarł” tylko dlatego, że nie pojawił się w \`monitor traffic\`.

**Dokumentacja:** [OSPF/IS-IS Traceoptions](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/traceoptions-edit-protocols-is-is-ospf.html), [monitor traffic](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html).

---

## 4.13. Rescue configuration — świadomie zapisany punkt powrotu [A]

### Czy „ostatnia zatwierdzona” znaczy „ostatnia dobra”?

Nie zawsze. Możesz zatwierdzić pięć kolejnych zmian, które składniowo są poprawne, ale stopniowo pogarszają bezpieczeństwo lub routing. Lokalna historia commit zawiera zapisane wersje, lecz sama nie wie, która była najlepszym stanem do przywrócenia.

Dlatego Junos posiada **rescue configuration** — specjalnie zapisywaną przez administratora wersję ratunkową. Powinna pochodzić z momentu, w którym urządzenie działa prawidłowo i istnieje udokumentowana możliwość zarządzania nim.

Nie powstaje automatycznie po każdym commit. Ktoś musi świadomie ją zapisać.

### Jak zapisać rescue po zakończeniu onboardingu?

Najpierw sprawdź, czy działa dostęp administracyjny, konta, podstawowy routing management i właściwe usługi. Dopiero potem zapisz stan jako rescue.

**R1-vSRX — Operational Mode, zapis rescue tylko po zaakceptowaniu poprawnej konfiguracji**

~~~junos
request system configuration rescue save
~~~

W ten sposób nie wykonujesz rollback; jedynie przechowujesz zatwierdzoną konfigurację jako przyszły punkt ratunkowy.

**R1-vSRX — Operational Mode, weryfikacja rescue**

~~~junos
show system configuration rescue
show system commit
~~~

Nie wystarczy zapisać rescue raz na zawsze. Po istotnym redesignie sieci stary punkt może być już nieadekwatny. Aktualizację rescue wykonuj świadomie jako element procesu przyjęcia nowego baseline.

### Co się dzieje podczas \`rollback rescue\`?

To bardzo ważny detal z rozdziału III: **\`rollback rescue\` wczytuje konfigurację ratunkową do candidate**, a nie natychmiast zmienia działające urządzenie. Nadal musisz porównać ją z active, ocenić konsekwencje i zatwierdzić zmianę, jeśli rzeczywiście chcesz ją zastosować.

**R1-vSRX — Configuration Mode [edit private], symulacja odczytu rescue bez aktywacji**

~~~junos
rollback rescue
show | compare
~~~

Zanim klikniesz commit, sprawdź, czy rescue zawiera poprawną adresację, konta, klucze i wymagane ustawienia. Rescue utworzona zbyt wcześnie może np. nie mieć jeszcze działającego SSH lub konta lokalnego. Przywrócenie jej bez analizy mogłoby ponownie odciąć dostęp.

### Rescue a \`commit confirmed\`

Te mechanizmy rozwiązują inne problemy. **\`commit confirmed\`** pozwala warunkowo wprowadzić zmianę i automatycznie ją cofnąć, jeśli nie zostanie potwierdzona. **Rescue configuration** jest statycznym, świadomie zapisanym punktem ratunkowym, który administrator może wybrać do przywrócenia.

Nie wykonuj testu rescue w trakcie oczekującego \`commit confirmed\`. Najpierw zakończ lub pozwól wycofać warunkową zmianę, upewnij się co do stanu active i dopiero potem przeprowadź osobny eksperyment.

Jeśli rescue nie istnieje, nie próbuj zgadywać, że \`rollback rescue\` załaduje coś sensownego. Brak punktu ratunkowego to wynik **NOT READY**, który trzeba naprawić przed oddaniem urządzenia do produkcji.

**Dokumentacja:** [Rescue Configuration](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-rescue-configuration.html), [commit CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/commit.html).

---

## 4.14. Co oznacza „router gotowy do pracy”? [A]

### To nie jest liczba wykonanych poleceń

Spróbuj spojrzeć na R1 z perspektywy inżyniera, który jutro ma odebrać telefon o awarii. Czy będzie umiał zalogować się na router? Czy zobaczy, kto wcześniej zmienił konfigurację? Czy odczyta poprawną godzinę zdarzenia? Czy monitoring pokaże stan? Czy istnieje backup poza uszkodzonym urządzeniem?

Dopiero kiedy te pytania mają potwierdzone odpowiedzi, można mówić o świadomym **commissioningu**, czyli przygotowaniu urządzenia do użycia i formalnym przekazaniu do utrzymania.

Dobrze zaprojektowana pierwsza konfiguracja łączy kilka niezależnych elementów. Nie zastępują się nawzajem:

- Poprawna **tożsamość urządzenia**: hostname, oznaczenie roli i dokumentacja.
- **Dostęp administracyjny**: niezależna konsola, działające konto lokalne i test SSH z właściwego źródła.
- **Autoryzacja**: uprawnienia właściwe dla poszczególnych ról.
- **Centralne AAA**, jeżeli jest wymagane, oraz zweryfikowany scenariusz jego niedostępności.
- **Interfejsy i routing management**: działająca droga do usług administracyjnych.
- **Czas**: rzeczywista synchronizacja NTP, nie tylko poprawny timezone.
- **Monitoring i logi**: odpowiedzi SNMPv3 oraz odbiór syslog po drugiej stronie.
- **Backup i recovery**: zdalne archiwum, rescue i kontrolowany plan rollback.

To nie jest lista komend do wpisania w tej kolejności, lecz mapa zależności. Przykładowo nie przetestujesz NTP, jeśli nie ma działającej drogi do serwera. Nie zabezpieczysz SSH przez wyłączenie root, dopóki nie ma alternatywnego konta. Nie udowodnisz archiwizacji bez pliku po stronie odbiornika.

### Dlaczego \`show configuration\` to za mało?

Gdy wykonujesz \`show configuration system ntp\`, dowiadujesz się, jaki serwer został wpisany. Ale nie wiesz jeszcze, czy zegar rzeczywiście jest zsynchronizowany. Podobnie \`show configuration snmp\` nie dowodzi, że NMS potrafi wykonać authPriv GET.

Dla każdej usługi trzeba zatem rozdzielić **konfigurację**, **stan systemu** i **test wykonany z właściwej strony**.

| Obszar | Dowód lokalny na R1 | Dowód końcowy |
|---|---|---|
| Tożsamość i management | \`show version\`, \`show interfaces terse\` | Zgodność z dokumentacją i działająca konsola |
| SSH | \`show configuration system services ssh\` | Nowe połączenie z LAB-ADMIN |
| Local accounts i role | \`show configuration system login\` | Logowanie \`labadmin\` i ograniczone \`nocview\` |
| AAA | \`show configuration system authentication-order\` | Udane logowanie, reject i timeout, logi serwera |
| NTP | \`show ntp associations\`, \`show ntp status\` | Synchronizacja do poprawnego źródła |
| SNMPv3 | \`show snmp statistics\`, \`show snmp v3\` | Zewnętrzny authPriv GET |
| Syslog | \`show log messages\` | To samo zdarzenie na kolektorze |
| Archival | \`show configuration system archival\` | Prawidłowy plik w repozytorium backupów |
| Rescue | \`show system configuration rescue\` | Punkt ratunkowy nadający się do kontrolowanego odtworzenia |

Jeśli nie masz działającego serwera, którego dotyczy wiersz, odpowiednim wynikiem jest **NOT TESTED**, a nie PASS. To nie odbiera wartości podręcznikowi; daje uczciwy obraz możliwości laboratorium.

### Jeden read-only zestaw na zakończenie

**R1-vSRX — Operational Mode, przegląd gotowości przed handover**

~~~junos
show version
show system uptime
show system alarms
show interfaces terse
show route table inet.0
show configuration system | display set
show configuration interfaces | display set
show system users
show ntp associations
show ntp status
show snmp statistics
show log messages | last 30
show system commit
show system configuration rescue
~~~

Nie uruchamiaj całego zestawu wklejonego bezmyślnie do ticketu. Niektóre wyniki mogą zawierać informacje wrażliwe, a ich zakres może być szeroki. Do dokumentacji przekazuj potrzebne, zanonimizowane potwierdzenia.

### Co wpisać w handover?

Dobry dokument odbiorowy zawiera nazwę i rolę urządzenia, model/release, interfejs management i sposób dostępu, znane zależności od AAA/NTP/NMS/SIEM, wyniki testów, lokalizację backupów, sposób uzyskania konsoli, aktualny status rescue, a także ograniczenia i zadania pozostałe do wykonania.

Warto uwzględnić również informację, czego **nie przetestowano**. Router może zostać przyjęty warunkowo do laboratorium z brakującym serwerem NTP, ale nie powinien zostać ogłoszony kompletnym wdrożeniem Enterprise tylko dlatego, że wszystkie polecenia \`set\` mają poprawną składnię.

---

