# Веб-компоненты в реальном проекте

Изолированные кастомные компоненты, поддерживаемые браузером из коробки и возможность расширять встроенные html-элементы - давно звучало как один из следующих шагов во фронтенд-разработке. Но при выборе технологий для реальных проектов всегда стоит вопрос "Будущее уже настало?"

О технологии веб-компонентов уже достаточно написано, и руки давно чесались попробовать ее на реальном проекте. Речь в статье пойдет об этом опыте (а не о технологии), о том с какими проблемами пришлось столкнуться, их решениях, и о том как многие концепции, к которым мы привыкли при работе с фреймворками, легко перекладываются на веб-компоненты. 


## Почему веб-компоненты? 
Проект был небольшим, но требовательным к размеру бандла; основными ui-составляющими были формы. Кто-то скажет, что все можно было сделать нативным html+js, что было бы максимально легковесно. Но если говорить о поддержке и расширении проекта, даже учитывая трабования к размеру бандла, отказываться от всех преимуществ компонентной разработки было бы подобно прыжку в прошлое.

С другой стороны, можно было бы использовать [Svelte](https://github.com/sveltejs/svelte) или, например, [Preact](https://github.com/preactjs/preact). Но попробовать сделать все, оперируя нативными концепциями, было слишком заманчивой идеей. 


## Будущее уже настало? 
Большинство современных браузеров, включая мобильные, поддерживают веб-компоненты из коробки. Для остальных есть [полифилы](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). Примечательно то, что для полифилов есть умный [лоадер](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~2kB), который не загружает полифил определенного функционала, если для него есть нативная поддержка браузером - таким образом в современные браузеры ничего не загружается. Полифилы заявляют поддержку вплоть до IE11 и версий Edge, которые не построены на chromium. К счастью, мы не поддерживаем IE в наших проектах, в Edge действительно все работает. Так же все работает в китайских UC Browser и QQ Browser, включая их мобильные версии.

> Небольшие ограничения по работе полифилов:
> - Полифильный тег ```<slot></slot>``` для IE11 & Edge не участвуют во вспытии событий
> - Этот набор полифилов не предоставляет функционал для расширения встроенных html-элементов. Об этом подробнее ниже  


## LitElement
"Как много boilerplate-кода для создания компонентов и реактивной работы с пропсами! Нужно ускорить процесс и написать класс расширяющий HTMLElement, реализующий этот код" - такой была первая мысль, которая возникла при погружении в веб-компоненты. Писать, конечно, ничего не пришлось - эту работу берет на себя класс [LitElement](https://lit-element.polymer-project.org/) (~7kb). А использующийся в нем [lit-html](https://lit-html.polymer-project.org/) (~3.5kB), предоставляет удобный и оптимизированный механизм рендеринга и упрощает привязку данных и подписку на события. 

Кроме того, LitElement расширяет стандартный **жизненный цикл веб-компонентов** (connectedCallback, disconnectedCallback, attributeChangedCallback, adoptedCallback) удобными [дополнениями](https://lit-element.polymer-project.org/guide/lifecycle), делая его во многом схожим с жизненным циклом компонентов популярных фреймворков.

Спешу согласиться, LitElement - это еще одна зависимость, кто-то даже скажет "еще один фреймворк", но я осознанно *решился на эти плюс 7kB,* потому что LitElement смотрится как один из вариантов естественной надстройки над существующим нативным API, которая так или иначе присутствовала бы в прокте, хотя бы для релизации boilerplate-кода. Надеюсь в будущем нативное API станет более удобным и лаконичным.


## Функциональные компоненты
lit-html предоставляет потрясaющую возможность описания интерфейса с помощью функций. Причем обновление DOM-структур таких компонентов происходит оптимизированно - рендерятся только те блоки, значения которых изменились с предыдущего рендера. 

< !!!директивы в сравнении с useState>
styleMap
classMap
ifDefined

< !!!ПРИМЕР c директивами>

https://lit-html.polymer-project.org/guide/template-reference#guard

## css через vars - в качестве кастомизации. 


## Компоненты высшего порядка
HOC - мощный паттерн, часто используемый при работае с React или Vue. При его использовании композиция компонентов становится простой и лаконичной, и хотелось бы использовать какой-то его аналог при работе с веб-компонентами. Так как веб-компоненты - это классы, для себя, в качестве аналога HOC, я решил использовать фунции, которые бы возвращали расширенние класса, переданного им в качестве параметра. 

### Расширение функционала
В проекте мне необходим был redux, поэтому в качестве примера рассмотрю [коннектора для него](https://www.npmjs.com/package/lite-redux). Ниже представлен код декоратора, принимающего store и возвращающего стандартный коннектор redux. Внутри класса происходит накопление mapStateToProps из всей цепочки наследования (для тех случаев если в ней будет HOC, который также общается с redux), чтобы в дальнейшем, когда компонент будет встроен в DOM, одним колбеком все их подписать на изменение состояния redux. При удалении компонента из DOM эта подписка удаляется.

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
Во многих случаях кроме расширения функционала компонента требуется обернуть его отображение. Для этого удобно использовать функцию рендеринга расширяемого компонента. Кроме того, бывает полезно расширить список наблюдаемых свойств для обеспечения реактивности.  Для иллюстрации приведу упрощенный код поля ввода пароля, которое расширяет переданный ей компонент текстового поля ввода:

< !!! static get observedAttributes() {
      return [...super.observedAttributes, "values", "errors", "touched"];
    }>


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

< !!! Ссылки типа "ПРИМЕРЫ HOC" на форму и редакс>

## Формы и UI
Как я уже говорил, основными ui-составляющими моего проекта являются формы и различные поля ввода. Для удобства хотелось использовать компонент реализующий стандартный функционал формы: хранение состояния (values, errors, touched), валидация, обработка submit, reset и предоставляющий удобные инструменты для работы с формой. 

Этот функционал не имеет отношения к теме статьи, но стоит поговорить о том, во что его обернуть. Забегая вперед скажу, что прежде чем сделать рабочий вариант я попробовал 3 упаковки: стандарнтый веб-компонент; расширение встроенного тега ```<form>``` без использования Shadow DOM; компонент высшего порядка. И это повод для небольшой истории...

### Версия 1. Shadow DOM & Slots
Первое что пришло мне на ум - сделать отдельные стандартные веб-компоненты полей ввода и формы, ипользуя Shadow DOM, как часть best practice, обернуть поля ввода в HOC для общения с формой, вставлять форму на старницу отдельным кастомным тегом. 

С одной стороны мне нужен был функцонал тега ```<form>``` (например, вызов submit при нажатии на Enter), но не хотелось в дополнение к кастомному тегу формы писать его каждый раз в Light DOM (дочерние элементы компонента, вроде children в React). С другой, я не хотел использовать передачу render-функции для формы - хотел чтобы верстка смотрелась максимально нативно. И безымянный ```<slot>``` позволил мне это сделать: 

```html
<!-- Использование формы -->
<lite-form>
 <my-custom-input name='login'></my-custom-input>
 <another-custom-input name='name'></another-custom-input>
</lite-form>
```

```js
// Компонент формы
class LiteForm extends LitElement {
  <... функционал формы ...>
  
  render() {
    return html`
      <form @submit=${this.handleSubmit}>
        <slot></slot>
      </form>
    `;
  }
}

customElements.define("lite-form", LiteForm);

```

Это означало, что HOC для общения полей ввода с формой должен быть способен самостоятельно найти свою форму или общаться с ней посредством событий. Перебрав несколько вариантов (общение через шину событий, общий контекст - все они подходили, если форма на странице только одна), я остановился на очень простом, но рабочем:

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

Тут все тривиально: рекурсивный поиск элемента с атрибутом, указывающим что это искомая форма. Хочется отметить, что благодаря функции [getRootNode](https://developer.mozilla.org/en-US/docs/Web/API/Node/getRootNode), поиск происходит сквозь дерево вложенных Shadow DOM - удобная функция при решении таких специфических задач.

< !!! :host-context(body[dir="rtl"]) .option {        text-align: right;      } >
***

репозиторий про исследование
смотреть историю коммитов sso-frontend и библиотек

lite-form: submit, blur 
(когда были slot, когда перестали быть, но использовал расширение html, почему отказался)

no shadowDom (Так у меня в основном получается custom elements??), 

 addClass: { attribute: "classname", reflect: false },
