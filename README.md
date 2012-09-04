[ [Домашняя страница](http://mekras.github.com/botobor/) ] [ [Документация по API](http://mekras.github.com/botobor/api/) ]

Принципы работы
===============

Самый распространённый на сегодняшний день способ защиты веб-форм от роботов — заставить
пользователя доказать, что он человек, путём выполнения действий, которые роботу выполнить
затруднительно ([CAPTCHA](http://ru.wikipedia.org/wiki/CAPTCHA)). Вот только с точки зрения
удобства использования это не очень хорошо. Ведь таким образом мы заставляем пользователя выполнять
ненужные ему и не всегда простые действия.

[Нед Батчелдер](http://nedbatchelder.com/site/aboutned.html) предлагает зайти с другого конца.
Вместо того, чтобы заставлять напрягаться наших любимых, но ленивых пользователей разбирать,
заставить роботов выдать себя. Подробно об этом написано в его статье
[Stopping spambots with hashes and honeypots](http://nedbatchelder.com/text/stopbots.html).

Ботобор представляет собой библиотеку PHP, реализующую идеи Неда. На данный момент используются
следующие проверки (любая из них может быть отключена):

* между созданием формы и её отправкой прошло слишком мало времени;
* между созданием формы и её отправкой прошло слишком много времени;
* заполнено хотя бы одно поле-приманка (см. ниже);
* заголовок REFERER не совпадает с адресом, где была размещена форма.

Кто может сказать: «Эй, да это же всё обходится в два счёта!». Конечно так и есть. Но Ботобор не
ставит своей целью абсолютную защиту (да такое и невозможно). Цель Ботобора скромнее — уменьшить
вероятность заполнения формы роботом, не напрягая при это пользователей-людей. Кстати, Ботобор и
CAPTCHA могут использоваться вместе: Ботобор в качестве первой линии обороны, CAPTCHA в качестве
второй, если остаются сомнения в человечности пользователя. Подробнее об этом будет написано ниже.

Установка
=========

Во-первых, Вы можете просто скачать файл botobor.php и подключить его к своему проекту любым
удобным Вам способом.

Во-вторых, можно использовать [composer](http://getcomposer.org/).

Установите composer в Ваш проект:

    curl -s http://getcomposer.org/installer | php

Создайте в корне своего проекта файл composer.json:

```json
{
    "repositories":
    [
        {
            "type": "vcs",
            "url": "https://github.com/mekras/botobor"
        }
    ],
    "require":
    {
        "mekras/botobor": "0.4.*"
    },
}
```

Установите, используя composer:

    php composer.phar install

Использование
=============

Простой пример
--------------

Код PHP, создающий форму:

```php
<?php
require 'path/to/botobor.php';
...
// Получите разметку формы тем способом, который предусмотрен у вас в проекте, например:
$html = $form->getHTML();
// Создайте объект-обёртку:
$bform = new Botobor_Form($html);
// Получите новую разметку формы
$html = $bform->getCode();
```

Код PHP, обрабатывающий форму:

```php
<?php
require 'path/to/botobor.php';
...
if (Botobor_Keeper::isRobot())
{
    // Форма отправлена роботом, выводим сообщение об ошибке.
}
```

Пример с опциями
----------------

Можно менять поведение Ботобора при помощи опций. Например, для форм комментариев имеет смысл
увеличить параметр lifetime (наибольший промежуток между созданием и отправкой формы), т. к.
посетители перед комментированием могут долго читать статью.

Это можно сделать так:

```php
<?php
$bform = new Botobor_Form($html);
$bform->setLifetime(60); // 60 минут
```

Подробнее об опциях см. описание методов setCheck, setDelay и setLifetime в
[документации API](http://mekras.github.com/botobor/api/).

Пример с приманкой
------------------

Поля-приманки предназначены для отлова роботов-пауков, которые находят формы самостоятельно.
Такие роботы, как правило, ищут в форме знакомые поля (например, name) и заполняют их. Ботобор
может добавить в форму скрытые от человека (при помощи CSS) поля с такими именами. Человек
оставит эти поля пустыми (т. к. просто не увидит), а робот заполнит и тем самым выдаст себя.

В этом примере поле «name» будет сделано приманкой. При этом имя настоящего поля «name» будет
заменено на случайное значение. Обратное преобразование будет сделано во время вызова
метода Botobor_Keeper::handleRequest (вызывается автоматически из Botobor_Keeper::isRobot).

```php
$bform = new Botobor_Form($html);
$bform->setHoneypot('name');
```

Совместное использование с CAPTCHA
----------------------------------

Ботобор может использоваться совместно с CAPTCHA. Один из вариантов может быть таким. Если проверка
Ботобором показала, что форму заполнил робот, можно поступить так же, как в похожей ситуации
поступает Яндекс — запросить ввод кода с картинки. В коде это может выглядеть как-то так:

```php
<?php
function checkRequest()
{
    // Проверяем, использовался ли CAPTCHA в этом запросе
    if ($someCaptcha->isUsedInThisRequest())
    {
        /*
         * Попадание сюда говорит о том, что посетитель уже проходил, но не прошёл проверку
         * Ботобором и сейчас проходит проверку CAPTCHA. По итогам этой проверки мы либо признаём в
         * посетителе человека, либо окончательно отказываем ему.
         */
         if (!$someCaptcha->isPassed())
         {
             $this->showErrorNotify();
         }
    }
    elseif (Botobor_Keeper::isRobot())
    {
        /*
         * Попадание сюда горовит о том, что посетитель не прошёл проверку Ботобором. На случай если
         * это было ложное срабатывание, мы дадим посетителю возможность пройти CAPTCHA, чтобы
         * доказать, что он не робот.
         */
         $this->showCaptcha();
    }

    // Посетитель — человек, можно обрабатывать его запрос
    $this->processRequest();
}
```
