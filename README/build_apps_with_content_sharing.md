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
# Sharing Files
# Sharing Files with NFC
