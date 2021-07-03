# Utilizzo di servizi conferenza basati su prossimità

Nel corso di incontri in cui partecipano molte persone, l'utilizzo di un unico _canale_ di comunicazione può risultare estremamente limitante.
È per questo che è già presente la necessità di poter organizzare gli incontri in più _stanze_, così che i singoli partecipanti possano decidere di passare da un argomento all'altro senza troppi problemi.
Questo tipo di soluzione, per quanto elementare, si trova già a fare i conti con alcune limitazioni, alcune legate ai mezzi tecnologici a disposizione, altre correlate all'uso delle _stanze_:
* Jitsi e servizi simili non hanno una sorta di _hub_ per passare facilmente tra le stanze
* Discord permette di passare velocemente tra stanze, ma stacca il canale video ad ogni passaggio, e le stanze audio-video non permettono testo
* L'attuale implementazione di Element si appoggia a Jitsi, quindi risolve parzialmente il problema. Tuttavia non c'è una profonda integrazione tra i due servizi, rimangono comunque piattaforme separate.

Provando a immaginare l'incontro virtuale come fisico, ci renderemmo facilmente conto che, per attività di tipo non unidirezionale (ad esempio seminari e presentazioni), è molto più funzionale l'idea di una stanza con diverse postazioni di lavoro piuttosto che diverse stanze isolate tra di loro.
Per tale motivo sarebbe più adatto l'utilizzo di una tecnologia di hangout virtuale **basata su prossimità** e non su più stanze.

## Necessità

Sistemi di incontri virtuali basati su prossimità esistono da moltissimo tempo, ma il loro sviluppo è stato legato al gaming per molto tempo, con un focus sulla sola funzionalità testuale o, in alcuni casi, quella audio.
Ci sono quindi una serie di necessità che la piattaforma utilizzata per organizzare gli incontri dovrebbe soddisfare:
* Supporto testuale
* Supporto audio-video
* Supporto allo screen sharing

All'attuale stato dell'organizzazione degli incontri, le seguenti caratteristiche sono estremamente gradite:
* Servizio gratuito
* Senza limiti al numero di partecipanti
* Leggero in termini di utilizzo risorse
* Client web
* Accesso senza autenticazione
* Semplice da usare

Alcune caratteristiche possono essere un valore aggiunto per i membri dalla comunità:
* Server free
* Client free
* Sistema distribuito invece che centralizzato
* Crittografia end-to-end

## Disponibilità

Esistono vari servizi, con pro e contro.

### Mozilla Hubs

Mozilla Hubs è una piattaforma free e open source sotto la [Mozilla Public License 2.0](https://github.com/mozilla/hubs/blob/master/LICENSE), ideata per fornire un ambiente 3d da utilizzare per incontri virtuali. La sua struttura evidenzia interesse verso il VR.

#### Pros
* Gratuito
* Senza limiti di partecipanti
* Client web
* Supporto per l'accesso anonimo
* Free e open source

#### Cons
* Il client non è leggero
* Può essere confusionario, sia in termini di utilizzo dei controlli che per funzionamento. In questo caso l'ambiente 3d sembra uno svantaggio, non un vantaggio.

#### Meh
* Centralizzato

### Online Town

[Online Town](https://theonline.town/) è un progetto carino che utilizza un ambiente ispirato dai JRPG per creare un mondo 2d in cui è possibile interagire. Sfortunatamente sembra che il servizio non funzioni più, e che gli sviluppatori si siano concentrati su [Gather Town](https://gather.town/). Ha senso citarlo perché è self-hostable, almeno sulla carta.

#### Pros
* Gratuito
* Senza limiti di partecipanti
* Client web
* Supporto per l'accesso anonimo
* Free e open source
* Leggero

#### Cons
* Attualmente il server pubblico sembra avere dei problemi. Potrebbe quindi avere delle limitazioni che non possiamo testare senza fare self-hosting.

#### Meh
* Centralizzato, per quanto sia self-hostable

### Gather Town

Evoluzione di Online Town, [Gather Town](https://gather.town/) sembra dare un aspetto più serio e professionale al concetto di meeting virtuale basato su grafica JRPG.

#### Pros
* Gratuito fino a 25 persone
* Client web
* Supporto per l'accesso anonimo
* Leggero
* Sembra molto più curato di Online Town

#### Cons
* A pagamento per un numero di persone maggiore di 25

#### Meh
* Non-free, closed
* Centralizzato

### Calla

[Calla](https://www.calla.chat/) prende (apparentemente) ispirazione da Online Town/Gather Town per costruire un sistema simile ma molto più open. Si appoggia a Jitsi per la parte audio-video.

#### Pros
* Gratuito
* Senza limiti di partecipanti
* Client web
* Supporto per l'accesso anonimo
* Free e open source
* Leggero

#### Cons
* Supporto audio-video, ma non testuale :shrug:

#### Meh
* Andrebbe sperimentato, non conosciamo l'affidabilità (per quanto utilizzi Jitsi)
* Centralizzato, per quanto sia self-hostable

### Here.Fm
[Here.Fm](https://here.fm) segue un approccio diverso dai servizi elencati precedentemente: invece ti avere un avatar virtuale per muoversi _fisicamente_ in uno spazio 2d/3d, questo servizio utilizza una sorta di _lavagna virtuale_ dove l'avatar è solo uno _sticker_ come altri. Spostando con il mouse il proprio _sticker avatar_ si va ad utilizzare le funzionalità di prossimità. Lo screensharing è _semplicemente_ uno sticker.

#### Pros
* Gratuito
* Senza limiti di partecipanti
* Client web
* Supporto per l'accesso anonimo
* Leggero
* Semplicissimo da utilizzare

#### Cons
* ??

#### Meh
* Andrebbe sperimentato, per constatare l'affidabilità
* Centralizzato
* Non-free, closed

## La proposta
Ritengo che Here.Fm possa essere la soluzione migliore in assoluto, per la sua semplicità e leggerezza. Sicuramente non conosciamo l'affidabilità, ma effettivamente non la conosciamo per nessuno dei servizi sopra citati (Gather Town e Mozilla Hubs dovrebbero essere solidi, ma oltre a non essere scontato hanno anche limitazioni non trascurabili).

Proposte future potrebbero considerare l'idea di utilizzare una delle alternative Open Source per creare un servizio self-hosted oppure migliorare l'esperienza attuale (ad esempio il supporto testuale a Calla), questa proposta punta a risolvere la questione della piattaforma tecnologica da utilizzare nell'immediato.

---

## Alternative secondarie

### Topia

[Topia](https://topia.io) sembra molto simile a Gather Town come approccio. Sembra avere simili pro e contro, però non sembra supportare Firefox. Meritava comunque di essere citato.

### CozyRoom

[CozyRoom](https://cozyroom.xyz/) è una piccola perla. Sfortunatamente non permette screensharing e condivisione video, ma per meeting solo audio e testo è semplicemente fantastico. Da tenere d'occhio nel caso integrassero funzionalità video.
