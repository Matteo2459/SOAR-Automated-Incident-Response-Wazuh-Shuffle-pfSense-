# SOAR & Automated Incident Response (Wazuh + Shuffle + pfSense)

In questo progetto ho progettato e implementato una pipeline di Security Orchestration, Automation, and Response (SOAR) integrando il SIEM Wazuh, il SOAR open-source Shuffle e il firewall pfSense.

L'obiettivo è automatizzare il ciclo di vita dell'Incident Response per attacchi di rete, riducendo a zero tempo di risposta attraverso la verifica della reputazione della minaccia in tempo reale e il blocco automatico dell'IP malevolo a livello di perimetro.

## Architettura e Flusso Logico dell'Automazione

L'infrastruttura di automazione esegue una catena di risposta dinamica suddivisa in 5 fasi sequenziali:

1. **Trigger & Data Ingestion (Wazuh Manager):** Al rilevamento di un attacco di rete (es. Brute Force SSH/HTTP), Wazuh attiva un'integrazione Webhook che invia il payload JSON grezzo dell'alert a Shuffle via  HTTP POST sulla porta 3001.
2. **Data Parsing & Extraction (Extract IP - Regex):** Un nodo Python personalizzato processa il campo `alert_raw` per estrarre con precisione l'indirizzo IP dell'attaccante (`attacker_ip`) tramite espressioni regolari (Regex).
3. **Threat Intelligence & Enrichment (VirusTotal IP Check):** L'IP estratto viene inviato in tempo reale alle API v3 di VirusTotal per verificare la reputazione dell'IP, collezionando score.
4. **Automated Containment & Blocking (pfSense Block IP - SSH):** Tramite una sessione SSH sicura, Shuffle si collega al firewall pfSense ed esegue l'inserimento dell'IP malevolo all'interno della tabella di blocco per interrompere l'attacco al perimetro.
5. **SOC Notification & Alerting (Send SOC Alert Email):** Se il punteggio dell' ip supera la soglia di sicurezza imposta nella condizione (`Condition > 0`), viene generata ed inviata una mail di reportistica per informare il team SOC dell'avvenuto contenimento.

## Componenti del Workflow SOAR (Shuffle)

Il workflow visivo sviluppato su Shuffle si compone dei seguenti nodi operativi:

<img width="1047" height="439" alt="Screenshot 2026-08-11 152849" src="https://github.com/user-attachments/assets/016260d9-c2dd-4437-9bd1-85920820ef88" />


* **Wazuh Alert Webhook:** Endpoint HTTP in ascolto per la ricezione dei log dal SIEM.
* **Extract IP (Regex):** Script Python per il parsing strutturato del JSON grezzo.
* **VirusTotal IP Check:** Modulo API per la Threat Intelligence.
* **pfSense Block IP (SSH):** Agent di attuazione per l'esecuzione dei comandi di mitigazione sul firewall.
* **Send SOC Alert Email:** Modulo di notifica automatica verso gli analisti della sicurezza.

## Test Pratico

Per validare l'intero ciclo di automazione, è stato simulato un attacco d'origine esterna inviando un payload di prova dal server Wazuh verso  Shuffle.

### 1. Simulazione dell'Evento dal Terminale Wazuh
Ho simulato l'invio dell'allert inviando la richiesta HTTP POST con il payload di test:

`curl -X POST "http://192.168.1.139:3001/api/v1/hooks/webhook_068e80e2-6c5a-49b2-b340-d6a985526dd9" -H "Content-Type: application/json" -d '{"alert_raw": "Attack detected from IP 185.220.101.5 on ssh service"}'`

### 2. Risultati dell'Esecuzione

* **Integrazione Webhook:** Risposta HTTP 200 ricevuta istantaneamente dal server Shuffle con generazione univoca dell'ID di esecuzione (`execution_id: ce8197ea-0991-4db3-84a4-960c7fc8888d`).

<br>

<img width="476" height="531" alt="Screenshot 2026-08-11 142719" src="https://github.com/user-attachments/assets/8d04c675-9931-4af9-9c4d-7a78240472c7" />


<br>
<br>

* **Parsing & Extraction (Shuffle Tools):** L'IP `185.220.101.5` viene correttamente estratto dallo script Python a partire dal messaggio grezzo inviato dal Webhook.

<br>

<img width="466" height="432" alt="Screenshot 2026-08-11 142731" src="https://github.com/user-attachments/assets/30030892-56c3-4a11-bc07-1f03d9f8492f" />



<br>
<br>

* **Enrichment & Threat Intelligence (VirusTotal):** L'IP estratto viene inviato via API v3 a VirusTotal, ricevendo risposta HTTP 200 con lo stato ed i dettagli di reputazione della risorsa (`id: 185.220.101.5`).

<br>

<img width="451" height="778" alt="Screenshot 2026-08-11 142743" src="https://github.com/user-attachments/assets/6141f5a4-f534-416a-8426-28fda25d73cb" />



<br>
<br>

* **Mitigazione sul Firewall:** Il nodo SSH stabilisce la connessione con pfSense, aggiungendo l'IP malevolo alla tabella `virus_total_blacklist` ed applicando il blocco sul perimetro.

<br>
<img width="1169" height="427" alt="Screenshot 2026-08-11 161147" src="https://github.com/user-attachments/assets/12e27132-4b35-45c0-ad65-65dfe63b6a14" />

