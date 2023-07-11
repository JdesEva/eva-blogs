# puppeteer，说声爱你不容易

> 本文是结合自身工作经历以及相关资料的方案，不代表最优解，请各位看官多多担待，多多支持。

> 转载自我的语雀，并做了适当修改和优化调整。

## Introduction

海报，或者说活动页面输出，可以理解为企业活动页面的对外宣传输出，一般会输出为大图，或者 PDF 等形式，在非常多的情况下，活动页面是由前端拿到UI设计稿后进行页面绘制，然后导出为大图（PDF 原理一致）。此时我们作为搬砖人一般会有几个选择

1. 调用 html2canvas 在客户端生成图片并导出下载
2. 利用 puppeteer
3. 服务端渲染后直接导入绘制并截图
4. 其他手段（~~消耗一个UI导出一副海报~~）

当然海报服务不仅仅只是用于生成海报这种特定场景，还能用于页面效果导出，分享页，结果页导出分享等等。

## 初识

有幸（~~被坑~~）接到过输出大图的需求，大概是结果页面导出为 PDF 和截图，当时第一版的方案是调研了好多内容，然后使用的 `html2canvas` 在客户端生成图片，然后使用 `pdf.js` 消费图片生成 PDF。

这个方案没什么太大设计上的问题，缺点就是客户端操作需要用户进行等待，等待时长视需要导出的结果页面的大小而定，~~当初的我也是这么认为的~~，直到我遇到了 iOS13.2 这个万恶的版本。是的，这个版本的微信 webview（微信H5）对于 `html2canvas` 以及 `pdf.js` 支持有致命问题，会导致不可用。微信官方于2020.4 收到了对于该问题的反馈，官方给出说法是将于2个月左右时间修复此问题，跟随微信版本更新发布修复，但是目前，此问题依旧未修复。

上面说到由于 `html2canvas` 在 iOS13+ 微信环境下有致命缺陷，导致服务不可用，为了规避这个问题，只能改变实现手段，这个时候 `puppeteer` 进入了视线。

`puppeteer` 是谷歌于2017年开源的基于 `Chrome headless` 特性的开源库，大概意思就是通过命令行去调用无界面版本的 Chrome，然后完成一系列的操作，更进一步的说，我们借助 `puppeteer`，实际上还是通过 [Chrome devTools](https://chromedevtools.github.io/devtools-protocol/) 的开放接口与 `Chrome` 通讯。

由于 `Chrome devTools` 极度复杂，具体可以查看如下图，因此 Chrome 团队开源了 `puppeteer` 去作为一个 API 工具去调用 `Chrome devTools`，这是一个非常明智的决定，从而解放了 `Chrome devTools`，让其不重蹈 node 的覆辙。

> node 的问题在于 C++ 的 `addon`，有兴趣同学欢迎私聊。

![Chrome devTools API](https://pangji-home.com/eva_blogs/image.png)

- `puppeteer` 使用的是 Chrome DevTool Protocol。具体可以查看[这篇文章](https://segmentfault.com/a/1190000037762174)。

作为一个前端搬砖人，Chrome 的强大不必多说，nodejs 背靠 v8 内核，以及支持调用 C、C++ 的功能硬生生打开了前端新世界的大门，从此世界上多了一门语言。

言归正传，我们可以在 [puppeteer代码仓库](https://github.com/puppeteer/puppeteer)的 markdown 文档下看到以下内容

- Generate screenshots and PDFs of pages.

生成页面截图或者导出为PDF

- Crawl a SPA (Single-Page Application) and generate pre-rendered content (i.e. "SSR" (Server-Side Rendering)).

抓取SPA页面内容（单页应用程序）并生成预渲染内容（SSR）

- Automate form submission, UI testing, keyboard input, etc.

自动填充表单提交，UI测试，键盘输入等等

- Create an up-to-date, automated testing environment. Run your tests directly in the latest version of Chrome using the latest JavaScript and browser features.

创建最新的自动化测试环境，并使用最新的JS和浏览器功能特性以及最新的Chrome运行测试。

- Capture a [timeline trace](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/reference) of your site to help diagnose performance issues.

捕获网站的性能时间线，以用来帮助诊断性能问题

- Test Chrome Extensions.

测试Chrome扩展程序

以上这些也只是 `puppeteer` 官方举的🌰，但是 `puppeteer` 的强大我们由此窥见一斑。话不多说，回归正题，既然它能解决问题，那就是它了。

## 上手

在确定 `puppeteer` 是一个好东西并且能完美达成所想之后，话不多说，直接就开始安装 `puppeteer`。

`npm i puppeteer --save`

![安装 puppeteer](https://pangji-home.com/eva_blogs/puppeteer.png)

执行过程中可以看到会下载 `Chromium`，`puppeteer` 基于 Chrome，下个 Chromium 不是很正常？查看文档会发现，在 Mac 下，Chromium 大概为170MB，Linux下为282MB，Windows下为280MB（不愧是你，Windows）。当然官方考虑到我们设备上可能存在了 Chrome，也提供了`puppeteer-core`这个包。

`puppeteer-core`只提供了API去调用 Chrome，并不会包含 Chromium，需要指定 Chromium 参数，考虑到我们只是为了梭哈需求，还是老老实实用 puppeteer 完整版更好。

> 另外提一嘴，由于 `puppeteer` 的**基础包**（也有 Python 包，和 Java 包，都是社区封装的）只支持在 node 环境下调用，因此，这个东西必须由服务端实现

话不多说，开始写代码。参考官方文档的🌰，我们可以写出如下代码
```javascript
'use strict';

const Service = require('egg').Service;
const puppeteer = require('puppeteer');
const path = require('path');


class CanvasService extends Service {
  
  async screenshot(data) {
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    await page.goto('https://www.baidu.com');
    await page.screenshot({
      path: path.resolve(__dirname, '../public/baidu.png'),
      type: 'png',
      fullPage: true,
    });
    await browser.close();
  }
}

module.exports = CanvasService;

```
服务端框架为 [egg](https://www.eggjs.org/zh-CN/basics/structure)。

然后我们调用这个 service，在等待一段时间后，可以在`/public`文件夹下看到对应的截图。

![生成的效果图](https://pangji-home.com/eva_blogs/baidu.png)
pdf 的生成也同理，只不过将 API 更换为 `page.pdf()`，如下代码

```javascript
async pdf() {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.goto('https://www.eggjs.org/zh-CN/basics/structure');
  await page.pdf({
    path: path.resolve(__dirname, '../public/egg.pdf'),
    format: 'A4',
    printBackground: true,
    displayHeaderFooter: true,
  });
  await browser.close();
}
```
![egg 页面](https://pangji-home.com/eva_blogs/egg.png)

当然，因为我们传递了 `path` 参数，所以 `puppeteer` 会将生成结果保存在我们定义的 `path` 路径下，因为这种情况相当于会在服务器上不停的写入临时文件，占用服务器资源，`puppeteer` 也很贴心的为我们提供了 `stream` 模式，将生成结果输出为 `stream`，让我们消费掉即可，不会占用服务器资源（仍然会占用内存），具体可以参考 `puppeteer` 的 API 即可。

从上面的🌰来看，`puppeteer` 用起来很简单是吗？没错，API 非常简单，并且从上述代码不难看出，代码非常语义化，几乎就是“所见即所得”。因为本身 `puppeteer` 就是在 node 环境下打开一个 Chrome，然后通过我们的输入参数，打开页面，然后去做我们需要的操作，因此本身并不难理解。

## 问题

你以为这个东西就那么简单的完成了我们所需要的，确实，它完成了我们所需要的，截图和 pdf 服务，但是你以为这就结束了？少年人，你太天真了。上文中我们可以看到，我们打开 [egg官网](https://www.eggjs.org/zh-CN/basics/structure)，然后生成 pdf，达到了我们所要的结果，那么过程呢？我们来看一下 postman 的请求。

![超长的生成时间](https://pangji-home.com/eva_blogs/postman.png)

看到这个**7.56s**的请求时间，惊喜吗？意外吗？试想一下，用户需要等待那么长时间才能得到结果会是什么反应，而且这还是一个简单的单页面，egg 的官网并不复杂，也没有花里胡哨的动态效果，那如果是一个炫酷的活动页面，那时间开销想必更加巨大。

回归到这个请求时间上来，我们反问一下自己，这个请求时间，能做性能优化吗？从开始请求到拿到结果，代码均在上面贴出，那么我们将逐行剖析一下，这些代码做了什么，为什么运行结束就变成了**7.65s**。

```javascript
async pdf() {
  const browser = await puppeteer.launch(); // 启动一个puppeteer实例（打开浏览器）
  const page = await browser.newPage(); // 新开一个页面
  // 将新页面的地址重定向到目标页面
  await page.goto('https://www.eggjs.org/zh-CN/basics/structure'); 
  // 开始生成pdf
  await page.pdf({
    path: path.resolve(__dirname, '../public/egg.pdf'),
    format: 'A4',
    printBackground: true,
    displayHeaderFooter: true,
  });
  // 关闭浏览器
  await browser.close();
}
```
上述代码注释中我们可以看到，做了以下的事情

1. 打开一个浏览器
2. 打开一个新的页面
3. 地址栏输入，并按回车
4. 保存页面为 pdf（Chrome 自带的打印功能）
5. 关闭浏览器

结合我们自身对 Chrome 的使用经验，我们一个个进行分析，上述都是必要操作，并且打印生成 pdf（截图类似原理）是必须要花费的时间，同时，跳转到新页面还要视网络状况，页面大小等原因影响，这也是必须花费的时间。

那么问题就可能出现在第一步第二步，由于脚本的执行速度极快，脚本执行时间我们可以忽略。设想一下在我们在自己的电脑打开一个新的 Chrome 的时候，仔细观察，会发现是需要卡顿（视电脑性能、CPU 内存占用、硬盘读写等情况）一下，可以理解为打开浏览器的操作是一个非常占用资源的操作。

即 `const browser = await puppeteer.launch(); `是一个高消耗的操作，同时，如果我们对这个服务使用非常频繁（相对）的情况下，此过程还是一个频繁消耗资源的操作，有点类似于链接数据库。类比一下，打开浏览器是链接并登录 MySQL 数据库，新开页面类似于进入其中的数据库，跳转新页面类似于数据查询（执行 SQL），然后返回我们需要的结果，是不是有那么点意思了？嘿嘿。

## 排查

后端对于数据库这种高 `IO（Input & Output）`的操作，比较推荐或者说主流的做法是新建一个连接池，保持这个服务进程常驻，定期检查，需要使用时，从池子里面取出来一个连接，用完了释放掉（release）。

参考后端做法，我们只需要做个 `puppeteer` 线程连接池，保持进程常驻，然后用的时候拿一个链接，用完了释放掉即可。桥豆麻袋！那我们还需要实现一个线程连接池？！当然不（~~能白嫖的我们为什么还要自己写呢？~~）。

实际上大佬们已经为我们写好了nodejs下的线程连接池 [generic-pool](https://www.npmjs.com/package/generic-pool)，并且还是一个通用的库，直接拿过来用就行，并且还天生支持Promise（也可以支持别的自定义的符合  [Promise/A+](https://promisesaplus.com/) 标准的 Promise库）。

根据 generic-pool 文档，以及网络上大佬们的踩坑记录，我们可以写出如下 puppeteer 线程连接池的生成代码。

```javascript
'use strict';
const puppeteer = require('puppeteer');
const genericPool = require('generic-pool');


/**
 * @Author: jdeseva
 * @description: 初始化 Puppeteer 线程池
 * @param {Object} options 创建池的配置配置
 * @description: options 列表
 * @param {Number} [options.max=10] 最多产生多少个 puppeteer 实例 。如果你设置它，请确保 在引用关闭时调用清理池。 pool.drain().then(()=>pool.clear())
 * @param {Number} [options.min=1] 保证池中最少有多少个实例存活
 * @param {Number} [options.maxUses=2048] 每一个 实例 最大可重用次数，超过后将重启实例。0表示不检验
 * @param {Number} [options.testOnBorrow=2048] 在将 实例 提供给用户之前，池应该验证这些实例。
 * @param {Boolean} [options.autostart=false] 是不是需要在 池 初始化时 初始化 实例
 * @param {Number} [options.idleTimeoutMillis=3600000] 如果一个实例 60分钟 都没访问就关掉他
 * @param {Number} [options.evictionRunIntervalMillis=180000] 每 3分钟 检查一次 实例的访问状态
 * @param {Object} [options.puppeteerArgs={}] puppeteer.launch 启动的参数
 * @param {Function} [options.validator=(instance)=>Promise.resolve(true))] 用户自定义校验 参数是 取到的一个实例
 * @param {Object} [options.otherConfig={}] 剩余的其他参数 // For all opts, see opts at https://github.com/coopernurse/node-pool#createpool
 * @return {puppeteerPool}
 */
const initPuppeteerPool = (options = {}) => {
  const {
    max = 10,
    min = 2,
    maxUses = 2048,
    testOnBorrow = true,
    autostart = false,
    idleTimeoutMillis = 3600000,
    evictionRunIntervalMillis = 180000,
    puppeteerArgs = {},
    validator = () => Promise.resolve(true),
    ...otherConfig
  } = options;

  const factory = {
    create: () =>
      puppeteer.launch(puppeteerArgs).then(instance => {
        // 创建一个 puppeteer 实例 ，并且初始化使用次数为 0
        instance.useCount = 0;
        return instance;
      }),
    destroy: instance => {
      instance.close();
    },
    validate: instance => {
      // 执行一次自定义校验，并且校验校验 实例已使用次数。 当 返回 reject 时 表示实例不可用
      return validator(instance).then(valid => Promise.resolve(valid && (maxUses <= 0 || instance.useCount < maxUses)));
    },
  };
  const config = {
    max,
    min,
    testOnBorrow,
    autostart,
    idleTimeoutMillis,
    evictionRunIntervalMillis,
    ...otherConfig,
  };
  const pool = genericPool.createPool(factory, config);
  const genericAcquire = pool.acquire.bind(pool);
  // 重写了原有池的消费实例的方法。添加一个实例使用次数的增加
  pool.acquire = () =>
    genericAcquire().then(instance => {
      instance.useCount += 1;
      return instance;
    });
  pool.use = fn => {
    let resource;
    return pool
      .acquire()
      .then(r => {
        resource = r;
        return resource;
      })
      .then(fn)
      .then(
        result => {
          // 不管业务方使用实例成功与否都表示一下实例消费完成
          pool.release(resource);
          return result;
        },
        err => {
          pool.release(resource);
          throw err;
        }
      );
  };
  return pool;
};

module.exports = { initPuppeteerPool };


```
在新增 `puppeteer` 线程连接池之后，我们修改一下 `service` 和 `controller`，大致如下（相关内容均可以查看文档，或者网络上类似的解决办法）

```javascript
async screenshot() {
  const { app } = this;
  await app.pool.use(async puppeteerInstance => {
    const page = await puppeteerInstance.newPage();
    await page.goto('https://www.baidu.com');
    await page.screenshot({
      path: path.resolve(__dirname, '../public/baidu.png'),
      type: 'png',
      fullPage: true,
    });
    await page.close();
  });
}


async pdf() {
  const { app } = this;
  await app.pool.use(async puppeteerInstance => {
    const page = await puppeteerInstance.newPage();
    await page.goto('https://www.eggjs.org/zh-CN/basics/structure');
    await page.pdf({
      path: path.resolve(__dirname, '../public/egg.pdf'),
      format: 'A4',
      printBackground: true,
      displayHeaderFooter: true,
    });
    await page.close();
  });
}
```

修改完毕之后，启动服务，正常前几次（1-3次）请求时间大致为2-3s，线程连接池稳定之后，时间如下图

![稳定之后的时间](https://pangji-home.com/eva_blogs/postman2.png)

**时间肉眼可见的缩短了10倍！！！**

当然这边也有一个问题就是为什么一开始的时间还是比较久？其实主要有以下两个原因

1. `puppeteer` 线程连接池被我们设置了是调用的时候初始化 `puppeteer` 实例，并不是在新建线程连接池（应用启动）的时候初始化，初始化 `puppeteer` 实例需要时间。
2. `puppeteer` 的原理是新开浏览器页面访问新页面，当完完全全访问一个新的页面，资源加载比较多（首次加载），后续因为可以合理利用浏览器缓存，将显著减少加载时间。（实际上这也是我们前端搬砖人需要解决的问题，这种情况下首次进入页面的首屏加载问题是前端开发的过程中不可避免会遇到的问题）

当然事情进行到这里，我们是不是可以画上一个句号了？当然没有！回到最开始的问题上来，我们的初衷是基于 `puppeteer` 设计一个稳定的海报服务，所以还需要部署到服务器上（上线）。

## 部署

在容器化大行其道的现在，很多服务都是容器化部署的，因此，我们也将使用 docker 对上述应用进行部署，首先部署之前要查看 [egg的文档](https://www.eggjs.org/zh-CN/core/deployment)。文档中很明确写了部署中需要注意的问题，不同部署方式对于命令的影响，**在任何项目需要部署（不只是部署，其他时候也是）的时候，仔细查看文档都是一件很重要的事情。**

服务构建过程不必多说，根据文档，我们可以看到 egg 部署需要 node 环境，于是我们拉取 node 镜像

```shell
docker pull node:latest
```

然后构建 docker 镜像，大致的 Dockerfile 如下（仅供参考）

```dockerfile
# node镜像
FROM node:latest

# 设置时区 并创建工作目录
ENV TZ "Asia/Shanghai"
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo '$TZ' > /etc/timezone && mkdir -p /usr/src

# 设置工作目录
WORKDIR /usr/src

# 拷贝package.json文件到工作目录
# !!重要：package.json需要单独添加。
# Docker在构建镜像的时候，是一层一层构建的，仅当这一层有变化时，重新构建对应的层。
# 如果package.json和源代码一起添加到镜像，则每次修改源码都需要重新安装npm模块，这样木有必要。
# 所以，正确的顺序是: 添加package.json；安装npm模块；添加源代码。
COPY package.json /usr/src/package.json

# 安装npm依赖(使用淘宝的镜像源)
# 如果使用的境外服务器，无需使用淘宝的镜像源，即改为`RUN npm i`。
RUN npm config set registry https://registry.npmmirror.com/ && npm config set sass_binary_site https://npm.taobao.org/mirrors/node-sass/ && npm i --production

# 拷贝所有源代码到工作目
COPY . /usr/src

# 暴露容器端口
EXPOSE 7400

CMD npm run start

```
然后打包完成镜像之后，执行如下命令运行镜像

```shell
docker run -d --name node-server -p 3000:7400 node-server
```

然后执行请求，接口GG，我们查看服务日志

```shell
docker logs -f --tail  300 node-server
```

![日志报错](https://pangji-home.com/eva_blogs/log.png)

可以看到接口基本是 G 的，当然有很多原因，主要会有两种情况

![postman](https://pangji-home.com/eva_blogs/postman3.png)

1. 权限不足
2. 没有 puppeteer 依赖

让我们带着疑问来到一个地方 [running-puppeteer-in-docker](https://github.com/puppeteer/puppeteer/blob/main/docs/troubleshooting.md#running-puppeteer-in-docker)。大概意思是如果在 docker 环境下运行  `puppeteer`，如果是使用 node 镜像那么会因为缺少某些共有依赖（系统级的依赖）而没有权限，因此 `puppeteer` 官方提供了一个官方的 `puppeteer` 在 docker 环境运行的基础 Dockerfile 以构建基础的服务镜像。

```dockerfile
FROM node:12-slim

# Install latest chrome dev package and fonts to support major charsets (Chinese, Japanese, Arabic, Hebrew, Thai and a few others)
# Note: this installs the necessary libs to make the bundled version of Chromium that Puppeteer
# installs, work.
RUN apt-get update \
    && apt-get install -y wget gnupg \
    && wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && sh -c 'echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list' \
    && apt-get update \
    && apt-get install -y google-chrome-stable fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-kacst fonts-freefont-ttf libxss1 \
      --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# If running Docker >= 1.13.0 use docker run's --init arg to reap zombie processes, otherwise
# uncomment the following lines to have `dumb-init` as PID 1
# ADD https://github.com/Yelp/dumb-init/releases/download/v1.2.2/dumb-init_1.2.2_x86_64 /usr/local/bin/dumb-init
# RUN chmod +x /usr/local/bin/dumb-init
# ENTRYPOINT ["dumb-init", "--"]

# Uncomment to skip the chromium download when installing puppeteer. If you do,
# you'll need to launch puppeteer with:
#     browser.launch({executablePath: 'google-chrome-stable'})
# ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD true

# Install puppeteer so it's available in the container.
RUN npm init -y &&  \
    npm i puppeteer \
    # Add user so we don't need --no-sandbox.
    # same layer as npm install to keep re-chowned files from using up several hundred MBs more space
    && groupadd -r pptruser && useradd -r -g pptruser -G audio,video pptruser \
    && mkdir -p /home/pptruser/Downloads \
    && chown -R pptruser:pptruser /home/pptruser \
    && chown -R pptruser:pptruser /node_modules \
    && chown -R pptruser:pptruser /package.json \
    && chown -R pptruser:pptruser /package-lock.json

# Run everything after as non-privileged user.
USER pptruser

CMD ["google-chrome-stable"]
```

非常复杂的一个镜像，如果我们不想思考，也懒得手动构建，我们可以偷懒，直接在 Docker Hub 上搜索 `puppeteer`，然后去下载一个稳定镜像，然后以此作为我们服务的基础镜像即可，然后我们就可以愉快的运行起我们的服务了！

当我们进行以上操作的时候，如果“幸运”的话，那么基本上也可以将其投入小规模应用了，当然，也只是“幸运”，实际上，还会大概率遇到以下问题

1. 并发问题
2. 字体问题
3. 性能问题

并发和性能问题非常好理解，因为 node （js）是单线程模型，尽管其提供了回调 IO 去解决高并发场景下的压力，但是实际上如果一个接口崩溃导致服务停止，那么大概率后面的接口也就GG，哪怕加了进程守护也是如此，因为重启进程需要时间，秒开也仍然是需要时间（重启线程池，重启服务等），更何况，这种服务本身就是非常消耗服务器资源的一种服务。

所以现在更流行的做法是使用多线程协同操作，实际上 egg 也为我们提供了[解决方案](https://www.eggjs.org/zh-CN/core/cluster-and-ipc)。该方案基于 Nodejs 官方提供的 [Cluster 模块](https://nodejs.org/api/cluster.html)，在 egg 场景下，是使用一个 Master 进程，N 个 worker 进程，一对多的模式。

回到问题本身，在 egg 部署文档中我们会看到如下内容

![image.png](https://pangji-home.com/eva_blogs/doc.png)

其中有一条 `--workers=2`这一条，很显然这里是启动多个 worker 进程，然后回归到我们的代码上来，我们启动了一个 `puppeteer` 线程池，正常情况下是启动自定义，然后进行启动线程池，如下代码

```javascript
'use strict';
const { EventEmitter } = require('events');

const { initPuppeteerPool } = require('./app/plugins/puppeteer-pool');


EventEmitter.defaultMaxListeners = 30;

class AppBootHook {
  constructor(app) {
    this.app = app;
  }

  configWillLoad() {
    // 此时 config 文件已经被读取并合并，但是还并未生效
    // 这是应用层修改配置的最后时机
    // 注意：此函数只支持同步调用


  }

  async beforeStart() {
    // doSomething
  }

  async didLoad() {
    // 所有的配置已经加载完毕
    // 可以用来加载应用自定义的文件，启动自定义的服务

    // 创建 PuppeteerPool
    this.app.pool = initPuppeteerPool({
      puppeteerArgs: {
        ignoreHTTPSErrors: true,
        headless: true, // 是否启用无头模式页面
        timeout: 0,
        pipe: true, // 不使用 websocket
        args: [
          '--start-maximized',
          '--no-sandbox',
          '--disable-gpu',
          '--disable-dev-shm-usage',
          '--disable-setuid-sandbox',
          '--no-first-run',
          '--no-zygote',
          '--single-process',
          '--disable-extensions',
        ],
      },
    });
  }

  async willReady() {
    // 所有的插件都已启动完毕，但是应用整体还未 ready
    // 可以做一些数据初始化等操作，这些操作成功才会启动应用


  }

  async didReady() {
    // 应用已经启动完毕


  }

  async serverDidReady() {
    // http / https server 已启动，开始接受外部请求
    // 此时可以从 app.server 拿到 server 的实例

  }

  async beforeClose() {
    // doSomething
    if (this.app.pool.drain) {
      await this.app.pool.drain().then(() => this.app.pool.clear());
    }
  }
}

module.exports = AppBootHook;

```

上述代码位于根目录的 app.js 中，然后查看文档我们就会发现，`app.js` 的代码实际上是运行在 worker 进程中的，意味着如果我们指定 `--works=2` 就会**启动2个线程池**，如果设置 n 就是 n 个线程池！显然这个和我们的想法背道而驰。

所以为了使用多线程，我们需要修改我们的代码模型，这个时候，egg 的一个[多进程模型](https://www.eggjs.org/zh-CN/advanced/cluster-client)进入了我们的视野，那就是 IPC（Inter-Process Communication），即进程间通讯，对应的还有 RPC（Remote Procedure Call，远端服务调用）。根据这种模式，我们可以使用 agent 启动 `puppeteer` 线程池，然后启动多个 worker，然后通过进程间通讯去调用服务，这样即达成了多个 worker 提升服务稳定性，又保证了全局只有一个 `puppeteer` 线程连接池，保证资源最大化的利用。

**注： 下面代码只是提供思路参考，不保证没有问题**

```javascript
'use strict';
const { initPuppeteerPool } = require('./app/plugins/puppeteer-pool');

module.exports = agent => {
  const { messenger } = agent;
  const ref = {};
  // 启动线程池
  messenger.on('start-pool', () => {
    ref.pool = initPuppeteerPool({
      puppeteerArgs: {
        ignoreHTTPSErrors: true,
        headless: true, // 是否启用无头模式页面
        timeout: 0,
        pipe: true, // 不使用 websocket
        args: [
          '--start-maximized',
          '--no-sandbox',
          '--disable-gpu',
          '--disable-dev-shm-usage',
          '--disable-setuid-sandbox',
          '--no-first-run',
          '--no-zygote',
          '--single-process',
          '--disable-extensions',
        ],
      },
    });
  });
  // 结束线程池
  messenger.on('stop-pool', () => {
    if (ref.pool.drain) {
      ref.pool.drain().then(() => ref.pool.clear());
    }
  });

  messenger.on('on-screenshot', async uid => {
    // console.log('on-screenshot', uid);
  });

  messenger.on('on-pdf', async (data, cb) => {
    console.log('on-pdf', data, cb);
  });
};

```

> 思考一个问题，这种做是否真的合适吗？有无更好的方案？

## 话外之音

在进行上面的所有操作之后，不出意外的话，我们是可以拿到结果了，但是！拿到的结果很可能是白的，或者仅有背景，有文字的地方都是空的，原因是因为 `puppeteer-docker` 基础镜像中，基本是无字体的，正儿八经的不会报错，但是基本结果都是空白的（或者都是长方形的格子），类似于下面这样
![字体无法显示的问题](https://pangji-home.com/eva_blogs/baidu_font.png)

因此我们需要为 `puppeteer-docker` 安装字体，因此需要手动打包一个基础镜像。下面这个 Dockerfile 是根据实际情况做出调整之后的，只能保证字体没有问题，不代表镜像没有安全漏洞（如果实际投入使用，建议联系平台部门做基础镜像）

```dockerfile
# A minimal Docker image with Node and Puppeteer
#
# Initially based upon:
# https://github.com/GoogleChrome/puppeteer/blob/master/docs/troubleshooting.md#running-puppeteer-in-docker

FROM buildkite/puppeteer:latest

LABEL maintainer="jdeseva<zjmqydes@163.com>"


# 设置 puppeteer 的字体 #fonts-droid fonts-arphic-ukai fonts-arphic-uming
COPY source-sans-pro.zip /tmp
RUN sed -i 's/deb.debian.org/mirrors.163.com/g' /etc/apt/sources.list && \
     apt update && \
     apt-get install libreoffice-writer -y && \
     apt-get install -y dpkg wget unzip && cd /tmp && wget http://ftp.cn.debian.org/debian/pool/main/f/fonts-noto-cjk/fonts-noto-cjk_20170601+repack1-3+deb10u1_all.deb && \
     dpkg -i fonts-noto-cjk_20170601+repack1-3+deb10u1_all.deb && \
     unzip source-sans-pro.zip && cd source-sans-pro  && mv ./OTF /usr/share/fonts/  && \
     fc-cache -f -v
```
字体安装完毕之后，即可使用该镜像作为服务基础镜像，这样即可保证结果达到预期效果

![最终效果](https://pangji-home.com/eva_blogs/baidu_success.png)

折腾了一大圈，可算是解决了，真的是，puppeteer，说声爱你不容易！想找你帮忙也不容易！！！

## 结束语

经过上面的沉淀，基本可以梳理出一个大致的服务模型以提供完整的 `puppeteer` 服务，尽管上面只提供了截图（PDF）服务，但是实际上背靠 `puppeteer` 我们还可以做非常多的功能，包括但不限于爬虫、自动化测试、UI自动走查等等，碍于本文篇幅有限，剩余内容不在阐述，最后，以一个流程设计模型结束本文。

![服务架构](https://pangji-home.com/eva_blogs/yuque_mind.jpeg)

> 思考一下，此种架构的缺陷和优化手段。可以很明确的指出，目前这种模式是有问题的。

### 致歉

由于最近有一些事情需要处理，耽搁了文章，都快赶不上一周一篇的进度，实在抱歉，后续会跟上进度，感谢大家的支持和理解。