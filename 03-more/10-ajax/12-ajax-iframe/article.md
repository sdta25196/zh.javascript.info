# IFRAME для AJAX-запросов

Транспорт `iframe` -- самый кросс-браузерный, это его основное преимущество. Он работает везде: от IE6 до Opera Mobile.

Для общения с сервером создается невидимый `IFRAME`. В него отправляются данные, и в него же сервер пишет ответ.

[cut]

## Введение

Сначала -- немного вспомогательных функций и особенности работы с `IFRAME`.

### Двуличность IFRAME: окно+документ

Что такое `IFRAME`? На этот вопрос у браузера два ответа
<ol>
	<li>`IFRAME` -- это HTML-тег: <code>&lt;iframe&gt;</code> со стандартным набором свойств.
<ul>
	<li>Тег можно создавать в JavaScript</li>
	<li>У тега есть стили, можно менять.</li>
	<li>К тегу можно обратиться через `document.getElementById` и другие методы.</li>
</ul>
</li>
	<li>`IFRAME` -- это окно браузера, вложенное в основное
<ul>
	<li>`IFRAME` -- такое же по функционалу окно браузера, как и основное, с адресом и т.п.</li>
        <li>Если документ в `IFRAME` и внешнее окно находятся на разных доменах, то прямой вызов методов друг друга невозможен.</li>
	<li>Ссылку на это окно можно получить через `window.frames['имя фрейма']`.</li>
</ul>
</li>
</ol>

Для достижения цели мы будем работать как с тегом, так и с окном. Они, конечно же, взаимосвязаны.

**В теге `IFRAME` свойство `contentWindow` хранит ссылку на окно.**

Окна также содержатся в коллекции `window.frames`.

Например:

```js
// Окно из ифрейма
var iframeWin = iframe.contentWindow;

// Можно получить и через frames, если мы знаем имя ифрейма (и оно у него есть)
var iframeWin = window.frames[iframe.name];
iframeWin.parent == window; // parent из iframe указывает на родительское окно

// Документ не будет доступен, если iframe с другого домена
var iframeDoc = iframe.contentWindow.document;
```

Больше информации об ифреймах вы можете получить в главе [](/iframes).

### IFRAME и история посещений

**`IFRAME` -- полноценное окно, поэтому навигация в нём попадает в историю посещений.**

Это означает, что при нажатии кнопки "Назад" браузер вернёт посетителя назад не в основном окне, а в ифрейме. В лучшем случае -- браузер возьмёт предыдущее состояние ифрейма из кэша и посетитель просто подумает, что кнопка не сработала. В худшем -- в ифрейм будет сделан предыдущий запрос, а это уже точно ни к чему. 

**Наши запросы в ифрейм -- служебные и для истории не предназначены. К счастью, есть ряд техник, которые позволяют обойти проблему.**

<ul>
<li>Ифрейм нужно создавать динамически, через JavaScript.
</li>
<li>Когда ифрейм уже создан, то единственный способ поменять его `src` без попадания запроса в историю посещений:

```js
 
// newSrc - новый адрес
iframeDoc.location.replace(newSrc);
```

Вы можете возразить: "но ведь `iframeDoc` не всегда доступен! `iframe` может быть с другого домена -- как быть тогда?". Ответ: вместо смены `src` этого ифрейма -- создать новый, с новым `src`.</li>
<li>POST-запросы в `iframe` всегда попадают в историю посещений.</li>
<li>... Но если `iframe` удалить, то лишняя история тоже исчезнет :). Сделать это можно по окончании запроса.</li>
</ul>

**Таким образом, общий принцип использования `IFRAME`: динамически создать, сделать запрос, удалить.**

Бывает так, что удалить по каким-то причинам нельзя, тогда возможны проблемы с историей, описанные выше.

### Функция createIframe

Приведенная ниже функция `createIframe(name, src, debug)` кросс-браузерно создаёт ифрейм с данным именем и `src`.

Аргументы:
<dl>
<dt>`name`</dt>
<dd>Имя и `id` ифрейма</dd>
<dt>`src`</dt>
<dd>Исходный адрес ифрейма. Необязательный параметр.</dd>
<dt>`debug`</dt>
<dd>Если параметр задан, то ифрейм после создания не прячется.</dd>
</dl>

```js
function createIframe(name, src, debug) {
  src = src || 'javascript:false'; // пустой src

  var tmpElem = document.createElement('div');

  // в старых IE нельзя присвоить name после создания iframe
  // поэтому создаём через innerHTML
  tmpElem.innerHTML = '<iframe name="'+name+'" id="'+name+'" src="'+src+'">';
  var iframe = tmpElem.firstChild;

  if (!debug) {
    iframe.style.display = 'none';
  }

  document.body.appendChild(iframe);

  return iframe;
}
```

Ифрейм здесь добавляется к `document.body`. Конечно, вы можете исправить этот код и добавлять его в любое другое место документа. 
 
Кстати, при вставке, если не указан `src`, тут же произойдёт событие `iframe.onload`. Пока обработчиков нет, поэтому оно будет проигнорировано.

### Функция postToIframe

Функция `postToIframe(url, data, target)` отправляет POST-запрос в ифрейм с именем `target`, на адрес `url` с данными `data`.

Аргументы:
<dl>
<dt>`url`</dt>
<dd>URL, на который отправлять запрос.</dd>
<dt>`data`</dt>
<dd>Объект содержит пары `ключ:значение` для полей формы. Значение будет приведено к строке.</dd>
<dt>`target`</dt>
<dd>Имя ифрейма, в который отправлять данные.</dd>
</dl>

```js
// Например: postToIframe('/vote.php', {mark:5}, 'frame1')

function postToIframe(url, data, target){
  var phonyForm = document.getElementById('phonyForm');
  if(!phonyForm){
    // временную форму создаем, если нет
    phonyForm = document.createElement("form");
    phonyForm.id = 'phonyForm';
    phonyForm.style.display = "none";
    phonyForm.method = "POST";        
    document.body.appendChild(phonyForm);
  }

  phonyForm.action = url;
  phonyForm.target = target;

  // заполнить форму данными из объекта
  var html = [];
  for(var key in data){
    var value = data[key].replace(/"/g, "&quot;");
    // в старых IE нельзя указать name после создания input
    // поэтому используем innerHTML вместо DOM-методов
    html.push("<input type='hidden' name=\""+key+"\" value=\""+value+"\">");
  }
  phonyForm.innerHTML = html.join('');

  phonyForm.submit();
}
```

Эта функция формирует форму динамически, но, конечно, это лишь один из возможных сценариев использования. 

В `IFRAME` можно отправлять и существующую форму, включающую файловые и другие поля. 

## Запросы GET и POST

Общий алгоритм обращения к серверу через ифрейм:

<ol>
<li>Создаём `iframe` со случайным именем `iframeName`.</li>
<li>Создаём в основном окне объект `CallbackRegistry`, в котором в `CallbackRegistry[iframeName]` сохраняем функцию, которая будет обрабатывать результат.</li>
<li>Отправляем GET или POST-запрос в него.</li>
<li>Сервер отвечает как-то так:

```html
<script>
  parent.CallbackRegistry[window.name]({данные});
</script>
```

..То есть, вызывает из основного окна функцию обработки (`window.name` в ифрейме -- его имя).</li>
<li>Дополнительно нужен обработчик `iframe.onload` -- он сработает и проверит, выполнилась ли функция `CallbackRegistry[window.name]`. Если нет, значит какая-то ошибка. Сервер при нормальном потоке выполнения всегда отвечает её вызовом.</li>
</ol>

Подробнее можно понять процесс, взглянув на код.

Мы будем использовать в нём две функции -- одну для GET, другую -- для POST:
<ul>
<li>`iframeGet(url, onSuccess, onError)` -- для GET-запросов на `url`. При успешном запросе вызывается `onSuccess(result)`, при неуспешном: `onError()`.</li>
<li>`iframePost(url, data, onSuccess, onError)` -- для POST-запросов на `url`. Значением `data` должен быть объект `ключ:значение` для пересылаемых данных, он конвертируется в поля формы.</li>
</ul>

```js
//+ src="samedomain.js"
```

Код с серверной стороны (важна лишь выделенная часть):

```js
res.write(200, {'Cache-Control': 'no-cache'});

var data = JSON.stringify(data из POST или дата сервера);

*!*
res.end('<script>parent.CallbackRegistry[window.name]('+data+')</script>');
*/!*
```

Пример в действии, возвращающий дату сервера при GET и клиента при POST:

[iframe src="samedomain" border="1" link]


**Прямой вызов функции внешнего окна из ифрейма отлично работает, потому что они с одного домена.**

..Но есть и способы делать запросы на другой домен при помощи ифреймов. Мы рассмотрим их в следующей главе.
[head]
<script src="/files/tutorial/ajax/script/scriptRequest.js"></script>
[/head]