# Private AI Stack Checklist

A practical, vendor-neutral checklist for answering one question: **is this AI setup actually private, or does it just use the word "private" on its pricing page?**

Private is a property of the architecture, not of a label. The short answer is: check where each component of the stack actually runs, what it persists, and who can read it, layer by layer, not as a single yes/no.

## How do I know whether an AI setup is really private?

Check four layers separately. A setup can be private on one layer and not on another, and a vendor's "private" claim usually describes only one of the four.

| Layer | What it is | What to check |
| --- | --- | --- |
| **Interface** | The chat app or assistant your team actually uses | Where it's hosted, who administers it, what it logs |
| **Model access** | Which model answers the prompt | Hosted API under your own account, a model you run yourself, or a mix, decided per task |
| **Data and indexes** | Documents, extracted text, embeddings, vector store, automation state | Where they live, how they're backed up, who can query them |
| **Operations** | Logs, updates, access rules | Who can read application logs and provider-side logs, who can change access |

An interface you host yourself (for example [LibreChat](https://www.librechat.ai/), an open-source platform) does not by itself tell you where the model runs, where the documents end up, or where the backups land. Each of the four layers needs its own answer.

## Data-path questions, per component

For each component, write down: what it receives, where it processes it, what persists, and who can access it. Don't stop at the original file. Extracted text and derived indexes carry the same sensitivity as the source.

| Component | Questions to ask | Why it matters |
| --- | --- | --- |
| **OCR / document extraction** | Which engine? Local process or external API? Does the extracted text or a copy of the document persist anywhere (temp files, logs, queues)? | OCR is a separate processing step from the model, with its own data destination. A local chat model does not make document extraction local. See [Tesseract's own documentation](https://tesseract-ocr.github.io/) for what a self-hosted OCR engine actually needs to run and what it leaves behind. |
| **Embeddings / vector search** | Local embedding model or an external embedding API? Project-scoped isolation, or one shared index? What's the retention policy on the index itself? | An external embedding service receives the text you send it, even if the chat model is local. Deleting the source document does not automatically remove its searchable, embedded copy from the index. |
| **Logging** | Application logs vs. provider-side logs (e.g. an API provider's abuse-monitoring logs)? Who can read each, and for how long? | A self-hosted interface still calls external services for some components; those services' own logs are outside your control unless you've checked their retention terms. |
| **Backups** | Where do backups land? Encrypted at rest? Who can restore them, and does a restore bypass your deletion/retention rules? | A deletion that only touches the live database does not touch a backup taken the day before. |

## ChatGPT Enterprise vs. self-hosted vs. hybrid open-weight models

| Option | What you get | What you still own |
| --- | --- | --- |
| **ChatGPT Enterprise** | Privacy by contract and provider controls. OpenAI states business data isn't used to train its models by default, with admin settings over who can use which tools ([CPLT's comparison, citing OpenAI's enterprise page](https://cplt.tech/blog/what-is-a-private-ai-stack)). | Nothing infrastructural. The interface, model, data store and operations all sit inside OpenAI's systems. Your job is reading the contract (see the DPA section below), not running servers. |
| **Self-hosted, open-weight models** | Full control of all four layers. Models such as [Llama](https://huggingface.co/meta-llama), [Mistral](https://mistral.ai/) or [Qwen](https://github.com/QwenLM), served through [vLLM](https://docs.vllm.ai/en/latest/), [Ollama](https://ollama.com/) or [llama.cpp](https://github.com/ggml-org/llama.cpp), run on hardware you control. | The hardware or server cost, and the operational work: patching, backups, access control and, past roughly 25 registered users or 5-8 concurrent requests, real infrastructure engineering (memory management, I/O, scheduling). A single-user demo running fine says nothing about whether the same setup holds up once a team starts using it. |
| **Hybrid** | An open-weight model for the documents and tasks that must stay local, a hosted API for everything else, with routing decided explicitly per task rather than by a model picking "the best" answer regardless of where the data is allowed to go. | The discipline of keeping that routing rule enforced. An automatic router that prefers a bigger external model when a task looks hard enough can quietly cross the boundary you set. |

Serving engine matters if you self-host past a handful of users: `llama.cpp`/Ollama use static memory slots that fragment under concurrent, variably-sized requests, while engines built for multi-user serving (like vLLM, using PagedAttention) are designed for that case. Benchmark your own model and hardware before assuming either one holds up at your expected concurrency. A fast single-user demo does not establish production capacity.

## What to check in a DPA

Signing a Data Processing Addendum is not the same as having checked it. [GDPR Article 28](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng) requires a written contract between you (the controller) and any processor handling personal data on your behalf. The DPA is that contract, not a substitute for reading your own configuration against it.

At minimum, check:

- **Which service and account** the DPA actually covers (API, a product subscription, and a self-hosted interface calling that API are three different things with three different retention and billing behaviours).
- **What instructions apply**: most DPAs treat your own configuration choices as part of the agreed instructions, which means a misconfigured setting is your responsibility, not a contract breach.
- **Who else processes the data**: the current subprocessor list, and what happens when it changes.
- **Default retention vs. an enabled feature**: "not used for training" is not the same claim as "not retained at all"; a specific zero-retention mode usually needs separate approval and has its own limitations.
- **Where processing actually happens**: a regional storage guarantee and a regional processing guarantee are different claims; confirm which one a given endpoint and model actually support.

This list is deliberately short. For the full walkthrough of what each DPA section covers, how to tell API/product/self-hosted usage apart, and the difference between "no training" and "no retention", see [OpenAI DPA and GDPR Article 28: what to check](https://cplt.tech/blog/gdpr-article-28-openai-dpa-self-hosted-ai), which this checklist draws on.

## FAQ

**Does hosting the chat interface myself keep everything local?**
No. The model, embedding service, OCR engine and backups each have their own destination. A self-hosted interface can still call an external API for any of them.

**Is Zero Data Retention enabled by default on API services that offer it?**
No, for the providers that document this. It typically requires separate approval and has endpoint-specific limitations. Check the current state for the exact endpoints and models you use, not just the feature's existence.

**Does running an open-weight model locally guarantee my data stays private?**
No. Running the model locally keeps that one call inside your systems, but OCR, embeddings, logging and backups can still call an external service. Check each component, not just where the model sits.

**Does an empty search result after deletion prove the data is gone?**
No. It confirms the result of that one query. Other stores, replicas and backups need their own check.

**Is "runs on our own servers" the same as "GDPR compliant"?**
No. Self-hosting shifts responsibility for access control, patching, deletion and lawful use to you. It doesn't remove any GDPR obligation.

## What this checklist is not

This is an engineering checklist, not legal advice. It doesn't certify any specific vendor or setup as compliant, and it makes no claim about CPLT's own capabilities beyond what's stated at [cplt.tech](https://cplt.tech). Your DPO or a qualified adviser still needs to assess lawful basis, purpose and risk for your actual use case.

## Sources

- [GDPR, including Article 28](https://eur-lex.europa.eu/eli/reg/2016/679/oj/eng)
- [OpenAI's Data Processing Addendum, January 2026 (PDF)](https://cdn.openai.com/pdf/openai-data-processing-addendum.pdf)
- [OpenAI API data controls](https://developers.openai.com/api/docs/guides/your-data) (training, retention, Zero Data Retention, regional processing)
- [CPLT's ChatGPT Enterprise comparison, citing OpenAI](https://cplt.tech/blog/what-is-a-private-ai-stack)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [vLLM documentation](https://docs.vllm.ai/en/latest/)
- [LibreChat](https://www.librechat.ai/)
- CPLT's own field notes: [What Is a Private AI Stack vs ChatGPT Enterprise?](https://cplt.tech/blog/what-is-a-private-ai-stack), [OpenAI DPA and GDPR Article 28: What to Check](https://cplt.tech/blog/gdpr-article-28-openai-dpa-self-hosted-ai), [OCR and Scanned PDFs: Map the Whole Document Path](https://cplt.tech/blog/sovereign-ocr-document-ai), [vLLM vs llama.cpp vs Ollama at 25 Concurrent Users](https://cplt.tech/blog/vllm-vs-llamacpp-vs-ollama-25-users), [3 Hidden Failure Modes of Self-Hosted LLMs Past 25 Users](https://cplt.tech/blog/self-hosted-llm-failure-modes)

## License

This checklist is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Reuse, adapt and republish it, with attribution. See [LICENSE](LICENSE).

Built and maintained by [Stéphane Lepain](https://cplt.tech/about), [CPLT](https://cplt.tech).
