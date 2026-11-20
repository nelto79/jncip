# DIAMOND-6R — wariant zaawansowany: sześć routerów vSRX

**Status:** SPECIFICATION / PLANNED — schemat i adresacja, nie uruchomiony lab.  
**Powiązanie:** [Rozdział 8 — Routing Policy and Firewall Filters](../../docs/chapters/JUNOS_Rozdzial_08_Routing_Policy_and_Firewall_Filters.md).  
**Inspiracja:** dostarczony diagram sześciu urządzeń, R5 jako CE-A i R6 jako CE-B.

Ten wariant jest **rozszerzeniem** dotychczasowej referencji [DIAMOND 4vSRX + 2 VPCS](../reference-diamond-4vsrx-2vpcs/README.md). R5 i R6 fizycznie zastępują PC-A i PC-B. **Nie zmieniamy** żadnego z czterech routerów rdzenia, ich interfejsów, istniejących loopbacków ani czterech łączy rombu. Nie ma ukrytego połączenia R2–R3 ani R1–R4.

## Topologia

[Otwórz szczegółowy diagram Mermaid](DIAMOND_TOPOLOGY.md).

| Rola | Urządzenie | AS w wariancie eBGP | Loopback router ID |
|---|---|---:|---|
| CE-A | R5-vSRX | 65105 | 10.255.255.5/32 |
| EDGE-A | R1-vSRX | 65001 | 10.255.255.1/32 |
| CORE-1 | R2-vSRX | 65002 | 10.255.255.2/32 |
| CORE-2 | R3-vSRX | 65003 | 10.255.255.3/32 |
| EDGE-B | R4-vSRX | 65004 | 10.255.255.4/32 |
| CE-B | R6-vSRX | 65106 | 10.255.255.6/32 |

## Pełna mapa interfejsów

| Łącze | Koniec A | Koniec B | Podsieć |
|---|---|---|---|
| LAN-A / CE-A | R5 \`ge-0/0/0.0\` — **192.0.2.10/24** | R1 \`ge-0/0/2.0\` — **192.0.2.1/24** | 192.0.2.0/24 |
| Górny lewy | R1 \`ge-0/0/0.0\` — **10.12.0.1/30** | R2 \`ge-0/0/0.0\` — **10.12.0.2/30** | 10.12.0.0/30 |
| Dolny lewy | R1 \`ge-0/0/1.0\` — **10.13.0.1/30** | R3 \`ge-0/0/0.0\` — **10.13.0.2/30** | 10.13.0.0/30 |
| Górny prawy | R2 \`ge-0/0/1.0\` — **10.24.0.1/30** | R4 \`ge-0/0/0.0\` — **10.24.0.2/30** | 10.24.0.0/30 |
| Dolny prawy | R3 \`ge-0/0/1.0\` — **10.34.0.1/30** | R4 \`ge-0/0/1.0\` — **10.34.0.2/30** | 10.34.0.0/30 |
| LAN-B / CE-B | R4 \`ge-0/0/2.0\` — **198.51.100.1/24** | R6 \`ge-0/0/0.0\` — **198.51.100.10/24** | 198.51.100.0/24 |

**Ważne:** oznaczenia \`ge-0/0/x\` na R5/R6 są propozycją, a na R1–R4 zachowują dotychczasowy projekt. Każda mapa fizycznego NIC w QEMU/EVE-NG wymaga sprawdzenia \`show interfaces terse\` na działającym obrazie. Management/OOB pozostaje poza sześcioma segmentami danych. Nie łącz tych adresów dokumentacyjnych z Internetem.

## Dodatkowe prefiksy używane do polityk

Oba CE są **routerami**, a nie hostami VPCS. Połączenie CE-A→CE-B testujemy ruchem generowanym z R5/R6 (np. \`ping ... source ...\`). To **nie** zastępuje osobnego testu prawdziwych hostów tranzytowych. Można je dodać później, ale nie są siódmym ani ósmym routerem i nie należą do tej stałej mapy.

| Węzeł | Prefiks | Sposób utworzenia | Cel dydaktyczny |
|---|---|---|---|
| R5 | 172.16.50.0/24 | \`lo0.0 172.16.50.1/24\` | Business-LAN-A, dozwolony |
| R5 | 172.16.51.0/24 | \`lo0.0 172.16.51.1/24\` | Management-A, kontrola eksportu |
| R6 | 172.16.60.0/24 | \`lo0.0 172.16.60.1/24\` | Business-LAN-B, ruch górną drogą |
| R6 | 172.16.61.0/24 | \`lo0.0 172.16.61.1/24\` | Services-B, ruch dolną drogą |
| R6 | 172.16.62.0/24 | \`static discard\` | Świadomie odrzucony prefix testowy; **nie jest realną usługą** |

W standardowym wariancie R5/R6 mają jeden \`lo0.0\` z wieloma adresami IPv4; przed wdrożeniem sprawdź strefy, host-inbound i zachowanie pingów do lokalnych adresów na konkretnej wersji vSRX. Nie ogłaszaj \`172.16.62.0/24\` poza izolowanym laboratorium.

## Peeringi eBGP na sześciu łączach

| Sąsiedzi | Adresy peerów |
|---|---|
| R5 ↔ R1 | 192.0.2.10 ↔ 192.0.2.1 |
| R1 ↔ R2 | 10.12.0.1 ↔ 10.12.0.2 |
| R1 ↔ R3 | 10.13.0.1 ↔ 10.13.0.2 |
| R2 ↔ R4 | 10.24.0.1 ↔ 10.24.0.2 |
| R3 ↔ R4 | 10.34.0.1 ↔ 10.34.0.2 |
| R4 ↔ R6 | 198.51.100.1 ↔ 198.51.100.10 |

Na R1 i R4 grupy CORE/CE mają **różne funkcje**: oddzielny export do rdzenia, import od CE oraz ewentualna kontrola ogłoszeń wracających do CE. Nie zalecamy globalnego \`export accept\`. Wszystkie łącza stanowią osobne izolowane segmenty L2; topology musi być zgodne z EVE-NG i z wymaganiami IPv4/security SRX.

## Kryteria testu

1. Każdy z sześciu peerów BGP ma stan Established i poprawny AS.
2. R5 propaguje sieć business do R6; R6 propaguje dwie sieci biznesowe do R5 przez dwa korytarze.
3. R1 może wybrać 172.16.60.0/24 przez R2, a 172.16.61.0/24 przez R3 przez LOCAL_PREF import policy, bez modyfikowania topologii.
4. Prefiks 172.16.62.0/24 może docierać do R1 jako BGP update, ale zostać odrzucony w import policy; widoczność \`hidden\` zależy od opcji \`keep\` oraz szczegółów release.
5. Eksport R1 do core ma jawną whitelistę; usunięcie terminalnego reject jest testem **route leak**, nie normalnym wariantem produkcyjnym.
6. Test \`lo0.0\` protection i stateless filter nie może trwale odciąć BGP ani zarządzania; wymagany niezależny console/OOB, \`commit confirmed\` i jawny rollback.
7. Każdy wynik CLI w dokumentacji pozostaje **EXPECTED**, dopóki nie zostanie uzyskany z realnego obrazu vSRX.

## Relacja do starszych labów

Rozdziały 2–7 historycznie korzystają z 4×vSRX+2×VPCS, a nowa wersja rozdziału 8 korzysta z **DIAMOND-6R**. Nie podmieniaj automatycznie w poprzednich rozdziałach słów PC-A/PC-B na R5/R6: na routerze ruch do własnego loopbacku to host-bound, a ping z routera nie jest tym samym co ruch tranzytowy z rzeczywistego komputera. Można później zdecydować o ujednoliceniu pozostałych rozdziałów, ale musi to być jawna migracja.
