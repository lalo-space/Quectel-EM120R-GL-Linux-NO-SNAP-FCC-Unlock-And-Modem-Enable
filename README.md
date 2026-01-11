# FCCUnlock

**Special thanks allo sviluppatore di Lenovo della libreria condivisa che non ha offuscato il codice e ha lasciato qualche indizio sui link alle altre librerie nei commenti per facilitare la decompilazione.**

## Introduzione

È stato utilizzato un portatile **Lenovo T14S Gen 2 Intel** con un modulo **WWAN Quectel EM120R-GL** (presente anche in altri modelli come l'X1 Carbon e molti altri, e compatibile con la scheda Quectel 160).

Il modulo non ha mai funzionato correttamente su Linux, pertanto è stato deciso di investigare una soluzione, evitando l'uso del pacchetto **SNAP** fornito da Lenovo (**lenovo-wwan-dpr**), per preferenze legate all'ambiente di sistema.

Lenovo spedisce alcuni dispositivi con un **FCC Lock** (Federal Communication Commission), che normalmente viene applicato solo ai dispositivi venduti sul suolo americano. Tuttavia, si è riscontrato che anche questo dispositivo, acquistato su Amazon Italia, presenta lo stesso blocco.

## Ambiente di test
```
Linux nes 6.1.0-25-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.106-3 (2024-08-26) x86_64 GNU/Linux
```

![immagine](https://github.com/user-attachments/assets/9c2d3b05-5d56-4881-9c55-72406e041ba6)

## Problema
```
[ 19.225293] thinkpad_acpi: rfkill switch tpacpi_wwan_sw: radio is blocked
```

Il **Network Manager** di XFCE4 rileva il dispositivo "WWAN", ma lo segnala come **Non Disponibile**. Tuttavia, nel tool di configurazione avanzata è possibile configurare i profili poiché l'interfaccia viene comunque rilevata.

## Soluzione proposta

Il pacchetto ufficiale di Lenovo **lenovo-wwan-dpr** è stato recuperato dallo **Snap Store**:

https://search.apps.ubuntu.com/api/v1/package/lenovo-wwan-dpr

### 1. Preparazione dell'ambiente

Creare una cartella di mount temporanea:
```bash
mkdir /mnt/snaptemp
```

Montare il pacchetto Snap scaricato:
```bash
mount -t squashfs -o ro /path/to/downloaded/file/LLzS0Nb7rtM5RH6dNadpAzWAiRfHuefk_26.snap /mnt/snaptemp/
```

### 2. Estrazione della libreria

Entrare nella directory `/mnt/snaptemp/usr/lib/` e prelevare il file `mbim2sar.so`.

Creare una cartella dedicata per l'operazione:
```bash
mkdir FCCUnlock
cd FCCUnlock
```

Copiare il file `mbim2sar.so` nella cartella creata.

### 3. Compilazione

Accedere al repository del codice sorgente `FCCUnlock.c` e copiarlo nella stessa cartella.

Avviare la compilazione del programma, facendo attenzione alle dipendenze:
```bash
gcc -o FCCUnlock FCCUnlock.c -ldl -Wall
```

> **Nota:** In caso di errore di segmentazione, verificare che `mbim2sar.so` sia collegato correttamente nel codice sorgente di FCCUnlock. Verificare la corretta definizione del percorso nel codice:
> ```c
> #define MBIM2SAR_SO_PATH "mbim2sar.so"
> ```

### 4. Esecuzione

Elevare i privilegi con `su` o `sudo` e seguire i comandi riportati di seguito.

#### Sblocco FCC
```bash
VERBOSE=1 ./FCCUnlock
```

Il comando esegue il programma FCCUnlock con il flag `VERBOSE=1`, che permette di ottenere un output dettagliato delle operazioni eseguite durante l'esecuzione, utile per il debugging e la verifica dello stato delle operazioni. In particolare, viene tentato il processo di sblocco FCC del modulo WWAN.

#### Verifica stato radio
```bash
mbimcli -p -d /dev/wwan0mbim0 -v --quectel-query-radio-state | grep sim
```

Il comando utilizza `mbimcli`, un'interfaccia a linea di comando per interagire con dispositivi MBIM. Con l'opzione `-p`, il dispositivo viene interrogato per lo stato della radio, mentre `-d /dev/wwan0mbim0` specifica il dispositivo da interrogare. L'opzione `-v` attiva la modalità verbose, e `--quectel-query-radio-state` interroga lo stato della radio del modem Quectel. Il comando finale `grep sim` filtra l'output per cercare informazioni relative alla SIM.

#### Elenco modem
```bash
mmcli -L
```

Questo comando elenca tutti i modem attualmente riconosciuti dal sistema tramite ModemManager. Il comando `mmcli -L` visualizza una lista dei modem, indicando per ciascuno il percorso DBus e le informazioni di base, come il modello e il produttore.

#### Configurazione slot SIM
```bash
mmcli -m 1 --set-primary-sim-slot=0
```

Il comando modifica lo slot SIM primario del modem con ID 1 (ottenuto dall'elenco dei modem con `mmcli -L`). L'opzione `--set-primary-sim-slot=0` imposta lo slot 0 come slot SIM primario.

#### Informazioni dettagliate modem
```bash
mmcli -m a
```

Questo comando visualizza le informazioni dettagliate del modem con ID `a`. Viene utilizzato per ottenere dati sullo stato del modem, inclusi dettagli su hardware, firmware, bande supportate, stato del segnale, modalità operative e stato della SIM.

#### Abilitazione modem
```bash
mmcli -v --modem=`mmcli -L | head -n1 | awk '{print$1}'` -e
```

Il comando `mmcli` viene utilizzato per abilitare il modem. L'opzione `-v` attiva la modalità verbose, fornendo un output dettagliato. Il comando interno `mmcli -L | head -n1 | awk '{print$1}'` recupera l'ID del primo modem dall'elenco e lo passa come argomento all'opzione `--modem`. L'opzione `-e` abilita il modem selezionato.

## Note finali

Resta solo da fare in modo che al caricamento del network manager venga abilitato il modem. Questo dipende molto dalla distribuzione e soprattutto dal DE che utilizzate.
