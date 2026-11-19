# LAB CH05 — Monitoring & Troubleshooting na DIAMOND (4×vSRX + 2×VPCS)

**Status: PLANNED / NOT RUNTIME VALIDATED.** Podręcznik i specyfikacja są przygotowane, ale `.unl` ani scenariusze runtime nie zostały uruchomione w EVE-NG. Nie deklaruj PASS bez dowodów.

**Źródło prawdy:** [Mermaid DIAMOND](../reference-diamond-4vsrx-2vpcs/DIAMOND_TOPOLOGY.md), [mapa adresacji](../reference-diamond-4vsrx-2vpcs/README.md) i [rozbudowany rozdział 5.9](../../docs/chapters/JUNOS_Rozdzial_05_Operational_Monitoring_and_Maintenance.md). Poprzedni wariant dwóch vSRX został [zarchiwizowany](../../docs/archive/CH05_LEGACY_TWO_VSRX_MONITORING_LAB.md).

## Cel

Jest jedna sieć: PC-A **192.0.2.10/24** za R1, PC-B **198.51.100.10/24** za R4, tranzyt górny R1–R2–R4 i dolny R1–R3–R4. Każda kolejna usterka ma inne źródło: status interfejsu, next-hop, RIB/FIB/ARP, brak drogi powrotnej, security flow. Dzięki temu diagnozujesz ruch całego kampusu, a nie izolowane dwa pudełka.

~~~mermaid
flowchart LR
  PCA["PC-A / VPC1<br/>VPCS<br/>Gateway: 192.0.2.1"]
  R1["R1-vSRX<br/>lo0.0: 10.255.255.1/32"]
  R2["R2-vSRX<br/>lo0.0: 10.255.255.2/32"]
  R3["R3-vSRX<br/>lo0.0: 10.255.255.3/32"]
  R4["R4-vSRX<br/>lo0.0: 10.255.255.4/32"]
  PCB["PC-B / VPC2<br/>VPCS<br/>Gateway: 198.51.100.1"]

  PCA ---|"LAN-A · 192.0.2.0/24<br/>PC-A eth0: 192.0.2.10/24<br/>R1 ge-0/0/2.0: 192.0.2.1/24"| R1
  R1 ---|"P2P-12 · 10.12.0.0/30<br/>R1 ge-0/0/0.0: 10.12.0.1/30<br/>R2 ge-0/0/0.0: 10.12.0.2/30"| R2
  R1 ---|"P2P-13 · 10.13.0.0/30<br/>R1 ge-0/0/1.0: 10.13.0.1/30<br/>R3 ge-0/0/0.0: 10.13.0.2/30"| R3
  R2 ---|"P2P-24 · 10.24.0.0/30<br/>R2 ge-0/0/1.0: 10.24.0.1/30<br/>R4 ge-0/0/0.0: 10.24.0.2/30"| R4
  R3 ---|"P2P-34 · 10.34.0.0/30<br/>R3 ge-0/0/1.0: 10.34.0.1/30<br/>R4 ge-0/0/1.0: 10.34.0.2/30"| R4
  R4 ---|"LAN-B · 198.51.100.0/24<br/>R4 ge-0/0/2.0: 198.51.100.1/24<br/>PC-B eth0: 198.51.100.10/24"| PCB

  classDef vsrx fill:#e6f1fc,stroke:#1565a6,stroke-width:2px,color:#13344f;
  classDef vpcs fill:#e8f5e9,stroke:#2f7d4a,stroke-width:1.5px,color:#17422a;
  class R1,R2,R3,R4 vsrx;
  class PCA,PCB vpcs;
~~~

## Zasoby i mapowanie

Dokładnie **4 vSRX i 2 VPCS**, bez łącza R2–R3 albo R1–R4. Sześć segmentów IPv4: `192.0.2.0/24`, `10.12.0.0/30`, `10.13.0.0/30`, `10.24.0.0/30`, `10.34.0.0/30`, `198.51.100.0/24`. Routery mają `lo0.0`: R1 `10.255.255.1/32`, R2 `.2/32`, R3 `.3/32`, R4 `.4/32`.

Nazwa `ge-0/0/x` w diagramie to **docelowy mapping**, nie potwierdzenie interfejsu w danej wersji obrazu. Sprawdź `show interfaces terse` i porty EVE/QEMU. `fxp0` nie jest tranzytem.

## Szybki baseline: routing + SRX

Na **R1** prefiks LAN-B `198.51.100.0/24`: primary next-hop R2 `10.12.0.2`, qualified backup R3 `10.13.0.2`. Na **R4** prefiks LAN-A `192.0.2.0/24`: primary R2 `10.24.0.1`, qualified backup R3 `10.34.0.1`. Na R2 i R3 są static do obu zdalnych LAN-ów przez właściwy sąsiedni router. Na wszystkich data-interfaces należy świadomie skonfigurować strefę `LAB`, odpowiedni host-inbound ping i `LAB→LAB ALLOW-LAB` dla ICMP. **Pełne komendy dla czterech routerów w sekcjach 5.9.4–5.9.5 podręcznika.**

Nie uruchamiaj `commit check` po rozpoczęciu pending `commit confirmed`. Wszystkie testy wykonywane na osobnym labie z niezależną konsolą/OOB.

## Scenariusze awarii

| ID | Fault i urządzenie | Oczekiwane rozumowanie | Recovery |
|---|---|---|---|
| CH05-A | R1 `ge-0/0/0 disable` | Lokalny link do R2 admin-down; R1 może wybrać R3, ale R4 **wciąż** może próbować wracać przez R2 — nie gwarantuj end-to-end PASS | Auto-rollback, pomiar obu kierunków |
| CH05-A2 | oba górne odcinki R1–R2 oraz R2–R4 down | Obie strony powinny lokalnie preferować R3 po odtworzeniu nowych sesji, z uwzględnieniem SRX | Przywróć oba linki i sesje |
| CH05-B | R1 `delete routing-options static route 198.51.100.0/24` | Porty up, brak obu statycznych next-hopów do LAN-B (jeżeli brak innego route) | Auto-rollback i RIB/FIB |
| CH05-C | R2 `LAB→LAB ALLOW-LAB` permit→deny | Nowe sesje ICMP mogą być odrzucone pomimo właściwego routingu | Auto-rollback i nowy flow |
| CH05-D | R1 `set interfaces ge-0/0/2 unit 0 family inet address 192.0.2.1/33` | Celowo błędny CIDR ma zostać odrzucony przez parser | Nie commituj |
| CH05-E | R1 `monitor traffic interface ge-0/0/0` podczas PC-A→PC-B | RE capture nie dowodzi wszystkich pakietów PFE; compare z sessions | Zakończ capture |
| CH05-F | Wariant maintenance na klonie R1 | Backup, storage, release check, plan rollback/reboot | Nie instaluj software bez zatwierdzonego planu |

## Verification: pomiar i interpretacja

**R1-vSRX i R4-vSRX — na każdym osobno, Operational Mode**

~~~junos
show version
show interfaces terse
show route
show route forwarding-table
show arp
show security flow session
show security policies
show system commit
~~~

**R2-vSRX i R3-vSRX — na każdym osobno, Operational Mode**

~~~junos
show interfaces terse
show route 192.0.2.0/24 exact
show route 198.51.100.0/24 exact
show security flow session
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

## Dokumenty wynikowe i bezpieczeństwo

Agent budujący lab ma umieścić w tej przestrzeni `VERIFY.md` (pre/post, interface states, route selection, ARP, session, raw evidence, PASS/FAIL/NOT TESTED), `TROUBLESHOOTING.md` (symptom → powód → naprawa), a plik `.unl` przygotować według **wspólnego** wzorca DIAMOND, bez duplikowania niezgodnej adresacji. Nie commituj obrazów QCOW2, haseł, kluczy, tokenów ani adresów rzeczywistego zarządzania.

**Ważne:** stan dokumentacji `PLANNED` nie jest tożsamy z `RUNTIME VALIDATED`. Platformowy DOM, fizyczne wkładki, STP/Virtual Chassis nie są ćwiczeniami, które można zweryfikować na czterech wirtualnych SRX.
