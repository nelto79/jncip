# Rozdział 4 — Initial Configuration and System Services
## Od urządzenia wyjętego z pudełka do zarządzalnego elementu sieci Enterprise

**Ścieżka:** JNCIA-Junos [A] → JNCIS-ENT → JNCIP-ENT  
**Zakres dokładnie według unified blueprint:** **4.1–4.14 [A]**, **4.15 [LAB]**  
**Stan merytoryczny:** 17 listopada 2026 · oficjalna dokumentacja Juniper/HPE + scenariusze autorskie  
**Platformy:** klasyczny Junos na EX/SRX/MX, kontekst Junos OS Evolved; laboratorium referencyjne **vSRX w EVE-NG**.  
**Status labu:** **PLANNED — nie zweryfikowano runtime**. Żaden przykładowy output w rozdziale nie jest przedstawiany jako rzeczywisty odczyt z Twojego urządzenia.

> **Umowa dydaktyczna:** w Cisco wiesz, jak postawić nowe urządzenie na nogi. Teraz chcemy zrobić to samo na Juniperze, ale zrozumieć każdą decyzję: *dlaczego ten port, skąd system zna godzinę, kto może się zalogować, co zostanie po utracie AAA i kto otrzyma informację o awarii?* Nie wystarczy „commit complete”. Włączony w sieć router ma być **bezpieczny, obserwowalny i możliwy do odzyskania**.

## Prolog: poniedziałek, 08:15 — instalator telefonuje z nowego biura

„W szafie stoi nowy Juniper. Kontrolka świeci na zielono. Wpiąłem go do patch panelu, ale nie mogę dostać się przez SSH. Na dwóch switchach widać ten sam domyślny hostname. NOC nie widzi ich w monitoringu. Logi mają niewłaściwą godzinę. Czy mogę już przekazać site do produkcji?”

W Cisco zaczynasz od \`show version\`, \`show run\`, \`show ip interface brief\`, AAA, NTP, syslog, SNMP i kopii konfiguracji. Z Juniperem będziesz myślał o dokładnie **tych samych obowiązkach operacyjnych**, lecz w innych miejscach hierarchii Junos.

Załóżmy, że projekt to dwa piętra biura:
- dwa przełączniki access EX4400;
- jeden brzegowy SRX (WAN/security);
- oddzielna sieć zarządzania;
- centralny TACACS+/RADIUS, NTP, syslog/SIEM, monitoring;
- SSH tylko z bastiona;
- backup konfiguracji i rescue na każdym urządzeniu.

Wybór konkretnego modelu, PoE i BOM przećwiczyliśmy w [rozdziale 1, punkcie 1.12](JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md). Teraz pozostaje pytanie bardziej niewygodne: **co musi być skonfigurowane, żeby operacje IT mogły takim site'em zarządzać o trzeciej nad ranem?**

~~~mermaid
flowchart LR
  ENGINEER["Inżynier / Bastion"] -->|SSH / konsola| OOB["Oddzielna sieć management"]
  OOB --> EX["EX4400 / me0"]
  OOB --> SRX["SRX / port właściwy modelowi"]
  EX -->|NTP| SERVICES["NTP / DNS / Syslog / NMS / AAA"]
  SRX -->|AAA / telemetry| SERVICES
  SRX -->|Firewall i WAN| USERS["VLAN-y biura i Internet"]
  EX -->|Access| USERS
~~~

**Co powinieneś zauważyć?** Zwykły ruch klientów, usługi zarządzające oraz dostęp ratunkowy to **trzy różne zależności**. Możesz mieć działające VLAN-y bez sysloga, działający SSH bez NTP i poprawny ping bez możliwości zalogowania się przez TACACS+. Zadaniem initial configuration jest zredukować liczbę takich ukrytych niewiadomych.

### Mapa podróży — czego chcę Cię nauczyć

W blueprint mamy **15 punktów**, a nie tylko „hostname + root password”. Przejdziemy kolejno: factory defaults (4.1), konfiguracja systemowa (4.2), lokalne konta (4.3), role (4.4), AAA (4.5), interfejsy (4.6), groups (4.7), NTP (4.8), SNMP (4.9), syslog (4.10), archiwizacja (4.11), tracing (4.12), rescue (4.13), zintegrowana weryfikacja (4.14) i pełne laboratorium (4.15).

**Na pamięć:** podstawowe hierarchie \`system\`, \`interfaces\`, \`routing-options\`, \`snmp\`; różnica \`me0/fxp0/irb\`; \`root-authentication\`; \`login user/class\`; \`authentication-order\`; \`apply-groups\`; \`system ntp\`; \`system syslog\`; \`system archival\`; \`request system configuration rescue save\`.

**Do zrozumienia:** to, że system się uruchamia, nie oznacza, że ma bezpieczny profil startowy; AAA timeout i AAA reject to dwa różne zdarzenia; routing management nie zawsze ma osobny VRF; \`show configuration\` może nie pokazać wartości odziedziczonych; NTP/syslog/SNMP mają różne drogi i kierunki ruchu; „mam rollback” nie oznacza „mam off-device backup”.

**Do przećwiczenia:** inicjalizacja przez konsolę, druga niezależna sesja administracyjna, RBAC, pomiar RTT do NTP/AAA, wysyłka sysloga, SNMPv3 authPriv z NMS, kontrolowany błąd DNS/NTP/ACL, backup i powrót do rescue — wyłącznie na odizolowanym labie.

---

## 4.1. Factory-default state — czy „z pudełka” znaczy to samo na SRX i EX? [A]

### 4.1.1. Pierwszy haczyk: jeden producent, wiele różnych punktów startowych

Wyobraź sobie, że pracownik remote hands przysyła zdjęcie przodu urządzenia i pyta: „Jakie IP ma Juniper domyślnie?”. Jeśli odpowiesz od razu \`192.168.1.1\`, możesz popełnić błąd jeszcze przed otwarciem PuTTY.

**SRX300 z typową fabryczną konfiguracją branch** i **EX4400 przygotowany do pierwszego uruchomienia** to nie identyczne profile bezpieczeństwa:

- Dla **SRX300** dokumentacja sprzętowa opisuje fabryczny układ \`trust/untrust\`; port WAN \`ge-0/0/0\` ma inną rolę niż porty LAN \`ge-0/0/1..6\`, połączone przez \`irb.0\` ze stanem usługi DHCP, z fabrycznym adresem \`192.168.1.1/24\`. To **opis określonego modelu i obrazu domyślnego**, nie uniwersalna obietnica dla każdej gałęzi SRX.
- Dla **EX4400** oficjalny Quick Start wskazuje dedykowany port management **\`me0\`** oraz pierwszą konfigurację przez console. Sprawdzasz, czy port ma fabryczne DHCP, czy działa provisioning/ZTP, a nie zgadujesz konkretny adres IP.
- Dla **vSRX w EVE-NG** zachowanie startowe zależy od obrazu, sposobu uruchomienia i mapowania wirtualnych NIC. Wcześniejszy snapshot może już zawierać konta, routing lub policies — dlatego **laboratoryjny vSRX nie jest automatycznie stanem factory-default**.
- Na części nowych platform **Junos OS Evolved** zmieniają się reguły początkowego logowania i zachowanie systemu. Nie przejmuj na ślepo procedury starego SRX300.

**Most z Cisco:** to ta sama pułapka, gdy ktoś miesza fabryczną konfigurację Catalyst 9300, Cisco Business, ISR i FTD. Nazwa producenta nie jest specyfikacją domyślnego IP ani polityki dostępu.

Źródła producenta: [SRX300 — Configuring Junos OS](https://www.juniper.net/documentation/us/en/hardware/srx300/topics/topic-map/srx300-configuring-junos.html), [EX4400 — initial configuration](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/topic-map/ex4400-configure-junos-os.html).

### 4.1.2. Co naprawdę oznacza fabryczne logowanie \`root\`?

W instrukcjach wielu klasycznych urządzeń Juniper znajdziesz informację o początkowym logowaniu \`root\` **z lokalnej konsoli bez ustawionego hasła**. To okno do pierwszej konfiguracji, **nie stan, w którym wolno pozostawić sprzęt w produkcji**.

**Nie zakładaj, że „root bez hasła” działa przez SSH.** Na przykład nowe platformy Junos OS Evolved mogą od razu blokować zdalne logowanie root. W praktycznym rolloutcie dostęp konsolowy i prawidłowy kanał zarządzania muszą być zaplanowane, zanim aktywujesz zdalne usługi.

Na Cisco root w tej formie nie występuje w IOS XE CLI; łatwiej myśleć o lokalnym administracyjnym dostępie console przed konfiguracją AAA. Różnica Junos: **\`root-authentication\`** jest elementem hierarchii, a hasło root często jest wymagane do pierwszego \`commit\` na minimalnym systemie.

**Urządzenie: LAB-SRX01 — Operational Mode (wyłącznie odczyt stanu, jeśli już jesteś w CLI)**

~~~junos
show version
show chassis hardware
show configuration | display set
show interfaces terse
show system alarms
~~~

Wyjściem z tego kroku nie jest „wykonano \`show\`”, lecz odpowiedzi: **jaki model, jaki Junos, jakie interfejsy, co jest skonfigurowane i czy urządzenie jest na pewno świeże?**

### 4.1.3. Zero-touch provisioning nie jest pustą konfiguracją

W nowoczesnym campie access możesz spotkać **ZTP/PnP/Mist onboarding**. Nowy EX podłączony do właściwej sieci może pobrać informacje potrzebne do zarządzania i konfiguracji z infrastruktury. Jeżeli bez sprawdzenia włożysz taki switch do cudzej sieci DHCP, możesz przypadkowo uruchomić proces, którego nie planowałeś.

**Gotcha:** \`factory-default\` to nie synonim „zero procesu w tle”. Właśnie dlatego pierwszy test i instalacja powinny odbywać się w **kontrolowanej sieci provisioningowej lub przez konsolę**, z wiedzą o polityce Mist/ZTP.

Na Cisco analogią jest PnP/ZTP na przełącznikach Catalyst. Nie traktuj jednak protokołów i formatów ofert DHCP jako identycznych.

### 4.1.4. \`load factory-default\` kontra wymazywanie urządzenia

Mamy dwa zupełnie różne cele:

1. **Wczytać fabryczny szablon** do candidate, aby zobaczyć, jaka konfiguracja byłaby po zatwierdzeniu.
2. **Wyzerować sprzęt do przekazania, zwrotu lub ponownego użycia**, usuwając wrażliwe dane, konta i lokalne pliki zgodnie z procedurą sanitizacji.

Polecenie **\`load factory-default\`** jest operacją na candidate, którą można porównać przed commit. Nie należy jej utożsamiać z pełną procedurą sanitizacji. W oficjalnym przewodniku EX4400 producent zauważa, że po \`load factory-default\` hasło root może zostać zachowane i procedura EZsetup nie musi ponownie wystartować.

Polecenia typu **\`request system zeroize\`** są destrukcyjne, a ich dokładny zakres i warianty zależą od platformy i wydania. **Nie zamieszczamy ich jako poleceń do automatycznego uruchomienia w naszym labie.** W produkcji mogą wywołać utratę zarządzania, plików, kluczy i konfiguracji.

**Urządzenie: LAB-SRX01 — Configuration Mode (wyłącznie ANALIZA składni, nie zatwierdzaj bez izolowanego labu i konsoli)**

~~~junos
load factory-default
show | compare
~~~

Jeśli nie masz absolutnej pewności, że candidate należy wyłącznie do Ciebie i jest to izolowany lab, **nie wykonuj tego ćwiczenia**. Po obejrzeniu diffu wróć do poprzedniej candidate tylko w kontrolowanym środowisku. Nie wykonuj przypadkowego \`commit\`.

### 4.1.5. Pułapka wersji: domyślne \`netconf ssh\` może zniknąć

Dokumentacja SRX300 podaje bardzo konkretną zmianę: **począwszy od Junos OS 25.2R1 fabryczna konfiguracja nie zawiera już automatycznie \`system services netconf ssh\`**. Czyli automatyzacja, która zakłada NETCONF od pierwszego startu na każdym SRX300, może przestać działać, mimo że samo SSH nadal działa.

**Urządzenie: LAB-SRX01 — Operational Mode**

~~~junos
show version
show configuration system services | display set
~~~

Nie próbuj rozwiązywać nieobecnego NETCONF przez „otwórz wszystkie porty”. Najpierw sprawdź, czy usługa jest włączona, który interfejs/strefa ją dopuszcza i czy rzeczywiście chcesz udostępniać NETCONF do narzędzia automatyzacyjnego.

**Do zapamiętania:** factory defaults są **per model i per release**. **Do przećwiczenia:** porównaj output twojego vSRX z dokumentacją dokładnego PID/obrazu. 

---

## 4.2. Initial configuration — tożsamość, bezpieczny dostęp i management [A]

### 4.2.1. Hostname to nie ozdoba: od niego zależy obsługa P1

W incydencie użytkownik opisuje problem lokalizacji „FRA-Office-2”, ale monitorowanie pokazuje trzy elementy \`router\`. Czy w logach, alertach i archiwach konfiguracji rozpoznasz właściwą lokalizację? Zły hostname nie zatrzyma od razu BGP, ale **może wydłużyć outage**, bo SOC/NOC stracą wiarygodną identyfikację urządzenia.

W Cisco zwykle konfigurujesz \`hostname\`, \`ip domain-name\`, DNS oraz NTP. W Junos naturalnym miejscem jest **\`system\`**.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], po potwierdzeniu izolacji**

~~~junos
set system host-name LAB-SRX01
set system domain-name lab.example
set system time-zone Europe/Warsaw
set system name-server 192.0.2.53
~~~

**Komentarz dydaktyczny:** \`lab.example\` to domena dokumentacyjna; serwer DNS \`192.0.2.53\` **jest przykładem** i nie musi istnieć w Twoim labie. Dodanie adresu DNS w konfiguracji **nie stworzy serwera DNS**. Jeśli serwis DNS ma być testowany, uruchom go w topologii.

**Most z Cisco:**
- \`hostname SITE-EX01\` ↔ \`set system host-name SITE-EX01\`;
- \`ip name-server\` ↔ \`set system name-server\`;
- \`clock timezone\` i konfiguracja NTP ↔ \`set system time-zone\` i \`system ntp\`.
Nie jest to kopiowanie 1:1: systemy różnią się zakresem CLI, obsługą strefy czasowej i sposobem zasilania usług z DNS.

### 4.2.2. Dwie bramy: routowana sieć użytkowników i sieć zarządzania

Wyobraź sobie EX4400 jako L3 access switch. Ma interfejsy \`irb\` dla VLAN-ów użytkowników, ale również dedykowane **\`me0\`** na tylnym panelu, przeznaczone do OOB management. Z drugiej strony vSRX może eksponować **\`fxp0\`**, a część innych platform używa innych nazw interfejsów zarządzania.

**Pułapka:** sam fakt użycia \`me0\` lub \`fxp0\` **nie dowodzi**, że urządzenie ma niezależną routing instance dla zarządzania. Możliwe jest współdzielenie domyślnej tablicy; w niektórych projektach konfiguruje się **\`system management-instance\`** i tablicę \`mgmt_junos.inet.0\`. Zmiana tego zachowania może odciąć SSH, NTP, syslog, TACACS+ i backup.

Dla nowego site ustalamy przed wpisaniem pierwszego \`set interfaces\`:
- **adres i maskę** interfejsu OOB;
- **gateway i routing instance**, jeśli potrzebna;
- **skąd przyjdzie SSH** i jak będzie filtruje dostęp do RE;
- **którędy wychodzi AAA/NTP/syslog/SNMP/archiwizacja**;
- czy bastion/monitoring ma fizyczną drogę do urządzenia po awarii campus/core.

~~~mermaid
flowchart LR
  ADMIN["Bastion 192.0.2.20"] --> SWITCH["OOB switch/segment 192.0.2.0/24"]
  SWITCH --> SRX["LAB-SRX01\nfxp0.0 192.0.2.10"]
  SWITCH --> SVR["LAB-SERVICES\n192.0.2.30"]
  SWITCH --> GW["OOB gateway 192.0.2.1"]
  SRX -. "niezależna płaszczyzna ruchu" .-> DATA["ge-0/0/x\ntraffic/firewall"]
~~~

**To jest ilustracja docelowego labu, nie odkryta automatycznie topologia EVE-NG.** Przed wdrożeniem potwierdź mapowanie QEMU NIC i obecność \`fxp0\`.

### 4.2.3. Dwie możliwe ścieżki konfiguracji OOB — nie mieszaj ich w jednej zmianie

Jeśli urządzenie i wydanie rzeczywiście używają \`fxp0.0\` jako zarządzania oraz **domyślnej tablicy \`inet.0\`**, możesz ustawić adres w \`interfaces\` i trasę do sieci zarządzania przez właściwy gateway. Ale jeśli włączono **\`management-instance\`**, potrzebujesz odpowiedniego routing context dla ruchu OOB.

**Wariant A — LAB-SRX01, klasyczny vSRX z \`fxp0.0\` i bez dedykowanej management-instance; Configuration Mode [edit]**

~~~junos
set interfaces fxp0 unit 0 family inet address 192.0.2.10/24
~~~

To wystarczy do komunikacji z bastionem i serwerami **na tej samej podsieci**. Dla ruchu poza podsiecią potrzebujesz świadomie zaprojektowanej trasy (np. statycznej w właściwej tablicy), a nie „magicznego default gateway”.

**Wariant B — EX4400 z portem OOB \`me0\`; Configuration Mode [edit], przykład osobnego urządzenia**

~~~junos
set interfaces me0 unit 0 family inet address 192.0.2.11/24
~~~

**Nigdy nie wklejaj obu wariantów do tego samego urządzenia** — są odpowiednie dla różnych platform. Jeśli fabrycznie masz na \`me0.0\` klienta DHCP, przy przejściu na adres statyczny najpierw sprawdź jego konfigurację i usuń sprzeczne elementy zgodnie z planem. Jeśli używasz **Virtual Chassis**, interfejs zarządzania może mieć postać **\`vme\`**, więc przeczytaj dokumentację topologii.

### 4.2.4. Root password i SSH: kolejność ma znaczenie

Znasz z Cisco sytuację, w której nowy router ma brakującego lokalnego użytkownika, źle ustawione \`aaa new-model\` albo VTY ACL. W Junos równie łatwo doprowadzić do lockout, jeśli zaczynasz od zakazu root SSH, zanim sprawdzisz drugą działającą metodę logowania.

**Bezpieczna kolejność:**
1. Konsola/OOB i pełna identyfikacja urządzenia.
2. Ustawienie hasła root i osobnego konta administratora.
3. Włączenie SSH na właściwych interfejsach i w odpowiednich strefach/filtrach.
4. Test NOWEJ sesji SSH z bastiona, nie tylko utrzymanie starej sesji.
5. Ograniczenie zdalnego login root i usunięcie zbędnych serwisów.
6. Zapisanie rescue **dopiero po weryfikacji całości**.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], ustawienie hasła root z interaktywnym promptem**

~~~junos
set system root-authentication plain-text-password
~~~

System poprosi o hasło i potwierdzenie, nie wpisuj go jako dosłownego fragmentu w publicznym GitHubie. Hasło zostanie zapisane w konfiguracji w formie hasha. **Nie używaj \`encrypted-password\` z hasłem podanym zwykłym tekstem** — ta opcja oczekuje już poprawnie zaszyfrowanej wartości i może skutkować utratą logowania.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], udostępnienie SSH pod warunkiem prawidłowego management access**

~~~junos
set system services ssh
set system services ssh root-login deny
~~~

**Uwaga:** na SRX samo \`set system services ssh\` nie zawsze wystarczy. Jeżeli połączenie przychodzi do interfejsu w security zone, musisz odpowiednio skonfigurować **\`host-inbound-traffic system-services ssh\`** (a do tego filtering i routing). Dla dedykowanego \`fxp0\` model ochrony może być inny. Zwróć uwagę również na fakt, że same SSH i NETCONF/SSH to osobne usługi w hierarchii.

**Ważna granica bezpieczeństwa:** powyższe ustawienie \`root-login deny\` ma sens dopiero **po** działającym osobnym koncie administracyjnym. W labie możesz odroczyć tę linijkę do punktu 4.3 i końcowego prechecku.

Źródła: [root-authentication](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/root-authentication-edit-system.html), [EX4400 Quick Start](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/topic-map/ex4400-configure-junos-os.html), [SSH root-login](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/ssh-edit-system.html).

### 4.2.5. Commit to dopiero początek testu

W rozdziale 3 omawialiśmy \`show | compare\`, \`commit check\` i \`commit confirmed\`. W initial setup jest jeszcze jeden etap: przetestuj **zalogowanie na nowe konto z nowego klienta**, ponieważ stara sesja console/SSH nie dowodzi sprawności nowej konfiguracji.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], dopiero po ustawieniu działającego konta w 4.3**

~~~junos
show | compare
commit check
commit confirmed 5
~~~

**Haczyk:** po \`commit confirmed\` **nie uruchamiaj ponownie \`commit check\` tylko z przyzwyczajenia** — na Junos może to potwierdzić oczekujący commit i wyłączyć timer rollback. Sprawdzaj stan **operacyjnie** (\`run show system commit\`) i realnym testem usługi; dopiero potem świadomie potwierdź commit.

---

## 4.3. Konta użytkowników lokalnych — nie zostawiaj jednego wspólnego admina [A]

### 4.3.1. Dlaczego \`root\` nie powinien być Twoim codziennym loginem

Masz trzy osoby w zespole, a wszystkie logują się na \`root\`. Incydent: ktoś zmienił routing policy i nie wiadomo kto. W logu widzisz jedynie jedno konto. To problem nie tylko security, ale także **operacyjnego przypisania odpowiedzialności**.

W Cisco tworzysz lokalne \`username ... privilege 15\` albo pracujesz z AAA. W Junos podstawowy obiekt to **\`system login user\`**, a zakres uprawnień daje **login class** (rozdział 4.4). Na urządzeniach produkcyjnych najlepiej mieć **osobiste konta**, centralne AAA oraz **kontrolowane konto break-glass**, dostępne gdy sieć zarządzania/AAA jest niedostępna.

### 4.3.2. Tworzenie użytkownika krok po kroku

Chcemy dwóch kont w labie: \`labadmin\` do pełnej administracji i \`nocview\` do obserwacji. Pierwsze jest odpowiednikiem indywidualnego uprzywilejowanego użytkownika Cisco; drugie później otrzyma login class ograniczającą uprawnienia.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], konta laboratoryjne z hasłami wpisanymi interaktywnie**

~~~junos
set system login user labadmin full-name "Junos Lab Administrator"
set system login user labadmin class super-user
set system login user labadmin authentication plain-text-password
set system login user nocview full-name "NOC Read Only"
set system login user nocview class read-only
set system login user nocview authentication plain-text-password
~~~

W trakcie \`plain-text-password\` CLI poprosi o sekret i potwierdzenie. **Nie umieszczaj przykładowego prawdziwego hasła w pliku Markdown**, nawet jeśli używasz \`lab\`. Hasło w logach, terminal scrollback i GitHub jest później trudne do usunięcia.

Przed commit sprawdź, czy nie ma konfliktów i czy nie nadpisałeś cudzych kont. Po zatwierdzeniu przeprowadź **nową sesję** osobno dla obu użytkowników.

**Urządzenie: LAB-SRX01 — Operational Mode, weryfikacja po commit i teście logowania**

~~~junos
show configuration system login | display set
show system users
show log messages | match "login|sshd"
~~~

Wynik \`show configuration\` może wyświetlać zaszyfrowane elementy kont; nie kopiuj pełnego outputu do publicznego repo ani ticketu z szerokim dostępem.

### 4.3.3. SSH keys — zamiast wklejać hasła do skryptów

W środowisku automatyzacji zamiast hasła w skrypcie wolisz **klucz publiczny** użytkownika z ograniczonymi uprawnieniami. Junos dopuszcza różne typy kluczy; ich wsparcie zależy od modelu, polityki kryptograficznej, FIPS i wydania.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], przykład z jawnym kluczem PUBLICZNYM (nie klucz prywatny)**

~~~junos
set system login user labadmin authentication ssh-ed25519 "ssh-ed25519 AAAA...TU-WKLEJ-ZWERYFIKOWANY-KLUCZ-PUBLICZNY..."
~~~

**To nie jest kompletny działający klucz** — przykład pokazuje miejsce i strukturę. Nie commituj tekstu z wielokropkiem. W realnym labie wklej poprawny własny **public key** i sprawdź, czy dany release wspiera \`ssh-ed25519\`.

Alternatywnie możesz użyć mechanizmu \`load-key-file\` do pobrania klucza publicznego z wcześniej dostępnego pliku. To lepsze niż kopiowanie długiego klucza z terminala, ale wymaga wiarygodnego źródła i sprawdzenia zawartości.

**Gotcha:** **klucz publiczny** zapisany w Junos nie nadaje uprawnień sam z siebie. O tym, co użytkownik może zrobić, decyduje jego **class** i odpowiednie policy uprawnień.

### 4.3.4. Kiedy można wyłączyć login root przez SSH?

Dopiero gdy:
- \`labadmin\` ma poprawne hasło lub klucz i odpowiadającą mu klasę;
- SSH jest osiągalny z bastiona i nowa sesja rzeczywiście loguje się na \`labadmin\`;
- istnieje metoda awaryjna console/OOB;
- nie jesteś w środku niepotwierdzonego \`commit confirmed\` z ryzykiem utraty dostępu.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], hardening po pomyślnym teście nowego administratora**

~~~junos
set system services ssh root-login deny
show | compare
~~~

Następnie stosujesz test \`commit confirmed\` oraz weryfikujesz **nową sesję**, zanim zatwierdzisz. Nie odcinaj sobie ostatniej metody administracji.

**Do zapamiętania:** root password, individual login, login class i centralne AAA to **cztery odrębne elementy**, nawet jeśli w codziennym języku nazywamy wszystko „logowaniem”.

Źródła: [Junos User Accounts](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-user-accounts.html), [Root Password](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/user-access-root-password.html).

