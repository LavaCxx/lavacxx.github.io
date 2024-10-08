---
title: 一个月的Formily使用经验总结
---
## 背景
因为工作原因近一个月换了完全没用过的技术栈开发新项目。React、Storybook这些倒还好，基本一天就上手了，倒是Formily这厮陆续摸索了半个多月才算入门，趁一阶段工作结束空隙我就把这些经验总结一下。

> 因为项目原因我使用Formily的方式不一定是最佳实践甚至不一定是对的，对表单方面的使用也约等于零，主要场景集中于使用[@formily/antd-v5](https://antd5.formilyjs.org/)和自制组件上。

## 基础
Formily是阿里开发的一套以Schema描述表单的开源框架，不过我们项目中使用它的理由并不是构建表单，而是用于动态构建页面。

Formily里有三个比较重要的概念：**Form**、**Field**和**Schema**。
- **Form**是整个表单的核心，其实例可用于获取和设置表单的校验、值、字段、字段联动等内容
- **Field**是Form的组成部件，也就是字段，比如表单里设置了一个叫name为`keyword`的输入框，那每次修改输入框的内容时，`form.values.keyword`就会跟着变动。
- **Schema**是描述Field的数据，可以简单的认为其与ReactNode等价，具体一个Schema能包含什么内容可以参考[官方文档](https://react.formilyjs.org/zh-CN/api/shared/schema)。
	- 在Schema转组件的过程中Formily会默认代理组件的`value`和`onChange`，所以要开发Formily组件时要考虑到这一点。
## 从一个简单的页面开始
先从一个简单的示例开始

![[Pasted image 20241003170934.png]]

![[Pasted image 20241003171026.png]]

上面的代码创建一个==“带边框的div，里面装着一个输入框和另一个带了两个输入框的容器”==的场景，从代码中我们可以得到几个结论：
- Schema的层级结构相当于HTML中的层级结构，其中的`properties`起到了类似`children`的作用。
-  `createForm`中可以定义表单的一些初始状态？
-  `x-component`代表当前位置需要渲染的元素，可以支持原生的标签可自定义的组件，其中的自定义组件需要在`createSchemaField`中进行注册。
-  `x-component-props`中定义了组件的props，`x-decorator`定义了包装这个组件的组件。

简单操作一下表单，可以发现Schema和实际的表单值的映射关系。

![[Pasted image 20241003171319.png]]

![image](https://github.com/user-attachments/assets/015a0e94-c679-4946-b5f7-25b32587bc6b)

如果要管理不同字段的值，schema的`type`非常关键。
- 当`type`为`void`的时候，表单会忽略这一层的路径。
- 当`type`为`object`的时候，该字段会成为承接子字段的对象。
- 当`type`为基本类型的时候，该字段代表具体的值。

当时我们本来打算封装一个组合了Card + Tabs功能的组件，但是在Formily的机制下，字段本身不能既表示自己的值（Tabs的activeKey），也成为包含子字段的对象，所以最后打消了这种做法。

## 函数处理
如果坚持使用JSON Schema，那往Schema里塞函数的做法就显得不那么合理了。在Schema中，Formily会把`{{}}`的字符串处理成函数，所以要转换一下写法。
```tsx
// ...
	input: {
		type: "string",
		"x-component": "Input",
		"x-component-props": {
			// onClick: `{{ (event) => { console.log(event) } }}
		}
	}
// ...
```

同样的，对需要传ReactNode的props也是如此处理，不过需要使用`React.createElement`

```tsx
import { createElement } from "react";
import { Input } from '@formily/antd-v5'
import { SearchOutlined } from '@ant-design/icons';

// ...
const SchemaField = createSchemaField({
	components: {
		Input
	},
	scope: {
		createElement,
		SearchOutlined
	}
})
// ...
input: {
	type: "string",
	"x-component": "Input",
	"x-component-props": {
		suffix: `{{ createElement(SearchOutlined) }}`
	}
}
// ...
```

## 字段联动
这部分内容[官方文档](https://formilyjs.org/zh-CN/guide/advanced/linkages)说得就挺好的，就不赘述了

## 数据传递
我们当时的页面有一个联动逻辑，当在顶部导航栏切换时间后，下面的各个图表都要同步更新数据，而我们的解决方式如下，不一定是最佳实践。

![image](https://github.com/user-attachments/assets/7b66b29d-b1fe-4865-a434-30ba5d0e1559)

## 组件开发流程
Formily组件和常规的React组件差别还是挺大的，如果把`@formily/antd-v5`作为官方的推荐实践方式的话，那就不能以之前的思路来开发组件。

以antd的Table为例，无论是`columns`还是`dataSource`都是作为props的一部分传递给组件的，但在Formily中，数据应只由于默认配置的`props.value`来处理，所以开发时应该做好渲染数据与`props.value`的转换。对接已有组件时也可以用官方的[mapProps](https://react.formilyjs.org/zh-CN/api/shared/map-props)方法做映射。

> 要较真的话数据放在哪都是可以实现功能的，但能简单的从`form.values`拿到所有值还是比把数据分散到field的`componentProps`要方便的。

除此之外，善用`useForm`、`useField`和`useFieldSchema`可以方便获取父子字段的内容。

## 一些坑
#### ReactNode转Schema
虽然可以用`{{createElement}}`的方式传递ReactNode，但对于自定义组件来说还需要提前把组件传入scope中，对于动态的Schema来说，这难以做到按需导入。

我当时的做法是组件内部做拦截。用`useFieldSchema`拿到Schema，然后判断对应的属性是否也是一个Schema对象，如果是则返回一个`RecursionField`来渲染Schema。

拦截写完后乍一看还没什么毛病，但是在antd的Table sortIcon时就傻眼了，渲染的图标在点击后完全没有状态变化。

个人猜测这跟Formily的渲染机制有关，`RecursionField`的渲染结果被缓存了，只是重新运行函数传递新的props还不足以触发其重新渲染。

#### onChange
还是跟Table的过滤有关，原流程中点击表头的排序会触发表格的`onChange`事件获得排序信息。

但我在尝试换了多种写法后发现仍不能触发这个`onChange`，看了[源码](https://github.com/alibaba/formily/commit/11e14a3973bd19bb38d5371aa8dd6cc2163c821b)后才发现为了防止冒泡官方给`onChange`覆盖了个空函数。你们是完全用不到这个功能吗？

️业务层面没什么解决方法，这里最后用了[[利用patch修改依赖源码|patch]]的方式解决的，颇为麻烦。
️️
