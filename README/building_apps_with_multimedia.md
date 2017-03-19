>创建实现用户期望行为的富媒体应用

# Capturing Photos
## Taking Photos Simply
### Taking Photos Simply
使用已有的拍照程序拍照.

#### Request Camera Permission
如果你的app必须有摄像硬件支持，则需要在Google Play上限制其安装渠道.

    <manifest ... >
        <uses-feature android:name="android.hardware.camera"
                      android:required="true" />
    </manifest>

如果你的app对于摄像硬件是可选的，则需要在运行时的时候检测系统是否有摄像硬件：`hasSystemFeature(PackageManager.FEATURE_CAMERA)`

#### Take a Photo with the Camera App
Android请求其他app来完成一个action通常会使用`Intent`.

    static final int REQUEST_IMAGE_CAPTURE = 1;

    private void dispatchTakePictureIntent() {
        Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
            startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);    
        }
    }

### Get the Thumbnail

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
            Bundle extras = data.getExtras();
            Bitmap imageBitmap = (Bitmap) extra.get("data");
            mImageView.setImageBitmap(imageBitmap);
        }    
    }

### Save the Full-size Photo
如果提供一个具体的文件对象，Android拍照软件会保存未压缩的照片. 你必须提供一个完全限定的文件名称.

通常，用户拍摄的任何照片都应该存储在外部公共存储区域，以便所有app进行访问.
`getExternalStorageDirectory()`+`DIRECTORY_PICTURES`: 共享照片的适宜存储位置.

权限请求：`<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />`

然而，如果你只想要照片可被你个人的app访问，你需要改变存储区域：`getExternalFilesDir()`.
在Android 4.3及以下，将照片写入该文件位置需要`WRITE_EXTERNAL_STORAGE`权限.
从Android 4.4开始，不再需要该权限是因为该文件位置不能再被其它app访问，所以你可以声明读权限的最高sdk版本: `<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="18" />`

> 你存储在`getExternalFilesDir()`或者`getFilesDir()`的文件会在用户卸载你的app以后一同被删除。

一旦你决定了存储文件的位置，你需要创建没有冲突的文件名称。以下为一个简单的小例子，演示如何生成独一无二的文件名称.

    String mCurrentPhotoPath;

    private File createImageFile() throws IOException {
        String timeStamp = new SimpleDateFormat("yyyyMMdd_HHmmss").format(new Date());
        String imageFileName = "JPEG_" + timeStamp + "_";
        File storageDir = getExternalFilesDir(Environment.DIRECTORY_PICTURES);
        File image = File.createTempFile(imageFileName, ".jpg", storageDir);

        mCurrentPhotoPath = image.getAbsolutePath();
        return image;
    }

通过结合该方法，你可以通过`Intent`来创建一个图片文件：

    static final int REQUEST_TAKE_PHOTO = 1;

    private void dispatchPictureIntent() {
        Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
            File photoFile = null;
            try {
                photoFile = createImageFile();    
            } catch (IOException ex) {}
           if (photoFile != null) {            
                Uri photoURI = FileProvider.getUriForFile(this, 
                        "com.example.android.fileprovider", photoFile);
                takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoURI);
                startActivityForResult(takePictureIntent, REQUEST_TAKE_PHOTO);    
            }
        }
    }

> `getUriForFile(Context, String, File)` => content://URI.在Android 7.0及以上的版本，使用`file://URI`将会抛出`FileUriExposedException`.因此，我们现在更常用`FileProvider`来生成图片的URI.

配置`FileProvider` => 在配置文件中添加一个provider

    <application>
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.example.android.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.name.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths">
            </meta-data>
        <provider>
    </application>

确保`android:authorities`与`getUriForFile(Context, String, File)`第二个参数匹配. 
在provider定义的meta-data部分，你可以看到provider提供了一个xml文件来配置合格的路径:
    
    <?xml version="1.0" encoding="utf-8"?>
    <paths xmlns:android="http://schemas.android.com/apk/res/android">
        <external-path name="my_images" path="Android/data/com.example.package.name/files/Pictures" />
    </paths>

当调用带有`Environment.DIRECTORY_PICTURES`的`getExternalFilesDir()`时，路径组件会返回定义好的路径.

### Add the Photo to a Gallery
当你通过intent创建了一张照片时，你应该知道图片的存储位置. 对于其他的人来说，访问你创建的照片最简单的方式就是使其可被系统的Media Provider访问.

> 如果你将图片存储在了`getExternalFilesDir()`，media scanner不能访问该图片，因为其只对你的app可见.

    private void galleryAddPic() {
        Intent mediaScanIntent = new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE);
        File f = new File(mCurrentPhotoPath);
        Uri contentUri = Uri.fromFile(f);
        mediaScanIntent.setData(contentUri);
        this.sendBroadcast(mediaScanIntent);
    }

### Decode a Scaled Image
在低内存下管理多个未压缩图片会很复杂. 如果你发现应用在展示了很少的图片就内存溢出，你可以通过将JPEG扩展到已经缩放到与目标视图大小匹配的内存数组，大大减少使用的动态堆的数量.

    private void setPic() {
        int targetW = mImageView.getWidth();
        int targetH = mImageView.getHeight();

        BitmapFactory.Options bmOptions = new BitmapFactory.Options();
        bmOptions.inJustDecodeBounds = true;
        BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
        int photoW = bmOptions.outWidth;
        int photoH = bmOptions.outHeight;

        int scaleFactor = Math.min(photoW/targetW, photoH/targetH);

        bmOptions.inJustDecodeBounds = false;
        bmOptions.isSampleSize = scaleFactor;
        bmOptions.inPurgeable = true;

        Bitmap bitmap = BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
        mImageView.setImageBitmap(bitmap);
    }

## Recording Videos Simply
使用已有的相机应用录制音频.

### Request Camera Permission
在 manifest 文件中使用`<uses-feature>`标签.
    
    <manifest ... >
        <uses-feature android:name="android.hardware.camera"
                      android:required="true" />
    </manifest>

### Record a Video with a Camera App

    static final int REQUEST_VIDEO_CAPTURE = 1;

    private void dispatchTakeVideoIntent() {
        Intent takeVideoIntent = new Intent(MediaStore.ACTION_VIDEO_CAPTURE);
        if (takeVideoIntent.resolveActivity(getPackageManager()) != null) {
            startActivityForResult(takeVideoIntent, REQUEST_VIDEO_CAPTURE);    
        }
    }

### View the Video
在`onActivityResult()`中Android相机应用将会返回视频的Uri.
    
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent intent) {
        if (requestCode == REQUEST_VIDEO_CAPTURE && resultCode == RESULT_OK) {
            Uri videoUri = intent.getData();
            mVideoView.setVideoURI(videoUri);
        }    
    }

## Controlling the Camera
### Open the Camera Object
1. 获取`Camera`对象的一个实例，推荐在`onCreate()`时创建.
2. 在`onResume()`方法中打开camera

如果在调用`Camera.open()`时相机正在被使用，会抛出一个异常.

    Private boolean safeCameraOpen(int id) {
        boolean qOpened = false;

        try {
            releaseCameraAndPreview();
            mCamera = Camera.open(id);
            qOpened = (mCamera != null);
        } catch (Exception e) {
            Log.e(getString(R.string.app_name), "failed to open Camera");
            e.printStackTrace();
        }
        return qOpened;
    }

    private void releaseCameraAndPreview() {
        mPreview.setCamera(null);
        if (mCamera != null) {
            mCamera.release();
            mCamera = null;
        }
    }

在API 9 及以上，camera framework 开始支持多个相机. 如果你调用`open()`不携带任何参数，你将会获取到后置摄像头的`Camera`对象.

### Create the Camera Preview
使用`SurfaceView`显示相机传感器检测到的图像.

#### Preview Class
实现`android.view.SurfaceHolder.Callback` 接口—— 接收相机硬件的图片数据到app上.

    class Preview extends ViewGroup implements SurfaceHolder.Callback {
        
        SurfaceView mSurfaceView;
        SufaceHolder mHolder;

        Preview(Context context) {
            super(context);
            
            mSurfaceView = new SurfaceView(context);
            addView(mSurfaceView);

            mHolder = mSurfaceView.getHolder();
            mHolder.addCallback(this);
            mHolder.setType(SurfaceHolder.SURFACE_TYPE_PUSH_BUFFERS);
        }
    }

#### Set and Start the Preview
Camera实例和其相关的界面展示必须有确切的创建时序，先创建Camera实例.

在下面的代码段中，初始化相机的过程被封装. 任何时间用户做了一些改变相机状态(`setCamera()`)的操作，`Camera.startPreview()`都会被调用. 预览也应该在`surfaceChanged()`中重启.

    public void setCamera(Camera camera) {
        if (mCamera == camera) { return; }

        stopPreviewAndFreeCamera();

        mCamera = camera;

        if (mCamera != null) {
            mCamera.setPreviewDisplay(mHolder);    
        } catch (IOException e) {
            e.printStackTrace();    
        }

        mCamera.startPreview();
    }

### Modify Camera Setting
Camera Settings 更改相机拍摄照片的方式，从缩放级别到曝光补偿. 此示例仅更改预览大小.

    public void surfaceChanged(SurfaceHolder holder, int format, int w, int h) {
        Camera.Parameters parameters = mCamera.getParameters();
        parameters.setPreviewSize(mPreviewSize.width, mPreviewSize.height);
        requestLayout();
        mCamera.setParameters(parameters);

        mCamera.startPreview();
    }

### Set the Preview Orientation
大多数相机应用将显示锁定为横向模式，因为这是相机传感器的自然方向. 此设置不会阻止你拍摄人像模式的照片，因为设备的方向记录在EXIF标头中.`setCameraDisplayOrientation()`允许你修改预览的显示方式，而不影响图像的记录方式. 但是在API 14以下，如果要改变方向，必须先停止预览，然后再更改方向，再重新启动.

### Take a Picture
调用`Camera.takePicture()`拍摄照片在preview已经开始的前提下. 你可以创建`Camera.PictureCallback`和`Camera.ShutterCallback`，并将他们传递给`Camera.takePicture()`.

如果你想延迟拍照，你可以创建`Camera.PreviewCallback`，实现`onPreviewFrame()`. 对于捕捉到的镜头数据，你可以仅捕获所选的预览帧，或设置一个延迟操作来调用`takePicture()`.

### Restart the Preview
    @Override
    public void onClick(View v) {
        switch(mPreviewState) {
            case K_STATE_FROZEN:
                mCamera.startPreview();
                mPreviewState = K_STATE_PREVIEW;
                break;

            default:
                mCamera.takePicture(null, rawCallback, null);
                mPreviewState = K_STATE_BUSY;
        }
        shutterBtnConfig();
    }

### Stop the Preview and Release the Camera
一旦你的app不再使用相机，就应该将其处理掉. 尤其需要释放掉`Camera`对象，否则将可能造成其他app的crash，当然包括你自己的app.

你什么时候需要停止预览释放相机资源？当你的预览窗口被摧毁的时候.

    public void surfaceDestroyed(SurfaceHolder holder) {
        if (mCamera != null) {
            mCamera.stopPreview();
        }    
    }

    private void stopPreviewAndFreeCamera() {
        if (mCamera != null) {
            mCamera.stopPreview();
            mCamera.release();

            mCamera = null;
        }    
    }

# Printing Content
Android 4.4 以上提供了直接显示图片和文件的框架.

## Photos
使用`PrintHelper`类来显示一张图片.

> `PrintHelper`: 来自Android v4 support library

### Print an Image
`PrintHelper`提供一种简单的方式来显示图片. 该类只有一个显示设置:`setScaleMode()`，有如下两种选择：

* SCALE_MODE_FIT: 按照图片的大小平铺在界面上
* SCALE_MODE_FILL: 默认值. 按照屏幕大小平铺整个图片. 

以下代码示例怎样显示图片的全过程.

    private void doPhotoPrint() {
        PrintHelper photoPrinter = new PrintHelper(getActivity());
        photoPrinter.setScaleMode(PrintHelper.SCALE_MODE_FIT);
        Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.droids);
        photoPrint.printBitmap("droids.jpg - test print", bitmap);
    }

## HTML Documents
在Android 4.4以上，`WebView`类做了更新，支持显示HTML内容. 该类允许你加载本地HTML或者从web端下载一个页面并显示.

### Load an HTML Document
创建本地输出视图的主要步骤如下：

1. 在HTML资源家在以后创建一个`WebViewClient`对象
2. 使用`WebView`对象加载HTML资源

示例.

    private WebView mWebView;

    private void doWebViewPrint() {
        WebView webView = new WebView(getActivity());
        webView.setWebViewContent(new WebViewClient() {
            public boolean shouldOverrideUriLoading(Webview webView, String url) {
                return false;    
            }    

            @Override
            public void onPageFinished(WebView view, String url) {
                Log.i(TAG, "page finished loading " + url);
                createWebPrintJob(view);
                mWebView = null;
            }
        });

        String htmlDocument = "<html><body><h1>Test Content</h1><p>Testing, testing, testing ...</p></body></html>";
        webView.loadDataWithBaseURL(null, htmlDocument, "text/HTML", "utf-8", null);
        mWebView = webView;
    }

> **Note** 一定要确认你在`WebViewClient.onPageFinished()`方法被调用以后再做资源输出的工作.  
> **Note** 以上的示例代码保存了对一个WebView对象实例的引用，所以在进行资源输出之前不会对其进行垃圾回收. 你需要确保在自己的实现中也要使用同样的思路来执行代码，否则打印过程可能会失败.

如果要在页面中包含图形，请将图形文件放在项目的`assets/`目录中，并在`loadDataWithBaseURL()`方法的第一个参数中指定基本URL.

    webView.loadDataWithBaseURL("file://android_asset/images", htmlBody,
            "text/HTML", "utf-8", null);

你也可以通过调用`loadUrl()`加载网页.

    webView.loadUrl("http://developer.android.com/about/index.html");

当使用`WebView`来创建输出文件时，有如下几条注意事项：

* 不能添加头惑尾，包括页码等.
* HTML 文档的打印选项不包括打印页面范围的功能，例如：不支持打印10页HTML文档的第2页到第4页
* 一个`WebView`一次只能支持一个输出工作
* 不支持包含CSS打印属性的HTML文档
* 不能在HTML中使用 JavaScript 

> **Note** 包含在布局中的WebView对象的内容也可以在加载文档后打印.

### Create a Print Job
做完以上事情以后，终于要做最后一步工作了：访问`PrintManager`，创建打印适配器，最后创建打印作业.

    private void createWebPrintJob(WebView webView) {
        PrintManager printManager = (PrintManager) getActivity()
                .getSystemService(Context.PRINT_SERVICE);

                PrintDocumentAdapter printAdapter = webView.createPrintDocumentAdapter();

                String jobName = getString(R.string.app_name) + "Document";
                PrintJob printJob = printManager.print(jobName, printAdapter,
                        new PrintAttributes.Builder().build());

                mPrintJobs.add(printJob);
    }   

## Custom Documents
### Connect to the Print Manager
当应用程序直接管理打印过程时，从用户收到打印请求后的第一步是连接到Android打印框架并获取PrintManager类的实例. 以下代码示例显示如何获取打印管理器并开始打印过程.

    private void doPrint() {
        PrintManager printManager = (PrintManager) getActivity().getSystemService(Context.PRINT_SERVICE);
        String jobName = getActivity().getString(R.string.app_name) + " Document";
        printManager.print(jobName, new MyPrintDocumentAdapter(getActivity(), null));
    }

> **Note** `print()` 最后一个参数需要是`PrintAttribute`对象. 你可以使用此参数向打印框架提供提示，并根据先前的打印周期提供预设选项，从而改善用户体验. 你还可以使用此参数设置更适合正在打印的内容的选项，例如在打印处于该方向的照片时将方向设置为横向.

### Create a Print Adapter
输出适配器与Android输出框架交互，处理输出流程. 该过程需要用户选择打印机和打印选项. 在打印过程中，用户可以选择取消打印行为，所以你的打印适配器需要监听和处理取消请求。

`PrintDocumentAdapter` 抽象类有4个主要的回调函数，被用来设计处理打印生命周期.
* `onStart()`: 一次调用. 如果你的应用有任何一次性的准备工作，都可以写在这里. 不是必须实现的方法.
* `onLayout()`: 任何时间用户的行为影响了输出都会被调用. 
* `onWrite()`: 调用将打印的页面转换为要打印的文件. 在每次`onLayout()`被调用后都必须必须被调用至少一次.
* `onFinish()`: 在打印任务结束时被调用一次.

> **Note** 这些适配器的方法在主线程中被调用. 如果你期望在执行这些方法时消耗大量的时间，需要在非UI线程执行实现函数.

#### Compute print document info
在实现`PrintDocumentAdapter`的过程中，app必须能够在打印作业中确认文件类型以及需要打印的页数，和打印页面的大小. 实现`onLayout()`，计算打印作业相关信息，将期望参数通过`PrintDocumentInfo`类传输.

    @Override
    public void onLayout(PrintAttributes oldAttributes,
                        PrintAttributes newAttributes,
                        CancellationSignal cancellationSignal,
                        Bundle metadata) {
        mPdfDocument = new PrintedPdfDocument(getActivity(), newAttributes);
        if (cancellationSignal.isCancelled()) {
            callback.onLayoutCancelled();
            return;
        }

        int pages = computPageCount(newAttributes);
        if (pages > 0) {
            PrintDocumentInfo info = new PrintDocumentInfo()
                        .Builder("print_output.pdf")
                        .setContentType(PrintDocumentInfo.CONTENT_TYPE_DOCUMENT)
                        .setPageCount(pages)
                        .build();
            callback.onLayoutFinished(info, true);
        } else {
            callback.onLayoutFailed("Page count calculation failed.");
        }
    }

执行`onLayout()`方法会有三种可能的执行结果：完成、取消或者失败. 你必须通过实现`PrintDocumentAdapter.LayoutResultCallback` 来处理执行结果.

> **Note** `onLayoutFinished()` 方法的布尔参数指示自上次请求以来布局内容是否实际已更改. 正确设置此参数允许打印框架避免不必要地调用`onWrite()`方法，缓存先前写入的打印文档并提高性能.

以下的代码示例展示如何根据打印方向计算页面页数：

    private int computePageCount(PrintAttributes printAttributes) {
        int itemsPerPage = 4;

        MediaSize pageSize = printAttributes.getMediaSize();
        if (!pageSize.isPortrait()) {
            itemsPerPage = 6;    
        }

        int printItemCount = getPrintItemCount();

        return (int) Math.ceil(printItemCount / itemsPerPage);
    }

#### Write a print document file
`PrintDocumentAdapter.onWrite()` 在要进行打印作业时调用. 
`onWrite(PageRange[] pageRanges, ParcelFileDescriptor destination, CancellationSignal cancellationSignal, WriteResultCallback callback)`

当打印任务完成时，需要调用`callback.onWriteFinished()`.

> **Note** 每次`onLayout()`被调用以后，`onWrite()`都会被调用至少一次. 因此如果输出内容没有改变的话，需要设置`onLayoutFinished()`为`false`，以避免不必要地重新打印.

> **Note** `onLayoutFinished()`的boolean型参数指示输出内容在上次调用时有没有改变.

    @Override
    public void onWrite(final PageRange[] pageRanges,
                        final ParcelFileDescriptor destination,
                        final CancellationSignal cancellationSignal,
                        final WriteResultCallback callback) {
        for (int i = 0; i < totalPages; i ++) {
            if (containsPage(pageRanges, i)) {
                writtenPagesArray.append(writtenPagesArray.size(), i);
                PdfDocument.Page page = mPdfDocument.startPage(i);

                if (cancellationSignal.isCancelled()) {
                    callback.onWriteCancelled();
                    mPdfDocument.close();
                    mPdfDocument = null;
                    return;
                }

                drawPage(page);

                mPdfDocument.finishPage(page);
            }    
        }

        try {
            mPdfDocument.writeTo(new FileOutputStream(
                    destination.getFileDescriptor()));
        } catch (IOException e) {
            callback.onWriteFailed(e.toString());
            return;
        } finally {
            mPdfDocument.close();
            mPdfDocument = null;
        }

        PageRange[] writtenPages = computeWrittenPages();
        callback.onWriteFinished(writtenPages);
        ...
    }

`PrintDocumentAdapter.WriteResultCallback` 监听`onWrite()` 结果.

### Drawing PDF Page Content
你的app在打印时必须生成一个PDF文件，并将其传输给Android打印框架. 你可以使用`PrintedPdfDocument`来收入能够承认那个PDF文件.

`PrintedPdfDocument`使用`Canvas`绘出PDF页面.

    private void drawPage(PdfDocument.Page page) {
        Canvas canvas = page.getCanvas();

        int titleBaseLine = 72;
        int leftMargin = 54;

        Paint paint = new Paint();
        paint.setColor(Color.BLACK);
        paint.setTextSize(36);
        canvas.drawText("Test Title", leftMargin, titleBaseLine, paint);

        paint.setTextSize(11);
        canvas.drawText("Test paragragh", leftMargin, titleBaseLine + 25, paint);

        paint.setColor(Color.BLUE);
        canvas.drawRect(100, 100, 172, 172, paint);
    }
