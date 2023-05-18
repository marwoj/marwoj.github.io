## Rozdział #1: Reliable, Scalable and Maintainable Applications

>The Internet was done so well that most people think of it as a natural resource like the Pacific Ocean, rather than
something that was man-made. When was the last time a technology with a scale like that was so error-free?—Alan Kay, in
interview with Dr Dobb’s Journal (2012)

Wiele dzisiejszych aplikacji wymaga intensywnego przetwarzania danych, w przeciwieństwie do aplikacji obliczeniowych.

Aplikacje DDIA są zwykle zbudowane z klocków:

* Przechowują dane, aby można je było później znaleźć (bazy danych)
* Zapamiętują wyniki kosztownej operacji w celu przyspieszenia odczytu (cache)
* Pozwalają na szukanie i filtrowanie według słów kluczowych (indeksy wyszukiwania)
* Przesyłają wiadomości, żeby inne aplikacje mogły obsłużyć je asynchronicznie (przetwarzanie strumieniowe, message
  queue)
* Przetwarzają porcje danych (przetwarzanie batchowe)

Brzmi to, jak oczywistość, a to dlatego, że systemy danych są tak udaną abstrakcją — nie piszemy ich od zera, tylko
wykorzystujemy istniejące narzędzia.

Standardowe wymagania aplikacji można opisać właśnie na wysokim poziomie abstrakcji. Pod nimi ukryta jest jednak
złożoność: szczegóły implementacji, charakterystyka działania, wymagania dot. szybkości i niezawodności.

Coraz to większe wymagania nowoczesnych aplikacji wymagają lepszych narzędzi. Abstrakcje opisujące te narzędzia jednak
się przeplatają. Redis to cache i message queue. Kafka to message queue i baza danych.

Często wiele narzędzi musi współpracować i łączone są one kodem aplikacji. Łącząc narzędzia z klocków, tworzymy nowe
narzędzia. W ten sposób stajemy się projektantami systemu danych, a do tego potrzebna jest wiedza z zakresu DDIA.

Reliability, Scalability and Maintainability to kluczowe wymagania, jakie stawia się przed nowoczesnymi i zaawansowanymi
systemami danych. Nie oznacza to jednak, że projektując taki system, nie należy wziąć pod uwagę innych czynników jak
umiejętności i doświadczenie zespołu, regulacje prawne, czas na implementację itp.

Książka skupia się jednak na:

* Reliability: zapewnienie poprawnego działania w przypadku awarii
* Scalability: kiedy system rośnie, aplikacja wciąż powinna sobie radzić
* Maintainability: wiele osób może współpracować, dostosowując system do nowych wymagań

### Reliability

Niezawodność intuicyjnie można opisać jako: Aplikacja nadal działa, nawet jeśli coś idzie źle. Taki system nazywamy
fault-tolerant. Bo należy tu rozróżnić:

* Fault — nie działa jakiś komponent systemu. Albo zostaje on zastąpiony, albo użytkownicy napotykają błędy w aplikacji.
* Failure — nie działa system cały system, użytkownicy nie mogą z niego korzystać

Książka skupia się na błędach których ryzyko można zminimalizować.

#### Hardware Faults

Kiedy mówimy o awarii, to często pierwsza myśl to awaria sprzętowa. Można zapewnić redundancję sprzętu: dyski RAID,
awaryjne zasilanie, hot-swap CPU. Jest to jednak kosztowne rozwiązanie a często niewystarczające. Zamiast tego, można
skorzystać z wielu maszyn, zapewniając redundancję poprzez software, który replikuje dane pomiędzy maszynami. Jako bonus,
można przy takim podejściu wykonać rolling upgrade.

Błąd oprogramowania w kluczowym elemencie infrastruktury łatwo może się jednak przerodzić w failure systemu. Dlatego
dodatkowo potrzebujemy testów, monitorowania i izolacji procesów.

#### Human Errors

Systemy tworzą ludzie. Główną przyczyną błędów są błędy w konfiguracji. Projektując system, należy to robić tak, żeby
ułatwiać poprawne z niego korzystanie. Należy zapewnić sandbox do testów, testy automatyczne, procedury manualne i
monitoring.

#### Jak istotna jest niezawodność?

Niezawodność nie dotyczy tylko elektrowni. Mamy odpowiedzialność, żeby nie zawieść zaufania użytkowników. Jeśli chcemy
poświęcić niezawodność, to musimy to zrobić świadomie, tj. robimy to, bo tworzymy prototyp albo mamy ograniczony budżet.

### Scalability

Skalowalność to zdolność systemu do poradzenia sobie ze zwiększonym obciążeniem. Nie jest to cecha całej aplikacji, ale
jej komponentów.

#### Określanie obciążenia

Miary obciążenia są zależne od komponentu. Dla serwera to może być ilość zapytań na sekundę, ilość otwartych
websocketów. Dla bazy danych, rozmiar danych zapisywanych na dysku, liczba operacji odczytu i zapisu na sekundę itp.
Czasem miara obciążenia będzie wynikała ze specyficznych przypadków — tu przykład timeline na Twitter, gdzie przeciętny
użytkownika jest obserwowany przez niewielką liczbę osób, jednak są osoby, które są obserwowane przez miliony osób.
Obydwie sytuacje wymagają innego sposobu dostępu do danych w przypadku notyfikacji i należy wybrać odpowiednie miary
obciążenia.

#### Określanie wydajności

Wiedząc, jak mierzyć obciążenie, możemy zmierzyć wydajność. Wydajność zawsze jest pewnym kompromisem, który możemy
określić, odpowiadając na pytania:

* kiedy wzrośnie obciążenie, o ile spadnie wydajność?
* kiedy wzrośnie obciążenie, o ile należy zwiększyć zasoby, żeby zachować wydajność?

Response time z powodów losowych jak context switching, garbage collector pause, zgubiony pakiet TCP, zajęte połączenie
do bazy, odczyt z dysku, a nie pamięci podręcznej, to nie pojedyncza liczba, a dystrybucja wartości. Średnia gubi jednak
informacje, ilu użytkowników otrzymało wolną odpowiedź. Lepszą metryką jest percentyl. Mediana, czyli p50 określa, że
połowa użytkowników otrzymało odpowiedź poniżej pewnego czasu, a połowa powyżej tego czasu. Wyższe percentyle jak p99
mówią, jak wolne są najwolniejsze zapytania.

Percentyle są używane do definiowania SLA, ale wyższe SLA to wyższe koszty. Granica musi być zdroworozsądkowa, bo
p999=1s oznacza, że tylko jedno zapytanie na 1000 może być wolniejsze.

#### Radzenie sobie z obciążeniem

Architektura skalowalności, w której chcemy obsłużyć 100k zapytań, gdzie każde zajmuje ok. 1 kB, będzie inna niż kilku
zapytań po 2GB/min.

Skalowanie ręczne jest prostsze, ale skalowanie automatyczne obsłuży nieprzewidziane sytuacje.

Skalowanie horyzontalne daje większe możliwości, ale jest trudniejsze niż wertykalne. Zwykle stosuje się mix tych
rozwiązań.

### Maintainability

Utrzymywalność systemu to naprawa błędów, poprawa wydajności, zmiany wymagań, nowe funkcjonalności, spłata długu
technicznego. Są to rzeczy może i nieprzyjemne, ale konieczne. Można jednak zminimalizować próg wejścia i niechęć osób,
które się tym zajmują, projektując system z zasadami: Operability, Simplicity, Evolvability

#### Operability: Making Life easy for Operations

“Good operations can often work around the limitations of bad (or incomplete) software, but good software cannot run
reliably with bad operations”

System to nie tylko kod. Warto zadbać o monitoring, detekcję błędów, aktualizację narzędzi, bezpieczeństwo,
współdzielenie wiedzy, dobre praktyki, przewidywanie problemów. Warto tworzyć system z myślą o tych obszarach i warto
automatyzować jak najwięcej.

#### Simplicity: Managing Complexity

Tworzenie prostego systemu nie musi oznaczać redukcji jego funkcjonalności. Można to osiągnąć poprzez usuwanie
przypadkowej złożoności.

Złożone systemy to zbytnio powiązane moduły, zależności, niespójne nazewnictwo, hacki i workaroundy. System może być
złożony, bo domena jest złożona. Gorsza jest złożoność przypadkowa, która nie wynika ze skomplikowania domeny, ale z
przypadkowości.

Złożone systemy wydłużają development, narażają na wyższe koszty i zwiększają ryzyko błędów. Pozbywanie się złożoności
procentuje. Jednym ze sposobów jest tworzenie dobrych abstrakcji, np.: SQL czy języki programowania.

#### Evolvability: Making Change Easy

Zmieniają się wymagania, priorytety. Zdobyta wiedza daje nowe możliwości. System musi być łatwo w rozwoju.

### Podsumowanie

Aplikacja, aby była użyteczna, musi spełniać różne wymagania. Istnieją wymagania funkcjonalne (co powinna robić, np.
pozwalać na przechowywanie, pobieranie, przeszukiwanie i przetwarzanie danych na różne sposoby) oraz pewne wymagania
niefunkcjonalne (ogólne właściwości, takie jak bezpieczeństwo, niezawodność, zgodność, skalowalność, kompatybilność i
łatwość utrzymania).

Mówiąc o wymaganiach aplikacji, myślimy o wymaganiach funkcjonalnych. Developer dba też o te niefunkcjonalne. Zespół
musi wypracować sensowny kompromis pomiędzy wymaganiami funkcjonalnymi a niefunkcjonalnymi.

**Niezawodność** oznacza, że systemy działają poprawnie, nawet jeśli wystąpią błędy. Usterki mogą dotyczyć sprzętu (
zazwyczaj losowe i nieskorelowane), oprogramowania (błędy są zazwyczaj systematyczne i trudne do usunięcia) oraz ludzi (
którzy nieuchronnie popełniają błędy od czasu do czasu). Techniki odporności na błędy mogą ukryć pewne rodzaje błędów
przed użytkownikiem końcowym.

**Skalowalność** oznacza posiadanie strategii utrzymania dobrej wydajności, nawet w przypadku wzrostu obciążenia. Aby
omówić skalowalność, potrzebujemy najpierw sposobów na ilościowe opisanie obciążenia i wydajności. Krótko przyjrzeliśmy
się domowym liniom czasu Twittera jako przykładowi opisu obciążenia oraz percentylom czasu odpowiedzi jako sposobowi
pomiaru wydajności. W skalowalnym systemie można dodać zdolność przetwarzania, aby pozostać niezawodnym przy dużym
obciążeniu

**Utrzymywalność** ma wiele aspektów, ale w gruncie rzeczy chodzi o to, aby poprawić życie zespołów inżynierskich i
operacyjnych, które muszą pracować z systemem. Dobre abstrakcje mogą pomóc zmniejszyć złożoność i sprawić, że system
będzie łatwiejszy do modyfikacji i dostosowania do nowych przypadków użycia. Dobra operacyjność oznacza dobrą widoczność
stanu systemu oraz skuteczne sposoby zarządzania nim.

Niestety, nie ma szybkiej odpowiedzi na to, jak sprawić, by aplikacje były niezawodne, skalowalne i możliwe do
utrzymania. Istnieją jednak pewne wzorce i techniki, które wciąż pojawiają się w różnych rodzajach aplikacji. W
kolejnych rozdziałach przyjrzymy się kilku przykładom systemów danych i przeanalizujemy, jak działają one w kierunku
tych celów.
