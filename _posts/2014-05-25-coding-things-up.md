---
layout: post
title: "Coding things up!"
description: ""
category: gsoc2014
tags: []
---
{% include JB/setup %}

**Update**: *MariaDB is now on [github](https://github.com/MariaDB/server). I have created my own [fork](https://github.com/igniting/server), and pushed my changes on to the [selfTuningOptimizer](https://github.com/igniting/server/tree/selfTuningOptimizer) branch.*

In the [previous blog post]({% post_url 2014-05-11-first-steps %}), I had stated that I will start coding the things discussed in it. I have set up a [branch](https://code.launchpad.net/~igniting/maria/maria) on launchpad and have pushed a [commit](http://bazaar.launchpad.net/~igniting/maria/maria/revision/4211) which I will be discussing in this post.

I first needed to make a table in the mysql database, from where we would be reading constants and later updating them too. The sql commands for creating a table in the mysql database are written in the file `scripts/mysql_system_tables.sql`. I have added a table named `all_constants` which currently has only two columns `const_name` and `const_value`.

{% highlight mysql %}
-- Tables for Self Tuning Cost Optimizer

CREATE TABLE IF NOT EXISTS all_constants (const_name varchar(64) NOT NULL, const_value double NOT NULL, PRIMARY KEY (const_name) ) ENGINE=MyISAM CHARACTER SET utf8 COLLATE utf8_bin comment='Constants for optimizer';

-- Remember for later if all_constants table already existed
set @had_all_constants_table= @@warning_count != 0;
{% endhighlight %}

I also added the constants `READ_TIME_FACTOR` and `SCAN_TIME_FACTOR` in the `all_constants` table. Here is the corresponding code from `scripts/mysql_system_tables_data.sql`.

{% highlight mysql %}
CREATE TEMPORARY TABLE tmp_all_constants LIKE all_constants;
INSERT INTO tmp_all_constants VALUES ('READ_TIME_FACTOR', 1.0);
INSERT INTO tmp_all_constants VALUES ('SCAN_TIME_FACTOR', 1.0);
INSERT INTO all_constants SELECT * FROM tmp_all_constants WHERE @had_all_constants_table=0;
DROP TABLE tmp_all_constants;
{% endhighlight %}

Next I needed to read these constants from the `all_constants` table. The file `sql/statistics.cc` has functions to update persistent statistical tables and to read from them. I have created two files `sql/opt_costmodel.h` and `sql/opt_costmodel.cc` which will have functions to read and update `all_constants` table. I have created `All_constants` class. Its methods allow us to read constants from the `all_constants` table. I have still not written the code for updating the table. Let us have a look at the function `read_constant_from_table`.

{% highlight cpp %}
static double read_constant_from_table(THD *thd, const LEX_STRING const_name)      
{ 
  TABLE_LIST table_list;
  Open_tables_backup open_tables_backup;                                           
  if (open_table(thd, &table_list, &open_tables_backup, FALSE))                    
  { 
    thd->clear_error();                                                            
    return 1.0;                                                                    
  }                                                                                
  All_constants all_constants(table_list.table, const_name);                       
  all_constants.set_key_fields();
  all_constants.read_const_value();                                                
  close_system_tables(thd, &open_tables_backup);                                   
  return all_constants.get_const_value();                                          
}
{% endhighlight %}

`open_table` is a helper function which opens the `all_constants` table for reading/writing. `set_key_fields` sets the value of `const_name_field`, which is the primary key and `read_const_value` searches for the constant having that name in `all_constants` table and fills that value in `const_value` variable.

{% highlight cpp %}
void set_key_fields()
{
  const_name_field->store(const_name.str, const_name.length, system_charset_info);
}
bool find_const()
{
  uchar key[MAX_KEY_LENGTH];
  key_copy(key, record[0], key_info, key_length);
  return !all_constants_file->ha_index_read_idx_map(record[0], key_idx, key,
                                                    HA_WHOLE_KEY, HA_READ_KEY_EXACT);
}
void read_const_value()
{
  if (find_const())
  {
    Field *const_field= all_constants_table->field[ALL_CONSTANTS_CONST_VALUE];
    if(!const_field->is_null())
    {
      const_value= const_field->val_real();
    }
  }
}
{% endhighlight %}

So to read `READ_TIME_FACTOR` from the `all_constants` table, a call to read_constant_from_table is made with `const_name` set as `READ_TIME_FACTOR`. Similar is the case for `SCAN_TIME_FACTOR`.

Next big thing was how to test the code which I had just written. `./mtr` was showing a lot of failed tests because I had added an extra table in the system database. So I decided to write a simple test of mine own in `mysql-test/t/costmodel.test`.

{% highlight mysql %}
--disable_warnings
DROP TABLE IF EXISTS t1;
--enable_warnings
CREATE TABLE t1 (a INT);
INSERT INTO t1 VALUES (1);
SELECT * FROM t1;
DROP TABLE t1;
{% endhighlight %}

This test just creates a table with one field and runs a select query. To test my code, I added calls to `get_read_time_factor` and `get_scan_time_factor` in `sql/sql_select.cc`. Then I ran `./mtr --gdb costmodel` to run the test-case with gdb. Using gdb I set up breakpoints on these functions and confirmed that the correct values were indeed being read.

Now that we have functions which will read the constants from the database, I need to figure out where all I need to call this functions. I made a list of places where `handler::read_time()` and `handler::scan_time()` were being called. The problem is that `get_read_time_factor` and `get_scan_time_factor` **requires a pointer to current thread as an argument**. I will tackle this problem in the coming week. Also I need **an example query** to add to my test case, which will take different execution plans on changing the values of `READ_TIME_FACTOR` and `SCAN_TIME_FACTOR`.

Suggestions/review on the code are welcome. Again, [this](http://bazaar.launchpad.net/~igniting/maria/maria/revision/4211) is the link to the commit. I will be pushing to that branch regularly from now on.
