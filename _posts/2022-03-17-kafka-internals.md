---
layout: post
title:  "Jak Kafka działa pod spodem?"
author: wojciech
categories: [ tech ]
image: assets/images/shorts/2022-03-17-kafka-internals.jpg
comments: false
tags: [featured]
---
Jak działa #Kafka? W sieci jest mnóstwo teorii, czym jest #topic, #broker, #partycja.

🤔 𝗔𝗹𝗲 𝗰𝗼 𝘀𝗶𝗲𝗱𝘇𝗶 𝗽𝗼𝗱 𝘀𝗽𝗼𝗱𝗲𝗺? 🤔

Oczywiście nic innego jak nieśmiertelne pliki 📂. Jak to wygląda?

📨 Topic grupujący wiadomości określonego typu dzielony jest na partycje.

🖥️ Żeby umożliwić skalowalność i replikację, partycje są przechowywane na różnych brokerach.

💽 Partycja dzielona jest na segmenty, a segment to nic innego jak plik na dysku
📂 Wiadomości są przechowywane w plikach .𝘭𝘰𝘨.
📂 Offsety ułatwiające znajdowanie wiadomości są w plikach .𝘪𝘯𝘥𝘦𝘹 i .𝘵𝘪𝘮𝘦𝘪𝘯𝘥𝘦𝘹

💽 Aktywny segment, do którego zapisywane są nowe wiadomości to 📂 plik read/write. Jest to append log, tj. nic nie jest
kasowane ani edytowane, tylko zapis — stąd wysoka wydajność Kafki.

📂 Kiedy .𝘭𝘰𝘨 za bardzo urośnie lub upłynie określona ilość czasu, następuje rolowanie. Rolowanie to zamknięcie pliku
do zapisu i otworzenie go tylko do odczytu.

💽 Czas, przez jaki wiadomości dostępne są do odczytu, liczony jest na podstawie daty ostatniej wiadomości w pliku.
Potem plik jest usuwany.

Przynajmniej częściowe zrozumienie, jak Kafka działa pod spodem, pozwala na oswojenie się z narzędziem i zachęca do
eksperymentowania. Myślę, że warto zapoznać się z artykułem, który podsyłam w komentarzu.

Zrozumiały blog post wyjaśniający jak #Kafka operuja na plikach: https://strimzi.io/blog/2021/12/17/kafka-segment-retention


