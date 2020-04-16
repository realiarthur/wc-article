# Веб-компоненты в реальном проекте

Изолированные кастомные компоненты, поддерживаемые браузером из коробки и возможность расширять нативные html-элементы - давно звучало как одно из возможных направлений развития frontend-разработки. Но при выборе технологий для реальных проектов всегда стоит вопрос: "Будущее уже настало?"

О веб-компонентах уже достаточно написано, и руки уже несколько лет чесались попробовать их на реальном проекте. Речь в статье пойдет об этом опыте (а не о технологии), о том с какими проблемами пришлось столкнуться, их решениях, полезных хаках и о том как многие концепции, к которым мы привыкли при работе с фреймворками, легко перекладываются на веб-компоненты. 


## Почему веб-компоненты? 
Проект был небольшим, но требовательным к размеру бандла; основными ui-составляющими были формы. Кто-то скажет, что все можно было сделать нативным html+js, что было бы максимально легковесно. Но если говорить о поддержке и расширении проекта, даже учитывая трабования к размеру бандла, отказываться от всех преимуществ компонентной разработки было бы подобно прыжку в прошлое.

С другой стороны, можно было бы использовать [Svelte](https://github.com/sveltejs/svelte) или, например, [Preact](https://github.com/preactjs/preact). Но попробовать сделать все, оперируя нативными концепциями, было слишком заманчивой идеей. 


## Будущее уже настало? 
Большинство современных браузеров, включая мобильные, поддерживают веб-компоненты из коробки. Для остальных есть [полифилы](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). Примечательно то, что для полифилов есть умный [лоадер](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~2kB), который не загружает полифил определенного функционала, если для него есть нативная поддержка браузером - таким образом в современные браузеры ничего не загружается. Полифилы заявляют поддержку вплоть до IE11 и версий Edge, которые не построены на chromium. К счастью, мы не поддерживаем IE в наших проектах, в Edge действительно все работает. Так же все работает в китайских UC Browser и QQ Browser, включая их мобильные версии.

> Небольшие ограничения по работе полифилов:
> - Полифильный тег ```<slot></slot>``` для IE11 & Edge не участвуют во вспытии событий
> - Этот набор полифилов не предоставляет функционал для расширения встроенных html-элементов. Об этом подробнее ниже  


## LitElement
"Как много boilerplate-кода для создания компонентов и реактивной работы с пропсами! Нужно ускорить процесс и написать класс расширяющий HTMLElement, реализующий этот код" - такой была первая мысль, которая возникла при погружении в веб-компоненты. Писать, конечно, ничего не пришлось - эту работу берет на себя класс [LitElement](https://lit-element.polymer-project.org/) (~7kb). А использующийся в нем [lit-html](https://lit-html.polymer-project.org/), предоставляет удобный и оптимизированный механизм рендеринга и упрощает привязку данных и подписку на события. 

Кроме того, LitElement расширяет стандартный **жизненный цикл веб-компонентов** (connectedCallback, disconnectedCallback, attributeChangedCallback, adoptedCallback) удобными [дополнениями](https://lit-element.polymer-project.org/guide/lifecycle), делая его во многом схожим с жизненным циклом компонентов популярных фреймворков.

Спешу согласиться, LitElement - это еще одна зависимость, кто-то даже скажет "еще один фреймворк", но я осознанно *решился на эти плюс 7kB,* потому что LitElement смотрится как один из вариантов естественной надстройки над существующим нативным API, которая так или иначе присутствовала бы в прокте, хотя бы для релизации boilerplate-кода. Надеюсь в будущем нативное API станет более удобным и лаконичным.


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
    
    // Коллбек для подписки на изменние store, который вызовет все mapStateToProps из цепочки наследования
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
        
        // addonIcon включен в properties только для иллюстрации.
        // В данном конкретном классе это излишне, потому что
        // реактивность обеспечит свойство type из super.properties
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

Внимание здесь хотелось бы обратить на строку ```...super.properties``` в методе get properties(), которая позволяет не определять те свойства которые уже описаны в расширяемом компоненте. И на строку ```super.render()``` в методе render, которая выводит в указанном месте разметки отображение расширяемого элемента.


При использовании такого подхода для реализации аналога HOC стоит соблюдать некоторые меры предосторожности: 
- быть внимательным при именовании свойств и методов, так как можно переопределить их в наследуемом компоненте
- помнить, что при передаче метода класса в качестве колбека на какое-либо событие , есть вероятность что этот метод будет  переопределен где-то в цепочке наследования, и на событие окажется подписанным только последний из HOC'ов, а не оба
- стараться максимально наглядно подписываться на изменения свойств, переданных из HOC.

## Формы и UI. Как я отказался от Shadow DOM
Как я уже говорил, основными ui-составляющими моего проекта являются формы и различные поля ввода. Для удобства хотелось использовать компонент реализующий стандартный функционал формы: хранение и обновление состояния (values, errors, touched), валидация, обработка submit, reset и предоставляющий удобные инструменты для работы с формой. 

Этот функционал не имеет отношения к теме статьи, но хотелось поговорить о том, во что его обернуть. Забегая вперед скажу, что прежде чем сделать рабочий вариант я попробовал 3 упаковки: стандарнтый веб-компонент, расширение нативного тега ```<form>``` и компонент высшего порядка. И это повод для небольшой истории...

### Версия 1. Shadow DOM & Slots
Первое что пришло мне на ум - сделать стандартные веб-компоненты полей ввода и формы, ипользуя Shadow DOM, как часть лучших практик, обернуть поля ввода в HOC для общения с формой, вставлять форму на старницу отдельным кастомным тегом. Обо всем по порядку.

С одной стороны мне нужен был функцонал тега `<form>` и его класса HTMLFormElement (например, вызов submit при нажатии на Enter), но не хотелось в дополнение к кастомному тегу формы писать его каждый раз в Light DOM (дочерние элементы компонента, вроде children в React). С другой, я не мог использовать слот, чтобы обернуть его содержимое в тег `<form>`, потому что тогда элементы формы фактически располагались бы снаружи (к слову, именно из-за этого стилизовать содержимое слотов можно из внешнего документа) компонента формы, и `<form>` не реагировал на их события. 

Решением стала передача шаблона в качетсве свойства для кастомной формы. Это некий аналог передачи Render-функции в React: 

```js
// Компонент формы
import { LitElement, html } from 'lit-element'

class LiteForm extends LitElement {
  <... функционал формы ...>
  
  render() {
    return html`<form @submit=${this.handleSubmit} method=${this.method}>
      ${this.formRender(this)}
    </form>`
  }
}

customElements.define("lite-form", LiteForm);
```

```js
// Использование формы
import { html, render } from 'lit-element'

const formRender = ({ values, handleBlur, handleChange, ...props }) =>
  html` <input
      name="login"
      .value=${values.login}
      @input=${handleChange}
      @blur=${handleBlur}
    />
    <input
      name="password"
      .value=${values.password}
      @input=${handleChange}
      @blur=${handleBlur}
    />
    <button type="submit">Submit</button>`

const MyForm = () =>
  html`<lite-form
    method="post"
    .formRender=${formRender}
    .onSubmit=${values => console.log(values)}
    .initialValues=${{
      login: '',
      password: ''
    }}
    .validationSchema=${{
      login: value => {
        if (!value) return 'Required'
      },
      password: value => {
        if (value.length < 5) return 'Must be more than 5 letters'
      }
    }}
  ></lite-form>`

render(
  html`MyForm()}`,  document.getElementById('root')
)
```

Конечно мне не хотелось каждый раз передавать свойства и события формы полям ввода, к тому же я хотел работать с кастомными полями ввода, и упростить вывод ошибок. Поэтому мне необходимы были компоненты высшего порядка для работы с формой. Такая реализация требовала, чтобы HOC'и были способны самостоятельно найти свою форму или общаться с ней посредством событий. Перебрав несколько вариантов (общение через шину событий, общий контекст - все они подходили, если форма на странице только одна), я остановился на очень простом, но рабочем:

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
Здесь все тривиально: рекурсивный поиск элемента с атрибутом, указывающим что это искомая форма. Хочется отметить, что благодаря функции [getRootNode](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode), поиск происходит сквозь дерево вложенных Shadow DOM - удобная функция при решении таких специфических задач.

С использованием HOC я мог сильно упростить шаблон формы:
```js
const formRender = (props) =>
  html` <custom-input name="login"></custom-input>
    <custom-input name="password" type="password"></custom-input>
    <button type="submit">Submit</button>`
```

В общем все работало отлично, но... Прежде чем рассказать почему я откался от Shadow DOM, еще пару слов о нем.

### Стилизация Shadow DOM извне и :host-context
Для кастомизации стилей компонентов из основного документа можно исользовать CSS переменные. Они проникают через Shadow DOM, видны повсюду и это вполне удобно. Но возникают ситуации когда этого оказывается недостаточно, например, когда необходима условная стилизация:
```css
:host-context(body[dir='rtl']) .text {
  text-align: right;
}
```
Псевдокласс `:host-context` позволяет понять в каком контексте находится Shadow DOM и реагировать на него. В примере блок с классом `.text` внутри компонента будет соответствующим образом выравнивать текст в зависимости от атрибута `dir` установленного для `body` документа. 

### Всплытие событий свозь Shadow DOM
События при всплытии сквозь Shadow DOM ведут себя несколько необычно: у них подменяется целевой элемент `target`, которым становится сам веб-компонент. Сделано это в целях поддержания инкапсуляции DOM, и это необходимо учитывать при разработке, потому что конечный `target` может не содержать всех тех свойств, которые содержал изначальный целевой элемент. 

Всплытие сквозь Shadow DOM можно регулировать свойством события `composed`. Чтобы событие всплывало сквозь Shadow DOM оба его свойства `composed` и `bubbles` должны быть установлены в `true`. Большинство стандартных событий успешно всплывает сквозь Shadow DOM.

### Почему я отказался от Shadow DOM
Форма в shadow DOM без слота - не работает
Форма в shadow DOM на верхнем уровне и слот (получается из основного дока) как версия 1 - работает
Если через двойной слот - работает - все остается на самом верху, вне теневого DOM
Форма в компоненте с теневым DOM - не работает. С первого же перестает работать
Если shadow DOM в инпутах = не работает

 
## Заключение 
В названии статьи "веб-компоненты" можно было бы заменить на "кастомные элементы", в итоге это стало единственным API, которым я пользовался в проекте, и самое проработанное из всей технологии. Но мне хотелось бы верить что и оно будет меняться, в части создания элементов, передачи свойств и обеспечения реактивности, чтобы стать более лаконичным. Я использовал LitElement в основном для этого, и мне почти не пригодился его жизненный цикл - хватало стандартного. (!!! точно не пригодился? что использовал из LIT? Что там еще есть кроме жизненного цикла)

 ***
 полезные ссылки 
 https://vaadin.com/router
 https://github.com/web-padawan/awesome-lit-html
