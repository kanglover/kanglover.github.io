---
layout: post
title: Tree Shaking
date: 2023-05-10
author: shaokang
header-img:
catalog: true
tags:
    - 工程化
---

## 什么是 Tree Shaking

Tree-Shaking 是一种基于 ES Module 规范的 Dead Code Elimination 技术，它会在运行过程中静态分析模块之间的导入导出，确定 ESM 模块中哪些导出值未曾其它模块使用，并将其删除，以此实现打包产物的优化。
Tree Shaking 较早前由 Rich Harris 在 Rollup 中率先实现，Webpack 自 2.0 版本开始接入，至今已经成为一种应用广泛的性能优化手段。

由此我们就知道了，tree-shaking 是一种消除无用代码的方式！

但要注意的是，tree-shaking 虽然能够消除无用代码，但仅针对 ES6 模块语法，因为 ES6 模块采用的是静态分析，从字面量对代码进行分析。

## Rollup 中的 Tree Shaking

前置知识：

> rollup 中的 tree-shaking 使用 acorn 实现 AST 抽象语法树的遍历解析，acorn 和 babel 功能相同，但 acorn 更加轻量，在此之前 AST 工作流也是必须要了解的；  
> rollup 使用 magic-string 工具操作字符串和生成 source-map。

tree-shaking 的核心原理详细地描述一下具体流程：

-   rollup()阶段，解析源码，生成 AST tree，对 AST tree 上的每个节点进行遍历，判断出是否 include(标记避免重复打包)，是的话标记，然后生成 chunks，最后导出。
-   generate()/write()阶段，根据 rollup()阶段做的标记，进行代码收集，最后生成真正用到的代码。

### 1. 模块解析

resolveId()方法解析文件地址，拿到文件绝对路径;

```js
export async function resolveId(
    source: string,
    importer: string | undefined,
    preserveSymlinks: boolean
) {
    // 不是以 . 或 / 开头的非入口模块在此步骤被跳过
    if (importer !== undefined && !isAbsolute(source) && source[0] !== '.')
        return null;
    // 调用 path.resolve，将合法文件路径转为绝对路径
    return addJsExtensionIfNecessary(
        importer ? resolve(dirname(importer), source) : resolve(source),
        preserveSymlinks
    );
}
```

rollup() 阶段做了很多工作，包括收集配置并标准化、分析文件并编译源码生成 AST、生成模块并解析依赖，最后生成 chunks。为了搞清楚 tree-shaking 作用的具体位置，我们需要解析更内层处理的代码。

首先，通过从入口文件的绝对路径出发找到它的模块定义，并获取这个入口模块所有的依赖语句并返回所有内容。

```js
private async fetchModule({
        id,
        meta,
        moduleSideEffects,
        syntheticNamedExports
    }: ResolvedId,
    importer: string | undefined, // 导入此模块的引用模块
    isEntry: boolean // 是否入口路径
): Promise < Module > {
    ...
    // 创建 Module 实例
    const module: Module = new Module(
        this.graph, // Graph 是全局唯一的图，包含入口以及各种依赖的相互关系，操作方法，缓存等
        id,
        this.options,
        isEntry,
        moduleSideEffects, // 模块副作用
        syntheticNamedExports,
        meta
    );
    this.modulesById.set(id, module);
    this.graph.watchFiles[id] = true;
    await this.addModuleSource(id, importer, module);
    await this.pluginDriver.hookParallel('moduleParsed', [module.info]);
    await Promise.all([
        // 处理静态依赖
        this.fetchStaticDependencies(module),
        // 处理动态依赖
        this.fetchDynamicDependencies(module)
    ]);
    module.linkImports();
    // 返回当前模块
    return module;
}
```

每个文件都是一个模块，每个模块都会有一个 Module 实例。在 Module 实例中，模块文件的代码通过 acorn 的 parse 方法遍历解析为 AST 语法树。

```js
const ast = this.acornParser.parse(code, {
 ...(this.options.acorn as acorn.Options),
 ...options
});
```

### 2. 标记模块是否可 Tree-shaking

处理当前 module，根据 isExecuted 的状态及 treeshakingy 相关配置进行模块以及 es tree node 的引入，isExecuted 为 true 意味着这个模块已被添加入结果，以后不需要重复添加，最后也是根据 isExecuted 收集所有需要的模块从而实现 tree-shaking。

```js
private includeStatements() {
    ......
    if (this.options.treeshake) {
        let treeshakingPass = 1;
        do {
            timeStart(`treeshaking pass ${treeshakingPass}`, 3);
            this.needsTreeshakingPass = false;
            for (const module of this.modules) {
                // 根据 isExecuted 进行标记
                if (module.isExecuted) {
                    if (module.info.hasModuleSideEffects === 'no-treeshake') {
                        module.includeAllInBundle();
                    } else {
                        module.include(); // 标记
                    }
                }
            }
            timeEnd(`treeshaking pass ${treeshakingPass++}`, 3);
        } while (this.needsTreeshakingPass);
    }
}
```

### 3. 去除无用代码

treeshakeNode() 方法去除无用代码

```js
export function treeshakeNode(
    node: Node,
    code: MagicString,
    start: number,
    end: number
) {
    code.remove(start, end);
    if (node.annotations) {
        for (const annotation of node.annotations) {
            if (!annotation.comment) {
                continue;
            }
            if (annotation.comment.start < start) {
                code.remove(annotation.comment.start, annotation.comment.end);
            } else {
                return;
            }
        }
    }
}
```

tree-shaking

```js
if (!node.included) {
    treeshakeNode(node, code, start, end);
    continue;
}
...
if (currentNode.included) {
    currentNodeNeedsBoundaries
        ?
        currentNode.render(code, options, {
            end: nextNodeStart,
            start: currentNodeStart
        }) :
        currentNode.render(code, options);
} else {
    treeshakeNode(currentNode, code, currentNodeStart!, nextNodeStart);
}
...
```

## Webpack 中的 Tree Shaking

沿用[工程化](2022-02-23-前端复习之路-工程化篇.md)中的记录

1. 收集模块导出

    - 将模块的所有 ESM 导出语句转换为 Dependency 对象，并记录到 module 对象的 dependencies 集合
    - 所有模块都编译完毕后，触发 compilation.hooks.finishModules 钩子，开始执行 FlagDependencyExportsPlugin 插件回调
    - FlagDependencyExportsPlugin 插件从 entry 开始读取 ModuleGraph 中存储的模块信息，遍历所有 module 对象
    - 遍历 module 对象的 dependencies 数组，找到所有 HarmonyExportXXXDependency 类型的依赖对象，将其转换为 ExportInfo 对象并记录到 ModuleGraph 体系中

2. 标记模块导出

    遍历 module 对象对应的 exportInfo 数组，确定其对应的 Dependency 对象是否被其他模块使用

3. 生成代码

    - import 被标记为 /_ harmony import _/
    - 被使用过的 export 标记为 /_ harmony export([type]) _/，其中[type]和 webpack 内部有关，可能是 binding，immutable 等。
    - 没有被使用的 export 标记为/_ unused harmony export [FuncName] _/，其中[FuncName]为 export 的方法名。

4. 删除 Dead Code

    Terser、UglifyJS 等 DCE 工具“摇”掉标记中无效的代码。

## 总结

Tree Shaking 的方式原理大致就是，在代码分析过程中，标记无用的代码。最后代码生成过程中，去掉这些无用代码。Webpack 则是通过压缩工具来去掉的无效代码。

Tree Shaking 利用 ES 模块来静态分析代码引用关系。对于无效的赋值（副作用）是采取保守的方式，并不会删除。我们可以利用 `/*#__PURE__*/` 注释的方式，或者 webpack 设置 `sideEffects`，标记哪些文件存在副作用，这样 webpack 就可以对除了指定的文件之外的其他文件进行安全 tree-shaking。
