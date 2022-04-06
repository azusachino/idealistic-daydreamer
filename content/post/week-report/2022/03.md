---
title: Weekly Report 2022.03
description: Hard to love
date: 2022-01-16
slug: 2022-03-week-report
image: img/2022/01/WinterBison.jpg
categories:
  - week-report
tags:
  - week-report
---

Illness makes people weak.

## Reading

### 变形记

变形记是一面镜子，帮助人们看到生活中真正“变形”的家伙们。

### 噪声

> 噪声是无规律的错误，偏差是系统性的错误。

无论偏差的大小如何，减少噪声都有益处。(评分时去掉最高分和最低分)

> 在充满噪声的系统中，**错误不会相互抵消，只会累加。**

在给一项保险套餐拟定价格的时候，价格过高可能会导致该套餐无法售出足够的份额，价格过低可能会导致利润率偏低；它们之间的共同点是，无法带来实际效益的错误决策。

## Learning

### React Router v6

1. Route 定义时使用 Nested 模式，Link 中 to 使用相对路径
2. 使用 Outlet 组件作为子组件的 PlaceHolder

```tsx
const App = () => {
  return (
    <div>
      <h1>RouterV6 Example</h1>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<div>Hello</div>} />
          {/* 父组件使用 Outlet 占位 */}
          <Route path="about" element={<Outlet />}>
            {/* 子组件可以用 path="" 以首屏展示 */}
            <Route path="" element={<div>Welcome to about page</div>} />
            <Route path="me" element={<div>It is me, surprise!</div>} />
          </Route>
          <Route
            path="*"
            element={
              <div>
                Page not found
                <p>
                  <Link to="/">Go to Home page</Link>
                </p>
              </div>
            }
          />
        </Route>
      </Routes>
    </div>
  );
};

const Layout = () => {
  return (
    <div>
      <div>
        <ul>
          <li>
            <Link to="/">Home</Link>
          </li>
          <li>
            <Link to="/about">About</Link>
          </li>
          <li>
            <Link to="/about/me">It's me</Link>
          </li>
          <li>
            <Link to="/not-found">Nothing here</Link>
          </li>
        </ul>
      </div>
      <hr />
      {/* 主页面Home的占位 */}
      <Outlet />
    </div>
  );
};
```

### WSL 部署项目

重要：**不能使用 docker-compose 中的 host 网络模式**

一般认为 WSL2 中的 docker 服务，只要通过映射 port 的方式就能在主机 localhost 访问。但在实际操作中，一直是无数据返回，主机和 WSL 相互是能够 ping 通的；在网上搜索了很多，却都没有找到正确的答案。卸载 Docker 重新安装，不手动创建 docker network，而是让 docker 自身去创建 default 的网络，这样的部署形式是可以的。可能和 Windows 代理 WSL 中端口的业务逻辑有关系，不过后续应该不会再用 WSL 部署项目了，暂时就这样吧。

## Life

生病让人在各方面提不起劲，甚至还会影响到对很多事情的看法；锻炼身体健康是保持身心健康的重要一环。由于限制了手机的网络流量，很多时候拿起手机也是一种不知所措的状态；但是有部分情况下，手边没有书籍和电脑，就真的不知道该做什么了。

近期用早上坐车上班的十几分钟闭目养神，顺便回顾前一天的学习内容，重要事项，是一个不错的体验；一方面也是对个人记忆力的考研，一方面也是费曼学习法的实践；后面需要保持这个习惯。

## Thought

- Objective Ignorance (客观无知)
- 相关性不代表因果性 (十分常见的误解成因)
