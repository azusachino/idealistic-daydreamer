---
title: React的简单实践
description: 要是没有css，前端的门槛也不会那么高
date: 2021-05-30
slug: react/info
image: img/2021/05/snow_miku.jpg
categories:
  - react
---

一直对前端都比较感兴趣，虽然有在学，但是既不系统化，也没有产生什么实际的作用。然后，某一天，在 YTB 上看到大佬在一周内学会了 React 并开发出了一个博客项目，感受颇深。

学习需要带着目标进行，目标完成就足够了。而不是像之前的我一样，对很多技术念念不忘，甚至还列出计划要去学习，但总是不了了之了。

所以，我也差不多花了一周左右的时间，重新学了一次 React，并写了一个小项目出来：[iris-react](https://github.com/AzusaChino/iris-react)。只是，初期的设计工作没做好，以及开发过程中，根本不会整样式，所以很简陋。。

## 简单介绍

本项目采用了 React+Redux+TypeScript+AntDesign 进行编写，可以在[这里](http://119.45.30.109/)进行预览。

## 聊一聊

其实吧，我也不知道该说啥。

### State 的初始化问题

果然，在用上 TS 之后，就会出现各种刁钻的问题。本来是想把 state 初始化为空对象，然后在 `componentDidMount` 钩子里进行更新的，但是编译器一直报错。

```ts
interface IState {
  app: {
    name: string;
  },
  list: Array<String>
}

// 定义state的初始值
const initState = {
  app: {
    name: "Home",
  },
  // 数组可以直接new出来
  list: new Array<String>
} as IState;

class Home extends React.Component<{}, IState, {}> {
  // 初始化为空对象会直接报错：空对象没有类型信息
  // state = {};

  state = initState;

  render() {
    // 由于初始化是空对象，获取不到具体的类型信息
    const { app } = this.state;
    // 通过这种语法，可以让编译器不提示错误
    return (<div> {{ app?.name }} </div>);
  }
}
```

### 常用的 HOOKs

React 中比较常用的 HOOK 包括`useState`, `useEffect`, `useSelector`, `useDispatch`。

```typescript
// 定义从全局State中取demo的SSelector
const demoSelect = (state: RootState) => state.demo;

// Functional Component
function Demo() {
  // Declare a new state variable, which we'll call "count"
  // const [count, setCount] = useState(0); //   function useState<S>(initialState: S | (() => S)): [S, Dispatch<SetStateAction<S>>];

  // 定义本地state以及更新函数
  const [fruta, setFruta] = useState("banana");

  // 使用useSelector获取数据
  const demo = useSelector(demoSelect);

  const { count, fruit } = demo;

  // 1. useEffect的第二个参数传递空数组：仅在组件mount的时候，调用 useEffect 函数。
  // 2. useEffect的第二个参数传递需要参数：当前组件会监听参数，根据变化重新生成组件。
  useEffect(() => {
    const fetchDetail = async () => {
      const result = await queryDetail(fruit);
      setFruta(result.data.data);
    };
    fetchDetail();
  }, [fruta]);

  // 使用useDispatch获取dispatch对象, 直接触发action
  const dispatch = useDispatch();

  const setCount = () => {
    dispatch({
      type: constants.COUNT,
      payload: {
        count: count + 1,
      },
    });
  };

  const setFruit = (fruit: string) => {
    dispatch({
      type: constants.FRUIT,
      payload: {
        fruit,
      },
    });
  };

  // Similar to componentDidMount and componentDidUpdate:
  // useEffect(() => {
  //   // Update the document title using the browser API
  //   document.title = `You clicked ${count} times`;
  // });

  return (
    <div>
      <p>You clicked {count} times</p>
      <Button onClick={() => setCount()}>Click me</Button>
      <div>
        <p>今天该吃 {fruit} 了</p>
        <div style={{ width: 360, margin: "auto" }}>
          <Search
            placeholder="吃什么呢"
            enterButton="吃这个"
            onChange={(e) => setFruta(e.target.value)}
            onSearch={() => setFruit(fruta)}
          />
        </div>
      </div>
    </div>
  );
}
```

## 最后

总的来说，还是不太行，没有站在一个高度看待 React，或者说找准角度，很多概念理解起来十分笼统。写这个的时候，脑子里基本上是一团浆糊，找不出什么可以写的，也没有一个大致的思路。**以己之昏昏，岂能使人昭昭。**

不管怎么样，我的 React 学习之路就这样告一段落了，毕竟工作中也用不上。如果有机会用的话，希望下一次能达到一个新的高度。

## 参考链接

- [Kennygunderman 大佬的七天之作](https://github.com/Kennygunderman/web-app)
- [React 文档](https://reactjs.org/docs/getting-started.html)
- [AntD 官网](https://ant.design/)
