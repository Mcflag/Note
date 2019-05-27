# 一、原生+injectJS+JSSDK方案

injectJS是直接放在Android和iOS项目中的js文件，由它调起系统的各种能力以及原生封装的方法。原生系统只与injectJS互相调用，而远程界面加入的JSSDK只与injectJS互相调用。

这样做的话远程界面就不用考虑Android和iOS的调用差异，因为这个差异已经被injectJS抹平了，另外还能够隐藏一些具体实现细节。

## 1. 客户端

客户端主要做四件事，封装端能力、注入injectJS、全局路由（包括调起微信和支付宝）、对一些业务逻辑进行封装和增强。

1. 由客户端直接封装WebView使用，将系统提供的功能封装为方法，供injectJS调用，这样做更加灵活可控。缺点是Android和iOS都需要一些时间封装各自端上的基本能力，同样的iOS和Android调用的差异在injectJS中抹平。

2. 注入injectJS。在每次加载页面时，由原生方法从WebView注入injectJS文件，这样injectJS里面的方法就可以供页面上导入JSSDK调用。

3. 全局路由。在跳转或者重定向的时候检查链接，如果链接是符合我们定义的schema，就通过路由调用。这个时候跳转的安全性可以做一个面向切面的方法，跳转时必须检查token之类的。

4. 对业务逻辑封装成通用方法，供Web页面调用。

## 2. injectJS

分为Android和iOS两个文件，是分别打包进项目的本地JS文件，两个文件的区别在于要用不同的方式调用各自封装的系统方法。另外就是对外调用封装成一套统一的API，由JSSDK调用。

## 3. JSSDK

JSSDK才是页面引入的JS文件，与injectJS互相调用，不与原生系统本身产生任何交互。

# 二、端能力封装

## 1. 原生的端能力

调用原生的端能力或者原生的逻辑需要定义操作标识。比如定义扫码为ScanQRCode。

为了方便Web调用，Android和iOS统一封装一个门面方法。门面方法传入的包括方法名、传入的参数、回调方法名。比如按JS可以如下方式调用：

	window.myJsBridge.execute("ScanQRCode", args, callback)
	function callback(error, data)

回调方法里面包含错误信息或者成功的数据，传入的参数是json字符串，便于解析。

# 三、路由

## 1. 路由定义

针对应用内路由的格式可以使用如下链接：

	_myhybrid://action/method?arg1=xxx&arg2=xxx&ios_version=xxx&andr_version=xxx&upgrade=1/0&callback=xxx&token=xxx

前面的_myhybrid是schema，通过拦截请求，解析这个scheme协议，符合约定规则的就扔给Native的方法处理。使用下划线开头是为了不会有冲突，不会被识别为别的URI。

action和method是对应的不同模块的不同方法。args可选，是调用所需的参数。

这里可能还需要考虑到的一些情况：

* ios_version/android_version，本协议从哪个版本开始支持的，低版本不支持则忽略。
* upgrade，是否引导用户升级。
* callback，异步回调的函数，callback里需要传入错误信息和成功的返回。
* token，路由的令牌。跳转之前先检查令牌是否合法，如果合法才跳转，如果不合法就不能跳转。

## 2. 原生界面路由

Android原生界面通过Alibaba开源的Arouter路由库实现，简单的使用步骤：

1. 集成Arouter库，在Application中初始化。
2. 在相应的Activity上加上`@Route(path = "/test/1")`注解。
3. 在原生代码中调用：

	// 1. 应用内简单的跳转
	ARouter.getInstance().build("/test/1").navigation();

	// 2. 跳转并携带参数
	ARouter.getInstance().build("/test/1")
			.withLong("key1", 666L)
			.withString("key3", "888")
			.navigation();

在Activity上添加的注解路径，就是我们约定好的模块和方法。

## 3. web网页跳转

如果读入的链接不是自定义的schema，而是http或者https，就认为是网址链接，用WebView直接打开网址。

## 4. 网页和原生调起支付宝及微信

支付宝和微信的schema进行特例处理，由app封装方法供Web调用打开手机端的支付宝或者微信。

# 四、JSSDK的设计

JSSDK的设计包括两部分：一个是应用内内置的一个js，可以称为injectJS，他的主要作用是封装JSBridge逻辑，通过随版更新实现减少端能力的版本分裂，降低整个sdk的代码复杂性。

当客户端加载一个页面的时候，由客户端在适当的时机注入到webview内执行，执行后的代码就会有给webview增加js方法，例如微信的 _WeixinJSBridge。

另一个就是云端的js，即实际暴露出来的js，称为JSSDK，通过script外链引入，例如wx.js，这个js文件通过和inject.js进行交换，完成端能力的调用、鉴权和客户端事件监听等操作。

对于iOS和android的差异可以通过两边集成不同的injectJS来抹平，然后通过单一的js进行业务操作。

对于客户端来说，injectJS的注入时机，虽然是越早越好，Android里可以在onPageFinished(WebView view, String url)或者onPageStarted (WebView view, String url, Bitmap favicon)方法中注入js文件。

在iOS中也有对应的时间点：webViewDidFinishLoad 和 didCreateJavaScriptContext。

Android中加载JS的方法有webview.loadUrl("javascript:xxx")或者webView.evaluateJavascript()方法两种。如果H5设置了防JS注入的话，loadUrl就无效。这种情况使用evaluateJavascript方法。

# 五、安全性

1. 对于Android的WebView来说，用@JavaScriptInterface标记Java方法或者使用JsBridge库，防止暴露所有的可供JS调用的方法。
2. WebView禁止代理，防止抓包或者代理访问。
3. 路由时通过面向切面编程验证跳转令牌，禁止非法的跳转链接。
4. JSSDK包括两个JS文件，一个本地JS，一个云端暴露的JS文件，两个JS文件中进行鉴权和端能力调用。

# 六、示例代码

## HTML及JS

```html
<html>
<head>
    <meta content="text/html; charset=utf-8" http-equiv="content-type">
    <title>
        js调用java
    </title>
</head>

<body>
<p>
    <xmp id="show">
    </xmp>
</p>
<p>
    <input type="file" value="打开文件"/>
</p>
<p>
    <input type="button" value="测试一下" onclick="test()"/>
    <input type="button" value="得到会员信息" onclick="test2()"/>
</p>
</body>
<script>
        function test() {
            testDiv()
        }

        function test2(){
            getInfo()
        }
</script>
</html>
```

## injectJS

```javascript
function testDiv() {
    document.getElementById("show").innerHTML =
    window.myJsBridge.execute("ScanQRCode","2","callback")
}

function getInfo(){
    window.myJsBridge.execute("GetInfo", "1", "infoCallback")
}

function callback(error, data){
    if(error.length>0){
        alert("error = "+error);
    }else{
        document.getElementById("show").innerHTML = data
    }
}

function infoCallback(error, data){
    if(error.length>0){
        alert("error = "+error);
    }else{
        document.getElementById("show").innerHTML = JSON.stringify(data)
    }
}
```

## 封装的WebView

示例包括调取扫码，上传文件两个功能。

```kotlin
abstract class BaseWebViewActivity : AppCompatActivity() {

    private val TAG = BaseWebViewActivity::class.java.simpleName
    private val REQUEST_CODE_TAKEPHOTO = 0x666
    private val REQUEST_CODE_OPENALBUM = 0x777
    private val REQUEST_CODE_SCAN = 0x888
    private var isJump = false
    private var isError: Boolean = false

    private lateinit var mUrl: String

    private lateinit var mPopupWindow: PopupWindow
    private lateinit var mPopupView: View
    private lateinit var photoTv: TextView
    private lateinit var albumTv: TextView
    //图片的Uri
    private var mImageUri: Uri? = null
    private var mUploadMessage: ValueCallback<Uri>? = null
    private var mUploadCallbackAboveL: ValueCallback<Array<Uri?>>? = null

    private var callbackName: String = ""
    private var jsString: String = ""

    protected fun back() {
        if (webview.canGoBack()) {
            webview.goBack()
        } else {
            finish()
        }
    }

    protected fun close() {
        finish()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_base_webview)
        initView()
        getInjectJS()
        initPopupWindow()
        webSettings()
        loadJs()
        webClient()
        loadUrl()
    }

    protected fun initView() {
        submit.setOnClickListener {
            isError = false
            progress.visibility = View.VISIBLE
            failure_layout.visibility = View.GONE
            webview.reload()
        }
    }

    @SuppressLint("AddJavascriptInterface")
    protected fun loadJs() {
        webview.addJavascriptInterface(AndroidToJs(), "myJsBridge")
    }

    protected fun loadUrl() {
        isError = false
        progress.visibility = View.VISIBLE
        webview.visibility = View.INVISIBLE
        mUrl = getUrl()
        webview.loadUrl(mUrl)
    }

    protected fun webClient() {
        webview.webViewClient = CustomWebViewClient()
        webChromeClient()
    }

    @SuppressLint("NewApi", "SetJavaScriptEnabled")
    protected fun webSettings() {
        WebView.setWebContentsDebuggingEnabled(BuildConfig.DEBUG)

        val settings = webview.settings

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            settings.mixedContentMode = WebSettings.MIXED_CONTENT_COMPATIBILITY_MODE
        }
        settings.javaScriptEnabled = true
        settings.domStorageEnabled = true
        settings.databaseEnabled = true
        settings.allowContentAccess = true
        settings.allowUniversalAccessFromFileURLs = true
        settings.allowFileAccess = true
    }

    protected abstract fun getUrl(): String

    inner class CustomWebViewClient : WebViewClient() {

        override fun shouldOverrideUrlLoading(view: WebView, url: String?): Boolean {
            if (url == null) return false

            // ------  对alipays:相关的scheme处理 -------
            if (url.startsWith("alipays:") || url.startsWith("alipay")) {
                try {
                    startActivity(Intent("android.intent.action.VIEW", Uri.parse(url)))
                } catch (e: Exception) {
                    AlertDialog.Builder(this@BaseWebViewActivity)
                        .setMessage("未检测到支付宝客户端，请安装后重试。")
                        .setPositiveButton("立即安装") { dialog, which ->
                            val alipayUrl = Uri.parse("https://d.alipay.com")
                            startActivity(Intent("android.intent.action.VIEW", alipayUrl))
                        }.setNegativeButton("取消", null).show()
                }

                return true
            }
            // ------- 处理结束 -------

            if (!(url.startsWith("http") || url.startsWith("https"))) {
                return true
            }

            webview.loadUrl(url)
            return true
        }

        override fun onPageStarted(view: WebView, url: String, favicon: Bitmap?) {
            super.onPageStarted(view, url, favicon)
            if (jsString.isNotEmpty() && Build.VERSION.SDK_INT >= 19) {
                webview.evaluateJavascript(jsString) { value ->
                    Log.d(TAG, "value=$value")
                }
            }

        }

        override fun onPageFinished(view: WebView, url: String) {
            super.onPageFinished(view, url)
            if (jsString.isNotEmpty() && Build.VERSION.SDK_INT < 19) {
                webview.loadUrl("javascript:$jsString")
            }
//            if (webview.canGoBack()) {
//                mTvTopbarWebClose.setVisibility(View.VISIBLE)
//            } else {
//                mTvTopbarWebClose.setVisibility(View.INVISIBLE)
//            }
        }

        override fun onReceivedError(view: WebView, request: WebResourceRequest, error: WebResourceError) {
            super.onReceivedError(view, request, error)
            isError = true
            failure_layout.visibility = View.VISIBLE
        }

        override fun onReceivedSslError(view: WebView, handler: SslErrorHandler, error: SslError) {
            super.onReceivedSslError(view, handler, error)
            handler.proceed()
        }
    }

    inner class MyWebChromeClient : WebChromeClient() {

        // For Android 3.0-
        fun openFileChooser(uploadMsg: ValueCallback<Uri>) {
            mUploadMessage = uploadMsg
            showPopupWindow()
        }

        // For Android 3.0+
        fun openFileChooser(uploadMsg: ValueCallback<Uri>, acceptType: String) {
            mUploadMessage = uploadMsg
            showPopupWindow()
        }

        //For Android 4.1
        fun openFileChooser(uploadMsg: ValueCallback<Uri>, acceptType: String, capture: String) {
            mUploadMessage = uploadMsg
            showPopupWindow()
        }

        // For Android 5.0+
        override fun onShowFileChooser(
            webView: WebView,
            filePathCallback: ValueCallback<Array<Uri?>>,
            fileChooserParams: WebChromeClient.FileChooserParams
        ): Boolean {
            mUploadCallbackAboveL = filePathCallback
            showPopupWindow()
            return true
        }

        override fun onProgressChanged(view: WebView, newProgress: Int) {
            super.onProgressChanged(view, newProgress)
            if (newProgress == 100) {
                progress.visibility = View.GONE
                if (!isError) {
                    webview.visibility = View.VISIBLE
                }
            }
        }
    }

    override fun onResume() {
        super.onResume()
        isJump = false
    }

    protected fun webChromeClient() {
        webview.webChromeClient = MyWebChromeClient()
    }

    /**
     * 初始化PopupWindow
     */
    private fun initPopupWindow() {
        mPopupView = LayoutInflater.from(this).inflate(R.layout.layout_popupwindow_merchantloan, null)
        mPopupWindow =
            PopupWindow(mPopupView, LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT)
        mPopupWindow.setBackgroundDrawable(ColorDrawable(0x00000000))
        mPopupWindow.isOutsideTouchable = true
        mPopupWindow.isFocusable = true
        mPopupWindow.setOnDismissListener {
            if (!isJump) {
                clearUploadCallback()
            }
            setWindowBackground(false)
        }

        photoTv = mPopupView.findViewById(R.id.takePhoto)
        albumTv = mPopupView.findViewById(R.id.openAlbum)

        photoTv.setOnClickListener { cameraPermitted() }

        albumTv.setOnClickListener { externalStoragePermitted() }
    }

    /**
     * 弹出PopupWindow
     */
    private fun showPopupWindow() {
        setWindowBackground(true)
        mPopupWindow.showAtLocation(webview, Gravity.BOTTOM, 0, 0)
    }

    /**
     * 改背景色
     *
     * @param show
     */
    private fun setWindowBackground(show: Boolean) {
        val layoutParams = window.attributes
        layoutParams.alpha = if (show) 0.7f else 1.0f
        window.attributes = layoutParams
    }

    protected fun cameraPermitted() {
        isJump = true
        mPopupWindow.dismiss()
        doTakePhoto()
    }

    protected fun externalStoragePermitted() {
        isJump = true
        mPopupWindow.dismiss()
        doOpenAlbum()
    }

    /**
     * 跳到系统拍照
     */
    private fun doTakePhoto() {
        val takePhotoIntent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
        if (takePhotoIntent.resolveActivity(packageManager) != null) {

            val dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES).absoluteFile
            val imageFile = File(dir, getImageFileName())

            if (!imageFile.parentFile.exists()) {
                imageFile.parentFile.mkdirs()
            }

            mImageUri = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
                /*7.0以上要通过FileProvider将File转化为Uri*/
                FileProvider.getUriForFile(this, "$packageName.fileProvider", imageFile)
            } else {
                /*7.0以下则直接使用Uri的fromFile方法将File转化为Uri*/
                Uri.fromFile(imageFile)
            }
            takePhotoIntent.putExtra(MediaStore.EXTRA_OUTPUT, mImageUri)//将用于输出的文件Uri传递给相机
            startActivityForResult(takePhotoIntent, REQUEST_CODE_TAKEPHOTO)//打开相机
        }
    }

    /**
     * 跳到系统相册
     */
    private fun doOpenAlbum() {
        val openAlbumIntent = Intent(Intent.ACTION_GET_CONTENT)
        openAlbumIntent.type = "image/*"
        startActivityForResult(openAlbumIntent, REQUEST_CODE_OPENALBUM)//打开相册
    }

    private fun getImageFileName(): String {
        return "my.jpg"
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        if (requestCode == REQUEST_CODE_OPENALBUM || requestCode == REQUEST_CODE_TAKEPHOTO) {
            if (mUploadMessage == null && mUploadCallbackAboveL == null) {
                return
            }

            //如果取消 则置为空
            if (resultCode != RESULT_OK) {
                clearUploadCallback()
            }

            if (resultCode == RESULT_OK) {
                when (requestCode) {
                    REQUEST_CODE_TAKEPHOTO//拍照成功
                    ->
                        //拍照成功后刷新相册
                        refreshAlbum(mImageUri)
                    REQUEST_CODE_OPENALBUM//选择照片成功
                    -> mImageUri = data?.data
                }
            }
            //上传文件
            if (mUploadMessage != null) {
                mUploadMessage?.onReceiveValue(mImageUri)
                mUploadMessage = null
            }
            if (mUploadCallbackAboveL != null) {
                mUploadCallbackAboveL?.onReceiveValue(arrayOf(mImageUri))
                mUploadCallbackAboveL = null
            }
        }
        if (requestCode == REQUEST_CODE_SCAN && resultCode == RESULT_OK) {
            if (data != null) {
                var content = data.getStringExtra(Constant.CODED_CONTENT)
                webview.post {
                    if (callbackName.isNotEmpty()) {
                        webview.loadUrl("javascript:$callbackName(\"\",$content)")
                    }
                }
            }
        }
    }

    /**
     * 清空回调，否则下次调不起来
     */
    private fun clearUploadCallback() {
        if (mUploadMessage != null) {
            mUploadMessage?.onReceiveValue(null)
            mUploadMessage = null
        }

        if (mUploadCallbackAboveL != null) {
            mUploadCallbackAboveL?.onReceiveValue(null)
            mUploadCallbackAboveL = null
        }
    }

    /**
     * 刷新图库
     *
     * @param uri
     */
    private fun refreshAlbum(uri: Uri?) {
        val mediaScanIntent = Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE)
        mediaScanIntent.data = uri
        sendBroadcast(mediaScanIntent)
    }

    private fun getInjectJS() {
        try {
            val inject = this.assets.open("inject.js")
            val buff = ByteArray(1024)
            val fromFile = ByteArrayOutputStream()
            do {
                val numRead = inject.read(buff)
                if (numRead <= 0) {
                    break
                }
                fromFile.write(buff, 0, numRead)
            } while (true)
            jsString = fromFile.toString()
            inject.close()
            fromFile.close()
        } catch (e: IOException) {
            e.printStackTrace()
        }
    }

    fun scanQRCode(args: String): String {
        val intent = Intent(this@BaseWebViewActivity, CaptureActivity::class.java)
        startActivityForResult(intent, REQUEST_CODE_SCAN)
        Log.d("alalalalal", "result:something  $args")
        return "{success:true, data:\"传入的数据为：$args\"}"
    }

    fun getInfo(args: String) {
        var info = "{id:1,name:\"aaaa\"}"
        webview.post {
            if (callbackName.isNotEmpty()) {
                webview.loadUrl("javascript:$callbackName(\"\",$info)")
            }
        }
    }

    inner class AndroidToJs : Any() {

        @JavascriptInterface
        fun execute(action: String, args: String, callback: String) {
            callbackName = callback
            when (action) {
                "ScanQRCode" -> scanQRCode(args)
                "GetInfo" -> getInfo(args)
                else -> webview.post {
                    if (callbackName.isNotEmpty()) {
                        webview.loadUrl("javascript:$callbackName(\"无效的操作\", \"\")")
                    }
                }
            }
        }
    }
}
```

MainActivity直接打开链接。

```kotlin
class MainActivity : BaseWebViewActivity() {

    override fun getUrl(): String = "file:///android_asset/demo.html"
}
```


