---
layout: post
title:  "Kafka pozbywa się zookepeara"
author: wojciech
categories: [ ddia, kafka ]
image: assets/images/shorts/2022-02-20-kafka-zookeper.jpg
comments: false
---

Kafka będzie jeszcze bardziej wydajna, skalowalna, łatwiejsza w utrzymaniu. Będzie też szybciej wstawała po awarii. 👏
Dodatkowo od strony twórców zmniejszą się koszty developmentu — czyli o co chodzi z #KIP500

𝗝𝗮𝗸 𝘁𝗼 𝗱𝘇𝗶𝗮ł𝗮 𝗼𝗯𝗲𝗰𝗻𝗶𝗲?
#Kafka to przede wszystkim szyna danych działająca na zasadzie commit-loga. Jednak żeby zapewnić wysoką dostępność oraz
horyzontalną skalowalność, Kafka działa w środowisku rozproszonym. To wymaga koordynacji i do tej pory zajmował się tym
ZooKeper 🐘 🐒 🦒

𝗡𝗶𝗲𝘀𝘁𝗲𝘁𝘆...
#CommitLog oraz #ZooKeper to dwie odrębne aplikacje różniące się warstwą sieciową, mechanizmami bezpieczeństwa,
sposobami konfiguracji itp. W praktyce wymaga to sporo wysiłku zarówno od twórców, jak i użytkowników Kafki.

𝗝𝗮𝗸 𝘀𝗼𝗯𝗶𝗲 𝘇 𝘁𝘆𝗺 𝗽𝗼𝗿𝗮𝗱𝘇𝗼𝗻𝗼?
Od jakiegoś czasu trwają intensywne prace nad pozbyciem się ZooKeepera. Zamiast tego każdy serwer, na którym uruchomiona
jest Kafka, będzie przechowywał metadane. Stan systemu będzie odtwarzany w oparciu o event-sourcing. Nie będzie
nadrzędnego koordynującego serwisu. Serwery Kafki będą głosowały i wypracowywały konsensus 👏

https://www.confluent.io/blog/kafka-without-zookeeper-a-sneak-peek/