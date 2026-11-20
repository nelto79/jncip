# DIAMOND-6R — sześć routerów vSRX, wariant do rozdziału 8

Pełne adresy, AS oraz zasady mapowania: [README.md](README.md). Diagram jest projektem, a nie potwierdzonym wynikiem EVE-NG.

~~~mermaid
flowchart LR
    R5["R5-vSRX<br/>CE-A · AS 65105<br/>192.0.2.10/24<br/>lo0 10.255.255.5/32"]
    R1["R1-vSRX<br/>EDGE-A · AS 65001<br/>lo0 10.255.255.1/32"]
    R2["R2-vSRX<br/>CORE-1 · AS 65002<br/>lo0 10.255.255.2/32"]
    R3["R3-vSRX<br/>CORE-2 · AS 65003<br/>lo0 10.255.255.3/32"]
    R4["R4-vSRX<br/>EDGE-B · AS 65004<br/>lo0 10.255.255.4/32"]
    R6["R6-vSRX<br/>CE-B · AS 65106<br/>198.51.100.10/24<br/>lo0 10.255.255.6/32"]

    R5 ---|"LAN-A 192.0.2.0/24<br/>R5 .10 ↔ R1 .1"| R1
    R1 ---|"10.12.0.0/30<br/>R1 .1 ↔ R2 .2"| R2
    R1 ---|"10.13.0.0/30<br/>R1 .1 ↔ R3 .2"| R3
    R2 ---|"10.24.0.0/30<br/>R2 .1 ↔ R4 .2"| R4
    R3 ---|"10.34.0.0/30<br/>R3 .1 ↔ R4 .2"| R4
    R4 ---|"LAN-B 198.51.100.0/24<br/>R4 .1 ↔ R6 .10"| R6

    classDef core fill:#e5efff,stroke:#3979bf,color:#173451;
    classDef ce fill:#e2f4e9,stroke:#32865a,color:#133e2b;
    class R1,R2,R3,R4 core;
    class R5,R6 ce;
~~~

**Górna ścieżka:** R5 → R1 → R2 → R4 → R6. **Dolna ścieżka:** R5 → R1 → R3 → R4 → R6.

Brak R2↔R3 i R1↔R4. Wszystkie R1–R4 mają tę samą adresację jak w starszym referencyjnym DIAMOND. R5 i R6 zajmują dokładnie odpowiednie adresy wcześniejszych hostów VPCS (ale są pełnymi vSRX, nie emulowanymi PC).
