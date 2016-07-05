---
category: ru
type: paper
hastr: true
layout: paper
tags: linux, разработка
title: Написание своих дополнений для Shell. Zsh
short: writting-own-completions-p1
---
<figure class="img">![bash_completion](/resources/papers/zsh_completion.png)</figure> В
данных статьях описываются некоторые основы создания файлов дополнений для
собственной программы.

<!--more-->

## <a href="#preamble" class="anchor" id="preamble"><span class="octicon octicon-link"></span></a>Преамбула

В процессе разработки [одного своего проекта](/ru/projects/netctl-gui
"Страница netctl-gui") возникло желание добавить также файлы дополнений (только
не спрашивайте зачем). Благо я как-то уже брался за написание подобных вещей, но
читать что-либо тогда мне было лень, и так и не осилил.

## <a href="#introduction" class="anchor" id="introduction"><span class="octicon octicon-link"></span></a>Введение

Существует несколько возможных вариантов написания файла автодополнения для zsh.
В случае данной статьи я остановлюсь только на одном из них, который
предоставляет большие возможности и не требует больших затрат (например, работы
с регулярными выражениями).

Рассмотрим на примере моего же приложения, часть справки к которому выглядит
таким образом:

```bash
netctl-gui [ -h | --help ] [ -e ESSID | --essid ESSID ] [ -с FILE | --config FILE ]
           [ -o PROFILE | --open PROFILE ] [ -t NUM | --tab NUM ] [ --set-opts OPTIONS ]
```

Список флагов:

* флаги `-h` и `--help` не требуют аргументов;
* флаги `-e` и `--essid` требуют аргумента в виде строки, без дополнения;
* флаги `-c` и `--config` требуют аргумента в виде строки, файл с произвольной
локацией;
* флаги `-o` и `--open` требуют аргумента в виде строки, дополнение по файлам из
определенной директории;
* флаги `-t` и `--tab` требуют аргумента в виде строки, дополнение из указанного
массива;
* флаг `--set-opts` требует аргумента в виде строки, дополнение из указанного
массива, разделены запятыми;

## <a href="#file" class="anchor" id="file"><span class="octicon octicon-link"></span></a>Структура файла

В заголовке должно быть обязательно указано, что это файл дополнений и для каких
приложений он служит (можно строкой, если в файле будет содержаться дополнение
для нескольких команд):

```bash
#compdef netctl-gui
```

Дальше идет описание флагов, вспомогательные функции и переменные. Замечу, что
функции и переменные, которые будут использоваться для дополнения **должны
возвращать массивы**, а не строки. В моем случае схема выглядит примерно так
(все функции и переменные в этой главе умышленно оставлены пустыми):

```bash
# variables
_netctl_gui_arglist=()
_netctl_gui_settings=()
_netctl_gui_tabs=()
_netctl_profiles() {}
```

Затем идут основные функции, которые будут вызываться для дополнения для
определенной команды. В моем случае команда одна, и функция одна:

```bash
# work block
_netctl-gui() {}
```

Далее **без выделения в отдельную функцию** идет небольшое шаманство, связанное
с соотнесением приложения, которое было декларировано в первой строке, с
функцией в теле скрипта:

```bash
case "$service" in
    netctl-gui)
        _netctl-gui "$@" && return 0
        ;;
esac
```

## <a href="#flags" class="anchor" id="flags"><span class="octicon octicon-link"></span></a>Флаги

Как я и говорил во введении, существует несколько способов создания подобных
файлов. В частности, они различаются декларацией флагов и их дальнейшей
обработкой. В данном случае я буду использовать команду `_arguments`, которая
требует специфичный формат переменных. Выглядит он таким образом `ФЛАГ[описание]:
СООБЩЕНИЕ:ДЕЙСТВИЕ`. Последние два поля не обязательны и, как Вы увидите чуть
ниже, вовсе и не нужны в некоторых местах. Если Вы предусматриваете два флага
(короткий и длинный формат) на одно действие, то формат чуть-чуть усложняется:
`{(ФЛАГ_2)ФЛАГ_1,(ФЛАГ_1)ФЛАГ_2}[описание]:СООБЩЕНИЕ:ДЕЙСТВИЕ`. Замечу, что,
если Вы хотите сделать дополнения для двух типов флагов, но некоторые флаги не
имеют второй записи, то Вам необходимо продублировать его таким образом:
`{ФЛАГ,ФЛАГ}[описание]:СООБЩЕНИЕ:ДЕЙСТВИЕ`. `СООБЩЕНИЕ` - сообщение, которое
будет показано, `ДЕЙСТВИЕ` - действие, которое будет выполнено после этого флага.
В случае данного туториала, `ДЕЙСТВИЕ` будет иметь вид `->СОСТОЯНИЕ`.

Итак, согласно нашим требованиям, получается такое объявление аргументов:

```bash
_netctl_gui_arglist=(
    {'(--help)-h','(-h)--help'}'[show help and exit]'
    {'(--essid)-e','(-e)--essid'}'[select ESSID]:type ESSID:->essid'
    {'(--config)-c','(-c)--config'}'[read configuration from this file]:select file:->files'
    {'(--open)-o','(-o)--open'}'[open profile]:select profile:->profiles'
    {'(--tab)-t','(-t)--tab'}'[open a tab with specified number]:select tab:->tab'
    {'--set-opts','--set-opts'}'[set options for this run, comma separated]:comma separated:->settings'
)
```

## <a href="#variables" class="anchor" id="variables"><span class="octicon octicon-link"></span></a>Массивы переменных

В нашем случае есть два статических массива (не изменятся ни сейчас, ни через
пять минут) (массивы умышленно уменьшены):

```bash
_netctl_gui_settings=(
    'CTRL_DIR'
    'CTRL_GROUP'
)

_netctl_gui_tabs=(
    '1'
    '2'
)
```

И есть динамический массив, который должен каждый раз генерироваться. Он содержит,
в данном случае, файлы в указанной директории (это можно сделать и средствами
zsh, кстати):

```bash
_netctl_profiles() {
    print $(find /etc/netctl -maxdepth 1 -type f -printf "%f\n")
}
```

## <a href="#body" class="anchor" id="body"><span class="octicon octicon-link"></span></a>Тело функции

Помните, там выше было что-то про состояние? Оно хранится в переменной `$state`,
и в теле функции делается проверка на то, чему оно равно, чтобы подобрать
соответствующие действия. В начале также нужно не забыть вызвать `_arguments` с
нашими флагами.

```bash
_netctl-gui() {
    _arguments $_netctl_gui_arglist
    case "$state" in
        essid)
            # не делать дополнения, ждать введенной строки
            ;;
        files)
            # дополнение по существующим файлам
            _files
            ;;
        profiles)
            # дополнение из функции
            # первая переменная описание
            # вторая массив для дополнения
            _values 'profiles' $(_netctl_profiles)
            ;;
        tab)
            # дополнение из массива
            _values 'tab' $_netctl_gui_tabs
            ;;
        settings)
            # дополнение из массива
            # флаг -s устанавливает разделитель и включает мультивыбор
            _values -s ',' 'settings' $_netctl_gui_settings
            ;;
    esac
}
```

## <a href="#conclusion" class="anchor" id="conclusion"><span class="octicon octicon-link"></span></a>Заключение

Файл хранится в директории `/usr/share/zsh/site-functions/` с произвольным, в
общем-то, именем с префиксом `_`. Файл примера полностью может быть найден
[в моем репозитории](//raw.githubusercontent.com/arcan1s/netctl-gui/master/sources/gui/zsh-completions
"Файл").

Дополнительная информация может быть найдена в репозитории [zsh-completions]
(//github.com/zsh-users/zsh-completions "GitHub"). Например, там есть такой
[How-To](//github.com/zsh-users/zsh-completions/blob/master/zsh-completions-howto.org
"Туториал"). А еще там есть много примеров.