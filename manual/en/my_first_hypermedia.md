---
layout: default
title: BEAR.Sunday | My First Hypermedia
category: My First - Tutorial
---

# My First Hypermedia

## What is Hypermedia?

In 1962 Ted Nelson proposed [**Hypertext**](http://en.wikipedia.org/wiki/Hypertext).
This is when in order to refer some text other text referral links are embedded in the text itself, the referrals that joins the text are called Hyperlinks.

The most famous and successful implementation of Hypertext is the Worldwide Web. (The href in a property of the `<a>` tag is an abbreviation for hyper-reference.)

Also to note is that PHP is an acronym of *PHP: Hypertext Preprocessor* ([PHP an acronym for what?](http://www.php.net/manual/en/faq.general.php#faq.general.acronym))

## Non Existent Hypermedia

Let's think in terms of a `REST API` for example when you order a coffee at a coffee shop.

When you order a coffee, the `REST API` is provided with the following.

| type   | value                               |
|--------|-------------------------------------|
| METHOD | POST                                |
| URI    | http://restbucks.com/order/{?drink} |
| Query  | drink=Drink Name                    |

You use this `API` when ordering a drink. When using this API you create a (POST) `Order Resource`.

```
post http://restbucks.com/order/?drink=latte
```

The order resource has been created and the order contents are returned.

```json
{
    "drink": "latte",
    "cost": 2.5,
    "id": "5052",
}
```

This is *not hypermedia*. The data does not have any attached uniquely displayed URI's or related links.

## HAL - Hypertext Application Language

JSON is not essentially a hypermedia format, however using JSON the
[HAL - Hypertext Application Language](http://stateless.co/hal_specification.html)
which is a [RFC Draft Standard](http://tools.ietf.org/html/draft-kelly-json-hal-00)
is used to provide `JSON+HAL` hyper-media.

In BEAR.Sunday when you set your resource rendering to `HalRenderer` you can output in HAL format.

```json
{
    "drink": "latte",
    "cost": 2.5,
    "id": "1545",
    "_links": {
        "self": {
            "href": "app://self/restbucks/order?id=1545"
        },
        "payment": {
            "href": "app://self/restbucks/payment?id=1545"
        }
    }
}
```

This is an order resource output in the `HAL` Format.
The URI's and related link information for itself are embedded in the `_links` property.
The order and payment relationship is not saved by the client, but by the service.

On the service side you can change the link references according to service circumstances.
In those times you need to change nothing on the client, just carry on following the provided link.
By having links you transform your service from just another data format to a self descriptive Hyper-Media resource.

## Adding Hyperlinks

You declare your resource object's `links` property like this.

{% highlight php startinline %}
    public $links = [
        'news' => [Link::HREF > 'page://self/news/today']
    ];
{% endhighlight %}

## Using a URI Template for your Query

When the URI to dynamically decided for example you can create a query in the onPost method like this.

{% highlight php startinline %}
$this->links['friend'] = [Link::HREF => "app://self/sns/friend?id{$id}"];
{% endhighlight %}

In the `links` property you can set the URI template like this.

{% highlight php startinline %}
    public $links => [
        'friend' => [
            Link::HREF => 'app://self/sns/friend{?id}',
            Link::TEMPLATED => true
        ]
    ];
{% endhighlight %}

Here the necessary variable `{id}` is retrieved from the resource `body`.

## Let's Try

Here is the class that assigns `$item` and creates the order resource.

{% highlight php startinline %}
<?php

namespace Demo\Sandbox\Resource\App\First\Hypermedia;

use BEAR\Resource\ResourceObject;
use BEAR\Resource\Link;

/**
 * Order resource
 */
class Order extends ResourceObject
{
    /**
     * @param string $item
     *
     * @return Order
     */
    public function onPost($item)
    {
        $this['item'] = $item;
        $this['id'] = date('is'); // min+sec
        return $this;
    }
}
{% endhighlight %}

In order to add hyperlinks setup the `links` property.

{% highlight php startinline %}
    public $links = [
        'payment' => [
            Link::HREF => 'app://self/first/hypermedia/payment{?id}',
            Link::TEMPLATED => true
        ]
    ];
{% endhighlight %}

## Make API Request From the Console

```
$ php apps/Demo.Sandbox/bootstrap/contexts/api.php post app://self/first/hypermedia/order?item=book

200 OK
content-type: ["application\/hal+json; charset=UTF-8"]
cache-control: ["no-cache"]
date: ["Thu, 26 Jun 2014 07:26:01 GMT"]
[BODY]
item book,
id 2601,

[VIEW]
{
    "item": "book",
    "id": "2601",
    "_links": {
        "self": {
            "href": "http://localhost/app/first/hypermedia/order/?item=book"
        },
        "payment": {
            "href": "http://localhost/app/first/hypermedia/payment{/?id}",
            "templated": true
        }
    }
}
```

The `payment` link now appears.

## Using Links in a Program

In order to use links in your code, inject the `A` object using the trait `AInject` and use the `href` method to retrieve links.
The resource body can retrieve the link composed by the URI template.

{% highlight php startinline %}
<?php

namespace Demo\Sandbox\Resource\App\First\Hypermedia;

use BEAR\Resource\ResourceObject;
use BEAR\Sunday\Inject\AInject;
use BEAR\Sunday\Inject\ResourceInject;

/**
 * Shop resource
 */
class Shop extends ResourceObject
{
    use ResourceInject;
    use AInject;

    /**
     * @param string $item
     * @param string $card_no
     *
     * @return Shop
     */
    public function onPost($item, $card_no)
    {
        $order = $this
            ->resource
            ->post
            ->uri('app://self/first/hypermedia/order')
            ->withQuery(['item' => $item])
            ->eager
            ->request();

        $payment = $this->a->href('payment', $order);

        $this->resource
            ->put
            ->uri($payment)
            ->withQuery(['card_no' => $card_no])
            ->request();

        $this->code = 204;

        return $this;
    }
}
{% endhighlight %}

Just like on a web page and you just click a link to go on to the next page, you are now able to control the next links in the service layer.
Even when the links change there is no need for any change in the client.
