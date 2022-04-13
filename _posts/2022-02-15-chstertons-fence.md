---
layout: post
title:  "Refactoring vs Chesterton's fence"
author: wojciech
categories: [ tech ]
image: assets/images/shorts/2022-02-15-chestertons-fence.jpg
comments: false
tags: [featured]
---

Mieliście kiedyś taki #refactoring? 💣 Usuwacie lub edytujecie bezsensowny, nieczytelny, źle napisany kod. Wrzucacie
zmianę jeden do jeden — w wersji czytelnej i optymalnej. Dopisujecie testy. Aż tu nagle na środowisku testowym, albo co
gorsza produkcyjnym coś wybucha 💥

Nie przewidzieliście, że ten kod jednak do czegoś służył, z jakiegoś powodu się tam znalazł i z jakiegoś powodu wyglądał
jak wyglądał 🤔 Okazuje się, że takie zjawisko ma swoją nazwę: Chesterton's fence i uczy ono
pokory https://lnkd.in/eZCWSVE

Termin ten nie odnosi się tylko do programowania, ale ma zastosowanie przy jakimkolwiek podejmowaniu decyzji.

PS Podczas refactoringu da się tego problemu uniknąć przy dobrym pokryciu testami istniejącego już kodu. Jeśli jednak
dopiszemy brakujące testy — zgodnie ze sztuką, przed wprowadzeniem zmian — możemy nawet nie wpaść na pomysł
przetestowania pewnych scenariuszy, jeśli nie posiadamy odpowiedniej wiedzy domenowej.