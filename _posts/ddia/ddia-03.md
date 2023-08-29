## Rozdział #3: Storage and Retrieval

>If you keep things tidily ordered, you’re just too lazy to go searching

Upraszczając, baza danych powinna pozwolić na zapis danych i ich pobranie, kiedy to jest potrzebne. Dlaczego zatem
oprócz języka dostępu do danych, powinniśmy być świadomi, jak baza działa pod spodem?
Aby móc ją konfigurować i wybrać bazę odpowiednią dla naszych potrzeb.

### Data Structures That Power Your Database

Najprostszą bazą danych może być nawet skrypt zapisujący i odczytujący dane z pliku. Zapis będzie wydajny, bo
dopisywanie na koniec pliku jest najszybszą operacją. Odczyt wymaga jednak w najgorszym scenariuszu przejrzenia całego
pliku. Taka baza działa, ale nie obsługuje wielowątkowości, nie usuwa duplikatów, nie obsługuje błędów. W prawdziwych
bazach, żeby przyspieszyć odczyt, tworzone są osobne struktury danych, tj. indeksy. Tworzenie indeksu spowalnia zapis,
ale jest to trade off pomiędzy szybkością zapisu i odczytu.

#### Hash Indexes

Jest to najprostszy indeks typu klucz-wartość, gdzie klucze przechowywane są w pamięci i wskazują adresy wartości
przechowywane na dysku. Może i brzmi to trywialnie, ale podejście to jest stosowane np. bazie Riak, która korzysta ze
storage engine Bitcask. W tym podejściu osiągamy bardzo szybki zapis i odczyt, tyle że klucze muszą mieścić się w
pamięci RAM. Wartości dopisywane są do pliku — jak zatem uniknąć wyczerpania się pamięci? Log dzielimy na segmenty o
określonym maksymalnym rozmiarze. Po jego osiągnięciu segment zamykamy i proces log compaction usuwa duplikaty, a merge
scala mniejsze segmenty. Po tej operacji można przełączyć się na nowy segment, a stare można usunąć.

![Proces compaction i merge](/assets/images/ddia/03-01.png "Proces compaction i merge")

Każdy segment ma własną hash-table, czyli klucz oraz adres do wartości na dysku. Żeby znaleźć wartość, najpierw
przeszukujemy najnowszy segment, potem starsze. Merge utrzymuje małą liczbę segmentów, dlatego nie jest to czasochłonne.

Taki hash index wydaje się prosty, ale w praktyce może wystąpić wiele problemów:

* usuwanie rekordów — stosuje się specjalny rekord tombstone
* format danych — po prostu format binarny?
* konieczność wczytywania do pamięci hashtable z kluczami
* częściowo zapisane rekordy — stosuje się checksumy
* wielowątkowość — zwykle tylko jeden wątek zapisujący

Skoro tyle problemów, dlaczego decydujemy się na append-log, a nie na update pliku? Po prostu random access jest
powolny, a obsługa błędów i awarii jest trudna w implementacji.

#### SSTables and LSM-Trees

Aby zoptymalizować odczyt oraz przestrzeń dyskową, można posortować klucze w logfile. Stąd pomysł na SSTable, czyli
Sorted String Table. Dzięki temu łatwiej jest mergować logi, korzystając z algorytmu mergesort, pozbywając się przy
okazji nadpisanych rekordów.

![Merge kilku segmentów SSTable, zachowując tylko ostatnią wartość klucza](/assets/images/ddia/03-02.png "Merge kilku segmentów SSTable, zachowując tylko ostatnią wartość klucza")

Zamiast przechowywać wszystkie klucze w pamięci, można przechowywać klucz jedynie do pierwszego rekordu z zakresu — tzw.
sparse in-memory index. Skoro i tak trzeba odczytać cały zakres, żeby odnaleźć rekord, to zakres można skompresować.
SSTable pozwala na efektywne range-queries. SSTable wprowadza optymalizację odczytu względem zapisu.

![SSTable z indeksem w pamięci](/assets/images/ddia/03-03.png "SSTable z indeksem w pamięci")

##### Constructing and maintaining SSTables

LSM-Tree (Log Structured Merge Tree) rozwiązuje problem tworzenia i utrzymywania posortowanej struktury na dysku, przy
założeniu, że sortowanie w pamięci jest łatwiejsze niż na dysku. Nowy rekord dodawany jest do zbalansowanej struktury w
pamięci (np. drzewo czerwono-czarne), zachowującej sortowanie przy odczycie. Do tego rekord zachowywany jest w
append-logu, na wypadek awarii, co pozwala odtworzyć drzewo w pamięci.

Po zapełnieniu drzewa (zwykle kilka Mb), drzewo zapisywane jest jako posortowany plik na dysku, a append-log jest
kasowany. Nowe rekordy trafiają do nowego drzewa (tzw. memtable).

Żeby odnaleźć rekord, przeszukiwana jest struktura membtable, a później najnowsze segmenty na dysku. Od czasu do czasu,
segmenty są mergowane i kompaktowane, żeby usunąć nadpisane lub usunięte wpisy.

W przypadku SSTable można stosować pewne optymalizacje.

* Wyszukiwanie nieistniejących kluczy trwa długo — dlatego stosuje się filtry Bloome’a.
* Stosuje się różne strategie mergowania i kompaktowania:
    * sized-tired — nowsze i małe SSTables mergowane w większe i starsze SSTables np. HBase, Cassandra
    * leveled — memtable dzielone na mniejsze SSTable, a starsze dane przenoszone do osobnych “level”, co pozwala na
      mergowanie inkrementalne, zużywające mniej przestrzeni dyskowej np. LevelDB, RocksDB, Cassandra

#### B-Trees

B-Tree jest najpopularniejszym indeksem dla baz danych (zwłaszcza relacyjnych). Dane posortowane są po kluczu, ale nie
są przechowywane w segmentach, lecz w blokach/stronach (page) o stałym rozmiarze, np.: 4kB. Ta struktura odpowiada
strukturze danych na dysku, gdzie również bloki mają stały rozmiar. Każdy blok można zlokalizować po adresie na dysku.
Wyszukiwanie elementów zaczyna się od root, przechodząc do dzieci, poprzez porównanie szukanego klucza, z zakresem
adresów dla segmentu. Liczba referencji dla jednego modułu to branching factor, zwykle równy kilkaset.

![Szukanie klucza korzystając z B-tree](/assets/images/ddia/03-04.png "Szukanie klucza korzystając z B-tree")

Aktualizacja wpisu polega na nadpisaniu bloku. Dodanie podobnie, ale jeśli blok wykracza poza ustalony rozmiar, tworzone
są dwa bloki, a dodatkowo aktualizowane są adresy w bloku rodzica. Ten algorytm sprawia, że drzewo jest zbalansowane.
Głębokość drzewa o _n_ elementach to _Olog(n)_ — zwykle wystarczają trzy lub cztery poziomy. Drzewo o 4kB rozmiarze
strony i branching factor=500, może przechowywać aż 256 TB danych.

![Proces page split w B-tree, uruchamiany kiedy rozmiar strony za bardzo rośnie](/assets/images/ddia/03-05.png "Proces page split w B-tree, uruchamiany kiedy rozmiar strony za bardzo rośnie")

##### Making B-trees reliable

Operacja na B-tree polega na odnalezieniu, odczycie, aktualizacji i zapisie strony na dysku. Czasem kilka stron wymaga
aktualizacji, a wtedy wiele rzeczy może pójść nie tak w przypadku awarii (np. orphan page). Dlatego B-trees posiadają
też Write Ahead Log (WAL) lub inaczej zwany Redo Log, czyli append file, do którego muszą być zapisywane wszystkie
operacje, zanim zostaną zaaplikowane na B-tree. Jest to pomocne przy odtwarzaniu stanu po awarii. Aktualizacja in-place
jest też ryzykowna z powodu wielowątkowości, bo wymaga ochrony przez mechanizm lock/latch.

##### B-tree optimizations

Metody optymalizacji to stosowanie copy-on-write zamiast nadpisywania strony. Nowa strona zapisywana jest w nowej
lokalizacji i aktualizowana jest strona rodzica. Dzięki temu nie potrzeba utrzymywać WAL a dodatkowo ułatwia to dostęp
wielu wątków. Takie podejście stosuje n.p.: LMDB.

Zamiast całych kluczy, można przechowywać tylko ich końcową część — bo początkowa część wyznaczona jest zakresem adresów
dla danej strony.

Można próbować układać dane na dysku obok siebie co wspomaga range queries.

Można dodać dodatkowe wskaźniki n.p.: na rodzeństwo, co pozwala na skanowanie drzewa bez skakania do rodzica i dalej do
rodzeństwa.

#### Comparing B-Trees and LSM-Trees

Zarówno B-tree jak i LSM Tree zapisując rekord, wymagają kilku zapisów na dysku. B-tree zapisuje dane page+WAL+page
split. LSM-tree zapisuje segment oraz segmenty podczas compaction i merging. To tzw. _write amplification_. Jednak
zwykle LSM-tree ma wyższą przepustowość zapisu. Do tego LSM-tree pozwala na kompresję danych na dysku, co pozwala na
zaoszczędzenie miejsca oraz szybszy odczyt i zapis na dysku.

Jednak w LSM Tree, kompaktowanie i mergowanie może spowolnić zapis nowych rekordów do bazy, na czym może ucierpieć
wydajność mierzona wyższymi percentylami czasów odpowiedzi. W drugą stronę, wiele zapisów do bazy, może
spowolnić mergowanie segmentów, co może zapełnić dysk.

Dla B-tree, każdy rekord występuje tylko raz. Właściwość ta pozwala na implementację transakcyjności, tworząc lock na
zakresie kluczy.

#### Other Indexing Structures

Do tej pory mówiliśmy o primary-index, ale jak zaimplementować secondary index, gdzie do jednego klucza można
przypisywać wiele wartości? Okazuje się, że podobnie, przy użyciu primary index, przechowując listę wartości, albo
doklejając do klucza raw-identifier. Nadają się do tego zarówno B-trees oraz log structures.

**Jak przechowywać dane w ramach indeksu?** Można albo przechowywać wartości bezpośrednio (clustered-index), albo tylko
wskaźnik do wartości w heap file (miejsce, gdzie przechowywane są wiersze z danymi na dysku). Clustered index to
duplikacja danych, heap file to wolniejszy odczyt. Bardziej powszechne jest przechowywanie wskaźników do heap file, bo w
przypadku kilku secondary-index, koszt duplikacji wzrasta.

Rozwiązaniem pomiędzy clustered-index a heap file jest covering index, który przechowuje wybrane kolumny, potrafiąc
pokryć (cover) całe zapytanie, korzystając tylko z indeksu.

**Indeksy wielokolumnowe** — to np.: concatenated index, powstały z połączenia dwóch wartości np.: nazwiska i imienia w
książce telefonicznej. Łatwo odnaleźć po nazwisku, nazwisku i imieniu, ale nie da się tylko po imieniu. Inną wersją są
indeksy wielowymiarowe n.p.: geolokalizacyjny z lat+lng albo indeks dla danych pogodowych po dacie i temperaturze.

**Full-text search and fuzzy indexes** — Co kiedy chcemy umożliwić przeszukiwanie po wartości zbliżonej do wartości
klucza, bo w wyszukiwanej frazie pojawiła się literówka, lub użyliśmy synonimu? Odpowiedzią są fuzzy indexes
wykorzystywane w przypadku full text search. Wyszukiwarki pełnotekstowe zwykle umożliwiają wyszukiwanie jednego słowa
rozszerzony o synonimy tego słowa, ignorując odmiany gramatyczne.

Biblioteka Lucene jest w stanie wyszukiwać słowa w tekście w pewnej odległości edycyjnej (odległość edycyjna 1 oznacza,
że jedna litera została dodane, usunięte lub zastąpione). Lucene używa struktury podobnej do SSTable, do tworzenia
słownika.

**Keeping everything in memory** — Bazy przechowujące dane na dysku zapewniają durability, ale wbrew pozorom nie są
wolniejsze, dlatego że muszą czytać z dysku. Systemowy mechanizm stronicowania może sprawić, że dane i tak będą czytane
z pamięci. Przewaga baz in-memory polega na wykorzystaniu innych modeli danych i na braku konieczności enkodowania
danych przed zapisem na dysku.

### Transaction Processing or Analytics

Początkowo bazy danych były wykorzystywane do przechowywania danych biznesowych — czyli transakcji. Z czasem znalazły
inne zastosowania, ale słowo transakcja pozostało w użyciu. Transakcja zwykle obsługuje małą liczbę rekordów,
korzystając z indeksu i jest to odpowiedź na interakcję z użytkownikiem — stąd OLTP (on-line transaction processing). Z
czasem, baz zaczęli używać analitycy i business intelligence. Zamiast dostępu do kilku wierszy, potrzebowali wyliczać
statystyki na wielu wierszach, ale na wybranych kolumnach. Dlatego powstał OLAP (on-line analitycs processing).

SQL sprawdza się zarówno w OLTP jak i OLAP, różnice występują jednak w implementacji silnika zapytań. OLAP w latach 90’
zaczęło wypierać OLTP w analityce — powstały osobne bazy — data warehouse. Dzięki temu rozdzieleniu, bazy OLTP nie były
obciążone zapytaniami analitycznymi.

#### Data Warehousing

Data warehouse to osobna baza read-only, w której dane są zapisywane z bazy OLTP, korzystając z ETL. Bazy OLAP zwykle
nie są potrzebne w małych firmach, ilość danych jest na tyle mała, że bazy OLTP mogą służyć do celów analitycznych.

![ETL pipeline dla data warehouse](/assets/images/ddia/03-06.png "ETL pipeline dla data warehouse")

#### Stars and Snowflakes: Schemas for Analytics

Tabele w data warehouse, mogą być bardzo szerokie — sto lub nawet kilkaset kolumny. Dotyczy to fact-table jak i
dimension-table. Jeśli chodzi o ilość wierszy, fact-table może zawierać o kilka rzędów wielkości więcej wierszy niż
dimension-table n.pl: miliardy vs dziesiątki tysięcy wierszy.

![Przykład star schema w data warehouse](/assets/images/ddia/03-07.png "Przykład star schema w data warehouse")

W przypadku data warehouse najpopularniejsze modele danych to star schema i snowflake schema. W star schema, w centrum
jest fact table, przechowująca zdarzenia, n.p.: zakup produktu, ramionami są dimension tables, przechowujące dane
dodatkowe n.p. co, gdzie, kiedy, dlaczego. Fact table posiada referencję do dimension table. Odmianą star schema jest
snowflake schema, gdzie dimension table może mieć sub-dimension table. Schemat snowflake jest bardziej znormalizowany,
ale schemat star jest preferowany, bo jest prostszy w obsłudze dla analityków.

### Column Oriented Storage

W przypadku danych analitycznych zwykle potrzebnych jest tylko kilka kolumn do wyliczenia wyniku. W row-oriented
storage trzeba wczytać cały wiersz z dysku — stąd pomysł na column-oriented, gdzie każda kolumna przechowywana jest
osobno. Realizując query, wystarczy odczytać kilka wybranych plików. Kolejność rekordów w plikach musi być taka sama,
żeby móc odtworzyć wiersze.

![Przechowywanie  danych relacyjnych w kolumnach](/assets/images/ddia/03-08.png "Przechowywanie  danych relacyjnych w kolumnach")

Column Compression — column-oriented storage korzysta z faktu, że w kolumnie dane mogą się powtarzać i łatwo je
skompresować n.p.: korzystając z bitmap-index.

#### Sort Order in Column Storage

Administrator może wskazać, po jakich kolumnach należy posortować dane — oczywiście w pełni posortowana będzie tylko
pierwsza kolumna, druga już zgodnie z pierwszą itp. Może to przyspieszyć niektóre zapytania, a dodatkowo poprawia
kompresję. Sprytnym rozwiązaniem jest tworzenie różnych sortowań na różnych replikach — n.p.: C-store/Vertica.

#### Writing to Column-Oriented Storage

W column-oriented zapis jest trudny z powodu kompresji, sortowania, przechowywania danych w wielu plikach. Dlatego
można wykorzystać LSM-Tree, które przechowa zapisy w pamięci i przygotuje je do zapisu na dysku.

#### Aggregation: Data Cubes and Materialized Views

Sposobem na przyspieszenie odczytu jest stworzenie materialized view, który może zawierać wyniki popularnych agregacji (
SUM, COUNT, AVG itp.). Inny sposób to OLAP cube przechowujący wyniki agregacji w wielowymiarowe tablicy, pozwalając na
efektywne zapytania z użyciem wybranych kolumn.

### Podsumowanie

Co się dzieje, gdy zapisujemy i odczytujemy dane z bazy danych? Na wysokim poziomie silniki bazodanowe dzielą się na
dwie szerokie kategorie: te zoptymalizowane pod kątem przetwarzania transakcji (OLTP) oraz te zoptymalizowane pod kątem
analityki (OLAP).

* Systemy OLTP zwykle obsługują wiele zapytań, jednak pojedyncze zapytanie dotyka małej liczby rekordów w bazie.
  Aplikacja obsługuje rekordy, korzystając z klucza, a silnik bazodanowy opiera swoje działanie na indeksie obsługującym
  dany klucz. Wąskim gardłem jest przeszukiwanie dysku (_seek time_).
* Systemy OLAP wykorzystywane są np. w data warehouses i innych systemy analitycznych. Są mniej powszechne, ponieważ
  korzystają z nich zespoły analityczne. Obsługują mniej zapytań, jednak pojedyncze zapytanie jest wymagające, agregując
  informację z milionów rekordów. Wąskim gardłem jest tu raczej przepustowość dysku _(disk bandwidth)_.

Systemy OLTP korzystają głównie z dwóch sposobów obsługi danych

* log-structured, gdzie dane są jedynie dopisywane do plików, a stare pliki są kasowane, nigdy nie są aktualizowane.
  Przykładem są algorytmy SSTables i LSM-Trees. Do tej grupy należą Bitcask, LevelDB, Cassandra, HBase, Lucene.
* update-in-place, gdzie dysk jest dzielony na strony o stałym rozmiarze, a strony mogą być nadpisywane. Przykładem jest
  algorytm B-trees, używany zarówno w bazach relacyjnych i nie relacyjnych.

Silniki log-structured są stosunkowo nowym wynalazkiem. Ich kluczową ideą jest to, że zamieniają random-access na
sekwencyjne zapisy na dysku, co umożliwia wyższą przepustowość zapisu dzięki charakterystyce wydajności dysków twardych
i dysków SSD.

Silniki OLAP istotnie różnią się od OLTP. Gdy zapytania wymagają sekwencyjnego skanowania dużej liczby wierszy, indeksy
są znacznie mniej istotne. Zamiast tego ważne staje się bardzo zwięzłe kodowanie danych, aby zminimalizować ilość
danych, które zapytanie musi odczytać z dysku. Pomocny jest w tym column-oriented storage.

Jako programiści aplikacji, jeśli jesteśmy uzbrojeni w wiedzę na temat wewnętrznych mechanizmów bazodanowych, jesteśmy w
znacznie lepszej sytuacji, aby wiedzieć, które narzędzie najlepiej pasuje do konkretnej aplikacji. Jeśli musimy
dostosować parametry bazy danych, zrozumienie tego pozwoli wyobrazić sobie, jaki wpływ może mieć wyższa lub niższa
wartość jakiegoś parametru.
