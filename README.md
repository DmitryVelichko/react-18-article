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

