---
published: true
layout: post
date: 2016-09-18T16:00:00.000Z
comments: true
---
> [Buildroot](https://buildroot.org/) is a tool that simplifies and automates the process of building a complete Linux system for an embedded system, using cross-compilation.

In pratica crea un sistema estremamente leggero contenente tutto quello che serve per far girare la propria applicazione e nulla più. In questo modo si possono spremere a fondo sistemi modesti senza cali prestazionali notevoli.

BuildRoot contiene già molti tool utili per la compilazione e la gestione di un sistema di base.
Per instalare buildroot si può andare nella pagina di [download](http://buildroot.org/downloads/), oppure è possibile scaricare il [vagrantfile](https://buildroot.org/downloads/Vagrantfile) per il previsioning, personalmente ho scelto quest'ultimo metodo.

Quindi per prima cosa scarico il Vagrantfile ed avvio la procedura di provisioning
```bash
curl -O https://buildroot.org/downloads/Vagrantfile;
vagrant up
```
