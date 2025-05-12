# MCP Workflow Builder - Technical Specification

This document provides detailed technical specifications for implementing the MCP Workflow Builder POC as outlined in the architecture design.

## 1. API Scraper Function (Azure Function)

### Implementation Details

#### Function Configuration
- **Runtime**: Node.js 18
- **Trigger Type**: HTTP Trigger
- **Authentication**: Function-level authentication

#### Core Components

1. **GitHub Repository Access**
   ```javascript
   const { Octokit } = require('@octokit/rest');
   
   async function cloneRepository(repoUrl, token) {
     const octokit = new Octokit({ auth: token });
     // Extract owner and repo from URL
     const [owner, repo] = extractRepoInfo(repoUrl);
     
     // Get repository contents
     const { data } = await octokit.repos.getContent({
       owner,
       repo,
       path: ''
     });
     
     return processRepositoryContents(data, owner, repo, octokit);
   }
   ```

2. **API Detection Logic**
   ```javascript
   async function detectAPIs(repoContents) {
     // First, look for OpenAPI/Swagger files
     const swaggerFiles = findSwaggerFiles(repoContents);
     
     if (swaggerFiles.length > 0) {
       return parseSwaggerFiles(swaggerFiles);
     }
     
     // If no Swagger files, analyze code to detect APIs
     return analyzeCodeForAPIs(repoContents);
   }
   ```

3. **OpenAI Integration for API Extraction**
   ```javascript
   const { OpenAIClient } = require('@azure/openai');
   
   async function enhanceAPIDefinitions(detectedAPIs) {
     const client = new OpenAIClient(
       process.env.AZURE_OPENAI_ENDPOINT,
       process.env.AZURE_OPENAI_KEY
     );
     
     // Use OpenAI to enhance API descriptions, parameter details, etc.
     const enhancedAPIs = await client.getCompletions({
       deploymentId: process.env.DEPLOYMENT_ID,
       messages: [
         { role: 'system', content: 'You are an API documentation expert.' },
         { role: 'user', content: `Enhance the following API definitions with better descriptions and examples: ${JSON.stringify(detectedAPIs)}` }
       ]
     });
     
     return processEnhancedAPIs(enhancedAPIs);
   }
   ```

4. **OpenAPI/Swagger Generation**
   ```javascript
   const SwaggerParser = require('@apidevtools/swagger-parser');
   
   async function generateOpenAPISpec(apis) {
     const openApiSpec = {
       openapi: '3.0.0',
       info: {
         title: 'Extracted APIs',
         version: '1.0.0',
         description: 'APIs extracted from repository'
       },
       paths: {}
     };
     
     // Add each API to the spec
     apis.forEach(api => {
       openApiSpec.paths[api.path] = {
         [api.method.toLowerCase()]: {
           summary: api.summary,
           description: api.description,
           parameters: api.parameters,
           responses: api.responses
         }
       };
     });
     
     // Validate the spec
     await SwaggerParser.validate(openApiSpec);
     
     return openApiSpec;
   }
   ```

### API Contract

#### Request
```json
{
  "repositoryUrl": "https://github.com/username/repo",
  "accessToken": "github_pat_optional"
}
```

#### Response
```json
{
  "apis": {
    "openapi": "3.0.0",
    "info": {
      "title": "Extracted APIs",
      "version": "1.0.0"
    },
    "paths": {
      "/api/resource": {
        "get": {
          "summary": "Get resource",
          "parameters": [],
          "responses": {
            "200": {
              "description": "Successful response"
            }
          }
        }
      }
    }
  }
}
```

## 2. Canvas UI (React.js)

### Implementation Details

#### Project Setup
```bash
npx create-react-app mcp-workflow-builder-client
cd mcp-workflow-builder-client
npm install @mui/material @emotion/react @emotion/styled
npm install reactflow axios
```

#### Key Components

1. **Main App Structure**
   ```jsx
   // src/App.js
   import React, { useState } from 'react';
   import { Box, Container, CssBaseline, ThemeProvider, createTheme } from '@mui/material';
   import Header from './components/Header';
   import RepositoryForm from './components/RepositoryForm';
   import ApiPanel from './components/ApiPanel';
   import WorkflowCanvas from './components/WorkflowCanvas';
   import ActionPanel from './components/ActionPanel';
   
   const theme = createTheme({
     palette: {
       primary: {
         main: '#1976d2',
       },
       secondary: {
         main: '#dc004e',
       },
     },
   });
   
   function App() {
     const [apis, setApis] = useState([]);
     const [workflow, setWorkflow] = useState({ nodes: [], edges: [] });
     
     const handleApiExtraction = async (repoUrl) => {
       // Call API Scraper Function
       // Update apis state
     };
     
     const handleExportMcp = async () => {
       // Call MCP Generator Function
       // Handle the response
     };
     
     const handleTestMcp = async () => {
       // Call MCP Testing Module
       // Display results
     };
     
     return (
       <ThemeProvider theme={theme}>
         <CssBaseline />
         <Container maxWidth="lg">
           <Header />
           <RepositoryForm onSubmit={handleApiExtraction} />
           <Box display="flex" mt={2}>
             <ApiPanel apis={apis} />
             <WorkflowCanvas workflow={workflow} setWorkflow={setWorkflow} />
           </Box>
           <ActionPanel onExport={handleExportMcp} onTest={handleTestMcp} />
         </Container>
       </ThemeProvider>
     );
   }
   
   export default App;
   ```

2. **Workflow Canvas Component**
   ```jsx
   // src/components/WorkflowCanvas.js
   import React, { useCallback } from 'react';
   import ReactFlow, {
     Background,
     Controls,
     MiniMap,
     addEdge,
     useNodesState,
     useEdgesState,
   } from 'reactflow';
   import 'reactflow/dist/style.css';
   
   function WorkflowCanvas({ workflow, setWorkflow }) {
     const [nodes, setNodes, onNodesChange] = useNodesState(workflow.nodes);
     const [edges, setEdges, onEdgesChange] = useEdgesState(workflow.edges);
     
     const onConnect = useCallback(
       (params) => setEdges((eds) => addEdge(params, eds)),
       [setEdges]
     );
     
     // Update parent workflow state when nodes or edges change
     React.useEffect(() => {
       setWorkflow({ nodes, edges });
     }, [nodes, edges, setWorkflow]);
     
     return (
       <div style={{ height: 500, width: '100%' }}>
         <ReactFlow
           nodes={nodes}
           edges={edges}
           onNodesChange={onNodesChange}
           onEdgesChange={onEdgesChange}
           onConnect={onConnect}
           fitView
         >
           <Background />
           <Controls />
           <MiniMap />
         </ReactFlow>
       </div>
     );
   }
   
   export default WorkflowCanvas;
   ```

3. **API Panel Component**
   ```jsx
   // src/components/ApiPanel.js
   import React from 'react';
   import { Box, List, ListItem, ListItemText, Typography, Paper } from '@mui/material';
   import { DragDropContext, Droppable, Draggable } from 'react-beautiful-dnd';
   
   function ApiPanel({ apis }) {
     const onDragEnd = (result) => {
       // Handle drag end - add to canvas
     };
     
     return (
       <Box width="30%" mr={2}>
         <Paper elevation={2} sx={{ p: 2 }}>
           <Typography variant="h6" gutterBottom>
             API Endpoints
           </Typography>
           <DragDropContext onDragEnd={onDragEnd}>
             <Droppable droppableId="apiList">
               {(provided) => (
                 <List
                   {...provided.droppableProps}
                   ref={provided.innerRef}
                 >
                   {apis.map((api, index) => (
                     <Draggable key={api.id} draggableId={api.id} index={index}>
                       {(provided) => (
                         <ListItem
                           ref={provided.innerRef}
                           {...provided.draggableProps}
                           {...provided.dragHandleProps}
                         >
                           <ListItemText
                             primary={`${api.method} ${api.path}`}
                             secondary={api.summary}
                           />
                         </ListItem>
                       )}
                     </Draggable>
                   ))}
                   {provided.placeholder}
                 </List>
               )}
             </Droppable>
           </DragDropContext>
         </Paper>
       </Box>
     );
   }
   
   export default ApiPanel;
   ```

### API Integration

```javascript
// src/services/api.js
import axios from 'axios';

const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || 'http://localhost:7071/api';

export const extractApis = async (repositoryUrl, accessToken = '') => {
  try {
    const response = await axios.post(`${API_BASE_URL}/extract-apis`, {
      repositoryUrl,
      accessToken
    });
    return response.data;
  } catch (error) {
    console.error('Error extracting APIs:', error);
    throw error;
  }
};

export const generateMcpTool = async (workflow) => {
  try {
    const response = await axios.post(`${API_BASE_URL}/generate-mcp`, {
      workflow
    });
    return response.data;
  } catch (error) {
    console.error('Error generating MCP tool:', error);
    throw error;
  }
};

export const testMcpTool = async (mcpTool) => {
  try {
    const response = await axios.post(`${API_BASE_URL}/test-mcp`, {
      mcpTool
    });
    return response.data;
  } catch (error) {
    console.error('Error testing MCP tool:', error);
    throw error;
  }
};
```

## 3. MCP Generator Function (Azure Function)

### Implementation Details

#### Function Configuration
- **Runtime**: Node.js 18
- **Trigger Type**: HTTP Trigger
- **Authentication**: Function-level authentication

#### Core Components

1. **Workflow Analysis**
   ```javascript
   function analyzeWorkflow(workflow) {
     const { nodes, edges } = workflow;
     
     // Identify start and end nodes
     const startNodes = nodes.filter(node => !edges.some(edge => edge.target === node.id));
     const endNodes = nodes.filter(node => !edges.some(edge => edge.source === node.id));
     
     // Validate workflow structure
     if (startNodes.length !== 1 || endNodes.length !== 1) {
       throw new Error('Workflow must have exactly one start and one end node');
     }
     
     // Build execution path
     return buildExecutionPath(startNodes[0], endNodes[0], nodes, edges);
   }
   ```

2. **MCP Tool Code Generation**
   ```javascript
   const Handlebars = require('handlebars');
   
   function generateMcpToolCode(executionPath, apiDefinitions) {
     // Create input schema from first API
     const inputSchema = generateInputSchema(executionPath[0], apiDefinitions);
     
     // Create output schema from last API
     const outputSchema = generateOutputSchema(executionPath[executionPath.length - 1], apiDefinitions);
     
     // Generate execution code
     const executionCode = generateExecutionCode(executionPath, apiDefinitions);
     
     // Apply to template
     const template = Handlebars.compile(`
       const axios = require('axios');
       
       module.exports = {
         name: "workflow-executor",
         description: "Executes a workflow of API calls",
         inputSchema: {{inputSchema}},
         execute: async function(input) {
           try {
             {{executionCode}}
           } catch (error) {
             throw new Error(\`Workflow execution failed: \${error.message}\`);
           }
         }
       };
     `);
     
     return template({
       inputSchema: JSON.stringify(inputSchema, null, 2),
       executionCode
     });
   }
   ```

3. **OpenAI Integration for Code Optimization**
   ```javascript
   const { OpenAIClient } = require('@azure/openai');
   
   async function optimizeCode(generatedCode) {
     const client = new OpenAIClient(
       process.env.AZURE_OPENAI_ENDPOINT,
       process.env.AZURE_OPENAI_KEY
     );
     
     // Use OpenAI to optimize and improve the generated code
     const optimizedCode = await client.getCompletions({
       deploymentId: process.env.DEPLOYMENT_ID,
       messages: [
         { role: 'system', content: 'You are an expert JavaScript developer.' },
         { role: 'user', content: `Optimize and improve the following MCP tool code:\n\n${generatedCode}` }
       ]
     });
     
     return optimizedCode.choices[0].message.content;
   }
   ```

### API Contract

#### Request
```json
{
  "workflow": {
    "nodes": [
      {
        "id": "1",
        "type": "api",
        "data": { "apiId": "get-users" },
        "position": { "x": 100, "y": 100 }
      },
      {
        "id": "2",
        "type": "api",
        "data": { "apiId": "get-user-details" },
        "position": { "x": 300, "y": 100 }
      }
    ],
    "edges": [
      {
        "id": "e1-2",
        "source": "1",
        "target": "2",
        "data": {
          "mapping": {
            "source": "$.users[0].id",
            "target": "$.userId"
          }
        }
      }
    ]
  },
  "apiDefinitions": {
    "get-users": {
      "path": "/api/users",
      "method": "GET",
      "responses": {
        "200": {
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "users": {
                    "type": "array",
                    "items": {
                      "type": "object",
                      "properties": {
                        "id": { "type": "string" },
                        "name": { "type": "string" }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    },
    "get-user-details": {
      "path": "/api/users/{userId}",
      "method": "GET",
      "parameters": [
        {
          "name": "userId",
          "in": "path",
          "required": true,
          "schema": { "type": "string" }
        }
      ],
      "responses": {
        "200": {
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "id": { "type": "string" },
                  "name": { "type": "string" },
                  "email": { "type": "string" }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

#### Response
```json
{
  "mcpTool": {
    "code": "const axios = require('axios');\n\nmodule.exports = {\n  name: \"workflow-executor\",\n  description: \"Executes a workflow of API calls\",\n  inputSchema: {\n    \"type\": \"object\",\n    \"properties\": {}\n  },\n  execute: async function(input) {\n    try {\n      // API call 1: Get users\n      const response1 = await axios.get('/api/users');\n      const users = response1.data.users;\n      \n      // API call 2: Get user details\n      const userId = users[0].id;\n      const response2 = await axios.get(`/api/users/${userId}`);\n      \n      return response2.data;\n    } catch (error) {\n      throw new Error(`Workflow execution failed: ${error.message}`);\n    }\n  }\n};"
  }
}
```

## 4. MCP Testing Module (Node.js)

### Implementation Details

#### Setup
```bash
mkdir mcp-testing-module
cd mcp-testing-module
npm init -y
npm install express body-parser
```

#### Core Components

1. **MCP Server Simulator**
   ```javascript
   // server.js
   const express = require('express');
   const bodyParser = require('body-parser');
   const fs = require('fs');
   const path = require('path');
   const { execSync } = require('child_process');
   
   const app = express();
   app.use(bodyParser.json());
   
   // Temporary directory for MCP tools
   const TEMP_DIR = path.join(__dirname, 'temp');
   if (!fs.existsSync(TEMP_DIR)) {
     fs.mkdirSync(TEMP_DIR);
   }
   
   app.post('/test-mcp', async (req, res) => {
     try {
       const { mcpTool } = req.body;
       
       // Create temporary MCP tool file
       const toolId = `tool-${Date.now()}`;
       const toolPath = path.join(TEMP_DIR, `${toolId}.js`);
       fs.writeFileSync(toolPath, mcpTool.code);
       
       // Create test server
       const serverPath = path.join(TEMP_DIR, `${toolId}-server.js`);
       fs.writeFileSync(serverPath, generateServerCode(toolId));
       
       // Start server
       const serverProcess = execSync(`node ${serverPath}`, { detached: true });
       
       // Test the tool
       const testResults = await testTool(toolId);
       
       // Clean up
       cleanUp(toolId);
       
       res.json({ success: true, results: testResults });
     } catch (error) {
       res.status(500).json({ success: false, error: error.message });
     }
   });
   
   function generateServerCode(toolId) {
     return `
       const express = require('express');
       const bodyParser = require('body-parser');
       const tool = require('./${toolId}.js');
       
       const app = express();
       app.use(bodyParser.json());
       
       app.post('/execute', async (req, res) => {
         try {
           const result = await tool.execute(req.body);
           res.json(result);
         } catch (error) {
           res.status(500).json({ error: error.message });
         }
       });
       
       const PORT = 3001;
       app.listen(PORT, () => {
         console.log(\`MCP tool server running on port \${PORT}\`);
       });
     `;
   }
   
   async function testTool(toolId) {
     // Implement test logic here
     // Send requests to the tool server
     // Validate responses
     // Return test results
   }
   
   function cleanUp(toolId) {
     // Remove temporary files
     fs.unlinkSync(path.join(TEMP_DIR, `${toolId}.js`));
     fs.unlinkSync(path.join(TEMP_DIR, `${toolId}-server.js`));
   }
   
   const PORT = 3000;
   app.listen(PORT, () => {
     console.log(`MCP testing server running on port ${PORT}`);
   });
   ```

### API Contract

#### Request
```json
{
  "mcpTool": {
    "code": "const axios = require('axios');\n\nmodule.exports = {\n  name: \"workflow-executor\",\n  description: \"Executes a workflow of API calls\",\n  inputSchema: {\n    \"type\": \"object\",\n    \"properties\": {}\n  },\n  execute: async function(input) {\n    try {\n      // API call 1: Get users\n      const response1 = await axios.get('/api/users');\n      const users = response1.data.users;\n      \n      // API call 2: Get user details\n      const userId = users[0].id;\n      const response2 = await axios.get(`/api/users/${userId}`);\n      \n      return response2.data;\n    } catch (error) {\n      throw new Error(`Workflow execution failed: ${error.message}`);\n    }\n  }\n};"
  }
}
```

#### Response
```json
{
  "success": true,
  "results": {
    "status": "passed",
    "executionTime": 120,
    "output": {
      "id": "user123",
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

## 5. API Storage (Local JSON/SQLite)

### Implementation Details

For simplicity, we'll use a local JSON file for storage in the POC. In a production environment, this would be replaced with a proper database.

```javascript
// storage.js
const fs = require('fs');
const path = require('path');

const STORAGE_DIR = path.join(__dirname, 'storage');
const API_FILE = path.join(STORAGE_DIR, 'apis.json');

// Ensure storage directory exists
if (!fs.existsSync(STORAGE_DIR)) {
  fs.mkdirSync(STORAGE_DIR, { recursive: true });
}

// Initialize empty APIs file if it doesn't exist
if (!fs.existsSync(API_FILE)) {
  fs.writeFileSync(API_FILE, JSON.stringify({ apis: {} }));
}

function getApis() {
  const data = fs.readFileSync(API_FILE, 'utf8');
  return JSON.parse(data).apis;
}

function saveApi(id, apiDefinition) {
  const data = fs.readFileSync(API_FILE, 'utf8');
  const json = JSON.parse(data);
  
  json.apis[id] = apiDefinition;
  
  fs.writeFileSync(API_FILE, JSON.stringify(json, null, 2));
  return apiDefinition;
}

function deleteApi(id) {
  const data = fs.readFileSync(API_FILE, 'utf8');
  const json = JSON.parse(data);
  
  if (json.apis[id]) {
    delete json.apis[id];
    fs.writeFileSync(API_FILE, JSON.stringify(json, null, 2));
    return true;
  }
  
  return false;
}

module.exports = {
  getApis,
  saveApi,
  deleteApi
};
```

## Deployment Instructions

### Azure Functions Setup

1. **Create Azure Function App**
   ```bash
   az login
   az group create --name mcp-workflow-builder-rg --location eastus
   az storage account create --name mcpworkflowstorage --location eastus --resource-group mcp-workflow-builder-rg --sku Standard_LRS
   az functionapp create --name mcp-workflow-builder-functions --storage-account mcpworkflowstorage --consumption-plan-location eastus --resource-group mcp-workflow-builder-rg --runtime node --runtime-version 18 --functions-version 4
   ```

2. **Configure Azure OpenAI**
   ```bash
   az cognitiveservices account create --name mcp-workflow-openai --resource-group mcp-workflow-builder-rg --kind OpenAI --sku S0 --location eastus
   az cognitiveservices account deployment create --name mcp-workflow-openai --resource-group mcp-workflow-builder-rg --deployment-name gpt-4 --model-name gpt-4 --model-version 0613
   ```

3. **Set Function App Settings**
   ```bash
   az functionapp config appsettings set --name mcp-workflow-builder-functions --resource-group mcp-workflow-builder-rg --settings "AZURE_OPENAI_ENDPOINT=https://mcp-workflow-openai.openai.azure.com/" "AZURE_OPENAI_KEY=your-key-here" "DEPLOYMENT_ID=gpt-4"
   ```

### Local Development Setup

1. **Frontend Setup**
   ```bash
   cd client
   npm install
   # Create .env file
   echo "REACT_APP_API_BASE_URL=http://localhost:7071/api" > .env
   npm start
   ```

2. **Azure Functions Local Development**
   ```bash
   cd server
   npm install
   # Create local.settings.json
   echo '{
     "IsEncrypted": false,
     "Values": {
       "FUNCTIONS_WORKER_RUNTIME": "node",
       "AzureWebJobsStorage": "UseDevelopmentStorage=true",
       "AZURE_OPENAI_ENDPOINT": "https://mcp-workflow-openai.openai.azure.com/",
       "AZURE_OPENAI_KEY": "your-key-here",
       "DEPLOYMENT_ID": "gpt-4"
     }
   }' > local.settings.json
   npm start
   ```

3. **MCP Testing Module Setup**
   ```bash
   cd mcp-testing-module
   npm install
   node server.js
   ```

## Testing Strategy

1. **Unit Testing**
   - Test individual components in isolation
   - Use Jest for JavaScript testing
   - Mock external dependencies

2. **Integration Testing**
   - Test API Scraper with real GitHub repositories
   - Test MCP Generator with sample workflows
   - Test MCP Testing Module with generated tools

3. **End-to-End Testing**
   - Test complete workflow from GitHub URL to MCP tool generation and testing
   - Verify UI interactions and data flow

## Security Considerations

1. **GitHub Token Handling**
   - Store tokens securely
   - Use minimal required permissions
   - Implement token rotation

2. **API Credentials**
   - Do not hardcode credentials in generated MCP tools
   - Use environment variables or secure credential storage

3. **Input Validation**
   - Validate all inputs to prevent injection attacks
   - Sanitize GitHub URLs and API definitions

4. **MCP Tool Isolation**
   - Run MCP tools in isolated environments
   - Implement timeouts and resource limits