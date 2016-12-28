# Capitolo 8: Tupperware

## Il Potente Container

<img src="images/jar.jpg" alt="http://blog.dwinegar.com/2011/06/another-jar.html" />

Abbiamo visto come scrivere programmi che concatenano i dati attraverso una serie di funzioni pure.
Queste risultano enunciati dichiarativi che ne specificano il comportamento. Ma per quanto riguarda il controllo di flusso, la gestione degli errori, comportamento asincrono, gli stato e, mi consenta(Berlusca), gli effetti collaterali?! In questo capitolo, scopriremo lo strumento fondamentale per costruire queste astrazioni.

Per prima cosa creeremo un contenitore. Questo contenitore è un oggetto che contiene valori di qualsiasi tipo (non può contenere solo budini) ma non gli daremo proprietà e metodi nel senso della programmazone OO. Piuttosto lo tratteremo come uno scrigno del tesoro - una scatola speciale che custodità i nostri dati di valore

```js
var Container = function(x) {
  this.__value = x;
}

Container.of = function(x) { return new Container(x); };
```

Ecco il nostro primo contenitore. Giustamente l' Abbiamo chiamato `Container`. Useremo `Container.of` come un costruttore che ci evita di scrivere (che Dio ce ne scampi) `new` dappertutto. In realtà la funzione `of` è molto di più di quello che sembra ma, per ora, pensaremo ad essa come il modo corretto di mettere i valori nel nostro contenitore.

Esaminiamo la nostra nuova scatola magica ...

```js
Container.of(3);
//=> Container(3)


Container.of('hotdogs');
//=> Container("hotdogs")


Container.of(Container.of({
  name: 'yoda',
}));
//=> Container(Container({name: "yoda" }))
```

Se si utilizza node, si vedrà `{__value: x}` nonostante sia un `Container(x)`. Chrome restituirà in output il tipo propriamente, ma non importa. In alcuni ambienti è possibile sovrascrivere il metodo `inspect`,ma spiegare la procedura esula dagli scopi di questo libro. D' ora in poi scriveremo l'uscita concettuale come se avessimo sovrascritto `inspect` in quanto è molto più istruttivo di` {__value: x} `sia per ragioni pedagogiche che per ragioni estetiche.

Mettiamo un paio di cose in chiaro prima di procedere:

* `Container` è un oggetto con una proprietà. Un sacco di contenitori contengono una sola cosa, sebbene non siano limitati a una. Abbiamo arbitrariamente chiamato la sua proprietà `__value`.

* `__value` non può essere uno tipo specifico o il nostro `Container` difficilmente sarà all'altezza del nome.

* Una volta che i dati vanno nel `Container` rimangono lì. Noi *potremmo* toglierlo usando `.__ value`, ma questo vanificherebbe l'obiettivo.

Le ragioni di tutto ciò diventeranno chiare come il sole, ma per ora, abbiate pazienza.

## Il mio primo Funtore

Una volta che il nostro valore ( qualunque esso sia ) è inserito nel `Container` ci serve un modo per eseguire funzioni su di esso.

```js
// (a -> b) -> Container a -> Container b
Container.prototype.map = function(f) {
  return Container.of(f(this.__value));
}
```
Sembra quasi come il famoso metodo `map` dell' oggetto `Array`, al posto di `[a]` abbiamo `Container of`. Essenzialmente funziona nello stesso modo:

```js
Container.of(2).map(function(two) {
  return two + 2;
});
//=> Container(4)


Container.of("flamethrowers").map(function(s) {
  return s.toUpperCase();
});
//=> Container("FLAMETHROWERS")


Container.of("bombs").map(_.concat(' away')).map(_.prop('length'));
//=> Container(10)
```
Siamo in grado di lavorare con il nostro valore, senza mai dover lasciare il `Container`. Il valore nel `Container` è passato alla funzione `map` ( che può pasticcirci come vuole ) e successivamente restituito al suo `Container` che lo tiene al sicuro. Il valore non lascia mai il `Container`, così possiamo continuare a `map`arci sopra funzioni come ci pare. Possiamo anche cambiarne il tipo come dimostrato nell' ultimo dei tre esempi.

Aspetta un minuto, questo continuo chiamare `map`, ricorda una sorta di composizione! Che magia matematica è questa? Bene, abbiamo appena scoperto i *Funtori*.

> Un funtore è un tipo che implementa `map` e obbedisce alcune leggi

Sì, *Functor* è semplicemente un'interfaccia con un contratto. Potremmo chiamarli *Mappable*, ma dove sarebbe il *Fun* ( divertimento gioco di parole ) in questo? I Funtori provengono una branca della matematica: la teoria delle categorie, c'è un paragrafo più dettegliato alla fine del capitolo. Per ora, cercheremo di lavorare sull' intuizione e vedere alcuni usi pratici per questa interfaccia chiamata in modo così bizzarro.

Che abbiamo di imbottigliare un valore e utilizzare `map` per raggiungerlo? Per rispondere, occorre pensare ad un' altra domanda: Che cosa ci guadagno dal chiedere al nostro Container di applicare funzioni per noi? Ebbene, l'astrazione dell' applicazione delle funzioni. Quando `map`iamo una funzione, chiediamo al tipo di Container di eseguirla per noi. Questo è un concetto molto potente.

## Il Maybe di Schrödinger

<img src="images/cat.png" alt="cool cat, need reference" />

`Container` è abbastanza noioso. In realtà, di solito è chiamato `Identity` e ha circa lo stesso impatto come della funzione `id` ( di nuovo c'è una connessione con la matematica, la vedremo quando sarà il momento giusto ). Tuttavia, ci sono altri funtori, cioè tipi di Container che hanno una specifica funzione `map`, che fornisce un comportamento utile durante la mappatura. Definiamone uno ora.

```js
var Maybe = function(x) {
  this.__value = x;
};

Maybe.of = function(x) {
  return new Maybe(x);
};

Maybe.prototype.isNothing = function() {
  return (this.__value === null || this.__value === undefined);
};

Maybe.prototype.map = function(f) {
  return this.isNothing() ? Maybe.of(null) : Maybe.of(f(this.__value));
};
```

Ora, `Maybe` assomiglia molto a `Container` con una piccola modifica: prima di chiamare la funzione fornita verifica se effettivamente contiene un valore. Questo ha l'effetto di evitare quei fastidiosi valori `null` mentre `map`piamo (Si noti che questa è la versione semplificata per scopi didattici).

```js
Maybe.of('Malkovich Malkovich').map(match(/a/ig));
//=> Maybe(['a', 'a'])

Maybe.of(null).map(match(/a/ig));
//=> Maybe(null)

Maybe.of({
  name: 'Boris',
}).map(_.prop('age')).map(add(10));
//=> Maybe(null)

Maybe.of({
  name: 'Dinah',
  age: 14,
}).map(_.prop('age')).map(add(10));
//=> Maybe(24)
```

La nostra applicazione non scoppia per gli errori mentre mappiamo le funzioni sui nostri valori nulli. Questo perché `Maybe` si preoccupa di verificare la presenza di un valore ogni volta che si applica una funzione.

Questa sintassi con la dot notation è perfettamente giusta e funzionale, ma per ragioni viste nella parte 1, vorremmo mantenere il nostro stile pointfree. Si dà il caso che `map` sia perfettamente attrezzato per delegare a qualunque funtore che riceve:

```js
//  map :: Functor f => (a -> b) -> f a -> f b
var map = curry(function(f, any_functor_at_all) {
  return any_functor_at_all.map(f);
});
```
In questo modo siamo in grado di usare la composizione e `map` funzionerà come previsto. Questo è il caso di `map` implementato nella libreria Ramda. Useremo la dot notation quando è istruttivo e la versione pointfree quando è conveniente. Ho furtivamente introdotto una notazione in più nella nostra type signature. Funtor F `=>` ci informa che `F` deve essere un funtore.

## Casi d' uso

`Maybe` viene usato quando ci sono funzioni che dovrebbero ritornare un valore ma potrebbero fallire.

```js
//  safeHead :: [a] -> Maybe(a)
var safeHead = function(xs) {
  return Maybe.of(xs[0]);
};

var streetName = compose(map(_.prop('street')), safeHead, _.prop('addresses'));

streetName({
  addresses: [],
});
// Maybe(null)

streetName({
  addresses: [{
    street: 'Shady Ln.',
    number: 4201,
  }],
});
// Maybe("Shady Ln.")
```

`SafeHead` è come il nostro normale` _.head`, ma con l' aggiunta della sicurezza di tipo. Una cosa curiosa accade quando `Maybe` viene introdotto nel nostro codice; siamo costretti a fare i conti con quei subdoli valori `null`. La funzione `safeHead` è onesta e anticipa i suoi possibili fallimenti: restituisce un `Maybe` per informarci di questa possibilità. In reltà siamo più che semplicemente informati, infatti siamo costretti a `map`pare per ottenere il valore che vogliamo dal momento che è nascosto dentro l'oggetto `Maybe`. In sostanza, si tratta di un `null check` forzato dalla funzione `safeHead` stessa. Ora possiamo dormire sonni tranquilli sapendo nessun `null` sollevarà la sua ripugnante testa decapitata quando meno ce lo aspettiamo. API come questa trasformano un' applicazione fragile di carta e puntine in una robusta di legno e chiodi. Quindi garantiscono software più sicuro.

A volte una funzione potrebbe restituire un `Maybe(null)` esplicitamente per segnalare un fallimento. Per esempio:

```js
//  withdraw :: Number -> Account -> Maybe(Account)
var withdraw = curry(function(amount, account) {
  return account.balance >= amount ?
    Maybe.of({
      balance: account.balance - amount,
    }) :
    Maybe.of(null);
});

//  finishTransaction :: Account -> String
var finishTransaction = compose(remainingBalance, updateLedger); // <- these composed functions are hypothetical, not implemented here...

//  getTwenty :: Account -> Maybe(String)
var getTwenty = compose(map(finishTransaction), withdraw(20));


getTwenty({
  balance: 200.00,
});
// Maybe("Your balance is $180.00")

getTwenty({
  balance: 10.00,
});
// Maybe(null)
```

Se siamo a corto di denaro `Withdraw` ritorna `Maybe(null)`. Questa funzione comunica anche la sua instabilità e non ci lascia altra scelta che `map`pare tutto alla fine. Qui `Null` era intenzionale. Invece di un `Maybe(String) `, otteniamo un `Maybe(null)` per segnalare guasti e la nostra applicazione si arresta in modo efficace. Importante da notare: se non riesce il `withdraw`, allora `map` taglierà il resto della nostra computazione in quanto non eseguirà la funzione mappata 'finishTransaction`. Questo è esattamente il comportamento che vorremmo ottenere: se non avessimo ritirato con successo fondi, preferiremmo non aggiornare il nostro libro mastro o mostrare un nuovo bilancio.

## Rilascio del valore

Una cosa che spesso si dimentica è che c'è sempre una fine della linea; una funzione che produce qualcosa: invia JSON, stampa a schermo, altera il filesystem o chi più ne ha più ne metta. Non possiamo inviare l' output con `return` dobbiamo eseguire una funzione che lo spedisca fuori nel mondo esterno. Come disse il Buddista Zen koan: "Se un programma non ha effetti osservabili allora è davvero in escuzione?". Funziona solo per la propria soddisfazione personale? Io sospetto che serva solo a bruciare qualche ciclo per poi tornarsene a dormire.

Il lavoro della nostra applicazione è quello di recuperare, trasformare e trasportare i dati fino a quando è il momento di dir loro addio, la funzione che fa questo può essere mappata, quindi il valore non deve neanche lasciare il grembo caldo del suo contenitore. In effetti, un errore comune è quello di cercare di rimuovere il valore della nostra `Maybe` in un modo o nell'altro, come se improvvisamente si materializzi per incanto. Dobbiamo però ricordare che potrebbe esserci un ramo di codice dove il nostro valore non è sopravvissuto al suo destino. Il nostro codice, proprio come il gatto di Schrödinger, è in due stati contemporaneamente e deve mantenere questa condizione fino alla funzione finale. Questo dà il nostro codice un flusso lineare nonostante la ramificazione logica.

Vi è, tuttavia, una via di fuga. Se volessimo restituire un valore personalizzato e proseguire, si potremmo utilizzare un piccolo aiutante chiamato `Maybe`.

```js
//  maybe :: b -> (a -> b) -> Maybe a -> b
var maybe = curry(function(x, f, m) {
  return m.isNothing() ? x : f(m.__value);
});

//  getTwenty :: Account -> String
var getTwenty = compose(
  maybe("You're broke!", finishTransaction), withdraw(20)
);


getTwenty({
  balance: 200.00,
});
// "Your balance is $180.00"

getTwenty({
  balance: 10.00,
});
// "You're broke!"
```

Ora noi potremmo o restituire un valore statico (dello stesso tipo che ritorna 'finishTransaction' ) oppure continuare finendo allegramente la transazione senza 'Maybe'. Con 'Maybe' assistiamo ad un analogo di un 'if/else' mentre con 'map' l' analogo imperativo sarebbe: 'if (x !== null) { return f(x) }'.

L'introduzione di `Maybe` può causare qualche disagio iniziale. Gli utenti di Swift e Scala sanno cosa intendo: è integrato direttamente nelle librerie di base sotto la maschera di `Opzion(al)`. Quando bisogna gestire `null check` dappertutto (e ci sono casi in cui sappiamo con assoluta certezzache il valore esiste), la maggior parte delle persone non hanno altra scelta ma sentono comunque la difficoltà di scrivere un tale codice. Tuttavia, con il tempo, diventerà una seconda natura ed è probabile che si cominci ad apprezzarne la sicurezza. Alla fine porterà a evitere angoli taglienti e salvarci la pelle.

Scrivere software non sicuro è come dipingere ogni uovo con pastelli e poi tirarli nel traffico; come costruire la casa con materiali dei primi due protagonisti della favola dei tre porcellini. Si fa sempre bene a mettere un po' di sicurezza nelle nostre funzioni e `Maybe` ci aiuta a fare proprio in questo.

L' implementazione reale divide `Maybe` in due tipi: uno per qualcosa e l'altro per niente. Questo ci permette di gestirli parametricamente in `map` così valori come `null` e `undefined` potranno ancora essere mappati e la qualificazione universale di funtore sarà rispettata. Si vedono spesso tipi come `Some(x)/None` o `Just(x)/Nothing` invece di un `Maybe` che fa solo un `null check` sui suoi valori.

## Error Handling Puro

<img src="images/fists.jpg" alt="pick a hand... need a reference" />

potrebbe sembrare scioccante ma il `try/catch` non è molto puro. Quando avviene un errore, al posto di restituire un valore di output, partono gli allarmi! La funzioni attaccano vomitando migliaia di 0 e 1 come in una battaglia all' ultimo sangue contro l' imput immesso dall' utente. Con il nostro nuovo amico `Either` possiamo fare di meglio che dichiarare guerra all' input, possimo rispondere con un messaggio diplomatico. Diamo un occhiata:

```js
var Left = function(x) {
  this.__value = x;
};

Left.of = function(x) {
  return new Left(x);
};

Left.prototype.map = function(f) {
  return this;
};

var Right = function(x) {
  this.__value = x;
};

Right.of = function(x) {
  return new Right(x);
};

Right.prototype.map = function(f) {
  return Right.of(f(this.__value));
}
```

`Left` e `Right` sono due sottoclassi un tipo astratto che chiamiamo `Either`. Ho saltato la cerimonia di creare la superclasse `Either` dato che non la useremo mai, ma è bene esserne consapevoli. Ora, non c'è niente di nuovo, oltre i due tipi. Vediamo come si comportano:

```js
Right.of('rain').map(function(str) {
  return 'b' + str;
});
// Right('brain')

Left.of('rain').map(function(str) {
  return 'b' + str;
});
// Left('rain')

Right.of({
  host: 'localhost',
  port: 80,
}).map(_.prop('host'));
// Right('localhost')

Left.of('rolls eyes...').map(_.prop('host'));
// Left('rolls eyes...')
```

`Left` è come un teenager in crisi e ignora tutte le nostre richieste `map`. `Right` funzionera proprio come `Container` (a.k.a identità). La figata del procedimento deriva dalla capacità di incorporare un messaggio di errore all'interno del `Left`.

Supponiamo di avere una funzione che potrebbe non avere successo, per esempio quella che restituisce l'età data la data di nascita. Potremmo usare `Maybe(null)` per segnalare guasti e creare un nuovo ramo del nostro programma, tuttavia, non ci dice molto. Ci piacerebbe sapere cosa è andato storto. Proviamo scriverlo con `Either`.

```js
var moment = require('moment');

//  getAge :: Date -> User -> Either(String, Number)
var getAge = curry(function(now, user) {
  var birthdate = moment(user.birthdate, 'YYYY-MM-DD');
  if (!birthdate.isValid()) return Left.of('Birth date could not be parsed');
  return Right.of(now.diff(birthdate, 'years'));
});

getAge(moment(), {
  birthdate: '2005-12-12',
});
// Right(9)

getAge(moment(), {
  birthdate: '20010704',
});
// Left('Birth date could not be parsed')
```
Ora, proprio come `Maybe(null)`, stiamo corto circuitando la nostra applicazione quando restituiamo un `Left`. La differenza, è ora abbiamo un indizio sul motivo per cui il nostro programma è deragliato. Da notare che viene restituito `Either(String, Number)`, che detiene uno 'string' come valore 'Left' e un 'number' come valore 'Right'. Questa type signature è un po' informale, in quanto non abbiamo avuto il tempo di definire una vera e propria superclasse 'Either', tuttavia, si apprende molto dal tipo. Esso ci informa che potremmo ottenere o un messaggio di errore o l'età esatta.

```js
//  fortune :: Number -> String
var fortune = compose(concat('If you survive, you will be '), add(1));

//  zoltar :: User -> Either(String, _)
var zoltar = compose(map(console.log), map(fortune), getAge(moment()));

zoltar({
  birthdate: '2005-12-12',
});
// 'If you survive, you will be 10'
// Right(undefined)

zoltar({
  birthdate: 'balloons!',
});
// Left('Birth date could not be parsed')
```
Quando la data di nascita è valida, il programma emette l' output. In caso contrario, viene restituito un `Left` con il messaggio di errore chiaro come il sole anche se ancora nascosto nel suo contenitore. Questo funziona proprio come se avessimo generato un errore, ma in una maniera calma, lieve e misurata che si oppone alla perdita totale di controllo che ti fa urlare come un bambino quando qualcosa va storto.

In questo esempio, stiamo logicamente ramificazione nostro flusso di controllo a seconda della validità della data di nascita, ma si può comunque leggere come un movimento lineare da destra a sinistra, piuttosto che salendo attraverso le parentesi graffe di un'istruzione condizionale. Di solito, ci piacerebbe spostare il 'console.log' fuori dalla nostra funzione 'zoltar' e 'map'parlo al momento giusto, ma è utile per vedere come si differenzia il ramo `Right`. Usiamo `_` nella type signature del ramo di destra per indicare che è un valore che deve essere ignorato (in alcuni browser è necessario utilizzare `console.log.bind(console)`).

Colgo l'occasione per sottolineare una cosa che non va tralasciata: `fortune`, nonostante il suo uso con` Either` in questo esempio, è del tutto ignaro dei funtori su cui opera. La stessa cosa che accade per `finishTransaction` nell'esempio precedente. Al momento della chiamata, una funzione può essere circondato da `map`, che la trasforma da una funzione non funtoriale ad un funtore, informalmente. Chiamiamo questo processo *lifting*. Le funzioni tendono ad essere migliori se lavorano con tipi di dato normali piuttosto che tipi di contenitori, poi sollevati nel contenitore giusto. Questo porta a funzioni più semplici, più riutilizzabili che possono facilmente essere modificate per lavorare con qualsiasi funtore su richiesta.

`Either` è grande sia per gli errori casuali come la convalida dell' input che per quelli più gravi ( da fermate tutto Marisa prend nome e cognome domani processo per direttissima cit.) come file o socket rotti mancanti. Prova a sostituire alcuni degli esempi `Maybe` con Either` per dare un feedback migliore.

Ora, non posso farne a meno, sento di aver fatto a Either un torto presentandolo solo come un semplice contenitore di messaggi di errore. Esso cattura disgiunzione logica (a.k.a `||`) in un tipo, codifica anche l'idea di *Coprodotto* dalla teoria delle categorie, che non sarà toccato in questo libro, ma vale la pena leggere per imparare a sfruttare le sue proprietà. E' il tipo della somma canonica (o unione disgiunta di insiemi) perchè l' insieme dei possibili valori è la somma dei due tipi contenuti (lo so che è un po' ostico ma ecco un [grande articolo](https://www.fpcomplete.com/school/to-infinity-and-beyond/pick-of-the-week/sum-types)). `Either` può essere molte cose, ma come un funtore, è utilizzato per la sua gestione degli errori.

Proprio come con `Maybe`, abbiamo `either` minuscolo, che si comporta in modo simile, ma prende due funzioni invece di una e un valore statico. Ogni funzione deve restituire lo stesso tipo:

```js
//  either :: (a -> c) -> (b -> c) -> Either a b -> c
var either = curry(function(f, g, e) {
  switch (e.constructor) {
    case Left:
      return f(e.__value);
    case Right:
      return g(e.__value);
  }
});

//  zoltar :: User -> _
var zoltar = compose(console.log, either(id, fortune), getAge(moment()));

zoltar({
  birthdate: '2005-12-12',
});
// "If you survive, you will be 10"
// undefined

zoltar({
  birthdate: 'balloons!',
});
// "Birth date could not be parsed"
// undefined
```

Infine, un utilizzo della misteriosa funzione `id`. Essa ripete a pappagallo il valore nel `Left` per passare il messaggio di errore a `console.log`. Abbiamo reso la nostra applicazione più robusta forzando la gestione dell' errore all' interno di `getAge`. Abbiamo o schiaffato in faccia all'utente la dura verità riguardo all' errore da lui commesso (come un dammi il cique dato da un chiromante (lettore della mano)) oppure eseguito le azioni che dovevamo svolgere. E con questo, siamo pronti a passare a un tipo completamente diverso tipo di funtore.

## Old McDonald had Effects...

<img src="images/dominoes.jpg" alt="dominoes.. need a reference" />

Nel nostro capitolo sulla purezza abbiamo visto una funzione che conteneva un effetto collaterale, ma, siamo riusciti a renderla pura avvolgendo la sua azione in un'altra funzione. Eccne un altro esempio:

```js
//  getFromStorage :: String -> (_ -> String)
var getFromStorage = function(key) {
  return function() {
    return localStorage[key];
  };
};
```

Se non avessimo circondato le sue budella in un'altra funzione, `getFromStorage` varierebbe il suo output a seconda del' ambiente esterno. Invece con questo robusto l'involucro, otteremo sempre lo stesso output per lo stesso input: una funzione che, quando viene chiamata, recuperarà un particolare elemento da `localStorage`. Così facendo (magari con anche qualche Ave Maria) abbiamo pulito la nostra coscienza e tutto è stato perdonato.

Peccato che, questo non è particolarmente utile vero. Come una action figure da collezione nella sua confezione originale, non possiamo realmente giocare con essa. Se solo ci fosse un modo per raggiungere all'interno del contenitore e arrivare al suo contenuto ... Entra in gioco `IO`.

```js
var IO = function(f) {
  this.__value = f;
};

IO.of = function(x) {
  return new IO(function() {
    return x;
  });
};

IO.prototype.map = function(f) {
  return new IO(_.compose(f, this.__value));
};
```

`IO` si differenzia dai precedenti funtori perchè `__value` è una funzione. Questo fatto è solo un dettaglio di implementazione e faremmo meglio a ignorarlo. Quello che sta accadendo è esattamente quello che abbiamo visto con l'esempio `getFromStorage`:` IO` ritarda l'azione impura catturandola in una funzione wrapper. `IO` come contiene il valore di ritorno dell'azione avvolto e non è il wrapped stesso. Questo è evidente nella funzione `of`: abbiamo un `IO(x)`, `IO ( function(){return x} )` è necessario solo per evitare la valutazione.

Vediamo come funziona:

```js
//  io_window :: IO Window
var io_window = new IO(function(){ return window; });

io_window.map(function(win){ return win.innerWidth });
// IO(1430)

io_window.map(_.prop('location')).map(_.prop('href')).map(split('/'));
// IO(["http:", "", "localhost:8000", "blog", "posts"])


//  $ :: String -> IO [DOM]
var $ = function(selector) {
  return new IO(function(){ return document.querySelectorAll(selector); });
}

$('#myDiv').map(head).map(function(div){ return div.innerHTML; });
// IO('I am some inner html')
```

Qui, `io_window` è un vero e proprio `IO` e siamo in grado di `map`parci sopra immediatamente, mentre`$` è una funzione che restituisce un `IO` dopo la sua chiamata. Ho scritto i valori di ritorno *concettuale*  di ritorno per esprimere al meglio il `IO`, anche se, in realtà, sarà sempre` {__value: [Funzione]} `. Quando `map`piamo su `IO`, faremo tale funzione al termine di una composizione che, a sua volta, diventa il nuovo `__value` e così via. Le nostre funzioni mappate non vengonono eseguite, vengono appiccicate alla fine della computazione che stiamo costruendo, funzione per funzione, come tessere del domino poste con attenzione senza farle cadere. Il risultato ricorda il command pattern della Gang of Four's o una coda.

Prendetevi un momento per riflettere sui funtorei. Se sorvoliamo i dettagli di implementazione, dovremmo sentire come a casa a mappare su qualsiasi contenitore, non importa quanto sia strano. Abbiamo le leggi dei funtori, che esploreremo verso la fine del capitolo, per ringraziare per questo potere pseudo-psichico. In ogni caso, possiamo finalmente giocare con valori impuri senza sacrificare la nostra preziosa purezza.

Ora, noi abbiamo ingabbiato la bestia, ma prima o poi dovremmo liberarla. Mappare su `IO` ha costruito una struttura molto impura e eseguendola andremo a disturbare la quiete. Allora, dove e quando possiamo premere il grilletto? E' possibile eseguire il nostro `IO` e presentarci al matrimonio in abito bianco? La risposta è sì, se mettiamo l'onere sul codice chiamante. Il nostro codice puro, nonostante il tracciato nefasto e intrigante, mantiene la sua innocenza ed è il chiamante che viene gravato con la responsabilità di esecuzione gli effetti. Vediamo un esempio concreto.


```js

////// Our pure library: lib/params.js ///////

//  url :: IO String
var url = new IO(function() { return window.location.href; });

//  toPairs =  String -> [[String]]
var toPairs = compose(map(split('=')), split('&'));

//  params :: String -> [[String]]
var params = compose(toPairs, last, split('?'));

//  findParam :: String -> IO Maybe [String]
var findParam = function(key) {
  return map(compose(Maybe.of, filter(compose(eq(key), head)), params), url);
};

////// Impure calling code: main.js ///////

// run it by calling __value()!
findParam("searchTerm").__value();
// Maybe([['searchTerm', 'wafflehouse']])
```
La nostra libreria mantiene le mani pulite avvolgendo `url` in un` IO` e passando la patata bollente al chiamante. Potresti aver anche notato che abbiamo impilato i contenitori; E' perfettamente ragionevole per avere un `IO(Maybe([x]))`, che è un' espressione funtoriale profonda 3 ( `Array` è sicuramente un tipo di container mappabile) inoltre è eccezionalmente espressiva.

C'è qualcosa che mi tormenta e che dovremmo subito rimediare: ` __value` di `IO` non è il suo valore contenuto, e neanche una proprietà privata come suggerisce il prefisso underscore. È la sicura della granata ed è destinato ad essere usato da un chiamante nel più pubblico dei modi. Rinominiamo questa proprietà `unsafePerformIO` ricordare la sua volatilità.

```js
var IO = function(f) {
  this.unsafePerformIO = f;
}

IO.prototype.map = function(f) {
  return new IO(_.compose(f, this.unsafePerformIO));
}
```

Ora il nostro codice chiamante diventa `findParam("searchTerm").unsafePerformIO()`, che è chiara come il giorno per gli utenti (e i lettori) dell'applicazione.

`IO` sarà un compagno fedele, che ci auitarà a domare quelle azioni impure selvatiche. Successivamente vedremo un funtore molto simile nello spirito, ma con applicazioni drasticamente diverse.

## Asynchronous Tasks

Callback sono la scala a chicciola che restringendosi scende inesorabilmente verso l'inferno. Sono state progettate da M.C. Escher per il controllo di flusso. Con tutte queste callback annidate e schiacciate in una giungla di parentesi graffe e tonde sembra di fare il limbo in un sotterraneo (quanto in basso riesciamo a scendere!). Mi vengono i brividi di claustrofobia al solo pensarci. Non c'è da preoccuparsi, abbiamo un modo molto migliore di trattare il codice asincrono e comincia con una "F".

Gli interni sono un po' troppo complicato da esaminare in questa pagina useremo `Data.Task` (in precedenza `Data.Future`) dal fantastico [fiaba] di Quildreen Motta (http://folktalejs.org/). Ecco qualche esempio:

```js
// Node readfile example:
//=======================

var fs = require('fs');

//  readFile :: String -> Task Error String
var readFile = function(filename) {
  return new Task(function(reject, result) {
    fs.readFile(filename, 'utf-8', function(err, data) {
      err ? reject(err) : result(data);
    });
  });
};

readFile("metamorphosis").map(split('\n')).map(head);
// Task("One morning, as Gregor Samsa was waking up from anxious dreams, he discovered that
// in bed he had been changed into a monstrous verminous bug.")


// jQuery getJSON example:
//========================

//  getJSON :: String -> {} -> Task Error JSON
var getJSON = curry(function(url, params) {
  return new Task(function(reject, result) {
    $.getJSON(url, params, result).fail(reject);
  });
});

getJSON('/video', {id: 10}).map(_.prop('title'));
// Task("Family Matters ep 15")

// We can put normal, non futuristic values inside as well
Task.of(3).map(function(three){ return three + 1 });
// Task(4)
```
Le funzioni Sto chiamando `reject` e` result` sono i nostri callback di errore e successo rispettivamente. Come potete vedere, abbiamo semplicemente `map`pato su` Task` per utilizzare il valore futuro come se fosse proprio già lì nella nostra portata. Ormai `map` dovrebbe essere vecchio cappello.

Se si ha familiarità con le promesse, si potrebbe riconoscere che la funzione di `map` su `then` con `Task` gioca il ruolo della nostra promessa. Non ti preoccupare se non hai familiarità con le promesse, non saremo li utilizzano in ogni caso, perché non sono puri, ma l'analogia tiene comunque.

Come `IO`, `Task` pazientemente aspetta il semaforo verde prima di partire. In realtà, dato che attende il nostro comando, `IO` è effettivamente sostituito da `Task` per tutte le cose asincrone; `ReadFile` e `getJSON` non richiedono un contenitore `IO` in più per essere puro. `Task` funziona in modo simile quando `map`piamo su di essa: stiamo ponendo le istruzioni per il futuro, come un lavoretto in una capsula del tempo - un atto di sofisticata procrastinazione tecnologica.

Per eseguire il nostro `Task`, dobbiamo chiamare il metodo `fork`. Questo funziona come `unsafePerformIO`, ma come suggerisce il nome, biforca il nostro processo e l'esecuzione continua senza bloccare il nostro thread. Può essere implementato in molti modi con threads e simili, ma qui si comporta come un normale chiamata asincrona e la grande ruota del ciclo degli eventi continua a girare. Diamo un'occhiata a `fork`:

```js
// Pure application
//=====================
// blogTemplate :: String

//  blogPage :: Posts -> HTML
var blogPage = Handlebars.compile(blogTemplate);

//  renderPage :: Posts -> HTML
var renderPage = compose(blogPage, sortBy('date'));

//  blog :: Params -> Task Error HTML
var blog = compose(map(renderPage), getJSON('/posts'));


// Impure calling code
//=====================
blog({}).fork(
  function(error){ $("#error").html(error.message); },
  function(page){ $("#main").html(page); }
);

$('#spinner').show();
```

Dopo aver chiamato `fork`, `Task` si affretta a cercare qualche post e aggiornare la pagina. Nel frattempo,dato che `fork` non attende una risposta, mostriamo una girella. Infine, visualizzeremo o un errore oppure il rendering della pagina a seconda che la chiamata `getJSON` riesca o meno.

Prendetevi un momento per considerare la linearità del flusso di controllo in questo esempio. Dobbiamo solo leggere dal basso verso l'alto, da destra a sinistra, anche se il programma salterà un po' durante l'esecuzione. Questo rende la lettura e il ragionamento circa la nostra applicazione più semplice pittosto che dover rimbalzare tra callback e blocchi di error handling.

Si può notare che `Task` ha inghiottito` Either`! Lo deve fare per gestire fallimenti futuri dato che il nostro normale controllo di flusso non si applica nel mondo asincrono. Inoltre questo fornisce un sufficiene e puro error handling.

Anche con `Task`, i nostri funtori `IO` e `Either` non sono fuori dai giochi. Questo esempio un po' più ipotetico e complesso, è molto utile per scopo illustrativo.

```js
// Postgres.connect :: Url -> IO DbConnection
// runQuery :: DbConnection -> ResultSet
// readFile :: String -> Task Error String

// Pure application
//=====================

//  dbUrl :: Config -> Either Error Url
var dbUrl = function(c) {
  return (c.uname && c.pass && c.host && c.db)
    ? Right.of("db:pg://"+c.uname+":"+c.pass+"@"+c.host+"5432/"+c.db)
    : Left.of(Error("Invalid config!"));
}

//  connectDb :: Config -> Either Error (IO DbConnection)
var connectDb = compose(map(Postgres.connect), dbUrl);

//  getConfig :: Filename -> Task Error (Either Error (IO DbConnection))
var getConfig = compose(map(compose(connectDb, JSON.parse)), readFile);


// Impure calling code
//=====================
getConfig("db.json").fork(
  logErr("couldn't read file"), either(console.log, map(runQuery))
);
```

In questo esempio, facciamo ancora uso di `Either` e `IO` dall'interno del ramo successo di `readFile`. `Task` si prende cura delle impurità di lettura di un file in modo asincrono, ma dobbiamo ancora affrontare la convalida della configurazione con `Either` e litigare la connessione al db con `IO`. Quindi, siamo ancora in ballo con tutte le cose sincrone.

Potrei andare avanti, ma questo è tutto ciò che devi fare. Semplice come `map`.

Nella pratica, è probabile che si abbiano più attività asincrone nello stesso flusso di lavoro ma noi non abbiamo ancora aquisito tutti gli strumenti per affrontare questo scenario. Non c'è da preoccuparsi, guarderemo presto monadi e simili, ma prima, dobbiamo esaminare la matematica che rendono tutto questo possibile.


## A Spot of Theory

Come accennato prima, i funtori provengono dalla teoria delle categorie e soddisfano alcune leggi. Cominciamo a esplorare queste utili proprietà.

```js
// identity
map(id) === id;

// composition
compose(map(f), map(g)) === map(compose(f, g));
```

The *identity* law is simple, but important. These laws are runnable bits of code so we can try them on our own functors to validate their legitimacy.

```js
var idLaw1 = map(id);
var idLaw2 = id;

idLaw1(Container.of(2));
//=> Container(2)

idLaw2(Container.of(2));
//=> Container(2)
```

Vedete, sono uguali. Prossimo diamo un'occhiata a composizione.

```js
var compLaw1 = compose(map(concat(" world")), map(concat(" cruel")));
var compLaw2 = map(compose(concat(" world"), concat(" cruel")));

compLaw1(Container.of("Goodbye"));
//=> Container(' world cruelGoodbye')

compLaw2(Container.of("Goodbye"));
//=> Container(' world cruelGoodbye')
```

Nella teoria delle categorie, i funtori prendono gli oggetti e morfismi di una categoria e li mappano in una categoria diversa. Per definizione, questa nuova categoria deve avere una identità e la capacità di comporre morfismi, ma non abbiamo bisogno di controllare perché le leggi di cui sopra ci assicurano che gli stessi sono conservati.

Forse la nostra definizione di una categoria è ancora un po' confusa. Si può pensare a una categoria come una rete di oggetti con morfismi che li collegano. Un funtore mappa una categoria in un' altra senza rompere la rete. Se un oggetto `a` nella categoria `C`, quando lo mappiamo nella categoria `D` attraverso un funtore 'F', ci riferiamo a questo oggetto come `F a` (Se lo metti insieme come si pronuncia?!). Forse, è meglio guardare un diagramma:

<img src="images/catmap.png" alt="Categories mapped" />

Per esempio, `Maybe` mappa la nostra categoria di tipi e funzioni di una categoria dove ogni oggetto può non esistere ed ogni morfismo ha un `null` check. Lo implementiamo nel codice circondando ogni funzione con `map` e ogni tipo con il nostro funtore. Sappiamo che ognuno dei nostri tipi e funzioni normali continueranno a comporsi in questo nuovo mondo. Tecnicamente, ogni funtore nel nostro codice mappa in una sottocategoria di tipi e funzioni, questo li rende tutti (i funtori ) degli endofuntori, ma per i nostri scopi, la penseremo come una categoria diversa.

Possiamo anche visualizzare la mappa di  morfismi e dei rispettivi oggetti corrispondenti con questo schema:

<img src="images/functormap.png" alt="functor diagram" />

Oltre a visualizzare il morfismo mappato da una categoria all'altra sotto il funtore `F`, vediamo che il diagramma commuta, vale a dire, se si seguono le frecce, ogni percorso produce lo stesso risultato. I diversi percorsi significa un comportamento diverso, ma si finisce sempre allo stesso tipo. Questo formalismo  fornisce un nuovo modo di ragionare su nostro codice - possiamo applicare con coraggio formule senza dover analizzare ed esaminare ogni singolo scenario. Facciamo un esempio concreto.

```js
//  topRoute :: String -> Maybe String
var topRoute = compose(Maybe.of, reverse);

//  bottomRoute :: String -> Maybe String
var bottomRoute = compose(map(reverse), Maybe.of);


topRoute("hi");
// Maybe("ih")

bottomRoute("hi");
// Maybe("ih")
```

O graficamente:

<img src="images/functormapmaybe.png" alt="functor diagram 2" />

Siamo in grado di vedere e ricostruire immediatamente e il codice basanoci su proprietà che hanno tutti i funtori.

Functors si possono impilare:

```js
var nested = Task.of([Right.of("pillows"), Left.of("no sleep for you")]);

map(map(map(toUpperCase)), nested);
// Task([Right("PILLOWS"), Left("no sleep for you")])
```

Quello che otteniamo con `nested` è un array futuro di elementi che potrebbero essere errori.`map`iamo ogni strato ed eseguiamo la nostra funzione sugli elementi. Non c'è traccia di `callback`, `if/else`, o cicli `for`; solo un contesto esplicito. Tuttavia, dobbiamo usare `map(map(map(f)))`. Possiamo comporre funtori? Sì, vediamo:

```js
var Compose = function(f_g_x){
  this.getCompose = f_g_x;
}

Compose.prototype.map = function(f){
  return new Compose(map(map(f), this.getCompose));
}

var tmd = Task.of(Maybe.of("Rock over London"))

var ctmd = new Compose(tmd);

map(concat(", rock on, Chicago"), ctmd);
// Compose(Task(Maybe("Rock over London, rock on, Chicago")))

ctmd.getCompose;
// Task(Maybe("Rock over London, rock on, Chicago"))
```

Solo un `map`. La composizione di funtori è associativa e precedentemente, abbiamo definito `Container`, che in realtà è chiamato il funtore `Identity`. Se abbiamo l'identità e la composizione associativa abbiamo una categoria. Questa particolare categoria ha categorie come gli oggetti e funtori come morfismi, il che è abbastanza per farci sudare il cervello. Non indagheremo troppo a fondo questa proprietà, ma è bello apprezzarne la semplice e astratta bellezza e le sue implicazioni architettoniche.


## Riassumendo

Abbiamo visto un po' di funtori diversi, ma ce ne sono infiniti. Alcune omissioni notevoli sono le strutture dati iteabili come: alberi, liste, mappe, coppie, e chi più ne ha più ne metta. Gli eventstream e gli observable sono entrambi funtori. Altri potrebbereo servire per l' encapsulation o solo per la modellizazione di tipi. I funtori sono tutti attorno a noi, e noi li useremo estensivamente nel resto del libro.

chiamare una funzione con più funtori come argomenti? lavorare con una sequenza ordinata di azioni impure o asincrone? Non abbiamo ancora acquisito il set completo di strumenti per lavorare in questo mondo impacchettato. Andremo dritti al punto studiando le monadi. 

[Chapter 9: Monadic Onions](ch9.md)

## Esercizi

```js
require('../../support');
var Task = require('data.task');
var _ = require('ramda');

// Esercizio 1
// ==========
// Usa _.add(x,y) e _.map(f,x) per creare una funzione che incrementa un valore 
// dentro un funtore

var ex1 = undefined



// Exercizio 2
// ==========
// Usa _.head per ottenere il primo elemento della lista.

var xs = Identity.of(['do', 'ray', 'me', 'fa', 'so', 'la', 'ti', 'do']);

var ex2 = undefined



// Esercizio 3
// ==========
// Usa safeProp e _.head per trovare la prima iniziale dell' utente.

var safeProp = _.curry(function (x, o) { return Maybe.of(o[x]); });

var user = { id: 2, name: "Albert" };

var ex3 = undefined


// Esercizio 4
// ==========
// Usa Maybe per riscrivere l' esercizio 4 senza il costrutto 'if'.

var ex4 = function (n) {
  if (n) { return parseInt(n); }
};

var ex4 = undefined



// Esercizio 5
// ==========
// scrivi una funzione che prende un post (getPost) 
// e metta in maiusolo(toUpperCase) il titolo.

// getPost :: Int -> Future({id: Int, title: String})
var getPost = function (i) {
  return new Task(function(rej, res) {
    setTimeout(function(){
      res({id: i, title: 'Love them futures'})  
    }, 300)
  });
}

var ex5 = undefined



// Esercizio 6
// ==========
// Scrivi una funzione che usa checkActive() per controllare le credenziali,
// se avviene l' accesso esegue showWelcome(),
// altrimenti restituisce un errore.

var showWelcome = _.compose(_.add( "Welcome "), _.prop('name'))

var checkActive = function(user) {
 return user.active ? Right.of(user) : Left.of('Your account is not active')
}

var ex6 = undefined



// Esercizio 7
// ==========
// scrivi una funzione di validazione che controlli la lunghezza > 3. Deve restituire
// Right(x) se è maggiore di 3 e Left("You need > 3") altrimenti.

var ex7 = function(x) {
  return undefined // <--- write me. (don't be pointfree)
}



// Esercizio 8
// ==========
// Usa l' esercizio 7 e Either come funtore per salvare l' utentza se è valida oppure
// ritorna una stringa con il messaggio di errore.
// Ricorda i due argomenti di Either devono essere dello stesso tipo.


var save = function(x){
  return new IO(function(){
    console.log("SAVED USER!");
    return x + '-saved';
  });
}

var ex8 = undefined
```