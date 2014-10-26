# Фреймворк Class.extend 

Можно использовать прототипное наследование и не повторять `Rabbit.prototype.method = ...` при определении каждого метода, не иметь проблем с конструкторами и так далее.

Для этого используют ООП-фреймворк -- библиотеку, в которой об этом подумали "за нас".

В этой главе мы рассмотрим один из таких фреймворков:
<ul>
<li>С удобным синтаксисом наследования.</li>
<li>С вызовом родительских методов.</li>
<li>С поддержкой статических методов и *примесей*.</li>
</ul>

Можно сказать, что фреймворк представляет собой "синтаксический сахар" к наследованию на классах.
[cut]

Оригинальный код этого фреймворка были предложены Джоном Ресигом: [Simple JavaScript Inheritance](http://ejohn.org/blog/simple-javascript-inheritance/), но подход это не новый, его варианты используются во многих фреймворках и знакомство с ним будет очень кстати.

Полный код фреймворка: [class-extend.js](http://js.cx/libs/class-extend.js). Он содержит много комментариев, чтобы было проще его понять, но смотреть его лучше после того, как ознакомитесь с возможностями.

## Создание класса

Итак, начнём. 

Фреймворк предоставляет всего один метод: `Class.extend`.

Чтобы представлять себе, как выглядит класс "на фреймворке", взглянем на рабочий пример:

```js
//+ run
// Объявление класса Animal
var Animal = *!*Class.extend*/!*({

  init: function(name) {
    this.name = name;
  },

  run: function() {
    alert(this.name + " бежит!");
  }

});

// Создать (вызовется `init`)
var animal = new Animal("Зверь");

// Вызвать метод
animal.run(); // "Зверь бежит!"
```

Готово, создан класс `Animal`. 

Внутри `Class.extend(props)` делает следующее:

<ul>
<li>Создаётся объект, в который копируются свойства и методы из `prop`. Это будет прототип.</li>
<li>Объявляется функция, которая вызывает `this.init`, в её `prototype` записывается созданный прототип.</li>
<li>Эта функция возвращается, она и есть конструктор `Animal`.</li>
</ul>

Как видим, всё весьма просто.

Но фреймворк этим не ограничивается и добавляет ряд других интересных возможностей.

## Статические свойства 

У метода `Class.extend` есть и второй, необязательный аргумент: объект `staticProps`. 

Если он есть, то его свойства копируются в саму функцию-конструктор.

Например:

```js
//+ run
// Объявить класс Animal
var Animal = Class.extend({

  init: function(name){
    this.name = name;
  },

  toString: function(){
    return this.name;
  }

}, 
*!*
{ // статические свойства
  compare: function(a, b) { 
    return a.name - b.name;
  }
});
*/!*

var arr = [new Animal('Зорька'), new Animal('Бурёнка')]

*!*
arr.sort(Animal.compare);
*/!*

alert(arr); // Бурёнка, Зорька
```

## Наследование

**Метод `extend` копируется в создаваемые классы.** 

Поэтому его можно вызывать на любом конструкторе, чтобы создать ему класс-наследник.

Например, создадим `Rabbit`, наследующий от `Animal`:

```js
//+ run
// Создать Animal, всё как обычно
var Animal = Class.extend({
  init: function(name) { 
    this.name = name; 
  },
  run: function() { 
    alert(this.name + ' бежит!'); 
  }
});

// Объявить класс Rabbit, *!*наследующий*/!* от Animal
var Rabbit = *!*Animal.extend*/!*({

  init: function(name) {
*!*
    this._super(name); // вызвать родительский init(name)
*/!*
  },

  run: function() {
    this._super();     // вызвать родительский run
    alert('..и мощно прыгает за морковкой!');
  }

});

*!*
var rabbit = new Rabbit("Кроль");
rabbit.run(); // "Кроль бежит!", затем "..и мощно прыгает за морковкой!"
*/!*
```

## Метод this._super

В коде выше появился ещё один замечательный метод: `this._super`.

**Вызов `this._super(аргументы)` вызывает метод *родительского класса*, с указанными аргументами.**

То есть, здесь он запустит родительский `init(name)`:

```js
init: function(name) {
  this._super(name); // вызвать родительский init(name)
}
```

...А здесь -- родительский `run`:

```js
run: function() {
  this._super();     // вызвать родительский run
  alert('..и мощно прыгает за морковкой!');
}
```

Работает это, примерно, так: когда фреймворк копирует методы в прототип, он смотрит их код, и если видит там слово `_super`, то оборачивает метод в обёртку, которая ставит `this._super` в метод родителя, затем вызывает метод, а затем возвращает `this._super` как было ранее.

Это вызывает некоторые дополнительные расходы при объявлении, так как чтобы проверить, есть ли обращение к `_super`, фреймворк при копировании методов преобразует их через `toString` в строку и ищет в ней обращение.

Как правило, эти расходы несущественны, если нужно их минимизировать  -- не составляет труда изъять эту возможность из фреймворка или учесть в инструментах сжатия (минификации) кода.

Кстати, примерно это минификатор Google Closure Compiler, когда сжимает код, написанный на "дружащей" с ним Google Closure Library. 


## Примеси [#mixins]

Согласно теории ООП, *примесь* (англ. mixin) -- класс, реализующий какое-либо чётко выделенное поведение, который не предназначен для порождения самостоятельно используемых объектов, а используется для *уточнения* поведения других классов.

Иными словами, *примесь* позволяет легко добавить в существующий класс новые возможности, например:
<ul>
<li>Публикация событий и подписка на них.</li>
<li>Работ c шаблонизатором.</li>
<li>... любое поведение, дополняющее объект.</li>
</ul>

**Как правило, примесь реализуется в виде объекта, свойства которого копируются в прототип.**

Например, напишем примесь `EventMixin` для работы с событиями. Она будет содержать три метода  -- `on/off` (подписка) и `trigger` (генерация события):

```js
var EventMixin = {

  on: function (eventName, handler) {
    if (!this._eventHandlers) this._eventHandlers = {};
    if (!this._eventHandlers[eventName]) {
      this._eventHandlers[eventName] = [];
    }

    this._eventHandlers[eventName].push(handler);
  },

  off: function(eventName, handler) {
    ...
  },

  trigger: function (eventName, args) {
    if (!this.eventHandlers || !this._eventHandlers[eventName]) {
      return; 
    }

    var handlers = this._eventHandlers[eventName];
    for (var i = 0; i < handlers.length; i++) {
      handlers[i].apply(this, args);
    }
  }
  
};
```

Скопировав свойства из `EventMixin` в любой объект, мы дадим ему возможность генерировать события (`trigger`) и  подписываться на них (`on/off`).

Чтобы было проще, во фреймворк добавлена возможность указания примесей при объявлении класса.

**Для добавления примесей у метода `Class.extend` существует синтаксис с первым аргументом-массивом:**
<ul>
<li>**`Class.extend([mixin1, mixin2...], props, staticProps)`.**</li>
</ul>

Если первый аргумент -- массив, то его элементы `mixin1, mixin2..` записываются в прототип по очереди, перед `props`, примерно так:

```js
for(var key in mixin1) prototype[key] = mixin1[key];
for(var key in mixin2) prototype[key] = mixin2[key];
...
for(var key in props) prototype[key] = props[key];
```

При этом, если названия методов совпадают, то последующий затрёт предыдущий, так как в объекте может быть только одно свойство с данным названием. Впрочем, обычно такого не происходит, т.к. примеси проектируются так, чтобы их методы были уникальными и ни с чем не конфликтовали.

Применение:

```js
*!*
var Rabbit = Class.extend( [ EventMixin ], {
*/!*

  /* свойства и методы для Rabbit */

});

var rabbit = new Rabbit();

*!*rabbit.on*/!*("jump", function() { // повесить функцию на событие jump
  alert("jump &-@!");
});

*!*rabbit.trigger*/!*('jump'); // alert сработает!
```

Примеси могут быть самыми разными. Например `TemplateMixin` для работы с шаблонами:

```js
Rabbit = Class.extend([EventMixin, TemplateMixin], {

  /* Теперь Rabbit умеет использовать события и шаблоны */

});
```

Красиво, не правда ли? Всего лишь указали одну-другую примесь и объект уже всё умеет!

Примеси могут задавать и многое другое, например автоматически подписывать компонент на стандартные события, добавлять AJAX-функционал и т.п.

## Итого

<ol>
<li>**Фреймворк имеет основной метод `Class.extend` с несколькими вариациями:**
<ul>
<li>`Class.extend(props)` -- просто класс с прототипом `props`.</li>
<li>`Class.extend(props, staticProps)` -- класс с прототипом `props` и статическими свойствами `staticProps`.</li>
<li>`Class.extend(mixins, props [, staticProps])` -- если первый аргумент массив, то он интерпретируется как примеси. Их свойства копируются в прототип перед `props`.</li>
</ul>
</li>
<li>**У созданных этим методом классов также есть `extend` для продолжения наследования.**</li>
<li>**Методы родителя можно вызвать при помощи `this._super(...)`.**</li>
</ol>

Плюсы и минусы:
[compare]
+Такой фреймворк удобен потому, что класс можно задать одним вызовом `Class.extend`, с читаемым синтаксисом, удобным наследованием и вызовом родительских методов.
-Редакторы и IDE, как правило, не понимают такой синтаксис, а значит, не предоставляют автодополнение. При этом они обычно понимают объявление методов через явную запись в объект или в прототип. 
-Есть некоторые дополнительные расходы, связанные с реализацией `_super`. Если они критичны, то их можно избежать.[/compare]

То, как работает фреймворк, подробно описано в комментариях: [class-extend.js](http://js.cx/libs/class-extend.js).
[head]
class-extend.js
[/head]