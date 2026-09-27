# Python UAST 迁移与构建问题总结

## 项目目标

本次工作将 Python 的 UAST 生成方式从旧版外部 `uast4py` 二进制迁移到基于 Tree-sitter 的实现。

旧流程：

```text
Python 源码 → python-ast-builder.ts → 外部 uast4py → UAST JSON
```

新流程：

```text
Python 源码 → web-tree-sitter → tree-sitter-python.wasm → Python UAST
```

新的 Python parser 在 engine 内部直接运行，不再依赖 `uast4py`。

先尝试用官方仓库构建engine

## Build.sh的使用

一开始用官方仓库，直接运行build.sh

### 旧版依赖安装与 `uast4py` 问题

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927234103501.png" alt="image-20260927234103501" style="zoom:50%;" />

这是因为旧版 `YASA-Engine` 的 Python parser 依赖外部二进制：

```text
deps/uast4py/uast4py
```

`npm install` 不会安装它，因为它不是 npm 依赖。旧项目的 `install_deps.sh` 会根据平台下载：

```text
uast4go
uast4py
```

运行

```bash
bash install_deps.sh
```

在迁移后的 Python parser 中不需要再下载 `uast4py`。如果日志出现 `uast4go binary not found`，那是 Go parser 的旧外部依赖，与 Python UAST 无关。

### 测试出错

但是运行完install_deps.sh后还是出错

![image-20260927234217310](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927234217310.png)

这是因为build.sh里会运行

![image-20260927233834213](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927233834213.png)

`npm run test-all` 读取 `engine/package.json` 中的脚本，并按 `&&` 顺序执行：

```text
test-callargs
test-fastapi-backgroundtasks
test-match-field
test-entrypoint-deadline-budget
test-entrypoint-deadline-scheduler
test-clone-util
test-sink-util-new-expr
test-taint-output-strategy
test-go-dynamic-key-taint
test-js
test-java
test-sarif-codeflow-quality
test-go
test-python
test-callchain
test-php
```

其中任意一个测试失败，后面的测试不会继续执行。而原仓库缺少java的benchmark

Java SARIF 测试需要先准备 benchmark：

```bash
npx tsx test/java/prepare-java-benchmark.ts
```

这样就可以通过测试了

### 缺少rosetta

<img src="https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927234400562.png" alt="image-20260927234400562" style="zoom: 33%;" />

还是打包失败

这是因为在 Apple Silicon 上打包多个目标时，项目配置通常包括：

```text
node18-macos-arm64 node18-macos-x64 node18-linux-x64
```

这里报错spawn Unknown system error -86，通常表示打包过程需要启动 x86_64 进程，但机器没有 Rosetta 2。可以安装：

```bash
softwareupdate --install-rosetta --agree-to-license
```

然后就可以运行打包成功了

![image-20260927234620619](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927234620619.png)

PHP WASM 如果直接使用裸的 `npx pkg .`，可能被 pkg 按文本处理，运行时出现：

```text
WebAssembly.instantiate(): unknown section code #0x3d
```

应使用项目的 `build.sh`，因为它会先执行 PHP WASM patch。

迁移 engine 的构建流程大致为：

```mermaid
flowchart LR
    A[安装依赖] --> B[准备 native addon]
    B --> C[Patch PHP WASM 资源]
    C --> D[TypeScript 类型检查]
    D --> E[运行全部测试]
    E --> F[tsc 编译到 dist]
    F --> G[pkg 打包]
    G --> H[清理 dist]
```

`npx tsc --noEmit` 只做类型检查，不生成文件；`npx tsc` 才会把 TypeScript 编译到 `dist`。

### 测试php

![image-20260927234835040](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927234835040.png)

测试php的dumpast，报错，但是java就可以

![image-20260927234910751](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927234910751.png)

可以从日志里找问题

![image-20260927235002855](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927235002855.png)

可以看到是没有进行初始化，那为什么其他语言可以？

在各个语言的ast-build.ts文件里

![image-20260927235130379](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927235130379.png)

![image-20260927235208313](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260927235208313.png)

可以看到php会多一步初始化，这是因为php的uast是利用tree-sitter实现的，得先调用里面的wasm文件

但是在interface接口这边的start.ts中可以看到

![image-20260928000157570](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928000157570.png)

都是直接从parseSingleFile开始的，没有初始化

最终采用的修复方式是在统一 parser 入口parser.ts增加：

![image-20260928000022237](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928000022237.png)

```ts
async function ensureInitialized(language?: string): Promise<void> {
  if (language === 'python') {
    const PythonParser = require('./python/python-ast-builder')
    await PythonParser.ensureInitialized()
  } else if (language === 'php') {
    const PhpParser = require('./php/php-ast-builder')
    await PhpParser.ensureInitialized()
  }
}
```

并在 `starter.ts` 确定语言后、进入 `dumpAST` 或 Analyzer 之前调用：

```ts
await Parser.ensureInitialized(Config.language)
```

然后就可以使用了

![image-20260928000557106](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928000557106.png)

## 迁移python

解决这个问题之后，我们开始迁移python

###  仿写parser.ts

![image-20260928001148015](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928001148015.png)

![image-20260928000917266](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928000917266.png)

模仿php可以写一个parser.ts，这里有一个和php不一样的地方，这里我们读取wasm是用拼接路径的方式，这是因为

php这种写法会导致pkg把wasm按utf-8来识别

原本仓库是在build里专门处理了这个问题，但如果按我们这样写就不需要了

![image-20260928001053622](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928001053622.png)

然后我们把php中后续把cst转化成uast的部分，拆分到了vistor.ts当中

### Enigine中修改ast-builder.ts

![image-20260928001433086](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928001433086.png)

这一块也直接放照php就行，原本这里是调用uastpy的

![image-20260928001543196](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928001543196.png)

### 测试可用性

可以用这些命令来测试

```bash
cd ~/Downloads/yasa-uast-migration/engine
./build.sh

./yasa-engine-macos-arm64 \
  /Users/infinite/Downloads/YASA-Engine/test/python/fastapi-backgroundtasks-cases/scanner_callback.py \
  --language python \
  --dumpAST
```

### 性能

原始的调用uast4py太慢了，需要2s

![image-20260928001709424](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928001709424.png)

修改完之后非常迅速

![image-20260928001818053](https://raw.githubusercontent.com/infinitepwn/note_picbed/main/image-20260928001818053.png)
