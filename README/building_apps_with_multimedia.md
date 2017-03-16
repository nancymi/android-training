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
## Photos
## HTML Documents
## Custom Documents
