---
layout:           post
title:            "MySQL returns duplicates when sorting in descending order."
date:             2018-03-05 15:29:00 +0900
last_modified_at: 2018-03-05 16:15:00 +0900
tags:             [mysql, docker, duplicate, entries, pagination, limit, offset, descending, order]
introduction:     "I discovered that MySQL sometimes returns the same entries multiple times when paginating a result set. I noticed that it does not happen when sorting in ascending order or when leaving out the order by clause from the query."
---

So I got a mail from a client this morning saying that there was an error in the API I'm working on because it was returning duplicates. The client noticed that in the result set that was paginated over four pages, pages two and three had two duplicate IDs in them. This shouldn't be possible since these IDs are in an auto-increment column in the database table.

This application is built with the Laravel based framework Lumen so I figured that it had to have something to do with the logic in the Laravel ORM, Eloquent. I thought that it might have been some bug that got fixed in more recent versions of the framework so I started googling for similar cases. I couldn't find any information on similar issues so I figured that I would just extract the raw queries from the ORM and run them directly in MySQL so I fired up Sequel Pro and started querying the database.

I'm running MySQL 5.7.21 in Docker and haven't tested other MySQL versions so I'm not sure if this is specific to this version. However, once I ran the queries in the database I found out that the problem was not in the Eloquent ORM but in MySQL itself. I was relieved that it was not the application but at the same time perplexed by not knowing how to fix this issue since I do not know the internals of MySQL.

The dataset that I was working with has a lot of created_at timestamps with the exact same date. MySQL returned the correct results when I was not using offset and limit to paginate the result. The same went for when I was sorting the data in ascending order and also when leaving out the order by clause in the query. The problem occurred when using offset and limit to paginate the result set and at the same time sorting it by created_at timestamp in descending order.

The following four queries were the ones that I executed to paginate the total 190 entries into four pages. Sure enough, the IDs `21990` and `21991` appeared on both page number 2 and 3.

{% highlight plaintext %}
select id,created_at from `products` where `status` <> 0 `created_at` desc limit 50 offset 0
select id,created_at from `products` where `status` <> 0 `created_at` desc limit 50 offset 50
select id,created_at from `products` where `status` <> 0 `created_at` desc limit 50 offset 100
select id,created_at from `products` where `status` <> 0 `created_at` desc limit 50 offset 150
{% endhighlight %}

I'm pretty sure this will not happen if you do not have many entries with the same value. This might be an optimization they have implemented in the MySQL engine on purpose so that the database does not scan all entries before getting the offset in the dataset. Instead, in this case it seems to start fetching entries when it has found, in this case, 50 matching entries.

Let me know if I got this completely wrong.

I'll finish the article by dropping the whole dataset here, have a good one!

{% highlight plaintext %}
select id,created_at from `products` where `status` <> 0 order by `created_at` desc
21914   2018-03-02 13:08:42
22077   2018-03-02 12:20:45
21975   2018-03-02 10:46:55
21786   2018-03-01 11:04:22
22050   2018-02-28 12:38:41
22051   2018-02-28 12:38:41
22052   2018-02-28 12:38:41
22053   2018-02-28 12:38:41
22054   2018-02-28 12:38:41
22055   2018-02-28 12:38:41
22056   2018-02-28 12:38:41
22057   2018-02-28 12:38:41
22058   2018-02-28 12:38:41
22059   2018-02-28 12:38:41
22060   2018-02-28 12:38:41
22061   2018-02-28 12:38:41
22062   2018-02-28 12:38:41
22063   2018-02-28 12:38:41
22064   2018-02-28 12:38:41
22065   2018-02-28 12:38:41
22066   2018-02-28 12:38:41
22067   2018-02-28 12:38:41
22068   2018-02-28 12:38:41
22069   2018-02-28 12:38:41
22070   2018-02-28 12:38:41
22071   2018-02-28 12:38:41
22072   2018-02-28 12:38:41
22073   2018-02-28 12:38:41
22074   2018-02-28 12:38:41
22075   2018-02-28 12:38:41
22076   2018-02-28 12:38:41
21995   2018-02-27 17:01:54
21998   2018-02-27 17:01:54
22000   2018-02-27 17:01:54
22001   2018-02-27 17:01:54
22003   2018-02-27 17:01:54
22005   2018-02-27 17:01:54
22007   2018-02-27 17:01:54
22009   2018-02-27 17:01:54
22010   2018-02-27 17:01:54
22011   2018-02-27 17:01:54
22013   2018-02-27 17:01:54
22014   2018-02-27 17:01:54
22015   2018-02-27 17:01:54
22018   2018-02-27 17:01:54
22019   2018-02-27 17:01:54
22020   2018-02-27 17:01:54
22021   2018-02-27 17:01:54
22022   2018-02-27 17:01:54
22023   2018-02-27 17:01:54
22024   2018-02-27 17:01:54
22025   2018-02-27 17:01:54
22026   2018-02-27 17:01:54
22028   2018-02-27 17:01:54
22030   2018-02-27 17:01:54
22033   2018-02-27 17:01:54
22036   2018-02-27 17:01:54
22038   2018-02-27 17:01:54
22040   2018-02-27 17:01:54
22041   2018-02-27 17:01:54
22043   2018-02-27 17:01:54
22045   2018-02-27 17:01:54
22047   2018-02-27 17:01:54
22048   2018-02-27 17:01:54
22049   2018-02-27 17:01:54
21869   2018-02-27 17:01:53
21870   2018-02-27 17:01:53
21872   2018-02-27 17:01:53
21873   2018-02-27 17:01:53
21874   2018-02-27 17:01:53
21876   2018-02-27 17:01:53
21877   2018-02-27 17:01:53
21878   2018-02-27 17:01:53
21879   2018-02-27 17:01:53
21880   2018-02-27 17:01:53
21881   2018-02-27 17:01:53
21882   2018-02-27 17:01:53
21884   2018-02-27 17:01:53
21885   2018-02-27 17:01:53
21886   2018-02-27 17:01:53
21887   2018-02-27 17:01:53
21888   2018-02-27 17:01:53
21889   2018-02-27 17:01:53
21890   2018-02-27 17:01:53
21893   2018-02-27 17:01:53
21894   2018-02-27 17:01:53
21895   2018-02-27 17:01:53
21896   2018-02-27 17:01:53
21898   2018-02-27 17:01:53
21899   2018-02-27 17:01:53
21901   2018-02-27 17:01:53
21904   2018-02-27 17:01:53
21906   2018-02-27 17:01:53
21909   2018-02-27 17:01:53
21910   2018-02-27 17:01:53
21911   2018-02-27 17:01:53
21912   2018-02-27 17:01:53
21915   2018-02-27 17:01:53
21916   2018-02-27 17:01:53
21917   2018-02-27 17:01:53
21918   2018-02-27 17:01:53
21919   2018-02-27 17:01:53
21920   2018-02-27 17:01:53
21921   2018-02-27 17:01:53
21922   2018-02-27 17:01:53
21923   2018-02-27 17:01:53
21924   2018-02-27 17:01:53
21925   2018-02-27 17:01:53
21927   2018-02-27 17:01:53
21929   2018-02-27 17:01:53
21931   2018-02-27 17:01:53
21933   2018-02-27 17:01:53
21934   2018-02-27 17:01:53
21936   2018-02-27 17:01:53
21938   2018-02-27 17:01:53
21940   2018-02-27 17:01:53
21942   2018-02-27 17:01:53
21944   2018-02-27 17:01:53
21945   2018-02-27 17:01:53
21946   2018-02-27 17:01:53
21947   2018-02-27 17:01:53
21948   2018-02-27 17:01:53
21951   2018-02-27 17:01:53
21953   2018-02-27 17:01:53
21956   2018-02-27 17:01:53
21958   2018-02-27 17:01:53
21960   2018-02-27 17:01:53
21961   2018-02-27 17:01:53
21963   2018-02-27 17:01:53
21965   2018-02-27 17:01:53
21967   2018-02-27 17:01:53
21968   2018-02-27 17:01:53
21969   2018-02-27 17:01:53
21970   2018-02-27 17:01:53
21972   2018-02-27 17:01:53
21977   2018-02-27 17:01:53
21978   2018-02-27 17:01:53
21980   2018-02-27 17:01:53
21981   2018-02-27 17:01:53
21982   2018-02-27 17:01:53
21983   2018-02-27 17:01:53
21985   2018-02-27 17:01:53
21987   2018-02-27 17:01:53
21988   2018-02-27 17:01:53
21989   2018-02-27 17:01:53
21990   2018-02-27 17:01:53
21991   2018-02-27 17:01:53
21992   2018-02-27 17:01:53
21855   2018-02-27 17:01:52
21857   2018-02-27 17:01:52
21859   2018-02-27 17:01:52
21861   2018-02-27 17:01:52
21864   2018-02-27 17:01:52
21867   2018-02-27 17:01:52
21780   2018-02-23 12:16:33
21772   2018-02-23 12:15:54
21812   2018-02-23 12:15:28
21802   2018-02-23 12:14:30
21733   2018-02-19 00:01:01
21735   2018-02-19 00:01:01
21736   2018-02-19 00:01:01
21737   2018-02-19 00:01:01
21738   2018-02-19 00:01:01
21739   2018-02-19 00:01:01
21741   2018-02-19 00:01:01
21742   2018-02-19 00:01:01
21743   2018-02-19 00:01:01
21746   2018-02-19 00:01:01
21747   2018-02-19 00:01:01
21748   2018-02-19 00:01:01
21749   2018-02-19 00:01:01
21751   2018-02-19 00:01:01
21752   2018-02-19 00:01:01
21753   2018-02-19 00:01:01
21754   2018-02-19 00:01:01
21755   2018-02-19 00:01:01
21756   2018-02-19 00:01:01
21757   2018-02-19 00:01:01
21759   2018-02-19 00:01:01
21761   2018-02-19 00:01:01
21763   2018-02-19 00:01:01
21766   2018-02-19 00:01:01
21767   2018-02-19 00:01:01
21503   2018-02-07 09:07:41
21547   2018-01-31 23:01:02
21548   2018-01-31 23:01:02
21281   2018-01-04 13:26:44
21240   2017-12-31 23:01:04
21250   2017-12-31 23:01:04
6   2006-02-04 14:50:49
{% endhighlight %}
