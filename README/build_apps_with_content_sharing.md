>如何创建apps与设备之间共享数据的app.

# Sharing Simple Data
使用`Intent`和`ActionProvider`在app之间收发简单数据。

## Send Simple Data to Other Apps
当你构造一个intent时，你必须确认你想让intent来`trigger`什么行为.

Android 定义了几种行为，其中有一种`ACTION_SEND`，用来在activity之间传递数据，甚至在进程之间也可以。

> **Note** 最佳实践是使用`ShareActionProvider`（Lesson -- Adding an Easy Share Action）添加action到`ActionBar`中（> API 14）.

### Send Text Content
Talk is cheap, I will show you the code.

    Intent sendIntent = new Intent();
    sendIntent.setAction(Intent.ACTION_SEND);
    sendIntent.putExtra(Intent.EXTRA_TEXT, "This is my text to send");
    sendIntent.setType("text/plain");
    startActivity(sendIntent);

如果有多个App接收到该intent，系统会自动显示一个可选的dialog，当然，你可以调用`Intent.createChooser()`，在任何版本任何情况下都会显示选择框。优点：

* 即使用户之前选择了默认的action，选择框还是会显示
* 如果没有App match到，Android会提示系统消息
* 你可以自定义选择框的title

    startActivity(Intent.createChooser(sendIntent, getResources().getText(R.string.send_to));

作为可选项，你也可以为intent设置一些基本的extra信息：`EXTRA_EMAIL`, `EXTRA_CC`, `EXTRA_BCC`, `EXTRA_SUBJECT`.

> **Note** 一些e-mail应用（例如Gmail）,会期望添加`String[]`作为extra.

### Send Binary Content
使用`ACTION_SEND`与`EXTRA_STREAM`(URI)结合来发送二进制内容。

    Intent shareIntent = new Intent();
    shareIntent.setAction(Intent.ACTION_SEND);
    shareIntent.putExtra(Intent.EXTRA_STREAM, uriToImage);
    shareIntent.setType("image/jpeg");
    startActivity(Intent.createChooser(shareIntent, getResources().getText(R.string.send_to)));

**Note**

* 可以使用`*/*` MIME类型，但只能match到处理普通数据流的receiver.
* 接收方需要权限来访问Uri指向的数据。这里有两种比较推荐的解决办法：
    
    * 创建app自身的`ContentProvider`，确保其他app可以访问你的provider：使用`per-URI permissions` -- 短期将权限授予给接收方.创建一个`ContentProvider`的简单方法是使用`FileProvider`的帮助类.
    * 使用系统`MediaStore`. 主要包含video, audio, image MIME 类型(在Android 3.0 以下还会包含一些非媒体类型). 文件可被插入到`MediaStore`中：使用`scanFile()`，然后将适合共享的Uri（content://）传递给`onScanCompleted()`回调.

### Send Multiple Pieces of Content
使用`ACTION_SEND_MULTIPLE` action来发送多块数据（多条指向content的uri）.
MIME 类型match你共享的所有文件.
    
    ArrayList<Uri> imageUris = new ArrayList<Uri>();
    imageUris.add(imageUri1);
    imageUris.add(imageUri2);

    Intent shareIntent = new Intent();
    shareIntent.setAction(Intent.ACTION_SEND_MULTIPLE);
    shareIntent.putParcelableArrayListExtra(Intent.EXTRA_STREAM, imageUris);
    shareIntent.setType("image/*");
    startActivity(Intent.createChooser(shareIntent, "Share images to.."));

## Receiving Simple Data from Other Apps
### Update Your Manifest
Intent filters告诉系统App可以接受怎样的intent事件.

    <activity android:name=".ui.MyActivity" >
        <intent-filter>
            <action android:name="android.intent.action.SEND" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="image/*" />
        </intent-filter>
        <intent-filter>
            <action android:name="android.intent.action.SEND_MULTIPLE" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="image/*" />
        </intent-filter>
    </activity>

### Handle the Incoming Content

    void onCreate(Bundle savedInstanceState) {
        Intent intent = getIntent();
        String action = intent.getAction();
        String type = intent.getType();

        if (Intent.ACTION_SEND.equals(action) && type != null) {
            if ("text/plain".equals(type)) {
                handleSendText(intent);    
            } else if (type.startsWith("image/")) {
                handleSendImage(intent);    
            }
        } else if (Intent.ACTION_SEND_MULTIPLE.equals(action) && type != null) {
            if (type.startsWith("image/")) {
                handleSendMultipleImages(intent);    
            } else {
                //Handle other intents    
            }   
        }
    }

    void handleSendText(Intent intent) {
        String sharedText = intent.getStringExtra(Intent.EXTRA_TEXT);
        if (sharedText != null) {
            //Update UI    
        }
    }

    void handleSendImage(Intent intent) {
        Uri imageUri = (Uri) intent.getParcelbleExtra(Intent.EXTRA_STREAM);
        if (imageUri != null) {
            //Update    
        }
    }

    void handleSendMultipleImages(Intent intent) {
        ArrayList<Uri> imageUris = intent.getParcelbleArrayList(Intent.EXTRA_STREAM);
        if (imageUris != null) {
            //Update    
        }
    }

## Adding an Easy Share Action
使用 `ShareActionProvider` 在ActionBar上添加share action（> Android 4.0）.

### Update Menu Declarations
    
    <menu xmlns:android="http://schemas.android.com/apk/res/android">
        <item
            android:id="@+id/menu_item_share"
            android:showAsAction="isRoom"
            android:title="Share"
            android:actionProviderClass="android.widget.ShareActionProvider" />
    </menu>

### Set the Share Intent
使用`ShareActionProvider`，必须为其提供一个intent，需要先调用`MenuItem.getActionProvider()`来取回`ShareActionProvider`实例，再调用`setShareIntent()`为其指定intent.

    private ShareActionProvider shareActionProvider;

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.share_menu, menu);

        MenuItem item = menu.findItem(R.id.menu_item_share);

        shareActionProvider = (ShareActionProvider) item.getActionProvider();
        return true;
    }

    private void setShareIntent(Intent shareIntent) {
        if (shareActionProvider != null) {
            shareActionProvider.setShareIntent(shareIntent);    
        }    
    }

# Sharing Files
共享文件：提供App文件的URI给接收方，并给予短暂的可读/可写权限.
`FileProvider` => `getUriForFile()` => 生成文件的URI

共享较小的text/numeric数据：发送包含数据的Intent.

## Setting Up File Sharing
`FileProvider` 是 `v4 Support Library` 的一部分.

### Specify the FileProvider
在配置文件中定义 `FileProvider`:

    <application
        ...>
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.example.myapp.fileprovider"
            android:grantUriPermissions="true"
            android:exported="false">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths" />
        </provider>
        ...
    </application>


* `android:authorities`: FileProvider 生成的content URIs

### Specify Sharable Directories
一旦在配置文件中添加了`FileProvider`，就需要指定一个目录来放置你想共享的文件：

在res/xml中创建`filepaths.xml`. 

    <paths>
        <files-path path="images/" name="myimages" />
    </paths>


* `files-path`
* `external-path`: share directories in external storage
* `cache-path`: share directories in internal storage

所有准备工作做完以后，`FileProvider`会生成固定格式的访问URI:
`content://com.example.myapp.fileprovider/myimages/<filename>`

## Sharing a File
从server app提供文件选择接口，以便让其他app可以唤起.

### Receive File Requests
如果收到文件访问请求，你的app应该提供一个文件选择`Activity`. 请求方通过调用 `startActivityForResult()` （包含`ACTION_PICK`的Intent）启动该Activity.

### Create a File Selection Activity
<intent-filter>

* action: `ACTION_PICK`
* category: `CATEGORY_DEFAULT` & `CATEGORY_OPENABLE`
* data: mimeType

#### Define the file selection Activity in code
定义Activity来展示你的app中`files/images`的文件，并允许用户点击想要的文件.

    public class MainActivity extends Activity {
        private File mPrivareRootDir;
        private File mImagesDir;
        File[] mImageFiles;
        String[] mImageFilenames;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            mResultIntent = new Intent("com.example.myapp.ACTION_RETURN_FILE");
            mPrivateRootDir = getFilesDir();
            mImagesDir = new File(mPrivateRootDir, "images");
            mImageFiles = mImageDir.listFiles();
            setResult(Activity.RESULT_CANCELED, null);
        }
    }

### Respond to a File Selection
如果用户选择了文件，你的app必须考虑为选中的文件提供URI.

因为Android 6.0以上只支持运行时授予权限，所以需要避免使用`Uri.fromFile()`:

* 不支持文件共享
* 你的app需要获得`WRITE_EXTERNAL_STORAGE`权限(ANDROID 4.4 及以下)
* 接收文件的app需要有`READ_EXTERNAL_STORAGE` 权限

`onItemClick()` => `File` Object => call `getUriFromFile()`.

        protected void onCreate(Bundle savedInstanceState) {
            mFileListView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
                @Override
                public void onItemClick(AdapterView<?> adapterView,
                            View view,
                            int position,
                            long rowId) {
                    File requestFile = new File(mImageFilename[position]);
                    try {
                        fileUri = FileProvider.getUriForFile(
                                MainActivity.this,
                                "com.example.myapp.fileprovider",
                                requestFile);    
                    } catch (IllegalArgumentException e) {
                        Log.e("File Selector", 
                                "The selected file can't be shared: " + 
                                clickedFileName);    
                    }
                }
            });    
        }

### Grant Permissions for the File
通过给 Intent 设置permission flags赋予读取文件的权限.

    protected void onCreate(Bundle savedInstanceState) {
        mFileListView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> adapterView,
                    View view,
                    int position,
                    long rowId) {
                ...
                if (fileUri != null) {
                    mResultIntent.addFlags(
                        Intent.FLAG_GRANT_READ_URI_PERMISSION);    
                }    
            }
        });    
    }

避免使用`Context.grantUriPermission()`：只能使用`Context.revokeUriPermission()`收回权限

### Share the File with the Requesting App
`setResult`

    protected void onCreate(Bundle savedInstanceState) {
        ...
        mFileListView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> adapterView,
                    View view,
                    int position,
                    long rowId) {
                ...
                if (fileUri != null) {
                    ...
                    mResultIntent.setDataAndType(
                        fileUri,
                        getContentResolver().getType(fileUri));
                    MainActivity.this.setResult(Activity.RESULT_OK,
                        mResultIntent);
                } else {
                    mResultIntent.setDataAndType(null, "");
                    MainActivity.this.setResult(Activity.RESULT_CANCELED,
                        mResultIntent);
                }
            }
        });
    }

## Requesting a Shared File
## Retrieving File Information

# Sharing Files with NFC
