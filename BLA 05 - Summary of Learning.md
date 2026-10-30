**Focus:** Other Services You Should Know — Analytics; Compute, Containers and Customer Engagement; Database; Developer Tools; Machine Learning
>**Resources:** AWS SkillBuilder Cloud Practitioner Essentials, Udemy AWS Generative AI course (Stephane Maarek & Frank Kane)

**Summary of Learning:**
>•	Analytics services can be grouped by whether data is at rest (stored files and tables) or in motion (a continuous stream of events), which determines whether batch or streaming tools are appropriate.
>•	Amazon Athena runs standard SQL queries directly against data in Amazon S3 without provisioning a database; it is billed by the amount of data scanned, so columnar formats such as Parquet and date-based partitioning reduce both cost and query time.
>•	Amazon EMR runs big-data frameworks such as Apache Spark and Hadoop on managed clusters made up of primary, core and task nodes; task nodes are well suited to Spot Instances, and EMR Serverless removes cluster management entirely — useful for large-scale document preparation before embedding.
>•	Amazon QuickSight provides interactive dashboards for business users, using the SPICE in-memory engine for performance and Amazon Q in QuickSight for natural-language questions.
>•	Amazon Kinesis Data Streams ingests real-time data across shards; the partition key determines shard placement, so a low-cardinality key creates a hot shard, and records are retained for 24 hours by default (extendable to 365 days).
>•	Amazon MSK provides managed Apache Kafka and is generally preferred when an organisation already uses Kafka, while Kinesis is simpler for new AWS-native workloads.
>•	Amazon EC2 provides virtual servers across instance families, including GPU instances and AWS Inferentia and Trainium chips for AI workloads; On-Demand, Savings Plans or Reserved, and Spot pricing should be matched to how predictable and interruptible the workload is.
>•	Amazon ECS is AWS’s native container orchestrator and Amazon EKS provides managed Kubernetes; AWS Fargate runs containers for either without managing servers.
>•	An ECS task uses two distinct IAM roles: the task execution role, used by ECS to pull images and write logs, and the task role, used by application code to call services such as Amazon Bedrock.
>•	Amazon Lex builds conversational chatbots using intents and slots, and Amazon Connect provides a cloud contact centre that can route customers from a Lex bot to a human agent.
>•	Amazon DocumentDB is a MongoDB-compatible document database for JSON data and supports vector search, making it usable as a store for RAG applications built on document data.
>•	Amazon ElastiCache provides in-memory caching and Amazon MemoryDB provides durable in-memory storage; both support Valkey, an open-source Redis-compatible engine, and are used in GenAI applications for session history, semantic caching and low-latency vector search.
>•	Amazon Neptune is a graph database for highly connected data, and Neptune Analytics adds vector search to graphs, enabling retrieval that combines semantic similarity with relationships between entities.
>•	Infrastructure as code makes environments repeatable and version-controlled; AWS CloudFormation deploys stacks from YAML or JSON templates, with change sets to preview changes and automatic rollback on failure.
>•	The AWS CDK defines infrastructure in programming languages such as Python and TypeScript and synthesises it into CloudFormation templates, supporting reusable constructs and logic.
>•	The AWS CLI and SDKs authenticate with credentials; long-term access keys must never be committed to code or repositories, and IAM Identity Center sign-in or IAM roles are preferred.
>•	AWS CodeArtifact provides a managed private repository for software packages, allowing teams to proxy public repositories such as PyPI and npm and control which package versions are approved.
>•	Amazon Rekognition analyses images and video, including content moderation with confidence scores; Amazon Textract extracts text, forms and tables from documents, using synchronous APIs for single pages and asynchronous jobs for multi-page files.
>•	Amazon Lex builds chatbots, Amazon Augmented AI routes low-confidence predictions to human reviewers, and Amazon Kendra provides enterprise search; Amazon has announced that Augmented AI and Kendra are being deprecated.
>•	Purpose-built AI services suit fixed, high-volume tasks with predictable output and cost, while foundation models suit open-ended tasks; many applications combine both, such as Textract extraction followed by model-generated explanation.

**Reflection:**
>The most useful idea this week was sorting services by the question they answer — is the data at rest or moving, and do I need a server, a container or neither. It also reinforced least privilege: an ECS task’s two roles each hold only what their user needs.
>This week showed me that not every AI problem needs a foundation model. Ready-made services like Textract and Rekognition do one job reliably and predictably, and they work well alongside a model. Infrastructure as code also made sense to me as the practical answer to every cleanup warning in this course.


