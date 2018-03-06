---
layout:           post
title:            "How to get the count of a subquery in Laravel."
date:             2018-03-06 13:40:00 +0900
last_modified_at: 2018-03-06 14:01:00 +0900
tags:             [laravel, eloquent, orm, mysql, query, subquery, count, wrap]
introduction:     "I figured out how to wrap a query and getting the count from it in Laravel. It was not really straightforward and I had to use a combination of the Eloquent ORM and raw database queries to get the result that I was looking for."
---

So this time I had this rather complex MySQL query that I was trying to rewrite into beautiful fluent Eloquent ORM syntax. The query that I tried to write in Eloquent was a query wrapping another query into a subquery by putting it in a `from` clause to get the count from it.

I went through the official documentation and the Laravel source code for hours, trying stuff like `selectRaw()` and `selectSub()` to create a subquery and to somehow wrap the original query so that I could get the count from it.

Getting desperate, I even tried to use the `from()` method to change the target from a table name to the result of a subquery. However, none of this worked and I was back at square one.

The query that I wanted to use in Laravel was this one:
{% highlight sql %}
select count(*) as aggregate from (
	select *,
	(select products_description.products_name from products_description where products.products_id = products_description.products_id and products_description.language_id = 1) as name_no,
	(select products_description.products_name from products_description where products.products_id = products_description.products_id and products_description.language_id = 6) as name_se,
	(select products_description.products_slug from products_description where products.products_id = products_description.products_id and products_description.language_id = 1) as slug_no,
	(select products_description.products_slug from products_description where products.products_id = products_description.products_id and products_description.language_id = 6) as slug_se
	from `products`
	where `price` > 10 ? having `name_no` = skøyter and `slug_se` = skridskor order by `name_no` asc
) c
{% endhighlight %}

After a lot of trial and error creating custom Builder macros and custom methods, I managed to create the inner part of the query with Eloquent ORM. Now I just had to figure out how to wrap that in another query, this was the tricky part.

I spent a couple of hours trying out different ways of achieving this while going deeper and deeper in the Eloquent implementation. After trying a bunch of different options I finally went with the `select()` method on the `Illuminate\Database\Connection` class which is a super handy method. It takes a query and is pretty close to running a raw query but it also takes bindings for the query as an argument and takes care of all that query binding logic which makes your life so much easier.

The final code ended up something like this and I managed to get the count of my query while still using as much as possible of the beautiful syntax of the Eloquent ORM and I believe I found the sweet spot.

{% highlight php %}
$query = Products::where('price', '>', 10)
			->having('name_no', 'skøyter')
			->having('slug_se', 'skridskor')
			->select(
				'*',
				DB::raw("(select products_description.products_name from products_description where products.products_id = products_description.products_id and products_description.language_id = 1) as name_no"),
				DB::raw("(select products_description.products_name from products_description where products.products_id = products_description.products_id and products_description.language_id = 6) as name_se"),
				DB::raw("(select products_description.products_slug from products_description where products.products_id = products_description.products_id and products_description.language_id = 1) as slug_no"),
				DB::raw("(select products_description.products_slug from products_description where products.products_id = products_description.products_id and products_description.language_id = 6) as slug_se"),
			)
			->orderBy('name_no', 'asc');

$countQuery = "select count(*) as aggregate from ({$query->toSql()}) c";

$count = collect(DB::select($countQuery, $query->getBindings()))->pluck('aggregate')->first();
{% endhighlight %}

Although I didn't succeed at first, it was really interesting and informative to try out new methods on the Builder and Collection classes that I hadn't had a chance to try before.

I hope that this was interesting and informative for anyone out there trying to get the count of a subquery in Laravel.

Happy coding! &#128077;
