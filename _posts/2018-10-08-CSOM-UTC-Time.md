---
layout      : single
title       : Field Value UTC Time vs Unspecial Time in CSOM?
summary     : Understand the field value is UTC Time or not
categories  : SharePointOnline
tags        : [SharePoint Online, CSOM, Field Value]
date        : 2018-10-08 22:00:00
commentId   : 2
permalink   : /CSOM-UTC-Time
toc         : true
classes     : wide
toc_icon    : "cog"
toc_label   : "My Table of Contents"
---

SharePoint provides date time filed to let user to set the date time for the list item or document, and you can use to CSOM API to update the date time field value, however what kind of time can we use to update the time? UTC or local Time? Let's figure out.

First I created a DateTime column **Last Saved Time**, and made the Today's Date to the default value. Then created one document.

![DateTime Field Value](/assets/img/SPO-2018-10-08-DateTimeFieldValue.png)

Let's try the all the available code to retrieve the field value.

```cs
using (var clientContext = InitializeContext(context))
{
    var list = clientContext.Web.Lists.GetByTitle("Documents");
    clientContext.Load(list, l => l.RootFolder);
    clientContext.ExecuteQuery();

    var fileServerRelativeUrl = list.RootFolder.ServerRelativeUrl + "/Document.docx";
    var fieldInternalName = "Last_x0020_Saved_x0020_Time";

    var itemById = list.GetItemById(1);

    var itemByFile = clientContext.Web.GetFileByServerRelativeUrl(fileServerRelativeUrl).ListItemAllFields;

    var itemByWeb = clientContext.Web.GetListItem(fileServerRelativeUrl);

    var itemsByQuery = list.GetItems(new Microsoft.SharePoint.Client.CamlQuery() {
        FolderServerRelativeUrl = list.RootFolder.ServerRelativeUrl
    });

    var itemsByQueryWithUTC = list.GetItems(new Microsoft.SharePoint.Client.CamlQuery()
    {
        DatesInUtc = true,
        FolderServerRelativeUrl = list.RootFolder.ServerRelativeUrl
    });

    clientContext.Load(itemById);
    clientContext.Load(itemByFile);
    clientContext.Load(itemByWeb);
    clientContext.Load(itemsByQuery);
    clientContext.Load(itemsByQueryWithUTC);
    clientContext.ExecuteQuery();

    OutputTime("GetItemById", itemById, fieldInternalName);
    OutputTime("File.ListItemAllFields", itemByFile, fieldInternalName);
    OutputTime("Web.GetListItem", itemByWeb, fieldInternalName);
    OutputTime("Query", itemsByQuery[0], fieldInternalName);
    OutputTime("QueryWithUTC", itemsByQueryWithUTC[0], fieldInternalName);
}
```
And the report is here, only the GetItemById and Query can get the UTC Time, the other method doesn't use the UTC Time.

```console
              GetItemById     2018-10-08T20:24:28Z            Utc
   File.ListItemAllFields     2018-10-08T13:24:28Z    Unspecified
          Web.GetListItem     2018-10-08T13:24:28Z    Unspecified
                    Query     2018-10-08T20:24:28Z            Utc
             QueryWithUTC     2018-10-08T20:24:28Z            Utc
```
Maybe you have question to ask why different API has different behaviour, you can find the answer with SharePoint On-Premise API.

And the question for you, do you know what's the timezone for the unspecified time?