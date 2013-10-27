---
layout: post
title: "Configuring Sphinx Search"
description: ""
categories: ["technology"]
tags: [sphinx, search]

---

{% include JB/setup %}

## Overview
  Sphinx is full-text search engine build in C++. All modern unix system which have C++ compiler should be able to compile and run Sphinx.

## Installing Sphinx on Ubuntu

### Prerequisites
  
  Install the required dependencies

    > sudo apt-get install libmysql++-dev libmysqlclient15-dev checkinstall

### Download and extract sphinx code

    > wget http://sphinxsearch.com/files/sphinx-2.1.2-release.tar.gz    
    > tar -xvzf sphinx-2.1.2-release.tar.gz

### Configure and Install

    > cd sphinx-2.1.2-release
    > ./configure --prefix=/usr/local/sphinx


  _Note: The prefix switch determines the location of sphinx installation._  


    > make -j4
    > sudo make install

## Installation Directory
  Lets see the sphinx installation directory

    > cd /usr/local/sphinx
    > ls

  You could see the following directory structure

      .
      |-- bin
          |-- indexer
          |-- indextool
          |-- search
          |-- searchd 
          |-- spelldump
      |-- etc
          |-- example.sql
          |-- sphinx.conf.dist
          |-- sphinx-min.conf.dist
      |-- share
          |-- (man pages)
      |-- var
          |-- data
                |-- (all index files)
          |-- log
                |-- query.log
                |-- searchd.log

  `bin` folder contains all the executalables. Out of the listed 5 executables `indexer` and `searchd` are very often used.  
  `indexer` tool is used to index the dataset. One the dataset is indexed it could be queried upon.  
  `searchd` is the daemon which is desired to be running all the time in order to query the indexes.  

  All the index files goes in to `/usr/local/sphinx/var/data` folder.

## Sphinx Configuration 
  A sample sphinx conf file(`/usr/local/sphinx/etc/sphinx.conf.dist`) is available with the installation. This file can be used for indexing and running searchd daemon. This file have lots of commented settings and is very annoying to understand it sometimes. So for the sake of simplicity I am listing a basic configuration file. The following code should be copied to a file named `sphinx.conf`. Please take look on sphinx documents for configuring with advanced settings.

      source src1
      {
        type        = mysql
        sql_host    = localhost
        sql_user    = root
        sql_pass    = password
        sql_db      = dbmain
        sql_port    = 3306  
        sql_query   = SELECT id, age, name FROM person
        sql_attr_uint           = age
        sql_ranged_throttle = 0
        sql_query_info    = SELECT * FROM person WHERE id=$id
      }

      index idx1
      {
        source      = src1
        path      = /usr/local/sphinx/var/data/idx1
        docinfo     = extern
        mlock     = 0
        morphology    = none
        min_word_len    = 1
        charset_type    = sbcs
        ignore_chars    = U+0027, U+27
        min_infix_len   = 4
        html_strip    = 0
      }


      indexer
      {
        mem_limit   = 32M
      }

      searchd
      {
        listen      = 9312
        listen      = 9306:mysql41
        log         = /usr/local/sphinx/var/log/searchd.log
        query_log   = /usr/local/sphinx/var/log/query.log
        read_timeout    = 5
        client_timeout    = 1800
        max_children    = 30
        pid_file    = /usr/local/sphinx/var/log/searchd.pid
        max_matches   = 5000
        seamless_rotate   = 1
        preopen_indexes   = 1
        unlink_old    = 1
        mva_updates_pool  = 1M
        max_packet_size   = 8M
        max_filters   = 256
        max_filter_values = 4096
        max_batch_queries = 32
        workers     = threads 
      }

  The sql settings in source block should be properly configured.   
  The `sql_query` and `sql_query_info` should also be filled with proper queries. 

## Index Creation and Querying the index
  We are ready to index the dataset once spinx configuration file is ready.  

      > cd /usr/local/sphinx/bin
      > sudo ./indexer --all -c <path to sphinx.conf file>

  following log will appear on the screen if the indexing is properly done.

      Sphinx 2.0.6-release (r3473)
      Copyright (c) 2001-2012, Andrew Aksyonoff
      Copyright (c) 2008-2012, Sphinx Technologies Inc (http://sphinxsearch.com)

      using config file 'sphinx.conf'...
      indexing index 'idx1'...
      collected 1202 docs, 0.0 MB
      sorted 0.0 Mhits, 100.0% done
      total 1202 docs, 207 bytes
      total 0.047 sec, 4351 bytes/sec, 25266.43 docs/sec
      total 3 reads, 0.000 sec, 5.2 kb/call avg, 0.0 msec/call avg
      total 9 writes, 0.000 sec, 3.4 kb/call avg, 0.0 msec/call avg

  The indexes will be created in the location `/usr/local/sphinx/var/data`.


  Once the indexes are created our task is almost done. All we need is to run the `searchd` daemon and query the index.

      > cd /usr/local/sphinx/bin
      > sudo ./searchd -c <path to sphinx.conf file>
  Folowing logs will appear on screen on successful execution of searchd

      Sphinx 2.0.6-release (r3473)
      Copyright (c) 2001-2012, Andrew Aksyonoff
      Copyright (c) 2008-2012, Sphinx Technologies Inc (http://sphinxsearch.com)

      using config file 'sphinx.conf'...
      WARNING: compat_sphinxql_magics=1 is deprecated; please update your application and config
      listening on all interfaces, port=9312
      listening on all interfaces, port=9306
      precaching index 'idx1'
      precached 1 indexes in 0.001 sec 


  Sphinx searchd daemon supports MySQL binary network protocol and can be accessed with regular MySQL API.

      > mysql -P 9306 --protocol=tcp
      
  Now query the index through the MySQL console.

      > select * from `idx1`;
      > select * from `idx1` WHERE MATCH ('sameer');


  This is it :) 

## In Conclusion 
  I hope this helps you in configuring and running sphinx. I have just scratched the surface here and this blog doesn't attempt to cover all the details.





