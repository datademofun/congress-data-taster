
# Finding who is running for federal office (and who is donating to them)


Bulk data can be downloaded from their serviceable if somewhat archaic web+FTP site:

[Detailed Files About Candidates, Parties and Other Committees](http://www.fec.gov/finance/disclosure/ftpdet.shtml)

To get a list of all candidates in this current cycle, look for the listing of __Candidate Master File__. The 2016 version links to a file that looks like this:

      ftp://ftp.fec.gov/FEC/2016/cn16.zip


An unzipped version of this file can be found at [data/fec/fetched/cn16.txt](data/fec/fetched/cn16.txt), but it basically looks like this:

|           |                           |     |      |    |   |    |   |   |           |                     |   |              |    |       |
|-----------|---------------------------|-----|------|----|---|----|---|---|-----------|---------------------|---|--------------|----|-------|
| H0AK00097 | COX, JOHN R.              | REP | 2014 | AK | H | 00 | C | N | C00525261 | P.O. BOX 1092       |   | ANCHOR POINT | AK | 99556 |
| H0AL02087 | ROBY, MARTHA              | REP | 2016 | AL | H | 02 | I | C | C00462143 | PO BOX 195          |   | MONTGOMERY   | AL | 36101 |
| H0AL02095 | JOHN, ROBERT E JR         | IND | 2016 | AL | H | 02 | C | N |           | 1465 W OVERBROOK RD |   | MILLBROOK    | AL | 36054 |
| H0AL05049 | CRAMER, ROBERT E "BUD" JR | DEM | 2008 | AL | H | 05 | C | P | C00239038 | PO BOX 2621         |   | HUNTSVILLE   | AL | 35804 |


What do any of those fields mean? The FEC doesn't include them in the actual data file. You have to go to the __data dictionary__:

[http://www.fec.gov/finance/disclosure/metadata/DataDictionaryCandidateMaster.shtml](http://www.fec.gov/finance/disclosure/metadata/DataDictionaryCandidateMaster.shtml)

Thankfully, the data dictionary has a link to a one-line CSV file that contains the headers:

[http://www.fec.gov/finance/disclosure/metadata/cn_header_file.csv](http://www.fec.gov/finance/disclosure/metadata/cn_header_file.csv)

It looks like as you imagine:

~~~
CAND_ID,CAND_NAME,CAND_PTY_AFFILIATION,CAND_ELECTION_YR,CAND_OFFICE_ST,CAND_OFFICE,CAND_OFFICE_DISTRICT,CAND_ICI,CAND_STATUS,CAND_PCC,CAND_ST1,CAND_ST2,CAND_CITY,CAND_ST,CAND_ZIP
~~~


## How to combine delimited files?

This is a problem that stumps people who don't realize how low-level programming can go.

The zipped data file is __pipe-delimited__. The headers file is __comma-delimited__...how are they supposed to mix?

If nothing else, you could do a find-and-replace on the headers file and change commas to pipes. Then copy and paste that line onto the top of the data file. That definitely works, but it's so cumbersome that you'll find yourself losing interest in campaign finance at an even faster pace. 

The concept is correct, we just need to implement it in Python.


## Download the headers programmatically

First step is just to replace the first _manual_ step: clicking and downloading a file. Let's use Requests:

~~~py
import requests
headers_url = 'http://www.fec.gov/finance/disclosure/metadata/cn_header_file.csv'
resp = requests.get(headers_url)
txt = resp.text
~~~

At this point, the variable `txt` just holds a string. We need to _deserialize_ it into a _list of strings_. This actually would work:

~~~py
headers = txt.split(',')
~~~

...but, that's a little _too_ low-level. Not all comma-delimited fields are that easy to separate. So just bring in the __csv__ module and let it do what it does:


~~~py
import csv
lines = txt.splitlines()
headers = list(csv.reader(lines))[0]
~~~

The code above may seem a little confusing to you. You know that the [header file](http://www.fec.gov/finance/disclosure/metadata/cn_header_file.csv) has just one line...so why call `splitlines()`? 

To get `csv.reader()` to parse the raw text correctly, we can't just give it a string of text; we have to give it a __list of strings__, even if that list is just one line long. Try doing `list(csv.reader(txt))` to see what goes wrong.



## Download the data file via FTP

OK, now that we have `headers`, we just have to attach it to the main candidates data file. We'll get to the how of that later. First, we need to programmatically download and unzip the file.

Because the data file is hosted on a FTP server, Requests __will not work__. We have to use Python's slightly cumbersome -- but built-in -- [urllib.request](https://docs.python.org/3/library/urllib.request.html#module-urllib.request) library. I like the __urlretrieve__ method.

Here's one way to use it:

~~~py
from urllib.request import urlretrieve
from shutil import unpack_archive
from os.path import join
from os import makedirs
DATA_DIR = join('tmp', 'cn16')
data_url = 'ftp://ftp.fec.gov/FEC/2016/cn16.zip'
zip_fname = join(DATA_DIR, 'cn16.zip')
urlretrieve(data_url, zip_fname)
# now unzip it
unpack_archive(zip_fname, extract_dir=DATA_DIR)
~~~

At the end of that sequence, you'll see that a zipfile exists wherever you pointed `unpack_archive` to do its work (via the `extract_dir` directory). The unzipped filename is predictable: it's just `cn.txt` -- although that is extremely annoying if you ever try to download more than one cycle's worth of data...

But let's just deal with putting together the data file. `headers` should already be a list. And when we open the `cn.txt` file, we can pass it to `csv.reader`...but what about the pipes? We pass in an argument to `csv.reader` to tell it to parse the file with pipes instead of commas:

~~~py
fname = join(DATA_DIR, 'cn.txt')
with open(fname) as rf:
    datarows = csv.reader(rf, delimiter='|')
~~~


## All together

Let's make sure we have the downloading steps correct. Here's all the code in one place:

~~~py
import csv
import requests
from urllib.request import urlretrieve
from shutil import unpack_archive
from os.path import join
from os import makedirs

DATA_DIR = join('tmp', 'cn16')
makedirs(DATA_DIR, exist_ok=True)
headers_url = 'http://www.fec.gov/finance/disclosure/metadata/cn_header_file.csv'
print("Downloading the headers", headers_url)
resp = requests.get(headers_url)
lines = resp.text.splitlines()
headers = list(csv.reader(lines))
# headers is a list of lists, we want the first list:
headers = headers[0]

## download the zip
data_url = 'ftp://ftp.fec.gov/FEC/2016/cn16.zip'
print("Downloading the zip file", data_url)
zip_fname = join(DATA_DIR, 'cn16.zip')
urlretrieve(data_url, zip_fname)
# now unzip it
unpack_archive(zip_fname, extract_dir=DATA_DIR)
fname = join(DATA_DIR, 'cn.txt')
with open(fname) as rf:
    datarows = list(csv.reader(rf, delimiter='|'))
~~~


How do we combine `headers` with `datarows`? If you're familiar with __zip__...well, this is how I would do it to get a list of dicts:

~~~py
mydata = []
for row in datarows:
    d = dict(zip(headers, row))
    mydata.append(d)
~~~

~~~py
mydata[5]
~~~

Output:

~~~py
{'CAND_CITY': 'KIMBERLY',
 'CAND_ELECTION_YR': '2010',
 'CAND_ICI': 'C',
 'CAND_ID': 'H0AL06088',
 'CAND_NAME': 'COOKE, STANLEY KYLE',
 'CAND_OFFICE': 'H',
 'CAND_OFFICE_DISTRICT': '06',
 'CAND_OFFICE_ST': 'AL',
 'CAND_PCC': 'C00464222',
 'CAND_PTY_AFFILIATION': 'REP',
 'CAND_ST': 'AL',
 'CAND_ST1': '723 CHERRY BROOK ROAD',
 'CAND_ST2': '',
 'CAND_STATUS': 'N',
 'CAND_ZIP': '35091'}
~~~


However, you'll probably find more utility of saving the combined file...write one script to download and wrangle the data into a form that another script can just pick up:


~~~py
dest_fname = 'cn16-with-headers.csv'
with open(dest_fname, 'w') as wf:
    csvfile = csv.writer(wf)
    csvfile.writerow(headers)
    csvfile.writerows(datarows)
~~~



# See the pattern

This is a _ton_ of boilerplate to just download a file and combine it with its headers. But if you have ambitions to do something that _scales_ -- such as downloading _all_ of the candidate data since 1980...or seeing that all of the other [data files on the FEC's site can be collected with nearly the same code](http://www.fec.gov/finance/disclosure/ftpdet.shtml)...then the half hour you may have spent writing code to download a couple of files will make short work of what would normally be a near-impossible manual-data gathering task.

Consider the file pattern for the candidate file for 2016:

      ftp://ftp.fec.gov/FEC/2016/cn16.zip

The 2014 file looks like this:

      ftp://ftp.fec.gov/FEC/2014/cn14.zip


So how would we generate all of the URLs for every file since 1980? Just a range and for-loop:

~~~py
urlbase = 'ftp://ftp.fec.gov/FEC/{year}/cn{yr}.zip'
# we go to 2017 to ensure that 2016 is part of the range
for year in range(1980, 2017, 2): 
    url = urlbase.format(year=year, yr=str(year)[-2:])
    print(url)
~~~

And if you look at the rest of the [data catalog](http://www.fec.gov/finance/disclosure/ftpdet.shtml), you'll see that every other file follows this simple pattern; to get the 2016 individual contributions data (i.e. every individual contribution over $200), you download this file:


      ftp://ftp.fec.gov/FEC/2016/indiv16.zip


Here's a loop that will generate the URLs for every cycle's worth of candidate, committee, individual contributions, and operating expenditures data:

~~~py
urlbase = 'ftp://ftp.fec.gov/FEC/{year}/{stub}{yr}.zip'
# we go to 2017 to ensure that 2016 is part of the range
for stub in ['cm', 'cn', 'indiv', 'oppexp']:
    for year in range(1980, 2017, 2): 
        url = urlbase.format(year=year, yr=str(year)[-2:], stub=stub)
        print(url)
~~~


Now, instead of just printing each URL, you just need to write the code to download and save them...





