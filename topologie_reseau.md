# Topologie Réseau : pfSense & Tailscale sur Proxmox

Ce document présente l'architecture réseau logique du montage complet.

## Schéma (Mermaid)

Vous pouvez visualiser ce schéma nativement sur GitHub (qui supporte la syntaxe Mermaid).

```mermaid
graph TD
    subgraph "Site Distant (Extérieur)"
        RD[Appareil Distant / Client VPN]
    end

    subgraph "Internet & VPN"
        INT((Internet))
        TS{Réseau Tailscale}
    end

    subgraph "Domicile / Entreprise"
        BOX[Box FAI / Routeur Internet]
        
        subgraph "Serveur Proxmox VE"
            BR0[Bridge WAN : vmbr0]
            
            subgraph "pfSense VM"
                WAN[Interface WAN : vtnet0]
                LAN[Interface LAN : vtnet1 <br> 192.168.20.1]
            end
            
            BR1[Bridge LAN isolé : vmbr1]
            CVM[Client VM LAN <br> 192.168.20.x]
        end
    end

    %% Connexions standards
    RD --> INT
    INT --> BOX
    BOX --> BR0
    BR0 --> WAN
    WAN --> LAN
    LAN --> BR1
    BR1 --> CVM

    %% Tunnel Tailscale
    RD -. "Tunnel Sécurisé (Tailscale)" .- TS
    TS -. "Routage du sous-réseau 192.168.20.0/24" .- pfSense VM
```

## Description des flux

1. **Trafic Local :** La `Client VM` navigue sur Internet en passant par `vmbr1` -> `pfSense (LAN)` -> `pfSense (WAN)` -> `vmbr0` -> `Box FAI` -> `Internet`.
2. **Trafic Distant (VPN) :** L'`Appareil Distant` se connecte à Tailscale. Le trafic à destination de `192.168.20.x` est chiffré, envoyé au nœud Tailscale de la VM pfSense, puis transféré vers la `Client VM` sur le `vmbr1`. L'interface WAN de pfSense n'expose aucun port sur Internet.
