Descrizione di come fare il deploy di un'applicazione rails con tutti i cazzo di crismi.
==========

Sto cercando di creare una configurazione che mi permetta di fare il deploy di una qualsiasi applicazione rails senza dovermi preoccupare troppo di metterci mano dopo.

L'esito di questa guida dovrebbe essere il seguente:

 - Adattamento del sistema di deploy al sistema traefik  
 - Adattamento allo swarm  
 - Adattamento a docker-compose versione 2
 - Utilizzo della gemma passenger per la gestione del server
 

##Adattamento del sistema di deploy al sistema traefik


Traefik è forse il miglior sistema per gestire automaticamente la configurazione del webserver in un ambiente docker. Le caratteristiche che lo rendono preferibile rispetto alla seppur valida alternativa nginx-proxy sono:

 - Il supporto nativo a **letsencypt** 
 - La leggerezza, essendo completamente scritto in **golang**
 - La presenza di un piccolo ma estremamente utile pannello di monitoring web
 - Il supporto allo swarm, che ormai diventa sempre di più un fattore fondamentale di docker
 - La maggior flessibilità in quanto a configurazione
 - Il pieno supporto alla sintassi recente di docker-compose version: '2'


Tenendo conto di queste caratteristiche è del tutto normale volerlo preferire alle alternative.
Andando quindi ad utilizzare tutte le nuove caratteristiche sopra elencate vado a spiegare nel dettaglio cosa serve per fare il deploy di un progetto rails.

Parto con una configurazione di base atta allo sviluppo ed il testing delle applicazioni in locale, cosa molto utile quando non si vuole intasare il proprio client di svariate versioni di ruby dovendole per altro gestire con altri sistemi quali rvm o rbenv entrambi ottimi per carità ma non privi di problemi e di sicuro non stabilissimi per un utilizzo in produzione. Utilizzando docker in produzione si ha infatti un ambiente pulito con tutto quello che ci serve per far girare la nostra applicazione e nulla di più, niente ambienti promiscui quindi. Facendo girare anche l'applicazione in fase di sviluppo all'interno di un ambiente docker si ha di fatto una sorta di provisioning senza doverci sbattere con macchine virtuali e vagrant, anch'essi estremamente utili e validissimi strumenti in ogni caso.

L'unica noia di questa guida è che in fase di sviluppo non posso avvalermi delle connessioni sicure https in quanto letsencrypt richiede un accesso diretto alla rete da parte del sito per poter confermare il certificato ma non essendo un fattore essenziale in questo momento rimando questa fase al deploy sul server finale e quindi alla produzione.

Parto quindi dalla base del progetto ovvero il Dockerfile che ci permette di creare l'immagine che conterrà tutto quel che mi serve per far girare l'applicazione rails.
Fortunatamente ce già qualcosa che fa al caso mio, ovvero l'immagine [ruby:2.3](https://hub.docker.com/_/ruby/) che ci mette sin da subito a disposizione un ambiente di base con ruby installato. Di certo questa non è l'immagine più piccola che si possa trovare ma di sicuro è una di quelle che ha il maggior supporto.
L'immagine in questione pesa all'incirca 700 MB e come è possibile vedere utilizzando l'utilissimo tool online [imagelayers.io](https://imagelayers.io/?images=ruby:2.3.0) la maggior parte dello spazio (circa 300 MB) è occupato dai pacchetti scaricati con apt-get per soddisfare le dipendenze.
Le dipendenze in rails sono un tasto dolente, sopratutto quando si lavora con docker, questo perché volendo sarebbe anche possibile utilizzare come base immagini molto più piccole in quanto a dimensioni, basti pensare alle immagini basate su alpine linux, il problema si pone quando nel progredire dello sviluppo del progetto ci si trova davanti ad una o più dipendenze che alpine non è in grado di soddisfare ed a quel punto ci si ritrova a dover riadattare il tutto prendendo un'altra immagine come base tornando alla fine alla medesima pesantezza dell'immagine che ho preso in considerazione per questo test.
Ciò non di meno l'utilizzo di una immagine molto più piccola ha i suoi vantaggi pertanto è sempre meglio andare in questa direzione, in fin dei conti riadattare il progetto ad un'altra immagine consiste semplicemente nella creazione di un nuovo Dockerfile, il problema è che quando si ha poco tempo non sempre ci si riesce in leggerezza e si vorrebbe un po' di tempo in più per testare accuratamente gli esiti.
Nello stesso [repo](https://hub.docker.com/_/ruby/) che ho mostrato vi sono anche delle immagini create con alpine linux.

##Il Dockerfile

    FROM ruby:2.3
    MAINTAINER bubodlack@gmail.com
    
    # Install apt based dependencies required to run Rails as
    # well as RubyGems. As the Ruby image itself is based on a
    # Debian image, we use apt-get to install those.
    RUN apt-get update && apt-get install -y \
      build-essential \
      nodejs
    
    # Configure the main working directory. This is the base
    # directory used in any further RUN, COPY, and ENTRYPOINT
    # commands.
    RUN mkdir -p /app
    WORKDIR /app
    
    # Copy the Gemfile as well as the Gemfile.lock and install
    # the RubyGems. This is a separate step so the dependencies
    # will be cached unless changes to one of those two files
    # are made.
    COPY Gemfile Gemfile.lock ./
    RUN gem install bundler && bundle install --jobs 20 --retry 5 
    
    # Copy the main application.
    COPY . ./
    
    # Expose port 3000 to the Docker host, so we can access it
    # from the outside.
    EXPOSE 3000
    # Configure an entry point, so we don't need to specify
    # "bundle exec" for each of our commands.
    ENTRYPOINT ["bundle", "exec"]
    
    # The main command to run when the container starts. Also
    # tell the Rails dev server to bind to all interfaces by
    # default.
    CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]

 Il file è commentato e non ha bisogno di spiegazioni tuttavia mi soffermo su due punti:
 

 1. Sto esponendo la porta 3000, questa operazione mi permette di non doverla specificare all'interno del docker-compose.yml, a meno che certo non ne abbia esposta più di una in quel caso devo specificare quale è destinata al web
 2. L'entrypoint richiama automaticamente bundle exec, ciò forza rails ad utilizzare le gemme definite all'interno del progetto preferendole a quelle di sistema evitando troppi messaggi in console di eventuali avvertimenti per contrasti tra le gemme installate nel sistema e quelle del progetto e ci assicura che tutti gli eseguibili che andiamo richiamando, come rails s o rails c o anche rake puntino a ciò che abbiamo importato assieme al progetto. Pertanto quando vorrò dare un comando al container mi basterà scrivere rake db:migrate e l'entrypoint gli verrà semplicemente anteposto.

##Il docker-compose.yml


Altro pezzo estremamente importante, qui bisogna specificare tutti comportamenti del container o dei conteiner che andranno ad utilizzare l'immagine che ho definito in precedenza nel Dockerfile.

    version: '2'
    services:
      sito:
        restart: always
        build: .
        networks: 
          - web
        environment:
          SECRET_KEY_BASE: "9yr93724ry924ry9nc20cfh29b4hcf20cnrq9prq24hcfqhefh42bhfcbuo42y42bòflqg83qyo4ò8vyoq3òv8yalh438byvqò4nq3hv4tyoq24òvhq4802bv2à"
          RAILS_ENV: "development"
        labels:
          - "traefik.backend=clan"
          - "traefik.frontend.rule=Host:sito.docker.localhost"
    
    networks:
      web:
        external:
          name: traefik_webgateway

Gli elementi essenziali per la gestione di un sito mediante traefik sono due:

 1. La definizione della rete all'interno della quale traefik ha il suo
    "dominio", in questo caso la rete web che a sua volta fa riferimento
    alla rete traefik_webgateway
 2. L'immissione di alcune label che ne gestiscono il comportamento per
    il container

L'impostazione del network va definita fuori dal contesto del servizio:

    networks:
      web:
        external:
          name: traefik_webgateway
Per poi richiamarla nello stesso mediante la sintassi:

    ...
    
    networks: 
          - web
    

Bisogna ora definire a quale indirizzo dovrà essere collegato il sito, per far ciò devo dare a traefik due informazioni essenziali, a che indirizzo rispondere ed a che container reindirizzarlo, tutto questo lo faccio tramite delle apposite label reperibili presso [questa](http://docs.traefik.io/toml/#docker-backend) pagina del progetto traefik.
Le label in questione sono le seguenti:

    labels:
          - "traefik.backend=sito"
          - "traefik.frontend.rule=Host:sito.docker.localhost"

Tutto sommato il funzionamento per roba così semplice non si discosta molto da quello di nginx-proxy solo che quello fa riferimento solo alle variabili di ambiente.

A questo punto non resta altro che avviare traefik!

Svolgo questya operazione con un altro docker-compose.yml:

    version: '2'
    
    services:
      proxy:
        restart: always
        image: traefik
        command: --web --docker --docker.domain=docker.localhost --logLevel=DEBUG
        networks:
          - webgateway
        ports:
          - "80:80"
          - "8080:8080"
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - /dev/null:/traefik.toml
    
    networks:
      webgateway:
        driver: bridge
A questo punto tutto è configurato e pronto per essere utilizzato nello sviluppo del software.