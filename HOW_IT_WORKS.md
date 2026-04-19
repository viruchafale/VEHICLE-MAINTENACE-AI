# ⚙️ How It Works: System Architecture & Workflow

Here is a detailed, step-by-step breakdown of exactly how the **Vehicle Maintenance AI** project works under the hood, from the moment a user inputs data to the final AI-generated outputs.

---

### Step 1: The UI and Data Intake (Frontend)
The application starts in **Streamlit** (`app.py`), serving as the interactive dashboard. When a user navigates to the **🚘 Single Vehicle** tab, they fill out a form containing information across three categories:
* **Vehicle Profile:** Model, Age, Engine size, Fuel type.
* **Usage & Service:** Mileage, Odometer, Days since last service, Reported issues.
* **Components & Cost:** Tire condition, Brake condition, Battery status.

Once the user clicks **"Predict Maintenance Need"**, this raw data is bundled into a Pandas DataFrame.

---

### Step 2: The Machine Learning Prediction (The "Deterministic" phase)
Before the generative AI gets involved, we use a traditional Machine Learning pipeline to anchor our system in real statistics. 
1. The raw DataFrame is passed through a **Preprocessor** (`preprocessor.pkl`), which scales numbers and encodes text (e.g., converting "Worn Out" brakes into numbers the model understands).
2. It is then fed into the pre-trained **Scikit-Learn Classifier** (`maintenance_model.pkl`).
3. The model outputs a raw probability—a **Risk Score between 0.0 and 1.0**. 
   * If the score is `< 0.50`, the UI tags the vehicle as ✅ *Routine / Low Risk*.
   * If `>= 0.50`, the vehicle is flagged as ⚠️ *High Risk*.

*This Risk Score and the vehicle data are then locked into the session's memory to give the AI context.*

---

### Step 3: The AI Agent Workflows (The "Generative" phase)
When the user switches to the **🤖 AI Copilot** tab, they are interacting with **LangGraph**—a framework designed to build stateful, multi-actor AI loops. There are four distinct Agent patterns operating here:

#### A. 🩺 Health Report (Pattern: Conditional Routing)
When you click "Generate Health Report", the AI doesn't just write a generic response. It uses a **Triage Agent**.
1. **Triage:** The LLM looks at the ML Risk Score and decides which path to take.
2. **Low-Risk Path:** If the vehicle is fine, the graph routes to a lightweight node that generates a quick, standard maintenance summary to save compute power and time.
3. **High-Risk Path:** If the vehicle is in danger, the graph routes to a heavy-duty **RAG (Retrieval-Augmented Generation)** node. The LLM queries your local **ChromaDB** vector database to pull actual vehicular repair manuals or historical context, synthesizing a detailed, critical action report.

#### B. 📅 Schedule Planner (Pattern: Human-in-the-Loop)
AI shouldn't single-handedly order expensive repairs based purely on generative assumptions. This agent requires human oversight.
1. The LLM drafts a 90-day maintenance timeline based on the vehicle's state and risk.
2. The LangGraph is programmed to execute an **`interrupt`**. The AI literally *pauses* its own background graph and hands the draft schedule back to the Streamlit UI as "Pending".
3. The human manager reads it, adds notes, and clicks **Approve** or **Modify & Approve**.
4. The graph captures this human command, *resumes* its process contextually, and produces the finalized schedule.

#### C. 💬 AI Chat (Pattern: ReAct Tool-Calling)
The chat interface isn't just a generic ChatGPT wrapper; it's an autonomous tool-calling loop (`run_chat_agent`).
1. You ask a question: *"How much will it cost to fix these brakes and is it urgent?"*
2. The LLM enters a "Reasoning" phase. It realizes it needs accurate pricing and urgency metrics.
3. It autonomously selects and executes predefined Python functions (**Tools**)—such as a `Cost Estimator` tool and an `Urgency Checker` tool.
4. It reads the raw output of those tools, synthesizes the information, and finally responds to you in the chat UI. 

---

### Step 4: The Fleet Overview (Macro Analysis)
In the final **📡 Fleet Overview** tab, we shift from micro to macro tracking. 
1. Streamlit simulates and loads aggregate data for hundreds of vehicles (e.g., 32 high-risk, 89 medium-risk, 129 low-risk).
2. Instead of standard charts, the user clicks "Generate Fleet Strategy Report".
3. The `fleet_graph` agent takes this massive payload of stats and uses the LLM to write an executive-level summary, highlighting supply-chain bottlenecks (e.g., "We have 80 trucks, 32 of which need urgent care; expect heavy downtime") and strategic recommendations.

---

### 📚 Summary of the Tech Stack Symphony
1. **Streamlit** handles the interactive UI, capturing inputs, and managing session state.
2. **Scikit-Learn** computes the deterministic math to predict exact failure risk probabilities.
3. **ChromaDB & Sentence-Transformers** act as the memory bank, holding searchable contextual knowledge (like manuals).
4. **LangGraph** orchestrates the stateful graphs, deciding confidently when the AI should fetch data, pause for a human, or write a final report.
5. **Groq (Llama-3) / OpenAI** acts as the high-speed "reasoning engine" empowering the LangGraph agents.