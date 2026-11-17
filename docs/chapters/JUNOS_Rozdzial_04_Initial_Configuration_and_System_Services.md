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



---

## 4.4. Login classes — skoro login działa, dlaczego komenda jest zabroniona? [A]

### 4.4.1. „Mogę się zalogować, ale nie mogę zrobić show configuration”

Pomyśl o budynku przemysłowym. Jedna karta wejściowa otwiera recepcję, inna serwerownię, a przepustka administratora otwiera szafę i pozwala wymienić zasilacz. Fakt, że **bramka rozpoznała kartę**, nie oznacza, że możesz wejść do każdego pomieszczenia.

Tak samo w Junos: **authentication** odpowiada na pytanie „kim jesteś?”, a **authorization** — „co wolno Ci robić?”. Lokalnie rolę i uprawnienia wyznacza **login class**, zdalny TACACS+/RADIUS może dostarczać dodatkowe atrybuty zależne od integracji.

**Most z Cisco:** zwykłe \`username noc privilege 1\` kontra \`privilege 15\` i TACACS+ command authorization. Jednak Junos nie sprowadza uprawnień do jednego numeru 0–15. Ma klasy i **permission flags**, a także mechanizmy dopuszczania/odmawiania konkretnych komend i gałęzi konfiguracji.

### 4.4.2. Cztery klasy, które powinieneś rozpoznawać bez Google

Zanim stworzysz własną klasę, zrozum cztery wbudowane:
- **\`super-user\` / \`superuser\`** — administracja z pełnym zakresem uprawnień według systemu i wydania;
- **\`operator\`** — możliwość wykonywania typowych działań operacyjnych, m.in. clear, network, reset, trace, view; to **nie** znaczy to samo co read-only;
- **\`read-only\`** — ograniczona obserwacja;
- **\`unauthorized\`** — brak użytecznych uprawnień CLI.

W praktyce do NOC L1 możesz chcieć **mniej niż operator**, bo operator potrafi wykonać niektóre działania destrukcyjne. Czy osoba odpowiedzialna tylko za obserwację powinna móc czyścić sesje lub resetować statystyki? To decyzja bezpieczeństwa i change management.

**Urządzenie: LAB-SRX01 — Operational Mode, sprawdzenie własnych uprawnień**

~~~junos
show cli authorization
show system users
~~~

Po zalogowaniu jako \`labadmin\` i \`nocview\` porównaj wyniki. Nie zgaduj, co oznacza read-only na konkretnej platformie — pokaż dowód.

### 4.4.3. Własna klasa bez przypadkowego przyznania pełnego dostępu

Tworzymy \`LAB-NOC\`, która może wyświetlać informacje, ale nie ma dostępu do typowych operacji administracyjnych.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit]**

~~~junos
set system login class LAB-NOC permissions view
set system login class LAB-NOC idle-timeout 15
set system login user nocview class LAB-NOC
show | compare
~~~

Po commit sprawdź sesję nowo zalogowanego \`nocview\`. **15** oznacza tu minuty bezczynności. Nie każda próba \`show configuration\` musi działać przy każdym zestawie permission flags. Jeśli potrzebujesz szerszego wglądu, rozważ osobną flagę \`view-configuration\`, ale sprawdź **dokładną listę uprawnień i nazwy w \`permissions ?\` na danym release**.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], interaktywne odkrywanie flag**

~~~junos
edit system login class LAB-NOC
set permissions ?
~~~

Dla takiego polecenia \`?\` jest narzędziem pomocy CLI, nie linijką automatycznie importowaną z pliku \`.set\`.

### 4.4.4. Gotcha: „allow-commands” nie jest magicznym obejściem pozostałych przywilejów

Junos pozwala definiować m.in. **\`allow-commands\`, \`deny-commands\`, \`allow-configuration\`, \`deny-configuration\`** oraz ich warianty dopasowań. To kuszące: „dam NOC dostęp tylko do \`show route\` jednym regexem”.

Ale pamiętaj o dwóch warstwach:
1. **Permission flags** decydują o bazowym dostępie do grup poleceń i hierarchii.
2. **Reguły allow/deny** precyzują wyjątki, a ich semantyka zależy od wybranego wariantu regex.

Oficjalna dokumentacja ostrzega, że stara rodzina \`allow/deny-commands\` i nowsza \`*-regexps\` mogą stosować różny mechanizm dopasowania. **Nie wgrywaj przypadkowego wyrażenia regularnego z forum do produkcji**; testuj go na koncie ograniczonym w labie.

Dodatkowy haczyk: przy pewnych rozszerzonych regexach występują ograniczenia dotyczące \`load override\`; nie projektuj RBAC tak, jakby jedna flaga „all” plus deny regex dawała automatycznie bezpieczną izolację. [Junos User Access Privileges](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-access-privileges.html).

### 4.4.5. Model inżynierski do projektowania uprawnień

Gdy projektujesz role dla klienta, podziel zespół:
- **NOC Read-Only** — odczyt statusów i konfiguracji w dopuszczonym zakresie;
- **LAN Operator** — określone komendy operacyjne, np. testy sieciowe, tylko jeśli akceptuje to procedura;
- **Network Engineer** — zarządzanie wybranymi gałęziami i zmiany;
- **Break Glass Administrator** — awaryjna, audytowana rola z pełnymi uprawnieniami.

Nie utożsamiaj „operator” z „bezpieczne polecenia”. Uprawnienia są zależne od komendy i **jej parametrów**: na Cisco \`clear ip bgp *\` ma inne skutki niż \`show ip bgp summary\`, i w Junos ta sama intuicja jest cenna.

Źródła: [Login Classes Overview](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-login-class-overview.html), [show cli authorization](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-cli-authorization.html), [Login Class Permissions](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-access-privileges.html).

---

## 4.5. Metody uwierzytelniania — AAA i pułapka „serwer milczy” kontra „serwer odmawia” [A]

### 4.5.1. Centrala AAA przestała odpowiadać w piątek po 18:00

Wyobraź sobie, że administrujesz kilkuset urządzeniami. Każdy inżynier ma indywidualne konto, a prawo do komend jest zarządzane przez firmowy TACACS+. W piątek jeden z serwerów AAA przestaje odpowiadać. Czy zespół utraci dostęp do wszystkich switchy? Czy lokalny administrator uratuje sytuację?

Prawidłowa odpowiedź zależy od **authentication order**, odpowiedzi serwera i konfiguracji lokalnych kont. Nie wystarczy zapamiętać „Junos fallback to local”.

W Cisco \`aaa authentication login default group tacacs+ local\` wygląda znajomo, ale na Junos implementacja **\`system authentication-order\`** ma kluczowy szczegół: **brak odpowiedzi** z serwera i **jawny reject** nie zawsze prowadzą do tego samego wyniku.

### 4.5.2. Zanim włączysz AAA, narysuj przepływ dostępu

~~~mermaid
flowchart TD
  LOGIN["Nowa sesja SSH: labadmin"] --> KEY{"Klucz publiczny skonfigurowany\n i poprawny?"}
  KEY -->|Tak| ACCESS["Dostęp zgodny z login class / AAA"]
  KEY -->|Nie lub nie użyto| TAC["TACACS+ / RADIUS według order"]
  TAC -->|Accept| ACCESS
  TAC -->|Timeout / brak odpowiedzi| LOCAL["Próba local password\njako last resort"]
  TAC -->|Reject| CHECK{"Czy password jest\njawnie w authentication-order?"}
  CHECK -->|Tak| LOCAL
  CHECK -->|Nie| DENY["Odmowa logowania"]
  LOCAL -->|Valid| ACCESS
  LOCAL -->|Invalid| DENY
~~~

**Zastrzeżenie:** diagram upraszcza szczegóły autoryzacji, typy kluczy SSH i złożone sekwencje dwóch usług AAA. Dla konkretnego order użyj oficjalnej tabeli zachowania z dokumentacji. Schemat pokazuje najważniejszy kontrast **reject vs timeout**.

### 4.5.3. Bezpieczny pierwszy etap: local only

Dopóki nie ma uruchomionego serwera AAA w EVE-NG, przećwicz lokalne konta i ich login classes. Samo wpisanie \`authentication-order tacplus\` nie uruchamia serwera TACACS+ w sieci.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], wariant laboratoryjny LOCAL ONLY**

~~~junos
set system authentication-order password
show | compare
~~~

To działa z kontami skonfigurowanymi w \`system login\`. Pamiętaj, że prawidłowo skonfigurowane **SSH public keys** mogą być sprawdzane przed metodami z \`authentication-order\`, dlatego przy testach z hasłami odróżnij sposób logowania klienta SSH od ustawień samego Junos.

### 4.5.4. Wariant Enterprise — TACACS+ jako pierwsza próba, local password jako fallback

Zakładamy wyizolowane środowisko szkoleniowe, w którym serwer TACACS+ istnieje naprawdę pod \`192.0.2.40\` i jest osiągalny. **Jeżeli go nie ma, nie wykonuj tego wariantu.** Nie zapisuj tajnego shared secret do GitHuba.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], wariant demonstracyjny wymagający serwera AAA**

~~~junos
set system authentication-order [ tacplus password ]
set system tacplus-server 192.0.2.40 source-address 192.0.2.10
~~~

Osobno trzeba wprowadzić wymagany **shared secret** w bezpiecznej sesji/zarządzanym kanale, ale nie dostarczamy tu wrażliwej wartości:

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], szablon polecenia z obowiązkowym sekretami z vault, nie do bezpośredniego commit**

~~~text
set system tacplus-server 192.0.2.40 secret <Z_VAULT_NIE_W_GITHUB>
~~~

**To polecenie jest NIEKOMPLETNE bez rzeczywistego sekretu** i powyższy wiersz z nawiasami kątowymi nie jest przeznaczony do wklejenia. Trzeba także skonfigurować zgodne konto/template i mapowanie praw na serwerze AAA. Same \`tacplus-server\` oraz \`authentication-order\` nie gwarantują, że użytkownik dostanie oczekiwaną klasę.

**Uwaga o priorytecie:** gdy ustawisz \`[ tacplus password ]\`, Junos spróbuje TACACS+, a następnie metodę lokalną zgodnie z warunkami. Jeśli zamiast tego określisz samo \`tacplus\`, zachowanie przy remote REJECT będzie inne.

### 4.5.5. Nieuświadomiony gotcha: jawny remote REJECT nie musi pozwolić użyć lokalnego konta

Zapamiętaj konkretny przypadek, bo może Cię zaskoczyć w production outage.

**Wariant 1 — \`authentication-order tacplus\`:** jeśli serwer jest osiągalny i **odrzuca login**, Junos zwykle **kończy próbę odmową**, zamiast przechodzić do local password. Gdy **wszystkie serwery nie odpowiadają**, Junos przewiduje awaryjne sięgnięcie po local password.

**Wariant 2 — \`authentication-order [ tacplus password ]\`:** local password jest jawnie dodane do kolejności, więc po **reject** z TACACS+ również może zostać podjęta lokalna próba (zgodnie z konfiguracją i specyfiką metody).

Nie zakładaj, że wszystko to działa identycznie jak Cisco \`group tacacs+ local\`. Wciąż musisz sprawdzić, **czy hasło lokalne istnieje** dla danego użytkownika i jaka klasa nada uprawnienia.

**Urządzenie: LAB-SRX01 — Operational Mode, weryfikacja po poprawnym wdrożeniu AAA**

~~~junos
show configuration system authentication-order
show configuration system tacplus-server
show log messages | match "sshd|tac|login"
show cli authorization
~~~

Źródło potwierdzające dokładnie różnicę timeout/reject: [Authentication Order for RADIUS, TACACS+, and Local Password](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/junos-os-authentication-order.html).

### 4.5.6. Wariant RADIUS i kwestia \`source-address\`

Centralne AAA może opierać się na RADIUS (np. ClearPass lub ISE), również w sieciach Juniper. Tu kluczowy błąd operatorski polega na zapomnieniu **adresu źródłowego**, z którego Junos wyśle zapytanie. Serwer AAA może akceptować pakiety tylko od zarejestrowanych klientów. Gdy Junos wybierze inny source IP, prawidłowy shared secret nie pomoże.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], wariant RADIUS alternatywny wobec TACACS**

~~~junos
set system authentication-order [ radius password ]
set system radius-server 192.0.2.41 source-address 192.0.2.10
~~~

Tutaj również wymagany jest odpowiedni sekret i konfiguracja po stronie serwera; **nie uruchamiaj tego wariantu bez rzeczywistej infrastruktury RADIUS**. Pamiętaj, że \`system radius-server\` do administracyjnego logowania na router to **inny kontekst** niż \`access profile\` dla użytkowników 802.1X/captive portal.

**Most z Cisco ISE/ClearPass:** często spotykasz \`ip radius source-interface Loopback0\` lub \`ip tacacs source-interface\`. W Junos wskazujesz \`source-address\` w konkretnym obiekcie serwera, a dodatkowe routing instances wymagają świadomego projektu ścieżki.

### 4.5.7. AAA to także authorization i accounting

Możesz mieć poprawnie uwierzytelnionego użytkownika TACACS+, ale serwer nie mapuje go na właściwy lokalny template albo nie dostarcza Juniper-specific VSA. Użytkownik wchodzi i widzi ograniczoną klasę lub brak polecenia.

To nie oznacza „TACACS jest zepsuty”. Odróżnij:
- **Authentication** — kto to?
- **Authorization** — co wolno wykonać?
- **Accounting** — jakie działania i sesje odnotowano?

W Cisco te trzy A są dobrze znane. W Junos dochodzi przejrzysta hierarchia kont oraz login classes. Prawidłowy greenfield wymaga także logowania operacji administratorów i synchronizacji czasu (NTP), bo bez niej accounting i syslog mają wątpliwą wartość dowodową.

Dalsza weryfikacja: [TACACS+ Authentication](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/user-access-tacacs-authentication.html), [RADIUS Authentication](https://www.juniper.net/documentation/us/en/software/junos/user-access/topics/topic-map/user-access-radius-authentication.html).

---

## 4.6. Typy interfejsów i ich właściwości — nie pomyl warstwy fizycznej z management [A]

### 4.6.1. Dlaczego \`ge-0/0/0\` nie jest całym interfejsem IP

W Cisco na ISR wpisujesz \`interface Gi0/0/0\`, a dalej \`ip address\`, \`description\`, \`speed\` itd. Na Junos struktura zwykle ma co najmniej trzy warstwy:
- **interfejs fizyczny** (np. \`ge-0/0/0\`);
- **logical unit** (np. \`.0\`);
- **protocol family** (np. \`family inet\`, \`inet6\`, \`ethernet-switching\`).

To nie jest kosmetyka parsera. Fizyczny port może mieć parametry niezależne od konkretnego logicznego \`unit\`, a rodzina protokołów określa, czy pracujemy z IPv4/IPv6/switching.

~~~mermaid
flowchart TD
  PHY["ge-0/0/0 — port fizyczny"] --> DES["description / mtu / link status"]
  PHY --> U0["unit 0 — logical interface"]
  U0 --> INET["family inet — adres IPv4"]
  U0 --> INET6["family inet6 — adres IPv6 (jeśli używany)"]
  PHY --> U100["unit 100 — przykład subinterface"]
  U100 --> TAG["vlan-id 100 — przy prawidłowym vlan-tagging"]
~~~

**Ograniczenie przykładu:** nie każda kombinacja families/units może współistnieć na każdym porcie i urządzeniu; trunki Ethernet i switching wymagają właściwego typu encapsulation i trybu interfejsu.

### 4.6.2. Rzeczywiste zdjęcie portów — ale zdjęcie nie zastąpi Hardware Guide

Zanim zaczniesz podawać numer portu remote hands, zwróć uwagę na fizyczne elementy w sprzęcie. Widzisz na poniższej fotografii interfejsy SFP i porty o różnej roli. **Nie odczytasz jednak z tego zdjęcia mapowania \`fxp0\`, \`me0\`, \`ge-0/0/0\` swojego modelu.**

![Panel frontowy przełącznika Juniper QFX5100 z portami SFP i uplinkami; fotografia ilustracyjna starszej platformy, nie urządzenia laboratoryjnego](https://upload.wikimedia.org/wikipedia/commons/9/99/Juniper_Networks_QFX5100_ethernet_switch.jpg)

*Źródło: Juniper Networks, „Juniper Networks QFX5100 ethernet switch.jpg”, [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Juniper_Networks_QFX5100_ethernet_switch.jpg), licencja [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/), bez modyfikacji; obraz hostowany zewnętrznie. **Model historyczny wyłącznie jako ilustracja portów**, a nie rekomendacja zakupowa.*

Na realnym EX4400 OOB \`me0\` jest z tyłu urządzenia; na części SRX port \`fxp0\` też może być w innym miejscu. W każdym rolloutcie porównuj **Hardware Guide, \`show chassis hardware\`, \`show interfaces terse\`** i opis patchpanelu.

### 4.6.3. Adres na routerze SRX a port switchingowy EX

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], przykład L3 IPv4 na jednym porcie danych**

~~~junos
set interfaces ge-0/0/0 description "P2P-TO-LAB-R2"
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/30
~~~

To nie jest polecenie z założenia odpowiednie do każdego EX access. Na EX w roli L2 najpierw definiujesz switching, VLAN i ewentualny **IRB** jako bramę. Na SRX musisz też rozważyć **security-zone** i host-inbound albo flow policies; routing IP sam nie pozwala automatycznie na ruch użytkowników.

**Urządzenie: EX-ACCESS-01 — Configuration Mode [edit], przykład adresu bramy L3 na logicznym IRB, wyłącznie dla wspieranej konfiguracji i VLAN**

~~~junos
set interfaces irb unit 20 family inet address 10.20.0.1/24
~~~

Ten wiersz **nie tworzy sam VLAN 20 ani nie podłącza go do IRB**. Szczegółowy model VLAN/IRB jest w rozdziale 14. W tym miejscu uczymy się rozpoznawać **role interfejsów** i zależności.

### 4.6.4. Co oznaczają nazwy \`lo0\`, \`irb\`, \`ae\`, \`me0\`, \`fxp0\`, \`vme\`?

Zamiast kolejnej tabeli bez kontekstu, pomyśl o projekcie nowego biura:

**\`ge-/xe-/et-\`** wskazują zwykle fizyczne rodziny Ethernet i ich nominalne prędkości (np. 1/10/100G, z licznymi wyjątkami sprzętowymi). **\`lo0\`** to loopback — stabilny adres kontrolny, np. router ID, source dla usług, gdzie ma to sens. **\`irb\`** to L3 gateway dla domeny bridging/VLAN. **\`ae\`** to aggregated Ethernet (porównanie Cisco Port-channel). **\`me0\`** to management w wielu EX; **\`fxp0\`** na wielu routerach/SRX; **\`vme\`** może służyć do zarządzania Virtual Chassis.

**Gotcha:** **\`ge-0/0/0.0\` i \`ge-0/0/0\` to nie identyczne poziomy konfiguracji**. Pierwsza forma wskazuje logical interface, druga fizyczną nazwę. Na Junos \`unit 0\` nie jest wirtualnym portem „niezależnym od physical”.

### 4.6.5. Diagnostyka: port jest up, ale IP nie odpowiada

**Urządzenie: LAB-SRX01 — Operational Mode**

~~~junos
show interfaces terse
show interfaces ge-0/0/0 extensive
show route table inet.0
show arp
show configuration interfaces | display set
~~~

Czy fizyczny link jest up? Czy logical unit jest up? Czy ma adres we właściwej rodzinie? Czy next-hop ma ARP? Czy forwarding nie jest blokowany security policy? Dopiero potem analizuj uplink.

**Most z Cisco:** \`show ip interface brief\`, \`show interfaces\`, \`show arp\`, \`show ip route\` — ta sama logika troubleshooting, ale różne widoki danych.

### 4.6.6. Management routing instance — nie wpisuj „default route” bez zrozumienia, gdzie trafi

Przy przenoszeniu management do odrębnej instancji warto znać kontekst \`mgmt_junos.inet.0\`. W Junos **\`set system management-instance\`** może wydzielić tę instancję dla management, ale **należy sprawdzić wsparcie i dokładne zachowanie dla modelu/release**.

Nie pokazujemy tego jako bezrefleksyjnego „jednolinijkowego hardeningu”, ponieważ w istniejącym site taka zmiana może przenieść ruch NTP/AAA/syslog poza aktualną trasę, odcinając urządzenie od infrastruktury.

**Urządzenie: LAB-SRX01 — Operational Mode, odczyt przed decyzją o wydzieleniu management**

~~~junos
show configuration system management-instance
show route table inet.0
show route table mgmt_junos.inet.0
show interfaces terse | match "fxp|me|vme"
~~~

Brak tabeli \`mgmt_junos.inet.0\` na danym obrazie nie jest awarią; to sygnał, że konfiguracja/wersja może jej nie wykorzystywać. Więcej o routing instances w rozdziale 6.

Źródła: [Management Interface in a Dedicated Instance](https://www.juniper.net/documentation/us/en/software/junos/junos-getting-started/topics/topic-map/management-interface-in-non-default-instance.html), [EX4400 hardware initial configuration](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/topic-map/ex4400-configure-junos-os.html).

---

## 4.7. Configuration groups — jedna konfiguracja dla setek portów bez kopiowania 100 razy [A]

### 4.7.1. Skąd bierze się różnica między template a rzeczywistą konfiguracją?

Klient ma 120 switchy EX i jeden standard firmowy: \`description\` według konwencji, syslog do SIEM, NTP, wspólny parametr interfejsów. Masz dwa rozwiązania:

**Pierwsze:** kopiujesz 120 razy te same \`set\` do każdego urządzenia. Kiedy zmienia się NTP, musisz sprawdzić i poprawić 120 konfiguracji.

**Drugie:** definiujesz w Junos **configuration group**, a następnie decydujesz, gdzie ta grupa jest stosowana. Konfiguracja dziedziczona nie musi być zapisana bezpośrednio w każdej gałęzi.

**Most z Cisco:** trochę jak template lub profile w Cisco Catalyst Center, \`interface range\` albo skrypt Ansible, ale **configuration groups działają wewnątrz samego Junos** i są elementem hierarchii. Nie są zewnętrznym systemem orkiestracji ani nie uruchamiają się na wszystkich switchach firmy bez uprzedniego wdrożenia.

### 4.7.2. Własna grupa i jej aktywacja — dwie oddzielne czynności

Zaczynamy od bezpiecznego parametru tekstowego, aby nie ryzykować routingu.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], izolowany LAB**

~~~junos
set groups LAB-IDENTITY system login message "LAB managed by group LAB-IDENTITY"
set apply-groups LAB-IDENTITY
show groups LAB-IDENTITY
show system login | display inheritance
show | compare
~~~

**Haczyk:** samo \`set groups ...\` **nie musi niczego zmienić**. Dopiero \`apply-groups\` na odpowiednim poziomie daje dziedziczenie. To jak przygotowanie template, którego jeszcze nie przypisałeś.

**Ostrzeżenie labowe:** jeśli masz już skonfigurowany lokalnie \`system login message\`, wartość lokalna zwykle **ma pierwszeństwo** nad odziedziczoną i nie zobaczysz nowego banneru jako skutecznego ustawienia. To **celowy test precedence**, a nie awaria grup.

### 4.7.3. Pierwszeństwo i efekt „niewidzialnej konfiguracji”

Wyobraź sobie, że grupa wymusza \`system login message "GLOBAL"\`, ale w konfiguracji lokalnej ktoś wcześniej wstawił \`"SITE-LOCAL"\`. Którą wartość zobaczy użytkownik? **Lokalną**, bo konkretna konfiguracja na danym poziomie nadpisuje dziedziczoną.

Teraz masz **dwie grupy** stosowane do tego samego poziomu. Oficjalna dokumentacja mówi: **pierwsza nazwa na liście \`apply-groups\` ma wyższy priorytet** niż kolejne. Dodatkowo grupa zastosowana głębiej w hierarchii może mieć pierwszeństwo przed grupami z wyższego poziomu.

~~~mermaid
flowchart TB
  G1["Group A: NTP / banner"] --> APPLY["apply-groups [ A B ]"]
  G2["Group B: fallback defaults"] --> APPLY
  APPLY --> MERGE["Efektywna konfiguracja"]
  LOCAL["Local statement (konkretny poziom)"] -->|ma pierwszeństwo| MERGE
  MERGE --> SHOW["display inheritance / show | display inheritance"]
~~~

**Pytanie podchwytliwe:** czy \`show configuration system login\` musi jawnie pokazać wszystkie inherited statements? **Nie.** \`| display inheritance\` jest potrzebne, żeby zobaczyć, skąd wzięły się ustawienia.

**Urządzenie: LAB-SRX01 — Operational Mode, weryfikacja dziedziczenia**

~~~junos
show configuration groups | display set
show configuration system login | display inheritance
show configuration | display inheritance
~~~

### 4.7.4. Jak wyłączyć dziedziczenie tylko tam, gdzie jest wyjątek?

W prawdziwym projekcie jedna lokalizacja ma osobny syslog, drugi budynek inne NTP, a trzeci odmienny link. Nie usuwaj grupy globalnie dla wszystkich. Junos ma **\`apply-groups-except\`** pozwalające ograniczyć dziedziczenie na wybranym poziomie.

Należy wiedzieć, że **nie da się zastosować \`apply-groups-except\` na samym root** w taki sam sposób jak wewnątrz gałęzi. Dokładny zakres wyłączenia i wynikowy config sprawdź w \`display inheritance\`.

**Urządzenie: LAB-SRX01 — Configuration Mode [edit], usunięcie laboratoryjnego przykładu po ćwiczeniu, nie zatwierdzaj cudzych zmian**

~~~junos
delete apply-groups LAB-IDENTITY
delete groups LAB-IDENTITY
show | compare
~~~

**Gotcha: \`junos-defaults\`** to specjalna, ukryta grupa systemowa; nie można jej traktować jak dowolnego własnego template ani przypisać zwykłym \`apply-groups junos-defaults\`. Zapisz w pamięci różnicę między **własną configuration group** i **systemowymi defaults**.

### 4.7.5. A co z wildcard?

W grupach możesz używać **wildcard** (np. dla rodziny portów). To przydatne, gdy konfigurowane są setki interfejsów, ale również bardzo niebezpieczne: zbyt szerokie dopasowanie może wpłynąć na **uplink core** albo port zarządzania, choć chciałeś zmienić tylko access.

Nie pokazujemy tego na produkcyjnym porcie. W laboratorium najpierw używamy pojedynczego banneru, a dopiero potem testujemy wildcard z wyraźną listą pasujących interfejsów i kontrolą \`show | compare\` oraz \`display inheritance\`.

**Na pamięć:** \`groups\` definiuje; \`apply-groups\` włącza; \`apply-groups-except\` wyłącza dziedziczenie w danej gałęzi; local statement zwykle wygrywa; kolejność grup ma znaczenie.  
**Do zrozumienia:** configuration groups zwiększają powtarzalność, ale utrudniają intuicyjny odczyt rzeczywistej konfiguracji bez \`display inheritance\`.

Źródła: [Use Configuration Groups to Quickly Configure Devices](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/configuration-groups-usage.html), [apply-groups CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/apply-groups.html).

