---
layout: post
title: "xUnit test reports for sbt"
description: "xUnit test reports integration for Jenkins and jacoco4sbt."
category: code quality
tags: [scala, code quality, ci]
---
{% include JB/setup %}

## Отчеты о прохождении тестов в формате xUnit

### Отчеты и Jenkins

Для Jenkins существует [плагин](https://wiki.jenkins-ci.org/display/JENKINS/xUnit+Plugin), разбирающий отчеты в формает xUnit. XML-отчет преобразуется в таблицу соответствия тестов и результатов.
Кроме того плагин позволяет отследить динамику прохождения тестов и их количества в виде графиков и таблицы на странице задачи Jenkins (сколько тестов добавлено в последней сборке, сколько тестов сломалось за последний месяц,...).
Визуализация отчетов позволяет быстрее локализовать место возможной ошибки и исправить ее (или тест).

### xUnit и PlayFramework

К сожалению при настройках по умолчанию при прохождении тестов в PlayFramework xUnit отчет не создается, а формат отчета не совпадает с xUnit. В официальной документации на сайте информации о создании подобных отчетов нет.

Однако, в действительности, отчет в формате xUnit создать можно. Как оказалось эта возможность заложена в sbt.
Для создания отчетов необходимо добавить следующий ключ в скрипт сборки в _Build.scala_:

    testOptions in Test += Tests.Argument("junitxml", "console")

При следующем выполнении команды _sbt test_ будет создан файл отчета в папке _$PROJECT/target/test-reports_.

Остается установить плагин для Jenkins и добавть послесборочный шаг публикации отчетов JUnit.

### xUnit и Jacoco

Если Вы используете jacoco4sbt для сборки статистики покрытия тестов, то обнаружите, что XML отчет не формируется при запуске команд _jacoco:test_ или _jacoco:cover_.
Проблема решается также просто:

* Необходимо добавить в зависимости проекта библиотеку _junit-interface_. Эта же библиотека используется для генерации отчетов xUnit в PlayFramework

{% highlight scala %}
    val junitInterface = "com.novocode" % "junit-interface" % "0.10" % "test"
    // Add to play project dependencies config
    appDependencies ++ Seq(junitInterface)
{% endhighlight %}

* Необходимо добавить ключ в конфигурацию сборки в _Build.scala_:

{% highlight scala %}
    testOptions in jacoco.Config += Tests.Argument("junitxml", "console")
{% endhighlight %}

После перезагрузки описания проекта при выполнении команд _jacoco:test_ или _jacoco:cover_ Вы получите отчета в папке _$PROJECT/target/test-reports_.

### Ссылки

[xUnit plugin](https://wiki.jenkins-ci.org/display/JENKINS/xUnit+Plugin)

[jUnit interface](https://github.com/sbt/junit-interface)
