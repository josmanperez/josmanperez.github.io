---
title: Movie Finder Application with MongoDB Vector Search + RAG
date: 2025-02-10 09:00:00 +/-0000
author: <josman>
categories: [MongoDB Atlas, Vector Search, GenAI, RAG]
tags: [mongodb, atlas, ai]
description: This tutorial will show how we can built a movie finder application using MongoDB Atlas Vector Search and React
toc: true
image: 
  path: ../assets/img/blog/movies-vector-search/recommendation-system-based-on-atlas-vector-search.jpg
  alt: Introductory image
comments: true
---

In this tutorial, I am going to explain an application project I have created showing how **Atlas Vector Search**, **Atlas Search** and a combined **RAG** system on movie finding by plot works.

We will take a step by step look at the three parts of this project to explain how it works and what is going on behind the scenes. To do so, **part 1** is going to start with the creation of vectors using **OpenAI + Langchain + MongoDB Atlas**.

You can find the repository of this project [here](https://github.com/josmanperez/atlas-vector-search-rag): which is important to download in advance as we are going to explain what each part does under the hood.

---

## Part 1: **CreateAtlasVectorSearch**

After clonning the [Github repository](https://github.com/josmanperez/atlas-vector-search-rag), you will find a folder with the name `createAtlasVectorSearch`. Inside this directory is where we are going to work in part 1 of this 3 part series. First, we need to do is to duplicate the `env.sample` file and rename it to `.env`.

```bash
OPENAI_API_KEY=
ATLAS_CLUSTER_URI=
GROUP_ID=
CLUSTER_NAME=
PUBLIC_KEY=
PRIVATE_KEY=
ATLAS_EMBEDDING_NAMESPACE=sample_mflix.embedded_movies
ATLAS_NAMESPACE=sample_mflix.movies
MODEL_NAME=text-embedding-3-small 
```

We need to fill out the variables here with the data needed for this portion of the application to work.

* `OPENAI_API_KEY`: The OpenAI key needed to make request to OpenAI. You can create yours in [the official OpenAI key section](https://platform.openai.com/docs/overview).
* `ATLAS_CLUSTER_URI`: This the MongoDB Atlas connection URI to connect to your cluster from an application. Please remember that, if you are using [Netwook Access Lists](https://www.mongodb.com/docs/atlas/security/ip-access-list/) to your cluster, add the IP from where you are going to run this application.
* `GROUP_ID`: [The Project ID displayed at the top of the page in Atlas](https://www.mongodb.com/docs/atlas/tutorial/manage-project-settings/#manage-project-settings-1).
* `CLUSTER_NAME`: The name of your Cluster.
* `PUBLIC_KEY`: The [public key](https://www.mongodb.com/docs/atlas/configure-api-access/) needed to access programmatically to you Atlas cluster.
* `PRIVATE_KEY`: The [private key](https://www.mongodb.com/docs/atlas/configure-api-access/) needed to access programmatically to you Atlas cluster.
* `ATLAS_EMBEDDING_NAMESPACE`: This is the namespace (Database.Collection) where the embedding vector will be located.
* `ATLAS_NAMESPACE`: This is the namespace (Database.Collection) where the original documents are stored. The ones we are going to use for generating the embedding vectors.
* `MODEL_NAME`: The OpenAI model we are going to use for embedding.
* `EMBEDDING_KEY`: The field name (collection field name) where our vector is located within each document.

An example of a `.env` file finished, would be somewhat similar to the following:

```bash
OPENAI_API_KEY=sk-proj-5zSKg5QIK1-***
ATLAS_CLUSTER_URI=mongodb+srv://<username>:<password>@cluster1.****.mongodb.net
GROUP_ID=658d46ca7605526eb452****
CLUSTER_NAME=Cluster1
PUBLIC_KEY=eapm****
PRIVATE_KEY=5976f0f4-2304-4042-****
ATLAS_EMBEDDING_NAMESPACE=movies.embedded_movies
ATLAS_NAMESPACE=sample_mflix.movies
MODEL_NAME=text-embedding-3-small
EMBEDDING_KEY=plot_embedding
```

After having this set up, we can now move to the `app.ts` file. This is the **main file of this application**.

When running the app with:

```bash
npm run start
```

We will be prompted with several different options to choose from:

```bash
1 - Create an Embedding array for the plot field on the sample_mflix.movies collection in your Atlas Cluster.
2 - Create an euclidean type Atlas Vector Search Index in your Atlas Cluster.
3 - Create an cosine type Atlas Vector Search Index in your Atlas Cluster.
4 - Create a dotProduct type Atlas Vector Search Index in your Atlas Cluster.
5 - Create a Atlas Search index in your Atlas Cluster.
6 - Query using an Atlas Vector Search Index.
```

Let's look at what these options do one by one:

### 1 - Create an Embedding array for the plot field on the sample_mflix.movies collection in your Atlas Cluster.

When this option is selected, the function `createEmbeddings` is called. This will execute an aggregation pipeline that will look for the field `plot` or `fullplot` (as we want to get the plot of the movies in the `sample_mflix.movies` collection).

> Please note that for this tutotial to work _as is_, the [MongoDB Sample collection](https://www.mongodb.com/docs/atlas/sample-data/sample-mflix/) must have been loaded in your cluster, specifically the `sample_mflix.movies`.
{: .prompt-info }

The aggregation pipeline looks like:

```json
[{
  "$match": {
    "$or": [
      {
        "plot": {
          "$exists": true
        }
      }, {
        "fullplot": {
          "$exists": true
        }
      }
    ]
  }
},{
  "$project": {
    "fullplot": {
      "$ifNull": [
        "$fullplot", "$plot"
      ]
    },
    "year": 1,
    "type": 1
  }
}]
```

This will return the `fullplot` of each movies, and if the `fullplot` field do not exitst will return the `plot` in the `fullplot` field. As long with this, the `year`, `type` and `_id` will be returned. The `year` and `type` would be useful for enabling [pre-filtering](https://www.mongodb.com/docs/atlas/atlas-vector-search/tune-vector-search/#pre-filter-data) when using Atlas Vector Search.

This part of the code will look like:

```js
const fullplot = [];
const ids = [];
for await (const doc of await collection.aggregate([
  {
    '$match': {
      '$or': [
        {
          'plot': {
            '$exists': true
          }
        }, {
          'fullplot': {
            '$exists': true
          }
        }
      ]
    }
  }, {
    '$project': {
      'fullplot': {
        '$ifNull': [
          '$fullplot', '$plot'
        ]
      },
      'year': 1,
      'type': 1
    }
  }
])
) {
  fullplot.push(doc.fullplot);
  ids.push({ _id: doc._id, namespace: namespace, type: doc.type, year: doc.year });
}
```

Therefore, we are iterating in each movies document and returning the `fullplot`, `year`, `type` and `_id` and adding that result in an array variable `fullplot`.

Once we have this array created, we are going to use the static [Langchain `MongoDBAtlasVectorSearch` class](https://v03.api.js.langchain.com/classes/_langchain_mongodb.MongoDBAtlasVectorSearch.html) to create a vector store the list of documents `fullplot` embedded in a vector. This first converts the documents to vectors and then adds them to the MongoDB collection. We are going to use `OpenAIEmbeddings` to create the embeddings from the `fullplot` text field.

The full code will be:

```js
const vectorstore = await MongoDBAtlasVectorSearch.fromTexts(
  fullplot,
  ids,
  new OpenAIEmbeddings({
    openAIApiKey: process.env.OPENAI_API_KEY,
    modelName: process.env.MODEL_NAME,
  }),
  {
    collection: tCollection,
    indexName: 'vs_euclidean', // The name of the Atlas search index. Defaults to 'vs_euclidean'
    textKey: 'text', // The name of the collection field containing the raw content. Defaults to 'fullplot'
    embeddingKey: `${process.env.EMBEDDING_KEY}`, // The name of the collection field containing the embedded text. Defaults to 'plot_embedding'
  }
)
  .finally(() => {
    clearInterval(vectorStoreInterval);
    console.log('\nfinished creating vector store embeddings');
    logger('finished creating vector store embeddings');
  });
```

There a few important things here that we are going to cover one by one:

* `modelName`: This is the embedding model we are going to use to generate the vector embeddings. For this particular example since we are going to embbed text fields, I am using `text-embedding-3-small` from OpenAI.

> Please note that in part two of this tutorial, the input/question **must be embedded using the same model**.
{: .prompt-info }

* `indexName`: This the name of the Vector Search index that we are going to create later on in Atlas and that we will use to retrieve the similary search.
* `textKey`: The name of the collection field containing the raw content and corresponds to the plaintext of `'pageContent'`.
* `embeddingKey`: The name of the collection field containing the embedded vector.

The rest of the code we need to add our recently created vector to MongoDB Atlas, is the following:

```js
const assignedIds = await vectorstore.addDocuments([
  { pageContent: "upsertable", metadata: {} },
]);
logger(`Nº assigned IDs: ${assignedIds.length}`);
const upsertedDocs = [{ pageContent: "overwritten", metadata: {} }];
logger(`Upserted Docs: ${upsertedDocs.length}`);

console.log('adding vector documents to the collection.\n');
logger('adding vector documents to the collection.\n');
vectorstore.addDocuments(upsertedDocs, { ids: assignedIds })
  .then(() => {
    console.log('finished adding vector documents to the collection');
    logger('finished adding vector documents to the collection');
    return
  })
  .catch(err => {
    throw new Error(`Promise rejected with error: ${err}`);
  })
  .finally(() => {
    client.close();
    return;
  });
```

We are using `upsertable` to avoid duplicating documents if this option is selected more than once and the vector embeddings is already created for a certain document.

This part of our application will take a bit to execute but by the time it ends, in your Atlas Cluster you wil find a new database and a new collection with documents with the following structure:

```bash
{ 
  "_id": ObjectId("573a1390f29313caabcd42e8"),
  "text": "Among the earliest existing films in American cinema - notable as the …", 
  "plot_embedding": [-0.0117434, ..., 0.014351842](1536),
  "namespace": "sample_mflix.movies",
  "type": "movie",
  "year": 1903
}
```

### 2 - Create an euclidean type Atlas Vector Search Index in your Atlas Cluster.

We need to create a [Vector Search Index](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-type/) in our Atlas Cluster so that we can query our embedding data. This option in the code will do that for us. The function that will do this is `createIndex`.

There are several ways we can create an Atlas Vector Search index (UI, CLI, API), in this particular example we are going to use the API. For the API call to work, we need to create the body of the request. An Atlas Vector Search index is composed of:

```js
interface IndexBody {
  database: string;
  collectionName: string;
  name?: string;
  type?: string;
  definition?: object;
}
```

Thus, our body will have:

```js
'database': `${embeddingNamespace.split('.')[0]}`, // The name of the database
'collectionName': `${embeddingNamespace.split('.')[1]}`, // The name of the collection,
'name': (type === 'euclidean') ? 'vs_euclidean' : `vs_${type}`, // The name of the index
'type': 'vectorSearch',
'definition': {
  'fields': [
    {
      'path': `${process.env.EMBEDDING_KEY}`, // Name of the field in the collection
      'similarity': `${type}`,
      'type': 'vector',
      'numDimensions': 1536, // This is for OpenAI Embeddings
    },
    {
      'path': 'year',
      'type': 'filter'
    },
    {
      'path': 'type',
      'type': 'filter'
    }
  ]
}
```

Lets talk a little bit about the above fields:

* `database` and `collectionName`: This will tell the index which namespace is the one that have the documents to query. In this case, when we are calling this function we are passing the `embeddingNamespace` variable that contains the `${process.env.ATLAS_EMBEDDING_NAMESPACE}` that we have defined in the `.env` file:

```js
createIndex(embeddingNamespace, type)
  .then((res) => {
    logger(`Atlas Vector Search Index created: ${res}`)
  })
  .catch(handleError);
```

* `name`: This will defined the name of the index. In this case, there are three types of algorithm (-_by the time this tutorial has been written_-) avaialble in Atlas. Euclidean, Cosine and dotProduct. For this, is the `type === 'euclidean'` we are going to use the name `vs_euclidean` but if it is `cosine` we are going to use the name `vs_cosine`. The same for `dotProduct` with `vs_dotProduct`.

> **euclidean** - measures the distance between ends of vectors. This allows you to measure similarity based on varying dimensions.\
**cosine** - measures similarity based on the angle between vectors. This allows you to measure similarity that isn't scaled by magnitude.\
**dotProduct** - measures similarity like cosine, but takes into account the magnitude of the vector. This allows you to efficiently measure similarity based on both angle and magnitude.
{: .prompt-info }

* `type`: For this example, we are going to create an **Atlas vector Search Index** and therefore we will create a `vectorSearch` index type.
* `definition.fields`: This is an array that contains, at least:
  * The primarly definition of our Vector Search index:

  ```js
  {
    'path': `${process.env.EMBEDDING_KEY}`, // Name of the field in the collection
    'similarity': `${type}`,
    'type': 'vector',
    'numDimensions': 1536, // This is for OpenAI Embeddings
  }
  ```

  * `path`: This is the name of the field in the embedding collection where the vectors are stored.
  * `similarity`: One of the three available algorithms: `euclidean`, `cosine` or `dotProduct`.
  * `type`: In this particular case, we are using `vector` as this is an Atlas Vector Search example project.
  * `numDimensions`: The dimension of the vectors. This will depend on the model that we are using for generating the embeddings. In this particular scenario for `OpenAI.text-embedding-3-small` is 1536.

Finally, as we mentioned earlier, we are going to issue a `POST` request using the [Atlas Admin API](https://www.mongodb.com/docs/atlas/reference/api-resources-spec/v2/) for creating this index in our Atlas cluster.

### 3 - Create an cosine type Atlas Vector Search Index in your Atlas Cluster.

This is exactly the same as before but in this case we are going to create a new index named `vs_cosine` for creating a similarity cosine vector search index in our Atlas Cluster.

### 4 - Create a dotProduct type Atlas Vector Search Index in your Atlas Cluster.

This is exactly the same as before but in this case we are going to create a new index named `vs_dotProduct` for creating a similarity dotProduct vector search index in our Atlas Cluster.

> Please note that an M0 Free tier cluster would only us to create a maximum of **3 indexes**.
{: .prompt-warning}

### 5 - Create a Atlas Search index in your Atlas Cluster.

This project will also allow us to create an [Atlas Search Index](https://www.mongodb.com/docs/atlas/atlas-search/create-index/) to compare results. Thus, we can use the same input query to evaluate the results using an Vector Search Index vs an Atlas Search Index.

The overall logic is very similar to what we have defined and used before except that the body of the `POST` request is sligthly different:

```js
createIndex(namespace, '', false)
  .then((res) => {
    logger(`Atlas Vector Search Index created: ${res}`)
  })
  .catch(handleError);
```

An Atlas Search index is defined with:

```js
{
  'database': `${embeddingNamespace.split('.')[0]}`, // The name of the database
  'collectionName': `${embeddingNamespace.split('.')[1]}`, // The name of the collection,
  'name': 'default_search',
  'type': 'search',
  'definition': {
    'mappings': {
      'dynamic': true,
      'fields': {
        'plot': {
          'type': 'string'
        }
      }
    }
  }
}
```

The firts fields: `database`, `collection`, `name`, `type` and `definition` will be present, however, there are some differences:

* `type`: In this case we are going to create a `search` type index instead of a `vector` type.
* `definition`:
  * `mappings.dynamic`: 
  * `mappings.fields.plot.type`: `string`

Once this Atlas Search index is created we can use the `plot` type for using full text search and compare the results with a similarity vector search.

### 6 - Query using an Atlas Vector Search Index.

This last option will allow us to test that the creation of the embeddings and the Vector Search is working as expected. For this option we are using the hardcoded input: `War in outer space` and using `Langchaing MongoDBAtlasVectorSearch` to return the first result with higher [vector score](https://www.mongodb.com/docs/atlas/atlas-vector-search/vector-search-stage/#atlas-vector-search-score).

If all previous steps worked fine, the result we will get is the following:

```bash
[
  Document {
    pageContent: "Near the end of the 20th century, WMDs (weapons of mass destruction) are retired. However, certain factions plan to use a science space station as a weapon against each other. The astronauts inside will decide the world's fate.",
    metadata: {
      _id: new ObjectId("573a1398f29313caabce8f9e"),
      namespace: 'sample_mflix.movies',
      type: 'movie',
      year: 1984
    }
  }
]
````

> If you are using an M0 cluster to test this, please note that a Vector Search index named `vs_euclidean` must be created.
{: .prompt-warning}

In **part 2**, we are going to cover the backend application written in Typescript that we are going to use to display the results in the frontend.

Stay tuned!