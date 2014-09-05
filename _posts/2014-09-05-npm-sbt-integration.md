---
layout: post
title: "NPM SBT integration"
description: "Integration of NPM and node tools in sbt build process."
category: Play framework
tags: [play, npm, tools]
---
{% include JB/setup %}

## Интеграция NPM в SBT-сборку

### Node.js, NPM

Зачем нужен Node.js и NPM для PlayFramework?

С момента своего появления Node.js и NPM стали не только популярным инструментом для разработки приложений.
Сейчас на базе node.js появилось множество инструментов, которые стали стандартом для front-end разработчиков.
Инструменты, аналогичные Grunt или Gulp, позволяют разработчикам избавиться от рутинных операций копирования файлов, компиляции Less, Coffeescript и прочих часто выполняемых операций.
Bower выступает в качетсве менеджера пакетов и позволяет автомтизировать загрузку библиотек.
Yeoman выполняет грязную работу по созданию шаблонных файлов для используемого JavaScript-фреймворка.

С другой стороны, новые инструменты в проекте требуют больших затрат на развертывание и обучения программиста сборке проекта.
Чтобы упростить процесс настройки и сборки проекта можно добавить интеграцию NPM-инструментов в существующую сборку на SBT.

### Интеграция команд в командный интерфейс SBT

Добавим интеграцию Grunt, Yo, NPM и любого другого необходимого инструмента в интерактивную командную строку SBT.
Для этого создадим в файле конфигурации сборки следующие объекты:

    object CustomTasks {
      def npm: Command = Command.args("npm", "<task>") { (state, args) =>
        ("npm " + args.mkString(" ")).!
        state
      }

      def yo: Command = Command.args("yo", "<task>") { (state, args) =>
        ("yo " + args.mkString(" ")).!
        state
      }

      def bower: Command = Command.args("bower", "<task>") { (state, args) =>
        ("bower " + args.mkString(" ")).!
        state
      }

      def all: Seq[Command] = Seq(npm, yo, bower)
    }

    object GruntTasks {

      def grunt: Command = Command.args("grunt", "<task>") { (state, args) =>
        ("grunt " + args.mkString(" ")).!
        state
      }

      def `grunt:watch`: Command = Command.command("grunt-watch") { state =>
        "grunt watch".!
        state
      }

      def `grunt:default`: Command = Command.command("grunt-default") { state =>
        "grunt".!
        state
      }

      def all: Seq[Command] = Seq(grunt, `grunt:watch`, `grunt:default`)
    }

Объекты CustomTasks и GruntTasks определяют набор задач (команд для исполнения в командной строке) для интеграции в интерактивную командную строку SBT.
Таким же методом можно добавить любые другие команды в sbt.

Теперь необходимо добавить команды в конфигурациюю проекта:

    val main = Project(appName, file(".")).enablePlugins(play.PlayScala).settings(
      // other configuration go here
      commands ++= GruntTasks.all,
      commands ++= CustomTasks.all
    )

После перезапуска командной строки, команды вида *sbt npm update* будут доступны.

### Запуск задачи Grunt при сборке проекта

Для постоянной пересборки элементов, которыми управляет Grunt, необходимо добавить исполнение команды во время компиляции приложений.
Для этого необходимо добавить следующее в файл сборки:

      // Grunt task to run on "sbt run"
      val gruntDefault = TaskKey[Unit]("grunt default")
      val gruntSettings = gruntDefault := GruntTasks.`grunt:default`

      val main = Project(appName, file(".")).enablePlugins(play.PlayScala).settings(
        // other configuration go here
        playRunHooks <+= baseDirectory.map(base => Grunt(base)),
        compile in Compile <<= (compile in Compile).dependsOn(gruntDefault)
      )

После перезагрузки проекта, при вызове команд _sbt run_ или _sbt compile_ будет вызываться процесс Grunt.

### Прочее

Данный метод доступен для проектов с использованием PlayFramework 2.3 и старше. В следующих версиях команда Typesafe обещала сделать более простую интеграцию с Grunt и другими инструментами из NPM.

### Ссылки

* [Grunt](http://gruntjs.com/)
* [Bower](http://bower.io/)
* [Yeoman](http://yeoman.io/)
