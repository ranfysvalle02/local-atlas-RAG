# local-atlas-RAG

## Setup 

### Step 1: Local Atlas Environment 

1. **Pull the Docker Image:**
   * **Latest Version:**
     ```bash
     docker pull mongodb/mongodb-atlas-local
     ```
   * **Specific Version:**
     ```bash
     docker pull mongodb/mongodb-atlas-local:<tag>
     ```
     Replace `<tag>` with the desired version.

2. **Run the Database:**
   ```bash
   docker run -p 27017:27017 mongodb/mongodb-atlas-local
   ```
   This command runs the Docker image, exposing port 27017 on your host machine to connect to the database.

## Using Sample Datasets with MongoDB

This section describes how to download and explore a complete sample dataset for MongoDB on your local machine.

### Downloading the Dataset

There is a complete sample dataset available for MongoDB. You can download it using either `wget` or `curl`:

**Using wget:**

```bash
wget https://atlas-education.s3.amazonaws.com/sampledata.archive
```

**Using curl:**

```bash
curl https://atlas-education.s3.amazonaws.com/sampledata.archive -o sampledata.archive
```

**Note:**

* Ensure you have `wget` or `curl` installed on your system.
* The downloaded file will be named `sampledata.archive`.

### Restoring the Dataset

Before restoring the dataset, make sure you have a local instance of `mongod` running. You can either use an existing instance or start a new one. This instance will be used to host a local copy of the sample dataset.

**To restore the dataset:**

```bash
mongorestore --archive=sampledata.archive
```

This command uses the `mongorestore` tool to unpack the downloaded archive (`sampledata.archive`) and populate your local `mongod` instance with the sample data.

## Creating an Atlas Vector Search Index Programmatically with mongosh

### Steps:

1. **Connect to the local Atlas Cluster:**
   Use `mongosh` to connect to the database:
   ```bash
   mongosh "mongodb://localhost/?directConnection=true"
   ```

2. **Switch to the Database:**
   Select the database that contains the collection you want to index:

   ```javascript
   use sample_mflix
   ```

3. **Create the Index:**
   Execute the following command:

```
db.embedded_movies.createSearchIndex(
  "vector_index",
  "vectorSearch", //index type
  {
    fields: [
      {
        "type": "vector",
        "numDimensions": 1536,
        "path": "plot_embedding",
        "similarity": "cosine"
      }
    ]
  }
);
```

3. **Check the Status of Index:**
   Execute the following command:

```
db.embedded_movies.getSearchIndexes()
```

4. **Wait for Status 'READY':**

```
[
  {
    id: '66f0c9c9ce6bb7512d7e8ba7',
    name: 'vector_index',
    type: 'vectorSearch',
    status: 'READY',
    queryable: true,
    latestVersion: 0,
    latestDefinition: {
      fields: [
        {
          type: 'vector',
          path: 'plot_embedding',
          numDimensions: 1536,
          similarity: 'cosine'
        }
      ]
    }
  }
]
```



- Once completed, head to the QnA section to start asking questions based on the sample embedded data, and you should get the desired response.

  ![image](https://github.com/utsavMongoDB/MongoDB-RAG-NextJS/assets/114057324/c76c8c19-e18a-46b1-834a-9a6bda7fec99)



## Reference Architechture 

![image](https://github.com/mongodb-partners/MongoDB-RAG-Vercel/assets/114057324/3a4b863e-cea3-4d89-a6f5-24a4ee44cfd4)


This architecture depicts a Retrieval-Augmented Generation (RAG) chatbot system built with LangChain, OpenAI, and MongoDB Atlas Vector Search. Let's break down its key players:

- **PDF File**: This serves as the knowledge base, containing the information the chatbot draws from to answer questions. The RAG system extracts and processes this data to fuel the chatbot's responses.
- **Text Chunks**: These are meticulously crafted segments extracted from the PDF. By dividing the document into smaller, targeted pieces, the system can efficiently search and retrieve the most relevant information for specific user queries.
- **LangChain**: This acts as the central control unit, coordinating the flow of information between the chatbot and the other components. It preprocesses user queries, selects the most appropriate text chunks based on relevance, and feeds them to OpenAI for response generation.
- **Query Prompt**: This signifies the user's question or input that the chatbot needs to respond to.
- **Actor**: This component acts as the trigger, initiating the retrieval and generation process based on the user query. It instructs LangChain and OpenAI to work together to retrieve relevant information and formulate a response.
- **OpenAI Embeddings**: OpenAI, a powerful large language model (LLM), takes centre stage in response generation. By processing the retrieved text chunks (potentially converted into numerical representations or embeddings), OpenAI crafts a response that aligns with the user's query and leverages the retrieved knowledge.
- **MongoDB Atlas Vector Store**: This specialized database is optimized for storing and searching vector embeddings. It efficiently retrieves the most relevant text chunks from the knowledge base based on the query prompt's embedding. These retrieved knowledge nuggets are then fed to OpenAI to inform its response generation.


This RAG-based architecture seamlessly integrates retrieval and generation. It retrieves the most relevant knowledge from the database and utilizes OpenAI's language processing capabilities to deliver informative and insightful answers to user queries.


## Implementation 

The below components are used to build up the bot, which can retrieve the required information from the vector store, feed it to the chain and stream responses to the client.

#### LLM Model 

        const model = new ChatOpenAI({
            temperature: 0.8,
            streaming: true,
            callbacks: [handlers],
        });


#### Vector Store

        const retriever = vectorStore().asRetriever({ 
            "searchType": "mmr", 
            "searchKwargs": { "fetchK": 10, "lambda": 0.25 } 
        })

#### Chain

       const conversationChain = ConversationalRetrievalQAChain.fromLLM(model, retriever, {
            memory: new BufferMemory({
              memoryKey: "chat_history",
            }),
          })
        conversationChain.invoke({
            "question": question
        })
