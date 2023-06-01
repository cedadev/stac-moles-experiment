# Report 
## Postgres DB into JSON file
I exported content of DB using `./manage.py dumpdata > fixture.json` command and compressed with gzip. Everything, including admin logs, was saved to the `.json` file, so I needed to tidy my data. `fixture1.5.json` represents subset of `fixture.json` objects and contains only `cedamoles_app` model. Typical `json` representation of `django model` looks like that:
```
{'model': 'cedamoles_app.referenceable',
  'pk': 1,
  'fields': {'uuid': '3b1a86cc61824d78ce195dc21b661c74',
             'short_code': 'coll'}
}
```
There are 33 of them. My goal was to reduce it down to 11, which is the number of `referenceable` objects (those ones with assigned `uuid`). All the work related to modifying or analyzing the DB content was done using `converting-fixtures.ipynb` notebook, which I will be describing in the next section.
## Modifying fixtures
### Referenceable
`Referenceable` is a small object with field `uuid` and `short_code`. It is connected with more significant object via foreign key. My initial step was to map `short_code` into proper model name. In order to assign `uuids` correctly I converted `referenceable` objects into `dictionary`, where `PKs (primary keys)` were keys and bodies of the objects were values. Then I iterated over the entire list of objects and appended `uuid` to objects where `PK` of the object was one of the keys in `referenceable dictionary` and `short_code` mapped to the model name matched object's model's name.

### Relations via PKs
Fields of models, which points to other models, uses PKs as reference e.g.
```
'party' in the object:
{'model': 'cedamoles_app.responsiblepartyinfo',
  'pk': 14017,
  'fields': {'priority': 1,
   'party': 19,
   'role': 'ceda_officer',
   'relatedTo': 3165}
 }
```
For every model, which had such a reference, I had to specify referenceable model and name of field, that this model should be inserted into. I figured out the list of model-field pairs by exploring moles project. `include_simple_field()` handles this operation. It takes 3 arguments: current json data saved to python `dict`; name of model to be inserted; list of model-field pairs, to specify where the model should be inserted e.g.
```
data = include_simple_field(data, 'vocabularyterm', [('observation', 'vocabularyKeywords')])
```
will insert `vocabularyterm` which `PK` matches `PK` stored in the `observation`, under `vocabularyKeywords` field. It works for both single values, and lists. 

### Relations via FKs
Some models stores information about the PK of the object they are related to. Look once more at the example I showed before:
```
{'model': 'cedamoles_app.responsiblepartyinfo',
  'pk': 14017,
  'fields': {'priority': 1,
   'party': 19,
   'role': 'ceda_officer',
   'relatedTo': 3165}
 }
```
`relatedTo` stores PK of referenceable objects. Methodology of inserting this kind of model into referenceable model if slightly different, so it required separate method. `include_on_foreign_key()` takes 3 arguments: current json data saved to python `dict`; name of the model to be inserted; alternative name of the field we want to store the model under. This method looks for fields like 'relatedTo' or 'relatedRecords' in the given model and inserts the model into one of the referenceable models if its PH matches the value of `relatedTo` (FK) field. The unique case was `relatedobservationinfo`, since this model contains 2 FKs and had to be handled separately. 

At this point data was saved to the `fixture2.json.gz` before further modifications. 

### Creating relations and removing PKs
Only 11 models remained in the json file, so it was time for restructuring records and establishing relations between them. References to the referenceable objects are still handled by PKs at this point e.g.

`member: 435` in the `observationcollection`

Once more I decided to create a map. This time with PKs as keys and corresponding `uuids` as values. Moreover, I created a list of fields which contains PKs, that need to be replaced with `uuids`. After all I removed PKs and restructured records from:
```
{
"model": "observation",
"pk": 2137,
"fields": {
  "field1" : "values1",
  ...
  }
}
```
to
```
{
"model": "observation",
"uuid": "yh98dbp9aufbdsjfh89",
"field1": "value1",
...
}
```
Those changes was saved to the `fixture3.json.gz`.
### Geographic extent
ElasticSearch allows geospatial searching, however it requires data to be in the specific format. One of the types available for `geographic extent` is `envelope`. It stores rectangular area as `[[W, N],[E, S]]` where W is a west bound and so on. In this iteration regular `latitude-longitude` box was replaced with the ES format and saved to the `fixture3.5.json.gz`.

### Analyses
`analysis_of_size.csv` shows comparison in number of models' instances, size in MB and number of fields between fixtures.
`analysis_of_deepness.txt` shows fields, grouped by level of deepness where `dep(data['a']['b']) == 2`. 
`distribution_3.txt` shows fields grouped by model, and number of non empty values within DB.
`distribution_3_grouped.txt` shows the same as above, but grouped by the number of non empty values.
`distribution_3_m_grouped.txt` shows the same as above, but model is ignored, to reduce number of combinations, and expose fields, which are actually rarely used.
`rare_fields_with_uuids.txt` shows pairs of uuids and model's names for every field with less than 15 non empty values within DB.
`runtime.tsv` compares times of executing the same, massive query (FAAM collection), on both `stac-moles-test` and `moles-haystack` indexes and shows that `stac-moles-test` responses 4x slower (I suppose it due to the amount of data stored on the index).

### Elasticsearch
`fixture3.5.json.gz` was put on the `stac-moles-test` index. I created `searching_examples.ipynb` notebook to present typical use cases. `search()` method takes few optional keyword arguments and returns list of `hits` from ES response. 

`bbox = [[W, N],[E, S]]` specifies bounding box 

`bbox_relation = 'intersect/within'` specifies relation

`fields = ['model']` specifies list of fields that will be taken from the ES

`source = True` set to False if you want to get only fields from the fields arg

`size = 100` max number of records to be returned 

`query_string = 'ozone'` free text search

`**terms` filtering by specified field

#### Emulating catalogue
One of the most resource-demanding operation in the catalogue is collecting and displaying records related to the record. Such relation maybe be going through a few levels of MOLES e.g. In order to get every platform associated with the instrument via instrument platform pair we have to get every acquisition and then IPP from each of them. `get_related_objects()` takes uuid of the record and returns list of the uuids of related records. Maximal number of ES requests needed to get related records for any record type is 4 (see number of calls of `search()` method inside any submethod of `get_related_objects()`). According to the data from `runtime.csv` file retrieving observations related to the FAAM collections takes 4 times longer comparing to the current Haystack index. I suppose the reason is that the number of records on 
