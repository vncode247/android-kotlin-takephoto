# Android-Kotlin Take photo step by step
### 1. Import library if not exists
```.gradle
// 
implementation 'com.jakewharton:butterknife:8.8.1'
kapt 'com.jakewharton:butterknife-compiler:8.8.1'
// 
implementation 'com.facebook.fresco:fresco:1.9.0'
// 
implementation 'com.karumi:dexter:4.2.0'
```

### 2. Init Fresco
```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Fresco.initialize(this)
    }
}
```

### 3. Create file res/xml/file_paths.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-files-path
        name="intent_images"
        path="Pictures"
        />
</paths>
```

### 4. AndroidManifest.xml add
```xml
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<application
    <....>
    <provider
        android:authorities="${applicationId}.fileprovider"
        android:name="android.support.v4.content.FileProvider"
        android:exported="false"
        android:grantUriPermissions="true">
        <meta-data android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths" />
    </provider>
</application>
```
### 5. activity_main.xml
```xml
<com.facebook.drawee.view.SimpleDraweeView xmlns:fresco="http://schemas.android.com/apk/res-auto"
    android:id="@+id/imgAvatar"
    android:layout_width="150dp"
    android:layout_height="150dp"
    android:layout_gravity="center_horizontal"
    fresco:placeholderImage="@drawable/avatar_default"
    fresco:roundAsCircle="true"
    fresco:roundingBorderColor="@android:color/darker_gray"
    fresco:roundingBorderWidth="2dp" />
```
### 6. MainActivity.kt
```kotlin
class MainActivity : AppCompatActivity() {

    val CAMERA_REQUEST_CODE = 0
    lateinit var imageFilePath: String
    @BindView(R.id.imgAvatar) lateinit var imgAvatar: SimpleDraweeView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        ButterKnife.bind(this)

    }

    @OnClick(R.id.imgAvatar)
    fun onClickImgAvatar() {
        Dexter.withActivity(this)
                .withPermissions(Manifest.permission.WRITE_EXTERNAL_STORAGE, Manifest.permission.READ_EXTERNAL_STORAGE, Manifest.permission.CAMERA)
                .withListener(object : MultiplePermissionsListener {
                    override fun onPermissionRationaleShouldBeShown(permissions: MutableList<PermissionRequest>?, token: PermissionToken?) {
                        AlertDialog.Builder(this@MainActivity)
                                .setTitle("getString(R.string.storage_permission_rationale_title)")
                                .setMessage("getString(R.string.storage_permission_rationale_message)")
                                .setNegativeButton(android.R.string.cancel, DialogInterface.OnClickListener {
                                    dialogInterface, i ->
                                    dialogInterface.dismiss()
                                    token?.cancelPermissionRequest()
                                })
                                .setPositiveButton(android.R.string.ok, DialogInterface.OnClickListener {
                                    dialogInterface, i ->
                                    dialogInterface.dismiss()
                                    token?.continuePermissionRequest()
                                })
                                .show()
                    }

                    override fun onPermissionsChecked(report: MultiplePermissionsReport?) {
                        if (report?.areAllPermissionsGranted()!!) {

                            try {
                                val imageFile = createImageFile()
                                val callCameraIntent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
                                if(callCameraIntent.resolveActivity(packageManager) != null) {
                                    val authorities = packageName + ".fileprovider"
                                    val imageUri = FileProvider.getUriForFile(this@MainActivity, authorities, imageFile)
                                    callCameraIntent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri)
                                    startActivityForResult(callCameraIntent, CAMERA_REQUEST_CODE)
                                }
                            } catch (e: IOException) {
                                Toast.makeText(this@MainActivity, "Could not create file!", Toast.LENGTH_SHORT).show()
                            }
                        }

                    }

                }
                ).check()
    }



    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        when(requestCode) {
            CAMERA_REQUEST_CODE -> {
                if (resultCode == Activity.RESULT_OK) {
                    val imgUri = Uri.Builder()
                            .scheme(UriUtil.LOCAL_FILE_SCHEME)
                            .path(imageFilePath)
                            .build()
                    imgAvatar.setImageURI(imgUri, this)
                }
            }
            else -> {
                Toast.makeText(this, "Unrecognized request code", Toast.LENGTH_SHORT).show()
            }
        }
    }

    @Throws(IOException::class)
    fun createImageFile(): File {
        val timeStamp: String = SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault()).format(Date())
        val imageFileName: String = "JPEG_" + timeStamp + "_"
        val storageDir: File = getExternalFilesDir(Environment.DIRECTORY_PICTURES)
        if(!storageDir.exists()) storageDir.mkdirs()
        val imageFile = File.createTempFile(imageFileName, ".jpg", storageDir)
        imageFilePath = imageFile.absolutePath
        return imageFile
    }

}
```
