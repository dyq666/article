## 用处

- 可以直接在页面级组件修改 document.title
- 退出页面级组件后恢复之前的 document.title

## 代码

- `useRef()` 保存的值在组件生命周期内是不变的（[The returned object will persist for the full lifetime of the component.](https://reactjs.org/docs/hooks-reference.html#useref)）。

```ts
const useDocumentTitle = (title: string) => {
  // 这里实际上可以不用 `useRef()`，由于闭包的特性，`useEffect()` 中用到的
  // `lastPageTitle` 就是组件 mount 时的 title。不过用 `useRef()` 的代码
  // 语义更强，直接表明这个变量在组件 mount 后不会变化。
  const lastPageTitle = useRef(document.title).current;
  
  // 同一个页面级组件中，如果 title 变了，直接修改 document.title
  useEffect(() => {
    document.title = title;
  }, [title]);
  
  // 在退出页面级组件后，恢复成上个页面级组件的 title
  useEffect(() => {
    return () => {
      document.title = lastPageTitle;
    };
  }, []);
};
```

## 使用方式

- 创建了 `<Counter>` 和 `<Name>` 两个组件，`<Counter>` 中修改数字同时可以修改 `document.title`。
- 由 `<Counter>` 切换到 `<Name>` 时，`document.title` 恢复成默认值。

```tsx
const Counter = () => {
  const [count, setCount] = useState(0);
  useDocumentTitle(`计数器 - ${count}`);
  return (
    <button onClick={() => setCount((prevCount) => prevCount + 1)}>
      {count}
    </button>
  );
};

const Name = () => {
  const [name, setName] = useState("Bella");
  useDocumentTitle(`姓名 - ${name}`);
  return <input type="text" onChange={(e) => setName(e.target.value)} />;
};

const App = () => {
  const [isCounterPage, setIsCounterPage] = useState(true);
  return (
    <>
      {isCounterPage ? <Counter /> : <Name />}
      <button
        onClick={() =>
          setIsCounterPage((prevIsCounterPage) => !prevIsCounterPage)
        }
      >
        切换到{isCounterPage ? "姓名" : "计数器"}
      </button>
    </>
  );
};
```
