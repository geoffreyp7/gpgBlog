---
title: Creating a Serverless NLP API using Spacy and AWS Lambda
category: "NLP"
cover: photo-spacy.jpg
author: Geoffrey Parry-Grass
---

##Introduction

Spacy allows us to perform many useful NLP processes easily and reliably.

When integrating Spacy into existing applications there are many ways to do this. A convenient and modular way to run a Spacy service is as an API using AWS Lambda and API Gateway. Spacy is not natively deployable to lambda as the python package includes a lot of additional modules which increase the size beyond Lambda's limit. These modules are actually not required for most tasks and by removing them we are able to run Spacy on Lambda functions.

##Creation of Spacy Lambda Package

(If you do not need to be able to create updated Spacy packages you can skip to the next section and deploy a pre-created Lambda layer)

To create a Spacy environment which is compatible with Lambda, we will need to modify the default package.

Since Lambda is a linux environment and Spacy relies on compiled C libraries the following procedure will have to be performed on a linux OS. In particular it must be an "manylinux" environment (e.g. not Ubuntu). I used a virtual machine running Kali Linux.

First create a new folder where we will create our Lambda python environment.

Create a python virtual environment in this directory.

```
python3 -m venv env
```

Activate the virtual environment.

```
source env/bin/activate
```

Now install Spacy.

```
pip install spacy
```

We can now remove the extra modules to reduce the size. The large files are located in the spacy/lang directory. As we can see Spacy supports many different languages. Chances are we are only supporting 1 or 2 languages and as such we are able to delete all the folders for the other languages; drastically reducing the size.

We will now create our deployable environment by compressing the base directory.

```
zip -r9 spacy.zip .
```

##Deployment

Now we can deploy our created environment as a Lambda layer. Follow [this guide](https://medium.com/@adhorn/getting-started-with-aws-lambda-layers-for-python-6e10b1f9a5d) to create a lambda layer from a zip file. (replace the mentioned zip file with our freshly created zip file from the last section).

Now to create our lambda function with access to the spacy layer we can simply create a new lambda function and choose our newly created layer in the layer area. Any code we write in the Lambda function will now have access to Spacy.

##Basic API

As a quick example we can now create a simple keyword/keyphrase extraction API with the following function code:

```
import json
import spacy
import en_core_web_sm

def lambda_handler(event, context):
    
    nlp = en_core_web_sm.load()

    doc = nlp(event['Text'])

    keyphrases_text = []
    keywords_text = []

    for chunk in doc.noun_chunks:
        if (not chunk.root.is_stop):
            keywords_text.append(chunk.root.lemma_)
            
        chunk_contains_non_stop = False
        noun_chunk = []
        for token in chunk:
            if(not token.is_stop and token.pos_ != 'PUNCT'):
                noun_chunk.append(token)
                chunk_contains_non_stop = True
        if chunk_contains_non_stop:
            noun_chunk_string = ' '.join([x.lemma_.lower() for x in noun_chunk])
            if not noun_chunk_string in keyphrases_text:
                keyphrases_text.append(noun_chunk_string)
    
    keyphrases_for_return = [{"Text":text} for text in keyphrases_text]
    keywords_for_return = [{"Text":text} for text in keywords_text]

    return {
        'statusCode': 200,
        'KeyPhrases': keyphrases_for_return,
        'KeyWords': keywords_for_return
    }
```

We can then create a simple API using API gateway by pointing a GET request to our new Lambda function!

##Conclusion
We can now create many useful NLP functions and deploy them as self-sufficient APIs for easy integration into our products thanks to Lambda and Layers! I will add more details on creating advanced APIs with API gateway soon.


