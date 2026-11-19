# ARCHIWUM — poprzedni wariant rozdziału 6.11

**ARCHIVE ONLY — NIE jest topologią referencyjną DIAMOND.** Zachowano historyczne przykłady dual-stack na dwóch routerach wyłącznie do porównania i ewentualnej redakcji. Aktualny rozdział i wszystkie nowe laboratoria używają DIAMOND.

## 6.11. [LAB] Konfiguracja IPv4/IPv6, tras statycznych i default routes na dwóch vSRX

### 6.11.1. Dlaczego ten lab ma osobne hosty po obu stronach?

Chcemy przećwiczyć **pełny forwarding end-to-end**, nie tylko ping pomiędzy dwoma adresami tranzytowymi routerów. W rozdziale 2 i 5 dowiedzieliśmy się, że ping generowany przez RE nie jest tym samym testem co ruch klienta. Dlatego instalujemy:
- **R1-vSRX**, **R2-vSRX** — dwa niezależne routery/firewalle Junos;
- **PC-A-LINUX** — lekki host Linux w sieci OFFICE;
- **PC-B-LINUX** — lekki host Linux w sieci ERP;
- **trzy odizolowane segmenty EVE**: OFFICE, TRANSIT, ERP;
- dostęp console EVE do każdego z routerów i hostów, **niezależny od interfejsów danych**.

Dlaczego Linux, a nie koniecznie dwa VPCS? VPCS świetnie nadaje się do prostych testów IPv4, ale w niektórych buildach obsługa **statycznego IPv6 default gateway** jest ograniczona i wymaga SLAAC. Aby uczciwie testować równoległe IPv4/IPv6, Linux z `ip route` i `ip -6 route` daje większą kontrolę.

**Status PLANNED.** Nie wykonałem tego labu w Twoim EVE-NG. Nie ma rzeczywistych outputów potwierdzających PASS, nie zakładam też mapowania NIC w konkretnym obrazie vSRX. Nie modyfikujemy CH02/CH03/CH04/CH05 ani istniejących laboratoriów Cisco.

~~~mermaid
flowchart LR
  subgraph OFFICE["OFFICE — IPv4 192.0.2.0/24\nIPv6 2001:db8:10::/64"]
   PC1["PC-A-LINUX\n192.0.2.10\n2001:db8:10::10"]
  end
  subgraph TRANSIT["TRANSIT — IPv4 198.51.100.0/30\nIPv6 2001:db8:12::/64"]
   R1["R1-vSRX\nge-0/0/1 OFFICE\nge-0/0/0 TRANSIT"]
   R2["R2-vSRX\nge-0/0/0 TRANSIT\nge-0/0/1 ERP"]
  end
  subgraph ERP["ERP — IPv4 203.0.113.0/24\nIPv6 2001:db8:20::/64"]
   PC2["PC-B-LINUX\n203.0.113.10\n2001:db8:20::10"]
  end
  PC1 --- R1
  R1 ---|"198.51.100.1 ↔ .2\n2001:db8:12::1 ↔ ::2"| R2
  R2 --- PC2
  OOB["Konsole EVE\nniezależne od data path"] -.-> R1
  OOB -.-> R2
~~~

### 6.11.2. Plan adresacji i rola każdej trasy

| Węzeł | Port / interface | IPv4 | IPv6 | Gateway / funkcja |
|---|---|---|---|---|
| PC-A-LINUX | eth0* | `192.0.2.10/24` | `2001:db8:10::10/64` | R1 `.1` / `::1` |
| R1-vSRX | ge-0/0/1.0 | `192.0.2.1/24` | `2001:db8:10::1/64` | OFFICE |
| R1-vSRX | ge-0/0/0.0 | `198.51.100.1/30` | `2001:db8:12::1/64` | TRANSIT |
| R2-vSRX | ge-0/0/0.0 | `198.51.100.2/30` | `2001:db8:12::2/64` | TRANSIT |
| R2-vSRX | ge-0/0/1.0 | `203.0.113.1/24` | `2001:db8:20::1/64` | ERP |
| PC-B-LINUX | eth0* | `203.0.113.10/24` | `2001:db8:20::10/64` | R2 `.1` / `::1` |

*Na Linux interface może nazywać się `ens3`, `ens4` itd. **Najpierw sprawdź `ip -br link` i dopiero potem podmień nazwę w komendach.*** Zakładane `ge-0/0/0` i `ge-0/0/1` także potwierdź na obu routerach. W wielu obrazach vSRX istnieje osobne `fxp0` jako management.

**Jeden warunek techniczny:** oba routery mają działać w kontrolowanym, izolowanym środowisku bez produkcyjnego NAT/zewnętrznych łączy. Korzystamy wyłącznie z dokumentacyjnych prefixów IPv4/IPv6 (RFC 5737 i RFC 3849).

### 6.11.3. Preflight — jak wykryć, czy bazowa konfiguracja jest naprawdę świeża

Nie wolno bez zastanowienia wgrywać konfiguracji na urządzenie, które już ma security zones, fabryczny NAT, DHCP server czy przypisane porty. Zacznij od read-only.

**Urządzenie: R1-vSRX — Operational Mode, niezależna konsola**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration interfaces | display set
show configuration routing-options | display set
show configuration security | display set
show system users
show system commit
~~~

**Urządzenie: R2-vSRX — Operational Mode, niezależna konsola**

~~~junos
show version
show chassis hardware
show interfaces terse
show configuration interfaces | display set
show configuration routing-options | display set
show configuration security | display set
show system users
show system commit
~~~

Sprawdź, czy porty ge nie są już w strefach typu trust/untrust, czy mają adresy i czy nie występuje fabryczny NAT. Jeśli tak, skoryguj **konkretny obiekt w kopii nowego labu**, nie wykonuj przypadkowego `load factory-default` na istniejącej instancji. Lokalny admin/root i SSH/console zostały omówione w rozdziale 4; to **warunek wcześniejszy**.

### 6.11.4. Etap A — konfigurowanie IPv4/IPv6 interface na R1

Zaczynamy od podłączenia R1 do sieci OFFICE i TRANSIT. W Junos IPv4 jest w `family inet`, IPv6 w `family inet6`, a adres wiążemy z **logical unit 0**.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], TYLKO osobna instancja LAB**

~~~junos
set system host-name R1-vSRX
set interfaces ge-0/0/1 description "CH06-OFFICE-TO-PC-A"
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/24
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:10::1/64
set interfaces ge-0/0/0 description "CH06-TRANSIT-TO-R2"
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/30
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::1/64
show | compare
commit check
~~~

**Dwie ważne rzeczy:** `commit check` potwierdza składnię i część ograniczeń systemu, **nie obecność peer**. Dodanie `2001:db8:12::1/64` nie gwarantuje, że R2 ma już IPv6 ani że ICMPv6 przejdzie policy.

### 6.11.5. Etap B — interfejsy R2

**Urządzenie: R2-vSRX — Configuration Mode [edit private], TYLKO osobna instancja LAB**

~~~junos
set system host-name R2-vSRX
set interfaces ge-0/0/0 description "CH06-TRANSIT-TO-R1"
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.2/30
set interfaces ge-0/0/0 unit 0 family inet6 address 2001:db8:12::2/64
set interfaces ge-0/0/1 description "CH06-ERP-TO-PC-B"
set interfaces ge-0/0/1 unit 0 family inet address 203.0.113.1/24
set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8:20::1/64
show | compare
commit check
~~~

Po commit i aktywacji interface spodziewamy się connected IPv4 i IPv6:
- R1: `192.0.2.0/24`, `198.51.100.0/30`, `2001:db8:10::/64`, `2001:db8:12::/64`;
- R2: `198.51.100.0/30`, `203.0.113.0/24`, `2001:db8:12::/64`, `2001:db8:20::/64`.

Pamiętaj o lokalnych host routes /32 i /128.

### 6.11.6. Etap C — statyczna droga R1 do ERP

**Urządzenie: R1-vSRX — Configuration Mode [edit private], statyczne trasy IPv4/IPv6**

~~~junos
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.2
set routing-options rib inet6.0 static route 2001:db8:20::/64 next-hop 2001:db8:12::2
show | compare
commit check
~~~

To dwie *niezależne* trasy, każda w innej rodzinie. **Nie dodawaj default na R1**, bo w podstawowym scenariuszu R1 zna dokładnie docelową sieć za R2. Ograniczamy złożoność do minimum.

### 6.11.7. Etap D — domyślna droga R2 do R1

**Urządzenie: R2-vSRX — Configuration Mode [edit private], IPv4 i IPv6 default**

~~~junos
set routing-options static route 0.0.0.0/0 next-hop 198.51.100.1
set routing-options rib inet6.0 static route ::/0 next-hop 2001:db8:12::1
show | compare
commit check
~~~

**Pytanie:** dlaczego R2 nie potrzebuje osobnego static `192.0.2.0/24`? Bo default przez R1 obejmuje tę sieć, jeżeli nie istnieje bardziej szczegółowa konkurencyjna route. W prawdziwym WAN możesz woleć jawny prefix i filtrowanie, ale na tym poziomie pokażemy mechanikę LPM na default.

### 6.11.8. Etap E — strefy i polityki SRX: nie pozwól, by firewall udawał problem routingu

Dwa vSRX to **firewalle z routingiem**, a nie neutralne routery IP. Bez security policies pakiety klientów mogą być odrzucane niezależnie od prawidłowego RIB/FIB.

Dla jasności konfigurujemy dwie strefy na każdym węźle. W odizolowanym, niedostępnym z zewnątrz labie zezwolimy na **ICMP i ICMPv6** między nimi, żeby testować routing w obie strony. Na produkcji musiałbyś ograniczyć adresy, usługi i właściwe role aplikacji.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], strefy oraz minimalne policy dla LAB ICMP**

~~~junos
set security zones security-zone OFFICE interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone TRANSIT interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD match source-address any
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD match destination-address any
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD match application [ junos-ping junos-icmp6-all ]
set security policies from-zone OFFICE to-zone TRANSIT policy CH06-ICMP-FWD then permit
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV match source-address any
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV match destination-address any
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV match application [ junos-ping junos-icmp6-all ]
set security policies from-zone TRANSIT to-zone OFFICE policy CH06-ICMP-REV then permit
show | compare
commit check
~~~

**Urządzenie: R2-vSRX — Configuration Mode [edit private], strefy oraz polityki LAB**

~~~junos
set security zones security-zone TRANSIT interfaces ge-0/0/0.0 host-inbound-traffic system-services ping
set security zones security-zone ERP interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD match source-address any
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD match destination-address any
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD match application [ junos-ping junos-icmp6-all ]
set security policies from-zone TRANSIT to-zone ERP policy CH06-ICMP-FWD then permit
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV match source-address any
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV match destination-address any
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV match application [ junos-ping junos-icmp6-all ]
set security policies from-zone ERP to-zone TRANSIT policy CH06-ICMP-REV then permit
show | compare
commit check
~~~

**Zastrzeżenia przed uruchomieniem:** sprawdź w danym obrazie vSRX istnienie predefiniowanych aplikacji `junos-ping` i `junos-icmp6-all` oraz prawidłowe przypisanie portów do zones. Jeśli parser inaczej implementuje dany parametr, nie zastępuj go od razu `application any` na urządzeniu z siecią produkcyjną. Powyższa struktura jest **planowanym LAB baseline**, nie zweryfikowaną konfiguracją uruchomionej instancji.

### 6.11.9. Commit workflow — czytelny i bezpieczny

Na każdej z obu instancji wykonaj **`show | compare` i `commit check` przed warunkową aktywacją**. Nie zaczynaj automatycznej aktywacji, jeśli nie masz poprawnej konsoli i niezależnego kanału odzyskania.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], aktywacja R1 dopiero po preflight**

~~~junos
commit confirmed 5
~~~

**Urządzenie: R2-vSRX — Configuration Mode [edit private], aktywacja R2 dopiero po preflight**

~~~junos
commit confirmed 5
~~~

Po aktywacji wykonaj **nową sesję administracyjną**, read-only `show` i testy na interfejsach. **Nie wykonuj kolejnego `commit check` w czasie pending commit confirmed**, bo może on potwierdzić zmianę i wyłączyć timer. Jeśli service działa, potwierdź na każdym urządzeniu.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], świadome potwierdzenie po testach**

~~~junos
commit
~~~

**Urządzenie: R2-vSRX — Configuration Mode [edit private], świadome potwierdzenie po testach**

~~~junos
commit
~~~

**Jeśli test się nie powiedzie:** nie potwierdzaj zmian. Ustal przyczynę przez konsolę; pozwól na auto-rollback lub świadomie przywróć poprzednią konfigurację zgodnie z procedurą CH03. Przy tym scenariuszu nie wolno polegać wyłącznie na SSH przez port testowy, który może zostać wyłączony.

### 6.11.10. Etap F — konfiguracja hostów Linux

**Najpierw przeczytaj realną nazwę NIC**. Na lekkim Linuxie w EVE może być `eth0`, `ens3` itd. Wykonaj poniższe polecenia z niezależnej konsoli. Używamy zmiennej `IFACE`, żeby nie zakładać nazwy portu. Wartość dostosuj po odczycie.

**Urządzenie: PC-A-LINUX — Linux shell, read-only discovery**

~~~bash
ip -br link
ip -br address
ip -4 route
ip -6 route
~~~

**Urządzenie: PC-A-LINUX — Linux shell, tylko świeża odizolowana maszyna; po weryfikacji interfejsu**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 address add 192.0.2.10/24 dev "$IFACE"
sudo ip -6 address add 2001:db8:10::10/64 dev "$IFACE"
sudo ip -4 route replace default via 192.0.2.1 dev "$IFACE"
sudo ip -6 route replace default via 2001:db8:10::1 dev "$IFACE"
~~~

**Urządzenie: PC-B-LINUX — Linux shell, read-only discovery**

~~~bash
ip -br link
ip -br address
ip -4 route
ip -6 route
~~~

**Urządzenie: PC-B-LINUX — Linux shell, tylko świeża odizolowana maszyna; po weryfikacji interfejsu**

~~~bash
IFACE=eth0
sudo ip link set "$IFACE" up
sudo ip -4 address add 203.0.113.10/24 dev "$IFACE"
sudo ip -6 address add 2001:db8:20::10/64 dev "$IFACE"
sudo ip -4 route replace default via 203.0.113.1 dev "$IFACE"
sudo ip -6 route replace default via 2001:db8:20::1 dev "$IFACE"
~~~

**Ważny detal Linux:** te polecenia mają charakter **tymczasowy do reboot** w zależności od dystrybucji. Nie zapisujemy ich jako permanentnego network config dla produkcyjnego hosta. Nie używaj `ip addr flush` bez potrzeby — mógłbyś usunąć istniejącą adresację management VM.

### 6.11.11. Etap G — pierwsze pomiary: zanim popsujemy routing, musimy znać stan dobry

Najpierw testujemy bezpośrednią komunikację PC-A ↔ R1 oraz PC-B ↔ R2.

**Urządzenie: PC-A-LINUX — Linux shell, IPv4/IPv6 gateway**

~~~bash
ping -c 5 192.0.2.1
ping -6 -c 5 2001:db8:10::1
ip -4 neigh show
ip -6 neigh show
~~~

**Urządzenie: PC-B-LINUX — Linux shell, IPv4/IPv6 gateway**

~~~bash
ping -c 5 203.0.113.1
ping -6 -c 5 2001:db8:20::1
ip -4 neigh show
ip -6 neigh show
~~~

Dopiero potem test tranzytu między R1 a R2.

**Urządzenie: R1-vSRX — Operational Mode, po baseline commit**

~~~junos
show interfaces terse
show route 198.51.100.2
show route 2001:db8:12::2
ping 198.51.100.2 source 198.51.100.1 count 5
ping 2001:db8:12::2 source 2001:db8:12::1 count 5
show arp
show ipv6 neighbors
~~~

**Urządzenie: R2-vSRX — Operational Mode, po baseline commit**

~~~junos
show route 198.51.100.1
show route 2001:db8:12::1
ping 198.51.100.1 source 198.51.100.2 count 5
ping 2001:db8:12::1 source 2001:db8:12::2 count 5
~~~

Na końcu pełny test end-to-end.

**Urządzenie: PC-A-LINUX — Linux shell, end-to-end dual stack**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
ip -4 route get 203.0.113.10
ip -6 route get 2001:db8:20::10
~~~

**Urządzenie: PC-B-LINUX — Linux shell, kontrola drogi powrotnej oraz nowego flow w odwrotną stronę**

~~~bash
ping -c 5 192.0.2.10
ping -6 -c 5 2001:db8:10::10
ip -4 route get 192.0.2.10
ip -6 route get 2001:db8:10::10
~~~

**Oczekiwany efekt:** IPv4 i IPv6 działają w obie strony. **To prognoza, nie wynik zmierzony w EVE-NG**. Jeżeli którykolwiek ping nie działa, **nie przechodź do fault injection**. Najpierw sprawdź IPv6 ND, host firewall, policies, brakujące adresy/default, port mapping i istniejący fabryczny NAT.

### 6.11.12. Etap H — pokaż rzeczywiste różnice RIB/FIB/ARP/ND

**Urządzenie: R1-vSRX — Operational Mode, snapshot routingu**

~~~junos
show route summary
show route protocol direct
show route protocol local
show route protocol static
show route 203.0.113.0/24 extensive
show route table inet6.0
show route forwarding-table destination 203.0.113.10
show route forwarding-table family inet6 destination 2001:db8:20::10
show arp
show ipv6 neighbors
show security flow session
~~~

**Pytanie do samodzielnego przewidzenia:**
- Czy do ERP IPv4 wygrywa R1 connected czy static? **Static**, bo ERP nie jest bezpośrednio w lokalnej sieci R1.
- Czy na R2 do ERP wygrywa default czy connected? **Connected**, bo /24 jest dłuższy niż /0.
- Czy na R2 do PC-A wygrywa connected czy default? **Default**, bo R2 nie ma bezpośredniego `192.0.2.0/24`.
- Czy R1 zna IPv6 `2001:db8:20::/64`? **Tak**, z `inet6.0` static po commit.
- Czy `show arp` dowodzi stanu IPv6 sąsiadów? **Nie**, do tego jest `show ipv6 neighbors`.

### 6.11.13. LAB-FAULT A — usuwamy trasę R1 do ERP, ale nie uszkadzamy portu

**Cel:** wykazać różnicę między connected interface up a brakującą statyczną RIB route. Przed awarią upewnij się, że **R1 nie ma alternatywnego default** prowadzącego do ERP.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], wyłącznie izolowany LAB, tylko IPv4 static**

~~~junos
delete routing-options static route 203.0.113.0/24
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R1-vSRX — Operational Mode przez niezależną konsolę, po fault, bez kolejnego commit/check**

~~~junos
show interfaces terse
show route 203.0.113.0/24
show route forwarding-table destination 203.0.113.10
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell, podczas fault**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Przewidywany wynik:** IPv4 przestanie działać, ponieważ usunięto route, ale **IPv6 może nadal działać**, bo nie zmieniliśmy `inet6.0`. Właśnie pokazaliśmy, że IPv4/IPv6 to dwa niezależne RIB/FIB.

Po upływie timera bez potwierdzenia wróci poprzednia konfiguracja. **Nie wykonuj `commit check` w trakcie oczekiwania.** Sprawdź w historii commit i powtórz obydwa testy.

### 6.11.14. LAB-FAULT B — poprawna składnia, zły next-hop

**Cel:** zobaczyć różnicę między składnią konfiguracji a faktyczną osiągalnością next-hopa. Najpierw poczekaj, aż LAB-FAULT A zakończy auto-rollback.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], tylko LAB**

~~~junos
delete routing-options static route 203.0.113.0/24 next-hop 198.51.100.2
set routing-options static route 203.0.113.0/24 next-hop 198.51.100.200
show | compare
commit check
commit confirmed 2
~~~

Adres `198.51.100.200` nie jest connected do R1 w naszym labie. Junos może nie zakwalifikować tej static route do active i wskazać **unresolved/hidden** w odpowiednim widoku. Przewidywane zachowanie trzeba potwierdzić **na konkretnym release**, nie zgadywać z pamięci.

**Urządzenie: R1-vSRX — Operational Mode przez konsolę, analiza route reason**

~~~junos
show route 203.0.113.0/24 extensive
show route hidden
show route 198.51.100.200
show interfaces ge-0/0/0 terse
show system commit
~~~

**Pytanie:** dlaczego `commit check` przechodzi, skoro ping nie działa? Bo sprawdza **konfigurację**, nie dowodzi reachability do każdego sąsiada.

### 6.11.15. LAB-FAULT C — zniknęła droga powrotna, więc w jedną stronę pakiet może dojść

**Cel:** nauczyć się nie zapominać o return path. Najpierw upewnij się, że R1 i R2 powróciły do baseline. Z R2 usuń IPv4 default.

**Urządzenie: R2-vSRX — Configuration Mode [edit private], tylko izolowany LAB**

~~~junos
delete routing-options static route 0.0.0.0/0
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R2-vSRX — Operational Mode przez konsolę, odczyt braku return path**

~~~junos
show route 192.0.2.10
show route 0.0.0.0/0
show route 203.0.113.10
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Przewidywany wynik:** IPv4 z PC-A do PC-B przestaje działać, choć R1 może nadal wysłać Echo Request do PC-B. IPv6 może nadal działać, bo `::/0` na R2 nie został usunięty. Nie twierdź, że pakiety IPv4 „na pewno doszły do serwera”, jeśli nie zbierasz dowodu na PC-B, np. krótkim `tcpdump` w izolowanym labie.

**Urządzenie: PC-B-LINUX — Linux shell, opcjonalna obserwacja podczas fault, z odpowiednią autoryzacją**

~~~bash
sudo tcpdump -ni eth0 icmp -c 20
~~~

Zatrzymaj po uzyskaniu dowodu. Sprawdź auto-rollback R2 i ponowny ping.

### 6.11.16. LAB-FAULT D — krótsza trasa default nie powinna wygrać z dłuższą /24

**Cel:** pokazać LPM na R2 w praktyce. R2 ma default do R1, a jednak dla **własnej** sieci ERP `203.0.113.0/24` używa connected. Trasa o preference 5 nie „wygrywa” z connected /24 tylko dlatego, że jest static default.

**Urządzenie: R2-vSRX — Operational Mode, bez wykonywania żadnej zmiany**

~~~junos
show route 0.0.0.0/0
show route 203.0.113.0/24
show route forwarding-table destination 203.0.113.10
show route forwarding-table destination 192.0.2.10
~~~

**Interpretacja:** w R2 `203.0.113.10` pasuje do connected /24, a `192.0.2.10` do default /0. To jest LPM, bez konieczności zmieniania żadnej wartości preference.

### 6.11.17. LAB-FAULT E — IPv6 independently: pozorna poprawność IPv4 nie wystarcza

**Cel:** usuń tylko IPv6 static route R1 do ERP i porównaj wyniki dual-stack. Upewnij się, że wszystkie wcześniejsze confirmed timers są zakończone.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], tylko LAB IPv6**

~~~junos
delete routing-options rib inet6.0 static route 2001:db8:20::/64
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R1-vSRX — Operational Mode z konsoli, podczas fault**

~~~junos
show route table inet6.0
show route 2001:db8:20::10
show ipv6 neighbors
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell, porównaj rodziny**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

Jeżeli IPv4 działa, a IPv6 nie, diagnozuj **`inet6.0`, ND i ICMPv6 policy** zamiast podmieniać kabel. To jest bardzo dobry przykład różnicy między dual-stack działającym na jednym fizycznym porcie a **dwoma niezależnymi control-plane routing contexts**.

### 6.11.18. LAB-FAULT F — `discard` bardziej szczegółowy niż normalna static route

**Cel:** przećwicz LPM na statycznej trasie /32. Mamy normalną R1 static `203.0.113.0/24` przez R2. Dodajemy **host-specific discard** dla jednego IP PC-B.

**Urządzenie: R1-vSRX — Configuration Mode [edit private], tylko izolowany LAB**

~~~junos
set routing-options static route 203.0.113.10/32 discard
show | compare
commit check
commit confirmed 2
~~~

**Urządzenie: R1-vSRX — Operational Mode przez konsolę, podczas fault**

~~~junos
show route 203.0.113.10/32
show route 203.0.113.0/24
show route forwarding-table destination 203.0.113.10
show system commit
~~~

**Urządzenie: PC-A-LINUX — Linux shell, podczas fault**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Gotcha:** `discard /32` ma wyższy poziom szczegółowości niż normalna `static /24`. IP `203.0.113.10` zostanie blackholowany, mimo że /24 pozostaje aktywna i sprawna dla innych hostów. Preference **nie służy do porównywania /32 z /24**.

Po upływie timera nie wykonuj żadnego commit. Sprawdź, czy /32 zniknęła i IPv4 ping wrócił.

### 6.11.19. LAB-FAULT G — L2/ND nie jest tym samym co RIB

Chcemy pokazać przypadek, w którym connected i static są prawidłowe, ale **adres MAC sąsiada nie jest dostępny**. Najprostszy bezpieczny sposób na izolowanym labie to administracyjnie zatrzymać kartę testową PC-B-LINUX z **jego lokalnej konsoli**, nie z SSH.

**Urządzenie: PC-B-LINUX — Linux shell / konsola, tylko w izolowanym labie, wyłączenie portu danych**

~~~bash
sudo ip link set eth0 down
~~~

**Urządzenie: R2-vSRX — Operational Mode, obserwacja po wyłączeniu hosta**

~~~junos
show route 203.0.113.0/24
show arp
show ipv6 neighbors
show interfaces ge-0/0/1 terse
~~~

Możesz zobaczyć nadal **connected /24 i /64 na R2**, bo routerowy port jest nadal up, ale host `203.0.113.10` nie odpowiada na ARP, a `2001:db8:20::10` nie odpowiada na ND. Sąsiad mógł pozostać chwilowo w cache, więc obserwuj stan i timeout zamiast uznawać pojedynczy odczyt za ostateczny dowód.

**Urządzenie: PC-B-LINUX — Linux shell / konsola, przywrócenie portu po teście**

~~~bash
sudo ip link set eth0 up
~~~

Następnie ponownie przetestuj ARP/ND i ping. Jeśli nazwa NIC jest inna, użyj wcześniej ustalonej prawidłowej nazwy. Nie wykonuj tego polecenia na maszynie, której **jedyny dostęp administracyjny** zależy od wyłączanego interfejsu.

### 6.11.20. Oczekiwane wyniki — nie pomyl tabeli „predykcje” z realnym raportem

| Test | Hipoteza | Oczekiwane zachowanie |
|---|---|---|
| Baseline IPv4/IPv6 | Obie strony znają drogę i return path | Ping z PC-A do PC-B w obu rodzinach PASS |
| A: brak IPv4 route na R1 | Nie ma odpowiedniego LPM do ERP | IPv4 FAIL, IPv6 może działać |
| B: niedostępny next-hop | Static niekwalifikowana lub niewykonalna | Brak active route / route reason; fizyczny port nadal up |
| C: brak default IPv4 na R2 | Odpowiedź nie wraca do OFFICE | IPv4 FAIL, IPv6 może działać |
| D: connected vs default | LPM preferuje /24 nad /0 | Do własnego ERP connected, do OFFICE default |
| E: brak IPv6 static na R1 | Oddzielna rodzina routingowa | IPv6 FAIL, IPv4 PASS |
| F: bardziej szczegółowy discard | /32 wygrywa z /24 | IPv4 PC-B jest blackholed |
| G: PC-B NIC down | RIB nadal może mieć connected | ARP/ND failed i host unreachable |

Wszystkie powyższe to **przewidywane wyniki**. Nie stawiaj przy nich statusu PASS, dopóki nie zobaczysz prawdziwego outputu w EVE-NG.

### 6.11.21. Cleanup i finalny dowód zakończenia

Po każdej próbie:
- sprawdź `show system commit`; nie pozostaw żadnego pending `commit confirmed`;
- potwierdź, że route z baseline jest w właściwej tablicy;
- sprawdź, czy `203.0.113.10/32 discard` już nie istnieje;
- sprawdź status portów i sąsiadów;
- zrób test klienta w obu rodzinach;
- udokumentuj faktyczną godzinę, wersję Junos i obserwację.

**Urządzenie: R1-vSRX — Operational Mode, finalny odczyt**

~~~junos
show system commit
show interfaces terse
show route 203.0.113.0/24
show route 203.0.113.10/32
show route table inet6.0
show route forwarding-table destination 203.0.113.10
show arp
show ipv6 neighbors
show security flow session summary
~~~

**Urządzenie: R2-vSRX — Operational Mode, finalny odczyt**

~~~junos
show system commit
show route 0.0.0.0/0
show route table inet6.0
show interfaces terse
show security flow session summary
~~~

**Urządzenie: PC-A-LINUX — Linux shell, finalny test**

~~~bash
ping -c 5 203.0.113.10
ping -6 -c 5 2001:db8:20::10
~~~

**Nie wykonuj `rollback 0` w shared candidate zawierającym potencjalne cudze zmiany.** W izolowanym labie możesz przywrócić baseline do właściwej private candidate po potwierdzeniu, że pending timer zakończył się zgodnie z planem.

### 6.11.22. Co wpisujesz do `VERIFY.md` zamiast „it works”?

Dla każdego eksperymentu zapisz:
- dokładny obraz/vSRX release i ewentualny rodzaj host Linux;
- rzeczywiste mapowanie QEMU NIC → `fxp0/ge`;
- timestamp przed i po;
- baseline `show interfaces`, `show route`, FIB i ARP/ND;
- dokładne `show | compare` i sposób zatwierdzenia zmiany;
- odczyt podczas fault oraz jego interpretację;
- wynik PC-A → PC-B i test zwrotny;
- czy `commit confirmed` automatycznie przywrócił konfigurację;
- co nie zadziałało i jaki był root cause;
- końcowy rezultat PASS / FAIL / NOT TESTED.

**Ważna zasada:** nie publikuj w publicznym repozytorium danych rzeczywistych klientów, kont, haseł, kluczy, tokenów ani całej konfiguracji urządzenia z sekretami.

**Status sekcji 6.11:** **PLANNED / NOT RUNTIME VALIDATED** — czytelnik ma pełny instruktaż, ale potwierdzenie wykonania na konkretnym obrazie jest osobnym zadaniem.



---

