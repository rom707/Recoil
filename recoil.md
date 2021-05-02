# Обзор библиотеки Recoil

### Вступление
Recoil - библиотека управления состоянием React-приложений созданная командой из Facebook и основанный на графе потока данных.

Библиотека появилась летом 2020 и ещё находится на раннем этапе разработки. Так чем же очередная библиотека может заинтересовать и стоит ли следить за ее развитием?

### Зачем всё это?
Прежде всего разберемся какие задачи решает библиотека относительно vanilla React, поскольку нередко можно обойтись использованием React Hooks и Context.

Рассмотрим задачу с синхронизацией состояния между компонентами разнесенными по дереву приложения.

Так как общий родитель компонентов находиться на несколько узлов выше, логично использовать React Context для общего состояния.
Если контекст содержит слишком много различных состояний, можно столкнуться с проблемой, когда компоненты которые используют лишь часть из этих состояний будут перерисовываться при любом изменении в контексте.

Можно вынести эти состояния в различные контексты, чтобы этого избежать, однако если такие состояния появляются динамически, то добавить новый контекст в корень дерева приложения будет еще худшим решением.

С этой проблемой и столкнулась команда разработчиков, решение чего вылилось в библиотеку Recoil.

Recoil позволяет строить направленный граф данных с использованием подписок. Компоненты могут создавать общие состояния и подписываться на них. При изменении состояния перерисовываются только подписанные компоненты.

### Основное API

#### Атомы (atoms)
Основная единица состояния это атомы. Компоненты подписываются на атомы и перерисовываются при обновлении. Отличительная особенность атомов — они могут создаваться динамически. Для отладки и согласованности атомов используется ключ, который указывается при создании.
Для создания используется функция `atom()`:

```js
const fontSizeState = atom({
  key: 'fontSizeState',
  default: 14
});
```

Для работы с атомом используется хук аналогичный useState - `useRecoilState()`:

```js
function FontButton() {
  const [fontSize, setFontSize] = useRecoilState(fontSizeState);

  return (
    <button onClick={() => setFontSize((size) => size + 1)} style={{fontSize}}>
      Увеличить размер шрифта
    </button>
  )
}
```
Вспомогательные функции `useRecoilValue` и `useSetRecoilState` заменяют getter и setter соответственно:
```js
  const fontSize = useRecoilValue(fontSizeState);
  const setFontSize = useSetRecoilState(fontSizeState);
```

#### Селекторы (selectors)
Селектор — это чистая функция которая вычисляет значение из переданного на вход атома или другого селектора. Селектор автоматически пересчитывается при изменении атома.

Задается селектор с помощью функции `selector()`:

```js
const fontSizeLabelState = selector({
  key: 'fontSizeLabelState',
  get: ({ get }) => {
    const fontSize = get(fontSizeState)
    const unit = 'px'

    return `${fontSize}${unit}`
  }
})
```

Функция `get()` используется для получения значения из атома или селектора. Селектор имеет схожий интерфейс с атомом, но доступен только для чтения, поэтому можно применить функцию `useRecoilValue()`:
```js
function FontButton() {
  const [fontSize, setFontSize] = useRecoilState(fontSizeState)
  const fontSizeLabel = useRecoilValue(fontSizeLabelState)

  return (
    <>
      <div>Текущий размер шрифта: {fontSizeLabel}</div>

      <button onClick={() => setFontSize(fontSize + 1)} style={{fontSize}}>
        Увеличить размер шрифта
      </button>
    </>
  )
}
```

#### Асинхронные селекторы
Функции в селекторах так же могут быть и асинхронными:
```js
const userNameQuery = selector({
  key: 'CurrentUserName',
  get: userId => async () => {
    const response = await myDBQuery({ userId });
    return response.name;
  },
});

function UserInfo({ userId }) {
  const userName = useRecoilValue(userNameQuery(userId));
  return <div>{userName}</div>;
}
```

Recoil может работать с `React.Suspense` что позволяет обработать ожидание выполнения функции селектора:
```js
<React.Suspense fallback={<div>Loading...</div>}>
  <UserInfo userId={1} />
</React.Suspense>
```

Либо использовать `useRecoilValueLoadable()` чтобы обработать ожидание самостоятельно:
```js
function UserInfo({userID}) {
  const userNameLoadable = useRecoilValueLoadable(userNameQuery(userID));
  switch (userNameLoadable.state) {
    case 'hasValue':
      return <div>{userNameLoadable.contents}</div>;
    case 'loading':
      return <div>Loading...</div>;
    case 'hasError':
      throw userNameLoadable.contents;
  }
}
```

#### Коллекции атомов (atomFamily)
Собственно одна из возможностей, доступная с Recoil - динамическое создание общих состояний, которые перерисовывают только подписанные компоненты.
Функция `atomFamily()` создает коллекцию атомов, в которой атомы создаются и получаются по уникальному ключу:
```js
const elementPositionStateFamily = atomFamily({
  key: 'ElementPosition',
  default: [0, 0],
});

function ElementListItem({elementID}) {
  const position = useRecoilValue(elementPositionStateFamily(elementID));
  return (
    <div>
      Element: {elementID}
      Position: {position}
    </div>
  );
}
```

#### Провайдер (RecoilRoot)
Собственно для того, чтобы использовать Recoil, необходимо обернуть приложение провайдером `RecoilRoot`. Есть поддержка дефолтного состояния:
```js
function MyApp() {
  function initializeState({set}) {
    set(myAtom, 'foo');
  }

  return (
    <RecoilRoot initializeState={initializeState}>
      ...
    </RecoilRoot>
  );
}
```

## Recoil vs Redux
Основное преимущество Recoil перед Redux - производительность

Redux в отличие от Recoil имеет одно централизованное хранилище, которое при обновлении создает полностью новый объект. Это может быть проблемой при хранении больших структур данных.

Также по заявлениям автора Recoil имеет огромное преимущество перед Redux при обновлении компонентов:
>"Well, I know that on one tool we saw a 20x or so speedup compared to using Redux. This is because Redux is O(n) in that it has to ask each connected component whether it needs to re-render, whereas we can be O(1)."

Однако все эти преимущества существенны при условии, что у вас достаточно большое и сложное приложение.
В остальном библиотеке Recoil пока что не может похвастаться тем выбором утилит и дополнительных инструментов созданных под Redux, а также сообществом и опытом использования в реальных проектах.
А большинство приложений сегодня вообще может быть написано без использования библиотек состояния только на React hooks и Context.

## Заключение
Хоть библиотека Recoil и имеет интересный подход она всё ещё достаточно сырая для использования в продакшене: api активно меняется, нет достаточного сообщества для поддержки.

Несмотря на это, у библиотеки есть шансы получить поддержку и популярность хотя бы за счёт проблем, которые она решает. Также, по словам автора, планируется использование `React Concurrent Mode`, что является плюсом.

Среди способов применения можно рассмотреть такие, как приложения с большим количеством виджетов с общими данными или приложение с микрофронтенд архитектурой.

## Источники

- [Сайт Recoil](https://recoiljs.org/)
- [Ознакомительное видео от автора Recoil](https://www.youtube.com/watch?v=_ISAA_Jt9kI)
- [Пример приложения с использованием Recoil](https://medium.com/swlh/learn-to-build-a-covid-tracker-with-react-and-recoil-208446971276)
- [Сравнение с Redux](https://dev.to/emma/redux-vs-recoil-which-should-you-use-57kd)
- [Гайд на русском языке](https://github.com/harryheman/React-Total/blob/main/md/recoil.md)
- [React Concurrent Mode](https://reactjs.org/docs/concurrent-mode-intro.html)
