---
creation_date: 23 maart 2020
date: 29 januari 2022
---

# Abstracte Klassen en Interfaces in Java 8

Sinds in Java 8 interfaces daadwerkelijke (`default`) implementaties van methoden kunnen hebben, is het onderscheid tussen interfaces en abstracte klassen behoorlijk verwaterd, of in ieder geval genuanceerd geworden.

In theorie is het grote onderscheid nog steeds valide: als je een *objecttype* wilt hebben die voor specifieke methoden een bepaalde standaard-implementatie heeft en de implementatie van andere methoden aan subklassen delegeert, dan gebruik je een abstracte klasse. Hiervan kun je niet direct instanties aanmaken: objecten van dit type worden geïnstantieerd wanneer een object van het sub-type wordt geïnstantieerd.

Omgekeerd, wanneer je alleen maar een *beschrijving* wilt hebben van een bepaald type – een beschrijving die alleen maar aangeeft wat een object van dat type kan doen, zonder uitspraken te doen over hoe die dat doet – dan maak je gebruik van een interface. Een interface wordt nooit geïnstantieerd en vertelt alleen maar aan de compiler wat objecten van dat type gegarandeerd kunnen doen.

Een andere manier om naar dit onderscheid te kijken is door te stellen dat abstracte klassen te maken hebben met een *status* – uiteindelijk worden hier objecten van geïnstantieerd met velden en dus een status – terwijl interfaces te maken hebben met *methoden* – er wordt alleen maar een uitspraak gedaan over wat een bepaald datatype kan doen.

Je kunt één en ander samenvatten met de onderstaande tabel (gemaakt op basis van [deze site](https://www.javatpoint.com/difference-between-abstract-class-and-interface)).

Abstracte klasse | Interface
----|----
heeft abstracte en niet-abstracte methoden | heeft alleen maar abstracte methoden, maar kan sinds J8 ook default en static methoden hebben
kun je niet gebruiken voor multiple inheritance | kun je wel gebruiken voor multiple inheritance
heeft final, non-final, static en non-static velden | heeft alleen final en static velden (variabelen, eigenlijk)
kent verschillende visibility modifiers | alle methoden zijn public

## Een nieuw sleutelwoord

Door gebruik te maken van het sleutelwoord `default` kun je sinds java 8 een methode-implementatie in je interface definiëren. Je kunt nog steeds geen instantie maken van een interface-type, maar klassen die deze interface implementeren hoeven dan niet allemaal die methode individueel te bevatten. Dat scheelt een heleboel boilerplate code. Zie onderstaand voorbeeld:

```Java
interface DemoInterface {
    // een interface kan ook velden bevatten
    int demo = 7;
    String omschrijving = "DemoInterface";
 
    // methode-definities zoals we gewend zijn
    String demofunction();
    void dingendoen(int a, int b);
 
    // maar je kunt ook standaard-implementatie hebben in je interface
    // zowel op object-niveau
    default String getLongDescription() {
        return String.format("Call vanuit de interface... %s", toString());
    }
 
    // als op het klasse-niveau
    static String getInterfaceDescription() {
        return "Een methode vanuit de interface die een string teruggeeft...";
    }
}
```

## Multiple inheritance

Eén van de grote verschillen tussen overerving van (abstracte) klassen en implementatie van een interface is dat een klasse maar van één andere klasse kan overerven, terwijl een klasse meerdere interfaces kan implementeren.

Wat betekent dat, wanneer meerdere interfaces een default methode hebben met dezelfde signature? Hoe weet de runtime-engine dan welke implementatie -ie moet hebben? Antwoord: dat weet -ie niet, dus als je een klasse hebt die meerdere interfaces implementeert die een methode met dezelfde signature hebben, krijg je een compile-time error (iets wat bekend staat onder het [diamant-problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem)).

Om dit probleem op te lossen, moet je expliciet aangeven welke van de verschillende default-implementaties je in de klasse wilt gebruiken, zoals in het voorbeeld hieronder (zie ook [de code op github](https://github.com/bart314/OOP3/tree/master/week2/interfaces)).

```Java
interface Foo {
    default String getLongDescription() {
        return "Lange beschrijving gekregen door de interface Foo.";
    }
}
 
interface Bar {
    default String getLongDescription() {
        return "Deze string komt bij Bar vandaan...";
    }
}
 
 
class DiamondProblem implements Foo, Bar {
    /*
     de methode getLongDescription is zowel gedefinieerd in de
     interfaces Foo en Bar, dus we moeten in deze klasse aangeven
     welke van beide we willen gebruiken. Als we dat niet doen,
     krijgen we een compile error.
 
     Let ook op de interessante syntax: Foo.super.method()
    */
 
    @Override
    public String getLongDescription() {
        return Foo.super.getLongDescription();
    }
```

## Toepassing binnen (abstract) factory pattern

Het [factory pattern](https://en.wikipedia.org/wiki/Factory_method_pattern) wordt gebruikt om de het aanmaken van concrete objecten niet direct door de gebruiker van die objecten (de client) te laten doen, maar om dit te delegeren aan een specifieke klasse, de factory. Op die manier wordt het gebruik van een object losgekoppeld van de creatie van dat object, met alle ([SOLID](https://en.wikipedia.org/wiki/SOLID)) voordelen van dien.

Een veralgemenisering van dit principe is het [Abstract Factory Pattern](https://en.wikipedia.org/wiki/Abstract_factory_pattern#Java_example). In dat geval wordt ook de concrete factory zelf van de client weg-geabstraheerd, die vervolgens alleen maar te maken heeft met het abstracte type. Wanneer de client dan een concreet object nodig heeft, wordt op dat moment de juiste factory aangeroepen om een bepaald object terug te geven – dit kan bijvoorbeeld op basis van een configuratie-bestand in runtime bepaald en aangepast worden.

Normaliter gebruik je bij een abstract factory een abstracte factory-klasse die verantwoordelijk is voor het delegeren van de creatie van objecten naar de juiste concrete factory. Zo’n abstract klasse zou er als volgt uit kunnen zien (zie ook [de hele code-base op github](https://github.com/bart314/OOP3/tree/master/week2/carfactory)):

```java
public abstract class AbstractCarFactory {
    public static Car buildCar(int t, int loc) {
        Car car = null;
 
        CarType type = CarType.values()[t];
        Location location = Location.values()[loc];
 
        switch(location){
            case EU:
                car = EUCarFactory.buildCar(type);
                break;
            case ASIA:
                car = AsiaCarFactory.buildCar(type);
                break;
            default:
                car = DefaultCarFactory.buildCar(type);
        }
 
        return car;
    }
}
```

Hetzelfde kun je nu bereiken door de implementatie van deze methode in een interface te zetten:

```Java
public interface InterfaceCarFactory {
    @SuppressWarnings("Duplicates")
    static Car buildCar(int t, int loc) {
        Car car = null;
 
        CarType type = CarType.values()[t];
        Location location = Location.values()[loc];
 
        switch(location){
            case EU:
                car = EUCarFactory.buildCar(type);
                break;
            case ASIA:
                car = AsiaCarFactory.buildCar(type);
                break;
            default:
                car = DefaultCarFactory.buildCar(type);
        }
 
        return car;
    }
}
```

Zoals je kunt zien, is er eigenlijk geen verschil in implementatie. Maar technisch en conceptueel is er wel degelijk een verschil. Wanneer we een methode in een klasse-definitie zetten (zelfs al is deze static), dan suggereren we dat het een methode is die bij een bepaald object behoort – compleet met constructors, een status en gedrag. Door net naar een interface te verplaatsen, geven we aan dat dit een soort methode is die niet gekoppeld is aan een bepaald (type) object.

Tevens kunnen we de interface-implementatie gebruiken om utility-methoden en applicatie-brede variabelen op te slaan – iets dat we voorheen bijvoorbeeld deden in een Settings-klasse, die alleen maar bestond voor dergelijke doeleinden.