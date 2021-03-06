# 连接到网络

> 编写:[kesenhoo](https://github.com/kesenhoo) - 原文:<http://developer.android.com/training/basics/network-ops/connecting.html>

这一课会演示如何实现一个简单的连接到网络的程序。它提供了一些你在创建即使最简单的网络连接程序时也应该遵循的最佳示例。请注意，想要执行本课的网络操作首先需要在程序的manifest文件中添加以下permissions:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

## 1)选择一个HTTP Client
大多数连接网络的Android app会使用HTTP来发送与接受数据。Android提供了两种HTTP clients: [HttpURLConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html) 与Apache [HttpClient](http://developer.android.com/reference/org/apache/http/client/HttpClient.html)。二者均支持HTTPS ，流媒体上传和下载，可配置的超时, IPv6 与连接池（connection pooling）. **推荐从Android 2.3 Gingerbread版本开始使用 HttpURLConnection** 。关于这部分的更多详情，请参考 [Android's HTTP Clients](http://android-developers.blogspot.com/2011/09/androids-http-clients.html)。

## 2)检查网络连接
在你的app尝试连接网络之前，应通过函数[getActiveNetworkInfo](http://developer.android.com/reference/android/net/ConnectivityManager.html#getActiveNetworkInfo())和[isConnected](http://developer.android.com/reference/android/net/NetworkInfo.html#isConnected())检测当前网络是否可用。请注意，设备可能不在网络覆盖范围内，或者用户可能关闭Wi-Fi与移动网络连接。关于这部分的更多详情，请参考 [管理网络的使用情况](managing.html)

```java
public void myClickHandler(View view) {
    ...
    ConnectivityManager connMgr = (ConnectivityManager)
        getSystemService(Context.CONNECTIVITY_SERVICE);
    NetworkInfo networkInfo = connMgr.getActiveNetworkInfo();
    if (networkInfo != null && networkInfo.isConnected()) {
        // fetch data
    } else {
        // display error
    }
    ...
}
```

## 3)在一个单独的线程中执行网络操作
网络操作会遇到不可预期的延迟。为了避免造成不好的用户体验，总是在UI线程之外执行网络操作。[AsyncTask](http://developer.android.com/reference/android/os/AsyncTask.html) 类提供了一种简单的方式来处理这个问题。关于更多的详情，请参考:[Multithreading For Performance](http://android-developers.blogspot.com/2010/07/multithreading-for-performance.html).

在下面的代码示例中，myClickHandler() 方法会执行new DownloadWebpageTask().execute(stringUrl).DownloadWebpageTask是AsyncTask的子类，它实现了下面两个方法:

* [doInBackground()](http://developer.android.com/reference/android/os/AsyncTask.html) 执行 downloadUrl()方法。它以Web页面的URL作为参数，方法downloadUrl() 获取并处理网页返回的数据，执行完毕后，返回一个结果字符串。
* [onPostExecute()](http://developer.android.com/reference/android/os/AsyncTask.html) 接受结果字符串并把它显示到UI上。

```java
public class HttpExampleActivity extends Activity {
    private static final String DEBUG_TAG = "HttpExample";
    private EditText urlText;
    private TextView textView;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        urlText = (EditText) findViewById(R.id.myUrl);
        textView = (TextView) findViewById(R.id.myText);
    }

    // When user clicks button, calls AsyncTask.
    // Before attempting to fetch the URL, makes sure that there is a network connection.
    public void myClickHandler(View view) {
        // Gets the URL from the UI's text field.
        String stringUrl = urlText.getText().toString();
        ConnectivityManager connMgr = (ConnectivityManager)
            getSystemService(Context.CONNECTIVITY_SERVICE);
        NetworkInfo networkInfo = connMgr.getActiveNetworkInfo();
        if (networkInfo != null && networkInfo.isConnected()) {
            new DownloadWebpageText().execute(stringUrl);
        } else {
            textView.setText("No network connection available.");
        }
    }

     // Uses AsyncTask to create a task away from the main UI thread. This task takes a
     // URL string and uses it to create an HttpUrlConnection. Once the connection
     // has been established, the AsyncTask downloads the contents of the webpage as
     // an InputStream. Finally, the InputStream is converted into a string, which is
     // displayed in the UI by the AsyncTask's onPostExecute method.
     private class DownloadWebpageText extends AsyncTask {
        @Override
        protected String doInBackground(String... urls) {

            // params comes from the execute() call: params[0] is the url.
            try {
                return downloadUrl(urls[0]);
            } catch (IOException e) {
                return "Unable to retrieve web page. URL may be invalid.";
            }
        }
        // onPostExecute displays the results of the AsyncTask.
        @Override
        protected void onPostExecute(String result) {
            textView.setText(result);
       }
    }
    ...
}
```

上面这段代码的事件顺序如下:

1.当用户点击按钮时调用myClickHandler()，app将指定的URL传给[AsyncTask](http://developer.android.com/reference/android/os/AsyncTask.html)的子类DownloadWebpageTask.
2.[AsyncTask](http://developer.android.com/reference/android/os/AsyncTask.html)的[doInBackground()](http://developer.android.com/reference/android/os/AsyncTask.html#doInBackground(Params...))方法调用downloadUrl()方法.
3.downloadUrl()以一个URL字符串作为参数，并用它创建一个[URL](http://developer.android.com/reference/java/net/URL.html)对象.
4.这个[URL](http://developer.android.com/reference/java/net/URL.html)对象被用来创建一个[HttpURLConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html).
5.一旦建立连接，[HttpURLConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html)对象将获取Web页面的内容并得到一个[InputStream](http://developer.android.com/reference/java/io/InputStream.html).
6.[InputStream](http://developer.android.com/reference/java/io/InputStream.html)被传给readIt()方法，该方法将流转换成字符串.
7.最后，[AsyncTask](http://developer.android.com/reference/android/os/AsyncTask.html)的[onPostExecute()](http://developer.android.com/reference/android/os/AsyncTask.html#onPostExecute(Result))方法将字符串展示在main activity的UI上.

## 4)连接并下载数据
在执行网络交互的线程里面，你可以使用 [HttpURLConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html) 来执行一个 GET 类型的操作并下载数据。在你调用 connect()之后，你可以通过调用getInputStream()来得到一个包含数据的[InputStream](http://developer.android.com/reference/java/io/InputStream.html) 对象。

在下面的代码示例中， [doInBackground()](http://developer.android.com/reference/android/os/AsyncTask.html#doInBackground(Params...)) 方法会调用downloadUrl(). 这个 downloadUrl() 方法使用给予的URL，通过 [HttpURLConnection](http://developer.android.com/reference/java/net/HttpURLConnection.html) 连接到网络。一旦建立连接后，app就会使用 getInputStream() 来获取包含数据的[InputStream](http://developer.android.com/reference/java/io/InputStream.html)。

```java
// Given a URL, establishes an HttpUrlConnection and retrieves
// the web page content as a InputStream, which it returns as
// a string.
private String downloadUrl(String myurl) throws IOException {
    InputStream is = null;
    // Only display the first 500 characters of the retrieved
    // web page content.
    int len = 500;

    try {
        URL url = new URL(myurl);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setReadTimeout(10000 /* milliseconds */);
        conn.setConnectTimeout(15000 /* milliseconds */);
        conn.setRequestMethod("GET");
        conn.setDoInput(true);
        // Starts the query
        conn.connect();
        int response = conn.getResponseCode();
        Log.d(DEBUG_TAG, "The response is: " + response);
        is = conn.getInputStream();

        // Convert the InputStream into a string
        String contentAsString = readIt(is, len);
        return contentAsString;

    // Makes sure that the InputStream is closed after the app is
    // finished using it.
    } finally {
        if (is != null) {
            is.close();
        }
    }
}
```

请注意，getResponseCode() 会返回连接状态码( status code). 这是一种获知额外网络连接信息的有效方式。status code 是 200 则意味着连接成功.

## 5)将输入流(InputStream)转换为字符串(String)
[InputStream](http://developer.android.com/reference/java/io/InputStream.html) 是一种可读的byte数据源。如果你获得了一个 [InputStream](http://developer.android.com/reference/java/io/InputStream.html), 通常会进行解码(decode)或者转换为目标数据类型。例如，如果你是在下载图片数据，你可能需要像下面这样解码并展示它：

```java
InputStream is = null;
...
Bitmap bitmap = BitmapFactory.decodeStream(is);
ImageView imageView = (ImageView) findViewById(R.id.image_view);
imageView.setImageBitmap(bitmap);
```

在上面演示的示例中，[InputStream](http://developer.android.com/reference/java/io/InputStream.html) 包含的是web页面的文本内容。下面会演示如何把 [InputStream](http://developer.android.com/reference/java/io/InputStream.html) 转换为字符串，以便显示在UI上。

```java
// Reads an InputStream and converts it to a String.
public String readIt(InputStream stream, int len) throws IOException, UnsupportedEncodingException {
    Reader reader = null;
    reader = new InputStreamReader(stream, "UTF-8");
    char[] buffer = new char[len];
    reader.read(buffer);
    return new String(buffer);
}
```
