# pfSense to Azure site-to-site VPN (IPsec)

Every setting from the video, in order: an IPsec tunnel from pfSense at home to an Azure virtual network, with static address ranges (no BGP).

▶️ Video: https://youtu.be/sEgKqQ8Gm3Y

**Newer version with BGP:** https://youtu.be/0HYaSX_fvBE

## What you need
- pfSense CE on hardware that meets the minimum requirements: 64-bit amd64 (x86-64) CPU, 1 GB or more RAM, 8 GB or larger disk, one or more compatible network cards
- An Azure subscription
- An Azure virtual network with a VM in it and a `GatewaySubnet` (the video starts with these already made)
- The public IP of your pfSense WAN

> **Cost:** the virtual network gateway is billed for the time it's provisioned, even with nothing connected. Delete it when you're done testing.

## Addressing used in the video

Change these to yours.

| What | Value |
|---|---|
| Home server VLAN behind pfSense | 10.37.0.0/20 (pfSense 10.37.0.1, workstation 10.37.0.98) |
| Azure VM network | 10.90.0.0/20 |
| Azure VM (Windows Server 2022) | 10.90.0.4 |
| Azure region | West US 3 |

## 1. Create the virtual network gateway
Search **Virtual network gateways** → Create:

| Field | Value |
|---|---|
| Name | `Azure_vnet` |
| Region | West US 3 |
| Gateway type | VPN |
| VPN type | Route-based |
| SKU | VpnGw2AZ |
| Generation | Generation2 |
| Virtual network | your existing VNet (`WIN_Resource-vnet`, where the VM runs) |
| Subnet | picks `GatewaySubnet` by itself |
| Public IP address | Create new, name `vpn_publicip` (Standard, Static) |
| Availability zone | 1 |
| Enable active-active mode | Disabled |
| Configure BGP | Disabled |

Review + create → Create. It took about 30 minutes in the video.

## 2. Local network gateway (your home side)
Search **Local network gateways** → Create:

| Field | Value |
|---|---|
| Resource group | same as the gateway (`WIN_Resource`) |
| Region | West US 3 |
| Name | `pfsense_home` |
| Endpoint | IP address |
| IP address | `YOUR-PUBLIC-IP` (home public IP) |
| Address Space(s) | `10.37.0.0/20` (the home network Azure should reach) |

Review + create → Create.

## 3. Site-to-site connection
Virtual network gateway `Azure_vnet` → Connections → Add:

| Field | Value |
|---|---|
| Name | `site2site_Pfsense` |
| Connection type | Site-to-site (IPsec) |
| Local network gateway | `pfsense_home` |
| Shared key (PSK) | `YOUR-PSK` (long and random, same key goes into pfSense) |
| Enable BGP | unticked |
| IKE Protocol | IKEv2 |

OK. Click Refresh if the connection doesn't show right away.

Get the gateway's public IP for pfSense: `Azure_vnet` → Overview → Public IP address.

## 4. pfSense IPsec phase 1
VPN → IPsec → Tunnels → Add P1:

| Field | Value |
|---|---|
| Description | `Azure` |
| Key Exchange version | IKEv2 |
| Interface | WAN |
| Remote Gateway | `YOUR-AZURE-GATEWAY-IP` |
| Authentication Method | Mutual PSK |
| My identifier | My IP address |
| Peer identifier | Peer IP address |
| Pre-Shared Key | `YOUR-PSK` |
| Encryption Algorithm | AES, 256 bits, SHA256, DH Group 2 (1024 bit) |
| Life Time | 28800 |
| NAT Traversal | Auto |
| Dead Peer Detection | Enable DPD, Delay 10, Max failures 5 |

Save. In the video the tunnel did not come up with DH group 14, so group 2 was used. pfSense marks DH group 2 as weak on this page.

## 5. pfSense IPsec phase 2
Show Phase 2 Entries → Add P2:

| Field | Value |
|---|---|
| Description | `AzureP2` |
| Mode | Tunnel IPv4 |
| Local Network | your LAN/VLAN subnet (`SERVERS subnet` in the video) |
| Remote Network | Network, address `10.90.0.0`, mask `20` |
| Protocol | ESP |
| Encryption Algorithms | AES 256 bits, AES128-GCM 128 bits |
| Hash Algorithms | SHA256 |
| PFS key group | 14 (2048 bit) |
| Life Time | 3600 |

Type only the address in the Address box and pick the mask from the dropdown next to it. Typing `10.90.0.0/20` in the box gives "A valid remote network IP address must be specified".

Save, Apply Changes.

## 6. Connect the tunnel
Status → IPsec → Connect P1 and P2s. It shows **Established**.

## 7. Ping test and Azure VM firewall rule
From the home workstation:
```
ping 10.90.0.4
```
It fails until the Azure VM allows your home network. RDP to the VM (over its public IP for now), then Windows Defender Firewall with Advanced Security → Inbound Rules → New Rule:
- Rule Type = Custom
- Program = All programs
- Protocol and Ports = Any
- Scope → Remote IP address = `10.37.0.0/20`
- Action = Allow the connection
- Profile = all
- Name = `Pfsense_network`

Finish.

## 8. Fix: the wrong network card
If the PC has more than one network card, the ping can leave through a card that isn't on the home VLAN. Control Panel → Network Connections (Change adapter settings), then either:
- disable the other card, or
- on the other card: Properties → Internet Protocol Version 4 (TCP/IPv4) → Properties → Advanced → set a higher interface metric

The video disables it. Then `ping 10.90.0.4` answers.

## 9. Ping from Azure and the IPsec firewall rule
From the Azure VM, `ping 10.37.0.98` fails. Two things fix it.

pfSense: Firewall → Rules → IPsec → Add:

| Action | Protocol | Source | Destination | Description |
|---|---|---|---|---|
| Pass | Any | any | any | `S2S traffic` |

Save, Apply Changes.

Home PC: Windows Defender Firewall with Advanced Security → Inbound Rules → New Rule:
- Custom → All programs → Any
- Remote IP address = `10.90.0.0/20`
- Allow
- Name = `Azure_network`

Finish. The ping from Azure now answers.

## 10. RDP over the tunnel
Close the RDP session that used the public IP. Open Remote Desktop Connection again and connect to the private IP `10.90.0.4`. RDP works through the tunnel.

## Links
- Newer version with BGP (2026): https://youtu.be/0HYaSX_fvBE
- Same setup with AWS: https://youtu.be/p83RmeT2Q-A
- Tailscale on pfSense: https://youtu.be/HDFw4EVDZj8
