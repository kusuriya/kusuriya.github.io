---
layout: default
title: Bending DuckDuckGo to do your bidding.
Date: 2016-02-16 06:28:00 +/-0000
published: yes
tags: 
    - Tools
---

# Bending DuckDuckGo to serve your will.  

It turns out while thinking ahead I decided I will at some point need to provide people with a way to search my blog. 
Not wanting to write or host an indexer myself I thought maybe Ill turn to my favorite search engine of all time, DuckDuckGo.

<!--more-->

If you havent ever used DuckDuckGo I suggest highly giving it a shot, it is a very powerful search engine and it respects your
privacy to boot. But getting past that the flexibility and power that makes them great also makes them a great tool for
site operators, and they are smart enough to recognize this.

My first run at adding it I discovered [DDG's search box generator](https://duckduckgo.com/search_box) The downside was
it only generated one height and one look, which may work for some sites but not for mine. Doing a bit more digging I discovered
[DDG's URL Parameters](https://duckduckgo.com/parameters) which seemed like a great way to get this going but they had
options for everything except site, so that left me one choice, I had to manipulate the search before it was submitted to
DDG to append `site:blog.corrupted.io` to the query.

I know what all the javascript folks out there are screaming and mentioning how easy that is with javascript, and they are
right. For the time being a pure HTML solution evades me so I had to hack some stuff together with javascript. so lets look at the code
~~~ html
<div id="search">
    <script type="text/javascript">
        function SearchMySite ()
        {
            SearchEngine = 'https://www.duckduckgo.com/?q='
            Site = 'blog.corrupted.io'
            url = SearchEngine+document.getElementById('SearchQuery').value+'+site:'+Site;
            window.location.href = url;
        }
    </script>
    <input type="search" id="SearchQuery" placeholder="DDG search this blog" onkeydown="if(event.which == 13){SearchMySite();}" style="border:1px solid;padding: 1px;height: 25px;width: 150px;margin-top: 2px;">
    <a onclick="SearchMySite();" href="#" style="border:0"><img src="/img/search.png" alt="Search" style="border:0;height:25px;align: bottom;vertical-align: bottom;padding: 0px;margin-left: -5px;"></a>
</div>
~~~
As we can see really nothing flashy showy or elegant here, just some brute force javascript that hacks together a URL with query and tells the browser to go there.
as you can see simple and gets the job done, if I threw a few of the parameter's from DDGs site in there I could also make DDG look like my blog, but that was more 
than I really felt like doing.