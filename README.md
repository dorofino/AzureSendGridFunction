[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

<!-- PROJECT LOGO -->
<br />
<div align="center">
<h2 align="center">Azure SendGrid Function</h3>

  <p align="center">
    How To Send Email Using SendGrid With Azure Function In .NET Core
    <br />
    <a href="https://github.com/dorofino/AzureSendGridFunction"><strong>Explore the docs »</strong></a>
    <br />
    <a href="https://github.com/dorofino/AzureSendGridFunction/issues">Report Bug</a>
    ·
    <a href="https://github.com/dorofino/AzureSendGridFunction/issues">Request Feature</a>
  </p>
</div>



<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>



<!-- ABOUT THE PROJECT -->
## About The Project

This article demonstrates how to send email from  **Azure Function**  using  **_SendGrid_**. We will build functions that will push and pull email content from queue *Azure Storage* and send emails to recipient. To implement this end-to-end workflow, we are going to create two functions:

Functions: 
1. A HTTP trigger that will create a queue storage and push some dummy email content into it.
2.  A queue trigger that will process each queue item and read respective email content from it and send email to the recipient using  _SendGridMessage_ client from the NuGet packages *SendGrid.Helpers.Mail*.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- GETTING STARTED -->
## Getting Started


### Prerequisites
-   Basic knowledge of Azure Function.
-   Azure and Azure Storage account already created.
-  [.NET Core 3.1](https://dotnet.microsoft.com/en-us/download/dotnet/3.1) installed in your PC.
-   Azure storage emulator installed. This is available as part of the  [Microsoft Azure SDK](https://azure.microsoft.com/downloads/). Check  [here](https://docs.microsoft.com/en-us/azure/storage/common/storage-use-emulator)  for more details.

## Function 1 (HTTP Trigger function)
### Push messages into the queue

The purpose of this function is to push some dummy email content into a Azure Storage queue so that when the queue trigger will happen later by another function, that function will read the content from the queue.

**NOTE:** _This function is (optional). 
If you are able to create your own dummy JSON data into a queue called "_email-queue_", then you can skip this function to create it. 
But I would recommend creating this function which will help you to understand the end to end flow._

Below sequential steps will be performed by this function :

1.  Create a queue called “_sendgrid-email-queue_” if it doesn't exist in your Azure Storage account.
 ![enter image description here](https://drive.google.com/uc?id=1psGmF2R0O3f_b91pr0K7DtBkQNAeR6wz)
3.  Create a JSON object with a unique Id and HTML email content through loop. This is for just a demo email content. Based on your need, you can create static or dynamic HTML email content.
4.  Add that JSON object as a queue message into “_sendgrid-email-queue_”.

Let's create a new Azure Function Project and add empty Azure Function app project from Visual Studio.
![enter image description here](https://drive.google.com/uc?id=1woTYErLZgDL0M9YrteUCudXnsxjOUG9B)
![enter image description here](https://drive.google.com/uc?id=1lP2Yw1z_16PHqT551y2RN8XKq1ArsC2j)

Once a newly created solution is loaded, 
right-click on function project 
=> Click on Add 
=> Click on New Azure Function
=> Give a function name as ***“DummyEmailMessagesHttpTriggerFunction.cs”*** 
![enter image description here](https://drive.google.com/uc?id=1uGxYFrBHbqDKfu67LHcJC6QLrLcKlujg)
=> Click on Add 
=> Select “Http Trigger” 
![enter image description here](https://drive.google.com/uc?id=1Cm9Vnq3DzWMXcOdR0tKXeH4anTIvRuiL)
=> Click on Create.
 
Next, install the required NuGet packages in the project.
-   _[Microsoft.Azure.WebJobs](https://www.nuget.org/packages/Microsoft.Azure.WebJobs)_
-   _[Microsoft.Azure.WebJobs.Extensions.Storage](https://www.nuget.org/packages/Microsoft.Azure.WebJobs.Extensions.Storage)_

Replace the default function with the following code into **DummyEmailMessagesHttpTriggerFunction.cs**

    using System;
    using System.IO;
    using System.Threading.Tasks;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Azure.WebJobs.Extensions.Http;
    using Microsoft.AspNetCore.Http;
    using Microsoft.Extensions.Logging;
    using Newtonsoft.Json;
    using Microsoft.Extensions.Configuration;
    using Microsoft.WindowsAzure.Storage.Queue;
    using Microsoft.WindowsAzure.Storage;
    
    namespace AzFunctions
    {
        public static class DummyEmailMessagesHttpTriggerFunction
        {
            [FunctionName("DummyEmailMessagesHttpTriggerFunction")]
            public static async Task<IActionResult> Run(
                [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req,
                ILogger log,
                ExecutionContext context)
            {
                log.LogInformation($"C# Http trigger function executed at: {DateTime.Now}");
    
                CreateQueueIfNotExists(log, context);
    
                for (int i = 1; i <= 5; i++)
                {
                    string randomStr = Guid.NewGuid().ToString();
                    var serializeJsonObject = JsonConvert.SerializeObject(
                                                 new
                                                 {
                                                     ID = randomStr,
                                                     Content = $"<html><body><h2> Sample HTML content {i}! </h2></body></html>"
                                                 });
    
                    CloudStorageAccount storageAccount = GetCloudStorageAccount(context);
                    CloudQueueClient cloudQueueClient = storageAccount.CreateCloudQueueClient();
                    CloudQueue cloudQueue = cloudQueueClient.GetQueueReference("sendgrid-email-queue");
                    var cloudQueueMessage = new CloudQueueMessage(serializeJsonObject);
    
                    await cloudQueue.AddMessageAsync(cloudQueueMessage);
                }
    
                return new OkObjectResult("Super DummyEmailMessagesHttpTriggerFunction executed successfully!");
            }
    
            private static void CreateQueueIfNotExists(ILogger logger, ExecutionContext executionContext)
            {
                CloudStorageAccount storageAccount = GetCloudStorageAccount(executionContext);
                CloudQueueClient cloudQueueClient = storageAccount.CreateCloudQueueClient();
                string[] queues = new string[] { "sendgrid-email-queue" };
                foreach (var item in queues)
                {
                    CloudQueue cloudQueue = cloudQueueClient.GetQueueReference(item);
                    cloudQueue.CreateIfNotExistsAsync();
                }
            }
            private static CloudStorageAccount GetCloudStorageAccount(ExecutionContext executionContext)
            {
                var config = new ConfigurationBuilder().SetBasePath(executionContext.FunctionAppDirectory)
                                                       .AddJsonFile("local.settings.json", true, true)
                                                       .AddEnvironmentVariables()
                                                       .Build();
                CloudStorageAccount storageAccount = CloudStorageAccount.Parse(config["AzureCloudStorageAccount"]);
                return storageAccount;
            }
        }
    }


### Running First Function
Run the function and copy the HTTP URL from the function console window and paste it into the browser.
![enter image description here](https://drive.google.com/uc?id=10uVLCqAd3EkNRrWsXGDS04BtsBdDRaQm)

Once it is executed you will see the message 
**“_Super DummyEmailMessagesHttpTriggerFunction executed successfully!_”.**
![enter image description here](https://drive.google.com/uc?id=1mJxumR8M_KYhcbMXFTfXISFwYus4LT7M)

Great jobs, now lets verify the storage account.
=> Open Azure Storage Explorer
=> Storage Accounts
=> (Emulator - Default Ports) (Keys)
=> Queues
=> sendgrid-email-queue
On the right window you will see the created new queues with a message ID and some sample messages.
![enter image description here](https://drive.google.com/uc?id=1HGbHmpUqQxIIYVJPoY51x7rU46mSlkc8)

<p align="right">(<a href="#readme-top">back to top</a>)</p>


