# memary: Open-Source longterm memory for autonomous agents 

## Why use memary? 
Agents use LLMs that are currently constraint to finite context windows. memary overcomes this limitation by allowing your agents to store a large corpus of information in knowledge graphs, infer user knowledge through our memory modules, and only retrieve relevant information for meaningful responses. 

## Features
- **Routing Agent:** Leverage a ReAct agent to route a query for execution amongst many tools. 
- **Knowledge Graph Creation:** Leverage Neo4j to create knowledge graphs storing agent responses.
- **Memory Stream:** Track all entities stored in the knowledge graph using entity extraction. This stream reflects the user's breadth of knowledge.
- **Entity Knowledge Store:** Group and order all the entities in the memory stream and pass top N entities into the context window. This knowledge store reflects the user's depth of knowledge. 

## How it works 
The current structure of memary is detailed in the diagram below.

![a text](diagrams/system.jpeg)

The above process includes the routing agent, knoweldge graph and memory module are all integrated into the `ChatAgent` class located in the `src/agent` directory.

Raw source code for these components can also be found in their respective directories including benchmarks, notebooks, and updates.

## Installation
1. Create your [virtual environment](https://packaging.python.org/en/latest/guides/installing-using-pip-and-virtual-environments/#create-and-use-virtual-environments) and activate it

2. Install Python dependencies:
   ```
   pip install -r requirements.txt
   ```

## Demo
todo: add video of demo? OR add @top

To run the Streamlit app: 
1. Ensure that a `.env` exists with necessary API keys and Neo4j credentials. 

2. Run:
   ```
   streamlit run streamlit_app/app.py
   ```

## Detailed Component Breakdown
### Routing Agent
![agent diagram](https://github.com/kingjulio8238/memary/assets/120517860/e5be38db-8c7a-4df2-8b1d-b578fa9c827f)
- Uses the [ReAct agent](https://react-lm.github.io/) to plan and execute a query given the tools provided. This type of agent has the ability to reason over which of the tools to use next to further the response, feed inputs into the selected tool and repeat the process with the output until it determines that the answer is satisfactory. 
- Current tool suite:
While we didn't place strong emphasis on equipping the agent with many tools, we hope to see memary help agents in the community equipped with a vast array of tools covering multi-modalities. 
  - **Location** - determines the user's current location and nearby sorroundings using geocoder and googlemaps.
  - **CV** - answers a query based on a provided image using gpt-4-vision-preview. 
  - **Search** - queries the knowledge graph for a response based on existing nodes, and executes an external search if no related entities exist.    
- How does it work?
  - Takes in each query &rarr; selects a tool &rarr; executes and finds an answer to current step &rarr; repeats this process until it reaches a satisfactory answer
- Purpose in larger system
  - Each response from the agent is saved in the knowledge graph. You can view responses from various tools as distinct elements that contribute to the user's knowledge.
- Future contributions
  - Make your own agent! Add as many tools as possible! Each tool is an expansion of the agent's ability to answer a wide variety of queries.
  - Create a LLM Judge that scores the routing agent and provides feedback. 
  - Integrate multiprocessing so that the agent can process multiple sub queries simultaneously. We have open sourced the query decomposition and reranking code to help with this! 

### Knowledge Graph
![KG diagram](https://github.com/kingjulio8238/memary/assets/120517860/fc009e5b-e3ea-4b9d-b54a-ae8404550bb4)
- What are knowledge graphs (KG)?
  - KGs are databases that store information in the form of entities, which can be anything from objects to more abstract concepts, and their relationships with one another.
- KGs vs other knowledge stores
  - KGs provide more depth of highly necessary context that can be easily retrieved. 
  - Graph structure of the knowledge store allows information to be centered around certain entities and their relationships with other entities, thus ensuring that the context of the information is relevant. 
  -  KGs are more adept to handling complex queries as varying relationships between different entities in the query can provide insight into how to join multiple sub graphs together. 
-  Knowledge graphs &harr; LLMs
   -  memary uses a Neo4j graph database to store knoweldge. 
   -  Llamaindex was used for adding nodes into the graph store based on documents. 
   -  Perplexity (mistral-7b-instruct model) was used for external queries. 
-  What can one do with the KG?
   -  Inject the final agent responses into existing KGs. 
   -  memary uses a [recursive retrieval](https://arxiv.org/pdf/2401.18059.pdf) approach to search the KG which involves determining what the key entities are in the query, building a subgraph of those entities with a maximum depth of 2 away, and finally using that subgraph to build up context. 
   -  When faced with multiple key entities in a query, memary uses [multi-hop](https://neo4j.com/developer-blog/knowledge-graphs-llms-multi-hop-question-answering/) reasoning to join multiple subgraphs into a larger subgraph to search through. 
   -  These techniques reduce latency when compared to searching the entire knowledge graph at once. 
-  Purpose in larger system
   -  Continuously update the memory module with each node insertion. 
-  Future contributions
   -  Expand graph’s capabilities to support multiple modalities.
   -  Graph optimizations to reduce latency of search times. 

### Memory Module
![memory module diagram](https://github.com/kingjulio8238/memary/assets/120517860/3b2aad33-d221-4858-b051-a181ee9ec421)
- What is the memory module?

The memory module is made up of the Memory Stream and Entity Knowledge Store. The memory module was influenced by the design of [K-LaMP](https://arxiv.org/pdf/2311.06318.pdf) proposed by Microsoft Research. 
1. The Memory Stream captures all entities inserted into the KG and their associated timestamps. This stream reflects the breadth of the users' knowledge ie. concepts users have had exposure to but no depth of exposure is inferred. 
   - Timeline Analysis: Map out a timeline of interactions, highlighting moments of high engagement or shifts in topic focus. This helps in understanding the evolution of the user's interests over time.
   - Extract Themes: Look for recurring themes or topics within the interactions. This thematic analysis can help in anticipating user interests or questions even before they are explicitly stated.
2. The Entity Knowledge Store tracks the frequency and recency of references to each entity stored in the memory stream. This knowledge store reflects users' depth of knowledge ie. concepts they are more familiar with than others.
   - Rank Entities by Relevance: Use both frequency and recency to rank entities. An entity frequently mentioned (high count) and referenced recently is likely of high importance and the user is well aware of this concept.
   - Categorize Entities: Group entities into categories based on their nature or the context in which they're mentioned (e.g., technical terms, personal interests). This categorization aids in quickly accessing relevant information tailored to the user's inquiries.
   - Highlight Changes Over Time: Identify any significant changes in the entities' ranking or categorization over time. A shift in the most frequently mentioned entities could indicate a change in the user's interests or knowledge.
- Purpose in larger system
  - Compress/summarize the top N ranked entities in the entity knowledge store and pass to the LLM’s finite context window alongside the agent's response and chat history for inference. 
  - Personalize Responses: Use the key categorized entities and themes associated with the user to tailor agent responses more closely to the user's current interests and knowledge level/expertise.
  - Anticipate Needs: Leverage trends and shifts identified in the summaries to anticipate users' future questions or needs. 

## Future Integrations
The source code for several other components that are not yet integrated into the main `ChatAgent` class. These include query decompostion and reranking. The diagram below shows what how the newly integrated system would work.

![Future Integrations](https://github.com/kingjulio8238/memary/assets/120517860/213f9547-6dcb-4adf-9715-b54e707072d3)

### Query decomposition
- What is query decomposition?
  - LLM preprocessing technique that breaks down advanced queries into simpler queries to expedite the LLM’s ability to answer the prompt. It is important to note that this process leaves simple queries unchanged.
- Why decompose?
  - User queries are complex and multifaceted, and base-model LLMs are often unable to fully understand all aspects of the query in order to create a succinct and accurate response.
  - Allows the same LLM to answer easier questions, and synthesize those answers to output a much better response.
- How it works
  - Initially, a LlamaIndex fine tuned query engine approach was taken. However, a LangChain query engine was found to be faster and easier to use. LangChain’s `PydanticToolsParser` framework was used. The query_engine_with_examples has been given 87 pre-decomposed queries (complex query + set of subqueries) to determine a pattern. Users can invoke the engine with individual queries, or collate them into a list and invoke by batch. 
- Purpose in larger system
  - In a parallel system, the Routing Agent will be able to parse multiple queries at once. The query decomposer will pass all subqueries (or original query if no subqueries exist) to the Routing Agent at once.
  - Simultaneously, the query decomposer will pass the original query to the reranker to allow it to optimize the output from the routing agent
- Future contributions
  - When multiprocessing system is provided, QD will be integrated. All inputted queries will be passed to QD, and the proper (sub)queries wil be passed to the routing agent for parallel processing.
  - Self-Learning: Whenever queries are decomposed properly, those examples will be appended to the engine’s example store for it to improve itself.

### Reranking
