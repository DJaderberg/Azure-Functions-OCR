# Cognitive Services - OCR

This tutorial shows how to obtain a [Cognitive Services](https://azure.microsoft.com/en-us/services/cognitive-services/) API Key and use a console app to return words shown on a image using the [Computer Vision OCR](https://www.microsoft.com/cognitive-services/en-us/Computer-Vision-API/documentation/OCR) API.

* This is **Part 1**.
* There's also Part 2 - Azure Functions.

## Get Azure Subscription

To use Azure you need a Microsoft Account. If you don't have one, [register on the web](https://signup.live.com/).

If you don't have an Azure Subscription already, there are several ways to get it:

* Start a [free trial](https://azure.microsoft.com/en-us/free/) and get $200 in credits for one month.
* Sign up for [Visual Studio Dev Essentials](https://www.visualstudio.com/dev-essentials/) and get $25 in free credits each month for one year.
* Get a [Visual Studio subscription](https://www.visualstudio.com/subscriptions/) or ask at your company if you have access to it.
* Or just get it any other way you can... :)

## Start up the service and get a key

1. Go to [Azure Portal](https://portal.azure.com)
2. Create new **Resource Group**
3. Add new **Cognitive Services API**
   1. Choose **Computer Vision**
   2. Pick a name for it, such as *Vision*
   3. Pick a pricing tier - **Free** will be enough for this case
   4. Review and accept **License Terms**
4. Go to the newly created API
5. Go to **Keys** and copy **KEY 1**

## Create a console app using the API

1. Start **Visual Studio**
2. Go to **File > New Project > Windows** and create new **Console Application**
3. Right-click the project and select **Manage NuGet packages**...
4. Search for and install package **Microsoft.ProjectOxford.Vision**
   - Alternatively you can use the Package Manager console: `Install-Package Microsoft.ProjectOxford.Vision`
5. Go to **Program.cs** and start coding

First we need to define a variable for the Cognitive Services API Key. 

> We will bake the code directly into the code, but in production you would probably use **ConfigurationManager** or some other settings/keys management method.

```c#
string apiKey = "keykeykey";
```

Then we need to find an image to be sent to the API. Go ahead and look around the internet, I will use this:

```c#
string imageUrl = "https://2.bp.blogspot.com/-SaoK-RZRX60/V1krBqEqXcI/AAAAAAAAcSY/TaMvJ8V6l4AkBlhmA-Z9noWcxBNNJ_PegCLcB/w800/2017-skoda-octavia-0.jpg";
```

Next, we finally initialize the Vision API client:

```c#
VisionServiceClient visionClient = new VisionServiceClient(apiKey);
```

And send our image to the OCR endpoint:

```c#
OcrResults results = visionClient.RecognizeTextAsync(imageUrl, "cs").Result;
```

> Few remarks for this line:
>
> * Parameter value *"cs"* is saing that the text is expected to be Czech. For English, you would use *"en"*.
> * For simplicity, we're running this method synchronously and blocking the thread. In real application you would use *async/await*.

After the call is done, we use LINQ to get specific words from the result:

```c#
var words = from r in results.Regions
			from l in r.Lines
             from w in l.Words
             select w.Text;
```

And print them to the console:

```c#
Console.WriteLine(String.Join("|", words.ToArray()));
Console.ReadKey();
```

If everything went well, your application should connect to the Cognitive Services Computer Vision API, send image URL to the OCR endpoint and write words separated by |.



Full code here:

```c#
using Microsoft.ProjectOxford.Vision;
using Microsoft.ProjectOxford.Vision.Contract;
using System;
using System.Linq;

namespace OCR_Demo
{
    class Program
    {
        static void Main(string[] args)
        {
            string apiKey = "keykeykey";
            string imageUrl = "https://2.bp.blogspot.com/-SaoK-RZRX60/V1krBqEqXcI/AAAAAAAAcSY/TaMvJ8V6l4AkBlhmA-Z9noWcxBNNJ_PegCLcB/w800/2017-skoda-octavia-0.jpg";

            VisionServiceClient visionClient = new VisionServiceClient(apiKey);

            OcrResults results = visionClient.RecognizeTextAsync(imageUrl, "cs").Result;

            var words = from r in results.Regions
                        from l in r.Lines
                        from w in l.Words
                        select w.Text;

            Console.WriteLine(String.Join("|", words.ToArray()));
            Console.ReadKey();
        }
    }
}
```

