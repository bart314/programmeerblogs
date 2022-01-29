---
date: 27 januari 2022
---

# Python string-representaties

## Inleiding

Vaak vragen studenten wat het verschil is tussen de dunders `__str__` en `__repr__` in een Python-klasse. Een begrijpelijke vraag, want intuïtief doen beide methoden hetzelfde: ze geven een string-representatie van het object in kwestie. Dus wat zijn inderdaad de overeenkomsten en verschillen?

Er zijn natuurlijk al diverse blog-posts die dit uitleggen. Zo is er een uitgebreide beschrijving op [journaldev.com](https://www.journaldev.com/22460/python-str-repr-functions), of [deze uitleg op StackOverflow](https://stackoverflow.com/questions/1436703/what-is-the-difference-between-str-and-repr/1436756#1436756). Maar het leek me handig om een Nederlandstalige beschrijving te maken, waarbij ook wordt ingegaan op de momenten waarop beide methoden worden aangeroepen.

## Wat zegt de documentatie?

De documentatie op python.org lijkt in eerste instantie duidelijk. De `__repr__`-methode geeft [de ‘officiële’ string-represenatie van het object terug](https://docs.python.org/3/reference/datamodel.html#object.__repr__), terwijl de `__str__`-methode [de ‘informele’ of netjes afdrukbare representatie van het object terug moet geven](https://docs.python.org/3/reference/datamodel.html#object.__str__). Let op dat deze documentatie gebruik maakt van apostrofen om aan te geven dat zij ook niet precies definiëren wat een ‘officiële’ of ‘informele’ is.

Hoewel, dat is niet helemaal waar. De documentatie bij `__repr__` vervolgt met de opmerking dat de string die deze methode teruggeeft een *valide Python-expressie* is die in het ideale geval gebruikt kan worden om het object opnieuw te genereren. In die betekenis is `__repr__` dus een soort serialisatie methode.

## Een verschil in aanroepen

Laten we eens kijken wanneer de verschillende methoden worden aangeroepen.

```ipython
In [1]: class Demo:
   ...:     pass
   ...: 
   ...: d = Demo()
   ...: print (d)
   ...: 
   ...: print (d.__repr__())
   ...: print (d.__str__())

<__main__.Demo object at 0x7f8a1441bf70>
<__main__.Demo object at 0x7f8a1441bf70>
<__main__.Demo object at 0x7f8a1441bf70>

In [2]:
```

De output van al deze print-statements is in alle gevallen hetzelfde: `'<__main__.Demo object at 0x7fe0adc4d910>'`. Dat is niet heel zinvolle informatie, en het is ook logisch dat bede methoden dezelfde output geven: de implementatie van `__str__` in Object roept gewoon `__repr__` aan.

Wat dan, wanneer we één van beide methoden in onze klasse implementeren?

```ipython
In [1]: class Demo:
   ...:     def __init__(self):
   ...:         self.demostr = 'hallo allemaal'
   ...: 
   ...:     def __repr__(self):
   ...:         return f'Demo object met demostr: {self.demostr}'
   ...: 
   ...: d = Demo()
   ...: print (d)

Demo object met demostr: hallo allemaal

In [2]:
```

Het resultaat van deze test is de string die door `__repr__` wordt teruggegeven: "Demo object met demostr: hallo allemaal". Blijkbaar wordt door print een call gedaan naar `__repr__`. Maar wat nu als we beide methoden implementen?

```ipython
In [1]: class Demo:
   ...:     def __init__(self):
   ...:         self.demostr = 'hallo allemaal'
   ...: 
   ...:     def __repr__(self):
   ...:         return f'Demo object met demostr: {self.demostr}'
   ...: 
   ...:     def __str__(self):
   ...:         return f'De str-methode aangeroepen.'
   ...: 
   ...: 
   ...: d = Demo()
   ...: print (d)

De str-methode aangeroepen.

In [2]:
```

Dit maakt duidelijk dat er een call naar `__str__` gedaan wordt. Blijkbaar kijkt Python bij een print eerst of er een implementatie van `__str__` bestaat; als dat niet het geval is, wordt een poging gedaan `__repr__` aan te roepen, en als die er ook niet is, wordt hetzelfde protocol bij de superklasse toegepast. En zo verder totdat we bij `Object` aankomen.

## Het automatisch aanroepen van `__repr__`

Dus print roept automatisch `__str__` aan. Maar is er ook een situatie denkbaar waarin automatisch `__repr__` wordt aangeroepen, zelfs wanneer beide methoden bestaan? Dit blijkt het geval te zijn wanneer we het object uitprinten zonder gebruik te maken van de methode `print`, zoals we vaak doen in de interactieve shell:

```ipython
In [1]: class Demo:
   ...:     def __init__(self):
   ...:         self.demostr = 'hallo allemaal'
   ...: 
   ...:     def __repr__(self):
   ...:         return f'Demo object met demostr: {self.demostr}'
   ...: 
   ...:     def __str__(self):
   ...:         return f'De str-methode aangeroepen.'
   ...: 
   ...: 
   ...: d = Demo()
   ...: print (d)
De str-methode aangeroepen.

In [2]: d
Out[2]: Demo object met demostr: hallo allemaal
```

Een andere situatie waarin `__repr__` wordt gebruikt is wanneer het object een onderdeel is van een tupel of een dictionary die wordt uitgeprint:

```ipython
In [4]: print ([d])
[Demo object met demostr: hallo allemaal]

In [5]: print ({'key':d})
{'key': Demo object met demostr: hallo allemaal}

In [7]:
```

## Serialisatie door middel van `__repr__`

Zoals aangegeven, is het de bedoeling van `__repr__` dat deze methode een representatie teruggeeft die in principe gebruikt kan worden om het object te regenereren. Dat betekent dat `obj == eval(repr(obj))` in de regel een waarde zou moeten hebben van `True` (dat kan niet altijd natuurlijk, maar dit is een ideaal). Hoe zou je zoiets kunnen realiseren?

In principe is een object niks meer dan een zooi methoden met een interne status. Dus om een op basis van een bestaan object een nieuw object te maken, volstaat het om deze interne status aan het nieuwe object door te geven. Zolang dat nieuwe object van hetzelfde type is, heeft dat object ook de gewenste methoden.

```python
class Demo:
    def __init__(self, **kwargs):
        self.__dict__.update(kwargs) 

    def update(self, key, value):
        self.__dict__[key] = value

    def __str__(self):
        return f'Demo object, verder niks aan de hand'

    def __repr__(self):
        return self.__dict__

# maak een object aan met wat data
object1 = Demo(naam='Bart', woonplaats='Groningen', huisnummer=116)
# verander de interne status van het object
object1.update('huisnummer', 200) 
# en printen de interne status
print (object1.__repr__())

# hier maken we een kopie aan van het eerste object
object2 = Demo(**object1.__repr__())
# en printen de interne status; die is hetzelfde als object1
print (object2.__repr__())

# nu passen we de interne status van het eerste object aan 
object2.update('huisnummer', 60)

# en printen van beide objecten de interne status af
print (object1.__repr__())
print (object2.__repr__())
```

De code hierboven levert het volgende op:

```python
{'naam': 'Bart', 'woonplaats': 'Groningen', 'huisnummer': 116}
{'naam': 'Bart', 'woonplaats': 'Groningen', 'huisnummer': 200}
{'naam': 'Bart', 'woonplaats': 'Groningen', 'huisnummer': 200}
{'naam': 'Bart', 'woonplaats': 'Groningen', 'huisnummer': 60}
```

## Conclusie

Dus `__str__` en `__repr__` lijken er op elkaar; ze verschillen op nuance maar daardoor wel radicaal. Het beste advies voor de alledaagse gang van zaken is gewoon in de eigen klassen `__str__` te implementeren, en `__repr__` te gebruiken voor klassen die geserialiseerd moeten worden.
