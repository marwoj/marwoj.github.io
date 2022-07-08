---
layout: post
title:  "Co się dzieje gdy wciskasz Play na Netflix?"
author: wojciech
categories: [ tech ]
image: assets/images/shorts/2022-04-18-netflix-vs-hbo-max.jpg
comments: false
tags: [sticky, main]
---

W tym roku żegnamy się z HBO Go i czekamy na opóźniającą się premierę HBO MAX.
W związku z tym, na platformie HBO pojawi się więcej treści, która niestety znieknie z Netflixa, m.in. filmy Warner
Bros.

Główny powód dla którego powstaje nowa platforma to próba unifikacji usług świadczonych przez HBO w różnych krajach, ale
co bardziej istotne z punktu widzenia użytkownika — poprawa jakości działania platformy.

Jak istotny to jest czynnik i ile pracy wymaga, pokazuje świetny artykuł: [Netflix: What Happens When You Press Play?
](http://highscalability.com/blog/2017/12/11/netflix-what-happens-when-you-press-play.html)

_Czy wiesz że..._ Netflix tworzy pliki wideo dostosowane do wielu urządzeń i szybkości sieci? Drugi sezon serialu
„Stranger Things" został nakręcony w rozdzielczości 8K i ma dziewięć odcinków — a to oznacza że pliki wideo miały wiele
terabajtów. Zakodowanie tylko jednego sezonu zajęło 190 000 godzin pracy procesora. Wynik? 9570 różnych plików wideo,
audio i tekstowych!

<h2>Ale od początku</h2>

Netflix został założony 1997 roku i na początku oferował __wypożyczanie płyt DVD z dostawą do domu__ za pośrednictwem
poczty. Od tego czasu wiele się zmieniło. Obecnie serwis posiada ponad 200 milionów subskrybentów i świadczy usługi w
niemal wszystkich krajach na świecie. Dziennie dostarcza __164 miliony godzin__ treści wideo.
YouTube co prawda dziennie dostarcza 1 miliard godzin wideo, jednak jakość oraz rozmiar wideo na Netflix są wyższe.

Jak to rozumieć? Netflix jest ogromny! Przy tej skali, wysoka jakość usług jest wyzwaniem. Jak radzą sobie z tyms w
Netflix?

<h2>Netflix - usługa w chmurze</h2>

Netflix kontroluje cały proces dostarczania treści — produkcję, obróbkę (AWS), streaming (Open Connect) oraz aplikacje
klienckie wraz z SDK na różne urządzenia. Dzięki kontroli całego procesu, Netflix działa tak dobrze.

<h3>Co się dzieje zanim klikasz Play</h3>
Żeby w momencie naciśnięcia Play, wszystko zadziałało płynnie, wiele musi się wydarzyć wcześniej.

Netflix działa na wielu urządzeniach — co znaczy wiele formatów oraz rozdzielczości. Pliki źródłowe muszą zostać
przekonwertowane, a to wymaga mocy obliczeniowej — nawet __300 000 CPU__ od AWS w danym momencie.
Lista popularnych wideo, rekomendacje, historia obejrzeń, odtwarzanie od ostatniego momentu - wszystko to dzieje się w
chmurze AWS.

<h3>Co się dzieje gdy klikasz Play?</h3>

Streaming danych jest kluczowy dla Netflixa. Teoretycznie można skorzystać z AWS S3 do przechowywania i dostarczania
plików wideo, jednak przy tej skali byłoby to obciążające dla światowego internetu. Dostawcy internetu mogliby się
zbuntować i spowolnić Netflixa — co zresztą miało już miejsce w 2014 roku.

Żeby tego uniknąć, Netflix posiada własną sieć CDN — _Open Connect_. Serwery, tzw. Open Connect Appliances (OCAs)
rozlokowane są na całym świecie, a do tego Netflix umiesza OCA u dostawców internetu. Dzięki temu dostawcy mogą
dostarczać treści przy mniejszym obciążeniu sieci a klienci korzystają z wysokiego transferu danych. Klasyczne win-win
situation.

<h2>Stare dobre DVD</h2>
Zanim uruchomimy wideo wiele się dzieje — rekomendacje, personalizacja, konwertowanie wideo. Kiedy już klikniemy Play,
cały internet pracuje żeby dostarczyć film lub serial do naszego domu.

Na koniec jednak trzeba pamiętać że nie każdy posiada (szybki) internet. Netflix o tym pamięta i nadal oferuje __wysyłkę
DVD pocztą__.

<h2>Więcej</h2>
<ul>
<li><a href="http://highscalability.com/blog/2017/12/11/netflix-what-happens-when-you-press-play.html">Netflix: What Happens When You Press Play?</a></li>
<li><a href="https://www.comparitech.com/blog/vpn-privacy/netflix-statistics-facts-figures">50+ Netflix statistics & facts that define the company’s dominance in 2022</a></li>
<li><a href="https://earthweb.com/netflix-statistics/">NETFLIX STATISTICS 2022: HOW MANY SUBSCRIBERS DOES NETFLIX HAVE?</a></li>
<li><a href="https://hotdog.com/tv/stream/netflix/during-quarantine/">What Americans Did During Quarantine</a></li>
<li><a href="https://www.businessinsider.nl/does-netflix-still-mail-dvds?international=true&r=US">Yes, Netflix still mails DVDs</a></li>
</ul>

