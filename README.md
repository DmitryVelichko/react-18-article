# Что нового в React 18?

Официального релиза React 18 пока еще не было, он ожидается в конце 2021 — первом квартале 2022 года (самый свежий релиз — версия React 17.0.2 от 22 марта 2021 года). 

<!--more-->

Но уже доступны предварительные версии, которые предназначены для внутреннего (alfa) и внешнего тестирования (beta).

Посмотреть актуальный список релизов можно [здесь](https://github.com/facebook/react/blob/main/CHANGELOG.md).

Хочется отметить, что процесс разработки новой версии React впервые стал публичным: теперь можно прочитать обсуждение рабочей группы создателей и разработчиков! Обсуждение представлено [здесь](https://github.com/reactwg/react-18/discussions).

Установить последнюю версию React 18 для внешнего теcтирования можно с помощью следующей команды:

`npm install react@beta react-dom@beta`

или

`yarn add react@beta react-dom@beta`

### **В React 18 представлены следующие обновления:**

1. Автоматическая пакетная обработка (батчинг)
2. Конкурентный режим + Новые API
    * Suspense API
    * Root API
3. Новинки SSR 
    * Выборочная гидратация
    * Потоковая отправка HTML
4. Продвинутый Strict Mode

И теперь мы расскажем про каждое из них.

## 1. Автоматическая пакетная обработка (батчинг)

Автоматическая пакетная обработка (батчинг, англ. batch — &laquo;пакет&raquo;. Например, пакет данных) — группировка множественных вызовов обновления состояния в один ререндер приложения для улучшения производительности.

Расскажем подробнее про пакетную обработку.
При использовании `setState` для изменения переменной внутри любой функции, React, вместо рендеринга в каждом `setState`, сначала собирает все `setState`, а затем выполняет их вместе.

При обновлении нескольких состояний нет необходимости перерисовывать приложение несколько раз. Поэтому стоит дождаться обновления всех состояний, после чего перерисовать приложение необходимо только один раз.

```function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    setCount(c => c + 1); // Ререндера не происходит
    setFlag(f => !f); // Ререндера не происходит
    // И только в конце происходит ререндер
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
}
```

Батчинг существовал уже в React 17, но автоматически работал только для обработчиков DOM-событий.

React 18 добавляет возможность автоматического батчинга обновления состояний для таких асинхронных операций как Promise, таймауты, fetch-запросы.

## **Автоматический батчинг можно отменить** 

Если необходимо обновить состояние, сразу прочитать изменения в DOM-дереве и после этого вызвать следующее обновление состояния, то нужно использовать `ReactDOM.flushSync()`.

```
import { flushSync } from 'react-dom'; // Обратите внимание: react-dom, а не react

function handleClick() {
  flushSync(() => {
    setCounter(c => c + 1);
  });
  // Реакт обновил DOM 
  flushSync(() => {
    setFlag(f => !f);
  });
  // Реакт обновил DOM 
}
```

### **Активировать батчинг** 

Активация батчинга в асинхронных операциях будет работать, если используется новый Root API с вызовом метода createRoot.

```
import ReactDOM from 'react-dom';
import App from 'App';

const container = document.getElementById('app');

// Старый вариант
ReactDOM.render(<App />, container);

// Новый вариант
const root = ReactDOM.createRoot(container);

root.render(<App />);

```

Подробнее про батчинг можно прочитать [здесь](https://github.com/reactwg/react-18/discussions/21).

## 2. Конкурентный режим (Concurrent Mode)

Конкурентный режим — это набор новых возможностей, которые помогают приложениям реагировать и корректно адаптироваться к устройствам пользователя и скорости сети.

Конкурентный режим позволяет добиться впечатляющих результатов: быстрой и плавной загрузки компонентов в любом удобном порядке, сверхтекучести интерфейса

Появляются новые инструменты, с которыми легко адаптировать приложение и к возможностям устройства, и к скорости сети:

* **Suspense**, благодаря которому можно указывать порядок загрузки данных.
* **SuspenseList**, который в конкурентном режиме помогает управлять порядком загрузки Suspense.
* **useTransition**, необходимый для создания плавных переходов между компонентами, обернутыми в Suspense (специальный хук, который откладывает обновление компонента до полной готовности и убирает промежуточное состояние загрузки).
* **useDeferredValue**, позволяющий показывать устаревшие данные во время операций ввода-вывода и обновления компонентов (например, когда нужно показать пользователю уже загруженные данные, пока загружаются новые).

В настоящее время для отрисовки компонентов есть два ограничения: мощность процессора и скорость передачи данных по сети. Когда требуется что-то показать пользователю, текущая версия React пытается отрисовать каждый компонент от начала и до конца. Неважно, что интерфейс может зависнуть на несколько секунд. Такая же история с передачей данных: React будет ждать абсолютно все необходимые компоненту данные, вместо того чтобы отрисовывать его по частям.

Конкурентный режим решает перечисленные проблемы. С ним React может приостанавливать, приоритизировать и даже отменять операции, которые раньше были блокирующими. В конкурентном режиме можно начинать отрисовывать компоненты независимо от получения всех данных или только их части.

React в конкурентном режиме может прервать текущее обновление, чтобы сделать что-то более важное, а затем вернуться к тому, что он делал раньше. 
Возможности конкурентного режима уменьшают необходимость применять ожидание (debouncing) и торможение (throttling) в пользовательском интерфейсе.

### **Пара слов про Suspense**

В React 18 представлен Suspense API, который позволяет разбить приложение на более мелкие независимые блоки, которые будут рендериться независимо друг от друга и не будут блокировать остальную часть приложения. В результате пользователи вашего приложения раньше увидят контент и смогут гораздо быстрее начать с ним взаимодействовать.

Suspense предназначен для отображения запасного интерфейса (например, спиннера) во время ожидания дочерних компонентов. Дочерние компоненты в это время могут выполнять асинхронные вызовы API, либо загружаться через lazy load. 

Основное изменение для пользователей заключается в рендере дочерних элементов внутри Suspense:

```const App = () => {
  return (
    <Suspense fallback={<Loading />}>
      <SuspendedComponent />
      <Sibling />
    </Suspense>
  );
};
```
В React 17 компонент `<Sibling />` сначала будет смонтирован, затем будут вызваны его эффекты, после чего он будет скрыт.

В React 18 это поведение исправлено: теперь компонент `<Sibling />` смонтируется только после того, как `<SuspendedComponent />` загрузится. 

Подробнее про изменения Suspense можно прочитать [здесь](https://github.com/reactwg/react-18/discussions/7).

### **Как переключить проект в Concurrent Mode?**

Обращаем внимание на то, что конкурентный режим следует включать, ведь это все-таки режим!

Сначала нужно убрать легаси. Избавляемся от всех устаревших методов в коде и убеждаемся, что их нет в библиотеках. Если приложение без проблем работает в React.StrictMode, то все в порядке. 

Потенциальная сложность — проблемы внутри библиотек. В этом случае нужно либо перейти на новую версию, либо сменить библиотеку. Или же отказаться от конкурентного режима. После избавления от легаси останется только переключить root.

С приходом Concurrent Mode будет доступно три режима подключения root:

* **Старый режим**\
`ReactDOM.render(<App />, rootNode)`\
Рендер после выхода конкурентного режима устареет.

* **Блокирующий режим**\
`ReactDOM.createBlockingRoot(rootNode).render(<App />)`\
В качестве промежуточного этапа будет добавлен блокирующий режим, который дает доступ к части возможностей конкурентного режима на проектах, где есть легаси или другие трудности с переездом.

* **Конкурентный режим**\
`ReactDOM.createRoot(rootNode).render(<App />)`\
Если нет легаси, то проект можно сразу переключить, нужно только заменить в проекте рендер на createRoot.

Подробнее про конкурентный режим можно прочитать [здесь.](https://ru.reactjs.org/docs/concurrent-mode-intro.html)

### **Root API**

Идем дальше. В обновлении нас ждут новый Root API и старый (legacy) Root API. Команда React специально оставила старый Root API, чтобы пользователи, которые обновили версию, могли постепенно перейти на новую, сравнивая при этом ее работу со старой. 

Использование старого Root API будет сопровождаться предупреждением в консоли о необходимости переключения на новую версию. Рассмотрим пример с новым Root API и увидим разницу с текущей реализацией:

```
import * as ReactDOM from 'react-dom';
import App from 'App';

const container = document.getElementById('app');

// До
ReactDOM.render(<App tab="home" />, container);

// После
const root = ReactDOM.createRoot(container);
root.render(<App tab="home" />);
```

Теперь отдельно создается &laquo;корень&raquo; — указатель верхнеуровневой структуры данных, которую React использует для отслеживания дерева для рендеринга. В предыдущих версиях React &laquo;корень&raquo; был недоступен для пользователя, React прикреплял его к DOM-узлу и никуда не возвращал. В новом Root API изменился метод гидратации контейнера. Теперь вместо hydrate нужно писать hydrateRoot: 

```import * as ReactDOM from 'react-dom';
import App from 'App';

const container = document.getElementById('app');

// До
ReactDOM.hydrate(<App tab="home" />, container);

// После
// Создание и рендер с гидратацией
const root = ReactDOM.hydrateRoot(container, <App tab="home" />);
// В отличие от createRoot(), не нужно отдельно вызывать root.render() 
```

Обращаем внимание, что hydrateRoot() принимает JSX вторым аргументом.

Дело в том, что первый рендер клиента является особенным и требует соответствия с серверным деревом.

Если надо обновить &laquo;корень&raquo; приложения после гидратации, можно сохранить его в переменную и вызывать render():

```import * as ReactDOM from 'react-dom';
import App from 'App';

const container = document.getElementById('app');

// Создаем и рендерим корень с гидратацией
const root = ReactDOM.hydrateRoot(container, <App tab="home" />);

// Обновляем корень приложения
root.render(<App tab="profile" />);
```
[Более подробно про Root API в теме на Github.](https://github.com/reactwg/react-18/discussions/5)

## 3. Новинки SSR

В React 18 были также внесены большие улучшения в Server-Side Rendering (SSR).

### **Выборочная гидратация**

Еще одно нововведение в 18-й версии React — так называемая &laquo;выборочная гидратация&raquo;.

Вообще, гидратация существовала и в более ранних версиях React. Это процесс на стороне клиента, во время которого React берет статический HTML, отправленный сервером, и превращает его в динамический DOM, который может реагировать на изменения данных на стороне клиента.

До React 18 гидратация не могла начаться, если не загружался полный код JavaScript для приложения. Для более крупных приложений этот процесс может занять некоторое время.

Но в React 18 `<Suspense>` позволяет гидратировать приложение до того, как дочерние компоненты загрузятся.

React начнет гидратацию компонентов с учетом взаимодействия пользователя с содержимым сайта. Например, при клике на комментарий React приоритезирует гидратацию HTML родительских блоков. Это называется выборочной гидратацией.

Еще одна особенность в том, что React не будет блокировать UI во время гидратации — этот процесс будет происходить во время простоя браузера, поэтому пользовательские события будут обрабатываться сразу.

### **Потоковая отправка HTML**

Позволяет отправить HTML клиенту без загрузки всех данных для рендера на сервер. А как только данные будут получены, они отрендерятся на сервере и отправятся на клиент. Например, есть блок с комментариями, и мы можем асинхронно загружать по ним информацию, отдавая HTML. А когда комментарии будут получены, отрендерить их и отправить клиенту в конце HTML-документа.

```<div hidden id="comments">
  <!-- Comments -->
  <p>First comment</p>
  <p>Second comment</p>
</div>
<script>
  // Примерная реализация
  document.getElementById('sections-spinner').replaceChildren(
    document.getElementById('comments')
  );
</script>
```
[Более подробно про SSR.](https://github.com/reactwg/react-18/discussions/37)

## 4. Продвинутый Strict Mode

Следующей особенностью React 18 станет улучшение режима Strict Mode. В него добавится новый режим под названием &laquo;strict effects&raquo;. Чтобы понять, что это за режим, вспомним, как Strict Mode работал до обновления. 

Компоненты, обернутые в `<StrictMode>` (только в dev режиме), умышленно рендерятся по два раза, чтобы избежать нежелательных сайд-эффектов, которые можно добавить в процессе разработки.  


С релизом React 18 в StrictMode добавляется новое поведение — &laquo;strict effects&raquo;. С ним эффекты для вновь смонтированных компонентов вызываются дважды (mount -> unmount -> mount).

Дополнительный вызов эффекта не только обеспечивает устойчивую работу компонента, но и необходим для правильной работы Fast Refresh, когда компоненты монтируются/размонтируются при обновлении в процессе разработки. Также это необходимо для работы новой фичи Offscreen API, находящейся в разработке.

[Более подробно про обновленный Strict Mode](https://github.com/reactwg/react-18/discussions/19).

### **Заключение**

Данный список новинок в React 18 далеко не полный, и ближе к релизу нас ждет еще много нового и интересного, в особенности - обновленная документация.
Об этом мы расскажем в отдельной статье.

Коллеги, следите за релизом новой версии React 18 [здесь.](https://github.com/reactwg/react-18/discussions/9)



# Документация по Unit-тестированию


Unit-тестирование (проверка на корректность работы отдельных модулей программы).

Для юнит-тестирования можно использовать, в зависимости от требований к проекту, фреймворк `Jest`, библиотеки `React Testing Library` и `react-test-renderer`.

Рассмотрим их по отдельности.

### Unit-тестирование с помощью Jest

1. Установите Jest с помощью `npm` или `yarn`:

`npm install -D jest`

`yarn add -D jest`

Также можно установить модуль `@types/jest` для текущей версии Jest, если вы используете TypeScript. Для этого используйте команду:

`yarn add -D @types/jest`

Для модулей `@types/*` рекомендуется сопоставлять версию Jest с версией связанного модуля. Например, если вы используете 26.4.0 версию Jest, то использование 26.4.x из `@types/jest` является идеальным. В целом, старайтесь максимально приблизить мажорную (26) и минорную (4) версию к текущей.

2. Далее добавляем в `package.json` следующий код:

```ts
{
  "scripts": {
    "test": "jest"
  }
}
```

Конфигурация для Jest может быть описана в файле `package.json`, `jest.config.js` или `jest.config.ts`.

Например, для `package.json`, вам нужно создать свойство под именем "jest" на самом верхнем уровне в `package.json`, иначе Jest не сможет корректно загрузить конфигурационные данные:

```ts
{
  "name": "my-project",
  "jest": {
    "verbose": true
  }
}
```

Либо, используя команду `jest --init` в корне проекта, вы получите файл с настройками `jest.config.js`.

Более подробно про настройку Jest вы можете прочитать в [официальной документации](https://jestjs.io/ru/docs/configuration).

3. Теперь необходимо написать сами тесты.

Допустим, ваш компонент `Component.tsx` хранится в папке `Component` (путь от корня проекта: `проект/src/components/Component`).

Чтобы протестировать данный компонент, необходимо создать папку `tests` в папке `Component`.

Далее в папке `tests` создаем файл под названием `Component.test.tsx`.

В этом файле мы и напишем тесты для данного компонента.

Создаем тестовый блок с использованием метода `describe()`, который объединяет несколько связанных между собой тестов.

```ts
    describe('Компонент Component', () => {
        // тесты
    });
```

Далее с помощью метода `it()` или аналогичного `test()` мы напишем простой тест. Оба этих метода по сути являются фактическим тестовым блоком.
Функциональной разницы между ними нет, `it()` был добавлен позднее для более понятного описания работы метода.

```ts
    describe('Компонент Component', () => {
        it('Максимальное число', () => {
            expect(Math.max(1, 2, 3)).toBe(3); // Здесь мы ожидаем, что результат выполнения функции 
                                               // будет совпадать с конкертным числовым значением.
        });
    });
```

Внутри функции теста мы сначала вызываем метод `expect()`. Ему мы передаем функцию, результат выполнения которой мы хотим проверить на эквивалентность. 

Метод `toBe()` мы используем для сравнения двух значений – ожидаемого и полученного в результате работы функции.

Более подробно про другие доступные методы `expect` вы можете прочитать в этом разделе [официальной документации](https://jestjs.io/ru/docs/expect).

4. Сохраним файл и запустим тесты с помощью команды `yarn test`.

После запуска можем посмотреть в терминале отчет о работе теста:

```
     ✓ Максимальное число (1 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.507 s, estimated 1 s
```

В данном случае мы видим, что все работает так, как мы задумали.


Ссылка на [официальную документацию](https://jestjs.io/ru/) по использованию Jest в проектах.


### Unit-тестирование с помощью `React Testing Library`

React Testing Library является подмножеством семейства пакетов @testing-library.

1. Установить библиотеку `@testing-library/react` можно с помощью следующих команд:

`npm install -D @testing-library/react` или `yarn add -D @testing-library/react`


2. Далее установите библиотеку `@testing-library/jest-dom` с помощью команды:

`npm install -D @testing-library/jest-dom` или `yarn add -D @testing-library/jest-dom`


3. И затем установите библиотеку `@testing-library/user-event` с помощью команды:

`npm install -D @testing-library/user-event` или `yarn add -D @testing-library/user-event`


4. В файле `package.json` в разделе `dependencies` у вас должны появится следующие зависимости:

```json
  "dependencies": {
    "@testing-library/jest-dom": "^4.2.4", // Версии пакетов в вашем конкретном случае могут отличаться.
    "@testing-library/react": "^9.3.2",
    "@testing-library/user-event": "^7.1.2",
  }
```
Данные пакеты предназначены специально для тестирования.

* `@testing-library/jest-dom`: предоставляет специальные средства сопоставления элементов DOM для Jest.

* `@testing-library/react`: предоставляет API для тестирования React-приложений.

* `@testing-library/user-event`: обеспечивает расширенную симуляцию взаимодействия с браузером.

5. Создадим файл `Component.test.tsx` и напишем тест для компонента `Component`:

```ts
import React from 'react';
import { render } from '@testing-library/react';
import Component from './Component';

describe("Компонент Component", () => {
  test("Компонент Component рендерится корректно", () => {
    const { getByText } = render(<Component />); // Метод render() отображает компонент <Component /> и возвращает объект, который деструктурирован для запроса getByText. Этот запрос находит элементы в DOM по их отображаемому тексту.
    expect(getByText(/Искомый текст/i)).toBeInTheDocument(); // Текст сопоставляется с регулярным выражением /искомый текст/i. Флаг i делает регулярное выражение не чувствительным к регистру. Здесь мы ожидаем, что 'искомый текст' будет найден в документе.
  });
});
```

6. Сохраним файл и запустим тесты с помощью команды `yarn test`.

После запуска можем посмотреть в терминале отчет о работе теста:

```
     ✓ Компонент Component (1 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.612 s, estimated 1 s
```

Ссылка на [официальную документацию](https://testing-library.com/docs/react-testing-library/intro/) по использованию библиотеки `React Testing Library` в проектах.

### Unit-тестирование с помощью `react-test-renderer`

1. Установить библиотеку `react-test-renderer` можно с помощью следующих команд:

`npm install -D react-test-renderer` или `yarn add -D react-test-renderer`

2. В файле `package.json` в разделе `dependencies` у вас должны появится следующая зависимость:

```json
  "devDependencies": {
    "react-test-renderer": "^17.0.2" // // Версия пакета в вашем конкретном случае могут отличаться.
  }
```

Пакет `react-test-renderer` облегчает получение снимка иерархии представления (чем-то похожего на DOM-дерево), отрендеренного компонентом React DOM.

3. Создадим файл `Component.test.tsx` и напишем тест для компонента `Component`:

```ts
import React from 'react';
import { testComponentRender } from '../../../utils/testingUtils';
import renderer from 'react-test-renderer';

describe('Компонент Component', () => {
  testComponentRender(renderComponent); // Рендерим компонент

  it('snapshot совпадает', () => {
    const tree = renderer.create(<Component />).toJSON();

    expect(tree).toMatchSnapshot(); // Проверяем отрендеренный компонент на соответствие снимку (snapshot-y) компонента.
  });
```


4. Сохраним файл и запустим тесты с помощью команды `yarn test`.

После запуска можем посмотреть в терминале отчет о работе теста:

```
     ✓ Компонент Component (1 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.429 s, estimated 1 s
```

Ссылка на [официальную документацию](https://ru.reactjs.org/docs/test-renderer.html) по использованию библиотеки `react-test-renderer` в проектах.

# Документация по QM

# Содержание
1. Вступление (коротко о QM)
2. Как собрать запускаемую утилиту QM из исходного кода
3. Как запускается линтинг посредством QM
4. Как запускаются автотесты всех видов с помощью QM
5. Как расширить QM и добавить новый способ контроля качества
6. Как QM используется на стажировках, какие настройки нужно сделать
7. Как установить QM на сервер 
8. Как выполнять все QM-проверки через CI/CD
9. Как выполняется интеграция с мониторингом


## Quality Management (Управление качеством, далее – QM).

Данный инструмент проводит анализ исходного кода и формирует метрики по результатам анализа. Метрики можно увидеть на мониторинговой панели.

Применять продукт QM возможно только в том случае, если ваш проект разработан на одной из следующих платформ: Java, .Net, Go и Node.js. Продукт QM можно найти по этой [ссылке](https://qm.gnivc.ru/).

GNIVC QM состоит из двух основных блоков:

1. Запуск различных проверок и сбор метрик с этих проверок: статический анализ кода линтером, автотесты и проверка с помощью CI-платформы SonarQube.

2. Аккумуляция этих данных и представление их в наглядном виде с помощью таблиц и диаграмм с целью получения общей оценки проекта. 

Важное замечание: QM не проводит проверки. Данная утилита просто инициализирует запуск той или иной проверки, собирает и хранит полученные результаты, которые в дальнейшем могут быть использованы для наглядной оценки проекта.

## Как собрать запускаемую утилиту QM из исходного кода

Итак, у нас есть исходный код в репозитории [QM](https://dev.gnivc.ru/gnivc/qm/src/branch/last).

Сначала необходимо создать один исполняемый бинарный файл `qm` для вашей операционной системы с помощью npm-модуля `pkg`.

Для этого нам нужно скачать и установить npm модуль `pkg`. Здесь вы найдете [подробную документацию](https://www.npmjs.com/package/pkg).

Данный модуль позволяет упаковать проект в исполняемый файл для любой операционной системы (Windows, Linux, MacOS), который можно запускать даже на устройствах без установленного Node.js.

Установить можно с помощью команды `npm` или `yarn` (глобально):
`npm i -g pkg`

или 

`yarn add pkg -g`

Для просмотра просмотра полного списка опций выполните команду `pkg --help`.

Чтобы создать исполняемый бинарный файл для вашей операционной системы, следует выполнить следующие действия:

1. Перейти в папку проекта, в терминале ввести:

`pkg packages/qm-cli/index.js -o qm`

2. После выполнения данной команды в корне проекта появятся бинарный файл для вашей операционной системы под названием `qm`.

Например, если у вас Windows, то вы автоматически получите файл `qm.exe`. С помощью опции `-o` вы можете назвать свой файл любым именем, в данном случае мы назвали его `qm`.

С помощью опции `-t` или `--targets` можно уточнить, для каких конкретно операционных систем вы хотите сгенерировать исполняемый файл. 
Например, вот так мы можем сгенирировать исполняемый файл для платформы macOS, node 14-й версии:

`pkg packages/qm-cli/index.js -t node14-macos`

3. Для запуска исполняемого файла следует ввести в командной строке команду на запуск, например, для Windows это будет `.\qm.exe`.

В консоли появится полный список опций, которые можно будет применить для запуска определенных проверок:

```s
Commands:
  run-tests [options]     Run tests
  run-linting [options]   Run Linting
  run-analysis [options]  Run Analysis
  project [options]       Projects managing in QM
  part [options]          Projects parts managing in QM
  help [command]          display help for command
  ```
Теперь, если вы хотите запустить, например, автотесты, следует выполнить следующую команду:

`.\qm.exe run-tests`

Итак, имея бинарный файл `qm` на вашем компьютере, вы можете локально запустить утилиту QM и проверить свой код перед тем, как отправлять коммит в удаленный репозиторий.


## Как запускается линтинг посредством QM

В качестве линтера будет использоваться линтинг ГНИВЦ [`@ff/linting`](https://dev.gnivc.ru/gnivc-ff/linting).

Для запуска линтера нужно запустить исполняемый бинарный файл qm вместе с командой `run-linting [options]`, например:

`./index-linux run-linting`

Либо можно использовать значение свойства `bin` в качестве точки входа:

`node packages/qm-cli/index.js run-linting`

Результат линтинга будет выведен в консоль по умолчанию.

Рассмотрим различные виды `[options]`, которые нам доступны. Они добавляются после команды `run-linting`.

1. Для CI нужно добавить флаг `--ci`, тогда результат будет распарсен и сложен в хранилище Prometheus:

`node точка входа run-linting --ci`

2. Опция `--console` выводит данные в консоль.

## Как запускаются автотесты всех видов с помощью QM

Чтобы запустить автоматические тесты локально, используйте следующую команду:

`.\qm.exe run-tests`

Данная команда по умолчанию распознает платформу (например, node) и запускает unit-тестирование.

Теперь давайте рассмотрим различные опций, которые нам доступны. Они добавляются после команды `run-tests`.

1. С помощью опции `-t` или `--type` можно выбрать отдельные виды тестов.

Например, вот так можно запустить unit-тесты:

`.\qm.exe run-tests --type unit`

В этом случае будут запущены все unit-тесты, лежащие в папке у каждого компонента.

А вот так запускаются end-to-end тесты:

`.\qm.exe run-tests --type e2e`

В этом случае будут запущены все e2e-тесты, лежащие в отдельной папке `tests`.

2. Опция `-p` или `--platform` позволяет задать платформу. По умолчанию будет использован node. Вот так можно явно задать GoLang:

`.\qm run-tests -p go`  

3. По умолчанию, по результатам тестирования пользователь получает текстовый лог на выходе в консоль.

4. Если вы хотите запустить тесты в CI-среде, то нужно добавить опцию `--ci`. Тогда результат тестирования отправлен в систему мониторинга [Prometheus](https://prometheus.io/) для дальнейшего визуального отображения с помощью приложения для интерактивной визуализации [Grafana](https://prometheus.io/).

Для CI используйте следующую команду:

`.\qm run-tests --ci`

## Как выполнять все QM-проверки через CI/CD (раздел рекомендован для пользователя)

CI/CD (Continuous Integration, Continuous Delivery — непрерывная интеграция, непрерывная поставка) — это технология автоматизации тестирования и поставки новых модулей разрабатываемого проекта заинтересованным сторонам (разработчики, аналитики, инженеры качества, конечные пользователи и др.).
Чтобы выполнить все QM-проверки и отобразить результаты в Grafana, нужно в корне проекта создать файл `.drone.yml` *(точка в названии)* и настроить должным образом.  
Рекомендуется отправлять результаты проверок на сервер только для ветки `master`.

```yml
kind: pipeline
type: docker
name: ui-constructor-frontend

steps:
  - name: install
    image: docker.gnivc.ru/front-dev-manager:1.0.4
    commands:
      - yarn install

  - name: linting
    image: docker.gnivc.ru/qm:1.1.0
    commands:
      - qm run-linting --ci
    depends_on:
      - install
    when:
      branch:
        - master

  - name: unit-testing
    image: docker.gnivc.ru/qm:1.1.0
    commands:
      - qm run-tests --type unit --ci
    depends_on:
      - install
    when:
      branch:
        - master
	 
  - name: e2e-testing
    image: docker.gnivc.ru/qm:1.1.0
    commands:
      - qm run-tests --type e2e --ci
    depends_on:
      - install
    when:
      branch:
        - master

  - name: deploy
    image: docker.gnivc.ru/front-dev-manager:1.0.4
    volumes:
      - name: demos
        path: /var/www/demos
    commands:
      - front-dev-manager build --repo-url ${DRONE_REPO_LINK} --repo-ref ${DRONE_BRANCH} --run-deploy
    depends_on:
      - linting
      - unit-testing
	    - e2e-testing
    when:
      branch:
        - master
        - develop
        - feature/*-demo

trigger:
  event:
    exclude:
      - pull_request
  branch:
    - master
    - develop
    - feature/*
```

После прохождения всех проверок в дашборде Grafana вашего проекта отобразятся метрики по результатам этих проверок (задержка может составлять до 5 минут):

## Как установить QM на сервер (раздел рекомендован для разработчика/администратора)

QM можно установить на свой собственный сервер, мы же рассмотрим установку на примере сервера **GNIVC**. *(На данный момент QM принимает запросы только с IP-адреса сервера **GNIVC**, на котором расположен Drone CI)*.

### Копирование QM на сервер

1. На локальной машине переходим в папку  **qm-backend**:  
	`cd proj/gnivc/qm/packages/qm-backend`
3. Копируем содержимое на сервер *(будет запрошен пароль)*:  
	`pscp -r * root@11.111.11.111/usr/local/qm-backend`
    - **pscp** - утилита для безопасного копирования и передачи файлов по SSH. [Скачать.](https://putty.org.ru/download.html)
    - **-r** - флаг, позволяющий рекурсивно пройтись по всем поддиректориям (иначе скопируются только файлы из данной директории).
    - **\*** - символ, означающий выбор всех файлов.
    - **root** - имя пользователя.
    - **11.111.11.111** - IP-адрес сервера.
    - **usr/local/qm-backend** - директория, куда следует скопировать данные.
4. Дожидаемся окончания процесса, после чего переходим к сборке и запуску.

### Сборка и запуск QM

1. Подключаемся к серверу по SHH *(будет запрошен пароль)*:  
	`ssh root@11.111.11.111`
2. Далее переходим в папку, куда ранее был скопирован проект:  
	`cd /usr/local/qm-backend`
3. Выводим содержимое директории, чтобы убедиться, что все было успешно скопировано:  
	`ls -l`
	4. Собираем образ проекта:  
	`docker build -t gnivc/qm-backend:1.1.0 .`
    - **gnivc/qm-backend** - название образа.
    - **:1.1.0** - версия образа.
    - **.** - точка в конце означает, что **dockerfile** берется из текущей директории. 
5. Выводим список образов и ищем свой, чтобы убедиться, что образ был собран:  
	`docker images`
6. Запускаем контейнер:  
	`docker run -it --publish 5000:5000 gnivc/qm-backend:1.1.0`
    - **-it** - два совмещенных флага **-i** и **-t**. Подробнее о них можно узнать в документации к docker.
    - **--publish 5000:5000** - читается порт 5000 в контейнере и публикуется на сервере на 5000 порту.
    - **gnivc/qm-backend:1.1.0** - название нашего образа.

Посмотреть список запущенных докер-контейнеров можно командой: `docker ps`

## Как выполняется интеграция с мониторингом (раздел рекомендован для разработчика/администратора)

### Просмотр результатов анализа на дашборде  в Grafana
Для просмотра результатов анализа проверок на дашборде в Grafana следует выполнить следующие действия:
1. Переходим на сайт проекта QM - https://qm.gnivc.ru (не забываем включить VPN).  
2. Выполняем вход в роли Админа или Супер Админа.
3. Выбираем проект и нажимаем кнопку "Подробнее", после чего выполняется переход на страницу с отображением результатов анализа проверок выбранного проекта на дашбордах в `Grafana`.

### Добавление проекта и его интеграция с мониторингом
Для добавления проекта следует выполнить следующие действия (доступно только Админу или Супер Админа): 
1. На [сайте проекта QM](https://qm.gnivc.ru) нажимаем на кнопку "Создать проект".
2. В появившемся окне вводим название нового проекта. Сохраняем изменения.
![Создание нового проекта](https://dev.gnivc.ru/gnivc/qm/raw/commit/566c447ac0bc0cd04a49f9ff5cf469d4d422d7ac/qm-add-project-1.jpg)

Для проверки части проекта следует выполнить следующие действия:
1. Нажимаем на кнопку "Добавить часть проекта".  
![Добавление части проекта](https://dev.gnivc.ru/gnivc/qm/raw/commit/566c447ac0bc0cd04a49f9ff5cf469d4d422d7ac/qm-add-project-2.jpg)

2. Заполняем все поля:
   - название может быть любым;
   - репозиторий - добавляем ссылку на репозиторий для проверки
   - ветка - указываем ту ветку, в которой нужно будет выполнить проверку
   - платформа - выбираем нужную из представленных вариантов;
   - поддиректория - указываем в том случае, если требуется запускать в отдельной поддиректории проекта (т.е. если в проекте есть несколько папок, то указываем в поддиректории ту папку, в которой следует выполнить проверку). Если в вашем проекте моно репозиторий (нет подпапок в репозитории проекта), то данный пункт пропускаем.
   - метрики - выбираем те проверки, которые следует провести для проекта.

Пример заполнения полей:
![Пример заполнения полей](https://dev.gnivc.ru/gnivc/qm/raw/commit/566c447ac0bc0cd04a49f9ff5cf469d4d422d7ac/qm-add-project-3.jpg)

3. Нажимаем кнопку "Сохранить".

Если необходимо произвести изменения части проекта, то следует выполнить следующие действия:
1. Изменяем необходимые поля.
2. Чтобы отменить все внесенные изменения следует нажать кнопку "Отменить".
3. Нажимаем кнопку "Обновить" для создания обновленной версии `.drone.yml`.

Количество звезд в ранге отображают количество использовать метрик в проекте. Ранг будет пересчитываться после выполнение проверки.

## Как расширить QM и добавить новый способ контроля качества?

В QM имеется возможность добавить свои собственные метрики или дополнить уже имеющиеся. Алгоритм действий достаточно прост:
  - Написать дополнительно CLI-команду для новой проверки.
  - Написать скрипт с логикой проверки.
  - Добавить новые проверки в Grafana.

Разберем код на примере уже существующей проверки `run-tests`, свои собственные проверки будут разрабатываться по аналогии с уже имеющимися.

### CLI-команда

В проекте [QM](https://dev.gnivc.ru/gnivc/qm/src/branch/last) CLI-команды находятся в файле `packages/qm-cli/index.js`:

```js
program
  .command('run-tests') // Название команды
  .description('Run tests') // Описание команды
  .option('-p, --platform <platform>', 'Platform', 'node') // Флаг для платформы (node, go, ...)
  .option('-t, --type <type>', 'Test type', 'unit') // Флаг для типа тестов
  .option('--ci', 'Run ci', false)  // Флаг для вывода результатов проверки в Grafana
  .option('--console', 'Output in console', true) // Флаг для вывода результатов в консоль
  .action(async (...args) => { // args - флаги, добавленные при запуске команды run-tests
    const { ci, console: toConsole, type } = args[0];

  console.log('>>>> Run Tests <<<<');
    if (type === 'unit') {
      console.log('>>>> Run Unit Tests <<<<');
      await unitTestingHandler(ci, toConsole); // Код функции указан ниже
      console.log('>>>> Unit Tests Finished <<<<');
    }
    if (type === 'e2e') {
      console.log('>>>> Run E2E Tests <<<<');
      await e2eTestingHandler(ci, toConsole);
      console.log('>>>> E2E Tests Finished <<<<');
    }
    console.log('>>>> Tests Finished <<<<');
  });
```

`unitTestingHandler()`:
```js
const unitTestingHandler = async (ci, toConsole) => {
  const { metrics, log } = await unitTestingRun(ci); // Функция с логикой данной проверки (ниже)
  if (toConsole) {
    if (!metrics.numFailedTests) {
      console.info('All unit testing success');
      console.log(metrics);
    } else {
      log();
    }
  }
};
```

### Основная логика проверки

Основной код должен находиться по адресу `packages/qm-core-название-проверки/index.js`  
У команды `run-tests` существует два подтипа проверки - **unit** и **e2e**, мы рассмотрим только **unit** - `packages/qm-core-jest-unit-testing/index.js`:


```js
const _ = require('lodash');
const {sendMetricHighLevel} = require('@sandbox/qm-core-monitoring');
const util = require('util');
const path = require('path');
const fs = require('fs');
const jsonfile = require('jsonfile');

const exec = util.promisify(require('child_process').exec);
const readFile = util.promisify(require('fs').readFile);
const rm = util.promisify(require('fs').rm);

// Имя файла с результатами проверки
const filename = './qm-unit-testing.json';

const run = async (ci) => {
  // Устанавливаем необходимые пакеты для работы нашей проверки
  const installJest = async () => {
    await exec(`yarn add -D jest`);
  };
  
  const run = async (ci) => {
  // Устанавливаем необходимые пакеты для работы нашей проверки
  const installJest = async () => {
    await exec(`yarn add -D jest`);
  };
  
    // Команда запуска проверки
  const runYarnReactScripts = async () => {
    await exec(`yarn react-scripts test --json=true ${ci ? '' : '--watchAll=false'} --outputFile=${filename}`); 
  };
  
  // Еще одна команда запуска проверки
  const runYarnJest = async () => {
    await exec(`yarn jest --json=true ${ci ? '' : '--watchAll=false'} --outputFile=${filename}`); 
  };
  try {
    const pathToPackageJson = path.join(process.cwd(), 'package.json');
  
   // Если package.json не найден, устанавливаем необходимые пакеты
    if (!fs.existsSync(pathToPackageJson)) {
      await installJest();
    }
