---
layout      : single
title       : How to query exchange online archive mailbox size?
summary     : Query exchange online archive mailbox size with multiple methods
categories  : ExchangeOnline
tags        : [Exchange Online, EWS, Archive mailbox, PowerShell]
date        : 2023-06-10 22:04:00
commentId   : 3
permalink   : /EWS-ArchiveMailbox-Size
toc         : true
classes     : wide
toc_icon    : "cog"
toc_label   : "My Table of Contents"
---

# Summary
There are multiple methods you can use to get regular mailbox size, for example, Exchange Online PowerShell, EWS API, Graph API, this page will show how to use these APIs to query archive mailbox size.

# Exchange Online PowerShell

Exchange Online PowerShell provides power cmelet for administrator to query the mailbox information, for example, we can use [Get-EXOMailboxFolderStatistics](https://learn.microsoft.com/en-us/powershell/module/exchange/get-exomailboxfolderstatistics?view=exchange-ps) to retrieve information about the folders in a specified mailbox, inlucde archive mailbox.


```PowerShell
PS C:\Users\xxx> get-exoMailboxFolderStatistics -Identity $mailbox -Archive | select Identity, FolderSize

Identity                                                    FolderSize
--------                                                    ----------
xxx@xxx\Top of Information Store                            1.868 GB (2,005,434,759 bytes)
xxx@xxx\Archive                                             67.66 GB (72,645,022,616 bytes)
xxx@xxx\Archive\000                                         9.818 GB (10,541,827,919 bytes)
xxx@xxx\Archive\111                                         6.355 GB (6,823,430,806 bytes)
xxx@xxx\Archive\Archive_2022 (Created on Jun 06, 2022 2_39) 46.88 GB (50,340,464,045 bytes)
...
```
And Mirosoft provides full example how to use Exchange Online PowerShell with application permission with [C# code](https://learn.microsoft.com/en-us/powershell/exchange/connect-to-exo-powershell-c-sharp?view=exchange-ps).


# EWS API

EWS API provides the similar API to retrieve the folder size and then you can calculate the total size for the mailbox, the key extended priperty is 0xe08, and we can get the size for each folder in the mailbox, however this API doesn't help the archive mailbox folder, from the below result, we cannot get size for some folder, the reason is that Archive mailbox supports auto-expanding feature, I guess the folder is a shortcut in the main archive mailbox, and there is no size information in that folder.

```bash
Name                                                   Folder Size    
Archive                                                72645022616    
Archive/000                                            0              
Archive/111                                            0              
Archive/Archive_2022 (Created on Jun 06, 2022 2_39)    0              
Deleted Items                                          95207499       
ExternalContacts                                       0              
Files                                                  0              

```

Below is the code which is used to get the folder size.

```csharp
    public static Dictionary<string, FolderInfo> LoadAllFolders(this ExchangeService exchangeService, WellKnownFolderName folderName)
    {
        var PidTagMessageSizeExtended = new ExtendedPropertyDefinition(0xe08, MapiPropertyType.Long);

        var folderView = new FolderView(100);
        folderView.PropertySet = new PropertySet(BasePropertySet.FirstClassProperties, PidTagMessageSizeExtended);
        folderView.Traversal = FolderTraversal.Deep;


        FindFoldersResults folders = null;

        var items = new Dictionary<string, FolderInfo>();
        var folderNameMapping = new Dictionary<string, string>();

        while (folders == null || folders.MoreAvailable)
        {
            //output folders
            folderView.Offset = folders == null ? 0 : folders.NextPageOffset.Value;
            folders = exchangeService.FindFolders(folderName, folderView);

            foreach (var folder in folders)
            {
                string parentName = null;
                string fullName = folder.DisplayName;
                if (folderNameMapping.TryGetValue(folder.ParentFolderId.ToString(), out parentName))
                {
                    fullName = parentName + "/" + folder.DisplayName;
                }

                long folderSize;

                folder.TryGetProperty(PidTagMessageSizeExtended, out folderSize);

                items.Add(fullName, new FolderInfo() { FullName = fullName, FolderSize = folderSize, ItemsCount = folder.TotalCount });
                folderNameMapping.Add(folder.Id.ToString(), fullName);
            }
        }

        return items;
    }
```

# MAPI

You can use this https://github.com/microsoft/mfcmapi to access the folder information. However you cannot use MAPI to access the mailbox directly per https://learn.microsoft.com/en-us/outlook/troubleshoot/authentication/expose-permissions-issue-with-mapi-oauth-tokens. 

![MFCMAPI Folder Size](/assets/img/EWS_Size_2023-09-03_16-33-29.png)

# Graph API

From https://learn.microsoft.com/en-us/graph/api/resources/mailfolder?view=graph-rest-1.0, the Graph API for the archive mailbox is not available yet, maybe it will be supported in future.