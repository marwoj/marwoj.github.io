## Rozdział #6: Partitioning

*Clearly, we must break away from the sequential and not limit the computers. We must state definitions and provide for
priorities and descriptions of data. We must state relation ‐ ships, not procedures. — Grace Murray Hopper, Management
and the Computer of the Future (1962)*

Partycjonowanie poprawia skalowalność. Dane zostają rozlokowane na wielu node, zwiększając throughput zapisu i odczytu –
ponieważ dane przetwarzane są równolegle na wielu maszynach. Partycjonowanie często idzie w parze z replikacją, jak to
jest na rysunku poniżej. W takim przypadku, do wyzwań skalowalności dochodzą wyzwania partycjonowania.

![Combining replication and partitioning: each node acts as leader for some partitions and follower for other partitions.](/assets/images/ddia/06-01.png "Combining replication and partitioning: each node acts as leader for some partitions and follower for other partitions.")

### Partitioning of Key-Value Data

Dane, byłyby rozłożone najbardziej równomiernie, przy losowym wyborze partycji. Prz odczycie, nie byłoby jednak wiadomo,
gdzie szukać danych. Dlatego wybiera się klucz, który służy do wyboru partycji. Dobrze wybrany klucz, pozwala unikać
hot-spot, czyli jednej partycji obsługującej większość zapisów i odczytów, a dane są rozłożone równomiernie – czyli nie
są skewed.

#### Partitioning by Key Range

Jedna z metod podziału to Key Range, gdzie partycja przechowuje klucze z wybranego zakresu jak n.p.: książkowa
encyklopedia. Zakresy kluczy mogą być wybrane manualnie przez admina lub automatycznie n.p.: A-D|E-O|P-Z. Taki podział
jest efektywny dla range-queries. Przy wyborze klucza jako n.p.: timestamp może to prowadzić do hot-spot – najnowsze
dane, które zapisywane na jeden node i prawdopodobnie też czytane z tego samego node – żeby zaprezentować ostatnie dane.
Można jednak zastosować prefix będący n.p. nazwą sensora IoT produkującego dane, żeby rozłożyć dane bardziej
równomiernie.

#### Partitioning by Hash of Key

Inna metoda partycjonowania to wyliczenie silnego kryptograficznie hash na podstawie klucza. Uwaga! – Java hash nie
spełnia tych warunków, bo na różnych procesorach, wartość hash może się różnić dla tych samych wartości. Hash eliminuje
jednak hot-spot i skewed workload. Stosujące jednak ten sposób, tracimy możliwość efektywnych range-queries – czyli
zapytań po zakresie klucza – zamiast tego stosuje się zapytania po range dla hash klucza.

MongoDB dla takich range-queries wykonuje zapytania na wszystkich node. Cassandra stosuje compound index – ale tylko
część składowych indeksu jest hashowana – aby umożliwić równomierne partycjonowanie. Pozostała część może służyć do
range queries.

#### Partitioning and Secondary Indexes

Partycjonowanie po kluczu głównym pomaga w równomiernym rozłożeniu danych na node, jednak kiedy dodatkowo stosujemy
secondary index, to wyszukiwane dane mogą znajdować się na wielu partycjach. Żeby korzystanie z secondary index było
możliwe i efektywne stosuje się

##### Partitioning by document

Inaczej zwane local index, gdzie każda partycja zawiera lokalny indeks dokumentów przechowywanych na tej partycji.
Pozwala to na szybki zapis, jednak odczyt wymaga odpytania wszystkich partycji. Taki odczyt wszystkich partycji nazywany
jest **scatter/gather** i jest wrażliwe na tail latency – najwolniejszy odczyt spowalnia całą operację. To podejście
stosuje MongoDB, Raik, Cassandra, Elasticsearch. Przykład to baza ofert sprzedaży samochodów – każda partycja zawiera
indeks po marce i kolorze – indeks na partycji zawiera wszystkie marki oraz kolory. Podejście to wymaga też agregacji i
sortowania po stronie klienta bazy danych.

![Partitioning secondary indexes by document.](/assets/images/ddia/06-02.png "Partitioning secondary indexes by document.")

##### Partitioning by term

Inaczej tzw. global index. W tym podejściu partycjonowane są nie tylko dane, ale i indeks. Jedna partycja zawiera
podzbiór wartości secondary index lub jego hash. W przykładzie z samochodami jedna partycja zawiera indeks samochodów o
kolorach red/black a druga silver/yellow. Podobnie podzielone są marki i inne secondary index. Usprawnia to odczyt, bo
na podstawie **term**, wiadomo, którą partycję odpytać. Zapis wymaga jednak zapisu na wielu partycjach jako jedna
transakcja – co może być wolne albo asynchronicznego zapisu – co przez pewien czas może powodować niespójność między
indeksem a tabelą. To podejście stosuje n.p. Amazon DynamoDB

![Partitioning secondary indexes by term.](/assets/images/ddia/06-03.png "Partitioning secondary indexes by term.")

### Rebalancing Partitions

Z czasem rozmiar danych rośnie – chcąc obsłużyć zwiększony throughput, konieczne może być dodanie kolejnego node i musi
nastąpić rebalancing, tj. dane muszą zostać przeniesione i równomiernie rozłożone. Podstawowe wymagania dla rebalancing
to:

* równomierne rozłożenie danych dla operacji read/write
* nieprzerwane działanie w trakcie rebalancing
* przenoszenie tylko tyle danych ile jest to konieczne, żeby zminimalizować zbędny ruch sieciowy i operacje I/O

#### Strategies for rebalancing

Strategia rebalancing zależy od stosowanego pod spodem partycjonowania. Partycjonowanie na podstawie *hash mod(N)*, mimo
że najbardziej intuicyjne, uniemożliwia efektywny rebalancing – dodanie nowego node wymaga przeniesienia prawie
wszystkich danych, bo *hash mod(N)* zwróci całkiem inne wartości niż *hash mod(N+1)*. Dlatego stosuje się inne metody
jak:

* Fixed number of partitions — tworzy się więcej partycji niż node n.p.: 1000 partycji przy 10 node. Rebalancing
  przenosi wybrane partycje na nowy node, a na czas transferu „stary” node obsługuje zapis i odczyt. Problematyczny tu
  jest odpowiedni wybór liczby partycji – za mało utrudni rebalancing, za dużo dda narzut na ich obsługę. To podejście
  stosuje Elasticsearch, Raik, Couchbase, Voldemort
* Dynamic rebalancing — niepoprawne założenia co do wyboru zakresów dla key-partitioning, wprowadzą nierównomierne
  rozłożenie danych. Zły wybór liczby partycji utrudni rebalancing. Zamiast podejmowania decyzji na starcie, można
  dynamicznie dzielić partycje. Na starcie node zawiera jedną partycję, ale gdy ta urośnie, jest dzielona, a część
  danych jest przenoszona na inny node. Podejście to wspiera HBase, RethinkDB, MongoDB. Dodatkowo HBase i MongoDB, żeby
  rozłożyć już ruch na starcie, umożliwiają stworzenie od razu kilku partycji, jeśli takie będzie wymaganie
* Partitioning proportional to nodes — w tym podejściu liczba partycji na node jest stała. Kiedy dołącza nowy node,
  część partycji jest dzielona, a dane są przenoszone na nowy node – co zachowuje sensowny rozmiar partycji

#### Operations: Automatic or Manual Rebalancing

Do rozstrzygnięcia pozostaje kwestia – rebalancing automatyczny czy manualny? Automatyczny jest wygodny, ale może być
nieprzewidywalny i przeciążać sieć, kiedy reguły nie są dobrze określone. Manualny może być trudny w przeprowadzeniu.
Rozsądne podejście to automatyczny plan na rebalancing i jego manualna akceptacja – mimo że to podejście jest
wolniejsze, to jest bardziej niezawodne i kontrolowane.

### Request Routing

Kiedy klient odczytuje/zapisuje dane, musi wiedzieć, z którym node się komunikować. Istnieją trzy podstawowe podejścia
do
tego problemu:

* klient pyta dowolnego node, a ten wie, gdzie są dane i tam kieruje zapytanie
* klient pyta dedykowanego pośrednika, a ten dział jak load-balancer, przesyłając request dalej
* klient zna topologię node i rozłożenie partycji – łączy się bezpośrednio do node

Niezależnie od typu routingu, informacja o zmianach po rebalancing musi być jakoś rozpropagowana. Może do tego służyć
serwis koordynujący jak Zookeper. Można też zastosować Gossip Protocol lub Consensus Protocol, które nie wymagają
zewnętrznego koordynatora. Consensus Protocol jest wykorzystywany w Kafka KRaft mode i zapewnia silniejsze gwarancje
konsystencji, przy nieco niższej wydajności.

![Three different ways of routing a request to the right node.](/assets/images/ddia/06-04.png "Three different ways of routing a request to the right node.")

### Summary

Partycjonowanie to **podział dużego zbioru danych na mniejsze części**. Partycjonowanie jest niezbędne, gdy ilość danych
przekracza możliwości przechowywania i przetwarzania na pojedynczej maszynie.

Celem partycjonowania jest **równomierne rozłożenie obciążenia** pomiędzy wiele maszyn, aby uniknąć tzw. "hot spots" (
węzłów o nadmiernym obciążeniu). Wybór odpowiedniego schematu partycjonowania i dynamiczne równoważenie partycji podczas
dodawania lub usuwania węzłów ma kluczowe znaczenie.

Istnieją dwa główne podejścia do partycjonowania:

* **key range partitioning** – dane są sortowane, a każda partycja zawiera klucze w określonym zakresie. Umożliwia to
  efektywne zapytania po zakresie, ale niesie ryzyko przeciążenia węzłów, jeśli aplikacja często odwołuje się do
  sąsiadujących kluczy. W razie potrzeby partycje są dynamicznie dzielone na mniejsze zakresy.
* **hash partitioning** – stosuje się funkcję haszującą, a każda partycja przechowuje zakres wartości haszy. Utrudnia to
  zapytania po zakresie, ale lepiej rozkłada obciążenie. W tej metodzie często tworzy się stałą liczbę partycji,
  przydzielając ich kilka do każdego węzła i przenosząc całe partycje między węzłami w razie zmian w klastrze.

Możliwe są także podejścia hybrydowe, np. użycie klucza złożonego, gdzie jedna część identyfikuje partycję, a druga
określa kolejność sortowania.

Omówiliśmy również wpływ partycjonowania na **secondary index**, który również wymaga podziału. Istnieją dwa podejścia:

* **Indeksy lokalne (document-partitioned indexes)** – przechowywane w tej samej partycji co klucz główny i wartość.
  Ułatwia to operacje zapisu, ale zapytania wymagają przeszukiwania wszystkich partycji.
* **Indeksy globalne (term-partitioned indexes)** – indeksy są podzielone według wartości indeksowanej. Zapisy wymagają
  aktualizacji wielu partycji, ale odczyty mogą być obsługiwane przez pojedynczą partycję.

Każda partycja działa w dużej mierze niezależnie, co pozwala na skalowanie bazy danych. Jednak operacje obejmujące wiele
partycji mogą być trudne do zarządzania – np. co się dzieje, gdy zapis do jednej partycji się powiedzie, a do innej nie?
Na to pytanie odpowiemy w kolejnych rozdziałach.
