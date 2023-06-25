# 当 rrweb 遇见 Sentry

> 当 rrweb 遇见 Sentry，又该擦出怎么样的火花？

今天来聊一下 Sentry，Sentry 是一个哨兵平台，它可以针对项目中的报错，或者自定义规则，利用本身的上报，或者自定义的探针，来搜集程序的报错，同时还可以根据一些列的规则，触发相关的告警，通知关注者来关注这些问题，总而言之，它是一个服务稳定的保障。

![Sentry](https://pangji-home.com/eva_blogs/WX20230625-094436%402x.png)

> 有人可能会问为什么图片都这么黄，是因为我开了护眼模式...

Sentry 如何使用想来是不用多说，依葫芦画瓢就好，今天来看看另外一个主角 —— rrweb。

[rrweb](https://www.rrweb.io/) 是一个录制回放库，它可以将用户的行为录制下来，然后再根据录制的信息播放出来，注意，它并不是真正的“录屏”。

我们可以审查一下 rrweb 官网给出的[demo](https://www.rrweb.io/demo/checkout-form)，会发现其回放的时候，还真的不是 GIF 或者视频。

![rrweb 回放](https://mmbiz.qpic.cn/sz_mmbiz_gif/ornkSiaPibFvlZIrzRamffqKCaUtHkYib0pLnmR6wZDaF1Hw072lHic1pibibUv4cvYMOPXCetCbias7qwCwoqglm1GPA/0?wx_fmt=gif)

可以看出其本身是一个 iframe，而用户的行为轨迹则是通过一个透明的 canvas 盖在 iframe 上面，然后画出来的，由此也说明其再录制的时候并不是简单的录屏（录屏必然会丢掉这些绘制页面需要的信息）

联想一下 Sentry，我们在 Sentry 排查问题的时候，最需要的就是出现问题的场景，或者复现问题的手段，这种手段可以是日志、录屏、调用栈。

而 Sentry 目前给我们提供的内容有日志、调用栈，并没有录屏。

![Sentry 错误](https://pangji-home.com/eva_blogs/WX20230625-102526%402x.png)

这是一个示例错误，看的出来实际上 Sentry 提供了面包屑（即出现错误前后的调用栈），日志信息。而如果这种时候，再来个用户操作的行为轨迹（录屏），那岂不是更加完美了？而 rrweb 就为我们提供了这种手段。

鉴于 rrweb 的强大，Sentry 自然不会放过他，所以 Sentry 也集成了 rrweb。我们可以参考 Sentry 的[文档](https://docs.sentry.io/product/session-replay/)。

接下来就是在 Sentry 中接入 rrweb。我们创建一个项目，画一个按钮，点击的时候触发一个 `error(1234)`

```html
<template>
  <div class="about">
    <h1>This is an about page</h1>

    <button @click="handleButtonClick">这是一个按钮</button>
  </div>
</template>
```

```js
<script lang="ts" setup>
const handleButtonClick = () => {
  throw new Error('1234')
}

</script>
```

然后根据官网的例子，我们在 Sentry 初始化中加入 Replay

```js
Sentry.init({
  dsn: 'http://5826f63bf0fc4ba1b5d316dda4e13dea@192.168.31.100:9000/3',
  release: 'vue-sfc(0.0.1)',
  integrations: [
    new Sentry.Replay({
      // Additional SDK configuration goes in here, for example:
      maskAllText: true,
      blockAllMedia: true
    })
  ],
  // Set tracesSampleRate to 1.0 to capture 100%
  // of transactions for performance monitoring.
  // We recommend adjusting this value in production
  tracesSampleRate: 1.0,
  // This sets the sample rate to be 10%. You may want this to be 100% while
  // in development and sample at a lower rate in production
  replaysSessionSampleRate: 0.1,

  // If the entire session is not sampled, use the below sample rate to sample
  // sessions when an error occurs.
  replaysOnErrorSampleRate: 1.0
})
```

然后测试一下上报，不出意外，不行。同时因为我们是私有化部署，并且版本还比较老（docker-compose 版本限制），所以菜单上面并没有这个功能..

那我们先借用一下 Sentry 官方提供的免费版

![replays](https://pangji-home.com/eva_blogs/WX20230625-145251%402x.png)

可以看到这里会有个 Replays 的菜单，没错，它就是录制回放的菜单。

然后我们更换一下 DSN，再试一下，嘿嘿，还是不行。

![问号](https://pangji-home.com/eva_blogs/WX20230625-150833%402x.png)

当然，正常按照我们上面的代码，是的确会出现录制回放的，然鹅，我们忽略了一个问题，那就是我们初始化的时候开始录制了，但是没有结束录制，所以看不到视频，就像你录屏的时候，你不结束录制，是肯定看不到录制的视频的，因为录制的内容会在结束的时候最终生成。

> 什么？那为什么抛错不会结束录制？！震怒！

上面这个问题问的好，其实 Sentry 的 rrweb 的停止条件比较操蛋，它是这样的

- 正在记录的选项卡/页面内的用户不活动。（如果用户超过 15 分钟没有单击或浏览网站。鼠标滚动、鼠标移动和键盘事件目前不属于活动。）
- 录音达到最大重播时长限制（当前为 60 分钟）
- 手动调用replay.stop()方法

然后就没了...就没了...!

![无语](https://pangji-home.com/eva_blogs/png_temp_1570524540782_2064.jpg)

因此，需要二次开发，为了方便演示，我们这里改一下，手动停止一下

```js
const replay = new Sentry.Replay({
  maskAllText: true,
  blockAllMedia: true
})

Sentry.init({
  app,
  dsn: '__YOU DSN__',
  release: 'vue-sfc(0.0.1)',
  integrations: [
    new BrowserTracing({
      routingInstrumentation: Sentry.vueRouterInstrumentation(router),
      tracingOrigins: ['localhost', 'my-site-url.com', /^\//]
    }),
    replay
  ],
  // Set tracesSampleRate to 1.0 to capture 100%
  // of transactions for performance monitoring.
  // We recommend adjusting this value in production
  tracesSampleRate: 1.0,
  // This sets the sample rate to be 10%. You may want this to be 100% while
  // in development and sample at a lower rate in production
  replaysSessionSampleRate: 1.0,

  // If the entire session is not sampled, use the below sample rate to sample
  // sessions when an error occurs.
  replaysOnErrorSampleRate: 1.0
})

app.use(store).use(router).mount('#app')

setTimeout(() => {
  console.log('录屏停止')
  replay.stop()
}, 10000)
```

然后我们再次运行项目，等待10秒钟，然后打开 Sentry 后台，就可以看到效果了

![效果图](https://mmbiz.qpic.cn/sz_mmbiz_gif/ornkSiaPibFvlZIrzRamffqKCaUtHkYib0pb1N8D1AZWpOsd4DbIjGeszL5OoqKNATaicoY1MiadWQcvibf6fo8Xn1IA/0?wx_fmt=gif)

这里你会发现，Sentry 上面的回放页面，全是 `*` 号，很显然，这是 Sentry 做的加密处理，为的就是保护页面隐私不会被泄露（多么贴心），毕竟万一有什么隐私的数据，比如金融相关表单数据泄密，那就是很大的安全事件了。

如此，Sentry 接入 rrweb 就完成了，如果要接入实际工作中的话，就是二次开发了，毕竟它本身不支持报错的时候停止录制...

Happy coding~



