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

### Send a Request for the File
从server app请求文件数据的方式一般为：`startActivityForResult()` + `Intent`(包含`action`、`MIME type`).

    public class MainActivity extends Activity {
        private Intent mRequestFileIntent;
        private ParcelFileDescriptor mInputPFD;
        ...
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            mRequestFileIntent = new Intent(Intent.ACTION_PICK);
            mRequestFileIntent.setType("image/jpg");
            ...
        }
        ...
        protected void requestFile() {
            /**
             * When the user requests a file, send an Intent to the server app files.
            **/    
            startActivityForResult(mRequestFileIntent, 0);
            ...
        }
        ...
    }

### Access the Requested File
override `onActivityResult()` 来处理接收文件，一旦客户端app有了文件的content URI，可以通过获取其`FileDescriptor`来处理文件。

只有在server app赋予了访问权限，client app获取到文件访问入口，文件才可被处理。由于权限是临时的，所以一旦client app的任务栈结束，文件不再可被外部访问。

    @Override
    public void onActivityResult(int requestCode, int resultCode, Intent returnIntent) {
        if (resultCode != RESULT_OK) {
            return;    
        } else {
            Uri returnUri = returnIntent.getData();
            try {
                mInputPFD = getContentResolver().openFileDescriptor(returnUri, "r");    
            } catch (FileNotFoundException e) {
                e.printStackTrace();
                Log.e("MainActivity", "File not found.");
                return;
            }

            FileDescriptor fd = mInputPFD.getFileDescriptor();
            ...
        }
    }

## Retrieving File Information
使用`FileProvider`来获取文件的类型和大小。

### Retrieve a File's MIME Type
调用`ContentResolver.getType()`获取文件的数据类型（MIME）。一般地，`FileProvider`定义文件的MIME类型为其文件后缀。

    Uri returnUri = returnIntent.getData();
    String mimeType = getContentResolver().getType(returnUri);

### Retrieve a File's Name and Size
`FileProvider` 有默认的`query()`实现：返回一个`Cursor`对象，用来查询含有文件名称和大小的content URI.

默认的实现中有两列：

* `DISPLAY_NAME`: 文件名称，和`File.getName()`返回的数据一致
* `SIZE`: 文件大小(long)，和`File.length()`返回的数据一致

   
    Uri returnUri = returnIntent.getData();
    Cursor returnCursor = getContentResolver().query(returnUri, null, null, null, null);
    int nameIndex = returnCursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
    int sizeIndex = returnCursor.getColumnIndex(OpenableColumns.SIZE);
    returnCursor.moveToFirst();
    TextView nameView = (TextView) findViewById(R.id.filename_Text);
    TextView sizeView = (TextView) findViewById(R.id.filesize_text);
    nameView.setText(returnCursor.getString(nameIndex));
    sizeView.setText(returnCursor.getString(sizeIndex));

# Sharing Files with NFC
使用Android Beam(Android 自己的一个app，仅支持Android 4.0以上) 文件传输功能传输较大的文件.

虽然Android Beam 传输API处理大量的数据，但是Android 4.0中引入的Android Beam NDFF 传输API只能处理少量数据.

## Sending Files to Another Device
使用Android Beam传输大型文件. 完成功能之前，需要申请使用NFC和外部存储的权限，测试你的设备是否支持NFC，提供URI给Android Beam文件传输.

Android Beam文件传输功能有以下需求：

* Android 4.1 以上
* 传输文件必须在外部存储中
* 所传输的文件必须是全局可读的. => 调用`File.setReadable(true, false)`来设置
* 必须提供文件的URI. Android Beam 文件传输不能处理通过`FileProvider.getUriForFile`生成的URI

### Declare Features in the Manifest
#### Request Permissions

* NFC: `<uses-permission android:name="android.permission.NFC" />`
* READ_EXTERNAL_STORAGE: `<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />`

#### Specify the NFC feature
在`<manifest>`下添加`<uses-feature>`标签，并设置`android:required="true"` => 声明如果没有NFC，app将无法工作.

    <uses-feature
        android:name="android.hardware.nfc"
        android:required="true" />

### Test for Android Beam File Transfer Support
如果没有NFC，你的app也可以工作，则设置`required="false"`.

测试是否支持Android Beam文件传输：`PackageManager.hasSystemFeature(FEATURE_NFC)`，然后检查Android版本是否在4.1以上.

如果支持：获取NFC控制器的实例(和NFC硬件进行交互).

    public class MainActivity extends Activity {
        NfcAdapter mNfcAdapter;

        boolean mAndroidBeamAvailable = false;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            if (!PackageManager.hasSystemFeature(PackageManager.FEATURE_NFC)) {
                //Disable NFC feature
            } else if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
                mAndroidBeamAvailable = false;    
            } else {
                mNfcAdapter = NfcAdapter.getDefaultAdapter(this);    
            }
        }
    }

### Create a Callback Method that Provides Files
Android Beam文件传输检测到用户想要给其他的NFC设备传输文件时，系统调用自定义的callback. 在这个callback方法中，返回一个Uri对象数组. Android Beam文件传输将这些文件传输给接收方.

实现`NfcAdapter.CreateBeamUrisCallback` 接口以及其方法`createBeamUris()`.

    public class MainActivity extends Activity {
        private Uri[] mFileUris = new Uri[10];

        private class FileUriCallback implements NfcAdapter.CreateBeamUrisCallback {
            public FileUriCallback() {} 

            @Override
            public Uri[] createBeamUri(NfcEvent event) {
                return mFileUris;    
            }
        }
    }

实现以后，通过`setBeamPushUrisCallback()`激活该callback.

    public class MainActivity extends Activity {
        private FileUriCallback mFileUriCallback;

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            mNfcAdapter = NfcAdapter.getDefaultAdapter(this);
            mFileUriCallback = new FileUriCallback();
            mNfcAdapter.setBeamPushUrisCallback(mFileUriCallback, this);
        }
    }

>你也可以直接提供一组uri给NfcAdapter：`NfcAdapter.setBeamPushUris()`

### Specify the Files to Send

    private Uri[] mFileUris = new Uri[10];
    String transferFile = "transferimage.jpg";
    File extDir = getExternalFileDir(null);
    File requestFile = new File(extDir, transferFile);
    requestFile.setReadable(true, false);
    fileUri = Uri.fromFile(requestFile);
    if (fileUri != null) {
        mFileUris[0] = fileUri;    
    } else {
        Log.e("My Activity", "No File URI available for file.");    
    }

## Receiving Files from Another Device
使用Android Media Scanner查看文件，使用`MediaStore`provider为媒体文件添加内容.

### Respond to a Request to Display Data
当Android Beam文件传输完毕，会发送一个包含`ACTION_VIEW`和MIME Type的Intent.
接收者需要定义`<intent-filter>`来接收对应的唤起事件:

* `<action android:name="android.intent.action.VIEW" />`
* `<category android:name="android.intent.category.CATEGORY_DEFAULT" />`
* `<data android:mimeType="mime-type" />`

**示例**

    <activity
        android:name="com.example.android.nfctransfer.ViewActivity"
        android:label="Android Beam Viewer" >
        ...
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            ...
        </intent-filter>
    </activity>

### Request File Permissions
权限申请：

* 读：`<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />`
* 写：`<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />`

### Get the Directory for Copied Files
Android Beam一次性传输多个文件，传输结束调用Intent的URI指向第一个文件. 然而，你的app可能也会接收到来自其他文件传输的intent(`ACTION_VIEW`). 为了处理接收时间，你需要检查其scheme和authority.

    public class MainActivity extends Activity {
        private File mParentPath;
        private Intent mIntent;

        private void handleViewIntent() {
            mIntent = getIntent();
            String action = mIntent.getAction();

            if (TextUtils.equals(action, Intent.ACTION_VIEW)) {
                Uri beamUri = mIntent.getData();
                if (TextUtils.equals(beamUri.getScheme(), "file") {
                    mParentPath = handleFileUri(beamUri);    
                } else if (TextUtils.equals(beamUri.getScheme(), "content")) {
                    mParentPath = handleContentUri(beamUri);
                }
            }
        }
    }

#### Get the directory from a file URI
如果接收到的intent包含文件的URI，该URI包含文件的绝对路径和文件名称.

    public String handleFileUri(Uri beamUri) {
        String fileName = beamUri.getPath();
        File copiedFile = new File(fileName);
        return copiedFile.getParent();
    }

#### Get the directory from a content URI
如果接收到的intent包含内容的URI，该URI指向存储在`MediaStore`的内容提供者（指向文件夹和文件名称）.你可以通过测试URI的认证值来检测`MediaStore`的content URI.

你也可以通过接收`ACTION_VIEW`intent，包含content URI（除MediaStore之外的content provider）.这种情况下，content URI不包含MediaStore权限值，并且content URI通常不指向目录。

#### Determine the content provider
调用`Uri.getAuthority()`获取URI的认证级别:

* `MediaStore.AUTHORITY`: 该URI用于由MediaStore追踪的一个或多个文件.从MediaStore检索完整的文件名，并从文件名获取目录.
* Any other authority value: 来自其他content provider的content URI. 只可以显示该文件，不能获取文件目录.

为了获取MediaStore的content URI，通过过滤条件：收到的content URI(Uri)和`MediaColumns.DATA`(projection)查找对应的目标.返回的`Cursor`对象包含所有的传输文件的完整路径和文件名称. 

    public String handleContentUri(Uri beamUri) {
        int filenameIndex;
        File copiedFile;
        String fileName;

        if (!TextUtils.equals(beamUri.getAuthority(), MediaStore.AUTHORITY) {
            //other content provider    
        } else {
            String[] projection = { MediaStore.MediaColumns.DATA };
            Cursor pathCursor = getContentResolver().query(beamUri, projection, null, null, null);
            if (pathCursor != null && pathCursor.moveToFirst()) {
                filenameIndex = pathCursor.getColumnIndex(MediaStore.MediaColumns.DATA);
                fileName = pathCursor.getString(filenameIndex);
                copiedFile = new File(fileName);
                return new File(copiedFile.getParent());
            }
        } else {
            return null;    
        }
    }
