# Preliminary Steps to Support the Infrastructure

These steps are not automated and involve real hardware – a router with OpenWRT firmware. The goal of these steps is to prepare a self-hosted CA and a self-hosted DNS server with a dynamic private local DNS zone that can be updated via TSIG.

- [110-openwrt_on_proxmox.md](https://github.com/graysievert-lab/Homelab-010_DNS_x509CA/blob/master/110-openwrt_on_proxmox.md) - This is a short tutorial on how to virtualize an arm64 OpenWRT router in Proxmox VE. It is advised to test other tutorials on a virtual router prior to modifying real hardware.
- [120-bind_on_openwrt.md](https://github.com/graysievert-lab/Homelab-010_DNS_x509CA/blob/master/120-bind_on_openwrt.md) - This is a tutorial on how to install BIND on an arm64 OpenWRT router and configure dynamic DNS zone updates.
- [130-acme_on_openwrt.md](https://github.com/graysievert-lab/Homelab-010_DNS_x509CA/blob/master/130-acme_onopenwrt.md) - This is a tutorial on how to install your own ACME server on an arm64 OpenWRT router backed by step-ca software.
