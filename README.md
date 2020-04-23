# Веб-компоненты в реальном проекте

Всем привет! Меня зовут Артур, я работаю frontend-разработчиком в Exness. Не так давно мы перевели один из наших проектов на веб-компоненты. Расскажу с какими проблемами пришлось столкнуться, и о том, как многие концепции, к которым мы привыкли при работе с фреймворками, легко перекладываются на веб-компоненты.  
 
Забегая вперед, скажу, что реализованный проект успешно прошел тестирование на нашей широкой аудитории, а размер бандла и время загрузки удалось ощутимо сократить.
 
Предполагаю, что у вас уже есть базовое представление о технологии, но и без него будет понятно, что с веб-компонентами можно вполне удобно работать и организовывать архитектуру проекта.


## Почему веб-компоненты? 
Проект был небольшим, но требовательным к размеру бандла; основными ui-составляющими были формы. Кто-то скажет, что все можно было сделать нативным html+js, что было бы максимально легковесно. Но если говорить о поддержке и расширении проекта, отказаться от всех преимуществ компонентной разработки было бы подобно прыжку в прошлое. С другой стороны, можно было бы использовать [Svelte](https://github.com/sveltejs/svelte) или, например, [Preact](https://github.com/preactjs/preact). Но попробовать сделать все, оперируя нативными концепциями, было слишком заманчивой идеей. 


## Будущее уже настало? 
Большинство современных браузеров, включая мобильные, поддерживают веб-компоненты из коробки. Для остальных есть [полифилы](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). Примечательно, что для полифилов есть умный [лоадер](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~2kB), который не загружает полифил определенного функционала, если для него есть нативная поддержка браузером — таким образом, в современные браузеры ничего не загружается. Полифилы заявляют поддержку вплоть до IE11 и старых версий Edge. К счастью, мы не поддерживаем IE в наших проектах, в Edge действительно все работает. Также все работает в китайских UC Browser и QQ Browser, включая их мобильные версии.

> Небольшие ограничения по работе полифилов:
> - Полифильный тег ```<slot></slot>``` для IE11 & Edge не участвуют во всплытии событий;
> - Этот набор полифилов не предоставляет функционал для расширения встроенных html-элементов. Об этом подробнее ниже.


## LitElement
«Как много boilerplate-кода для создания компонентов и реактивной работы с пропсами! Нужно ускорить процесс и написать класс, расширяющий HTMLElement и реализующий этот код» — такой была первая мысль, которая возникла при погружении в веб-компоненты. Писать, конечно, ничего не пришлось — эту работу берет на себя класс [LitElement](https://lit-element.polymer-project.org/) (~7kb). А использующийся в нем [lit-html](https://lit-html.polymer-project.org/) предоставляет удобный и оптимизированный механизм рендеринга и упрощает привязку данных и подписку на события. 

Кроме того, LitElement расширяет стандартный **жизненный цикл веб-компонентов** удобными [дополнениями](https://lit-element.polymer-project.org/guide/lifecycle), делая его во многом схожим с жизненным циклом компонентов популярных фреймворков.

LitElement — это еще одна зависимость, кто-то даже скажет «еще один фреймворк». Но я осознанно *решился на эти плюс 7kB,* потому что LitElement смотрится как один из вариантов естественной надстройки над существующим нативным API. Она так или иначе присутствовала бы в проекте хотя бы для реализации boilerplate-кода.


## Функциональные компоненты-шаблоны
К теме статьи это имеет косвенное отношение, но lit-html в свои ~3.5kB вмещает очень удобную возможность описания интерфейса с помощью функций. Причем обновление DOM-структур таких функций-компонентов происходит оптимизировано: рендерятся только те блоки, значения которых изменились с предыдущего рендера. В некоторых случаях и при должной находчивости весь интерфейс можно описать только функциями, декораторами и директивами (о них чуть ниже):

```js
import { html, render } from 'lit-html'

const ui = data => html`...${data}...`

render(ui('Hello!'), document.body)
```

При этом в одних шаблонах можно использовать другие:
```js 
const myHeader = html`<h1>Header</h1>`
const myPage = html`
  ${myHeader}
  <div>Here's my main page.</div>
`
```

В других случаях можно придумать обертку для создания кастомных элементов из таких функций: 
```js
const defineFxComponent = (tagName, FxComponent, Parent = LitElement) => {
  const Component = class extends Parent {
    render() {
      return FxComponent(this.data)
    }
  }
  customElements.define(tagName, Component)
}

defineFxComponent('custom-ui', ui)

render(html`<custom-ui .data="Hello!"></custom-ui>`, document.body)
```

Я не буду останавливаться на удобствах создания шаблонов, стилизации, передачи атрибутов, привязки данных и подписки на события, условном и цикличном рендере при работе с lit-html. Все это подробно описано в [документации](https://lit-html.polymer-project.org/guide/writing-templates). Остановлюсь на том, что можно упустить при беглом взгляде на руководство, но может быть полезным.

### svg
Самая скрытая из таких функций, о которой нет упоминаний в руководстве (но есть в [API](https://lit-html.polymer-project.org/api/modules/lit_html.html#svg)) — это тег ```svg`` ```. Как несложно догадаться, он используется для работы с векторной графикой, при работе с которой через ```html`` ``` могут возникать некоторые проблемы. У меня они возникли, когда я пытался передать TemplateResult (это результат выполнения ```html`` ```) вовнутрь моего компонента иконки — появлялись ненужные закрывающие теги, и графика не отрисовывалась. При использовании ```svg`` ``` и передачи SVGTemplateResult все встало на свои места. 

### Директивы
Директивы — это функции, которые описывают то, каким образом будет выводиться их содержимое. В lit-html для хранения и вывода значений для представления DOM используются классы, реализующие интерфейс Part. Именно интерфейс Part обеспечивает умный рендеринг, который обновляет только то, что изменилось, а директивы — это способ получить доступ к этому механизму и влиять на него. 

Директивы могут быть одного из пяти типов, каждая из которых имеет доступ к соответствующей реализации Part: 
- Для вывода контента (NodePart);
- Для передачи атрибута (AttributePart);
- Для передачи булевого атрибута (BooleanAttributePart);
- Для привязки данных или передачи свойства (PropertyPart);
- Для подписки на события (EventPart).

Каждый тип директив может быть использован только в подходящем для него месте и не имеет значения в других местах.

Директива хранит в себе значение `value` — это то, что было выведено на ее месте при последнем рендере. Для установки нового значения существует функция `setValue()`. Для принудительного обновления значений в DOM после того, как рендер был завершен, используется функция `commit()` (полезно при асинхронных действиях).

Пример кастомной директивы (имеющей доступ к классу NodePart — для вывода контента), которая хранит и выводит количество рендеров: 
```js
import { directive } from 'lit-html'

const renderCounter = directive(() => part =>
  part.setValue(part.value === undefined ? 0 : part.value + 1)
)
```

Lit-html имеет полезный [набор встроенных директив](https://lit-html.polymer-project.org/guide/template-reference#built-in-directives). Есть аналоги некоторых реактовских хуков, функции для работы со стилями и классами как с объектами, функции асинхронного обновления контента, оптимизации, небезопасной установки html и др. 

Рядом с директивой можно хранить более сложный state для использования внутри директивы. Пример [тут](https://lit-html.polymer-project.org/guide/creating-directives#maintaining-state). 

Кастомные директивы и декораторы можно также использовать как аналог компонентов высшего порядка. При таком подходе необходимо самостоятельно позаботиться о реактивности внутри директивы. Пример [тут](https://github.com/jmas/lit-redux/blob/master/index.js). 

### shady-render
Если вы создаете базовый класс с использованием lit-html и Shadow DOM, вам необходимы будут полифилы для старых браузеров. У lit-html есть отдельный [модуль shady-render](https://lit-html.polymer-project.org/guide/styling-templates#polyfilled-shadow-dom:-shadydom-and-shadycss), который интегрирует эти полифилы. 



## Компоненты высшего порядка
HOC — мощный паттерн, часто используемый при работе с React или Vue. При его использовании композиция компонентов становится простой и лаконичной, и хотелось бы использовать какой-то его аналог при работе с веб-компонентами. Так как веб-компоненты это классы, то для себя в качестве аналога HOC я решил использовать функции, которые бы возвращали расширение класса, переданного им в качестве параметра. 

### Расширение функционала
В проекте мне необходим был redux, поэтому в качестве примера рассмотрим [коннектор для него](https://github.com/realiarthur/lite-redux). Ниже представлен код декоратора, принимающего store и возвращающего стандартный коннектор redux. Внутри класса происходит накопление mapStateToProps из всей цепочки наследования (для тех случаев, если в ней будет HOC, который также общается с redux), чтобы в дальнейшем, когда компонент будет встроен в DOM, одним колбеком подписать их все на изменение состояния redux. При удалении компонента из DOM эта подписка удаляется.

```js
import { bindActionCreators } from 'redux'

export default store => (mapStateToProps, mapDispatchToProps) => Component =>
  class Connect extends Component {
    constructor(props) {
      super(props)
      this._getPropsFromStore = this._getPropsFromStore.bind(this)
      this._getInheritChainProps = this._getInheritChainProps.bind(this)

      // Накопление mapStateToProps
      this._inheritChainProps = (this._inheritChainProps || []).concat(
        mapStateToProps
      )
    }

    // Функция для получения данных из store
    _getPropsFromStore(mapStateToProps) {
      if (!mapStateToProps) return
      const state = store.getState()
      const props = mapStateToProps(state)

      for (const prop in props) {
        this[prop] = props[prop]
      }
    }

    // Callback для подписки на изменение store, который вызовет все mapStateToProps из цепочки наследования
    _getInheritChainProps() {
      this._inheritChainProps.forEach(i => this._getPropsFromStore(i))
    }

    connectedCallback() {
      this._getPropsFromStore(mapStateToProps)

      this._unsubscriber = store.subscribe(this._getInheritChainProps)

      if (mapDispatchToProps) {
        const dispatchers =
          typeof mapDispatchToProps === 'function'
            ? mapDispatchToProps(store.dispatch)
            : mapDispatchToProps
        for (const dispatcher in dispatchers) {
          typeof mapDispatchToProps === 'function'
            ? (this[dispatcher] = dispatchers[dispatcher])
            : (this[dispatcher] = bindActionCreators(
                dispatchers[dispatcher],
                store.dispatch,
                () => store.getState()
              ))
        }
      }

      super.connectedCallback()
    }

    disconnectedCallback() {
      // Удаление подписки на изменение store
      this._unsubscriber()
      super.disconnectedCallback()
    }
  }
```

Удобнее всего использовать этот метод при инициализации store для создания и экспорта обычного коннектора, который можно использовать в качестве компонента высшего порядка:

```js
// store.js
import { createStore } from 'redux'
import makeConnect from 'lite-redux'
import reducer from './reducer'

const store = createStore(reducer)

export default store

// Создание стандартного коннектора
export const connect = makeConnect(store)
```

```js
// Component.js
import { connect } from './store'

class Component extends WhatEver {
  /* ... */
}

export default connect(mapStateToProps, mapDispatchToProps)(Component)
```

### Расширение отображения и наблюдаемых свойств
Во многих случаях кроме расширения функционала компонента требуется обернуть его отображение. Для этого удобно использовать функцию рендеринга расширяемого компонента. Кроме того, бывает полезно расширить список наблюдаемых свойств для обеспечения реактивности: `get observedAttributes()` для веб-компонентов или `get properties()` для LitElement. Для иллюстрации приведу пример поля ввода пароля, которое расширяет переданный ей компонент текстового поля ввода:

```js
const withPassword = Component =>
  class PasswordInput extends Component {
    static get properties() {
      return {
        // Предполагается что super.properties уже содержит свойство type
        ...super.properties,
        addonIcon: { type: String }
      }
    }

    constructor(props) {
      super(props)
      this.type = 'password'
      this.addonIcon = 'invisible'
    }

    setType(e) {
      this.type = this.type === 'text' ? 'password' : 'text'
      this.addonIcon = this.type === 'password' ? 'invisible' : 'visible'
    }

    render() {
      return html`
        <div class="with-addon">
          <!-- Отображение расширяемого класса -->
          ${super.render()}
          <div @click=${this.setType}>
            <custom-icon icon=${this.addonIcon}></custom-icon>
          </div>
        </div>
      `
    }
  }

customElements.define('password-input', withPassword(TextInput))
```

Внимание здесь хотелось бы обратить на строку `...super.properties` в методе `get properties()`, которая позволяет не определять уже описанные в расширяемом компоненте свойства. И на строку `super.render()` в методе `render`, которая выводит в указанном месте разметки отображение расширяемого компонента.


При использовании такого подхода для реализации аналога HOC стоит соблюдать некоторые меры предосторожности: 
- Быть внимательным при именовании свойств и методов, так как можно переопределить их в наследуемом компоненте;
- Помнить, что при передаче метода класса в качестве колбека на какое-либо событие есть вероятность, что этот метод будет переопределен где-то в цепочке наследования, и на событие окажется подписанным только последний из HOC'ов, а не оба;
- Стараться максимально наглядно подписываться на изменения свойств, переданных из HOC.


## Как я отказался от Shadow DOM и расширения встроенных элементов
Как я уже говорил, основными ui-составляющими моего проекта являются формы и различные поля ввода. Для удобства хотелось бы использовать компонент, предоставляющий удобные инструменты для работы с формой и реализующий необходимый функционал: хранение и обновление состояния (values, errors, touched), валидация, обработка submit, reset. 

Этот функционал не имеет отношения к теме статьи, но хотелось бы поговорить о том, во что его обернуть. Забегая вперед, скажу, что прежде чем сделать рабочий вариант, я попробовал три упаковки: стандартный веб-компонент, расширение встроенного элемента ```<form>``` и компонент высшего порядка. И это повод для небольшой истории...

### Версия 1. Веб-компоненты и Shadow DOM
Первое, что пришло мне на ум — сделать стандартные веб-компоненты полей ввода и формы, используя Shadow DOM как часть лучших практик, обернуть поля ввода в HOC для общения с формой, вставлять форму на страницу отдельным кастомным тегом. Обо всем по порядку.

С одной стороны, мне нужен был функционал тега `<form>` и его класса HTMLFormElement (например, вызов submit при нажатии на Enter), но не хотелось каждый раз дописывать его в дополнение к кастомному тегу формы. С другой стороны, я не мог использовать слот, чтобы обернуть его содержимое в тег `<form>`, потому что события формы переставали работать — видимо `<form>` пока не до конца готов к слоту внутри себя. 

Решением стала передача шаблона в качестве свойства для кастомной формы. Это некий аналог передачи render-функции в React: 

```js
// Компонент формы
import { LitElement, html } from 'lit-element'

class LiteForm extends LitElement {
  /* ...функционал формы ... */

  render() {
    return html`<form @submit=${this.handleSubmit} method=${this.method}>
      ${this.formTemplate(this)}
    </form>`
  }
}

customElements.define('lite-form', LiteForm)
```

```js
// Пример формы
import { html, render } from 'lit-element'

const formTemplate = ({ values, handleBlur, handleChange, ...props }) =>
  html`<input
      .value=${values.firstName}
      @input=${handleChange}
      @blur=${handleBlur}
    />
    <input
      .value=${values.lastName}
      @input=${handleChange}
      @blur=${handleBlur}
    />
    <button type="submit">Submit</button>`

const MyForm = html`<lite-form
  method="POST"
  .formTemplate=${formTemplate}
  .onSubmit=${{/*...*/}}
  .initialValues=${{/*...*/}}
  .validationSchema=${{/*...*/}}
></lite-form>`

render(html`${MyForm}`, document.getElementById('root'))
```

Конечно, мне не хотелось каждому полю ввода передавать свойства и события формы. К тому же я хотел работать с кастомными полями ввода и упростить вывод ошибок. Поэтому мне необходимы были несколько компонентов высшего порядка для работы с формой, таких как `withField` или `withError`. 

Такая реализация требовала, чтобы HOC'и были способны самостоятельно найти свою форму или общаться с ней посредством событий. Перебрав несколько вариантов (общение через шину событий, общий контекст — все они подходили, если форма на странице только одна), я остановился на очень простом, но рабочем:

```js
// здесь константа IS_LITE_FORM — это имя булевого атрибута, который имеет каждый элемент кастомной формы
const getFormClass = element => {
  const form = element.closest(`[${IS_LITE_FORM}]`)
  if (form) return form

  const host = element.getRootNode().host
  if (!host) throw new Error('Lite-form not found')
  return host[IS_LITE_FORM] ? host : getFormClass(host)
}
```
Здесь все тривиально: рекурсивный поиск элемента с атрибутом, указывающим что это искомая форма. 

Отметить хотелось функцию [getRootNode](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode), благодаря которой поиск проходит сквозь дерево вложенных Shadow DOM — необходимая функция при решении таких специфичных задач.

С использованием `withField` я мог сильно упростить шаблон формы:
```js
const formTemplate = props =>
  html`<custom-input name="firstName"></custom-input>
    <custom-input name="lastName"></custom-input>
    <button type="submit">Submit</button>`
```

В общем, все работало отлично, но... Прежде чем рассказать, почему я отказался от Shadow DOM, еще пару слов о нем.

### Стилизация Shadow DOM извне и псевдоклассы :host и :host-context
Для кастомизации стилей компонентов из основного документа можно использовать CSS переменные. Они проникают через Shadow DOM, видны повсюду, и это вполне удобно. Но возникают ситуации, когда этого оказывается недостаточно, например, когда необходима условная стилизация:
```css
:host([rtl]) {
  text-align: right;
}

:host-context(body[dir='rtl']) .text {
  text-align: right;
}
```
Псевдокласс `:host` позволяет применить стиль к корневому элементу компонента, только если он соответствует заданному селектору. В примере текст будет выровнен по правому краю только в случае, если корневой элемент имеет атрибут `rtl`.

Псевдокласс `:host-context` позволяет понять, в каком контексте находится Shadow DOM, и реагировать на него. В примере блок с классом `.text` внутри компонента будет соответствующим образом выравнивать текст в зависимости от атрибута `dir`, установленного для `body`. 

### Всплытие событий сквозь Shadow DOM
События при всплытии сквозь Shadow DOM ведут себя несколько необычно: у них подменяется целевой элемент `target`, которым становится сам веб-компонент. Сделано это в целях поддержания инкапсуляции DOM, и это необходимо учитывать при разработке, потому что конечный `target` может не содержать всех тех свойств, которые содержал изначальный целевой элемент. 

Всплытие сквозь Shadow DOM можно регулировать свойством события `composed`. Чтобы событие всплывало сквозь Shadow DOM, оба его свойства `composed` и `bubbles` должны быть установлены в `true`. Большинство стандартных событий успешно всплывает сквозь Shadow DOM.

### Почему я отказался от Shadow DOM
Во-первых, из-за невозможности автозаполнения форм хранимыми пользовательскими данными (логин-пароль, адрес, данные кредитных карт) если форма или поля ввода скрыты за Shadow DOM. Также браузер не предлагает сохранить пользовательские данные с таких форм. В моем проекте этот недостаток (или фича, кому как ближе) стал критичным.

>  Конечно, если для передачи шаблона формы используются только слоты, и этот шаблон описан в основном документе, которому он и будет принадлежать, то автозаполнение произойдет даже при использовании Shadow DOM в компоненте формы. Но кроме этого, все компоненты, в которые вложена форма, и все поля ввода не должны использовать Shadow DOM. Это настолько сильные ограничения, что проще отказаться от Shadow DOM.

Во-вторых, из-за отсутствия аналога querySelector, который работал бы сквозь Shadow DOM. Для метрик и аналитики такой инструмент был бы полезен, чтобы, например в Google Tag Manager не писать длинные конструкции типа  `document.querySelector(...) .shadowRoot.querySelector(...) .shadowRoot.querySelector(...) .shadowRoot.querySelector(...)`

С отказом от Shadow DOM особых трудностей не возникло. В LitElement для этого достаточно такого кода:
```js
createRenderRoot() {
  return this
}
```
Вместе с Shadow DOM я потерял проводника всплытия события `blur` из полей ввода. Я использовал его внутри `withField`, чтобы мне не приходилось пробрасывать его из компонента вручную. Но я подписался на него на стадии погружения, и это сработало.

### Версия 2. Расширение встроенного элемента
После отказа от Shadow DOM мне показалось хорошей идеей расширить встроенный класс HTMLFormElement и тег `<form>` — верстка бы смотрелась как нативная, и при этом сохранился бы доступ ко всем событиям формы, и это требовало минимальных изменений кода:
```js
// Компонент формы
class LiteForm extends HTMLFormElement {
  connectedCallback() {
    this.addEventListener('submit', this.handleSubmit)
  }

  disconnectedCallback() {
    this.removeEventListener('submit', this.handleSubmit)
  }

  /* ...функционал формы ... */
}

customElements.define('lite-form', LiteForm, { extends: 'form' })
```
Все работало как в обычной форме, только с дополнительным функционалом:
```js
// Пример формы
const MyForm = html`<form
  method="POST"
  is="lite-form"
  .onSubmit=${{...}}
  .initialValues=${{...}}
  .validationSchema=${{...}}
>
  <custom-input name="firstName"></custom-input>
  <custom-input name="lastName"></custom-input>
  <button type="submit">Submit</button>
</form>`

render(html`${MyForm}`, document.getElementById('root'))
```
Здесь хотелось бы обратить внимание на первый аргумент в функции `customElements.define` и на атрибут `is` в элементе формы, которые указывают на то, какого «типа» будет тег. А также на третий аргумент в `customElements.define`, в котором указывается, какой именно тег будет расширяться.

Все работало отлично и прекрасно смотрелось. Но не во всех браузерах: Safari не поддерживает расширение встроенных элементов, потому что считает это небезопасным. Это также касается и браузеров для iOS (включая Chrome, Firefox и др.). Ходят слухи, что Apple возможно пересмотрит такое поведение, но в 13 версии Safari и iOS все остается как есть. И хотя есть [полифилы](https://www.npmjs.com/package/@ungap/custom-elements-builtin), я решил не пользоваться этой концепцией до тех пор, пока Safari и iOS не начнут поддерживать расширение встроенных элементов.

### Версия 3. Компонент высшего порядка
После пары неудачных попыток обернуть функционал формы я решил оставить это за ее пределами и сделать компонент высшего порядка, чтобы оборачивать им все, что понадобится. Это потребовало небольших изменений кода:
```js
// Компонент высшего порядка формы
export const withForm = ({
  onSubmit,
  initialValues,
  validationSchema,
  ...config
} = {}) => Component =>
  class LiteForm extends Component {
    connectedCallback() {
      this._onSubmit = (onSubmit || this.onSubmit || function () {}).bind(this)
      this._initialValues = initialValues || this.initialValues || {}
      this._validationSchema = validationSchema || this.validationSchema || {}
      /* ... */
      super.connectedCallback && super.connectedCallback()
    }

    /* ...функционал формы ... */
  }
```

Здесь в функции `connectedCallback()` форма принимает конфиг (onSubmit, initialValues, validationSchema, и др.) либо из аргументов, переданных в `withForm()`, либо из самого расширяемого компонента. Это позволяет оборачивать любые классы, а также строить базовые классы, которые можно использовать в верстке, передавая конфиг в ней. К слову, этим способом можно построить оба базовых класса из первых реализаций формы:

```js
// Пример базового класса из первой реализации формы
class LiteForm extends LitElement {
  render() {
    return html`<form @submit=${this.handleSubmit} method=${this.method}>
      ${this.formRender(this)}
    </form>`
  }
}

enhance = withForm()

customElements.define('lite-form', enhance(LiteForm))
```

С другой стороны, можно не создавать базовый класс формы, и оборачивать в `withForm()` конечные компоненты, содержащие шаблоны форм:

```js
// Пример формы
class UserForm extends LitElement {
  render() {
    return html`
      <form method="POST" @submit=${this.handleSubmit}>
        <custom-input name="firstName"></custom-input>
        <custom-input name="lastName"></custom-input>
        <button type="submit">Submit</button>
      </form>
    `
  }
}

const enhance = withForm({
  initialValues: {/*...*/},
  onSubmit: {/*...*/},
  validationSchema: {/*...*/}
})

customElements.define('user-form', enhance(UserForm))
```
Расширяемые классы могут как использовать Shadow DOM, так и оставаться открытыми и лучше отвечать специфике проекта. Полный код компонента формы с примерами можно [посмотреть тут](https://github.com/realiarthur/lite-form).


## В заключение
В названии статьи «веб-компоненты» можно было бы заменить на «кастомные элементы» — в итоге это стало единственным API, которое я использовал в проекте, и самое проработанное из всей технологии. Но мне хотелось бы верить, что и оно будет развиваться в части создания элементов, передачи свойств и подписки на их изменения, чтобы стать более лаконичным. Я использовал LitElement в основном для этого, и мне не пригодился его жизненный цикл — хватало стандартного. 

Слоты также многообещающая технология, особенно если их использование стало бы возможным вне концепции Shadow DOM ([без хаков](https://stackoverflow.com/questions/48726904/is-there-a-way-workaround-to-have-the-slot-principle-in-hyperhtml-without-using)). Это дало бы широкие возможности для композиции компонентов.

Shadow DOM определенно будет полезен для ряда случаев, но обоснованность его повсеместного применения вызывает сомнения. Использование Shadow DOM сопряжено с некоторыми ограничениями и неудобствами. При этом для инкапсуляции CSS можно использовать более производительные технологии, а инкапсуляция DOM нужна далеко не всегда.

Хотя при применении веб-компонентов еще остается ряд открытых вопросов, в целом, технология смотрится перспективной, а ее использование уже сейчас вполне может себя оправдать на небольших проектах. Но в текущем состоянии использовать ее как замену фреймворков на большом проекте, над которым работает группа разработчиков, кажется преждевременным.



### Полезные ссылки 
[webcomponents.org,](https://www.webcomponents.org/)
[Polymer,](https://www.polymer-project.org/)
[LitElement,](https://lit-element.polymer-project.org/)
[lit-html,](https://lit-html.polymer-project.org/)
[Vaadin Router,](https://vaadin.com/router)
[Vaadin Components,](https://vaadin.com/components)
[lite-redux,](https://www.npmjs.com/package/lite-redux) 
[lite-form,](https://www.npmjs.com/package/lite-form)
[awesome-lit-html,](https://github.com/web-padawan/awesome-lit-html)
[Polyfills,](https://github.com/webcomponents/polyfills/tree/master/packages/webcomponentsjs)
[custom-elements-builtin polyfill](https://www.npmjs.com/package/@ungap/custom-elements-builtin)
