## Rozdział #5: Replication

> The major difference between a thing that might go wrong and a thing that cannot possibly go wrong is that when a
> thing that cannot possibly go wrong goes wrong it usually turns out to be impossible to get at or repair.
> — Douglas Adams, Mostly Harmless (1992)

### Leaders and Followers

Jedno z najpopularniejszych podejść do replikacji to single-leader replication. Jedna replika działa jako lider –
wszystkie zapisy przechodzą przez lidera, który zapisuje dane lokalnie, a następnie wysyła zmiany do followersów jako
replication Lag lub Change Stream. Każdy follower aktualizuje lokalną bazę danych. Kiedy klient odczytuje dane, może je
pobrać od lidera lub od replik. Podejście to wykorzystywane jest w PostgreSQL, MySQL, MongoDB, Kafka, RabbitMQ.

#### Synchronous Versus Asynchronous Replication

Replikacja synchroniczna wymaga potwierdzenia od replik – kiedy jednak replika nie odpowiada, zapisy zostają wstrzymane.
Replikacja asynchroniczna nie czeka na potwierdzenie – kiedy jednak replikacja się nie powiedzie, a lider przestanie
działać, dane zostaną utracone. Jako kompromis, stosuje się mix tych dwóch podejść – jedna replika jest synchroniczna a
reszta a synchroniczna – jest to tak zwane podejście semi-synchronous. Microsoft Azure Storage korzysta z chane
replication – który jest wariantem replikacji synchronicznej.

![Leader-based replication with one synchronous and one asynchronous follower](/assets/images/ddia/05-01.png "Leader-based replication with one synchronous and one asynchronous follower")

#### Setting Up New Followers

Dodanie nowego follower przy ciągłym zapisie do lidera może skutkować niespójnością danych podczas kopiowania pliku –
dlatego przed kopiowaniem należy utworzyć Snapshot. Snapshot jest kopiowany przez followera, a następnie follower
dociąga tylko zmiany.

#### Handling Node Outages

Dowolny Node może być niedostępny. Jeśli będzie to follower, to recovery jest proste – wystarczy poprosić log zmian od
ostatniego loga dostępnego na followerze. Jeśli będzie to lider, to sprawy się komplikują. Follower zostaje nowym
liderem, klienty zapisują dane do nowego lidera a pozostałe repliki odczytują zmiany z nowego lidera. Jest to tak zwany
failover. Failover może być manualny lub automatyczny. W automatycznym trzeba określić że lider jest niedostępny – na
przykład nie odpowiada przez 30 sekund, potem następuje wybór nowego lidera – tak zwany election proces, wymagający
zgody większości followerów. Następuje rekonfiguracja replik i klientów. Jeśli Lider wróci, to powinien zostać
followerem.

W trakcie failover wiele rzeczy może pójść nie tak:

1. jeśli korzystamy z asynchronicznej replikacji, nowy Lider może nie otrzymać wszystkich danych – które zostaną
   utracone
2. czasem dwa node mogą wierzyć, że są liderami – tak zwany Split Brain. Obydwa akceptują zapisy – to również prowadzi
   do
   utraty danych
3. utrata danych jest szczególnie niebezpieczna, kiedy korzystamy z dodatkowego systemu na przykład Redis. Dwóch liderów
   może dla różnych danych użyć tego samego primary key – przez to odczytując dane z cache, można pobrać dane na
   przykład innego użytkownika.
4. istotne jest określenie, jaki timeout jest właściwy, aby orzec brak lidera – zbyt mały spowoduje częste failovery, co
   jeszcze pogorszy problemy na przykład z wolną siecią

#### Implementation of Replication Logs

Istnieją różne implementacje replikacji.

**Statement-based**: Lider loguje i wysyła każda polecenie zapisu – dla RDB będą to Create, Update, Delete – do
followerów. Problem pojawia się z wartościami niedeterministycznymi jak _NOW()_, _RAND()_ lub poleceniach, które zależą
od stanu w bazie na przykład:

*update … where …* - takie polecenia muszą być wykonane w tej samej kolejności.

**Write Ahead Log (WAL)** - zawiera sekwencję bajtów zawierającą dane jakie mają być zapisane w bazie. Pełni funkcję
kopii zapasowej, ale może być też przesłany do replik, które po zaaplikowaniu będą posiadały kopię danych. Przez to, że
jest to metoda niskopoziomowa – informująca które bajty zmieniono w którym bloku na dysku – jest zależne od storage
engine. Może to być problematyczne przy upgrade bez down-time, gdzie różne wersje bazy są online. Porter SQL lub Oracle
korzystają z tej metody.

**Logical (row-based) replication** – alternatywne rozwiązanie wykorzystuje dwa formaty: jeden dla Storage Engine, drugi
dla replikacji. Wpis dla replikacji zawiera dane na poziomie wiersza – wartości dodanych kolumn, primary key kolumny
usuniętej itp. Taki format może być też czytany przez narzędzia na przykład do analizy danych – tak zwany Change Data
Capture (CDC).

### Problems with Replication Lag

#### Reading Your Own Writes

Replikacja synchroniczna gdzie zapis zostaje potwierdzony przez lidera dopiero po potwierdzeniu od replik, może trwać
długo albo może przestać działać, jeśli replika padła. Replikacja asynchroniczna sprawia, że lider i follower mogą przez
jakiś – oby krótki – czas zwracać inne dane. Ostatecznie replika powinna mieć aktualne dane – jest to tak zwany eventual
consistency.

![A user makes a write, followed by a read from a stale replica. To prevent this anomaly, we need read-after-write consistency](/assets/images/ddia/05-02.png "A user makes a write, followed by a read from a stale replica. To prevent this anomaly, we need read-after-write consistency")

W sytuacji jak na rysunku 5.3 użytkownik zapisał dane, ale ich nie widzi, ponieważ odczyt odbywa się z repliki, która
nie otrzymała jeszcze kopii danych. Użytkownik podejrzewa, że zapis się nie powiódł – rozwiązaniem może być zapewnienie,
że użytkownik odczytuje dane, które może edytować – na przykład swój profil – z instancji lidera. Inne rozwiązanie to na
przykład śledzenie czasu ostatniej aktualizacji i przed upływem minuty – zanim repliki zsynchronizują dane dla tego
użytkownika – kierowanie odczytów do lidera, a po minucie do replik.

#### Monotonic Reads

Monotonic reads to gwarancja, że użytkownik odczytuje dane w kolejności zapisu. Na rysunku 5.4 użytkownik najpierw widzi
dane, a potem one znikają. Żeby zapewnić tę gwarancję, odczyt nie powinien być kierowany losowo do replik, ale replika
powinna być wybierana na przykład na podstawie hash dla user ID

![A user first reads from a fresh replica, then from a stale replica. Time appears to go backward. To prevent this anomaly, we need monotonic reads](/assets/images/ddia/05-03.png "A user first reads from a fresh replica, then from a stale replica. Time appears to go backward. To prevent this anomaly, we need monotonic reads")

#### Consistent Prefix Reads

Trzeci przypadek — Consistent Prefix Reads — jest typowy dla baz rozproszonych z kilkoma partycjami. Występuje, kiedy
zapisy są ciągiem przyczynowo-skutkowym, ale przez to, że są przechowywane na różnych partycjach, mogą być odczytane w
odwrotnej kolejności. Rozwiązaniem jest przechowywanie skorelowanych zapisów na jednej partycji.

![If some partitions are replicated slower than others, an observer may see the answer before they see the question](/assets/images/ddia/05-04.png "If some partitions are replicated slower than others, an observer may see the answer before they see the question")

#### Solutions for Replication Lag

W systemach gdzie dane nie są rozproszone i replikowane, replication lag nie jest problemem. W systemach rozproszonych
należy odpowiedzieć sobie na pytanie – co się stanie, jeśli replication lag wyniesie kilka minut lub godzin. Jeśli jest
to problem, warto rozważyć wspomniane scenariusze.

### Multi-Leader Replication

#### Use Cases for Multi-Leader Replication

Replikacja z wieloma liderami ma sens, jeśli liderzy rozproszeni są w wielu centrach danych. Dzięki temu operacje zapisu
obsługiwane są przez data centers rozlokowane geograficznie, zmniejszając latency. W razie awarii data center, inne data
center może przejąć jego funkcję, a w razie problemów sieciowych, zapytania mogą podążyć innymi ścieżkami w internecie –
sieć data center nie staje się single point of failure. Multileader replication w PostgreSQL umożliwia narzędzie BDR, w
mySQL — Tungster Replication. Wadą tej replikacji jest konieczność rozwiązywania konfliktów.

###### Clients with offline replication

Mimo że temat wydaje się egzotyczny – okazuje się, że w pewnym stopniu powszechny. Aplikacje działające offline jak
kalendarz, czy umożliwiające współpracę wielu osób jak Google Docs posiadają lokalną bazę danych pełniącą funkcję
lidera. Zapisy propagowane są do innych liderów — żeby uniknąć konfliktów — można albo zablokować edycję pozostałym
użytkownikom — co nie byłoby najlepszym rozwiązaniem — albo można przesyłać małe zmiany nawet takie jak naciśnięcie
klawisza i rozwiązywać konflikty.

#### Handling Write Conflicts

![A write conflict caused by two leaders concurrently updating the same record](/assets/images/ddia/05-05.png "A write conflict caused by two leaders concurrently updating the same record")

Jak pokazuje rysunek 5.7, w systemach z wieloma liderami, konflikt może wystąpić łatwo – edycja tytułu tego samego
dokumentu, zarezerwowanie sali na spotkanie w kalendarzu – operacje te zapisane na różnych liderach prowadzą do
konfliktu i niespójnych danych. Eventual Consistency jest kluczowym wymaganiem – stąd potrzeba rozwiązania konfliktów.
Najlepiej konfliktów unikać – n.p.: zapisy dotyczące dokumentu kierować na jednego lidera. Jeśli jednak trzeba rozwiązać
konflikt, może się to wiązać z utratą danych – n.p.: w strategii, w której wygrywa na przykład najwyższy timestamp.

Można też stworzyć logikę, która zapisze informacje o konflikcie i w momencie odczytu poprosi użytkownika o jego
rozwiązanie. Ogólnie temat jest skomplikowany.

###### Automatic Conflict Resolution

Automatyczne rozwiązywanie konfliktów jest skomplikowane, jednak istnieją pewne rozwiązania, które to ułatwiają takie
jak Conflict Free Replicated Data Types (CRDTs), Mergeable Persistent Data Structures, Operational Transformations.

### Multi-Leader Replication Topologies

Mając kilku liderów, można stworzyć różne topologie replikacji jak na rysunku 5.8.

![Three example topologies in which multi-leader replication can be set up](/assets/images/ddia/05-06.png "Three example topologies in which multi-leader replication can be set up")

Topologie Circular i Star są zawodne, ponieważ awaria jednego Note wymaga rekonfiguracji manualnej. W all-to-all z kolei
może dojść do braku konsystencji danych jak na 5.9, kiedy z powodu wolnej sieci lider-1 Napisał dane, ale tylko na node
lidera-2

![With multi-leader replication, writes may arrive in the wrong order at some replicas](/assets/images/ddia/05-07.png "With multi-leader replication, writes may arrive in the wrong order at some replicas")

### Leaderless Replication

Replikacja bez lidera implementowana jest przez DynamoDB oraz w bazach Open Source jak Riak, Cassandra, Voldemort. W
tych bazach zwykle to klient wysyła zapytania do wszystkich node. Czasem robi to node koordynujący. Kiedy jeden z node
jest niedostępny jak na 5.10, zapisy trafiają tylko na dwa node. Żeby klient odczytujący dane pobrał najnowsze dane,
pobiera je ze wszystkich node i wybiera wartość z najwyższą wersją.

![A quorum write, quorum read and read repair after a node outage](/assets/images/ddia/05-08.png "A quorum write, quorum read and read repair after a node outage")

Jeśli klient wykryje niespójność, może wysłać zapytanie aktualizująca dane do node, który jest z tyłu, tak zwany Read
Repair. Inna opcja to Anti-Entropy Process, który w tle wyszukuje i aktualizuje niekonsystentne dane.

###### Quorums for reading and writing

To na ilu node zapis musi zakończyć się sukcesem i to z ilu node należy odczytać dane, określa wartość read and write
quorum. Zwykle przy n node, jest to w + r = n/2 zaokrąglone w górę. Dla n=3, w=r=2. oznacza to, że zapis musi się udać
na
dwóch node i odczyt musi być co najmniej dwóch node. Jeśli chcemy mieć szybkie odczyty, możemy ustalić, że w=3, r=1,
wtedy wystarczy odpytać jednego node, ale jeśli którykolwiek node przestanie działać, żaden zapis się nie uda.

#### Limitations of Quorum Consistency

Możemy celowo określić w+r&lt;=n, żeby zmniejszyć opóźnienia i zwiększyć dostępność bazy, kosztem niepoprawnych
odczytów. Jednak nawet jeśli w+r>n, dane mogą okazać się nieaktualne. Jednymi z przykładów są dwa równoległe zapisy,
które wykonają się w różnej kolejności na node lub poprawny zapis na części z replik, na których występują zapisy i
niepoprawny zapis na przykład z powodu pełnego dysku na pozostałych wymaganych replikach – wtedy nie nastąpi rollback.

#### Sloppy Quorums and Hinted Handoff

Sloppy Quorum występuje, kiedy któryś z node, na jakie powinien trafić zapis, jest niedostępny. Wtedy można tymczasowo
zapisać dane na innym node, żeby zwiększyć durability. Jednak jeśli odczyt następuje z określonych node, odczytane dane
mogą być nieaktualne.

#### Detecting Concurrent Writes

Równoległe zapisy nie oznaczają zapisów, które wydarzyły się w tym samym momencie – ze względu na problemy z
synchronizacją zegarów, trudno byłoby to ustalić. Są to zapisy, które nie wiedziały o sobie w momencie wystąpienia.
Takie zapisy mogą doprowadzić do braku spójności danych jak na 5.12

![Concurrent writes in a Dynamo-style datastore: there is no well-defined ordering](/assets/images/ddia/05-09.png "Concurrent writes in a Dynamo-style datastore: there is no well-defined ordering")

Jedną z metod przywrócenia spójności jest nadpisanie danych – wygrywa zapis o wyższym timestamp – co powoduje utratę
danych. Inna metoda to wprowadzenie relacji przyczyna skutek. Jeśli jakiś zapis ma określonego poprzednika, to może
nadpisać dane. Można to osiągnąć poprzez wprowadzenie wersjonowania zapisów – zapis podaję wersję poprzednika. Jeśli
wartość jest niższa niż ostatnia wartość w bazie – zapis jest odrzucony.

### Podsumowanie

Replikacja może służyć kilku celom:

* **Wysoka dostępność** – utrzymanie działania systemu, nawet gdy jedna maszyna, kilka maszyn lub całe centrum danych
  uległy awarii
* **Praca w trybie rozłączenia** – umożliwienie aplikacji kontynuowania pracy, gdy wystąpi przerwa w dostępie do sieci
* **Minimalizowanie opóźnień** – umieszczanie danych geograficznie blisko użytkowników, dzięki czemu mogą oni szybciej z
  nich korzystać
* **Skalowalność** – zdolność do obsługi większej ilości odczytów niż może obsłużyć pojedyncza maszyna przez wykonywanie
  odczytów na replikach

Utrzymywanie kopii tych samych danych na kilku maszynach okazuje się niezwykle trudnym problemem. Musimy radzić sobie z
niedostępnymi węzłami i przerwami w dostępie do sieci.

Można wyszczególnić trzy podejścia do replikacji

* Single-leader replication – klienci wysyłają wszystkie zapisy do pojedynczego węzła (lidera), który wysyła dane do
  pozostałych replik (followers). Odczyty mogą być wykonywane na dowolnych replikach, ale odczyty te mogą być
  nieaktualne
* Multi-leader replication – klienci wysyłają każdy zapis do jednego z kilku węzłów-liderów, z których każdy może
  zaakceptować zapis. Liderzy przesyłają dane do siebie i do replik
* Leaderless replication – klienci wysyłają każdy zapis do kilku węzłów i odczytują z kilku węzłów równolegle w celu
  wykrycia i skorygowania węzłów z nieaktualnymi danymi

Każde podejście ma swoje wady i zalety. Replikacja z jednym liderem jest popularna, ponieważ jest dość łatwa do
zrozumienia i nie trzeba się martwić o rozwiązywanie konfliktów. Replikacja z wieloma liderami i replikacja bez liderów
mogą być bardziej niezawodne w obecności wadliwych węzłów, przerw w sieci i opóźnień — kosztem tego, że są trudniejsze w
implementacji i zapewniają jedynie słabe gwarancje spójności.

Replikacja może być synchroniczna lub asynchroniczna, co ma wpływ na zachowanie systemu w przypadku awarii. Chociaż
replikacja asynchroniczna może być szybka, gdy system działa płynnie, w razie awarii może nastąpić utrata danych.

Opóźnienia mogą powodować dziwaczne zachowania aplikacji. Można jednak postulować spełnienie kilku gwarancji, aby
zapewnić spójność danych

* Read-after-write consistency – użytkownicy powinni zawsze widzieć dane, które sami przesłali
* Monotonic reads – po tym, jak użytkownicy zobaczą dane w pewnym momencie, nie powinni później widzieć danych z
  jakiegoś wcześniejszego punktu w czasie
* Consistent prefix reads — użytkownicy powinni widzieć dane w stanie, który ma sens przyczynowo-skutkowy — na przykład
  widząc pytanie i jego odpowiedź we właściwej kolejności
