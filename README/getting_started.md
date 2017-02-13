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

#### Add a Fragment to an Activity using XML
`FragmentActivity` 是用来支持 API level 11 以下的版本，如果版本在 11 及以上，则可以使用普通的Activity.

可以通过在 xml 文件中指定`Fragment`的`name`属性，从而指定特定的Fragment class.

### Building a Flexible UI
`FragmentManager` 类提供添加、移除、替换fragment的方法，给用户动态的适配体验。

#### Add a Fragment to an Activity at Runtime
使用`FragmentManager` 创建一个 `FragmentTransaction`。

在Activity运行时添加Fragment有一点需要注意：Activity必须包含一个你可以插入 fragment 的 View.

* Get a `FragmentManager`: `getSupportFragmentManager()`
* Create a `FragmentTransaction`: `beginTransaction()`
* Add a `Fragment`: `add()`

#### Replace One Fragment with Another
使用 `replace()` 代替 `add()`.

Best Practice：在进行Fragment替换时，最好允许用户返回或者取消操作：`addToBackStack()`（在 `FragmentTransaction.commit()` 之前，Fragment 将不会被销毁，只会被remove掉）.

## Communicating with Other Fragments
所有Fragment之间的信息交换都是通过与其相关的Activity来完成，任何Fragment不应该直接交流。

### Define an Interface
在 Fragment 中定义一个 interface，在 Activity 中实现这个 interface。Fragment 将会在 `onAttach()` 中通过得到 Activity 对象捕获这个实现，从而通过 Activity 来进行信息交流。

#### Implement the Interface
Activity 需要实现 Fragment 中声明的 interface.

#### Deliver a Message to a Fragment
Activity 可以通过 `findFragmentById()` 获取到 Fragment 实例，直接调用 Fragment 的方法。

**场景：AFragment 有一堆文章列表，点击某个文章，进入 BFragment，阅读这篇文章。**

1. AFragment: callback.click(title)
2. Activity: click(title) {title -> content -> replaceToBFragment(content) -> BFragment.updateArticleView(content)}
3. BFragment: updateArticleView(content)

## Saving Data
在 Android 中，有三种数据存储方式：

* shared preferences 文件：key-value
* 文件系统：任何文件
* SQLite: 数据库

### Saving Key-Value Sets
一个 `SharedPreferences` 对象指向一个包含`key-value`的文件，提供简单的方法进行读写。

#### Get a Handle to a SharedPreferences
创建或获取一个 shared preference 文件：

* `getSharedPreferences()`：拥有多个shared preference文件，通过传入文件名获取对象，可以从任意的`context`中获取。
* `getPreferences()`：如果一个activity只有一个shared preference文件，通过这个方法可以获取activity对应的SP文件

#### Write to Shared Preferences
1. 创建 `SharedPreferences.Editor`：调用 SharedPreferences 对象的 `edit()` 方法
2. 写入键值对：`putInt()`, `putString`, .etc
3. 保存更改：`commit()`

#### Read from Shared Preferences
`getInt()`, `getString`, .etc.

### Saving Files
使用 `File` API 来操作 Android 中的文件系统。

一个`File`对象适合读写大数据文件，从头到尾没有中断的顺序读取。

#### Choose Internal or External Storage
所有的Android设备拥有两个文件存储域：“internal” 和 “external”：

* Internal Storage
** 永远可用
** 文件只能被 App 访问
** 当 App 被卸载时，所有存储的 internal 的文件都会被删除
* External Storage
** 不一定可用
** 可被全局访问
** 当 App 被卸载时，系统只会删除特定的文件夹（`getExternalFilesDir()`）

App 会被默认载入到internal中，在代码中如何设置下载位置？

- 在AndroidManifest中：更改`android:installLocation`

#### Obtain Permissions for External Storage
在external写文件：需要权限 `android.permission.WRITE_EXTERNAL_STORAGE`

在external写文件（in future）：需要权限：`android.permission.READ_EXTERNAL_STORAGE`

#### Save a File on Internal Storage
* `getFilesDir()`：返回app在internal中的位置
* `getCacheDir()`: 返回app在internal中保存cache的位置，一定要在不需要的时候及时删掉
* 写文件：`FileOutputStream fos = openFileOutput(filename, file_mode);`
* 创建cache文件：`File file = File.createTempFile(filename, null, context.getCacheDir());`

#### Save a File on External Storage
由于external文件有很多不在场的不确定因素，所以在访问文件前最好验证其可用性：
`getExternalStorageState()` 获取external storage状态：如果返回`MEDIA_MOUNTED`，则可用。

* Public Files: 需要留存 -> create from -> `getExternalStoragePublicDirectory()`
* Private Files: 需要删除 -> create from -> `getExternalFilesDir()`
* 文件类型：`DIRECTORY_PICTURES`, `DIRECTORY_MUSIC`, `DIRECTORY_RINGTONES`, .etc

#### Query Free Space
* `getFreeSpace()`
* `getTotalSpace()`

#### Delete a File
* `file.delete()`
* `context.deleteFile(filename)`

### Saving Data in SQL Databases
#### Define a Schema and Contract
在 Contract 类中通过实现`BaseColumns`内部类，可以获得内部key`_ID`。

#### Create a Database Using a SQL Helper
`SQLiteOpenHelper` 用来提供仅在的需要时候可长时间运行的操作（添加/更新数据库），避免在项目运行时就实例化数据库操作类。

* `getWritableDatabase()`：获取可写的database
* `getReadableDatabase()`：获取可读的database

只可在非UI线程调用以上两种方法，例如`AsyncTask`和`IntentService`.

继承`SQLiteOpenHelper`，需要重写`onCreate()`, `onUpgrade()`, `onOpen()`, (可选)`onDowngrade()`.

#### Put Information into a Database
Insert: `Database` -> `ContentValues` -> `db.insert(table_name, action_if_content_values_empty, content_values)`

#### Read Information from a Database
Read: `Database` -> `db.query(table_name, columns_to_return, column_where, column_where_value, group, filter, sort_order)`

Return: `Cursor` -> cursor starts at position -1.
* `moveToNext()`: position+1
* `getXXX()`: 获取列值
* `getColumnIndex()/getColumnIndexOrThrow()`: 获取当前 position
* `close()`: 关闭游标

#### Delete Information from Database
Delete: `Database` -> `db.delete(table_name, selection, selection_args)`

#### Update a Database
Update: combine `insert()` & `delete()` -> `db.update(table_name, content_values, selection, selection_args)`

#### Persisting Database Connection
一般在`Activity`被摧毁时关闭`DBHelper` -> `dbHelper.close()`




