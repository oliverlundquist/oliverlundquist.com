---
layout:           post
title:            "How to rewrite a legacy PHP application to Laravel."
date:             2018-12-22 13:18:20 +0900
last_modified_at: 2018-12-22 14:25:30 +0900
tags:             [legacy, app, application, refactor, rewrite, fdd, laravel, lumen]
introduction:     "Rewriting a legacy PHP application to Laravel doesn't have to be an all or nothing process, in this article, I go through how you can rewrite your app into feature classes in Laravel step by step while still using code from your old app."
---

I've done a couple of rewrites of existing systems in my career. The largest one so far was a rewrite of a big blackbox e-commerce system, where I was the lead engineer of a team of 3-4 engineers which took 1 year and 3 months in total to complete.

My experience is that if you own the code that you'll be rewriting, it will make it easier to start small and iterate quick. Getting results quickly and constantly is going to keep the motivation level up and increase the chance of a successfully completed rewrite. A far worse scenario would be if you had to do a full replacement and rewrite the whole old application before switching over to the new one. These kind of rewrites demand a lot more planning and resource allocation and are harder to complete.

One approach that I think works very well with this type of iterative, step by step rewrites is Feature Driven Development (FDD). If you haven't heard about it, take a quick look at an article I wrote a couple of months ago [here](/2018/08/27/why-fdd-is-better-than-ddd.html).

When starting off a rewrite, instead of spending time defining domain models and it's boundaries in your business logic, it's better to think simple and start rewriting the actual code as early as possible and in as small chunks as possible to get results quick.

I think this is where FDD really shines. FDD is based on features and therefore makes it easy to isolate tasks without them having dependencies on each other. Since tasks are isolated and not blocking each other it also makes it easier to split up the work between different engineers to proceed with the rewrite in parallel.

### The legacy app

So you have your old application that hasn't got the attention it deserves and it has been neglected for too long when you or your company finally decides that it's time to do something about it. When I'm talking about a legacy application in this article, I'm referring to how we used to write PHP applications back in the day. That is, with inline PHP inside HTML markup, without any separation of logic whatsoever. What you see in your single PHP file is what you'll get when you access it in a web browser.

Rewriting a legacy application into more structured code is a huge gain itself. But probably the bigger merit is that you can leverage the power of modern frameworks like Laravel, where advanced features such as queues, caches, websockets, object-relational mapping, templates and more, are just one line of code away. Thanks to open-source code projects, we don't need to write loads of boilerplate code anymore, hurray! 🎉

### Checking for dependency conflicts

I'm using Laravel in this rewrite so the first step is to make sure that we can use it in our application. Start with checking the system requirements for Laravel [here](https://laravel.com/docs/5.7#server-requirements){:target="_blank"}. If you're using composer in your application already we need to check that your dependencies aren't too old and satisfy the requirements that Laravel has. Run the following command to check if you can install Laravel in your application with your current dependencies:

{% highlight plaintext %}
composer require laravel/laravel && composer remove laravel/laravel
{% endhighlight %}

If this command finished without errors, you're all set and good to go.

### Install Laravel in a subfolder

Now, we're going to work with Laravel installed in a subfolder in your current application folder root. Run this command to install Laravel in a subfolder called, laravel.

{% highlight plaintext %}
composer create-project --prefer-dist laravel/laravel laravel
{% endhighlight %}

### Sample legacy app

For the purpose of this demo, I've created a simple product catalog application.

![Legacy App Demo Screenshot](/assets/legacy-app-demo-screenshot.jpg){:class="image-with-border"}

The code is all contained in one PHP file and looks like this.

{% highlight php %}
<?php

require_once './lib/database.php';

// check if it's an update request, if so, perform update operations
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if ($_POST['action'] === 'update_display_in_shop') {
        if ($_POST['display_in_shop'] !== 'yes' && $_POST['display_in_shop'] !== 'no') {
            exit(1);
        }
        $sql = 'UPDATE products SET display_in_shop=' . ($_POST['display_in_shop'] === 'yes' ? '1' : '0') . ' WHERE id=' . intval($_POST['product_id']);
        if ($mysqli->query($sql) !== TRUE) {
            exit(1);
        }
    }
    header('Location: /products_list.php');
    exit(0);
}

// get products
$sql = "SELECT * FROM products";

// error
if (!$result = $mysqli->query($sql)) {
    echo "Sorry, the website is experiencing problems.";
    echo "Error: Our query failed to execute and here is why: \n";
    echo "Query: " . $sql . "\n";
    echo "Errno: " . $mysqli->errno . "\n";
    echo "Error: " . $mysqli->error . "\n";
    exit;
}

// results
if ($result->num_rows === 0) {
    echo "Your product catalogue is empty.";
    exit;
}


$products = [];
while ($product = $result->fetch_object()) {
    $products[] = $product;
}

// html
?>
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>Product List</title>
        <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
        <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.1.3/css/bootstrap.min.css" integrity="sha384-MCw98/SFnGE8fJT3GXwEOngsV7Zt27NXFoaoApmYm81iuXoPkFOJwJ8ERdknLPMO" crossorigin="anonymous">
    </head>
    <body>
        <div class="container">
            <div class="row">
                <div class="col">
                    <h1>Product List</h1>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Product ID</th>
                                <th>Product Name</th>
                                <th>Display in Shop?</th>
                            </tr>
                        </thead>
                        <tbody>
                            <?php foreach ($products as $product) : ?>
                                <tr>
                                    <td><?php echo $product->id;?></td>
                                    <td><?php echo $product->name;?></td>
                                    <td>
                                        <form method="post" action="products_list.php">
                                            <input type="hidden" name="action" value="update_display_in_shop">
                                            <input type="hidden" name="product_id" value="<?php echo $product->id;?>">
                                            <input type="radio" name="display_in_shop" value="yes"<?php echo $product->display_in_shop === '1' ? ' checked' : '';?>> Yes
                                            <input type="radio" name="display_in_shop" value="no"<?php echo $product->display_in_shop !== '1' ? ' checked' : '';?>> No
                                            <button type="submit" class="btn btn-info">Update!</button>
                                        </form>
                                    </td>
                                </tr>
                            <?php endforeach; ?>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </body>
</html>
{% endhighlight %}

### Identifying features in your code

Now, to start the rewriting of your code you need to identify features in your code. In the example application above we only have two, one is listing of products and one is updating the "display in shop" attribute.

To keep things simple, let's create a folder called "Features" inside our `./laravel/app` folder, you could put these feature classes anywhere you want inside the laravel folder, I'm putting them here because this folder is already set to be auto-loaded in composer by Laravel.

Here are the features encapsulated into feature classes.

{% highlight php %}
<?php

namespace App\Features\Products\Fetch;

class GetAllProducts
{
    public function execute()
    {
        global $mysqli;

        $products = [];

        $sql = "SELECT * FROM products";

        // error
        if (!$result = $mysqli->query($sql)) {
            echo "Sorry, the website is experiencing problems.";
            echo "Error: Our query failed to execute and here is why: \n";
            echo "Query: " . $sql . "\n";
            echo "Errno: " . $mysqli->errno . "\n";
            echo "Error: " . $mysqli->error . "\n";
            exit;
        }

        // results
        if ($result->num_rows === 0) {
            echo "Your product catalogue is empty.";
            exit;
        }

        while ($product = $result->fetch_object()) {
            $products[] = $product;
        }

        return $products;
    }
}
{% endhighlight %}

{% highlight php %}
<?php

namespace App\Features\Products\Update;

use Illuminate\Http\Request;
use App\Models\Products;

class UpdateDisplayInShopAttribute
{
    public function execute()
    {
        $request       = app('request');
        $input         = $request->input();
        $validatedData = $request->validate([
            'product_id'      => 'required|exists:products,id',
            'display_in_shop' => 'required|in:yes,no'
        ]);
        $product_id      = $validatedData['product_id'];
        $display_in_shop = $validatedData['display_in_shop'] === 'yes' ? 1 : 0;

        // update database
        $product = Products::find($product_id);
        $product->display_in_shop = $display_in_shop;
        $product->save();
    }
}
{% endhighlight %}

### Testing your feature classes

The good thing about small feature classes is that they're really easy to test. Before rewriting them to use Laravel code, write a quick test to make sure that the code works as expected. While this is an optional step, I strongly encourage writing some test cases, it will save you many headaches further down the road and also give you confidence that the code works as excepted.

{% highlight php %}
<?php

namespace Tests\Feature\Products\Fetch;

use Tests\TestCase;
use App\Features\Products\Fetch\GetAllProducts;

// TODO: remove once we've rewritten GetAllProducts to use Eloquent
require_once __DIR__ . '/../../../../../lib/database.php';

class GetAllProductsTest extends TestCase
{
    public function testGetAllProductsTest()
    {
        $products = (new GetAllProducts)->execute();

        $this->assertSame(3, count($products));
        $this->assertSame('Product A', $products[0]->name);
        $this->assertSame('Product B', $products[1]->name);
        $this->assertSame('Product C', $products[2]->name);
    }
}
{% endhighlight %}

{% highlight php %}
<?php

namespace Tests\Feature\Products\Update;

use Tests\TestCase;
use App\Features\Products\Update\UpdateDisplayInShopAttribute;
use App\Models\Products;

class UpdateDisplayInShopAttributeTest extends TestCase
{
    public function testUpdateDisplayAttributeSuccess()
    {
        $request = app('request');
        $request->replace([
            'product_id'      => 1,
            'display_in_shop' => 'yes'
        ]);
        (new UpdateDisplayInShopAttribute)->execute();

        $this->assertSame(1, Products::find(1)->display_in_shop);
    }

    public function testUpdateFailWithInvalidId()
    {
        // assert exception
        $this->expectException(\Illuminate\Validation\ValidationException::class);

        $request = app('request');
        $request->replace([
            'product_id'      => 9999,
            'display_in_shop' => 'yes'
        ]);
        (new UpdateDisplayInShopAttribute)->execute();
    }

    public function testUpdateFailWithInvalidDisplayInShop()
    {
        // assert exception
        $this->expectException(\Illuminate\Validation\ValidationException::class);

        $request = app('request');
        $request->replace([
            'product_id'      => 1,
            'display_in_shop' => 'invalid-value'
        ]);
        (new UpdateDisplayInShopAttribute)->execute();
    }
}
{% endhighlight %}

### Bootstrapping Laravel inside your legacy app

Now that we've moved our features into feature classes inside the laravel subfolder we need to bootstrap the Laravel framework to start using them. However, we can't just bootstrap the framework like we normally would through the router since we're not going to be using the Laravel routing system just yet.

Let's create a file called bootstrap_laravel.php in the root of our legacy application with the following contents.
{% highlight php %}
<?php

require __DIR__.'/laravel/vendor/autoload.php';

$app = require_once __DIR__.'/laravel/bootstrap/app.php';

$request = Illuminate\Http\Request::capture();
$app->instance('request', $request);
Illuminate\Support\Facades\Facade::clearResolvedInstance('request');

$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);
$kernel->bootstrap();

// below you could set global configuration values, etc.
config(['global_configuration_value' => 'value']);
{% endhighlight %}

Now, let's tweak the original PHP file to bootstrap Laravel and use these feature classes.

{% highlight php %}
<?php

use App\Features\Products\Update\UpdateDisplayInShopAttribute;
use App\Features\Products\Fetch\GetAllProducts;

require_once './lib/database.php';

// bootstrap laravel
require_once './bootstrap_laravel.php';

// check if it's an update request, if so, perform update operations
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    (new UpdateDisplayInShopAttribute)->execute();
    header('Location: /products_list.php');
    exit(0);
}

// get products
$products = (new GetAllProducts)->execute();

// html
?>
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>Product List</title>
        <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
        <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.1.3/css/bootstrap.min.css" integrity="sha384-MCw98/SFnGE8fJT3GXwEOngsV7Zt27NXFoaoApmYm81iuXoPkFOJwJ8ERdknLPMO" crossorigin="anonymous">
    </head>
    <body>
<!-- ... -->
{% endhighlight %}

### Rewriting your features to Laravel code

After moving your code into feature classes in the laravel subfolder and bootstrapping Laravel the application will work just like before. But we don't want to stop there, the whole point of this rewrite is to enable you to use all the powerful features in the Laravel framework. Let's remove the MySQLi logic and create an Eloquent model in a new folder called Models in the `./laravel/app` folder.

{% highlight php %}
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Products extends Model {
    /**
     * Table definition variable.
     *
     * @var string
     */
    protected $table = 'products';

    /**
     * Mass-assign guarded keys.
     *
     * @var array
     */
    protected $guarded = ['id'];

    /**
     * Set primary key for table.
     *
     * @var string
     */
    protected $primaryKey = 'id';

    /**
     * Auto-increment primary key.
     *
     * @var bool
     */
    public $incrementing = true;

    /**
     * Toggle insertion of timestamps.
     *
     * @var bool
     */
    public $timestamps = false;
}
{% endhighlight %}

The GetAllProducts feature class can now be slimmed down to just a one-liner replacing all that raw database fetching logic we had before.

{% highlight php %}
<?php

namespace App\Features\Products\Fetch;

use App\Models\Products;

class GetAllProducts
{
    public function execute()
    {
        return Products::all();
    }
}
{% endhighlight %}

### Finishing off one page

Now that we've rewritten all our features, let's take the next and final step and move the last piece into Laravel, the template. We'll need to create a route, controller and a blade template file for this.

First, start by defining new routes in your `./laravel/routes/web.php` file. Since we now have a powerful router at our hands, we do not want to handle update requests and products list requests in the same file anymore, so let's define two routes, one GET and one PUT for updating.

{% highlight php %}
<?php

Route::get('products_list.php', 'ProductsListController@index');
Route::put('products_list.php', 'ProductsListController@update');
{% endhighlight %}

Next, create your controller, add a `ProductsListController.php` file into your `./laravel/app/Http/Controllers` folder with the following content.
{% highlight php %}
<?php

namespace App\Http\Controllers;

use App\Features\Products\Fetch\GetAllProducts;
use App\Features\Products\Update\UpdateDisplayInShopAttribute;

class ProductsListController {

    public function index()
    {
        $products = (new GetAllProducts)->execute();

        return view('products_list', compact('products'));
    }

    public function update()
    {
        (new UpdateDisplayInShopAttribute)->execute();

        // Once we run Laravel as a standalone app,
        // we can use Laravel's redirect helpers, like so:
        // redirect('/products_list.php');
        // but for now we have to redirect back to the /products_list.php file
        // to bootstrap Laravel
        header('Location: /products_list.php');
        exit(0);
    }
}
{% endhighlight %}

As you see, we bind all the products to the products_list view so that we can use them in our Laravel Blade template. Let's add a `products_list.blade.php` file in `./laravel/resources/views` with the following content.

{% highlight html %}
{% raw %}
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>Product List</title>
        <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
        <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.1.3/css/bootstrap.min.css" integrity="sha384-MCw98/SFnGE8fJT3GXwEOngsV7Zt27NXFoaoApmYm81iuXoPkFOJwJ8ERdknLPMO" crossorigin="anonymous">
    </head>
    <body>
        <div class="container">
            <div class="row">
                <div class="col">
                    <h1>Product List</h1>
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Product ID</th>
                                <th>Product Name</th>
                                <th>Display in Shop?</th>
                            </tr>
                        </thead>
                        <tbody>
                            @foreach ($products as $product)
                                <tr>
                                    <td>{{ $product->id }}</td>
                                    <td>{{ $product->name }}</td>
                                    <td>
                                        <form method="post" action="products_list.php">
                                            @csrf
                                            @method('PUT')
                                            <input type="hidden" name="action" value="update_display_in_shop">
                                            <input type="hidden" name="product_id" value="{{ $product->id }}">
                                            <input type="radio" name="display_in_shop" value="yes"{{ $product->display_in_shop === 1 ? ' checked' : '' }}> Yes
                                            <input type="radio" name="display_in_shop" value="no"{{ $product->display_in_shop !== 1 ? ' checked' : '' }}> No
                                            <button type="submit" class="btn btn-info">Update!</button>
                                        </form>
                                    </td>
                                </tr>
                            @endforeach
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </body>
</html>
{% endraw %}
{% endhighlight %}

Now that we do not have anything left of our old application logic in our old PHP file, it is now possible to use Laravel's own routing and we don't need the custom `./boostrap_laravel.php` file anymore. We can now bootstrap Laravel by simply doing a require on the `./laravel/public/index.php` file, like so.

{% highlight php %}
<?php
$_SERVER['SCRIPT_FILENAME'] = '';
require_once './laravel/public/index.php';
{% endhighlight %}

### Further steps

Now that we've completely rewritten one file in our legacy application and that we're using the Laravel router, one further step that you could take is to setup a rewrite from the old file `products_list.php` to just `/product_list`, that way we can be completely independent on old logic, making the transition to Laravel smooth.

I hope that this was an interesting read and that it gave you some new ideas on how to proceed with a rewrite of your legacy app. I would love your input and thoughts, please let me know in the comment box down below.

Until next time, have a good one!

For a complete diff of the files that were tweaked, checkout the link below:
- [https://github.com/oliverlundquist/rewrite-legacy-app-in-laravel/pull/1/files](https://github.com/oliverlundquist/rewrite-legacy-app-in-laravel/pull/1/files).
