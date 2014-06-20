---
layout: post
title: "Play with P-JAX"
description: "Play и P-JAX"
category: code quality
tags: [play, front-end]
---
{% include JB/setup %}

## P-JAX

### Что такое P-JAX.

P-JAX представляет собой плагин для jQuery, позволяющий перезагружать участок web-страницы и при этом изменять состояние адресной строки браузера и историю посещения страниц.
Проект открытый, исходный код доступен [на GitHub](https://github.com/defunkt/jquery-pjax). Также доступно приложение, чтобы потестировать плагин, [на Heroku](http://pjax.heroku.com).
P-JAX используется самим GitHub.

### Как работает P-JAX

Название представляет собой объединенные 2 термина: PushState и AJAX.
Магия технологии заключается в следующем:

* При нажатии на ссылку, для которой включен P-JAX, отпраляется AJAX запрос на сервер с указанием заголовка _X-PJAX: true_.
* В это время значение в адресной строке браузера меняется на адрес в P-JAX-ссылке, а в истории посещений добавляется новая строка.
* После получения данных от сервера, P-JAX вырезает указанный участок web-страницы и вставляет туда участок полученный от сервера.

Такой подход позволяет существенно сократить время загрузки и отрисовки страницы.
Минусом является необходимость обработки дополнительного заголовка на стороне сервера, чтобы определить передавать ли клиенту страницу целиком или только ее участок.

## Интеграция в Play-приложение

### Подключаем P-JAX на страницу

Для подключения надстройки потребуется добавить jQuery и сам P-JAX

{% highlight html %}
    <script type="text/javascript" src="//ajax.googleapis.com/ajax/libs/jquery/1.11.1/jquery.min.js"></script>
    <script type="text/javascript" src="@controllers.routes.Assets.at("javascripts/jquery.pjax.js")"></script>
{% endhighlight %}

Далее необходимо настроить P-JAX, для этого добавим скрипт:

{% highlight html %}
    <script type="text/javascript">
        $(document).pjax("a[data-pjax="true"]", "#pjax-container", {timeout: 5000, push: true});
    </script>
{% endhighlight %}

Теперь все ссылки с атрибутом _data-pjax="true"_ будут перезагружать содержимое элемента с id _pjax-container_.
Теперь необходимо позаботиться о формах. Для этого добавим следующее:

{% highlight javascript %}
    $(document).on("submit", "form", function(event) {
        $.pjax.submit(event, "#pjax-container", {timeout: 5000, push: true})
    });
{% endhighlight %}

### Обрабатываем P-JAX на сервере

На сервере необходимо разобрать запрос и получить значение заголовка _X-PJAX_ и, в зависимости от него, отправить HTML-представление целиком или частично.
Для этого можно использовать следующий метод

{% highlight scala %}
    object PjaxController extends Controller {

      protected val pjaxHeader = "X-PJAX"
      protected val defaultValue = "false"
      protected val pjaxUrlHeader = "X-PJAX-URL"

      def status : EssentialAction = Action {
        implicit request =>
          val renderFullView = request.headers.toSimpleMap.getOrElse(pjaxHeader, defaultValue).toBoolean
          Ok(view(renderFullView))
      }

    }
{% endhighlight %}

В файл с базовым шаблоном (_layout_) необходимо поместить код управления отрисовкой статической части:

{% highlight scala %}
    @(renderFullView: Boolean)(content: Html)(implicit request: RequestHeader)

    @if(renderFullView) {
      <!DOCTYPE html>
      <!-- full page -->
      @content
    }else{
      <!-- part of view -->
      @content
    }
{% endhighlight %}

Из представления необходимо передать параметр отрисовки в базовый шаблон.

{% highlight scala %}
    @(renderFullView: Boolean)

    @views.html.layout(renderFullView) {
      <!-- View content will be here.-->
    }
{% endhighlight %}

### Подводные камни

* Redirects

По умолчанию P-JAX не будет менять адрес в адресной строке при перенаправлении. Для изменения значения необходимо явно указать какой URL необходимо вставить в адресную строку.

{% highlight scala %}
    pjax_s = request.headers.toSimpleMap.getOrElse(pjaxHeader, defaultValue)
    Redirect(call).withHeaders((pjaxHeader, pjax_s), (pjaxUrlHeader, request.uri))
{% endhighlight %}

* Подтверждение оперции

Если Вы хотите сделать диалог подтверждения, например при удалении элемента, необходимо воспользоваться событием _pjax:beforeSend_ и перехватить событие отправки запроса:

{% highlight html %}
    <script type="text/javascript">
        $(document).on("pjax:beforeSend", function(e) {
            link = $(e.relatedTarget)
            e.preventDefault()
            e.stopImmediatePropagation()
            // do something useful
        });
    </script>
{% endhighlight %}

* JavaScript

При работе с JavaScript и P-JAX необходимо помнить, что фактически страница, с которой мы работаем одна. Весь JavaScript, единожды загруженный, будет храниться в памяти и выполняться.
При этом могут возникать ошибки, связанные с повторной загрузкой скриптов или повторной привязкой событий, например _onClick_.

Также необходимо помнить о дублировании кода инициализации событий при загрузке страницы и при срабатывании _pjax:complete_

* Работа с формами

При возврате ошибок в заполнении формы от сервера клиенту использование HTT-кодов ответов, отличных от 200-ой серии, опасно. По умолчанию такие ответы будут попадать в _error_-callback методов P-JAX и не будут приводить к перерисовке области.
Решением может служить использование _OK_ вместо _BadRequest_ (хотя в примерах PlayFramework используется второе).

### Выводы

Хотя P-JAX не очень распространен в сети, его использование позволяет существенно ускорить загрузку страниц Вашего web-приложения и тем самым улучшить UX.
Для сравнения у нас загрузка страницы без P-JAX занимает 600-2500 мс, с P-JAX 100-300мс.
Также плюсом можно считать неизменность привычного workflow при работе с web-страницами: мы также собираем html на сервере и не задействуем большое количество JavaScript на клиенте.

Можно отметить следующие минусы:

* увеличение количества кода и возможности ошибиться при обработке заголовков и в коде инициализации на клиенте
* меньшая производительность относительно single-page клиентских фремворков (Angular, Ember, Backbone...)
* меньшая или отсутствующая структуризация клиентского кода относительно single-page клиентских фремворков (Angular, Ember, Backbone...)

## Ссылки

* [P-JAX плагин](https://github.com/defunkt/jquery-pjax)
* [Приложение-пример P-JAX](http://pjax.heroku.com)
* [Пример использования PJAX и Play](https://github.com/pvillega/pjax-Forms)
