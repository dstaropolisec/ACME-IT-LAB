# TCK-002 — Domain controller DC01 non si avvia dopo un errore di I/O

| Campo | Valore |
|---|---|
| Data | 26/10/2026 |
| Categoria | Infrastruttura › Virtualizzazione / Active Directory |
| Priorità | Alta (unico domain controller del dominio fermo) |
| Sistema | DC01 (Windows Server 2022, VM VirtualBox) |
| Tecnico | Domenico Staropoli |
| Stato | Chiuso |

## Sintomo

1. Durante il lavoro la VM DC01 si è messa in pausa con l'errore VirtualBox
   `BLKCACHE_IOERR` — *"The I/O cache encountered an error while updating data in medium ahci-0-0 (VERR_INVALID_HANDLE)"*.
2. Dopo la chiusura, la VM non si avviava più:
   `VERR_FILE_NOT_FOUND opening image file ...\Snapshots\{be1b1410-...}.vdi`.

## Diagnosi

- **Analisi della catena dei dischi** nel file di configurazione `.vbox`:
  - `DC01_.vdi` → disco base: **presente**
  - `{ae24fd99-...}.vdi` → disco differenziale dello snapshot "lab 20/10/2026": **presente**
  - `{be1b1410-...}.vdi` → disco differenziale dello stato corrente (modifiche dopo lo snapshot): **mancante**
- Lo stato salvato `.sav` dello snapshot era **presente**.
- **Verifica del disco host** (NVMe interno), da prompt amministratore:
  - `chkdsk F: /scan` → nessun problema nel file system, 0 KB in settori danneggiati, ~570 GB liberi
  - `Get-PhysicalDisk` → HealthStatus: Healthy
- Conclusione: disco fisico sano e spazio sufficiente. È andato perso solo il file delle modifiche recenti, probabilmente durante l'errore di I/O.

## Soluzione

1. Ripristino dello snapshot **"lab 20/10/2026"** (senza salvare lo stato corrotto).
2. Cartella delle VM esclusa dalla scansione in tempo reale di Defender:
   `Add-MpPreference -ExclusionPath "F:\vm virtualbox"`.
3. **Check-up del domain controller dopo il ripristino**:

```powershell
Get-Service NTDS, DNS, Netlogon, KDC, W32Time, DFSR | Format-Table Name, Status   # tutti Running
w32tm /resync /force                                                             # ora riallineata
dcdiag /q                                                                        # nessun errore
Get-ADDefaultDomainPasswordPolicy | Format-List LockoutThreshold, LockoutDuration
Get-ADComputer -Filter * | Format-Table Name, Enabled
```

4. Ricreate le due GPO perse con il rollback (vedi LAB-08) e creato un nuovo snapshot.

## Effetti collaterali

Il rollback del DC ha causato la rottura della relazione di trust con CLIENT01 (vedi **TCK-003**).

## Cosa ho imparato

- **Uno snapshot non è un backup:** vive sullo stesso disco della VM. Per un DC in produzione serve un backup vero (Windows Server Backup / System State, oppure backup a livello di hypervisor su un altro storage).
- Dopo un arresto anomalo di un DC si verificano **servizi AD, ora e `dcdiag`** prima di considerarlo operativo.
- Il rollback di un DC ha conseguenze sul dominio: password macchina dei client e, con più DC, rischio di *USN rollback*. Per questo in produzione un DC non si ripristina da snapshot alla leggera.

## Racconto STAR (colloquio)

- **Situazione:** l'unico domain controller del lab non si avviava per un file di disco virtuale mancante.
- **Compito:** rimettere in servizio il dominio con la minima perdita di dati.
- **Azione:** ho letto la catena dei dischi nel file `.vbox` per capire esattamente quale file mancasse, ho escluso un guasto hardware con `chkdsk` e lo stato SMART, ho ripristinato l'ultimo snapshot valido e verificato il DC con `dcdiag`.
- **Risultato:** dominio di nuovo operativo in circa 20 minuti. Persa solo la configurazione di due GPO, ricreata da script, e procedura documentata.
