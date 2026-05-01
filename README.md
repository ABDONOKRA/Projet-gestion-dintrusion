# 🛡️ Guide Complet — Analyse Réseau et Détection avec Zeek

---
 
## 📌 Vue d'ensemble du projet
 
| Élément | Détail |
|---|---|
| **Objectif** | Produire et analyser des logs réseau pour détecter scans et activités suspectes |
| **Livrables** | Rapport, scénarios de test, journaux analysés |
| **Durée estimée** | 4 à 6 sessions de travail |
 
---
 
## 🗺️ Plan des étapes
 
```
PHASE 1 — Préparation de l'environnement
  └── Étape 1 : Comprendre l'architecture du projet
  └── Étape 2 : Installer et configurer Zeek sur Linux
  └── Étape 3 : Vérifier l'installation et découvrir l'interface
 
PHASE 2 — Capture et analyse du trafic normal
  └── Étape 4 : Capturer du trafic réseau avec Wireshark
  └── Étape 5 : Lancer Zeek sur une interface réseau
  └── Étape 6 : Explorer les logs générés par Zeek
 
PHASE 3 — Simulation d'attaques
  └── Étape 7 : Simuler un scan de ports (Nmap)
  └── Étape 8 : Simuler d'autres activités suspectes
 
PHASE 4 — Détection et analyse
  └── Étape 9 : Analyser les logs Zeek pour détecter les attaques
  └── Étape 10 : Écrire des scripts de détection personnalisés (Zeek Scripting)
 
PHASE 5 — Rédaction des livrables
  └── Étape 11 : Rédiger les scénarios de test
  └── Étape 12 : Rédiger le rapport final d'analyse
```
 
---
 
## PHASE 1 — Préparation de l'environnement
 
---
 
### ✅ Étape 1 — Comprendre l'architecture du projet
 
**Objectif :** Comprendre comment les outils s'articulent ensemble avant de toucher quoi que ce soit.
 
#### Schéma de l'architecture
 
```
┌─────────────────────────────────────────────────────┐
│                  Machine Linux                       │
│                                                     │
│  ┌──────────┐    ┌──────────┐    ┌───────────────┐  │
│  │ Wireshark│    │   Zeek   │    │  Logs générés │  │
│  │(capture  │───▶│(analyse  │───▶│  conn.log     │  │
│  │ .pcap)   │    │ trafic)  │    │  dns.log      │  │
│  └──────────┘    └──────────┘    │  http.log     │  │
│                                  │  notice.log   │  │
│  ┌──────────┐                    └───────────────┘  │
│  │  Nmap    │──── Génère du trafic suspect ──────▶  │
│  │(attaquant│                                       │
│  │simulé)   │                                       │
│  └──────────┘                                       │
└─────────────────────────────────────────────────────┘
```
 
#### Rôle de chaque outil
 
| Outil | Rôle | Analogie |
|---|---|---|
| **Wireshark** | Capturer les paquets réseau bruts | Enregistreur de caméra de surveillance |
| **Zeek** | Analyser le trafic et générer des logs intelligents | Analyste de sécurité qui regarde les enregistrements |
| **Nmap** | Simuler un attaquant qui scanne le réseau | Le "cambrioleur" qu'on veut détecter |
 
#### Concepts clés à comprendre
 
- **Paquet réseau** : Unité de données qui circule sur le réseau (comme une lettre avec une enveloppe)
- **Log** : Fichier journal qui enregistre les événements (qui a communiqué avec qui, quand, comment)
- **Scan de ports** : Technique utilisée par les attaquants pour découvrir quels services sont ouverts sur une machine
- **IDS (Intrusion Detection System)** : Système qui détecte les intrusions — Zeek peut jouer ce rôle
---
 
### ✅ Étape 2 — Installer et configurer Zeek sur Linux
 
**Objectif :** Avoir un environnement fonctionnel.
 
#### Prérequis système
 
- Distribution Linux recommandée : **Ubuntu 22.04 LTS** ou **Kali Linux**
- RAM minimum : 4 GB
- Droits sudo/root
#### Installation de Zeek
 
```bash
# Méthode 1 : Via les dépôts officiels Zeek (recommandé)
 
# Ajouter le dépôt Zeek
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' \
  | sudo tee /etc/apt/sources.list.d/security:zeek.list
 
# Importer la clé GPG
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key \
  | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
 
# Mettre à jour et installer
sudo apt update
sudo apt install zeek -y
```
 
```bash
# Installation de Wireshark et Nmap
sudo apt install wireshark nmap -y
```
 
#### Configurer le PATH pour Zeek
 
```bash
# Ajouter Zeek au PATH
echo 'export PATH=/opt/zeek/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```
 
#### Vérifier les installations
 
```bash
zeek --version
wireshark --version
nmap --version
```
 
**📌 Note :** Si tu utilises une VM (VirtualBox/VMware), assure-toi que ta carte réseau est en mode **"Bridge"** ou **"Host-Only"** pour voir du vrai trafic.
 
---
 
### ✅ Étape 3 — Découvrir l'interface et la structure de Zeek
 
**Objectif :** Comprendre où sont les fichiers importants avant de commencer.
 
#### Structure de Zeek
 
```
/opt/zeek/
├── bin/
│   ├── zeek          ← Binaire principal
│   └── zeekctl       ← Outil de contrôle (démarrer/arrêter)
├── etc/
│   ├── node.cfg      ← Configuration des nœuds
│   ├── networks.cfg  ← Définir ton réseau local
│   └── zeekctl.cfg   ← Configuration générale
├── logs/
│   └── current/      ← Logs en temps réel
│       ├── conn.log   ← Toutes les connexions
│       ├── dns.log    ← Requêtes DNS
│       ├── http.log   ← Trafic HTTP
│       └── notice.log ← Alertes et détections
└── share/zeek/
    └── site/
        └── local.zeek ← Scripts personnalisés (on l'utilisera plus tard)
```
 
#### Commandes de base à retenir
 
```bash
# Démarrer Zeek en mode standalone
sudo zeekctl deploy
 
# Voir l'état
sudo zeekctl status
 
# Arrêter Zeek
sudo zeekctl stop
 
# Analyser un fichier pcap offline (sans carte réseau)
zeek -r fichier.pcap
 
# Lancer Zeek sur une interface en temps réel
sudo zeek -i eth0
```
 
---
 
## PHASE 2 — Capture et analyse du trafic normal
 
---
 
### ✅ Étape 4 — Capturer du trafic réseau avec Wireshark
 
**Objectif :** Obtenir un fichier `.pcap` avec du trafic réseau pour l'analyser.
 
#### Méthode A : Capturer en direct
 
```bash
# Lister les interfaces disponibles
ip link show
 
# Capturer avec tcpdump (alternative légère à Wireshark)
sudo tcpdump -i eth0 -w capture_normale.pcap -c 1000
# -i eth0 : interface réseau
# -w : écrire dans un fichier
# -c 1000 : capturer 1000 paquets puis s'arrêter
```
 
#### Méthode B : Utiliser Wireshark (interface graphique)
 
1. Ouvrir Wireshark
2. Sélectionner l'interface réseau (eth0, wlan0...)
3. Cliquer sur **Start Capture** (bouton bleu)
4. Naviguer sur internet pendant 2-3 minutes pour générer du trafic
5. Cliquer sur **Stop** puis **File > Save As > capture_normale.pcap**
#### Vérifier la capture
 
```bash
# Voir les statistiques du fichier pcap
capinfos capture_normale.pcap
 
# Lister les premières lignes
tcpdump -r capture_normale.pcap | head -20
```
 
---
 
### ✅ Étape 5 — Lancer Zeek sur la capture
 
**Objectif :** Générer les premiers logs Zeek à partir de la capture.
 
```bash
# Créer un dossier de travail
mkdir ~/zeek_projet && cd ~/zeek_projet
 
# Analyser le fichier pcap avec Zeek
zeek -r ~/capture_normale.pcap
 
# Lister les logs générés
ls -la
```
 
Tu devrais voir apparaître plusieurs fichiers `.log`.
 
---
 
### ✅ Étape 6 — Explorer les logs générés
 
**Objectif :** Comprendre la structure des logs Zeek.
 
#### Le fichier conn.log (le plus important)
 
```bash
# Voir les colonnes disponibles
head -1 conn.log
 
# Afficher lisiblement (avec zeek-cut)
cat conn.log | zeek-cut ts id.orig_h id.orig_p id.resp_h id.resp_p proto duration
 
# Compter les connexions par IP source
cat conn.log | zeek-cut id.orig_h | sort | uniq -c | sort -rn | head -10
```
 
#### Structure d'une ligne conn.log
 
| Champ | Description | Exemple |
|---|---|---|
| `ts` | Timestamp (horodatage) | 1620000000.000 |
| `id.orig_h` | IP source (qui initie) | 192.168.1.10 |
| `id.orig_p` | Port source | 54321 |
| `id.resp_h` | IP destination | 8.8.8.8 |
| `id.resp_p` | Port destination | 443 |
| `proto` | Protocole | tcp / udp |
| `duration` | Durée de la connexion | 0.5 |
| `conn_state` | État de la connexion | S1, SF, REJ... |
 
#### États de connexion importants
 
| État | Signification | Suspicion ? |
|---|---|---|
| `SF` | Connexion établie et fermée normalement | Non |
| `S0` | SYN envoyé, pas de réponse | ⚠️ Possible scan |
| `REJ` | Connexion rejetée | ⚠️ Port fermé |
| `RSTO` | Reset de la connexion | ⚠️ Comportement anormal |
 
---
 
## PHASE 3 — Simulation d'attaques
 
---
 
### ✅ Étape 7 — Simuler un scan de ports avec Nmap
 
**Objectif :** Créer du trafic "malveillant" que Zeek devra détecter.
 
> ⚠️ **IMPORTANT — Légalité :** Ne scanner QUE des machines qui t'appartiennent ou sur lesquelles tu as une autorisation écrite. Pour ce projet, utilise uniquement `localhost` (127.0.0.1) ou des VMs de ton lab.
 
#### Préparer Zeek pour capturer l'attaque
 
```bash
# Dans un terminal 1 : démarrer la capture Zeek
cd ~/zeek_projet
sudo zeek -i lo  # lo = interface loopback (localhost)
```
 
#### Lancer les scans dans un autre terminal
 
```bash
# Terminal 2 : Scan SYN (le plus courant)
sudo nmap -sS 127.0.0.1
 
# Scan de tous les ports
sudo nmap -sS -p- 127.0.0.1
 
# Scan agressif (OS detection, version, scripts)
sudo nmap -A 127.0.0.1
 
# Scan UDP
sudo nmap -sU 127.0.0.1
```
 
#### Alternative : Capturer avec tcpdump puis analyser
 
```bash
# Capturer pendant le scan
sudo tcpdump -i lo -w scan_nmap.pcap &
sudo nmap -sS 127.0.0.1
sudo pkill tcpdump
 
# Analyser avec Zeek
mkdir ~/zeek_projet/analyse_scan && cd ~/zeek_projet/analyse_scan
zeek -r ~/scan_nmap.pcap
```
 
---
 
### ✅ Étape 8 — Simuler d'autres activités suspectes
 
**Objectif :** Générer différents types de trafic suspect.
 
#### Scénario 1 : Beaucoup de connexions rapides (DoS léger)
 
```bash
# Générer des connexions répétées rapidement
for i in {1..100}; do curl -s http://localhost > /dev/null 2>&1; done
```
 
#### Scénario 2 : Requêtes DNS suspectes
 
```bash
# Requêtes vers des domaines inexistants (exfiltration DNS simulée)
for domain in test1 test2 test3 random1 random2; do
  nslookup $domain.suspicious-domain.example 2>/dev/null
done
```
 
#### Scénario 3 : Scan avec délai (scan lent, difficile à détecter)
 
```bash
# Scan lent pour éviter la détection
sudo nmap -sS --scan-delay 1s 127.0.0.1
```
 
---
 
## PHASE 4 — Détection et analyse
 
---
 
### ✅ Étape 9 — Analyser les logs pour détecter les attaques
 
**Objectif :** Identifier les traces d'attaque dans les logs Zeek.
 
#### Détecter un scan de ports dans conn.log
 
```bash
# Un scan génère beaucoup de connexions S0 (pas de réponse)
cat conn.log | zeek-cut id.orig_h id.resp_p conn_state | grep "S0" | wc -l
 
# Si le nombre est > 100, c'est très suspect !
 
# Identifier l'IP qui scan
cat conn.log | zeek-cut id.orig_h conn_state | grep "S0" | \
  awk '{print $1}' | sort | uniq -c | sort -rn
```
 
#### Script bash d'analyse rapide
 
```bash
#!/bin/bash
# analyse_scan.sh — Script de détection simple
 
echo "=== ANALYSE DES LOGS ZEEK ==="
echo ""
 
echo "--- Top 10 des IPs sources ---"
cat conn.log | zeek-cut id.orig_h | sort | uniq -c | sort -rn | head -10
 
echo ""
echo "--- Connexions rejetées (REJ) par IP ---"
cat conn.log | zeek-cut id.orig_h conn_state | grep "REJ" | \
  awk '{print $1}' | sort | uniq -c | sort -rn | head -10
 
echo ""
echo "--- Connexions sans réponse (S0) par IP ---"
cat conn.log | zeek-cut id.orig_h conn_state | grep "S0" | \
  awk '{print $1}' | sort | uniq -c | sort -rn | head -10
 
echo ""
echo "--- Ports les plus ciblés ---"
cat conn.log | zeek-cut id.resp_p conn_state | grep -E "S0|REJ" | \
  awk '{print $1}' | sort | uniq -c | sort -rn | head -20
```
 
#### Vérifier les alertes dans notice.log
 
```bash
# Zeek génère automatiquement des alertes
cat notice.log | zeek-cut ts note msg
 
# Types d'alertes courants
# Scan::Port_Scan         ← Scan de ports détecté
# Scan::Address_Scan      ← Scan d'adresses
# DNS::External_Name      ← DNS suspect
```
 
---
 
### ✅ Étape 10 — Écrire un script de détection Zeek (Zeek Scripting)
 
**Objectif :** Créer une règle de détection personnalisée.
 
#### Script de base : détecter les scans de ports
 
```zeek
# /opt/zeek/share/zeek/site/detect_scan.zeek
 
@load base/frameworks/notice
 
module PortScan;
 
export {
    redef enum Notice::Type += {
        PortScan::Detected
    };
    
    # Seuil : plus de 20 ports différents en moins de 5 secondes
    const scan_threshold = 20 &redef;
    const scan_interval = 5sec &redef;
}
 
# Table pour compter les ports tentés par chaque IP
global ports_tried: table[addr] of set[port] &create_expire=1min;
 
event connection_attempt(c: connection)
{
    local src = c$id$orig_h;
    local dst_port = c$id$resp_p;
    
    if (src !in ports_tried)
        ports_tried[src] = set();
    
    add ports_tried[src][dst_port];
    
    if (|ports_tried[src]| >= scan_threshold)
    {
        NOTICE([
            $note = PortScan::Detected,
            $src = src,
            $msg = fmt("Scan de ports détecté depuis %s (%d ports tentés)", 
                       src, |ports_tried[src]|),
            $identifier = cat(src)
        ]);
    }
}
```
 
#### Charger et tester le script
 
```bash
# Ajouter au fichier de configuration local
echo "@load site/detect_scan" >> /opt/zeek/share/zeek/site/local.zeek
 
# Tester sur le fichier pcap
zeek -r scan_nmap.pcap /opt/zeek/share/zeek/site/detect_scan.zeek
 
# Vérifier les notices générées
cat notice.log
```
 
---
 
## PHASE 5 — Rédaction des livrables
 
---
 
### ✅ Étape 11 — Rédiger les scénarios de test
 
**Structure d'un scénario de test :**
 
```
SCÉNARIO #1 : Scan SYN avec Nmap
─────────────────────────────────
Date/Heure    : [à compléter]
Outil utilisé : Nmap
Commande      : sudo nmap -sS 127.0.0.1
Cible         : 127.0.0.1 (localhost)
Objectif      : Détecter un scan de ports dans conn.log
 
Résultats attendus :
  - conn.log : nombreuses connexions état "S0" ou "REJ"
  - notice.log : alerte Scan::Port_Scan
 
Résultats obtenus :
  [à compléter après les tests]
 
Conclusion :
  [Zeek a-t-il détecté l'attaque ? Avec quel délai ?]
```
 
---
 
### ✅ Étape 12 — Rédiger le rapport final
 
**Structure du rapport :**
 
```
1. Introduction
   - Contexte et objectifs
   - Outils utilisés et leurs rôles
 
2. Environnement de test
   - Configuration du lab
   - Topologie réseau
 
3. Méthodologie
   - Comment les captures ont été effectuées
   - Comment Zeek a été configuré
 
4. Scénarios de test et résultats
   - Scénario 1 : Scan de ports SYN
   - Scénario 2 : [autre scénario]
   - Scénario 3 : [autre scénario]
 
5. Analyse des logs
   - Logs conn.log : analyse des connexions suspectes
   - Logs notice.log : alertes générées
   - Captures d'écran des preuves
 
6. Scripts de détection
   - Description du script Zeek créé
   - Tests et validation
 
7. Conclusion
   - Efficacité de Zeek
   - Limites identifiées
   - Recommandations
 
8. Annexes
   - Logs bruts
   - Fichiers pcap
   - Scripts complets
```
 
---
 
## 📊 Checklist de progression
 
| Étape | Description | Statut |
|---|---|---|
| Étape 1 | Comprendre l'architecture | ⬜ |
| Étape 2 | Installer Zeek, Wireshark, Nmap | ⬜ |
| Étape 3 | Explorer la structure de Zeek | ⬜ |
| Étape 4 | Première capture réseau | ⬜ |
| Étape 5 | Premier lancement de Zeek | ⬜ |
| Étape 6 | Analyser conn.log, dns.log | ⬜ |
| Étape 7 | Simuler un scan Nmap | ⬜ |
| Étape 8 | Simuler d'autres attaques | ⬜ |
| Étape 9 | Détecter les traces d'attaque | ⬜ |
| Étape 10 | Écrire un script Zeek | ⬜ |
| Étape 11 | Rédiger les scénarios de test | ⬜ |
| Étape 12 | Rédiger le rapport final | ⬜ |
 
---
 
## 🔑 Ressources utiles
 
- **Documentation officielle Zeek :** https://docs.zeek.org
- **Zeek Scripting Reference :** https://docs.zeek.org/en/master/script-reference/
- **Logs de référence :** https://docs.zeek.org/en/master/logs/
- **Nmap guide :** https://nmap.org/book/
---
 
*Guide rédigé pour le cours de Gestion d'Intrusion — À utiliser uniquement dans un environnement de lab autorisé.*
















-------------------------------------------------------------------------------------------

### 🔍 Analyse des ports ciblés

| Port | Connexions S0 | Service |
| :--- | :--- | :--- |
| `5353` | 894 | mDNS — découverte Apple/Android |
| `1900` | 557 | UPnP — découverte appareils réseau |
| `53` | 175 | DNS |
| `995` | 138 | POP3S — email chiffré |
| `256` | 368 | Route Access Protocol |

Ces ports sont typiques d'un scan réseau qui cherche à **cartographier tous les services** disponibles. ✅
