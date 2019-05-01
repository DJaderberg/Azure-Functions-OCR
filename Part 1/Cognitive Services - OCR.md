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
3. Add new **Cognitive Services**
   1. Pick a name for it, such as *Vision*
   2. Pick a pricing tier
   3. Review and accept **License Terms**
4. Go to the newly created API
5. Go to **Keys** and copy **KEY 1**

## Create a console app using the API

1. Start **Visual Studio**
2. Go to **File > New Project > Windows** and create new **Console Application**
3. Right-click the project and select **Manage NuGet packages**...
4. Search for and install package **Microsoft.Azure.CognitiveServices.Vision.ComputerVision**
   - Alternatively you can use the Package Manager console: `Install-Package Microsoft.Azure.CognitiveServices.Vision.ComputerVision`
5. Go to **Program.cs** and start coding


First we need to define a variable for the Cognitive Services API Key. 

We will bake the code directly into the code, but in production you would probably use **ConfigurationManager** or some other settings/keys management method. We also define the
base URI of the region that we are using, in my case, `eastus`. You should set this to 
the region that your resource group is created in.

```c#
private const string SubscriptionKey = "keykeykey";
private const string ApiUri = "https://eastus.api.cognitive.microsoft.com/";
```

We also define some visual features that Azure should be looking for, in `Features`.

In the `Main` method we need to find image(s) to be sent to the API. Go ahead and look around the internet, I will use these:

```c#
static void Main(string[] args)
{
    const string waterfall = "https://upload.wikimedia.org/wikipedia/commons/3/3c/Shaki_waterfall.jpg";
    const string skoda = "https://2.bp.blogspot.com/-SaoK-RZRX60/V1krBqEqXcI/AAAAAAAAcSY/TaMvJ8V6l4AkBlhmA-Z9noWcxBNNJ_PegCLcB/w800/2017-skoda-octavia-0.jpg";

```

Next, we finally initialize the Vision API client:

```c#
    ComputerVisionClient computerVision = new ComputerVisionClient(
        new ApiKeyServiceClientCredentials(SubscriptionKey),
        new System.Net.Http.DelegatingHandler[] { });

    computerVision.Endpoint = ApiUri;
```

And send our images to the vision and OCR endpoints:

```c#
    var t1 = AnalyzeRemoteAsync(computerVision, waterfall);
    var t2 = AnalyzeRemoteAsync(computerVision, skoda);
    var ocr = MakeOcrRequest(computerVision, skoda);
```

Then, we wait for everything to finish before we shut down the program
```c#
    Task.WhenAll(t1, t2, ocr).Wait(5000);
    Console.WriteLine("Press ENTER to exit");
    Console.ReadLine();
}
```

## Description/Caption
Lets take a look at `AnalyzeRemoteAsync` first, which tries to find a good description of an image.

```c#
private static async Task AnalyzeRemoteAsync(
    ComputerVisionClient computerVision, string imageUrl)
{
    if (!Uri.IsWellFormedUriString(imageUrl, UriKind.Absolute))
    {
        Console.WriteLine(
            "\nInvalid remoteImageUrl:\n{0} \n", imageUrl);
        return;
    }

    ImageAnalysis analysis =
        await computerVision.AnalyzeImageAsync(imageUrl, Features);
    DisplayResults(analysis, imageUrl);
}
```
We check that the URI is OK and send a request to Azure. `DisplayResults` simply prints the captions that Azure vision has found.
```c#
private static void DisplayResults(ImageAnalysis analysis, string imageUri)
{
    Console.WriteLine(imageUri);
    if (analysis.Description.Captions.Count != 0)
    {
        Console.WriteLine(analysis.Description.Captions[0].Text + "\n");
    }
    else
    {
        Console.WriteLine("No description generated.");
    }
}
```

## OCR

```c#
static async Task MakeOcrRequest(ComputerVisionClient computerVision, string imageUrl)
{
    try
    {
        var httpResponse = await computerVision.RecognizePrintedTextWithHttpMessagesAsync(true, imageUrl, OcrLanguages.Cs);
```
> Few remarks for this API call:
>
> * Parameter value *"OcrLanguages.cs"* is saing that the text is expected to be Czech. Does anything change if use the default value or switch to English?
> * For simplicity, we're running this method synchronously and blocking the thread. In real application you would use *async/await*.

After the call is done, we use LINQ to get specific words from the result:
```c#
        var words = httpResponse.Body.Regions.SelectMany(r => r.Lines)
            .SelectMany(l => l.Words)
            .Select(w => w.Text);
```
This is possibly more clear in query syntax,
```c#
        var words = from r in results.Regions
                    from l in r.Lines
                    from w in l.Words
                    select w.Text;
```
And print them to the console:
```c#
        Console.WriteLine($"OCR request: {imageUrl}");
        Console.WriteLine("OCR result: " + words.Aggregate((a, b) => a + "|" + b));
    }
    catch (Exception e)
    {
        Console.WriteLine("\n" + e.Message);
    }
}
```

If everything went well, your application should connect to the Cognitive Services Computer Vision API, send image URL to the OCR endpoint and write words separated by |.



Full code here:

```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.Azure.CognitiveServices.Vision.ComputerVision;
using Microsoft.Azure.CognitiveServices.Vision.ComputerVision.Models;

namespace ocr_cli
{
    class Program
    {
        private const string SubscriptionKey = "keykeykey";
        private const string ApiUri = "https://eastus.api.cognitive.microsoft.com/";

        // Specify the features to return
        private static readonly List<VisualFeatureTypes> Features =
            new List<VisualFeatureTypes>()
            {
                VisualFeatureTypes.Categories, VisualFeatureTypes.Description,
                VisualFeatureTypes.Faces, VisualFeatureTypes.ImageType,
                VisualFeatureTypes.Tags
            };
        static void Main(string[] args)
        {
            const string waterfall = "https://upload.wikimedia.org/wikipedia/commons/3/3c/Shaki_waterfall.jpg";
            const string skoda = "https://2.bp.blogspot.com/-SaoK-RZRX60/V1krBqEqXcI/AAAAAAAAcSY/TaMvJ8V6l4AkBlhmA-Z9noWcxBNNJ_PegCLcB/w800/2017-skoda-octavia-0.jpg";

            ComputerVisionClient computerVision = new ComputerVisionClient(
                new ApiKeyServiceClientCredentials(SubscriptionKey),
                new System.Net.Http.DelegatingHandler[] { });

            computerVision.Endpoint = ApiUri;

            Console.WriteLine("Images being analyzed ...");
            var t1 = AnalyzeRemoteAsync(computerVision, waterfall);
            var t2 = AnalyzeRemoteAsync(computerVision, skoda);
            var ocr = MakeOcrRequest(computerVision, skoda);

            Task.WhenAll(t1, t2, ocr).Wait(5000);
            Console.WriteLine("Press ENTER to exit");
            Console.ReadLine();
        }

        // Analyze a remote image
        private static async Task AnalyzeRemoteAsync(
            ComputerVisionClient computerVision, string imageUrl)
        {
            if (!Uri.IsWellFormedUriString(imageUrl, UriKind.Absolute))
            {
                Console.WriteLine(
                    "\nInvalid remoteImageUrl:\n{0} \n", imageUrl);
                return;
            }

            ImageAnalysis analysis =
                await computerVision.AnalyzeImageAsync(imageUrl, Features);
            DisplayResults(analysis, imageUrl);
        }


        // Display the most relevant caption for the image
        private static void DisplayResults(ImageAnalysis analysis, string imageUri)
        {
            Console.WriteLine(imageUri);
            if (analysis.Description.Captions.Count != 0)
            {
                Console.WriteLine(analysis.Description.Captions[0].Text + "\n");
            }
            else
            {
                Console.WriteLine("No description generated.");
            }
        }

        static async Task MakeOcrRequest(ComputerVisionClient computerVision, string imageUrl)
        {
            try
            {
                var httpResponse = await computerVision.RecognizePrintedTextWithHttpMessagesAsync(true, imageUrl, OcrLanguages.Cs);

                var words = httpResponse.Body.Regions.SelectMany(r => r.Lines)
                    .SelectMany(l => l.Words)
                    .Select(w => w.Text);

                Console.WriteLine($"OCR request: {imageUrl}");
                Console.WriteLine("OCR result: " + words.Aggregate((a, b) => a + "|" + b));
            }
            catch (Exception e)
            {
                Console.WriteLine("\n" + e.Message);
            }
        }
    }
}
```
