---
layout: post
title:  "Docker-machine and Swarm"
date:   2016-09-18 16:00:00
comments: true
---
#DOCUMENTAZIONE
  
  In pratica con le ultime modifiche che hanno apportato a docker sono state introdotte parecchie chicche.
  Prima di tutto swarm, simile a fleet utilizzato invece in coreos, permette di associare più host all'interno di un unico ambiente comune gestibile in maniera abbastanza semplice.
  
##Docker-machine
Una delle migliori cose cha abbiano potuto mettere in docker, permette una gestione centralizzata di tutto le macchine connesse o no ad uno swarm.

```bash
docker-machine
Usage: docker-machine [OPTIONS] COMMAND [arg...]

Create and manage machines running Docker.

Version: 0.8.0, build b85aac1

Author:
  Docker Machine Contributors - <https://github.com/docker/machine>

Options:
  --debug, -D						Enable debug mode
  --storage-path, -s "/Users/bubodlack/.docker/machine"	Configures storage path [$MACHINE_STORAGE_PATH]
  --tls-ca-cert 					CA to verify remotes against [$MACHINE_TLS_CA_CERT]
  --tls-ca-key 						Private key to generate certificates [$MACHINE_TLS_CA_KEY]
  --tls-client-cert 					Client cert to use for TLS [$MACHINE_TLS_CLIENT_CERT]
  --tls-client-key 					Private key used in client TLS auth [$MACHINE_TLS_CLIENT_KEY]
  --github-api-token 					Token to use for requests to the Github API [$MACHINE_GITHUB_API_TOKEN]
  --native-ssh						Use the native (Go-based) SSH implementation. [$MACHINE_NATIVE_SSH]
  --bugsnag-api-token 					BugSnag API token for crash reporting [$MACHINE_BUGSNAG_API_TOKEN]
  --help, -h						show help
  --version, -v						print the version
  
Commands:
  active		Print which machine is active
  config		Print the connection config for machine
  create		Create a machine
  env			Display the commands to set up the environment for the Docker client
  inspect		Inspect information about a machine
  ip			Get the IP address of a machine
  kill			Kill a machine
  ls			List machines
  provision		Re-provision existing machines
  regenerate-certs	Regenerate TLS Certificates for a machine
  restart		Restart a machine
  rm			Remove a machine
  ssh			Log into or run a command on a machine with SSH.
  scp			Copy files between machines
  start			Start a machine
  status		Get the status of a machine
  stop			Stop a machine
  upgrade		Upgrade a machine to the latest version of Docker
  url			Get the URL of a machine
  version		Show the Docker Machine version or a machine docker version
  help			Shows a list of commands or help for one command
  
Run 'docker-machine COMMAND --help' for more information on a command.

```
Tramite questa maciata di comandi possiamo sapere vita morte e miracoli su tutte le macchine interconnesse al sistema 
    
##Struttura di swarm
Swarm per funzionare ha bisogno di alcuni elementi essenziali:

1. Un sistema distribuito di chiavi/valori per la registrazione di ogni macchina presente all'interno del cloud
2. Una macchina detta master alla quale ci si connetterà per orchestrare tutte le altre 
3. Una o più macchine slave che si suddivideranno il lavoro 

###Consul
Crea una macchina per l'installazione di consul, come detto nel punto uno servirà per registrare tutte le macchine ed i servizi attive in esse.
####Creazione macchina
```bash
docker-machine create \
  --driver generic \
  --generic-ip-address=163.172.180.186 \
  --generic-ssh-key ~/.ssh/id_rsa \
  consul;
```
- `--driver generic` indica che si sta utilizzando una macchina virtuale preesistente generica mediante l'ausilio di ssh.
- `--generic-ip-address=163.172.180.186` indica l'indirizzo ip sulla quale verrà eseguita la connessione ssh.
- `--generic-ssh-key ~/.ssh/id_rsa` indica la chiave ssh che verrà utilizzata per l'accesso alla macchina.
- `consul` il nome da dare per la macchina virtuale.
####Variabile di ambiente
Ora creo una variabile di ambiente con all'interno l'indirizzo ip della macchina consul:
`export KV_IP=$(docker-machine ssh consul 'ifconfig eth0 | grep "inet addr:" | cut -d: -f2 | cut -d" " -f1');`

####Avvio consul

```bash
docker run -d \
  -p ${KV_IP}:8500:8500 \
  -h consul \
  --restart always \
  gliderlabs/consul-server -bootstrap;
```

Bisogna anche considerare che esiste il comando `docker-machine ip consul` torna l'ip pubblico mentre invece il comando precedentemente utilizzato per inizializzare la variabile di ambiente di bash ritorna l'indirizzo ip di eth0 che non sempre equivale all'ip pubblico.

###Master
La macchina master si occupa della gestione dello swarm mediante il container swarm manager ed inoltre rappresenta uno dei tanti nodi che eseguiranno materialmente i lavori.
####Creazione macchina
```bash
docker-machine create \
  --driver generic \
  --generic-ip-address=163.172.169.209 \
  --generic-ssh-key ~/.ssh/id_rsa \
  --swarm \
  --swarm-master \
  --swarm-discovery="consul://${KV_IP}:8500" \
  --engine-storage-driver=overlay \
  --engine-opt="cluster-store=consul://${KV_IP}:8500" \
  --engine-opt="cluster-advertise=eth0:2376" \
  master;
```
Similarmente a quanto avvenuto per la macchina di consul utilizzo le optioni `--driver generic`, `--generic-ip-address`, `--generic-ssh-key` per definire le credenziali di accesso ssh.
Ma in questo caso ve ne sono anche altre che vanno attenzionate:
- `--swarm` indica che bisogna inizializzare lo swarm all'interno di questa macchina.
- `--swarm-master` definisce questa macchina come master dello swarm o manager.
- `--swarm-discovery` definisce la macchina che viene utilizzata come consul e per far questo utilizza la variabile d'ambiente inizializzata in precedenza.
- `--engine-storage-driver` definisce lo sotrage driver che va impostato su `overlay` per evitare problemi con lo swarm.
- `--engine-opt="cluster-store=consul://${KV_IP}:8500"`
- `--engine-opt="cluster-advertise=eth0:2376"`

####Variabile d'ambiente
Ora creo una variabile di ambiente con all'interno l'indirizzo ip della macchina master:
`export MASTER_IP=$(docker-machine ssh master 'ifconfig eth0 | grep "inet addr:" | cut -d: -f2 | cut -d" " -f1');`

###Slave
Le macchine slave possono essere un numero indefinito e servono per suddividere il carico di lavoro.

####Creazione macchina
```bash
docker-machine create \
  --driver generic \
  --generic-ip-address=163.172.165.225 \
  --generic-ssh-key ~/.ssh/id_rsa \
  --swarm \
  --swarm-discovery="consul://${KV_IP}:8500" \
  --engine-storage-driver=overlay \
  --engine-opt="cluster-store=consul://${KV_IP}:8500" \
  --engine-opt="cluster-advertise=eth0:2376" \
  slave;
```
Tutto come nella macchina master con la sola assenza dell'opzione `--swarm-master`

####Variabile di ambiente
Ora creo una variabile di ambiente con all'interno l'indirizzo ip della macchina slave:
`export SLAVE_IP=$(docker-machine ssh slave 'ifconfig eth0 | grep "inet addr:" | cut -d: -f2 | cut -d" " -f1');`

La creazione di altre macchine slave è uguale a questa ma bisogna variare tutti i nomi e gli indirizzi.

##Connettersi ai nodi
Una bella cosa che ho imparato scrivendo questa guida è che non è necessario loggarsi all'interno di una macchina per poter impartire comandi con docker. I vari comandi `docker`, `docker-compose` come pure `docker-machine` possono lavorare anche da remoto se vengono inizializzate le opportune variabili nell'ambiente nella quale si opera.
Tali variabili vengono fornite da `docker-machine` con la seguente sintassi:

```bash
docker-machine env master
  
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://163.172.169.209:2376"
export DOCKER_CERT_PATH="/Users/bubodlack/.docker/machine/machines/master"
export DOCKER_MACHINE_NAME="master"
# Run this command to configure your shell: 
# eval $(docker-machine env master)
```
Come suggerito dall'output di questo comando ci basterà dare il comando `eval $(docker-machine env master)` per settare queste variabili nel nostro ambiente attuale facendo si che i comandi sopra descritti vengano eseguiti sulla relativa macchina e non su quella locale dalla quale vengono inviati.

##Registrazione dei nodi nello swarm
Una volta create le macchine, appositamente configurate ed una volta capito come mandargli comandi direttamente dalla propria postazione è il momento di innestargli un `registrator`.
Il `registrator` non è altro che un container il cui scopo è quello di notificare al `consul` tutto quello che avviene all'interno della macchina ed il suo status, come pure i servizi.

###Registrazione master
Prima di tutto bisogna registrare il nodo master affinché venga eletto a gestore di tutti gli altri nodi.
####Connessione a master
Per connettermi al nodo master utilizzerò la tecnica utilizzata nel capitolo "Connettersi ai nodi".
```bash
eval $(docker-machine env master)
```
####Avvio del registrator
A questo punto posso procedere ad innestare il registrator.
```bash
docker run -d \
  --name=registrator \
  -h ${MASTER_IP} \
  --volume=/var/run/docker.sock:/tmp/docker.sock \
  gliderlabs/registrator:v6 \
  consul://${KV_IP}:8500
```
Qui vengono espressi diversi parametri, bisogna vederli nel dettaglio:
- `--name=registrator` semplicemente il nome che viene dato al container all'interno del nodo.
- `-h` la varibile di ambiente che contiene l'ip relativo all'interfaccia eth0 del nodo.
- `--volume` qui viene passato il socket di docker per tracciare gli eventi e comunicarli a consul.
- `consul://${KV_IP}:8500` in ultimo viene definito l'indirizzo di consul sempre servendomi della variabile di ambiente inpostata in precedenza.

###Registrazione slave
Ora è il momento di registrare i nodi
####Connessione a slave
Utilizzo un'altra volta la tecnica utilizzata in precedenza
`eval $(docker-machine env slave)`
####Avvio del registrator
A questo punto, similarmente a come fatto in precedenza utilizzo il codice per l'avvio del registrator all'inderno del nodo slave.
Questa procedura va attuata per tutti i nodi slave.
```bash
docker run -d \
  --name=registrator \
  -h ${SLAVE_IP} \
  --volume=/var/run/docker.sock:/tmp/docker.sock \
  gliderlabs/registrator:v6 \
  consul://${KV_IP}:8500
```
Non vi è bisogno di spiegazioni inquanto il procedimento è lo stesso utilizzato con master.

##Swarm network
Grazie al comando -docker network- è possibile creare e disfare dei network virtuali che possono o meno avere accesso alla rete globale.
```bash
docker network ls
NETWORK ID          NAME                     DRIVER              SCOPE
1a85f210874b        master/bridge            bridge              local               
ff6476573cc3        master/docker_gwbridge   bridge              local               
3d4a350b8e2d        master/host              host                local               
15f02bafcef5        master/none              null                local               
075e681f2ad2        slave/bridge             bridge              local               
7a3e313a6576        slave/docker_gwbridge    bridge              local               
62c2db4039f6        slave/host               host                local               
306eccc35767        slave/none               null                local               
37d66ac046c3        slave2/bridge            bridge              local               
5bb167a9eb01        slave2/docker_gwbridge   bridge              local               
5de885728dfb        slave2/host              host                local               
6a107ddfb02b        slave2/none              null                local               
62b8b63cbed4        swarmtest_back-tier      overlay             global          
```

##Nginx-proxy per bilanciare i nodi
Allo stato attuale nginx-proxy gestisce swarm pertanto è possibile utilizzarlo, bisogna però adeguarlo a quelle che sono le richieste di rete.
Per esempio bisogna dargli l'opzione --publish 80:80 per istruire swarm che qualsiasi richiesta venga fatta verso la porta 80 deve essere inviata al container di nginx-proxy a prescindere dalla sua posizione all'interno dello swarm.
Inoltre bisogna fare in modo che nginx-proxy sia presente nei network in cui risiedono i container che deve bilanciare.

Quando viene avviato il container è possibile stabilire solo una rete:
```bash
docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro \
    --name my-nginx-proxy --net my-network jwilder/nginx-proxy
```

    
Tuttavia a posteriori è possibile aggiungere altre reti:
```bash
docker network connect my-other-network my-nginx-proxy
```


Ho avuto qualche problema relativo al fatto che la porta 80 sembra utilizzata da qualche altro programma 
Considerando che sto utilizzando delle immagini con solo docker installato di default e non ho installato altri programmi dovrebbe essere docker stesso che sta occupando la porta.
A questo punto ho dovuto installare il pacchetto dsniff per poter utilizzare il comando 
```bash
tcpkill -9 port 80
```


##Scalare un servizio 
Utilizzando il file docker-compose.yml do il comando:
```bash
docker-compose up -d

docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                             NAMES
676635ac9b00        hanzel/tutum-nodejs-redis   "npm start"              45 seconds ago      Up 31 seconds       163.172.168.247:32768->4000/tcp   slave/swarmtest_web_1
33f7520c7881        redis                       "docker-entrypoint.sh"   48 seconds ago      Up 34 seconds       6379/tcp                          slave2/redis

```
In questo modo tutti i container specificati al suo interno vengono fati partire all'interno dei vari nodi.
A questo punto se si vuole scalare l'applicazione web per esempio basta utilizzare il seguente comando:
```bash
docker-compose scale web=3
Creating and starting swarmtest_web_2 ... done
Creating and starting swarmtest_web_3 ... done

docker-compose ps
     Name                    Command               State                Ports              
------------------------------------------------------------------------------------------
redis             docker-entrypoint.sh redis ...   Up      6379/tcp                        
swarmtest_web_1   npm start                        Up      163.172.168.247:32768->4000/tcp 
swarmtest_web_2   npm start                        Up      163.172.169.209:32768->4000/tcp 
swarmtest_web_3   npm start                        Up      163.172.168.247:32769->4000/tcp 
```
Volendo scalare a 5 il numero delle istanze:
```bash
docker-compose scale web=5
Creating and starting swarmtest_web_4 ... done
Creating and starting swarmtest_web_5 ... done

docker-compose ps
     Name                    Command               State                Ports              
------------------------------------------------------------------------------------------
redis             docker-entrypoint.sh redis ...   Up      6379/tcp                        
swarmtest_web_1   npm start                        Up      163.172.168.247:32768->4000/tcp 
swarmtest_web_2   npm start                        Up      163.172.169.209:32768->4000/tcp 
swarmtest_web_3   npm start                        Up      163.172.168.247:32769->4000/tcp 
swarmtest_web_4   npm start                        Up      163.172.169.209:32769->4000/tcp 
swarmtest_web_5   npm start                        Up      163.172.165.225:32768->4000/tcp 
```
Da notare che vengono utilizzati tutti e tre gli indirizzi 163.172.168.247, 163.172.169.209, 163.172.165.225.

Volendo ridimensionare le istanze:
```bash
docker-compose scale web=1
Stopping and removing swarmtest_web_2 ... done
Stopping and removing swarmtest_web_3 ... done
Stopping and removing swarmtest_web_4 ... done
Stopping and removing swarmtest_web_5 ... done

docker-compose ps
     Name                    Command               State                Ports              
------------------------------------------------------------------------------------------
redis             docker-entrypoint.sh redis ...   Up      6379/tcp                        
swarmtest_web_1   npm start                        Up      163.172.168.247:32768->4000/tcp 
```

