﻿# SQL Backup retention for Azure Storage (Blobs)

This app has one **very specific** purpose:

If you are storing backup files generated by Microsoft SQL Server in Azure storage containers (blob storage **only**), you can use this utility to enforce a fairly sophisticated **backup retention policy**. In other words, this utility will **DELETE** backup files from your Azure Sorage account automatically, whenever executed, according to the rules you'll define in the app settings. How to schedule its execution is up to end-users, in our case, this utility is (/was originally) deployed on an "always on" Azure Virtual Machine and executed as a scheduled task.

## General features

- As this utility is dedicated to Microsoft SQL Server backup files, it **only** considers files with ".bak" and ".trn" extensions. All other files are ignored.
- It allows/requires to define an individual retention policy for each individual database you are backing up to the Azure cloud. (See warning about filenames below!)

This utility implements 3 paramaters for backup retention:

1. **`RetainAllInDays`**. This parameter defines a period of time from the execution time backwards, for which all SQL backup files will kept. This included ".bak" and ".trn", while longer retention periods will not retain (will delete) ".trn" files.
1. **`WeeksRetention`**. This parameter defines a period of time in number of weeks, and **starting from the end** of the "retain all" period.
1. **`MonthsRetention`**. This parameter defines a period of time in number of months, and **starting from the end** of the "retain all" period.

In broad terms, the utility will thus:

1. Keep all files (".bak" and ".trn") for the "RetainAll" period.
    1. 1. Delete all ".trn" files which are older than that.
1. Keep the **newest** file that exists for each "single week" period, starting **after** the "RetainAll" period, up to the number of weeks specified.
    1. Thus, if you specifiy 5 days for "RetainAll", and then 3 for the "WeeksRetention" at any given time, the oldest "weekly file" you may be storing will be from between (7\*2) + 5 days and (7\*3) + 5 days ago.
1. Keep the **newest** file that exists for each "single month" period, starting **after** the "RetainAll" period, up to the number of weeks specified.
    1. Thus, if you specifiy 5 days for "RetainAll", and then 2 for the "MonthsRetention" at any given time, the oldest "monthly file" you may be storing will be from between one month + 5 days and two months + 5 days ago.
    1. If both **`WeeksRetention`** and **`MonthsRetention`** are bigger than zero, the two "ranges" will overlap, meaning that the same file(s) retained because of the "weekly" setting will be retained *also because* of the "monthly" setting.

## Usage

In order to use this utility, you need to obtain a Shared Access Signature (SAS) with (at least) `Read`, `List` and `Delete` permissions, it needs to be produced at the level of the whole Azure Storage Account, not at the level of individual containers.

> You can obtain a SAS via the Azure portal, the Microsoft Azure Storage Explorer, and/or Powershell, Azure CLI, etc... Our recommendation is to make a "long lasting" SAS with **only** `Read`, `List` and `Delete` permissions, and to guard its value with extreme care.

Moreover, this utility is designed to work for "Blob Containers" and has not been tested against any other kind of Azure storage container. 

All parameters are defined via the `appsettings.json` file, included in this project, with sample data. Thus, in order to use this utility, **you need to edit** the `appsettings.json`, to supply the correct parameters (including the aforementioned SAS string, preferably "url encoded"). 

To function properly, the utility needs to write to a log file, thus you will need to ensure that the account under which the app will run has write access to the folder `[app root]\LogFiles`.



### Appsettings parameters
These are divided in two parts `General` and `BackedUpDatabases`.

**General:**
- `StorageURI` [Required] - this contains the https address of your blob storage account, as in `https://[storage_account_name].blob.core.windows.net/`
- `SAS` [Required] - the URL-encoded Shared Access Signature (SAS) that provides access to said storage account.
- `VerboseLogging` **Default value is `true`.** - If true, the app will log to file (and console) every single thing it does, and then some. Otherwise it will log far less details, but never nothing.
- `DoNotUseBlobTimestamps` **Default value is `true`.** - See below, this is a risky option (if set to `false`) which allows to use the timestamp of the blob files to decide what to keep and what to delete. Might produce unexpected results, as the blob timestamps refer to the blob files themselves, but the backup files might be older (have been generated elsewhere, and uploaded to the cloud later). 
- `AsIf` **Default value is `true`.** - Enables simulation mode. If set to `true` the app will not make any changes and just report back what files will be deleted/retained. It is **highly recommended** to run the utility in this "testing mode" with VerboseLogging, so to check what it will do exactly. If you'll find that it does plan to do what you wanted, then it's OK to schedule its execution with `AsIf` set to `false`.

**BackedUpDatabases** is a JSON array of objects, each defining the retention policy for a single database. Elements for each database are:

- - `FullBackupsContainer` [Required]: the name of the blob container where your backups are stored.
- `DatabaseName`: "reviewer" [Required, case-insensitive]: the name of the database covered by this "policy".
- `IsStriped` [Boolean, defaults to `false`]: if set to `true` the app will expect that 1 backup set is comprised by multiple numbered files, where their "number" is the last number appearing in the filename, like in: `DBName_backup_2023_11_24_stripe1.bak`, `DBName_backup_2023_11_24_stripe2.bak`, [...]. 
    - App can work for up to 100 stripes, as it will recognise filenames that use both `stripe01` and `stripe1` convention. Note that the only requirment is that the numbers which change need to be _the last numbers that change_ (from within a single backup set) the filename doesn't need to contain the word "stripe" or anything. On the other hand, if the filenames contain a very detailed timestamp, to the point where `[long name with date-time]file1.bak` might have a different "date-time" to the last stripe in the set `[long name with later date-time]file25.bak`. This utility **does not** read the file headers, and considers **only** filenames for identifying stripes.
- `RetainAllInDays`, `WeeksRetention` and `MonthsRetention` [All required, all integers equal or greater than zero], please see above, these numbers define the actual **retention policy** for the database indicated.


## More details
- For a given "database setting", only files with a filename that begin with the database name (case-insensitive), followed by an underscore (`_`) will be processed. This means that **it's safe to store backup files from multiple databases in the same container**, even if the retention policy for each database is different, **but only if (Warning!)** your database names don't start in similar ways. For example, if you are backing up a databases called "Sales" and another one called "Sales_Archive", keeping the backups in the same container will not work. Files belonging to "Sales_Archive" will be treated twice, once for their own "policy", but also once for the policy defined for "Sales", leading to **very undesirable** results!
- The SAS needs to be produced at the level of the whole Azure Storage Account and **will NOT work** if it is a SAS providing access to a single container. If you rotate the access keys, or if the SAS expires (All SASs have an expiry date), this utility will stop working.
- App will *always* look for a date in "_yyyy_MM_dd_" (like `_2023_11_24_`) format first, if that attempt fails, it will consider the `DoNotUseBlobTimestamps` flag:
    - if `true` files for which a date wasn't found in the filename will be always kept (in the keep-all set, will appear with the spoof date of "now + 1 day")
    - otherwise, if the `DoNotUseBlobTimestamps` flag is set to false, then the `CreatedOn` Date will be used for the Blob File (which **may or may not be the creation date of the backup file itself(!!)**, depending on how the file was produced and uploaded to blob storage).
- No other date-formats in the filename are supported!
- "\*.trn" files are only kept for the "retain all" period - if such files are found and their date is older than the "keep all" period, they will be deleted.
- `IsStriped` only applies to \*.bak files! If you are saving transaction logs in stripes, then this utility cannot work for you (not without changing the source code).
- When multiple bak files are present in a weekly/monthly range, the newest is selected.