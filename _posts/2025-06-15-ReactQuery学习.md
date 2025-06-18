---
title: 2025-06-15-ReactQuery基本使用
date: 2025-06-15 16:19:00 +0800
categories: [React]
tags: [ReactQuery]
---
### 背景介绍

最近在做一个TodoList的项目，使用的技术栈是 React + FastAPI 的技术栈，进行到前端请求的部分的时候了解到了 ReactQuery 这个请求库，感觉还不错。

### 快速开始

文章软件版本参考

| 名称      | 版本 |
| ----------- | ----------- |
| NodeJs      | 20.12.1       |
| React   | ^19.1.0        |
| tanstack/react-query   | ^5.80.7        |

### 1.安装

```shell
pnpm add @tanstack/react-query
```

### 2.基本使用

1. 新建一个 React 项目，完成之后项目结构目录如下(使用 JSX 的后缀名可能不一样)

```shell
npx create-vite@latest
```

<pre>
├── public
│   └── vite.svg
├── src
│   ├── App.css
│   ├── App.tsx
│   ├── assets
│   ├── index.css
│   ├── main.tsx
│   └── vite-env.d.ts
</pre>

`QueryClientProvider` 包裹 `App`标签，这样做是为了把你配置好的 `QueryClient` 实例传递给整个 React 组件树，这样你在任何地方就能使用 `useQuery`, `useMutation` 等功能。

```tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import "./index.css";
import App from "./App.tsx";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const client = new QueryClient();

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <QueryClientProvider client={client}>
      <App />
    </QueryClientProvider>
  </StrictMode>
);

```

### 2.1 简单GET请求

以公开接口为例: `https://jsonplaceholder.typicode.com/posts`
1.新建一个Post.tsx 文件，内容如下

```tsx
import { useQuery } from "@tanstack/react-query";

const fetchPosts = async () => {
  const response = await fetch("https://jsonplaceholder.typicode.com/posts");
  if (!response.ok) throw new Error("request fail");
  return response.json();
};

interface Data {
  userId: number;
  id: number;
  title: string;
  body: string;
}

const Post = () => {
  const { data, isLoading, isError } = useQuery({
    queryFn: fetchPosts,
    queryKey: ["post"],
  });

  if (isLoading) {
    return <p>Loading</p>;
  }

  if (isError) {
    return <p>Request Error</p>;
  }
  //   return isLoading && <BarLoader />;

  return (
    <div>
      {data.map((item: Data) => (
        <p>{item.title}</p>
      ))}
    </div>
  );
};

export default Post;


```

### 2.2 POST请求

### 2.3 缓存

### 2.4 请求间隔
