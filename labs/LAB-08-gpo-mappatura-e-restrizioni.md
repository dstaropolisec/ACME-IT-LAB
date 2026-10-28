# LAB-08 — Condivisione di reparto, mappatura unità e restrizioni utente via GPO

**Dominio:** `lab.internal` · **OU:** `OU=utenti,OU=SEDE-CATANZARO` · **Data:** 26/10/2026

## Obiettivo

Simulare la configurazione tipica di una sede aziendale:

- cartella dell'ufficio Amministrazione accessibile **solo** al gruppo `GRP-Amministrazione`;
- unità **Z:** mappata automaticamente solo per chi è nel gruppo;
- postazioni degli utenti standard senza Pannello di controllo, prompt dei comandi e Registro.

## 1. Cartella condivisa e permessi

- Condivisione `Amministrazione` → `C:\Condivisioni\Amministrazione`.
- NTFS: ereditarietà disattivata, rimosso `BUILTIN\Users`. Restano `GRP-Amministrazione`, `SYSTEM`, `Administrators` e `CREATOR OWNER`.
- **Hardening:** rimossa la condivisione della cartella madre `C:\condivisioni`, che offriva un percorso alternativo alla stessa cartella con permessi di condivisione diversi.

```powershell
Remove-SmbShare -Name "condivisioni" -Force
Get-SmbShare | Where-Object Name -notlike "*$"
```

## 2. GPO "U-Mappatura-Z-Amministrazione"

```powershell
New-GPO -Name "U-Mappatura-Z-Amministrazione" |
  New-GPLink -Target "OU=utenti,OU=SEDE-CATANZARO,DC=lab,DC=internal"
```

Configurazione in GPMC → *Configurazione utente › Preferenze › Impostazioni di Windows › Mapping unità*:

| Impostazione | Valore |
|---|---|
| Azione | Aggiorna |
| Percorso | `\\<DC>\Amministrazione` |
| Lettera | Z: |
| Etichetta | Amministrazione |
| Destinazione a livello di elemento | Gruppo di sicurezza = `GRP-Amministrazione` |

## 3. GPO "U-Restrizioni-Utenti"

```powershell
$gpo = "U-Restrizioni-Utenti"
New-GPO -Name $gpo | New-GPLink -Target "OU=utenti,OU=SEDE-CATANZARO,DC=lab,DC=internal"
Set-GPRegistryValue -Name $gpo -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" -ValueName "NoControlPanel" -Type DWord -Value 1
Set-GPRegistryValue -Name $gpo -Key "HKCU\Software\Policies\Microsoft\Windows\System" -ValueName "DisableCMD" -Type DWord -Value 2
Set-GPRegistryValue -Name $gpo -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System" -ValueName "DisableRegistryTools" -Type DWord -Value 1
```

`DisableCMD = 2` blocca il prompt interattivo ma lascia funzionare gli script `.bat` di logon.

## 4. Test

| Utente | Gruppo | Z: visibile | Accesso cartella | cmd / control / regedit |
|---|---|---|---|---|
| anna.bianchi | GRP-Amministrazione | ✅ | Lettura/scrittura | Bloccati |
| luca.verdi | — | ❌ | Accesso negato | Bloccati |

Verifiche usate: `gpupdate /force`, `gpresult /r /scope user`.

## Problemi incontrati

- **GPO non applicate al primo accesso:** dovuto alla *Fast Logon Optimization* di Windows. Risolto con `gpupdate /force` + nuovo logon. Per evitarlo su tutti i client: *Configurazione computer › Modelli amministrativi › Sistema › Accesso › "Attendi sempre la rete all'avvio del computer e all'accesso"*.
- **Le modifiche ai gruppi non hanno effetto subito:** l'appartenenza ai gruppi è nel token Kerberos e si aggiorna solo al logon successivo.

## Limiti e sviluppi

- Bloccare `cmd.exe` è una **restrizione di interfaccia, non un controllo di sicurezza**: PowerShell resta disponibile. La protezione reale è che l'utente non sia amministratore locale. Per il controllo delle applicazioni servono **AppLocker** o **WDAC** (prossimo lab).
- I permessi effettivi su una risorsa di rete sono il **più restrittivo** tra permessi di condivisione e NTFS.
