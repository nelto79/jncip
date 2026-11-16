# JUNOS Enterprise Routing & Switching — Unified Study Blueprint

**Cel:** jedna struktura nauki obejmująca pełen jawny zakres JNCIA-Junos (JN0-106), JNCIS-ENT (JN0-352) i JNCIP-ENT (JN0-650). Zagadnienia współdzielone są ujęte raz, ale rozwijane stopniowo do poziomu Professional. Materiał służy przygotowaniu merytorycznemu do JN0-650, również w rozważanej ścieżce Cisco → Juniper, bez przesądzania o formalnej kwalifikacji do programu migracyjnego.

**Stan weryfikacji:** 15 listopada 2026.

**Oznaczenia:** `[A]` = Associate (JN0-106); `[S]` = Specialist (JN0-352); `[P]` = Professional (JN0-650); `[LAB]` = proponowane ćwiczenie; `[EXT]` = rozwinięcie dydaktyczne niebędące osobnym literalnym punktem blueprintu.

**Zasada pracy z rozdziałami:** w zagadnieniach, które blueprint określa jako „describe / identify” — wyjaśnij koncepcję; w zagadnieniach typu „configure, monitor, troubleshoot” — pokaż konfigurację Junos, polecenia weryfikacyjne, typowe błędy i diagnostykę. Te cztery rodzaje kompetencji dotyczą odpowiednich sekcji każdego rozdziału, a nie tylko końcowego rozdziału laboratoryjnego.

## CZĘŚĆ I — FUNDAMENTY SIECI I SYSTEMU JUNOS

### 1. Networking Fundamentals — fundamenty transmisji i adresacji [A]
1.1. Funkcje routera i przełącznika; różnice między działaniem L2 i L3.  
1.2. Sieci Ethernet i podstawy przetwarzania ramek.  
1.3. Collision domains i broadcast domains.  
1.4. Adresowanie warstwy 2; rozpoznawanie adresów (address resolution, ARP/ND w kontekście IPv4/IPv6).  
1.5. Adresowanie IP warstwy 3; maska i długość prefiksu.  
1.6. IPv4 i IPv6 — cechy adresowania, podstawowe różnice.  
1.7. Subnetting i supernetting.  
1.8. Konwersja systemu dziesiętnego na binarny i odwrotnie.  
1.9. Longest Prefix Match — wybór najbardziej szczegółowego prefiksu.  
1.10. Connection-oriented vs. connectionless protocols (TCP vs. UDP jako przykład).  
1.11. Wprowadzenie do Class of Service (CoS) — po co sieć klasyfikuje ruch.  
1.12. [EXT] **Juniper Enterprise Portfolio & Greenfield Design** — EX access/distribution/core, QFX campus fabric, SRX security, MX/SSR WAN edge, Mist WLAN, Junos OS vs Evolved vs AP/SSR firmware; podstawy HLD/LLD, wyboru SKU, PoE/uplink budget, BOM, lifecycle i sugerowane wersje JTAC. *Rozszerzenie praktyczne dla inżyniera; nie jest osobnym celem oficjalnego JNCIA.* Zobacz [`docs/chapters/JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md`](chapters/JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md).  

### 2. Junos OS Fundamentals — architektura systemu [A]
2.1. Architektura oprogramowania Junos OS.  
2.2. Rozdzielenie control plane i forwarding plane.  
2.3. Routing Engine (RE) — rola i funkcje.  
2.4. Packet Forwarding Engine (PFE) — rola i funkcje.  
2.5. Przetwarzanie ruchu tranzytowego (transit traffic).  
2.6. Przetwarzanie ruchu wyjątkowego (exception traffic) i relacja do control plane.

### 3. Junos User Interfaces — CLI, konfiguracja i J-Web [A]
3.1. Funkcje CLI i zasady interakcji z systemem.  
3.2. Operational mode vs. configuration mode.  
3.3. Nawigacja po hierarchii konfiguracji i po poleceniach CLI.  
3.4. Kontekstowa pomoc CLI i uzupełnianie poleceń.  
3.5. Filtrowanie wyników poleceń.  
3.6. Konfiguracja active vs. candidate.  
3.7. Zmiany konfiguracji: modyfikowanie, usuwanie, zarządzanie i zapisywanie.  
3.8. Przeglądanie, porównywanie i wczytywanie plików konfiguracji.  
3.9. Cofanie zmian i powrót do wcześniejszych konfiguracji (rollback).  
3.10. Podstawowe, wspólne funkcje J-Web.  
3.11. [EXT] Praktyczna organizacja pracy: `display set`, `commit check`, `commit confirmed`, porównanie zmian.

### 4. Initial Configuration and System Services — podstawowa administracja [A]
4.1. Factory-default state i zachowanie urządzenia po pierwszym uruchomieniu.  
4.2. Initial configuration — podstawowe parametry systemu.  
4.3. Konta użytkowników lokalnych.  
4.4. Login classes — role i przywileje.  
4.5. Metody uwierzytelniania użytkowników.  
4.6. Typy interfejsów i ich właściwości — podstawowe parametry konfiguracji.  
4.7. Configuration groups — dziedziczenie i wielokrotne wykorzystanie fragmentów konfiguracji.  
4.8. NTP — synchronizacja czasu.  
4.9. SNMP — podstawy monitoringu.  
4.10. System log / syslog — podstawy rejestracji zdarzeń.  
4.11. Configuration archival — archiwizowanie konfiguracji.  
4.12. Logging and tracing — diagnostyka systemu i protokołów.  
4.13. Rescue configuration — zapis i odtwarzanie konfiguracji ratunkowej.  
4.14. Konfiguracja i weryfikacja wszystkich podstawowych elementów systemu.  
4.15. [LAB] Konfiguracja nowego routera: konta, dostęp SSH, NTP, syslog, konfiguracja ratunkowa.

### 5. Operational Monitoring and Maintenance — eksploatacja [A]
5.1. Polecenia `show` — odczyt stanu systemu, konfiguracji, tras i interfejsów.  
5.2. Polecenia `monitor` — bieżąca obserwacja działania systemu.  
5.3. Statystyki interfejsów i rozpoznawanie błędów.  
5.4. Narzędzia sieciowe: ping, traceroute, telnet, SSH i podobne.  
5.5. Instalacja i aktualizacja Junos OS.  
5.6. Uruchamianie i bezpieczne wyłączanie urządzeń.  
5.7. Odzyskiwanie hasła root.  
5.8. Procedury monitorowania, konserwacji i diagnozowania problemów.  
5.9. [LAB] Diagnostyka niedziałającego interfejsu, brakującej trasy i błędnej konfiguracji.

## CZĘŚĆ II — ROUTING PODSTAWOWY, RIB/FIB I POLICIES

### 6. Routing Fundamentals — przekazywanie pakietów i podstawowe trasy [A → S]
6.1. Podstawowe pojęcia przekazywania pakietów IP. [A]  
6.2. Routing tables i ich rola. [A]  
6.3. Routing Information Base (RIB) vs. forwarding table / FIB. [A]  
6.4. Route preference i wybór preferowanego źródła trasy. [A]  
6.5. Longest Prefix Match w rzeczywistym forwarding. [A]  
6.6. Routing instances — po co i kiedy izolujemy tablice routingu. [A]  
6.7. Trasy connected i static; trasa domyślna. [A]  
6.8. Dynamic routing — korzyści i przypadki użycia; kontekst IGP/EGP. [A]  
6.9. Konfiguracja i monitorowanie podstawowych funkcji routingu. [A]  
6.10. Static routes — warianty i troubleshooting. [S]  
6.11. [LAB] Konfiguracja IPv4/IPv6, statycznych tras i tras domyślnych na dwóch vSRX.

### 7. Protocol-Independent Routing — trasy i mechanizmy niezależne od protokołu [S]
7.1. Static, aggregate i generated routes.  
7.2. Martian addresses — cel i wpływ na routing.  
7.3. Routing instances — rozwinięcie konfiguracji i diagnostyki.  
7.4. Routing Information Base (RIB) groups.  
7.5. Load balancing — mechanizmy i konfiguracja.  
7.6. Filter-Based Forwarding (FBF) — podstawy i zastosowania.  
7.7. Konfiguracja, monitoring i troubleshooting: static/aggregate/generated routes, load balancing, FBF.  
7.8. [LAB] Aggregate route + FBF + RIB group — weryfikacja decyzji routingu.

### 8. Routing Policy and Firewall Filters — polityki i filtrowanie [A → S → P]
8.1. Default routing policies. [A]  
8.2. Import policy vs. export policy. [A]  
8.3. Routing policy flow — kolejność przetwarzania i zakończenie ewaluacji. [A]  
8.4. Wpływ policies na trasy i routing tables. [A]  
8.5. Struktura polityki: policy statements i terms. [A]  
8.6. Match criteria, typy dopasowań i actions w routing policy. [A]  
8.7. Firewall filters — koncepcje i cele. [A]  
8.8. Struktura filtrów: terms, match criteria, actions. [A]  
8.9. Wpływ firewall filters na pakiety. [A]  
8.10. Unicast Reverse Path Forwarding (uRPF). [A]  
8.11. Konfiguracja i monitoring podstawowych routing policies i firewall filters. [A]  
8.12. Routing policy applied to OSPF, IS-IS and BGP. [S]  
8.13. Zaawansowane zastosowania routing policy w OSPF/IGP i BGP. [P]  
8.14. [LAB] Import/export BGP, filtrowanie prefiksów i weryfikacja ścieżki polityki.

## CZĘŚĆ III — DYNAMIC ROUTING I TUNELE

### 9. OSPF — od protokołu link-state do zaawansowanego IGP [S → P]
9.1. Link-state database (LSDB). [S]  
9.2. Rodzaje pakietów OSPF. [S]  
9.3. Router ID. [S]  
9.4. Neighbors i adjacencies. [S]  
9.5. Designated Router (DR) i Backup Designated Router (BDR). [S]  
9.6. Area types i router types. [S]  
9.7. Realms. [S]  
9.8. Rodzaje LSA. [S]  
9.9. Konfiguracja obszarów, interfejsów i sąsiadów. [S]  
9.10. Dodatkowe podstawowe opcje OSPF. [S]  
9.11. Zastosowanie routing policies w OSPF. [S]  
9.12. Troubleshooting: ping, traceroute, traceoptions, show commands, logging. [S]  
9.13. OSPFv2 — koncepcje, działanie, konfiguracja i diagnostyka. [P]  
9.14. OSPFv3 — koncepcje, działanie, konfiguracja i diagnostyka. [P]  
9.15. Routing policy dla IGP — zaawansowane scenariusze. [P]  
9.16. [LAB] Multi-area OSPF z błędem sąsiedztwa i routing policy.

### 10. IS-IS — kompletny zakres Specialist [S]
10.1. IS-IS link-state database.  
10.2. IS-IS Protocol Data Units (PDUs).  
10.3. Type-Length-Value (TLV).  
10.4. Neighbors i adjacencies.  
10.5. Levels i areas.  
10.6. Designated Intermediate System (DIS).  
10.7. Metrics.  
10.8. Konfiguracja levels, interfejsów i sąsiedztw.  
10.9. Dodatkowe podstawowe opcje IS-IS.  
10.10. Zastosowanie routing policy do IS-IS.  
10.11. Monitoring i troubleshooting: ping, traceroute, traceoptions, show commands, logging.  
10.12. [LAB] IS-IS L1/L2 i analiza nieustanowionej adjacency.

### 11. BGP — od eBGP/iBGP do zaawansowanego routingu [S → P]
11.1. Podstawowe zasady działania BGP. [S]  
11.2. BGP message types. [S]  
11.3. BGP attributes — wprowadzenie. [S]  
11.4. Route/path selection process — podstawy. [S]  
11.5. Internal BGP (iBGP) i external BGP (eBGP) — działanie i wzajemne zależności. [S]  
11.6. Konfiguracja groups i peers. [S]  
11.7. Additional basic BGP options. [S]  
11.8. Zastosowanie routing policies w BGP. [S]  
11.9. Monitoring i troubleshooting: ping, traceroute, traceoptions, show, logging. [S]  
11.10. Zaawansowany BGP route selection process. [P]  
11.11. Next-hop resolution. [P]  
11.12. Szczegółowe działanie atrybutów BGP. [P]  
11.13. BGP communities. [P]  
11.14. Load balancing: multipath, multihop, forwarding table. [P]  
11.15. NLRI families: inet i inet6. [P]  
11.16. Advanced BGP options. [P]  
11.17. Implementacja BGP routing policies w scenariuszach zaawansowanych. [P]  
11.18. Zaawansowana konfiguracja, monitoring i troubleshooting BGP. [P]  
11.19. [LAB] eBGP/iBGP, policy, community, next-hop i multipath.

### 12. IP Tunnels — tunelowanie na poziomie Specialist [S]
12.1. Zastosowania tuneli IP i ograniczenia projektowe.  
12.2. Generic Routing Encapsulation (GRE).  
12.3. IP-in-IP (IP-IP).  
12.4. Konfiguracja, monitoring i troubleshooting GRE.  
12.5. Konfiguracja, monitoring i troubleshooting IP-IP.  
12.6. Narzędzia diagnostyczne: ping, traceroute, traceoptions, show, logging.  
12.7. [LAB] GRE / IP-IP pomiędzy vSRX, z trasą przez tunel.

### 13. IP Multicast — routing grupowy [P]
13.1. Komponenty architektury multicast i adresowanie multicast.  
13.2. IP multicast traffic flow.  
13.3. Any-Source Multicast (ASM) vs. Source-Specific Multicast (SSM).  
13.4. Reverse Path Forwarding (RPF) w multicast — koncepcje i działanie.  
13.5. Internet Group Management Protocol (IGMP).  
13.6. IGMP snooping.  
13.7. Protocol Independent Multicast Sparse Mode (PIM-SM).  
13.8. Rendezvous Point (RP): koncepcja, działanie, discovery, election.  
13.9. Anycast RP.  
13.10. Multicast Source Discovery Protocol (MSDP).  
13.11. Multicast routing policy i scoping.  
13.12. Konfiguracja, monitoring i troubleshooting IGMP, PIM-SM i SSM.  
13.13. Implementacja multicast routing policy.  
13.14. [LAB] IGMP + PIM-SM, RP, RPF failure i multicast policy.

## CZĘŚĆ IV — ENTERPRISE SWITCHING I DOSTĘP L2

### 14. Ethernet Switching, Bridging and VLANs — od L2 do advanced switching [A → S → P]
14.1. Ethernet, adresowanie L2, broadcast domain i zasada działania switcha. [A]  
14.2. Bridging components w Junos. [S]  
14.3. Frame processing. [S]  
14.4. Ports — typy i funkcje w VLAN. [S]  
14.5. VLAN tagging. [S]  
14.6. Native VLAN. [S]  
14.7. Voice VLAN — podstawy. [S]  
14.8. Inter-VLAN routing / IRB — koncepcje i konfiguracja. [S]  
14.9. Konfiguracja, monitoring i troubleshooting interfejsów/portów, VLAN i inter-VLAN routing. [S]  
14.10. Filter-based VLANs — działanie, konfiguracja i diagnostyka. [P]  
14.11. Private VLANs — koncepcje i działanie. [P]  
14.12. Multiple VLAN Registration Protocol (MVRP) — dynamiczna rejestracja VLAN, konfiguracja i diagnostyka. [P]  
14.13. Tunneling ruchu Layer 2 przez sieci Ethernet — koncepcje i implementacja. [P]  
14.14. Q-in-Q — Layer 2 tunneling, konfiguracja i diagnostyka. [P]  
14.15. L2PT — Layer 2 Protocol Tunneling, konfiguracja i diagnostyka. [P]  
14.16. [LAB] Access/trunk/IRB, następnie Q-in-Q i advanced VLAN.

### 15. Spanning Tree Protocols — STP/RSTP → MSTP/VSTP [S → P]
15.1. STP i RSTP — założenia i działanie. [S]  
15.2. Port roles i port states. [S]  
15.3. Bridge Protocol Data Units (BPDUs). [S]  
15.4. Convergence i reconvergence. [S]  
15.5. Konfiguracja, monitoring i troubleshooting STP. [S]  
15.6. Konfiguracja, monitoring i troubleshooting RSTP. [S]  
15.7. Multiple Spanning Tree Protocol (MSTP) — koncepcja, działanie, konfiguracja i diagnostyka. [P]  
15.8. VLAN Spanning Tree Protocol (VSTP) — koncepcja, działanie, konfiguracja i diagnostyka. [P]  
15.9. [LAB] Root election, pętla L2, RSTP convergence, MSTP/VSTP.

### 16. Layer 2 Security — ochrona dostępu i pętli [S]
16.1. BPDU protection.  
16.2. Loop protection.  
16.3. Root protection.  
16.4. Port security.  
16.5. MAC limiting.  
16.6. DHCP snooping.  
16.7. Dynamic ARP Inspection (DAI).  
16.8. IP Source Guard.  
16.9. MACsec.  
16.10. Storm control.  
16.11. Layer 2 firewall filter types.  
16.12. Layer 2 filter processing order.  
16.13. Layer 2 filter match criteria and actions.  
16.14. Konfiguracja, monitoring i troubleshooting L2 protection, port security, storm control oraz filtrów L2.  
16.15. [LAB] DHCP snooping + DAI + BPDU/loop/root protection.

### 17. High Availability — odporność usług i urządzeń [S]
17.1. Link Aggregation Groups (LAG).  
17.2. Redundant Trunk Groups (RTG).  
17.3. Virtual Chassis.  
17.4. Graceful Restart (GR).  
17.5. Graceful Routing Engine Switchover (GRES).  
17.6. Nonstop Active Routing (NSR).  
17.7. Nonstop Bridging (NSB).  
17.8. Bidirectional Forwarding Detection (BFD).  
17.9. Virtual Router Redundancy Protocol (VRRP).  
17.10. Unified In-Service Software Upgrade (ISSU).  
17.11. Konfiguracja, monitoring i troubleshooting LAG/RTG i Virtual Chassis.  
17.12. Konfiguracja, monitoring i troubleshooting GR/GRES/NSR/NSB.  
17.13. Konfiguracja, monitoring i troubleshooting VRRP i ISSU.  
17.14. Narzędzia diagnostyczne: traceoptions, show commands, logging.  
17.15. [LAB] LAG + VRRP + BFD i scenariusz awarii uplinku.

### 18. Layer 2 Authentication and Access Control — NAC [P]
18.1. Authentication process flow.  
18.2. IEEE 802.1X — koncepcja i działanie.  
18.3. MAC RADIUS.  
18.4. Captive portal.  
18.5. Server fail fallback.  
18.6. Guest VLAN.  
18.7. Interakcje i ograniczenia równoczesnego użycia wielu metod uwierzytelniania i kontroli dostępu.  
18.8. Konfiguracja, monitoring i troubleshooting L2 authentication i access control.  
18.9. [LAB] 802.1X/MAC RADIUS, guest VLAN, brak RADIUS i zachowanie fallback.

### 19. IP Telephony Features — funkcje switcha dla VoIP [S → P]
19.1. Voice VLAN — rola i podstawy. [S]  
19.2. Power over Ethernet (PoE) — zasilanie endpointów. [P]  
19.3. Link Layer Discovery Protocol (LLDP). [P]  
19.4. LLDP Media Endpoint Discovery (LLDP-MED). [P]  
19.5. Voice VLAN — zastosowanie w scenariuszu IP telephony. [P]  
19.6. Konfiguracja, monitoring i troubleshooting funkcji wspierających IP telephony. [P]  
19.7. [LAB] Telefon VoIP + komputer na porcie, LLDP-MED, voice VLAN, PoE.

## CZĘŚĆ V — QoS/CoS I FABRIC

### 20. Class of Service (CoS) — od pojęć do konfiguracji L2/L3 [A → P]
20.1. CoS jako element podstaw sieci — klasyfikacja i priorytetyzacja ruchu. [A]  
20.2. CoS processing na urządzeniach Junos. [P]  
20.3. CoS header fields. [P]  
20.4. Forwarding classes. [P]  
20.5. Classification. [P]  
20.6. Packet Loss Priority (PLP). [P]  
20.7. Policers. [P]  
20.8. Schedulers. [P]  
20.9. Drop profiles. [P]  
20.10. Shaping. [P]  
20.11. Rewrite rules. [P]  
20.12. Konfiguracja, monitoring i troubleshooting CoS w sieciach Layer 2 i Layer 3. [P]  
20.13. [LAB] Classification → forwarding class → scheduler → rewrite rule.

### 21. EVPN and VXLAN — zaawansowany overlay [P]
21.1. EVPN — koncepcja, funkcje i działanie w Junos.  
21.2. EVPN route types: Type 1, Type 2 i Type 3 (oraz związek aktywnych route types z multihoming).  
21.3. VXLAN — enkapsulacja i rola w EVPN.  
21.4. EVPN multihoming — koncepcje i działanie.  
21.5. Konfiguracja, monitoring i troubleshooting EVPN.  
21.6. [LAB] EVPN/VXLAN, rozgłaszanie MAC/IP, weryfikacja tras EVPN i failover multihoming.

## CZĘŚĆ VI — PRZYGOTOWANIE PRAKTYCZNE I EGZAMINACYJNE

### 22. Integrated Configuration, Monitoring and Troubleshooting [A + S + P; LAB/EXT]
22.1. Schemat rozwiązywania scenariuszy: stan urządzenia → interfejs → RIB/FIB → policy → protokół → forwarding.  
22.2. Odczytywanie CLI i interpretacja niepełnej lub błędnej konfiguracji.  
22.3. Workflow: `show`, `monitor`, `ping`, `traceroute`, `traceoptions`, logi, statystyki.  
22.4. Metoda zmian: candidate config → diff → commit check → commit confirmed → rollback.  
22.5. Labs routing: static, routing instances, RIB groups, OSPF, IS-IS, BGP, GRE/IP-IP, multicast.  
22.6. Labs switching: VLAN/IRB, STP/RSTP/MSTP/VSTP, security, HA, 802.1X, CoS, EVPN/VXLAN.  
22.7. Ograniczenia platform: vSRX nadaje się do CLI i wielu zagadnień L3, ale nie zastępuje Junos EX w ćwiczeniach switchingowych.  
22.8. Mock exams i powtórka: blok A, blok S i blok P.  
22.9. Matryca gotowości: „potrafię wyjaśnić / skonfigurować / zweryfikować / naprawić”.

## CZĘŚĆ VII — MAPA ŚLEDZENIA ŹRÓDŁOWYCH BLUEPRINTÓW

### 23. Official Blueprint Coverage and Traceability

| Poziom | Oficjalna domena | Rozdziały niniejszego opracowania |
|---|---|---|
| JNCIA | Networking Fundamentals | 1, 6, 14, 20 |
| JNCIA | Junos OS Fundamentals | 2 |
| JNCIA | User Interfaces | 3 |
| JNCIA | Configuration Basics | 4, 6 |
| JNCIA | Operational Monitoring and Maintenance | 5, 22 |
| JNCIA | Routing Fundamentals | 6, 7 |
| JNCIA | Routing Policy and Firewall Filters | 8 |
| JNCIS | Layer 2 Switching or VLANs | 14, 19 |
| JNCIS | Spanning Tree | 15 |
| JNCIS | Layer 2 Security | 16, 8 |
| JNCIS | Protocol Independent Routing | 7, 6 |
| JNCIS | OSPF | 9 |
| JNCIS | IS-IS | 10 |
| JNCIS | BGP | 11 |
| JNCIS | Tunnels | 12 |
| JNCIS | High Availability | 17 |
| JNCIP | Interior Gateway Protocols (IGPs) | 9, 8 |
| JNCIP | BGP | 11, 8 |
| JNCIP | IP Multicast | 13 |
| JNCIP | Ethernet Switching and Spanning Tree | 14, 15 |
| JNCIP | Layer 2 Authentication and Access Control | 18 |
| JNCIP | IP Telephony Features | 19 |
| JNCIP | Class of Service (CoS) | 20 |
| JNCIP | EVPN | 21 |

**Źródła pierwotne — oficjalne objectives:**
- JNCIA-Junos, JN0-106: https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=14354
- JNCIS-ENT, JN0-352: https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=14355
- JNCIP-ENT, JN0-650: https://learningportal.juniper.net/juniper/user_activity_info.aspx?id=14356

**Uwaga metodyczna:** Podział na 23 rozdziały, kolejność, dodatkowe podsekcje `[EXT]` i laboratoria `[LAB]` są autorską organizacją materiału. Nie są kolejnym, oficjalnym blueprintem egzaminu. Cele poziomu JNCIS pozostają w spisie również wtedy, gdy nie są wymienione ponownie jako osobne cele JNCIP. „Pokrycie blueprintu” oznacza pokrycie jawnie opublikowanych celów, a nie gwarancję zakresu wszystkich pytań egzaminacyjnych.
