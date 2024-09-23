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
               }
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

```

## Reference Architechture 

![image](https://github.com/mongodb-partners/MongoDB-RAG-Vercel/assets/114057324/3a4b863e-cea3-4d89-a6f5-24a4ee44cfd4)
