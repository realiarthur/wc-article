# Веб-компоненты в реальном проекте

Изолированные кастомные компоненты, поддерживаемые браузером из коробки и возможность расширять встроенные html-элементы - давно звучало как один из следующих шагов во фронтенд-разработке. Но при выборе технологий для реальных проектов всегда стоит вопрос "Будущее уже настало?"

О технологии веб-компонентов уже достаточно написано, и руки давно чесались попробовать ее на реальном проекте. Речь в статье пойдет об этом опыте (а не о технологии), о том с какими проблемами пришлось столкнуться, их решениях, и о том как многие концепции, к которым мы привыкли при работе с фреймворками, легко перекладываются на веб-компоненты. 


## Почему веб-компоненты? 
Проект был небольшим и требовательным к размеру бандла, основными ui-составляющими которого являлись формы. Кто-то скажет, что все можно было сделать нативным html+js, что было бы максимально лаконично. Но если говорить о поддержке и расширении проекта, даже учитывая трабования к размеру бандла, отказываться от всех преимуществ компонентной разработки было бы подобно прыжку в прошлое.

С другой стороны, можно было бы использовать [Svelte](https://github.com/sveltejs/svelte) или, например, [Preact](https://github.com/preactjs/preact). Но попробовать сделать все, оперируя нативными концепциями, было слишком заманчивой идеей. 


## Будущее уже настало? 
Большинство современных браузеров, включая мобильные, поддерживают веб-компоненты из коробки. Для остальных есть [полифилы](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). Примечательно то, что для полифилов есть умный [лоадер](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~2kB), который не загружает полифил определенного функционала, если для него есть нативная поддержка браузером - таким образом в современные браузеры ничего не загружается. Полифилы заявляют поддержку вплоть до IE11 и версий Edge, которые не построены на chromium. К счастью, мы не поддерживаем IE в наших проектах, в Edge действительно все работает. Так же все работает в китайских UC Browser и QQ Browser, включая их мобильные версии.

> Небольшие ограничения по работе полифилов:
> - Полифильный тег ```<slot></slot>``` для IE11 & Edge не участвуют во вспытии событий
> - Этот набор полифилов не предоставляет функционал для расширения встроенных html-элементов. Об этом подробнее ниже  


## LitElement
"Как много boilerplate-кода для создания компонентов и реактивной работы с пропсами! Нужно ускорить процесс, и написать класс расширяющий HTMLElement, реализующий этот код" - такой была первая мысль, которая возникла при погружении в веб-компоненты. Писать, конечно, ничего не пришлось - эту работу берет на себя класс [LitElement](https://lit-element.polymer-project.org/) (~7kb). А использующийся в нем [lit-html](https://lit-html.polymer-project.org/) (~3.5kB), предоставляет удобный и оптимизированный механизм рендеринга и упрощает привязку данных и подписку на события. 

Кроме того, LitElement расширяет стандартный **жизненный цикл веб-компонентов** (connectedCallback, disconnectedCallback, attributeChangedCallback, adoptedCallback) удобными [дополнениями](https://lit-element.polymer-project.org/guide/lifecycle), делая его во многом схожим с жизненным циклом компонентов популярных фреймворков.

Спешу согласиться, LitElement - это еще одна зависимость, еще один фреймворк, но я осознанно решился на эти плюс 7kB, потому что LitElement смотрится как один из вариантов естественной надстройки над существующим нативным API, которая так или иначе присутствовала бы в прокте, хотя бы для релизации boilerplate-кода. 


## Функциональные компоненты
lit-html предоставляет потрясaющую возможность описания интерфейса с помощью функций. Причем обновление DOM-структур таких компонентов происходит оптимизированно - рендерятся только те блоки, значения которых изменились с предыдущего рендера. 

<директивы в сравнении с useState>

<ПРИМЕР c директивами>

 
  
## Компоненты высшего порядка
HOC - мощный паттерн, часто используемый при работае с React или Vue. При его использовании композиция компонентов становится простой и лаконичной, и хотелось бы использовать какой-то его аналог при работе с веб-компонентами. Так как веб-компоненты - это классы, для себя, в качестве аналога HOC, я решил использовать фунции, которые бы возвращали расширенние класса, переданного им в качестве параметра. 

В проекте мне необходим был redux, поэтому в качестве примера рассмотрю [коннектора для него](https://www.npmjs.com/package/lite-redux). Ниже представлен код декоратора, принимающего store и возвращающего стандартный коннектор redux. Внутри класса происходит накопление mapStateToProps из всей цепочки наследования (для тех случаев если в ней будет HoС, который также общается с redux), чтобы в дальнейшем, когда компонент будет встроен в DOM, одним колбеком все их подписать на изменение состояния redux. При удалении компонента из DOM эта подписка удаляется.

```js
import { bindActionCreators } from "redux";

export default store => (mapStateToProps, mapDispatchToProps) => Component =>
  class Connect extends Component {
    constructor(props) {
      super(props);
      this._getPropsFromStore=this._getPropsFromStore.bind(this)
      this._getInheritChainProps=this._getInheritChainProps.bind(this)

      this._inheritChainProps = (this._inheritChainProps || []).concat(
        mapStateToProps
      );
    }

    _getPropsFromStore (mapStateToProps) {
      if (!mapStateToProps) return;
      const state = store.getState();
      const props = mapStateToProps(state);

      for (const prop in props) {
        this[prop] = props[prop];
      }
    };

    _getInheritChainProps () {
      this._inheritChainProps.forEach(i => this._getPropsFromStore(i));
    };

    connectedCallback() {
      super.connectedCallback();

      this._getPropsFromStore(mapStateToProps);

      this._unsubscriber = store.subscribe(this._getInheritChainProps);

      if (!mapDispatchToProps) return;

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

    disconnectedCallback() {
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

При использовании такого подхода для реализации аналога HOC нужно соблюдать некоторые меры предосторожности: 
- быть внимательным при именовании свойств и методов, так как можно переопределить их в наследуемом компоненте
- не забывать, что при передаче в качестве колбека на какое-либо событие метода класса, есть вероятность что этот метод будет  переопределен где-то в цепочке наследования, и на событие окажется подписанным только последний из "HOC'ов", а не оба
- стараться максимально наглядно обявлять доступные извне (по отношению к "HOC") свойства. 

***





репозиторий про исследование
смотреть историю коммитов sso-frontend и библиотек

Привычные концепции (жизненный цикл, hoc) 

lite-form: submit, blur 
(когда были slot, когда перестали быть, но использовал расширение html, почему отказался)

redux
css host
no shadowDom (Так у меня в основном получается custom elements??), 
css через vars
