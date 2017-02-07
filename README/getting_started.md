# Building Your First App
# Supporting Different Devices
# Building a Dynamic UI with Fragments
为了适配不同的屏幕尺寸（大屏幕可以比小屏幕多显示几个 Fragment）,这一节主要说明如何通过 Fragments 创造动态化的用户体验，使你的 App 在不同的屏幕尺寸上都可获得最优的用户体验，设备最低支持到 Android 1.6.
## Creating a Fragment
可以认为Fragment是Activity组合的一部分，有自己独立的生命周期，自己的输入事件，当其所依附的Activity还在运行时，可以自由添加或删除Fragment。

**在创建Fragment之前，需要让App使用 Support Library**。

### Create a Fragment Class
1. extend `Fragment`
2. override key lifecycle methods

**必须使用 `onCreateView()` 的callback来定义layout组件**

### Add a Fragment to an Activity using XML
`FragmentActivity` 是用来支持 API level 11 以下的版本，如果版本在 11 及以上，则可以使用普通的Activity.

可以通过在 xml 文件中指定`Fragment`的`name`属性，从而指定特定的Fragment class.

## Building a Flexible UI
`FragmentManager` 类提供添加、移除、替换fragment的方法，给用户动态的适配体验。

### Add a Fragment to an Activity at Runtime
使用`FragmentManager` 创建一个 `FragmentTransaction`。

在Activity运行时添加Fragment有一点需要注意：Activity必须包含一个你可以插入 fragment 的 View.

* Get a `FragmentManager`: `getSupportFragmentManager()`
* Create a `FragmentTransaction`: `beginTransaction()`
* Add a `Fragment`: `add()`

### Replace One Fragment with Another
使用 `replace()` 代替 `add()`.

Best Practice：在进行Fragment替换时，最好允许用户返回或者取消操作：`addToBackStack()`（在 `FragmentTransaction.commit()` 之前，Fragment 将不会被销毁，只会被remove掉）.

## Communicating with Other Fragments
所有Fragment之间的信息交换都是通过与其相关的Activity来完成，任何Fragment不应该直接交流。

### Define an Interface
在 Fragment 中定义一个 interface，在 Activity 中实现这个 interface。Fragment 将会在 `onAttach()` 中通过得到 Activity 对象捕获这个实现，从而通过 Activity 来进行信息交流。

### Implement the Interface
Activity 需要实现 Fragment 中声明的 interface.

### Deliver a Message to a Fragment
Activity 可以通过 `findFragmentById()` 获取到 Fragment 实例，直接调用 Fragment 的方法。

**场景：AFragment 有一堆文章列表，点击某个文章，进入 BFragment，阅读这篇文章。**

1. AFragment: callback.click(title)
2. Activity: click(title) {title -> content -> replaceToBFragment(content) -> BFragment.updateArticleView(content)}
3. BFragment: updateArticleView(content)

