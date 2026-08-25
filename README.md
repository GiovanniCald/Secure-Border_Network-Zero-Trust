# Secure-Border Network Architecture
### Progettazione di un'infrastruttura Enterprise multisede con routing ibrido OSPF/eBGP, indirizzamento predittivo VLSM e micro-segmentazione difensiva.

---

## 📊 Project Metadata & KPI
* **Ruolo:** Network & Security Architect
* **Tempo di Sviluppo:** ~7 Ore (Pianificazione, CLI & Validazione)
* **Ambiente di Simulazione:** Cisco Packet Tracer
* **Livello di Complessità:** CCNA Advanced / CCNP Enterprise Core
* **Tasso di Successo (SLA):** 100% di pacchetti consegnati (0% loss) sulle tratte autorizzate.

---

## 1. Introduzione e Obiettivi del Progetto
Il presente progetto descrive l'ingegnerizzazione e l'implementazione di un'infrastruttura di rete aziendale *Enterprise* distribuita geograficamente. L'architettura è stata sviluppata per soddisfare i requisiti di scalabilità, alta affidabilità e sicurezza interna richiesti dai moderni scenari industriali. 

Il sistema interconnette una Sede Centrale (**HQ**), deputata alla gestione dei servizi interni e del server aziendale, e una Sede Periferica (**Branch**), destinata ai reparti distaccati. Il perimetro aziendale si interfaccia inoltre con un fornitore di servizi Internet esterno (**ISP**). La progettazione adotta il paradigma **Zero-Trust**, isolando i segmenti produttivi critici e minimizzando i domini di broadcast.

---

## 2. Piano di Indirizzamento e Algoritmo VLSM
Per ottimizzare lo spazio di indirizzamento IP (RFC 1918) ed evitare lo spreco di host, è stato applicato l'algoritmo **VLSM (Variable Length Subnet Mask)**.

### 2.1 Sede Centrale (HQ) - Blocco Base: `192.168.1.0/24`
* **VLAN 10 (Amministrazione HQ):** Subnet `192.168.1.0/25` (Gateway: `.1`)
* **VLAN 20 (Produzione HQ):** Subnet `192.168.1.128/26` (Gateway: `.129`)
* **VLAN 30 (Server Farm HQ):** Subnet `192.168.1.192/29` (Gateway: `.193`) | Ospita il **Server Aziendale** su IP statico **`192.168.1.194`**.
* **Link di Transito Interno (HQ-CORE <-> HQ-ROUTER):** Subnet punto-punto `192.168.1.200/30`.

### 2.2 Sede Periferica (Branch) - Blocco Base: `192.168.2.0/24`
* **Interfaccia Gig0/1 (Amministrazione Branch):** Subnet `192.168.2.0/26` (Gateway: `.1`)
* **Interfaccia Gig0/2 (Produzione Branch):** Subnet `192.168.2.64/26` (Gateway: `.65`)

### 2.3 Dorsale WAN Geografica
* **Link Punto-Punto Diretto Layer 3:** Subnet `192.168.2.192/30` (HQ Gig0/0: `.193` | Branch Gig0/0: `.194`).

---

## 3. Architettura di Rete e Flussi di Traffico
* **Core LAN (HQ):** La commutazione inter-VLAN è affidata allo switch Multilayer `HQ-CORE`. La porta fisica `GigabitEthernet 0/1` è stata convertita in **Routed Port Layer 3 pura** (`no switchport`), abbattendo i tempi di convergenza ed escludendo l'overhead logico di STP.
* **Routing Dinamico (OSPFv2):** L'instradamento è affidato al protocollo **OSPFv2** (Process ID 1, Area 0 Backbone). L'adiacenza OSPF tra i due router di confine (`HQ-ROUTER` e `BORDER-ROUT`) raggiunge stabilmente lo stato **`FULL`** sul link fisico diretto, scambiando le rotte dinamicamente.
* **Border Gateway (eBGP e NAT):** Il router `HQ-ROUTER` gestisce il perimetro. Tramite una sessione eBGP attiva con l'ISP (AS privato 65001 vs AS pubblico 100) apprende la rotta di default (`0.0.0.0/0`), poi iniettata in OSPF via `default-information originate`. La navigazione internet è garantita dal NAT/PAT Overload attivo sulla porta pubblica.

---

## 4. Implementazione delle Politiche di Sicurezza (ACL)
* **Sicurezza LAN (HQ):** Sullo switch `HQ-CORE` è applicata l'**ACL Estesa 110** all'interfaccia `Vlan20` in ingresso (`in`). La lista isola completamente il reparto Produzione, vietando l'accesso verso la VLAN 10 e l'IP del Server Aziendale (`192.168.1.194`).
* **Sicurezza WAN (Branch):** Sul router `BORDER-ROUT` è configurata un'ACL Estesa 110 in ingresso (`in`) sulla porta fisica della Produzione (`Gig0/2`). Blocca i tentativi della Produzione Branch di raggiungere l'Amministrazione locale e il Server centrale. I pacchetti non autorizzati vengono distrutti alla sorgente, ottimizzando le performance e risparmiando banda sulla WAN geografica.

---

## 5. Servizi di Infrastruttura (DHCP Pools)
La Sede Periferica gestisce in autonomia l'assegnazione degli IP. Il router `BORDER-ROUT` agisce come server DHCP locale configurando due pool di indirizzamento dedicati:
* **Pool Amministrazione:** Nome `DHCP_AMMI_BRANCH` (Range `.2 - .62`)
* **Pool Produzione:** Nome `DHCP_PROD_BRANCH` (Range `.66 - .126`)
* Gli IP dei Gateway (`.1` e `.65`) sono inseriti nel comando `ip dhcp excluded-address` per prevenire conflitti.

---

## 6. Verifica e Collaudo (Troubleshooting)
L'efficacia e la convergenza del sistema sono state verificate tramite test diagnostici sulla CLI:
* `show ip ospf neighbor`: Certifica l'adiacenza OSPF nello stato `FULL` su tutte le tratte.
* `show ip route`: Evidenzia la rotta di default esterna contrassegnata dal flag **`O*E2`** su `HQ-CORE`.
* `show ip dhcp binding`: Conferma l'erogazione automatica e pulita degli IP ai PC della filiale.
* **Ping Test:** I PC dell'Amministrazione raggiungono il Server con lo 0% di pacchetti persi. I PC della Produzione vengono bloccati con stringa `Destination host unreachable`.
