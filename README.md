Guida operativa WebSphere + Linux — Troubleshooting

1. Il modello mentale

Quando un'applicazione su WebSphere non funziona, conviene evitare di partire subito da "SystemOut.log".

Una richiesta normalmente attraversa diversi livelli:

Client → DNS → rete → Load Balancer/F5 → Web Server (IHS/Apache/Nginx) → WebSphere → applicazione → DB/MQ/servizi esterni

La prima domanda deve quindi essere:

«Dove si interrompe la chiamata?»

---

2. Primo controllo: il server è raggiungibile?

DNS

nslookup hostname

oppure:

dig hostname

Controllare i resolver configurati:

cat /etc/resolv.conf

Verificare la risoluzione Linux:

getent hosts hostname

Cosa cerco?

Se:

nslookup applicazione.bnl.it

restituisce un IP, il DNS sta risolvendo.

Se non restituisce nulla o va in timeout, bisogna investigare DNS/resolver prima di WebSphere.

---

3. Controllare la connettività

Ping

ping hostname

Attenzione: un ping fallito non significa necessariamente server irraggiungibile, perché ICMP può essere bloccato.

È più interessante verificare direttamente la porta.

TCP

nc -vz hostname 443

oppure:

telnet hostname 443

Possibili risultati:

Connection refused

Il server risponde, ma probabilmente nessun processo ascolta su quella porta.

Connection timed out

Possibili cause:

- firewall
- routing
- security policy
- server non raggiungibile
- LB
- problema di rete

Connected / succeeded

La connettività TCP esiste.

A quel punto si sale di livello.

---

4. Controllare una chiamata HTTP/HTTPS

Uno degli strumenti più importanti:

curl -v https://hostname/path

Per vedere principalmente gli header:

curl -I https://hostname/path

Per ignorare temporaneamente errori certificato durante un test diagnostico:

curl -vk https://hostname/path

"-k" va usato per diagnosi, non come soluzione a un problema TLS.

POST

curl -vk -X POST https://hostname/path

Con body:

curl -vk \
  -H "Content-Type: application/json" \
  -d '{"test":"value"}' \
  https://hostname/path

---

5. Interpretare gli HTTP status

200

La richiesta ha avuto successo.

301 / 302

Redirect.

Esempio:

HTTP/1.1 302 Found
Location: https://altro-host/login

Questo non significa necessariamente errore WebSphere.

Bisogna vedere:

curl -vk https://host/path

e cercare:

< HTTP/1.1 302
< Location: ...

Per seguire automaticamente il redirect:

curl -vkL https://host/path

401

Problema di autenticazione.

403

Il server ha ricevuto la richiesta ma nega l'accesso.

404

Risorsa/path non trovato.

Possibili livelli:

- proxy
- IHS
- context root WebSphere
- applicazione

500

Errore lato server/applicazione.

Qui diventa importante controllare WebSphere.

502

Tipicamente proxy/LB non riesce correttamente a comunicare con il backend.

503

Servizio/backend non disponibile.

504

Timeout verso il backend.

---

6. Capire se una chiamata arriva al server

Uno dei controlli più utili è l'access log.

Per cercare i log:

find / -type f -name "*access*log*" 2>/dev/null

Oppure, conoscendo l'installazione:

find /opt /var /logs -type f -iname "*access*" 2>/dev/null

Seguire il log:

tail -f access.log

Ultime 100 righe:

tail -100 access.log

Cercare un IP:

grep "10.20.30.40" access.log

Cercare un endpoint:

grep "/api/payment" access.log

Cercare errori HTTP:

grep ' 500 ' access.log

---

7. Come leggere un access log

Esempio:

10.20.30.40 - - [26/Aug/2026:09:21:13] "POST /api/test HTTP/1.1" 500 1234

Significa:

10.20.30.40
↓
IP sorgente/client

POST
↓
metodo HTTP

/api/test
↓
endpoint chiamato

500
↓
HTTP status

1234
↓
dimensione risposta

Questo permette di rispondere a una domanda fondamentale:

«La chiamata è arrivata almeno al web server?»

Se non compare nell'access log, il problema probabilmente è prima del web server.

Se compare, possiamo continuare verso WebSphere.

---

8. Controllare WebSphere

I log fondamentali sono normalmente:

SystemOut.log
SystemErr.log

Tipicamente sotto una struttura simile a:

profiles/<PROFILE>/logs/<SERVER>/

Esempio:

cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1

Poi:

ls -ltr

SystemOut

tail -f SystemOut.log

Ultime 200 righe:

tail -200 SystemOut.log

SystemErr

tail -f SystemErr.log

---

9. SystemOut vs SystemErr

SystemOut.log

È il primo posto dove guardare per:

- startup applicazioni
- messaggi WebSphere
- applicazioni Java
- datasource
- connection pool
- errori applicativi
- warning
- deployment
- alcune eccezioni

SystemErr.log

Contiene principalmente output/errori scritti sullo standard error.

Non bisogna pensare:

«errore = sempre SystemErr»

Molte eccezioni importanti finiscono tranquillamente in "SystemOut.log".

---

10. Raffinare una ricerca nei log

Non conviene leggere migliaia di righe manualmente.

Error

grep -i "error" SystemOut.log

Exception

grep -i "exception" SystemOut.log

Failed

grep -i "failed" SystemOut.log

Java exception

grep -iE "exception|error|failed|caused by" SystemOut.log

Una ricerca molto utile:

grep -i -C 10 "exception" SystemOut.log

"-C 10" mostra 10 righe prima e 10 dopo.

Oppure:

grep -i -A 20 "exception" SystemOut.log

mostra le 20 successive.

---

11. Il punto più importante delle Java exception

Esempio:

java.sql.SQLException
...
...
Caused by: java.net.ConnectException: Connection refused

Non bisogna fermarsi alla prima riga.

Spesso la parte interessante è:

Caused by:

Quindi:

grep -i -A 30 "Caused by" SystemOut.log

Bisogna ricostruire la root cause.

Esempio:

ServletException
    ↓
SQLException
    ↓
SocketTimeoutException

Il problema reale potrebbe essere la connessione di rete verso il DB, non la servlet.

---

12. Cercare un errore avvenuto a un'ora precisa

Se l'utente dice:

«"Alle 09:32 la chiamata è fallita"»

non cerchiamo genericamente "error".

Cerchiamo prima il timestamp:

grep "09:32" SystemOut.log

Poi allarghiamo:

grep -C 30 "09:32" SystemOut.log

Questo riduce enormemente il rumore.

---

13. Verificare se WebSphere è attivo

Processi Java:

ps -ef | grep java

Più specificamente:

ps -ef | grep WebSphere

oppure:

ps -ef | grep server1

Per evitare il processo "grep" stesso:

ps -ef | grep '[s]erver1'

---

14. Controllare le porte in ascolto

ss -lntp

Cercare una porta:

ss -lntp | grep 9080

oppure:

netstat -lntp | grep 9080

Questo risponde alla domanda:

«Qualcuno sta effettivamente ascoltando sulla porta WebSphere?»

---

15. Processo vs porta

Supponiamo che:

ps -ef | grep server1

mostri WebSphere attivo.

Ma:

ss -lntp | grep 9080

non restituisca niente.

Questo significa che:

processo esistente ≠ applicazione necessariamente disponibile

Potrebbe esserci un problema durante lo startup o con il transport chain.

---

16. Datasource / Database

Se l'applicazione restituisce 500 e troviamo:

SQLException

bisogna investigare il DB.

Cercare:

grep -iE "datasource|sql|jdbc|connection" SystemOut.log

Possibili problemi:

- DB irraggiungibile
- credenziali errate
- password scaduta
- connection pool esaurito
- certificato DB
- JDBC URL errata
- timeout

---

17. Testare il DB dal server

Prima ancora di accusare WebSphere:

nc -vz dbhostname 1521

per Oracle, ad esempio.

Oppure PostgreSQL:

nc -vz dbhostname 5432

DB2:

nc -vz dbhostname 50000

Se la porta non è raggiungibile, WebSphere non può risolvere magicamente il problema.

---

18. SSL/TLS

Per verificare un endpoint TLS:

openssl s_client -connect hostname:443

Con SNI:

openssl s_client -connect hostname:443 -servername hostname

Informazioni utili:

- certificato presentato
- issuer
- chain
- handshake
- verify return code

Visualizzare il certificato:

openssl s_client -connect hostname:443 -servername hostname </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates

Otteniamo:

subject=
issuer=
notBefore=
notAfter=

Molto utile per certificati scaduti.

---

19. Certificati Java

Per un JKS:

keytool -list -v -keystore truststore.jks

Oppure:

keytool -list -keystore truststore.jks

Per cercare un alias:

keytool -list -v -keystore truststore.jks | grep -i alias

Errori tipici:

PKIX path building failed

unable to find valid certification path

SSLHandshakeException

Qui bisogna verificare:

certificato remoto → chain → truststore usato dalla JVM/WebSphere

---

20. IBM MQ

Errori importanti:

2035
2059
2009

2035

Authorization error.

Pensare a:

- utente
- CHLAUTH
- autorizzazioni MQ

2059

Queue Manager unavailable.

Controllare:

- hostname
- porta
- channel
- queue manager
- rete

2009

Connection broken.

Possibili cause:

- rete
- timeout
- queue manager
- channel
- firewall

Prima verifica:

nc -vz mqhostname 1414

Se nemmeno TCP funziona, bisogna partire dalla rete.

---

21. Disco

Controllare:

df -h

Molti problemi apparentemente applicativi possono derivare da filesystem pieno.

Inode:

df -i

Directory pesanti:

du -sh *

Ordinamento:

du -sh * | sort -h

---

22. Memoria

free -m

oppure:

free -h

Processi:

top

Oppure:

ps aux --sort=-%mem | head

---

23. CPU

top

Processi più pesanti:

ps aux --sort=-%cpu | head

Per una JVM WebSphere che consuma molta CPU, questo è solo il primo livello: successivamente può servire correlare PID, thread e thread dump.

---

24. Cercare file WebSphere

Quando non sappiamo dove sia qualcosa:

find /opt -name "SystemOut.log" 2>/dev/null

Configurazioni:

find /opt -name "server.xml" 2>/dev/null

JKS:

find / -name "*.jks" 2>/dev/null

Log:

find /opt -type f -name "*.log" 2>/dev/null

---

25. Verificare file modificati recentemente

Molto utile durante incidenti/deploy:

find /opt/app -type f -mmin -60

Trova file modificati nell'ultima ora.

Oppure:

ls -ltr

Gli ultimi modificati saranno in fondo.

---

26. Pipeline mentale per un errore 500

Caso:

«"Il client riceve HTTP 500."»

Step 1 — riprodurre

curl -vk https://app/path

Step 2 — Web server

tail -f access.log

La chiamata compare?

Step 3 — timestamp

Annotare:

09:42:31

Step 4 — WebSphere

grep -C 30 "09:42" SystemOut.log

Step 5 — cercare root cause

grep -iE "exception|caused by|error|failed" SystemOut.log

Step 6 — classificare

Se trovi:

SQLException

→ DB

Se trovi:

SSLHandshakeException

→ TLS/certificati

Se trovi:

SocketTimeoutException

→ rete/backend lento

Se trovi:

UnknownHostException

→ DNS

Se trovi:

Connection refused

→ host raggiungibile ma servizio/porta non disponibile

Se trovi:

MQRC 2059

→ MQ

---

27. Pipeline mentale per un 302

Caso:

POST → 302

Prima:

curl -vk https://host/path

Cercare:

Location:

Poi:

curl -vkL https://host/path

Domande:

1. Chi genera il redirect?
2. Dove punta?
3. Il redirect era previsto?
4. Lo genera IHS/proxy?
5. Lo genera WebSphere?
6. Lo genera l'applicazione/autenticazione?

Non bisogna trattare automaticamente un 302 come problema di rete.

---

28. Pipeline mentale per "l'applicazione è lenta"

Non partire immediatamente dalla CPU.

Bisogna capire:

Client
 ↓
Proxy
 ↓
WAS
 ↓
DB
 ↓
MQ
 ↓
API esterne

Misurare:

curl -w "\nTotal: %{time_total}s\n" -o /dev/null -s https://host/path

Più dettagliato:

curl -w '
DNS: %{time_namelookup}
TCP: %{time_connect}
TLS: %{time_appconnect}
TTFB: %{time_starttransfer}
TOTAL: %{time_total}
' -o /dev/null -s https://host/path

Questo permette di capire se il tempo viene perso prima della connessione o durante l'elaborazione backend.

---

29. La regola fondamentale

Durante un incidente non fare:

500
↓
apro SystemOut
↓
cerco "ERROR"
↓
vedo 200 errori
↓
non capisco niente

Fare invece:

ERRORE
   ↓
quando?
   ↓
da quale client?
   ↓
verso quale URL?
   ↓
quale HTTP status?
   ↓
arriva al Web Server?
   ↓
arriva a WAS?
   ↓
quale exception coincide temporalmente?
   ↓
qual è l'ultimo "Caused by" significativo?
   ↓
DB / MQ / TLS / DNS / rete / applicazione

---

30. Cheat sheet

Processo

ps -ef | grep java

Porte

ss -lntp

HTTP

curl -vk URL

DNS

nslookup host
dig host
getent hosts host

TCP

nc -vz host porta

TLS

openssl s_client -connect host:443 -servername host

WAS

tail -f SystemOut.log
tail -f SystemErr.log

Exception

grep -iE "exception|error|failed|caused by" SystemOut.log

Contesto

grep -C 20 "testo" SystemOut.log

Disco

df -h

RAM

free -h

CPU

top

File

find /opt -name "nomefile" 2>/dev/null

---

31. Metodo da usare sul lavoro

L'obiettivo finale è riuscire a classificare rapidamente il problema:

Sintomo| Prima verifica
Host non trovato| DNS
Timeout connessione| rete/firewall
Connection refused| servizio/porta
302| redirect + Location
401| autenticazione
403| autorizzazione
404| routing/context root
500| WAS/applicazione
502| proxy/backend
503| servizio/backend
504| timeout backend
SQLException| DB/datasource
PKIX| truststore/certificato
SSLHandshakeException| TLS
UnknownHostException| DNS
SocketTimeoutException| rete/backend
MQ 2035| autorizzazione MQ
MQ 2059| connessione/QM
filesystem 100%| spazio disco

Il principio è sempre lo stesso:

non cercare subito la soluzione; prima identifica con precisione il livello che sta fallendo.
