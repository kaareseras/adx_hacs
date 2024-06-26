# This integration is from Home Assistant v. 2024.6 now an official part of the Home Assistnat repo. please stop using this HACS. This repo is now being used for development and may be unstable


# Home Azure Data Explorer Beta testing

This is a Home Assistant Custom integration for beta testing the Azure Data Explorer integration.
I will use this when making some bigger changes to the Azure Data Exploerer integration in Home Assisant for beta testing before the code is commited to the main Home Assistant.
This makes it easier for you to do testing, and it also makes it easier and faster to do changes without having to wait for a PR in home assistant.

## Installing

Intall using [hacs](https://hacs.xyz/).
Add a custom repository to hacs (using the github url of this repository).

Once the custom repository is added, restart Home Assistant.

Then, in the Settings/Devices & Services/Integrations, click the '+Add Integration' button (lower right) and search for Azure Data Explorer.  (note you may need to press CTRL-F5 to refresh your browser before the integration shows up).

You will be prompted with a configuration pane.  In order to fill out the values, you will need to follow the steps below to create an Azure Data Explorer Cluster, create a Service Principal (AppId) via the Azure portal, and create some tables and set the appropriate permissions and ingestion policies.

Alternatively you can copy the custom_components/azure_data_explorer folder manually to your Home Assistant configuration folder.


[Azure Data Explorer](https://azure.microsoft.com/en-us/services/data-explorer/)  is a high performace timeseries database, query engine and dashboarding tool.

The Home Assistant `Azure Data Explorer` integration allows you to hook into the Home Assistant event bus and forward events to Azure Data Explorer for analytics and dashboarding.

## Create a free Azure account
* Create a  [free Azure account](https://azure.microsoft.com/). you will be asked for creditcard info, but all rescources created here are free.

## Create a Service Principal (App registration)
For Home Assistant to authenticate with Azure Data Explorer, it need a **Service Principal**
1. Create a [Service Principal](https://docs.microsoft.com/en-us/azure/data-explorer/provision-azure-ad-app) follow guide step 1-7.  Note that you do not need to create an Azure subscription to do this, you just need a Microsoft account (e.g. hotmail.com, live.com etc.).
2. Copy values for later use:
    * Application (client) ID  <--From App registration owerwiev
    * Directory (tenant) ID    <--From App registration owerwiev
    * Secret value             <--From when the secret was created in 1.7 (note that this is the 'Value', not the 'Secret Id')

## Create Free Azure Dataexplorer cluster and Database
There are two ways of creating an Azure Data Explorer Cluster: **Pay as you go (PAYG)** or **Free**
to create a paid cluster follow instructions from here: [Microsoft quickstart](https://docs.microsoft.com/en-us/azure/data-explorer/create-cluster-database-portal)
However Microsoft has released a free offer and this guide describes how to set up a free Azure Data Explorer Cluster and database:

There are a few different between the **PAYG** and **Free** versions:
| Feature         | PAYG Cluster           | Free Cluster                    |
| --------------- | ---------------------- | ------------------------------- |
| Ingestion       | Streaming and Queueing | Queueing only (for now)         |
| Cluster size    | Fully scalable         | 4 vCPU, 8GB Menory, ~100GB data |

1. Navigate to [aka.ms/kustofree](https://aka.ms/kustofree).
2. Navigate to **My Cluster** .
3. And click the **Create Cluster** button.
4. Name the Cluster and database.
5. Copy the **database name** for later use
5. Check terms and condition (after reading them) and click **Create Cluster**.

Within a minute, you will have a Azure Data Explorer cluster ready

After the creation of the database, copy the **Data ingestion URI** from the top of the page

## Create Azure Data Table
1. Navigate to [aka.ms/kustofree](https://aka.ms/kustofree).
2. Navigate to **Query**.
3. Write and execute the foloing statements one by one, replacing the content between the <> with the copied values (including the brackets)

```KQL
// Give the Service Pricipal write access to the database
.add database ['<databasename>'] ingestors ('aadapp=<ApplicationID>;<DirectoryID>');

// Give the Service Pricipal read access to database (used for connectivity checks) 
.add database ['<databasename>'] viewers ('aadapp=<ApplicationID>;<DirectoryID>');

// Create a table for the data to be ingested into (Replace name and copy inserted *name* for later use)
.create table ['<name_to_be_replaced>'] (entity_id: string, state: string, attributes: dynamic, last_changed: datetime, last_updated: datetime, context: dynamic)

// Creat a mapping from the incomming JSON to the table and collums just created (replace name with table name from previous step)
.create table ['<name_to_be_replaced>'] ingestion json mapping 'ha_json_mapping' '[{"column":"entity_id","path":"$.entity_id"},{"column":"state","path":"$.state"},{"column":"attributes","path":"$.attributes"},{"column":"last_changed","path":"$.last_changed"},{"column":"last_updated","path":"$.last_updated"},{"column":"context","path":"$.context"}]'
```

This is an example with a free cluster for reference

```KQL
.add database ['HomeAssistant'] ingestors ('aadapp=b5253d02-c8f4-1234-a0f0-818491ba2a1f;72f123bf-86f1-41af-91ab-2d7cd011db93');

.add database ['HomeAssistant'] viewers ('aadapp=b5253d02-c8f4-1234-a0f0-818491ba2a1f;72f123bf-86f1-41af-91ab-2d7cd011db93');

.create table ['raw'] (entity_id: string, state: string, attributes: dynamic, last_changed: datetime, last_updated: datetime, context: dynamic)

.create table ['raw'] ingestion json mapping 'ha_json_mapping' '[{"column":"entity_id","path":"$.entity_id"},{"column":"state","path":"$.state"},{"column":"attributes","path":"$.attributes"},{"column":"last_changed","path":"$.last_changed"},{"column":"last_updated","path":"$.last_updated"},{"column":"context","path":"$.context"}]'
```

## Configuration
>if using a free cluste, check the **Use Queueing client** in the form

{% include integrations/config_flow.md %}

After completiing the flow, Home Assistant is sending data to Azure Data Explorer. 

> Home Assistant is buffering for default 5 seconds before sending, and Batching Policy in Azure Data Explorer will futher batch up for default 

## Filters

Optinaly add the following lines to your `configuration.yaml` file for filtering data ingested into Azure Data Explorer:

```yaml
# Example configuration.yaml entry
azure_data_explorer:
  filter:
    include_domains:
    - homeassistant
    - light
    - media_player
```

{% configuration %}
filter:
  description: Filter domains and entities for Event Hub. ([Configure Filter](#configure-filter))
  required: true
  type: map
  default: Includes all entities from all domains
  keys:
    include_domains:
      description: List of domains to include (e.g., `light`).
      required: false
      type: list
    exclude_domains:
      description: List of domains to exclude (e.g., `light`).
      required: false
      type: list
    include_entity_globs:
      description: Include all entities matching a listed pattern (e.g., `sensor.weather_*`).
      required: false
      type: list
    exclude_entity_globs:
      description: Exclude all entities matching a listed pattern (e.g., `sensor.weather_*`).
      required: false
      type: list
    include_entities:
      description: List of entities to include (e.g., `light.attic`).
      required: false
      type: list
    exclude_entities:
      description: List of entities to include (e.g., `light.attic`).
      required: false
      type: list
{% endconfiguration %}

<div class='note warning'>
Not filtering domains or entities will send every event to Azure Data Explorer.
</div>

<div class='note warning'>
Event Hubs have a retention time of at most 7 days, if you do not capture or use the events they are deleted automatically from the Event Hub, the default retention is 1 day.
</div>

### Configure Filter

By default, no entity will be excluded. To limit which entities are being exposed to `Azure Data Explorer`, you can use the `filter` parameter.

```yaml
# Example filter to include specified domains and exclude specified entities
azure_event_hub:
  filter:
    include_domains:
      - alarm_control_panel
      - light
    include_entity_globs:
      - binary_sensor.*_occupancy
    exclude_entities:
      - light.kitchen_light
```

Filters are applied as follows:

1. No includes or excludes - pass all entities
2. Includes, no excludes - only include specified entities
3. Excludes, no includes - only exclude specified entities
4. Both includes and excludes:
   - Include domain and/or glob patterns specified
      - If domain is included, and entity not excluded or match exclude glob pattern, pass
      - If entity matches include glob pattern, and entity does not match any exclude criteria (domain, glob pattern or listed), pass
      - If domain is not included, glob pattern does not match, and entity not included, fail
   - Exclude domain and/or glob patterns specified and include does not list domains or glob patterns
      - If domain is excluded and entity not included, fail
      - If entity matches exclude glob pattern and entity not included, fail
      - If entity does not match any exclude criteria (domain, glob pattern or listed), pass
   - Neither include or exclude specifies domains or glob patterns
      - If entity is included, pass (as #2 above)
      - If entity include and exclude, the entity exclude is ignored


## Using Azure Data Explorer
As the setup is complete, data is being sent to Azure Data Explorer, and you can start exploring your data.
Here are som rescources to learn to use Azure Data Explorer

* MS Learn: [https://aka.ms/learn.kql](https://aka.ms/learn.kql), [https://aka.ms/learn.adx](https://aka.ms/learn.adx)
* You tube: [Official Microsoft Azure Data Explorer YouTube channal](https://www.youtube.com/channel/UCPgPN-0DLaImaaDR_TtKR8A)
* Blog: [Official Microsoft Data Explorer blog](https://techcommunity.microsoft.com/t5/azure-data-explorer-blog/bg-p/AzureDataExplorer)

## Reporting bugs

If you find any bugs/problems use the issues here in github to report your findings.

## Current state

Currently the beta testing has these changes:

* Initial test



