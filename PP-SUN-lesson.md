# ProPublica and Sunlight's Congress APIs

Homepages and documentation:

- ProPublica: https://propublica.github.io/congress-api-docs/#congress-api-documentation
- Sunlight: 
    homepage: https://sunlightlabs.github.io/congress/
    api-registration: https://sunlightlabs.github.io/congress/index.html#parameters/api-key


## How to work with an API key

Both ProPublica and Sunlight require that you use API Keys when making a request. For the purposes of this exercise, I'll assume you have a folder named `keys` and in it, two text files, `propublica.txt` and `sunlight.txt`.

How do you save a key? It's just a piece of text that they sent you via email:


        v0H99XNSy1jvzv2RDo4gazW5h4qaX74y3Pws6FtPY

And you know how to open a text file and read a string:

~~~py
fname = 'keys/sunlight.txt'
with open(fname) as f:
    key = f.read().strip()
~~~

Or, since it's such a small file, you can be lazy:

~~~py
key = open('keys/sunlight.txt').read().strip()
~~~


## Using Sunlight's API key to make requests

The first thing we need to do is get a list of legislators. Read the documentation here:

[https://sunlightlabs.github.io/congress/legislators.html](https://sunlightlabs.github.io/congress/legislators.html)

The base endpoint is this:

      https://congress.api.sunlightfoundation.com

The path for the __legislators__ data is simply:

      /legislators

This ends up being a very easy URL to get the data:

https://congress.api.sunlightfoundation.com/legislators

Unfortunately, without a key, you'll get this response:

~~~json
{
  error: "API key required, you can obtain one from http://services.sunlightlabs.com/accounts/register/",
  status: 403
}
~~~


Once you [have a key](https://sunlightlabs.github.io/congress/index.html#parameters/api-key), you can attach it to the URL as a query string:


https://congress.api.sunlightfoundation.com/legislators?apikey=YOURKEYHERE


The Python way, with Requests:

~~~py
import requests
import json
key = open('keys/sunlight.txt').read().strip()
API_ENDPOINT = 'https://congress.api.sunlightfoundation.com/legislators'
myparams={'apikey': key}
resp = requests.get(API_ENDPOINT, params=myparams)
data = json.loads(resp.text)
~~~
