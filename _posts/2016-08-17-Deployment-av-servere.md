---
layout: post
title: "Pakkebasert installasjon"
author: bpj
tags: [powershell, nuget, chocolatey, installasjon]
---
Vi i DIPS ASA ønsker å legge til rette for enkel deployment av programvaren vår på kundenes maskinvare. La oss ta en titt hva vi gjør for å komme i mål med dette med [PowerShell](https://msdn.microsoft.com/en-us/powershell/mt173057.aspx), [NuGet](https://www.nuget.org/) og [Chocolatey](https://chocolatey.org/).    

Jeg heter Bjørn-Petter Johannessen og er i dag medisinstudent. På grunn av teknisk bakgrunn fra Helse Nord IKT har jeg sett på og forbedret installasjonsprosessen av DIPS sin programvare.

<!--more-->

## Fra MSI til scriptbar installasjon

Tradisjonelt er DIPS Classic en monolitisk applikasjon som blir publisert til kunden vha. MSI. Dette fungerer bra når man har én, eller i alle fall ganske få, MSI-pakker som systemet består av.

DIPS Arena er en plugin-basert applikasjon. Arena kan kjøre med få eller mange moduler installert avhengig av kundens behov. Siden antallet MSI-er da økte betraktelig, trengte vi en bedre måte å publisere programvaren på.

Løsningen ble en scriptbar installasjon basert på [Chocolatey](https://chocolatey.org/). Fordelen med dette er en enklere og plug-in-basert installasjon. Kunden kjører ett PowerShell-script som starter installasjonen av det vi har pakket. Dette scriptet kan modifiseres av kunden dersom det er noen spesielle behov. Vi har lagt til rette for å kunne legge inn kode som kjøres før og etter vårt installasjonsscript. Det er også mulig å skrive om PowerShell-scriptet, eller lage sin egen måte å installere DIPS på. 

## Men, hold an litt.. Hva er PowerShell, NuGet og Chocolatey?

### PowerShell

[PowerShell](https://msdn.microsoft.com/en-us/powershell/mt173057.aspx) ble lansert i 2006 som standard Shell på Windows Server, og er ment for å automatisere administrative oppgaver i Windows. Vi har brukt PowerShell for å kalle `choco install`, parse output fra Chocolatey og til å behandle konfigurasjonen for DIPS-pakkene.

### NuGet
[NuGet](https://www.nuget.org/) er pakkebehandleren Microsoft har laget for .NET, og brukes til å installere kodebiblioteker og -avhengigeter. Ruby har gems, Python har pip, JavaScript har nmp og .NET har NuGet. I oppførsel kan en NuGet pakke sees på som [en ZIP-pakke med litt meta-data](https://docs.nuget.org/create/nuspec-reference), som id, versjonsnummer, tittel, osv. 

Pakkebehandleren (`nuget.exe`), pakkeformatet(`*.nuspec` filer), NuGet pakker(`*.nupkg` filer) og den offisielle nettsiden hvor du kan laste ned pakker(`nuget.org`) heter så å si det samme. Videre vil vi i hovedsak referere til NuGet pakker når vi nevner NuGet. 

![NuGet](../../../img/bpj/nuget.png)

### Chocolatey 
[Chocolatey](https://chocolatey.org/) er en installasjonsmekansime som benytter NuGet-formatet for å distribuere og installere pakker. En Chocolatey-pakke er tilsynelatende lik en vanlig NuGet-pakke, med unntak av en ting: En Chocolatey pakke har en script kalt ``ChocolateyInstall.ps1`` pakket med som beskriver hvordan pakken skal installeres. Chocolatey kan benytte en  `Packages.config` fil som inneholder pakkene man ønsker å installere. En Chocolatey-pakke kan også ta mot parametere, som vi har samlet i en egen `parameters.config` fil. Konfigurasjonsparametrene blir lagt inn i DIPS sin konfigurasjon ved hjelp av en PowerShell-funksjon vi har laget og pakket med pakkene.

## Installasjon

Chocolatey som verktøy benytter seg av en pakkebrønn (eller webserver som tilbyr pakker om man vil) for å hente ned pakker. Siden vi idag ikke har en slik tjener som alle sykehusene når, har vi valgt å lage en "offline" installatør som benytter ferdig nedlastete Chocolatey-pakker og et enkelt wrapperscript som installerer disse. Det hele blir pakket via ZIP og distribuert ut på vår kundeportal. 

I _svært_ korte trekk består installasjonsscriptet vårt av:

{% highlight powershell %}
Function Install-DIPS
{
    Param (
        // Håndtering av parametre
    )
    
    // Håndtering av bl.a. $PackagesConfigFile som inneholder en liste over moduler og versjoner

    // Kode brukeren vil kjøre før installasjonen ($BeforeInstall)

    // Dette er det interessante som skjer:
    $chocoCommand = "choco install $PackagesConfigFile -s $packagesFolder -y -pre --package-parameters $ChocolateyParameters $forceParameter"
    Invoke-Expression $chocoCommand

    // Kode brukeren vil kjøre etter installasjonen ($AfterInstall)
}

Install-DIPS -InstallChocolatey -PackagesConfigFile Packages.config -ParametersFile *-parameters.config -BeforeInstall $beforeInstall -AfterInstall $afterInstall -ForceParameter $forceParameter
{% endhighlight %}

`Packages.config` inneholder en liste over pakkene Chocolatey skal installere. Eksempelvis klienten for DIPS Arena:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<packages>
    <package id="dips-arena-besttrack-client"     version="x.y.z" />
    <package id="dips-arena-desktop-client"       version="x.y.z" />
    <package id="dips-arena-framework-client"     version="x.y.z" />
    ..
{% endhighlight %}

`parameters.config` inneholder konfigurasjonen som kunden må endre, bl.a. installasjonslokasjon for klienten og serveradresse:

{% highlight xml %}

<?xml version="1.0" encoding="utf-8"?>
<parameters>
	<!-- Installation folder -->
    <parameter name="InstallLocation"     value="" /> <!-- The installation location. Example: “C:\DIPS” -->
	<!-- Config parameters -->
    <parameter name="ArenaServerHostName" value="" /> <!-- Hostname of the Arena-Server the client should connect to. Example: “arena-srv-01.domain.no” -->
    ..
{% endhighlight %}

I utgangspunktet vil output fra `choco install` komme uforandret ut. Men siden vi bruker en `Packages.config`-fil som inneholder alle pakkene Chocolatey skal installere, får vi ut av esken bare følgende output:

{% highlight powershell %}
Installing the following packages: Packages.config
{% endhighlight %}

Ikke så greit å vite hva man egentlig installerer. Vi laget først en funksjon som parset innholdet i `Packages.config` og ga bedre output:

{% highlight powershell %}
Packages to be installed:
 dips-arena-besttrack-client - x.y.z
 dips-arena-desktop-client - x.y.z
 dips-arena-framework-client - x.y.z
 ...
{% endhighlight %}

Deretter lagde vi en issue hos Chocolatey på dette. Siden Chocolatey er åpen kildekode, skrev [Andreas Mosti](/authors/anm) C#-koden som løste issuen. Dette er med i [Chocolatey versjon 0.10.0](https://github.com/chocolatey/choco/issues/878). 

## Hvordan kan kundene rulle ut programvare?
Vi har lagt til rette for å bruke vår installasjon på både servere og klienter. Kundene står fritt til å installere vår programvare slik de selv vil. Spesielt på klientsiden finnes det mange alternativer, bl.a. Microsoft System Center Configuration Manager og Altiris. Klienten kan enkelt rulles ut med disse verktøyene uten å bruke vårt installasjonsscript.

Scriptene er laget slik at kundene selv kan modifisere dem etter deres behov. Vi tar også mer enn gjerne imot tilbakemeldinger fra våre kunder. 

## Vårt installasjonsscript i fremtiden
I skrivende stund er det funksjonalitet vi savner i Chocolatey. Allerede installerte pakker kan reinstalleres med å legge på `-force`-parameteret, men man kan ikke velge hvilke pakker man ønsker å reinstallere. I tillegg er all output direkte fra Chocolatey, og vi gjør ingen parsing av denne. Vi ønsker å få plass en bedre håndtering av advarsler og feilmeldinger slik at det blir lettere å feilsøke installasjoner.

## Konklusjon

Med en modularisert arkitektur, må utrullingen av applikasjonen være smidig. Chocolatey gir oss blant annet avhengighetshåndtering mellom pakker, og sørger for at vi alltid får med oss bitene en modul er avhengig av. Tiden med avglemte MSI-pakker er derfor forbi. Så langt har utrulling av DIPS i våre testmiljøer fungert mye mer effektivt, med store muligheter for scriptbarhet og automasjon.