# 当小程序遇见 tailwindcss

你是否还在为怎么写样式而烦恼？是否因为样式命名而痛苦？又或者为团队其他同学的命名、书写风格不一致而苦恼？那么现在有一种技术，可以彻底解决这个问题，让你不在苦恼~

没错，它就是 **tailwindcss** ~

> 声明，本人除了个人博客项目里面有 tailwindcss，其他地方还没接入~ 不过本文也本身是探索性的文章，欢迎大家不吝赐教。

也许大家都听过这个东西，但是却没有用过；又或者已经在使用了，这些都不重要，今天我们就从头开始探索一下 tailwindcss。

回到 css 的世界上来，本身 css 并没有规定我们应该怎么写类名，或者说怎么写样式，只是现代前端工具链做出了一些规范，比较著名的就是 `stylelint`，它规定了样式的顺序，甚至也可以对命名做出一些规定，比如类名不能用短横杠（-），不能用连续的下划线（__）等等。

联想一下比较久远的过去，当大家都还在用 bootstrap 的年代，是否还记得那些经典的样式类？比如下面这些

```xml
<div class="col-sm-12">你好呀<div>

<div class="dropdown">你好呀<div>
```

当初还是小白的我们会觉得，卧槽！好牛逼，都不需要自己写 css！而现在，我们不难理解，它其实是将常见的 css 进行了封装，集成进了 bootstrap 本身的样式类里面。

**不只是 utility class ！**

tailwindcss 也是类似的思路，只不过更加专业，它将样式全部拆分为原子（Atomic CSS），通过自己定义的 `token`，组合成一个句子，然后进行编译。

tailwindcss 本身并不是一个 utility class，其本身应该算是一个 DSL，它本身每一个 `token` 就是一个原子化的 class，或者说属于一个细小颗粒度的 mixin，当本身的 `token` 组合成了一个句子，其自身的 DSL 编译器会将其编译成我们看到的 css 效果。

铺垫了这么多，我们来实际使用一下，实际上 vue， react等项目都有很成熟的接入方案，我们回到标题，实操一下在小程序里面如何接入 tailwindcss。

根据官网的文档，第一步是先安装

```shell
yarn add tailwindcss -D
npx tailwindcss init
```

然后就会在项目下创建一个 `tailwind.config.js`

![初始化](https://cdn.nlark.com/yuque/0/2023/png/583529/1687313808146-4db7bf76-0dbd-48e8-b08d-52b62f339235.png)

然后我们配置项目文件

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./pages/**/*.{wxml,js}", "./components/**/*.{wxml,js,ts}"],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

然后根据官网的文档去书写主入口文件 `tailwind.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

并且配置命令

```package.json
{
  "scripts": {
    "dev": "npx tailwindcss -i ./tailwind.css -o ./app.scss --watch"
  }
}
```

然后执行，不出意外应该是好了，然鹅不出意外它就要出意外了~ 

![控制台报错](https://cdn.nlark.com/yuque/0/2023/png/583529/1687317459196-8c63da42-28e2-4167-9ad8-e704c4360560.png?x-oss-process=image%2Fresize%2Cw_1148%2Climit_0)

显然，小程序不支持通配符 `*`，并且我们观察一下生成的 app.scss 会发现，其生成的很多都是 html 的配置样式

![app.scss](https://cdn.nlark.com/yuque/0/2023/png/583529/1687317546469-44053d57-d45d-478b-b4d9-f86b9f3c1b19.png?x-oss-process=image%2Fresize%2Cw_1282%2Climit_0)

这显然是不行的，为了解决这个问题，我们就需要对配置内容进行修改，让 tailwindcss 生成小程序的内容。

这个时候我们新增一个小程序的预设，即 `presets`，让 tailwindcss 生成小程序相关的代码即可

```shell
yarn add tailwindcss-miniprogram-preset -D
```

然后调整配置文件

```javascript
const {
  defaultPreset,
  // createPreset 可选
} = require('tailwindcss-miniprogram-preset')

/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./pages/**/*.{wxml,js}", "./components/**/*.{wxml,js,ts}"],
  plugins: [],
  presets: [defaultPreset],
  darkMode: false,
  important: true, // 需要用它去覆盖原生的样式
  theme: {
    extend: {},
    screens: false, //小程序它不需要 pc 那种自适应边界
  },
}
```

然后重新执行，会发现还是不行！并且我们观察 `app.scss` 这个输出文件，发现，文件内容变了，HTML相关的内容没有了，但是还是有 `*` 通配符，这个时候我们需要修改一下入口文件 `tailwind.css`

```css
/* @tailwind base; */

/* @tailwind components; */
@tailwind utilities;
```

这是因为小程序并不需要 `base` 和 `components` 这2个东西，然后重新生成 `app.scss`，此时！见证奇迹的时刻就到了

![app.scss](https://cdn.nlark.com/yuque/0/2023/png/583529/1687318391899-d63956ff-88a7-4920-8986-e4d82fe664b8.png?x-oss-process=image%2Fresize%2Cw_1176%2Climit_0)

我们在小程序代码中写上 tailwindcss 的类名

```xml
<view class="usermotto">
  <text class="user-motto text-3xl font-bold underline">{{motto}}</text>
</view>
```

效果如下

![效果图](https://cdn.nlark.com/yuque/0/2023/png/583529/1687318490542-0126e1a3-9951-416d-b13b-7673debe7aee.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0)

你以为这样就完了？并没有！

我们观察一下生成的 `app.scss`，会发现，`app.scss` 是没有压缩的，因此我们可以对 `app.scss` 文件进行压缩。


```json
{
  "scripts": {
    "dev": "npx tailwindcss -i ./tailwind.css -o ./tailwind.scss --minify"
  }
}
```

如此就能输出压缩的文件了。

这里还有一个问题，很多情况下，我们会有一些自定义的样式类，我们需要在 `app.scss` 中完成，因此生成的文件不能是 `app.scss`，我们需要调整一下输出文件

```json
{
  "scripts": {
    "dev": "npx tailwindcss -i ./tailwind.css -o ./tailwind.scss --watch"
  }
}
```

然后 `app.scss` 调整引用

```scss
@import './tailwind.scss';
```

> 另外，需要注意的是 tailwindcss 的版本需要 V2 版本。

不知道大家发现没有，我们这个文章中的例子已经不知不觉的把 tailwindcss 和 scss 相结合起来，那也就是说明，它不仅仅只是 css，还可以和 scss，less 等 css 预处理器结合，发挥其更大的作用！

enjoy coding~