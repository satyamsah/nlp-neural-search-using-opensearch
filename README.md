# Neural Search with OpenSearch

This guide walks through setting up a **Neural Search pipeline** in OpenSearch using **ML models, ingestion pipelines, k-NN indexing, and querying with both keyword and neural search**.

## **Summary**
1. **Setup OpenSearch ML settings** ✅  
2. **Register and deploy an ML model** ✅  
3. **Create an ingest pipeline for embeddings** ✅  
4. **Create a k-NN index** ✅  
5. **Ingest text documents (automatic embedding generation)** ✅  
6. **Perform keyword-based and neural search queries** ✅  


🚀 **This setup allows OpenSearch to automatically convert text queries into embeddings and retrieve the most semantically relevant documents using vector search.**
---


## **1️⃣ Check Cluster Health**
```sh
GET /_cat/health
```

To use OpenSearch-provided machine learning models **without dedicated ML nodes**, update the cluster settings:

```json
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
```

---

## **2️⃣ Set Up an ML Language Model**

### **Register a Model Group**
To register a **model group** with public access:
```json
POST /_plugins/_ml/model_groups/_register
{
  "name": "NLP_model_group",
  "description": "A model group for NLP models",
  "access_mode": "public"
}
```

_Response (Example):_
```json
{
  "model_group_id": "IxEFEpUBnf1r1bXAYtqp"
}
```

---

### **Register a Model to the Model Group**
```json
POST /_plugins/_ml/models/_register
{
  "name": "ml-model/huggingface/sentence-transformers/all-distilroberta-v1",
  "version": "1.0.1",
  "model_group_id": "Z1eQf4oB5Vm0Tdw8EIP2",
  "model_format": "TORCH_SCRIPT"
}
```

_Response (Example):_
```json
{
  "task_id": "NecHEpUBjB2oluuy33AV"
}
```

To check model registration status:
```sh
GET /_plugins/_ml/tasks/NecHEpUBjB2oluuy33AV
```

If successful:
```json
{
  "model_id": "aVeif4oB5Vm0Tdw8zYO2",
  "task_type": "REGISTER_MODEL",
  "function_name": "TEXT_EMBEDDING",
  "state": "COMPLETED"
}
```

---

## **3️⃣ Deploy the Model**
```json
POST /_plugins/_ml/models/aVeif4oB5Vm0Tdw8zYO2/_deploy
```

_Response:_
```json
{
  "task_id": "ale6f4oB5Vm0Tdw8NINO",
  "status": "CREATED"
}
```

To check deployment status:
```sh
GET /_plugins/_ml/tasks/ale6f4oB5Vm0Tdw8NINO
```

---

## **4️⃣ Create an Ingest Pipeline**
The ingest pipeline **automatically converts text into embeddings** before indexing.

```json
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
```

Verify the pipeline:
```sh
GET /_ingest/pipeline
```

---

## **5️⃣ Create a k-NN Index**
Create an **index with k-NN enabled** to store embeddings.

```json
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
```

---


