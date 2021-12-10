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
