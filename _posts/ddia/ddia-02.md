## Rozdział #2: Data Models and Query Languages

> The limits of my language mean the limits of my world — Ludwig Wittgenstein, Tractatus Logico-Philosophicus (1922)

Model pozwala na spójną reprezentację danych, ukrywając ich złożoność. Rzeczywista domena ukrywana jest za API,
reprezentowana jest przez encje. Encje zapisywane są w bazie danych, korzystającej z systemu plików na komputerze.
Komputer operuje na bitach. Każda z tych warstw, tworzy pewną abstrakcję.

### Relational Model Versus Document Model

Model danych SQL oparty o relacje zaproponował w 1970 r. Edgar Codd. Wykorzystywany był do reprezentacji transakcji
biznesowych jak płatności, rezerwacje, sprzedaż. Przetrwał jednak próbę czasu i dominował przez 30 lat.

#### The Birth of NoSQL

NoSQL, czyli Not Only SQL, próbuje rozwiązać problemy, jak wysoka skalowalność przy intensywnym zapisie i odczycie,
obsługa zaawansowanych zapytań, zbyt restrykcyjny schemat danych oraz komercjalizacja narzędzi.

#### The Object-Relational Mismatch

Obecnie Object Oriented Programming to najpopularniejszy paradygmat programowania. Obiekty przechowywane w bazie
relacyjnej, wymagają skomplikowanej warstwy translacyjnej, jaką jest ORM. Ta różnica w reprezentacji to **impedance
mismatch**. Poniżej, prosta struktura dokumentu, reprezentowana jest przez bardziej złożone relacje pomiędzy tabelami.

![Struktura dokumentu reprezentowana przez relacje](/assets/images/ddia/bill.jpg "Struktura dokumentu reprezentowana przez relacje")

Reprezentacja danych w postaci JSON redukuje impedance mismatch, ponieważ lepiej odwzorowuje strukturę obiektu. Zarówno
obiekt, jak i JSON mają drzewiastą strukturę, a nie relacyjną. Do tego ma lepszą lokalność — jedno zapytanie zwraca cały
obiekt, bez join.
JSON jest krytykowany za brak schematu danych i faktycznie czasami schemat jest konieczny, ale czasami może być plusem —
np. kiedy dane są heterogeniczne.

### Many-to-One and Many-to-Many Relationships

Można przechowywać informacje o kraju użytkownika jako nazwę kraju lub jako kod ISO kraju. Zaletą używania drugiego
podejścia (słownikowego) jest spójność pisowni/stylu dla wszystkich użytkowników, łatwość aktualizacji, i13n — łatwość w
tłumaczeniu oraz łatwe przeszukiwanie i dodawanie kontekstu — na przykład zapytanie: znajdź wszystkich użytkowników z
Europy, jest możliwe, bo możemy dodać kontekst: Polska leży w Europie.

Dane są znormalizowane, kiedy informacja rozumiana przez człowieka i niosąca jakieś znaczenie, nie jest powielana.
Zamiast duplikacji, tworzy się relacje poprzez ID, mające znaczenie dla bazy. Żeby stworzyć encje, należy użyć relacji
one-to-many i many-to-many, korzystając z join — który ma dobre wsparcie w bazach relacyjnych a słabe w bazach
dokumentowych, gdzie emuluje to aplikacja.

### Are Document Databases Repeating History?

Dyskusja na temat reprezentacji danych nie została zapoczątkowana przez NoSQL. Baza IMS od IBM (1968 r) korzystała ze
struktury hierarchicznej — zbliżonej do JSON — reprezentując dane, jako struktura drzewiasta. Problemem były relacje
many-to-many, gdzie tak jak współcześnie, programiści stawali przed wyborem — denormalizacja (czyli duplikacja) albo
rozwiązywanie powiązań w kodzie. Żeby rozwiązać te dylematy, zaproponowano relational model i network model.

**Model sieciowy** zakładał, że węzeł może mieć wielu rodziców. Był to wydajny model, ale przy złożonych strukturach,
komplikował kod aplikacji — aplikacja musiał pilnować, żeby ścieżka, w jakiej zapytanie przechodzi po bazie, nie
zapętlała się. Aplikacja musiał być świadoma struktury danych i powiązań w bazie, co komplikowało kod.

**Model relacyjny** nie wprowadza zagnieżdżonych struktur, dane przechowywane są w tabelach, a funkcję agregowania
danych spełnia query-optimizer — skomplikowane rozwiązanie, ale zdejmuje ten obowiązek z programistów korzystających z
bazy. Bazy dokumentowe, podobnie jak IMS zachowują strukturę, ale relacje many-to-many działają podobnie jak w bazie
relacyjnej — tam _foreign-key_, tu _document reference_, tam referencje rozwiązywane przez _join_, tu dodatkowe
zapytanie i łączenie danych w kodzie aplikacji.

### Relational Versus Document Databases Today

Biorąc pod uwagę bazy dokumentowe i relacyjne, należy wziąć pod uwagę wiele czynników

Baza dokumentowa to schema flexibility, wydajność ze względu na lokalność danych, odwzorowanie struktur domenowych. Baza
relacyjna to wsparcie dla join i relacji many-to-many.

#### Which data model leads to simpler application code?

Pod kątem prostoty kodu, wybór pomiędzy tymi modelami, zależny jest od domeny aplikacji i relacji między danymi. To czy
wybrać bazy dokumentowe, relacyjne, a może grafowe, zależy od ilości połączeń między danymi oraz od tego, jak
dynamicznie
zmieniają się te połączenia, jak i same dane.

#### Schema flexibility in the document model

Wbrew powszechnej opinii, bazy dokumentowe nie są schemaless. Schemat wymuszany jest przez struktury w kodzie, stąd
lepiej mówić o _schema-on-read_ dla baz dokumentowych i _schema-on-write_ dla baz relacyjnych.

Schema-on-read działa jak runtime check. Zmianę struktury danych można obsłużyć w kodzie. Jest to lepsze rozwiązanie dla
danych heterogenicznych. Heterogeniczność danych to niejednorodność, wynikająca z różnych typów dla tych samych danych
lub dane pochodzą z zewnętrznego systemu i nie panujemy nad tym co dostajemy.

Schema-on-write to wyższa spójność danych, ale zmiana struktury wymaga migracji.

#### Data locality for queries

Lokalność zapytania osiągamy, kiedy jednym zapytaniem możemy pobrać lub zaktualizować interesujące nas dane, bez dostępu
do innych tabel lub kolekcji.

Dokumenty przechowywane są jako string: enkodowany JSON, XML lub BSON dla Mongo. Lokalność danych daje wartość, kiedy
zwykle czytany jest cały dokument. Odczyt tylko część i tak wymaga załadowania całości. Przy zapisie, cały dokument jest
aktualizowany, co również jest nieefektywne. Bazy dokumentowe są zatem efektywne dla małych dokumentów, bez zapisów
gdzie następuje przyrost danych — bo całkiem możliwe, że dokument będzie musiał być zapisany w innym obszarze pamięci.

Nie tylko bazy dokumentowe wspierają lokalność danych — Google Spanner wspiera lokalność danych poprzez zagnieżdżenia
danych w wierszach.

#### Convergence of document and relational databases

Można zauważyć, że występuje konwergencja pomiędzy bazami relacyjnymi i dokumentami. Bazy relacyjne wspierają dokumenty
w
JSON bez schematu, np.: PostgreSQL, MySQL. Bazy dokumentowe wspierają relację i joiny np.: Rethink DB. Poszerza to
możliwości i zakres zastosowań, co jest dobrą wiadomością dla użytkowników.

### Query Languages for Data

Relacyjne bazy danych pozwalają na deklaratywny sposób dostępu do danych. Ma on tę zaletę nad dostępem imperatywnym, że
bez zmian w kodzie, query optimizer może wprowadzać usprawnienia np.: zrównoleglenie. Dostęp imperatywny to lista
instrukcji do wykonania, dostęp deklaratywny to wzorzec dostępu z określonymi warunkami — prosty i ukrywa szczegóły
implementacji.

Abstrahując od danych, przykładem deklaratywnego języka jest CSS.

#### MapReduce Querying

Map reduce jest modelem dostępu do danych pomiędzy deklaratywnym a imperatywnym. Query jest definiowane przez fragmenty
kodu, a te są wykonywane przez framework. Map-reduce wykorzystywany jest do przetwarzania dużych ilości danych w
porcjach i na wielu maszynach równocześnie. Został adaptowany do NoSQL jak MongoDB i CouchDB.

Funkcje dla map-reduce to pure functions — bez stanu, bez zapytań do bazy, ale wciąż mogą być nieoptymalnie napisane ze
względu na swój imperatywny charakter. Deklaratywny dostęp do bazy może być usprawniony przez query-optimizer. Dlatego
np. MongoDB oferuje aggregation pipeline, gdzie składnia podobna jest do SQL, tyle że w JSON. NoSQL odkrywa SQL na nowo.

#### Graph-Like Data Models

Obecność relacji many-to-many w aplikacji sugeruje wybór bazy relacyjnej, a nie dokumentowej. Jeśli liczba relacji jest
spora, lepszym rozwiązaniem jest baza grafowa.

Baza grafowa jest świetnym wyborem, jeśli chodzi o algorytmy wyznaczające trasę, Page Rank. Dane w grafie mogą być
homogeniczne — graf składa się z wierzchołków dowolnego typu i krawędzi określających dowolne relacje.

![Przykład struktury grafowej](/assets/images/ddia/graph-structure.jpg "Przykład struktury grafowej")

Istnieje kilka modeli strukturyzowania i tworzenia zapytań w grafach. Dwa z nich to:

* property graph — implementowany w Neo4J, Titan, InfiniteGraph
* triple-store — Datomic, AllegroGraph.

Deklaratywne języki zapytań dla grafów to np.: Cypher, SPARQL i Datalog.

#### Property Graphs

W modelu tym wierzchołek składa się z:

* ID
* krawędzi wychodzących (head)
* krawędzi wchodzących (tail)
* danych w postaci klucz-wartość

Krawędzie zaś zawierają

* ID
* wierzchołek początkowy (tail vertex)
* wierzchołek końcowy (head vertex)
* label — opisuje rodzaj relacji pomiędzy dwoma wierzchołkami
* dane w postaci klucz-wartość

Baza taka nie posiada schematu. Dowolny wierzchołek łączy się z dowolnym innym wierzchołkiem. Mając wierzchołek, łatwo
przechodzić w przód i w tył. Dane rozróżniane przez labelki określające relacje, mogą być homogeniczne.

#### The Cypher Query Language

Cypher to język deklaratywny dla Neo4j.

Notacja jak niżej pozwala na utworzenie dwóch wierzchołków połączonych relacją WITHIN

**(Idaho)-[:WITHIN]-USA**

Bardziej złożony przykład, z większą liczbą relacji:

```
CREATE_
(NAmerica:Location {name:'North America', type:'continent'}),
(USA:Location {name:'United States', type:'country' }),
(Idaho:Location {name:'Idaho', type:'state'}),
(Lucy:Person {name:'Lucy' }),_
(Idaho)-[:WITHIN]-> (USA)-[:WITHIN]-> (NAmerica),
(Lucy) -[:BORN_IN]-> (Idaho)
```

Żeby znaleźć wszystkie osoby, które wyemigrowały z US do Europy, można utworzyć zapytanie:

```
MATCH
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
(person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```

Notacja **-[:WITHIN*0..]->** oznacza: podążaj za krawędzią WITHIN zero lub więcej razy.

Notacja **(person) -[:BORN_IN]-> ()** oznacza — lewy node to type **person**, prawy **()** node jest nienazwany/dowolny

#### Graph Queries in SQL

Co ciekawe, w SQL można pisać zapytania rekurencyjne, odpowiadające przechodzeniu po grafie, ale poziom skomplikowania
zapytania sugeruje, że nie jest to typowe zastosowanie.

#### Triple-Stores and SPARQL

Triple-Store jest modelem podobnym do Property Graph, tyle że używa innych pojęć do opisywania tych samych idei.
Struktura triple store to: **(subject, predicate, object)**. Object to może być primitive data type, n.p. 
(lucy, age, 33) lub inny wierzchołek w grafie, n.p.: (lucy, marriedTo, alan).

Triple-Store posiada też format _Turtle_, który jest czytelny dla człowieka. Odwzorowanie struktury stworzonej w Cypher
przy pomocy _Turtle_ wygląda następująco:

```
@prefix : &lt;urn:example:>.
_:lucy a :Person.
_:lucy :name "Lucy".
_:lucy :bornIn _:idaho.
_:idaho a :Location.
_:idaho :name "Idaho".
_:idaho :type "state".
_:idaho :within _:usa.
_:usa a :Location.
_:usa :name "United States".
_:usa :type "country".
_:usa :within _:namerica.
_:namerica a :Location.
_:namerica :name North America".
_:namerica :type "continent".
```

Co można zapisać w skróconej wersji, korzystając ze średników:

```
@prefix : &lt;urn:example:>.
_:lucy a :Person; :name "Lucy"; :bornIn _:idaho.
_:idaho a :Location; :name "Idaho"; :type "state"; :within _:usa.
_:usa a :Location; :name "United States"; :type "country"; :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```

##### The SPARQL query language

SPARQL to język deklaratywny dla baz Triple-Store w formacie RDF. Cypher wzoruje się na SPARQL, stąd podobieństwa między
tymi językami.

Side note: RDF to Resource Description Format, który powstał z myślą o semantic web. Może jednak być wykorzystywany w
dowolnym celu w bazach grafowych. RDF był popularny w latach 2000 i miał być mechanizmem standaryzującym publikowanie
danych przez strony internetowe — co pozwoliłoby na stworzenie _web of data_ — rodzaju bazy danych, rozpropagowanej po
całym internecie, która może być czytana przez maszyny.

#### The Foundation: Datalog

Datalog ma początki w latach 80’ i dał podwaliny pod SPARQL i Cypher. Model podobny do triple-store, tyle że ma
składnię: **predicate (subject, object)**. Datalog jest podzbiorem prologa. Wymaga innego myślenia o tworzeniu zapytań.
Dla prostych zapytań jest skomplikowany, ale ma spore możliwości budowania zapytań skomplikowanych.

### Podsumowanie

Historycznie, dane zaczęły być reprezentowane jako jedno duże drzewo (model hierarchiczny), ale podejście to nie
nadawało się to do reprezentowania relacji many-to-many. Aby rozwiązać ten problem, stworzono model relacyjny. Niektóre
aplikacje nie pasują jednak do modelu relacyjnego — dlatego powstał model nierelacyjny, czyli NoSQL, który ewoluował w
dwóch głównych kierunkach:

* Dokumentowe bazy danych, w których dane są dostarczane w postaci samodzielnych dokumentów, a relacje między jednym
  dokumentem a drugim są rzadkie.
* Grafowe bazy danych, w których wszystko jest potencjalnie powiązany ze wszystkim.

Wszystkie trzy modele (dokumentowy, relacyjny i grafowy) są dziś szeroko stosowane i każdy z nich jest dobre w swojej
dziedzinie. Jeden model można naśladować niektóre aspekty innego modelu — na przykład dane grafowe mogą być
reprezentowane w relacyjnej bazie danych — ale implementacja ta jest uciążliwa. Dlatego mamy różne systemy do różnych
celów, a nie a jedno uniwersalne rozwiązanie.

Jedną wspólną cechą baz danych dokumentowych i grafowych jest to, że zazwyczaj nie narzucaj schematu dla przechowywanych
danych, co może ułatwić dostosowanie aplikacji do zmieniających się wymagań. Jednak aplikacja najprawdopodobniej nadal
zakłada, że dane mają określoną strukturę; to tylko kwestia tego, czy schemat jest jawny (wymuszone przy zapisie —
schema on write) lub niejawny (obsługiwane przy odczycie - schema on read).

Każdy model danych ma swój własny język zapytań lub framework, n.p.: SQL, MapReduce, MongoDB aggregation pipeline,
Cypher, SPARQL.

Istnieją też inne modele reprezentacji danych, n.p.: full-text search czy bazy genowe przetwarzające ciągi znaków
ogromnej długości.
