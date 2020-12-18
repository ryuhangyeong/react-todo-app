# 리액트 할일 관리 애플리케이션 성능 최적화에 대한 정리

## 많은 데이터 렌더링하기

```javascript
function createBulkTodos() {
  const array = [];
  for (let i = 0; i <= 2500; i++) {
    array.push({
      id: i,
      text: `할 일 ${i}`,
      checked: false,
    });
  }

  return array;
}
```

## 크롬 개발자 도구를 통한 성능 모니터링

개발자 도구에서 Performance 탭으로 측정한다. 녹화 방식이기 때문에 사용자 입력 후 녹화중지

## 느려지는 원인 분석

컴포넌트는 다음과 같은 상황에서 리렌더링(재조정)됩니다. 이를 기억하고 있으면 좋습니다.

1. 자신이 전달받은 props가 변경될 때
2. 자신의 state가 바뀔 때
3. 부모 컴포넌트가 리렌더링될 때
4. forceUpdate 함수가 실행될 때

## React.memo

사용법은 매우 간단합니다. 컴포넌트를 만들고 감싸주기만 하면 됩니다.

```javascript
const TodoListItem = ({ todo, onRemove, onToggle }) => {
  return (
    (...)
  );
};

export default React.memo(TodoListItem);
```

이제 해당 컴포넌트는 todo, onRemove, onToggle이 바뀌지 않으면 리렌더링되지 않습니다. 현재 프로젝트에서는 todos 배열이 업데이트되면 onRemove, onToggle 함수도 새롭게 바뀌게 됩니다.
onRemove와 onToggle이 최신 todos를 참자호가 때문에 todos가 바뀌면 새롭게 함수를 만듭니다. 이런 현상을 방지하려면 두 가지 방법이 있습니다. 1) userState 2) useReducer

## useState 함수형 업데이트

기존 setTodos 함수를 사용할 때 새로운 상태를 파라미터에 넣어주었습니다. setTodos를 사용할 때 새로운 파라미터를 넣어주는 댓딘, 상태 업데이트를 어떻게 할지 정의해주는 업데이트 함수를 넣을수도 있습니다. 이를 함수형 업데이트라고 부릅니다.

```javascript
const [number, setNumber] = useState(0);
const onIncreate = useCallback(
  () => setNumber((prevNumber) => prevNumber + 1),
  [],
);
```

이렇게 하면 useCallback 두 번째 인자에 number를 넣어주지 않아도 됩니다.

```javascript
// 변경 전
const onInsert = useCallback(
  (text) => {
    const todo = {
      id: nextId.current,
      text,
      checked: false,
    };
    setTodos(todos.concat(todo));
    nextId.current += 1;
  },
  [todos],
);

// 변경 후
const onInsert = useCallback((text) => {
  const todo = {
    id: nextId.current,
    text,
    checked: false,
  };
  setTodos((todos) => todos.concat(todo));
  nextId.current += 1;
}, []);
```

## useReducer 사용하기

useState 함수형 업데이트를 사용하는 대신 useReducer를 사용해도 onToggle과 onRemove가 새로워지는 문제를 해결할 수 있습니다.

```javascript
function todoReducer(todos, action) {
  switch (action.type) {
    case 'INSERT':
      return todos.concat(action.todo);
    case 'REMOVE':
      return todos.filter((todo) => todo.id !== action.id);
    case 'TOGGLE':
      return todos.map((todo) =>
        todo.id === action.id ? { ...todo, checked: !todo.checked } : todo,
      );
    default:
      return todos;
  }
}

const App = () => {
  const [todos, setTodos] = useReducer(todoReducer, undefined, []);

  const nextId = useRef(1)

  const onInsert = useCallback(text => {
    const todo = {
      id: nextId.current,
      text,
      checked: false,
    };
    dispatch({ type: 'INSERT', todo })
  }, [])
  (...)
}
```

성능은 함수형 업데이트(useState), useReducer와 비슷하기에 자신의 취향에 따라 선택하세요.

## TodoList 컴포넌트 최적화 하기

리스트 관련 최적화를 하는 경우 리스트와 아이템 둘다 최적화를 해주는게 좋습니다. 둘다 React.memo로 감싸주세요.

## react-virtualized를 사용한 렌더링 최적화

현재 우리는 2500개의 아이템을 화면에 출력했습니다. 실제 화면에 나오는 항목은 9개뿐인데 2500개를 렌더링할 필요가 있을까요? 나머지 2491개는 스크롤시에 보여주면 성능이 향상될 것입니다.
이번에는 react-virtualized를 사용하여 이 부분을 최적화하겠습니다.

```
npm i react-virtualized
```

1. 최적화를 수횅하려면 사전에 먼저 해야할 작업이 있는데. 바로 각 항목의 실제 크기를 px 단위로 알아내는 것입니다. 개발자 도구에서 두 번째 항목을 확인해보니 57px입니다.
2. List 설정

```javascript
import React, { useCallback } from 'react';
import TodoListItem from './TodoListItem';
import './TodoList.scss';
import { List } from 'react-virtualized';

const TodoList = ({ todos, onRemove, onToggle }) => {
  const rowRenderer = useCallback(({ index, key, style }) => {
    const todo = todos[index];
    return (
      <TodoListItem
        todo={todo}
        key={key}
        onRemove={onRemove}
        onToggle={onToggle}
        style={style}
      />
    );
  });
  return (
    <List
      className="TodoList"
      width={512} // 전체 크기 (리스트를 감싸는)
      height={513} // 전체 높이 (리스트를 감싸는)
      rowCount={todos.length} // 항목 전체 개수
      rowHeight={57}
      rowRenderer={rowRenderer} // 항목을 렌더링할 때 쓰는 함수
      list={todos} // 배열
      style={{ outline: 'none' }} // List에 기본 적용되는 outline 스타일 제거
    />
  );
};

export default React.memo(TodoList);
```

3. Item 설정

```javascript
import React from 'react';
import {
  MdCheckBoxOutlineBlank,
  MdCheckBox,
  MdRemoveCircleOutline,
} from 'react-icons/md';
import cn from 'classnames';
import './TodoListItem.scss';

const TodoListItem = ({ todo, onRemove, onToggle, style }) => {
  const { id, text, checked } = todo;
  return (
    <div className="TodoListItem-virtualized" style={style}>
      <div className="TodoListItem">
        <div
          className={cn('checkbox', { checked })}
          onClick={() => onToggle(id)}
        >
          {checked ? <MdCheckBox /> : <MdCheckBoxOutlineBlank />}

          <div className="text">{text}</div>
        </div>
        <div className="remove" onClick={() => onRemove(id)}>
          <MdRemoveCircleOutline />
        </div>
      </div>
    </div>
  );
};

export default React.memo(TodoListItem);
```

```scss
.TodoListItem-virtualized {
  & + & {
    border-top: 1px solid #dee2e6;
  }

  &:nth-child(even) {
    background: #f8f9fa;
  }
}
```

render 함수에서 기존에 보여주던 내용을 div로 한번 더 감싸줍니다.
