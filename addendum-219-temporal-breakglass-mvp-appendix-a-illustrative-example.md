# Addendum 219 — Appendix A: illustrative example supplied with the prompt (NOT our domain)

> **READ THIS FIRST — ILLUSTRATIVE ONLY.**
> This appendix reproduces, verbatim, an example guide the user supplies alongside the implementation
> prompt (§19). It describes a **procurement** proof-of-concept (budget, hardware, software). It is
> **not related to our problem** and must **not** be built, seeded or copied:
> - Do not create procurement workflows, `budget` / `total_spend` variables, or the
>   `PROCURE_HARDWARE` / `PROCURE_SOFTWARE` nodes.
> - Do not copy its code. It contains defects listed in §19.1 (part 3): `tokio::select!` in workflow
>   code, `Rc<RefCell<…>>` shared state, `make_signal_channel`, a fixed workflow ID,
>   `CorsLayer::permissive()`, `unwrap()` everywhere, signals for human actions, hard-coded React Flow
>   nodes, all state in workflow memory.
>
> **What it is for.** It illustrates *engine features* our MVP must also demonstrate, on our own
> break-glass domain. The mapping is normative and lives in part 4 §25.3:
>
> | Appendix A feature | Our equivalent |
> |---|---|
> | INIT: seed variables | `wf_variables` init and system values (§25.1, AC-18.1.1) |
> | FORK with branch signals mutating `total_spend` | `split_review` with reviewer fields; `add_sec` / `add_log` mutating `review_score`; lanes mutating `items_received` (AC-19.1.1, AC-19.1.3, AC-20.3.1) |
> | MERGE waiting for both branches | `join_review` (all; k-of-n in v2) and the per-item region join (AC-19.1.2, AC-19.2.1) |
> | CONDITION `left_var operator right_var`, operator configurable | `rule` / `switch` with `cmp(left, op, right)`, operator chosen in the editor (§21.4, AC-18.3.1, AC-18.3.2) |
> | APPROVED / REJECTED | `score_ok` true → owner; false → `rejected`; every approval's `reject` port (AC-24.2.1) |
> | COMPLETED | `join_done` → `completed` (AC-20.4.1) |
> | React Flow live view (blue active, green done) plus variables panel | Request graph from `/plan` and the Variables panel (AC-20.5.1); editor simulation animation (§24.3) |
>
> Everything below the line is the user's text, unedited.

---

# **Advanced Implementation Guide: End-to-End Workflows with Variables, Fork/Join, and Rule Evaluation**

This comprehensive guide details the implementation of a complex workflow Proof of Concept (POC). It demonstrates the complete lifecycle of workflow state: initializing variables, mutating them across parallel branching (fork) and merging (join) paths, and finally routing the workflow via configurable operations that evaluate those variables.

## **1\. The End-to-End Workflow Architecture**

The workflow models a dynamic procurement approval process:

> 1. **INIT (Initialization):** Variables are seeded (e.g., budget \= 5000, total\_spend \= 0).  
> 2. **FORK (Parallel Paths):** The workflow branches into PROCURE\_HARDWARE and PROCURE\_SOFTWARE. Each branch accepts external signals that mutate the total\_spend variable independently.  
> 3. **MERGE:** The workflow waits until both parallel procurements are complete before moving forward.  
> 4. **CONDITION (Rule Evaluation):** A configurable rule (e.g., total\_spend \<= budget) is evaluated dynamically using the manipulated variables.  
> 5. **APPROVED / REJECTED:** The graph branches based on the rule engine's boolean result.  
> 6. **COMPLETED:** Final termination of the workflow.

## **2\. Rust Backend: State, Variables, and Parallel Execution**

The Rust worker uses a HashMap to store variables, tokio::select\! to handle parallel mutations safely, and a custom evaluation function to process dynamic conditional rules.

### **backend/src/main.rs**

use axum::{extract::{State, Path}, routing::{get, post}, Json, Router};  
use serde::{Deserialize, Serialize};  
use std::{sync::Arc, rc::Rc, cell::RefCell, collections::HashMap};  
use temporalio::{client::{Client, ClientOptions, WorkflowOptions}, sdk::{Worker, WorkflowContext, WorkflowResult}};  
use temporalio::sdk::workflow;  
use tower\_http::cors::CorsLayer;

// \--- 1\. DATA STRUCTURES \---  
\#\[derive(Serialize, Deserialize, Clone, Debug)\]  
pub struct AppState {   
    pub active\_nodes: Vec\<String\>,  
    pub completed\_nodes: Vec\<String\>,  
    pub variables: HashMap\<String, i32\>,  
}

\#\[derive(Serialize, Deserialize, Clone, Debug)\]  
pub struct Rule {  
    pub left\_var: String,  
    pub operator: String,  
    pub right\_var: String,  
}

\#\[derive(Serialize, Deserialize, Clone, Debug)\]  
pub struct CostPayload {  
    pub amount: i32,  
}

// Helper function to evaluate dynamic rules across variables  
fn evaluate\_rule(vars: \&HashMap\<String, i32\>, rule: \&Rule) \-\> bool {  
    let left \= vars.get(\&rule.left\_var).copied().unwrap\_or(0);  
    let right \= vars.get(\&rule.right\_var).copied().unwrap\_or(0);  
    match rule.operator.as\_str() {  
        "\>" \=\> left \> right,  
        "\<" \=\> left \< right,  
        "\>=" \=\> left \>= right,  
        "\<=" \=\> left \<= right,  
        "==" \=\> left \== right,  
        \_ \=\> false,  
    }  
}

// \--- 2\. TEMPORAL WORKFLOW DEFINITION \---  
\#\[temporalio::workflow\]  
pub struct ComplexWorkflow;

\#\[temporalio::workflow\_methods\]  
impl ComplexWorkflow {  
    \#\[workflow::run\]  
    pub async fn run(self, mut ctx: WorkflowContext) \-\> WorkflowResult\<()\> {  
        // Initializing variables  
        let mut initial\_vars \= HashMap::new();  
        initial\_vars.insert("budget".to\_string(), 5000);  
        initial\_vars.insert("total\_spend".to\_string(), 0);

        let state \= Rc::new(RefCell::new(AppState {   
            active\_nodes: vec\!\["INIT".to\_string()\],   
            completed\_nodes: vec\!\[\],  
            variables: initial\_vars,  
        }));  
          
        let mut advance\_sig \= ctx.make\_signal\_channel::\<()\>("advance");  
        let mut hw\_sig \= ctx.make\_signal\_channel::\<CostPayload\>("procure-hw");  
        let mut sw\_sig \= ctx.make\_signal\_channel::\<CostPayload\>("procure-sw");  
        let mut eval\_sig \= ctx.make\_signal\_channel::\<Rule\>("evaluate-rule");  
          
        let query\_state \= state.clone();  
        ctx.set\_query\_handler("get-state", move \|\| Ok(query\_state.borrow().clone()));

        // Step 1: Wait to fork  
        advance\_sig.recv().await;  
        {  
            let mut s \= state.borrow\_mut();  
            s.completed\_nodes.push("INIT".to\_string());  
            s.active\_nodes \= vec\!\["PROCURE\_HARDWARE".to\_string(), "PROCURE\_SOFTWARE".to\_string()\];  
        }

        // Step 2: Parallel Branching (Fork) & Variable Manipulation  
        let mut hw\_done \= false;  
        let mut sw\_done \= false;

        while \!(hw\_done && sw\_done) {  
            tokio::select\! {  
                Some(payload) \= hw\_sig.recv(), if \!hw\_done \=\> {  
                    hw\_done \= true;  
                    let mut s \= state.borrow\_mut();  
                    let spend \= s.variables.entry("total\_spend".to\_string()).or\_insert(0);  
                    \*spend \+= payload.amount; // Manipulate variable  
                    s.active\_nodes.retain(\|x\| x \!= "PROCURE\_HARDWARE");  
                    s.completed\_nodes.push("PROCURE\_HARDWARE".to\_string());  
                },  
                Some(payload) \= sw\_sig.recv(), if \!sw\_done \=\> {  
                    sw\_done \= true;  
                    let mut s \= state.borrow\_mut();  
                    let spend \= s.variables.entry("total\_spend".to\_string()).or\_insert(0);  
                    \*spend \+= payload.amount; // Manipulate variable  
                    s.active\_nodes.retain(\|x\| x \!= "PROCURE\_SOFTWARE");  
                    s.completed\_nodes.push("PROCURE\_SOFTWARE".to\_string());  
                }  
            }  
        }

        // Step 3: Merge back together  
        state.borrow\_mut().active\_nodes \= vec\!\["MERGED\_EVALUATION".to\_string()\];

        // Step 4: Condition Evaluation using configurable operations  
        if let Some(rule) \= eval\_sig.recv().await {  
            let is\_approved \= evaluate\_rule(\&state.borrow().variables, \&rule);  
              
            let mut s \= state.borrow\_mut();  
            s.completed\_nodes.push("MERGED\_EVALUATION".to\_string());  
            if is\_approved {  
                s.active\_nodes \= vec\!\["APPROVED".to\_string()\];  
            } else {  
                s.active\_nodes \= vec\!\["REJECTED".to\_string()\];  
            }  
        }

        // Step 5: Final completion  
        advance\_sig.recv().await;  
        {  
            let mut s \= state.borrow\_mut();  
            if s.active\_nodes.contains(&"APPROVED".to\_string()) {  
                s.completed\_nodes.push("APPROVED".to\_string());  
            } else {  
                s.completed\_nodes.push("REJECTED".to\_string());  
            }  
            s.active\_nodes \= vec\!\["COMPLETED".to\_string()\];  
        }

        Ok(())  
    }  
}

// \--- 3\. API SERVER \---  
\#\[derive(Clone)\]  
struct ApiContext { temporal\_client: Arc\<Client\> }

\#\[tokio::main\]  
async fn main() \-\> Result\<(), Box\<dyn std::error::Error\>\> {  
    let client \= Arc::new(Client::connect(ClientOptions::default()).await?);  
      
    let worker\_client \= client.clone();  
    tokio::spawn(async move {  
        let mut worker \= Worker::new(worker\_client, "complex-queue");  
        worker.register\_workflow(ComplexWorkflow);  
        worker.run().await.unwrap();  
    });

    let app \= Router::new()  
        .route("/api/start", post(start\_workflow))  
        .route("/api/state", get(get\_state))  
        .route("/api/signal/:action", post(send\_simple\_signal))  
        .route("/api/cost/:action", post(send\_cost\_signal))  
        .route("/api/evaluate", post(send\_eval\_signal))  
        .layer(CorsLayer::permissive())  
        .with\_state(ApiContext { temporal\_client: client });

    let listener \= tokio::net::TcpListener::bind("0.0.0.0:3001").await?;  
    println\!("Backend running on http\://localhost:3001");  
    axum::serve(listener, app).await?;  
    Ok(())  
}

async fn start\_workflow(State(ctx): State\<ApiContext\>) \-\> Json\<String\> {  
    let opts \= WorkflowOptions {  
        id: "poc-complex-1".to\_string(),  
        task\_queue: "complex-queue".to\_string(),  
        ..Default::default()  
    };  
    ctx.temporal\_client.start\_workflow(ComplexWorkflow, (), opts).await.unwrap();  
    Json("Started".to\_string())  
}

async fn get\_state(State(ctx): State\<ApiContext\>) \-\> Json\<AppState\> {  
    let handle \= ctx.temporal\_client.get\_workflow\_handle("poc-complex-1", None);  
    let default\_state \= AppState { active\_nodes: vec\!\[\], completed\_nodes: vec\!\[\], variables: HashMap::new() };  
    let state: AppState \= handle.query("get-state", ()).await.unwrap\_or(default\_state);  
    Json(state)  
}

async fn send\_simple\_signal(State(ctx): State\<ApiContext\>, Path(action): Path\<String\>) \-\> Json\<String\> {  
    let handle \= ctx.temporal\_client.get\_workflow\_handle("poc-complex-1", None);  
    handle.signal(action.as\_str(), ()).await.unwrap();  
    Json("Signaled".to\_string())  
}

async fn send\_cost\_signal(State(ctx): State\<ApiContext\>, Path(action): Path\<String\>, Json(payload): Json\<CostPayload\>) \-\> Json\<String\> {  
    let handle \= ctx.temporal\_client.get\_workflow\_handle("poc-complex-1", None);  
    handle.signal(action.as\_str(), payload).await.unwrap();  
    Json("Cost Signaled".to\_string())  
}

async fn send\_eval\_signal(State(ctx): State\<ApiContext\>, Json(payload): Json\<Rule\>) \-\> Json\<String\> {  
    let handle \= ctx.temporal\_client.get\_workflow\_handle("poc-complex-1", None);  
    handle.signal("evaluate-rule", payload).await.unwrap();  
    Json("Rule Evaluated".to\_string())  
}

## **3\. React Flow Frontend: Visualizing Variables and Logic**

The frontend visualizes the fork and merge structure, alongside conditional branches. It also includes forms to inject variables and configure the operation rules.

### **frontend/src/App.tsx**

import { useState, useEffect } from 'react';  
import { ReactFlow, Background, Controls, Node, Edge } from '@xyflow/react';  
import '@xyflow/react/dist/style.css';  
import axios from 'axios';

const API\_BASE \= 'http\://localhost:3001/api';

const initialNodes: Node\[\] \= \[  
  { id: 'INIT', position: { x: 250, y: 0 }, data: { label: '1. Init Variables' }, style: {} },  
  { id: 'PROCURE\_HARDWARE', position: { x: 50, y: 100 }, data: { label: '2A. Procure HW' }, style: {} },  
  { id: 'PROCURE\_SOFTWARE', position: { x: 450, y: 100 }, data: { label: '2B. Procure SW' }, style: {} },  
  { id: 'MERGED\_EVALUATION', position: { x: 250, y: 200 }, data: { label: '3. Condition Eval (Merge)' }, style: {} },  
  { id: 'APPROVED', position: { x: 100, y: 300 }, data: { label: '4A. Approved' }, style: {} },  
  { id: 'REJECTED', position: { x: 400, y: 300 }, data: { label: '4B. Rejected' }, style: {} },  
  { id: 'COMPLETED', position: { x: 250, y: 400 }, data: { label: '5. Completed' }, style: {} },  
\];

const initialEdges: Edge\[\] \= \[  
  { id: 'e1-2a', source: 'INIT', target: 'PROCURE\_HARDWARE', animated: true },  
  { id: 'e1-2b', source: 'INIT', target: 'PROCURE\_SOFTWARE', animated: true },  
  { id: 'e2a-3', source: 'PROCURE\_HARDWARE', target: 'MERGED\_EVALUATION', animated: true },  
  { id: 'e2b-3', source: 'PROCURE\_SOFTWARE', target: 'MERGED\_EVALUATION', animated: true },  
  { id: 'e3-4a', source: 'MERGED\_EVALUATION', target: 'APPROVED', animated: true, label: 'If Condition Met' },  
  { id: 'e3-4b', source: 'MERGED\_EVALUATION', target: 'REJECTED', animated: true, label: 'If Condition Fails' },  
  { id: 'e4a-5', source: 'APPROVED', target: 'COMPLETED', animated: true },  
  { id: 'e4b-5', source: 'REJECTED', target: 'COMPLETED', animated: true },  
\];

export default function App() {  
  const \[nodes, setNodes\] \= useState(initialNodes);  
  const \[workflowState, setWorkflowState\] \= useState({ active\_nodes: \[\], completed\_nodes: \[\], variables: {} });  
    
  // Inputs  
  const \[hwCost, setHwCost\] \= useState(1500);  
  const \[swCost, setSwCost\] \= useState(2000);  
  const \[ruleOp, setRuleOp\] \= useState("\<=");

  // Poll Backend State  
  useEffect(() \=\> {  
    const fetchState \= async () \=\> {  
      try {  
        const res \= await axios.get(\`\${API\_BASE}/state\`);  
        const state \= res.data;  
        setWorkflowState(state);

        setNodes((nds) \=\>  
          nds.map((node) \=\> {  
            const isActive \= state.active\_nodes.includes(node.id);  
            const isCompleted \= state.completed\_nodes.includes(node.id);  
            let border \= '1px solid \#222';  
            let bg \= '\#fff';  
            let shadow \= 'none';

            if (isActive) {  
              border \= '3px solid \#3b82f6';   
              bg \= '\#bfdbfe';  
              shadow \= '0 0 10px rgba(59, 130, 246, 0.5)';  
            } else if (isCompleted) {  
              border \= '3px solid \#10b981';   
              bg \= '\#d1fae5';  
            }  
            return { ...node, style: { ...node.style, border, backgroundColor: bg, boxShadow: shadow } };  
          })  
        );  
      } catch (e) {  
        // Workflow not running  
      }  
    };  
    const interval \= setInterval(fetchState, 1500);  
    return () \=\> clearInterval(interval);  
  }, \[\]);

  const isActive \= (nodeId: string) \=\> workflowState.active\_nodes.includes(nodeId);  
  const startWorkflow \= () \=\> axios.post(\`\${API\_BASE}/start\`);  
  const advanceStandard \= () \=\> axios.post(\`\${API\_BASE}/signal/advance\`);  
    
  const submitCost \= (action: string, amount: number) \=\> {  
    axios.post(\`\${API\_BASE}/cost/\${action}\`, { amount: Number(amount) });  
  };

  const evaluateRule \= () \=\> {  
    // Configurable operation mapping Left Var (total\_spend) against Right Var (budget)  
    axios.post(\`\${API\_BASE}/evaluate\`, {  
      left\_var: "total\_spend",  
      operator: ruleOp,  
      right\_var: "budget"  
    });  
  };

  return (  
    \<div style={{ height: '100vh', display: 'flex', flexDirection: 'column' }}\>  
      \<div style={{ padding: '20px', background: '\#f8f9fa', borderBottom: '1px solid \#ccc', display: 'flex', flexDirection: 'column', gap: '15px' }}\>  
          
        \<div style={{ display: 'flex', gap: '20px', background: '\#333', color: '\#fff', padding: '10px', borderRadius: '5px' }}\>  
          \<strong\>Engine Variables:\</strong\>  
          \<span\>Budget: {workflowState.variables?.budget ?? 'N/A'}\</span\>  
          \<span\>Total Spend: {workflowState.variables?.total\_spend ?? 'N/A'}\</span\>  
        \</div\>

        \<div style={{ display: 'flex', gap: '15px', flexWrap: 'wrap', alignItems: 'center' }}\>  
          {workflowState.active\_nodes.length \=== 0 && \<button onClick={startWorkflow}\>1. Initialize Workflow\</button\>}  
          {isActive('INIT') && \<button onClick={advanceStandard}\>Fork Branches\</button\>}  
            
          {isActive('PROCURE\_HARDWARE') && (  
            \<div style={{ border: '1px solid \#ccc', padding: '10px', borderRadius: '5px' }}\>  
              \<strong\>Hardware:\</strong\>   
              \<input type="number" value={hwCost} onChange={e \=\> setHwCost(e.target.value)} style={{ width: '80px', margin: '0 10px' }}/\>  
              \<button onClick={() \=\> submitCost('procure-hw', hwCost)}\>Purchase HW\</button\>  
            \</div\>  
          )}  
            
          {isActive('PROCURE\_SOFTWARE') && (  
            \<div style={{ border: '1px solid \#ccc', padding: '10px', borderRadius: '5px' }}\>  
              \<strong\>Software:\</strong\>   
              \<input type="number" value={swCost} onChange={e \=\> setSwCost(e.target.value)} style={{ width: '80px', margin: '0 10px' }}/\>  
              \<button onClick={() \=\> submitCost('procure-sw', swCost)}\>Purchase SW\</button\>  
            \</div\>  
          )}

          {isActive('MERGED\_EVALUATION') && (  
            \<div style={{ border: '1px solid \#ccc', padding: '10px', borderRadius: '5px', background: '\#fff3cd' }}\>  
              \<strong\>Rule Eval:\</strong\> (total\_spend   
              \<select value={ruleOp} onChange={e \=\> setRuleOp(e.target.value)} style={{ margin: '0 10px' }}\>  
                \<option value="\<"\>\<\</option\>  
                \<option value="\<="\>\<=\</option\>  
                \<option value="\>"\>\>\</option\>  
                \<option value="\>="\>\>=\</option\>  
                \<option value="=="\>==\</option\>  
              \</select\>  
              budget)   
              \<button onClick={evaluateRule} style={{ marginLeft: '10px', background: '\#0d6efd', color: '\#fff' }}\>Evaluate & Route\</button\>  
            \</div\>  
          )}

          {(isActive('APPROVED') \|\| isActive('REJECTED')) && \<button onClick={advanceStandard}\>Close Case\</button\>}  
        \</div\>  
      \</div\>  
        
      \<div style={{ flex: 1 }}\>  
        \<ReactFlow nodes={nodes} edges={initialEdges} fitView\>  
          \<Background /\>  
          \<Controls /\>  
        \</ReactFlow\>  
      \</div\>  
    \</div\>  
  );  
}
