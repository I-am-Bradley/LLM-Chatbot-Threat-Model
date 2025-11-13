## Initial requirements for threat modeling analysis and eventual testing
pandas
numpy
openai
pydantic
## Discuss the flow diagram of how the LLM works from user input all the way to data output.

## Defining All Possible Attack Surfaces and Threat Enumeration 
The best way to be able to analyze it all is to follow the flow diagram from
user query all the way to the output by the LLM.
At the user interface or the front-end attack surface, we have the following:
| Attack Surface          | In-scope Threat          | Specific Enumeration                                                                                                    |
| ----------------------- |:-----------------------: | ----------------------------------------------------------------------------------------------------------------------: |
| User Prompt Input Field | Prompt Injection         | Direct Injection                                                                                                        |
| Output Display Area     | Improper Output Handling | Attacker uses a payload that forces the LLM to output unsanitized HTML/JavaScript                                       |
| Session/Authentication  | Model Denial of Service  | Attacker submits extremely long, complex, or recursive prompts to intentionally consume all available compute resources |

At the application backend/middleware attack surface we have:
| Attack Surface                      | In-scope Threat                  | Specific Enumeration                                                                                                                                       |
| -----------------------             |:-------------------------------: | --------------------------------------------------------------------------------------------------------------------------------------------------------:  |
| Input Validation/Sanitization Logic | Prompt Injection                 | Attacker bypasses the input filter to pass a malicious prompt to the LLM.                                                                                  |
| Logging and Monitoring Systems      | Sensitive Information Disclosure | Sensitive user data, conversation history, or even system prompt fragments are inadvertently logged or sent to an insecure monitoring endpoint.            |
| API Call to LLM                     | System Prompt Leakage            | Attacker crafts a prompt that the model interprets as a command to include the start of its own confidential system instructions in the final API response |
| Model Response Handling Logic       | Improper Output Handling         | The middleware receives the LLM output and passes it unsanitized to an internal database query                                                             |

We also have the Model Data (RAG Component) Attack Surface
| Attack Surface                 | In-scope Threat                  | Specific Enumeration                                                                                                                                                                                                                                       |
| ------------------------------ |:-------------------------------: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| RAG Knowledge Base (Vector DB) | Sensitive Information Disclosure | Attacker crafts a prompt that causes the RAG mechanism to retrieve and expose highly sensitive data that the model subsequently uses in its response.                                                                                                      |
| RAG Retrieval Logic            | Prompt Injection                 | Indirect Injection: An attacker inserts malicious instructions into a document that the RAG system indexes. query retrieves this malicious document, and the instruction is then passed to the LLM as part of the context, triggering an unintended action |
| Vector/Embedding Indexing      | Sensitive Information Disclosure | Lack of fine-grained access controls on the vector embeddings allows the LLM to retrieve data chunks belonging to a user/group who shouldn't have access                                                                                                   |

At downstream systems, that is also another attack surface where we can possible have;
| Attack Surface                    | In-scope Threat                  | Specific Enumeration                                                                                                                                                                                                   |
| --------------------------------- |:-------------------------------: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| Tool/Function Calling Interface   | Sensitive Information Disclosure | Attacker manipulates the LLM into calling a function (e.g., send_email(target, content) or delete_file(path)) with malicious or unintended arguments, such as targeting a privileged account or a critical system file |
| Tool Permissions                  | Excessive Agency                 |The Agent is granted over-privileged API keys (e.g., a "read-write" key when only "read" is needed) for a downstream system, allowing a prompt injection to perform an escalation attack                                |
| External API Input/Schema         | Improper Output Handling         | The LLM is tricked into generating output that is valid for a tool's function call but contains a secondary malicious payload                                                                                          |

And lastly while the host operating system is out of scope, we have the MLOps and infrastructure which can serve as attacking surfaces and have possible attacks such as;
| Attack Surface                     | In-scope Threat                     | Specific Enumeration                                                                                                                   |
| ---------------------------------- |:----------------------------------: | -------------------------------------------------------------------------------------------------------------------------------------: |
| Configuration Files/Secrets Manager| Configuration Files/Secrets Manager | Misconfiguration allows the LLM application to leak API keys or database credentials required to access the RAG or Downstream Systems. |
| Resource Quota Settings            | Model Denial of Service             | Failure to implement or enforce strict resource quotas                                                                                 |
