# TCK-001 — Account utente bloccato

| Campo | Valore |
|---|---|
| Data | 26/10/2026 |
| Categoria | Active Directory › Account |
| Priorità | Media (un utente non può lavorare) |
| Utente | luca.verdi (SEDE-CATANZARO) |
| Postazione | CLIENT01 |
| Tecnico | Domenico Staropoli |
| Tempo di risoluzione | ~10 minuti |
| Stato | Chiuso |

## Segnalazione

> "Non riesco più a entrare, dice che l'account è bloccato. Ma la password è giusta."

## Contesto

La policy di dominio blocca l'account dopo **5 tentativi errati** per **15 minuti** (`Set-ADDefaultDomainPasswordPolicy`).

## Diagnosi

1. Stato dell'account:

```powershell
Get-ADUser luca.verdi -Properties LockedOut, BadLogonCount, LastBadPasswordAttempt |
  Format-List SamAccountName, LockedOut, BadLogonCount, LastBadPasswordAttempt
```

Risultato: `LockedOut: True`, `BadLogonCount: 5`, ultimo tentativo errato alle 17:37:43.

2. Origine dei tentativi (evento **4740** nel registro Security del DC):

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4740} -MaxEvents 5 |
  Format-List TimeCreated, @{n='Utente';e={$_.Properties[0].Value}}, @{n='Da PC';e={$_.Properties[1].Value}}
```

Risultato: blocco generato da **CLIENT01**, cioè la postazione dell'utente. Quindi i tentativi errati erano dell'utente stesso e non di un dispositivo con una password vecchia salvata (telefono, vecchio portatile, servizio).

## Soluzione

1. Verifica dell'identità dell'utente prima di intervenire.
2. Sblocco da **Utenti e computer di Active Directory** (`dsa.msc`) → Account → *Sblocca account*.
   In alternativa da PowerShell: `Unlock-ADAccount -Identity luca.verdi`.
3. Verifica: `LockedOut: False`, poi accesso riuscito sulla postazione.

Se l'utente non avesse ricordato la password, la procedura sarebbe stata: reset della password con cambio obbligatorio al primo accesso (`Set-ADAccountPassword -Reset` + `Set-ADUser -ChangePasswordAtLogon $true`).

## Cosa ho imparato

- L'evento **4740** indica **da quale computer** sono arrivati i tentativi. È il primo controllo da fare quando l'utente giura di non aver sbagliato la password.
- Lo sblocco si può fare da interfaccia grafica e va **verificato da PowerShell**.

## Racconto STAR (colloquio)

- **Situazione:** un utente non riesce ad accedere, l'account risulta bloccato.
- **Compito:** ripristinare l'accesso e capire la causa, per evitare che si ripeta.
- **Azione:** ho verificato lo stato con `Get-ADUser`, ho individuato la postazione di origine dall'evento 4740, ho sbloccato l'account e controllato l'esito.
- **Risultato:** utente operativo in pochi minuti, causa documentata (errori di digitazione dalla propria postazione, nessun dispositivo con password vecchia).
