[TOC]

# 史上最全vscode配置使用教程

[原文链接](https://zhuanlan.zhihu.com/p/113222681)

## 软件下载

直接在官网进行下载

[Visual Studio Code - Code Editing. Redefinedcode.visualstudio.com/![img](https://pic4.zhimg.com/v2-beaba009c542a9f6fe1d2034a7ed568b_180x120.jpg)](https://link.zhihu.com/?target=https%3A//code.visualstudio.com/)

首页

![img](https://pic1.zhimg.com/80/v2-0f049e5f50eb1175eea793f15fa95b08_1440w.jpg)

## vscode设置成中文

vscode默认的语言是英文，对于英文不好的小伙伴可能不太友好。简单几步教大家如何将vscode设置成中文。

1. 按快捷键“Ctrl+Shift+P”。
2. 在“vscode”顶部会出现一个搜索框。
3. 输入“configure language”，然后回车。
4. “vscode”里面就会打开一个语言配置文件。
5. 将“en-us”修改成“zh-cn”。
6. 按“Ctrl+S”保存设置。
7. 关闭“vscode”，再次打开就可以看到中文界面了。

当然如果你不愿意设置，也可以直接安装它的中文插件，还是很人性化的。

![img](https://pic4.zhimg.com/80/v2-ddbfa1f88bf57ffe6a2cee34a973243f_1440w.jpg)

## VScode用户设置

\1. 打开设置

文件--首选项--设置，打开用户设置。VScode支持选择配置，也支持编辑[setting.json](https://www.zhihu.com/search?q=setting.json&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})文件修改默认配置。个人更倾向于编写json的方式进行配置，下面会附上我个人的配置代码

这里解析几个常用配置项：

（1）[editor.fontsize](https://www.zhihu.com/search?q=editor.fontsize&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})用来设置字体大小，可以设置editor.fontsize : 14;

（2）files.autoSave这个属性是表示文件是否进行自动保存，推荐设置为onFocusChange——文件焦点变化时自动保存。

（3）editor.tabCompletion用来在出现[推荐值](https://www.zhihu.com/search?q=推荐值&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})时，按下Tab键是否自动填入最佳推荐值，推荐设置为on;

（4）editor.codeActionsOnSave中的source.organizeImports属性，这个属性能够在保存时，自动调整 import 语句相关顺序，能够让你的 import 语句按照字母顺序进行排列，推荐设置为true,即"editor.codeActionsOnSave": { "source.organizeImports": true }；

（5）editor.lineNumbers设置代码行号,即editor.lineNumbers ：true；

我的个人配置，供参考：

```json
{
  "files.associations": {
  "*.vue": "vue",
  "*.wpy": "vue",
  "*.wxml": "html",
  "*.wxss": "css"
  },
  "terminal.integrated.shell.windows": "C:\\Windows\\System32\\cmd.exe",
  "git.enableSmartCommit": true,
  "git.autofetch": true,
  "emmet.triggerExpansionOnTab": true,
  "emmet.showAbbreviationSuggestions": true,
  "emmet.showExpandedAbbreviation": "always",
  "emmet.includeLanguages": {
  "vue-html": "html",
  "vue": "html",
  "wpy": "html"
  },
  //主题颜色 
  //"workbench.colorTheme": "Monokai",
  "git.confirmSync": false,
  "explorer.confirmDelete": false,
  "editor.fontSize": 14,
  "window.zoomLevel": 1,
  "editor.wordWrap": "on",
  "editor.detectIndentation": false,
  // 重新设定tabsize
  "editor.tabSize": 2,
  //失去焦点后自动保存
  "files.autoSave": "onFocusChange",
  // #值设置为true时，每次保存的时候自动格式化；
  "editor.formatOnSave": false,
   //每120行就显示一条线
  "editor.rulers": [
  ],
  // 在使用搜索功能时，将这些文件夹/文件排除在外
  "search.exclude": {
      "**/node_modules": true,
      "**/bower_components": true,
      "**/target": true,
      "**/logs": true,
  }, 
  // 这些文件将不会显示在工作空间中
  "files.exclude": {
      "**/.git": true,
      "**/.svn": true,
      "**/.hg": true,
      "**/CVS": true,
      "**/.DS_Store": true,
      "**/*.js": {
          "when": "$(basename).ts" //ts编译后生成的js文件将不会显示在工作空中
      },
      "**/node_modules": true
  }, 
  // #让vue中的js按"prettier"格式进行格式化
  "vetur.format.defaultFormatter.html": "js-beautify-html",
  "vetur.format.defaultFormatter.js": "prettier",
  "vetur.format.defaultFormatterOptions": {
      "js-beautify-html": {
          // #vue组件中html代码格式化样式
          "wrap_attributes": "force-aligned", //也可以设置为“auto”，效果会不一样
          "wrap_line_length": 200,
          "end_with_newline": false,
          "semi": false,
          "singleQuote": true
      },
      "prettier": {
          "semi": false,
          "singleQuote": true
      }
  }
}
```

**最近经常有人微信问我，这个配置代码写在哪里？**

新版的vscode设置默认为UI的设置，而非之前的json设置。如果你想复制我上面这段代码进行配置，可以进行下面的修改

文件>首选项>设置 > 搜索workbench.settings.editor，选中json即可改成json设置；

**禁用自动更新**

文件 > 首选项 > 设置（macOS：代码 > 首选项 > 设置，搜索update mode并将设置更改为none。

**开启代码提示设置**

第一步：点击左下角点击设置图标，找到并点击“setting”

![img](https://pic3.zhimg.com/80/v2-ee37d59c1d7c4da0725fefb38b58abde_1440w.jpg)

第二步：到搜索框里搜索“prevent”--->并取消此项的勾选

![img](https://pic3.zhimg.com/80/v2-a9f44938ee7d576eac69b3ada0c4a94e_1440w.jpg)

## 常用的快捷键

高效的使用vscode,记住一些常用的快捷键是必不可少的，我给大家罗列了一些日常工作过程中用的多的快捷键。

以下以Windows为主，windows的 Ctrl，mac下换成Command就行了

对于 **行** 的操作：

- 重开一行：光标在行尾的话，回车即可；不在行尾，ctrl` + enter` 向下重开一行；ctrl+`shift + enter` 则是在上一行重开一行
- 删除一行：光标没有选择内容时，ctrl` + x` 剪切一行；ctrl +`shift + k` 直接删除一行
- 移动一行：`alt + ↑` 向上移动一行；`alt + ↓` 向下移动一行
- 复制一行：`shift + alt + ↓` 向下复制一行；`shift + alt + ↑` 向上复制一行
- ctrl + z 回退

对于 **词** 的操作：

- 选中一个词：ctrl` + d`

搜索或者替换：

- ctrl` + f` ：搜索
- ctrl` + alt + f`： 替换
- ctrl` + shift + f`：在项目内搜索

通过**Ctrl + `** 可以打开或关闭终端

Ctrl+P 快速打开最近打开的文件

Ctrl+Shift+N 打开新的编辑器窗口

Ctrl+Shift+W 关闭编辑器

Home 光标跳转到行头

End 光标跳转到行尾

Ctrl + Home 跳转到页头

Ctrl + End 跳转到页尾

Ctrl + Shift + [ 折叠区域代码

Ctrl + Shift + ] 展开区域代码

Ctrl + / 添加关闭行注释

Shift + Alt +A 块区域注释

## 插件安装

在输入框中输入想要安装的插件名称，点击安装即可。安装后没有效果，可以重启vscode

![img](https://pic1.zhimg.com/80/v2-4e57c8d10598038ba80642446e7ea698_1440w.jpg)

**必备插件**

**1、View In Browser**

在浏览器里预览网页必备。运行html文件

![img](https://pic2.zhimg.com/v2-69b3dfead6052e0702f5970ae9f232d9_b.jpg)

### 2、[vscode-icons](https://www.zhihu.com/search?q=vscode-icons&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})

改变编辑器里面的文件图标

### 3、Bracket Pair Colorizer

给嵌套的各种括号加上不同的颜色。

### 4、Auto Rename Tag

自动修改匹配的 HTML 标签。

![img](https://pic1.zhimg.com/v2-304d2802620974eeb369a6302f50cfc0_b.jpg)

### 5、Path Intellisense

**智能路径提示**，可以在你输入文件路径时智能提示。

![img](https://pic1.zhimg.com/v2-304d2802620974eeb369a6302f50cfc0_b.jpg)

### 6、Markdown Preview

实时预览 markdown。

### 7、stylelint

CSS / SCSS / Less 语法检查

### 8、Import Cost

引入包大小计算,对于项目打包后体积掌握很有帮助

![img](https://pic3.zhimg.com/v2-2db7c8e03f575e26584b54a5c2ccace6_b.jpg)

### 9、Prettier

比Beautify更好用的代码格式化插件

## Vue插件

### vetur

语法高亮、[智能感知](https://www.zhihu.com/search?q=智能感知&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})、Emmet等

![img](https://pic3.zhimg.com/80/v2-8abdcb8f10bcd6009ffcad29ebdfe576_1440w.jpg)

### VueHelper

snippet代码片段

![img](https://pic2.zhimg.com/v2-e7396a249f4ad2b6c68e94a37cb49ff5_b.jpg)

## 其它插件

**1、CSScomb**

CSS 书写顺序规则，这里我推荐腾讯 AollyTeam 团队的规范：

[http://alloyteam.github.io/CodeGuide/#css-declaration-orderalloyteam.github.io/CodeGuide/#css-declaration-order](https://link.zhihu.com/?target=http%3A//alloyteam.github.io/CodeGuide/%23css-declaration-order)

简单说下这个插件怎么用：

在项目的根目录下创建一个名为csscomb.json的文件，然后添加一些配置项。也可以将配置项写入项目的 package.json 文件中的 csscombConfig 字段。

至于添加的配置项，CSScomb 提供了示例配置文件：

[https://github.com/csscomb/csscomb.js/blob/master/config/csscomb.jsongithub.com/csscomb/csscomb.js/blob/master/config/csscomb.json](https://link.zhihu.com/?target=https%3A//github.com/csscomb/csscomb.js/blob/master/config/csscomb.json)

其中的 sort-order 就是 CSS 属性书写顺序，可以按照自己遵循的规范设置，所以我直接替换成了腾讯的。

这个配置文件里面各个字段的作用可以戳这里查看：

[csscomb/csscomb.jsgithub.com/csscomb/csscomb.js/blob/master/doc/options.md](https://link.zhihu.com/?target=https%3A//github.com/csscomb/csscomb.js/blob/master/doc/options.md)

**2、Turbo Console Log**

快捷添加 [console.log](https://www.zhihu.com/search?q=console.log&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})，一键 注释 / 启用 / 删除 所有 console.log。这也是我最常用的一个插件

![img](https://pic4.zhimg.com/v2-b14aec7821adc57a3f41ac746a5689a7_b.jpg)

简单说下这个插件要用到的[快捷键](https://www.zhihu.com/search?q=快捷键&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681}):

```text
ctrl + alt + l 选中变量之后，使用这个快捷键生成 console.log
alt + shift + c 注释所有 console.log
alt + shift + u 启用所有 console.log
alt + shift + d 删除所有 console.log
```

**3、GitLens**

详细的 Git 提交日志。

Git 重度使用者必备，尤其是多人协作时：哪一行代码，何时、何人提交都有记录。

妈妈再也不用担心我背锅了！

![img](https://pic3.zhimg.com/80/v2-5e020466c135639aa33ba9a26e8ed2fa_1440w.png)

### 4、css-auto-prefix

**自动添加 CSS 私有前缀**。

![img](https://pic1.zhimg.com/v2-01d42a4fb0039acb17ff5a7b708d1030_b.jpg)

### 5、change-case

**转换命名风格**。

![img](https://pic3.zhimg.com/v2-1085adfd3dc66da8ed6b40c5961a786e_b.jpg)

**6、CSS Peek**

**定位 CSS 类名**。

![img](https://pic2.zhimg.com/v2-247271ae745aaffee6497624e335b7b9_b.jpg)

**7、vscode-json**

处理 JSON 文件，用法看图：

![img](https://pic2.zhimg.com/v2-5a6867c294dafe411cc58a43de69875d_b.jpg)

### 8、Regex Previewer

**实时预览[正则表达式](https://www.zhihu.com/search?q=正则表达式&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})的效果**。

![img](https://pic3.zhimg.com/v2-ff1f00be77163d4e24b146211c618212_b.jpg)



## 设置同步

花了一天终于把vscode配置成自己满意的样子，如果每换一次电脑就要重新来一次，大家一定会手撕了我。放心，早就帮大家准备好了。Settings Sync，在不同电脑间同步你的插件。

首先要想在不同的设备间同步你的插件, 需要用到 Token 和Gist id

Token 就是你把插件上传到 github 上时, 让你保存的那段字符，Gist id 在你上传插件的那台电脑上保存着。

先给大家来三个快捷键，后面会用到

```text
1、CTRL+SHIFT+P 我也不知道叫什么，暂且就叫它功能搜索功能吧
2、ALT+SHIFT+D 下载配置
3、ALT+SHIFT+U 上传配置
```

现在手把手教大家配置：

1、安装Settings Sync
2、登陆Github>settings>Developer settings>personal access tokens>generate new token，输入名称，勾选Gist，提交

![img](https://pic3.zhimg.com/80/v2-9898e9d1d90e227fdb6da88a829b01d2_1440w.jpg)

3、保存Github Access Token
4、打开vscode，Ctrl+Shift+P打开命令框-->输入sync-->选择高级设置-->编辑本地扩展设置-->编辑token

5、Ctrl+Shift+P打开命令框-->输入sync-->找到update/upload settings，上传成功后会返回Gist ID，保存此Gist ID.

![img](https://pic2.zhimg.com/v2-76ae0fd6b07c5c7949fc3a6e02dff2c1_b.jpg)

6、在 VSCode 里，依次打开: 文件 -> 首选项 -> 设置，然后输入 Sync 进行搜索:能找到你gist id

![img](https://pic3.zhimg.com/80/v2-86515bfd4e5aada27c7f998dcdaf1dfa_1440w.jpg)

7、若需在其他机器上DownLoad插件的话，同样，Ctrl+Shift+P打开命令框，输入sync，找到Download settings，会跳转到Github的Token编辑界面，点Edit，regenerate token，保存新生成的token，在vscode命令框中输入此Token，回车，再输入之前的Gist ID，即可同步插件和设置

## 开启一个本地服务

**第一种方式**

1.安装Debugger for Chrome插件

![img](https://pic1.zhimg.com/80/v2-80a0108d05009387f257d9b70226c3c4_1440w.jpg)

2.使用ctrl+`快捷键打开终端，然后输入npm install -g live-server

3.在命令行里输入 live-server即可

**第二种方式**

在写前端页面中，经常会在浏览器运行HTML页面，从本地文件夹中直接打开的一般都是file协议，当代码中存在http或https的链接时，HTML页面就无法正常打开，为了解决这种情况，需要在在本地开启一个本地的服务器。 本文是利用[node.js](https://www.zhihu.com/search?q=node.js&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})中的http-server，开启本地服务，步骤如下：

*1.安装[http-server](https://www.zhihu.com/search?q=http-server&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A113222681})*

在终端输入： $ npm install http-server -g

*2.开启 http-server服务*

终端进入目标文件夹，然后在终端输入：

```text
$ http-server -c-1   （⚠️只输入http-server的话，更新了代码后，页面不会同步更新）
Starting up http-server, serving ./
Available on:
  http://127.0.0.1:8080
  http://192.168.8.196:8080
Hit CTRL-C to stop the server
```

*3.关闭 http-server服务*

按快捷键CTRL-C 终端显示^Chttp-server stopped.即关闭服务成功。