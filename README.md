# Backup to Dropbox Automation - A Formula Tutorial

This tutorial accompanies the [Automating workflows for your business apps using Cloud Elements](https://dropbox.tech/developers) post featured on the Dropbox Developer blog. This repo along with the linked blog provides a template for creating an automated business application data backup process using Hubspot as an example source and Dropbox as a destination. For more background and context check it out!

## Prerequisites
This tutorial assumes the reader has accounts on Dropbox, HubSpot, and Cloud Elements. Each product has a free tier, so please jump in and build along with us! 
Here’s what you’ll need:

- A free account from Cloud Elements to build and deploy our integration
- A Dropbox app that is scoped to “Dropbox API → Full Dropbox”
- A free HubSpot developer account

Nice to haves:

- Familiarity with JSON, JavaScript, and public APIs

## Getting Started

To connect your Dropbox and Hubspot accounts to Cloud Elements [create Element Instances](docs.cloud-elements.com/cretae-an-instance) for each. An Element Instance is a normalized API key that represents the required credentials you would need to access a particular service. 

With your Element instances created, the file in this repo are the only additional things needed to get up and running. However this tutorial utilizes the Cloud Elements asset management CLI as it walks through the contents of the repo and the steps to activate the automation in Cloud Elements.

Quick setup via npm is as follows:
```
$ npm install -g ce-util
$ doctor init
```

> More detailed instructions for installing the CLI can be found here: https://github.com/CloudElementsOpenLabs/the-doctor


Be sure to have Cloud Elements org and user secrets available, which can be retrieved from the user menu in the [Cloud Elements UI](https://docs.cloud-elements.com/home/authentication).

### Formulas Summary

Formulas are reusable workflow templates that are completely independent of API providers. Each Formula is an independent bundle of logic; a trigger(s) that kick off a series of steps. More context and documentation for Formulas can be found at the [Cloud Elements Developer Docs](https://docs.cloud-elements.com/home/introduction-to-formulas), but the main components used in this tutorial are as follows:

* [Triggers](https://docs.cloud-elements.com/home/triggers)
  * [Scheduled](https://docs.cloud-elements.com/home/triggers#scheduled)
  * [Manual](https://docs.cloud-elements.com/home/triggers#manual)
* [Configuration Variables](https://docs.cloud-elements.com/home/formula-variables)
  * Element Instance Variables
* [Steps](https://docs.cloud-elements.com/home/formula-step-types)
  * Element Request
  * Script
  * Stream File

# Solution

As explained in the linked blog, this repository contains two separate formulas, one for each half of our automation solution.

1. [start-hubspot-bulk-job](/start-hubspot-bulk-job): get contact data from Hubspot and consolidate into a single file
2. [bulk-listener-to-dropbox](/bulk-listener-to-dropbox): upload the file to dropbox

## Start Hubspot Bulk Job Formula

This first formula will begin the asynchronous process of compiling a file of data from the Hubspot account source so that it can easily be streamed to the Dropbox folder where backup data will be stored.

### Configuration Variables

Read more about configuration variables here: docs.cloud-elements

This formula has just one configuration property, an Element Instance variable for the Hubspot account that will be the source of the bulk download. This configuration variable is found in `/start-hubspot-bulk-job/formula.json`:

```
    "configuration": [{
        "key": "HubSpot.source",
        "name": "HubSpot Source Account",
        "type": "elementInstance",
        "description": "Source HubSpot instance to backup from",
        "required": true
    }]
```

### Triggers

Read more about Formula Triggers here: docs.cloud-elements.com

This formula uses a scheduled trigger property with a cron statement that evaluates to every Monday at 1am. This trigger is found in `/start-hubspot-bulk-job/formula.json`:

```
    "triggers": [{
        "name": "trigger",
        "type": "scheduled",
        "onSuccess": [],
        "properties": {
            "cron": "0 0 01 ? * MON *"
        }
    }]
```

### Steps

Read more about Formula Step types here: docs.cloud-elements.com
Read more about using JavaScript in Formulas here: docs.cloud-elements.com

The steps array is the primary definition of what should be done and in what order. This formula uses 2 steps:
1. createQuery - a JavaScript step to dynamically set properties used in step 2.
2. createBulkJob - an Element Request step to initiate an asynchronous bulk data download 

Find references to both steps in `/start-hubspot-bulk-job/formula.json` but find the referenced JavaScript in its own file `/start-hubspot-bulk-job/createRequestProperties.js`.

Notice that in the below JavaScript snippet there are two URLs for use as callbackURLs. The one that is commented out will be our production version that will reference the `bulk-listener-to-dropbox` formula once it is created. The uncommented URL references [requestbin](https://requestbin.com) for use as a testing destination before the second formula is created.

```
const queryStatement = "select * from contacts";
// const callbackURL = `/formulas/instances/:id/executions`; USE LATER
const callbackURL = "https://requestbin.com/r/{yourAutoGeneratedBinId}";

done({
    "body":{
        "query": queryStatement,
    },
    "headers":{
        "Elements-Async-Callback-Url": callbackURL
    },
});
```

> Note that JavaScript step filenames must match the step's `name` property in the formula.json file.

### Uploading and Testing

Our first formula is complete! Let’s fire up that CLI to upload the formula to your Cloud Elements account.

```
$ doctor upload formulas {accountName} --dir ./automated-dropbox-uploader
```

> Tip: run the command `doctor accounts list` to see the account name you set up earlier.

Jot down the returned Formula ID found in the doctor success message to use in creating an instance of the Formula.


```    
    curl -X POST "https://staging.cloud-elements.com/elements/api-v2/formulas/{formulaId}/instances" 
    -H "accept: application/json" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json" 
    -d "{ \"active\": true, \"configuration\": { \"HubspotSource\": \"{HubSpotInstanceID}\" }, \"name\": \"my hubspot bulk job test\"}"
```

The response body from that request will contain an `id` field, this is your formula instance ID. Even though it’s also scheduled to trigger on Mondays at 1AM, we can use this instance id to trigger this formula manually. Since it is usually set to a chron trigger we don’t have to pass any context in the POST body, but we can still test it with the following API


```    
- curl -X POST "https://staging.cloud-elements.com/elements/api-v2/formulas/instances/{instanceIDFromPrevResponse}/executions" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json"
    -H "accept: application/json"  
    -d "{}"
```

After triggering a formula execution, revisit your RequestBin and take a look at the message that you received. There’s a useful summary of the bulk job, but most importantly is the `id` field. Take note of this as we’ll be making use of it in part two.

We’re done with the first half of our integration app, now all we have to do is upload the results of that Bulk job to Dropbox. We’ll write another short formula for that and then tie the two together.

## Bulk Listener to Dropbox Formula

This second formula will listen for the completion of the asynchronous data compiling process and then upload the file to Dropbox. To do this it will use a the built in formula [stream step]().

### Configuration Variables

```
"configuration": [{
        "key": "HubSpotSource",
        "name": "HubSpot Source Account",
        "type": "elementInstance",
        "description": "Source HubSpot instance to take backup from",
        "required": true
    },{
        "key": "DropboxTarget",
        "name": "Dropbox Source Account",
        "type": "elementInstance",
        "description": "Destination Dropbox account to save files in",
        "required": true
    }]
```

### Triggers

Let’s add a manual trigger to this formula which will listen for the webhook. And don’t forget to update the `streamBulkResults` step `downloadApi` property to use the job ID that will come from the trigger.

```
"triggers": [{
    "type": "manual",
    "name": "trigger",
    "properties": {},
    "onSuccess": ["streamBulkResults"],
    "onFailure": [],
}]
```

### Steps

Let’s add a manual trigger to this formula which will listen for the webhook. And don’t forget to update the `streamBulkResults` step `downloadApi` property to use the job ID that will come from the trigger.

```
{
    "name": "streamBulkResults",
    "type": "elementRequestStream",
    "properties":{
        "downloadMethod": "GET",
        "downloadApi": "/bulk/{bulkJobId}/contacts",
        "downloadElementInstanceId": "${config.HubSpotInstance}",
        "uploadMethod": "POST",
        "uploadApi": "/files"
        "uploadElementInstanceId": "${config.dropboxInstance}"
    },
    "onSuccess": [],
    "onFailure": [],
}
```

### Upload and Test

Now we have our formula for phase 2. Let’s deploy that using the upload function from the Cloud Elements CLI.

```
$ doctor upload formulas {accountName} --dir ~/path/to/this/directory
```

Like with our last formula, we’ll create an instance so that we can have a specific version of this formula listening to the specific version of the first formula. If we decide to automate backups for all of our data in HubSpot, not just Contact data, we could have several instances of both of these formulas. 

```   
    curl -X POST "https://{yourAccountDomain}.cloud-elements.com/elements/api-v2/formulas/{formulaID}/instances" 
    -H "accept: application/json" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json" 
    -d "{ \"active\": true, \"configuration\": { \"HubspotSource\": \"{HubSpotInstanceID}\" }, \"name\": \"Hubspot to Dropbox Upload\"}"
```

The response to our `curl` command will look something like this:

```
    [
        {
            "id": 428520,
            "formula": {
                "id": 32968,
                "active": true,
                "debugLoggingEnabled": false,
                "singleThreaded": false
            },
            "name": "Hubspot to Dropbox Upload",
            "createdDate": "2020-02-18T17:47:54Z",
            "configuration": {
                "HubspotSource": "48902",
                "DropboxTarget": "50134"
            },
            "active": true,
            "settings": {}
        }
    ]
```

With the first class ID that came back (`428520`), update your first formula instance so that it knows which formula to call on when it’s done. (replace RequestBin with the live formula).

Now we have instances, but our instance won’t be triggered until next Monday, so let’s test it by triggering it via API like we did at the end of phase 1.

```    
curl -X POST "https://staging.cloud-elements.com/elements/api-v2/formulas/{formulaId}/instances/{instanceIDFromPreviousResponse}/executions" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json"
    -H "accept: application/json"  
    -d "{}"
```

It might take a minute or two, but when we check Dropbox we should see the new file was created! We can also ping Cloud Elements to check on the status of the execution. 

```
curl "https://staging.cloud-elements.com/elements/api-v2/formulas/{formulaId}/instances/{instanceId}/executions/{id}"
```