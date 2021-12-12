## 参考链接

- [React 官方文档 - State and Lifecycle](https://reactjs.org/docs/state-and-lifecycle.html)
- [React Hooks 官方文档 - useState()](https://reactjs.org/docs/hooks-reference.html#usestate)

## 问题原因

- state 更新可能是异步的（[State Updates May Be Asynchronous](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)）。
- 为了提高性能，React 有可能批量更新 `setState()`（[React may batch multiple `setState()` calls into a single update for performance.](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)）。

## 问题复现

### 代码

```tsx
const App = () => {
  const [count, setCount] = useState(0);

  const handleCountClick = () => {
    console.log(`before running, count is ${count}`);
    for (let i = 0; i < 10; i++) {
      setCount(count + 1);
    }
    console.log(`After  running, count is ${count}`);
  };

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={handleCountClick}>Click me</button>
    </div>
  );
};
```

### 输出

- 点击三次 `<button>` 后, Console 的输出。

![](./不要在`setState()`中直接使用`state`-problem.png)

### 原因

- 在例子中，`setCount()` 是异步执行的，因此第一次点击 `<button>` 时创建了十个 `setCount(1)`。

## 解决办法

### 代码

- 使用函数参数形式，修改 state（[To fix it, use a second form of `setState()` that accepts a function rather than an object.](https://reactjs.org/docs/state-and-lifecycle.html#state-updates-may-be-asynchronous)）。

```tsx
const App = () => {
  const [count, setCount] = useState(0);

  const handleCountClick = () => {
    console.log(`before running, count is ${count}`);
    for (let i = 0; i < 10; i++) {
      // 就改了一行，之前是：`setCount(count + 1);`
      setCount((prevCount) => prevCount + 1);
    }
    console.log(`After  running, count is ${count}`);
  };

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={handleCountClick}>Click me</button>
    </div>
  );
};
```

### 输出

- 点击三次 `<button>` 后, Console 的输出。

![](./不要在`setState()`中直接使用`state`-fix.png)