# Python InvenioRDM SDK proposal (PyInvenioRDM or PyRDM)

- **Goal**: Provide a **simple**, intuitive, high level  SDK for researchers/data managers to easily integrate InvenioRDM in their worflows

- The SDK will be made up by a Python **package**, providing classes and methods to abstract InvenioRDM REST APIs, available on PyPi and be used from Python scripts and Jupyter notebooks. 

- The Python SDK could be later used to build a Command Line Interface (CLI) to be used directly by the shell and/or in shell scripts

## Design guidelines

This SDK is devoted to researchers, students, data scientists and not towards pro developers (that could use probably the direct HTTP API) that know "just a bit" of Python for their data analysis. 

One of design goal would be to facilitate API exploration by means of autocompletition for methods and paramenters, using Enums for example to choose values from list of options, and method keyword arguments in opposition to positional arguments.

We should use Python constructs as much as possibile, instead of providing new methods. For example, adding a new version should be implemented using the `.append()` method of the built-in type `list` vs the creation of a new `add_new_version()` method.

If our users would be able to use the SDK without consulting any documentation (or at least after just the getting started guide) we reached our goal.


## Existing SDKs

We are aware of two existing implementations of other Python SDK over InvenioRDM REST APIs:

- [Enijo Connector](https://enijo-connector.readthedocs.io/en/latest/)
This is a Python SDK built to provide two simple use cases: record creation and file download. It's built using a bottom-up approach. It wraps the InvenioRDM REST APIs using [openapi-generator](https://github.com/openapi-generators/openapi-python-client) starting from a [YAML description](https://gitlab.com/qyanu/enijo-connector/-/blob/main/openapi/invenio-api.yaml). On top of those low level API, a higher level APIs were created. This probably helped a lot to have a working implementaion in a short amount of time, but the high level APIs are not very pythonic. Have a look to the [provided notebooks](https://gitlab.com/qyanu/enijo-connector/-/blob/main/show-cases/publish_new_record.ipynb) to have an idea of how the API looks like, especially for creating new records (using the Builder pattern)

- [Tu Graz Repository CLI](https://github.com/tu-graz-library/repository-cli) 
This is a CLI to be used in the same machine of InvenioRDM as it uses InvenioRDM instance python modules.



## Use cases

> Here we should insert a sample set of use cases






## Features

The SDK will give access to the same features exposed by the InvenioRDM REST APIs, but avoiding researchers to deal with direct HTTP requests. It will also massage/marshall input paraments and output response as appropriate for consuming (i.e. returning Python data structures, like dictionaries/list or Pandas DataFrames).


Here a list of features we plan to implement (in order of availability/priority):

### Repository and authentication setup

A class and methods to setup the endpoint of a InvenioRDM instance and configure the authentication (token). 

#### Sample API

```python
import Repository from inveniordm_sdk

infn_repo = Repository(url="https://www.openaccessrepository.it")
# alternatively, a second `token` parameter could be set in the constructor
infn_repo.set_token("3i4jg3i4jgi34gi3jgi34jgfaketoken")
```

### Record metadata retrieval

A set of classes/methods/enums to retrieve metadata of published/draft records by recordID (or by DOI) 
> we need to check if the RecordID is always the final part of a DOI

> https://inveniordm.docs.cern.ch/customize/dois/#generated-doi
- DOI could be arbitrary including somewhere or nowhere the id
- what if a record doesn't get a DOI
- what if a record gets another identifier

> double check if we can query records by DOIs via REST APIs


The `Record` class will have properties that map each *key* returned by the JSON response of the corresponding REST API, so that syntax completition available in code editors/notebook can be used for metatada exploration or editing, avoding typos.



#### Sample APIs

```python
r = infn_repo.get_record(id="dbn00-8d609")
#r2 = infn_repo.get_record(doi="10.81088/h8ry5-12745")
# r is of type Record
print(r.languages) ## return a Python List of `Languages` enums representing languages
print(r.type) ## return a enum. Ex. Types.PHOTO
print(r.raw_metadata) ## returns the raw data as dict
print(r.publisher)
print(r.description)
print(r.title)

# In case we have different metadata for each resource type, it will return a subclass of the Record type, like PhotoRecord, VideoRecord 

```



### Record files download

A set of methods of the Record class to download selected record's `File`s (that has it's own class)

#### Sample APIs

```python

print(r.files) # returns the list of Files
# Let's download the first file and save it locally with the same name
local_file = r.files[0].name
with open(local_file, 'wb') as output:
    output.write(r.download(r.files[0]))

```

Other than direct download, we plan to support streaming download (the underlying `request` module supports this) for large files, to avoid loading the binary payload all in memory before writing it on disk.

### Draft record creation

A set of classes/methods/enums for defining a new record and it's metadata. Enums can be created dynamically at runtime using the [Enum functional APIs](https://docs.python.org/3/library/enum.html#functional-api). For example we can build Enums for _Languages_, _Licenses_ and _Resource types_ using the [Vocabularies API](https://inveniordm.docs.cern.ch/reference/rest_api/#vocabularies) 

#### Sample API 

```python
new_record = Record()
author = Creator(
    name = "Antonio",
    surname = "Calanducci",
    type = Creators.Types.PERSONAL,
)
author_id = Identifier(
    scheme = Schemes.ORCID,
    id = "040345-30345034"
)
author.identifiers = [author_id]
new_record.creators = [author]
new_record.title = "Sample record"
new_record.keywords = ["sample", "test", "hello"]
new_record.description = "sample description"
new_record.doi = "10.81088/dkzva-7rn41"  # if empty it will be autogenerated
new_record.type = Types.IMAGES.PLOT #enums
new_record.licenses = [Licenses.GPL] # Licenses is a class/enums with some default licenses and methods to create custom ones
new_record.save() # new_record will be updated with the response metadata from the server (i.e. id, doi, etc)
print(new_record.id)

# Record metadata could be set/read via instance variables or using keyword parameters in the constructor
another_record = Record(
    title="Hello", 
    resource_type=Types.PRESENTATION, #we cannot use type as it's reserved built-in in Python (double check this) 
    licenses=[Licenses.LGPL, Licenses.MIT],
    keywords=["un","dos","tres"],
    creators=[author, Creator(name="Giovanni", surname="Paoli")]
).save()
print(another_record.doi)

```


### Draft files upload

This API will allow to upload draft file's content. This API will hide the underlying start and complete/commit API calls.


#### Sample API 

```python
new_record.files = [File("picture.jpg"), File("article.pdf"), File("test.zip")]
for input_file in new_record.files
    with open(input_file.name, 'rb') as data:
        new_record.upload(data)

new_record.publish()
```



### Version management

This API will allow to retrieve all version of a record and to create a new version

#### Sample API

```python
print(new_record.versions) # return a list of `Record`s
new_record_v2 = Record(parent = new_record) # create and return a new record version
```
### Records search

 This API will allow to create creare without hiding the query syntax of Elastic Search. It will provide anyway a generic search() method to manually specify the query string

All queries will return a list of `Record`. A future implementation could provide an `Iterable` instance with a default pagination to improve performance

#### Sample API

```python
repository.search_by_fields(title="data science", resource_type="publication", version="v2")
repository.search_all("open", "science", exclude="python").sort_by_pub_date()
repository.search_any("open", "data", "Python").sort_by_title()
# if want to issue a generic elastic search API
repository.search(q="metadata.title='Sample'")

```









