# VisionPaws.ai

🦁 An intelligent animal recognition system powered by Dify AI and Vue 3.

## System Architecture

```mermaid
flowchart LR
    User([User]) --> Frontend[Vue3 Frontend]
    Frontend --> UploadAPI[Dify Upload API]
    Frontend --> WorkflowAPI[Dify Workflow API]

    subgraph Dify[Dify AI Platform]
        UploadAPI --> FileStorage[(File Storage)]
        FileStorage --> WorkflowAPI
        WorkflowAPI --> AIModel[AI Model]
        AIModel --> Analysis[Analysis Result]
    end

    Analysis --> Frontend
    Frontend --> Display[Display Result]
```

## Image Recognition Flow

```mermaid
sequenceDiagram
    actor User
    participant Frontend as Vue3 Frontend
    participant Upload as Dify Upload API
    participant Workflow as Dify Workflow API
    participant AI as AI Model

    User->>Frontend: Upload Image
    Frontend->>Upload: Send Image File
    Upload-->>Frontend: Return file_id
    Frontend->>Workflow: Send file_id
    Workflow->>AI: Process Image
    AI-->>Workflow: Analysis Results
    Workflow-->>Frontend: Return Analysis
    Frontend-->>User: Display Results
```

## Data Analysis Flow

```mermaid
flowchart TD
    A[Upload Image] --> B[Image Processing]
    B --> C{Analysis Steps}
    C --> D[Basic Information]
    C --> E[Educational Points]
    C --> F[Fun Facts]

    D --> G[Display Results]
    E --> G
    F --> G

    subgraph Analysis Content
    D --> |Contains| D1[Species Name]
    D --> |Contains| D2[Key Features]
    D --> |Contains| D3[Appearance]

    E --> |Contains| E1[Habitat]
    E --> |Contains| E2[Behavior]
    E --> |Contains| E3[Role in Ecosystem]

    F --> |Contains| F1[Adaptations]
    F --> |Contains| F2[Living Habits]
    F --> |Contains| F3[Cultural Stories]
    end
```

## Features

- Instant animal image recognition
- Detailed species analysis
- Educational insights
- Cultural and behavioral information

## Tech Stack

- Vue 3
- PrimeVue
- Dify AI API
- Vite

## Getting Started

1. Clone the repository
2. Install dependencies: `npm install`
3. Set up environment variables
4. Run development server: `npm run dev`

## Environment Variables

```env
VITE_DIFY_API_KEY=your-api-key
VITE_DIFY_UPLOAD_ENDPOINT=your-upload-endpoint
VITE_DIFY_WORKFLOW_ENDPOINT=your-workflow-endpoint
```
