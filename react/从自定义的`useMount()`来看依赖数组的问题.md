## useMount()

### 来源

- 在 `useEffect(callback, []);` 的用法中，有时 `callback` 代码过长，导致看不清依赖数组，或者忘记写空的依赖数组。
- 一个很常见的想法就是封装一个函数。

### 代码

```tsx
const useMount = (callback: EffectCallback) => useEffect(callback, []);

const App = () => {
  const [names, setNames] = useState([]);
  useMount(() => {
    console.log("%cApp.tsx useEffect()", "color: #007acc;");
    setNames([]);
  });
  return <p>{names}</p>;
};
```

## 问题

### eslint 提示

- eslint:react-hooks/exhaustive-deps 提示我们应该将 `callback` 放到依赖数组中。

> React Hook useEffect has a missing dependency: 'callback'. Either include it or remove the dependency array  react-hooks/exhaustive-deps

### 满足 eslint

- 修改后，在 Console 中能看到程序死循环了

```tsx
// 其他代码不变
const useMount = (callback: EffectCallback) => useEffect(callback, [callback]);
```

## 死循环的原因

### 先将函数复原到调用方

- 函数本质就是将重复代码抽离出来，再通过变量交互。因此将函数放回调用方，有时问题更加清晰。

```tsx
const App = () => {
  const [names, setNames] = useState([]);
  
  // 和之前传给 `useMount()` 的参数 `callback` 等价
  const callback = () => {
    console.log("%cApp.tsx useEffect()", "color: #007acc;");
    setNames([]);
  };

  useEffect(callback, [callback]);

  return <p>{names}</p>;
};
```

### 原因分析

- 每次渲染生成的 `callback` 的引用不同，导致每次 `useEffect()` 的依赖数组 `[callback]` 不同。
- `callback()` 每次将 `names` 修改为 `[]`，每次 `[]` 的引用不同。
- 每次 `names` 不同会触发重渲染，重渲染时 `[callback]` 不同会触发修改 `names`。

## 解决办法 1 - 使用 useCallback()

### 还是先用复原函数的方式说明问题

- 通过 `useCallback()` 创建了一个 memoized 函数，因为 `useCallback()` 的依赖数组是空数组，所以变量 `callback` 的引用是恒定的。
- 意味着 `useEffect()` 中的依赖数组 `[callback]` 也是恒定的。

```tsx
const App = () => {
  const [names, setNames] = useState(["Diana"]);

  const callback = useCallback(() => {
    console.log("%cApp.tsx useEffect()", "color: #007acc;");
    setNames(["Bella"]);
  }, []);

  useEffect(callback, [callback]);

  return <p>{names}</p>;
};
```

### 没办法在 useMount() 中使用 useCallback()

- 因为在 `useCallback()` 中用了变量 `callback`，所以 eslint 希望你把 `callback` 放到 `useCallback()` 的依赖数组中。
- 这个问题实际上和最初把 `callback` 放到 `useEffect()` 的依赖数组中一样，都会触发死循环。

```tsx
const useMount = (callback: EffectCallback) => {
  // eslint: React Hook useCallback has a missing dependency: 'callback'.
  // Either include it or remove the dependency array  react-hooks/exhaustive-deps
  const memorizedCallback = useCallback(callback, [])
  useEffect(memorizedCallback, [memorizedCallback]);
};
```

### 在调用方使用 useCallback()

- 缺点是，对于使用者来说有些麻烦

```tsx
const useMount = (callback: EffectCallback) => {
  useEffect(callback, [callback]);
};

const App = () => {
  const [names, setNames] = useState(["Diana"]);

  useMount(
    useCallback(() => {
      console.log("%cApp.tsx useEffect()", "color: #007acc;");
      setNames(["Bella"]);
    }, [])
  );

  return <p>{names}</p>;
};
```

## 解决办法 2 - 屏蔽 eslint

```tsx
// eslint-disable-next-line react-hooks/exhaustive-deps
const useMount = (callback: EffectCallback) => useEffect(callback, []);
```