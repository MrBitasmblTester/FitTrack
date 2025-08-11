# FitTrack — title

Description

FitTrack is a web and mobile platform that uses AI (PyTorch) to generate personalized workout and nutrition plans. Plans are produced from user goals, explicit preferences, and real-time health-tracking data. The back end is implemented in Node.js and exposes a GraphQL API to clients (web and mobile). A PyTorch model (training + inference) provides the AI plan generation logic and is invoked by the Node.js GraphQL server.

## Tech Stack

- PyTorch (AI/ML model training and inference)
- GraphQL (API layer)
- Node.js (back-end server and GraphQL resolvers)

## Requirements

This README and repository follow the required structure and include the items listed below:

- Project README using the specified structure
- Use of PyTorch for AI/ML
- Use of GraphQL for API
- Use of Node.js for back-end
- Include stack-specific setup commands and environment variable instructions
- Include implementation steps for training, serving, and integration between PyTorch and Node.js
- Include typical files used in these stacks (package.json, server entry, Python requirements, model artifact, .env example)

## Installation

Prerequisites

- Node.js (>= 16 recommended) and npm
- Python 3.8+ (system or virtualenv)
- GPU/CPU-compatible PyTorch installation as needed for your environment

Repository layout (recommended)

- package.json (Node.js back-end)
- server/ (Node.js GraphQL server files)
- python/ (PyTorch model training and inference scripts)
- .env (environment variables)

Node.js setup

1. Clone the repo and change to the project root:

   git clone <repo-url>
   cd FitTrack

2. Install Node.js dependencies (server/ contains package.json):

   cd server
   npm install

   The server package.json should include dependencies such as:
   - apollo-server (GraphQL server implementation on Node.js)
   - graphql

Python (PyTorch) setup

1. Create and activate a Python virtual environment in the python/ directory:

   cd ../python
   python3 -m venv .venv
   source .venv/bin/activate   # macOS/Linux
   .\.venv\Scripts\activate  # Windows (PowerShell/CMD)

2. Install Python requirements:

   pip install -r requirements.txt

   The python/requirements.txt should include at minimum:
   - torch
   - numpy

Environment variables

Create a .env file in the project root (or server/) with these example variables:

PORT=4000
GRAPHQL_PATH=/graphql
MODEL_PATH=./python/artifacts/fittrack_model.pt
PYTHON_BIN=python3
SECRET_KEY=replace-with-your-secret

Notes:
- MODEL_PATH should point to the exported PyTorch model file created by training.
- PYTHON_BIN is the command used by Node.js to run the inference script (can be absolute path).

## Usage

Build and train the PyTorch model (local development)

1. Train a model (example):

   cd python
   source .venv/bin/activate
   python train.py --output artifacts/fittrack_model.pt

   train.py should save a serialized PyTorch model (.pt) to the path defined by --output.

Run inference from Node.js GraphQL server

1. Ensure MODEL_PATH and PYTHON_BIN are set in .env and the model exists at MODEL_PATH.

2. Start the Node.js GraphQL server:

   cd ../server
   npm start

3. The server will expose a GraphQL endpoint (default http://localhost:4000/graphql).
   Use the GraphQL API to call the mutation/query that requests a personalized plan. The Node.js resolvers will invoke the Python inference script (python/inference.py) using the configured PYTHON_BIN and pass user input via STDIN/STDOUT.

Example GraphQL mutation (client):

mutation GeneratePlan($input: GeneratePlanInput!) {
  generatePlan(input: $input) {
    planId
    type
    durationWeeks
    workouts {
      day
      exercises {
        name
        sets
        reps
      }
    }
    nutrition {
      dailyCalories
      macros {
        protein
        carbs
        fats
      }
    }
  }
}

Variables (example):

{
  "input": {
    "userId": "user-123",
    "goals": ["muscle_gain"],
    "preferences": {"equipment": "dumbbells", "diet": "balanced"},
    "healthData": {"weightKg": 78, "restingHR": 62}
  }
}

Example response (abridged):

{
  "data": {
    "generatePlan": {
      "planId": "plan-abc-001",
      "type": "workout_nutrition",
      "durationWeeks": 12,
      "workouts": [ ... ],
      "nutrition": { ... }
    }
  }
}

## Implementation Steps

1. Project scaffolding
   - Create server/ with package.json, server.js (entry), schema.graphql (or schema.js), and resolvers.js.
   - Create python/ with train.py, inference.py, requirements.txt, and an artifacts/ directory.

2. Define the GraphQL schema (server/schema.graphql)
   - Types: User, GeneratePlanInput, Plan, Workout, Nutrition, Macro
   - Query/Mutation: generatePlan(input: GeneratePlanInput!): Plan
   - The schema file is required for GraphQL servers and tools.

3. Implement PyTorch model training (python/train.py)
   - Use PyTorch to create a model that learns mapping from user features + health data to plan parameters.
   - Save the trained model to ./artifacts/fittrack_model.pt (MODEL_PATH).
   - Provide flags for hyperparameters, dataset path, and output path.

4. Implement Python inference script (python/inference.py)
   - Loads the serialized model from MODEL_PATH.
   - Accepts JSON via STDIN or CLI arguments (user profile, goals, preferences, health data).
   - Runs model.forward(...) and returns a JSON plan on STDOUT.
   - Keep dependencies minimal (torch, numpy, json).

5. Implement Node.js GraphQL server (server/server.js)
   - Use apollo-server and graphql to host /graphql.
   - Load schema.graphql and resolvers.
   - resolvers/generatePlan resolver should:
     a. Validate and sanitize incoming GraphQL input.
     b. Spawn PYTHON_BIN with python/inference.py and pass input JSON via STDIN.
     c. Read JSON output from the Python process and return it as the GraphQL response.
   - Handle timeouts, errors from the Python process, and model unavailability gracefully.

6. Environment and secrets
   - Add .env and dotenv integration in Node.js (or read process.env directly).
   - Expose PORT, MODEL_PATH, PYTHON_BIN, SECRET_KEY, and GRAPHQL_PATH.

7. Tests and local validation
   - Add unit tests for resolvers (mock the Python process).
   - Add integration test that runs a simple inference on a toy model and ensures JSON contract.

8. Packaging and artifacts
   - Include python/artifacts/fittrack_model.pt in .gitignore if large; instead include a sample small model for CI tests.
   - Document model versioning and expected input schema for the inference script.

(Optional) 9. Mobile and web clients
   - Clients should call the GraphQL endpoint and handle plan state locally.
   - Real-time health tracking data should be pushed to the server through appropriate client APIs; the server can enrich calls to generatePlan with recent health snapshots.

## API Endpoints

GraphQL endpoint (server)

- POST http://localhost:4000/graphql
  - Single endpoint for all queries and mutations.
  - Main mutation: generatePlan(input: GeneratePlanInput!): Plan
    - Input fields to include: userId, goals[], preferences, healthData (latest sensors), optional constraints.
    - Returns a Plan object with workouts and nutrition recommendations.

Example schema snippets (conceptual)

type Mutation {
  generatePlan(input: GeneratePlanInput!): Plan
}

type Plan {
  planId: ID!
  type: String
  durationWeeks: Int
  workouts: [Workout]
  nutrition: Nutrition
}

input GeneratePlanInput {
  userId: ID!
  goals: [String!]
  preferences: JSON
  healthData: JSON
}

Notes on implementation details

- The GraphQL server acts as the single integration point for web/mobile clients.
- The actual AI inference is performed by the PyTorch model via the Python inference script. Node.js should treat that script as an external subprocess to keep the Node process lightweight and avoid shipping heavy torch binaries in the Node runtime.

Files you should expect to find or create

- server/package.json
- server/server.js (or index.js)
- server/schema.graphql (or schema.js)
- server/resolvers.js
- python/train.py
- python/inference.py
- python/requirements.txt
- python/artifacts/fittrack_model.pt (model artifact)
- .env.example

License / Contribution

- Add an appropriate LICENSE file for your project.
- Include CONTRIBUTING.md with guidelines for how to add models, update schema, and test resolvers.

Troubleshooting

- If the Node.js server cannot find the PYTHON_BIN command, set an absolute path to the Python binary in .env (e.g., /usr/bin/python3 or C:\Python39\python.exe).
- If torch installation fails, follow official PyTorch install instructions appropriate to your OS and CUDA/CPU configuration.
- For local testing, provide a small, deterministic model artifact in python/artifacts to speed CI and developer onboarding.