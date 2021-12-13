## 用处

- 一个常见的场景就是搜索框。为了节省资源，我们希望在用户一定时间不输入内容后，再给出搜索提示信息。

## 代码

- 修改输入框后，`name` 是实时更新的，`debouncedName` 1 秒后才更新。
- 可以监听 `debouncedName` 代替 `name` 去做一些事情了，比如发送请求。

```tsx
const useDebouncedState = <Type extends {}>(value: Type, ms: number) => {
  const [debouncedValue, setDebouncedValue] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), ms);
    return () => clearTimeout(timer);
  }, [value, ms]);
  return debouncedValue;
};

const App = () => {
  const [name, setName] = useState("Bella");
  const debouncedName = useDebouncedState(name, 1000);
  const handleNameChange = (e: ChangeEvent<HTMLInputElement>) =>
    setName(e.target.value);
  return (
    <>
      <input type="text" value={name} onChange={handleNameChange} />
      <p>name is: {name}</p>
      <p>debouncedName is: {debouncedName}</p>
    </>
  );
};
```

## 一些核心思路

- 传入的 `value` 通常是一个 state，当 `value` 改变后，组件触发重渲染。
- 因为 `useDebouncedState()` 中的 `useEffect()` 的依赖数组 `[value, ms]` 发生了变化，执行 `useEffect()`。
- `useEffect()` 中创建了一个定时器，等待 `ms` 后修改 `debouncedValue`。
- 如果 `ms` 中没有修改 `value`，那么 `ms` 后由于 `debouncedValue` 发生了变化，触发重渲染（`debouncedValue` 也是一个 state！！）。这次渲染时，就可以通过 `useDeounbcedState()` 拿到最新值了。
- 如果 `ms` 内修改了 `value`，那么 `useEffect()` 将注销掉上次的定时器，创建一个新的，然后继续等待 `ms`。

```tsx
const useDebouncedState = <Type extends {}>(value: Type, ms: number) => {
  const [debouncedValue, setDebouncedValue] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), ms);
    return () => clearTimeout(timer);
  }, [value, ms]);
  return debouncedValue;
};
```
