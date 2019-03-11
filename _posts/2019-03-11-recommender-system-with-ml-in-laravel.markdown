---
layout:           post
title:            "Building a Product Recommender System with Machine Learning in Laravel."
date:             2019-03-11 12:15:10 +0900
last_modified_at: 2019-03-11 15:26:40 +0900
tags:             [machine learning, machine, learning, ml, algorithm, product recommender, product, recommender, system, hamming distance, euclidean distance, jaccard similarity coefficient]
introduction:     "Building a product recommender system from scratch with machine learning algorithms such as Hamming distance, Euclidean distance and Jaccard index in Laravel with PHP."
---

In this article, we're going to build an item-based product recommender system from scratch. We are going to use machine learning to decide how similar two products are to each other in our sample dataset.

There are many different algorithms that you can use with machine learning and choosing the most appropriate one depends heavily on your dataset and what you want to accomplish. Often it makes sense to try out a few to see which one works out the best for your needs.

In the demo app we are creating in this article I will cover three of these algorithms.

### Supervised and Unsupervised Algorithms

There are two main groups of machine learning algorithms, supervised and unsupervised.

Supervised machine learning is used when you have labeled input data and you know the right answers for the output. It's very similar to having a teacher correcting the predictions that the algorithms make until they have reached an acceptable level of performance.

Unsupervised machine learning, on the other hand, is when your input data is not labeled and you do not have the correct answers for the output.
The algorithms will distribute and structure the input data so that you can more easily discover patterns and learn more about the data.

### User-Based and Item-Based Collaborative Filtering

There are two main methods of performing recommendations to users.

The first one is user-based. This method finds users with similar taste by comparing ratings and reviews of products that both you and that user liked. It then gets products that you haven't seen or rated but the other user with similar taste likes. These products will then get recommended to you.

The other method is based on items. This method doesn't take into account what other users like, but instead compare features of products that you liked, to find products with similar features to recommend.

With the first, user-based approach, you will need ratings and preferences from many users to be able to make good predictions which makes it a bit more difficult to get going when you're just starting out.

This is because not all users rate or review all products and finding users that overlap when you have many products will need a fair share of data from users. This is in contrast to the item-based approach, which only needs preferences from the specific user to make accurate recommendations.

Also, the similarity between users changes more often than the similarity between products. The similarity between users will change every time a user rate or review new products. In addition, a store usually has more users than products and therefore calculating the similarity between users will be heavier than calculating the similarity between products.

### Let's Get Started

In the product recommender system we're building in this article we will be using an item-based approach with supervised algorithms.

We will be using product features (such as material, color, warmth rating etc.), product price and product categories as a base for our similarity calculations. The algorithms we will be using are [Hamming Distance](https://en.wikipedia.org/wiki/Hamming_distance){:target="_blank"} for product features, [Euclidean Distance](https://en.wikipedia.org/wiki/Euclidean_distance){:target="_blank"} for product price and [Jaccard similarity coefficient](https://en.wikipedia.org/wiki/Jaccard_index){:target="_blank"} for product categories.

![Recommender System Screenshot](/assets/recommender-system-screenshot-01.png){:class="image-with-border"}
*This is what we're building ladies and gentlemen.*

**Step 1**: So let's get right to it. Start by creating a new blank Laravel repository.

{% highlight plaintext %}
composer create-project --prefer-dist laravel/laravel laravel-recommender-system
{% endhighlight %}

**Step 2**: Get the dataset from the link below and save it to `./storage/data` in your Laravel repository. If you are planning to use this recommender system in production environments I strongly recommend that you build a couple of Eloquent models and seed this data into a database instead of having it as a JSON file.

[[storage/data/products-data.json]](https://github.com/oliverlundquist/laravel-recommender-system/blob/ed525e911ce1240f9bda6cb3c845fe494cc5e243/storage/data/products-data.json){:target="_blank"}

**Step 3**: Here's a utility class that I created with all the algorithms we'll need to calculate the similarity between products. Copy the contents to a file named `Similarity.php` in `./app`.

{% highlight php %}
<?php declare(strict_types=1);

namespace App;

class Similarity
{
    public static function hamming(string $string1, string $string2, bool $returnDistance = false): float
    {
        $a        = str_pad($string1, strlen($string2) - strlen($string1), ' ');
        $b        = str_pad($string2, strlen($string1) - strlen($string2), ' ');
        $distance = count(array_diff_assoc(str_split($a), str_split($b)));

        if ($returnDistance) {
            return $distance;
        }
        return (strlen($a) - $distance) / strlen($a);
    }

    public static function euclidean(array $array1, array $array2, bool $returnDistance = false): float
    {
        $a   = $array1;
        $b   = $array2;
        $set = [];

        foreach ($a as $index => $value) {
            $set[] = $value - $b[$index] ?? 0;
        }

        $distance = sqrt(array_sum(array_map(function ($x) { return pow($x, 2); }, $set)));

        if ($returnDistance) {
            return $distance;
        }
        // doesn't work well with distances larger than 1
        // return 1 / (1 + $distance);
        // so we'll use angular similarity instead
        return 1 - $distance;
    }

    public static function jaccard(string $string1, string $string2, string $separator = ','): float
    {
        $a            = explode($separator, $string1);
        $b            = explode($separator, $string2);
        $intersection = array_unique(array_intersect($a, $b));
        $union        = array_unique(array_merge($a, $b));

        return count($intersection) / count($union);
    }

    public static function minMaxNorm(array $values, $min = null, $max = null): array
    {
        $norm = [];
        $min  = $min ?? min($values);
        $max  = $max ?? max($values);

        foreach ($values as $value) {
            $numerator   = $value - $min;
            $denominator = $max - $min;
            $minMaxNorm  = $numerator / $denominator;
            $norm[]      = $minMaxNorm;
        }
        return $norm;
    }
}
{% endhighlight %}

**Step 4**: We will also need a class for managing the products and calculating the similarity between the products. Download the following contents to a file called `ProductSimilarity.php`, also in the root of your `./app` folder.

The `calculateSimilarityMatrix` method will calculate the similarity between all the products and create a matrix. If you are going to use this recommender system in production or if you have many products, I recommend that you make an Artisan command that calls this function periodically through a Scheduler and cache the result in Redis or similar. That way, you have the full similarity matrix cached and do not have to calculate it on every request.

{% highlight php %}
<?php declare(strict_types=1);

namespace App;

use Exception;

class ProductSimilarity
{
    protected $products       = [];
    protected $featureWeight  = 1;
    protected $priceWeight    = 1;
    protected $categoryWeight = 1;
    protected $priceHighRange = 1000;

    public function __construct(array $products)
    {
        $this->products       = $products;
        $this->priceHighRange = max(array_column($products, 'price'));
    }

    public function setFeatureWeight(float $weight): void
    {
        $this->featureWeight = $weight;
    }

    public function setPriceWeight(float $weight): void
    {
        $this->priceWeight = $weight;
    }

    public function setCategoryWeight(float $weight): void
    {
        $this->categoryWeight = $weight;
    }

    public function calculateSimilarityMatrix(): array
    {
        $matrix = [];

        foreach ($this->products as $product) {

            $similarityScores = [];

            foreach ($this->products as $_product) {
                if ($product->id === $_product->id) {
                    continue;
                }
                $similarityScores['product_id_' . $_product->id] = $this->calculateSimilarityScore($product, $_product);
            }
            $matrix['product_id_' . $product->id] = $similarityScores;
        }
        return $matrix;
    }

    public function getProductsSortedBySimularity(int $productId, array $matrix): array
    {
        $similarities   = $matrix['product_id_' . $productId] ?? null;
        $sortedProducts = [];

        if (is_null($similarities)) {
            throw new Exception('Can\'t find product with that ID.');
        }
        arsort($similarities);

        foreach ($similarities as $productIdKey => $similarity) {
            $id       = intval(str_replace('product_id_', '', $productIdKey));
            $products = array_filter($this->products, function ($product) use ($id) { return $product->id === $id; });
            if (! count($products)) {
                continue;
            }
            $product = $products[array_keys($products)[0]];
            $product->similarity = $similarity;
            $sortedProducts[] = $product;
        }
        return $sortedProducts;
    }

    protected function calculateSimilarityScore($productA, $productB)
    {
        $productAFeatures = implode('', get_object_vars($productA->features));
        $productBFeatures = implode('', get_object_vars($productB->features));

        return array_sum([
            (Similarity::hamming($productAFeatures, $productBFeatures) * $this->featureWeight),
            (Similarity::euclidean(
                Similarity::minMaxNorm([$productA->price], 0, $this->priceHighRange),
                Similarity::minMaxNorm([$productB->price], 0, $this->priceHighRange)
            ) * $this->priceWeight),
            (Similarity::jaccard($productA->categories, $productB->categories) * $this->categoryWeight)
        ]) / ($this->featureWeight + $this->priceWeight + $this->categoryWeight);
    }
}
{% endhighlight %}

**Step 5**: Now we only have to revise 2 more files and we will have our demo app up and running. These two files already exist in your Laravel repository, so just replace the contents with the ones that are in the links below.

[[resources/views/welcome.blade.php]](https://github.com/oliverlundquist/laravel-recommender-system/blob/ed525e911ce1240f9bda6cb3c845fe494cc5e243/resources/views/welcome.blade.php){:target="_blank"}

[[routes/web.php]](https://github.com/oliverlundquist/laravel-recommender-system/blob/ed525e911ce1240f9bda6cb3c845fe494cc5e243/routes/web.php){:target="_blank"}

**Step 6**: Now run `php artisan serve`, and you should see something like this!

![Recommender System Screenshot](/assets/recommender-system-screenshot-02.png){:class="image-with-border"}

Success! The similarity score is properly calculated and the recommendations are listed by similarity from highest to lowest.

### Adding Weights

In the above example, we calculate the product features, product price and product categories with the same importance or, weight. Depending on your dataset and needs, you might want to prioritize one of the criteria more than others and make that part of the similarity calculation more important.

As you might have noticed, there are properties for weight in the `ProductSimilarity.php` class used in the example above. If you adjust the values you can adjust the similarity calculations by, for example, putting a higher weight on product price to make that of greater importance when you compare products.

I have noticed that the Jaccard index, that I use in this article to calculate categories similarity might not be the best algorithm to use for this kind of calculations. That is because it makes it difficult to add weight to specific categories.

If you, for example, have: `men, the-north-face, winter-jacket` as your categories, the category `men` will have the same weight as the brand `the-north-face` and `winter-jacket` which would make women's jacket count as similar if it's a The North Face winter jacket. This is probably not the result you would be hoping for if you're a man looking for a jacket in an ecom store.

In this case, it will make more sense to use an algorithm where you can add weight to certain categories, so that you when you click on a men's jacket, you will actually get other recommendations for men's jackets. One example of this would be [tf–idf (frequency–inverse document frequency)](https://en.wikipedia.org/wiki/Tf%E2%80%93idf){:target="_blank"}.

That's all for this time, I hope that this was an interesting read and that it gave you some inspiration to dig deeper into this topic. I would love to hear your thoughts and comments, feel free to post anything in the comment box down below.

Until next time, have a good one!

For a complete diff of the files that were added in this blog post, check out the link below:
- [https://github.com/oliverlundquist/laravel-recommender-system/commit/ed525e911ce1240f9bda6cb3c845fe494cc5e243](https://github.com/oliverlundquist/laravel-recommender-system/commit/ed525e911ce1240f9bda6cb3c845fe494cc5e243).
