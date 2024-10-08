# .NET HybridWebView experiment

This repo has the experimental .NET MAUI HybridWebView control, which enables hosting arbitrary HTML/JS/CSS content in a WebView and enables communication between the code _in_ the WebView (JavaScript) and the code that hosts the WebView (C#/.NET). For example, if you have an existing React JS application, you could host it in a cross-platform .NET MAUI native application, and build the back-end of the application using C# and .NET.

Example usage of the control:

```xaml
<hwv:HybridWebView
    HybridAssetRoot="hybrid_root"
    MainFile="hybrid_app.html"
    RawMessageReceived="OnHybridWebViewRawMessageReceived" />
```

And here's how .NET code can call a JavaScript method:

```csharp
var sum = await myHybridWebView.InvokeJsMethodAsync<int>("JsAddNumbers", 123, 456);
```

And the reverse, JavaScript code calling a .NET method:

```js
HybridWebView.SendInvokeMessageToDotNet("CallMeFromScript", ["msg from js", 987]);
```

With JavaScript you can also asynchronously call a .NET method and get a result:

```js
HybridWebView.SendInvokeMessageToDotNetAsync("CallMeFromScriptAsync", ["msg from js", 987])
	.then(result => console.log("Got result from .NET: " + result));
```

or 

```js
const result = await HybridWebView.SendInvokeMessageToDotNetAsync("CallMeFromScriptAsync", ["msg from js", 987]);
```

In addition to method invocation, sending "raw" messages is also supported.

## Discussions/questions/comments

please log an issue in this repo for any other topics.

## Getting Started

To get started, you'll need a .NET MAUI 8 project, then add the HybridWebView control, and add some web content to it.

1. Ensure you have Visual Studio 2022 with the .NET MAUI workload installed
1. Create a **.NET MAUI App** project that targets .NET 8 (or use an existing one)
1. Add a reference to the `EJL.HybridWebView` package ([NuGet package]
    1. Right-click on the **Dependencies** node in Solution Explorer and select **Manage NuGet Packages**
    1. Select the **Browse** tab
    1. Ensure the **Include prerelease** checkbox is checked
    1. In the search box enter    1. Select the matching package, and click the **Install** button
1. Register and add the `HybridWebView` control to a page in your app:
    1. Delete the `count` field, and the `OnCounterClicked` method, and replace it with the following code:
        ```csharp
        private async void OnHybridWebViewRawMessageReceived(object sender, HybridWebView.HybridWebViewRawMessageReceivedEventArgs e)
        {
            await Dispatcher.DispatchAsync(async () =>
            {
                await DisplayAlert("JavaScript message", e.Message, "OK");
            });
        }
        ```

1. You can now run your .NET MAUI app with the HybridWebView control!
    1. You can run it on Windows, Android, iOS, or macOS
    1. When you launch the app, type text into the textbox and click the button to receive the message in C# code

## Proxy URLs

The `HybridWebView` control can redirect URL requests to native code, and allow custom responses streams to be set. 
This allows scenarios such as dynamically generating content, loading content from compressed files like ZIP or SQLite, or loading content from the internet that doesn't support CORS.

To use this feature, handle the `ProxyRequestReceived` event in the `HybridWebView` control. 
When the event handler is called set the `ResponseStream` and optionally the `ResponseContentType` of the `HybridWebViewProxyEventArgs` object received in the `OnProxyRequest` method.

The `HybridWebViewProxyEventArgs` has the following properties:

| Property | Type | Description |
| -------- | ---- | ----------- |
| `QueryParams` | `IDictionary<string, string>` | The query string parameters of the request. Note that all values will be strings. |
| `ResponseContentType` | `string` | The content type of the response body. Default: `"text/plain"` |
| `ResponseStream` | `Stream` | The stream to use as the response body. |
| `Url` | `string` | The full URL that was requested. |

```csharp
myWebView.ProxyRequestReceived += async (args) =>
{
    //Use the query string parameters to determine what to do.
    if (args.QueryParams.TryGetValue("myParameter", out var myParameter))
	{
        //Add your logic to determine what to do. 
		if (myParameter == "myValue")
		{
            //Add logic to get your content (e.g. from a database, or generate it).
            //Can be anything that can be turned into a stream.

            //Set the response stream and optionally the content type.
			args.ResponseStream = new MemoryStream(Encoding.UTF8.GetBytes("This is the file content"));
			args.ResponseContentType = "text/plain";
		}
	}
};
```

In your web app, you can make requests to the proxy URL by either using relative paths like `/proxy?myParameter=myValue` or an absolute path by appending the relative path tot he pages origin location `window.location.origin + '/proxy?myParameter=myValue'`.
Be sure to encode the query string parameters so that they are properly handled. Here are some ways to implement proxy URLs in your web app:

1. Use proxy URLs with HTML tags. 
   ```html
   <img src="/proxy?myParameter=myValue" />
   ```
2. Use proxy URLs in JavaScript. 
   ```js
   var request = window.location.origin + '/proxy?myParameter=' + encodeURIComponent('myValue');

   fetch(request)
	   .then(response => response.text())
	   .then(data => console.log(data));
   ```
  3. Use proxy URLs with other JavaScript libraries. Some libraries only allow you to pass in string URLs. If you want to create the response in C# (for example, to generate content, or load from a compressed file), you can use a proxy URL to allow you to fulfil the response in C#.
 
**NOTE:** Data from the webview can only be set in the proxy query string. POST body data is not supported as the native `WebView` in platforms do not support it.

To make changes to the fork and update the .NET MAUI app, here's what I do:

1. Clone of the forked repo and open a terminal/console window in that folder
1. Run `yarn` to ensure the JavaScript dependencies are installed
1. Run `set PUBLIC_URL=/` to establish the root of the app as `/` because that's the root of the .NET MAUI HybridWebView app
1. Run `npm run build` to compile the app and produce a static version of it
   * If you get this error: `Error: error:0308010C:digital envelope routines::unsupported`
   * Then run `set NODE_OPTIONS=--openssl-legacy-provider`
   * And run this again: `npm run build`
1. This will build the HTML/JS/CSS output into a new `./build` folder
1. Then run the MauiReactJSHybridApp from Visual Studio