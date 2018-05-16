---
layout:           post
title:            "MySQL is not returning NULL values."
date:             2018-05-14 14:25:10 +0900
last_modified_at: 2018-05-14 15:37:20 +0900
tags:             [mysql, negative, values, null]
introduction:     "I discovered that MySQL doesn't return NULL values when running an exclude query while developing a filtering function for a system that I've been working on."
---

A couple of days ago I was working on a rather simple filtering function in a system that I'm building. I added a dropdown box in the UI where a user can select a status to exclude from a table view with results from a MySQL database.

A simplified version of the MySQL table looks something like this:

{% highlight sql %}
CREATE TABLE `orders` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `status` varchar(255) DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
{% endhighlight %}

In the status column, the table had string data but also NULL values since that column schema allowed NULL for that column. Here's an example of what kind of data that was in the table.

{% highlight sql %}
INSERT INTO `orders` (`id`, `status`)
VALUES
  (1, 'processed'),
  (2, 'pending'),
  (3, NULL),
  (4, 'shipped'),
  (5, ''),
  (6, 'paid');
{% endhighlight %}

The frontend part of the system sends requests to an API backend which then executes a MySQL query that looked something like this:

{% highlight sql %}
SELECT * FROM orders WHERE status <> 'pending'
{% endhighlight %}

I was expecting to get all the results back except the record with `ID = 2` since that row has the value of the status that I was excluding. However, to my surprise I got this:
{% highlight plaintext %}
id  status
1 processed
4 shipped
5 ''
6 paid
{% endhighlight %}

I found out later, that when you use not equal to (`<>` or `!=`) in MySQL it also excludes all NULL values. To include NULL values you have to explicitly include them by running a query like so:

{% highlight sql %}
SELECT * FROM orders WHERE status != 'pending' OR status IS NULL
{% endhighlight %}

Some of you out there might know this already, but it was the first time for me to stumble upon this, so I thought I'd write a quick blog post to share what I discovered and learned.

Hope it was useful!

Until next time, have a good one!
