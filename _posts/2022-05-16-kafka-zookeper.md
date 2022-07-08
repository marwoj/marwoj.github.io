---
layout: post
title:  "Kafka pozbywa się zookepeara"
author: wojciech
categories: [ ddia, kafka ]
tags: [ main ]
image: assets/images/shorts/2022-05-16-kafka-zookeper.jpg
comments: false
---

Kafka będzie jeszcze bardziej wydajna, skalowalna, łatwiejsza w utrzymaniu. Będzie też szybciej wstawała po awarii.
Dodatkowo od strony twórców zmniejszą się koszty developmentu — czyli o co chodzi z __KIP500__?

<h2>Jak to działa obecnie</h2>
Kafka to przede wszystkim szyna danych działająca na zasadzie commit-loga. Jednak żeby zapewnić wysoką dostępność oraz
horyzontalną skalowalność, Kafka działa w środowisku rozproszonym. To wymaga koordynacji i do tej pory zajmował się tym
ZooKeper 🐘 🐒 🦒

<h2>Niestety</h2>
__CommitLog__ oraz __ZooKeper__ to dwie odrębne aplikacje różniące się warstwą sieciową, mechanizmami bezpieczeństwa,
sposobami konfiguracji itp. W praktyce wymaga to sporo wysiłku zarówno od twórców, jak i użytkowników Kafki.

<h2>Jak sobie z tym poradzono?</h2>
Od jakiegoś czasu trwają intensywne prace nad pozbyciem się ZooKeepera. Zamiast tego każdy serwer, na którym uruchomiona
jest Kafka, będzie przechowywał metadane. Stan systemu będzie odtwarzany w oparciu o event-sourcing. Nie będzie
nadrzędnego koordynującego serwisu. Serwery Kafki będą głosowały i wypracowywały konsensus 👏

Więcej:

- [Apache Kafka Made Simple: A First Glimpse of a Kafka Without ZooKeeper](https://www.confluent.io/blog/kafka-without-zookeeper-a-sneak-peek)