# Reference:

- https://cloud.google.com/bigquery/docs/text-embedding-semantic-search#gcloud

- https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-text-embedding

# Base table:

 - **Table name:** arindam-2banerjee-0525-pimy.provider0526.goodclinic

    **Schema**:

    ```bash
    #Schema:
        
    url	STRING	NULLABLE				
    name	STRING	NULLABLE				
    specialty	STRING	REPEATED				
    acceptingstatus	STRING	NULLABLE				
    gender	STRING	NULLABLE				
    language	STRING	NULLABLE				
    services	STRING	REPEATED				
    professional	STRING	REPEATED				
    fellowship	STRING	REPEATED				
    residency	STRING	REPEATED				
    affiliations	STRING	REPEATED				
    internship	STRING	REPEATED				
    certifications	STRING	REPEATED				
    address	STRING	NULLABLE				
    address_saddress	STRING	NULLABLE	
    ```

  **It has 5.5K rows with some columns as a repeated string.**

 - **Let's create a table with 100 rows only to have costs reduced.**

```bash
create or replace table `arindam-2banerjee-0525-pimy.provider0526.goodclinic_limited`
as 
select 'Name: '||name||' - Speciality: '||specialty||' - Accepting Status: '||acceptingstatus||' - Gender: '||gender||' - Language speaks: '||language ||' - Services: '||services||' - Residency: '||residency||' - Address: '||address||' - Reference link: '||url as details
from `arindam-2banerjee-0525-pimy.provider0526.goodclinic` ,  unnest(specialty) as specialty, unnest(services) as services, unnest(residency) as residency limit 100;
```

**Table name:** goodclinic_limited

**Schema:**

```bash
details	STRING	NULLABLE	
```



# Create a connection

 -  **Create a Cloud resource connection and get the connection's service account.**

 ```bash
 # Command:
 #bq mk --connection --location=REGION --project_id=PROJECT_ID \
 #   --connection_type=CLOUD_RESOURCE CONNECTION_ID

 bq mk --connection --location=us-central1 --project_id=arindam-2banerjee-0525-pimy \
    --connection_type=CLOUD_RESOURCE bq-embding-llm-connect 
 ```

 - **Retrieve and copy the service account ID because you need it in a later step:**

 ```bash
 # Command:
 # bq show --connection PROJECT_ID.REGION.CONNECTION_ID

 bq show --connection arindam-2banerjee-0525-pimy.us-central1.bq-embding-llm-connect
 ```

  ![](assets/01-bq-llm-connect.PNG)

 - **Give your service account permission to use the connection: Vertex AI User and BigQuery Connection User**

 ```bash
    #gcloud projects add-iam-policy-binding 'PROJECT_NUMBER' --member='serviceAccount:MEMBER' --role='roles/serviceusage.serviceUsageConsumer' --condition=None

    #gcloud projects add-iam-policy-binding 'PROJECT_NUMBER' --member='serviceAccount:MEMBER' --role='roles/bigquery.connectionUser' --condition=None

    #gcloud projects add-iam-policy-binding 'PROJECT_NUMBER' --member='serviceAccount:MEMBER' --role='roles/aiplatform.user' --condition=None

    gcloud projects add-iam-policy-binding '12959087622' --member='serviceAccount:bqcx-12959087622-o5dc@gcp-sa-bigquery-condel.iam.gserviceaccount.com' --role='roles/serviceusage.serviceUsageConsumer' --condition=None

    gcloud projects add-iam-policy-binding '12959087622' --member='serviceAccount:bqcx-12959087622-o5dc@gcp-sa-bigquery-condel.iam.gserviceaccount.com' --role='roles/bigquery.connectionUser' --condition=None

    gcloud projects add-iam-policy-binding '12959087622' --member='serviceAccount:bqcx-12959087622-o5dc@gcp-sa-bigquery-condel.iam.gserviceaccount.com' --role='roles/aiplatform.user' --condition=None

 ```

 - **Create a model for specific Dataset.**

 **Here the dataset is:** provider0526

  ```bash
 # Command:
 # CREATE OR REPLACE MODEL `semantic_search_tutorial.embedding_model`
 #   REMOTE WITH CONNECTION `PROJECT_ID.REGION.CONNECTION_ID`
 #   OPTIONS (REMOTE_SERVICE_TYPE = 'CLOUD_AI_TEXT_EMBEDDING_MODEL_V1');


  CREATE OR REPLACE MODEL `provider0526.embedding_model`
    REMOTE WITH CONNECTION `arindam-2banerjee-0525-pimy.us-central1.bq-embding-llm-connect`
    OPTIONS (REMOTE_SERVICE_TYPE = 'CLOUD_AI_TEXT_EMBEDDING_MODEL_V1');

  ```
  
 - **Embed text**

 Create a new table that contains the text embedding of the good clinic data by using the **ML.GENERATE_TEXT_EMBEDDING** function:

 ```bash

    # Command
    # Reference: https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-generate-text-embedding
    # CREATE OR REPLACE TABLE semantic_search_tutorial.wind_reports_embedding AS (
    # SELECT *
    # FROM
    # ML.GENERATE_TEXT_EMBEDDING(
    #     MODEL `bqml_tutorial.embedding_model`,
    #     (SELECT "Example text to embed" AS content),
    #     STRUCT(TRUE AS flatten_json_output)
    # );

    # New table created goodclinic_limited_embedding
    # with embeddings
    # from existing data from the table goodclinic_limited
    # by the model provider0526.embedding_model
    CREATE OR REPLACE TABLE `arindam-2banerjee-0525-pimy.provider0526.goodclinic_embeddings` AS (
    SELECT *
        FROM
        ML.GENERATE_TEXT_EMBEDDING(
            MODEL `arindam-2banerjee-0525-pimy.provider0526.embedding_model`,
                (select details as content from  provider0526.goodclinic_limited),
                STRUCT(TRUE AS flatten_json_output)
            )
    );
 ```
 
 **Query Checked**:

 ![](assets/03-embeddings.PNG)
 
 **Table schema created from above embedding query**:
 
 ![](assets/04-embeddings.PNG)
   
 **Data**:
 
 ![](assets/05-preview-embeddings.PNG)
 
 

- **Create a k-means model**:

 **Create a k-means model that clusters the wind reports text embeddings into 10 clusters:**

 ```bash
 # Command
 #CREATE OR REPLACE MODEL `semantic_search_tutorial.clustering`
 #   OPTIONS (
 #   model_type = 'KMEANS',
 #   KMEANS_INIT_METHOD = 'KMEANS++',
 #   num_clusters = 10) AS (
 #       SELECT
 #       text_embedding
 #       FROM
 #       semantic_search_tutorial.wind_reports_embedding
 #   );


    CREATE OR REPLACE MODEL `provider0526.clustering`
        OPTIONS (
        model_type = 'KMEANS',
        KMEANS_INIT_METHOD = 'KMEANS++',
        num_clusters = 10) AS (
            SELECT
                text_embedding
            FROM
                `arindam-2banerjee-0525-pimy.provider0526.goodclinic_embeddings`
        );
 ```

  ![](assets/06-kmeans.PNG)

  