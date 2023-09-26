# Docker med Spring boot, Docker hub & AWS ECR 

## Beskrivelse

* Dette repoet inneholder en veldig enkel Spring Boot applikasjon som sier "hello" når en request kommer til context root (/)
* I denne øvingen skal dere bli bedre kjent med Docker og hvordan vi lager et Docker container Image av en Spring boot applikasjon.
* Vi skal bli kjent med både Docker hub og AWS ECR
* Vi skal også sette opp en CI pipeline for å automatisk bygge et nytt container image på hver push til main branch.

## Lag en fork

Du må start med å lage en fork av dette repoet til din egen GitHub konto. 

## Lag en klone av din Fork (av dette repoet) inn i ditt Cloud 9 miljø

Fra Terminal i Cloud 9. Klone repository med HTTPS URL. Eksempel ;
```
git clone https://github.com/≤github bruker>/spring-docker-dockerhub.git
```

## Logg i Cloud 9 miljøet ditt

* URL for innlogging er https://244530008913.signin.aws.amazon.com/console
* Logg på med brukernavn og passord gitt i klassrommet
* Gå til tjenesten Cloud9 (Du nå søke på Cloud9 uten mellomrom i søket)
* Velg "My environments" - pass på at du er i Ireland region.
* Velg "Open IDE"

### Lag et Access Token for GitHub

Du kan hoppe over dette steget hvis du allerede har laget et Token

* Når du skal autentisere deg mot din GitHub konto fra Cloud 9 trenger du et access token.  Gå til  https://github.com/settings/tokens og lag et nytt.
* Velg "Classic"
* Gi token et navn, det skal også ha "repository", og "workflow" rettigheter
* NB. Ta vare på tokenet et sted, du trenger dette senere når du skal gjøre ```git push```


Får du denne feilmeldingen ```bash: /spring-docker-dockerhub: Permission denied``` - så glemte du å bytte ut <github bruker> med
ditt eget Github brukernavn :-)

## Konfigurer Git i Cloud9

(NB! Det kan hende du har gjort dette før, du kan da hoppe over stegene)

Følgende steg trenger du bare gjøre en gang i Cloud9 miljøet ditt. Du kan hoppe over hele steget hvis du har gjort det tidligere.
For å slippe å autentisere seg hele tiden kan man få git til å cache nøkler i et valgfritt antall sekunder på denne måten.

```shell
git config --global credential.helper "cache --timeout=86400"
```

Konfigurer også brukernavnet og eposten din for GitHub CLI. Da slipepr du advarsler i terminalen
når du gjør commit senere.

````shell
git config --global user.name <github brukernavn>
git config --global user.email <email for github bruker>

````

#  "Dockerize"  en Spring Boot applikasjon og push til Docker hub

Verifiser at Docker er installert i Cloud 9

```docker run hello-world``` 

Forventet resultat  

```Unable to find image hello-world:latest locally
 Pulling repository hello-world
 91c95931e552: Download complete
 a8219747be10: Download complete
 Status: 
 Downloaded newer image for hello-world:latest
 Hello from Docker.
 This message shows that your installation appears to be working correctly.

 To generate this message, Docker took the following steps:
  1. The Docker Engine CLI client contacted the Docker Engine daemon.
  2. The Docker Engine daemon pulled the "hello-world" image from the Docker Hub.
     (Assuming it was not already locally available.)
  3. The Docker Engine daemon created a new container from that image which runs the
     executable that produces the output you are currently reading.
  4. The Docker Engine daemon streamed that output to the Docker Engine CLI client, which sent it
     to your terminal.

 To try something more ambitious, you can run an Ubuntu container with:
  $ docker run -it ubuntu bash

 For more examples and ideas, visit:
  https://docs.docker.com/userguide/

```

Kjør kommandoen 

```aidl
docker images
```
Du vil se at Docker har lastet ned et *hello-world* container image til Cloud 9 miljøet ditt. 
Vi skal nå slette dette, men vi må først fjerne en stoppet container som er basert på dette imaget

Kjør først kommandoen ```docker ps``` for å se hvilke containere som kjører. Du vil få en tom liste

```aidl
docker ps
```

Legger du på -a argumentet, vil du også se toppede containere  

```aidl
docker ps -a 
```

Du kan få output som for eksempel 

```aidl
CONTAINER ID   IMAGE         COMMAND    CREATED         STATUS                     PORTS     NAMES
5a89931c5af6   hello-world   "/hello"   2 minutes ago   Exited (0) 2 minutes ago             fervent_bell
```

Avslutt den stoppede containeren med 

```aidl
docker rm <container id> - i eksemplet over 5a89931c5af6
```

Kjør ```docker images``` igjen, og kjør kommandoen 

```aidl
docker image rm <REPOSITORY>
```


Installer maven i Cloud 9. Vi skal forsøke å kjøre Spring Boot applikasjonen fra Maven i terminalen

```
sudo wget http://repos.fedorapeople.org/repos/dchen/apache-maven/epel-apache-maven.repo -O /etc/yum.repos.d/epel-apache-maven.repo
sudo sed -i s/\$releasever/6/g /etc/yum.repos.d/epel-apache-maven.repo
sudo yum install -y apache-maven
```

Sjekk at du kan kjøre Spring Boot applikasjonen med Maven 
```
cd spring-docker-dockerhub
mvn spring-boot:run
```

Sjekk at applikasjonen kjører. Åpne en ny terminal i Cloud 9 og kjør  
```
curl localhost:8080                                                                                                            
```

Hvis du vil kan du også velge Tools/Preview running application fra menyen i Cloud 9 istedet.

Vi skal nå lage en Dockerfile for Spring boot applikasjonen. Vi skal bruke en "multi stage" Docker fil, som 
først lager en container som har alle verktøy til å bygge applikasjonen, maven osv. Spring boot applikasjonen blir kompilert og bygget i denne containeren. 
Deretter bruker den resultatet fra byggeprosessen, JAR filen til å lage en runtime container for applikasjonen. 

Ta gjerne en pause og kes gjerne mer om multi stage builds her; https://docs.docker.com/develop/develop-images/multistage-build/

Kopier dette innholder inn i en  ```Dockerfile``` i rotkatalogen

```dockerfile
FROM maven:3.6-jdk-11 as builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package

FROM adoptopenjdk/openjdk11:alpine-slim
COPY --from=builder /app/target/*.jar /app/application.jar
ENTRYPOINT ["java","-jar","/app/application.jar"]

```

Prøv å byggee en Docker container

```sh
    docker build . --tag <give the image a name>
```

Du må først huske å avslutte (Ctrl+c) applikasjonen du started med maven.
Prøv å starte en container basert dette container image.  
```sh
docker run <image tag used above>
```

Når du starter en container, så lytter ikke applikasjonen i Cloud 9 på port  8080. Hvorfor ikke ? Hint; port mapping 
Kan du start to versjoner av samme container, hvor en lytter på port 8080 og den andre på 8081?

## Registrer deg på Docker hub

!Viktig! Passordet ditt til Dockerhub blir liggende på EC2 Instansen (Serveren) i AWS, og i utgangspunktet kan alle dine medstudenter 
logge seg inn i ditt Cloud9 miljø, siden vi deler passord. Jeg vil derfor anbefale å lage en helt ny DockerHub bruker for PGR301, og 
ikke bruke et passord du vanligvis bruker. 

https://hub.docker.com/signup

## Bygg en container og push til Docker hub 

```
docker login
docker tag <tag> <dockerhub_username>/<tag_remote>
docker push <username>/<tag_remote>
```

Example:
```
docker login
docker tag fantasticapp glennbech/fantasticapp
docker push glennbech/fantasticapp
```

## Share the joy! 

Del gjerne Docker hub container image navnet med andre, så de kan forsøke å kjre det med ```docker run``` 

## Lag et AWS  ECR repository for din container

Finn ut av det selv :-) Kan du gjøre det fra CLI istedet for UI?  Velg et navn med dine initialer, så
vi ikke får navnekonflikter. 

# Autentiser Docker i ditt Cloud 9 miljø mot AWS ECR

Fra dit Cloud9 miljøe, autentiser docker mot AWS ECR med kommandoen
```
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 244530008913.dkr.ecr.eu-west-1.amazonaws.com
````

##  Push et container image til dit ECR repository

Eksempel:
```sh

docker build -t <ditt tagnavn> .
docker tag <ditt tagnavn> 244530008913.dkr.ecr.eu-west-1.amazonaws.com/<ditt ECR repo navn>
docker push 244530008913.dkr.ecr.eu-west-1.amazonaws.com/<ditt ECR repo navn>
```

## Få GitHub Actions til å bygge & pushe et nytt Image hver gang noen lager en ny commit på main branch 

Her er et eksempel på en workflow tatt fra foreleser sitt miljø, du må gjøre endringer for å tilpasse den ditt eget? 
Lykke til!

* Du må også leggee til Repository secrets, og gi GitHub actions nøkler. Husker du hvordan vi gjorde det ?

```yaml
name: Publish Docker image

on:
  push:
    branches:
      - main

jobs:
  push_to_registry:
    name: Push Docker image to ECR
    runs-on: ubuntu-latest
    steps:
      - name: Check out the repo
        uses: actions/checkout@v2

      - name: Build and push Docker image
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        run: |
          aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 244530008913.dkr.ecr.eu-west-1.amazonaws.com
          rev=$(git rev-parse --short HEAD)
          docker build . -t hello
          docker tag hello 244530008913.dkr.ecr.eu-west-1.amazonaws.com/glenn:$rev
          docker push 244530008913.dkr.ecr.eu-west-1.amazonaws.com/glenn:$rev
```
Gjør endringer på koden i main branch - se at GitHub actions lager et nytt container image og laster opp til ECR. 
