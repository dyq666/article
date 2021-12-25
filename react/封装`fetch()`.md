## 请求方法

- 先写个 enum，五种常用的 method，如果还需要用到其他的 method，再加就行。

```ts
const enum HTTPMethod {
  GET = "GET",
  POST = "POST",
  PUT = "PUT",
  PATCH = "PATCH",
  DELETE = "DELETE",
}
```

## 处理请求数据

- 过滤下空数据
- 如果是 GET 方法，就将数据放到 url 上
- 如果不是 GET 方法，就将数据放到 body 中

```ts
const isEmptyObject = (obj: Record<string, unknown>) => {
  /**
   * 判断对象是否为空。
   *
   * @param obj: 被判断的对象
   */
  return Object.keys(obj).length === 0;
};

const cleanObject = <Type>(obj: Type) => {
  return Object.fromEntries(
    Object.entries(obj).filter(
      ([key, value]) => value !== null && value !== undefined && value !== ""
    )
  ) as Type;
};

// 假设有传入的变量：数据 data；地址 url；方法 method
// qs 为安装的第三方库
const filteredData = cleanObject(data);
url +=
method === HTTPMethod.GET && !isEmptyObject(filteredData)
  ? `?${qs.stringify(filteredData)}`
  : "";
const body = method !== HTTPMethod.GET ? JSON.stringify(filteredData) : null;
```

## 处理 Headers

- 预置一些请求头，比如 `Content-Type`
- headers 可以覆盖预置的请求头

```ts
// 假设有传入的变量：请求头 `headers`
const filteredHeaders = cleanObject({
  "Content-Type": "application/json",
  ...headers,
});
```

## 处理返回的数据

- 即使状态是 404 和 500 这种，`fetch()` 也不会返回 project promise（[The Promise returned from `fetch()` **won’t reject on HTTP error status** even if the response is an HTTP `404` or `500`. ](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API#differences_from_jquery)）。

- 如果是 HTTP 状态是成功，返回拿到数据的 fulfilled promise
- 如果是 HTTP 状态是失败，返回拿到错误信息的 rejected promise

```ts
const response = await fetch(url, {
  method: method,
  headers: filteredHeaders,
  body: body,
});
return response.ok
  ? await response.json()
  : Promise.reject(await response.json());
```

## 封装好的 fetch

```ts
import qs from "qs";

const enum HTTPMethod {
  GET = "GET",
  POST = "POST",
  PUT = "PUT",
  PATCH = "PATCH",
  DELETE = "DELETE",
}

const request = async <Type>(
  method: HTTPMethod,
  url: string,
  data: Record<string, string> = {},
  headers: Record<string, string> = {},
): Promise<Type> => {
  /**
   * 发送 JSON 请求，返回解析后的 JSON 数据。
   *
   * - 过滤 `data` & `headers`
   * - 根据 `method` 判断，将 `data` 放到 url 还是 body 中
   * - 统一处理 401 未登录情况
   * - 如果是 HTTP 状态是成功，返回拿到数据的 fulfilled promise
   * - 如果是 HTTP 状态是失败，返回拿到错误信息的 rejected promise
   *
   * @param method: 请求方法
   * @param url: 请求地址
   * @param data: 请求数据
   * @param headers: 请求 headers，优先级最高，可以覆盖 `request()` 中设定好的 headers
   */
  // 过滤数据和 headers
  const filteredData = cleanObject(data);
  const filteredHeaders = cleanObject({
    "Content-Type": "application/json",
    ...headers,
  });

  // 数据放 url 还是 body 中
  url +=
    method === HTTPMethod.GET && !isEmptyObject(filteredData)
      ? `?${qs.stringify(filteredData)}`
      : "";
  const body = method !== HTTPMethod.GET ? JSON.stringify(filteredData) : null;

  const response = await fetch(url, {
    method: method,
    headers: filteredHeaders,
    body: body,
  });

  return response.ok
    ? await response.json()
    : Promise.reject(await response.json());
};
```

## 多封装几个方法

- 仿照 python [requests](https://docs.python-requests.org/en/latest/) 的用法，增加了一些封装，这样代码中可以清晰的看出来是用的哪种 HTTP Method。
- 一个缺陷是：没找到比较好的办法兼容类型和 rest 参数传值，所以先用了这种笨的方法，手动补齐每个参数。

```ts
export const get = <Type>(
  url: string,
  data: Record<string, string> = {},
  headers: Record<string, string> = {},
) => request<Type>(HTTPMethod.GET, url, data, headers);

export const post = <Type>(
  url: string,
  data: Record<string, string> = {},
  headers: Record<string, string> = {},
) => request<Type>(HTTPMethod.POST, url, data, headers);

export const put = <Type>(
  url: string,
  data: Record<string, string> = {},
  headers: Record<string, string> = {},
  withAuthToken: boolean = true
) => request<Type>(HTTPMethod.PUT, url, data, headers);

export const patch = <Type>(
  url: string,
  data: Record<string, string> = {},
  headers: Record<string, string> = {},
  withAuthToken: boolean = true
) => request<Type>(HTTPMethod.PATCH, url, data, headers);

// delete 是 reversed var，所以命名为 delete_
export const delete_ = <Type>(
  url: string,
  data: Record<string, string> = {},
  headers: Record<string, string> = {},
  withAuthToken: boolean = true
) => request<Type>(HTTPMethod.DELETE, url, data, headers);
```

## 使用案例

- 放到一个文件中，比如叫做 `request.ts`

```ts
import * as request from "util/request";
request.get(url)
```

