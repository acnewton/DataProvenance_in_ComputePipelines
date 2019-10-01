---
title: "Data analysis with iRODS data"
teaching: 90
exercises: 0
questions:
- "what are necessary steps for data provenance in a compute pipeline?"
- "how can we use the python API in a compute pipeline?"
objectives:
- "Understand how to have data provenance in compute pipelines."
keypoints:
- "iRODS API and data provenance"
- "leveraging iRODS metadata for data provenance"
---

<!--<a href="http://binder.pangeo.io/v2/gh/jcrist/anacondacon-2019-tutorial/master">
  <img src="http://binder.pangeo.io/badge.svg" width="200px">
</a>
-->
<a href="https://mybinder.org/v2/gh/acnewton/IntroPythonAPIiRODS/master">
  <img src="https://mybinder.org/badge_logo.svg" width="400px">
</a>


## Connect to iRODS
Note this is a recap from the previous chapter.
To connect to iRODS we will need to authenticate via a username and password (for this course username='mara' and password='NdxJvJQujF7dq5TyBms2Xp'). Passing sensitive information over internet can be insecure. Therefore, it is best practice to do this by encoding the password first.
The module `getpass ` asks for passwords without printing the input on screen. With  encoding function we prevent that the variable contains the plain password. 

~~~
import getpass
from irods.session import iRODSSession

pw = getpass.getpass().encode('base64')
session = iRODSSession(host='<hostname>', port=1247, user='<username>', password=pw.decode('base64'),zone='<zoneName>')
~~~
{: .language-python}

Now we can create an iRODS session. The iRODS instance you will be connecting to is hosted at SURFsara HPC Cloud. For connecting to iRODS you will need the following information:

~~~
hostname='irodscourse.irodspoc-sara.surf-hosted.nl'
port=1247
username='mara'
zonename='tempZone'
~~~
{: .language-python}

with this information we can create a session object:

~~~
from irods.session import iRODSSession
session = iRODSSession(host=hostname, port=port, user=username, password=pw.decode('utf-16'), zone=zonename)
~~~
{: .language-python}

and test whether we have access:

~~~
coll = session.collections.get('/tempZone/home/mara')
print(coll.data_objects)
print(coll.subcollections)
~~~
{: .language-python}

~~~
iHome = coll.path
~~~
{: .language-python}

## Setup computation
We will now prepare the compute workflow as it should be executed on any worker node in a cluster. We still do that in interactive mode and on the user interface node (remember it behaves as any node in the cluster).

~~~
import os
from helperFunctions import *

print('Creating directories for analysis and results')
dataDir = os.environ['TMPDIR']+'/wordcountData' + '<your id>'
ensure_dir(dataDir)
print(dataDir)
resultsDir = os.environ['TMPDIR']+'/wordcountResults' + '<your id>'
ensure_dir(resultsDir)
print(resultsDir)
~~~
{: .language-python}


## Searching for the dataset
We need to perform a query to obtain the dataset we are interested in. For this use case, we are interested a simple analysis of word frequency in Lewis Carroll books.

~~~
from irods.models import Collection, DataObject, CollectionMeta, DataObjectMeta

print('Searching for files')
ATTR_NAME = 'author'
ATTR_VALUE = 'Lewis Carroll'

query = session.query(Collection.name, DataObject.name)
filteredQuery = query.filter(DataObjectMeta.name == ATTR_NAME).\
                          filter(DataObjectMeta.value == ATTR_VALUE)
print filteredQuery.all()
iPaths = iParseQuery(filteredQuery)
~~~
{: .language-python}

## Downloading files from iRODS to scratch
Remember from the first part of the tutorial, that for downloading data from iRODS you first have to read the data object into memory and after that write it to a file on your local filesystem. For convenience we provide a function wich does that for us for a list of data objects.

~~~
print('Downloading: ')
print()'\n'.join(iPaths))
iGetList(session, iPaths, dataDir)
~~~
{: .language-python}

## Execute compute workflow

With everything setup, we can finally do our analysis:

~~~
dataFiles = [dataDir+'/'+f for f in os.listdir(dataDir)]
resFile = wordcount(dataFiles,resultsDir)
~~~
{: .language-python}

## Upload results to iRODS
After the computation has finished. We can make the upload of the results into iRODS part of our jobscript.

~~~
#Check if results are actually created and ingested into iRODS
coll = session.collections.get('/aliceZone/home/irods-user1')
objNames = [obj.name for obj in coll.data_objects]
f = os.path.basename(resFile)
count = 0

while f in objNames:
    f = os.path.basename(resFile) + '_' +str(count)
    count = count + 1

#upload
print('Upload results to: ', coll.path + '/' + f)
session.data_objects.put(resFile, coll.path + '/' + f)
~~~
{: .language-python}


## Introduce some metadata for provenance
Now that the results are stored in iRODS, we want to link our new data to the computation and old data. 
Here we put some generic metadata that could link the new results with the original data. 
This is definitely not exhaustive and a much more detailed description would be more appropriate.

~~~
import datetime
obj = session.data_objects.get(coll.path + '/' + f)
for iPath in iPaths:
    obj.metadata.add('INPUTDAT', iPath)

obj.metadata.add('ISEARCH', ATTR_NAME + '==' + ATTR_VALUE)
obj.metadata.add('ISEARCHDATE', str(datetime.date.today()))
print('\n'.join([item.name +' \t'+ item.value for item in obj.metadata.items()]))
~~~
{: .language-python}


## Summarize
We have performed a simple calculation with data from iRODS. 
This data was downloaded to our current location as this typically gives the best performance. 
This staging procedure could also be done well before the computation and is also typically advised as long as the size of for instance the scratch space on the HPC cluster you are working in allows the size of the dataset. 
In principle, a more complex computation won't differ that much from what we have done here.



<!--
[The next episode]({{ page.root }}/04-AdvancedUseCase/) will in the future cover a more advanced compute pipeline.
-->

{% include links.md %}
