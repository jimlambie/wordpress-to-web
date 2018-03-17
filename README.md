# Migrating Wordpress to DADI Web

It is reasonably easy to get your website up and running using a combination of web services from the DADI suite, but what if you already have a lot of content stored in your existing Wordpress site? Today we'll show you how you can migrate your Wordpress posts and pages to [DADI Web](https://github.com/dadi/web), and in a future article we'll add API and Publish into the mix to give you end-to-end publishing.

## Exporting the Wordpress content

The first step is to export the content from WordPress. To do this, use the WordPress exporter under Tools > Export in the WordPress admin interface. Follow the instructions to download an XML file, for example `helloworld.wordpress.2018-03-15.xml`

## Convert Wordpress XML to Web-friendly Markdown

The next step is to convert the exported XML export to Markdown files.

There are plenty of alternative solutions available for converting XML to Markdown. One we particularly like is [exitwp](https://github.com/thomasf/exitwp), a project designed for migrating from one or more Wordpress blogs to the Jekyll blog engine.

To perform the conversion we're going to use a tool we quickly put together at DADI, called `dadiwp2md`. It doesn't handle all the XML tags possible in a Wordpress export, but it's useful as a demonstration.

> *Note:* if you're just following along and don't have a Wordpress XML export, download a sample one:
> `curl https://www.mhthemes.com/docs/wordpress-sample-content.xml > wordpress-sample-content.xml`

1. Install the tool globally: `npm i -g @dadi/wp2md`
2. Run the command `dadiwp2md`, specifying the input XML file and the path to the directory where you want to save the Markdown files: `dadiwp2md helloworld.wordpress.2018-03-15.xml ./posts`
3. You should now have a bunch of Markdown files in the `posts` directory

## Setting up DADI Web

If you don't already have a DADI Web installation, [install it using DADI CLI](https://forum.dadi.tech/topic/5/installing-web-using-dadi-cli). We're going to use Dust.js for the templates in this example, so make sure you install the [@dadi/web-dustjs](https://www.npmjs.com/package/@dadi/web-dustjs) template engine for Web.

```
$ dadi new web my-blog
```

### Add the content 

Create the directory `workspace/posts`. This is where we'll store the Markdown files we generated earlier, to enable Web to load them. Copy the Markdown files from the `posts` directory to `workspace/posts`.

### Loading the content

Create a new datasource to load the posts. It should use Web's Markdown provider and point to the directory containing the posts:

#### workspace/datasources/posts.json

```json
{
  "datasource": {
    "key": "posts",
    "name": "Blog posts",
    "source": {
      "type": "markdown",
      "path": "workspace/posts"
    },
    "count": 10,
    "sort": {},
    "fields": []
  }
}
```

### Displaying the posts

In this simple example, we'll display a list of post titles on the index page, and link each one to a "post" page where the content can be read in full.

First we'll create two files for the index page:

* `workspace/pages/index.json` - the page specification file
* `workspace/pages/index.dust` - the page template, using Dust.js

#### workspace/pages/index.json

```json
{
  "page": {
    "key": "index",
    "name": "index",
    "description": "My homepage."
  },
  "settings": {},
  "routes": [
    {
      "path": "/"
    }
  ],
  "datasources": [
    "posts"
  ]
}
```
You can see we've attached the posts datasource to the page using the `datasources` property.

#### workspace/pages/index.dust

```htmlmixed=
{#posts.results}
  <ul>
    {?attributes.title}
      <li>
        <a href="/post/{attributes.slug}">{attributes.title}</a>
      </li>
    {/attributes.title}
  </ul>
{/posts.results}
```

If you start Web now (`npm start` should do it) and browse to your Web's homepage, you should see a list of your Wordpress posts. 

### Viewing a single post

To allow us to view a post in full after clicking on it's link in the index page, we'll need to modify the `posts` datasource and add an additional page specification and template.

* `workspace/pages/post.json` - the page specification file
* `workspace/pages/post.dust` - the page template for a single post

#### workspace/pages/post.json

```json
{
  "page": {
    "key": "post",
    "name": "Blog post",
    "description": "A single blog post."
  },
  "settings": {},
  "routes": [
    {
      "path": "/post/:slug"
    }
  ],
  "datasources": [
    "posts"
  ]
}
```

#### workspace/pages/post.dust

```=
{#posts.results}
  <article>
    <header>
      <h2><a href="/post/{attributes.slug}">{attributes.title}</a></h2>
      <p>
        By <strong>{attributes.author}</strong> on <time datetime="{attributes.date}">attributes.date</time>
      </p>
    </header>

    {contentHtml|s}
  </article>
{/posts.results}
```

#### Modify the datasource

In the single post page we've specified the route as `/post/:slug`. The last part of this is a dynamic parameter that is extracted from the request URL. We can use it in the datasource to filter the posts collection. 

Add the following to the `posts.json` datasource:

```json
"requestParams": [
  {
    "param": "slug",
    "field": "_id"
  }
]
```

Now, if you browse to `/post/hello-world`, the `hello-world` is extracted from the URL and used to filter the posts collection, for example `filter={"_id": "hello-world"}`.


To check it works as intended, restart Web and click a link to a post. You should see that post in full.