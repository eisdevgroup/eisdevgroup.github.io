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

### Интеграция в Play-приложение

#### Подключаем P-JAX на страницу

Для подключения надстройки потребуется добавить jQuery и сам P-JAX

```
<script type="text/javascript" src="//ajax.googleapis.com/ajax/libs/jquery/1.11.1/jquery.min.js"></script>
<script type="text/javascript" src="@controllers.routes.Assets.at("javascripts/jquery.pjax.js")"></script>
```

Далее необходимо настроить P-JAX, для этого добавим скрипт:

```
<script type="text/javascript">
$(document).pjax("a[data-pjax="true"]", "#pjax-container", {timeout: 5000, push: true});
</script>
```

Теперь все ссылки с атрибутом _data-pjax="true"_ будут перезагружать содержимое элемента с id _pjax-container_.
Теперь необходимо позаботиться о формах. Для этого добавим следующее:

```
$(document).on("submit", "form", function(event) {
    $.pjax.submit(event, "#pjax-container", {timeout: 5000, push: true})
});
```

#### Обрабатываем P-JAX на сервере

На сервере необходимо разобрать запрос и получить значение заголовка _X-PJAX_ и, в зависимости от него, отправить HTML-представление целиком или частично.
Для этого можно использовать следующий метод

```
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
```

В файл с базовым шаблоном (_layout_) необходимо поместить код управления отрисовкой статической части:

```
@(renderFullView: Boolean)(content: Html)(implicit request: RequestHeader)

@if(renderFullView) {
  <!DOCTYPE html>
  <!-- full page -->
  @content
}else{
  <!-- part of view -->
  @content
}
```

Из представления необходимо передать параметр отрисовки в базовый шаблон.

```
@(renderFullView: Boolean)

@views.html.layout(renderFullView) {
  <!-- View content will be here.-->
}
```



