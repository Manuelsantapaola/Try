Guida rapida ai restart WebSphere e IHS su Linux

1. Prima cosa: cosa sto riavviando?

Sul server puoi avere almeno due componenti distinti:

CLIENT
   │
   ▼
F5 / Load Balancer
   │
   ▼
IHS / HTTP Server
   │
   ▼
WebSphere Application Server
   │
   ▼
Applicazione Java
   │
   ├── DB
   ├── MQ
   └── servizi esterni

Quindi "riavviare il server" può voler dire cose completamente diverse.

IHS

È il web server HTTP davanti a WebSphere.

Il suo processo è tipicamente:

httpd

WebSphere

È l'application server Java.

Troverai processi:

java

relativi alle JVM WebSphere.

---

2. Nel nostro ambiente

Sono presenti script sotto:

cd /scripts

Per esempio:

./start_admserver.sh

e:

./start_https-admserver.sh

Dalla schermata analizzata:

start_admserver.sh
        ↓
WebSphere

start_https-admserver.sh
        ↓
IHS / httpd

Gli script in "/scripts" sono quindi dei wrapper: al loro interno richiamano i veri comandi del prodotto.

Per sapere con certezza cosa fa uno script:

cat start_admserver.sh

oppure:

less start_admserver.sh

---

3. Prima di qualsiasi restart

Prima di riavviare qualcosa bisogna capire cosa non sta funzionando.

Controllare i processi:

ps -ef | grep java

per WebSphere.

Per IHS:

ps -ef | grep httpd

Possiamo raffinare:

ps -ef | grep '[h]ttpd'

e:

ps -ef | grep '[j]ava'

---

4. Verificare WebSphere

Prima del restart controllare:

ps -ef | grep java

Se conosci il nome della JVM:

ps -ef | grep '[s]erver1'

oppure il nome effettivamente utilizzato nell'ambiente.

Poi controllare:

SystemOut.log
SystemErr.log

Ad esempio:

tail -200 SystemOut.log

e:

grep -iE "error|exception|failed|caused by" SystemOut.log

Questo serve a evitare:

PROBLEMA
   ↓
restart immediato
   ↓
funziona
   ↓
nessuno sa cosa fosse successo

Meglio:

PROBLEMA
   ↓
raccolgo evidenze
   ↓
identifico componente
   ↓
restart se necessario
   ↓
verifico

---

5. Restart WebSphere

Se la procedura aziendale prevede gli script "/scripts", utilizzare quelli anziché inventare un comando diverso.

Concettualmente:

STOP WebSphere
     ↓
termina JVM Java
     ↓
verifico che sia DOWN
     ↓
START WebSphere
     ↓
nuova JVM
     ↓
controllo SystemOut
     ↓
applicazioni inizializzate

Il nome esatto dello script di stop va verificato nella directory:

ls -ltr /scripts

Potresti trovare, per esempio, una coppia del tipo:

start_admserver.sh
stop_admserver.sh

Non bisogna supporre il nome: va verificato.

---

6. Verificare che lo STOP sia realmente avvenuto

Dopo lo stop:

ps -ef | grep java

oppure filtrando la JVM interessata.

Il processo relativo alla JVM non deve più esserci.

Questo è importante perché non vogliamo fare:

STOP
 ↓
JVM ancora attiva
 ↓
START

senza capire cosa sia successo.

---

7. Avviare WebSphere

Nel vostro caso, ad esempio:

cd /scripts
./start_admserver.sh

Ma il comando va considerato riuscito solo dopo aver verificato il risultato.

Subito dopo:

ps -ef | grep java

e soprattutto:

tail -f SystemOut.log

---

8. Perché SystemOut è fondamentale dopo uno start

Quando parte WebSphere succedono molte cose:

START JVM
   ↓
inizializzazione WebSphere
   ↓
caricamento configurazione
   ↓
inizializzazione risorse
   ↓
datasource
   ↓
MQ
   ↓
applicazioni
   ↓
server pronto

La JVM potrebbe quindi esistere:

ps -ef | grep java

ma un'applicazione potrebbe comunque aver fallito lo startup.

Per questo:

processo Java presente ≠ servizio sicuramente funzionante.

---

9. Se lo start restituisce "not found"

Caso simile alla schermata:

.../profiles/svr/bin/starServer.sh: not found

Non significa necessariamente che WebSphere sia guasto.

Significa prima di tutto che lo script wrapper sta cercando di eseguire qualcosa che il sistema non trova.

Controllare lo script:

cat /scripts/start_admserver.sh

Poi verificare il path richiamato:

ls -l /apps/WebSphere9S/profiles/svr/bin/

e verificare il nome del comando.

Gli script standard WebSphere usano normalmente:

startServer.sh
stopServer.sh
serverStatus.sh

Quindi un riferimento a:

starServer.sh

andrebbe controllato attentamente.

---

10. Restart IHS

IHS è separato da WebSphere.

Nel vostro ambiente lo script visto è:

./start_https-admserver.sh

Dalla schermata restituisce:

httpd (pid 2673681) already running

Questo significa:

IHS
 ↓
processo httpd
 ↓
già attivo

Non significa che WebSphere sia attivo.

---

11. Controllare IHS

ps -ef | grep '[h]ttpd'

Se trovi processi "httpd", IHS è in esecuzione.

Puoi inoltre controllare le porte:

ss -lntp

o filtrare, ad esempio:

ss -lntp | grep ':443'

La porta effettiva dipende dalla configurazione dell'ambiente.

---

12. IHS UP ma WAS DOWN

Può tranquillamente succedere:

CLIENT
   │
   ▼
IHS ✓
   │
   ▼
WAS ✗

Quindi:

httpd already running

non dimostra che l'applicazione funzioni.

IHS può essere perfettamente attivo mentre il backend WebSphere è fermo.

---

13. WAS UP ma IHS DOWN

Può verificarsi anche il contrario:

CLIENT
   │
   ▼
IHS ✗
   │
   ▼
WAS ✓

WebSphere funziona internamente, ma il traffico esterno non riesce ad arrivarci attraverso IHS.

In questo caso riavviare WAS potrebbe essere completamente inutile.

---

14. Quando riavviare WAS?

Può avere senso in caso di:

- JVM bloccata;
- modifica di configurazione che richiede restart;
- applicazioni/resources non inizializzate correttamente;
- problemi runtime;
- connection pool in stato problematico;
- problemi di memoria;
- attività di deploy/manutenzione che lo prevedono.

Ma prima bisogna raccogliere evidenze.

---

15. Quando NON partire da un restart WAS

Se trovi:

UnknownHostException

probabile problema DNS.

Se trovi:

Connection refused

verso un backend:

WAS → DB/MQ/API
        ✗

controllare il backend.

Se trovi:

SSLHandshakeException

controllare TLS/certificati/truststore.

Se trovi:

MQRC 2059

investigare MQ/connettività.

Se il filesystem è:

100%

investigare spazio disco.

Un restart WAS non corregge necessariamente nessuno di questi problemi.

---

16. Evitare kill -9 come normale procedura

Non fare direttamente:

kill -9 PID

perché "SIGKILL" termina brutalmente il processo.

La sequenza normale deve essere:

STOP previsto
     ↓
attendo
     ↓
verifico processo
     ↓
solo se lo stop è bloccato:
procedura di escalation prevista

---

17. Procedura completa di restart WAS

PRIMA

ps -ef | grep java

Controllo JVM.

Poi:

tail -200 SystemOut.log

Controllo errori.

Eventualmente:

df -h
free -h
top

se sospetto problemi della macchina.

STOP

Utilizzo lo script previsto nell'ambiente:

cd /scripts
./<script_stop_WAS>

VERIFICA DOWN

ps -ef | grep java

La JVM interessata deve essere terminata.

START

cd /scripts
./start_admserver.sh

VERIFICA STARTUP

tail -f SystemOut.log

Controllo che WebSphere e le applicazioni partano correttamente.

VERIFICA PROCESSO

ps -ef | grep java

VERIFICA PORTA

ss -lntp

VERIFICA APPLICAZIONE

Se disponibile:

curl -vk https://host/path

---

18. Procedura mentale velocissima

Quando qualcuno dice:

«"Facciamo restart."»

Prima chiedersi:

COSA NON FUNZIONA?
       ↓
IHS?
WAS?
Applicazione?
DB?
MQ?
Rete?
       ↓
QUAL È IL COMPONENTE?
       ↓
RACCOLGO LOG
       ↓
RESTART SOLO DEL COMPONENTE NECESSARIO
       ↓
VERIFICO

---

19. I tre concetti da ricordare

1 — IHS e WAS non sono la stessa cosa

IHS = httpd / web server

WAS = java / application server

2 — Uno script "start" non garantisce che il servizio sia partito

Bisogna verificare:

processo
+
log
+
porta
+
applicazione

3 — Il restart non sostituisce la diagnosi

Restart → servizio torna UP

non significa automaticamente:

problema risolto definitivamente

Potresti aver semplicemente eliminato temporaneamente lo stato che causava il problema.

Mini cheat sheet

# Cosa c'è negli script
ls -ltr /scripts

# Leggere uno script
cat /scripts/start_admserver.sh

# JVM WebSphere
ps -ef | grep java

# IHS
ps -ef | grep '[h]ttpd'

# Porte
ss -lntp

# Ultimi log WAS
tail -200 SystemOut.log

# Seguire WAS durante lo startup
tail -f SystemOut.log

# Cercare problemi
grep -iE "error|exception|failed|caused by" SystemOut.log

# Disco
df -h

# Memoria
free -h

# Connettività backend
nc -vz HOST PORTA

Sequenza da memorizzare: "identifico → raccolgo log → stop → verifico DOWN → start → SystemOut → processo → porta → test applicativo".
