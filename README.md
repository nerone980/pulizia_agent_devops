# Pulizia Agent DevOps

Applicazione web **single-file** (`PuliziaAgent.html`) per la gestione e la pulizia degli agenti registrati su un **Agent Pool di Azure DevOps Server (TFS on-premises)**.

Non richiede build, backend o dipendenze esterne: è un unico file HTML con CSS e JavaScript vanilla, pensato per essere aperto direttamente nel browser da chi ha accesso alla rete interna dove risiede il server TFS.

## Cosa fa

- **Connessione al server**: si collega a un'istanza TFS/Azure DevOps Server tramite le REST API `_apis/distributedtask/pools`, usando un URL configurabile (di default `http://censrvvftfs002:8080/tfs/ProgettiGit`) e un **Personal Access Token (PAT)**.
- **Caricamento pool e agenti**: elenca gli Agent Pool disponibili e, selezionato un pool, carica l'elenco completo degli agenti con relative capability e ultima attività.
- **Raggruppamento automatico**: gli agenti vengono raggruppati per "tipologia", dedotta da una capability personalizzata (`Tipologia`) oppure calcolata dalle capability di sistema (OS + stack rilevato: .NET, Java, Maven, Node.js, Xcode, Docker).
- **Dashboard con statistiche**: conteggio di agenti totali, online, offline e "da eliminare" secondo il filtro attivo.
- **Filtro per data**: permette di considerare "da eliminare" solo gli agenti offline la cui ultima attività è precedente a una data scelta.
- **Eliminazione agenti**: singola, per gruppo (tipologia) o massiva su tutti gli agenti offline che rispettano il filtro, sempre con modale di conferma.
- **Log e feedback**: console di log con timestamp e toast di notifica per ogni operazione.
- **Tema chiaro/scuro**: selezionabile e persistito in `localStorage`.

## Come si usa

1. Aprire `PuliziaAgent.html` in un browser con accesso alla rete dove risiede il server TFS.
2. Inserire l'URL del server (se diverso da quello precompilato) e il proprio PAT (permessi minimi richiesti: gestione degli Agent Pool).
3. Cliccare "Carica Pool", selezionare il pool desiderato: gli agenti vengono caricati automaticamente.
4. (Opzionale) impostare una data di soglia per filtrare gli agenti offline da eliminare.
5. Eliminare gli agenti non più necessari, singolarmente, per gruppo o in blocco.

## Note di sicurezza

- Il PAT viene mantenuto **solo in memoria** per la durata della sessione della tab: non viene salvato su disco, in `localStorage` o inviato altrove se non come header `Authorization: Basic` verso il server TFS configurato.
- L'applicazione effettua chiamate dirette dal browser al server TFS: va quindi usata solo su reti fidate e con server raggiungibili in modo sicuro (idealmente HTTPS).
- Le operazioni di eliminazione sono distruttive e irreversibili: è sempre richiesta una conferma esplicita prima di procedere.

## Correzioni applicate in questa revisione

Durante la scansione del repository è stato individuato e corretto un problema di **escaping HTML incompleto**: i nomi degli agenti e le tipologie (valori che provengono dalle capability configurate sulle macchine agent, quindi non completamente sotto il controllo di chi usa questo tool) venivano inseriti nel DOM tramite `innerHTML` senza un escaping HTML corretto, e i valori usati negli attributi `onclick` gestivano solo l'apice singolo (non le doppie virgolette o il backslash). Un nome agente o una tipologia contenenti questi caratteri potevano quindi rompere il markup generato o, in casi limite, permettere l'iniezione di markup/script arbitrario.

La correzione:
- introduce una funzione `escapeHtml()` usata per tutti i valori dinamici (nome agente, tipologia) inseriti come testo o come attributi;
- sostituisce l'interpolazione diretta di stringhe negli attributi `onclick` con attributi `data-*`, letti a runtime tramite `dataset` (il browser gestisce automaticamente la decodifica degli entity HTML), eliminando la necessità di un escaping "ibrido" HTML/JavaScript fragile.

## Possibili miglioramenti futuri (non implementati)

- **Paginazione API**: la chiamata di caricamento agenti non gestisce un eventuale `continuationToken` restituito dalle API di Azure DevOps; per pool con un numero molto elevato di agenti alcuni risultati potrebbero non essere recuperati (da verificare rispetto al comportamento reale del server TFS in uso).
- **Eliminazioni sequenziali**: le eliminazioni di gruppo/massive vengono eseguite una alla volta (`await` in sequenza) anziché in parallelo, per evitare di sovraccaricare il server; su pool molto grandi l'operazione può risultare lenta.
- **Storage del PAT**: attualmente va reinserito ad ogni apertura della pagina; si potrebbe valutare un'opzione esplicita (opt-in) di salvataggio cifrato locale, se il flusso d'uso lo richiede.
