Web Scraping with RVest
================
Wouter van Atteveldt & Kasper Welbers
2020-10

`rvest` is a package that allows you to retrieve and parse HTML pages in
order to extract information from them. How easy it is to do so ranges
from the almost trivial for simple and well-designed web sites, to
almost impossible for web sites that actively try to prevent it.

As a general rule, you should make sure not to break any laws and
respect copyright and terms of service (although whether these terms are
binding without you explicitly agreeing probably depends on your
jurisdiction).

Finally, web scraping is finicky and your scraper will probably stop
functioning whenever the target web site is redesigned. For this reason,
if there is an API you can use (e.g. for the guardian) or there is
another way to get the data without having to scrape it that will
generally be preferable.

## Motivational example

The example below scrapes the world hapiness report from wikipedia and
shows the relationship between generosity and wealth:

``` r
library(rvest)
library(tidyverse)
url = "https://en.wikipedia.org/wiki/World_Happiness_Report"
t = read_html(url) %>% html_node(".wikitable") %>% html_table() 
ggplot(t) + geom_point(aes(x=`GDP per capita`, y=Generosity))
```

# Steps in web scraping

In general, the first step is to understand the structure of the web
site you want to parse, and determine how you can find the information
you are after. For this, the network view and inspect functions in most
modern browsers are invaluable tools.

After this, you can retrieve the desired web page and parse the
resulting HTML code. From this, you extract the required information
using the patterns identified in the first step. In many cases, this
information will contain or consist of links to other web pages, in
which case you will then retrieve and parse these pages as well.

# Parsing HTML using CSS selectors

In theory, you could perform these steps using basic R commands to
download files over the internet and parse the resulting HTML code as a
text, e.g. using regular expressions. Parsing HTML correctly with pure
regular expressions, however, is theoretically impossible and also a bad
idea, because the same HTML structure can be expressed in multiple ways
and HTML is very verbose. Thus, it is a much better idea to parse the
HTML and treat it as a tree structure. For example, consider the
structure of a simple HTML
fragment:

``` r
html = "<body><div id='doc1'><p class='lead'>The lead</p><p>The body</p></div></body>"
d = xml2::read_html(html)
html_structure(d)
```

As you can see, both paragraphs (`p`) are nested in a `div`, which is
nested in a `body`. The easiest way to identify an element in such a
tree is using CSS selectors. CSS (cascading style sheets) are designed
to apply styling to certain aspects of a document, for example to format
the lead paragraph in a different font. As part of such a style sheet,
the selectors are used to identify which elements to apply styling to.

In scraping, we can use the same selectors to identify which tags
contain the information we are interested in. Below is a table with some
of the more common css selectors, but if you google something like ‘css
selector cheat sheet’ you will find a lot of more exhaustive
lists.

| selector    | example       | meaning                                     |
| ----------- | ------------- | ------------------------------------------- |
| child       | div \> p      | any `p` directly nested in a div            |
| descendant  | div p         | any `p` below a div in the hierachy         |
| first child | p:first-child | a `p` that is the first child of its parent |
| class       | p.lead        | a `p` with `class='lead'`                   |
| class       | .lead         | any tag with `class='lead'`                 |
| id          | div\#doc1     | a `div` with `id='doc1'`                    |

These selectors can be combined, so e.g. `body > #doc1 p.lead`
identifies any `p` that has `class='lead'`, is a descendant of a tag
with `id='doc1'` which is a direct child of `body`.

Using rvest, you can easily extract information using such a selector:

``` r
lead = html_node(d, "div .lead")
lead
```

``` r
html_text(lead)
```

As an alternative to CSS selectors, you can also use xpath. Xpath
patterns are more difficult to read and write, but are in the end more
powerful than CSS selectors, meaning there are some patterns that you
can express in xpath but cannot express in css. For example, in CSS you
cannot use arbitrary attributes to select a nore (such as
`data="report"`), which is possible in xpath. For example, the same
selection as above can be expressed in xpath as follows:

``` r
lead = html_node(d, xpath="//div//*[@class='lead']")
html_text(lead)
```

A full explanation of xpath is beyond the scope of this tutorial, but
you can easily find such an explanation on the internet. In general, I
would recommend using CSS whenever possible due to the increased
readability, and only use xpath if what you’re looking for cannot be
found using CSS.

# Scraping with rvest

Now that you have a basic understanding of web scraping and css, let’s
review how to scrape using RVest step by step.

## Retrieving and parsing a web page

This is the simple part `read_html(url)` will automatically download,
read, and parse the web site at url:

``` r
url = "https://en.wikipedia.org/wiki/World_Happiness_Report"
html_doc = read_html(url) 
```

## Identifting the right node(s)

Using the CSS selectors shown above, you can use `html_node` or
`html_nodes` to select nodes from the HTML document. Note that the
singular form only returns the first result, while the plural form
returns all results on the page:

``` r
html_doc %>% html_node("h2")
```

``` r
html_doc %>% html_nodes("h2")
```

The first call returns only the first `h2` header (the table of
contents), while the second call returns all 11 headers.

## Extracting information from the node

There are four functions that extract information from the selected
nodes. First, `html_name` extracts just the tag name of the node, e.g.
`p` or `h2`:

``` r
html_doc %>% html_node("h2") %>% html_name()
```

Next, `html_attr` extracts HTML attributes, e.g. the `href=` of a
hyperlink. For example, this selects all hyperlinks from the wikipedia
page:

``` r
urls = html_doc %>% html_nodes("a") %>% html_attr('href') %>%  na.omit()
head(urls)
```

Third, you extract the text contained in an element with `html_text`:

``` r
html_doc %>% html_node("h1") %>% html_text()
```

Finally, the `html_table` function converts an HTML table into an R data
frame. Note, though, that HTML tables are often not as neatly
rectangular as a data frame, so unless the table is well formatted the
result might not be as nice as you would hope.

``` r
t= html_doc %>% html_node(".wikitable") %>% html_table()
head(t)
```

## Following links

Often, you will be scraping multiple pages where the first page simply
is a list linking to other pages of interest. In that case, you will
need to extract those links from the first page, and then scrape all the
remaining pages. This can be done using a *for loop* or using a *mapping
function*.

Let’s consider the countries in the world hapiness list, and suppose we
would like to extract the official name of each country from it’s own
wiki page (which is linked in the table).

First, we need to extract the urls from the table. To make sure we only
use the country names, I am specifying that we only want the `a` that is
the direct child of the `td` that is the first sibling of the first
child of the row, i.e. the second
column:

``` r
urls = html_doc %>% html_node(".wikitable") %>% html_nodes("td:first-child + td > a") %>% html_attr("href")
urls = str_c("https://en.wikipedia.org/", urls)
head(urls)
```

Now, let’s extract the name from a single country:

``` r
read_html(urls[1]) %>% html_node(".country-name") %>% html_text()
```

That worked\! But how do we generalize this to all countries? First, we
can use a *for loop*: (note that to speed things up, we only use the
first five results here)

``` r
results = list()
for (url in urls[1:5]) {
  message(url)
  name = read_html(url) %>% html_node(".country-name") %>% html_text()
  results[[url]] = tibble(name=name)
}
names = bind_rows(results, .id="url")
names
```

This might seem convoluted, but it is relatively idiomatic: create an
empty list, assign the result of each call to an element of the list,
and combine the results at the end. The benefit of this approach is that
is easy to control and you can e.g. modify it so it skips countries it
already scraped (e.g. by adding `if (url %in% names(results)) continue`
just after the for). The downside is that it is rather verbose,
relatively slow (but for scraping the speed of the R code is rarely the
problem compared to how long it takes to retrieve a web page), and
somehow un-R-like.

The more *functional* approach is to `map` the extraction to the
elements in the urls vector. For this, you define the extraction as a
function:

``` r
get_name = function(url) {
  read_html(url) %>% html_node(".country-name") %>%
    html_text() %>% as_tibble()
}
get_name(urls[1])
```

Now, you can map this function over all urls:

``` r
map_df(setNames(nm=urls[1:5]), get_name, .id = "url")
```

Note that in order to have the result be a tibble with urls and names
(so we can then e.g. join it to the hapiness table), we had to (1)
return a tibble rather than just the name, and (2) use the `setNames`
function to set the urls as the names on the input.

Finally, you can create an anonymous function using the `~f(.)`
notation, which essentially creates `function (x) f(x)`:

``` r
map_df(setNames(nm=urls[1:5]), 
       ~read_html(.) %>% html_node(".country-name") %>% html_text() %>% as_tibble(), 
       .id = "url") 
```

Whether that is more readable than explicitly defining the function
depends on the complexity of the function, but in this case I would
certainly argue for the explicit function definition, which is also a
lot easier to write and debug.

(as a challenge, try to extract the population from the infobox rather
than the full name. Hint: You might need to use xpath to identify the
right cell of the table)
