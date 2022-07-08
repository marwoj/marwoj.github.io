---
layout: post
title:  "Refactoring vs Chesterton's fence"
author: wojciech
categories: [ tech ]
image: assets/images/shorts/2022-05-09-chstertons-fence.jpg
comments: false
tags: [featured, main]
---
<h2>Płot Chestertona w pracy programisty</h2>
Mieliście kiedyś taki refactoring? Usuwacie lub edytujecie bezsensowny, nieczytelny, źle napisany kod. Wrzucacie
zmianę jeden do jeden — w wersji czytelnej i optymalnej. Dopisujecie testy. Aż tu nagle na środowisku testowym, albo co
gorsza produkcyjnym coś wybucha?

Nie przewidzieliście, że ten kod jednak do czegoś służył, z jakiegoś powodu się tam znalazł i z jakiegoś powodu wyglądał
jak wyglądał. Okazuje się, że takie zjawisko ma swoją nazwę: Chesterton's fence i uczy ono
pokory. 

Podczas refactoringu da się tego problemu uniknąć przy dobrym pokryciu testami istniejącego już kodu. Jeśli jednak
dopiszemy brakujące testy — zgodnie ze sztuką, przed wprowadzeniem zmian — możemy nawet nie wpaść na pomysł
przetestowania pewnych scenariuszy, jeśli nie posiadamy odpowiedniej wiedzy domenowej.

<h2>Etymologia pojęcia</h2>
Termin ten nie odnosi się tylko do programowania, ale ma zastosowanie przy jakimkolwiek podejmowaniu decyzji. Zasada ta
mówi, że nie powinno się wprowadzać zmian, bez dogłębnego zrozumienia decyzji stojących za podjętymi działaniami.

Nazwa została uknuta w opraciu o eksperyment myślowy. Wyobraź sobie, że podróżujesz samochodem po pustej polnej drodze
na odludziu, a nagle znikąd pojawia się płot. Pierwszym odruchem może być stwierdzenie, że ten płot jest niepotrzebny,
ale być może pełni on jakąś funkcję? Ktoś jednak włożył pewnien wysiłek w jego zbudowanie i raczej nie zrobił tego bez
powodu.

Więcej:

- [Chesterton’s Fence: A Lesson in Second Order Thinking](https://fs.blog/chestertons-fence/)