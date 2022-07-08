---
layout: post
title:  "Jak Kafka działa pod spodem?"
author: wojciech
categories: [ kafka, ddia ]
image: assets/images/shorts/2022-06-20-kafka-internals.jpg
comments: false
tags: [featured, main]
---
Jak działa Kafka? W sieci jest mnóstwo teorii, czym jest topic, broker, partycja.

Ale co siedzi pod spodem?

Oczywiście nic innego jak nieśmiertelne pliki 📂. Jak to wygląda?

Topic jest to logiczny podział, grupujący wiadomości określonego typu, np. topic _messages_, oraz topic _statistics_.
Każdy topic dzielony jest na partycje.

🖥️ Skąd podział na partyce? Żeby umożliwić skalowalność i replikację, partycje są przechowywane na różnych serwerach
gdzie zainstalowana jest Kafka. Serwery te nazywamy brokerami.

💽 Partycja dzielona jest na segmenty, a segment to nic innego jak plik na dysku<br>
📂 Wiadomości są przechowywane w plikach _.log_<br>
📂 Offsety ułatwiające znajdowanie wiadomości są w plikach _.indes_ i _.timeindes_

💽 Aktywny segment, do którego zapisywane są nowe wiadomości to plik read/write. Jest to append log, tj. nic nie jest
kasowane ani edytowane, tylko zapis — stąd wysoka wydajność Kafki.

📂 Kiedy _.log_ za bardzo urośnie lub upłynie określona ilość czasu, następuje rolowanie. Rolowanie to zamknięcie pliku
do zapisu i otworzenie go tylko do odczytu.

💽 Czas, przez jaki wiadomości dostępne są do odczytu, liczony jest na podstawie daty ostatniej wiadomości w pliku.
Potem plik jest usuwany.

Przynajmniej częściowe zrozumienie, jak Kafka działa pod spodem, pozwala na oswojenie się z narzędziem i zachęca do
eksperymentowania.

Warto zapoznać się z artykułem <a href="https://strimzi.io/blog/2021/12/17/kafka-segment-retention">Deep dive into
Apache Kafka storage internals: segments, rolling and retention</a>, gdzie mechnizmy Kafki są opisane bardziej
szczegółowo.


