# Shelly-EM- Energy Monitor con Telegraf-Influx-Grafana. EnergeticAmbiente.
Questa guida nasce dall'esperienza di implementazione di un sistema di monitoraggio di un impianto fotovoltaico realizzata con un dispositivo Shelly-EM interfacciato con il pacchetto software Telegraf-Influx-Grafana installato su un Raspberry Pi4.
Questa implementazione è nata e si è evoluta in parallelo a quelle realizzate dagli amici del sito Energeticambiente: 
- glfp che ha implementato un monitoraggio dell'impianto fotovoltaico analogo con  sensori SDM230
- raffaelem che ha implementato un monitoraggio di impianto a Pompa di Calore con altri sensori

Entrambi gli amici hanno pubblicato i propri tutorial: 
- glfp: https://github.com/glfp/SolarEnergyMonitorInfluxGrafanaDocker
- raffaelem: https://github.com/cassiel74/Grafana-Influx-PDC-monitor 

Il sottoscritto non aveva esperienza diretta nè di Raspberry, nè di Telegraf-Influx-Grafana.
Lo scopo di questa guida è quello di descrivere sinteticamente le principali fasi dell'implementazione: per gli approfondimenti si rimanderà alle varie guide e tutorial reperibili in rete.

**##1.MATERIALE HARDWARE**

Il materiale occorrente è il seguente:

**- Raspberry Pi4 (4GB)**

**- Dispositivo SHELLY-EM** dotato di n° 2 pinze amperometriche con portata 50A oppure 120A
Lo Shelly_EM è un dispositivo domotizzato che permette la misura di potenza istantanea(Watt), Energia (Wh), Tensione (V) transitante su due linee e collegabile via wifi con la propria LAN (la stessa della quale fa parte il Raspberry).

**2. SCHEMA DI MONTAGGIO DEI DISPOSITIVI** 

E' riportato nella figura seguente

![Figura 1](https://github.com/githubbyte/Shelly-EM-Monitor-Telegraf-Influx-Grafana/blob/master/shellydef.jpg)

In esso si nota:

a) lo  Shelly-EM va alimentato a valle dell'interruttore generale

b) la posizione delle pinze è tale da poter misurare direttamente le potenze ed energie transitanti sulla linea proveniente dalla Centrale fotovoltaica e su quella di scambio. Attenzione al verso di inserimento: K-->L deve essere rispettato come in figura. Nel seguito chiamerò "Pinza Produzione" o "Pinza FV" quella sulla linea del fotovoltaico e "Pinza scambio" quella sulla linea del contatore di scambio.

Questa è la configurazione "standard"  delle pinze ed è applicabile se lo schema di montaggio del vs impianto coincide con quello in figura. Esiste un altra configurazione di base degli impianti fotovoltaici nelle quali la linea dall'inverter va direttamente al contatore di scambio anzichè al quadro generale: in questo  caso è necessaria una piccola modifica all'impianto elettrico per riportarsi allo schema di cui sopra.


**3. CONFIGURAZIONE DELLO SHELLY-EM** 

Seguite le istruzioni dello Shelly-EM.

Esso si associerà alla rete Wifi di casa e potrete quindi usarlo con l'App Shelly-Cloud.

**L'accortezza da usare è quella di assegnare un indirizzo IP statico allo Shelly.
Nel mio caso ho assegnato 192.168.1.202.** (Voi potete scegliere ovviamente un valore a vs piacere, nel seguito indicherò questo indirizzo con la dizione IPSHELLY).

Se la vostra coonfigurazione è andata a buon fine e funzionante avrete sul cellulare l'App Shelly-Cloud funzionante che vi dà le varie potenze ed energie prodotte, immesse ed assorbite dalla vostra casa.

In vista dell'interfaccia con il software di monitoraggio che andremo a fare provate a verificarne il corretto fuzionamento
aprite una finestra col browser e digitate:
http://IPSHELLY/emeter/0

otterrete la risposta del tipo:

{"power":171.92,"reactive":86.37,"voltage":242.74,"is_valid":true,"total":883778.6,"total_returned":0.0}

Digitate ora :
http://IPSHELLY/emeter/1

otterrete ancora una risposta del tipo:

{"power":272.48,"reactive":-453.41,"voltage":242.84,"is_valid":true,"total":709044.9,"total_returned":459949.9}

Bene: queste risposte in formato Json sono quelle alla base del nostro monitoraggio e rappresentano le grandezze misurate rispetivamente dalle pinze 1 (Produzione) (per Shelly è la pinza con index=0) e pinza 2 (Scambio) (per Shelly è la pinza con index=1).

Le risposte contengono:

-Power: Potenza eletrica istantanea (W) positiva se è nel verso K-->L, negativa se nel verso opposto. Quindi sempre >0 per la pinza produzione, mentre >0 in fase di prelievo e <0 in fase di immissione per la pinza scambio

-reactive: potenza elettrica reattiva (var) non ha interesse immediato per noi

-voltage: valore istantaneo della tensione (V)

-is valid: nessun interesse

-Total: Energia (wh) transitata in senso k--> nello strumento. Per la pinza FV è il totale dell'energia prodotta dal momento dell'installazione dello Shelly. Dividendo per 1000 si ottiene il dato in Kwh. Per la Pinza scambio è il totale dell'energia che il vostro impianto ha assorbito dalla rete dal momento dell'installazione dello Shelly

-total-returned: Energia (wh) transitata in senso inverso a "total" (cioè L-->K). Nel caso della Pinza FV questo valore è sempre nullo. Nel caso della Pinza scambio questo valore rappresenta il totale dell'energia immessa in rete dal vostro impianto (dividendo per 1000 si hanno i Kwh)


**4. SOFTWARE**
Prerequisiti
Raspberry Pi4 configurato e funzionante operativo h24 per poter monitorare le grandezze in continuazione.
Io ho installato la versione più recente alla data del 01 aprile 2020:
VERSION="10 (buster)"
Indirizzo IP del Raspberry statico (nel mio caso 192.168.1.49), lo indicherò nel seguito IPRASPY

Il sowtare necessario
