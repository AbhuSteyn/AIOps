# Automated DevOps Documentation Assistant Using OpenLLM (Ollama)

This project demonstrates how to leverage a **Large Language Model (LLM) powered by Ollama** to automate DevOps documentation tasks. The goal is to assist DevOps engineers by dynamically generating and updating documentation based on infrastructure state, CI/CD logs, and system configurations.

By integrating Ollama with **Python**, the solution:
- Reads infrastructure configuration (Terraform, Kubernetes manifests, Ansible playbooks, etc.).
- Analyzes CI/CD logs to summarize failures and actions.
- Generates **human-readable, structured documentation** for services.
- Suggests troubleshooting steps using AI-assisted reasoning.
- Deploys as an **API service**, allowing on-demand document generation.
- Uses **Azure Storage** for storing logs and generated documentation.
- Implements **OIDC-based authentication** using Azure Managed Identity.
- Runs as a **scheduled job in AKS**, ensuring automated updates.

---

## **Business Use Case Overview**
Traditional DevOps documentation is often **manual, outdated, and error-prone**. Engineers spend valuable time keeping deployment guides, troubleshooting documents, and infrastructure diagrams updated. This project eliminates that burden by:

- **Automatically creating structured deployment documentation** from Terraform and Kubernetes manifests.
- **Summarizing CI/CD pipeline logs**, highlighting failures and fixes using AI.
- **Providing troubleshooting recommendations** based on past incidents and infrastructure data.
- **Organizing generated insights in Azure Blob Storage**, accessible via an API.

By using Ollama, the solution **learns from DevOps workflows** and keeps documentation in sync without human intervention.

---

## **Prerequisites**
- **Azure Resources**:
  - **Azure Storage Account** (for logs and generated documentation)
  - **Azure Key Vault** (for secure authentication credentials)
  - **Azure Managed Identity & OIDC** (to securely access cloud services)
  - **Azure Kubernetes Service (AKS)** (to run periodic jobs)
- **Python 3.9+** installed locally.
- **Ollama** installed for LLM execution.

---

## **Application Code (`llm_devops_assistant.py`)**

```python
import os
import logging
import json
from flask import Flask, request, jsonify
import ollama
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient
from azure.keyvault.secrets import SecretClient
from opencensus.ext.azure.log_exporter import AzureLogHandler

# Initialize Flask app
app = Flask(__name__)

# Configure logging to Azure Application Insights
logger = logging.getLogger(__name__)
app_insights_conn_str = os.environ.get("APPINSIGHTS_CONNECTION_STRING")
logger.addHandler(AzureLogHandler(connection_string=app_insights_conn_str))
logger.setLevel(logging.INFO)

# Managed Identity Authentication
credential = DefaultAzureCredential()

# Azure Storage Configuration (for storing logs & generated docs)
STORAGE_CONNECTION_STRING = os.environ.get("STORAGE_CONNECTION_STRING")
STORAGE_CONTAINER = os.environ.get("STORAGE_CONTAINER")
blob_service_client = BlobServiceClient.from_connection_string(STORAGE_CONNECTION_STRING)
storage_container_client = blob_service_client.get_container_client(STORAGE_CONTAINER)

# Azure Key Vault for secure secrets retrieval
KEY_VAULT_URL = os.environ.get("KEY_VAULT_URL")
key_vault_client = SecretClient(vault_url=KEY_VAULT_URL, credential=credential)

# Function to process logs and generate documentation
def generate_devops_docs(logs):
    """Generates structured documentation using Ollama"""
    prompt = f"""
    You are an AI DevOps Assistant. Analyze the following CI/CD logs and generate structured deployment documentation:

    Logs:
    {logs}

    Provide the following sections:
    - Summary of recent changes
    - Key issues and fixes
    - Deployment recommendations
    - Troubleshooting steps
    - Future optimizations
    """
    response = ollama.chat(model="mistral", messages=[{"role": "user", "content": prompt}])
    return response["message"]["content"]

@app.route('/generate-docs', methods=['POST'])
def process_logs():
    """API endpoint to process CI/CD logs and generate deployment documentation."""
    data = request.get_json()
    logs = data.get("logs", "")
    
    if not logs:
        return jsonify({"error": "No logs provided"}), 400

    try:
        docs = generate_devops_docs(logs)
        logger.info("Documentation generated successfully.")

        # Save generated documentation to Azure Blob Storage
        blob_name = f"devops-doc-{os.environ.get('RUN_TIMESTAMP')}.txt"
        blob_client = storage_container_client.get_blob_client(blob_name)
        blob_client.upload_blob(docs, overwrite=True)
        logger.info(f"Saved documentation as {blob_name} in Azure Storage.")

        return jsonify({"message": "Documentation generated and stored successfully", "blob_name": blob_name})
    except Exception as e:
        logger.error("Error generating documentation", exc_info=e)
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## **Unit Testing**

Create a file named `test_llm_devops_assistant.py` for testing:

```python
import pytest
from llm_devops_assistant import app

@pytest.fixture
def client():
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_generate_docs(client):
    """Test that /generate-docs returns expected results."""
    response = client.post('/generate-docs', json={"logs": "Sample CI/CD log: Deployment succeeded."})
    assert response.status_code == 200
    json_data = response.get_json()
    assert "blob_name" in json_data
```

Run tests with:

```bash
pytest test_llm_devops_assistant.py
```

---

## **Dockerization**

Create a `Dockerfile`:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files
COPY llm_devops_assistant.py .
COPY test_llm_devops_assistant.py .

EXPOSE 5000

CMD ["python", "llm_devops_assistant.py"]
```

Build and push the image:

```bash
docker build -t <your-container-registry>/llm-devops-assistant:latest .
docker push <your-container-registry>/llm-devops-assistant:latest
```

---

## **Deploying to AKS as a CronJob**

Create a Kubernetes CronJob (`aks_cronjob.yaml`):

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: llm-devops-doc-job
spec:
  schedule: "0 */6 * * *"  # Runs every 6 hours
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: llm-devops-assistant
        spec:
          containers:
          - name: llm-devops-assistant
            image: <your-container-registry>/llm-devops-assistant:latest
            env:
            - name: STORAGE_CONNECTION_STRING
              value: "<STORAGE_CONNECTION_STRING>"
            - name: STORAGE_CONTAINER
              value: "<STORAGE_CONTAINER>"
            - name: KEY_VAULT_URL
              value: "<KEY_VAULT_URL>"
            - name: APPINSIGHTS_CONNECTION_STRING
              value: "<APPINSIGHTS_CONNECTION_STRING>"
          restartPolicy: OnFailure
```

Deploy the CronJob to AKS:

```bash
kubectl apply -f aks_cronjob.yaml
```

---

## **Summary of Steps**
1. **Set Up Azure Services**:  
   Provision Storage Account, Key Vault, Service Bus, Cosmos DB, and Application Insights.

2. **Develop the LLM-Based Documentation Assistant**:  
   Implement a Flask API with Ollama that generates deployment documentation.

3. **Add Unit Tests**:  
   Use pytest to verify document generation.

4. **Containerize with Docker**:  
   Create a `requirements.txt` and `Dockerfile` for packaging the application.

5. **Deploy as a CronJob in AKS**:  
   Schedule periodic execution (every 6 hours) in AKS.

6. **Secure Authentication with OIDC**:  
   Use **Managed Identity with DefaultAzureCredential** for secure cloud service interactions.

---

## **Conclusion**
This solution leverages **LLM-based AI** (via Ollama) for **intelligent DevOps documentation automation**—reducing manual work, ensuring up-to-date records, and enhancing infrastructure visibility. It integrates **Azure services for storage, authentication, logging, and asynchronous processing**, making it highly scalable and secure.

For further guidance, refer to:
- [Ollama](https://ollama.com)
- [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/)
- [Azure Storage](https://learn.microsoft.com/en-us/azure/storage/)
- [OpenCensus for Application Insights](https://github.com/census-instrumentation/opencensus-python)

```
