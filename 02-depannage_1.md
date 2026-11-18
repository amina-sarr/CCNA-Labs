# Fiche de dépannage — CCNA 200-301

5 pannes types, chacune développée : **symptôme → cause → commande → réflexe**.
Le réflexe clé du dépannage : **isoler la couche fautive avant de corriger** — chaque
test (ping, `show`) élimine une hypothèse.

---

## Panne 1 — SSH refusé (Telnet passe)

**Symptôme :** impossible de se connecter en SSH à R1, alors que Telnet fonctionne.

**Cause :** les lignes VTY n'autorisent que Telnet (`transport input telnet`). SSH est bloqué à l'entrée.

**Correction :**
```
R1(config)# line vty 0 4
R1(config-line)#  transport input ssh
```

**Réflexe :** SSH « bien configuré mais refusé » → 99 % du temps, le coupable est sur la ligne VTY
(`transport input` ou `login` manquant).

---

## Panne 2 — Plus aucun PC n'obtient d'IP (DHCP snooping)

**Symptôme :** après activation de DHCP snooping, plus personne ne reçoit d'adresse.

**Cause :** le port du vrai serveur DHCP est resté *untrusted* : ses réponses OFFER/ACK sont jetées.

**Correction :**
```
SW1(config)# interface gigabitEthernet0/1
SW1(config-if)#  ip dhcp snooping trust
```

**Réflexe :** les ports PC restent *untrusted* (c'est voulu). Oublier de *truster* le port du
serveur = couper tout le réseau.

---

## Panne 3 — Un port qui tombe tout seul (port-security)

**Symptôme :** un utilisateur remplace son PC par un portable ; son port passe en err-disabled, plus de réseau.

**Cause :** la nouvelle MAC ≠ la MAC *sticky* enregistrée (maximum = 1) → violation → Secure-shutdown.

**Correction :**
```
SW1(config)# interface fa0/2
SW1(config-if)#  no switchport port-security mac-address sticky <ancienne-mac>
SW1(config-if)#  shutdown
SW1(config-if)#  no shutdown
```

**Réflexe :** err-disabled → toujours `shutdown` / `no shutdown`. Si la MAC autorisée a changé,
nettoyer **aussi** la vieille entrée sticky, sinon le port retombe aussitôt.

---

## Panne 4 — Internet pour certains, pas pour d'autres (NAT)

**Symptôme :** le PC du VLAN 20 atteint Internet, celui du VLAN 10 non. Pings inter-VLAN et passerelles OK.

**Cause :** l'ACL du NAT n'autorise que `192.168.20.0` : le réseau 10 n'est jamais traduit
(absent de `show ip nat translations`).

**Correction :**
```
R1(config)# access-list 1 permit 192.168.10.0 0.0.0.255
```

**Réflexe :** un réseau sort, l'autre pas → presque toujours l'ACL du NAT qui a oublié un
sous-réseau. Inutile de refaire `overload` ni les interfaces inside/outside si un VLAN marche déjà.

---

## Panne 5 — Deux routeurs OSPF qui s'ignorent

**Symptôme :** R1 et R2 directement connectés, lien up/up, même area 0, mais aucun voisin OSPF.

**Cause :** masques d'**interface** différents (R1 /30, R2 /24). Sur un lien broadcast, les masques
doivent correspondre, sinon les hellos sont rejetés.

**Correction :**
```
R2(config)# interface g0/0
R2(config-if)#  ip address 10.0.12.2 255.255.255.252
R2# clear ip ospf process          ! si l'adjacence tarde
```

**Réflexe :** voisins OSPF absents → comparer **masque d'interface, area et timers**. Le masque qui
compte pour l'adjacence est celui de l'interface (`ip address`), pas le wildcard du `network`.

---

## Les 4 réflexes universels

1. **Isoler la couche AVANT de corriger** : chaque `ping` / `show` élimine une hypothèse (VLAN ? routage ? NAT ? service ?).
2. **err-disabled** → `shutdown` / `no shutdown` (et nettoyer la cause : MAC sticky, BPDU reçue…).
3. **« Un réseau sort, l'autre pas »** → ACL du NAT. **« SSH refusé »** → ligne VTY. **« Pas d'IP »** → port DHCP *trust*.
4. **Voisins OSPF absents** → masque d'interface, area, timers doivent correspondre.

---

*Fiche réalisée dans le cadre de la préparation CCNA 200-301 v2.0 — Amina Sarr.*
