# Веб-компоненты в реальном проекте

Изолированные кастомные компоненты, поддерживаемые браузером из коробки и возможность расширять встроенные html-элементы - давно звучало как одно из возможных направлений развития frontend-разработки. Но при выборе технологий для реальных проектов всегда стоит вопрос: "Будущее уже настало?"

О веб-компонентах уже достаточно написано, и руки уже несколько лет чесались попробовать их на реальном проекте. Речь в статье пойдет об этом опыте (а не о технологии), о том с какими проблемами пришлось столкнуться, их решениях, полезных хаках и о том как многие концепции, к которым мы привыкли при работе с фреймворками, легко перекладываются на веб-компоненты. 


## Почему веб-компоненты? 
Проект был небольшим, но требовательным к размеру бандла; основными ui-составляющими были формы. Кто-то скажет, что все можно было сделать нативным html+js, что было бы максимально легковесно. Но если говорить о поддержке и расширении проекта, даже учитывая трабования к размеру бандла, отказываться от всех преимуществ компонентной разработки было бы подобно прыжку в прошлое.

С другой стороны, можно было бы использовать [Svelte](https://github.com/sveltejs/svelte) или, например, [Preact](https://github.com/preactjs/preact). Но попробовать сделать все, оперируя нативными концепциями, было слишком заманчивой идеей. 


## Будущее уже настало? 
Большинство современных браузеров, включая мобильные, поддерживают веб-компоненты из коробки. Для остальных есть [полифилы](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). Примечательно, что для полифилов есть умный [лоадер](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~2kB), который не загружает полифил определенного функционала, если для него есть нативная поддержка браузером - таким образом в современные браузеры ничего не загружается. Полифилы заявляют поддержку вплоть до IE11 и версий Edge, которые не построены на chromium. К счастью, мы не поддерживаем IE в наших проектах, в Edge действительно все работает. Также все работает в китайских UC Browser и QQ Browser, включая их мобильные версии.

> Небольшие ограничения по работе полифилов:
> - Полифильный тег ```<slot></slot>``` для IE11 & Edge не участвуют во вспытии событий
> - Этот набор полифилов не предоставляет функционал для расширения встроенных html-элементов. Об этом подробнее ниже  


## LitElement
"Как много boilerplate-кода для создания компонентов и реактивной работы с пропсами! Нужно ускорить процесс и написать класс расширяющий HTMLElement, реализующий этот код" - такой была первая мысль, которая возникла при погружении в веб-компоненты. Писать, конечно, ничего не пришлось - эту работу берет на себя класс [LitElement](https://lit-element.polymer-project.org/) (~7kb). А использующийся в нем [lit-html](https://lit-html.polymer-project.org/), предоставляет удобный и оптимизированный механизм рендеринга и упрощает привязку данных и подписку на события. 

Кроме того, LitElement расширяет стандартный **жизненный цикл веб-компонентов** (connectedCallback, disconnectedCallback, attributeChangedCallback, adoptedCallback) удобными [дополнениями](https://lit-element.polymer-project.org/guide/lifecycle), делая его во многом схожим с жизненным циклом компонентов популярных фреймворков.

LitElement - это еще одна зависимость, кто-то даже скажет "еще один фреймворк", но я осознанно *решился на эти плюс 7kB,* потому что LitElement смотрится как один из вариантов естественной надстройки над существующим нативным API, которая так или иначе присутствовала бы в прокте, хотя бы для релизации boilerplate-кода.


## Функциональные компоненты-шаблоны
К теме статьи это имеет косвенное отношение, но lit-html в свои ~3.5kB вмещает настолько удобную возможность описания интерфейса с помощью функций, что невозможно умолчать. Причем обновление DOM-структур таких "компонентов" происходит оптимизированно - рендерятся только те блоки, значения которых изменились с предыдущего рендера. И все это без Virtual DOM! В некоторых случаях и при должной находчивости весь интерфейс можно описать только функциями, декораторами и директивами (о них чуть ниже):

```js
import {html, render} from 'lit-html';

const ui = (data) => html`...${data}...`;

render(ui('Hello!'), document.body);
```

При этом в одних шаблонах можно использовать другие:
```js 
const myHeader = html`<h1>Header</h1>`;
const myPage = html`
  ${myHeader}
  <div>Here's my main page.</div>
`;
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

render(html`<custom-ui .data='Hello!'></custom-ui>`, document.body);
```

Я не буду останавливаться на удобствах создания шаблонов, стилизации, передачи атрибутов, привязки данных и подписки на события, условном и цикличном рендере при работе с lit-html - это подробно описано в [документации](https://lit-html.polymer-project.org/guide/writing-templates). Остановлюсь на том, что возможно не ухватить при беглом взгляде на руководство, но может быть полезно.

### svg
Самая скрытая из таких функций, о которой нет упоминаний в руководстве (но есть в [API](https://lit-html.polymer-project.org/api/modules/lit_html.html#svg)) - это тег ```svg`` ```. Как несложно догадаться, он используется для работы с векторной графикой, при работе с которой через ```html`` ``` могут возникать некоторые проблемы. У меня они возникли когда я пытался передать TemplateResult (это результат выполнения ```html`` ```) вовнутрь моего компонента иконки - появлялись ненужные закрывающие теги и графика не отрисовывалась. При использовании ```svg`` ``` и передачи SVGTemplateResult все встало на свои места. 

### Директивы
Директивы - это функции, которые описывают то, каким образом и что будет и будет ли выводится в месте их использования внутри тега ```html`` ``` . В lit-html для хранения и вывода значений для представления DOM используются классы реализующие интерфейс Part. Именно интерфейс Part обеспечивает умный рендеринг, который обновляет только то что изменилось, и директивы - это способ получения доступа к этому механизму и влияния на него. Директивы могут быть одного из 5 типов, каждая из которых имеет доступ к соответствующей реализации Part: 
- для вывода контента (NodePart)
- для передачи атрибута (AttributePart)
- для передачи булевого атрибута (BooleanAttributePart)
- для привязки данных или передачи свойства (PropertyPart)
- для подписки на события (EventPart). 

Каждый тип директив может быть использован только в подходящем для него месте и не имеет значения в других местах.

Директива хранит в себе значение - `value`, это то что было выведено на ее месте при последнем рендере. Для установки нового значения существует функция `setValue()`, для принудительного обновления значений в DOM, после того как рендер был завершен используется функция `commit()` (полезно при асинхронных действиях).

Пример кастомной директивы (имеющей доступ к классу NodePart - для вывода контента), которая хранит и выводит количество рендеров: 
```js
import { directive } from 'lit-html'

const renderCounter = directive(() => (part) =>
  part.setValue(part.value === undefined
     ? 0
     :  part.value + 1);
 );
```

lit-html имеет полезный [набор встроенных директив](https://lit-html.polymer-project.org/guide/template-reference#built-in-directives). Не буду останавливаться на этом подробно, но там есть аналоги некоторых реактовских хуков, интересные функции для работы со стилями и классами как с объектами, функции асинхронного обновления контента, оптимизации, небезопасной установки html и др. 

Рядом с директивой можно хранить более сложный state для использования внутри директивы. Пример [тут](https://lit-html.polymer-project.org/guide/creating-directives#maintaining-state). 

Кастомные директивы и декораторы можно также использовать как аналог компонентов высшего порядка. При таком подходе необходимо самостоятельно позаботится о реактивности внутри директивы. Пример [тут](https://github.com/jmas/lit-redux/blob/master/index.js). 

### shady-render
Если Вы создаете базовый класс с использованием lit-html и Shadow DOM, Вам необходимы будут полифилы для старых браузеров. У lit-html есть отдельный [модуль shady-render](https://lit-html.polymer-project.org/guide/styling-templates#polyfilled-shadow-dom:-shadydom-and-shadycss) который интегрирует эти полифилы. 



## Компоненты высшего порядка
HOC - мощный паттерн, часто используемый при работае с React или Vue. При его использовании композиция компонентов становится простой и лаконичной, и хотелось бы использовать какой-то его аналог при работе с веб-компонентами. Так как веб-компоненты - это классы, для себя, в качестве аналога HOC, я решил использовать фунции, которые бы возвращали расширенние класса, переданного им в качестве параметра. 

### Расширение функционала
В проекте мне необходим был redux, поэтому в качестве примера рассмотрю [коннектор для него](https://github.com/realiarthur/lite-redux). Ниже представлен код декоратора, принимающего store и возвращающего стандартный коннектор redux. Внутри класса происходит накопление mapStateToProps из всей цепочки наследования (для тех случаев если в ней будет HOC, который также общается с redux), чтобы в дальнейшем, когда компонент будет встроен в DOM, одним колбеком все их подписать на изменение состояния redux. При удалении компонента из DOM эта подписка удаляется.

```js
import { bindActionCreators } from "redux";

export default store => (mapStateToProps, mapDispatchToProps) => Component =>
  class Connect extends Component {
    constructor(props) {
      super(props);
      this._getPropsFromStore=this._getPropsFromStore.bind(this)
      this._getInheritChainProps=this._getInheritChainProps.bind(this)
      
      // Накопление mapStateToProps
      this._inheritChainProps = (this._inheritChainProps || []).concat(
        mapStateToProps
      );
    }
     
    // Функция для получения данных из store
    _getPropsFromStore (mapStateToProps) {
      if (!mapStateToProps) return;   
      const state = store.getState();
      const props = mapStateToProps(state);

      for (const prop in props) {
        this[prop] = props[prop];
      }
    };
    
    // Callback для подписки на изменние store, который вызовет все mapStateToProps из цепочки наследования
    _getInheritChainProps () {
      this._inheritChainProps.forEach(i => this._getPropsFromStore(i));
    };

    connectedCallback() {
      this._getPropsFromStore(mapStateToProps);

      this._unsubscriber = store.subscribe(this._getInheritChainProps);

      if (mapDispatchToProps) {
        const dispatchers =
          typeof mapDispatchToProps === "function"
            ? mapDispatchToProps(store.dispatch)
            : mapDispatchToProps;
        for (const dispatcher in dispatchers) {
          typeof mapDispatchToProps === "function"
            ? (this[dispatcher] = dispatchers[dispatcher])
            : (this[dispatcher] = bindActionCreators(
                dispatchers[dispatcher],
                store.dispatch,
                () => store.getState()
              ));
        }
      }

      super.connectedCallback();
    }

    disconnectedCallback() {
      // Удаление подписки на изменение store
      this._unsubscriber();
      super.disconnectedCallback();
    }
  };
```

Удобнее всего использовать этот метод для создания и экпорта обычного коннектора, при инициализации store, который потом и использовать в качестве HOC:

```js
// store.js
import { createStore, combineReducers } from "redux";
import makeConnect from "lite-redux";

const reducer = combineReducers({ ... });

const store = createStore(reducer);

export default store;

// Создание стандартного коннектора
export const connect = makeConnect(store);
```

```js
// Component.js
import { connect } from "./store";

class Component extends WhatEver {
    ...
}

export default connect(mapStateToProps, mapDispatchToProps)(Component)

```

### Расширение отображения и наблюдаемых свойств
Во многих случаях кроме расширения функционала компонента требуется обернуть его отображение. Для этого удобно использовать функцию рендеринга расширяемого компонента. Кроме того, бывает полезно расширить список наблюдаемых свойств для обеспечения реактивности: `get observedAttributes()` для веб-компонентов или `get properties()` для LitElement. Для иллюстрации приведу упрощенный код поля ввода пароля, которое расширяет переданный ей компонент текстового поля ввода:

```js
const withPassword = Component =>
  class PasswordInput extends Component {
    static get properties() {
      return {
        // Предполагается что super.properties уже содержит свойство type
        ...super.properties,
        addonIcon: { type: String },
      };
    }
    
    constructor(props) {
      super(props);
      this.type = "password";
      this.addonIcon = "invisible";
    }

    setType(e) {
      this.type = this.type === "text" ? "password" : "text";
      this.addonIcon = this.type === "password" ? "invisible" : "visible";
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
      `;
    }
  };

customElements.define("password-input", withPassword(TextInput));
```

Внимание здесь хотелось бы обратить на строку `...super.properties` в методе `get properties()`, которая позволяет не определять те свойства которые уже описаны в расширяемом компоненте. И на строку `super.render()` в методе `render`, которая выводит в указанном месте разметки отображение расширяемого элемента.


При использовании такого подхода для реализации аналога HOC стоит соблюдать некоторые меры предосторожности: 
- быть внимательным при именовании свойств и методов, так как можно переопределить их в наследуемом компоненте
- помнить, что при передаче метода класса в качестве колбека на какое-либо событие , есть вероятность что этот метод будет  переопределен где-то в цепочке наследования, и на событие окажется подписанным только последний из HOC'ов, а не оба
- стараться максимально наглядно подписываться на изменения свойств, переданных из HOC.

## Как я отказался от Shadow DOM и расширения встроенных элементов.
Как я уже говорил, основными ui-составляющими моего проекта являются формы и различные поля ввода. Для удобства хотелось использовать компонент реализующий стандартный функционал формы: хранение и обновление состояния (values, errors, touched), валидация, обработка submit, reset и предоставляющий удобные инструменты для работы с формой. 

Этот функционал не имеет отношения к теме статьи, но хотелось поговорить о том, во что его обернуть. Забегая вперед скажу, что прежде чем сделать рабочий вариант я попробовал 3 упаковки: стандарнтый веб-компонент, расширение встроенного элемента ```<form>``` и компонент высшего порядка. И это повод для небольшой истории...

### Версия 1. Веб-компоненты и Shadow DOM
Первое что пришло мне на ум - сделать стандартные веб-компоненты полей ввода и формы, ипользуя Shadow DOM, как часть лучших практик, обернуть поля ввода в HOC для общения с формой, вставлять форму на старницу отдельным кастомным тегом. Обо всем по порядку.

С одной стороны мне нужен был функцонал тега `<form>` и его класса HTMLFormElement (например, вызов submit при нажатии на Enter), но не хотелось в дополнение к кастомному тегу формы писать его каждый раз в Light DOM (дочерние элементы компонента, вроде children в React). С другой, я не мог использовать слот, чтобы обернуть его содержимое в тег `<form>`, потому что тогда элементы формы фактически располагались бы снаружи (к слову, именно из-за этого стилизовать содержимое слотов можно из внешнего документа) компонента формы, и `<form>` не реагировал бы на их события. 

Решением стала передача шаблона в качестве свойства для кастомной формы. Это некий аналог передачи render-функции в React: 

```js
// Компонент формы
import { LitElement, html } from 'lit-element'

class LiteForm extends LitElement {
  //... функционал формы ...
  
  render() {
    return html`<form @submit=${this.handleSubmit} method=${this.method}>
      ${this.formTemplate(this)}
    </form>`
  }
}

customElements.define("lite-form", LiteForm);
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
    .onSubmit=${{...}}
    .initialValues=${{...}}
    .validationSchema=${{...}}
  ></lite-form>`

render(html`${MyForm}`, document.getElementById('root'))
```

Конечно мне не хотелось каждому полю ввода передавать свойства и события формы, к тому же я хотел работать с кастомными полями ввода, и упростить вывод ошибок. Поэтому мне необходимы были несколько компонентов высшего порядка для работы с формой, таких как `withField` или `withError`. 

Такая реализация требовала, чтобы HOC'и были способны самостоятельно найти свою форму или общаться с ней посредством событий. Перебрав несколько вариантов (общение через шину событий, общий контекст - все они подходили, если форма на странице только одна), я остановился на очень простом, но рабочем:

```js
// здесь константа IS_LITE_FORM - это назывние булевого атрибута, который имеет каждый элемент кастомной формы
const getFormClass = element => {
  const form = element.closest(`[${IS_LITE_FORM}]`);
  if (form) return form;

  const host = element.getRootNode().host;
  if (!host) throw new Error("Lite-form not found");
  return host[IS_LITE_FORM] ? host : getFormClass(host);
};
```
Здесь все тривиально: рекурсивный поиск элемента с атрибутом, указывающим что это искомая форма. Отметить хотелось функцию [getRootNode](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode), благодаря которой поиск происходит сквозь дерево вложенных Shadow DOM - необходимая функция при решении таких специфических задач.

С использованием `withField` я мог сильно упростить шаблон формы:
```js
const formTemplate = (props) =>
  html`<custom-input name="firstName"></custom-input>
    <custom-input name="lastName"></custom-input>
    <button type="submit">Submit</button>`
```

В общем все работало отлично, но... Прежде чем рассказать почему я откался от Shadow DOM, еще пару слов о нем.

### Стилизация Shadow DOM извне и псевдоклассы :host и :host-context
Для кастомизации стилей компонентов из основного документа можно исользовать css переменные. Они проникают через Shadow DOM, видны повсюду и это вполне удобно. Но возникают ситуации когда этого оказывается недостаточно, например, когда необходима условная стилизация:
```css
:host([rtl]) {
  text-align: right;
}

:host-context(body[dir='rtl']) .text {
  text-align: right;
}
```
Псевдокласс `:host` позволяет применить стиль к корневому элементу, только если он соответствует переданному селектору. В примере текст будет выровнен по правому краю только в случае, если корневой элемент имеет атрибут `rtl`.

Псевдокласс `:host-context` позволяет понять в каком контексте находится Shadow DOM и реагировать на него. В примере блок с классом `.text` внутри компонента будет соответствующим образом выравнивать текст в зависимости от атрибута `dir` установленного для `body` документа. 

### Всплытие событий свозь Shadow DOM
События при всплытии сквозь Shadow DOM ведут себя несколько необычно: у них подменяется целевой элемент `target`, которым становится сам веб-компонент. Сделано это в целях поддержания инкапсуляции DOM, и это необходимо учитывать при разработке, потому что конечный `target` может не содержать всех тех свойств, которые содержал изначальный целевой элемент. 

Всплытие сквозь Shadow DOM можно регулировать свойством события `composed`. Чтобы событие всплывало сквозь Shadow DOM оба его свойства `composed` и `bubbles` должны быть установлены в `true`. Большинство стандартных событий успешно всплывает сквозь Shadow DOM.

### Почему я отказался от Shadow DOM
Во-первых, из-за невозможности автозаполнения форм сохраненными пользовательскими данными, такими как логин-пароль, адрес, данные кредитных карт, если форма или поля ввода вложены в компонент, использующий Shadow DOM, или сами его используют. В моем проекте этот недостаток (или фича, кому как ближе) стал критичным.

>  Конечно, если для передачи шаблона формы используются только слоты, и этот шаблон описан в основном документе, которому он и будет принадлежать, то автозаполнение произойдет, даже при использовании Shadow DOM в компоненте формы. Но кроме этого все компоненты в которые вложена форма и все поля ввода не должны использовать Shadow DOM. Это настолько сильные ограничения что проще отказаться от Shadow DOM

Во-вторых, из-за отсутствия аналога querySelector, который работает сквозь Shadow DOM. Для метрик и аналитики такой инструмент был бы полезен, чтобы не писать длинные коснтрукции типа `document.querySelector(...) .shadowRoot.querySelector(...) .shadowRoot.querySelector(...) .shadowRoot.querySelector("...")`

С отказом от Shadow DOM особых трудностей не возникло. В LitElement для этого достаточно такого кода:
```js
createRenderRoot() {
  return this
}
```
Вместе с Shadow DOM я потерял проводника всплытия для события `blur` из полей ввода. Я использовал его внутри `withField`, чтобы мне не приходилось пробрасывать его из компонента вручную. Но я подписался на него на стадии погружения и это сработало.

### Версия 2. Расширение встроенного элемента
После отказа от Shadow DOM мне показалось хорошей идеей расширить встроенный класс HTMLFormElement и тег `<form>` - верстка бы смотрелось как нативная, при этом сохранился бы доступ ко всем событиям формы, и это требовало минимальных изменений кода:
```js
// Компонент формы
class LiteForm extends HTMLFormElement {
  connectedCallback() {
    this.addEventListener('submit', this.handleSubmit)
    this.addEventListener('reset', this.handleReset)
  }

  disconnectedCallback() {
    this.removeEventListener('submit', this.handleSubmit)
    this.removeEventListener('reset', this.handleReset)
  }

  //... функционал формы ...
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
Здесь хотелось бы обратить внимание на первый аргумент в функции `customElements.define` и на атрибут `is` в элементе формы, которые указывают на то, какого "типа" будет тег, они должны быть равны. А также на третий аргумент `customElements.define`, в котором указывается какой именно тег будет расширяться.

Все работало отлично и прекрасно смотрелось. Но не во всех браузерах: Safari не поддерживает расширение встроенных элементов, потому что считает это небезопасным. Это также касается и браузеров для iOS (включая Chrome, Firefox и др.). Ходят слухи, что Apple возможно пересмотрит это поведение, но в 13 версии Safari и iOS все остается как есть. И хотя есть [полифилы](https://www.npmjs.com/package/@ungap/custom-elements-builtin), я решил не пользоваться этой концепцией до тех пор, и если Safari и iOS не начнут поддерживать расширение встроенных элементов.

### Версия 3. Компонент высшего порядка
После пары неудачных попыток обернуть функционал формы я решил, что оставлю это за пределами формы и сделаю компонент высшего порядка, чтобы оборачивать им все, что понадобиться. Это потребовало небольших изменений кода:
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

      /* ...функционал формы ... */

      super.connectedCallback && super.connectedCallback()
    }

    /* ...функционал формы ... */
  }
```

Здесь в функции `connectedCallback()` форма принимает конфиг (onSubmit, initialValues, validationSchema, и др.)  либо из аргументов, переданных в `withForm()` либо из самого расширяемого компонента. Это позволяет оборачивать любые классы, а так же строить базовые и использовать их в верстке и в ней же передавать им конфиг. К слову, этим способом можно построить оба базовых класса из первых реализаций формы:

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

С другой стророны можно не вводить кастомный элемент для формы, и оборачивать в `withForm()` конечные компоненты содержащие шаблоны форм:

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
  initialValues: {
    /*...*/
  },
  onSubmit: {
    /*...*/
  },
  validationSchema: {
    /*...*/
  }
})

customElements.define('user-form', enhance(UserForm))
```
И расширяемые классы могут как использовать Shadow DOM, так и оставаться открытыми и лучше подходить под специфику проекта. Полный код компонента формы с примерами можно [посмотреть тут](https://github.com/realiarthur/lite-form).


## В заключении 
В названии статьи "веб-компоненты" можно было бы заменить на "кастомные элементы" - в итоге это стало единственным API, которое я пользовал в проекте, и самое проработанное из всей технологии. Но мне хотелось бы верить что и оно будет развиваться, в части создания элементов, передачи свойств и подписки на их изменения, чтобы стать более лаконичным. Я использовал LitElement в основном для этого, и мне не пригодился его жизненный цикл - хватало стандартного. 

Слоты также многообещающая технология, особенно если бы их использование стало возможным вне концепции Shadow DOM ([без хаков](https://stackoverflow.com/questions/48726904/is-there-a-way-workaround-to-have-the-slot-principle-in-hyperhtml-without-using)), а события элементов в слоте погружались и всплывали через элементы в которые они вставлены, хотя бы опционально. Это стало бы аналогом `children` в React, который бы дал шикрокие возможности для композиции компонентов.

Shadow DOM определенна будет полезна для ряда случаев, но необходимость его повсеместного применения вызывает сомнения. Использование Shadow DOM сопражено с некоторыми ограничениями и неудобствами, при этом для инкапсуляции css можно использовать более производительные технологии, а инкапсуляция DOM нужна далеко не всегда.

Проект успешно прошел тестирование на нашей широкой аудитории, размер бандла удалось ощутимо снизить, как и время загрузки. В целом технология кажется перспективной а ее использование уже сейчас вполне может себя оправдать на небольших проектах. Но в текущем состоянии я бы не взял ее на большой проект, над которым работает несколько человек. 


### Полезные ссылки 
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
