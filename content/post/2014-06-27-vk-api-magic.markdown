+++
author = "Sergey Nuzhdin"
categories = ["coding"]
date = "2014-06-27"
title = "VK API magic"
type = "post"
aliases = [
    "/blog/2014/06/27/vk-api-magic/"
]
+++

There are lots of posts about magic behaviour of VK API. But yeasterday I faced another one. 
I have simple button on my site to share page on VK using `Wall.post` of OpenAPI, and yesterday I found out that it stopped working.
After debuging I found error in API response:

    Permission to perform this action is denied for non-standalone applications: you should request token using blank.html page.

The magical thing was that this error appears only on one type of pages, while on another everything was fine.

Nothing I've found in google helped me.
And the last idea was that is has something with subdomains. I saw this error only on subdomain. I added subdomains to application settings.
Nothing changed.

After comparing everything of two request/responses the only difference was in `#` in in url I was trying to share.
I tried to remove it and... it works.

I dont know why VK don't like hashtags in urls and why it returns such a strange response but the fact is - you cant share urls with `#`.


