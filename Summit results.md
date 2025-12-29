## Autoinstrumentation: eBPF checks results
Decisions/Actions:
	Tail based tracing
		SAAS onboard first; But it should have calculable benefits (need to create use case)
			Need users of Tracing - (Action Item) To discuss with Egor Budrin about SAAS users and cases
			Sergey's proposal - to use traces in Autotesting process for troubleshooting
	EBPF not have benefits without tail-based tracing --> Hold it, big chance to be prohibited in Production
		(Action Item) Need to investigate the technologies that used by product/projects - Alexey (HTTP 1.1, HTTP 2.0, gRPC etc)
			Metrics?
		(Action Item) How many customers will reject using EBPF in prod - Alexey?
		SAAS on-boarding if it's make sense
	(red team) To find a way how to check services that not supported tracing (like tracetest.io) during testing
		Vladimir propose to create one generic test or several tests (client-server spans) and implement it on SAAS envs first
		SaaS based tracing tests - need to identify the way and scenario
			To integrate tests into SAAS processes/tests

## CDT/Pyroscope: Next steps + HWE
Decisions/Actions:
	Start using Pyroscope in SAAS/Projects
		Onboard it in SAAS team ASAP
			Need deployment schema for DEV/QA/CI/SIP/SVT
			To estimate resources - CDT vs Pyroscope
			To estimate scope - SVT with CDT VS SVT with Pyroscope (missed call tree, missed SQK calls, etc)
	Hold CDT java in NC
		Alexey will finish to fix Springboot security issue next week
		CDT will be in support mode (no new features)
		Find SVT teams / DEV teams who really need CDT in daily work
			Create question list for all SVT/Support related people about CDT descope - analyze it and decide 
				Egor will provide names from their expirience
			One problem that will not be solved in CDT Java - Cassandra storage
	Stop CDT Go in Open Source
		To check maybe it's possible to save CDT data to Pyroscope DB
	To plan to create new collector - to collect dumps (thread, heap)
		
## Replacement plan of Jaeger to Grafana Tempo
Decisions/Actions
	Most important driver to replacing - stop using Cassnadra
	Need to go to SAAS for plan replacement activities 
		Perform real user testing and SVT
		Use data from SAAS for third-party Committee approval
	Abhishek highlighted that new type of Observabilty data will store with business data and it should be analyzed and solved. (policies in single S3 or dedicated s3 for ops data)
	Tickets will be crated 

## Monitoring Configurations: WorkShop
Abhishek propose to have only one Application for SAAS and Project bundles.
	Helm Chart should be single
Bare minimum flavor for Monitoring Bundle - we have some applications that depends on monitoring. Are we need to create some additional "Minimal bundle". Need to plan activity.
	Actually we need additional investigation of Keda.
To think about 3ldb controller in Monitoring
	Need to have a session with Shoaib about 3ldb onboarding and challenges
		CSMP
		Dashboards generator
	What should we do with "gray" blocks in Dashboard - is it absent by design OR it's a bug
Need to plan a session with Pavel Karpievich and align about Monitoring pack artifacts onboarding and content. Vipul will schedule the meeting.  

## Exporters: Scope & Timeline Alignment
We don't need to invest in any PAAS-dashboard related works if it's not utilized in SAAS
To check with Pavel about current configuration of Monitoring Operator in SAAS clusters
Need to compare metrics between PAAS exporters and Platform Exporters
	Try to assign it to Yulia.
	Need to analyze the metrics that used in Grafana Dashboard in PAAS and communicate with PAAS team (Vladislav). They use one dashboard - OpenShift metrics
Question to Egor and Yogesh - Is SAAS team use previous PAAS dashboards? If not - we don't need to invest in any PAAS-dashboard related works. 

SAAS used for infra mon - IT team dashboards. (Roman Smirnov)

## Mon Catalog
Problems discovered and proofs of problems
	(Delivery) No way to quickly create documentation for external customers
		Proofs
			List or real requests (Etisalat, Google Fiber, Andorra). Want to have demos, check dashboards. 
			ORT required it
		Solution
			Generate docs using Generator
			(optional)??? Check in some way that Dashboards in working and filled by data
		Questions
			How to fix problems because of bad quality (in report, like no descriptions of panels, explicit errors) of some Dashboards?
				PSUP tickets should be created
				How to push product to fix it?
			How to find metrics that created only in case of execution of particular case
				Try to use ADIT for find "in-code" business metrics
	(General knowledge) No way to search metrics/dashboards by Developers/Ops teams
		Proofs
			List of requests and cases
		Solution
			Upload all docs in BASS OR Static Site (Per version or latest)
				Alexey K. will check how to create static site like MKDocks Material in NC
			Upload in Bill AI Chat for start Q&A
	On Hold - (Monitoring Excellence) No release checks of Monitoring
		Proofs
			Why it's required? Who is the consumer of this info?
		Solution
			No solution now

## Brainstorm - How to improve tracing in Projects
Main problem
	Project usually don't fix problems related to observability/tracing
		 
Base of NC Tracing approach - libraries in each Application 
	How to onboard any test in SAAS pipelines? Need technical ways and accountable/responsible persons.
	How to identify that applications not support tracing?
		Is it possible to use static code analysis for check tracing enabled?
			Yes, but only partially
			We cannot check propagation logic and success using static code analysis
				Need live tests for that
		How to do live test tracing for applications? 
			In QA tests we need to ADD trace_id in each request
			In release tests we GENERATE trace_id in each request 
				--> To check that during release tests we gather all traces and have only GENERATED trace id or linked to it.
			To scrape all services from k8s --> push request with GENERATED trace_id --> wait response with particular trace_id --> fail test if no trace with particular trace_id existing in tracing
		To create a test when we find all spans with empty parent_span --> if it's empty - bug in micro-service
		In BSS the breaks of traces was identified when we have new trace_id with old request_id
		Savio propose the next way: scrape all nodes from Grafana Nodes graph --> compare it with all possible services from logs --> create a table with differences and suggestions why is that and potential problems. 
EBPF tracing is optional and can be used internally and in projects where EBPF allowed.
	Need it in case when we need Tracing in current installed ENV where:
		Project applications without implemented Tracing Guide
		Product applications without implemented Tracing Guide
		3-rd party Application without Tracing
Buid-in auto-instrumentation agents for supported technologies (Java, Python, Node.js)
	To add auto-instrumentation agents in Base Image
	OCI volume attach as a Volume (when it's GA)
		To create MRs for all products

## Alignment | Monitoring pack artifacts onboarding and content
Need to create new SAAS template for POC SAAS Cluster with Tempo and Jaeger in parallel:
	MinIO installation (NC pack)
	Grafana Tempo Installation (community HELM)
	Open Telemetry collector with parameters
	Probabilistic = 100 parameters
	Any other changes for have tracing E2E

## Feedback from MTD team: project Observability experience

## ADIT
Why we need ADIT
	Graph - But Grafana Tempo is better for realtime dependencies
	Docs generation - cross links - not analyzed