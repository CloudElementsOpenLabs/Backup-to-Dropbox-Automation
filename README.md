# Backup to Dropbox Automation - A Formula Tutorial

This tutorial accompanies the [Automating workflows for your business apps using Cloud Elements](https://dropbox.tech/developers) post featured on the Dropbox Developer blog. This repo along with the linked blog provides a template for creating an automated business application data backup process using Hubspot as an example source and Dropbox as a destination. For more background and context check it out!

## Prerequisites
This tutorial assumes the reader has accounts on Dropbox, HubSpot, and Cloud Elements. Each company has a free tier, so please jump in and build along with us! 
Here’s what you’ll need:

- A free account from Cloud Elements to build and deploy our integration
- A Dropbox app that is scoped to “Dropbox API → Full Dropbox”
- A test account created from a free HubSpot developer account

Nice to haves:

- Familiarity with JSON, JavaScript, and public APIs

## Getting Started

With the above accounts set up, the file in this repo are the only additional things needed to get up and running. However this tutorial utilizes the Cloud Elements asset management CLI as it walks through the contents of the repo and the steps to activate the automation in Cloud Elements.

Quick setup via npm is as follows:
```
$ npm install -g ce-util
$ doctor init
```

> More detailed instructions for installing the CLI can be found here: https://github.com/CloudElementsOpenLabs/the-doctor


Be sure to have Cloud Elements org and user secrets available, which can be retrieved from the user menu in Cloud Elements as shown below:

![](https://paper-attachments.dropbox.com/s_D556D2D96CED5F08D2B3111161D681753640B68BB062CAB3662E4FB1198A0B5B_1580856358552_image.png)

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

Now that we’re setup, we can begin configuring our formula with the `configuration` property. The properties stored in this array are essentially global variables we can use to help specify what the formula does when it runs. We may want to make our formula even more generic in the future so we’ll likely add more `configuration` variables to these. For now we only need to store one thing here: the HubSpot credentials we’ll be using. 

An Element Instance is a normalized API key that represents the required credentials you would need to access a particular service. This is a good thing to store as a global variable in our formula because we will likely want to test our formula with one set of HubSpot API keys (for a sandbox) and then easily switch to live HubSpot API keys when we’re ready. And by making our HubSpot credentials a configuration variable we’ll be ready to reuse our data collection app for backing up other services beyond HubSpot.

Let’s add that basic configuration data for credentials to our Formula JSON file:

    // formula.json
    {
        "name": "start-hubspot-bulk-job",
        "description": "Start a bulk download from HubSpot",
        "engine": "v3",
        "configuration": [{
            "key": "HubSpot.source",
            "name": "HubSpot Source Account",
            "type": "elementInstance",
            "description": "Source HubSpot instance to backup from",
            "required": true
        }],
        "triggers": [],
        "steps": []
    }

### Triggers

We mentioned these earlier, this is how we will tell the formula when to run. Formulas can have several different types of Triggers. For our use case we’re going to use a scheduled trigger with a cron statement so that the formula performs our backup every Monday at 1am. This way the backup will be ready for use before the work week starts, and if it goes wrong somehow, we don’t have to check on it over the weekend.

Let’s add this cron trigger object to our formula:


```
    {
        "name": "start-hubspot-bulk-job",
        "description": "Start a bulk download from HubSpot",
        "engine": "v3",
        "configuration": [{
            "key": "HubspotSource",
            "name": "HubSpot Source Account",
            "type": "elementInstance",
            "description": "Source HubSpot instance to backup from",
            "required": true
        }],
        "triggers": [{
            "name": "trigger",
            "type": "scheduled",
            "onSuccess": [],
            "properties": {
                "cron": "0 0 01 ? * MON *"
            }
        }],
        "steps": []
    }
```

### Steps

The steps array is the primary definition of what should be done and in what order.

All we really need is this first formula to start the bulk download for HubSpot data. But before making a request to that API, we’re going to define the properties of that request in a Javascript file. This way we can easily, and perhaps dynamically, change them in the future. Find the following code in the `createRequestProperties.js` file of the formula folder.

```
const queryStatement = "select * from contacts";
const bulkListenerFormulaInstanceID = 00000;
const callbackURL = `/formulas/instances/${bulkListenerFormulaInstanceID}/executions`; 
// const callbackURL = "https://requestbin.com/r/{yourAutoGeneratedBinId}"; FOR TESTING

done({
    "body": {
        "query": queryStatement
    },
    "headers": {
        "Elements-Async-Callback-Url": callbackURL
    }
});
```


If we refer back to the documentation on the Cloud Elements Bulk API, there are multiple properties that we can use to configure a bulk job which can be fine-tuned in the future. For now, let’s try to do a complete backup of all of our business’ contacts. We’ll pass a very simple `queryStatement` variable to the request. The only other thing we need is the callback URL where we want the “job completed” notification to go. Since we’re going to use another formula for this, we already defined the path for the formula trigger in the `createRequestProperties.js` file:


    /formulas/instances/:id/executions

Source: docs.cloud-elements.com/home/triggers#manual

The only part we don’t know is the ID of that formula instance, because we haven’t created it yet. Let’s put a RequestBin URL there (or another URL you can monitor) for some quick visibility. A public RequestBin URL can be created without registering, which will be helpful for testing our first formula. 


> Remember to replace the `{yourAutoGeneratedBinId}` part of the callbackURL string with the actual ID that was generated when you created the RequestBin or replace the whole string with the path of your monitored URL. 

The `done()` method passes an object to the next step of our formula. But before that, we need to tell our formula where to find that script file. Add the following object to the `steps` array in our `formula.json` file.


    // formula.json
    {
        "name": "start-hubspot-bulk-job",
    ...
        "steps": [{
            "name": "createRequestProperties",
            "type": "script",
            "onSuccess": [],
            "onFailure": []
        }]
    }

As long as the step’s `name` field matches the filename in our root directory, the Cloud Elements CLI will link the formula step and the script file. And, before we forget, let’s update the `trigger` object so the formula knows it should start with the `createRequestProperties` step after the trigger has been activated. To do this we’ll add the name of our step to the `onSuccess` array.


    // formula.json
    {
        "name": "start-hubspot-bulk-job",
    ...
        "triggers": [{
            "name": "trigger",
            "type": "scheduled",
            "onSuccess": ["createRequestProperties"],
            "properties": {
                "cron": "0 0 01 ? * MON *"
            }
        }],
        "steps": [{
            "name": "createRequestProperties",
            "type": "script",
            "onSuccess": [],
            "onFailure": []
        }]
    }

The last step in this formula will be to make the API request which initiates the bulk job with the properties we specified in our script. For this we essentially just need to make an HTTP request to the Cloud Elements Bulk API.

We could write another script that uses a common library for this, but I already mentioned that there were some helpful shorthand steps that the formula provides. The next object we add to the `steps` array will have a `type` of `elementRequest` which will abstract our HTTP request to the Cloud Elements Bulk API. Add the following object to the `steps` array:


    // forumla.json
    {
        "name": "start-hubspot-bulk-job",
        "description": "Start a bulk download from HubSpot",
    ...
    "steps": [{
            "name": "createRequestProperties",
            "type": "script",
            "onSuccess": [],
            "onFailure": []
        }, {
            "name": "createBulkJob",
            "type": "elementRequest",
            "properties": {
                "method": "POST",
                "headers": "${steps.createQuery.headers}",
                "body": "${steps.createQuery.body}",
                "api": "/bulk/query",
                "elementInstanceId": "${config.HubspotSource}"
            }
        }]
    }

Lastly, remember to add the name of this step to the `onSuccess` property of our previous step. Our final `formula.json` file should look like this:


    // start-hubspot-bulk-job/formula.json
    {
        "name": "start-hubspot-bulk-job",
        "description": "Start a bulk download from HubSpot",
        "engine": "v3",
        "configuration": [{
            "key": "HubSpot.source",
            "name": "HubSpot Source Account",
            "type": "elementInstance",
            "description": "Source HubSpot instance to take backup from",
            "required": true
        }],
        "triggers": [{
            "name": "trigger",
            "type": "scheduled",
            "onSuccess": ["createQuery"],
            "properties": {
                "cron": "0 0 01 ? * MON *"
            }
        }],
        "steps": [{
            "name": "createQuery",
            "type": "script",
            "onSuccess": ["createBulkJob"],
            "onFailure": [],
            "properties": {}
        }, {
            "name": "createBulkJob",
            "type": "elementRequest",
            "properties": {
                "method": "POST",
                "headers": "${steps.createQuery.headers}",
                "body": "${steps.createQuery.body}",
                "api": "/bulk/query",
                "elementInstanceId": "${config.HubSpotInstance}"
            }
        }]
    }

We’ve told the `elementRequest` step to make a `POST` request with the `headers` and `body` that we defined in our script step. We’ve sent this POST request to the generic bulk API because we also provided the `elementInstanceId` from our HubSpot Instance for Cloud Elements to make the correct API calls to HubSpot and compile our CSV file.

Our first formula is complete! Let’s fire up that CLI to upload the formula to your Cloud Elements account.


    $ doctor upload formulas {accountName} --dir ./automated-dropbox-uploader

Note: run `doctor accounts list` to see the account name you set up earlier.

Nice work! Let’s test this first phase before we move on to uploading our results to Dropbox. At this point if we run our formula that we just uploaded, we should get a message in the RequestBin when our formula and our Bulk job is done. But first we need an instance of our Formula that is attached to our HubSpot account, probably a sandbox account, but if you want to start testing with your live data, go crazy!


    // sample curl command to create formula instance
    
    curl -X POST "https://{yourAccountDomain}.cloud-elements.com/elements/api-v2/formulas/{formulaID}/instances" 
    -H "accept: application/json" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json" 
    -d "{ \"active\": true, \"configuration\": { \"HubspotSource\": \"{HubSpotInstanceID}\" }, \"name\": \"my hubspot bulk job test\"}"

The response body from that request will contain an `id` field, this is your formula instance ID. Even though it’s also scheduled to trigger on Mondays at 1AM, we can use this instance id to trigger this formula manually. Since it is usually set to a chron trigger we don’t have to pass any context in the POST body, but we can still test it with the following API


    // sample curl to trigger formula instance execution
    
- curl -X POST "https://{yourAccountDomain}.cloud-elements.com/elements/api-v2/formulas/instances/{instanceIDFromPrevResponse}/executions" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json"
    -H "accept: application/json"  
    -d "{}"

After triggering a formula execution, revisit your RequestBin and take a look at the message that you received. There’s a useful summary of the bulk job, but most importantly is the `id` field. Take note of this as we’ll be making use of it in part two.

We’re done with the first half of our integration app, now all we have to do is upload the results of that Bulk job to Dropbox. We’ll write another short formula for that and then tie the two together.

Phase 2: Upload Results to Dropbox

This second half of our process needs to be triggered by the webhook sent from the bulk job when it’s done compiling our CSV file. Once we get the notification, we’ll need a formula to download the CSV and then upload it to the Dropbox API. Again, this could be a bit tricky, but the Formulas tool has a pre-built step that can stream a file from one API to another. Let’s go back to our main `cloudelements-formulas` directory and create a new sub-directory for this second formula:


    $ mkdir bulk-listener-to-dropbox
    $ cd bulk-listener-to-dropbox
    $ touch formula.json


![current directory structure](https://paper-attachments.dropbox.com/s_D556D2D96CED5F08D2B3111161D681753640B68BB062CAB3662E4FB1198A0B5B_1580419894896_image.png)


Start with this basic formula object below:


    // formula.json
    {
        "name": "bulk-download-listener-to-dropbox",
        "description": "Stream a file from bulk download to Dropbox folder",
        "engine": "v3"
        "configuration": [],
        "triggers": [],
        "steps": []
    }

We know we’re going to need to add some configurations and a trigger similar to the first formula, but let’s first add the stream step I mentioned above to clarify what we’ll need to make it work. Add the following stream step object to the `steps` array.


    // formula.json
    {
        "name": "bulk-download-listener-to-dropbox",
        "description": "Stream a file from bulk download to Dropbox folder",
        "engine": "v3"
        "configuration": [],
        "triggers": [],
        "steps": [{
        "name": "streamBulkResults",
        "type": "elementRequestStream",
        "properties": {
            "downloadMethod": "",
            "downloadApi": "",
            "downloadElementInstanceId": "",
            "uploadMethod": "",
            "uploadApi": ""
            "uploadElementInstanceId": ""
          },
        "onSuccess": [],
        "onFailure": [],
        }]
    }

There are a few more advanced properties that we may put to use later, but primarily we are just telling the step what API it should use to download a file from, and which API it should stream that file to in order to upload it to a destination. The download properties will point to the Cloud Elements Bulk API that we used to create our CSV file and the upload properties will point to our Dropbox instance.


    // formula.json
    {
        "name": "bulk-download-listener-to-dropbox",
        "description": "Stream a file from bulk download to Dropbox folder",
        "engine": "v3"
        "configuration": [],
        "triggers": [],
        "steps": [{
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
        }]
    }


The two things we need here are the Element instances that point to the credentials for your HubSpot and Dropbox accounts. The other piece we need is the ID of the bulk job that is part of the download API path. The bulk download API requires the ID of the bulk job because you could have jobs running or recently finished that you wouldn’t want to get confused with. This ID comes from the asynchronous webhook provided when the bulk job is finished like we saw in our RequestBin at the end of phase 1. Let’s add a manual trigger to this formula which will listen for the webhook. And don’t forget to update the `streamBulkResults` step `downloadApi` property to use the job ID that will come from the trigger.



    // formula.json
    {
        "name": "bulk-download-listener-to-dropbox",
        "description": "Stream a file from bulk download to Dropbox folder",
        "engine": "v3"
        "configuration": [],
        "triggers": [{
          "type": "manual",
          "name": "trigger",
          "properties": {},
          "onSuccess": ["streamBulkResults"],
          "onFailure": [],
        }],
        "steps": [{
        "name": "streamBulkResults",
        "type": "elementRequestStream",
        "properties":{
            "downloadMethod": "GET",
            "downloadApi": "/bulk/${trigger.args.id}/contacts",
            "downloadElementInstanceId": "${config.HubSpotInstance}",
            "uploadMethod": "POST",
            "uploadApi": "/files?path={/pathToYourBackupFolder}"
            "uploadElementInstanceId": "${config.dropboxInstance}"
          },
        "onSuccess": [],
        "onFailure": [],
        }]
    }

Now, we could just point our trigger directly at the `streamBulkResults` step, but as we start running this backup more often, we may occasionally start running into errors in our bulk job. Maybe our HubSpot credential gets revoked, or maybe the Contacts API is offline for maintenance while the Bulk job is running against it. Whatever the case, it’s a good idea to make sure the bulk job was successful before moving on to the stream step.  

… add script step between trigger and stream step (or adjust paragraph above)

The last pieces to add to this formula are our `configurations` . We’ll need one for our HubSpot account as the data source and one for our Dropbox account which will be our data destination. Add these objects to the `formula.json` file:


    // formula.json
    {
        "name": "bulk-download-listener-to-dropbox",
        "description": "Stream a file from bulk download to Dropbox folder",
        "engine": "v3"
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
        }],
        "triggers": [{
          "type": "manual",
          "name": "trigger",
          "properties": {},
          "onSuccess": ["streamBulkResults"],
          "onFailure": [],
        }],
        "steps": [...]
    }

Now we have our formula for phase 2. Let’s deploy that using the upload function from the Cloud Elements CLI.


    $ ceutil upload formulas snapshot-dev --dir ~/cloudelements/formulas

Like with our last formula, we’ll create an instance so that we can have a specific version of this formula listening to the specific version of the first formula. If we decide to automate backups for all of our data in HubSpot, not just Contact data, we could have several instances of both of these formulas. 


    // sample curl command to create formula instance
    
    curl -X POST "https://{yourAccountDomain}.cloud-elements.com/elements/api-v2/formulas/{formulaID}/instances" 
    -H "accept: application/json" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json" 
    -d "{ \"active\": true, \"configuration\": { \"HubspotSource\": \"{HubSpotInstanceID}\" }, \"name\": \"Hubspot to Dropbox Upload\"}"

The response to our `curl` command will look something like this:


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

With the first class ID that came back, update your first formula instance so that it knows which formula to call on when it’s done. (replace RequestBin with the live formula).

Now we have instances, but our instance won’t be triggered until next Monday, so let’s test it by triggering it via API like we did at the end of phase 1.


    // sample curl to trigger formula instance execution
    
    curl -X POST "https://{yourAccountDomain}.cloud-elements.com/elements/api-v2/formulas/{id}/instances/{instanceIDFromPrevResponse}/executions" 
    -H "Authorization: User {userToken}, Organization {organizationToken}" 
    -H "Content-Type: application/json"
    -H "accept: application/json"  
    -d "{}"

It might take a minute or two, but when we check Dropbox we should see the new file was created! We can also ping Cloud Elements to check on the status of the execution. 


    curl "https://{yourAccountDomain}.cloud-elements.com/elements/api-v2/formulas/{id}/instances/{id}/executions/{id}"
