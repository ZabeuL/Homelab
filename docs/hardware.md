# Hardware

## Server

| Component | Details |
|---|---|
| CPU | Intel i7-4790 (4c/8t, 3.6GHz base) |
| RAM | 32GB DDR3-1600 — 2× Kingston KVR16N11/8 + 2× HyperX KHX16C9B1BK2/8X |
| Boot Drive | 240GB SATA SSD |
| Motherboard | ASUS B85M-E (BIOS 3602) |
| NIC | GLOTRENDS LE8105 — PCIe 2.1 x1, Realtek RTL8125B chipset, 2.5GbE |

**RAM slot config:** HyperX kit in slots 1 & 3, Kingston in slots 2 & 4. XMP enabled for 1600MHz. MemTest86 passed — note flagged potential vulnerability to high-frequency row hammer bit flips, which is acceptable for a home lab.

**NIC:** The onboard NIC was replaced with the GLOTRENDS 2.5GbE card to match the Bell Gigahub's 2.5GbE WAN port. The RTL8125B is natively supported by Proxmox without extra drivers.

## Networking

| Device | Model | Notes |
|---|---|---|
| Router | Bell Gigahub 2.0 | ISP-provided, significantly locked down — see [Architecture](architecture.md) |

## Planned (Phase 2)

- 2× Seagate IronWolf or WD Red Plus 4TB (CMR only — no SMR)
- ZFS mirror pool

## Planned (Phase 3)

- PoE switch (TBD)
- 2× Reolink RLC-810A PoE cameras
- Google Coral USB TPU (required for Frigate hardware acceleration)
