egg自带了参数验证的函数`ctx.validate()`，它默认的校验规则由[parameter](https://github.com/node-modules/parameter)模块提供，下面是一些对参数验证使用的总结  (基于parameter版本:3.6.0)

## 通用规则

|                   | 作用                            | 默认值 |
| ----------------- | ------------------------------- | ------ |
| `type`            |                                 |        |
| `required`        | 是否允许值为`null`或`undefined` | true   |
| `convertType`     |                                 |        |
| `default`         |                                 |        |
| `widelyUndefined` |                                 |        |



## 类型转换

### 默认转换





## 自定义规则



