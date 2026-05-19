# Basic workflow with ManGO

<div class="alert alert-block alert-info">
<h3>Authentication with Python</h3>

You can authenticate to the Python client with the mango_auth library. 
The first time you do this (or when you switch to a different ManGO zone), you need to follow a url to authenticate. 
This can be done from your terminal, or inside the script.

Next times, a refresh token is used, so no human interaction is needed.

You can also authenticate via some other clients (iron and iCommands), whose authentication also works for Python scripts.
For instructions go to https://mango.vscentrum.be/ and in the tab of your zone, click on "How to connect".
</div>


```python
# Option 1: Authentication via the terminal
!bash -c "mango_auth <username> gbiomed gbiomed.irods.icts.kuleuven.be"
```

    2026-05-19 10:42:08,445 - irods.auth.pam_interactive - INFO - Server prompt: You are authenticated using a refresh token.


## Setting up

The first step _in each session_ is to set up ManGO and load any other libraries you need.
For demonstration purposes, we will load the necessary libraries in each section of the tutorial,
but on actual code the best practice is to load them all at the beginning.


```python
import os
from irods.session import iRODSSession

try:
    env_file = os.environ['IRODS_ENVIRONMENT_FILE']
except KeyError:
    env_file = os.path.expanduser('~/.irods/irods_environment.json')
```

Since we are working interactively we will create an `irods.session.iRODSSession` object and then close it at the end of the notebook with `session.cleanup()`. If you were working on a script, you could run all your code inside a context manager like so:

```python
with iRODSSession(irods_env_file=env_file) as session:
    # do your stuff!
```


```python
session = iRODSSession(irods_env_file=env_file)
```

The final step to set up your environment is to define your working directory in a variable. For this notebook, it's "/gbiomed/home/training/Exercises" (a collection within the training project).

This is particularly useful because in order to manipulate collections and data objects you will need to provide the _absolute path_,
so having the common path as a variable will save you some typing -- and typos.


```python
home_dir = "/gbiomed/home/training/Exercises/"
```

You will need all the code above at the start of any notebook that needs to connect to ManGO.

--------------

The code below is illustration of basic functions to communicate with ManGO; take them as a cheatsheet and use them at your convenience.

### An unnecessarily technical intro for curious Pythonistas

The Python iRODS Client has classes to represent a.o. collections (`irods.collection.iRODSCollection`), data objects (`irods.data_object.iRODSDataObject`), metadata items (`irods.meta.iRODSMeta`) and permissions (`irods.access.iRODSAccess`), but also manager classes for each of these classes. These managers can be accessed as attributes of the session (`irods.session.iRODSSession`) or, in case of the metadata, of the collections and data objects:

- `session.collections` is the collection manager and lets you create, remove and obtain information about specific collections.
- `session.data_objects` is the data object manager and lets you create, remove, upload, download, open and obtain information about specific data objects.
- `session.acls` is the permissions manager and lets you set and remove permissions to data objects and collections.
- `.metadata` is the metadata manager for a given collection or data object and lets you list, add, remove and modify its metadata.

## Collections

You can _get information_ about a specific iRODS collection by instantiating it as an `iRODSCOllection` object with `session.collections.get("/path/to/collection")`; this could be your project collection or any other sub-collection.

This action in itself does not change anything in iRODS or download anything locally.

Let's look at the collection we chose above:


```python
coll = session.collections.get(home_dir)
coll
```




    <iRODSCollection 588345786 Exercises>




```python
coll.path
```




    '/gbiomed/home/training/Exercises'



The subcollections and data objects contained in a collection can be listed with the `subcollections` and `data_objects` attributes, respectively. We can also use the `.walk()` method to get the full tree.

<div class="alert alert-block alert-info">
<b>Note</b>: Your output will be different depending on your reading permissions; you'll only see the dataset that you have access to and the collection of your team.
</div>


```python
coll.subcollections
```




    [<iRODSCollection 588345795 example>,
     <iRODSCollection 588345789 input>,
     <iRODSCollection 588345792 output>]




```python
coll.subcollections[0].path
```




    '/gbiomed/home/training/Exercises/example'




```python
coll.data_objects
```




    []




```python
for subcoll, subsubcolls, objects in coll.walk():
    if len(subsubcolls) == 0 and len(objects) == 0:
        continue
    print(f"{subcoll.path} contains {len(subsubcolls)} subcollections and {len(objects)} data objects.")
    if len(subsubcolls) == 1:
        print(f"The subcollection is: {subsubcolls[0].name} and was created on {subsubcolls[0].create_time}")
    if len(objects) == 1:
        print(f"The data object is: {objects[0].name}")
    print()
```

    /gbiomed/home/training/Exercises contains 3 subcollections and 0 data objects.
    
    /gbiomed/home/training/Exercises/example contains 1 subcollections and 0 data objects.
    The subcollection is: input and was created on 2026-05-19 08:40:01+00:00
    
    /gbiomed/home/training/Exercises/example/input contains 0 subcollections and 1 data objects.
    The data object is: 2_OHara_S1_18S_2019_minq7.fastq
    
    /gbiomed/home/training/Exercises/input contains 0 subcollections and 1 data objects.
    The data object is: 2_OHara_S1_18S_2019_minq7.fastq
    


## Uploading data to ManGO

This section shows how to create new collections, upload data to ManGO and remove data objects from ManGO.
Here we will create the "input" collection inside the "example" collection.


```python
input_coll = session.collections.create(os.path.join(home_dir, "example", "input"))
input_coll.path
```




    '/gbiomed/home/training/Exercises/example/input'




```python
input_coll.data_objects # obviously empty
```




    [<iRODSDataObject 589120612 2_OHara_S1_18S_2019_minq7.fastq>]



When you have a local file to upload to ManGO you can upload it with `put()` on the data objects manager (`session.data_objects`), providing first the source (local) path and then the target (iRODS) path.


```python
# let's get a local path
fastq_file = [file for file in os.listdir() if file.endswith(".fastq")][0]
fastq_file
```




    '2_OHara_S1_18S_2019_minq7.fastq'




```python
# send a local file
fastq_object_path = os.path.join(input_coll.path, fastq_file)
session.data_objects.put(fastq_file, fastq_object_path)
input_coll.data_objects
```




    [<iRODSDataObject 589120612 2_OHara_S1_18S_2019_minq7.fastq>]



## Stream data from memory

You can also stream data that you have in memory by opening a data object and writing content into it, like you would do with a local file.

<div class="alert alert-info">The <code>size</code> of a data object is an <em>inmutable</em> property, so if it changes, i.e. because we have updated the file, we need to <code>get()</code> the data object again to see the updated size.</div>


```python
streamed_object_path = os.path.join(input_coll.path, "from_streaming.txt")
streamed_file = session.data_objects.create(streamed_object_path)
streamed_file.size # obviously empty
```




    0




```python
with streamed_file.open("w") as f:
    f.write(b"This is one line\n") # open as 'w' but write bytes!
session.data_objects.get(streamed_object_path).size
```




    17




```python
with streamed_file.open("a") as f:
    f.write(b"This is the second line\n") # open as 'a' but write bytes!
session.data_objects.get(streamed_object_path).size
```




    41



## Remove data objects

Data objects can be removed from ManGO with `unlink()`. The `force` argument indicates whether it should be permanently deleted (`True`) or sent to trash (`False`). Objects in the trash get removed automatically after 14 days.


```python
session.data_objects.unlink(streamed_object_path, force=True)
input_coll.data_objects
```




    [<iRODSDataObject 589120612 2_OHara_S1_18S_2019_minq7.fastq>]



# Checksum

You can check the sha2 checksums with the `checksum` attribute, if they have been set with the `chksum()` method.

<div class="alert alert-info">The <code>checksum</code> of a data object is an <em>inmutable</em> property, so if it changes, i.e. because we have calculated it or the file has changed, we need to <code>get()</code> the data object again to see the checksum.</div>


```python
obj = input_coll.data_objects[0]
obj.path
```




    '/gbiomed/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq'




```python
obj.checksum # there is none (yet)
```


```python
obj.chksum() # return value: the checksum
```




    'sha2:DLAm/LOJQUvsD3OfA6w5cdIxtNsZEiFz3hrEud29ukE='




```python
obj = input_coll.data_objects[0] # update the variable
obj.checksum
```




    'sha2:DLAm/LOJQUvsD3OfA6w5cdIxtNsZEiFz3hrEud29ukE='



When you have uploaded a file, specially a big one, you might want to check that the checksum in iRODS is the same as in your local system, i.e. that the integrity of the file has been preserved.


```python
import base64
import hashlib
def get_sha256(path: str) -> str:
    """Compute the checksum of a local file"""
    hash_sha256 = hashlib.sha256()
    BUFFER = 32 * 1024 * 1024
    with open(path, "rb") as file:
        for chunk in iter(lambda: file.read(BUFFER), b""):
            hash_sha256.update(chunk)
    local_checksum_sha256 = hash_sha256.hexdigest()
    return local_checksum_sha256
def irods_checksum_as_hex(checksum: str) -> str:
    """Translate the iRODS checksum into hexadecimal"""
    return base64.b64decode(checksum.replace("sha2:", "")).hex()
```


```python
local_checksum = get_sha256(fastq_file)
local_checksum
```




    '0cb026fcb389414bec0f739f03ac3971d231b4db19122173de1ac4b9ddbdba41'




```python
irods_checksum_as_hex(obj.checksum)
```




    '0cb026fcb389414bec0f739f03ac3971d231b4db19122173de1ac4b9ddbdba41'




```python
local_checksum == irods_checksum_as_hex(obj.checksum)
```




    True



# Permissions

In order to upload data you need at least "write" permissions on a collection. If the inheritance is not on for that collection, only you will have access (and own access) to that data unless you give it to someone else as well.

With the access manager in `session.acls` you can check the permissions of data and, if you have "own" access, give permissions to someone else.
In order to _read_ permissions, you have to provide an `iRODSCollection` or `iRODSDataObject` _instance_ to the permissions manager (`session.acls`).


```python
session.acls.get(obj) # only the creator has access
```




    [<iRODSAccess own /gbiomed/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq <username>(rodsuser) gbiomed>,
     <iRODSAccess read_object /gbiomed/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq training(rodsgroup) gbiomed>]



In order to _set_ permissions will need to import `iRODSAccess` as well, and give it to the permissions manager with the following arguments:

- the permission:
    - "read", "write" or "own" to give one of those permissions
    - "none" to remove someone's permissions
    - "inherit"/"noinherit" to set inheritance on or off on a collection
- the target, i.e. the _path_ to the data object or collection
- for read/modify/own/none access, the user_name (to which user/group you want to give access) -> we recommend groups!


```python
from irods.access import iRODSAccess
```


```python
session.acls.set(iRODSAccess("read", obj.path, user_name="training"))
```


```python
session.acls.get(obj)
```




    [<iRODSAccess own /gbiomed/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq <username>(rodsuser) gbiomed>,
     <iRODSAccess read_object /gbiomed/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq training(rodsgroup) gbiomed>]



## Inheritance

The access can also be given at the level of a collection, recursively or not. If the permission is "inherit" or "noinherit", this sets the inheritance.
Because this is an _inmutable_ property, we need to `get()` the collection again to see if it has changed.


```python
input_coll.inheritance # current inheritance
```




    False




```python
session.acls.set(iRODSAccess("inherit", input_coll.path)) # set inheritance to True
session.collections.get(input_coll.path).inheritance # new inheritance - the example_subcoll variable has not been updated!
```




    True




```python
session.acls.set(iRODSAccess("noinherit", input_coll.path))
session.collections.get(input_coll.path).inheritance
```




    False



# Metadata

Metadata is listed and managed from the metadata manager of a data object (as we'll illustrate below) or a collection. We can list it with `items()`.


```python
obj.metadata.items() # no metadata (yet)
```




    []



## Arbitrary metadata
If you want to add arbitrary metadata, you can create instances of `irods.meta.iRODSMeta` and use `add` or `set` to add them and `remove` to remove them.


```python
from irods.meta import iRODSMeta
```


```python
obj.metadata.add(iRODSMeta("organism", "mouse"))
obj.metadata.items()
```




    [<iRODSMeta 588545133 organism mouse None>]




```python
obj.metadata.add(iRODSMeta("organism", "human")) # this will add a new item
obj.metadata.items()
```




    [<iRODSMeta 588545133 organism mouse None>,
     <iRODSMeta 588545136 organism human None>]




```python
obj.metadata.set(iRODSMeta("organism", "mouse")) # this will overwrite all items of that name
obj.metadata.items()
```




    [<iRODSMeta 588545133 organism mouse None>]




```python
obj.metadata.remove(iRODSMeta("organism", "mouse"))
obj.metadata.items()
```




    []



For multiple similar operations with metadata, you can use "atomic operations" and send a sequence of metadata items in one go:


```python
from irods.meta import AVUOperation
```


```python
obj.metadata.apply_atomic_operations(*[AVUOperation(operation="add", avu=iRODSMeta("organism", org)) for org in ["mouse", "human", "cat"]])
obj.metadata.items()
```




    [<iRODSMeta 588545133 organism mouse None>,
     <iRODSMeta 588545136 organism human None>,
     <iRODSMeta 588545733 organism cat None>]




```python
obj.metadata.apply_atomic_operations(*[AVUOperation(operation="remove", avu=iRODSMeta("organism", org)) for org in ["mouse", "human", "cat"]])
obj.metadata.items()
```




    []



## Schema metadata
For ManGO metadata schemas you can use the mango-mdschema package to retrieve a schema from ManGO and use it to validate and convert a Python dictionary into schema metadata.


```python
from mango_mdschema import Schema, ValidationError, ConversionError # to add structured metadata
from mango_mdschema.schema import get_mango_schema
import logging
logger = logging.getLogger("mango_mdschema") # to read the validation
logger.setLevel(logging.INFO)
```


```python
mango_schema = get_mango_schema(session, "training", "fastq") # get the schema from iRODS
mango_schema
```




    <iRODSDataObject 588461820 fastq-v1.0.0-published.json>




```python
fastq_schema = Schema(mango_schema)
```

The package allows you to inspect the schema and its requirements in detail.


```python
print(fastq_schema)
```

    [1mFastQ[0m
    Metadata annotated with the schema 'fastq' (1.0.0) carry the prefix 'mgs'.
    This schema contains the following 3 fields:
    - [1msample[0m, of type 'object'.
    - [1morganism[0m, of type 'select'.
    - [1mfastq[0m, of type 'object'.



```python
fastq_schema.print_requirements("sample")
```

    [1mType[0m: object.
    [1mRequired[0m: False.
    [1mRepeatable[0m: False.
    
    Composed of the following fields:
    [4mfastq.sample.sample_id[0m
    [1mType[0m: text.
    [1mRequired[0m: False.
    [1mRepeatable[0m: False.
    
    [4mfastq.sample.condition[0m
    [1mType[0m: select.
    [1mRequired[0m: False.
    [1mRepeatable[0m: False.
    Choose only one of the following values:
    - normal
    - tumor



```python
fastq_schema.print_requirements("organism")
```

    [1mType[0m: select.
    [1mRequired[0m: False.
    [1mRepeatable[0m: False.
    Choose only one of the following values:
    - human
    - mouse



```python
fastq_schema.print_requirements("fastq")
```

    [1mType[0m: object.
    [1mRequired[0m: False.
    [1mRepeatable[0m: False.
    
    Composed of the following fields:
    [4mfastq.fastq.encoding[0m
    [1mType[0m: select.
    [1mRequired[0m: False.
    [1mRepeatable[0m: False.
    Choose only one of the following values:
    - Phred+33
    - Phred+64
    
    [4mfastq.fastq.no_records[0m
    [1mType[0m: integer.
    [1mRequired[0m: False.
    [1mRepeatable[0m: False.


Given a Python dictionary, you can simply validate it against a schema before trying to apply it.


```python
example_metadata = {
    "sample": {
        "sample_id": "18S_amplicon",
        "condition": "normal"
    },
    "organism": "Mouse",
    "fastq" : {
        "encoding": "phred64",
        "no_records": 109831
    }
}
fastq_schema.validate(example_metadata)
```

    INFO:mango_mdschema:'fastq.organism' must be one of the following values: human, mouse; got 'Mouse'. It was discarded.
    INFO:mango_mdschema:'fastq.fastq.encoding' must be one of the following values: Phred+33, Phred+64; got 'phred64'. It was discarded.





    {'sample': {'sample_id': '18S_amplicon', 'condition': 'normal'},
     'organism': None,
     'fastq': {'encoding': None, 'no_records': 109831}}




```python
example_metadata = {
    "sample": {
        "sample_id": "18S_amplicon",
        "condition": "normal"
    },
    "organism": "mouse",
    "fastq" : {
        "encoding": "Phred+64",
        "no_records": "109831"
    }
}
fastq_schema.validate(example_metadata)
```




    {'sample': {'sample_id': '18S_amplicon', 'condition': 'normal'},
     'organism': 'mouse',
     'fastq': {'encoding': 'Phred+64', 'no_records': 109831}}



You can also see how the metadata items would look like in iRODS and finally apply them to the target item, which removes any preexisting metadata items matching the same schema.


```python
fastq_schema.to_avus(example_metadata)
```




    [<iRODSMeta None mgs.fastq.sample.sample_id 18S_amplicon 1>,
     <iRODSMeta None mgs.fastq.sample.condition normal 1>,
     <iRODSMeta None mgs.fastq.organism mouse None>,
     <iRODSMeta None mgs.fastq.fastq.encoding Phred+64 1>,
     <iRODSMeta None mgs.fastq.fastq.no_records 109831 1>]




```python
fastq_schema.apply(obj, example_metadata)
```

    INFO:mango_mdschema:0 existing AVUs linked to the schema 'fastq' are removed.



```python
obj.metadata.items()
```




    [<iRODSMeta 588471066 mgs.fastq.sample.sample_id 18S_amplicon 1>,
     <iRODSMeta 588471069 mgs.fastq.sample.condition normal 1>,
     <iRODSMeta 588471072 mgs.fastq.organism mouse None>,
     <iRODSMeta 588471075 mgs.fastq.fastq.encoding Phred+64 1>,
     <iRODSMeta 588471078 mgs.fastq.fastq.no_records 109831 1>,
     <iRODSMeta 588471081 mgs.fastq.__version__ 1.0.0 None>]



Conversely, if you have schema metadata in an object or collection, you can retrieve it and convert it to a Python dictionary.


```python
fastq_schema.from_avus(obj.metadata.items())
```

    INFO:mango_mdschema:Following unknown fields in 'fastq' will be ignored: ['__version__'].





    {'fastq': {'encoding': 'Phred+64', 'no_records': 109831},
     'organism': 'mouse',
     'sample': {'condition': 'normal', 'sample_id': '18S_amplicon'}}



## Queries

We can run queries with `session.query()`, which collects information from collections, data objects, and their metadata with specific classes. More interestingly, we can filter that information based on certain Criteria.


Class | Information about | Useful attributes
---- | ------ | ----------
`Collection` | A collection | `name`, `owner_name`, `id` ...
`DataObject` | A data object | `name`, `path`, `size`, `owner_name`, `id` ...
`CollectionMeta` | The metadata of a collection | `name`, `value`, `units`, ...
`DataObjectMeta` | The metadata of a data object | `name`, `value`, `units`, ...


```python
from irods.models import Collection, DataObject, CollectionMeta, DataObjectMeta
from irods.column import Criterion
```

The following query retrieves all the collections inside our project collection (`home_dir`), regardless of their depth, and prints their paths.


```python
query = session.query(Collection.name).filter(Criterion("like", Collection.name, home_dir + "%"))
for result in query:
    print(result[Collection.name])
```

    /gbiomed/home/training/Exercises/example
    /gbiomed/home/training/Exercises/example/input
    /gbiomed/home/training/Exercises/input
    /gbiomed/home/training/Exercises/output


In the cells below, we **request** the path of our collections and the names and date of creation of our data objects: those are the arguments in `session.query()`.

Then we **filter** the results based on the following criteria (which may touch or properties not included in the query):

- The collection path has to end in "put" ('like' + '%put')
- The data object is smaller than 1GB in size.
- The data object must have been created after '2026-04-27 13:45:25'.
    + In order to define the date-time threshold we use the `datetime` library.
- The data object should have an "organism" metadata item from the "fastq" schema with value "mouse".


```python
import datetime
threshold = datetime.datetime.fromisoformat('2026-04-27 13:45:25')
```


```python
my_files = session.query(Collection.name, DataObject.name, DataObject.create_time, DataObject.path).filter(
    Criterion("like", Collection.name, home_dir + "%")).filter(
    Criterion('like', Collection.name, '%put')).filter(
    Criterion('<', DataObject.size, 1000000000)).filter(
    Criterion('>', DataObject.create_time, threshold)).filter(
    Criterion('=', DataObjectMeta.name, 'mgs.fastq.organism')).filter(
    Criterion('=', DataObjectMeta.value, 'mouse')
    )
for item in my_files:
    print(item[DataObject.name], item[Collection.name], item[DataObject.create_time])
```

    2_OHara_S1_18S_2019_minq7.fastq /gbiomed/home/training/Exercises/example/input 2026-05-19 08:40:40+00:00
    2_OHara_S1_18S_2019_minq7.fastq /gbiomed/home/training/Exercises/input 2026-05-18 09:23:16+00:00


<div class="alert alert-block alert-warning">The <code>path</code> of an <code>iRODSDataObject</code> is its full path, but the <code>path</code> of a <code>DataObject</code> in a query is NOT. (It is the path in the physical storage, but you cannot access it as an end user.) If you want to retrieve the full path of a data object from a query, you need to concatenate the <code>Collection.name</code> and the <code>DataObject.name</code>!!</div>


```python
[item[DataObject.path] for item in my_files] # DO NOT USE THIS FOR DOWNLOADING
```




    ['/data/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq',
     '/data/home/training/Exercises/input/2_OHara_S1_18S_2019_minq7.fastq']




```python
[os.path.join(item[Collection.name], item[DataObject.name]) for item in my_files] # THIS WORKS FOR DOWNLOADING
```




    ['/gbiomed/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq',
     '/gbiomed/home/training/Exercises/input/2_OHara_S1_18S_2019_minq7.fastq']



The `.execute()` method returns a printable table with the columns requested in `.query()`.


```python
print(my_files.execute())
```

    +------------------------------------------------+---------------------------------+---------------+-----------------------------------------------------------------------------+
    | COLL_NAME                                      | DATA_NAME                       | D_CREATE_TIME | D_DATA_PATH                                                                 |
    +------------------------------------------------+---------------------------------+---------------+-----------------------------------------------------------------------------+
    | /gbiomed/home/training/Exercises/example/input | 2_OHara_S1_18S_2019_minq7.fastq | 01779180040   | /data/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq |
    | /gbiomed/home/training/Exercises/input         | 2_OHara_S1_18S_2019_minq7.fastq | 01779096196   | /data/home/training/Exercises/input/2_OHara_S1_18S_2019_minq7.fastq         |
    +------------------------------------------------+---------------------------------+---------------+-----------------------------------------------------------------------------+


## Download data from ManGO

In order to access the data you have on ManGO, you can download it with `get()`, providing a local path as second argument.


```python
source_path = obj.path
filename = "COPIED-" + obj.name
f"We will move the object in '{source_path}' to (local) '{filename}'."
```




    "We will move the object in '/gbiomed/home/training/Exercises/example/input/2_OHara_S1_18S_2019_minq7.fastq' to (local) 'COPIED-2_OHara_S1_18S_2019_minq7.fastq'."




```python
os.path.exists(filename)
```




    False




```python
session.data_objects.get(source_path, filename)
os.path.exists(filename)
```




    True



But just like we opened a data object before to write data into it, we can open one to read data from it.


```python
with obj.open() as f:
    first_line = f.readline().decode().strip() # decode() to go from bytes to string
first_line
```




    '@49189868-cc2e-4767-bd11-3b6498462435 runid=f53ee40429765e7817081d4bcdee6c1199c2f91d sampleid=18S_amplicon read=102315 ch=61 start_time=2019-09-12T12:04:38Z'



## CLEAN UP 
<div class="alert alert-block alert-warning">
    <font size=4><b>Do not forget to clean up your session!</b></font>
</div>


```python
session.data_objects.unlink(obj.path)
os.remove(filename)
session.collections.remove(input_coll.path, force=True)
```


```python
# leave this cell at the end and running every time you are done
session.cleanup()
```
