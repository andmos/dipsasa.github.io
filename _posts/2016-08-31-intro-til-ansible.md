---
layout: post
title: "Kort intro til Ansible"
author: anm
tags: [CMS, Devops, Windows, Linux, Ansible]
img: ../../../img/anm/ansible.svg 
---

[Ansible](https://docs.ansible.com/ansible/) er et configuration management system (CMS) som brukes for å enkelt automatisere oppsett av servere og / eller klienter for å holde disse like. Ansible er et par abstraksjoner høyere enn f.eks. [Puppet](https://puppet.com/) og [Chef](https://www.chef.io/chef/) som gjør at terskelen for å ta det i bruk er mye lavere en hos konkurrentene. Kort fortalt kan man med Ansible sitte ett sentralt sted og rulle ut programvare til flere servere på en enkel og grei måte.

Ansible bruker SSH på Linux eller WinRM for Windows for å provisjonere servere, som igjen gjør at maskinene vi ønsker å sette opp ikke trenger noen agent installert. For Windows kan Ansible også snakke med Active Directory for autentisering og admin-rettigheter.    

<!--more-->

## Ansible i praksis

Selv om maskinene man ønsker å kontrollere kan kjøre Linux, Windows eller BSD må selve kontroller-maskinen kjøre på Linux eller OS X. For å installere Ansible. [Følgende guide](https://docs.ansible.com/ansible/intro_installation.html) guider gjennom oppsett på ønsket *nix plattform.
For å vise hvordan vi har eksperimentert i bruk Ansible følger her en del eksempler på oppbyggning og filstruktur. Vi benytter mye Windows Server i DIPS, så de fleste eksemplene er for denne platformen.

### Relevante servere

Etter Ansible-sjargong defineres servere etter ansvarsområde. Det kan være webserver, byggeserver, applikasjonsserver osv. Hvilke servere som har hvilke ansvarsområder definieres i en ``inventory`` fil. Denne ser typisk slik ut:

```
[buildservers]
vt-build01.domain.local

[appservers]
vt-build01.domain.local
vt-build01.domain.local
```

`ìnventory`` filen gjør heller ikke forskjell på om serveren kjører Windows eller Linux - det bestemmes av rollene.

### Roller

Et ansvarsområde bygges videre opp av roller. Det kan være applikasjonscontainer (f.eks. IIS, Tomcat) eller database (f.eks MySQL, Postgres, MSSQL).

**Rollene** defineres i en mappestruktur: ``roles/[rollenavn]/tasks/main.yaml``.

Rollene bygges opp av en eller flere tasks, som igjen bruker ferdige [Ansible moduler](http://docs.ansible.com/ansible/list_of_windows_modules.html) for å kunne abstrahere bort så mange installasjonsdetaljer som mulig.

Her er f.eks. Rollen ``DocBuildAgent`` (ligger i ``/roles/DocBuildAgent/tasks/main.yaml``)

```
- name: Install ruby
  win_chocolatey: name=ruby

- name: Install asciidoctor
  raw: gem install asciidoctor

- name: Install asciidoctor-pdf
  raw: gem install --pre asciidoctor-pdf
```

En rolle kan også være mye enklere enn dette, som f.eks. Git:

```
- name: Install Git
  win_chocolatey: name=git
```

Legg merge til at Ansible har egen modul for å bruke Chocolatey (win_chocolatey) direkte. [Denne modulen](http://docs.ansible.com/ansible/win_chocolatey_module.html) gir lett tilgang på funksjonalitet vi kjenner igjen fra Chocolatey. Bruk av modulen vil også installere Chocolatey om det ikke finnes på serveren.

En annen modul som kommer rett ut av esken med Ansible er [win_feature](http://docs.ansible.com/ansible/win_feature_module.html) som lar oss installere Windows Features vi setter opp.

Denne rollen setter opp IIS for oss:

```
- name: Install IIS
    win_feature: name=Web-Server,Web-Common-Http,Web-App-Dev,Web-Net-Ext45,Web-Asp-Net45 include_management_tools=True

  - name: Install .NET 4.5 Feature
    win_feature: name=NET-Framework-45-Features include_sub_features=true
```



En rolle kan også inneholde templates for editering av config-filer. Et eksempel er TeamCity-agent rollen som krever editering av config-filen for at den skal komme opp av seg selv og melde seg inn i clusteret vårt. Mappestrukturen er her utvidet med en ``templates/`` mappe som inneholder filen ``buildAgent.properties``. Her er et utdrag fra denne filen:

```
# {{ ansible_managed }}
## TeamCity build agent configuration file

######################################
#   Required Agent Properties        #
######################################

## The address of the TeamCity server. The same as is used to open TeamCity web interface in the browser.
## The address may contain "http://" or "https://" prefix. If omitted, the http protocol is used.
## Example 1:  serverUrl=buildserver.mydomain.com
## Example 2:  serverUrl=https://buildserver.mydomain.com:8111
serverUrl=http\://teamcity

## The unique name of the agent used to identify this agent on the TeamCity server
## Use blank name to let server generate it.
## By default, this name would be created from the build agent's host name
name={{ ansible_hostname }}

```

Den øverste tag'en forteller oss at Ansible får lov til å håndtere fila. Dette gjør igjen at vi kan bruke variabler i config-fila, som f.eks. ``name={{ ansible_hostname }}`` som gjør at serveren dette blir rullet ut på får navnet sitt satt i config-fila. Ansible tilbyr et [hav av variabler](http://docs.ansible.com/ansible/playbooks_variables.html#) som kan brukes.

Med bruk av templates-fila vil selve tasken se slik ut:

```
- name: Download Team City Build Agent
  win_get_url: url='http://teamcity/update/buildAgent.zip' dest='C:\\tcagent.zip' force=false

- name: Unzip Team City Build Agent  
  win_unzip: src="C:\\tcagent.zip" dest="C:\\tcagent" creates="C:\\tcagent"

- name: Copy buildAgent.properties file to Team City Build Agent
  win_template: src=templates/buildAgent.properties dest="C:\\tcagent\\conf\\buildAgent.properties"

- name: Install Team City Build Agent Service
  raw:   cd C:\tcagent\bin; ./service.install.bat

- name: Start Team City Build Agent Service  
  raw:  cd C:\tcagent\bin; ./service.start.bat
```

Legg merke til modulen ``win_template`` her.

Videre kan en rolle inneholde ``handlers``. En handler er en eller flere opperasjoner som skjer etter at en task er kjørt.
Følgende eksempel viser en task for installasjon av lastbalansereren [HAProxy](http://www.haproxy.org/) (for Linux) og medfølgende handler: 

```
   - name: Install HAProxy
     apt: pkg=haproxy state=installed update_cache=true
      
   - name: Copy Config 
     copy: src=haproxy.cfg dest=/etc/haproxy/haproxy.cfg
     notify: 
       - restart haproxy
```

Legg merke til ``notify`` som kaller på handleren sin for å restarte tjenesten: 

```
- name: restart haproxy
  service: name=haproxy state=restarted
```


### Playbooks


Oppskriften for et ansvarsområde skrives i en [playbook](http://docs.ansible.com/ansible/playbooks.html). En playbook lister opp roller som inngår i ansvarsområdet, samt hvilke servere denne skal installeres på.

Playbooken ``BuildServer.yaml`` inneholder oppsummeringen av hva som inngår for en byggeserver. Denne kan se slik ut:

```
- name: Provisioning for BuildServer
  hosts: buildservers
  roles:
   - Chocolatey
   - Git
   - ScriptCS
   - DocBuildAgent
   - PsTools
   - MicrosoftBuildTools
   - JDK    
   - TeamCityAgent
```

Denne playbooken vil installere og konfigurere en TeamCity-agent og melde denne inn i TeamCity clusteret vårt på alle maskiner som ligger i ``inventory`` under ``[buildservers]``.

Selve utrullingen skjer med en enkel kommando:

```
ansible-playbook BuildServer.yaml -i inventory

PLAY [Provisioning for BuildServer] ********************************************

TASK [setup] *******************************************************************
ok: [vt-build01.domain.local]

TASK [Chocolatey : Install Chocolatey 0.9.9.12] ********************************
ok: [vt-build01.domain.local]

TASK [Git : Install Git] *******************************************************
ok: [vt-build01.domain.local]

TASK [ScriptCS : Install ScriptCS] *********************************************
ok: [vt-build01.domain.local]

TASK [DocBuildAgent : Install ruby] ********************************************
ok: [vt-build01.domain.local]

TASK [DocBuildAgent : Install asciidoctor] *************************************
ok: [vt-build01.domain.local]

TASK [DocBuildAgent : Install asciidoctor-pdf] *********************************
ok: [vt-build01.domain.local]

TASK [PsTools : Install PsTools] ***********************************************
ok: [vt-build01.domain.local]

TASK [MicrosoftBuildTools : Install Microsoft Build Tools 2015 update 3] *******
changed: [vt-build01.domain.local]

TASK [JDK : Install Java JDK 8] ************************************************
ok: [vt-build01.domain.local]

TASK [TeamCityAgent : Download Team City Build Agent] **************************
changed: [vt-build01.domain.local]

TASK [TeamCityAgent : Unzip Team City Build Agent] *****************************
changed: [vt-build01.domain.local]

TASK [TeamCityAgent : Copy buildAgent.properties file to Team City Build Agent]
changed: [vt-build01.domain.local]

TASK [TeamCityAgent : Install Team City Build Agent Service] *******************
ok: [vt-build01.domain.local]

TASK [TeamCityAgent : Start Team City Build Agent Service] *********************
ok: [vt-build01.domain.local]

PLAY RECAP *********************************************************************
vt-build01.domain.local    : ok=15   changed=4    unreachable=0    failed=0
```

Her viser Ansible en av sine styrker. Den har kontroll på hva som allerede er rullet ut på serverne, som gjør at den kan holde miljø i sync uten at man trenger å være redd for å ødelegge noe på systemer som allerede er rullet ut. Roller som er nye eller endret for en server viser ``changed`` taggen når rollen utføres.

## Konklusjon

Ansible er et utrolig enkelt og ikke minst **kraftig** verktøy som gjør oppgaven med å provisjonere servere på alle plattformer til en lek. 
Arbeidsflyten vår nå når vi setter opp nye tjenester er at vi ført finner ut hvordan vi gjør det manuelt, før vi lager en Playbook av det og får det automatisert.
Krav til infrastruktur som må være på plass skal være repeterbart - og en gangs manuell jobb er nok.

For å lese mere om hvordan man enkelt kommer i gang med Ansible for Windows, [sjekk ut ansible windows support](https://docs.ansible.com/ansible/intro_windows.html).
