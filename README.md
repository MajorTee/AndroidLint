# Android Lint

## 简介

这是一个通用Android Lint库，你可以用它来检查代码规范、bug、资源命名等。

本库最大特点是通用，相较于其他Lint库（规则直接在代码写死），最大的不同是，规则全靠配置生成，更加灵活，在多团队协作下，只需拷贝配置文件，改下提示信息即可完成迁移。

**目前已经支持增量扫描功能，速度更快。**

**本来加入lint发现错误后执行脚本的功能，但由于在Ci中无法通过Git命令拿到想要信息，夭折了。**

具体在项目中使用可以参考[Android Lint代码检查实践](https://juejin.im/post/6861562664582119432)。如果有感兴趣的大佬，欢迎一起开发、交流。



## 项目结构
> .<br>
  ├── app												// demo工程用来展示lint检查效果<br>
  ├── checks										  // lint规则代码<br>
  ├── custom_lint_config.json			// 规则配置文件<br>
  ├── lintlibrary									// 空项目，依赖了checks用来生成aar包<br>
  ├── lintincrement							 // 实现lint增量扫描注入的代码<br>
  ├── lintplugin									// lint插件，用来增量扫描、简化配置，日志输出<br>



## 依赖

**需要Android Gradle Plugin在3.5.0以上，目前我使用的是3.5.3实测ok。**

根目录gradle
```groovy
buildscript {
  repositories {
    jcenter()
  }
  dependencies {
    classpath "com.rocketzly:lintPlugin:0.0.5"
  }
}
```
module gradle
```groovy
apply plugin: "com.rocketzly.lintPlugin"
```



## 使用

添加依赖后，并在项目根目录下添加custom_lint_config.json规则配置文件（关于配置可以看后面的规则配置）

### 编码实时提示

添加依赖和配置文件后Rebuild一下（如果还不行则执行次`./gradlew :${moduleName}:lintFull`）即可编码实时提示。

![](http://rocketzly.androider.top/lint_as_effect.png)

**有一点需要注意，AS对于Lint实时提示支持的不算特别好，少数情况下会出现编码时提示不了的情况，但是命令行执行生成报告结果都是正常的，所以以报告结果为准。**

### 命令行执行

目前支持两个命令

- `./gradlew lintFull` 全量扫描（只扫自定义issue）
- `./gradlew lintIncrement -Pcurrent="xxx" -Ptarget="xxx"`  增量扫描（只扫自定义issue）

**提一嘴增量扫描是通过`git diff $target $current --name-only --diff-filter=ACMRTUXB `去找到变更文件的，所以只要是git命令支持的都可以作为current和target的入参。**

那么添加依赖和配置文件后，Terminal执行`./gradlew :app:lintFull`或者`./gradlew :app:lintIncrement -Pcurrent="xxx" -Ptarget="xxx`就可以看到检查结果。

以本库为例，执行`./gradlew :app:lintIncrement -Pcurrent="HEAD" -Ptarget="dev"`结果如下：

![](http://rocketzly.androider.top/lint_result3.png)

查看生成的Html文件可以查看详细lint报告

![](http://rocketzly.androider.top/lint_result4.png)

**最新版本代码日志和报告可能稍有不同但不影响结果**

### 

## 插件配置

### module gradle

```groovy
lintConfig {
    baseline = true//生成baseline文件，默认为false不会生成
}
```

- baseline：是否生成baseline文件，默认为false不生成，如果需要生成则设置为true（可选项）

### 默认值（目前还不支持配置，如有需要可以提issue）

- 只扫描自定义Issue
- 不会将warning视为error
- 只有发现error的时候才会停止task执行
- html报告地址：`${modulePath}/build/reports/lint-results.html`
- xml报告地址：`${modulePath}/build/reports/lint-results.xml`
- baseline地址：`${modulePath}/lint-baseline.xml`



## 规则配置

自定义Issue规则是从根目录custom_lint_config.json文件读取。

目前支持五大类规则（后期会继续增加、完善规则，欢迎有兴趣大佬一起开发）

- 避免使用Api（又可细分为三类）
  - 避免使用的方法
  - 避免创建的类
  - 避免继承或实现的类
- 需要处理异常的方法
- 有依赖关系Api（即trigger_method方法调用后，必须调用dependency_method方法）
- 资源命名规范
- Serializerable对象的引用类型成员变量也必须要实现Serializerable接口

具体配置规则和说明如下（注：由于markdown表格没法表示层级关系，故用+代表json层级关系）


| 字段 | 类型 | 默认值 | 是否必须 | 备注 |
| :-----| :--: | :----: | :----: | :----: |
| avoid_usage_api | Object | 无 |非必须 |避免使用api |
| +method | Object[] | 无 |非必须 |避免使用的方法 |
| ++name | String | 无 |name和name_regex有一个即可 |方法名，匹配方法全路径，以Toast#show()为例，则应填入android.widget.Toast.show |
| ++name_regex | String | 无 |name和name_regex有一个即可 |方法名正则匹配，以Log#i()为例，则应填入android\.util\.Log\.(v\|d\|i\|w\|e) |
| ++message | String | 无 |必须 |提示信息 |
| ++exclude | String[] | 无 |非必须 |需要排除检查的类，匹配全路径类名，例如com.rocketzly.androidlint.Test |
| ++exclude_regex | String | 无 |非必须 |需要排除检查的类正则匹配 |
| ++severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |
| +construction | Object[] | 无 |非必须 |避免创建类 |
| ++name | String | 无 |name和name_regex有一个即可 |构造函数名，匹配方法全路径，以Thread构造函数为例java.lang.Thread |
| ++name_regex | String | 无 |name和name_regex有一个即可 |方法名正则匹配 |
| ++message | String | 无 |必须 |提示信息 |
| ++exclude | String[] | 无 |非必须 |需要排除检查的类，匹配全路径类名，例如com.rocketzly.androidlint.Test |
| ++exclude_regex | String | 无 |非必须 |需要排除检查的类正则匹配 |
| ++severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |
| +inherit | Object[] | 无 |非必须 |避免继承或实现的类 |
| ++name | String | 无 |name和name_regex有一个即可 |类名，匹配全路径类名，例如com.rocketzly.androidlint.Test |
| ++name_regex | String | 无 |name和name_regex有一个即可 |类名正则匹配，例如\\.(AppCompat\|Main)?Activity$ |
| ++message | String | 无 |必须 |提示信息 |
| ++exclude | String[] | 无 |非必须 |需要排除检查的类，匹配全路径类名，例如com.rocketzly.androidlint.Test |
| ++exclude_regex | String | 无 |非必须 |需要排除检查的类正则匹配 |
| ++severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |
| handle_exception_method | Object[] | 无 |非必须 |需要处理异常的方法 |
| +name | String | 无 |name和name_regex有一个即可 |方法名，匹配方法全路径，以Toast#show()为例，则应填入android.widget.Toast.show |
| +name_regex | String | 无 |name和name_regex有一个即可 |方法名正则匹配，以Log#i()为例，则应填入android\.util\.Log\.(v\|d\|i\|w\|e) |
| +exception | String | 无 |必须 |需要处理的异常全路径类名，如java.lang.IllegalArgumentException |
| +message | String | 无 |必须 |提示信息 |
| +exclude | String[] | 无 |非必须 |需要排除检查的类，匹配全路径类名，例如com.rocketzly.androidlint.Test |
| +exclude_regex | String | 无 |非必须 |需要排除检查的类正则匹配 |
| +severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |
| dependency_api | Object[] | 无 |非必须 |有依赖关系api |
| +trigger_method | String | 无 |必须 |触发方法，匹配方法全路径，以Toast#show()为例，则应填入android.widget.Toast.show |
| +dependency_method | String | 无 |必须 |依赖方法，匹配方法全路径，即当调用trigger_method后需要调用dependency_method方法 |
| +message | String | 无 |必须 |提示信息 |
| +severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |
| resource_name | Object | 无 |非必须 |资源命名规范 |
| +drawable | Object | 无 |非必须 |drawable命名规范 |
| ++name_regex | String | 无 |必须 |资源名正则规范 |
| ++message | String | 无 |必须 |提示信息 |
| ++severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |
| +layout | Object | 无 |非必须 |layout命名规范 |
| ++name_regex | String | 无 |必须 |资源名正则规范 |
| ++message | String | 无 |必须 |提示信息 |
| ++severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |
| serializable_config | Object | 无 |非必须 |Serializable配置 |
| +name_regex | String | 无 |必须 |检查哪些包下的类，例如只检查自己包下的类^com\\.rocketzly\\.androidlint |
| +message | String | 无 |必须 |提示信息 |
| +severity | String | "error" |非必须 |lint错误严重程度默认error，可输入fatal、error、warning、informational、ignore |

匹配规则是先排除后匹配，优先级上exclude > exclude_regex > name > name_regex。

message会影响到AS实时提示展示的信息和report中展示信息。

severity则是对应AS实时提示的错误等级和report中错误等级。

具体demo可以看项目根目录下[custom_lint_config.json](./custom_lint_config.json)。



## 协议

[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).




