## Adding a profile picture to the app

In this chapter you will learn:
* How to extend ASP.NET Identity by adding an image to a user profile
* How to upload files to Azure blob storage

In the last chapter, you learned how to extend ASP.NET Identity by adding a biography to a user's profile. You also saw that by adding a public property to the `ApplicationUser` class, ASP.NET Identity managed the schema.

We'll continue extending ASP.NET Identity in this chapter by adding an image to the user profile management page.

### Visualizing What You're Building

Before we jump in, let's take a minute to visualize what you'll be building in this chapter.

<img src="images/chapter6/upload-picture.gif" class="img-large" />

We'll be adding the *Profile Picture* field. Once added, you'll be able to see your uploaded picture on the main page and navigate to a second page to update it.

Let's get started! We'll begin by modifying ASP.NET Identity to include the new field, then move on to modifying the MVC code.

### Extending ASP.NET Identity

You'll recall that we previously updated the `ApplicationUser` and `ApplicationUserManager` classes to extend ASP.NET Identity. We'll be doing something similar for the profile picture. 

<h4 class="exercise-start">
    <b>Exercise</b>: Extending ASP.NET Identity with a Profile Picture
</h4>

To extend ASP.NET Identity, add two things:
1. public property to the `ApplicationUser` class
2. async accessor function for the property to the `ApplicationUserManager` class

#### Updating the `ApplicationUser` class

Add a public property to the `ApplicationUser` class, located in the the *IdentityModel.cs* file. You can find this file in the *Models* folder.

You'll remember this is the class that inherits from `IdentityUser`, the class ASP.NET Identity uses to represent a user. 

```csharp
public string ProfilePicUrl { get; set; }
```

You may have noticed something interesting about the property you just added: it's a string, not an image. But why?

> **NOTE:** We could add a binary property to store the actual image, but that would store the images as a column in the *AspNetUsers* table in Azure table storage. Table storage is great for semi-structured text data, but not so good for blob (binary large object) data, like an image. Instead, we'll be using a different type of Azure storage to store our images, called blob storage. We're not going to cover the details of blob storage right now, but it's important you understand why we're not adding a binary property to the `ApplicationUser` class.

So, if we're not storing binary data in the `ApplicationUser` class, why are we storing a URL? You may recall from a previous chapter that data stored in an Azure storage account can be accessed via URL in the form of *https://{storage-account-name}.{storage-type}.core.windows.net*. 

After uploading an image to blob storage, we'll save the image URL to our user profile.  

#### Updating the `ApplicationUserManager` class

We also need to add an asynchronous accessor function to the `ApplicationUserManager` class (just like we did for the biography property). The method will take a `userId` and return the user's profile picture URL. You can find the `ApplicationUserManager` class in the *IdentityConfig.cs* file, located in the *App_Start* folder.

Add this code after the constructor.

```csharp
public async Task<string> GetProfilePicUrlAsync(string userId)
{
    var user = await this.Store.FindByIdAsync(userId);
    return (user != null) ? user.ProfilePicUrl : string.Empty;
}
``` 

This function uses the `Store` object, which is of type `IUserStore<ApplicationUser>`, which (simply put) is the object that manages storing and retrieving user data from our Azure table. 

The `this.Store.FindByIdAsync(userId)` call gets a reference to our user. We then return the user's profile picture URL, or an empty string if the user isn't found.

<div class="exercise-end"></div>

That's it. Let's move on to updating our MVC app to support the profile picture property.

### Updating MVC code to Support the Profile Picture Property

Now that you've added the property, let's update the MVC models, views, and controller actions to support the addition of the profile picture property.

<h4 class="exercise-start">
    <b>Exercise</b>: Updating the MVC code to support the profile picture property
</h4>

This exercise (just like the biography property exercise) is a bit longer than others, because we'll be updating a lot of files. At a high level, we'll be adding the profile picture property to our web app in several steps. 

We'll start with the profile management page, which will show the profile picture:
* **Step 1:** Update the `IndexViewModel` in *ManageViewModels.cs*
* **Step 2:** Update the Manage\Index view 

Then we'll move on to a new page that uploads the profile picture:
* **Step 3:** Update the `UpdateBiographyViewModel` class in *ManageViewModels.cs*
* **Step 4:** Update the Update Biography view
* **Step 5:** Update the GET controller action for the Update Biography view to *ManageController.cs*
* **Step 6:** Update the POST controller action for the Update Biography view to *ManageController.cs*
* **Step 7:** Add application settings to *web.config* 

Finally, we'll return to the profile management page:
* **Step 8:** Update the GET controller action for the Index view in *ManageController.cs* to populate the view with the updated profile picture URL

There's a lot to do, so let's get moving!

#### **Step 1:** Update the `IndexViewModel` in *ManageViewModels.cs*

Start by updating the `IndexViewModel` class. Add a property for the profile picture URL.

```csharp
public class IndexViewModel
{
    public bool HasPassword { get; set; }
    public IList<UserLoginInfo> Logins { get; set; }
    public string PhoneNumber { get; set; }
    public bool TwoFactor { get; set; }
    public bool BrowserRemembered { get; set; }
    public string Biography { get; set; }
    public string ProfilePicUrl { get; set; }
}
```

> Adding this property will allow the index view to display the profile picture when it loads. We'll be setting the value of the URL later in this exercise when we update the index controller's GET action.

#### **Step 2:** Update the Manage\Index view 

Update *Index.cshtml* in the *Views\Manage* folder to display:
* Image with it's source set to the profile picture URL
* An MVC HiddenFor element

> **NOTE:** You might be wondering what the MVC HiddenFor element is for. We'll be using this in a future chapter, so you can ignore it for now. 

Add this markup before the `<dl class="dl-horizontal">` element declaration:

```html
<img id="profilePicture" src="@Model.ProfilePicUrl" alt="No profile picture specified." class="has-border" style="width: 100%; max-width:300px;" />
@Html.HiddenFor(x => x.ProfilePicUrl, new { id = "profilePictureUrl" })
```

When the page loads, the profile picture will be downloaded from the URL specified. 

#### **Step 3:** Update the `UpdateBiographyViewModel` class in *ManageViewModels.cs*

In the last chapter, you created the `UpdateBiographyViewModel` class in the *ManageViewModels.cs* file. It was used to view and update a user's biography from the Update Biography view. 

Update the class by adding several fields for the profile picture.

```csharp
public class UpdateBiographyViewModel
{
    [Display(Name = "Biography")]
    public string Biography { get; set; }

    [Display(Name = "Profile Picture")]
    [DataType(DataType.Upload)]
    public HttpPostedFileBase ProfilePicture { get; set; }

    public string ProfilePicUrl { get; set; }
}
```
You'll notice we added two properties: `ProfilePicture` and `ProfilePicUrl`:
* **ProfilePicture:** stores the image binary, when uploaded
* **ProfilePictureURL:** displays the current profile picture to users 

When you add the `ProfilePicture` property, you'll need to add an additional reference to the top of the file because `HttpPosedFileBase` resides in the `System.Web` namespace.

Add the `System.Web` reference to the *ManageViewModels.cs* file.

```csharp
using System.Web;
```

#### **Step 4:** Update the Update Biography view

Update the view named `UpdateBiography.cshtml` in the *Views\Manage* folder. This view will use the previously created `UpdateBiographyViewModel` to show and update a user's profile picture.

Add an additional `<div class="form-group">...</div>` element between the existing `<div>` elements. 

```html
<div class="form-group">
    @Html.LabelFor(m => m.ProfilePicture, new { @class = "col-md-2 control-label" })
    <div class="col-md-10">
        <img src="@Model.ProfilePicUrl" alt="Profile picture not present or not approved." class="has-border" style="width: 100%; max-width: 300px;" />
        @Html.TextBoxFor(m => m.ProfilePicture, new { type = "file" })
        @Html.HiddenFor(m => m.ProfilePicUrl)
    </div>
</div>
```

#### **Step 5:** Update the GET controller action for the Update Biography view in *ManageController.cs*

Update the GET controller action in the *ManageController.cs* file to populate the profile picture URL property of the model.

```csharp
//
// Get: /Manage/UpdateBiography
public async Task<ActionResult> UpdateBiography()
{
    var userId = User.Identity.GetUserId();
    var updateBiographyViewModel = new UpdateBiographyViewModel()
    {
        Biography = await UserManager.GetBiographyAsync(userId),
        ProfilePicUrl = await UserManager.GetProfilePicUrlAsync(userId)
    };

    return View(updateBiographyViewModel);
}
``` 

#### **Step 6:** Update the POST controller action for the Update Biography view in *ManageController.cs*

Update the POST controller action in the Manage controller, using the `UpdateBiographySuccess` enum value just created.

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<ActionResult> UpdateBiography(UpdateBiographyViewModel model)
{
    if (!ModelState.IsValid)
    {
        return View(model);
    }
    var user = await UserManager.FindByIdAsync(User.Identity.GetUserId());
    if (user != null)
    {
        user.Biography = model.Biography;
        user.ProfilePicUrl = await UploadImageAsync(model.ProfilePicUrl, model.ProfilePicture) ?? user.ProfilePicUrl;
        await UserManager.UpdateAsync(user);
    }
    return RedirectToAction("Index", new { Message = ManageMessageId.UpdateBiographySuccess });
}
```

We've added a function call to populate the profile picture URL: `user.ProfilePicUrl = await UploadImageAsync(model.ProfilePicUrl, model.ProfilePicture) ?? user.ProfilePicUrl;`, but it may be a bit confusing, so let's break it down.

First, we call a function that uploads the profile picture to Azure blob storage: `UploadImageAsync(model.ProfilePicUrl, model.ProfilePicture)`. If the upload is successful, it returns the URL of the uploaded image. Otherwise, it returns `null`. 

If that function returns `null`, the existing profile picture url is kept.

> **NOTE:** You may not recognize the `??` syntax, as it's a feature of C# called the null-coalescing operator. It returns the left-hand operand if the operand is not null; otherwise it returns the right hand operand. For more information on this operator, check out the official [documentation](https://msdn.microsoft.com/en-us/library/ms173224.aspx).

Add the `UploadImageAsync` function below the POST controller action.

```csharp
public async Task<string> UploadImageAsync(string currentBlobUrl, HttpPostedFileBase imageToUpload)
{
    string imageFullPath = null;
    if (imageToUpload == null || imageToUpload.ContentLength == 0)
    {
        return null;
    }
    try
    {
        // connect to our storage account
        var cloudStorageAccount = CloudStorageAccount.Parse($"DefaultEndpointsProtocol=https;AccountName={ConfigurationManager.AppSettings["StorageAccountName"]};AccountKey={ConfigurationManager.AppSettings["StorageAccountKey"]};");
        var cloudBlobClient = cloudStorageAccount.CreateCloudBlobClient();
        var cloudBlobContainer = cloudBlobClient.GetContainerReference(ConfigurationManager.AppSettings["ProfilePicBlobContainer"]);

        // create the blob storage container, if needed
        if (await cloudBlobContainer.CreateIfNotExistsAsync())
        {
            await cloudBlobContainer.SetPermissionsAsync(
                new BlobContainerPermissions
                {
                    PublicAccess = BlobContainerPublicAccessType.Blob
                }
            );
        }

        var imageName = $"{Guid.NewGuid()}{Path.GetExtension(imageToUpload.FileName)}";

        // upload image blob
        var cloudBlockBlob = cloudBlobContainer.GetBlockBlobReference(imageName);
        cloudBlockBlob.Properties.ContentType = imageToUpload.ContentType;
        await cloudBlockBlob.UploadFromStreamAsync(imageToUpload.InputStream);

        // get URL of uploaded image blob
        imageFullPath = cloudBlockBlob.Uri.ToString();
    }
    catch (Exception ex)
    {
        // in reality, you should handle this...
    }
    return imageFullPath;
}
```

There are 4 things happening in the upload function:
* **Connect to Storage Account:** Using the `CloudStorageAccount` class, parse the storage account connection string, create a blob client, and a reference to the container (or folder) we'd like to access. The blob client works like a web service proxy that can interact with Azure blob storage to get a reference to the blob folders (a.k.a. containers) in the account. 
* **Create the blob container:** It's important to know that the reference to the blob container doesn't mean the cloud container exists. Whenever you access a container, you should first ensure that it exists. If not, create it. This may seem extra, but always checking for the container helps you to write more defensive code that ensures any implicit assumptions (like the existence of the container) are valid.
* **Upload image blob:** With the container reference, we get a reference to a blob block (a.k.a. file) that will contain our image. After setting the content type of the blob block, the image bits are uploaded.
* **Get URL of the uploaded image blob:** When the image is uploaded, we get the URL of it via the `Uri` property. 

> **NOTE:** Throughout the code we've written to access cloud resources, you'll notice a clear asynchronous programming pattern. When you're interacting with the cloud, you should always perform action in an asynchronous manner because you never know how long an action will take. In the event an action takes longer than expected, executing the command asynchronously won't prevent other code from executing.

The last step is to add several references to the top of the *ManageController.cs* file:

```csharp
using Microsoft.WindowsAzure.Storage;
using System.Configuration;
using Microsoft.WindowsAzure.Storage.Blob;
using System.IO;
```

#### **Step 7:** Add application settings to *web.config* 

Various functions and code we've added reference application settings via the `ConfigurationManager` class. Add the following keys to the `<appSettings>...</appSettings>` element of the *web.config* file.

```xml
    <add key="StorageAccountName" value="STORAGE_ACCOUNT_NAME" />
    <add key="StorageAccountKey" value="STORAGE_ACCOUNT_KEY" />
    <add key="ProfilePicBlobContainer" value="profile-pics" />
``` 

The `StorageAccountName` and `StorageAccountKey` keys are the same account name and key you added to the storage account connection string earlier. If you're unsure of these values, look at the El Camino configuration section.

The `ProfilePicBlobContainer` key is a blob container name that will hold our uploaded images. Don't change these. In this chapter, the image upload function uses the key to upload profile pictures to the blob container named *profile-pics*. 

#### **Step 8:** Update the GET controller action for the Index view in *ManageController.cs* 

The final step is to update the GET controller action of the Manage\Index view. The entire function is included below. 

```csharp
//
// GET: /Manage/Index
public async Task<ActionResult> Index(ManageMessageId? message)
{
    ViewBag.StatusMessage =
        message == ManageMessageId.ChangePasswordSuccess ? "Your password has been changed."
        : message == ManageMessageId.SetPasswordSuccess ? "Your password has been set."
        : message == ManageMessageId.SetTwoFactorSuccess ? "Your two-factor authentication provider has been set."
        : message == ManageMessageId.Error ? "An error has occurred."
        : message == ManageMessageId.AddPhoneSuccess ? "Your phone number was added."
        : message == ManageMessageId.RemovePhoneSuccess ? "Your phone number was removed."
        : message == ManageMessageId.UpdateBiographySuccess ? "Your biography was updated."
        : "";

    var userId = User.Identity.GetUserId();
    var model = new IndexViewModel
    {
        HasPassword = HasPassword(),
        PhoneNumber = await UserManager.GetPhoneNumberAsync(userId),
        TwoFactor = await UserManager.GetTwoFactorEnabledAsync(userId),
        Logins = await UserManager.GetLoginsAsync(userId),
        BrowserRemembered = await AuthenticationManager.TwoFactorBrowserRememberedAsync(userId),
        Biography = await UserManager.GetBiographyAsync(userId),
        ProfilePicUrl = await UserManager.GetProfilePicUrlAsync(userId)
    };
    return View(model);
}
```

<div class="exercise-end"></div>

Another *huge* update. Compile it again.

When you launch the app to test, login and navigate to the profile management page. You should see something similar to the image below. If you don't, that's ok. You can always grab the code from the `chapter7` branch of the Github repository.

<img src="images/chapter6/upload-picture.gif" class="img-large" />

Before we're finished, let's take a look at Storage Explorer to see our uploaded profile picture.

Click *Refresh All*, browse to your Azure storage account, and open the *Blob Containers* element. Inside, you'll see the *profile-pics* container. Selecting the container will show the uploaded profile pictures.

<img src="images/chapter6/profile-pics-uploaded.gif" class="img-large" />

#### Summary

In this chapter, you learned:
* Why you shouldn't store blobs in Azure table storage, and that blob storage is a much better choice
* When interacting with the cloud, you should use asynchronous method calls
* When you reference blob containers, always check if they exist before using them

In the next chapter, we'll continue to explore blob storage by uploading profile pictures to a holding zone where images will need to be approved prior to being accepted as a profile picture.