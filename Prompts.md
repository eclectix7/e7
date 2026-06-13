- [x] MacBook Todo list
- Great. Now that you have the details about e7 & myself, I want you to draft a todo list for setting up the new MacBook for continued development when it's here.
- Note also that e7 is set up on Google as an organization, not individual developer. It will be the same for Apple.
- We have a DUNS number 
- Our repos are in Github
- We previously registered as a developer over a year ago, but it was refunded for unclear reasons. I think it was because were were unaware of the DUNS number.
- You know our tech stack & objective so you should be able to created lists of everything i need to do on the mac, iphone, in the cloud, etc to begin the process of deploying our apps on iOS
- Please also draft a 2nd document with proposals to securely install you on the macbook so that you can be of more assistance, but also unable to harm the system. There should be cross refs with the docs where needed if some things must be decided and done before others.
- Please write your output files in a new folder in the obsidian kaizen folder so I can view & edit them in Obsidian
- [ ]  Looking at the wxpi user memory documents that were written, can you confirm if the documents in the obsidian vault were written with the intent to make the utilities reusable? We want the utilities that are built by Cursor in that app so be something we can use as part of our e7 react library
- [x] Something is wrong with the final summary output text, there is almost no background/foreground contrast in light or dark mode I cannot read what you have written
- [x] We want the ask mode to work well with plan mode so that we can develop an understanding of the code base & change intent then immediately transition into /plan mode to build from that information
- [ ] Create a folder called `agents`  in the `social` folder of the obsidian vault.
- This folder will be for content related to the planning and design of agents. 
- We want to design a (potentially) multi-step LLM utility for programmatically building/designing an agent for a user in order to capture & recreate the type of responses the user is looking for
- Example usage:
	1. The user will select a genre they're interested in
	2. They will select 1+ personalities they like
	3. They will optionally provide detail about why they like them
	4. We will record this information about the user preferences
	5. We will research the personalities / people describe along with the details shared about why they like them to form a better understanding of the personalities' traits and characteristics
	6. Using that information we will create agent prompts for that persona along with their own skill definitions unique to them that they can invoke if/when appropriate. 
		1. The agent / persona's main prompt will outline their defining details, traits, main characteristics.
		2. The skills will add greater nuanced detail for that persona but only on demand to keep the context smaller.
		3. Skills might looks like a style of photo or painting, signature cooking or baking recipe creation, poem or inspirational quotes, etc
	7. The prompts and skills will work along with hard coded app utilities to mimic that persona & related skills on demand
- What we are building will be a reusable utility that can be dropped into existing applications
- Access to LLMs will be through OpenRouter to allow for run time model switching. Model access will be injected into the code so the calling code(application) will be responsible for providing it
- The applications will be React based using TypeScript
- We do not need to build the UI, we are focused only on the utility right now

- [ ] I'm not sure what you did.
1. You were given instruction and confirmed that you noted to put your work in ~/e7/kaizen not ~/e7 but you did it anyway
2. There's nothing in the folder (plan,architecture, notes, diagrams, etc)
3. You dumped a plan to the CLI and said you wrote the plan in .hermes/plans/2026-05-24_132000-agent-design-utility.md  (useless here because you also have instructions that we using this machine for development)
4. There is nothing *in* .hermes/plans/2026-05-24_132000-agent-design-utility.md to copy

- [ ] Write a plan for building an API for my own RAG service. 
- I want a PHP UI for uploading my documents on a HostPapa LAMP server.
 The UI & documents will live on that server
- We will use Convex & OpenRouter for the RAG implementation
- I need to be able to upload documents, txt or pdf that can be catalogued with metadata that can be used to narrow down the documents that should be used
- We want to be able to send a prompt/query with an optional list of  document references to use in order to respond. If the list of document references are empty, then, the API should use the document vectors & metadata tables to find the be n=3 documents (n should also be an optional parameter), to limit how many documents to use in the query.
- [ ]  Our plan in @file:e7/kaizen/rag-service-plan.md needs some updates:
- We do not want to use claude models
- This service will be used as part of an larger organization RAG system & queried by multiple *other* applications & APIs with their own context & system prompts, so it will be in charge of gathering relevant information to form a response, but not in charge of forming the user facing response.
- The documents in this resource will be static 
- This system will be the research side of the response formation
- 







