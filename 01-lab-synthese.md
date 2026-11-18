# Lab de Synthèse — CCNA 200-301

Ce laboratoire relie **les 5 domaines du CCNA** dans un seul scénario de bout en bout :
adressage, commutation, routage, services et sécurité. Chaque point est validé par des
commandes de vérification réelles.

- **Partie 1 — Routage** (montée sous **GNS3**) : OSPF, route par défaut, NAT/PAT
- **Partie 2 — Commutation** (montée sous **Packet Tracer**) : VLAN, trunk, inter-VLAN, STP, port-security
- **Partie 3 — Services** (Packet Tracer) : DHCP, DNS, NTP

---

## Partie 1 — Routage (GNS3)

### Topologie

```
                 200.0.0.0/30
   [ R1 ]---g0/1--------------( ISP )  8.8.8.8
     |  .1              .2
     | g0/0  10.0.12.0/30
     |  .1
     |  .2
   [ R2 ]
     | g0/1  10.0.23.0/30
     |  .1
     |  .2
   [ R3 ]---g0/1--- LAN 192.168.30.0/24 --- PC3 (.10)
              .1
```

### Plan d'adressage

| Lien / réseau | Préfixe | R1 | R2 | R3 |
|---|---|---|---|---|
| R1–ISP | 200.0.0.0/30 | .1 | — | — |
| R1–R2 | 10.0.12.0/30 | .1 | .2 | — |
| R2–R3 | 10.0.23.0/30 | — | .1 | .2 |
| LAN de R3 | 192.168.30.0/24 | — | — | .1 |

### Configuration R1 (routeur de bordure : OSPF + route par défaut + NAT)

```
hostname R1
!
interface g0/0
 ip address 10.0.12.1 255.255.255.252
 ip nat inside
 no shutdown
!
interface g0/1
 ip address 200.0.0.1 255.255.255.252
 ip nat outside
 no shutdown
!
! --- OSPF ---
router ospf 1
 network 10.0.12.0 0.0.0.3 area 0
 default-information originate
!
! --- Route par defaut vers l'ISP ---
ip route 0.0.0.0 0.0.0.0 200.0.0.2
!
! --- NAT / PAT (overload) ---
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface g0/1 overload
```

### Configuration R2 (routeur de transit)

```
hostname R2
!
interface g0/0
 ip address 10.0.12.2 255.255.255.252
 no shutdown
!
interface g0/1
 ip address 10.0.23.1 255.255.255.252
 no shutdown
!
router ospf 1
 network 10.0.12.0 0.0.0.3 area 0
 network 10.0.23.0 0.0.0.3 area 0
```

### Configuration R3 (routeur du LAN)

```
hostname R3
!
interface g0/0
 ip address 10.0.23.2 255.255.255.252
 no shutdown
!
interface g0/1
 ip address 192.168.30.1 255.255.255.0
 no shutdown
!
router ospf 1
 network 10.0.23.0 0.0.0.3 area 0
 network 192.168.30.0 0.0.0.255 area 0
```

### Vérifications

```
! Voisinage OSPF (doit afficher FULL)
R2# show ip ospf neighbor

! Table de routage : les routes O, et la defaut O*E2 sur R2/R3
R3# show ip route

! Connectivite complete depuis le LAN vers "Internet"
PC3> ping 8.8.8.8

! Preuve du NAT : l'IP privee traduite en IP publique
R1# show ip nat translations
```

**Attendu :** PC3 (192.168.30.10) atteint 8.8.8.8 ; `show ip nat translations` montre
`192.168.30.10` (inside local) traduit en `200.0.0.1` (inside global).

---

## Partie 2 — Commutation (Packet Tracer)

### Topologie (router-on-a-stick)

```
        trunk 802.1Q
 [ R1 ]============[ SW1 ]
  g0/0              |  |  \
                    |  |   \
                  PC1 SRV  PC2
                 VLAN10    VLAN20
```

### Plan d'adressage

| Élément | VLAN | Adresse |
|---|---|---|
| Passerelle VLAN 10 (g0/0.10) | 10 | 192.168.10.1/24 |
| Passerelle VLAN 20 (g0/0.20) | 20 | 192.168.20.1/24 |
| Serveur SRV | 10 | 192.168.10.3/24 |
| PC1 | 10 | DHCP |
| PC2 | 20 | DHCP |

### VLAN et trunk (sur SW1)

```
SW1(config)# vlan 10
SW1(config-vlan)#  name DATA
SW1(config)# vlan 20
SW1(config-vlan)#  name VOICE
!
! Ports d'acces
SW1(config)# interface f0/1
SW1(config-if)#  switchport mode access
SW1(config-if)#  switchport access vlan 10
SW1(config)# interface f0/3
SW1(config-if)#  switchport mode access
SW1(config-if)#  switchport access vlan 20
!
! Lien trunk vers le routeur
SW1(config)# interface g0/1
SW1(config-if)#  switchport mode trunk
```

### Inter-VLAN (router-on-a-stick sur R1)

```
R1(config)# interface g0/0
R1(config-if)#  no shutdown
!
R1(config)# interface g0/0.10
R1(config-subif)#  encapsulation dot1Q 10
R1(config-subif)#  ip address 192.168.10.1 255.255.255.0
!
R1(config)# interface g0/0.20
R1(config-subif)#  encapsulation dot1Q 20
R1(config-subif)#  ip address 192.168.20.1 255.255.255.0
```

### STP + PortFast + BPDU guard

```
SW1(config)# interface range f0/1 - 3
SW1(config-if-range)#  spanning-tree portfast
SW1(config-if-range)#  spanning-tree bpduguard enable
```

### Port-security (sticky MAC)

```
SW1(config)# interface range f0/1 - 3
SW1(config-if-range)#  switchport port-security
SW1(config-if-range)#  switchport port-security maximum 1
SW1(config-if-range)#  switchport port-security mac-address sticky
SW1(config-if-range)#  switchport port-security violation shutdown
```

### Vérifications

```
SW1# show vlan brief
SW1# show interfaces trunk
SW1# show spanning-tree vlan 10
SW1# show port-security interface f0/1
! Ping inter-VLAN : PC1 (VLAN10) doit joindre PC2 (VLAN20)
```

---

## Partie 3 — Services (Packet Tracer)

### DHCP — un pool par VLAN (sur R1)

```
R1(config)# ip dhcp excluded-address 192.168.10.1
R1(config)# ip dhcp excluded-address 192.168.10.3
R1(config)# ip dhcp excluded-address 192.168.20.1
!
R1(config)# ip dhcp pool VLAN10
R1(dhcp-config)#  network 192.168.10.0 255.255.255.0
R1(dhcp-config)#  default-router 192.168.10.1
R1(dhcp-config)#  dns-server 192.168.10.3
!
R1(config)# ip dhcp pool VLAN20
R1(dhcp-config)#  network 192.168.20.0 255.255.255.0
R1(dhcp-config)#  default-router 192.168.20.1
R1(dhcp-config)#  dns-server 192.168.10.3
```

Vérification : `R1# show ip dhcp binding`

### DNS — sur le serveur SRV

> ⚠️ Le routeur de Packet Tracer ne supporte pas `ip dns server`. Le DNS se configure
> donc sur le **serveur** (plus réaliste : le DNS tourne sur un serveur, pas sur un routeur).

```
SRV  (onglet Services -> DNS)
  DNS Service : On
  Name  : www.ccna.local
  Type  : A Record
  Address : 192.168.10.3
```

Vérification : depuis un PC → `ping www.ccna.local` (le nom doit se résoudre en 192.168.10.3).

### NTP — horloge commune (master / client)

```
! --- R1 : serveur de temps (master) ---
R1# clock set 10:00:00 18 Nov 2026
R1(config)# ntp master 3
!
! --- SW1 : IP de gestion + client NTP ---
SW1(config)# interface vlan 10
SW1(config-if)#  ip address 192.168.10.250 255.255.255.0
SW1(config-if)#  no shutdown
SW1(config)# ip default-gateway 192.168.10.1
SW1(config)# ntp server 192.168.10.1
```

Vérification :

```
SW1# show ntp status          ! doit afficher "Clock is synchronized, stratum 4"
SW1# show ntp associations     ! un * devant 192.168.10.1 = source active
```

---

## Pièges rencontrés (journal de dépannage)

| Symptôme | Cause | Correction |
|---|---|---|
| Traceroute avec timeouts intermittents | Double route par défaut (dont une vers soi-même) | `no ip route 0.0.0.0 0.0.0.0 <mauvais-next-hop>` |
| `% Invalid next hop address (it's this router)` | Next-hop = IP d'une interface locale | Corriger l'IP / le bon routeur |
| Voisins OSPF absents (lien up) | Masques d'interface différents | Aligner le masque (ex. /30 des deux côtés) |
| Plus aucun PC n'a d'IP après DHCP snooping | Port du serveur resté untrusted | `ip dhcp snooping trust` sur le port du serveur |
| DNS "could not find host" | Enregistrement A pointant vers une IP inexistante | Pointer le A record vers la vraie IP du serveur |
| Un VLAN sort sur Internet, pas l'autre | ACL du NAT incomplète | Ajouter le sous-réseau manquant à l'ACL |

---

*Lab réalisé dans le cadre de la préparation CCNA 200-301 v2.0.*
