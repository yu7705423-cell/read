# 喵喵书阁(meow-reader.html)项目交接文档

## 这是什么

一个纯前端、单文件 HTML 的"伪阅读器"应用,用来存放 AIRP(AI角色扮演)过程中 user 和 char 之间产生的番外剧情、长篇小说,并附带一些配套工具(搜索、文件解析导入、语音朗读、AI总结前文)。

- **无后端**,所有数据存在浏览器的 **IndexedDB** 里(数据库名 `meowReaderDB`,四个 object store:`characters`、`books`、`materials`、`presets`)
- **无构建流程**,纯 vanilla JS + 原生 HTML/CSS,一个 `.html` 文件打开就能用
- 引用了三个 CDN 库(写死在 `<head>` 里):
  - `JSZip 3.10.1` — 解压/生成 zip
  - `mammoth 1.6.0`(browser 版)— 解析 docx 提取文本
  - `pako 2.1.0` — 解压 gzip(给 tar.gz 用)
- 当前文件体积约 115KB,全部逻辑都在一个 `<script>` 标签里

最新文件在 `/mnt/user-data/outputs/meow-reader.html`(如果继续用 Claude 对话,直接说"看一下这个文件"就能定位到)。

---

## 原始需求 vs 完成情况

用户最初的需求(按对话原文整理),逐条标注完成度:

| 需求 | 状态 |
|---|---|
| 简约整体风格 | ✅ 已实现,自定义配色变量,非模板化视觉 |
| 四个Tab:书架/搜索/工具/设置 | ✅ |
| 内置明暗主题 + 自定义CSS导入 | ✅ 设置页里,粘贴CSS或传.css文件 |
| 书架加号:新建书籍 / 导入章节 | ✅ |
| 新建书籍:配置char/user姓名(留存)、封面、番外页/长篇选择、自定义标签、自定义分类 | ✅ |
| 支持多个char和多个user(不止一对一) | ✅ participants数组,每个含characterId+role |
| 已有char/user可复用,也可新建(放在"作者"概念下) | ✅ 角色库独立于书籍,可跨书复用 |
| 番外页 = 多条独立剧情;长篇 = 背景+多章节 | ✅ |
| 番外结构应为:番外要求(生成指令/主题) → 标题 → 正文 | ✅ 后期修正加上了`prompt`字段 |
| 番外/长篇中可能嵌入HTML/CSS,要能正常显示和点击 | ✅ sandboxed iframe渲染,支持整段是HTML或大段文字中夹小HTML块 |
| 搜索:关键字/章节名/角色名/番外剧情 + 分类(自定义/角色/番外剧情/标签) | ✅ |
| 工具:导入txt/解析zip/rar,一键复制 | ⚠️ txt/docx/zip/tar/tar.gz 都做了,**RAR 没做**(见下方说明) |
| ZIP要能解到嵌套最深层 | ✅ 递归解包,zip里套zip无限层都会挖到底 |
| 素材库要单独分类,不进书架体系 | ✅ 完全独立的 `materials` store |
| 素材库分文件夹 | ✅ 导入时选/建文件夹,单条素材可改归属文件夹 |
| 素材库支持多选批量复制/导出 | ✅ 多选模式,复制拼接文本 / 导出打包zip |
| 素材库支持直接手写文字新建 | ✅ 右上角"+"按钮 |
| 每本书自定义背景/翻页效果/注入CSS/自定义字体URL | ✅ |
| 翻页效果可选:竖屏滑动 或 左右翻页 | ⚠️ 已实现,但翻页模式有个bug**修复后未经用户确认**(见"已知问题") |
| 语音朗读配置(自填接口)、保存预设、绑定角色 | ⚠️ 预设的增删改**已完成**,但"点朗读实际调API出声音"**还没接线** |
| 一键复制(背景+前文)方便去续写 | ✅ 支持选择是否带上番外要求/背景 |
| 配置API总结前文(可自定义prompt) | ⚠️ 预设增删改**已完成**,"点了实际生成总结"**还没接线** |
| 朗读也可注入语音prompt让角色读更好 | ⚠️ 预设里有`promptInjection`字段,同上,还没接到实际调用 |
| 整体设置可导出导入,便于分享 | ✅ 一键导出/导入全部数据的JSON |
| 全站不用emoji,改用SVG图标 | ✅ 用户中途明确要求后已经全部替换,已存入用户记忆偏好 |

---

## 数据结构

### `characters`(角色库,可被任意书复用)
```js
{ id, name, roleDefault: 'char'|'user', avatar, note, createdAt }
```

### `books`(书架条目)
```js
{
  id, title, cover, type: 'extra'|'novel',
  tags: [], category: '',
  participants: [{ characterId, name, avatar, role: 'char'|'user' }], // 支持多个
  createdAt, updatedAt,

  // type==='extra' 时:
  episodes: [{ id, title, prompt, content, createdAt }],
  // prompt = 番外要求/生成指令,content = 正文

  // type==='novel' 时:
  background: '',
  chapters: [{ id, title, content, order, createdAt }],

  readerSettings: {
    background: '',       // CSS背景值,或 `url(dataURL) center/cover no-repeat`
    pageEffect: 'scroll'|'paginate',
    customCss: '',
    fontUrl: '',           // 例如 Google Fonts 的 css2 链接
    voicePresetId: '',     // 关联到 presets 里 kind:'voice' 的记录
    summaryPresetId: ''    // 关联到 presets 里 kind:'summary' 的记录
  }
}
```

### `materials`(素材库,独立于书架)
```js
{ id, name, path, folder, content, importedAt }
// path 记录来源路径,比如嵌套解压时会是 "外层.zip / 内层.zip / 文件名.txt"
```

### `presets`(语音预设 + 总结预设,同一个 store,用 kind 区分)
```js
// 语音预设
{ id, kind:'voice', name, characterId, endpoint, apiKey, voiceId, extraParams(JSON字符串), promptInjection, createdAt }

// 总结预设
{ id, kind:'summary', name, endpoint, apiKey, model, promptTemplate, createdAt }
```

---

## 各模块实现要点

### 阅读器(第3块)
- 全屏 `.reader-overlay`,正文用 **sandboxed iframe**(`sandbox="allow-scripts"`)渲染,通过 `srcdoc` 注入生成的 HTML 文档,这样自定义CSS/字体/嵌入HTML都不会污染外层App样式,也能保证番外里嵌的HTML小组件正常显示和交互。
- 内容渲染逻辑(`renderContentHtml`):按空行分段落包成 `<p>`,但如果某一段本身以 `<` 开头(判断是完整HTML块),原样保留不做转义包裹——这样"大段文字里插一个小HTML卡片"和"整条内容就是一段HTML"两种情况都能正常显示。
- **工具栏可折叠**:顶部工具栏默认展开,点击顶栏的收起箭头、或点击正文空白处(iframe内部通过 `postMessage` 把点击事件传回父页面,排除了链接/按钮等交互元素的点击)都会让工具栏滑走,只留一个小箭头挂件在顶部中间,点一下再唤出来。翻页模式下的页码导航条同理跟随折叠。
- **一键复制**:番外可选是否带上"番外要求";长篇默认是"背景+从第一章到当前章"的正文,也可选是否带背景。用 `navigator.clipboard.writeText`,失败时降级用隐藏 textarea + `execCommand('copy')`。

### 关于 `<script>` 标签转义的坑(重要,继续改代码时必须注意)
`buildReaderDoc()` 函数会拼一段**字符串**作为 iframe 的 `srcdoc`,这段字符串里必然要包含 `<script>...</script>`。但因为这整个函数本身是写在外层 `.html` 文件的 `<script>` 标签**内部**的 JS 源码,如果字符串字面量里直接出现 `</script>`(哪怕只是JS字符串里的普通字符),**浏览器的HTML解析器会在扫描到这个字符序列时提前结束外层的 `<script>` 标签**,导致整个应用崩掉。

解决方法:在源码里凡是要拼出 `</script>` 的地方,必须写成 `<\/script>`(反斜杠转义),JS 求值后会变成正常的 `</script>` 字符串,但 HTML 解析阶段看到的是 `<`, `\`, `/` 这样不连续的字符,不会被误判为闭合标签。**以后改这部分代码,新增任何内嵌 `<script>` 都要用这个写法**,可以用这两条命令自查:
```bash
grep -c "</script>" meow-reader.html      # 应该恰好是 1(外层真正的闭合标签)
grep -c "<\\\\/script>" meow-reader.html   # 应该等于文件里内嵌script的数量
```

### 翻页模式(⚠️ 已知问题,未确认修复)
最早用的是 CSS `column-width` + `column-fill:auto` 让浏览器"高度不够时自动往右边生成新分栏"的特性做分页,但这个特性在用户的实际浏览器上表现不稳定,反复出现"只有第一页有内容,后面全空白"。

最后一次的修复方案(**代码已经写上,但用户还没回来确认是否真的解决了**):放弃依赖浏览器的隐式分栏行为,改成 JS 精确测量:
1. 先让内容以单栏、`height:auto` 的方式正常撑开,量出 `scrollHeight`
2. 用 `Math.ceil(总高度 / 屏幕高度)` 算出精确需要几页
3. 手动把容器宽度设成 `页数 × 屏幕宽度`(像素精确值,不用 `vw` 单位避免可能的取整误差),再显式设置 `column-width` 等于屏幕宽度、`column-fill:auto`
4. 用 `translateX(-页码 × 屏幕宽度)` 做左右翻页动画
5. 支持点击左右1/3区域翻页,也支持真正的触摸滑动手势(记录 touchstart/touchmove/touchend 判断滑动方向和距离,和点击翻页互不冲突)

**建议接手后第一件事就是测试这个翻页模式**,如果还有问题,大概率还是分栏宽度和 `window.innerWidth` 计算不一致导致的错位,或者是内容里有超宽元素(比如没设 `max-width:100%` 的图片/HTML块)撑破了单页宽度导致测量的总高度不准。

代码位置:`buildReaderDoc()` 函数里 `isPaginate` 分支,以及生成的 iframe 内嵌 `<script>` 里的 `layout()` / `goTo()` 函数。

---

## 还没做的部分(第7、8块)

### 第7块:语音朗读接入
现状:设置页已经能增删改"语音预设"(名称/绑定角色/接口地址/API Key/音色ID/额外JSON参数/朗读语气提示词),阅读设置里也有个 `voicePresetId` 字段可以关联到某本书,但**点击"朗读"实际调用API出声音这一步完全没写**。

因为用户自己的API和minimax这种服务的请求/响应格式各不相同,建议实现思路(之前讨论过但没来得及写):
- 在阅读页"更多"菜单加一个"朗读正文"选项
- 取 `book.readerSettings.voicePresetId` 对应的预设,如果没设置就提示先去阅读设置里选
- 正文要先去掉HTML标签转成纯文本
- 请求体建议做成**可自定义模板**而不是写死字段名(因为不同API的字段名不一样):给语音预设加一个"请求体模板"字段,允许用户写JSON模板并用 `{{text}}` `{{voice}}` `{{prompt}}` 占位符,发请求前做字符串替换再 `JSON.parse`;如果用户没填模板,就用默认结构 `{text, voice_id, prompt, ...额外参数}`
- 响应解析同理建议加一个"响应音频字段路径"配置项(比如 `data.audio_url`),因为返回格式也是五花八门(可能是二进制音频、JSON里的URL、或base64)。做不到通用兜底的话,起码在解析失败时把原始响应内容展示出来方便用户自己调整配置。
- 播放用一个 `<audio>` 元素 + 简单的悬浮播放条(播放/暂停/停止)

### 第8块:API总结前文接入
现状:设置页已经能增删改"总结预设"(名称/接口地址/API Key/模型名/总结用的prompt模板),但**点击"生成总结"实际调用API这一步没写**。

建议实现思路:
- 只对"长篇"类型的书生效,在阅读页"更多"菜单加"生成/更新前文总结"选项
- 取 `book.readerSettings.summaryPresetId` 对应预设,没设置则提示去阅读设置里选
- 组装要总结的内容:背景 + 从第一章到当前章的正文(可以复用已有的 `composeCopyText` 逻辑)
- 用类似 OpenAI Chat Completions 的请求格式比较通用:
  ```js
  { model: preset.model, messages: [
      { role:'system', content: preset.promptTemplate || 默认总结指令 },
      { role:'user', content: 组装好的正文 }
  ]}
  ```
- 响应优先尝试解析 `data.choices[0].message.content`(OpenAI格式),失败则尝试其他常见字段,都不行就把原始响应展示出来
- 把生成的总结存在 `book.summary = { text, updatedAt, forChapterId }`
- 在"一键复制"弹窗里(`openCopySheet`)给长篇书加一个开关:"用已生成的总结代替完整前文",勾选后 `composeCopyText` 改用总结文本而不是全部正文,这样长篇写到几十章的时候复制出来的内容不会爆炸

---

## 其他注意事项

- **RAR 完全没做**:纯前端没有可靠的RAR解析方案(专有格式,只有实验性WASM移植,RAR5/固实压缩/分卷经常直接失败),已经跟用户确认过放弃。TAR/TAR.GZ 因为格式简单、纯JS就能解析,所以顺手做了。
- 所有输入框字号强制设为 `16px`,是为了避免 iOS Safari 在输入框字号小于16px时聚焦自动放大页面的问题(之前真实踩过这个坑,表现为"主页面一直在放大缩小拉伸")。
- 弹窗系统 `.modal-overlay` 的 `z-index` 必须高于阅读页 `.reader-overlay`,否则从阅读页内打开的"更多菜单/阅读设置/复制"弹窗会被阅读页盖住(之前的真实bug,现在 modal-overlay 是 90,reader-overlay 是 70)。
- FAB(书架加号按钮)的底部间距要用 `calc(78px + env(safe-area-inset-bottom))`,不能写死数字,否则在有Home Indicator的手机上会被底部导航栏遮住。

祝接手顺利,喵喵书阁就交给你了 TvT
