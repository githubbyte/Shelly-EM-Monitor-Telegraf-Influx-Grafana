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

## **1.HARDWARE** ##

Il materiale occorrente è il seguente:

**- Raspberry Pi4 (4GB)** collegato alla rete di casa (wifi o ethernet) e funzionante h24.

**- Dispositivo SHELLY-EM** dotato di n° 2 pinze amperometriche con portata 50A oppure 120A.
Lo Shelly_EM è un dispositivo domotizzato che permette la misura di potenza istantanea(Watt), Energia (Wh), Tensione (V) transitante su due linee e collegabile via wifi con la propria LAN (la stessa della quale fa parte il Raspberry).

## **2. SCHEMA DI MONTAGGIO DELLO SHELLY-EM** ##

E' riportato nella figura seguente

![Figura 1](https://github.com/githubbyte/Shelly-EM-Monitor-Telegraf-Influx-Grafana/blob/master/shellydef.jpg)

In esso si nota:

a) lo  Shelly-EM va alimentato a valle dell'interruttore generale

b) la posizione delle pinze è tale da poter misurare direttamente le potenze ed energie transitanti sulla linea proveniente dalla Centrale fotovoltaica e su quella di scambio. Attenzione al verso di inserimento: K-->L deve essere rispettato come in figura. Nel seguito chiamerò "Pinza Produzione" o "Pinza FV" quella sulla linea del fotovoltaico e "Pinza scambio" quella sulla linea del contatore di scambio.

Questa è la configurazione "standard"  delle pinze ed è applicabile se lo schema di montaggio del vs impianto coincide con quello in figura. 

Esiste un altra configurazione di base degli impianti fotovoltaici nelle quali la linea dall'inverter va direttamente al contatore di scambio anzichè al quadro generale. Lo schema è quello della figura seguente ![Figura 2]( https://github.com/githubbyte/Shelly-EM-Monitor-Telegraf-Influx-Grafana/blob/master/FOTOVOLTAICO_CONF2.jpg)

In questo  caso è necessaria una piccola modifica all'impianto elettrico per riportarsi allo schema di cui sopra, vedi figura seguente:![figura 3](https://github.com/githubbyte/Shelly-EM-Monitor-Telegraf-Influx-Grafana/blob/master/FOTOVOLTAICO_CONF2%20CON%20SHELLY%20(2).jpg)


## **3. CONFIGURAZIONE DELLO SHELLY-EM** ##

Seguite le istruzioni dello Shelly-EM.

Esso si associerà alla rete Wifi di casa e potrete quindi usarlo con l'App Shelly-Cloud.

**L'accortezza da usare è quella di assegnare un indirizzo IP statico allo Shelly.
Nel mio caso ho assegnato 192.168.1.202.** (Voi potete scegliere ovviamente un valore a vs piacere, nel seguito indicherò questo indirizzo con la dizione SHELLY_IP).

Se la vostra coonfigurazione è andata a buon fine e funzionante avrete sul cellulare l'App Shelly-Cloud funzionante che vi dà le varie potenze ed energie prodotte, immesse ed assorbite dalla vostra casa.

In vista dell'interfaccia con il software di monitoraggio che andremo a fare provate a verificarne il corretto fuzionamento
aprite una finestra col browser e digitate:
http://SHELLY_IP/emeter/0

otterrete la risposta del tipo:

{"power":171.92,"reactive":86.37,"voltage":242.74,"is_valid":true,"total":883778.6,"total_returned":0.0}

Digitate ora :
http://SHELLY_IP/emeter/1

otterrete ancora una risposta del tipo:

{"power":272.48,"reactive":-453.41,"voltage":242.84,"is_valid":true,"total":709044.9,"total_returned":459949.9}

Bene: queste risposte in formato Json sono quelle alla base del nostro monitoraggio e rappresentano le grandezze misurate rispetivamente dalle pinze 1 (Produzione) (per Shelly è la pinza con index=0) e pinza 2 (Scambio) (per Shelly è la pinza con index=1).

Le risposte contengono:

- Power: Potenza eletrica istantanea (W) positiva se è nel verso K-->L, negativa se nel verso opposto. Quindi sempre >0 per la pinza produzione, mentre >0 in fase di prelievo e <0 in fase di immissione per la pinza scambio

- reactive: potenza elettrica reattiva (var) non ha interesse immediato per noi

- voltage: valore istantaneo della tensione (V)

- is valid: nessun interesse

- total: Energia (wh) transitata in senso K-->L nello strumento. Per la pinza FV è il totale dell'energia prodotta dal momento dell'installazione dello Shelly. Dividendo per 1000 si ottiene il dato in Kwh. Per la Pinza scambio è il totale dell'energia che il vostro impianto ha assorbito dalla rete dal momento dell'installazione dello Shelly

- total-returned: Energia (wh) transitata in senso inverso a "total" (cioè L-->K). Nel caso della Pinza FV questo valore è sempre nullo. Nel caso della Pinza scambio questo valore rappresenta il totale dell'energia immessa in rete dal vostro impianto (dividendo per 1000 si hanno i Kwh)


## **4. SOFTWARE** ##
Prerequisiti
Raspberry Pi4 configurato e funzionante operativo h24 per poter monitorare le grandezze in continuazione.
Il sistema operativo installato è la versione più recente alla data del 15 marzo 2020:

VERSION="10 (buster)"

**Indirizzo IP del Raspberry statico (nel mio caso 192.168.1.49)**, lo indicherò nel seguito con la dizione IP_RASPY.

Il sowtare necessario per far funzionare il tutto è il seguente:

**- Telegraf** : pacchetto che legge in continuazione ad intervalli di tempo da noi fissati i dati dallo Shelly e li impacchetta in formato di "timeseries" da passare a Influxdb

**- Influxdb** : gestore di Databases del tipo "timeseries"

**- Grafana** : software di grafica che inoltra queries a Influx e visualizza i grafici

**- Docker** : software "contenitore" all'interno del quale possono essere caricati tutti o parte dei pacchetti di cui sopra. 

Qui si aprono vari scenari:

1. Inserire tutti i pacchetti in Docker : è la scelta fatta da glfp

2. Fare a meno di Docker ed installare i pacchetti singolarmente : è la scelta fatta da raffaelem

3. Inserire alcuni pacchetti fuori Docker ed altri in Docker. E' la scelta più complicata e meno convincente, ma per motivi "storici" miei alla fine è quella in cui mi sono trovato. Io ho Telegraf e Influxdb che girano fuori Docker e Grafana che gira in Docker. Agli effetti pratici però non cambia nulla.


## **4.1 Installazione INFLUXDB** (fuori Docker)

Ho seguito la guida, fatta molto bene, di Michele Dal Bosco datata 19/10/2019 che trovate [qui](https://www.uiblog.it/2019/10/configuriamo-tig-influxdb-su-raspbian-buster-1-parte/).

Attenzione alla parte dell'autenticazione: segnatevi la password che assegnate all'amministratore.


Alla fine della procedura troverete Influxdb installato sul Raspy.

Potete seguire anche la guida di raffaelem sopra citata.
E' del tutto equivalente.

**Prime operazioni con INFLUXDB**

- Entrare in Influx:

`influx -username 'admin' -password 'XXXX'`

Risposta attesa:

```
Connected to http://localhost:8086 version 1.7.10
InfluxDB shell version: 1.7.10
```


- Creare il Database che ci servirà in seguito per caricarci le misure dello Shelly (nome a piacere, io ho scelto **SHELLYDB**):

`CREATE SHELLYDB`

Verifica che sia stato creato:

`SHOW DATABASES`

Risposta attesa:

```
name: databases
name
----
-internal
SHELLYDB
```
A questo punto non è necessario dare comandi particolari a Influx.
La creazione delle misure, dei tags e dei fields del database sarà gestita da Telegraf nella fase successiva.

Consiglio di leggere la guida di raffaelem per familiarizzare con le varie clausole di Influx: SELECT,WHERE 

## 4.2 Installazione TELEGRAF (parte 1 generale)(fuori Docker)

Per effettuare l'installazione ho seguito  per la parte generale anche in questo caso la guida di Michele Dal Bosco datata 25/10/2019 che trovate [qui](https://www.uiblog.it/2019/10/configuriamo-tig-telegraf-su-raspbian-buster-2-parte/), ma nella parte speciifica di interfacciamento con lo Shelly ho proceduto autonomamente.

La fase 1 generale è la seguente:

**4.2.1 Installazione**

Prima aggiorniamo
```
sudo apt-get update
```
quindi installiamo telegraf:
```
sudo apt-get install telegraf
```

Telegraf dovrebbe essere in esecuzione sul raspberry. 
Verifichiamolo col seguente comando:
```
sudo systemctl status telegraf
```
Dovrebbe apparire una risposta del tipo:
```
● telegraf.service - The plugin-driven server agent for reporting metrics into InfluxDB
   Loaded: loaded (/lib/systemd/system/telegraf.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-04-19 04:59:43 CEST; 5 days ago
     Docs: https://github.com/influxdata/telegraf
 Main PID: 484 (telegraf)
    Tasks: 15 (limit: 4915)
   Memory: 46.0M
   CGroup: /system.slice/telegraf.service
           └─484 /usr/bin/telegraf -config /etc/telegraf/telegraf.conf -config-directory /etc/telegraf/telegraf.d
```
Nel caso non fosse in esecuzione lanciare il comando:
```
sudo systemctl start telegraf
```
Telegraf è in esecuzione ma non è ancora operativo e necessita di varie operazioni di configurazione.

**4.2.2 Creazione utente "Telegraf" in ambiente Influxdb**
La prima operazione da fare è far sì che telegraf possa comunicare con Influxdb (Telegraf dovrà scrivere le metriche sul database).
Per far questo procediamo ad accreditare un utente di nome "telegraf" in Influx entrando con la passord XXXX che avevamo creato per l'amministratore admin al punto precedente:
```
sudo influx -username 'admin' password 'XXXX'
```
quindi creiamo l'utente telegraf per Influxdb:
```
> CREATE USER telegraf WITH PASSWORD 'YYYY' WITH ALL PRIVILEGES
> SHOW USERS

user      admin
----      -----
admin     true
telegraf  true
> exit
```
**4.2.3 Alcune note sul funzionamento di Telegraf**

Per chi fosse interessato a capire ed approfondire Telegraf ecco il link alla guida completa: https://github.com/influxdata/telegraf

Prima di passare a configurare telegraf spendo comunque qualche riga per illustrarne il funzionamento e l'architettura dei suoi files.

Telegraf è un programma che gira sul Raspy come "service" in background ed in continuazione effettua le seguenti operazioni:

- lettura  ad intervalli di tempo prefissati (e da noi scelti) da tutte le sorgenti abilitate (i cosiddetti "inputs") delle grandezze più svariate utilizzando i cosiddetti "plugins" di input. Questi plugins sono tutti già presenti in Telegraf e sono numerosissimi, verranno attivati solo quelli scelti dall'utilizzatore. Nel nostro caso per leggere da Shelly attiveremo il plugin: inputs.http
- elaborazione delle grandezze lette tramite due tipi di procedimenti: processamento ("processor plugins") e/o aggregazione ("aggregator plugins") che possono filtrare e/o modificare le grandezze lette nello step precedente (questi due procedimenti non ci serviranno e non attiveremo nessun plugins)
- invio delle grandezze ricevute a varie destinazioni scelte dall'utente ("outputs plugins"). Fra queste destinazioni quella che interessa noi è influxdb su cui andremo a scrivere in un determinato Database che andremo a definire

Nel nostro caso avremo bisogno quindi di :
- definire i parametri generali di funzionamento di telegraf (intervallo di lettura e scrittura dei dati, ammontare dei bytes del buffer di scrittura, etc)
- definire i parametri del plugin di input (incaricato di leggere da Shelly-EM)
- definire i parametri del plugin di output (incaricato di scrivere su Influx)

Per fortuna Telegraf è molto bene organizzato e non sarà nostro compito dover scrivere nuove istruzioni, dovremo solo attivare delle scelte già predisposte e quindi il lavoro è molto semplice e comprensibile.

Le informazioni le faremo digerire a telegraf andandole a depositare in due files:

1° File: File di definizione delle variabili d'ambiente di telegraf ("environnement variables") di nome **telegraf** (senza estensione) da creare nella cartella **/etc/telegraf/**

2° File: File di configurazione di telegraf che troviamo già presente nella cartella **/etc/telegraf/** con nome **telegraf.conf**

**4.2.4 Configurazione di Telegraf per renderlo operativo**

Ricapitoliamo i dati di partenza :

- indirizzo IP dello ShellyEM: **http://xxx.yyy.z.w** (è quello che avete assegnato allo ShellyEM: SHELLY_IP, http://192.168.1.202 nel mio caso)
- indirizzo IP del Raspberry: **http://xxx.yyy.z.t** (è quello che avete assegnato al Raspberry: RASPY_IP, http://192.168.1.49 nel mio caso)
- nome del database di Influx su cui andrete a scrivere:**SHELLYDB**  (è quello creato al paragrafo 4.1, se avete scelto un nome diverso scrivete il vostro nome)


