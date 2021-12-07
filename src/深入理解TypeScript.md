# tsconfig.json配置

```js
{
	"coompilerOptions" {
  	"target": "es5", // 指定ECMAScript目标版本 ES3(default),ES5,ES2015,ES2016,ES2017,ESNEXT
    "module": "commonjs", // 指定模块: commonjs, amd, system, umd, es2015
    "lib": [], // 指定要包含在编译中的库文件
    "allowJs": true, // 允许编译JavaScript文件
    "checkJs": true, // 报告JavaScript文件中的错误
    "jsx": "preserve", // 指定jsx代码生成：preserve, react-native, react
    "declaration": true, // 生成相应的 .d.ts 文件
    "sourceMap": true, // 生成响应的 .map 文件
    "outFile": "./lib/init.js", // 将输出文件合并为一个文件
    "outDir": "./dist/assets/", // 指定输出目录
    "rootDir": "./", // 用来控制输出目录结构 --outDir
    "removeComments": false, // 删除编译后的所有的注释
    "noEmit": true, // 不生成输出文件
    "importHelpers": true, // 从tslib导入辅助工具函数
    "downlevelIteration": true, // 当以'ES5'或'ES3'为目标时，在'for-of'、spread和destructuring中提供对可迭代对象的完全支持
    "isolatedModules": true, // 将每个文件作为单独的模块（与ts.transpileModule 类似）
      
    // 严格的类型检查选项
    "strict": true, // 启用所有严格类型检查选项
    "noImplicitAny": false, // 在表达式和声明上有隐含的any类型时报错
    "strictNullChecks": true, // 启用严格的null检查
    "strictFunctionTypes": true,  // 开启对功能类型的严格检查
    "noImplicitThis": true,  // 当this表达式值为any类型时，生成一个错误
    "alwaysStrict": true, // 以严格模式检查每个模块，并在每个文件里加入‘user strict’

    // 额外的检查
    "noUnusedLocals": true, // 有未使用的变量时，抛出错误
    "noUnusedParameters": true, // 有未使用的参数时，抛出错误
    "noImplicitReturns": true, // 并不是所有函数里的代码都有返回值时，抛出错误
    "noFallthroughCasesInSwitch": true, // 报告switch语句的fallthrough错误（即不允许switch的case语句贯穿）

    // 模块解析选项
    "moduleResolution": "node", // 选择模块解析策略： node(node.js) or classic(TypeScript pre-1.6)
    "baseUrl": "./", // 用于解析非相对模块名称的基目录
    "paths": { // 模块名到基于baseUrl的路径映射的列表
      "*": [
        "*",
        "src/*",
        "chore/tests/*"
      ],
      // 要求 tsconfig 中 paths 的每个 key 只能配置一个
      // 【此处会自动被 webpack 引用：由于 webpack 的限制，单个 alias 只能代表一个路径】
      "@apps/*": ["src/apps/*"],
      "@shared/*": ["src/shared/*"],
      "@libs/*": ["src/libs/*"],
      "@assets/*": ["static/assets/*"]
    },
    "rootDirs": [ // 根文件夹列表，其组合内容表示项目运行时的结构内容
      "src",
      "chore/tests"
    ],
    "typeRoots": [], // 包含类型声明的文件列表
    "types": [ // 需要包含的类型声明文件名列表
      // "core-js",
      "node"
    ],
    "allowSyntheticDefaultImports": true, // 允许从没有设置默认导出的模块中默认导入
    "preserveSymlinks": true, // 不解决符号链接的实际路径

    /* Source Map Options */
    "sourceRoot": "./", // 指定调试器应该找到TypeScript文件而不是源文件的位置
    "mapRoot": "./",  // 指定调试器应该找到映射文件而不是生成文件的位置
    "inlineSourceMap": true, // 生成单个sourcemaps文件，而不是将sourcemaps生成不用的文件
    "inlineSources": true, // 将代码与sourcemaps生成到一个文件中，要求同时设置了 --inlineSourceMap 或 --sourceMap属性

    // 其他选项
    "experimentalDecorators": true, // 启用装饰器
    "emitDecoratorMetadata": true // 为装饰器提供元数据的支持
  },
  "include": [
    "./src/",
    "./declaration.d.ts"
  ],
  "exclude": [
    "node_modules",
    "**/*.spec.ts"
  ]
	}
}
```



# 基础类型

```typescript
 // 布尔值
let isDnoe: boolean = false;

// 数字 ts中的数字类型支持十进制、十六进制、二进制和八进制
let num1: number = 10;
let num2: number = 0o744

// 字符串 可以使用模板字符串
let name: string = 'wangyu';
let sentence: string = `Hello, my name is ${ name }`;

// 数组
let list: Array<number> = [1, 2, 3];
let list1: number[] = [1, 2, 3];

// 元祖 当访问一个越界的元素，会使用联合类型替代：
let tupleList: [number, string] = [10, '1'];
tupleList[3] = 'world'; // OK, 字符串可以赋值给(string | number)类型

// 枚举 enum类型是对JavaScript标准数据类型的一个补充。
// 默认情况下，从0开始为元素编号。 你也可以手动的指定成员的数值。
enum Gender {
    Girl,
    Boy
}
console.log(Gender.Girl); // 0
console.log(Gender[0]); // Girl
// 枚举解析成js后的代码
var Gender;
(function (Gender) {
    Gender[Gender["Girl"] = 0] = "Girl";
    Gender[Gender["Boy"] = 1] = "Boy";
})(Gender || (Gender = {}));

// 常量枚举 编译后直接把值打印出来，不对常量枚举进行编译，节省内存开销
const enum Colors {
    red, yellow, blue
}
const colorArr = [Colors.red];
console.log(Colors.red);

// 任意类型 any  比如第三方库使用时不确定类型、对现有代码进行改写是 都适合用any 不进行类型检查
let list: any[] = [1, true, "free"];
let element: (HTMLElement | null) = document.getElementById('root'); // HTMLElement 任意dom元素类型

// 非空断言
element!.style.color = 'red';

// null undefined
// 默认情况下null和undefined是所有类型的子类型。 就是说你可以把 null和undefined赋值给number类型的变量。然而，当你指定了--"strictNullChecks": true  标记，null和undefined只能赋值给void和它们各自。 这能避免 很多常见的问题。
let x: number;
x = null;
x = undefined;
```



```typescript
// never 代表不会出现的值
// never类型表示的是那些永不存在的值的类型。 例如， never类型是那些总是会抛出异常或根本就不会有返回值的函数表达式或箭头函数表达式的返回值类型； 变量也可能是 never类型，当它们被永不为真的类型保护所约束时。
// never类型是任何类型的子类型，也可以赋值给任何类型；然而，没有类型是never的子类型或可以赋值给never类型（除了never本身之外）。 即使 any也不可以赋值给never。

// 返回never的函数必须存在无法达到的终点
function error(message: string): never {
    throw new Error(message);
}

// 推断的返回值类型为never
function fail() {
    return error("Something failed");
}

// 返回never的函数必须存在无法达到的终点
function infiniteLoop(): never {
    while (true) {
    }
}

// Void 表示没有任何类型 void 可以兼容undefined 和 null（strictNullChecks = false）
function foo(): void {
    console.log(1);
}

// never 和 void 的区别
// void 可以被赋值为null和undefined never 不能包含任何类型
// 函数返回void函数能正常执行，返回never无法正常执行
```



```typescript
// Symbol
const sym1 = Symbol('key');

// Bigint bigint 和 number 不通用
const num = BigInt(Number.MAX_SAFE_INTEGER);

// Object 表示非原始类型，也就是除number，string，boolean，symbol，null或undefined之外的类型。
declare function create(o: object | null): void;

create({ prop: 0 }); // OK
create(null); // OK

create(42); // Error
```



# 复杂类型

```typescript
// 包装对象类型 ts会想js一样在声明的原始类型中使用方法和属性，并不会进行报错
let name: string = 'wangyu';
console.log(name.length);

// 但是不能把对象类型直接赋值给原始类型定义的变量
let bol: boolean = !1;
let bol1: boolean = Boolean(1);
let bol2: boolean = new Boolean(1); // “boolean”是基元，但“Boolean”是包装器对象。如可能首选使用“boolean”。

// 联合类型 明确类型是什么之后才能使用对应类型的方法
let name1: string | number;
console.log(name1.length); // 类型“string | number”上不存在属性“length”。类型“number”上不存在属性“length”
name1 = 'wang';
console.log(name1.length);
name1 = 10;
console.log(name1.toFixed(2));

// 类型断言
let name2: string | number;
console.log((name2! as number).toFixed(2));
console.log(name2! as any as boolean); // 双重断言

// 字面量类型
const up: 'UP' = 'UP';
const down: 'Down' = "Down";
const num1: 1 = 1;
type Dorection = 'Up' | 'Down';
function move(direction: Dorection) {

}
move('Down');

//类型字面量
type Person = {
    name: string;
    age: number;
}
let p1: Person = {name: 'wangyu', age: 20};
```



# 接口

接口是TypeScript的一个核心知识，他它能合并众多类型声明至一个类型声明

```typescript
interface IInfo {
	name: string;
  age: number;
}
let info: IInfo;
info = {
  name: 'wangyu',
  age: 20
}
info = {
  // Error: 'name is missing in type ...'
  age: 20
}
info = {
  name: 'wangyu',
  age: '20' // Type 'string' is not assignable to type 'number'. | The expected type comes from property 'age' which is declared here on type 'IInfo'
}
```

