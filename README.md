# Cordova Hot Upgrade by Plugin
目前基于Cordova的App热更新一直是一个困扰许多开发者的难题之一。而且目前几套解决方案都不是很好。我在新单位正好负责调查这个问题，于是自己写了一套解决方案。测试通过。

## 目前Cordova的App的几种热更新方式
- 加载远端站点
  - 优点：此方法非常给力，可以直接更新服务器端完成整个App的更新。
  - 缺点：由于每次请求都要加载服务器，App不能离线运行，且加载页面受网速影响，会影响用户体验。且部署的时候需要对Android和iOS单独部署才能满足一些JS-Module对外API接口不一致的Plugin才能正常工作。
- 修改Cordova源代码，加载指定路径的Html
  - 优点：此方法解决了第一种方案的难题，可以本地存储，离线运行，界面切换不再受网络限制。
  - 缺点：Cordova源代码修改后内核升级是一个麻烦。却修改内核影响面比较大，在技术能力不足的时候很容易导致更多问题。
- Microsoft/code-push
  - 优点：可以实现部分更新和回滚，满足了我们对热更新的绝大多数需求
  - 缺点：需要依赖微软的云服务，且需要上传更新部分的源代码
- nordnet/cordova-hot-code-push
  - 优点：同code-push
  - 缺点：同code-push
- cordova-plugin-hotpushes
  - 优点：这个插件基本满足了我们对热更新的需求，需要依赖phonegap-plugin-contentsync插件。
  - 缺点：目前只实现了Replace功能。没有服务器端

## Solution of CordovaHotUpgrade
其实以上几个方案已经能够满足绝大多数对于热更新的需求，但是如果受公司策略影响不能使用3，4的话，就是很麻烦的问题。而1，2的方案虽然可以解决我们的问题，但是同样缺点也是我们不能接受的。<br>

对于第五个方案，其实他的思路已经有很多公司使用了，而且很多公司也按照这个思路进行了自己的开发。我也是参照这个方案自己进行了开发。<br>

该思路主要使用plugin的方案，这样不需要修改Cordova源代码，不影响客户端App升级与维护。非常方便。不需要使用的时候直接卸载Plugin即可。

## Server端设计
对于Server端，无论我们采用什么技术Java，Net，Nodejs，Php。其实无所谓。但是我们需要满足以下几个API。
- 后台控制端：
  - 登陆 ： 用户登录功能，要保证后台管理系统的安全性。当然，如果是部署在公司防火墙内部这一点可以忽略。
  - 修改密码：这也是为安全性考虑
  - 发布新版本：每一次发布新版本我们都需要至少保存以下信息
    ```json
    {
      'version':'1.0.0',
      'package':'www.zip',
      'path':'xxx/upload/xxxxxxxx_www.zip',
      'operator':'ryouaki(46517115@qq.com)',
      'date':'2016-01-01 01:01:01',
      'isReplace':true
      ...
    }
    ```
  - 设置当前版本：我们可以提供一个界面灵活的指定一个我们发布过的版本，提供给客户下载。这样我们可以实现任意版本的回退功能。
  - 生成变更文件列表：解压上传的zip文件，生成一个变更文件列表的json文件，这样客户端就不需要遍历目录了。减少客户端性能压力。直接根据这个json文件拷贝替换对应文件。
- 后台API
  - 取得当前版本 ：用于向客户端提供当前指定的客户端版本。如果需要实现部分更新最好提供文件变更列表
  ```json
  {
    'version':'1.0.0',
    'change':[
      {
        'src':'压缩包内相对路径',
        'dest':'www目录的位置'
      },
      ...
    ]
  }
  ```
  - 下载服务 ：向客户端提供下载package功能。package是一个zip文件，包含更新的文件。

## 客户端plugin设计
- 客户端主要使用Cordova的Plugin的方式。我们需要定义一个JS-Module的API来统一接口__hotUpgradePlugin.check__，通过这个API我们将服务器端设置的版本信息以及变更文件列表传入Plugin的原生代码。
```js
  $.ajax({
    url:'http://localhost:3000/api/getCurrentVersion',
    type:'GET',
    success:function(result){
      hotUpgradePlugin.check(result);
    },error:function(error){
      console.log(error);
    }
  });
```

- Plugin原生代码实现，很遗憾，我们无法修改打包后的www的内容。而且Cordova的核心会主动加载打包后的www目录的内容(android:assets,iOS:/www)。为了能够使用我们自己修改后的代码，我们需要在我们的Plugin中实现这些东西：
  - 由于我们的热更新插件在App启动后第一个运行，所以我们需要做的第一件事就是检查App的内部存储目录是否存在www目录，如果没有，那么就从package/www目录将所有内容拷贝到
