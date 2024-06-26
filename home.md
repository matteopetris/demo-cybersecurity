# Report demo Cybersecurity: DNS spoofing e possibile risvolti
# Introduzione

Il mio progetto consiste nell’intercettare e modificare le risposte DNS per un particolare sito web: [http://www.comunitamontanacarnia.it](http://www.comunitamontanacarnia.it). L'obiettivo è far collegare il client a una copia del sito gestita da un web server controllato dall'attaccante. Nel mio caso, il web server è allocato nella macchina dell’attaccante. L’esecuzione di questo attacco ha come presupposto che l’attaccante sia nella stessa rete della vittima, utilizzando tecniche di ARP cache poisoning per diventare man-in-the-middle (MITM) e quindi poter modificare le risposte DNS.

# Threat Model 

Il threat model preso in considerazione necessita che l’avversario sia sulla stessa rete della vittima e che possa utilizzare ARP cache poisoning per diventare MITM. Una volta ottenuto questo controllo, l'attaccante modifica le iptables in modo da reindirizzare il traffico DNS in uscita dalla sua macchina, verso uno script Python in grado di manipolare i pacchetti. Inoltre, l’attaccante deve essere in possesso di un web server in cui caricare la pagina falsa.

# Ambiente

Per eseguire questo attacco ho usato due macchine virtuali Kali Linux connesse alla stessa rete. Per farlo basta andare sulle impostazioni delle due macchine su rete, selezionare “connesso a: Rete con Nat” e anche “nome: NatNetwork”.

La configurazione nel mio caso specifico è:
- **Macchina dell’attaccante:** IP 10.0.2.15 MAC 08:00:27:9f:20:92
- **Macchina della vittima:** IP 10.0.2.6 MAC 08:00:27:65:4d:1c
- **Gateway:** IP 10.0.2.1 MAC 52:54:00:12:35:00

Per trovare l’indirizzo MAC dell’attaccante basta digitare da terminale `ip addr show eth0` mentre per trovare l’indirizzo IP si può usare `ifconfig`. Si può trovare l’indirizzo IP e MAC del gateway con il comando `ip neigh`. Per trovare l'idirizzo ip della vittima si può usare il comando 'nmap', in questo particolare caso `nmap 10.0.2.0/24`. 

# Svolgimento

## Indice
1. [Diventare MITM](#diventare-mitm)
2. [Deviare i pacchetti in uscita](#deviare-i-pacchetti-in-uscita)
3. [Modificare i pacchetti in uscita](#modificare-i-pacchetti-in-uscita)
4. [Prova della modifica dei pacchetti](#prova-della-modifica-dei-pacchetti)
5. [Preparazione web server](#preparazione-web-server)

## Diventare MITM

Dalla macchina dell’attaccante ho aperto ettercap (inserendo la password della macchina in quanto deve essere lanciato con sudo). Ho accettato la configurazione con la spunta in alto a destra. Ho avviato una scansione della rete per trovare gli host (lente di ingrandimento in alto a sinistra). Una volta finita la scansione posso visualizzare la lista degli host (il tasto a destra della lente di ingrandimento). Seleziono la macchina da attaccare e la aggiungo alla lista Target (schiaccio Add to Target 1). Ora avvio l’attacco ARP poisoning (vado sul pulsante in alto a destra simile a un planisfero e poi seleziono ARP poisoning quindi schiaccio ok).

ARP Poisoning sfrutta la natura non autenticata delle risposte ARP. L'attaccante invia pacchetti ARP falsificati sulla rete locale. In particolare viene inviato al gateway un pacchetto ARP che dice che l’indirizzo IP della vittima corrisponde all’indirizzo MAC dell’attaccante. Mentre alla vittima viene inviato un pacchetto ARP affermante che l’indirizzo del gateway corrisponde all’indirizzo MAC dell’attaccante. Il gateway e la vittima aggiornano la loro cache ARP. Ora, quando la vittima vuole inviare un pacchetto al gateway, in verità lo invierà all’attaccante. Quando il gateway vorrà inviare un pacchetto alla vittima, lo invierà all’attaccante. Se l’attaccante non modifica nulla, i pacchetti poi proseguiranno verso l’IP di destinazione corretto in quanto lui conosce il vero indirizzo MAC sia del gateway che della vittima. I pacchetti ARP per deviare la connessione attraverso l’attaccante devono essere poi reinviati periodicamente al fine di mantenere la connessione. L’immagine sottostante riporta, al seguito di una richiesta e risposta (corretta) ARP vengono poi inviati i due pacche falsi, il primo al gateway mentre il secondo alla vittima.

![ARP Poisoning](images/image1.png)

## Deviare i pacchetti in uscita

Adesso che sono diventato MITM, voglio modificare la risposta DNS del sito quindi modificare i pacchetti in uscita dalla macchina dell’attaccante che abbiano come source port 53 usata nel protocollo UDP per le comunicazioni DNS. Per farlo aggiungo una regola a iptables: `sudo iptables -I OUTPUT -p udp --sport 53 -j NFQUEUE --queue-num 1` questa regola permette di inviare i pacchetti in questione a una coda netfilter dove possono essere poi elaborati.



## Modificare i pacchetti in uscita

Con l’uso di un codice Python modifico i pacchetti che hanno come `rrname='www.comunitamontanacarnia.it.'` mettendo al posto dell’indirizzo IP del web server di www.comunitamontanacarnia.it l’indirizzo IP del web server in cui è allocata la copia della pagina web (in questo caso IP macchina attaccante). A causa di alcuni problemi nell’effettuare questa modifica, ho preferito usare il formato esadecimale per eseguirla (riga 24 dell’immagine sottostante) andando a sostituire l’indirizzo in questione (da 'c12bb1eb' a '0a00020f'= 10.0.2.15). Infine, nelle ultime righe dello script vengono inviati regolarmente tutti i pacchetti DNS non modificati mentre se vengono modificati viene inviato il pacchetto modificato. Per avviare il codice si usa il comando `sudo python “nome del file”` (nel mio caso `sudo python print_packet2.py`).

![Deviare i pacchetti](images/image2.png)


## Prova della modifica dei pacchetti

Ora, quando la vittima prova a connettersi a [http://www.comunitamontanacarnia.it](http://www.comunitamontanacarnia.it), invierà una richiesta DNS verso il gateway. Questa passerà attraverso l’attaccante. Poi il gateway invierà la risposta alla vittima ma questa passerà attraverso l’attaccante che la modificherà mettendo l’indirizzo IP del suo web server. Quindi la vittima vedrà come risposta 'www.comunitamontanacarnia.it A 10.0.2.15'. Di seguito sono riportate delle scansioni eseguite con Wireshark, conseguenti all'invio di richieste DNS da parte della vittima per www.comunitamontanacarnia.it. Nella prima immagine sottostante vediamo ciò che passa attraverso l’attaccante, mentre nella seconda immagine vediamo ciò che vede la vittima.

![Modificare i pacchetti](images/image3.png)
![Prova della modifica](images/image4.png)

## Preparazione web server

Ho scaricato una copia della pagina web `www.comunitamontanacarnia.it` usando il software HTTrack. Sono stati necessari pochi passaggi tra cui incollare il link della pagina da scaricare. Dopo aver eliminato l’index già presente nella cartella `/var/www/html` della macchina attaccante, ho quindi inserito la pagina scaricata. Per renderla distinguibile dall’originale ho aggiunto un’immagine esplicativa. Per avviare il web server dalla macchina dell’attaccante, usando il terminale, ho digitato `/var/www/html` e poi `sudo service apache2 start` questo avvia un web server. Digitando l’URL del web server (nel mio caso 10.0.2.15) apparirà la pagina web inserita nella cartella `/var/www/html`.Ora, se la vittima cercherà di collegarsi a 'www.comunitamontanacarnia.it' invierà una richiesta DNS per questa pagina. La risposta che le arriverà conterrà l’indirizzo IP del web server dell’attaccante. Quindi lei si collegherà e le comparirà la pagina controllata dall’attaccante. L’immagine sottostante riporta come si presenta alla vittima la pagina.
![Preparazione web server](images/image5.png)

## Un possibile risvolto

Poiché la pagina è sotto il controllo dell’attaccante, sfruttando la tecnica "Template Injection", ho modificato il link associato alla scritta “Comunità di montagna della Carnia” in modo che si colleghi a una copia della pagina [https://www.carnia.comunitafvg.it/](https://www.carnia.comunitafvg.it/). Anche questa è stata scaricata e caricata nel web server dell’attaccante. Inoltre, ho modificato anche il link alla pagina “accedi all’area personale” in alto a destra. Il sito web iniziale prevedeva vari metodi di autenticazione ma per semplicità ho selezionato solo quello con posteid. Quindi, quando la vittima schiaccerà su “accedi all’area personale” comparirà una copia della pagina di accesso con posteid, anche questa caricata nel web server dell'attaccante. Affinché l’accesso vada a buon fine, è necessario che l’attaccante riesca ad aprire una pagina di accesso con posteid autentica, quindi copiare velocemente il QRcode della pagina di accesso posteid, e inserirlo nella cartella `/var/www/html`. Nell'immagine sottostante è riportata la schermata di come si presenta la pagina al client, si può notare dal l'url che non è l'originale, però potrebbe facilmente trarre in inganno la vittima. Infine, poichè ho impostato che la pagina web visualizzata dalla vittima carichi lo screenshot contenuto nella cartella `/var/www/html`, se la vittima scannerizzerà il QRcode mentre è ancora valido, permetterà all'attaccante di autenticazione alla pagina a cui voleva autenticarsi la vittima. 



![Pagina vittima](images/image6.png)



# Conclusioni

Questo attacco evidenzia le vulnerabilità già note di un sito HTTP, dimostrando quanto sia facile manipolare le risposte DNS per reindirizzare la vittima verso un altro sito HTTP, in particolare uno controllato da un attaccante. È stato anche dimostrato quanto sia semplice ricreare un sito web molto simile, se non identico, all'originale per ingannare la vittima. Un punto cruciale da sottolineare è quanto sia facile, partendo dalla pagina originale, modificare i link ipertestuali originariamente in HTTPS per reindirizzare la vittima a pagine controllate dall'attaccante. In questo caso, l'attaccante ha sfruttato il QR code per ingannare la vittima e autenticarsi al suo posto. Questo metodo, nello specifico, potrebbe essere difficile da attuare nella pratica a causa del breve tempo di validità del QR code. Tuttavia, è possibile replicare la stessa tipologia di attacco inserendo un link di accesso a un altro sito che utilizza solo l'autenticazione con username e password. Una volta che la vittima ha inserito le credenziali, potrebbe essere reindirizzata alla pagina originale fingendo un problema di connessione. In questo modo, la vittima potrebbe non accorgersi dell'inganno, permettendo all'attaccante di ottenere le sue credenziali.

# Bibliografia
## MITRE ATT&CK
[MITRE ATT&CK](https://attack.mitre.org/)
- [Adversary-in-the-Middle](https://attack.mitre.org/techniques/T1557/)
- [ARP Cache Poisoning](https://attack.mitre.org/techniques/T1557/002/)
- [Template Injection](https://attack.mitre.org/techniques/T1221/)

## ChatGPT
[ChatGPT](https://chatgpt.com)

## Stack Overflow
[Stack Overflow](https://stackoverflow.com)

## YouTube
[YouTube](https://www.youtube.com)
- [ettercap](https://www.youtube.com/watch?v=cVTUeEoJgEg&t=309s)

