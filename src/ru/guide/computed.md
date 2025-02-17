# Вычисляемые свойства и наблюдатели

## Вычисляемые свойства

Выражения внутри шаблона удобны, но предназначены для простых операций. Большое количество логики в шаблоне сделает его раздутым и сложным для поддержки. Например, если есть объект с вложенным массивом:

```js
Vue.createApp({
  data() {
    return {
      author: {
        name: 'John Doe',
        books: [
          'Vue 2 - Advanced Guide',
          'Vue 3 - Basic Guide',
          'Vue 4 - The Mystery'
        ]
      }
    }
  }
})
```

И потребуется отображать разные сообщения, в зависимости от того, есть ли у `author` какие-то книги или нет:

```html
<div id="computed-basics">
  <p>Есть опубликованные книги:</p>
  <span>{{ author.books.length > 0 ? 'Да' : 'Нет' }}</span>
</div>
```

В таком случае шаблон уже не будет простым и декларативным. Потребуется взглянуть на него, прежде чем понять, что он выполняет вычисления в зависимости от `author.books`. Проблема усугубится, если подобные вычисления в шаблоне потребуются не один раз.

Поэтому для сложной логики, включающей реактивные данные, следует использовать **вычисляемые свойства**.

### Простой пример

```html
<div id="computed-basics">
  <p>Есть опубликованные книги:</p>
  <span>{{ publishedBooksMessage }}</span>
</div>
```

```js
Vue.createApp({
  data() {
    return {
      author: {
        name: 'John Doe',
        books: [
          'Vue 2 - Advanced Guide',
          'Vue 3 - Basic Guide',
          'Vue 4 - The Mystery'
        ]
      }
    }
  },
  computed: {
    // геттер вычисляемого свойства
    publishedBooksMessage() {
      // `this` указывает на экземпляр vm
      return this.author.books.length > 0 ? 'Да' : 'Нет'
    }
  }
}).mount('#computed-basics')
```

Результат:

<common-codepen-snippet title="Computed basic example" slug="NWqzrjr" tab="js,result" :preview="false" />

В этом примере объявляем новое вычисляемое свойство `publishedBooksMessage`.

Попробуйте изменить в массиве `books` количество книг в `data` приложения и увидите, как обновится и значение `publishedBooksMessage`.

В шаблоне к вычисляемым свойствам можно обращаться как и к обычным свойствам. Vue знает, что `vm.publishedBooksMessage` зависит от значения `vm.author.books`, поэтому будет обновлять все привязки, которые зависят от `vm.publishedBooksMessage`, при изменениях `vm.author.books`. И самая лучшая часть — эту зависимость создали декларативно: геттер-функция вычисляемого свойства не имеет побочных эффектов, что облегчает понимание и тестирование.

### Кэширование вычисляемых свойств vs Методы

Можно заметить, что того же результата можно достичь с помощью метода в выражении:

```html
<p>{{ calculateBooksMessage() }}</p>
```

```js
// в компоненте
methods: {
  calculateBooksMessage() {
    return this.author.books.length > 0 ? 'Да' : 'Нет'
  }
}
```

Вместо вычисляемого свойства можно объявить эту же функцию в качестве метода. Для отображаемого результата эти два подхода действительно одинаковы. Однако разница заключается в том, что **вычисляемые свойства кэшируются на основе своих реактивных зависимостей.** Вычисляемое свойство будет пересчитываться только при изменении одной из своих зависимостей. А значит, пока не изменится `author.books`, любое число обращений к вычисляемому свойству `publishedBooksMessage` будет немедленно возвращать ранее вычисленный результат, без необходимости повторного запуска функции.

Это также означает, что следующее вычисляемое свойство никогда не будет обновляться, потому что `Date.now()` не является реактивной зависимостью:

```js
computed: {
  // НЕ БУДЕТ РАБОТАТЬ
  now() {
    return Date.now()
  }
}
```

Для сравнения, вызов метода **будет всегда запускать** функцию, когда будет перерисовка.

Зачем нужно кэширование? Представьте, что есть затратное вычисляемое свойство `list`, которому требуется проходить по большому массиву и выполнять различные вычисления. Далее, могут быть другие вычисляемые свойства, которые зависят от значения `list`. Без кэширования выполнять геттер `list` потребуется во много раз больше, чем это нужно! Когда же необходимо обойтись без кэширования — стоит использовать `methods`.

### Сеттер вычисляемого свойства

Вычисляемые свойства по умолчанию состоят только из геттера и поэтому доступны только для чтения. Но при необходимости можно также определить и сеттер:

```js
// ...
computed: {
  fullName: {
    // геттер (для получения значения)
    get() {
      return this.firstName + ' ' + this.lastName
    },
    // сеттер (при присвоении нового значения)
    set(newValue) {
      const names = newValue.split(' ')
      this.firstName = names[0]
      this.lastName = names[names.length - 1]
    }
  }
}
// ...
```

Теперь, при выполнении `vm.fullName = 'John Doe'` вызовется сеттер вычисляемого свойства и значения `vm.firstName` и `vm.lastName` соответственно обновлены.

## Методы-наблюдатели

Обычно вычисляемые свойства подходят в большинстве случаев, но иногда нужно отследить сам факт изменений. Поэтому Vue предоставляет ещё один способ реагировать на изменения данных с помощью опции `watch`. Это полезно, если необходимо выполнять асинхронные или затратные операции в ответ на изменение данных.

Например:

```html
<div id="watch-example">
  <p>
    Задайте вопрос, на который можно ответить «да» или «нет»:
    <input v-model="question" />
  </p>
  <p>{{ answer }}</p>
</div>
```

```html
<!-- Поскольку уже существует обширная экосистема ajax-библиотек -->
<!-- и библиотек функций общего назначения, ядро Vue может       -->
<!-- оставаться маленьким и не изобретать их заново. Кроме того, -->
<!-- это позволяет использовать только знакомые инструменты.     -->
<script src="https://cdn.jsdelivr.net/npm/axios@0.12.0/dist/axios.min.js"></script>
<script>
  const watchExampleVM = Vue.createApp({
    data() {
      return {
        question: '',
        answer: 'Вопросы обычно заканчиваются вопросительным знаком. ;-)'
      }
    },
    watch: {
      // при каждом изменении `question` эта функция будет запускаться
      question(newQuestion, oldQuestion) {
        if (newQuestion.indexOf('?') > -1) {
          this.getAnswer()
        }
      }
    },
    methods: {
      getAnswer() {
        this.answer = 'Думаю...'
        axios
          .get('https://yesno.wtf/api')
          .then(response => {
            this.answer = response.data.answer
          })
          .catch(error => {
            this.answer = 'Ошибка! Нет доступа к API. ' + error
          })
      }
    }
  }).mount('#watch-example')
</script>
```

Результат:

<common-codepen-snippet title="Простой пример watch" slug="GRJGqXp" tab="result" :preview="false" />

В этом случае, использовании опции `watch` позволяет выполнить асинхронную операцию (обращение к API) и устанавливает условие для выполнения этой операции. Ничего из этого нельзя сделать с помощью вычисляемых свойств.

Кроме опции `watch`, можно использовать императивную запись с помощью [vm.$watch API](../api/instance-methods.md#watch).

### Вычисляемые свойства vs Методы-наблюдатели

Vue предоставляет универсальный способ для наблюдения и реагирования на изменения данных в текущем активном экземпляре: **свойства watch**. Когда есть данные, которые нужно изменять на основе других данных, кажется удобным реализовать всё через `watch` (особенно, если ранее работали с AngularJS). Однако, чаще всего уместнее использовать вычисляемые свойства, чем вызовы методов-наблюдателей `watch`. Рассмотрим пример:

```html
<div id="demo">{{ fullName }}</div>
```

```js
const vm = Vue.createApp({
  data() {
    return {
      firstName: 'Foo',
      lastName: 'Bar',
      fullName: 'Foo Bar'
    }
  },
  watch: {
    firstName(val) {
      this.fullName = val + ' ' + this.lastName
    },
    lastName(val) {
      this.fullName = this.firstName + ' ' + val
    }
  }
}).mount('#demo')
```

Код выше является императивным и повторяющимся. Сравним с вычисляемым свойством:

```js
const vm = Vue.createApp({
  data() {
    return {
      firstName: 'Foo',
      lastName: 'Bar'
    }
  },
  computed: {
    fullName() {
      return this.firstName + ' ' + this.lastName
    }
  }
}).mount('#demo')
```

Значительно лучше, не так ли?
