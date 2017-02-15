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

# Saving Data
在 Android 中，有三种数据存储方式：

* shared preferences 文件：key-value
* 文件系统：任何文件
* SQLite: 数据库

## Saving Key-Value Sets
一个 `SharedPreferences` 对象指向一个包含`key-value`的文件，提供简单的方法进行读写。

### Get a Handle to a SharedPreferences
创建或获取一个 shared preference 文件：

* `getSharedPreferences()`：拥有多个shared preference文件，通过传入文件名获取对象，可以从任意的`context`中获取。
* `getPreferences()`：如果一个activity只有一个shared preference文件，通过这个方法可以获取activity对应的SP文件

### Write to Shared Preferences
1. 创建 `SharedPreferences.Editor`：调用 SharedPreferences 对象的 `edit()` 方法
2. 写入键值对：`putInt()`, `putString`, .etc
3. 保存更改：`commit()`

### Read from Shared Preferences
`getInt()`, `getString`, .etc.

## Saving Files
使用 `File` API 来操作 Android 中的文件系统。

一个`File`对象适合读写大数据文件，从头到尾没有中断的顺序读取。

### Choose Internal or External Storage
所有的Android设备拥有两个文件存储域：“internal” 和 “external”：

* Internal Storage
	* 永远可用
	* 文件只能被 App 访问
 	* 当 App 被卸载时，所有存储的 internal 的文件都会被删除
* External Storage
	* 不一定可用
	* 可被全局访问
	* 当 App 被卸载时，系统只会删除特定的文件夹（`getExternalFilesDir()`）

App 会被默认载入到internal中，在代码中如何设置下载位置？

>在AndroidManifest中：更改`android:installLocation`

### Obtain Permissions for External Storage
在external写文件：需要权限 `android.permission.WRITE_EXTERNAL_STORAGE`

在external写文件（in future）：需要权限：`android.permission.READ_EXTERNAL_STORAGE`

### Save a File on Internal Storage
* `getFilesDir()`：返回app在internal中的位置
* `getCacheDir()`: 返回app在internal中保存cache的位置，一定要在不需要的时候及时删掉
* 写文件：`FileOutputStream fos = openFileOutput(filename, file_mode);`
* 创建cache文件：`File file = File.createTempFile(filename, null, context.getCacheDir());`

### Save a File on External Storage
由于external文件有很多不在场的不确定因素，所以在访问文件前最好验证其可用性：
`getExternalStorageState()` 获取external storage状态：如果返回`MEDIA_MOUNTED`，则可用。

* Public Files: 需要留存 -> create from -> `getExternalStoragePublicDirectory()`
* Private Files: 需要删除 -> create from -> `getExternalFilesDir()`
* 文件类型：`DIRECTORY_PICTURES`, `DIRECTORY_MUSIC`, `DIRECTORY_RINGTONES`, .etc

### Query Free Space
* `getFreeSpace()`
* `getTotalSpace()`

### Delete a File
* `file.delete()`
* `context.deleteFile(filename)`

## Saving Data in SQL Databases
### Define a Schema and Contract
在 Contract 类中通过实现`BaseColumns`内部类，可以获得内部key`_ID`。

### Create a Database Using a SQL Helper
`SQLiteOpenHelper` 用来提供仅在的需要时候可长时间运行的操作（添加/更新数据库），避免在项目运行时就实例化数据库操作类。

* `getWritableDatabase()`：获取可写的database
* `getReadableDatabase()`：获取可读的database

只可在非UI线程调用以上两种方法，例如`AsyncTask`和`IntentService`.

继承`SQLiteOpenHelper`，需要重写`onCreate()`, `onUpgrade()`, `onOpen()`, (可选)`onDowngrade()`.

### Put Information into a Database
Insert: `Database` -> `ContentValues` -> `db.insert(table_name, action_if_content_values_empty, content_values)`

### Read Information from a Database
Read: `Database` -> `db.query(table_name, columns_to_return, column_where, column_where_value, group, filter, sort_order)`

Return: `Cursor` -> cursor starts at position -1.
* `moveToNext()`: position+1
* `getXXX()`: 获取列值
* `getColumnIndex()/getColumnIndexOrThrow()`: 获取当前 position
* `close()`: 关闭游标

### Delete Information from Database
Delete: `Database` -> `db.delete(table_name, selection, selection_args)`

### Update a Database
Update: combine `insert()` & `delete()` -> `db.update(table_name, content_values, selection, selection_args)`

### Persisting Database Connection
一般在`Activity`被摧毁时关闭`DBHelper` -> `dbHelper.close()`

# Interacting with Other Apps
## Sending the User to Another App
在与其它的App进行交互时，只能使用implicit intent。

### Build an Implicit Intent
定义`Action`去具体化启动事件。

* 使用`Uri`定义启动事件：
	* 打开拨号页面

			Uri number = Uri.parse("tel:5551234");
			Intent callIntent = new Intent(Intent.ACTION_DIAL, number);

	* 打开地图页面

			Uri location = Uri.parse("geo:0,0?q=1600+Amphitheatre+Parkway,+Mountain+View,+California");
			Intent mapIntent = new Intent(Intent.ACTION_VIEW, location);

	* 打开网页

			Uri webpage = Uri.parse("http://www.android.com");
			Intent webIntent = new Intent(Intent.ACTION_VIEW, webpage);

	* 使用 extra data 具体化启动事件：
	`setType()`: 指定MIME(Multipurpose Internet Mail Extensions) Type
		* 发送 email
	
				Intent emailIntent = new Intent(Intent.ACTION_SEND);
				emailIntent.setType(HTTP.PLAIN_TEXT_TYPE);
			   	emailIntent.putExtra(Intent.EXTRA_EMAIL, new String[] {"jon@example.com"});
			    emailIntent.putExtra(Intent.EXTRA_SUBJECT, "Email subject");
    			emailIntent.putExtra(Intent.EXTAR_TEXT, "Email message text");
    			emailIntent.putExtra(Intent.EXTRA_STREAM, Uri.parse("content://path/to/email/attachment"));

		* 发送 calendar 事件
    
   				Intent calendarIntent = new Intent(Intent.ACTION_INSERT, Events.CONTENT_URI);
    			Calendar beginTime = Calendar.getInstance().set(2012, 0, 19, 7, 30);
    			Calendar endTime = Calendar.getInstance().set(2012, 0, 19, 10, 30);
    			calendarIntent.putExrea(CalendarContract.EXTRA_EVENT_BEGIN_TIME, beginTime.getTimeInMillis());
   				calendarIntent.putExtra(CalendarContract.EXTRA_EVENT_END_TIME, endTime.getTimeInMillis());
   				calendarIntent.putExtra(Events.TITLE, "Ninja class");
    			calendarIntent.putExtra(Events.EVENT_LOCATION, "Secret dojo);

### Verify There is an App to Receive the Intent
如果intent声明的唤起事件并不存在，app将会crash。

* `quertIntentActivities()`: 查看可用事件

    	PackageManager packageManager = getPackageManager();
    	List activities = packageManager.queryIntentActivities(intent, PackageManager.MATCH_DEFAULT_ONLY);
    	boolean isIntentSafe = activities.size() > 0;

### Start an Activity with the Intent
`startActivity(intent)`

### Show an App Chooser
如果有多个唤起事件存在，需要用户自行选择具体的唤起事件，调用`createChooser()`来调起具体的被选事件。

    Intent intent = new Intent(Intent.ACTION_SEND);
    String title = getResources().getString(R.string.choose_title);
    Intent chooser = Intent.createChooser(intent, title);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivity(chooser);
    }

## Getting a Result from an Activity

使用`startActivityForResult()`来启动一个activity并接收返回数据。
使用`onActivityResult()`来处理返回的数据

### Start the Activity

    static final int PICK_CONTACT_REQUEST = 1;
    private void pickContact() {
        Intent pickContactIntent = new Intent(Intent.ACTION_PICK, Uri.parse("content://contacts"));
        pickContactIntent.setType(Phone.CONTENT_TYPE);
        startActivityForResult(pickContactIntent, PICK_CONTACT_REQUEST);
    }

### Receive the Result
通过 `resultCode` 来判断返回类型：
* RESULT_OK: 操作成功
* RESULT_CANCELED: 操作取消

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (requestCode == PICK_CONTACT_REQUEST) {
            if (resultCode == RESULT_OK) {
                //do something...    
            }    
        }    
    }

## Allowing Other Apps to Start Your Activity
通过定义支持`ACTION_SEND`的intent：`<intent-filter>`.

### Add an Intent Filter
在`intent-filter`中定义以下几种criteria：
* Action : action 名称，一般定义为`ACTION_XXX`格式
* Data : 与 intent 相关的数据描述，可以多重定义：MIME Type / URI prefix / URI scheme / combination.
* Category : 提供额外的方式描述处理intent的activity，通常与用户行为或地址相关。一般定义为`CATEGORY_DEFAULT`.

    <activity android:name="ShareActivity"
        <intent-filter>
            <action android:name="android.intent.action.SEND" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="text/plain" />
            <data android:mimeType="image/*" />
        </intent-filter>
    </activity>

**必须定义 `CATEGORY_DEFAULT` 否则implicit intent 无法处理跳转事件.**

### Handle the Intent in Your Activity
调用`getIntent()`.
在activity的生命周期的任何时间段都可以调用，但是最好在`onCreate() / onStart()`中处理。

### Return a Result

    Intent result = new Intent("com.example.RESULT_ACTION", Uri.parse("content://result_uri"));
    setResult(Activity.RESULT_OK, result);
    finish();

# Working with System Permissions
为了保证App的数据安全，Android 在每一个有权限控制的沙箱中运行App。

## Declaring Permissions
### Determine What Permissions Your App Needs
Android 5.1 以下，用户会在安装App的时候赋予权限，在Android 6.0 以上，用户会在App运行时动态赋予权限。

### Add Permissions to the Manifest
在`manifest`属性下，申请permission使用`uses-permission`标签。

## Requesting Permissions at Run Time
系统权限分为两种：normal 和 dangerous：

* 系统会自动赋予 normal 权限
* dangerous 权限需要用户手动授予

在Android 6.0以上，由于权限是动态授予的，所以需要保证在某些权限不可用时，App依然可以正常运行。

### Check for Permissions
`ContextCompat.checkSelfPermissions()`

    int permissionCheck = ContextCompat.checkSelfPermission(thisActivity, Manifest.permission.WRITE_CALENDAR);

* `PackageManager.PERMISSION_GRANTED` = permissionCheck: 权限被授予
* `PackageManager.PERMISSION_DENED` = permissionCheck: 权限被拒绝

### Request Permissions
最佳实践：在用户已经关闭权限时，App运行到需要使用权限才能正常运行的功能时，可以为用户提供权限解释。

#### Explain why the app needs permissions
`shouldShowRequestPermissionRationale()`: 如果App曾经请求过permission，用户拒绝了请求，该方法会返回`true`;
如果App曾经请求过permission，用户拒绝了请求，且选择`Don't ask again`，该方法会返回`false`;
如果设备安全等级拒绝授予该permission请求，该方法会返回`false`.

#### Request the permissions you need
`requestPermissions()` 用来请求权限。

    if (ContextCompat.checkSelfPermission(thisActivity, Manifest.permission.READ_CONTACTS) != PackageManager.PERMISSION_GRANTED) {
        if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity, Manifest.permission.READ_CONTACTS)) {
            //Show an explanation    
        } else {
            ActivityCompat.requestPermissions(thisActivity, new String[]{Manifest.permission.READ_CONTACTS}, MY_PERMISSIONS_REQUEST_READ_CONTACTS);
        }    
    }

#### Handle the permissions request response
`onRequestPermissionsResult()` override 该方法用来查询permission是否成功申请。

    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        switch (requestCode) {
            case MY_PERMISSIONS_REQUEST_READ_CONTACTS: {
                if (grantResults.length > 0
                    && grantResults[0] == PackageManager.PERMISSION_GRANTE) {
                    //Do what you want    
                } else {
                    //Do what when permission was denied    
                }
                return;
            }    
        }    
    }

## Permissions Usage Notes
**权限控制准则**

* Consider Using an Intent
* Only Ask for Permissions You Need
* Don't Overwhelm the User
* Explain Why You Need Permissions
* Test for Both Permissions Models

使用 adb 工具管理权限：

* 分组列出权限和状态

    adb shell pm list permissions -d -g

* 赋予/禁止一或多个权限

    adb shell pm [grant|revoke] <permission-name> ...
