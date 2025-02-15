+++
date = '2025-02-15T18:29:02+07:00'
title = 'Миграция с Goland на VS Code'
showTags = false
hideBackToTop = true
+++
## А зачем?

Некоторое время назад компания Jetbrains отказала мне в продлении лицензии на "All Product Pack" по моему действующему студенческому билету. Оплатить их продукт с карты российского банка оказалось невозможным. Соответственно пришлось смириться с тем, что Goland в ближайшее время вернуть не получится и нужно искать замену. 

Консольные решения по типу neovim'а мне не нравились, поэтому выбор пал на VS Code. 

## Список плагинов

Ниже приведен список бесплатных плагинов для VS Code, установив которые я смог получить большую часть функционала Goland.

1. [Go](https://marketplace.visualstudio.com/items?itemName=golang.go) - обеспечивает базовую поддержку Go в VS Code.
2. [Tooltitude for Go](https://marketplace.visualstudio.com/items?itemName=tooltitudeteam.tooltitude) - обеспечивает дополнительную поддержку GO в VS Code. Умеет выводить связи между интерфейсами и структурами, которые их реализуют. Есть удобные постфиксные сниппеты для ускорения написания кода.
3. [Go struct tag](https://marketplace.visualstudio.com/items?itemName=liuchao.go-struct-tag) - позволяет быстро дописывать структурные теги.
4. [IntelliJ IDEA Keybindings](https://marketplace.visualstudio.com/items?itemName=k--kato.intellij-idea-keybindings) - переопределяет большинство горячих клавиш на те, которые были в Goland.
5. [JetBrains IDE Icons](https://marketplace.visualstudio.com/items?itemName=Mostik.JetBrainsIcons) - переопределяет иконки файлов на те, что были в Goland.
6. [TODO Highlights](https://marketplace.visualstudio.com/items?itemName=wayou.vscode-todo-highlight) - подсвечивает todo в комментариях.
7. [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one) - позволяет удобно работать с markdown файлами.
8. [YAML](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml) - позволяет удобно работать с yaml файлами.
9. [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) - позволяет высылать запросы к сервисам по http примерно с таким же удобством, как и встроенный в Goland http клиент.
10. [Proto3](https://marketplace.visualstudio.com/items?itemName=zxh404.vscode-proto3) - добавляет поддержку Protocol Buffers Version 3. 
11. [Code Spell Checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) - подсвечивает опечатки в английских словах.
12. [Code Spell Checker Russian](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker-russian) - подсвечивает опечатки в русских словах.
