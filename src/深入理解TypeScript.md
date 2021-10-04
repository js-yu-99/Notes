## tsconfig.json配置

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



## 接口

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

