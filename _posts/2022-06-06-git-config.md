---
layout: post
title:  "Praca z git - wiele kont na jednym komputerze"
author: wojciech
categories: [ tech ]
image: assets/images/shorts/2022-06-06-git-config.jpg
comments: false
---

💡 Tip of the day. Jeśli na komputerze macie repozytoria firmowe, jak i prywatne, które korzystają z różnych adresów
mailowych, możecie skorzystać z conditional git configuration. Dzięki temu nie trzeba pamiętać o aktualizacji .gitconfig
dla każdego repozytorium z osobna żeby rozróżniać autora commitów.

Jak to działa u mnie?
W `~/.gitconfig` mam warunki które wymuszają użycie innych konfiguracji w zależności od katalogu:

```
[includeIf "gitdir:~/workspace/nexocode/"]
path = ~/workspace/nexocode/.gitconfig
```

```
[includeIf "gitdir:~/workspace/projects/"]
path = ~/workspace/projects/.gitconfig
```

Wtedy w `~/workspace/nexocode/.gitconfig` ląduje:
```
[user]
email = my-company-mail@nexocode.com
name = Wojciech Marusarz
```

A w `~/workspace/projects/.gitconfig`
```
[user]
email = my-private-mail@gmail.com
name = Wojciech Marusarz
```

Ważne, żeby z `~/.gitconfig` usunąć sekcję `[user]`

W katalogu __nexocode__ git korzysta z mojego konta firmowego, a w katalogu __projects__ z konta prywatnego.
Zawsze to jedna rzecz mniej o której trzeba pamiętać.