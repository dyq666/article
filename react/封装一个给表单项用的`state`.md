## 用处

表单项经常需要写一个 `onChange` 的处理函数，因此直接封装一个 hook，返回事件处理函数。

## 代码

### 给 string 类型用的

```tsx
const useFormStringItem = <Element extends { value: string }>(
  value: string = ""
) => { 
  const [item, setItem] = useState(value);
  return [
    item,
    (e: ChangeEvent<Element>) => setItem(e.target.value),
    setItem,
  ] as const;
};
```

### 给 boolean 类型用的

```tsx
const useFormBooleanItem = <Element extends { checked: boolean }>(
  value: boolean = true
) => {
  const [item, setItem] = useState(value);
  return [
    item,
    (e: ChangeEvent<Element>) => setItem(e.target.checked),
    setItem,
  ] as const;
};
```

### 待改进？

- 不知道有没有容易的办法将 `useFormBooleanItem()` 和 `useFormStringItem()` 融成一个函数。
- 我尝试了一下，需要各种类型 assertions，感觉不如分开写方便。

### 使用案例

目前使用了五种类型：

- `<input type="text">`
- `<input type="checkbox">`
- `<input type="radio">`
- `<select>`
- `<textarea>`

```tsx
const App = () => {
  const [name, handleNameChange] = useFormStringItem("bella");
  const [isMale, handleIsMaleChange] = useFormBooleanItem();
  const [fruit, handleFruitChange] = useFormStringItem("apple");
  const [description, handleDescription] = useFormStringItem("dancer");
  const [food, handleFoodChange] = useFormStringItem("cola");
  const onSubmit = () => {
    alert(`
      name: ${name}
      isMale: ${isMale}
      fruit: ${fruit}
      description: ${description}
      food: ${food}
    `);
  };
  return (
    <form onSubmit={onSubmit}>
      <input type="text" value={name} onChange={handleNameChange} />
      <input type="checkbox" checked={isMale} onChange={handleIsMaleChange} />
      <select value={fruit} onChange={handleFruitChange}>
        <option value="">默认</option>
        <option value="apple">苹果</option>
      </select>
      <textarea value={description} onChange={handleDescription} />
      {["hamburger", "cola"].map((item) => (
        <input
          key={item}
          type="radio"
          value={item}
          checked={food === item}
          onChange={handleFoodChange}
        />
      ))}
      <input type="submit" value="提交" />
    </form>
  );
};
```