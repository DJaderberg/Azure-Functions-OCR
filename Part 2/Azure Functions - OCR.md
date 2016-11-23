# Azure Functions - OCR

We will now extend the application from [Part 1] and create a function that will listen to a Storage Queue, take an image link from it, process through OCR and save results to Storage Table.

## Portal

You can use the Azure Management Portal or CLI. In this tutorial, we will show both with more focus on the Portal.

First, we need to prepare a Storage Account. Azure Storage provides us with three types of storage: Blobs, Tables and Queues. We will need queue and table.

1. Go to the **Resource Group** you created in Part 1
2. Add new **Storage Account**
   1. Call it in a way that you will recognize it later
   2. Select **Resource Manager** as Deployment model
   3. Select **General purpose** as Account kind
   4. Select **Standard** as Performance
   5. Select **Locally-redundant storage (LRS)** as Replication
3. Download and install [Azure Storage Explorer](http://storageexplorer.com/) 
4. Log in to your Azure Account
5. List Storage Accounts and find your newly created account
   1. Go to **Queues**, right click and select **Create Queue**
   2. Call it **images**

Now we're ready to start integrating with Azure Functions. This function will listen to a queue and send every message to Azure Cognitive Services OCR.

1. Go back to your **Resrouce Group**
2. Add new **Function App**

1. 1. Give it a unique name
   2. Select **Consumption** plan
2. Choose C# template triggered by Queue
3. ...

The Integrate tab should already have one trigger preared for you: Azure Queue Storage. Select it and change **Message parameter name** to `imageUrl`, **Queue name** to `images` and add new connection to your Storage Account (click **new** and select specific account).

To easily output to a storage table, you need to add new Output. Click **New Output** and select **Azure Table Storage**. **Table parameter name** will be `outputName`, **Table name** will be `outTable`, **Storage account** will be again your account.

Integrations are ready and we can start coding the function itself.

Go to **Develop** and start by referencing assemblies:

```c#
#r "System.Threading.Tasks"
#r "System.IO"
#r "System.Runtime"
```

Add usings:

```c#
using System;
using System.Linq;
using Microsoft.ProjectOxford.Vision;
using Microsoft.ProjectOxford.Vision.Contract;
```

You can see that we're using namespace `Microsoft.ProjectOxford.Vision`, which isn't part of libraries loaded by default. To make Functions download and install the corresponding NuGet package, you need to create a *project.json* file in the same folder.

1. Click **View Files** on the top right side of the editor
2. Click **Add**
3. Name the file **project.json**

Project.json contents will be the following:

```json
{
  "frameworks": {
    "net46":{
      "dependencies": {
        "Microsoft.ProjectOxford.Vision": "1.0.370"
      }
    }
   }
}
```

> If you're not sure what is the latest version, simply look into Visual Studio or NuGet website.

Moving on, we can start building our function in the run.csx file.

```c#
public static void Run(string imageUrl, ICollector<OCRRes> outputTable, TraceWriter log)
{
  
}
```

Run method signature is a little different from default - we added the output table reference.

Then we can log what's happened:

```c#
log.Info($"C# Queue trigger function processed: {imageUrl}");
```

And insert code which is similar to the version we used in Part 1:

```c#
string apiKey = "keykeykey";

VisionServiceClient visionClient = new VisionServiceClient(apiKey);

OcrResults results = visionClient.RecognizeTextAsync(imageUrl, "cs").Result;

var words = from r in results.Regions
            from l in r.Lines
            from w in l.Words
            select w.Text;
```

Finally, we're adding the resulting words to a table:

```c#
outputTable.Add(
    new OCRRes() { 
        PartitionKey = "OCR", 
        RowKey = Guid.NewGuid().ToString(), 
        Result = String.Join("|", words.ToArray()) 
    }
);
```

To make this work, don't forget to create a helper class below the Run method.

```c#
public class OCRRes {
    public string PartitionKey { get; set; }
    public string RowKey { get; set; }
    public string Result { get; set; }
}
```

This isn't absolutely necessary, but we're showing here that it's possible :)

Final code for the functions looks like this:

```c#
#r "System.Threading.Tasks"
#r "System.IO"
#r "System.Runtime"

using System;
using System.Linq;
using Microsoft.ProjectOxford.Vision;
using Microsoft.ProjectOxford.Vision.Contract;

public static void Run(string imageUrl, ICollector<OCRRes> outputTable, TraceWriter log)
{
    log.Info($"C# Queue trigger function processed: {imageUrl}");
    
    string apiKey = "keykeykey";

    VisionServiceClient visionClient = new VisionServiceClient(apiKey);

    OcrResults results = visionClient.RecognizeTextAsync(imageUrl, "cs").Result;

    var words = from r in results.Regions
                from l in r.Lines
                from w in l.Words
                select w.Text;

    outputTable.Add(
        new OCRRes() { 
            PartitionKey = "OCR", 
            RowKey = Guid.NewGuid().ToString(), 
            Result = String.Join("|", words.ToArray()) 
        }
    );
}

public class OCRRes {
    public string PartitionKey { get; set; }
    public string RowKey { get; set; }
    public string Result { get; set; }
}
```



## CLI

`mkdir OCR-Demo`

`cd OCR-Demo`

`func init`

`func new`

- zvolit filtr podle jazyka
- zvolit jazyk
- zvolit trigger
- zvolit šablonu
- zadat název funkce

`code .`

`func run`

Vytvořit Storage Account

Vytvořit blob container images

`func azure login`

`func azure subscription set [id]`

`func azure storage list`

`func settings add-storage-account [name]`

?? upravit ručně connection string ??