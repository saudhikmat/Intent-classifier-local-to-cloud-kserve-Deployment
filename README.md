# Intent Classifier Model - Local Deployment Demo

This project demonstrates how to train a lightweight text-classification model for intent detection and expose it through a local Flask API. It is useful as a small ML demo, a backend service for a chatbot, or a base for a simple local GUI or web app.

## Overview

The workflow is intentionally simple:

- A small training script builds a text classifier from example phrases.
- The trained pipeline is saved as a serialized model artifact.
- A Flask app loads the artifact and serves predictions through a `/predict` endpoint.
- The API can be integrated into a front-end or GUI layer for user interaction.

## Project structure

- `app.py` - Flask application that exposes prediction endpoints.
- `model/train.py` - Trains the intent classifier and saves the model artifact.
- `model/intent_model.py` - Loads the saved model and performs inference.
- `model/artifacts/intent_model.pkl` - Serialized trained model.
- `requirements.txt` - Python dependencies for the project.
- `wsgi.py` - WSGI entry point for deployment frameworks.

## Step 1: Create a virtual environment

From the project root, create and activate a Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

## Step 2: Install dependencies

```bash
pip install -r requirements.txt
```

This installs Flask, scikit-learn, joblib, pytest, and gunicorn.

## Step 3: Train the model

Run the training script:

```bash
python model/train.py
```

This creates the model file:

```text
model/artifacts/intent_model.pkl
```

The training script uses a simple `CountVectorizer` + `MultinomialNB` pipeline on a few example phrases such as greeting, complaint, question, and praise.

## Step 4: Start the Flask API

Launch the app locally:

```bash
python app.py
```

The service will run on:

```text
http://127.0.0.1:6000
```

## Step 5: Check service health

Use this endpoint to confirm the server is live:

```bash
curl http://127.0.0.1:6000/health
```

Example response:

```json
{"status":"ok"}
```

## Step 6: Send a prediction request

Example request:

```bash
curl -X POST http://127.0.0.1:6000/predict \
  -H "Content-Type: application/json" \
  -d '{"text":"I want to cancel my subscription"}'
```

Example response:

```json
{
  "intent": "complaint"
}
```

The actual model also exposes probability values internally through the model pipeline and can be extended to return them in the API response if needed.

## Optional: Run with WSGI

If you want a production-style server entry point:

```bash
gunicorn --bind 0.0.0.0:6000 wsgi:app
```

## Troubleshooting

- If you see a `ModuleNotFoundError`, ensure your virtual environment is activated and dependencies are installed.
- If the model file is missing, run `python model/train.py` before starting the API.
- If port 6000 is already in use, change the port in `app.py` and restart the service.

## GUI architecture

A graphical interface can be added on top of this API using a browser-based frontend that sends text input to the Flask backend and displays the returned intent. See [GUI_ARCHITECTURE.md](./GUI_ARCHITECTURE.md) for the UI and interaction design.
