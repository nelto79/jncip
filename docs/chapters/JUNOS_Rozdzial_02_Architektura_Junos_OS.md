# Rozdział 2. Junos OS Fundamentals — architektura systemu

**Podręcznik:** JUNOS Enterprise Routing & Switching — JNCIA-Junos (JN0-106) → JNCIS-ENT (JN0-352) → JNCIP-ENT (JN0-650)  
**Status:** rozdział szkoleniowy 1.0, 15 listopada 2026 r.  
**Zakres wg wspólnego blueprintu:** 2.1–2.6 (Associate, fundament niezbędny także na wyższych poziomach)  
**Odbiorca:** inżynier znający Cisco IOS/IOS XE, odświeżający Junos OS i przygotowujący się docelowo do JN0-650.  
**Lab:** vSRX na EVE-NG + 2 hosty VPCS; polecenia nie wymagają uruchamiania vJunos-router ani vJunos-switch.  

> **Jak czytać ten rozdział.** Punkty **2.1–2.6** odpowiadają dokładnie numeracji wcześniejszego `JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md`. Sekcje **2.LAB**, **2.TS** i **2.TEST** są rozszerzeniami dydaktycznymi — nie udają dodatkowych, literalnych pozycji blueprintu. Komendy podano dla trybu operacyjnego (`>`) lub konfiguracyjnego (`#`). Przykładowe wyniki są **ilustracyjne** i mogą różnić się od konkretnej wersji oraz platformy. Nie wykonuj zmian z rozdziału na urządzeniu produkcyjnym.

## Mapa nauki — co musisz umieć po rozdziale

| Cel | Wiedza i umiejętność | Powiązanie z blueprintem |
|---|---|---|
| 2.1 | Wyjaśnić architekturę Junos, modularność procesów i zależności pomiędzy komponentami | JNCIA — Software architecture |
| 2.2 | Odróżnić control plane, forwarding/data plane i management plane | JNCIA — Control and forwarding planes |
| 2.3 | Wyjaśnić rolę RE, procesów `rpd`, `mgd`, `chassisd`, `dcd` i jądra systemu | JNCIA — Routing Engine |
| 2.4 | Wyjaśnić rolę PFE; rozróżnić RIB/FIB i sposób instalacji tras | JNCIA — Packet Forwarding Engine |
| 2.5 | Opisać krok po kroku przechodzenie pakietu tranzytowego przez router/switch | JNCIA — Transit traffic processing |
| 2.6 | Rozpoznać ruch kierowany do RE i ruch generowany lokalnie; analizować wyjątki | JNCIA — Exception traffic |

### Poziomy ważności

- **MUSISZ ZNAĆ (egzamin):** definicje RE/PFE, control vs forwarding, transit vs exception, ścieżka routingu od protokołu do tablicy forwarding.
- **MUSISZ POTRAFIĆ (praktyka):** właściwie dobrać `show` do problemu i nie utożsamiać pingowania routera z przechodzeniem ruchu przez router.
- **ROZSZERZENIE (pod JNCIS/JNCIP):** synchronizacja FIB, diagnozowanie zjawisk „trasa jest, ruch nie działa”, ochrona control plane, wpływ stateful flow na vSRX.

---

## 2.1. Architektura oprogramowania Junos OS [A]

### 2.1.1. Czym jest Junos OS i po co ma taką architekturę?

Junos OS to system operacyjny urządzeń sieciowych Juniper: routerów, przełączników i — w odpowiednich odmianach — firewalli SRX. Jego zadaniem nie jest wyłącznie udostępnienie CLI: system ma utrzymywać stan protokołów, programować sprzęt odpowiedzialny za przekazywanie pakietów, zarządzać interfejsami, udostępniać konfigurację i raportować awarie.

Junos rozdziela **wyznaczanie decyzji** (np. „która trasa BGP jest najlepsza?”) od **wykonywania decyzji dla każdego pakietu** (np. „wyślij pakiet z portu ge-0/0/1”). Dlatego protokoły routingu mogą stale przeliczać topologię, a odpowiednio skonstruowany forwarding plane może przenosić duży strumień danych bez angażowania głównego CPU w każdy pakiet.

**Analogia:** RE jest dyspozytorem, który przygotowuje instrukcję przejazdu. PFE to bramki i zwrotnice realizujące instrukcję dla każdego pojazdu. W zwykłej sytuacji każdy samochód nie pyta dyspozytora, którędy jechać.

### 2.1.2. Architektura logiczna — od protokołu do portu

```text
  Zarządzanie: SSH / CLI / NETCONF / syslog
                      |
          +-----------v-------------+
          |     ROUTING ENGINE      |   CONTROL + MANAGEMENT
          |  mgd  rpd  chassisd      |
          |  dcd  kernel  ...       |
          |  konfiguracja aktywna   |
          |  RIB: inet.0 / inet6.0  |
          |  master forwarding FIB  |
          +-----------+-------------+
                      | programowanie / synchronizacja
          +-----------v-------------+
          |  PACKET FORWARDING      |   DATA / FORWARDING
          |       ENGINE (PFE)      |
          |  lookup / filtry / QoS  |
          |  next-hop / L2 / L3     |
          +-----+-------------+-----+
                |             |
             port A         port B
                |--- dane ---->|
```

**Ważne:** to schemat **logiczny**, nie mapa fizycznego połączenia wewnątrz każdego modelu. W MX/EX/QFX część funkcji PFE często realizują dedykowane układy ASIC; w vSRX funkcje przetwarzania pakietów realizuje oprogramowanie wirtualnego urządzenia. Zasada rozdzielenia ról pozostaje przydatna, lecz nie należy zakładać, że każdy model ma dwa fizycznie osobne procesory.

### 2.1.3. Modułowość i procesy (daemons)

Junos nie jest jednym wielkim procesem obsługującym wszystko. Składa się z wielu współdziałających procesów systemowych. Każdy odpowiada za określony zakres i zwykle pracuje we własnej przestrzeni pamięci. Upraszcza to analizę błędów i ogranicza skutki części awarii; **nie znaczy jednak**, że awaria krytycznego procesu nigdy nie wpływa na forwarding.

| Proces / komponent | Najważniejsze zadanie | Gdy sprawia kłopot |
|---|---|---|
| `rpd` — routing protocol daemon | OSPF, IS-IS, BGP, RIB, wybór tras, routing policy | Sąsiedztwa nie powstają, trasy nie są odświeżane |
| `mgd` — management daemon | CLI, zarządzanie konfiguracją i zatwierdzaniem zmian | Problemy z poleceniami, commit lub sesją zarządzania |
| `chassisd` | Nadzór nad chassis, kartami, stanem sprzętu i alarmami | FPC/PIC offline, alarmy chassis |
| `dcd` / procesy interfejsów | Obsługa konfiguracji i stanu interfejsów | Problemy z interfejsami logicznymi/fizycznymi |
| `snmpd`, `mib2d` | Udostępnianie danych monitoringowych SNMP | Brak odczytów SNMP, choć forwarding działa |
| Kernel | Komunikacja między procesami i płaszczyznami, synchronizacja forwarding | Problemy z programowaniem tras, interfejsami lub zasobami OS |

Nie każdy proces występuje w tej samej postaci na każdej platformie i wersji. W szczególności **Junos OS Evolved** może inaczej organizować komponenty systemu.

### 2.1.4. Junos OS klasyczny a Junos OS Evolved

- **Junos OS:** w klasycznej architekturze opiera się na FreeBSD; zależnie od platformy może działać w różnych wariantach sprzętowych i wirtualnych.
- **Junos OS Evolved:** wykorzystuje jądro Linux i bardziej rozproszony model węzłów (*nodes*). Wspólne idee RE/PFE, routingu i CLI pozostają, ale wybrane polecenia oraz szczegóły monitoringu mogą być inne.
- **vSRX:** to platforma bezpieczeństwa korzystająca z Junos i dodatkowej warstwy przetwarzania *flow-based* — nie jest wierną symulacją przełącznika EX ani całego routera MX.

Na egzaminie JNCIA najważniejsze jest rozumienie logicznych funkcji, a nie pamięciowa znajomość procesów jądra Linux czy FreeBSD. Jednak dla diagnostyki na poziomie JNCIP różnice platformowe stają się istotne.

### 2.1.5. Pierwsze polecenia rozpoznawcze

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show version
show system uptime
show chassis hardware
show system processes extensive | match "rpd|mgd|chassisd|dcd"
show system alarms
show chassis alarms
```

**Co sprawdzasz?**

1. `show version` — jakiego Junos i jakiej platformy dotyczy dalsza diagnostyka.
2. `show system uptime` — czy restart systemu lub usługi nie tłumaczy problemu.
3. `show chassis hardware` — co urządzenie identyfikuje jako elementy sprzętowe/wirtualne.
4. `show system processes...` — czy podstawowe procesy istnieją (obciążenie oceniaj pełniejszym widokiem, a nie tylko filtrem tekstowym).
5. `show system alarms` i `show chassis alarms` — odróżnienie alarmów systemowych od sprzętowych.

> **Pułapka praktyczna:** brak procesu w wyniku `| match` nie musi oznaczać awarii — nazwy, sposób wyświetlania i układ procesów mogą się różnić. Najpierw ustal wersję systemu i sprawdź pełny wynik.

### 2.1.6. Porównanie do Cisco

| Junos | Przybliżona analogia w Cisco IOS XE | Ważna różnica |
|---|---|---|
| RE | Control plane / RP | RE to szerszy komponent operacyjny, nie tylko proces protokołu |
| PFE | Data plane / ASIC / QFP | Fizyczna realizacja zależy od modelu; QFP jest technologią wybranych platform Cisco |
| `rpd` | Procesy routingu IOS XE | W Junos jeden główny routing daemon obsługuje wiele protokołów |
| RIB (`inet.0`) | `show ip route` | W Junos wiele rodzin/tabel i routing instances |
| Master FIB (`show route forwarding-table`) | `show ip cef` (przybliżenie) | To nie zawsze bezpośredni odczyt tablicy z układu forwarding |

---

## 2.2. Rozdzielenie control plane i forwarding plane [A]

### 2.2.1. Trzy pojęcia: control, forwarding i management

**Control plane** buduje informacje pozwalające podejmować decyzje: tworzy sąsiedztwa OSPF, odbiera aktualizacje BGP, oblicza najlepsze trasy, utrzymuje RIB i przekazuje wyniki do mechanizmów forwarding.

**Forwarding plane (data plane)** wykonuje przekazywanie pakietów na podstawie już przygotowanych struktur. Przykładowe działania to: odczyt docelowego adresu IP, dopasowanie prefiksu, wybór next-hopu, obsługa L2/ARP/ND, modyfikacje nagłówków, zastosowanie filtrów i wysłanie pakietu odpowiednim portem.

**Management plane** obejmuje dostęp operatora i automatyzacji: CLI/SSH, NETCONF, SNMP, syslog. W podstawowym modelu Junos wiele tych funkcji działa na RE. To pojęcie organizacyjne, niekoniecznie trzeci osobny procesor.

### 2.2.2. Dlaczego rozdzielenie jest korzystne?

- **Skalowanie:** większość pakietów nie wymaga uruchomienia protokołu routingu ani kodu CLI.
- **Stabilność:** zmiany w topologii mogą być obliczane na RE, podczas gdy wcześniej zaprogramowane ścieżki nadal przenoszą ruch.
- **Bezpieczeństwo:** można oddzielnie ograniczać dostęp do funkcji zarządzania i ruch do RE.
- **Diagnostyka:** niezależnie sprawdzasz stan protokołu, tablicę routingu, tablicę forwarding i stan interfejsu.

**Ale uwaga:** rozdział nie jest absolutną gwarancją dostępności. Niektóre awarie RE, kernel, PFE, synchronizacji lub konfiguracji mogą zatrzymać ruch; HA (GRES/NSR i podobne) to osobne mechanizmy zwiększania odporności.

### 2.2.3. Jak pakietowi pomaga control plane, skoro zwykle go nie widzi?

Przykład: OSPF wyznacza trasę do `203.0.113.0/24` przez `10.0.12.2`.

```text
  OSPF neighbor/LSDB
          |
          v
     rpd: wybór trasy
          |
          v
      RIB (inet.0)
          |
          v
    master FIB na RE
          |
   synchronizacja do PFE
          |
          v
   lookup pakietu 203.0.113.55
          |
          v
   next-hop 10.0.12.2 -> port wyjściowy
```

RE nie otrzymuje każdego nowego pakietu do `203.0.113.55`; bierze udział przede wszystkim w tworzeniu i aktualizowaniu instrukcji forwarding.

### 2.2.4. Jak badać obie płaszczyzny?

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show route table inet.0
show route forwarding-table
show interfaces terse
show interfaces ge-0/0/1 extensive
show system processes extensive
```

**Sposób czytania:** `show route` odpowiada na pytanie, **co wybrał routing**; `show route forwarding-table` pokazuje **forwarding table po stronie RE**; `show interfaces` mówi, **czy interfejs może fizycznie/logicznie pracować**. Jeżeli wszystkie trzy wyglądają dobrze, nie kończy to jeszcze diagnostyki SRX — sesje i security policy mogą blokować przepływ.

### 2.2.5. Typowy błąd myślowy

> „SSH do routera działa, więc data plane też działa.” — **Nie**. SSH sprawdza ścieżkę dostępu do funkcji management, a nie całą trasę tranzytową przez PFE.  
> „Ping do adresu routera nie działa, więc router nie routuje.” — **Też nie**. Może być blokowany *host-inbound* albo filtr chroniący RE, podczas gdy ruch przechodzący przez urządzenie działa.

---

## 2.3. Routing Engine (RE) — rola i funkcje [A]

### 2.3.1. Co dokładnie robi RE?

RE odpowiada za:

1. Uruchomienie i nadzorowanie odpowiednich usług OS i procesów sieciowych.
2. Obsługę konfiguracji urządzenia — CLI, zatwierdzanie zmian i zarządzanie aktywną konfiguracją.
3. Utrzymywanie protokołów sterujących (OSPF, BGP, IS-IS itd.).
4. Utrzymywanie routing information base (RIB) i wybór aktywnych tras.
5. Tworzenie oraz aktualizowanie master forwarding table, przekazywanej dalej do PFE.
6. Monitorowanie stanu systemu i interfejsów, alarmy, logi, telemetria/management.
7. Obsługę pakietów, które rzeczywiście wymagają kontaktu z control plane.

### 2.3.2. RIB, aktywna trasa i FIB — fundamentalne rozróżnienie

**Routing Information Base (RIB)**: zbiór informacji o trasach, które urządzenie zna z różnych źródeł. Dla IPv4 podstawową tablicą jest zwykle `inet.0`, a dla IPv6 `inet6.0`.

Jeśli do tego samego prefiksu istnieje kilka kandydatów, Junos wybiera trasę aktywną zgodnie z zasadami preferencji i selekcji. Nie wszystko, co jest w RIB, musi zostać użyte do forwarding.

**Forwarding Information Base (FIB)**: dane wykorzystywane do szybkiego kierowania pakietów do właściwego next-hopu/interfejsu. W praktyce ważne są dwie perspektywy: master forwarding table po stronie RE i tablica faktycznie zainstalowana w PFE. Ich zgodność jest ważna w diagnostyce.

**Nie myl:** `show route` ≠ `show route forwarding-table` ≠ `show pfe route`.

| Polecenie | Co oglądasz | Uwaga |
|---|---|---|
| `show route` | Wybrane aktywne trasy w RIB | Nie potwierdza samodzielnie programowania PFE |
| `show route ... extensive` | Szczegóły tras i ich stan | Może pokazać powody braku aktywności po dobraniu odpowiedniego widoku |
| `show route forwarding-table` | Master FIB po stronie RE | Nie jest to zawsze bezpośredni dump forwarding ASIC |
| `show pfe route` | Tablicę tras PFE | Dostępność i składnia zależą od platformy/release |

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show route table inet.0
show route 198.51.100.0/24 detail
show route forwarding-table family inet
show route forwarding-table destination 198.51.100.0/24
```

### 2.3.2a. Jak wygląda wynik w praktyce — czytanie trasy

Załóżmy, że w laboratorium skonfigurowano interfejs `ge-0/0/2.0` z adresem `198.51.100.1/24`. Odczyt trasy może wyglądać następująco (**przykładowy, uproszczony output, nie zrzut z rzeczywistego urządzenia**):

**Urządzenie: vSRX-R1 — przykład wyniku operational**

```text
user@vSRX-R1> show route 198.51.100.10

inet.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

198.51.100.0/24   *[Direct/0] 00:01:23
                    > via ge-0/0/2.0
```

Czytaj kolejno: `inet.0` to IPv4 RIB; `198.51.100.0/24` jest dopasowanym prefiksem; `Direct/0` oznacza trasę bezpośrednią z domyślną preferencją 0; `*` wskazuje aktywny wybór; `via ge-0/0/2.0` podaje interfejs. **Nie ma tu jeszcze dowodu**, że host H2 odpowie na ARP, że port działa albo że policy SRX pozwoli na transit.

Dalej sprawdź tablicę forwarding po stronie RE:

**Urządzenie: vSRX-R1 — operational**

```junos
show route forwarding-table destination 198.51.100.0/24
show interfaces terse | match ge-0/0/2
show arp no-resolve
```

Wynik FIB zawiera m.in. **Destination**, typ next-hopu i **Netif** (interfejs wyjściowy). Szczegóły są zależne od platformy; nie interpretuj „Index” jako numeru portu. `show arp` w IPv4 pomaga ustalić, czy rozpoznano MAC hosta/next-hopu.

**Urządzenie: EX/QFX/MX z obsługą polecenia PFE — tryb operational**

```junos
show pfe route
show pfe route summary
```

Na vSRX nie traktuj `show pfe route` jako gwarantowanej komendy. Używaj `?` do sprawdzenia dostępności, a do sprawdzenia rzeczywistego ruchu również narzędzi `security flow`.

### 2.3.3. RE i zarządzanie konfiguracją

CLI Junos ma tryb operacyjny `>` i konfiguracyjny `#`. Wprowadzane zmiany są przygotowywane jako **candidate configuration**, a dopiero `commit` aktywuje je w systemie. Komponent `mgd` zarządza wprowadzaniem i zatwierdzaniem konfiguracji, po czym inne procesy otrzymują odpowiednie zmiany. Szczegółowy cykl `candidate → commit → active` rozwiniemy w rozdziale 3.

**Urządzenie: vSRX-R1 — odczyt konfiguracji bez jej zmieniania**

```junos
show configuration system
show configuration protocols
show configuration interfaces | display set
```

**Urządzenie: vSRX-R1 — przykład bezpiecznej zmiany w LAB**

```junos
configure
set system host-name vSRX-R1
show | compare
commit check
commit confirmed 5
```

`commit check` bada poprawność syntaktyczno-konfiguracyjną, ale **nie gwarantuje poprawnej łączności ani routingu**. `commit confirmed 5` aktywuje zmiany z mechanizmem automatycznego wycofania po 5 minutach, jeśli ich nie potwierdzisz. Po weryfikacji wpisz `commit` w trybie konfiguracyjnym, aby potwierdzić zmianę, albo pozwól jej się wycofać. W realnej sieci upewnij się wcześniej, czy rollback jest bezpieczny dla reszty operatorów i zmian współbieżnych.

**Urządzenie: vSRX-R1 — po sprawdzeniu zmiany, tryb configuration**

```junos
commit
exit
```

### 2.3.4. CPU Routing Engine — dlaczego bywa wysoki?

Przyczyny wysokiego wykorzystania CPU RE mogą obejmować intensywne obliczenia protokołów, duże zmiany RIB, burzę wyjątków kierowanych do RE, proces zarządzania, logowanie albo problem z usługą. **Wysoki CPU RE nie jest automatycznym dowodem przeciążenia data plane.** Z kolei niski CPU RE nie potwierdza, że porty lub forwarding pracują poprawnie.

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show chassis routing-engine
show system processes extensive
show log messages | match "rpd|mgd|chassisd|dcd"
show system uptime
```

**Interpretacja:** sprawdź trend, obciążony proces, logi, zmianę konfiguracji i to, czy występują jednocześnie problemy z sąsiedztwami. Nie restartuj `rpd` ani urządzenia na ślepo — restart protokołów może zerwać sąsiedztwa i wycofać trasy.

### 2.3.5. Co z redundantnymi RE?

Wybrane urządzenia mają dwa RE (primary/backup), co umożliwia mechanizmy wysokiej dostępności. **Nie należy jednak utożsamiać zapasowego RE z zapasowym PFE**, ani zakładać, że samo istnienie RE1 zapewnia bezprzerwową pracę każdego protokołu. GRES, NSR i podobne mechanizmy mają własne wymagania i stanowią oddzielną część podręcznika (rozdział 17).

---

## 2.4. Packet Forwarding Engine (PFE) — rola i funkcje [A]

### 2.4.1. Co robi PFE?

PFE jest wykonawczą częścią architektury sieciowej. Zależnie od platformy realizuje m.in.:

- odbiór ramki na porcie wejściowym;
- rozpoznanie kontekstu L2/L3, VLAN i interfejsu logicznego;
- wyszukanie odpowiedniego next-hopu w tablicy forwarding;
- sprawdzenie i zastosowanie wybranych filtrów, klasyfikacji i polityk forwarding;
- obsługę kolejki, buforowania i CoS/QoS;
- modyfikację nagłówków konieczną przy przekazywaniu;
- wysłanie ramki przez wskazany port wyjściowy;
- kierowanie pakietów wyjątkowych do odpowiednich procesów/control plane, jeśli to konieczne.

Na sprzętowych routerach i switchach wiele operacji odbywa się w ASIC-ach. W vSRX odpowiednik przetwarzania ruchu odbywa się programowo w wirtualnym datapath z warstwą bezpieczeństwa.

### 2.4.2. Różnica między forwarding a analizą bezpieczeństwa na vSRX

Klasyczny router może na bieżąco forwardować pakiety na podstawie FIB. **vSRX z reguły pracuje w trybie flow-based**, co dodaje decyzję o sesji i security policy:

1. Pierwszy pakiet nowego strumienia (*first path*): klasyfikacja i wyszukanie trasy, przypisanie stref, kontrola polityk, ewentualny NAT/usługi, utworzenie sesji.
2. Następne pakiety pasujące do sesji (*fast path*): korzystają z istniejącego kontekstu sesji, bez pełnego powtarzania wszystkich decyzji pierwszego pakietu.

**To osobny mechanizm od podziału control plane / forwarding plane.** Statefulness SRX nie oznacza, że każdy nowy strumień musi być analizowany przez proces `rpd` na RE.

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show security flow status
show security flow session summary
show security flow session
show security flow statistics
```

Przy polu `Inet forwarding mode: flow based` spodziewaj się sesji i działania polityk strefowych. Gdy tryb jest inny, zasady diagnostyki trzeba dostosować do konkretnej wersji i konfiguracji — **nie przełączaj trybu produkcyjnego tylko po to, aby wymusić działanie laboratorium**.

### 2.4.3. Skąd PFE wie, gdzie wysłać pakiet?

Najpierw odpowiednie protokoły i RE wyznaczają aktywne ścieżki. Następnie wynik jest instalowany w master FIB i synchronizowany z PFE. Przy pakiecie tranzytowym to PFE korzysta z tej tablicy i odpowiedniego wpisu next-hop. W typowym routingu IPv4 obowiązuje dopasowanie **najdłuższego prefiksu** (*longest prefix match*).

Przykładowa tablica:

| Prefix | Next hop |
|---|---|
| `0.0.0.0/0` | `10.0.12.254` |
| `198.51.100.0/24` | `10.0.12.2` |
| `198.51.100.128/25` | `10.0.12.3` |

Pakiet do `198.51.100.140` dopasuje `/25`, niezależnie od tego, że istnieją także `/24` i `/0`. Kolejność zasad jest ważna: **najpierw wybór najlepiej dopasowanego prefiksu do adresu docelowego**, a wewnątrz danego prefiksu routing już wcześniej wybrał odpowiedni zestaw aktywnych next-hopów.

### 2.4.4. Czy `show route forwarding-table` na pewno pokazuje PFE?

**Nie.** Oficjalna referencja Junipera określa ją jako tablicę forwarding po stronie RE. To szczegół, który często myli początkujących. Tablica RE jest kopiowana do PFE, a komendą mogącą pokazać tablicę po stronie PFE jest `show pfe route` (o ile platforma to wspiera). Przykład diagnostyczny:

```text
Route in RIB? -------- NO --> Szukaj w routingu/protokole/policy
       |
      YES
       v
Entry in RE FIB? ---- NO --> Selekcja trasy / KRT / kernel
       |
      YES
       v
Entry in PFE? ------- NO --> Synchronizacja/instalacja PFE (jeśli można sprawdzić)
       |
      YES
       v
Port/link, ARP/ND, filters, security flow, MTU, next-hop
```

Nie zakładaj, że komenda PFE będzie dostępna na każdym vSRX. Na rzeczywistych EX/QFX/MX dostęp i format zależą od rodziny oraz wydania.

### 2.4.5. Symptomy awarii PFE / data plane

- CLI/SSH i routing adjacency mogą nadal odpowiadać, ale ruch użytkowników nie przechodzi.
- `show route` pokazuje prawidłowy prefiks, a forwarding nie działa.
- Jeden FPC/PIC lub określona grupa portów nie przekazuje ruchu.
- Pojawiają się dropy, błędy interfejsów, przeciążone kolejki albo alarm FPC.

**Urządzenie: fizyczny EX/QFX/MX — tryb operational**

```junos
show chassis fpc
show chassis fpc pic-status
show interfaces terse
show interfaces extensive
show chassis alarms
show log messages | match "fpc|pfe|chassisd"
```

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show interfaces terse
show interfaces ge-0/0/1 extensive
show route forwarding-table family inet
show security flow status
show security flow session summary
show security flow statistics
```

**Pułapka:** na vSRX nie oczekuj identycznych liczników ASIC co na QFX/EX. Brak konkretnego licznika nie jest dowodem, że nie ma strat.

---

## 2.5. Przetwarzanie ruchu tranzytowego — transit traffic [A]

### 2.5.1. Definicja

**Transit traffic** to ruch, którego urządzenie **nie jest ostatecznym odbiorcą**, tylko przekazuje go dalej. Dla routingu L3 pakiet ma docelowy adres IP hosta znajdującego się poza samym routerem. Router odbiera ramkę L2 zaadresowaną do jego MAC na interfejsie wejściowym (gdy stanowi next-hop), ale docelowy adres **IP** pakietu dotyczy innego hosta.

**Przykład:** H1 `192.0.2.10` wysyła ICMP Echo Request do H2 `198.51.100.10`, a vSRX-R1 ma interfejsy `192.0.2.1/24` i `198.51.100.1/24`. Dla vSRX ten pakiet jest **transit** — choć protokół pakietu to ICMP.

### 2.5.2. Przebieg pakietu tranzytowego

1. **Ingress:** pakiet przychodzi na interfejs fizyczny/logiczny.
2. **Kontrola L2:** urządzenie identyfikuje ramkę oraz jej VLAN i kontekst wejściowy.
3. **Decyzja L3:** docelowy IP nie należy do routera; urządzenie szuka drogi w forwarding table.
4. **Wybór next-hop:** stosowane są tablica przekazywania oraz właściwe reguły LPM/ECMP.
5. **Polityki i akcje:** zależnie od platformy mogą działać ACL/firewall filters, CoS, a na SRX strefy, security policies i sesje.
6. **Rewrite:** router zmniejsza IPv4 TTL (albo IPv6 Hop Limit), aktualizuje konieczne informacje w nagłówku i przygotowuje nową ramkę L2 dla sąsiada/hosta.
7. **Egress:** pakiet wychodzi właściwym interfejsem, jeśli MTU, ARP/ND, link i kolejki na to pozwalają.

**Ważne:** pakiet IP przechodzi przez router, ale typowo zmieniają się źródłowy i docelowy MAC **na każdym hopie**; adresy źródłowy i docelowy IP pozostają takie same, o ile nie wykonano NAT lub innej transformacji.

### 2.5.3. Przykład z dokładnymi nagłówkami

```text
H1 (192.0.2.10) -- ge-0/0/1 [vSRX-R1] ge-0/0/2 -- H2 (198.51.100.10)
        192.0.2.1/24                    198.51.100.1/24

Na wejściu (H1 -> R1):
  L2 DST = MAC R1-ge-0/0/1
  IP SRC = 192.0.2.10
  IP DST = 198.51.100.10

Na wyjściu (R1 -> H2):
  L2 SRC = MAC R1-ge-0/0/2
  L2 DST = MAC H2
  IP SRC = 192.0.2.10
  IP DST = 198.51.100.10
```

Jeśli H1 zamiast H2 pingowałby `192.0.2.1`, pakiet byłby skierowany do **lokalnego IP routera** i należałby do ruchu kierowanego do RE/host stack (w rozumieniu podstawowej klasyfikacji exception/host-bound), a nie L3 transit.

### 2.5.4. Transit na vSRX jest również ruchem security flow

W trybie flow-based sam poprawny routing **nie wystarcza**. vSRX zwykle potrzebuje pozwalającej polityki pomiędzy odpowiednimi strefami bezpieczeństwa. Typowe problemy:

- na wejściu interfejs znajduje się w innej strefie niż zakładałeś;
- brak `security policy` z właściwej strefy do docelowej;
- NAT niepotrzebnie zmienia adresację laboratorium;
- sesja nie powstaje z powodu polityki;
- stara sesja zachowuje wcześniejszy stan po zmianie konfiguracji (zależnie od zmiany i platformy).

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show security zones
show security policies
show security flow session source-prefix 192.0.2.10/32
show security flow statistics
show route 198.51.100.10
```

### 2.5.5. Co mówi test `ping`?

| Test | Co naprawdę sprawdza | Czego nie dowodzi |
|---|---|---|
| H1 → IP interfejsu R1 | ARP/L2, dostarczenie do host stack, polityka host-inbound | Że R1 forwarduje między sieciami |
| R1 → H2 (`ping` z CLI) | Ruch **originated** z routera i powrót ICMP | Że polityka transit H1→H2 pozwala na sesję |
| H1 → H2 | Prawdziwy transit przez R1 i powrót | Nie wskazuje automatycznie miejsca błędu bez dalszej diagnostyki |
| SSH → R1 | Dostęp management/exception | Poprawności routingu host-to-host |

Wnioski z testów trzeba łączyć z route, interface, zone, session i policy. **„Router pingowany” i „router routuje” to odrębne hipotezy.**

---

## 2.6. Exception traffic — ruch wyjątkowy i relacja do control plane [A]

### 2.6.1. Co oznacza exception traffic?

W uproszczeniu JNCIA **exception traffic** to pakiety, które nie mogą być potraktowane tak jak zwykłe pakiety tranzytowe i wymagają obsługi przez control plane lub specjalną ścieżkę przetwarzania. Obejmuje przede wszystkim pakiety **adresowane do samego urządzenia** (*host-bound*) oraz wybrane zdarzenia wymagające reakcji systemu. Ruch **generowany lokalnie** przez router (np. ping z CLI) jest osobnym kierunkiem ruchu control/management, ale dla celów podstawowego rozróżnienia nie jest transit.

**Kluczowa zasada:** o rozróżnieniu nie decyduje nazwa protokołu (`ICMP`, `TCP`, `OSPF`), lecz to, **czy pakiet ma zostać przez urządzenie przekazany dalej, czy obsłużony lokalnie / specjalnie**.

### 2.6.2. Przykłady

| Pakiet / zdarzenie | Typowa klasyfikacja | Dlaczego |
|---|---|---|
| SSH do IP routera (`TCP/22`) | Host-bound / exception | Terminacja usługi na routerze |
| BGP do lokalnego adresu peer-a (`TCP/179`) | Control/exception | Obsługiwane przez stos TCP i proces routingu |
| OSPF Hello (`IP protocol 89`) | Control/exception | Protokół budujący sąsiedztwo |
| ICMP Echo do adresu interfejsu R1 | Host-bound / exception | Router jest odbiorcą pakietu |
| ICMP Echo z H1 przez R1 do H2 | **Transit** | Router jest jedynie pośrednikiem |
| ICMP Echo wygenerowany `ping` na R1 | Locally originated | Router sam tworzy pakiet |
| Pakiet, którego TTL wygasł na routerze | Specjalna obsługa / wyjątek | Normalny forwarding nie może być kontynuowany; możliwy ICMP Time Exceeded |
| ARP/ND dotyczący interfejsu R1 | Local control / neighbor discovery | Utrzymanie sąsiedztwa L2/L3; realizacja zależna od platformy |

**Uwaga platformowa:** część wyjątków może być obsłużona albo odfiltrowana już w PFE/line-card CPU i wcale nie musi dotrzeć do głównego RE. „Exception” nie oznacza gwarancji, że każdy taki pakiet zobaczysz w `monitor traffic`.

### 2.6.3. Co to jest packet punt?

*Punt* to skierowanie pakietu ze zwykłej ścieżki forwarding do ścieżki wyjątku/control plane. Może być konieczne, gdy pakiet jest adresowany do routera, wymaga obsługi protokołu albo wystąpił stan, którego szybka ścieżka nie obsługuje w zwykły sposób.

**Nie zakładaj automatycznie:** „pakiet puntowany do RE” = „każdy pakiet pakowany do CPU przy nowym połączeniu”. Na vSRX pierwszy pakiet nowej sesji może być obsługiwany przez software datapath/security flow — to nadal nie oznacza konwersacji z `rpd`.

### 2.6.4. Jak oglądać ruch do RE?

**Urządzenie: vSRX-R1 — tryb operational, tylko w kontrolowanym LAB**

```junos
monitor traffic interface ge-0/0/1 no-resolve matching "icmp" count 10
```

Komenda pokazuje pakiety dostępne dla narzędzia przechwytującego po stronie RE/host stack, a **nie kompletny tranzyt w PFE**. Jeśli host H1 pinguję IP routera `192.0.2.1`, w typowym labie możesz zobaczyć Echo Request / Reply. Jeśli H1 pinguję H2 przez router, brak ramek tranzytowych w tym widoku jest **oczekiwany**, nie oznacza braku przepływu.

Na części platform/wersji i przy szczególnych ścieżkach przetwarzania zachowanie capture może być inne. Oficjalny opis `monitor traffic` podkreśla, że polecenie dotyczy pakietów odbieranych i wysyłanych przez RE oraz może wpływać na wydajność. Zawsze używaj `count`, `no-resolve` i kończ capture po wykonaniu testu.

### 2.6.5. Ochrona control plane

RE jest zasobem krytycznym. Zalew pakietów host-bound (np. niechciane próby połączeń, kontrolne pakiety L3) może wpływać na routing i management. Dlatego platformy Junos mają mechanizmy ochrony, w tym filtry na odpowiednim interfejsie control plane (często `lo0` na routerach/switchach), policery/DDoS protection na wspieranych platformach i — na SRX — reguły `host-inbound-traffic`.

**Nie są to synonimy:**

- **Firewall filter na `lo0`**: często używany do ochrony ruchu do RE w routerach/switchach; musi uwzględniać protokoły routingu i management.
- **Control-plane DDoS policers**: limitowanie określonych kategorii pakietów puntowanych, zależne od platformy.
- **`host-inbound-traffic` na SRX**: kontroluje, jakie usługi/protokoły mogą terminować na SRX przez dany interfejs/strefę. To nie zastępuje polityki transit.

**Urządzenie: vSRX-R1 — kontrola host-inbound w trybie operational**

```junos
show configuration security zones | display set
show security zones
show security flow status
```

**Urządzenie: vSRX-R1 — przykład dopuszczenia ping do interfejsu LAB (configuration)**

```junos
configure
set security zones security-zone LAN-A interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
commit check
commit confirmed 5
```

**Uwaga:** instrukcja dotyczy wyłącznie osobnego laboratorium. Na produkcyjnym SRX zmiana stref/host-inbound może odsłonić management albo zmienić dostępność usług. Po przetestowaniu w LAB potwierdź `commit`; pełny lab z dwoma hostami znajduje się poniżej.

### 2.6.6. Dlaczego brak odpowiedzi na ping routera nie dowodzi awarii routingu?

Scenariusz: hosty H1 i H2 wymieniają pakiety przez SRX, ale H1 nie otrzymuje odpowiedzi od `192.0.2.1`.

Możliwe wyjaśnienie: `host-inbound-traffic system-services ping` jest zablokowane. Wówczas ruch do lokalnego IP SRX nie jest obsługiwany, mimo że **policy transit** dla H1→H2 może pozwalać na forwardowanie. Odwrotnie: ping do routera działa, ale brak `security policies from-zone LAN-A to-zone LAN-B` może zatrzymać ruch użytkowników.

---

## 2.LAB. Ćwiczenie kompleksowe: control vs transit vs exception na jednym vSRX [EXT]

### 2.LAB.1. Cel i topologia

Celem jest **na własne oczy** zobaczyć różnicę między ruchem do RE a pakietami tranzytowymi z obsługą sesji SRX.

```text
       VPCS-H1                  vSRX-R1                    VPCS-H2
   192.0.2.10/24         ge-0/0/1     ge-0/0/2       198.51.100.10/24
   GW 192.0.2.1  -------- 192.0.2.1 | 198.51.100.1 -------- GW 198.51.100.1
                              LAN-A  |  LAN-B
```

**Warunki:** odizolowany lab na vSRX 3.0, dostęp do konsoli, dwa interfejsy danych. Nazwy portów musisz **najpierw sprawdzić** (`show interfaces terse`), ponieważ mapowanie NIC w EVE-NG zależy od obrazu. Obie podsieci to adresy dokumentacyjne RFC 5737, nie stosuj ich do routingu w rzeczywistej firmowej sieci. Jeśli na interfejsach są istniejące przypisania/strefy, dostosuj przykład — nie usuwaj ich automatycznie.

### 2.LAB.2. Krok A — rozpoznanie systemu (bez zmian)

**Urządzenie: vSRX-R1**

```junos
show version
show interfaces terse
show security zones
show security flow status
show route table inet.0
```

Sprawdź, że `ge-0/0/1` i `ge-0/0/2` są portami, z których skorzystasz, oraz że nie są już przypisane do kolidującej konfiguracji.

### 2.LAB.3. Krok B — konfiguracja interfejsów, stref i policies

**Urządzenie: vSRX-R1 — tryb configuration**

```junos
configure
set system host-name vSRX-R1
set interfaces ge-0/0/1 unit 0 family inet address 192.0.2.1/24
set interfaces ge-0/0/2 unit 0 family inet address 198.51.100.1/24
set security zones security-zone LAN-A interfaces ge-0/0/1.0
set security zones security-zone LAN-B interfaces ge-0/0/2.0
set security zones security-zone LAN-A interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
set security zones security-zone LAN-B interfaces ge-0/0/2.0 host-inbound-traffic system-services ping
set security policies from-zone LAN-A to-zone LAN-B policy ALLOW-LAB match source-address any
set security policies from-zone LAN-A to-zone LAN-B policy ALLOW-LAB match destination-address any
set security policies from-zone LAN-A to-zone LAN-B policy ALLOW-LAB match application any
set security policies from-zone LAN-A to-zone LAN-B policy ALLOW-LAB then permit
set security policies from-zone LAN-B to-zone LAN-A policy ALLOW-LAB match source-address any
set security policies from-zone LAN-B to-zone LAN-A policy ALLOW-LAB match destination-address any
set security policies from-zone LAN-B to-zone LAN-A policy ALLOW-LAB match application any
set security policies from-zone LAN-B to-zone LAN-A policy ALLOW-LAB then permit
show | compare
commit check
commit confirmed 5
```

**Dlaczego dwa zestawy polityk?** Stanowa sesja dla H1→H2 obsługuje ruch powrotny, ale osobna polityka B→A umożliwi **nowe połączenie inicjowane z H2**. W labie świadomie zezwalamy na `any` — w produkcji stosuj ograniczone adresy/usługi.

**Urządzenie: vSRX-R1 — potwierdzenie po weryfikacji (configuration)**

```junos
commit
exit
```

### 2.LAB.4. Krok C — ustaw adresację hostów

**Urządzenie: VPCS-H1 — konsola VPCS**

```text
ip 192.0.2.10 255.255.255.0 192.0.2.1
show ip
```

**Urządzenie: VPCS-H2 — konsola VPCS**

```text
ip 198.51.100.10 255.255.255.0 198.51.100.1
show ip
```

### 2.LAB.5. Krok D — trzy testy i trzy interpretacje

**Test 1: ping do urządzenia (host-bound/exception).**

**Urządzenie: VPCS-H1**

```text
ping 192.0.2.1
```

**Urządzenie: vSRX-R1 — obserwacja ICMP skierowanego do R1**

```junos
monitor traffic interface ge-0/0/1 no-resolve matching "icmp" count 10
```

Jeśli pakiety nie są widoczne, rozważ ograniczenia capture danej platformy, a dopiero potem problem protokołu. Jeśli ping się nie udaje, sprawdź strefę, `host-inbound-traffic`, stan interfejsu i ARP.

**Test 2: ping router → host (locally originated).**

**Urządzenie: vSRX-R1**

```junos
ping 198.51.100.10 source 198.51.100.1 count 5
```

To sprawdza lokalnie generowany ruch z R1, ale **nadal nie udowadnia**, że H1↔H2 ma pozwolenie na transit.

**Test 3: ping host → host (transit).**

**Urządzenie: VPCS-H1**

```text
ping 198.51.100.10
```

**Urządzenie: vSRX-R1**

```junos
show route 198.51.100.10
show route forwarding-table destination 198.51.100.0/24
show security flow session source-prefix 192.0.2.10/32
show security flow statistics
show interfaces ge-0/0/1 extensive
show interfaces ge-0/0/2 extensive
```

Oczekujesz trasy connected do `198.51.100.0/24`, sesji flow (jeśli ping jest w toku/nie wygasła) i ruchu na interfejsach. Capture po stronie RE **nie musi** pokazywać ICMP H1→H2, mimo że sesja i ruch występują.

### 2.LAB.5a. Jak interpretować sesję flow — rzeczywisty dowód tranzytu

Na potrzeby ćwiczenia możesz otrzymać wynik podobny do poniższego (**przykład poglądowy; liczby, nazwy FPC i układ pól różnią się**):

**Urządzenie: vSRX-R1 — przykładowy wynik operational**

```text
user@vSRX-R1> show security flow session source-prefix 192.0.2.10/32
Session ID: 12345, Policy name: ALLOW-LAB/4, Timeout: 50, Valid
  In: 192.0.2.10/3 --> 198.51.100.10/0;icmp,
      If: ge-0/0/1.0, Pkts: 5, Bytes: 420
  Out: 198.51.100.10/0 --> 192.0.2.10/3;icmp,
      If: ge-0/0/2.0, Pkts: 5, Bytes: 420
```

Sprawdź: **Policy name** (która reguła zezwoliła), **In** (kierunek inicjujący), **Out** (kierunek powrotny), **If** (interfejsy), **Pkts/Bytes** (czy pakiety przyrastają) oraz **Timeout/Valid**. Przy braku sesji nie wyciągaj automatycznie wniosku „policy deny”: pakiet mógł nie dotrzeć, mieć błędną adresację lub zostać zatrzymany wcześniej w ścieżce.

**Kontrolny wniosek:** `show route` dowodzi znajomości trasy, a `show security flow session` pokazuje, czy SRX utworzył stanowy kontekst konkretnego ruchu. Te narzędzia odpowiadają na inne pytania.

### 2.LAB.6. Krok E — kontrolowany test problemu z host-inbound

Nie zmieniaj konfiguracji security policy transit. Usuń tylko możliwość pingowania **samego SRX** na interfejsie H1.

**Urządzenie: vSRX-R1 — tryb configuration**

```junos
configure
delete security zones security-zone LAN-A interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
commit confirmed 5
```

**Urządzenie: VPCS-H1**

```text
ping 192.0.2.1
ping 198.51.100.10
```

**Oczekiwane zachowanie:** ping do `192.0.2.1` może przestać odpowiadać, ale ping H1→H2 powinien nadal działać, ponieważ jego forwarding kontroluje odrębna polityka transit. Jeśli zachowanie jest inne, wyjaśnij je przez rzeczywiste zones, sessions, route i konfigurację — nie „dopasowuj” obserwacji do teorii.

**Urządzenie: vSRX-R1 — przywrócenie testowanego zezwolenia (configuration)**

```junos
set security zones security-zone LAN-A interfaces ge-0/0/1.0 host-inbound-traffic system-services ping
commit
exit
```

### 2.LAB.7. Krok F — kontrolowany test blokady transit

Zostaw ping na interfejsie R1 aktywny. Tym razem wyłącz **wyłącznie** politykę przekazywania pakietów z LAN-A do LAN-B.

**Urządzenie: vSRX-R1 — tryb configuration**

```junos
configure
deactivate security policies from-zone LAN-A to-zone LAN-B policy ALLOW-LAB
commit confirmed 5
```

**Urządzenie: VPCS-H1**

```text
ping 192.0.2.1
ping 198.51.100.10
```

Możesz zobaczyć poprawną odpowiedź z IP lokalnego R1, podczas gdy nowy ruch H1→H2 nie przejdzie. **Uwaga na istniejące sesje:** rozpoczęta przed zmianą sesja może przez pewien czas działać. Powtórz test po wygaśnięciu sesji lub, tylko w odizolowanym labie, rozważ wyczyszczenie konkretnej sesji zgodnie z dokumentacją danego wydania. Nie czyść wszystkich sesji na systemie produkcyjnym.

**Urządzenie: vSRX-R1 — przywrócenie polityki (configuration)**

```junos
activate security policies from-zone LAN-A to-zone LAN-B policy ALLOW-LAB
commit
exit
```

### 2.LAB.8. Kryteria zaliczenia laboratorium

Po wykonaniu zadania potrafisz w 1–2 minuty odpowiedzieć:

- Co w tym laboratorium robi RE, a co przetwarza software datapath vSRX?
- Który ping jest host-bound, który originated, a który transit?
- Dlaczego `show route` i `show security flow session` to dwa niezależne źródła dowodów?
- Dlaczego brak ICMP w `monitor traffic` nie jest dowodem na brak transit?
- Dlaczego zablokowanie pingu do SRX nie powinno automatycznie blokować H1→H2?

---

## 2.TS. Troubleshooting — typowe awarie, symptomy i metoda diagnozy [EXT]

### 2.TS.1. Szybka macierz diagnostyczna

| Objaw | Podejrzany obszar | Co sprawdzasz | Możliwe rozwiązanie |
|---|---|---|---|
| Brak sąsiadów OSPF/BGP, routing statyczny działa | RE / protokół / host-inbound | `show route`, `show system processes`, protokół, security zone | Sprawdź sąsiedztwo, adresację, protokoły host-inbound; nie restartuj od razu procesu |
| Trasa obecna w `show route`, brak forwarding | FIB / PFE / interfejs / policy | `show route forwarding-table`, stan portu, flow | Zidentyfikuj etap, na którym ścieżka się urywa |
| SSH i ping do routera działają, H1→H2 nie działa | Transit / security flow | `show security policies`, `show security flow session` | Popraw politykę z właściwej strefy do właściwej strefy |
| H1→H2 działa, H1→IP SRX nie działa | Host-inbound/RE access | `show configuration security zones` | Dopuść konkretną usługę do SRX w LAB; w produkcji zgodnie z polityką bezpieczeństwa |
| RE CPU wysoki, interfejsy nie wykazują dużego ruchu | Control plane | `show chassis routing-engine`, `show system processes extensive`, logi | Ustal źródło CPU, protokół, wyjątki, proces; eskaluj problem |
| Część portów przestała działać, RE dostępny | PFE/FPC/PIC/link | `show chassis fpc`, `show interfaces extensive`, chassis alarms | Sprawdź awarię portu/karty i fizyczną ścieżkę, zgodnie z procedurą |
| `monitor traffic` nie pokazuje ICMP host-host | Normalny transit nie jest przechwytywany na RE | `show security flow session`, counters; ewentualnie mirroring | Nie używaj RE capture jako jedynego dowodu na obecność ruchu transit |
| Ping router→host działa, host→host nie | Testy badają inne ścieżki | Routes, host gateway, security policy, sesje | Sprawdź transit end-to-end, nie tylko ruch originated |

### 2.TS.2. Runbook: „Routing wygląda dobrze, ale aplikacja nie działa”

**Etap 1: zidentyfikuj, co właściwie testujesz.** Czy źródło i cel to dwa hosty po różnych stronach urządzenia? Czy pingujesz IP samego urządzenia? Czy test jest realizowany z CLI routera? To różne ścieżki.

**Urządzenie: vSRX-R1**

```junos
show interfaces terse
show route table inet.0
show route 198.51.100.10
show route forwarding-table destination 198.51.100.0/24
```

**Etap 2: sprawdź security flow.**

**Urządzenie: vSRX-R1**

```junos
show security zones
show security policies
show security flow status
show security flow session source-prefix 192.0.2.10/32
show security flow statistics
```

**Etap 3: sprawdź fizyczną ścieżkę i alarmy.**

**Urządzenie: vSRX-R1**

```junos
show interfaces ge-0/0/1 extensive
show interfaces ge-0/0/2 extensive
show system alarms
show chassis alarms
show log messages | last 30
```

**Etap 4:** porównaj z działającą ścieżką. Jeżeli problem pojawił się po zmianie konfiguracji, sprawdź zmiany w `show configuration` / `show | compare` i ostatnie logi. Dopiero gdy masz dowód, zmieniaj parametry; nie usuwaj security policies na chybił trafił.

### 2.TS.3. Trzy błędne „diagnozy”, których należy unikać

1. **„`show route` ma trasę, więc PFE na pewno poprawnie forwarduje.”** — trasa może nie być zainstalowana lub transmisja może być zablokowana dalej.
2. **„CPU RE = 90%, więc ruch dataplane musi się dusić.”** — to możliwe w niektórych awariach, ale wymaga dowodu: sesje, dropy, porty, linie błędów, czas występowania.
3. **„`monitor traffic` nie pokazuje pakietu, więc pakiet nie dotarł do interfejsu.”** — capture RE nie jest uniwersalnym snifferem całego PFE.

### 2.TS.3a. Rozwinięty przypadek awarii — BGP nie zestawia sesji na SRX

**Sytuacja:** na R1 istnieje route do IP sąsiada BGP, TCP/179 jest osiągalny na ścieżce sieciowej, ale sesja BGP pozostaje w stanie nieustanowionym. Na firewallu SRX jest poprawna security policy dla ruchu użytkowników.

**Dlaczego to możliwe?** BGP do adresu routera jest **ruchem control/host-bound**, nie zwykłym transit. Dla SRX znaczenie ma m.in. zezwolenie na odpowiedni protokół w `host-inbound-traffic` strefy/interfejsu, na którym przychodzi ruch oraz konfiguracja protokołu. Policy między strefami nie jest bezpośrednim zamiennikiem host-inbound.

**Urządzenie: vSRX-R1 — tryb operational**

```junos
show bgp summary
show route <adres_IP_sasiada_BGP>
show configuration security zones | display set
show configuration protocols bgp | display set
show log messages | match "bgp|rpd"
```

W poleceniu `show route <adres_IP_sasiada_BGP>` zastąp tekst w nawiasach rzeczywistym adresem, **bez dosłownego wpisywania nawiasów**. Sprawdź trasę, port/zone, zgodność peer AS/adresu, uprawnienia host-inbound i logi. W tym rozdziale nie zmieniamy BGP — pełna diagnoza jest w rozdziale 11.

### 2.TS.3b. Rozwinięty przypadek awarii — traceroute gubi hop, ale transmisja działa

**Sytuacja:** ruch H1→H2 przechodzi, a pośrednie urządzenie nie odpowiada na część prób `traceroute`.

**Wyjaśnienie:** przy wyczerpaniu TTL/Hop Limit potrzebna jest specjalna obsługa i ewentualne wygenerowanie ICMP Time Exceeded. W zależności od platformy, filtrów i ochrony control plane urządzenie może nie generować odpowiedzi dla każdego probe, **mimo poprawnego przekazywania normalnego ruchu**.

**Wniosek praktyczny:** brak konkretnego „hopu” w traceroute nie jest dowodem uszkodzenia łącza. Porównaj end-to-end TCP/ICMP, routing, statystyki interfejsów oraz zmiany w czasie.

### 2.TS.4. Komendy, których nie uruchamiać bez potrzeby

- `request system reboot`, `request chassis ...` — mogą spowodować przerwę usługi.
- `restart routing` / `restart ...` — może zerwać protokoły i zmienić routing.
- `clear security flow session all` — usuwa **wszystkie** sesje i może zrywać połączenia.
- Długotrwałe `monitor traffic` / szerokie traceoptions — możliwy wpływ na wydajność lub ujawnienie danych.
- Globalna zmiana forwarding mode SRX — zależnie od wersji może wymagać restartu i zmienia funkcje bezpieczeństwa.

---

## 2.CLI. Ściąga poleceń do rozdziału [EXT]

| Cel | Operational CLI | Typ platformy |
|---|---|---|
| Wersja i typ systemu | `show version` | Większość Junos |
| RE CPU, pamięć, stan | `show chassis routing-engine` | Zależnie od modelu |
| Procesy | `show system processes extensive` | Większość Junos |
| Alarmy systemowe | `show system alarms` | Większość Junos |
| Alarmy chassis | `show chassis alarms` | Sprzęt/virtual zależnie od modelu |
| Karty/porty | `show chassis fpc` | Głównie sprzętowe platformy, zależnie od modelu |
| Interfejsy | `show interfaces terse` | Większość Junos |
| Liczniki portu | `show interfaces ge-0/0/1 extensive` | Po podstawieniu realnego portu |
| Trasy IPv4 | `show route table inet.0` | Router / SRX / L3 switch |
| Master FIB na RE | `show route forwarding-table` | Większość Junos |
| FIB PFE | `show pfe route` | Tylko gdzie komenda jest dostępna |
| Monitorowanie host-bound/RE | `monitor traffic interface ...` | Ostrożnie, w LAB |
| Tryb forwarding SRX | `show security flow status` | SRX/vSRX |
| Sesje transit | `show security flow session` | SRX/vSRX |
| Security policy | `show security policies` | SRX/vSRX |
| Host-inbound | `show configuration security zones` | SRX/vSRX |
| Logi | `show log messages` | Większość Junos |

**Uwaga do filtrowania poleceń:** `| match` jest filtrowaniem CLI Junos, a nie dokładnym odpowiednikiem IOS `| include` w każdym przypadku; formaty wyników i obsługiwane filtry są opisane w rozdziale 3.

---

## 2.TEST. Sprawdź swoją wiedzę — JNCIA z rozwinięciem JNCIP [EXT]

Odpowiedz bez zaglądania do rozdziału. Pytania są **autorskie**, a nie oryginalnymi pytaniami egzaminacyjnymi Junipera.

**1.** Który komponent jest głównym wykonawcą przekazywania pakietów tranzytowych w klasycznym modelu Junos?  
A. `mgd`  B. PFE  C. `rpd`  D. SNMP

**2.** Proces `rpd` przede wszystkim:  
A. wykonuje NAT dla każdej sesji  B. oblicza i utrzymuje informacje routingu  C. przełącza ramki w ASIC  D. obsługuje logowanie SSH

**3.** Masz trasę aktywną w `inet.0`. Czy jest to ostateczny dowód jej instalacji w PFE? Wyjaśnij.

**4.** Host H1 wysyła ICMP Echo Request do IP H2 przez vSRX. Czy jest to exception traffic tylko dlatego, że protokół to ICMP?

**5.** Czym różnią się `show route forwarding-table` i `show pfe route`?

**6.** Dlaczego działające SSH do routera nie dowodzi, że router przenosi ruch użytkowników?

**7.** Co może spowodować, że vSRX nie odpowiada na ping do lokalnego interfejsu, chociaż sieć host→host działa?

**8.** Co sprawdzisz na vSRX, jeśli routing wygląda dobrze, ale pakiety host→host nie przechodzą?

**9.** Jaka jest różnica pomiędzy RE i PFE na fizycznym QFX a wirtualnym vSRX?

**10.** Wymień po jednym przykładzie ruchu *transit*, *host-bound* i *locally originated*.

**11. (Scenariusz)** `show route` i `show route forwarding-table` pokazują dobry wpis; `show security flow session` nie pokazuje sesji dla H1→H2. Jakie trzy przyczyny sprawdzisz przed restartem systemu?

**12. (Scenariusz)** H1→H2 działa, ale `monitor traffic interface ge-0/0/1 matching "icmp"` nic nie pokazuje. Czy to awaria? Jak potwierdzisz przepływ?

### Klucz odpowiedzi

1. **B.** PFE (na vSRX logiczny datapath/flow engine).
2. **B.** `rpd` prowadzi protokoły routingu, wybór aktywnych tras i routing policy.
3. **Nie.** RIB ≠ FIB; sprawdź master forwarding table i jeśli wspierane FIB PFE, potem porty, next-hop, security flow.
4. **Nie.** ICMP H1→H2 przez router to transit. ICMP do IP routera jest host-bound.
5. Pierwsza komenda pokazuje **master FIB na RE**, druga — tablicę PFE na obsługiwanych platformach.
6. SSH trafia do management/control i może działać, mimo że PFE, port, policy lub strefa blokuje tranzyt.
7. Np. brak `host-inbound-traffic system-services ping`, filtr do RE lub inny problem host-bound.
8. Route/FIB, interfejsy, `security zones`, policies, status flow, sesje i liczniki.
9. Na QFX forwarding często realizują ASIC; w vSRX datapath i bezpieczeństwo realizuje software. Logiczne role RE/PFE nadal są pomocne.
10. Transit: H1→H2; host-bound: SSH do IP R1; locally originated: `ping` z CLI R1.
11. Strefy/położenie interfejsów; brak/dozwolenie policies; rzeczywisty ingress i adresacja źródłowa/trasowanie/next-hop (oraz status procesu flow).
12. Zwykle **nie**. Capture RE nie obejmuje całego transit. Sprawdź end-to-end ping, sesje `show security flow session`, counters i ewentualnie mirroring.

### Jak ocenić gotowość

- **12/12:** możesz przejść do rozdziału 3, zachowując lab jako punkt odniesienia.
- **9–11/12:** powtórz sekcje, które sprawiły problem; koniecznie zrób lab.
- **≤8/12:** przejdź ponownie 2.2–2.6 i rozrysuj RE ↔ FIB ↔ PFE własnymi słowami.

---

## Jednostronicowe podsumowanie — do zapamiętania przed egzaminem

1. **RE buduje i utrzymuje informacje sterujące; PFE przekazuje większość ruchu tranzytowego.**
2. **Control plane i forwarding plane mają inne zadania**, więc test ich działania wymaga innych komend.
3. **Management plane to rola usług administracyjnych**, często realizowanych przez RE, a nie koniecznie osobny hardware.
4. **`rpd`** to kluczowy proces protokołów i tras; **`mgd`** kontroluje konfigurację.
5. **RIB (`inet.0`) nie jest tym samym co FIB.** Na RE jest master forwarding table, a PFE może mieć zainstalowaną kopię.
6. **`show route forwarding-table` pokazuje perspektywę RE**, a `show pfe route` — PFE, o ile dostępne.
7. **Transit:** pakiet **przechodzi przez** urządzenie. **Host-bound/exception:** pakiet **jest kierowany do** urządzenia lub wymaga specjalnej obsługi. **Originated:** pakiet **powstaje na** urządzeniu.
8. **ICMP nie oznacza automatycznie exception.** Liczą się adres docelowy i ścieżka obsługi.
9. **`monitor traffic` nie jest snifferem całego PFE** — obserwuje przede wszystkim ruch widoczny po stronie RE.
10. **Na SRX/vSRX poprawny route to za mało**: sprawdź zones, security policies, flow mode i sesje.
11. **Ping do IP routera ≠ test routingu end-to-end.** Nie wyciągaj wniosków o transit z samego SSH/ping do RE.
12. **Nie przenoś 1:1 komend i zachowań MX/EX/QFX na vSRX**; zweryfikuj wersję, model i dostępność polecenia.

## Powiązanie z dalszym podręcznikiem

- **Rozdział 3:** CLI, candidate/active, commit/rollback i techniki bezpiecznych zmian.
- **Rozdział 5:** monitoring, `show`, `monitor`, logi, interfejsy, narzędzia sieciowe.
- **Rozdział 6:** RIB, FIB, route preference, longest prefix match, static routes.
- **Rozdział 8:** routing policies i stateless firewall filters.
- **Rozdział 11:** BGP, next-hop resolution i zaawansowane programowanie forwarding.
- **Rozdział 17:** HA — co faktycznie dzieje się z routingiem i forwarding przy awarii RE.

---

## Bibliografia i źródła (oficjalne)

**Źródło organizacji rozdziału:** `JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md`, punkty 2.1–2.6 (plik opracowany wcześniej w tym projekcie). Rozdział zachowuje jego zakres i numerację; dodatkowe sekcje są oznaczone `[EXT]`.

1. [JNCIA-Junos Certification Overview — official objectives](https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=14354) — domena *Junos OS Fundamentals*: software architecture, control/forwarding planes, RE/PFE, transit i exception traffic.
2. [Junos OS Architecture Overview](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-architecture.html) — podstawowe role RE/PFE.
3. [Junos OS Routing Engine Components and Processes](https://www.juniper.net/documentation/us/en/software/junos/junos-overview/topics/concept/junos-software-components-and-processes.html) — kernel, `mgd`, `rpd`, procesy interfejsów, chassis.
4. [show route forwarding-table — Junos CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route-forwarding-table.html) — master forwarding table i relacja do PFE.
5. [show pfe route — Junos CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-pfe-route.html) — bezpośrednia perspektywa PFE (gdy obsługiwana).
6. [monitor traffic — Junos CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/monitor-traffic.html) — obserwacja pakietów do/z RE, ograniczenia i ostrzeżenia wydajnościowe.
7. [Traffic Processing on SRX Series Firewalls](https://www.juniper.net/documentation/us/en/software/junos/flow-packet-processing/topics/topic-map/security-srx-devices-processing-overview.html) — first-path/fast-path, sesje i przepływ na SRX.
8. [show security flow session — Junos CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-security-flow-session.html) — filtry sesji i interpretacja.
9. [show security flow status — Junos CLI Reference](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-security-flow-status.html) — flow/packet forwarding modes.
10. [Top Differences Between Junos OS Evolved and Junos OS](https://www.juniper.net/documentation/us/en/software/junos/overview-evo/topics/concept/evo-top-differences.html) — FreeBSD vs. Linux/nodes.
11. [Control Plane DDoS Protection Overview](https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/concept/subscriber-management-ddos-protection.html) — ograniczanie exception traffic na wspieranych platformach.
12. [Junos OS Architecture — Control Plane and Data Plane (41-min training module)](https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=13926) — oficjalna lekcja HPE/Juniper powiązana z darmowym JNCIA Open Learning.

> **Nota techniczna:** dokumentacja Juniper bywa automatycznie przekierowywana do domen HPE Networking Docs. Linki zachowano jako oficjalne adresy producenta. Materiał objaśniający, diagramy, lab i pytania są opracowaniem autorskim, nie reprodukcją podręcznika lub banku pytań Juniper.
