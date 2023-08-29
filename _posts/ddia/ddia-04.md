## Rozdział #4: Encoding and Evolution

> Everything changes and nothing stands still — Heraclitus of Ephesus, as quoted by Plato in Cratylus (360 BCE)

Tworząc aplikację, warto uwzględnić sytuacje jak np. rolling upgrade albo niespójne wersje klienta i serwera, kiedy
użytkownik nie zaktualizował aplikacji. Może dojść do sytuacji, kiedy:

* nowy kod odczytuje dane zapisane przez stary kod, tj. wymagana jest kompatybilność wsteczna
* stary kod odczytuje dane zapisane przez nowy kod — tj. wymagana jest kompatybilność wprzód

W tym rozdziale zobaczymy, jak osiągnąć kompatybilność przy użyciu formatów danych jak JSON, XML, Avro, Thrift, Protocol
Buffer w systemach opartych o REST, RPC i messaging.

### Formats for Encoding Data

Reprezentacja obiektu w pamięci bardzo często wymaga zapisania na dysku. Konwersja pomiędzy tymi postaciami to
serializacja i deserializacja lub encoding i decoding. Języki programowania dostarczają zwykle biblioteki, które to
potrafią, n.p.: Serializable w Java. Jest to jednak rozwiązanie nieprzenoszalne między językami, niewydajne, ryzykowne,
ponieważ może posiadać luki bezpieczeństwa, niedostosowane do wersjonowania.

#### JSON, XML, and Binary Variants

Najpopularniejsze formaty niezależne od języka programowania to CSV, XML i JSON. Rodzą one kilka problemów, jak
przechowywanie dużych lub zmiennoprzecinkowych liczb, lub stringów w formacie binarnym. Nie posiadają schematu, a
wszystkie nazwy pól przekazywane są z każdym obiektem — stąd ich duży rozmiar. Istnieją formaty binarne np. BSON, BISON,
BJSON, Smile dla JSON, WBXML dla XML — oszczędność rozmiaru danych nie jest aż tak wyraźna. Dla przykładu Message Pack
dla JSON z wiadomości o długości 81 bajtów utworzy wiadomość o rozmiarze 66 bajtów.

#### Thrift and Protocol Buffers

Apache Thrift od Facebook i Protocol Buffers od Google to biblioteki wymagające schematu.

Thrift posiada dwa warianty.

**Thrift Binary Protocol**

Poprzednia wiadomość o rozmiarze 81 bajtów zajmuje 59 bajtów.

![Thrift Binary Protocol encoding](/assets/images/ddia/04-01.png "Thrift Binary Protocol encoding")

**Thrift Compact Protocol**

Dzięki typowi oraz tag number zapisanym w pojedynczym bajcie, oraz dzięki użyciu zmiennej długości pola na wartości typu
integer poprzednia wiadomość zajmuje 34 bajty

![Thrift Compact Protocol encoding](/assets/images/ddia/04-02.png "Thrift Compact Protocol encoding")

**Protocol Buffers**

Format bardzo zbliżony do Thrift Compact Protocol, drobne różnice to n.p. brak znaku końca. Poprzednia wiadomość zajmuje
33 bajty

![Protocol Buffers encoding](/assets/images/ddia/04-03.png "Protocol Buffers encoding")

Dzięki unikalnym tagom można modyfikować schemat. Można dodać pole opcjonalne lub wymagane z wartością domyślną. Można
usunąć pole, ale tylko jeśli nie jest wymagane. Można zmieniać nazwy pól, ale nie można zmieniać tagów. W wąskim
zakresie można zmieniać typy pól. Protocol Buffers pozwala dzięki słowu kluczowemu _repeat_ zamienić pojedynczą wartość
na listę. Thrift nie daje takiej możliwości, ale za to pozwala na zagnieżdżone listy.

#### Avro

Avro to kolejna binarna reprezentacja danych, ale w odróżnieniu od poprzednich nie zawiera tagów z numerami pól. Dane
nie
zawierają informacji o typie, jest to tylko zlepek pól.

Avro pozwala na określenie schematu poprzez Avro IDL — jest to postać czytelna dla człowieka

![Avro IDL](/assets/images/ddia/04-04.png "Avro IDL]")

Avro posiada też reprezentację JSON czytelną dla maszyny

![Avro JSON schema representation](/assets/images/ddia/04-05.png "Avro JSON schema representation]")

Awro definiuje pole opcjonalne jako union — nie ma wartości opcjonalnych. Poprzednie dane w afro zajmuje on 32 bajty.

![Avro encoding](/assets/images/ddia/04-06.png "Avro encoding]")

##### The writer’s schema and the reader’s schema

Avro umożliwia zmiany w schemacie, definiując reader schema oraz writer schema. Schematy nie muszą być identyczne, ale
muszą być kompatybilne. Przy odczycie wiadomości, potrzebne są zarówno writer schema, jak i reader schema. Jeśli pole
jest w definicji writer schema, ale nie ma w reader schema, to jest ignorowane przy odczycie. Jeśli jest reader schema,
ale nie w definicji writer schema, to musi być podana wartość domyślna.

Avro odczytuje dane przy pomocy writer schema i konwertuje zgodnie z reader schema. Kolejność pól w schematach może być
różna, ponieważ pola dopasowywane są po nazwie.

![Avro schema evolution](/assets/images/ddia/04-07.png "Avro schema evolution]")

##### Schema evolution rules

W Avro kompatybilność wprzód oznacza, że writer ma nowy schemat reader stary. Kompatybilność wstecz oznacza, że writer
ma stary schemat a reader nowy. Aby utrzymać kompatybilność, można dodać lub usunąć pole, jeśli ma wartość domyślną.
Zmiana typu jest możliwa w pewnych przypadkach. Zmiana nazwy jest możliwa, jeśli do reader schema dodamy alias — jest to
jednak tylko kompatybilność wstecz, ale nie w przód.

##### But what is the writer’s schema?

Avro zostało opracowane z myślą o hadoop — podstawowe zastosowanie to przechowywanie dużej ilości rekordów w pliku.
Wtedy reader odczytuje schemat z początku pliku — jest to tak zwany Object Container File. W bazie danych każdy rekord
może być zapisany przy pomocy innego schematu – dlatego każdy rekord ma ID schematu, do której reader ma dostęp. Przy
połączeniu przez sieć, schemat może być ustalony przy nawiązywaniu połączenia.

Schemat Avro może być wygenerowany dynamicznie, na podstawie danych — dzięki temu, że nie posiada field ID, tylko same
nazwy pól — było to jedno z założeń Avro

#### The Merits of Schemas

Formaty tekstowe jak XML, JSON, CSV ze względu na swoją prostotę są popularne i chętnie stosowane do zapisu schematu
danych. Jednak enkodowanie binarne ze schematem w Avro czy Protocol Buffers ma kilka przewag:

* dane zajmują mniej miejsca, bo nie trzeba przesyłać kluczy
* schemat jest sam w sobie dokumentacją, która jest utrzymywana z kodem
* schemat narzuca programiście dbałość o kompatybilność w przód i wstecz

### Modes of Dataflow

Można rozróżnić trzy metody wymiany danych

**Poprzez bazę danych** – jeden proces zapisuje dane, a drugi te dane odczytuje. Pojawia się tu jednak problem z
ewolucją
schematu. Jeśli pierwszy proces doda pole, zapisze rekord w bazie, a drugi proces nieświadomy zmian, odczyta ten sam
rekord i zapisze dane w bazie ponownie – zmiana dodana przez pierwszy proces zostanie usunięta.

Ciekawostka – zapis i odczyt w ramach jednego procesu można traktować jako komunikacja z przyszłym sobą.

**Poprzez sieć** — Rest i RPC pozwalają na komunikację sieciową między serwisami. Można zaimplementować dzięki tym
podejściom klasyczny model klient serwer albo SOA, czyli Service Oriented Architecture, na przykład mikroserwisy. Cel
mikroserwisów to rozbicie aplikacji na komponenty, które mogą być rozwijane i releasowane niezależnie. Kiedy jako sposób
komunikacji używany jest protokół HTTP, mówimy o web serwisach. Web serwis można zaimplementować, korzystając na
przykład z Rest czy SOAP. REST nie jest protokołem, to raczej filozofia lub umowa określająca sposób komunikacji. SOAP
to standard oparty o XML, a dokładniej WSDL, czyli Web Service Description Language. WSDL pozwala na generowanie kodu,
jednak ze względu na swoją złożoność, skomplikowane i trudne w utrzymaniu wiadomości, nie jest tak popularny, jak REST.

RPC pozwala na wywołanie zapytania na serwisie sieciowym tak, jakby było to wywołanie funkcji lokalnej. Jednak RPC ma
fundamentalne problemy, bo request sieciowy jest zupełnie inny niż wywołanie funkcji lokalnej:

* zapytania sieciowe może zakończyć się sukcesem, błędem, ale też serwis może być niedostępny albo po prostu bardzo
  wolny
* timeout nie oznacza ani błędu, ani poprawnego wykonania – w zasadzie to nie wiadomo
* lokalnie można korzystać z referencji. Żeby przesłać dane przez sieć, trzeba je enkodować i dekodować – jeśli serwisy
  zaimplementowano w różnych językach, należy tłumaczyć typy danych

Mimo tego, że REST jest łatwy w użyciu, wspierany przez większość języków, RPC może być bardziej wydajny, a jego wady
można omijać.

Protokół gRPC korzysta z Protocol Buffers do enkodowania i dekodowania. Metody RPC w Finagle czy Rest.li zwracają
futures i promises, żeby było jasne, że metoda nie wykona się od razu.

Do tego Thrift, gRPC lub AvroRPC pozwalają na ewolucję danych w takim zakresie jak użyty pod spodem protokół to jest
Avro, lub Protocol Buffers.

Rest nie pozwala na ewolucję, ale można korzystać z wersjonowania API.

**Poprzez message brokera** — Message Broker jak Kafka czy RabbitMQ pozwalają na komunikację z niskim opóźnieniem, ale
poprzez serwis pośredni. Serwis pośredni może służyć jako bufor, jeśli odbiorca nie jest to dostępny. Nie ma też
potrzeby, żeby nadawca znał adresy IP wszystkich odbiorców, co jest przydatne, kiedy wiele serwisów działa w chmurze –
są kasowane i dodawane dynamicznie. W tym podejściu nadawca i odbiorca są logicznie rozdzieleni, dowolny odbiorca może
konsumować wiadomości zapisane w brokerze.

### Podsumowanie

Rozdział ten poświęcony był sposobem enkodowania i dekodowania danych przesyłanych przez sieć lub zapisywane na dysku.
Szczegóły tych kodowań wpływają nie tylko ich wydajność, ale przede wszystkim na architekturę aplikacji.

W szczególności jest to istotne, kiedy w aplikacji mamy klienty i serwery aktualizowane niezależnie, bądź chcemy
przeprowadzić rolling upgrade. Rolling upgrade umożliwia wdrażanie nowych wersji aplikacji bez przestojów — zachęcając w
ten sposób do częstych małych releasów zamiast rzadkich i dużych. Dzięki temu wdrożenia są mniej ryzykowne, umożliwiając
wykrywanie i wycofywanie wadliwych zmian, zanim wpłyną one na dużą liczbę użytkowników.

W trakcie rolling upgrade musimy założyć, że różne węzły serwują różne wersje aplikacji. Dlatego ważne jest, aby
wszystkie dane przepływające przez system były zakodowane w sposób zapewniający kompatybilność wsteczną (nowy kod może
odczytywać stare dane) i kompatybilność wprzód (stary kod może odczytywać nowe dane).

Nie wszystkie popularne formaty danych zapewniają kompatybilność wstecz i w przód:

* Kodowanie specyficzne dla języka programowania są ograniczone do jednego języka programowania i często nie zapewniają
  kompatybilności do przodu i wstecz.
* Formaty tekstowe, takie jak JSON, XML i CSV, są szeroko rozpowszechnione, a ich kompatybilność zależy od sposobu ich
  użycia. Formaty te są nieco niejasne pod względem typów danych, więc trzeba uważać na takie rzeczy jak liczby i
  _binary strings_.
* Formaty binarne oparte na schematach, takie jak Thrift, Protocol Buffers i Avro, umożliwiają kompaktowe, wydajne
  kodowanie z jasno zdefiniowaną semantyką kompatybilności w przód i w tył. Schematy mogą być przydatne do tworzenia
  dokumentacji i generowania kodu w statycznie typowanych językach. Mają jednak tę wadę, że dane muszą zostać
  zdekodowane, zanim będą czytelne dla człowieka.

Zakodowana dane są następnie przesyłane bądź zapisywane na dysku. Można wyróżnić kilka trybów przepływu danych:

* Bazy danych, w których proces zapisujący dane do bazy danych koduje je, a proces odczytujący dane z bazy danych je
  dekoduje.
* RPC i REST API, gdzie klient koduje request, serwer go dekoduje — podobnie w drugą stronę
* Asynchroniczne przekazywanie wiadomości (przy użyciu brokerów wiadomości lub aktorów), gdzie węzły komunikują się,
  wysyłając wiadomości przez pośrednika. Wiadomości są kodowane przez nadawcę i dekodowane przez odbiorcę.
