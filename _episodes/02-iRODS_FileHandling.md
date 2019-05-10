---
title: "Introduction iRODS Python API"
teaching: 90
exercises: 0
questions:
- "How to do basic file handling with the iRODS Python API"
- "How to use the iRODS Python API in data provenance"
objectives:
- "Connect to iRODS via Python API"
- "Upload/Download data objects"
- "Create and run through collections"
- "Metadata handling of data objects and collections"
- "Querying data objects and collections based on metadata"
keypoints:
- "All iRODS objects or Python objects"
- "Querying is tricky"
---

This episodes describes how to interact with your data in iRODS via the Python API.

## Connect to iRODS

We can connect to our iRODS instance by passing our credentials to a session object.
For this we need to import the iRODS session library.
However, we also want to savely pass our password.
Therefore, we will also import the getpass library.
It would even be better if we could have TOTP connected to our iRODS account. 
However, this is not possible with iRODS yet.
After importing the correct libraries, we can safely store our password and send our credentials to iRODS.

~~~
import getpass
from irods.session import iRODSSession

pw = getpass.getpass().encode('base64')
session = iRODSSession(host='<hostname>', port=1247, user='<username>', password=pw.decode('base64'),zone='<zoneName>')
~~~
{: .language-python}

We can test whether our connection succeeded by trying to get our home folder:
~~~
coll = session.collections.get('/aliceZone/home/irods-user1')
print(coll.data_objects)
print(coll.subcollections)
~~~
{: .language-python}

So far no data is stored in your iRODS collection.
Let us upload some data.

We will need our homefolder more often as a reference point, so let us store the collection path:

~~~
iHome = coll.path
~~~
{: .language-python}


## Upload a data object

The preferred way to upload data to iRODS is a data object put. 
Now we create the logical path and upload the German version of Alice in wonderland to iRODS:

~~~
iPath = iHome + 'Alice-DE.txt'
session.data_objects.put('aliceInWonderland-DE.txt.utf-8',iPath)
~~~
{: .language-python}

The object carries some vital system information, otherwise it is empty.

~~~
obj = session.data_objects.get(iPath)
print "Name: ", obj.name
print "Owner: ", obj.owner_name
print "Size: ", obj.size
print "Checksum:", obj.checksum
print "Create: ", obj.create_time
print "Modify: ", obj.modify_time
print "Metadata: ", obj.metadata.items()
~~~
{: .language-python}

Less code to write to display the full object:
~~~
vars(obj)
~~~
{: .language-python}

You can also rename an iRODS data object or move it to a different collection:
~~~
session.data_objects.move(obj.path, iHome + '/Alice.txt')
print coll.data_objects
~~~
{: .language-python}

## Metadata handling
Working with metadata is not completely intuitive, you need a good understanding of python dictionaries and the iRODS python API classes *dataobject*, *collection*, *iRODSMetaData* and *iRODSMetaCollection*.

We start slowly with first creating some metadata for our data.
Currently, our data object does not carry any user-defined metadata:

~~~
iPath = iHome + '/Alice.txt'
obj = session.data_objects.get(iPath)
print obj.metadata.items()
~~~
{: .language-python}


Create a key, value, unit entry for our data object:
~~~
obj.metadata.add('SOURCE', 'python API training', 'version 1')
obj.metadata.add('TYPE', 'test file')
~~~
{: .language-python}

If you now print the metadata again, you will see a cryptic list:
~~~
print obj.metadata.items()
~~~
{: .language-python}

The list contains two metadata python objects.
To work with the metadata you need to iterate over them and extract the AVU triples:

~~~
[(item.name, item.value, item.units) for item in obj.metadata.items()]
~~~
{: .language-python}

Metadata can be used to search for your own data but also for data that someone shared with you. You do not need to know the exact iRODS logical path to retrieve the file, you can search for data wich is annotated accordingly. We will see that in the next section.

**Watch out:** If you do another `data_object.put` you will overwrite not only the bitstream but also all metadata. User-defined metadata will be set to empty.

## Download a data object
The current release of the API does not support a 'download' or 'get' function for iRODS objects. It is planned though for the next release.
We will now stream the data object. Data streaming can become handy when up and downloading large files.
You first open the file and read its contents into memory:


~~~
import os
buff = session.data_objects.open(obj.path, 'r').read()
print "Downloading to:", os.environ['HOME']+'/'+os.path.basename(obj.path)
with open(os.environ['HOME']+'/'+os.path.basename(obj.path), 'wb') as f:
    f.write(buff) 
~~~
{: .language-python}

## Streaming data
Streaming data is an alternative to upload large data to iRODS or to accumulate data in a data object over time. First you need to create an empty data object in iRODS beofre you can stream in the data.

~~~
content = 'My contents!'
obj = session.data_objects.create(iHome + '/stream.txt')
~~~
{: .language-python}

This will create a place holder for the data object with no further metadata:

~~~
print "Name: ", obj.name
print "Owner: ", obj.owner_name
print "Size: ", obj.size
print "Checksum:", obj.checksum
print "Create: ", obj.create_time
print "Modify: ", obj.modify_time
print "Metadata: ", obj.metadata.items()
~~~
{: .language-python}

~~~
vars(obj)
~~~
{: .language-python}

We can now stream in our data into that placeholder.

~~~
with obj.open('w') as obj_desc:
    obj_desc.write(content)
obj = session.data_objects.get(iHome + '/stream.txt')
~~~
{: .language-python}

Now we check the metadata again:

~~~
print "Name: ", obj.name
print "Owner: ", obj.owner_name
print "Size: ", obj.size
print "Checksum:", obj.checksum
print "Create: ", obj.create_time
print "Modify: ", obj.modify_time
print "Metadata: ", obj.metadata.items()
~~~
{: .language-python}


~~~
vars(obj)
~~~
{: .language-python}


## iRODS collections
You can organise your data in iRODS just like on a POSIX file system.

### Create a collection
~~~
session.collections.create(iHome + '/Books/Alice')
~~~
{: .language-python}

And test:

~~~
coll.path
coll.subcollections
~~~
{: .language-python}


### Move a collection
Just as data objects you can also move and rename collections with all their data objects and subcollections:

~~~
session.collections.move(iHome + '/Books', iHome + '/MyBooks')
coll = session.collections.get(iHome)
coll.subcollections
~~~
{: .language-python}


~~~
[vars(c) for c in coll.subcollections]
~~~
{: .language-python}

### Remove a collection

Remove a collection recursively with all data objects.

~~~
coll = session.collections.get(iHome + '/MyBooks')
coll.remove(recurse=True)
~~~
{: .language-python}


Do not be fooled, the python object 'coll' looks like as if the collection is still in iRODS. You need to refetch the collection (refresh).

~~~
coll = session.collections.get(iHome)
coll.subcollections
[vars(c) for c in coll.subcollections]
~~~
{: .language-python}


### Upload a collection

To upload a collection from the unix file system one has to iterate over the directory and create collections and data objects.
We will upload the directory 'aliceInWonderland'

~~~
import os
dPath = os.environ['HOME'] + '/aliceInWonderland'
walk = [dPath]
while len(walk) > 0:
    for srcDir, dirs, files in os.walk(walk.pop()):
        print srcDir, dirs, files
        walk.extend(dirs)
        iPath = iHome + srcDir.split(os.environ['HOME'])[1]
        print "CREATE", iPath
        newColl = session.collections.create(iPath)
        for fname in files:
            print "CREATE", newColl.path+'/'+fname
            session.data_objects.put(srcDir+'/'+fname, newColl.path+'/'+fname)
~~~
{: .language-python}

There is mixed tab and whitespace in the code. When copy&paste the code, it is not working,
The standard recommends 4 spaces.
https://www.python.org/dev/peps/pep-0008/#tabs-or-spaces


### Iterate over collection
Similar to we walked over a directory with sub directories and files in the unix file system we can walk over collections and subcollections in iRODS. Here we walk over the whole aliceInWonderland collection and list Collections and Data objects:


~~~
for srcColl, colls, objs in coll.walk():
    print 'C-', srcColl.path
    for o in objs:
        print o.name
~~~
{: .language-python}

## Sharing data

You can set ACLs on data objects and collections in iRODS.
To check the default ACLs do:

~~~
print session.permissions.get(coll)
print session.permissions.get(obj)

[vars(p) for p in session.permissions.get(coll)]
~~~
{: .language-python}

Here we share a collection with the iRODS group public. Every member of the group will have read rights.

~~~
from irods.access import iRODSAccess
acl = iRODSAccess('read', coll.path, 'public', session.zone)
session.permissions.set(acl)
print session.permissions.get(coll)
~~~
{: .language-python}

To withdraw certain ACLs do:

~~~
acl = iRODSAccess('null', coll.path, 'public', session.zone)
session.permissions.set(acl)
print session.permissions.get(coll)
~~~
{: .language-python}

One can also give 'write' access or set the 'own'ership.

Collections have a special ACL, the 'inherit' ACL. If 'inherit' is set, all subcollections and data objects will inherit their ACLs from their parent collection automatically.

## Searching for data in iRODS

We will now try to find all data in this iRODS instance we have access to and which carries the key *author* with value *Lewis Carroll*. And we need to assemble the iRODS logical path.

~~~
from irods.models import Collection, DataObject, CollectionMeta, DataObjectMeta
~~~
{: .language-python}

We need the collection name and data object name of the data objects. This command will give us all data objects we have access to:

~~~
query = session.query(Collection.name, DataObject.name)
~~~
{: .language-python}

Now we can filter the results for data objects which carry a user-defined metadata item with name 'author' and value 'Lewis Carroll'. To this end we have to chain two filters:

~~~
filteredQuery = query.filter(DataObjectMeta.name == 'author').\
    filter(DataObjectMeta.value == 'Lewis Carroll')
print filteredQuery.all()
~~~
{: .language-python}


Python prints the results neatly on the prompt, however to extract the information and parsing it to other functions is pretty complicated. Every entry you see in the output is not a string, but actually a python object with many functions. That gives you the advantage to link the output to the rows and comlumns in the sql database running in the background of iRODS. For normal user interaction, however, it needs some explanation and help.

### Parsing the iquest output
To work with the results of the query, we need to get them in an iterable format:

~~~
results = filteredQuery.get_results()
~~~
{: .language-python}

**Watch out**: *results* is a generator which you can only use once to iterate over.

We can now iterate over the results and build our iRODS paths (*COLL_NAME/DATA_NAME*) of the data files:

~~~
iPaths = []

for item in results:
    for k in item.keys():
        if k.icat_key == 'DATA_NAME':
            name = item[k]
        elif k.icat_key == 'COLL_NAME':
            coll = item[k]
        else:
            continue
    iPaths.append(coll+'/'+name)
print '\n'.join(iPaths)
~~~
{: .language-python}

How did we know which keys to use?
We asked in the query for *Collection.name* and *DataObject.name*.
Have look at these two objects:


~~~
print Collection.name.icat_key
print DataObject.name.icat_key
~~~
{: .language-python}

The *icat_key* is the keyword used in the database behind iRODS to store the information.

> ## Putting it all together
> 1. Search for all data in the iRODS instance by the author Lewis Carroll.
> 2. Create an own collection in your iRODS home collection.
> 3. Copy (`session.data_objects.copy()`) the data objects to the new collection without downloading and uploading them.
> 4. Verify the checksums.
> 5. Open the collection to your neighbour by giving read rights.





[The next episode]({{ page.root }}/03-dataProcessingWithRodsData/) describes these files.

{% include links.md %}
