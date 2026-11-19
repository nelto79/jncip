# LAB CH02 — JUNOS Architecture: 4 × vSRX + 2 × VPCS

**Status: PLANNED — specyfikacja; plik .unl nie został jeszcze wygenerowany ani uruchomiony.**

Powiązanie: [Rozdział 2 — Junos OS Fundamentals](../../docs/chapters/JUNOS_Rozdzial_02_Architektura_Junos_OS.md) oraz [Unified Blueprint](../../docs/JUNOS_JNCIA_JNCIS_JNCIP_unified_blueprint.md).

**Wspólne źródło prawdy dla całego kursu:** [diagram DIAMOND w Mermaid — 4 vSRX + 2 VPCS, adresacja IP/CIDR i loopbacki](../reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md) oraz [kanoniczna tabela adresacji](../reference-diamond-4vsrx-2vpcs/README.md). Ten rozdział jest scenariuszem lekcyjnym, a dokumenty referencyjne definiują stałą topologię.


## 1. Cel

Rozbuduj jedno-routerowe ćwiczenie z podręcznika do topologii, której można używać wielokrotnie. Naucz się w praktyce:
- control plane: RE, rpd, mgd, RIB, route preferences;
- data plane: forwarding, next-hop resolution, sesje SRX, ingress/egress;
- transit traffic vs host-bound/exception traffic;
- rozróżnienia: route present / route active / route installed / packet actually forwarded;
- diagnostyki: interfejs, ARP, route, forwarding table, session, security zones/policies;
- awarii jednej gałęzi i przełączenia na alternatywną ścieżkę;
- późniejszej migracji z tras statycznych na OSPF/BGP w kolejnych rozdziałach.

Nie jest celem rozdziału 2 nauczenie całego OSPF/BGP ani zaawansowanego switchingu.

## 2. Topologia logiczna

~~~text
                       R2-vSRX
                     /         \
                    /           \
PC-A --- R1-vSRX                  R4-vSRX --- PC-B
                    \           /
                     \         /
                       R3-vSRX

Ścieżka podstawowa: R1 -> R2 -> R4
Ścieżka zapasowa:   R1 -> R3 -> R4
~~~

Sprzęt: **4 × vSRX 3.0 (KVM/QCOW2) + 2 × VPCS**, 6 węzłów łącznie, 6 izolowanych segmentów L2 point-to-point / LAN. Dla vSRX zacznij od 2 vCPU i 4 GiB RAM na węzeł — łącznie ok. 16 GiB przydzielonego RAM + EVE/host, a ostateczne wymagania sprawdź względem pobranej wersji.

Uwaga: przy EVE uruchomionym jako VM sprawdź zasoby i nested virtualization. Jeśli host nie udźwignie jednocześnie 4 vSRX, można budować i testować wariant 2-vSRX; jednak docelowy plik musi zachowywać 4-vSRX.

## 3. Interfejsy i adresacja

**Mapowanie ge-0/0/x to propozycja.** Codex ma je potwierdzić komendą show interfaces terse na faktycznie uruchomionym obrazie, ponieważ vSRX/EVE może wystawić osobny fxp0 i inne mapowanie NIC.

| Segment | Węzeł A — interfejs / adres | Węzeł B — interfejs / adres | Podsieć |
|---|---|---|---|
| PC-A ↔ R1 | PC-A eth0: 192.0.2.10/24 | R1 ge-0/0/2: 192.0.2.1/24 | 192.0.2.0/24 |
| R1 ↔ R2 | R1 ge-0/0/0: 10.12.0.1/30 | R2 ge-0/0/0: 10.12.0.2/30 | 10.12.0.0/30 |
| R1 ↔ R3 | R1 ge-0/0/1: 10.13.0.1/30 | R3 ge-0/0/0: 10.13.0.2/30 | 10.13.0.0/30 |
| R2 ↔ R4 | R2 ge-0/0/1: 10.24.0.1/30 | R4 ge-0/0/0: 10.24.0.2/30 | 10.24.0.0/30 |
| R3 ↔ R4 | R3 ge-0/0/1: 10.34.0.1/30 | R4 ge-0/0/1: 10.34.0.2/30 | 10.34.0.0/30 |
| R4 ↔ PC-B | R4 ge-0/0/2: 198.51.100.1/24 | PC-B eth0: 198.51.100.10/24 | 198.51.100.0/24 |

PC-A default gateway: 192.0.2.1. PC-B default gateway: 198.51.100.1. Segmenty 192.0.2.0/24 i 198.51.100.0/24 to prefiksy dokumentacyjne RFC 5737 — **nie używać do Internetu**.

**Loopbacki — standard wspólny dla wszystkich labów:** R1 `lo0.0` **10.255.255.1/32**, R2 **10.255.255.2/32**, R3 **10.255.255.3/32**, R4 **10.255.255.4/32**. Ustaw takie same wartości `routing-options router-id`. W bazowym CH02 z trasami statycznymi nie zakładaj osiągalności loopbacków zdalnych: potrzebne są jawne trasy lub późniejsze OSPF/BGP. Szczegółowa tabela i komendy w [topologii kanonicznej](../reference-diamond-4vsrx-2vpcs/README.md).

## 4. Routing w wersji bazowej

Użyj tras statycznych, w pełni symetrycznych na poziomie sieci docelowych:

| Urządzenie | Sieć docelowa | Next-hop podstawowy | Next-hop zapasowy |
|---|---|---|---|
| R1 | 198.51.100.0/24 | 10.12.0.2 (via R2) | 10.13.0.2 (via R3) |
| R2 | 192.0.2.0/24 | 10.12.0.1 | — |
| R2 | 198.51.100.0/24 | 10.24.0.2 | — |
| R3 | 192.0.2.0/24 | 10.13.0.1 | — |
| R3 | 198.51.100.0/24 | 10.34.0.2 | — |
| R4 | 192.0.2.0/24 | 10.24.0.1 (via R2) | 10.34.0.1 (via R3) |

W R1/R4 ustaw preferencję primary/backup tak, by normalnie używać R2, a gdy **lokalny** primary link utraci stan operacyjny — R3. **Uwaga: lokalny wybór backupu nie oznacza automatycznej konwergencji całej sieci.** Jeśli padnie tylko R1–R2, R1 może przełączyć się na R3, ale R4 może nadal wysyłać odpowiedzi do R2 i powstanie blackhole lub problem ze stanem sesji SRX. W scenariuszu *pełnego failoveru* wyłącz oba łącza górnego korytarza (R1–R2 oraz R2–R4); w scenariuszu *pojedynczej awarii* oczekuj diagnostyki problemu, a nie gwarantowanego ping PASS. OSPF, RPM lub inna kontrola reachability to następne etapy szkolenia.

## 5. SRX security: kontrolowane umożliwienie ruchu

vSRX jest stateful firewallem — sam routing nie gwarantuje pingów między hostami. Dla baseline:
- wyraźnie przypisz każdy interfejs danych do security zone (np. LAB);
- dla demonstracji jawnie skonfiguruj politykę ruchu wewnątrz tej strefy pozwalającą ICMP (albo szerszą politykę tylko do izolowanego laba);
- dopuść host-inbound ICMP tylko na interfejsach, na których jest wymagany ping do samego vSRX;
- pokaż, czym różni się security policy dla **transit** od host-inbound-traffic dla **RE**;
- nie włączaj permanentnie flow trace ani nie generuj logów bez limitu;
- nie mieszaj packet-mode z flow-mode bez osobnego, uzasadnionego ćwiczenia.

Wartość oczekiwana: PC-A ↔ PC-B działa tylko przy prawidłowym routingu zwrotnym, ARP i security policy.

## 6. Zestaw obowiązkowych ćwiczeń

**LAB-02-01 — Junos inventory and process check:** show version, show chassis hardware, show chassis routing-engine, show system processes extensive, show interfaces terse. Wyjaśnij, które elementy klasycznej architektury RE/PFE mają sens na vSRX.

**LAB-02-02 — RIB vs forwarding:** show route inet.0, show route 198.51.100.0/24 exact, show route forwarding-table (na platformach wspierających), show arp. Nie utożsamiaj kopii forwarding table na RE z bezpośrednim odczytem sprzętowego PFE.

**LAB-02-03 — Transit traffic:** PC-A ping PC-B; traceroute; show security flow session, show security policies hit-count (jeśli obsługiwane). Zbierz ścieżkę PC-A → R1 → R2 → R4 → PC-B.

**LAB-02-04 — Host-bound/exception traffic:** PC-A ping 192.0.2.1, a następnie odebranie host-inbound ping (tylko w izolowanym labie); zweryfikuj, że routing tranzytowy może działać mimo braku pingu do interfejsu routera.

**LAB-02-05 — Transit blocked by security policy:** celowo zablokuj ICMP forwarding przy zachowaniu host-inbound ping; diagnozuj ruch na wejściu, trasę, session/security. Przywróć stan, nie zostawiaj zablokowanego baseline.

**LAB-02-06 — Preferred path failure:** (A) w celu potwierdzenia end-to-end failover wyłącz **R1–R2 i R2–R4** i sprawdź, czy obie strony używają R3; (B) wyłącz **tylko R1–R2** i zdiagnozuj możliwy blackhole niespójnej drogi zwrotnej. W obu przypadkach uwzględnij stateful sessions vSRX. Napraw łącza i odtwórz baseline. Nie przedstawiaj (B) jako gwarantowanego PASS.

**LAB-02-07 — Control vs data plane observation:** w bezpieczny sposób obserwuj rpd/mgd/CPU podczas ruchu, bez zabijania procesów albo zrywania sesji zarządzającej.

## 7. Komendy weryfikacyjne (operational mode)

**R1-vSRX, R2-vSRX, R3-vSRX, R4-vSRX — per node, operational mode**

~~~junos
show version
show chassis hardware
show chassis routing-engine
show system processes extensive
show interfaces terse
show route table inet.0
show route forwarding-table
show arp
show security flow session
show security zones
show security policies
show log messages | last 50
~~~

**R1-vSRX — operational mode**

~~~junos
show route 198.51.100.0/24 exact
show route forwarding-table destination 198.51.100.10
traceroute 198.51.100.10 source 192.0.2.1
~~~

**PC-A — VPCS console**

~~~text
ip 192.0.2.10/24 192.0.2.1
ping 198.51.100.10
trace 198.51.100.10
~~~

**PC-B — VPCS console**

~~~text
ip 198.51.100.10/24 198.51.100.1
ping 192.0.2.10
trace 192.0.2.10
~~~

Sprawdź dostępność każdej komendy w konkretnej wersji Junos/vSRX. Jeżeli command not supported, zanotuj to i zastąp komendą równoważną; nie traktuj tego jako awarii forwarding.

## 8. Kryteria akceptacji i artefakty

W katalogu tego labu Codex powinien utworzyć (docelowo):
- topology/CH02_JUNOS_ARCHITECTURE_4vSRX.unl — nowy EVE-NG lab, unikalny UUID, dokładnie 4 vSRX + 2 VPCS;
- configs/R1.set, R2.set, R3.set, R4.set, PC-A.txt, PC-B.txt — sprawdzone konfiguracje;
- VERIFY.md — kroki, expected output i rzeczywiste wyniki;
- TROUBLESHOOTING.md — fault→symptom→diagnosis→repair;
- adresację i mapowanie EVE-NG portów udokumentowane w tym README.

Plik .unl powinien parsować się jako XML i spełniać schemat EVE, ale to nie wystarcza: wymagana jest walidacja w EVE-NG po uruchomieniu.

**Nie deklaruj runtime PASS przed zebraniem realnych logów weryfikacyjnych.**

## 9. Stan i zależności

- Pliki źródłowe podręcznika i blueprint są już w repozytorium; nie wolno ich zastąpić przez lab.
- vSRX QCOW2 musi zostać pozyskany legalnie, samodzielnie przez właściciela i udostępniony EVE zgodnie z licencją.
- Dostęp SSH/API do instancji EVE, mapowanie portów oraz zasoby hosta są jeszcze do potwierdzenia.
- Zanim rozpoczniesz operacje modyfikujące EVE, przeczytaj [CODEX_TASK.md](CODEX_TASK.md).
