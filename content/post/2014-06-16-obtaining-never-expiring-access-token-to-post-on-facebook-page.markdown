+++
author = "Sergey Nuzhdin"
categories = ["Python", "coding"]
date = "2014-06-16"
title = "Obtaining never-expiring access_token to post on Facebook page"
type = "post"
aliases = [
    "/blog/2014/06/16/obtaining-never-expiring-access-token-to-post-on-facebook-page/"
]
+++

For one of my projects I needed to post messages to Facebook Fan Page.
After initial search I found out that Facebook has never-expiring token specially for this.
It looks like trivial task and I was really surprised when I didn't find any one-page step-by-step manual to get it.
I decided to write all steps from the beginning, not only part about gettting never-expiring token.
<!--more-->

## Proper permissions
It's obvious that to post on Page you need special set of permissions - _publish_actions_, _manage_pages_.</br>
You do not need to set these permissions to the whole application in settings, because you need it only for your user (user from whom messages will be posted). 
Facebook provides tool for generating access_tokens:

    https://developers.facebook.com/tools/explorer/

Here you should select your application and click `Get Access Token`. In popup select at least permissions mentioned above and click `Get Access Token`.
Now you have access_token able to post to Page. But it is valid only for 60 minutes. Not good.

## Long-lived access_token
Getting long-lived access_token is the easiest part. You just need to send your current active `short-term access_token`, `client_id` and `client_secret` to the url:

    https://graph.facebook.com/oauth/access_token?
        client_id=my_app_id&
        client_secret=my_app_secret&
        grant_type=fb_exchange_token&
        fb_exchange_token=access_token

Where `client_id` and `client_secret` are in your application settings.

And you will get response with new access_token and its expiration date in number of seconds.
Now you have good access_token valid for 60 days which you can use everywhere. but you need to extend it from time to time.

## Eternal token
To get never-expiring token you need to use your long-live token.
All you need is to enter it on: 
    https://developers.facebook.com/tools/explorer

And send request to `/v2.0/me/accounts`. You should see JSON with all your pages and some basic information about it: ID, Category, Name etc. And among them you will see `access_token` for each page. This access_tokens are what we are searching for. They are never-ending. 

{{< figure src="/img/2014/06/token.png" title="Access_token" >}}

## Get info about token
Facebook has another helpful endpoint to get information about any token, e.g. Expiration-date, app_id, user_id and scopes associated with it.

    https://developers.facebook.com/tools/debug/accesstoken

If you submit our final token here you will see `Expires:Never` in response.

## Python way
Here is few lines to show how to get it with Python.



    # Facebook SDK
    import facebook

    # Init graphAPI with short-lived token
    graph = facebook.GraphAPI(short_token)

    # Exchange short-lived-token to long-lived
    long_token = graph.extend_access_token(client_id, client_secret)
  
    # Init graphAPI with long-lived token
    graph = facebook.GraphAPI(long_token['access_token'])

    # Request all pages for user
    pages = graph.get_object('me/accounts')

    for page in pages['data']:
        print page


For each page you will see JSON like this. 

``` json 
{
   'access_token': never-expiring-page-token,
   'category': 'Product/service',
   'id': PAGE_ID,
   'name': PAGE_NAME,
   'perms': [
       'ADMINISTER', 
       'EDIT_PROFILE',
       'CREATE_CONTENT',
       'MODERATE_CONTENT',
       'CREATE_ADS',
       'BASIC_ADMIN'
    ]
}
```

Now, when I know how to do it, I see that most, if not all, of this could be found on facebook documentation pages.
But, let it be here.

