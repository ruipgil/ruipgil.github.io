---
layout: post
title: Introduction to web scraping
discussion: "https://news.ycombinator.com/item?id=10755576"
discussion_site: Hacker News
---

> **scrape** | verb | \ˈskrāp\
>
> to remove (something) from a surface by rubbing an object or tool against it
>
> in [Merriam-Webster](http://www.merriam-webster.com/dictionary/scrape)

In practice, and formal definitions apart, web scraping is the act of extracting information of a web page. Where is this information to be scraped? It can be in a range of different places. It could be in plain text, images, code, or event values hidden from the eyes of people (runtime variables).

If you're reading this post, chances are that you have already opened the Developer Tools and most specifically the javascript console in [Google Chrome](https://developer.chrome.com/devtools) or [Mozilla Firefox](https://developer.mozilla.org/en-US/docs/Tools/Web_Console/Opening_the_Web_Console). If not, give it a try.

The javascript console is great tool to learn how the web works. And guess what? It is the best place to start when you want to extract information.

Do you want to scrape all the links off of this page? Let's go,

1. Open up the javascript console;
2. Type ``` document.querySelectorAll('a') ``` and hit enter.

Simple, isn't it?
But not very practical if you want to do it for every single webpage.

To simplify this process I've created *[scraperjs](https://github.com/ruipgil/scraperjs)*.
It's a *node* package that simplifies the automation process of scraping. The fact that *scraperjs* is a *node* package might be a turn off for some people. However, as you've seen before, you're using the same language that the web is built upon.

## Scraperjs

Can you guess what the example below does?

``` javascript
var scraperjs = require('scraperjs');

scraperjs.StaticScraper.create('{{ site.url }}{{ site.baseurl }}{{ page.url }}')
  .scrape(function($) {
    return $('a').map(function() {
      return $(this).attr('href');
    }).get();
  })
  .then(function(links) {
    console.log(links);
  });

```

The small scraper above does the same thing as the Developer Tools example.

Now let me explain it,

+ We first load *scraperjs* to a variable with the same name;
+ We create a static scraper, more on that later, and give it a page url;
+ Then we set the ``` scrape ``` [promise](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise), with the function to use to scrape. We give it a callback function, that has one argument, and were the body is similar to what we've seen before. The dollar variable, it's *jQuery*.
You can do operations just like you'd do in your browser's console, but will have to use *jQuery* to access the DOM. Want to get the entire html as a string? Replace the that line with ``` return $('html').html(); ```. By the way, the ``` .get ``` call is to return a javascript array from *jQuery*'s internal representation;

+ Now, you've extracted what you wanted, i.e. the links of this page, it's time to process it, or just show it to the world. To make control flow less of a mess, *scraperjs* operates based on promises. You set a chain of promises if there are no errors the execution follows the chain that you set.

### Type of content

> The web is big and evolving. Simplicity paves the way to complexity, only to be reduced to simplicity.

In the web scraping lingo there are two big groups of webpages, the ones that render server-side and the ones that render client-side, i.e. in your browser.

Pages that render server-side are considered static, as it's internal structure doesn't change with time or events. An example of this is *Github Pages* or even *Github* in general. In the first case the pages are served as-is without information added to them, one could cache this pages around and serve them from a CDN, and that's [why *Wikipedia* is so fast and efficient](http://highscalability.com/wikimedia-architecture) at delivering their content. When some personalization is needed a server needs to put the information in the template and serve the personalized content, like *Github* or *Wikipedia* do when you're logged in.

In the other side of the table there are pages that render on the client which are considered dynamic, as its structure changes in the client, with time and through events. This kind of webpages is becoming widely used and relevant. An example of it is a page built with *Angular*, *Backbone* or *React*. It's more difficult to get information of this webpages since we need to render them ourselves and only then extract the information.

Of course this two ways of rendering a webpage are not exclusive and most of websites have a dynamic component somewhere. [*IMDB*](https://imdb.com) is mainly a static page, the crude data is static, but there's dynamic content when they show you either the *Add To Watchlist* or the *It is in the Watchlist* button, that fetches data through an AJAX call.

To accommodate to this differences *scraperjs* provides a **static** and a **dynamic** scraper. Remember that while a dynamic scraper will *almost* always cover all cases, it will be more taxing on your machine than the static scraper.

The static scraper organizes the html code so that it can be queried easily, while the dynamic scraper will render the page and execute all the code that the page would execute in your browser.
So, imagine that you want to get a wikipedia article you'd only need to use a static scraper. But if you'd like to get something off of a page that uses something like *Angular*, or that it's hidden behind a variable, you'll need to use a dynamic scraper. A page that takes three seconds to load will take the dynamic scraper more than three seconds (loading + scraping) while the static scraper will only need the html code to start scraping.

In *scraperjs* the usage difference between the two types of scrapers lies on the scope that the callback of the scrape function has access to. In the dynamic scraper you can't access functions in you code as you would normally, but you can use browser specific functions and objects without declaring them (like ``` document.querySelectorAll('a') ```).

``` js
var scraperjs = require('scraperjs');

scraperjs.DynamicScraper.create('{{ site.url }}{{ site.baseurl }}{{ page.url }}')
  .scrape(function($) {
    return $('a').map(function() {
      return $(this).attr('href');
    }).get();
  })
  .then(function(links) {
    console.log(links);
  });

```

This example is the same as the one above, but instead of a static scraper, we use a dynamic scraper. They will produce different results since I'm adding the link to my email address with javascript.

### Routes

*Scraperjs* also provides a new and modular way to scrape a variety of webpages, a router.
The routes are specified, usually through a url pattern. If the url provided to the router and the pattern of the route match, then that route will be executed.
This becomes really useful when you want to support efficiently the scraping of multiple websites.

A route is created with the ``` on ``` promise. Next you spefiffy the scraper for that route by either calling the promises ```createStatic```, ```createDynamic``` or ```use```, which will receive a scraper instantiated elsewhere.

Below there is a router with three different routes that use the same scraper function.

```js
var urlLib = require('url');
var scraperjs = require('scraperjs');

var router = new scraperjs.Router();
var scrapeFn = function($) {
  return $('a').map(function() {
    return $(this).text();
  }).get();
};
var toConsole = function(links) {
  console.log(links);
};

var scraper = scraperjs.StaticScraper
  .create()
  .scrape(scrapeFn)
  .then(toConsole);

router
  .on('https?://google.com')
  .createStatic()
  .scrape(scrapeFn)
  .then(function(links) {
    console.log(links);
  });

router
  .on('http://news.ycombinator.com/*')
  .createStatic()
  .scrape(scrapeFn)
  .then(function(links) {
    console.log(links);
  });

router
  .on(function(url) {
    return urlLib.parse(url).host.split('.').slice(-2).join('.') == 'wikipedia.com';
  })
  .use(scraper)

router
  .otherwise(function() {
    console.log('Not google, hacker news nor wikipedia.com');
  });

router.route('{{ site.url }}{{ site.baseurl }}{{ page.url }}', function(found, result) {
  if( found ) {
    console.log('A route was found');
    console.log(result);
  } else {
    console.log('A route was not found');
  }
});

```

The first two routes are based on url matching, while the third is using a function.
The third route also applies a router instantiated, but not fired, elsewhere.
Note that while routes to google are using a static scraper, routes to *Hacker News* are using a static scraper, as it doesn't have any dynamic content, whereas *Google* might have.
Finally, we give the router a url to be routed through our router. There is no route that matches the url of this page, so the (optional) ``` otherwise ``` route is fired.

### Waiting on the world to change

Now, we want to create a router, with just one route, to scrape a *Wikipedia*'s article name from one languange to another one. This way we can build a simple term translator. We are goining to use a fake *MongoDB* function to store our findings, using the ``` async ``` promise we can have a finer control of the promise chain.

```js
var scraperjs = require('scraperjs');

var router = new scraperjs.Router();

router
  .on('https?://:lang.wikipedia.org/wiki/:article')
  .createStatic()
  .scrape(function($) {
    return {
      links: $("#p-lang a[lang]").map(function() {
        return $(this).attr('title').replace(/ – .+$/g, "");
      }).get(),
      term: $(".firstHeading").text()
    };
  })
  .async(function(result, done, utils) {
    var obj = {
      term: result.term,
      links: result.links,
      lang: utils.params.lang
    };

    // Puts obj into mongodb
    storeInMongo(obj, function(err) {
      done(err, obj);
    });
  })
  .catch(function(err) {
    console.error(err);
  });

router.route("https://en.wikipedia.org/wiki/Web_scraping", function(found, result) {
  console.log(result);
});

```


### Command line

What if you wanted to use *scraperjs* with another language and you don't care for routes and specially for *node* programming?
There's an *experimental* command line utility that you can use to call easily *scraperjs* from the command line, or even other program. It's very simple in functionality,

``` scraperjs {{ site.url }}{{ site.baseurl }}{{ page.url }} --attr href --selector a --static ```

*You'll need to install scraperjs globally with ``` npm install -g scraperjs ```*

## Best practices

### Separate the scraping phase from the processing phase the most you can

The scrape function should always be as leaner as possible, and I'm not just talking about splitting it into different functions and just calling one. You should always return data that can be serialized easily.
This way you'll follow the principle of separation of responsibilities, making your code more readable and easy to understand. As a plus you'll be able to easily swap libraries around, and in case you're using *scraperjs* you'll be able to swap between static and dynamic scrapers in a blink of an eye, without breaking anything.
**Scraping is a map-reduce job. Scraping is mapping, reduce comes later.**

### Beware of the never ending recursion

When you are creating scrapers for generic webpages, for instance for web crawling, do the work in small chunks, that's will be easier to debug and, probably, to develop.

Recursion in most cases does not scale!

### API first

Imagine using scraping to get data of Twitter. It's possible and I get it, web scraping is shinny and gives you the power of gods, but it's only a viable option if you can't access the data through an API or [publicly available dataset](https://github.com/caesar0301/awesome-public-datasets).

Before creating your own scraper to map a website, search around. These days a lot of websites provide APIs or other ways to get their data. For instance, IMDB allows you to [download their database](http://www.imdb.com/interfaces) of movies and actors and use it for non-commercial purposes.

### Don't go to places unnecessarily

Do you really need to visit the same page milliseconds apart? Maybe you just need to visit the page once every minute, or every hour, or even every month.
The best way to filter out pages that you've visited is by using a [bloom filter](https://en.wikipedia.org/wiki/Bloom_filter).
Is there anything that as changed? If not, you are better of not visiting.

### Be a good lad

When you're experimenting for the first time with web scraping you probably got some *404* status code. *404* means that you're being a bit too annoying. There's little to no reason to make that many requests, unless of course, you are doing a *DDoS* attack and that's not a good thing to do.
So, **be careful**.

### Be a great lad

Besides the hit rate to a website you should provide some sort of information on yourself.
Setting a proper *user agent* header is a good start, specially if your purpose is scientific or just recreational.

Let other people know who you are and why you are doing what you are doing. Providing a contact address is also a big plus.

```js
var scraperjs = require('scraperjs');

scraperjs
  .StaticScraper.create()
  .get({
    url: '{{ site.url }}{{ site.baseurl }}{{ page.url }}',
    header: {
      'User-Agent': 'Collecting data to project XXX;mail@gmail.com'
    }
  })
  .scrape(function($) {
    return $('a').map(function() {
      return $(this).attr('href');
    }).get();
  });

```

#### Blocked

Some websites block non-standard user agents or lack thereof.
In fact, if you try to scrape a *Google* search, you'll see that even the dynamic scraper can't do you good. That's because *Google* is detecting that the page is not being rendered to a real user, but to a machine. There are ways to circunvey this. By cleverly setting cookies, faking a browser request or, when using the dynamic scraper (that uses [PhantomJS](http://phantomjs.org/) behind the scenes), by a [clever manipulation of
variables](http://engineering.shapesecurity.com/2015/01/detecting-phantomjs-based-visitors.html).

### Free, but not free as in beer

The web is open to ~~everyone~~ most people, as well as most of its content. And though the content is free, it is not free as in beer. Most webpages content are copyrighted, or have some sort of license, again beware of it and use your own best judgment.

