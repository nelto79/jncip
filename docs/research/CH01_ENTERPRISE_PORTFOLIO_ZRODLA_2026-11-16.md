# Chapter 1 — Enterprise Portfolio, Greenfield & Junos version verification

**Research date:** 2026-11-16. **Canonical learner chapter:** [Chapter 1](../chapters/JUNOS_Rozdzial_01_Networking_Fundamentals_i_Enterprise_Portfolio.md). This register is for the author/Codex, not an extra catalogue to memorize.

## Official authoritative sources

- **Campus devices by layer (EX/QFX), architecture and topologies:** [Juniper choose Campus Fabric topology](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/concept/choose-campus-fabric-topology.html) and [Campus migration from traditional networks](https://www.juniper.net/documentation/us/en/software/mist/mist-wired/topics/concept/migrate-to-campus-fabric.html), updated autumn 2026. Models listed include EX4100/EX4400 access, EX4400-24X/EX4650/QFX5120/QFX5130/EX9200 in core/distribution depending on design.
- **Vendor's 2025 greenfield guide with inventory, PoE, wired and WLAN RF survey:** [AI-Driven Wired/Wireless Network Deployment Guide](https://www.juniper.net/documentation/us/en/software/nce/juniper-ai-driven-wired-wireless-network-deployment-guide/juniper-ai-driven-wired-wireless-network-deployment-guide.pdf). Useful for HLD, Mist/ZTP and lessons on distributed vs tunneled WLAN datapath.
- **EX4000 physical SKU and port profiles:** [EX4000 datasheet](https://www.juniper.net/gb/en/products/switches/ex-series/ex4000-line-of-ethernet-switches-datasheet.html).
- **EX4100 (regular)**: [specs](https://www.juniper.net/us/en/products/switches/ex-series/ex4100-ethernet-switch/specs.html).
- **EX4100-F (fixed PSU/fan or fanless variants)**: [model matrix](https://www.juniper.net/documentation/us/en/hardware/ex4100-f/topics/concept/EX4100-f-models.html) updated October 2026.
- **EX4400 variants, 24X vs PoE and mGig**: [models](https://www.juniper.net/documentation/us/en/hardware/ex4400/topics/concept/ex4400-models.html), [24X specs](https://www.juniper.net/us/en/products/switches/ex-series/ex4400-24x-ethernet-switch/specs.html), [mGig specs](https://www.juniper.net/us/en/products/switches/ex-series/ex4400-multigigabit-enterprise-switch/specs.html).
- **EX4650 distribution/core**: [specs](https://www.juniper.net/us/en/products/switches/ex-series/ex4650-campus-aggregation-and-core-switch/specs.html).
- **QFX5130 uses Junos OS Evolved**: [system overview](https://www.juniper.net/documentation/us/en/hardware/qfx5130/topics/topic-map/qfx5130-system-overview.html).
- **AP models and radios**: [official AP comparison](https://www.juniper.net/documentation/us/en/software/nce/juniper-ai-driven-wired-wireless-network-deployment-guide/wireless_network_deployment_juniper_aps.html) and [AP portfolio](https://www.juniper.net/us/en/products/access-points.html).
- **AP47 full features require 802.3bt power, about 29W, and reduced capability on 802.3at**: [AP47 hardware deployment guide](https://www.juniper.net/documentation/us/en/hardware/ap47/ap47.pdf). [dual-uplink Wi-Fi 7 setup](https://www.juniper.net/documentation/us/en/software/mist/mist-wireless/topics/topic-map/wifi-7-tech-ref.html).
- **SRX branch vs campus and different performance metrics**: [SRX family](https://www.juniper.net/us/en/products/security/srx-series.html/).
- **SSR is Session Smart software, not Junos**: [SSR product](https://www.juniper.net/us/en/products/routers/session-smart-router.html) and [appliances](https://www.juniper.net/us/en/products/routers/session-smart-router/session-smart-networking-datasheet.html).
- **Software suggestions per platform**: [JTAC KB21476](https://supportportal.juniper.net/s/article/Junos-Software-Versions-Suggested-Releases-to-Consider-and-Evaluate) updated **2026-11-04**. Contains EX4000/4100/4400/4650 suggested **24.4R2-Sx**, EX9200 **23.4R2-Sx**, QFX5120 **25.2R2-Sx**, QFX5130 **25.2X100-D21-EVO**, QFX5700 **25.2X100-D35-EVO**, SRX300/380 **24.4R2-Sx**, SRX1600/2300 **23.4R2-Sx**, MX204/MX304 **24.4R2-Sx**. Check exact sub-variant and release notes in any actual project.
- **Junos release support milestones**: https://support.juniper.net/support/eol/software/junos/ .
- **SSR release milestones** (7.2 published July 2026, not necessarily recommendation): https://support.juniper.net/support/eol/software/ssr/ .
- **Mist AP hardware lifecycle** (do not select old EOL model by default): https://support.juniper.net/support/eol/product/juniper_ap_series/ .
- **Hardware Explorer and Feature Explorer** for exact PIC/optic/software capabilities: https://apps.juniper.net/hct/ and https://apps.juniper.net/feature-explorer/.

## Caveats for future chapter edits

1. These model shortlists are **examples, not verified popularity rankings or supplier quote**. “Most frequently used” should not be claimed from market-share data without actual deployment/popularity evidence.
2. Product family vs precise orderable SKU: EX4400 has PoE access and 24X distribution variants; EX4400-48MP port speeds are mixed. No final BOM without port count, PoE, optics, subscriptions, installation and support.
3. Recommended releases are a **dated support article snapshot**, not device-specific installed version and not always most recent release; \`-Sx\` is a wildcard for service releases.
4. Mist AP and SSR have their own software systems and do not accept Junos images.
5. AP47 802.3bt detail is a concrete licensing/PoE/electrical BOM gotcha; verify AP SKU, 6 GHz region, power redundancy and exact switch PoE budget.
6. Official vendor PDF diagrams may be linked but not automatically licensed for republication; use own Mermaid or properly licensed images rather than copying manufacturer imagery into repository.
7. Revalidate availability, lifecycle and exact software support immediately before quoting to a customer, or mark **TO VERIFY**.

