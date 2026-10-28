# TCK-003 — "Relazione di trust tra la workstation e il dominio primario non riuscita"

| Campo | Valore |
|---|---|
| Data | 26/10/2026 |
| Categoria | Active Directory › Computer |
| Priorità | Media |
| Postazione | CLIENT01 (Windows 11 Enterprise) |
| Tecnico | Domenico Staropoli |
| Stato | Chiuso |

## Segnalazione

Dopo lo sblocco dell'account (TCK-001), l'utente luca.verdi non riusciva ad accedere neanche con la password corretta:

> "La relazione di trust tra questa workstation e il dominio primario non è riuscita."

## Causa

Ogni computer del dominio ha una **password macchina** che cambia automaticamente (di default ogni 30 giorni) ed è salvata sia sul client sia sul DC. Il ripristino di DC01 a uno snapshot del 20/10 (TCK-002) ha riportato sul DC una password macchina vecchia: client e DC non coincidevano più.

## Ostacoli

1. **Password dell'amministratore locale di CLIENT01 non disponibile.**
   Soluzione: scollegata la scheda di rete della VM e fatto accesso con un account di dominio amministratore usando le **credenziali in cache**. Senza rete il client non contatta il DC e verifica le credenziali salvate localmente. Poi rete ricollegata.
2. **La finestra di `Get-Credential` non si apriva** (bloccata dietro la console).
   Soluzione: credenziale passata come oggetto `PSCredential` (accettabile solo in lab, vedi note).

## Soluzione

```powershell
# 1. Il client raggiunge il DC?
Test-Connection 192.168.50.10 -Count 2

# 2. Riparazione del secure channel (restituisce True)
$cred = Get-Credential LAB\Administrator
Test-ComputerSecureChannel -Repair -Credential $cred

# 3. Verifica (restituisce True)
Test-ComputerSecureChannel

# 4. Password dell'amministratore locale reimpostata e documentata
net user admin.lab *

# 5. Pulizia della sessione
Remove-Variable cred
Clear-History
```

`net user admin.lab *` chiede la nuova password senza mostrarla e senza lasciarla nella cronologia.

Dopo il primo accesso di luca.verdi, `gpupdate /force` e un nuovo logon per applicare le GPO utente (Fast Logon Optimization).

Verifica finale: accesso riuscito, unità Z: assente (utente non in GRP-Amministrazione), cmd e Pannello di controllo bloccati.

## Cosa ho imparato

- `Test-ComputerSecureChannel -Repair` reimposta la password macchina **senza togliere il PC dal dominio** e senza riavvio. L'uscita e il rientro nel dominio, invece, comportano riavvii e rischi per i profili.
- Le **credenziali in cache** permettono di entrare in un client isolato dalla rete.
- In produzione **non si scrive mai una password in chiaro** in un comando, perché finisce nella cronologia di PowerShell. Si usa `Get-Credential`.
- Le password degli admin locali vanno gestite con **Windows LAPS**, così sono univoche per ogni PC e conservate in AD.

## Racconto STAR (colloquio)

- **Situazione:** dopo il ripristino del DC una postazione non riusciva più ad autenticarsi al dominio, e la password dell'admin locale non era disponibile.
- **Compito:** riparare la relazione di trust senza reinstallare né togliere il PC dal dominio.
- **Azione:** sono entrato con le credenziali di dominio in cache isolando la rete, ho riparato il secure channel con `Test-ComputerSecureChannel -Repair` e ho impostato una password admin locale documentata.
- **Risultato:** postazione di nuovo nel dominio in pochi minuti, profili utente intatti, GPO applicate correttamente.
