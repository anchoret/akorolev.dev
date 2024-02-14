---
title: "Git — контентно-адресуемая файловая система"
date: 2024-02-14T08:50:24+04:00
draft: false
description: "Механизмы работы внутренней базы данных, типы объектов и их жизненный цикл."
noindex: false
featured: true
pinned: true
comments: true
slug: git-content-addressable-file-system
series:
  - git-plumbing-and-porcelain
categories:
  - education
tags:
  - git
images:
  - mr-cup-fabien-barral-o6GEPQXnqMY-unsplash.jpg
---
{{% bs/collapse "Общее предисловие серии статей" secondary %}}
Систему контроля версий Git первый раз довелось использовать в 2011 году.
Это был достаточно "сомнительный" переход с SVN, потому что еще до старта проекта горели все сроки, лид разработки решил использовать новый фреймворк, но через неделю был переведен в другую команду.
В качестве СУБД "прилетел" неведанный Firebird с ворохом хранимых процедур, потому что мы создавали web-канал взаимодействия для существующего desktop решения.  
Меня торжественно нарекли новым лидом над тремя младшими специалистами, невзирая на обстоятельство, что я был четвертым участником аналогичной позиции.

За последние несколько лет моей практики альтернативы Git перестали встречаться окончательно.
Единственное, что остается неизменным - это младшие специалисты и регулярные ситуации "сломалось/потерялось" и "забыли/перепутали".

Данная статья написана по мотивам официальной [документации Git](https://git-scm.com/doc) и личного опыта.
В примерах команд используется Git версии 2.43 в виде консольного клиента Git Bash.
В большинстве утверждений предполагается работа со стандартной конфигурацией, без отдельных оговорок про различные возможности кастомизации через настройки, параметры команд и переменные окружения.

Git - это распределенная система контроля версий. Технически здесь нет деления на клиента и сервер, а все участники могут существовать независимо, при необходимости равноправно обмениваясь изменениями напрямую между собой.
Но с целью снижения количества связей и формализации "источника правды", обычно выделяют сервер, с которым уже работают остальные пользователи.
Репозитории участников принято называть локальными, а репозиторий на сервере - удаленным.
{{% /bs/collapse %}}

## Вводной тест
{{< bs/collapse "Хотите проверить свои представления по теме? Небольшой тест из 7 вопросов." info >}}
{{< bs/ratio 16x9 >}}
<iframe
src="https://docs.google.com/forms/d/e/1FAIpQLSeGrjJBKjw3A2A8vmWU1HSEPpCCoexc115m2TFxvyzRfmbxbA/viewform"
allowfullscreen="true"
loading="lazy">Загрузка…</iframe>
{{< /bs/ratio >}}
{{< /bs/collapse >}}

## Основные принципы работы
В большинстве сфер нашей жизни, и IT не стало здесь исключением, поведение чего-то становится для нас достаточно понятным и логичным,
если есть представление о внутреннем устройстве данных механизмов.

Одним из базовых определений, которое используют все, является **репозиторий**.
Репозиторий Git - это хранилище данных, представленное специальным набором файлов и "папок".
В локальной копии обычно существует директория **.git**, которая и является непосредственным корнем репозитория, а не его частью.
Файлы и директории, которые находятся на одном уровне с .git - это рабочая область/дерево (working directory/tree/space).
Рабочая область является разделом проекта Git, но не частью репозитория.

Третьим основным разделом проекта выделяют **индекс** (index/staging area), данные относящиеся к нему, хранятся внутри репозитория.
На серверах же создаются голые (bare) репозитории, т.е. какой-нибудь `git@github.com:akorolev-dev/git-plumbing-and-porcelain-content.git`
представляет собой директорию аналогичной структуры, что и локальная .git. 
Рабочая область и индекс в них отсутствуют.

Обычно говорят, что есть два варианта создания нового репозитория:
1. Инициализация пустого локального.
2. Клонирование удаленного репозитория в локальный.

Информационные сообщения команды init подтверждают, что корнем репозитория является либо .git, либо корень проекта при варианте bare.

![Инициализация новых репозиториев](git-init.jpg#center "Инициализация новых репозиториев")

Но по факту клонирование просто начинается с инициализации пустого локального репозитория.
Это можно воспроизвести разными способами.
Например, запросить какой-то репозиторий на существующем сервере, где потребуется пройти аутентификацию вызова.
В момент остановки команды клонирования из-за запроса аутентификационных данных уже произойдет инициализация нового пустого репозитория.
А в файл конфигурации проекта Git будет добавлен первый псевдоним удаленного репозитория **origin**.
Если аутентификация происходит успешно, то начинается загрузка данных (git fetch).

![Инициализация репозитория в процессе клонирования](git-init-while-clone.jpg#center "Инициализация репозитория в процессе клонирования")

Также, если предварительно изменить шаблон локального репозитория, то мы увидим данные изменения по итогам завершения клонирования.

##  Устройство файловой системы
### Пустой репозиторий
Репозиторий Git реализован в виде набора файлов и директорий определенной структуры, с адресацией на основе содержимого.
Подробно внутреннее устройство описано в [10 разделе книги](https://git-scm.com/book/en/v2), мы затронем только отдельные моменты.

Создадим новый проект **content** с веткой **content-article-01**, и для отслеживания изменений внутренней структуры этого репозитория
сделаем его самого рабочей областью второго Git проекта, который для удобства будем называть наблюдателем (**watcher**), с веткой **watcher-article-01**.
![Файловая система пустого репозитория](git-empty-repository-file-system.jpg#center "Файловая система пустого репозитория")
Зафиксируем это начальное состояние файловой системы.
![Фиксация файловой системы пустого репозитория](watcher-empty-repository-file-system.jpg#center "Фиксация файловой системы пустого репозитория")
{{% bs/alert warning %}}
Обратите внимание, что в репозитории уже существуют директории **objects** и **refs**, но в коммит они добавлены не были, так как еще не содержат никаких файлов.
{{% /bs/alert %}}

### Создание первого файла
Давайте в качестве первого файла создадим MIT лицензию, скачав её с github.
![Файловая система после добавления в индекс нового файла](watcher-after-add-license-into-index.jpg#center "Файловая система после добавления в индекс нового файла")
После создания нового файла, в репозитории никаких изменений не произошло.
Но вот после включения файла в индекс, создался не только сам файл индекса, но и новый файл в поддиректории внутри **objects**.

В Git существует забавное разделение всех команд на два типа:
1. "Фарфор" (Porcelain) - это как раз привычные большинству пользователей команды.
2. "Сантехника" (Plumbing) - это команды низкого уровня, которые в первую очередь предназначены для создания инструментов работы с репозиторием Git.
   Но именно "сантехника" позволяет понять принципы работы данной системы контроля версий.

Посмотрим содержимое нового объекта. Только стоит учесть, что все объекты базы данных Git сжаты алгоритмом {{< abbr "zlib" >}} и открыть напрямую их не получится.
![Содержимое первого blob объекта](first-object-zlib-deflate.jpg#center "Содержимое первого blob объекта")
После первых 9 символов наблюдаем полный текст лицензии, которая была включена в индекс.
Воспользовавшись некоторыми "сантехническими" командами, можно объяснить происхождение всех данных, авторами которых мы являемся опосредованно.
![Анализ данных первого blob объекта](first-object-inspections.jpg#center "Анализ данных первого blob объекта")
Репозиторий Git представляет собой базу данных типа ключ-значение. Значение называется объектом Git и может быть одного из четырёх типов.
Ранее была упомянута "адресация на основе содержимого", этот принцип заключается в способе формирования ключа, который представляет собой контрольную сумму по алгоритму {{< abbr "SHA-1" >}}.
Каждый сорокасимвольный хэш вычисляется на основе содержимого соответствующего объекта.

Любой объект репозитория создается в поддиректории каталога **objects**.
Наименование файла объекта и наименование его родительской директории получается путем разделения вычисленной контрольной суммы на две части:
1. Первые 2 символа формируют название родительской директории.
2. Оставшиеся 38 символов становятся наименованием файла с данными по объекту.

**Blob** - это первый тип создаваемых объектов, который представляет собой снимок отдельного файла.
А 1069 - количество байт исходной информации в файле LICENSE, до её сжатия.
Метаинформация о типе и размере отделяются от исходного содержимого нулевым символом (NUL/null byte).
Стоит обратить внимание, что метаинформация не содержит сведений о наименовании и временных метках, как и назначенных на файл прав.

### Модификация первого файла
Проверим что будет, если удалить файл из индекса, изменить в нём строку и опять добавить в индекс.
![Изменения в репозитории при незначительной модификации данных](first-file-modification.jpg#center "Изменения в репозитории при незначительной модификации данных")
Удаление файла лицензии из индекса не приводит к удалению соответствующего объекта из базы данных.
А после добавления в индекс уже модифицированного файла, происходит создание второго объекта.

![Анализ данных второго blob объекта](second-object-inspections.jpg#center "Анализ данных второго blob объекта")
Таким образом можно выделить еще несколько ключевых принципов работы репозитория Git:
- База данных практически всегда работает только на добавление новых объектов, не торопясь удалять ранее созданные, для которых уже нет исходных файлов.
- Даже при небольшом изменении файла (модификация 1 строки из 22, и добавлении 18 символов к 1069) производится полный снимок содержимого исходного файла.

### Создание первого коммита
Что же, самое время отследить изменения, которые произойдут после создания первого коммита в репозитории "content".
![Файловая система после создания коммита](first-commit.jpg#center "Файловая система после создания коммита")
Можно увидеть, что создание коммита привело к появлению уже не просто одного нового объекта, а шести разных файлов, среди которых два новых объекта.
Многие из этих файлов будут рассмотрены в рамках других статей этой серии, сейчас проанализируем только объекты. И не забудем предварительно зафиксировать состояние отслеживаемого репозитория внутри **watcher**.

![Анализ файловой системы после создания первого коммита](first-commit-inspections.jpg#center "Анализ файловой системы после создания первого коммита")
Первый объект обладает типом **commit**. В содержимом можно увидеть данные по пользователям, которые сделали изменение и сформировали сам коммит, в данном случае они совпадают.
Содержится хэш объекта **tree**, он как раз совпадает с адресом второго объекта, который появился после создания коммита.
После почтового ящика пользователя содержится временная метка создания коммита, с точностью до секунд.

Объект tree — это представление какой-то директории, в данном случае корня рабочей области. Он содержит ключи объектов дочерних tree и blob.
Именно в нем отражаются информация по правам доступа и наименованию файлов или директорий.

Таким образом любой коммит кроме метаинформации о себе содержит ключ полного снимка состояния рабочей директории, начиная с её корня.
Да, Git не является системой контроля версий, работающей с отдельными дельтами изменений. Но и снимки состояний делаются только при необходимости.

### Создание второго коммита
![Анализ файловой системы после создания второго коммита](second-commit-inspections.jpg#center "Анализ файловой системы после создания второго коммита")
Если добавить второй файл и создать новый коммит, то будет создано три новых объекта:
- commit
- новое tree, так как теперь внутри него два файла
- новый blob для main.txt
  А вот по части файла LICENSE, содержимое которого не поменялось, будет переиспользован ранее созданный объект, ведь его хэш-адрес остался тем же.
  Так что "полный снимок" не влечет многократное дублирование tree и blob объектов, если их содержимое не было изменено.

Единственное на что хотелось бы здесь заострить внимание, так это поле **parent**, которое содержит адрес предыдущего коммита.
Именно оно формирует историю изменений. И одновременно является причиной обновления всех хеш-ключей в ветке коммитов, если внести изменения в любое не последнее звено этой цепочки.

## Дополнение
Внимательный читатель мог заметить, что мы рассмотрели объекты трех типов, а в тексте было упоминание, что всего их четыре.
Это не опечатка. Четвертым типом является tag, но такие объекты порождаются для аннотированных тэгов.
А вот "лёгкие" тэги работают только через механизм ссылок и не приводят к созданию нового объекта.
![Создание легкого и аннотированного тэгов](object-type-tag.jpg#center "Создание легкого и аннотированного тэгов")

{{% bs/alert secondary %}}
Источник изображения в заголовке Unsplash. Автор [Mr Cup / Fabien Barral](https://unsplash.com/@iammrcup).
{{% /bs/alert %}}