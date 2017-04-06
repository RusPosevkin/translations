# Как написать ваш собственный виртуальный DOM
*Перевод статьи [deathmood](https://twitter.com/_deathmood): [How to write your own Virtual DOM](https://medium.com/@deathmood/how-to-write-your-own-virtual-dom-ee74acc13060)*

Есть две вещи, которые вам необходимо знать для написания вашего собственного виртуального DOM. Вам даже не нужно погружаться в исходный код React или в исходный код любой другой имплементации виртуального DOM. Они такие большие и комплексные, но в действительности основная часть виртуального DOM может быть написана меньше чем за 50 строк кода. 50. Строк. Кода. !!!

Здесь заложено две концепции:
* Виртуальный DOM - любое представление настоящего DOM
* Когда мы что-то меняем в нашем виртуальном DOM дереве, мы создаем новое виртуальное дерево. Алгоритм сравнивает эти два дерева (старое и новое), находит разницу и вносит только необходимые минимальные изменения в настоящий DOM, чтобы он соответствовал виртуальному.

Это все! Давайте углубимся в каждую из этих концепций.

## Представление нашего DOM дерева
Хорошо, в первую очередь нам нужно как-то хранить наше DOM дерево в памяти. И мы можем сделать это с помощью простых JS объектов. Предположим, у нас есть это дерево:

```html
<ul class="list">
    <li>item 1</li>
    <li>item 2</li>
</ul>
```

Выглядит просто, не так ли? Как мы можем это представить с помощью простых JS объектов:

```js
{
  type: 'ul', props: { 'class': 'list' }, children: [
    { type: 'li', props: {}, children: ['item 1'] },
    { type: 'li', props: {}, children: ['item 2'] }
  ]
}
```

Здесь вы можете заметить две вещи:

* Мы представляем DOM элементы в объектом виде

```js
{ type: '…', props: { … }, children: [ … ] }
```

* Мы описываем текстовые ноды простыми JS строками

Но писать большие деревья таким способом довольно сложно. Так давайте напишем вспомогательную функцию, чтобы нам стало проще понимать структуру:

```js
function h(type, props, …children) {
  return { type, props, children };
}
```

Теперь мы можем описывать наше DOM дерево в таком виде:

```js
h('ul', { class: 'list' },
  h('li', {}, 'item 1'),
  h('li', {}, 'item 2'),
);
```

Выглядит намного чище, не правда ли? Но мы даже можем пойти дальше. Вы ведь слышали про JSX, не так ли? Да, я хотел бы применить его идеи тут. Так, как же он работает?

Если вы читали официальную документацию Babel к JSX [здесь](https://babeljs.io/docs/plugins/transform-react-jsx/),
вы знаете, что Babel транспилирует этот код:

```html
<ul className=”list”>
  <li>item 1</li>
  <li>item 2</li>
</ul>
```

во что-то вроде этого:

```js
React.createElement('ul', { className: 'list' },
  React.createElement('li', {}, 'item 1'),
  React.createElement('li', {}, 'item 2'),
);
```

Заметили сходство? Да, да... Если бы мы только могли просто заменить вызов `React.createElement(...)` на наш `h(...)`... Оказывается, мы можем, используя jsx pragma. Нам только требуется вставить похожую на комментарий строчку в начале нашего файла с исходным кодом.

```html
/** @jsx helper */
<ul className=”list”>
  <li>item 1</li>
  <li>item 2</li>
</ul>
```

Хорошо, на самом деле она сообщает Babel: «Эй, скомпилируй этот jsx, но вместо `React.createElement`, подставь `h`». Здесь вы можете подставить что угодно вместо `h`. И это будет скомпилированно.

Итак, мы будем писать наш DOM таким образом:

```html
/** @jsx h */
const a = (
  <ul className=”list”>
    <li>item 1</li>
    <li>item 2</li>
  </ul>
);
```

И это будет скомпилированно Babel в такой код:

```js
const a = (
  h(‘ul’, { className: ‘list’ },
    h(‘li’, {}, ‘item 1’),
    h(‘li’, {}, ‘item 2’),
  );
);
```

Выполнение функции `h` вернет простой JS объект - наше представление виртуального DOM.

```js
const a = (
  { type: ‘ul’, props: { className: ‘list’ }, children: [
    { type: ‘li’, props: {}, children: [‘item 1’] },
    { type: ‘li’, props: {}, children: [‘item 2’] }
  ] }
);
```

Попробуйте сами на [JSFiddle](https://jsfiddle.net/deathmood/5qyLubt4/?utm_source=website&utm_medium=embed&utm_campaign=5qyLubt4) (не забудьте указать Babel в качестве языка).

## Применение нашего представления DOM
Хорошо, теперь мы имеем представление нашего DOM дерева в JS объекте, с нашей собственной структурой. Это круто, но нам нужно как-то создать настоящий DOM из него. Конечно, мы не можем просто применить наше представление к DOM.

Прежде всего давайте установим некоторые допущения и определимся с терминологией:

* Я буду писать все переменные, хранящие настоящие DOM ноды (элементы, текстовые ноды), начиная с `$`, таким образом **$parent** будет настоящим DOM элементом
* Виртуальное DOM представление будет храниться в переменной с именем **node**
* Как и в React, у вас может быть только одна **корневая нода**, все остальные ноды будут внутри

Теперь, когда мы со всем этим разобрались, давайте напишем функцию **createElement(…)**, которая возьмет виртуальную DOM ноду и вернет настоящую DOM ноду. Забудьте пока о `props` и `children`, мы займемся ими позже:

```js
function createElement(node) {
  if (typeof node === ‘string’) {
    return document.createTextNode(node);
  }
  return document.createElement(node.type);
}
```

Содержание функции такое, потому что у нас могут быть и **текстовые ноды**, являющиеся простыми JS строками, и **элементы**, представляющие собой JS объекты такого вида:

```js
{ type: ‘…’, props: { … }, children: [ … ] }
```

Таким образом, мы можем передавать в нее как виртуальные текстовые ноды, так и ноды виртуальных элементов, и это будет работать.

Теперь давайте подумаем о детях: каждый из них также является текстовой нодой или элементом. Поэтому они также могут быть созданы с помощью нашей функции **createElement(...)**. Вы чувствуете это? Попахивает рекурсией :)) Итак, мы можем вызвать **createElement(...)** для каждого из дочерних элементов, а затем **appendChild()** их в наш элемент следующим образом:

```js
function createElement(node) {
  if (typeof node === ‘string’) {
    return document.createTextNode(node);
  }
  const $el = document.createElement(node.type);
  node.children
    .map(createElement)
    .forEach($el.appendChild.bind($el));
  return $el;
}
```

Вау, выглядит отлично. Давайте пока оставим в стороне **props** ноды. Мы поговорим о них позже. Они не нужны нам для базового понимания концепций виртуального DOM и только добавят сложности.

Давайте пойдем дальше и попробую это в [JSFiddle](https://jsfiddle.net/deathmood/cL0Lc7au/?utm_source=website&utm_medium=embed&utm_campaign=cL0Lc7au).

## Обработка изменений
Хорошо, теперь мы можем превратить наш виртуальный DOM в настоящий DOM, пора подумать про сравнение наших виртуальных деревьев.
Нам нужно написать алгоритм, который будет сравнивать два виртуальных дерева
— старое и новое — и делать только необходимые изменения в настоящем DOM.

Как сравнить деревья? Нам необходимо обработать следующие ситуации:

* Если в определенном месте нет старой ноды: нода была добавлена и нам необходимо использовать **appendChild(…)**

![appendChild](https://cdn-images-1.medium.com/max/800/1*GFUWrX6pBgiDQ5Z-IvzjUw.png "appendChild")

* Нет новой ноды в определенном месте: эта нода была удалена и нам нужно использовать **removeChild(…)**

![removeChild](https://cdn-images-1.medium.com/max/800/1*VRoYwAeWPF0jbiWXsKb2HA.png "removeChild")

* Другая нода в этом месте: нода изменилась и нам нужно использовать **replaceChild(…)**

![replaceChild](https://cdn-images-1.medium.com/max/800/1*6iQYEH0APjbuPvYmnD7Qlw.png "replaceChild")

* Ноды те же: нам нужно идти глубже и сравнивать дочерние ноды

![](https://cdn-images-1.medium.com/max/800/1*x1Eq-uuqgL0z9d9qn_opww.png "")

Хорошо, давайте напишем функцию **updateElement(...)**, принимающую три параметра: **$parent**, **newNode** и **oldNode**.
**$parent** - родительский элемент реального DOM нашей виртуальной ноды. Сейчас мы видим, как обработать все ситуации, описанные выше.

### Если нет старой ноды
Хорошо, это довольно просто, я даже не буду комментировать:

```js
function updateElement($parent, newNode, oldNode) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  }
}
```

### Если нет новой ноды

Тут у нас появляется проблема: если нет ноды в определенном месте в новом виртуальном дереве, мы должны удалить ее из реального DOM. Но как это сделать? Мы знаем родительский элемент (он передается в функцию) и поэтому мы должны вызывать **$parent.removeChild(...)** и передавать ссылку на реальный элемент DOM. Но у нас его нет. Если бы мы знали положение нашей ноды в родителе, мы могли бы получить его ссылку с помощью **$parent.childNodes[index],** где index - позиция нашей ноды в родительском элементе.

Хорошо, давайте предположим, что этот index будет передаваться нашей функции (и это действительно произойдет, как вы увидите позже), тогда наш код будет таким:

```js
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  }
}
```

### Нода изменилась
Прежде всего нам необходимо написать функцию, сравнивающую две ноды и сообщающую нам, если узел действительно изменился. Мы должны учитывать, что это могут быть как элементы, так и текстовые ноды:

```js
function changed(node1, node2) {
  return typeof node1 !== typeof node2 ||
         typeof node1 === ‘string’ && node1 !== node2 ||
         node1.type !== node2.type
}
```

И теперь, имея **index** текущей ноды родителя, мы можем легко заменить ее на вновь созданную ноду:

```js
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  } else if (changed(newNode, oldNode)) {
    $parent.replaceChild(
      createElement(newNode),
      $parent.childNodes[index]
    );
  }
}
```

## Сравнение дочерних элементов
И последнее, но не менее важное: мы должны пройти каждый дочерний элемент на обоих нодах, сравнивать их и вызвать **updateElement(…)** для каждого из них. Да, снова рекурсия.

Но есть несколько вещей, требующие рассмотрения перед написанием кода:

* Мы должны сравнивать детей, если нода является элементом (текстовые ноды не могут иметь детей)
* Сейчас мы передаем ссылку на текущую ноду как на родителя
* Мы должны сравнить всех детей один за одним, даже если в какой-то момент мы получим `undefined` (это нормально, наша функция умеет обрабатывать это)
* И, наконец, index - это просто указатель на дочернюю ноду в массиве `children`

```js
function updateElement($parent, newNode, oldNode, index = 0) {
  if (!oldNode) {
    $parent.appendChild(
      createElement(newNode)
    );
  } else if (!newNode) {
    $parent.removeChild(
      $parent.childNodes[index]
    );
  } else if (changed(newNode, oldNode)) {
    $parent.replaceChild(
      createElement(newNode),
      $parent.childNodes[index]
    );
  } else if (newNode.type) {
    const newLength = newNode.children.length;
    const oldLength = oldNode.children.length;
    for (let i = 0; i < newLength || i < oldLength; i++) {
      updateElement(
        $parent.childNodes[index],
        newNode.children[i],
        oldNode.children[i],
        i
      );
    }
  }
}
```

## Собираем все вместе
Да, это оно. Мы добрались. Я поместил весь код в [JSFiddle](https://jsfiddle.net/deathmood/0htedLra/?utm_source=website&utm_medium=embed&utm_campaign=0htedLra) и реализация действительно составляет 50 LOC, как я вам и обещал.  Можете пойти и поиграть с ним.

Откройте инструменты разработчика и посмотрите как происходят изменения при нажатии кнопки `Reload`.
![reload](https://cdn-images-1.medium.com/max/800/1*e1s_Zc_fVxL3i0un2ZNEtg.gif '')

## Заключение
Поздравляю! Мы сделали это. Мы написали свою реализацию виртуального DOM. И он работает. Я надеюсь, что прочитав эту статью, вы поняли основные концепции и то, как виртуальный DOM должен работать и как React работает под капотом.

Однако, есть вещи, которые мы не осветили здесь (я постараюсь раскрыть их в будущих статьях):

* Установка атрибутов элементов и их сравнение и обновление
* Добавление обработчиков событий для наших элементов
* Возможность нашего виртуального DOM работать с компонентами, как React
* Получение ссылок на реальные DOM ноды
* Использование виртуального DOM с библиотеками, которые непосредственно мутируют настоящий DOM: jQuery и ее плагины
* И многое другое…

Вторая статья о работе с атрибутами и событиями в виртуальном DOM [здесь](https://medium.com/@deathmood/write-your-virtual-dom-2-props-events-a957608f5c76).

- - - -

*Читайте нас на [Медиуме](https://medium.com/devschacht), контрибьютьте на [Гитхабе](https://github.com/devSchacht), общайтесь в [группе Телеграма](https://t.me/devSchacht), следите в [Твиттере](https://twitter.com/DevSchacht) и [канале Телеграма](https://t.me/devSchachtChannel), рекомендуйте в [VK](https://vk.com/devschacht) и [Facebook](https://www.facebook.com/devSchacht). Скоро подъедет подкаст, не теряйтесь.*

[Ссылка на medium](https://medium.com/p/c166b56cf01f)