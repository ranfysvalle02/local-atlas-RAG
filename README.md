# local-atlas-RAG

This README guides you through setting up a local Retrieval-Augmented Generation (RAG) environment with MongoDB Atlas and sample datasets.

### Prerequisites

* **MongoDB Tools:**
  * **mongosh:** The official MongoDB shell for interacting with MongoDB databases.
  * **mongorestore:** A tool for restoring data from a dump file to a MongoDB database.
* **Docker:** Installed on your system ([https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/))
* **wget or curl:** Installed on your system (package managers usually handle this)


### Setting Up a Local Atlas Environment

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
   This command runs the Docker image, exposing port 27017 on your machine for connecting to the database.

### Using Sample Datasets with MongoDB

This section demonstrates downloading and exploring a sample dataset for MongoDB on your local system.

#### Downloading the Dataset

There's a complete sample dataset available for MongoDB. Download it using either `wget` or `curl`:

* **Using wget:**

```bash
wget https://atlas-education.s3.amazonaws.com/sampledata.archive
```

* **Using curl:**

```bash
curl https://atlas-education.s3.amazonaws.com/sampledata.archive -o sampledata.archive
```

**Note:**

* Ensure you have `wget` or `curl` installed.
* The downloaded file will be named `sampledata.archive`.

#### Restoring the Dataset

Before restoring, ensure you have a local `mongod` instance running (either existing or newly started). This instance will host the dataset.

**To restore the dataset:**

```bash
mongorestore --archive=sampledata.archive
```

This command uses the `mongorestore` tool to unpack the downloaded archive (`sampledata.archive`) and populate your local `mongod` instance with the sample data.

### Creating an Atlas Vector Search Index with mongosh

**Steps:**

1. **Connect to Local Atlas Cluster:**

   Use `mongosh` to connect to the database:

   ```bash
   mongosh "mongodb://localhost/?directConnection=true"
   ```

2. **Switch to the Database:**

   Select the database containing the collection you want to index:

   ```javascript
   use sample_mflix
   ```

3. **Create the Index:**

   ```javascript
   db.embedded_movies.createSearchIndex(
       "vector_index",
       "vectorSearch", // index type
       {
           fields: [
               {
                   "type": "vector",
                   "numDimensions": 1536,
                   "path": "plot_embedding",
                   "similarity": "cosine"
               },
               {"type":"filter","path":"genres"},
               {"type":"filter","path":"type"}
           ]
       }
   );
   ```

4. **Check the Index Status:**

   ```javascript
   db.embedded_movies.getSearchIndexes()
   ```

5. **Wait for Status 'READY'**:

   A successful response will look similar to this:

   ```json
   [
       {
           "id": "...",
           "name": "vector_index",
           "type": "vectorSearch",
           "status": "READY",
           "queryable": true,
           "latestVersion": 0,
           "latestDefinition": {
               "fields": [
                   {
                       "type": "vector",
                       "numDimensions": 1536,
                       "path": "plot_embedding",
                       "similarity": "cosine"
                   }
               ]
           }
       }
   ]
   ```

**Next Steps:**

Once the index is ready, you can proceed to the QnA section (not included here) to start asking questions based on the sample data and receive relevant responses.

## Sample code for RAG
```python
import pymongo
from openai import AzureOpenAI

MONGODB_URI = "mongodb://localhost/?directConnection=true"
client = pymongo.MongoClient(MONGODB_URI)
db = client["sample_mflix"]
collection = db["embedded_movies"]

AZURE_OPENAI_ENDPOINT = ""
AZURE_OPENAI_API_KEY = "" 
deployment_name = "text-embedding-ada-002"  # The name of your model deployment
az_client = AzureOpenAI(azure_endpoint=AZURE_OPENAI_ENDPOINT,api_version="2023-07-01-preview",api_key=AZURE_OPENAI_API_KEY)

#the $vectorSearch filter option matches only BSON boolean, string, and numeric values 
# so you must index the fields as one of the following Atlas Search field types.
LOCAL_ACL = {
     "UserA":{
          "genres":{"$eq":"Horror"}, # only has access to horror movies
     },
     "UserB":{
          "genres":{"$in":["Romance","Comedy"]}, # only has access to romance movies
     },
     "UserC":{
          "type":{"$ne":"movie"}, # only has access to non-movies
     }
}

def vs_tool(text,user_id):
        #$vectorSearch
        chunk_max_length = 1000
        response = collection.aggregate([
        {
            "$vectorSearch": {
                "index": "vector_index",
                "queryVector": az_client.embeddings.create(model=deployment_name,input=text).data[0].embedding,
                "path": "plot_embedding",
                "filter": LOCAL_ACL[str(user_id)],
                "limit": 5, #Number (of type int only) of documents to return in the results. Value can't exceed the value of numCandidates.
                "numCandidates": 30 #Number of nearest neighbors to use during the search. You can't specify a number less than the number of documents to return (limit).
            }
        },{"$project":{"_id":0, "title":1, "genres":1, "released":1, "type":1}},{"$sort":{"released":-1,"awards.wins":-1}}
       ])
        str_response = []
        for d in response:
            str_response.append({"title":d["title"],"genres":d["genres"],"released":d["released"],"type":d["type"]})
        
        if len(str_response)>0:
            return f"Knowledgebase Results for User={user_id} [{len(str_response)}]:\n{str(str_response)}\n"
        else:
            return "N/A"

       
print(
    vs_tool("Santa Claus is coming to town", "UserA")
)

      
print(
    vs_tool("Santa Claus is coming to town", "UserB")
)

      
print(
    vs_tool("Santa Claus is coming to town", "UserC")
)
"""
Knowledgebase Results for User=UserA [5]:
[{'title': 'Jack Frost 2: Revenge of the Mutant Killer Snowman'}, {'title': 'Jack Frost 2: Revenge of the Mutant Killer Snowman'}, {'title': 'Rare Exports: A Christmas Tale'}, {'title': 'Carny'}, {'title': 'The Witches of Eastwick'}]
Knowledgebase Results for User=UserB [5]:
[{'title': 'Cancel Christmas'}, {'title': 'Santa Who?'}, {'title': 'Mrs. Santa Claus'}, {'title': 'The Perfect Holiday'}, {'title': "Beethoven's Christmas Adventure"}]
Knowledgebase Results for User=UserC [5]:
[{'title': 'The Storyteller'}, {'title': 'Tin Man'}, {'title': "Dead Man's Walk"}, {'title': "Gulliver's Travels"}, {'title': 'Going Postal'}]
"""
```

## Reference Architechture 

![image](https://github.com/mongodb-partners/MongoDB-RAG-Vercel/assets/114057324/3a4b863e-cea3-4d89-a6f5-24a4ee44cfd4)
