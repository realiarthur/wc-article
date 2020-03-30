# Веб-компоненты в реальном проекте

Кастомные компоненты, поддерживаемые браузером из коробки и возможность расширять встроенные html-элементы - давно звучало как один из следующих шагов во фронтенд-разработке. Но при выборе технологий для реальных проектов всегда стоит вопрос "Будущее уже настало?"

О самой технологии веб-компонентов уже достаточно написано, и руки давно чесались попробовать ее на реальном проекте. Речь в статье пойдет об этом опыте (а не о технологии), о том с какими проблемами пришлось столкнуться, их решениях, и о том как многие концепции, к которым мы привыкли при работе с фреймворками, легко перекладываются на веб-компоненты. 


## Почему веб-компоненты? 
Проект был небольшим и требовательным к размеру бандла, основными ui-составляющими которого являлись формы. Кто-то скажет, что все можно было сделать нативным html+js, что было бы максимально лаконично. Но если говорить о поддержке и расширении проекта, даже учитывая трабования к размеру бандла, отказываться от всех преимуществ компонентной разработки было бы подобно прыжку в прошлое.

С другой стороны, можно было бы использовать [Svelte](https://github.com/sveltejs/svelte) или, например, [Preact](https://github.com/preactjs/preact). Но попробовать сделать все, оперируя нативными концепциями, было слишком заманчивой идеей. 


## Будущее уже настало? 
Большинство современных браузеров поддерживают веб-компоненты из коробки, включая мобильные. Для остальных есть [полифилы](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). Примечательно то, что для полифилов есть умный [лоадер](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~2kB), который не загружает полифил определенного функционала, если для него есть нативная поддержка браузером - таким образом в современные браузеры ничего не загружается. Полифилы заявляют поддержку вплоть до IE11 и версий Edge, которые не построены на chromium. К счастью, мы не поддерживаем IE в наших проектах, в Edge действительно все работает.

> Небольшие ограничения по работе полифилов:
> - Полифильный тег ```<slot></slot>``` для IE11 & Edge не участвуют во вспытии событий
> - Этот набор полифилов не предоставляет функционал для расширения встроенных html-элементов. Об этом подробнее ниже  


## LitElement
"Как много boilerplate-кода для создания компонентов и реактивной работы с пропсами! Нужно ускорить процесс, и написать класс расширяющий HTMLElement, реализующий этот код" - такой была первая мысль, которая возникла при погружении в веб-компоненты. Писать, конечно, ничего не пришлось - эту работу берет на себя класс [LitElement](https://lit-element.polymer-project.org/) (~7kb). А использующийся в нем [lit-html](https://lit-html.polymer-project.org/) (~3.5kB), предоставляет удобный и оптимизированный механизм рендеринга и упрощает привязку данных и подписку на события. 

Спешу согласиться, LitElement - это еще одна зависимость, еще один фреймворк, но я осознанно решился на эти плюс 7kB, потому что оперируя только нативным API рисковал сильно затянуть проект. LitElement смотрится как один из вариантов естественной надстройки над существующим API, которая так или иначе присутствовала бы во многих проектах, хотя бы для вынесения boilerplate-кода.

## Жизненный цикл
Кроме того, LitElement расширяет стандартный жизненный цикл веб-компонентов удобными [дополнениями](https://lit-element.polymer-project.org/guide/lifecycle), делая его во многом схожим с жизненным циклом компонента React.


## Функциональные компоненты
--- функциональный рендеринг и директивы в сравнении с useState
 
  
## Компоненты высшего порядка
lit-redux
- посмотреть реализацию react redux
- https://github.com/jmas/lit-redux
- написать о том что из замыкания использовать, чтобы все классы обернулись





репозиторий про исследование
смотреть историю коммитов sso-frontend и библиотек

Привычные концепции (жизненный цикл, hoc) 

lite-form: submit, blur 
(когда были slot, когда перестали быть, но использовал расширение html, почему отказался)

redux
css host
no shadowDom, css через vars
