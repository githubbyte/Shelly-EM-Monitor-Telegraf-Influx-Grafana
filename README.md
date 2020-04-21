# Shelly-EM- Energy Monitor con Telegraf-Influx-Grafana. EnergeticAmbiente.
Questa guida nasce dall'esperienza di implementazione di un sistema di monitoraggio di un impianto fotovoltaico realizzata con un dispositivo Shelly-EM interfacciato con il pacchetto software Telegraf-Influx-Grafana installato su un Raspberry Pi4.
Questa implementazione è nata e si è evoluta in parallelo a quelle realizzate dagli amici del sito Energeticambiente: 
- glfp che ha implementato un monitoraggio dell'impianto fotovoltaico analogo con  sensori SMD20
- raffaelem che ha implementato un monitoraggio di impianto a Pompa di Calore con altri sensori
Entrambi gli amici hanno pubblicato i propri tutorial: 
- glfp: https://github.com/glfp/SolarEnergyMonitorInfluxGrafanaDocker
- raffaelem: https://github.com/cassiel74/Grafana-Influx-PDC-monitor 

Il sottoscritto non aveva esperienza diretta nè di Raspberry, nè di Telegraf-Influx-Grafana.
Lo scopo di questa guida è quello di descrivere sinteticamente le principali fasi dell'implementazione: per gli approfondimenti si rimanderà alle varie guide e tutorial reperibili in rete.
