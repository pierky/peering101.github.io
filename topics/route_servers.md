---
layout: default
title: Route Servers
subtitle: Definizione e introduzione ai route servers
---

# Route Servers

I route servers sono uno strumento spesso messo a disposizione degli afferenti di un Internet Exchange Point per facilitare lo scambio di rotte sulla piattaforma di public peering.

Al fine di beneficiare dei vantaggi offerti da un Internet Exchange, ogni afferente ha la necessità di scambiare le rotte con gli altri afferenti e potenzialmente instradare il traffico sulla LAN di peering condivisa con le altre reti. Per massimizzare l'uso della piattaforma, il numero di rotte scambiate deve essere il maggiore possibile, ovvero è necessario che ogni afferente stabilisca una o più sessioni BGP di peering "bilaterali" con quanti più membri dell'IXP possibile. Come è facilmente immaginabile, coordinare la configurazione di una sessione BGP con un'altra rete, eseguirne la configurazione e verificarne l'effettiva funzionalità richiede del tempo ed del lavoro.

Il ruolo dei route servers è esattamente quello di massimizzare l'uso della piattaforma di peering di un IXP minimizzando lo sforzo richiesto ai suo membri, consentendo loro di instradare e ricevere traffico fin dal primo momento utile dopo la connessione alla LAN di peering. Questo vantaggio può essere ottenuto stabilendo una o più sessioni BGP di peering "multilaterale" direttamente con i route servers (normalmente una coppia, per questioni di ridondanza), che svolgono una funzione di intermediari tra le varie reti, annunciando a ciascun client tutte le rotte ricevute dagli altri clients. In questo modo, anziché dover coordinare, configurare, testare e mantenere tante sessioni quante sono le reti che si vogliono raggiungere tramite la LAN di peering, ogni membro può ricevere la stessa quantità di rotte mediante una singola sessione.

Ovviamente, affinché le rotte di due reti vengano scambiate in assenza di una sessione BGP bilaterale, è necessario che entrambe le reti siano client dei route servers.

# Specifiche e funzionamento

Gli RFC 7947 e 7948 definiscono alcune specifiche e considerazioni operative riguardo l'implementazione e la gestione dei route servers presso un IXP.

Lato server, l'implementazione di una specifica funzionalità nei daemons BGP consente di evitare l'alterazione dell'AS_PATH delle rotte scambiate tra i clients. Normalmente, usando una sessione BGP tradizionale, l'ASN del route server verrebbe inserito di fronte all'ASN del client annunciante una specifica rotta:

   client A - AS65001 -> route server AS65000 -> client B - AS65002   [65000 65001]

L'implementazione della funzionalità "route server client" consente al daemon BGP utilizzato sul route server di evitare l'inserimento del propio ASN nelle rotte annunciato verso lo specifico neighbor:

   client A - AS65001 -> route server AS65000 -> client B - AS65002   [65001]

Similmente a ciò che avviene con l'AS_PATH, anche il NEXT_HOP delle rotte scambiate tra route server e clients non viene alterato; questa funzionalità consente ai route servers di rimanere sempre al di fuori del percorso dei dati, in quanto il traffico scambiato tra clients viene instradato direttamente mediate gli indirizzi IP dei clients stessi, senza quindi attraversare i route servers stessi.

Lato client, l'unica accortezza necessaria per la configurazione di una sessione verso un route server è quella di bypassare eventuali controlli che verifichino la corrispondenza del primo ASN negli AS_PATH delle rotte ricevute con l'ASN del neighbor dal quale si ricevono le rotte. Nell'esempio di cui sopra, client B deve "rilassare" i controlli sulla sessione con il route server, accettando rotte dal neighbor in AS65000 anche se l'AS_PATH non inizia con AS65000.

# Sicurezza

Un aspetto molto importante delle configurazioni lato route server è dato dalle features di sicurezza che essi offrono in ambito routing.

Le tradizionali sessioni BGP bilateriali consentono normalmente alle due reti che le stabiliscono di implementare "in casa" tutti i controlli di sicurezza per prevenire problemi di routing quali "route leaks": annuncio e ricezione di rotte non desiderate o non autorizzate. Nel momento in cui una rete A desidera stabilire una sessione di peering con la rete B, lo scambio dei relativi AS-SETs consente ad entrambe di conoscere esattamente quali rotte sia possibile accettare (e per contro, quali filtrare) sulla relativa sessione di peering.
L'implementazione di questi controlli non è di fatto possibile quando la sessione BGP viene stabilita con un route server, in quanto l'insieme di rotte ricevute su quella unica sessione ricopre una moltitudine variabile e non determinata di reti. Tornando all'esempio, se la rete A stabilisce una sessione BGP con un route server, essa non ha modo di implementare "in casa" dei filtri da poter applicare alla sessione stessa, in quanto non conosce a piori quali reti (e quindi quali AS-SETs) debba aspettarsi di ricevere.
Al fine di garantire la massima qualità delle rotte scambiate è fondamentale che i route servers implementino filtri e policies finalizzati alla validazione delle rotte ricevute dai propri clients.

Attualmente, meccanismi di filtering basati sugli oggetti RPSL, RPKI ROAs e BGP Origin Validation, come anche basilari principi di routing hygiene (filtraggio dei prefissi bogons, ASNs riservati, etc...) consentono di garantire ottimi livelli di fiducia che i clients possono riporre nelle rotte ricevute dai route servers che li implementino.

Al fine di tracciare la presenza di meccanismi di sicurezza implementati dai vari Internet Exchanges esiste un sito, https://peering.exposed/, sul quale è possibile trovare un elenco con una specifica colonna ("Secure Route servers") che rappresenta lo stato di sicurezza dei relativi route servers.

# Traffic engineering

Un'ulteriore importante caratteristica di un route server è rappresentata dai meccanismi di controllo offerti riguardo la propazione delle rotte che vengono esso annunciate dai clients.

Tornando allo scenario rappresentato da tradizionali sessioni BGP bilaterali, la rete A con una sessione di peering verso la rete B può implementare su tale sessione le proprie policies per controllare come le rotte annunciate vengano propagate al proprio peer: prepending per un particolare prefisso, evitare l'annuncio di un altro, aggiunta della community NO_EXPORT per un altro ancora. In questo scenario tutte queste azioni di controllo hanno un ambito ben definito, relativo alla sola rete B.

In caso di peering con un route server, l'implementazione di queste policies di controllo avrebbe un effetto globale, con impatto su tutte le reti connesse al route server. Similmente a quanto già visto per la sicurezza del routing, è necessario che il confine per la manipolazione degli annunci venga spostato sui route servers. Al fine di consentire ai clients di manipolare le politiche con cui i propri annunci vengono propagati agli altri membri, i route servers sono chiamati ad esporre dei meccanismi di segnalazione sotto forma di BGP communities. A ciascuna policy di controllo viene assegnato un valore sotto forma di BGP community: la presenza (o meno) della relativa community nella rotta annunciata da un client ne determina quindi l'applicazione. Ad esempio, una specifica community può essere definita per evitare l'annuncio delle rotte verso gli altri clients: la large BGP community 65000:0:<x>, qualora associata ad una rotta annunciata al route server, potrebbe essere utilizzata per evitare la propagazione della stessa verso i clients con ASN <x>.

# Cons

Da sviluppare??????

## path hiding

## blackholing dato da control plane path != data plane path

# Implementazioni e configurazioni

Le maggiori implementazioni software in ambito route servers sono attualmente due: [BIRD](https://bird.network.cz/) ed [OpenBGPD](http://www.openbgpd.org/). Entrambe questi software sono disponibili in forma open source e sono costantemente aggiornati e manutenuti dai rispettivi gruppi di lavoro.

Sempre in ambito OSS, alcuni tool consentono agli IXPs di facilitare la configurazione dei route servers, automatizzando l'implementazione delle varie politiche di sicurezza e delle features di controllo degli annunci: [ARouteServer](https://github.com/pierky/arouteserver), [IXPManager](https://www.ixpmanager.org/).

# Autore
[Pier Carlo Chiodi](https://www.linkedin.com/in/piercarlochiodi/)