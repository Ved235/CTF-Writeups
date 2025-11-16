# The Fall of the Great Wall (PwnSec)

> Can you pass the Great Wall's defenses and retrieve the hidden artifact? [https://greatwall.blob.core.windows.net/](https://greatwall.blob.core.windows.net/) by @CodeBreaker\
> \
> The challenge was flagged as medium and we 2nd blooded the challenge/

## 1. Fuzz and find the container name

As it is related to azure blob storage the conventional approach is to use a fuzzing tool to search for common container names.  I used this tool called [goblob](https://github.com/Macmod/goblob) and it comes preinstalled with some useful wordlists. The tool gave me this result:

{% code overflow="wrap" fullWidth="false" %}
```bash
[~] Analyzing container 'storage' in account 'greatwall' (page 1) [+][C=200] https://greatwall.blob.core.windows.net/storage?restype=container [~][1/1] Finished searching account 'greatwall'
```
{% endcode %}

## 2. Exploring the container

After we found the storage container let's explore its contents. I prefer to use the Azure Storage Explorer for this because it has an easy GUI and you don't have to worry about request headers. I find a file called `connection_info`  and I open its version history:&#x20;

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

The first file had a larger file size, so I just downloaded that file 😂...

## 3. Examining the zip file

The zip file is password protected... so using **John** we found that the password is `Wall$treet1` . Unlocking it, we get these contents:

{% code overflow="wrap" %}
```
#This Document should be stored in a secret place 
*************************************************
psql connection info:
username: gw_watcher
password: MkhqalhuVUd5cDVhQkdvQ2xCN25OaDY3SDlnOA==
target: dragongate.postgres.database.azure.com
database: dragonlair
*************************************************
```
{% endcode %}

## 4. Connecting to psql database

Using the credentials and psql client we can connect to the provided database. I used this line:

{% code overflow="wrap" %}
```bash
psql -h dragongate.postgres.database.azure.com -p 5432 -U gw_watcher -d dragonlair
```
{% endcode %}

In this exploring the database we find the data that we were looking for:

{% code overflow="wrap" %}
```markdown
 id |     name      |      origin       | power_level |   guardian   |                                                        encoded_secret                                                        
----+---------------+-------------------+-------------+--------------+------------------------------------------------------------------------------------------------------------------------------
  1 | Crimson Scale | Northern Fortress |        8800 | General Wei  | Q3JpbXNvbl9TY2FsZV9IZWFydA==
  2 | Golden Core   | Central Bastion   |        9600 | Lady Zhen    | R29sZGVuX0NvcmVfUG93ZXI=
  3 | Verdant Gem   | Eastern Tower     |        8700 | Captain Lin  | VmVyZGFudF9HZW1fQmxvb20=
  4 | Onyx Heart    | Southern Gate     |        9400 | Lord Chen    | T255eF9IZWFydF9TaGFkb3c=
  5 | Azure Flame   | Western Keep      |        9200 | Master Liang | ZmxhZ3s3aDNyM18xbl83aDNfbTE1N18zbjBybTB1NV9tNGozNTcxY181MWwzbjdfNG5kXzczcnIxYmwzXzU3MDBkXzdoM182cjM0N193NGxsXzBmX2NoMW40fQ==
```
{% endcode %}

```
flag{7h3r3_1n_7h3_m157_3n0rm0u5_m4j3571c_51l3n7_4nd_73rr1bl3_5700d_7h3_6r347_w4ll_0f_ch1n4}
```
