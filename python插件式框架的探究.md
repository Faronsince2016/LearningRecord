为了扩充软件的功能，通常我们会把软件设计成插件式结构。插件机制是代码/功能反向依赖注入到主体程序的一种方法，编译型语言通过动态加载动态库实现插件。对于Python这样的脚本语言，实现插件机制更简单。`Python`这样的动态语言天生就支持插件式编程。与`C++`相比，`Python`已经定义好模块的接口，想要载入一个插件，一个`__import__()`就能很轻松地搞定。不需要特定的底层知识。而且与`C++`等静态语言相比，`Python`的插件式结构更显灵活。因为插件载入后，可以利用`Python`语言的动态性，充分地修改核心的逻辑。

简单地说一个`__import__()`可能不大清楚。现在就来看一个最简单的插件式结构程序。它会扫描plugins文件夹下的所有`.py`文件。然后把它们载入。
```python
#-*- encoding: utf-8 -*-
#main1.py
import os

class Platform:
    def __init__(self):
        self.loadPlugins()

    def sayHello(self, from_):
        print "hello from %s." % from_

    def loadPlugins(self):
        for filename in os.listdir("plugins"):
            if not filename.endswith(".py") or filename.startswith("_"):
                continue
            self.runPlugin(filename)

    def runPlugin(self, filename):
        pluginName=os.path.splitext(filename)[0]
        plugin=__import__("plugins."+pluginName, fromlist=[pluginName])
        #Errors may be occured. Handle it yourself.
        plugin.run(self)

if __name__=="__main__":
    platform=Platform()
```
然后在plugins子目录里面放入两个文件：
```python
#plugins1.py
def run(platform):
    platform.sayHello("plugin1")
```
```python
#plugins2.py
def run(platform):
    platform.sayHello("plugin2")
```
再创建一个空的`__init__.py`在plugins文件夹里面。从`package`里面导入模块的时候，Python要求一个`__init__.py`。

运行`main1.py`，看一下运行的结果。首先是打印一下文件夹结构方便大家理解：
```ini
|---main1.py
\---plugins
        plugin1.py
        plugin2.py
        __init__.py
```
```python
python3 main1.py
hello from plugin1.
hello from plugin2.
```
一般地，载入插件前要首先扫描插件，然后依次载入并运行插件。我们上面的示例程序main1.py也是如此，分为两个函数。第一个`loadPlugins()`扫描插件。它把`plugins`目录下面所有`.py`的文件除了`__init__.py`都当成插件。`runPlugin()`载入并运行插件。其中两个关键：使用`__import__()`函数把插件当成模块导入，它要求所有的插件都定义一个`run()`函数。各种语言实现的插件式结构其实也基本上分为这两个步骤。所不同的是，`Python`语言实现起来更加的简洁。

或许听起来还有点玄奥。详细地说一下`__import__()`。它和常见的import语句很相似，只不过换成函数形式并且返回模块以供调用。
`import module`相当于`__import__("module")`
`from module import func`相当于`__import__("module", fromlist=["func"])`
不过与想象有点不同
`import package.module`相当于`__import__("package.module", fromlist=["module"])`

如何调用插件一般有个约定。像我们这里就约定每个插件都实现一个`run()`。有时候还可以约定实现一个类，并且要求这个类实现某个管理接口，以方便核心随时启动、停止插件。要求所有的插件都有这几个接口方法：
```python
#interfaces.py
class Plugin:
    def setPlatform(self, platform):
        self.platform=platform

    def start(self):
        pass

    def stop(self):
        pass
```
想要运行这个插件，我们的`runPlugin()`要改一改，另外增加一个`shutdown()`来停止插件:

```python
class Platform:
    def __init__(self):
        self.plugins=[]
        self.loadPlugins()

    def sayHello(self, from_):
        print "hello from %s." % from_

    def loadPlugins(self):
        for filename in os.listdir("plugins"):
            if not filename.endswith(".py") or filename.startswith("_"):
                continue
            self.runPlugin(filename)

    def runPlugin(self, filename):
        pluginName=os.path.splitext(filename)[0]
        plugin=__import__("plugins."+pluginName, fromlist=[pluginName])
        clazz=plugin.getPluginClass()
        o=clazz()
        o.setPlatform(self)
        o.start()
        self.plugins.append(o)

    def shutdown(self):
        for o in self.plugins:
            o.stop()
            o.setPlatform(None)
        self.plugins=[]

if __name__=="__main__":
    platform=Platform()
    platform.shutdown()
```
插件改成这样：
```python
#plugins1.py
class Plugin1:
    def setPlatform(self, platform):
        self.platform=platform

    def start(self):
        self.platform.sayHello("plugin1")

    def stop(self):
        self.platform.sayGoodbye("plugin1")

def getPluginClass():
    return Plugin1
```

```python
#plugins2.py
def sayGoodbye(self, from_):
    print "goodbye from %s." % from_

class Plugin2:
    def setPlatform(self, platform):
        self.platform=platform
        if platform is not None:
            platform.__class__.sayGoodbye=sayGoodbye

    def start(self):
        self.platform.sayHello("plugin2")

    def stop(self):
        self.platform.sayGoodbye("plugin2")

def getPluginClass():
    return Plugin2
```
运行结果：
```python
python main.py
hello from plugin1.
hello from plugin2.
goodbye from plugin1.
goodbye from plugin2.
```
详细观察的朋友们可能会发现，上面的`main.py`,`plugin1.py`, `plugin2.py`干了好几件令人惊奇的事。

首先，`plugin1.py`和`plugin2.py`里面的插件类并没有继承自`interfaces.Plugin`，而`platform`仍然可以直接调用它们的`start()`和`stop()`方法。这件事在`Java、C++`里面可能是件麻烦的事情，但是在`Python`里面却是件稀疏平常的事，仿佛吃饭喝水一般正常。事实上，这正是`Python`鼓励的约定编程。Python的文件接口协议就只规定了`read()`, `write()`, `close()`少数几个方法。多数以文件作为参数的函数都可以传入自定义的文件对象，只要实现其中一两个方法就行了，而不必实现一个什么`FileInterface`。如果那样的话，需要实现的函数就多了，可能要有十几个。

再仔细看下来，`getPluginClass()`可以把类型当成值返回。其实不止是类型，`Python`的函数、模块都可以被当成普通的对象使用。从类型生成一个实例也很简单，直接调用`clazz()`就创建一个对象。不仅如此，`Python`还能够修改类型。上面的例子我们就演示了如何给`Platform`增加一个方法。在两个插件的`stop()`里面我们都调用了`sayGoodbye()`，但是仔细观察`Platform`的定义，里面并没有定义。原理就在这里：
```python
#plugins2.py
def sayGoodbye(self, from_):
    print "goodbye from %s." % from_

class Plugin2:
    def setPlatform(self, platform):
        self.platform=platform
        if platform is not None:
            platform.__class__.sayGoodbye=sayGoodbye
```
这里首先通过`platform.__class__`得到`Platform`类型，然后`Platform.sayGoodbye=sayGoodbye`新增了一个方法。使用这种方法，我们可以让插件任意修改核心的逻辑。这正在文首所说的`Python`实现插件式结构的灵活性，是静态语言如`C++、Java`等无法比拟的。当然，这只是演示，我不大建议使用这种方式，它改变了核心的API，可能会给其它程序员造成困惑。但是可以采用这种方式替换原来的方法，还可以利用**“面向切面编程”**，增强系统的功能。

接下来我们还要再改进一下载入插件的方法，或者说插件的布署方法。前面我们实现的插件体系主要的缺点是每个插件只能有一个源代码。如果想附带一些图片、声音数据，又怕它们会和其它的插件冲突。即使不冲突，下载时分成单独的文件也不方便。最好是把一个插件压缩成一个文件供下载安装。

Firefox是一个支持插件的著名软件。它的插件以.xpi作为扩展名，实际上是一个.zip文件，里面包含了javascript代码、数据文件等很多内容。它会把插件包下载复制并解压到`%APPDATA%\Mozilla\Firefox\Profiles\XXXX.default\extensions`里面，然后调用其中的`install.js`安装。与此类似，实用的`Python`程序也不大可能只有一个源代码，也要像`Firefox`那样支持.zip包格式。

实现一个类似于`Firefox`那样的插件布署体系并不会很难，因为`Python`支持读写`.zip`文件，只要写几行代码来做压缩与解压缩就行了。首先要看一下`zipfile`这个模块。用它解压缩的代码如下：
```python
import zipfile, os

def installPlugin(filename):
    with zipfile.ZipFile(filename) as pluginzip:
        subdir=os.path.splitext(filename)[0]
        topath=os.path.join("plugins", subdir)
        pluginzip.extractall(topath)
```
`ZipFile.extractall()`是`Python 2.6`后新增的函数。它直接解压所有压缩包内的文件。不过这个函数只能用于受信任的压缩包。如果压缩包内包含了以`/`或者`盘符`开始的绝对路径，很有可能会损坏系统。推荐看一下`zipfile`模块的说明文档，事先过滤非法的路径名。

这里只有解压缩的一小段代码，安装过程的界面交互相关的代码很多，不可能在这里举例说明。我觉得UI是非常考验软件设计师的部分。常见的软件会要求用户到网站上查找并下载插件。而`Firefox`和`KDE`提供了一个“组件(部件)管理界面”，用户可以直接在界面内查找插件，查看它的描述，然后直接点击安装。安装后，我们的程序遍历插件目录，载入所有的插件。一般地，软件还需要向用户提供插件的*启用*、*禁用*、*依赖*等功能，甚至可以让用户直接在软件界面上给插件评分，这里就不再详述了。

有个小技巧，安装到`plugins/subdir`下的插件可以通过`__file__`得到它自己的绝对路径。如果这个插件带有图片、声音等数据的时候，可以利用这个功能载入它们。比如上面的`plugin1.py`这个插件，如果它想在启动的时候播放同目录的`message.wav`，可以这样子：
```python
#plugins1.py
import os

def alert():
    soundFile=os.path.join(os.path.dirname(__file__), "message.wav")
    try:
        import winsound
        winsound.PlaySound(soundFile, winsound.SND_FILENAME)
    except (ImportError, RuntimeError):
        pass

class Plugin1:
    def setPlatform(self, platform):
        self.platform=platform

    def start(self):
        self.platform.sayHello("plugin1")
        alert()

    def stop(self):
        self.platform.sayGoodbye("plugin1")

def getPluginClass():
    return Plugin1
```
接下来我们再介绍一种`Python/Java`语言常用的插件管理方式。它不需要事先有一个插件解压过程，因为`Python`支持从`.zip`文件导入模块，很类似于`Java`直接从`.jar`文件载入代码。所谓安装，只要简单地把插件复制到特定的目录即可，Python代码自动扫描并从`.zip`文件内载入代码。下面是一个最简单的例子，它和上面的几个例子一样，包含一个`main.py`，这是主程序，一个`plugins`子目录，用于存放插件。我们这里只有一个插件，名为`plugin1.zip`。`plugin1.zip`有以下两个文件，其中`description.txt`保存了插件内的入口函数和插件的名字等信息，而`plugin1.py`是插件的主要代码:
```ini
description.txt
plugin1.py
```
其中`description.txt`的内容是:
```ini
[general]
name=plugin1
description=Just a test 
code=plugin1.Plugin1
```
`plugin1.py`与前面的例子类似，为了省事，我们去掉了`stop()`方法，它的内容是:
```python
class Plugin1:
    def setPlatform(self, platform):
        self.platform=platform

    def start(self):
        self.platform.sayHello("plugin1")
```
重写的main.py的内容是:
```python
# -*- coding: utf-8 -*-
import os, zipfile, sys, ConfigParser

class Platform:
    def __init__(self):
        self.loadPlugins()

    def sayHello(self, from_):
        print "hello from %s." % from_

    def loadPlugins(self):
        for filename in os.listdir("plugins"):
            if not filename.endswith(".zip"):
                continue
            self.runPlugin(filename)

    def runPlugin(self, filename):
        pluginPath=os.path.join("plugins", filename)
        pluginInfo, plugin = self.getPlugin(pluginPath)
        print "loading plugin: %s, description: %s" % \
                (pluginInfo["name"], pluginInfo["description"])
        plugin.setPlatform(self)
        plugin.start()

    def getPlugin(self, pluginPath):
        pluginzip=zipfile.ZipFile(pluginPath, "r")
        description_txt=pluginzip.open("description.txt")
        parser=ConfigParser.ConfigParser()
        parser.readfp(description_txt)
        pluginInfo={}
        pluginInfo["name"]=parser.get("general", "name")
        pluginInfo["description"]=parser.get("general", "description")
        pluginInfo["code"]=parser.get("general", "code")

        sys.path.append(pluginPath)
        moduleName, pluginClassName=pluginInfo["code"].rsplit(".", 1)
        module=__import__(moduleName, fromlist=[pluginClassName, ])
        pluginClass=getattr(module, pluginClassName)
        plugin=pluginClass()
        return pluginInfo, plugin

if __name__=="__main__":
    platform=Platform()
```
与前一个例子的主要不同之处是getPlugin()。它首先从.zip文件内读取描述信息，然后把这个.zip文件添加到sys.path里面。最后与前面类似地导入模块并执行。

解压还是不解压，两种方案各有优劣。一般地，把.zip文件解压到独立的文件夹内需要一个解压缩过程，或者是人工解压，或者是由软件解压。解压后的运行效率会高一些。而直接使用.zip包的话，只需要让用户把插件复制到特定的位置即可，但是每次运行的时候都需要在内存里面解压缩，效率降低。另外，从.zip文件读取数据总是比较麻烦。推荐不包含没有数据文件的时候使用。