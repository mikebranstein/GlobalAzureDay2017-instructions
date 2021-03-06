## Using SignalR to Asynchronously Update Profile Pictures

In the last several chapters, you learned how to create an asynchronous image analysis process using Azure functions, blob storage, and the Cognitive Services Computer Vision API. Even though we automated the profile picture review and approval process, we inadvertently created another problem: our web app doesn't support the asynchronous process (meaning, users must continuously refresh their browser to check if their profile picture was approved). 

If there were only a way to notify the web browser when a profile picture was approved. There is: [ASP.NET SignalR](https://www.asp.net/signalr).

* **DEFINITION:** ASP.NET SignalR (commonly known as simple SignalR) is a library for ASP.NET developers that makes developing real-time web functionality easy. SignalR allows bi-directional communication between server and client. Servers can now push content to connected clients instantly as it becomes available. SignalR supports Web Sockets, and falls back to other compatible techniques for older browsers.

In this chapter, you'll learn how to use SignalR to notify users on the profile management page when profile pictures are approved.

### What You're Building

Before we jump into modifying our web app, let's take a quick look at the process SignalR will help us establish:

After a user updates their profile picture, our app will redirect to the profile management page. When the page loads, JavaScript on the page will establish a connection to a SignalR hub. SignalR hubs allow apps to make remote procedure calls (RPCs) from a server to connected clients. After connecting to the SignalR hub, the profile management page waits and listens for a message from the server. If the user browses away from the page, the browser stops listening and disconnects from the SignalR hub.

When the Azure function finishes processing an acceptable profile picture, it POSTs an HTTP request to a Web API endpoint hosted in our web app. The Web API endpoint pushes a message to the clients listening to the SignalR hub. The message includes the profile picture file name that was just accepted.

Back on the client side, when a message is received, the page determines if the profile picture accepted by the Azure function is the picture uploaded by the client. If it is, the image is reloaded by appending a random query string to the image's source URL.

### Adding SignalR to the Web App

In this exercise, you'll be adding SignalR to the web app by creating:
* client-side JavaScript to listen for hub messages
* a SignalR hub to send messages 

<h4 class="exercise-start">
    <b>Exercise</b>: Add SignalR to the web app
</h4>

Install SignalR by adding the following NuGet packages. Note that by installing *Microsoft.AspNet.SignalR*, the remainder of the packages should install because they're dependencies.

* Microsoft.AspNet.SignalR
* Microsoft.AspNet.SignalR.Core
* Microsoft.AspNet.SignalR.JS
* Microsoft.AspNet.SignalR.SystemWeb

![image](images/chapter9/install-signalr.png)

#### Manage\Index.cshtml

Add a SignalR script reference and hub listener code to the *Manage\Index* view. When you add this reference, be sure to check the version of SignalR installed. You can find the version number by looking in the *Scripts* folder of the MVC app. Look for a file named *jquery.signalr-X.Y.Z.js*, where X.Y.Z is the MAJOR.MINOR.PATCH version number of SignalR. Our version is v2.2.1.

![image](images/chapter9/check-signalr-version.png)

Add the JavaScript code snippet to the bottom of the view.

```javascript
@section scripts {
    <!--Reference the SignalR library - make sure it's version matches the installed version.-->
    <script src="~/Scripts/jquery.signalR-2.2.1.min.js"></script>
    <!--Reference the auto-generated SignalR hub script.-->
    <script src="~/signalr/hubs"></script>
    <!--SignalR script to handle profile picture updated message.-->
    <script>
        $(function () {
            // Reference the auto-generated proxy for the hub
            var profilePic = $.connection.profilePicHub;

            // Function the hub calls to notify the page a profile picture has 
            // been updated
            profilePic.client.profilePicUpdated = function (profilePicUrl) {
                if (profilePicUrl) {
                    var expectedProfilePicUrl = $("#profilePictureUrl").val();
                    if (expectedProfilePicUrl === profilePicUrl) {
                        $("#profilePicture").attr("src", profilePicUrl + "?" + Math.random());
                    }
                }
            };

            // Start the hub connection
            $.connection.hub.start().done(function () {
                // do nothing extra on load of the hub, but if we needed
                // to do something special, we could
            });
        });
    </script>
}
```

In the code snippet above, 3 things happen when the page loads:

1. A reference to the profile picture SignalR hub (`$.connection.profilePicHub`) is stored. You may be wondering why/how the JavaScript code knows what `profilePicHub` is. For now, just know that SignalR creates that for you automatically when the page loads, and that you'll learn more about it later in this chapter.

2. We establish a client-side function (`profilePic.client.profilePicUpdated`) that is called when the server pushes a method. The URL of the profile picture is passed to this function. The function uses a hidden field on the page (remember adding `@Html.HiddenFor(x => x.ProfilePicUrl, new { id = "profilePictureUrl" })` to the MVC view earlier?) . The hidden field is used to check whether the URL passed to the function is the URL of the current user's profile picture. If it is, the profile picture image's source property is updated with a random query string: this is a cool trick to force the image to be reloaded. 

3. The connection with the SignalR hub is established, beginning the listening process. When the hub connection starts, we have the opportunity to run additional code, but in our circumstance, there's no need.

> **NOTE:** You may be wondering why we're checking to ensure the URL passed to a client is the right URL. Imagine several users upload profile pictures simultaneously, each placing their picture in the *uploaded* blob container. The Azure function indiscriminately processes each image, and POSTs back to our Web API endpoint. The web app then broadcasts the picture URL to all clients connected to the SignalR hub. It's because of this broadcast that clients receive messages from the server about their image and others images. There *are* ways to change the way SignalR works, but for the purposes of this workshop, we've stuck with the broadcast approach. But, in a more professional setting, you wouldn't want to broadcast messages to clients that either aren't expecting them or to which they don't pertain.

#### ProfilePicHub.cs

Next, we'll create our SignalR hub by adding a new class to the root of the web app project. Name the class *ProfilePicHub* and add the code below. 

>* **DEFINITION:** SignalR hubs allow apps to make remote procedure calls (RPCs) from a server to connected clients. The hub manages and maintains the list of connected clients transparently, so you don't have to worry about it.

```csharp
using Microsoft.AspNet.SignalR;

namespace Web
{
    public class ProfilePicHub : Hub
    {
        private readonly ProfilePicBroadcaster _profilePicture;

        public ProfilePicHub(ProfilePicBroadcaster profilePicture)
        {
            _profilePicture = profilePicture;
        }
    }
}
```

As you can see, there's not much going on in the *ProfilePicHub* class, but that's because all the heavy lifting is being done in the inherited *Hub* class.

> **NOTE:** The name of the *ProfilePicHub* class isn't a coincidence. Earlier in this exercise, you created a reference to the SignalR hub in JavaScript: `var profilePic = $.connection.profilePicHub;`. When you inherit from the *Hub* class, SignalR will create a JavaScript object of the same name, placing it inside of the `$.connection` object.

You'll also notice the *ProfilePicBroadcaster* class that is passed in as a dependency to this class. This class lives up to it's name and broadcasts messages when a profile picture is updated. We have yet to define the class, so let's tackle that now.

#### ProfilePicBroadcaster.cs

Add another class to the root of the web app named *ProfilePicBroadcaster* and add the following code to it.

```csharp
using Microsoft.AspNet.SignalR;
using Microsoft.AspNet.SignalR.Hubs;
using System;

namespace Web
{
    /// <summary>
    ///  FROM https://docs.microsoft.com/en-us/aspnet/signalr/overview/getting-started/tutorial-server-broadcast-with-signalr
    /// </summary>
    public class ProfilePicBroadcaster
    {
        // Code block 1: singleton instance of this class to maintain a single
        // list of connected clients across the entire app
        private readonly static Lazy<ProfilePicBroadcaster> _instance = new Lazy<ProfilePicBroadcaster>(() => new ProfilePicBroadcaster(GlobalHost.ConnectionManager.GetHubContext<ProfilePicHub>().Clients));

        // Code block 2: constructor
        private ProfilePicBroadcaster(IHubConnectionContext<dynamic> clients)
        {
            Clients = clients;
        }

        // Code block 3: public static accessor to create the singleton of this class
        public static ProfilePicBroadcaster Instance
        {
            get
            {
                return _instance.Value;
            }
        }

        // Code block 4: private list of connected clients
        private IHubConnectionContext<dynamic> Clients
        {
            get;
            set;
        }

        // Code block 5: public function to broadcast a message to all connected clients
        public void BroadcastUpdatedProfilePic(string profilePicUrl)
        {
            Clients.All.profilePicUpdated(profilePicUrl);
        }
    }
}
```

This looks like a lot of code, but it's really straight forward if you break it down the right way. The code was adapted from the official [SignalR getting started tutorial](https://docs.microsoft.com/en-us/aspnet/signalr/overview/getting-started/tutorial-server-broadcast-with-signalr). It's worth checking out if you want to get an idea of everything SignalR can do.
 
In whole, the purpose of this class is to store a reference to the connected SignalR clients, and provide a mechanism to broadcast a message to all clients.

More specifically, there's 5 code blocks:

* Blocks 1-4 work together to lazily instantiate a singleton instance of this class. The class is intended to be used like a factory class, where you obtain an instantiated reference of the class through the `ProfilePicBroadcaster.Instance` property. When this property is referenced the first time, the `private readonly static _instance` variable is created. The code is a bit pretentious and may look confusing if you don't have any experience with the `Lazy<>` class and the singleton pattern. An easy way to think of these 4 code blocks is to know that when you call `ProfilePicBroadcaster.Instance` one or more times, you'll get the same instance of the object, meaning .NET only fires the constructor once and only uses the memory space once. 

* Block 5 is the only public instance method of the class. When called, it broadcasts a message to all connected clients by calling a method on `Clients.All`. You may notice something interesting about the method name called: `profilePicUpdated(profilePicUrl)`. This method is actually the method name we created in JavaScript earlier on the `client` object: `profilePic.client.profilePicUpdated = function (profilePicUrl) { ... };`. As long as the names of these methods are identical, SignalR passes data between the C# function call you've written here and the JavaScript function. Pretty cool.

> **NOTE:** We dropped a few development pattern names (singleton and factory) in this exercise, but didn't take the time to define them. We're not going to dive into the definition of these here, because others have written about them extensively. We particularly like Martin Fowler's explanation of these. His articles on [Inversion of Control](https://martinfowler.com/articles/injection.html) covers both of these. If you've never taken the time to read it, it's highly recommended.

#### Startup.cs

Ok, we're almost finished. The last step to configuring SignalR is to add it to the web app's startup process. Open the *Startup.cs* class in the root of the web project and add a line to configure SignalR: `app.MapSignalR();`.

```csharp
using Microsoft.Owin;
using Owin;

[assembly: OwinStartupAttribute(typeof(Web.Startup))]
namespace Web
{
    public partial class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            ConfigureAuth(app);
            app.MapSignalR();
        }
    }
}
```

<div class="exercise-end"></div>

### Adding a Web API Endpoint 

Nice work! You've added SignalR to your web project. But, it doesn't do us much good if we can't trigger `ProfilePicBroadcaster.BroadcastUpdatedProfilePic()` from our Azure function. 

There's a multitude of ways to solve this problem, but we've chosen something straight-forward: add a Web API endpoint that can trigger the broadcast method. Let's get started building this out.

<h4 class="exercise-start">
    <b>Exercise</b>: Add a Web API Endpoint
</h4>

Add Web API to the solution by adding 4 packages. Note that you should only need to install `Microsoft.AspNet.WebApi` because the other packages are dependencies that will be added automatically.

Install these 4 NuGet packages:

* Microsoft.AspNet.WebApi
* Microsoft.AspNet.WebApi.Core
* Microsoft.AspNet.WebApi.WebHost
* Microsoft.AspNet.WebApi.Client

#### WebApiConfig.cs

Create a file named *WebApiConfig.cs* in the *App_Start* folder. Add this code to the file, which configures several defaults for Web API.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web.Http;

namespace Web
{
    public static class WebApiConfig
    {
        public static void Register(HttpConfiguration config)
        {
            config.MapHttpAttributeRoutes();

            config.Routes.MapHttpRoute(
                name: "DefaultApi",
                routeTemplate: "api/{controller}/{id}",
                defaults: new { id = RouteParameter.Optional }
            );
        }
    }
}
```

#### Global.asax.cs

Open the *Global.asax.cs* file and add a reference to `System.Web.Http` at the top:

```csharp
using System.Web.Http;
```

Keep *Global.asax.cs* open and add register Web API by adding `GlobalConfiguration.Configure(WebApiConfig.Register);` before `RouteConfig.RegisterRoutes(...)` is called:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Web.Mvc;
using System.Web.Optimization;
using System.Web.Routing;
using System.Web.Http;

namespace Web
{
    public class MvcApplication : System.Web.HttpApplication
    {
        protected void Application_Start()
        {
            //ElCamino - Added to create azure tables
            ApplicationUserManager.StartupAsync();

            AreaRegistration.RegisterAllAreas();
            FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
            GlobalConfiguration.Configure(WebApiConfig.Register); // register Web API before registering routes
            RouteConfig.RegisterRoutes(RouteTable.Routes);
            BundleConfig.RegisterBundles(BundleTable.Bundles);
        }
    }
}
```

Well, that's all it takes to add Web API to the project, so let's start using it by adding an endpoint. 

#### ProfilePictureController.cs

Add a new Web API controller named *ProfilePicture* by adding a class to the *Controllers* folder named *ProfilePictureController.cs*. Add this code:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Net;
using System.Net.Http;
using System.Web.Http;

namespace Web.Controllers
{
    public class ProfilePictureController : ApiController
    {
        [HttpPost]
        public void Post([FromBody] string profilePicUrl)
        {
            ProfilePicBroadcaster.Instance.BroadcastUpdatedProfilePic(profilePicUrl);
        }
    }
}
```

This Web API endpoint will be hosted from *~\api\ProfilePicture* and accept a URL in the HTTP POST body. When it's called, it uses the `ProfilePicBroadcaster` class you created earlier to broadcast the URL to all connected clients. 

> **NOTE:** In a real-world scenario, I'd *never* leave a Web API endpoint completely open to the public. I'd secure it with some type of authentication and authorization like OAuth, or at least an API key. But, for our purposes of this lab, we're going to leave it wide open. Just understand this is a *bad* practice. 

<div class="exercise-end"></div>

That's it! We're finished with the web project. But, there's something you'll need to do before moving on.

> **WARNING:** It's critical that you re-publish the project to the Azure web app you created earlier. In the next section, you'll be updating your Azure function to POST the update profile picture URL to a *public* Web API endpoint. If your Web API endpoint isn't publicly exposed, the Azure function won't be able to communicate with it.

### Calling a Web API Endpoint from an Azure Function

The last step is to call the Web API endpoint from the Azure function after a profile picture is accepted as an appropriate picture. 

<h4 class="exercise-start">
    <b>Exercise</b>: Call a Web API Endpoint from an Azure Function
</h4>

Navigate back to the [Azure portal](https://portal.azure.com) and find your Azure function app on the dashboard you created earlier.

<img src="images/chapter9/open-function-app.png" class="img-small" />

Open the function app and locate the `BlobImageAnalysis` function.

<img src="images/chapter9/open-function.png" class="img-small" />

Replace the function code with the code below.

```csharp
using Microsoft.WindowsAzure.Storage.Blob;
using Microsoft.WindowsAzure.Storage;
using System.Net.Http.Headers;
using System.Configuration;
using System.Text;
using System.Web.Http;
using System.Net;

public async static Task Run(Stream myBlob, string name, string ext, TraceWriter log)
{       
    log.Info($"Analyzing uploaded image {name} for appropriate content...");

    var array = await ToByteArrayAsync(myBlob);
    var result = await AnalyzeImageAsync(array, log);

    log.Info("Is Adult: " + result.adult.isAdultContent.ToString());
    log.Info("Adult Score: " + result.adult.adultScore.ToString());
    log.Info("Is Racy: " + result.adult.isRacyContent.ToString()); 
    log.Info("Racy Score: " + result.adult.racyScore.ToString());

    // Reset stream location
    myBlob.Seek(0, SeekOrigin.Begin);
    if (result.adult.isAdultContent || result.adult.isRacyContent)
    {
        // profile picture is NOT acceptable - copy blob to the "rejected" container
        StoreBlobWithMetadata(myBlob, "rejected", name, ext, result, log);
    }
    else
    {
        // profile picture is acceptable - copy blob to the "profile-pics" container
        StoreBlobWithMetadata(myBlob, "profile-pics", name, ext, result, log);

        log.Info($"Calling signalR for image {name}.{ext}");

        // alert SignalR hub
        var webApiEndpointBaseUrl = ConfigurationManager.AppSettings["WebAPIEndpointBaseUrl"].ToString();
        WebRequest request = WebRequest.Create($"{webApiEndpointBaseUrl}/api/ProfilePicture");
        request.Method = "POST";
        request.ContentType = "application/x-www-form-urlencoded";

        ASCIIEncoding encoding = new ASCIIEncoding();
        var storageAccountName = ConfigurationManager.AppSettings["StorageAccountName"].ToString();
        var stringData = $"=https://{storageAccountName}.blob.core.windows.net/profile-pics/{name}.{ext}";
        var data = encoding.GetBytes(stringData);
        request.ContentLength = data.Length;

        Stream newStream = request.GetRequestStream();
        newStream.Write(data, 0, data.Length);
        newStream.Close();
        request.GetResponse();

        log.Info($"SignalR call finished");
    }
}

private async static Task<ImageAnalysisInfo> AnalyzeImageAsync(byte[] bytes, TraceWriter log)
{
    HttpClient client = new HttpClient();

    var key = ConfigurationManager.AppSettings["SubscriptionKey"].ToString();
    client.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", key);

    HttpContent payload = new ByteArrayContent(bytes);
    payload.Headers.ContentType = new MediaTypeWithQualityHeaderValue("application/octet-stream");
    
    var results = await client.PostAsync("https://westus.api.cognitive.microsoft.com/vision/v1.0/analyze?visualFeatures=Adult", payload);
    var result = await results.Content.ReadAsAsync<ImageAnalysisInfo>();
    return result;
}

// Writes a blob to a specified container and stores metadata with it
private static void StoreBlobWithMetadata(Stream image, string containerName, string blobName, string ext, ImageAnalysisInfo info, TraceWriter log)
{
    log.Info($"Writing blob and metadata to {containerName} container...");
    
    var connection = ConfigurationManager.AppSettings["AzureWebJobsStorage"].ToString();
    var account = CloudStorageAccount.Parse(connection);
    var client = account.CreateCloudBlobClient();
    var container = client.GetContainerReference(containerName);

    try
    {
        var blob = container.GetBlockBlobReference($"{blobName}.{ext}");
    
        if (blob != null) 
        {
            // Upload the blob
            blob.UploadFromStream(image);

            // Set the content type of the image
            blob.Properties.ContentType = "image/" + ext;
            blob.SetProperties();

            // Get the blob attributes
            blob.FetchAttributes();
            
			// Write the blob metadata
            blob.Metadata["isAdultContent"] = info.adult.isAdultContent.ToString(); 
            blob.Metadata["adultScore"] = info.adult.adultScore.ToString("P0").Replace(" ",""); 
            blob.Metadata["isRacyContent"] = info.adult.isRacyContent.ToString(); 
            blob.Metadata["racyScore"] = info.adult.racyScore.ToString("P0").Replace(" ",""); 
            
			// Save the blob metadata
            blob.SetMetadata();
        }
    }
    catch (Exception ex)
    {
        log.Info(ex.Message);
    }
}

// Converts a stream to a byte array 
private async static Task<byte[]> ToByteArrayAsync(Stream stream)
{
    Int32 length = stream.Length > Int32.MaxValue ? Int32.MaxValue : Convert.ToInt32(stream.Length);
    byte[] buffer = new Byte[length];
    await stream.ReadAsync(buffer, 0, length);
    return buffer;
}

public class ImageAnalysisInfo
{
    public Adult adult { get; set; }
    public string requestId { get; set; }
}

public class Adult
{
    public bool isAdultContent { get; set; }
    public bool isRacyContent { get; set; }
    public float adultScore { get; set; }
    public float racyScore { get; set; }
}
```

The code added uses two configuration app settings to POST the URL of the profile picture when it's acceptable.

Return to the *Application Settings* area of the function app.

<img src="images/chapter9/goto-app-settings.gif" class="img-large" />

Add app settings for `WebAPIEndpointBaseUrl` and `StorageAccountName`:
* **WebAPIEndpointBaseUrl:** set to the URL of the web app you created for this workshop, i.e., *http://globalazurelouisville.azurewebsites.net*. Do not include an ending slash.
* **StorageAccountName:** set to the name of the storage account you created for this workshop

<img src="images/chapter9/add-app-settings.png" class="img-medium" />

You're finished!

<div class="exercise-end"></div>

Wow! That was a lot of changes, but we think it was worth it. Browse out to the Azure-hosted URL of your web app (ours is *http://louglobalazure2017.azurewebsites.net/*). You'll note this URL is different from the others in the guide, but it has the same functionality.

When you upload a new profile picture, you'll navigate back to the profile management page. After a few seconds, you should see the image update after the the Azure function analyzes it with the Cognitive Services API and calls the Web API endpoint we created. The Web API endpoint broadcasts the profile picture URL to all connected SignalR hub clients. When this is received by the listener in JavaScript, the source property of the image is updated to include a random query string, causing the image to re-fresh.

It's beautiful.

<img src="images/chapter9/signalr-update.gif" class="img-large" />

#### Summary

In this chapter you learned:
* SignalR hubs can be used to broadcast messages from a server to connected clients
* How to call a Web API endpoint from an Azure function