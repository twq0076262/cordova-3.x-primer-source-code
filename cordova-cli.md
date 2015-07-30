# Cordova 3.x 源码分析（1） -- Cordova CLI

**(1)Node.js 的使用** 

Cordova CLI 基于 node.js，所以有必要知道 nodejs 最基本的知识。 

Js **代码**

```
// define：1个module1个js文件
exports.printFoo = function(){ return "foo" }
// import
var foo = require('./foo.js');
// call
console.log(foo.printFoo());

```

**引用**

```
node main.js
```

**（2）2个重要的路径** 

- C:\Documents and Settings\RenSanNing\Application Data\npm\node_modules\cordova
- C:\Documents and Settings\RenSanNing\.cordova\lib (以下简称 LIB_ROOT)

最新源码可以从 Github 下载，目前稳定版是3.4。   
[https://github.com/apache/cordova-cli](https://github.com/apache/cordova-cli)   
[https://github.com/apache/cordova-android](https://github.com/apache/cordova-android) 

**(3)输入命令到执行完的过程**

安装完 Nodejs 后，npm 的路径就被放到了环境变量 PATH 中。 

**引用**
	
	C:\Documents and Settings\RenSanNing\Application Data\npm

以下简称 NPM_ROOT，安装完 Cordova 后，在这个文件夹下有： 

- cordova.cmd（windows batch）
- cordova（linux shell）

所以在输入 cordova cli 命令时，入口就是这两个文件，以下以 cordova.cmd 为例说明。 

a） 输入命令（根据不同的命令处理不同，这里以添加平台支持为例） 

**引用**

```
cordova platform add android
```

b） 执行<NPM_ROOT>/cordova.cmd 

**引用**

```
node "%~dp0\node_modules\cordova\bin\cordova" %*
```

其中%~dp0代表的是 batch 文件所在路径，比如执行 C:\bat_files\example.bat，那么%~dp0 就是 C:\bat_files\。这里就是指 NPM_ROOT。类似 shell 的$0就是 Windows 下 batch 文件中获取参数的一种方式，可以在命令行窗口执行“for /?”可以查看详细说明。启动 node 执行 cordova(nodejs)文件，并把所有参数传给它。(%*：输入的所有参数) 

c） 执行 cordova(nodejs) 

路径：<NPM_ROOT>\node_modules\cordova\bin   
nodejs文件：cordova   
addTs()函数是打印执行时间，默认未开启，所以只要代码是：   

Js **代码**

```
var CLI = require('../src/cli');
new CLI(process.argv);
```

****cordova.cmd 作用和 npm 文件夹下的 cordova.cmd 一样，%~dpn0 代表带路径的 batch 文件名。 

d) 进入<NPM_ROOT>\node_modules\cordova\src\cli.js 
   d.1)// 导入 node_modules\cordova\cordova.js 
       var cordova = require('../cordova');   
       ->  // 导入node_modules\cordova\src\util.js   
       ->  var cordova_util = require('./src/util');   
       ->  // 通过 addModuleProperty()方法加载不同命令的（比如这里的 platform）代码   
       ->  addModuleProperty(module, 'platform', './src/platform', true);   
   d.2)对输入参数进行解析提取   
   d.3)根据不同命令执行 cordova.raw[cmd].call();   
        比如调用 platform()方法。   

e) 进入<NPM_ROOT>\node_modules\cordova\src\platform.js 

Js **代码**

```
function platform(command, targets) {
  // 验证当前文件夹是否是Cordova-based project
  var projectRoot = cordova_util.cdProjectRoot();
  // 获取Hooks文件
  var hooks = new hooker(projectRoot);
  // ...
  // 调用相应的方法（比如：add）******
  add(hooks, projectRoot, targets, opts)
}
```

Js **代码**

```
function add(hooks, projectRoot, targets, opts) {
  // 读入config.xml
  var xml = cordova_util.projectConfig(projectRoot);
  var cfg = new ConfigParser(xml);

  // 执行before_platform_add的Hook
  hooks.fire('before_platform_add', opts)

  // 获取libDir的目录即： <LIB_ROOT>\android\cordova\3.4.0
  lazy_load.based_on_config(projectRoot, t)
  // 调用Android SDK******
  call_into_create(t, projectRoot, cfg, libDir, template, copts)

  // 执行after_platform_add的Hook
  hooks.fire('after_platform_add', opts);
}
```

Js **代码**

```
function call_into_create(target, projectRoot, cfg, libDir, template_dir, opts) {
  // ...

  // 检查平台依赖***0***
  module.exports.supports(projectRoot, target)

  var bin = path.join(libDir, 'bin', 'create');

  // 调用bat创建project***1***
  superspawn.spawn(bin, args, opts || { stdio: 'inherit' })

  // 调用prepare
  require('../cordova').raw.prepare(target);

  // 把merges文件夹下的文件覆盖过去
  createOverrides(projectRoot, target);

  // 通过plugman安装plugins下的所有插件***2***
  plugman.raw.install(target, output, path.basename(plugin), plugins_dir);
}
```

***0*** 

Js **代码**

```
function supports(project_root, name) {、
  // 平台配置解析文件 <NPM_ROOT>\node_modules\cordova\platforms.js
  // 具体Android在：src\metadata\android_parser
  var platforms = require('../platforms');
  var platformParser = platforms[name].parser;

  // 检查平台依赖lib是否存在
  platformParser.check_requirements(project_root);
}
```

***1*** 

进入 <LIB_ROOT>\android\cordova\3.4.0\bin   
create.bat   

**引用**

```
SET script_path="%~dp0create" 
node %script_path% %*
```

create(nodejs) 

Js **代码**

```
var create = require('./lib/create');
create.createProject(args._[0], args._[1], args._[2], args._[3], args['--shared'], args['--cli']).done();
```

进入 <LIB_ROOT>\android\cordova\3.4.0\bin\lib   
create.js   
说是创建其实大部分都是从 libdir 拷贝过来的，执行了一下“android update project”。 

**引用**

	C:\Documents and Settings\RenSanNing\.cordova\lib\android\cordova\3.4.0

Js **代码**

```
exports.createProject = function(project_path, package_name, project_name, project_template_dir, use_shared_project, use_cli_template) {
  // ...

  // 检测Ant（ant -version），Java（java -version），Android（android list targets）
  check_reqs.run();

  // 前边有很多Copy文件的准备工作，其中最重要的cordova.js就是从以下路径Copy过来的。
  // <LIB_ROOT>\android\cordova\3.4.0\framework\assets\www\cordova.js

  // 这里就是创建Android工程的具体实现
  // cmd：android update project --subprojects --path "platforms\android" --target android-19 --library "CordovaLib"
  // CordovaLib工程也是从<LIB_ROOT>\android\cordova\3.4.0\frameworkCopy过去的
  // target_api取得是<LIB_ROOT>\android\cordova\3.4.0\framework\project.properties的target=android-19
  runAndroidUpdate(project_path, target_api, use_shared_project);
}
```

***2*** 

进入 <NPM_ROOT>\node_modules\cordova\node_modules\plugman   
plugman.js   

进入 <NPM_ROOT>\node_modules\cordova\node_modules\plugman\src   
install.js   

Js **代码**

```
function installPlugin(platform, project_dir, id, plugins_dir, options) {
  //...
  // 这里就是解析plugin.xml后安装plugin的具体实现
  runInstall(current_stack, platform, project_dir, plugin_dir, plugins_dir, options)
  //...
}
```

Js **代码**

```
function runInstall(actions, platform, project_dir, plugin_dir, plugins_dir, options) {
  //...
  // Copy文件
  copyPlugin();
  handleInstall();
  //...
}
```

Js **代码**

```
function handleInstall(actions, plugin_id, plugin_et, platform, project_dir, plugins_dir, plugin_dir, filtered_variables, www_dir, is_top_level) {
  //...
  plugman.prepare(project_dir, platform, plugins_dir, www_dir);
  //...
}
```

进入 <NPM_ROOT>\node_modules\cordova\node_modules\plugman\src   
prepare.js   

Js **代码**

```
scriptContent = 'cordova.define("' + moduleName + '", function(require, exports, module) { ' + scriptContent + '\n});\n';
```

从 plugins 往\platforms\android\assets\www\plugins 下 Copy 插件 JS 代码的时候，添加了模块的定义，所以最终执行的插件的 JS 和安装的 JS 是不一样的。 

Js **代码**

```
// 生成cordova_plugins.js
var final_contents = "cordova.define('cordova/plugin_list', function(require, exports, module) {\n";
final_contents += 'module.exports = ' + JSON.stringify(moduleObjects,null,'    ') + ';\n';
final_contents += 'module.exports.metadata = \n';
final_contents += '// TOP OF METADATA\n';
final_contents += JSON.stringify(pluginMetadata, null, '    ') + '\n';
final_contents += '// BOTTOM OF METADATA\n';
final_contents += '});';
```

以上过程只是主要的处理流程，至此 Android 项目创建成功，并且以下两个 Cordova 核心的 js 也放置到了相应的位置。 

- platforms\android\assets\www\cordova.js
- platforms\android\assets\www\cordova_plugins.js


其他的命令各自有各自的作用，所以处理内容不同。特别要说的是执行和 project 相关的命令时，最终会调用到各个平台工程下的脚本，比如：platforms\android\cordova。放在 project 下的目的除过各个平台的脚本不一样以外，也使该工程更独立，只要有 Nodejs 环境即可编译运行。 

- prepare
- compile(platforms\android\cordova\build)
- build（prepare->compile）
- run(prepare->platforms\android\cordova\run）
- emulate(prepare->platforms\android\cordova\run） 比run多了个参数“--emulator”

参考：   
[http://blog.csdn.net/mociml/article/category/1409992](http://blog.csdn.net/mociml/article/category/1409992)
