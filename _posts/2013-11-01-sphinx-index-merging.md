---
layout: post
title: "Sphinx Index Merging"
description: ""
categories: ["tech"]
tags: [sphinx, search]

---
{% include JB/setup %}

## Overview

  Sphinx indexing is fast on small dataset. But if your indexes are huge in size, reindexing the whole dataset takes time and also consumes extra CPU cycle / memory. So reindexing the whole data isn't a good choice if your indexes are huge. Sphinx comes with a feature of index merging which solves this problem.


## How to Recreate Indexes

  Once `searchd` daemon is running reindexing has to be done with `--rotate` flag otherwise `searchd` have to stopped and the data to be reindexed. 

    > cd /usr/local/sphinx/bin
    > sudo ./indexer --rotate --all -c <path to sphinx.conf file>

  This reindexing is very quick if the dataset is small. But gradualy the data size increases and this reindexing takes time.  
  The other scalable approach is through `main+delta` approach. 

## Main+Delta Approach
  We have 2 sources and 2 indexes in this approach. There is a `main` index which contains the historical data. The other index
  is the `delta` index which contains the newly populated data. All newly populated data goes into `delta`. The reindexing of delta is usually done in every 5-10 min. In our production setup we are reindeing `delta` in every 5 min through cron. So the end user will have maximum delay of 5 min until the data in available in search results.

  `main` index is created only once i.e at the time of setting up sphinx indexing for a data source.
  `delta` index is merged to `main` index once the size of `delta` is significantly increased. This merging is usually done once daily. 

  So by the start of next day all the previous day's data reaches `main` index through index merging.

  Inorder to determine what all data is merged in `main` and what all data have to be indexed in `delta` we create a table and keep the pimary id of the last indexed row. This approach works if the primary key(pk) is an `AUTO INCREMENT` field.

  schema of table looks like -

    CREATE TABLE `delta_count` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `max_id` int(11) NOT NULL,
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB;

  The primary key of the indexed row is stored in `max_id` field.  

  Source and Index configuration looks like below 

    source mainsource {
      ...

      sql_query        = SELECT id, age, name FROM person WHERE id >=$start AND id<=$end

      sql_query_range  = SELECT MIN(id),(SELECT max_id FROM delta_count WHERE id=1)  as 'MAX(id)' FROM person;

      sql_range_step   = 1000
      
      sql_attr_uint    = age

      ...
    }

    source deltasource : mainsource {

      sql_query_range  =

      sql_query_pre    = REPLACE INTO delta_count  SELECT 2, MAX(id) FROM person 

      sql_query        = SELECT id, age, name FROM person WHERE id > (SELECT max_id FROM delta_count WHERE id=1) and id <=(SELECT max_id FROM delta_count WHERE id=2)

    }

    index mainindex {

      source           =  mainsource

      path             = /usr/local/sphinx/var/data/mainsource

      ...

    }

    index deltaindex : mainindex {

      source           =  deltasource

      path             = /usr/local/sphinx/var/data/deltasource

    }


  The table `delta_count` will have 2 rows. The row with id=1 will contain the primary key of the row which is recently indexed in `main` index. 

  The 'row' with id=2 contains primary key of row which is recently index in `delta` index.

  As I told earlier in the post the `main` index is created once and after that only merging to it is done.

    > sudo /usr/local/sphinx/bin/indexer -c <path to sphinx.conf file> mainindex   // run first time

  The command to index delta looks as follows -

    > sudo /usr/local/sphinx/bin/indexer --rotate -c <path to sphinx.conf file> deltaindex // run every 5 or 10 min

  Every day `delta` in merged to `main` index -

    > sudo /usr/local/sphinx/bin/indexer -c <path to sphinx.conf file> --merge mainindex deltaindex --rotate  // run once daily

  Once `delta` is merged to `main` an update is done to `delta_count` table - 

    sql > REPLACE INTO delta_count  SELECT 1, max_id FROM delta_count where id = 2

  The sql above makes indexer to index only newly populated data in `delta` index.

## In Conclusion 
  This is how index merging is done in sphinx. I hope this will be of some help who is new to sphinx and puzzled. 














