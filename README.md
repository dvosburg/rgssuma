# RGS SUMA Disconnected
Rancher Government Services needs a way to replicate channel content to multiple customer SUMA servers.  This document describes how this can be done using a primary RGS SUMA with channel data getting updated.  This server then exports that data to be imported by customer SUSE Manager server(s). 

The method is based on SUMA ISSv2, with documentation here:

[Inter-Server Synchronization - Version 2](https://documentation.suse.com/suma/4.3/en/suse-manager/administration/iss_v2.html)


## Deploy RGS SUSE Manager
Start with the latest SUSE Manager [SUSE Manager 4.3](https://www.suse.com/download/suse-manager/). 

- Use SUSE Manager 4.3, updated with the latest packages
- Install the `inter-server-sync` package on the server:
```
	zypper in inter-server-sync
```
- Provide SUSE Customer Center credentials
  - Log in to [SUSE Customer Center](https://scc.suse.com)
    - Go to `My Organization`, select your organization
    - Go to `Users` `->` `Organization Credentials` and copy your Organization Username and Password
  - In the RGS instance of SUSE Manager
    - Go to `Admin` `->` `Setup Wizard` `->` `Organization Credentials`
    - Click on `Add new credential` An use the Username and Paswword provided in SCC and obtained in previous step
- Sync the required channels in SUSE Manager
  - Go to `Admin` `->` `Setup Wizard` `->` `Products`
  - Select the Products that customers require
  - Click on the top right green button `'+ Add products'`
  - You can check progress by viewing the `Products` page, or by viewing the list of `Software` in the SUSE Manager webUI

Optionally you can sync the RGS SUMA to RMT.  This requires additional administrative effort to continue to receive updates.  The important thing for this project is to have all required channels synchronized before performing exports.

### Adding the Organization for Sync process
ISSv2 uses SUMA credentials for exporting and importing channels.  It is essential that those credentials match between the SUMA servers. For this project we will create a special Organization on both the RGS (export) SUMA and the customer (import) SUMA.  In this documentation, we assume there previously is only the primary organization on the RGS SUMA server  
 - Create a new Organization on the RGS SUMA Server 
 	- `Admin` `->` `Organizations` and select `+ Create Organization`
 	- Name it '**rgsimportexport**'
 	- The default user for this org should be something unique  on the SUMA servers, so you can use the same '**rgsimportexport**'
 	- Password will be used in the import process, and for this example we will use `super-secret-password`

This newly created user has the `Organization Administrator` role assigned to it automatically for this new organization.   Note that it does not have nor need other roles.


## Deploy Customer SUSE Manager
First requirement is to have [SUSE Manager 4.3](https://www.suse.com/download/suse-manager/) running. 

- Requirement: Use SUSE Manager 4.3, updated with the latest packages
- Install the `inter-server-sync` package on the server:
	+ `zypper in inter-server-sync`
- No credentials need to be added to this server - content will come from ISSv2

### Adding Organization for Sync process
ISSv2 uses SUMA credentials for exporting and importing channels.  It is essential that those credentials match between the SUMA servers. The primary organization can be independent of this synchronization for any customized channels, users, etc.  
 - Create a new Organization on the Customer SUMA Server 
 	- `Admin` `->` `Organizations` and select `+ Create Organization`
 	- Name it `rgsimportexport`
 	- The default user for this org should be something unique  on the SUMA server, so you can use the same `rgsimportexport`
 	- Password will be used in the import process, and for this example we will use `super-secret-password`

This newly created user has the `Organization Administrator` role assigned to it automatically for this new organization.   Note that it does not have nor need other roles.


## Preparing for Export/Import
- External devices or network mount points can be used to accomplish the ISSv2 process.  In this example we are mounting an NFS exported directory to the RGS server at:

 `/media/export`
 
- It is critical that this have adequate space for the files and database that this process uses.  If you are planning to export many channels it will be quite large and take significant time.
- The directory for export needs to be empty at the start of the process.

## Export channels from the RGS SUMA Server

- View all the options for export with

 `inter-server-sync export -h`
- Select the `Parent` channels you wish to export.  Only select the needed channels - as it takes significant time for both export and import.  You can get a hierarchical list of these channels from the SUMA CLI with:

`spacewalk-remove-channel -l`

- For initial export, there will be no date filter, it will get all channel information
- Here is an example export command with two different `Parent` channels selected, and limiting it to the new Organization ID unit we created
```
    inter-server-sync export \
    --channel-with-children=sle-micro-5.4-pool-x86_64 \
    --channel-with-children=sle-micro-5.3-pool-x86_64 \
    --outputDir='/media/export' \
    --orgLimit=2
```

Wait until this process is fully completed before trying any import.


## Initial import of channels in the Customer SUMA Server

- Open a root terminal on the `Customer` SUSE Manager server
- Mount the exported directory for use in importing.  Here we will use 

`/media/import`
- Ensure that you allow sufficient time to complete the import process.  It may take hours, depending on how much you are importing.  You may wish to execute this in the background with the `screen` utility.
- Here is an example import that correlated with the export issued above:

``` 
inter-server-sync import \
--importDir=/media/import  \
--xmlRpcUser=rgsimportexport \
--xmlRpcPassword='super-secret-password' \
--logLevel=debug
```

- Once completed, all the SUSE channels you imported will be available for use by any Organization on the customer SUMA Server.
- Run this command to create any required bootstrap repositories:
```
mgr-create-bootstrap-repo -a
```
- Create needed `Activation Keys` and `Bootstrap Scripts` on the customer SUMA server as required.  The ISSv2 process does not import these.

## Subsequent updates of channels on the customer SUMA server

Following the initial channel sync, you will likely need to make periodic updates with a very similar export/import process.  The only real change will be on the command used to export channels on the RGS server.  

- In our example, we will export the same channels as above, only restrict it to a "from" date as below.  Our new directory is `/media/export/updates`.

On the RGS SUMA server:

```
inter-server-sync export  \
--channel-with-children=sle-micro-5.4-pool-x86_64 \
--channel-with-children=sle-micro-5.3-pool-x86_64 \
--outputDir='/media/export/updates' \
--orgLimit=2 \
--packagesOnlyAfter=2023-11-01
```

- The correlating import command can be exactly the same as was used previously.
On the customer SUMA server, mount this `updates` folder on `/media/import`, and run:

```
inter-server-sync import \
--importDir=/media/import  \
--xmlRpcUser=rgsimportexport \
--xmlRpcPassword='super-secret-password' \
--logLevel=debug
```
- Update the bootstrap repositories with the same command issued above:
```
mgr-create-bootstrap-repo -a
```


## Considerations for Automating the processes

- Ensure you accommodate space and mount points for the exports.  Consider scripting creation of new directories each time
- Write an export cron job that calculates the 'from' date on the RGS SUMA server as needed 
- Allow for completion time for any exports on the RGS SUMA before scripting imports on the customer SUMA.
- Make note of any new channels that may need to be added as new products or updated versions are released, and add them to the export
- Seek to limit exports to only required products - but consolidate exports if customers have identical channels or update needs
