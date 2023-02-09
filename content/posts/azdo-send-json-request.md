---
title: "(Azure DevOps) Send Json Request(Parameters) to Azure Pipelines"
date: 2021-06-21T11:16:23+04:00
draft: false
toc: false

tags:
  - azure devops
  - pipelines
---
![azure](/azure-pipelines.png)
### Introduction
If you've been using Azure DevOps, you would know that a pipeline can be triggered with runtime parameters in the format `key: value` pair and this is great for doing almost all of the tasks. 

For our use case, we had been looking at an option to send a JSON-based parameter dictionary and I couldn't find any way at the time of writing this article. We came up with a hack to achieve this and I would want to write it up in this blog post.

### Flatten JSON & operation
We look at the option to use the Flatten JSON Objects extension to convert a nested data layer object into a new object with only one layer of key/value pairs. For this, we used the **flatten_json** library

`pip install flatten_json`


```
'''
flatten the JSON data with a '_' separator; you can use different separators as well
'''
# flatten.py
import flatten_json

data = {
    "a": 1,
    "b": 2,
    "c": [{"d": [2, 3, 4], "e": [{"f": 1, "g": 2}]}]
}
flat_json = flatten_json.flatten(data,'_')
print(flat_json)
'''
the above command gives us the below output, which is single layer of JSON
{
 'a': 1,
 'b': 2,
 'c_0_d_0': 2,
 'c_0_d_1': 3,
 'c_0_d_2': 4,
 'c_0_e_0_f': 1,
 'c_0_e_0_g': 2
 }
'''
```

Now that we had a single layer `key: value` pair, we added another hack to be able to read these variables properly in the Azure DevOps pipeline. We added a prefix (any prefix that's not part of your JSON) **RizwanGotNoChill-**

```
prefixed_flat_json = {f"RizwanGotNoChill-{key}": val for key, val in flat_json.items()}
print(prefixed_flat_json)
'''
the above command gives us the below output, which is single layer prefixed JSON
{
 'RizwanGotNoChill-a': 1,
 'RizwanGotNoChill-b': 2,
 'RizwanGotNoChill-c_0_d_0': 2,
 'RizwanGotNoChill-c_0_d_1': 3,
 'RizwanGotNoChill-c_0_d_2': 4,
 'RizwanGotNoChill-c_0_e_0_f': 1,
 'RizwanGotNoChill-c_0_e_0_g': 2
 }
'''
```

This part is all we needed, now you could use the Azure DevOps Rest API with the above `key: value` pair to trigger any pipeline and I will show, how we interpreted these in the pipeline and crafted a JSON out.

### Trigger Azure DevOps Pipeline

I found the below two ways to trigger a pipeline, you may want to see which one fits your case - 

- Using REST API-based curl/HTTP command calling the **build API** - https://dev.azure.com/YOURORG/YOURPROJECT/_apis/build/builds?api-version=6.1-preview.6)
1. Convert your JSON to an escaped one using this [link](https://jsonformatter.org/json-escape)
2. Create the request param as below and send 
```
# Replace the values for YOURORG, YOURPROJECT, PATTOKEN and the value for id(pipeline ID)
curl --request POST \
  --url 'https://dev.azure.com/YOURORG/YOURPROJECT/_apis/build/builds?api-version=6.1-preview.6' \
  --header 'Authorization: Basic PATTOKEN' \
  --header 'Content-Type: application/json' \

  --data '{
    "parameters": "{\"RizwanGotNoChill-a\":1,\"RizwanGotNoChill-b\":2,\"RizwanGotNoChill-c_0_d_0\":2,\"RizwanGotNoChill-c_0_d_1\":3,\"RizwanGotNoChill-c_0_d_2\":4,\"RizwanGotNoChill-c_0_e_0_f\":1,\"RizwanGotNoChill-c_0_e_0_g\":2}",
    "definition":  { "id":  27 }
}'
```

- The other way to trigger a pipeline is by calling the **RunPipeline API**, I wrote a small program 

{{< gist rizwan-kh c08955a24bfc1eb3eaa48248acd012e0 >}}

### Unflatten JSON & operation

Now the baton is at the Azure DevOps end to decode and re-configure this as a JSON, I used a simple job and Python script to achieve this

```
# azure-pipeline.yml
- job: install_library
  steps:
  - checkout: none
  - script: |
      pip3 install flatten_json
      pip3 freeze
    displayName: Install python library

- job: generate_files
  steps:
  - script: |
      echo "Here, we will grep the content based on the prefix and store in a file"
      env | grep RizwanGotNoChill > request.txt
      sed -i 's/RizwanGotNoChill-//' request.txt
      echo "utils.py first creates a dict based on request.txt and then unflattens the files and write to request.json"
      python3 utils.py
      cat request.json | jq .
    displayName: Produce JSON
```

```
# utils.py
import os, sys
import json
import flatten_json

dict_from_file = {}
with open("request.txt") as fp:
    for line in fp:
        (key, val) = line.split("=")
        dict_from_file[key.lower()] = val.rstrip("\n")
print(dict_from_file)

request_json = flatten_json.unflatten(dict_from_file, '-')
print(request_json)

with open('request.json', 'w') as fp:
     fp.write(json.dumps(request_json))

```

Here, the `request.json` is a proper JSON file ready to be used by any application/program in the Azure Pipeline; 
We noticed one issue though, the conversion process, converts all other data types to string datatype, so may be you would need to change that or add some logic to create json/dict with type-safe.