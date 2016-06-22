---
layout: post
title:  Forbedring av ytelsen i XAML-baserte brukergrensesnitt
author: roh
tags: [CSharp, WPF, XAML, Ytelse]
---

Neste generasjon av DIPS, DIPS Arena, er en Windows Desktop-applikasjon skrevet i WPF. 

![Skjermbilde av DIPS Arena](../../../img/arena.jpg)

WPF er et meget kraftig UI-rammeverk, men nesten alle tilfeller og situasjoner kan løses på flere måter. Dermed finnes det både gode og dårlige løsninger på de fleste utfordringer. Her er noe av det vi har lært om ytelse i WPF under en profileringsrunde vi har kjørt i vår.

<!--more-->

## Generelle punkter

På [Build 2015]( https://channel9.msdn.com/Events/Build/2015/3-698) gav Microsoft oss denne smørbrødlisten for å forbedre ytelsen i XAML-baserte brukergrensesnitt. Listen er god og flere av punktene var relevante for oss:

1. *Embrace the Zen of performance.* Ikke gjør mer enn du må, og utsett det du kan.
2. *Define the problem and set a target.* Ha et mål i tankene. Hvor raskt er fort nok?
3. *Convert to a Universal Windows app (UWP).* Krever Windows 10 og ikke relevant for oss som ennå har brukere på Windows 7.
4. *Profile your application.* Alltid profiler før du går i gang med å forbedre ytelse. Profileringen forteller deg hvor problemet ligger, og gjør at du kan legge inn innsatsen der det monner mest.
5. *Defer work on startup.* Ha et forhold til hva brukeren må ha for å kunne begynne arbeidet. I store skjermbilder er kanskje ikke alt nødvendig å laste med en gang.
6. *Reduce number of elements.* XAML-baserte skjermbilder er meget følsomme for dype visuelle trær og antallet elementer det skal kjøres layout på.
7. *Use virtualization.* Ikke tegn opp 100 elementer når brukeren bare ser 20 av dem.
8. *Optimize databinding.* [Fjern binding-feil](http://pelebyte.net/blog/2011/07/11/twelve-ways-to-improve-wpf-performance/#FixBindingErrors) og husk at ikke alle bindinger trenger å gå to veier.
9. *Optimize your images.* Bruk rett bildeformat basert på typen bilde og bruk rett størrelse med tanke på brukeropplevelsen. [PNG for de fleste skjermbilder, JPEG for resten.](http://www.hanselman.com/blog/BloggersKnowWhenToUseAJPGAndWhenToUseAPNGAndAlwaysSquishThemBoth.aspx)
10. *Optimize text rendering* Relevant for UWP-applikasjoner.

## Spesielle punkter

Siden DIPS Arena er en modulær applikasjon, har vi funnet fram til noen egne punkter som har gitt god effekt hos oss. 

### Benytt Timeline tool i Visual Studio 2015

[Visual Studio 2015s Timeline profileringsverktøy]( https://blogs.msdn.microsoft.com/wpf/2015/01/16/new-ui-performance-analysis-tool-for-wpf-applications/) er fantastisk bra! Verktøyet viser detaljert hva som årsaken til et ytelsesproblem i klienten. 

Her er et worst case-eksempel på hvordan det kan se ut:

![Timeline-visning fra en testapplikasjon](../../../img/roh/timeline.png)

Se først på grafen `Visual throughput (FPS)`, lilla linje. Grafen går i bunn og brukeren opplever 0 FPS flere ganger.

Når vi har verifisert problemet, kan vi se på årsaken. `UI thread utilization (%)` forteller oss hvor tiden gikk tapt:

- **Grønn:** Kode som har kjørt på GUI-tråden. Dette bør holdes til et minimum, uansett hvor man er i applikasjonens levetid. I eksemplet har vi flere bolker som burde vært kjørt på bakgrunnstråd. Tommelfingerregelen er at raske operasjoner på 30ms eller mindre kan kjøre på UI-tråden, noe mer og vi må gå over til et av de enkle async-verktøyene.
- **Blå:** Parsing av XAML. Enklere XAML, statiske ressurser osv. vil hjelpe på denne tiden.
- **Orange:** Layout gjør livet krevende når vi bruker for mange visuelle elementer eller designer skjermbildene for komplekst. 
- **Lilla, lyseblå og grå:** Rendring, I/O og Xaml Other er svært sjeldent et problem i vår applikasjon.

Her et eksempel fra en modul DIPS Arena:

{: .center}
![Eksempelprofilering av DIPS Arena modul](../../../img/timeline-arena.png)

[Dette er en Arena-modul som bruker for lang tid på å lastes pga. kompleks layout.](http://pelebyte.net/blog/2011/07/11/twelve-ways-to-improve-wpf-performance/#VisualTree) Kompleks i denne sammenhengen er å f.eks. bruke flere Grids og dype trær for å oppnå noe som egentlig krever kun et Grid. Tommelfingerregelen er at 1 000 visuelle elementer som skal legges ut, kan gi opptil 1 sekunds venting. Microsoft har en grense på 1 500 – 2 000 elementer i Visual Studio. 

Tiden layout tar er meget case-avhengig, men det er her vi måtte gjøre den største jobben. Heldigvis kan skjermbilder ofte forenkles uten at det går ut over verken kompleksitet eller brukeropplevelse.

### Timeline med vanlige C# Class Libraries

Ut av esken kan vi kun velge Timeline som profileringsverktøy dersom et WPF-prosjekt (eller et annet XAML-basert prosjekt) er valgt som *StartUp project*.  Hos oss er modulene vanlige C# Class Libraries og Visual Studio ble litt forvirret. 

Dette fungerer på vanlig Library-prosjekt ved å legge inn følgende i prosjektfilen i samme PropertyGroup som ProjectGuid:

{% highlight xml %}

<ProjectTypeGuids>{60dc8134-eba5-43b8-bcc9-bb4bc16c2548};{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}</ProjectTypeGuids> 

{% endhighlight %}

[60dc8134-eba5-43b8-bcc9-bb4bc16c2548](http://www.codeproject.com/Reference/720512/List-of-Visual-Studio-Project-Type-GUIDs) forteller Visual Studio at dette egentlig er et WPF-prosjekt. [FAE04EC0-301F-11D3-BF4B-00C04F79EFBC](http://www.codeproject.com/Reference/720512/List-of-Visual-Studio-Project-Type-GUIDs) er GUIDen for et C#-prosjekt.

### Vær obs på bruk av ResourceDictionaries

DIPS Arena bruker [Managed Extensibility Framework (MEF)]( https://msdn.microsoft.com/en-us/library/dd460648(v=vs.110).aspx) for å sette sammen applikasjonen. De forskjellige skjermbildene kommer fra forskjellige moduler, og alt blir satt sammen når applikasjonen starter.

{: .center}
![Modulær oppbygging](../../../img/prism.png)

Basert på funn fra profilering fant vi flere forbedringer rundt ressurshåndteringen i de forskjellige modulene:

-	Ikke merge alt inn i en stor `ResourceDictionary`. Arena er bygget på *ViewModel first*-prinsippet. Hver modul har eksportert sine ressurser og disse er blitt merget sammen under oppstart. Fordelen har vært at alle moduler kan dele ressurser, ulempen er at dette fører til treghet jo flere moduler som er med i systemet. Denne mergingen har vi nå gått bort fra og ressurser kan hentes inn vha. en ComponentResourceKey. [Hold ressursene nært viewene som skal bruke dem.](http://pelebyte.net/blog/2011/07/11/twelve-ways-to-improve-wpf-performance/#ResourceDictionary)
-	[Fortrekk StaticResource over DynamicResource.](http://pelebyte.net/blog/2011/07/11/twelve-ways-to-improve-wpf-performance/#DynamicResource) DynamicResource er tyngre og kun nødvendig dersom verdiene endrer seg under kjøring. Vi hadde fått for mange DynamicResource, [særlig fra Blend]( https://blogs.msdn.microsoft.com/unnir/2009/03/31/blend-wpf-and-resource-references/).
-	[Frys Geometry og Brush og andre ressurser.]( https://msdn.microsoft.com/en-us/library/ms750509(v=vs.100).aspx) Når vi fryser objekter som arver fra `Freezable`, kan verdiene ikke lenger endres. Dette har flere fordeler. Objekter som ikke er frosne, blir kopiert av WPF, fryst og deretter vist på skjermen. Når vi eksplisitt fryser, slipper vi å kopiere hvert objekt som skal rendres. I tillegg kan vi lage objektene på en bakgrunnstråd for senere å vise dem vha. UI-tråden.

## Konklusjon

Sett ytelsesmål selv for brukergrensesnittet. Dersom din applikasjon er skrevet vha. XAML kommer du langt med å følge rådene i denne posten, men det aller viktigste er å profilere før du begynner med ytelsesforbedringer. 

Små endringer på riktig plass kan ha stor effekt.

