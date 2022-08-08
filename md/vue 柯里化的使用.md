``` typescript
// src\platforms\web\runtime-with-compiler.ts
// 通过 template 获取对应的 render 和 staticRenderFns
const { render, staticRenderFns } = compileToFunctions(
        template,
        {
          outputSourceRange: __DEV__,
          shouldDecodeNewlines,
          shouldDecodeNewlinesForHref,
          delimiters: options.delimiters,
          comments: options.comments
        },
        this
)

const { compile, compileToFunctions } = createCompiler(baseOptions)

const createCompiler = createCompilerCreator(function baseCompile(
  template: string,
  options: CompilerOptions
): CompiledResult {
  // 对模板做解析，生成 ast语法树
  const ast = parse(template.trim(), options)
  // 优化 ast语法树   
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  // 优化后的 ast语法树转换成可执行代码
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})

function createCompilerCreator(baseCompile: Function): Function {
    return function createCompiler(baseOptions: CompilerOptions) {
        // some code deal baseOptions to finalOptions
        function compile(
            template: string,
            options?: CompilerOptions
        ): CompiledResult {
            const compiled = baseCompile(template.trim(), finalOptions)
            return compiled
        }

        return {
            compile,
            compileToFunctions: createCompileToFunctionFn(compile)
        }
    }
}

function createCompileToFunctionFn(compile: Function): Function {
    return function compileToFunctions(
        template: string,
        options?: CompilerOptions,
        vm?: Component
    ): CompiledFunctionResult {
        if (cache[key]) {
            return cache[key]
        }

        const compiled = compile(template, options)
        res.render = createFunction(compiled.render, fnGenErrors)
        res.staticRenderFns = compiled.staticRenderFns.map(code => {
            return createFunction(code, fnGenErrors)
        })

        
        return (cache[key] = res)
    }
}
```
## baseCompile(): { ast, render , staticRenderFns }

## createCompilerCreator(baseCompile: Function): createCompiler

## createCompiler(baseOptions: CompilerOptions):{ compile, compileToFunctions: createCompileToFunctionFn(compile) }
- 处理 baseOptions => finalOptions 
- 使用 createCompilerCreator 提供的 baseCompile 函数和 template ， finalOptions 构造 compile 函数
- 将 createCompileToFunctionFn(compile) 的结果作为 compileToFunctions 
- 返回 { compile, compileToFunctions }

## createCompileToFunctionFn(compile: Function): compileToFunctions

## compileToFunctions(template: string,options?: CompilerOptions,vm?: Component): { render , staticRenderFns }
- 使用 compile 生成 render 和 staticRenderFns
- 利用 cache 优化性能
- 返回 { render , staticRenderFns }


# 总结
这段绕来绕去的逻辑其实就是使用函数柯里化的方式对配置参数 baseCompile 和 baseOptions 进行了复用










