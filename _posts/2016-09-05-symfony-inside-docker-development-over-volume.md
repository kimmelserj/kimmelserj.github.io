---
layout: post
title: "Symfony внутри Docker (разработка при помощи разделов (volumes))"
---
У разработки при помощи разделов я вижу один существенный плюс - это молниеносное обновление кода в контейнере не требующее каких либо дополнительных телодвижений. Но и минусов тоже предостаточно, чтобы свести на нет такое удобство. О минусах данного подхода я кратко написал [здесь]({% post_url 2016-08-29-symfony-inside-docker-development-variants %}).

## Начинаем

Сразу скажу что в контейнере код проекта будет располагаться в директории `/app`. Это важно, так как это будет одной из первых причин наших проблем в дальнейшем. Примеры кода представлены для проектов использующих Symfony 3.

В проекте будет активно использоваться `docker-compose`, так как он очень сильно облегчает задачу по поднятию у себя на машине рабочего окружения для проекта. Сейчас я приведу пример того, как обычно выглядит `docker-compose.yml` в разрабатываемых Symfony-проектах:

    version: '2'
    services:
        db:
            image: postgres:9.5
            environment:
                POSTGRES_PASSWORD: megapassword
                POSTGRES_USER: app_user
                POSTGRES_DB: app_db
        app:
            image: app-base-image
            volumes:
                - .:/app
            links:
                - db

Образ с именем `app-base-image` должен содержать всё необходимое для функционирования кода в контейнере. В данной статье не будем заострять внимание на сборке подобного образа, а пока будем считать, что у нас есть такой образ (может я освещу данный аспект в отдельной статье).

А основное внимание мы уделим следующим строчкам:

            volumes:
                - .:/app

Именно эти строки обеспечивают возможность видеть директорию с кодом проекта внутри контейнера. Причём если какой-то процесс в контейнере создаст или модифицирует какой-либо файл в директории `/app`, то эти изменения сразу же появятся в вашей директории с кодом проекта. Но в этом есть и минус, который я описал [здесь]({% post_url 2016-08-29-symfony-inside-docker-development-variants %}).

## Драка за кэш и проблема с абсолютными путями

Когда я только начал использовать данный вариант, то в течение нескольких дней периодически замечал проблему в виде ошибки "не могу найти файл /home/serj/projects/symfony-app/var/cache/...". По пути было видно, что код пытался подгрузить кэш основываясь на абсолютных путях моей основной системы. А это происходило по причине того, что я на основной системе прогревал кэш при помощи команды `bin/console cache:warmup` и в результате выполнения команды в кэш записывались абсолютные пути к директории с кодом проекта, а в файловой системе контейнера другие пути.

После небольших раздумий я пришёл к выводу, что проблема, в основном, в файлах директории `var`. И решил использовать возможность Symfony переопределять путь к директориям `var/cache` и `var/log`. В результате, в файле `app/AppKernel.php` были модифицированы методы `getCacheDir` и `getLogDir`:

{% highlight php startinline=true %}
public function getCacheDir()
{
    if (in_array($this->getEnvironment(), ['dev', 'test'])) {
        $baseDir = '/tmp/app';
    } else {
        $baseDir = dirname(__DIR__);
    }

    return $baseDir.'/var/cache/'.$this->getEnvironment();
}

public function getLogDir()
{
    if (in_array($this->getEnvironment(), ['dev', 'test'])) {
        $baseDir = '/tmp/app';
    } else {
        $baseDir = dirname(__DIR__);
    }

    return $baseDir.'/var/logs';
}
{% endhighlight %}

Также нужно было переопределить местоположение директории для сессий `var/sessions` в файле `app/config/config.yml`:

    framework:
        ...
        session:
            handler_id: session.handler.native_file
            save_path: "%kernel.cache_dir%/../../var/sessions/%kernel.environment%"

В директории `var` генерируются ещё два файла: `bootstrap.php.cache` и `SymfonyRequirements.php`. Их переносить нельзя, так как эти файлы инклудятся в других PHP-файлах по относительному пути. Так что мне периодически приходится фиксить владельца файлов на основной системе следующими командами:

    sudo chown serj:serj var/bootstrap.php.cache
    sudo chown serj:serj var/SymfonyRequirements.php

Со временем, это начинает жуть как раздражать, но пока другого варианта я не нашёл.

## PHPStorm и плагин для Symfony

В работе я очень активно использую плагин [Symfony Plugin](https://plugins.jetbrains.com/plugin/7219) в PHPStorm. И после выполненых выше манипуляций, нужно будет подправить настройки плагина. Для этого нужно перейти в настройки плагина `File -> Settings -> Languages & Framework -> PHP -> Symfony` и поменять пути:

Path to urlGenerator.php: `/tmp/app/var/cache/dev/appDevUrlGenerator.php`

Translation Root Path: `/tmp/app/var/cache/dev/translations`

Иногда плагин не подтягивает новые изменения, так как нужно прогреть кэш следующей командой:

    bin/console cache:warmup

## Заключение

В целом, если вы готовы терпеть периодические ошибки при выполнении команды `composer install` из-за нехватки прав на файлы `bootstrap.php.cache` и `SymfonyRequirements.php` и вы frontend-разработчик, то это ваш вариант.
