Jenkins pipeline where each stage uses a different Docker-based agent/image like

Checkout → Git agent
Build → Maven agent
Frontend → Node.js agent
Test → Python agent
Deploy → AWS CLI agent

Pipeline architecture

                                                Jenkins Pipeline
                                                    │
                                                agent none
                                                    │
        ┌───────────────────────────────────────────┼───────────────────────────────────────────┐
        │                                           │                                           │
        ▼                                           ▼                                           ▼
   Checkout, Git Agent, alpine/git         Backend, Maven Agent, Maven + JDK           Frontend,Node Agent, Node.js
        │                                           │                                           │
        └───────────────────────────────────────────┼───────────────────────────────────────────┘
                                                    │
                                                    ▼
                                                Testing
                                                Python Agent
                                                    │
                                                    ▼
                                                Deployment
                                                AWS CLI Agent





| Configuration                           | Meaning                                                      |
| --------------------------------------- | ------------------------------------------------------------ |
| `agent none`                            | No global agent; each stage chooses its own                  |
| `agent { docker { image 'maven...' } }` | Run stage inside a Maven Docker container                    |
| `reuseNode true`                        | Use the existing Jenkins node/workspace for the Docker stage |
| `reuseNode false` / omitted             | Jenkins can use a new workspace for the Docker stage         |
