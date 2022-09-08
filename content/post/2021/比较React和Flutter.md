---
title: 对比 React，Flutter
description: 世上本没有那么多技术，场景多了，技术也多了
date: 2021-06-05
slug: compare-flutter-and-react
image: img/2021/06/HowgillFells.jpg
categories: [Conclusion]
tags: [Conclusion, React, Flutter]
---

- [iris-flutter](https://github.com/AzusaChino/iris-flutter)，一个用 Flutter 写的记录东西的 APP
- [iris-react](https://github.com/AzusaChino/iris-react)，一个用 React 写的博客

在写这两个的项目的时候，能够明显感受到两者之间的相似程度，毕竟 Flutter 就是参考 React 开发出来的。

作为学习 Flutter 和 React 的镇魂曲，在这里做一点小小的总结。

## 基本目录结构

注意：flutter 中的`lib`对应 react 中的`src`。

![src](img/2021/06/folder-layout.png)

我们可以清晰看到目录结构十分相似，大致上都包括了 API 层，Service 层，View 层，状态管理(provider 和 redux)，main 入口文件。

## 入口文件 main

### flutter

```dart
// Flutter程序主方法
void main() {
  // 待全局配置完成后，异步启动
  Global.init().then((_) => runApp(App()));
}

class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // Flutter的全局状态管理 provider
    return MultiProvider(
      providers: [
        // 全局变量
        ChangeNotifierProvider.value(value: UserModel()),
        ChangeNotifierProvider.value(value: ThemeModel())
      ],
      child: Consumer<ThemeModel>(
        // builder相当于ReactDOM.render
        builder: (context, themeModel, child) {
          // Flutter的App组件，相当于React中的App组件
          return MaterialApp(
            title: 'IrisFlutter',
            theme: ThemeData(primarySwatch: themeModel.theme),
            home: HomePage(), // 首页
            // Flutter的路由配置，相当于React中的Router
            routes: {
              "/login": (context) => LoginPage(),
              "/themes": (context) => ThemeChangePage(),
              "/register": (context) => RegisterPage()
            },
          );
        },
      ),
    );
  }
}
```

### react

```tsx
// React入口文件 index.tsx
ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById("root")
);

// App.tsx
class App extends React.Component {
  render() {
    return (
      // React中的全局状态管理 store
      <Provider store={store}>
        <div className="App">
          <BrowserRouter>
            <Layout className="layout">
              <HeaderItem />
              <Content style={{ padding: "0 50px" }}>
                <div className="layout-content">
                  <Switch>
                    {routes.map((route: RouteProps) => (
                      <Route {...route} />
                    ))}
                  </Switch>
                </div>
              </Content>
              <Footer>
                Iris React ©2021 Built by
                <a
                  href="https://azusachino.cn"
                  target="_blank"
                  rel="noreferrer"
                >
                  &nbsp;az
                </a>
              </Footer>
            </Layout>
          </BrowserRouter>
        </div>
      </Provider>
    );
  }
}

// React的路由配置 router.ts
export const routes: Array<RouteProps> = [
  {
    path: "/",
    component: Home,
    exact: true,
  },
  {
    path: "/about",
    component: About,
  },
  {
    path: "/demo",
    component: Demo,
  },
  {
    path: "/article/:id",
    component: ArticleDetail,
  },
  {
    path: "*",
    component: NotFound,
  },
];
```

## 状态管理

全局的状态管理对于程序来说是一个很重要的功能，比如用户的登录信息，全局的配置信息。

### flutter.ChangeNotifier

在 flutter 中，我们在定义 state 的时候，继承`ChangeNotifier`就能获取监听当前 state 变化的能力，并通过`notifyListeners`通知所有的子组件。

```dart
// 用户信息State
class ProfileChangeNotifier extends ChangeNotifier {
  Profile get profile => Global.profile;

  @override
  void notifyListeners() {
    // 当用户信息发生变化时，重新保存
    Global.saveProfile();
    // 通知子组件刷新
    super.notifyListeners();
  }
}
```

### react.Redux

在 react 中，状态管理是一串比较复杂的链路，简单介绍如下：

1. 首先需要定义 State
2. 其次定义 Action，即有哪些方式来改变 State
3. 结合 State 和 Action 定义 Reducer
4. 通过`configureStore(reducer)`创建真正的状态管理对象
5. 通过 dispatcher 触发 action 以更新 State => 组件刷新

整体还蛮复杂的，我就不细说了，毕竟我自己都不太懂，感兴趣参考官网[State, Actions, and Reducers](https://redux.js.org/tutorials/fundamentals/part-3-state-actions-reducers)

## 简单总结

总的来说，开发一个简单的程序并不复杂，项目整体的结构也很相似，我觉得难点主要在于掌握那些更具有语言特色的内容。而且，并不能说入门了就满足了，任重而道远。

不过，我也只是基于兴趣才学习的，后续暂时没有继续学下去的计划了。就这样~
