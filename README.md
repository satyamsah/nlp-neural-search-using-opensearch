Neural Search with OpenSearch
This guide walks through setting up a Neural Search pipeline in OpenSearch using ML models, ingestion pipelines, k-NN indexing, and querying with both keyword and neural search.

1️⃣ Check Cluster Health
sh
Copy
Edit
GET /_cat/health
To use OpenSearch-provided machine learning models without dedicated ML nodes, update the cluster settings:

json
Copy
Edit
PUT _cluster/settings
{
  "persistent": {
    "plugins": {
      "ml_commons": {
        "only_run_on_ml_node": "false",
        "model_access_control_enabled": "true",
        "native_memory_threshold": "99",
        "rag_pipeline_feature_enabled": "true",
        "memory_feature_enabled": "true",
        "allow_registering_model_via_local_file": "true",
        "allow_registering_model_via_url": "true",
        "model_auto_redeploy.enable": "true",
        "model_auto_redeploy.lifetime_retry_times": 10
      }
    }
  }
}
2️⃣ Set Up an ML Language Model
Register a Model Group
To register a model group with public access:

json
Copy
Edit
POST /_plugins/_ml/model_groups/_register
{
  "name": "NLP_model_group",
  "description": "A model group for NLP models",
  "access_mode": "public"
}
Response (Example):

json
Copy
Edit
{
  "model_group_id": "IxEFEpUBnf1r1bXAYtqp"
}
Register a Model to the Model Group
json
Copy
Edit
POST /_plugins/_ml/models/_register
{
  "name": "ml-model/huggingface/sentence-transformers/all-distilroberta-v1",
  "version": "1.0.1",
  "model_group_id": "Z1eQf4oB5Vm0Tdw8EIP2",
  "model_format": "TORCH_SCRIPT"
}
Response (Example):

json
Copy
Edit
{
  "task_id": "NecHEpUBjB2oluuy33AV"
}
To check model registration status:

sh
Copy
Edit
GET /_plugins/_ml/tasks/NecHEpUBjB2oluuy33AV
If successful:

json
Copy
Edit
{
  "model_id": "aVeif4oB5Vm0Tdw8zYO2",
  "task_type": "REGISTER_MODEL",
  "function_name": "TEXT_EMBEDDING",
  "state": "COMPLETED"
}
3️⃣ Deploy the Model
json
Copy
Edit
POST /_plugins/_ml/models/aVeif4oB5Vm0Tdw8zYO2/_deploy
Response:

json
Copy
Edit
{
  "task_id": "ale6f4oB5Vm0Tdw8NINO",
  "status": "CREATED"
}
To check deployment status:

sh
Copy
Edit
GET /_plugins/_ml/tasks/ale6f4oB5Vm0Tdw8NINO
4️⃣ Create an Ingest Pipeline
The ingest pipeline automatically converts text into embeddings before indexing.

json
Copy
Edit
PUT /_ingest/pipeline/nlp-ingest-pipeline
{
  "description": "An NLP ingest pipeline",
  "processors": [
    {
      "text_embedding": {
        "model_id": "aVeif4oB5Vm0Tdw8zYO2",
        "field_map": {
          "text": "passage_embedding"
        }
      }
    }
  ]
}
Verify the pipeline:

sh
Copy
Edit
GET /_ingest/pipeline
5️⃣ Create a k-NN Index
Create an index with k-NN enabled to store embeddings.

json
Copy
Edit
PUT /my-nlp-index
{
  "settings": {
    "index.knn": true,
    "default_pipeline": "nlp-ingest-pipeline"
  },
  "mappings": {
    "properties": {
      "id": {
        "type": "text"
      },
      "passage_embedding": {
        "type": "knn_vector",
        "dimension": 768,
        "method": {
          "engine": "lucene",
          "space_type": "l2",
          "name": "hnsw",
          "parameters": {}
        }
      },
      "text": {
        "type": "text"
      }
    }
  }
}
6️⃣ Ingest Documents
Add documents to the index. The ingest pipeline automatically generates embeddings.

json
Copy
Edit
PUT /my-nlp-index/_doc/1
{
  "text": "A West Virginia university women 's basketball team , officials , and a small gathering of fans are in a West Virginia arena.",
  "id": "4319130149.jpg"
}

PUT /my-nlp-index/_doc/2
{
  "text": "A wild animal races across an uncut field with a minimal amount of trees.",
  "id": "1775029934.jpg"
}
Verify a document:

sh
Copy
Edit
GET /my-nlp-index/_doc/1
7️⃣ Search Queries
Traditional Keyword Search
json
Copy
Edit
GET /my-nlp-index/_search
{
  "_source": {
    "excludes": ["passage_embedding"]
  },
  "query": {
    "match": {
      "text": {
        "query": "wild west"
      }
    }
  }
}
Neural Search
Uses ML-generated embeddings instead of keyword matching.

json
Copy
Edit
GET /my-nlp-index/_search
{
  "_source": {
    "excludes": ["passage_embedding"]
  },
  "query": {
    "neural": {
      "passage_embedding": {
        "query_text": "wild west",
        "model_id": "aVeif4oB5Vm0Tdw8zYO2",
        "k": 5
      }
    }
  }
}
8️⃣ Sample Neural Search Response
json
Copy
Edit
{
  "took": 25,
  "timed_out": false,
  "hits": {
    "total": {
      "value": 5,
      "relation": "eq"
    },
    "hits": [
      {
        "_index": "my-nlp-index",
        "_id": "4",
        "_score": 0.0158,
        "_source": {
          "text": "A man who is riding a wild horse in the rodeo is very near to falling off.",
          "id": "4427058951.jpg"
        }
      },
      {
        "_index": "my-nlp-index",
        "_id": "2",
        "_score": 0.0157,
        "_source": {
          "text": "A wild animal races across an uncut field with a minimal amount of trees.",
          "id": "1775029934.jpg"
        }
      }
    ]
  }
}
9️⃣ Summary
Setup OpenSearch ML settings ✅
Register and deploy an ML model ✅
Create an ingest pipeline for embeddings ✅
Create a k-NN index ✅
Ingest text documents (automatic embedding generation) ✅
Perform keyword-based and neural search queries ✅
🚀 This setup allows OpenSearch to automatically convert text queries into embeddings and retrieve the most semantically relevant documents using vector search.

