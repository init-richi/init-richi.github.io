---
layout: post
title:  "Migrating Nextcloud local storage to s3 compatible local storage"
date:   2022-11-04 09:28:29 +0100
categories: nextcloud s3 storage
---

Today i finished migrating my Nextcloud instances primary storage from local to an s3 compatible object storage.  

## But why, tho?
You may ask yourself, why should someone do something like that? To answer this question I need to tell you about my current server setup:  
I use my hardware-server for virtualizing. For each application i use a different virtual machine to publish my services. E.g. my web-vm runs apache2 and web related software exclusivly and a database-vm runs all my Database software.  
This just works and I could be happy with this setup, but I'm not. I don't like, that a possible vulnerble application, runs next to my personal Data, stored on my Nextcloud instance, even if I have set user permissions, seperate document roots to different user, etc. Also I want my applications to be as portable as possible. If the datacenter, where my server is currently located, catches fire, or my server just throws a tantrum and then stops working, I want to setup a new instance, wherever I want, to have all my services working again as soon as possible.  

I just want to spin up a, you might guessed it already, container, attach it to my storage and restore my database backup. My current restore plan needs a server, I need to rewrite my configurations (which I don't backup currently), setup letsencrypt, install Nextcloud inside a vHost, restore my Database and my Data.  
Later I also want to play around with k8s, just to learn some new things (I know... using a sledge-hammer to crack a nut...).

## Preparation

I needed a S3 compatible object storage. I saw an offer by [contabo](https://contabo.com/de/object-storage/), so I thought I'll give it a try, but this should work with any S3 compatible object storage.  
If you have a nextcloud-instance, which is used by multiple people, you may consider activating the maintenance mode, to stop them from interfering your migration process. As I'm the only user on my instance, I just thought: Meh, it should work anyway.  
I used the solution described in this [GitHub Issue](https://github.com/nextcloud/server/issues/25781#issuecomment-1162604945).  
As I'm the only user on my instance and have several nextcloud clients, which are holding the current data, I did not make a backup... (I know... bad practice).  
I'm using the `s3cmd` commandline tool, to sync my files to my bucket, but it should work with any other s3 client.
Also I created a new Bucket exclusively for my nextcloud data.

## Getting things done

Let's assume that my nextcloud data was stored in `/srv/nextcloud`, as descibed in the linked *GitHub Issue*. I'm working as my www-data user.  
As suggested, I'm working with sym-links inside my working directory.

First I needed to get a shell with my www-data user and setup a migration folder:
```shell
$ sudo -u www-data  /bin/bash
$ mkdir -p ~/migration/data && cd ~/migration
```

### Create user file list
Then we need to map the user files, described in my DB with it's paths inside my data folder.  
You should change the DB-name and the data path according to your setup, in this case the data path is `/srv/nextcloud` and the DB name is `nextcloud`.  
Also you might have a .my.cnf containing the credentials and mysql hosts, then you also can omit the options `-u nextcloud_user -h 192.168.222.1 -p`.
```shell
$ mysql -B --disable-column-names -u nextcloud_user -h 192.168.222.1 -p -D nextcloud << EOF > user_file_list   
     select concat('urn:oid:', fileid, ' ', '/srv/nextcloud/',
          substring(id from 7), '/', path)     
     from oc_filecache     
     join oc_storages      
     on storage = numeric_id   
     where id like 'home::%'   
     order by id;
EOF
```
This creates a file named `user_file_list` inside your `~/migration/` folder.  

### Create meta file list
You also need to create a similar list for meta files, with the following command:

```shell
$ mysql -B --disable-column-names -u nextcloud_user -h 192.168.222.1 -p -D nextcloud << EOF > meta_file_list
     select concat('urn:oid:', fileid, ' ', substring(id from 8), path)
     from oc_filecache
     join oc_storages
     on storage = numeric_id
     where id like 'local::%'
     order by id;
EOF
```

### Create symlinks
I already created the folder *data* inside `~/migration`, so let's cd into it.
```shell
$ cd data
$ pwd
/path/to/home/migration/data
```
Now I create the symlinks for the actual data. To do this, I iterate over the file lists:
```shell
$ while read target source ; do
    if [ -f "$source" ] ; then
        ln -s "$source" "$target"
    fi
done < ../user_file_list

$ while read target source ; do
    if [ -f "$source" ] ; then
        ln -s "$source" "$target"
    fi
done < ../meta_file_list
```

### Sync data to S3
I already configured my `s3cmd` client, you may need to do this now.  
To sync my files I used the following command:
```shell
$ nohup s3cmd sync . s3://nc-data -F > ../nohup.out
```
I used nohup, to ensure my sync keeps running, after sending it to background and logging off my vm.  
It took around 2 Days to sync my ~26GB to contabo object storage, so be patient.

### Update your DB and config

Please assure, that you have done everything correctly, before continuing! This could break your nextcloud and you have to restore everything from your backups (pro: you can setup a new instance, configure S3 primary storage from beginning and upload your files from your nextcloud client).  

I can now finally adjust my DB using `mysql -u nextcloud_user -h 192.168.222.1 -p -D nextcloud`:
```sql
UPDATE oc_storages
  SET id = CONCAT('object::user:', SUBSTRING(id FROM 7)) 
  WHERE id LIKE 'home::%';
UPDATE oc_storages 
  SET id = 'object::store:amazon::YOUR_BUCKET_NAME'
  WHERE id LIKE 'local::%';
```

And my config:
```php
...
'objectstore' => [
        'class' => '\\OC\\Files\\ObjectStore\\S3',
        'arguments' => [
                'bucket' => 'YOUR_BUCKET_NAME',
                'autocreate' => true,
                'key'    => 'YOUR_ACCESS_KEY',
                'secret' => 'YOUR_ACCESS_SECRET',
                'hostname' => 'HOSTNAME_OBJECT_STORAGE_PROVIDER',
		'port' => 443,
                'use_ssl' => true,
		'region' => 'optional',
                // required for some non Amazon S3 implementations
                'use_path_style'=>true
        ],
],
...
```

## Finalize

Everything should now be setup, as needed. You can now deactivate the maintenance mode, if previously activated, and watch your nextcloud logs, if any error comes up.

## Conclusion

Well this was a journey, but everything works as expected. Now I can continue to migrate my infrastructure to container. I ran over some problems, which led me to manually deleting/updating some rows inside *oc_storages* table, but if you followed everything correctly, you shouldn't need to.  

I'm not responsible, if you break your nextcloud instance, loose data or opening vulnerbilitys inside your infrastructure. Read everything first, then act! And only follow this guide, when you know what you are doing.
