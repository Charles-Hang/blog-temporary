# Typescript在React中的实践与个人感想
## 引言
虽然去年就接触并使用上了ts，但一直感觉自己只是在低水平的使用，对ts并没有特别上心。随着ts的呼声越来越高，眼看ts就要成为前端的标配。感觉是时候该重新审视一下ts了。这几个月，我放了很大的重心在ts的使用上，我感觉打开了一个新世界，可能ts不是我一直想象的那样。ts可能不仅仅是类型的声明与限制，它的确能让我们在编程时不易错过一些常见的会因类型考虑不周而引起的错误，但经过我的实践，它吸引我的地方确是写代码时编辑器实时跳出的提示和良好的可维护性。刚写下a，它就立马告诉我a里应该有什么，可以不费力去记忆，直接顺着写的感觉非常爽。还有通过一定的设计，在遇到需求变动时，一点代码的改动，编辑器就能将需要跟着变动的地方提示出来，提高了迭代效率，即快速又能减少改动遗漏。这是我这段时间刻意实践ts给我的感受，它很好很强大，我想我是无法再退回不使用ts的日子了。我有必要在这里尽自己所能分享关于ts的使用经验，希望大家都能喜欢并开始认真对待ts，它不会让你失望。
## 开发原则
1. 通过对变量、函数参数、函数等的类型声明避免因类型处理不完善导致的易忽视的类型错误。
2. 通过类型声明清晰的告诉开发者一个组件一个函数需要什么参数，输出什么参数，利于团队协作与迭代开发。能避免多次迭代之后，产生冗余代码，高耦合不明所以的代码。
3. 通过完善的类型声明将有联系的代码连接起来，方便后续的迭代维护。

前两点是最基本的开发原则，定义好了类型既方便协作，也能借助编辑器提示方便开发。这就要求我们定义好组件的props、state类型，关键函数的输入输出类型等。在react开发中定义高阶组件的props类型是一个难点。

第三点则相对复杂，在我看来这里才是ts的精髓。从业务开发角度看，良好的类型声明能为开发者在迭代维护时带来巨大的便利，代码一经修改，其他有联系的需要改动的代码部分则会报错，一下就能定位改动点，减少许多筛查的时间。什么意思呢？看下面一个例子：
```tsx
// A组件定义并使用了一个对象，将一部分传给了B组件

// A.tsx

// ...
export interface IObj {
    h: {
        h1: string;
        h2: number;
    };
    j: {
        j1: J1Type;
        j2: J2Type;
    };
}

function A(props: IAProps) {
    // ...
    const obj: IObj = data;
    // ...
    const parseH = (h: IObj['h']) => {
        // ...
    }

    return (
        <div>
            // ...
            <B data={obj.j}>
        </div>
    )
}
// ...

// B.tsx

type DataType = import('A').IObj['j'];

interface IBProps {
    data: DataType;
}

function B(props: IBProps) {
    // ...
}
// ...
```
上面A组件中的parseH方法的参数和B组件的data通过索引类型与`IObj`连接了起来，而不是如下的各自定义，当obj类型因迭代而有所改变时，parseH方法和B组件内自然会报错，从而起到提示的作用。上面用的是索引类型，更多的时候则是将obj内的h和j单独提出来定义，或者当其他更复杂的情况出现时用映射类型将有关联的类型联系起来。
```tsx
// 错误的方式：各自定义
// ...
const parseH = (h: { h1: string; h2: number }) => {
    // ...
}
// ...
interface IBProps {
    data: {
        j1: J1Type;
        j2: J2Type;
    };
}
```
上面是非常简单的例子，当一个业务很复杂，有好多模块，各个模块之间类型的联系更多的时候都是通过映射的方式实现，即从一个类型中创建出新的与之关联的类型。就这样旧的类型与由其映射产生的许多的新的类型一起将代码保护了起来。一个业务中定义好了几个原始的类型（或者说是这个业务的起点类型），其他的相关类型均通过原始类型映射产生，业务变动时，修改对应的原始类型就会引起其他映射类型相应的变动，使迭代维护变得相当轻松。但这就比较考验原始类型的选取以及映射的功力了。最理想的代码就是将所有有联系的地方都通过类型映射联系起来。
## 场景与用例
### 泛型组件
当一个组件的props得根据调用时的具体情况来决定类型时，可以使用泛型组件，如下：
```ts
// 一个泛型组件
type ListProps<T> = { list: T[] };

class List<T> extends React.Component<ListProps<T>> {
    ...
}

// 使用
const SomeList = () => <List<string> list={['a', 'b']} />;
```
上面泛型组件的函数式写法：
```ts
function List<T>(props: { list: T[] }) {
    ...
}
// 或者
const List = <T extends {}>(props: { list: T[] }) => {
    ...
} // 这里要使用extends泛型约束来告诉编译器这里是个泛型，否则会报错T标签没有关闭
```
### 关于React.FC接口
`React`里有个`FunctionComponent`(`FC`)接口可以用来定义函数式组件，如下：
```ts
interface Props {
    foo: string;
}

const MyComponent: React.FC<Props> = props => {
  return <span>{props.foo}</span>;
}
```
但是`React.FC`有一些问题，它会导致组件的静态属性如defaultProps和displayName的类型解析失效，如下：
```ts
...
interface Props {
    name: string;
    address: string;
}

const Client: React.FC<Props> = (props) => {
    return (
        <div>
            <span>name: {props.name}</span>
            <span>address: {props.address}</span>
        </div>
    )
}

Client.defaultProps = {
    address: 'some place'
}

Client.displayName = 'Client';

const ClientList = () => {
    return (
        <div>
            {/* 报错缺少address属性 */}
            <Client name="a" />
            ...
        </div>
    )
}

ClientList.displayName = Client.displayName + ' list'; // 这里Client的displayName类型被推导成了string | undefined，应该为string才对
...
```
而且`React.FC`不能用来定义泛型组件，如上面的List泛型组件：
```ts
interface Props<T> {
    list: T[];
}
// 报错找不到T
const List: React.FC<Props<T>> = (props) => {
    ...
}
```
综上，还是不建议使用`React.FC`定义函数式组件，用最普通的方式就好：`const List = (props: Props) => {}`
### 高阶组件
#### 增强型
增强型高阶组件能赋予组件新的能力，比如：
```ts
interface IEnhanceLoadingProps {
    loading: boolean;
}

const enhanceLoading = <P extends object>(Component: React.ComponentType<P>) => {
    return (props : P & IEnhanceLoadingProps) => {
        const { loading, ...componentProps } = props;
        return loading ? <Loading /> : <Component {...componentProps as P} />
    }
}
```
这个高阶组件赋予传入的组件展示loading状态的能力，新的组件增加了一个loading的props，这里使用泛型拿到了传入组件的props类型P，使用交叉类型将`IEnhanceLoadingProps`合进了传入组件的props类型里
#### 注入型
注入型高阶组件是能向传入的组件注入所需的属性的组件，比如：
```ts
interface InjectedProps {
    user: UserProps;
}

const withUser = <P extends InjectedProps>(Component: React.ComponentType<P>) => {
    return (props: Omit<P, keyof InjectedProps>) => {
        const [user, setUser] = useState();
        const fetchUser = async () => {
            try {
                const { data } = await http.get('/user');
                setUser(data);
            } catch (err) {
                //
            }
        }
        useEffect(() => {
            fetchUser();
        }, []);
        return <Component {...props as P} user={user} />
    }
}
```
这个高阶组件获取user信息并注入到传入的组件中，这里用泛型约束确保传入的组件确实是需要user的`<P extends InjectedProps>`，再通过条件类型Omit剔除了返回组件的props类型的user属性，防止再从外部传入user`Omit<P, keyof InjectedProps>`
***
高阶组件十分灵活多样，通过上面两个例子大致能明白高阶组件的类型定义其实主要就是对泛型的一系列操作，需要对高级类型（包括交叉类型、联合类型、索引类型、映射类型和条件类型等）有一定的掌握。
### import类型
当只访问一个模块里导出的类型而并不用该模块时，可以使用`import('mod')`：
```ts
// a 模块
export interface DataProps {
    id: number;
    name: string;
}
...

// b 模块
...
// import { DataProps } from './a' // 不好的方式，引入了a模块

type DataProps = import('./a').DataProps; // 好的方式，不会引入a模块
...
```
### 动态setState
有时由于代码的复用，经常会出现动态设置组件state值的情况，定义state接口时经常会使用索引签名，这样可以随意引入额外的state而不报错，可能对以后的维护造成困扰。应避免索引签名的使用，在动态设置的地方使用断言并确认好state即可：
```ts
// 不好的
interface IComponentState {
    a: string;
    b: number;
    [props: string]: any;
}
...
this.setState({
    // a 或 b
    [key]: value
})

// 好的
interface IComponentState {
    a: string;
    b: number;
}
...
this.setState({
    // a 或 b
    [key]: value
} as IComponentState)
```
类型定义时尽可能精确，局部灵活时可用断言
### location携带state的路由跳转
有时路由跳转需要向新页面传递一些数据，随着业务的迭代，如果不对这些数据的类型做好定义，后面会出现难以维护的情况，不同的页面带着不同的数据向同一页跳转，很容易让人头晕。所以最好还是对location的state做好类型定义，跳转时做好必要的处理，方便维护：
```ts
// 目标页 a
import { RouteComponentProps } from 'react-router-dom';

// location state
export interface AState {
    origin: 'b' | 'c';
    id: number;
    name: string;
}

// RouteComopnentProps第三个泛型参数定义location state的类型
interface IAProps extends RouteComponentProps<{}, {}, AState> {
    ...
}

const A = (props: IAProps) => {
    // 类型为AState
    const locationState = props.location.state;
    ...
}

// b 跳转到目标页 a
type History<S> = import('history').History<S>;
type AState = import('./a').AState;

const B = (props: RouteComponentProps) => {
    const onClick = () => {
        // 用断言定义history的类型保证state传参正确
        (props.history as History<AState>).push({
            pathname: '/a',
            state: { // 报错origin number不能赋值给类型'b' | 'c'
                origin: 1,
                id: 1,
                name: 'name'
            }
        });
    }
    ...
}
```
不过当使用`react-router-dom`的`Link`跳转时没办法确保state类型的正确，好在目标页中有明确地定义，不至于混乱
### 一些补充
```ts
type StoreType = ReturnType<typeof createStore>;

function createStore() {
    return {
        statusA: false,
        statusB: 'status' as BStatus,
        fn() {
            return Math.random();
        }
    }
}

// 内置的条件类型
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
```
上面先用`typeof`推断出`createStore`的类型，再用内置的`ReturnType`获取了`createStore`类型的返回值类型。ts类型推断时，上面的`false`会推断成`boolean`，`status`会推断成`string`，即它们的基础类型。如果想推断成其他结果则使用断言即可，像上面的`'status' as BStatus`，`statusB`就推断为`BStatus`而不是`string`。再来看一看`ReturnType`，用到了条件类型和infer关键字，条件类型类似三元运算，在两种类型中选择其一；infer则代表extends条件语句中待推断的类型，只能用在条件语句中。`ReturnType`中`infer R`将函数类型的返回值类型推断出来。

说到条件类型还有个有意思的点：如`T extends U ? X : Y`，如果T的类型是`A | B | C`，则这个条件类型会被解析为`(A extends U ? X : Y) | (B extends U ? X : Y) | (C extends U ? X : Y)`，这是一个非常常用的特性。
## 结束语
要提高ts水平，还得时常翻看官网文档，GitHub，了解都有哪些特性，可能有的什么用法。当你的工具箱里有锤子时，你才可能看见钉子。
