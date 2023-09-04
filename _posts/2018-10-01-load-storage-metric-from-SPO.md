---
layout      : single
title       : Load storage metric from SharePoint Online site collection
summary     : Introduce how to load storage metric with CSOM and rest api
categories  : SharePointOnline
tags        : [SharePoint Online, CSOM, Storage Metric]
date        : 2018-10-01 23:00:00
commentId   : 1
permalink   : /load-storage-metric-from-SPO
toc         : true
classes     : wide
toc_icon    : "cog"
toc_label   : "My Table of Contents"
---

I have a requirement to query the storage metric information from SharePoint Online site collection, so basicially I try to use the following code without any luck.

```cs
///Bad sample
using (var clientContext = InitializeContext(context))
{
    var storageMetrics = clientContext.Web.RootFolder.StorageMetrics;

    clientContext.Load(storageMetrics);
    clientContext.ExecuteQuery();
    Console.WriteLine(storageMetrics.TotalSize);
    //Error, no information is returned!!!
}
```

Then I tried to remember the API from the On-Premise SharePoint, and we need to enable the **SPWeb.IncludeStorageMetrics** before retrieving the storage metrics information. However there is no property Web.IncludeStorageMetrics in CSOM API!!! So how to figure out this? Let's try to research the SharePoint On-Premise Code to know who enable the property in the background for the CSOM.

With free decompile tool, I found the following code in **SPFolder** class, and it means you need to load the Folder and the StorageMetrics property together to retrieve this information.

```cs
internal void OnQuerying(ClientQuery query, ClientQuery childItemQuery)
{
    if (query != null && query.ContainsProperty("StorageMetrics"))
    {
        this.m_Web.IncludeStorageMetrics = true;
    }
}
```

So I changed the code like this, and I finally got the report!

```cs
using (var clientContext = InitializeContext(context))
{
    var rootFolder = clientContext.Web.RootFolder;

    clientContext.Load(rootFolder, f => f.StorageMetrics);
    clientContext.ExecuteQuery();
    Console.WriteLine(rootFolder.StorageMetrics.TotalSize);
}
```

And the REST API sample is 

https://tenant.sharepoint.com/_api/web/rootfolder?$select=storagemetrics&$expand=storagemetrics