# Using Fragments to Make Marketplace Snappy, But Keep it Webby

When web developers build webapps, they frequently find the default behaviors of traditonal web sites conflicting with things that we frequently associate with those of Apps. One of these is the concept of "deep linking". One of the foundations of the web is the concept of the URL. Those of you stroking your mighty (and possibly figurative) beards of internet knowledge know that stands for Uniform Resource Locator. In the ideal form of the web, each "thing" on a website has a URL. Visiting that URL returns the content associated with that particular chunk of data. This is, while seemingly obvious these days, a pretty nifty idea.

In the early days of building the Firefox Marketplace, we knew that we wanted to use AJAX to keep browsing around the site snappy and as app-like as possible. We also decided, importantly, that the site have useful URLs and that each URL would result in a unique document.

In order to keep the URL bar up-to-date as the user navigates around our site, we take advantage of the [HTML5 history manipulation APIs](https://developer.mozilla.org/en-US/docs/DOM/Manipulating_the_browser_history#Adding_and_modifying_history_entries). HTML5 introduces the history-twiddling one-two punch of `history.pushState` and its partner the `popstate` event. `pushState` ensures that even though every page a user loads after the first is fetched asynchronously, the URL in the address bar reflects that of the currently viewed content. If the user bookmarks a URL, re-visiting it will bring them to the same content. If the user presses the back button, instead of leaving the current page, our JavaScript catches a `popstate` event, and we update the displayed content appropriately.

How do we render the same content asynchronously and synchronously? When we request a URL from our server, we indicate whether the request is a normal browser navigation or an AJAX request. While the actual backend implementation is (arguably) a little bit more clever, the essense of the server-side logic works like this:

    {% if not ajax %}
      <!doctype html>
      <html>
        <head></head>
        <body>
          <header></header>
          <div id="container">
    {% endif %}
            <section id="content" <!-- metadata about the page goes here --> >
              <h1>Potch's Awesome App!</h1>
              <!-- more content goes here -->
            </section>
    {% if not ajax %}
          </div>
          <footer></footer>
        </body>
      </html>      
    {% endif %}

When an AJAX request is made, we only return the tasty content center!

On the client side, we use a module called fragments.js to manage the process of turning clicked links into AJAX requests and handling the responses. Here's how it works:

* Listen for any clicked link on the body of our site (using [event delegation](http://davidwalsh.name/event-delegate), please!)
* Evaluate whether the link's `href` is an internal link.
* If it is an internal link, `stopPropagation` and `preventDefault`. Otherwise, let the browser handle it (bail out fragments code).
* Request the new page, and notify the user of a pending page request.
* If there were errors (loss of connection, 404/500), show an error.
* If the request was successful, take the HTML fragment (`#content` in our above example) and replace the contents of `#container`.
    * We also cache this response for the length of the page session.
* We use data-attributes in the HTML fragment to pass metadata about the current page, for instance the title in the `<header>`. Extract that data and apply it to the page.
* Use `pushState` to update the current URL in the address bar.

The [actual implementation of fragments.js](https://github.com/mozilla/zamboni/blob/master/media/js/mkt/fragments.js) has a bunch more logic to detect error cases, handle `popstate`, hijack form submission, and paper over browser bugs, but the essense is as above.

What are the benefits to this approach? In my opinion the best thing is that your site still serves HTML! The server generates the primary document markup as opposed to having to use a client-side template library (or worse, keep client and server-side templates in sync). Search engines can crawl your site and index the HTML documents they know best, while users get a snappy HTML5 app experience. If a users browser doesn't support `pushState`, Marketplace degrades gracefully. If a user agent doesn't support HTML5 history manipulation, we don't capture clicked links and they can browse the site using synchronous page-views. Additionally, this kind of solution is a great first migration for an existing website that currently isn't a single page app. By identifying which parts of your content are fixed between page views and wrapping the content fragment, you can port a complex app to use fragments without re-writing your server.

Are there downsides? Well, yes. If your pages have lots of data, transmitting JSON is more compact than HTML. In fact, if your app is mostly about data manipulation and CRUD, this approach won't net you many benefits other than speeding up subsequent page-loads.

In the coming weeks, we hope to refactor fragments into a standalone client library. If you want to get started right away and don't mind a healthy dose of jQuery, I can highly recommend the [jquery-pjax library](https://github.com/defunkt/jquery-pjax/).

I hope I've shown that apps don't have to have one URL or rely on the dreaded `#!` to allow users to access deep-linked content, and that you'll consider a similar approach for your projects!
