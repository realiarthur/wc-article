# Компонентная разработка без фреймворков. Веб-компоненты в реальном проекте. !!!Какое название выбрать!!!

Кастомные компоненты, поддерживаемые браузером и возможность расширять встроенные html-элементы - звучит, как один из следующих шагов во фронтенд-разработке. Но при выборе технологий для реальных проектов всегда стоит вопрос "Будущее уже настало?"

Руки давно чесались попробовать веб-компоненты в проде, и я был рад, когда мне предоставилась возможность провести такое исследование на реальном проекте. Речь в статье пойдет об этом опыте, о том с какими проблемами пришлось столкнуться, их решениях, и о том как многие концепции, к которым мы привыкли при работе с фреймворками, легко перекладываются на веб-компоненты.


## Почему веб-компоненты? 
Проект был небольшим и требовательным к размеру бандла, основными ui-составляющими которого являлись формы. Кто-то скажет, что все можно было сделать нативным html+js, что было бы максимально лаконично. Но если говорить о поддержке и расширении проекта, даже учитывая трабования к размеру бандла, отказываться от всех преимуществ компонентной разработки было бы подобно прыжку в прошлое.

С другой стороны, можно было бы использовать [Svelte](https://github.com/sveltejs/svelte) или, например, [Preact](https://github.com/preactjs/preact). Но попробовать сделать все нативно, используя только встроенные концепции и не используя фреймворки, было слишком заманчивой идеей.


## Будущее уже настало? 
Да, большинство современных браузеров поддерживают веб-компоненты из коробки. Для остальных есть [полифилы](https://www.npmjs.com/package/@webcomponents/webcomponentsjs). Примечательно то, что для полифилов есть умный [лоадер](https://www.npmjs.com/package/@webcomponents/webcomponentsjs#using-webcomponents-loaderjs) (~2kB), который не загружает полифил определенного функционала, если есть нативная поддержка браузером. Таким образом в современные браузеры ничего не загружается.

Полифилы заявляют поддержку вплоть до IE11 и ранних версий Edge. К счастью, мы не поддерживаем IE11 в наших проектах, в Edge действительно все работает (самая ранняя зарегистрированная нами версия пользователя - 13). !!!Проверить как в 13 работает!!!

> Небольшие ограничения по работе полифилов:
> - Полифильный тег ```<slot></slot>``` для IE11 & Edge не участвуют во вспытии событий
> - Этот набор полифилов не предоставляет функционал для расширения встроенных html-элементов. Об этом подробнее ниже  





 
  
   

lit-redux
- посмотреть реализацию react redux
- https://github.com/jmas/lit-redux
- написать о том что из замыкания использовать, чтобы все классы обернулись




Полифилы
Lit element (почему не Polymer - потому что LIT это что то среднее между чистыми и фреймворком. Просто обертка упрощающая..)
репозиторий про исследование

Привычные концепции

lite-form: submit, blur

redux
css host
no shadowDom, css через vars
