---
creation_date: 24 februari 2022
---

# Group by in Javascript

## Introductie

Voor [de blog over mijn vlieglessen](https://mandarin.nl/vlieglessen/) maak ik gebruik van Javascript om de opgeslagen gpx-data visueel weer te geven. Het plotten van de gevlogen route is redelijk arbitrair: eerst gebruik ik [deze site](http://garmin.stevegordon.co.uk/) om het fit-bestand dat de Garmin Express genereert om in een standaard GPX-bestand. Vervolgens parseer ik dat bestand met behulp van [`GPXParser`](https://luuka.github.io/GPXParser.js/) om het resultaat uiteindelijk op [openstreetmap](https://www.openstreetmap.org/) te plotten:

```javascript
fetch(`flights/flight${f_number}.gpx`) //f_number is het nummer van de vlucht
.then ( r => r.text() )
.then ( gpx => {
    let parser = new gpxParser();
    parser.parse(gpx);
    let coordinates = parser.tracks[0].points.map(p => [p.lat.toFixed(5), p.lon.toFixed(5)]);
    var polyline = L.polyline(coordinates, { weight: 2, color: 'darkblue' }).addTo(mymap);

    // zoom the map to the polyline
    mymap.fitBounds(polyline.getBounds());
})
```

## Vreemde metingen

Het plotten van een grafiekje van *hoogte* bleek evenwel een stuk ingewikkelder. Ik wilde eigenlijk zo'n zelfde interactie hebben als op [strava](https://www.strava.com/athletes/bartbarnard), waarbij je over de weergegeven hoogte kunt gaan en dan op de kaart zien waar je die hoogte had. Strava zelf begrijpt deze gpx-data niet (je krijgt een melding van 'your Garmin seemed to have a bad day'), dus ik moest het zelf maar maken.

Om de plot te maken wilde ik gebruik maken van [chartJS](https://www.chartjs.org/) op een manier die beschreven stond op [deze post op StackOverflow](https://stackoverflow.com/a/43658507/10974490). Bij het bestuderen van de GPX-data bleek evenwel dat er geen touw aan vast te knopen was wanneer er een meting gedaan werd. Het dataformaat zelf is redelijk transparant, maar kijk in het fragment hieronder eens naar de meet-momenten (ik heb hier wat dingen weggelaten, omdat het anders te groot werd; `trkp` staat, neem ik aan, voor *trackpoint*):


```html hl_lines="3 7 11 15 19 24 28 32 36 40"
 <trkpt lat="53.1200012" lon="6.1368227">
    <ele>-1.6</ele>
    <time>2021-10-18T13:05:50Z</time>
   </trkpt>
   <trkpt lat="53.1200012" lon="6.1368227">
    <ele>-1.6</ele>
    <time>2021-10-18T13:06:01Z</time>
   </trkpt>
   <trkpt lat="53.1199994" lon="6.1368221">
    <ele>-1.8</ele>
    <time>2021-10-18T13:06:17Z</time>
   </trkpt>
   <trkpt lat="53.1199994" lon="6.1368221">
    <ele>-1.8</ele>
    <time>2021-10-18T13:06:33Z</time>
   </trkpt>
   <trkpt lat="53.1199994" lon="6.1368221">
    <ele>-1.8</ele>
    <time>2021-10-18T13:06:43Z</time>
   </trkpt>
   ...
    <trkpt lat="53.1196990" lon="6.1361980">
    <ele>-0.6</ele>
    <time>2021-10-18T13:15:41Z</time>
   </trkpt>
   <trkpt lat="53.1195914" lon="6.1359441">
    <ele>-0.6</ele>
    <time>2021-10-18T13:15:50Z</time>
   </trkpt>
   <trkpt lat="53.1195264" lon="6.1359676">
    <ele>-0.6</ele>
    <time>2021-10-18T13:15:59Z</time>
   </trkpt>
   <trkpt lat="53.1195264" lon="6.1359676">
    <ele>-0.8</ele>
    <time>2021-10-18T13:16:09Z</time>
   </trkpt>
   <trkpt lat="53.1195264" lon="6.1359676">
    <ele>-1</ele>
    <time>2021-10-18T13:16:35Z</time>
   </trkpt>
```

Als je naar de tijden kijk, kun je daar geen duidelijke logica in ontwaren. De les was op 18 oktober 2021 en begon zo rond één uur. Maar de meetmomenten van de Garmin zijn 13:05:50, 13:06:01, 13:06:17, 13:06 33, 13:06, 43 en verder 13:15:41, 13:15:50, 13:15:59, 13:16:09, 13:16:34, ... Ik heb even een uurtje gespendeerd om te bekijken of ik hier een structuur in kon vinden, maar dat is me uiteindelijk niet gelukt.

## Data groeperen

Om de data goed weer te kunnen geven, wilde ik dus de metingen *groeperen*. Het idee was om de metingen van elke minuut bij elkaar op te tellen en de *positie* (`lat` en `lon`) en de *hoogte* (`ele`) in die minuut te middelen. Hiermee kon ik dan een json-bestand maken dat ik eenvoudig aan chartjs zou kunnen geven om te plotten. Om dit voor elkaar te kunnen krijgen, moest ik dus de data per minuut *groeperen*, zodat die rare seconden hierbij weg zouden vallen.

Nu heeft JavaScript geen `groupBy` functie, maar als snel had ik [op StackOverflow](https://stackoverflow.com/a/34890276/1376063) een goede oplossing voor dit gemis gehad (en er bleek ook een [uitgebreid beschreven versie op github te staat](https://gist.github.com/robmathers/1830ce09695f759bf2c4df15c29dd22d)). Deze methode maakt gebruik van de [`reduce`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce) functie in Javascript.

Op zich is een `group by` te vergelijken met een `reduce`-functie. Het gaat er in beide gevallen om een set van waarden te *reduceren* tot een algemene waarde. Normaliter wordt `reduce` evenwel gebruikt om van een set van waarden één waarde over te houden, maar met wat creatieve inzichten kun je natuurlijk ook reduceren naar een *aantal* waarden.

Laten we beginnen met een min of meer standaard gebruik van `reduce`.

```javascript 
> const demo = [
  {"key":"demo1", "val":3},
  {"key":"demo2", "val":5},
  {"key":"demo3", "val":7},
  {"key":"demo1", "val":2},
  {"key":"demo2", "val":4},
  {"key":"demo3", "val":6}
]

> demo.reduce ( (acc, el) => acc+el.val, 0)
27
```

Hier wordt over elk element van de array `demo` heen geïtereerd. Bij elke iteratie wordt het object opgeslagen in de tijdelijke variabele `el`, en wordt de *accumulator* (de variabele `acc`) verhoogd met de waarde van `el.val`. De initiële waarde van `acc` is `0`, dus de uiteindelijke waarde is $3+5+7+2+4+6=27$, precies wat we uiteindelijk ook terugkrijgen.

Maar laten we nu eens kijken of we de waarden van `val` voor dezelfde waarden van `key` kunnen optellen. Dit is op zich precies wat `group by` in standaard sql doet. Op basis van [de genoemde post op StackOverflow](https://stackoverflow.com/a/34890276/1376063) zouden we dan het volgende krijgen:

```javascript
> let data = demo.reduce ( (store, item) => { //1
    let key = item.key                        //2
    store[key] = store[key] || []             //3
    store[key].push(item.val)                 //4
    return store                              //5
}, {})                                        //7
> data
{ demo1: [ 3, 2 ], demo2: [ 5, 4 ], demo3: [ 7, 6 ] }
```

Je ziet wat hier gebeurt: we iteren over de variabele `data` heen en elke iteratie stoppen dat wat we tegenkomen in de variabele `item` (*1*). Vervolgens halen we de `key` op van dat item (*2*) en checken of dat al in onze variabele `store` aanwezig is; als dat niet zo is maken we een lege array aan die hoort bij die key (*3*). Vervolgens voegen we *waarde* van het item toe aan die betreffende `key` in de `store` (*4*) en retourneren we de `store` (*5*). Het resultaat is een object met alle *keys* uit `demo` en daaraan gekoppeld een array met alle *values* die die *key* hebben.

Die `store` krijg initieel de waarde van een leeg object (*7*). Deze variabele heeft exact dezelfde functie als de *accumulator* in de eerdere code-listing: hij houdt de waarden bij elke iteratie bij. Alleen nu is het een object in plaats van een integer.

## Toepassen van de theorie

Nu we hebben bewezen dat deze materie werkt, is het toepassen hiervan op die gpx-data redelijk triviaal. Het enige wat we moeten doen is over die data heen itereren, een *key* maken die gebaseerd is op de uren en de minuten en alle metingen in die minuut daaraan toevoegen. Vervolgens kunnen we al die metingen bij elkaar optellen en delen door het aantal metingen om het gemiddelde van die minuut te krijgen.

Om dit helemaal goed te laten werken, heb ik een hulp-functie `get_decimals` geschreven die de uren en minuten aan de rechterkant voorziet van een 0 als deze lager zijn dan 10.

```javascript

> let data = points.reduce ( (store, item) => {
    // key maken op basis van de uren en de minuten
    let t = new Date(item.time)
    let key = `${get_decimals(t.getHours())}:${get_decimals(t.getMinutes())}`

    // alle metingen aan deze key in de store toevoegen
    store[key] = store[key] || []
    store[key].push(item)
    return store
}, {})
> let end_result = []
> for (item in data) {
     let l = data[item].length

     let lat = (data[item].map ( e => e.lat).reduce( (a,b)=>a+b ) / l).toFixed(7) 
     let lon = (data[item].map ( e => e.lon).reduce( (a,b)=>a+b ) / l).toFixed(7)
     let ele = (data[item].map ( e => e.ele).reduce( (a,b)=>a+b ) / l).toFixed(2)

     end_result.push ( { "time":item, "data":{lat, lon, ele}} )
 }
```

De variabele `end_result` is nu een array met alle metingen (hoogte en positie) per minuut. Precies wat we nodig hebben voor die chartsjs. Het enige punt was dat bleek dat een minuut eigenlijk een te grote tijdseenheid is: ik vlieg zo 140 kilomter per uur, dus in een minuut heb ik al ruim twee kilometer afgelegd. Het resultaat is daardoor wat 'schokkerig'. Dat ga ik ook ooit nog wel eens oplossen...

De hele repo van die vlieglessenblog [is op github te vinden](https://github.com/bart314/vlieglessen).







