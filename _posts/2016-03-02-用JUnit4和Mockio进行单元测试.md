---
layout: post
title: 用JUnit4和Mockio进行单元测试
date:   2016-03-02 23:10:32
---

**在这一节里，我们将对项目进行配置以进行测试，设置Android Studio运行我们的测试，探索如何通过单元测试在组件之间建立契约关系。**

我们将菜这里采用一种测试驱动（test-driven）方法。这意味着我们在着手实现功能之前要先编写一些描述这些功能的测试。这些测试还将帮助我们为Presenter定义“契约”。采用这种方法对你的应用程序的设计将有极大帮助。它帮助你在具体实现功能之前理解你的对象们的职责以及它们之间如何通信。首先我们将目光转向**notes list**的实现，它在程序启动时展示一个包含所有已添加的note的列表。我们来编写一组测试以定义presenter与repository之间的交互。

###配置你的项目以支持单元测试###

在开始编写测试之前，让我们过一个步骤清单以确保我们的项目的配置能够执行本地测试。所谓本地测试就是运行在你电脑上而不需要方位Android框架或者Android设备。（记住，我们首先测试的是组件之间的基本交互。）

####切换到项目视图####

测试存储在`src`文件夹下的`test`和`androidTest`目录。好消息是——我们已经替你创建了这些目录。

在Android视图下，这些文件夹是隐藏的。所以你需要从左上角的下拉列表切换到项目视图。

> 如果你开始于一个代码快照，请确保重新切换到项目视图。

![img01](/img/android_testing/img01.png)

你最终的视图层级看上去应当是这样的：注意我们应用模块中出现的新文件夹。

![img02](/img/android_testing/img02.png)

####源码组定义Product Flavors####

`src`文件夹下的子目录我们称之为源码组，它们组成了一个product flavor。我们有4个不同的源码组，不同的构建和测试流程将选取对应的代码组。我们的单元测试将进入`test`子目录。

Product flavor定义了一个定制的应用程序版本。一个项目可以有不同的flavors，构建出来的应用也随它们而变化。

这个新概念的设计用途，是在代码库的某些部分随应用程序各个变体的差异而有区别时起作用。本项目中定义了两个flavors：`mock`和`prod`。前者用来替换特定的部分以使测试更加简便。

代码的主体仍存在于主目录下。只有那下可能需要变体的代码须要往`mock`和`prod`目录下放置不同的版本。同样的方式也用在Android测试（多数代码在`androidTest`下，但是调用`mock`中不同实现的那部分则在`androidTestMock`下）。


>Flavor对应的文件夹中的文件并不替换主源码组中的文件。这样做会导致类重复异常。这是一种常见的错误认识，因为资源就是这样合并的。

>查看Gradle任务列表，你会发现像installMockDebug或installProdDebug这样的新任务。现在，每一个操作都需要你指定对应的vatiant。在Android Studio中，可以从Build Variants窗口中选取。


源码组              |说明            
----------------- | ------------- 
androidTest       | Android测试（运行于设备或者模拟器）所在的位置
androidTestMock   | mock flavor对应的Android测试位置。用在只需要在隔离环境运行测试的时候
main              | 主源码组，所有源码的默认位置
mock              | 用来测试的自定义app flavor。在这里，类Injection为孤立运行，通过假数据实现了伪装的依赖，而不必触及网络或者外部存储（mock = 模仿复杂真实对象行为的模拟依赖）
prod              | 与mock flavor相反，这里的Injection类提供了所依赖的真实环境上下文（prod = 产品实现）
test              | 本地测试的位置


####为JUnit4添加Gradle依赖####

我们要添加的第一个测试是JUnit测试。我们已经在我们app模块的build.gradle文件中添加了必要的依赖。我们还要在这一步骤以及以后使用Mockito和Hamcrest matchers。现在，我们的依赖应该如下：

**app/build.gradle**

{% highlight java %}
// Dependencies for local unit tests
testCompile "junit:junit:$rootProject.ext.junitVersion"
testCompile "org.mockito:mockito-all:$rootProject.ext.mockitoVersion"
testCompile "org.hamcrest:hamcrest-all:$rootProject.ext.hamcrestVersion"
testCompile "org.powermock:powermock-module-junit4:$rootProject.ext.powerMockito"
testCompile "org.powermock:powermock-api-mockito:$rootProject.ext.powerMockito"
{% endhighlight %}


> **Gradle专业提示：用变量定义版本号**
>
>注意我们用$rootProject.ext.junitVersion代替了版本号。我们在根项目的gradle构建文件里定义这些变量（build.gradle）。这样我们修改一次变量就可以对全部模块起作用。
>
>在我们所有的依赖配置中都沿用此语法。


切换到“Unit Tests” Build Variant
将**Build Variant**切换到`Unit Tests`以激活`tests/`目录，使用该目录下的Java文件。

打开**Build Variants**边栏（左下角），将“Test Artifact”切换到`Unit Tests`：

![img03](/img/android_testing/img03.png)

####可选：同步你的项目####

。。。

####创建你的第一个单元测试：Notes Presenter####

现在所有的配置已经完成，是时候编写第一个测试了。一个单元测试执行你应用程序中的一个小组件，通常是一个方法或者回调。记住，我们编写单元测试在先，随后才实现它们的逻辑。

我们来为`NotesPresenter`实现几个测试。

Notes presenter处在数据模型与视图的中间。它基于接收到的回调来重置视图（显示加载提示，展示note列表，或者提示内容为空）。它还负责根据用户在屏幕上选定的选项来显示不同的视图，比如显示note详情，或者“添加note”操作界面。

####用Mockito模拟依赖####

我们第一个测试单纯关注`NotesPresenter`。不过，`NotesPresenter`类并不是孤立存在的。它还需要一个视图对象（NotesContract.View）来更新UI，以及存储notes的仓库（repository）。此时，我们还不用操心视图和仓库的具体实现。我们只用确保presenter能按照我们的期望运行。

在隔离测试条件下，我们能够使用`模拟对象`来测试我们的类，而拜托对真实实现的依赖。模拟对象是一个虚拟类，你可以根据输入来定义它的方法的输出和行为。

这对于我们的测试来说简直完美——我们并不知道视图如何显示进度，我们只知道presenter`必定`会调用视图的某个方法来通知它显示进度。我们接下来编写的测试将关注presenter是否在我们调用它的方法的时候能够按照预期对视图进行调用。

[Mockito](http://mockito.org)是Android平台上一种流行的mocking框架。你将在下一节看到它的一些语法。

使用Mockito这类mocking框架的另一大好处是针对调用参数测试的选项。这意味着你可以编写测试来捕获传送给方法的参数，然后对它们进行测试。（在Mockito中这项功能通过`ArgumentCaptor`实现）。

####单元测试1：打开“添加Notes”界面####

单元测试位于`test`/源码组中。

确认你已经选择了**Unit Tests**build variant，否则单元测试相关类会被禁用。

**打开文件**`app/src/test/java/.../notes/NotesPresenterTest.java`，

> 如果你无法看到此文件，确认你已经按照前面章节中所说的切换到**Project perspective**视图，并且你已经选择了**Unit Tests**。

我们的第一个测试验证的是，请求Presenter打开"Add new note"界面的操作是否按照期望的那样调用视图来更新当前界面。

我们来实现`clickOnFab_ShowsAddsNoteUi()`测试。

首先，删除对`fail(...)`调用，因为我们现在正在实现这个测试。

然后，调用presenter的`addNewNote()`方法，接着我们验证视图的`showAddNote()`方法被调用到。

以下使我们最终的实现：

{% highlight java %}
@Test
public void clickOnFab_ShowsAddsNoteUi() {
   // When adding a new note
   mNotesPresenter.addNewNote();

   // Then add note UI is shown
   verify(mNotesView).showAddNote();
}
{% endhighlight %}

> 我们可以直接调用`verfy(...)`，因为我们已经为它做了静态导入：`import static org.mockito.Mockito.verify;`

####运行你的单元测试####

现在是运行你的测试的最后时刻！

在`NotesPresenterTest`类上右键单击，并选择**Run > NotesPresenterTest**。

![img04](/img/android_testing/img04.png)

也能从命令行运行测试：

> ./gradlew -Dtest.single=NotesPresenterTest test

> 你可能已经注意到Android Studio并没有要求你连接设备或者启动模拟器。这是因为src/tests文件夹下的测试时本地单元测试，它们运行在你电脑的Java虚拟机上。这样确实能够实现非常快速的迭代流，然后实现功能以通过测试，接着添加更多测试。。。这就是测试驱动开发了。

**恭喜！**你已经编写了你第一个单元测试！**（这个测试此时应当会失败）**

![img05](/img/android_testing/img05.png)

现在测试失败，因为我们还没有在presenter中实现对应的逻辑。

####实现NotesPresenter#addNewNote####

**NotesPresenter.java**

{% highlight java %}
@Override
public void addNewNote() {
   mNotesView.showAddNote();
}
{% endhighlight %}

重新运行测试，现在完全成功了。













