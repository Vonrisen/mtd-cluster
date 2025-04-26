# Guida all'installazione e configurazione di una VM Linux con IP Statico

Questa guida illustra i passaggi per installare una macchina virtuale Linux (utilizzando un'immagine ISO standard) e configurarla con un indirizzo IP statico.

## 1. Download ISO e Setup Iniziale

1.  **Scarica l'ISO**: Visita la pagina ufficiale della distribuzione Linux scelta (ad esempio, Ubuntu, Debian, CentOS) per scaricare l'immagine ISO.
2.  **Crea la Macchina Virtuale**: Utilizza un hypervisor come VMware Workstation, VirtualBox o un altro di tua preferenza per creare una nuova macchina virtuale.
3.  **Processo di Installazione**: Avvia la VM con l'immagine ISO scaricata. Durante il processo di installazione:
    * Lascia tutte le impostazioni di default, a meno che non sia necessario modificarle per esigenze specifiche.
    * **Molto importante**: Seleziona l'opzione "Install OpenSSH Server" o equivalente per abilitare l'accesso remoto via SSH fin da subito.
4.  **Aggiornamento del Sistema**: Una volta completata l'installazione e avviata la VM:
    * Apri un terminale.
    * Esegui i seguenti comandi per aggiornare il sistema:
        ```bash
        sudo apt update
        sudo apt upgrade
        ```
    * **Installazione SSH (se non selezionato durante l'installazione)**: Se non hai installato l'OpenSSH Server durante il setup iniziale (ad esempio, usando un'installazione minimale o non-GUI), installalo manualmente:
        ```bash
        sudo apt install openssh-server
        ```
    * Riavvia il sistema dopo l'installazione di OpenSSH Server per assicurarti che il servizio sia attivo.

## 2. Configurazione dell'IP Statico

Per evitare che l'indirizzo IP della VM cambi ad ogni riavvio, è consigliabile impostare un IP statico. Utilizzeremo Netplan, il sistema di configurazione di rete predefinito in molte distribuzioni Linux recenti come Ubuntu 18.04+.

1.  **Identifica l'Interfaccia di Rete e l'Indirizzo Corrente**: Apri un terminale e esegui il seguente comando per vedere le tue interfacce di rete attive e i loro indirizzi IP:
    ```bash
    ip a
    ```
    Annota il nome dell'interfaccia di rete che vuoi configurare (es. `eth0`, `ens33`) e l'indirizzo IP attuale della VM (ti servirà come base per scegliere un nuovo IP statico nella stessa subnet). Annota anche l'indirizzo del Gateway (lo trovi solitamente con `ip r` o è l'indirizzo del tuo router/hypervisor).
2.  **Modifica il File di Configurazione Netplan**: Apri il file di configurazione di Netplan (il nome potrebbe variare leggermente, `50-cloud-init.yaml` è comune, ma verifica nella directory `/etc/netplan/` quale file presente):
    ```bash
    sudo nano /etc/netplan/50-cloud-init.yaml
    ```
3.  **Configura l'IP Statico**: Modifica il contenuto del file per farlo assomigliare a questo schema. **Sostituisci i valori tra `< >`** con le informazioni che hai raccolto nel Passaggio 1 e l'indirizzo IP desiderato per la VM.

    ```yaml
    network:
      version: 2
      ethernets:
        <network_interface>: # Esempio: ens33 o eth0
          addresses:
            - <VM_Address>/24 # Esempio: 192.168.1.100/24 (il /24 indica la subnet mask 255.255.255.0)
          routes:
            - to: default
              via: <GatewayIP> # Esempio: 192.168.1.1 (l'IP del tuo router)
          nameservers:
            addresses: [8.8.8.8, 8.8.4.4] # Server DNS di Google, puoi usare altri DNS se preferisci
          dhcp4: false # Disabilita DHCP per usare l'IP statico
    ```
    * `<network_interface>`: Il nome dell'interfaccia di rete (es. `ens33`).
    * `<VM_Address>/24`: L'indirizzo IP statico che desideri assegnare alla VM, seguito dalla notazione CIDR per la subnet mask ( `/24` è comune per `255.255.255.0`). Scegli un IP libero all'interno della subnet del tuo Gateway.
    * `<GatewayIP>`: L'indirizzo IP del tuo router o gateway di rete (es. `192.168.1.1`).
    * `addresses: [8.8.8.8, 8.8.4.4]`: Indirizzi dei server DNS (Google DNS in questo caso). Puoi usare quelli del tuo provider o altri a tua scelta.
    * `dhcp4: false`: Questa riga è fondamentale per disabilitare l'ottenimento automatico dell'IP via DHCP e utilizzare la configurazione statica definita. Assicurati di aggiungerla.

4.  **Applica la Configurazione**: Salva le modifiche al file (`Ctrl + X`, `Y`, `Invio` in `nano`) ed esegui il comando per applicare la nuova configurazione di rete:
    ```bash
    sudo netplan apply
    ```
5.  **Verifica**: Esegui `ip a` nuovamente per verificare che la VM abbia ora l'indirizzo IP statico che hai impostato.

La configurazione della tua macchina virtuale è ora completa e ha un indirizzo IP fisso.
