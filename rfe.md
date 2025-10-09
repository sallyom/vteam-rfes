# RHEL AI Monitoring with Performance Co-Pilot Demo and Blog

**Feature Overview:**

This RFE outlines the creation of a comprehensive demo and blog post for setting up complete monitoring and observability for RHEL AI workloads using Performance Co-Pilot (PCP), Grafana, OpenTelemetry Collector, and OpenLit GPU monitoring. The demo will showcase how to monitor both system-level metrics and AI workload metrics (specifically vLLM) with GPU utilization tracking, providing users with a production-ready observability stack for RHEL AI deployments.

**Goals:**

* Demonstrate a complete, production-ready monitoring solution for RHEL AI that combines system metrics (PCP) with AI workload metrics (vLLM) and GPU monitoring (OpenLit)
* Provide hands-on guidance for deploying grafana-pcp with PCP on RHEL AI systems to visualize performance data
* Enable RHEL AI users to monitor their AI inference workloads end-to-end, from hardware resources (CPU, memory, GPU) to application-level metrics (vLLM inference statistics)
* Create reusable, copy-paste ready configurations and systemd unit files that users can deploy immediately
* Establish PCP + Grafana as the standard monitoring approach for RHEL AI, differentiating it from generic Kubernetes-based solutions
* Target audience: DevOps engineers, platform engineers, and AI/ML practitioners deploying RHEL AI in production environments
* Expected outcome: Users can set up comprehensive monitoring in under 30 minutes and gain immediate visibility into their RHEL AI workloads

**Out of Scope:**

* Monitoring solutions that require Kubernetes or OpenShift (this is focused on bare-metal/VM RHEL AI deployments)
* Custom application instrumentation beyond standard OpenTelemetry SDKs
* Log aggregation and analysis (focused on metrics and traces)
* Multi-host/distributed monitoring architectures (single RHEL AI host focus)
* Windows or non-RHEL operating systems
* Production-scale high-availability configurations
* Integration with external commercial observability vendors (though architecture supports this)

**Requirements:**

* **[MVP]** Provide step-by-step instructions for installing and configuring PCP with pcp-zeroconf package on RHEL AI
* **[MVP]** Document deployment of Redis container as systemd service for PCP metric storage backend
* **[MVP]** Document deployment of Grafana container with PCP datasource plugins as systemd service
* **[MVP]** Include configuration for vLLM deployment with Prometheus metrics endpoint exposure
* **[MVP]** Document OpenLit GPU collector deployment for GPU utilization monitoring
* **[MVP]** Provide OpenTelemetry Collector configuration that unifies vLLM metrics and GPU metrics into a single stream
* **[MVP]** Include working Grafana dashboard JSON files for visualizing vLLM and GPU metrics
* **[MVP]** All services must be configured to start automatically on system boot via systemd
* Document the PCP openmetrics PMDA plugin configuration for ingesting workload metrics
* Provide troubleshooting guide for common setup issues (pmproxy/redis connection, plugin registration, etc.)
* Include examples of extending the monitoring stack to additional workloads
* Demonstrate how telemetry can be exported to external observability platforms (architecture diagram)
* Blog post must include architecture diagram showing data flow between components
* All code samples and configurations must be tested on RHEL AI 1.1 or later

**Done - Acceptance Criteria:**

* Demo successfully deploys on a clean RHEL AI system with GPU access
* PCP services (pmcd, pmlogger, pmproxy) are running and connected to Redis
* Grafana UI is accessible at http://localhost:3000 with PCP datasources configured
* vLLM container is serving Llama-3.2-3B-Instruct model with metrics at http://localhost:8000/metrics
* OpenLit GPU collector is running and sending GPU metrics to OpenTelemetry Collector
* OpenTelemetry Collector is scraping vLLM metrics and receiving GPU metrics, exporting unified stream
* PCP is ingesting the unified metrics stream from OpenTelemetry Collector via openmetrics PMDA
* Grafana dashboards display real-time vLLM inference metrics (requests/sec, token throughput, queue depth, etc.)
* Grafana dashboards display real-time GPU metrics (utilization, memory usage, temperature, power)
* System-level metrics (CPU, memory, disk, network) are visible in PCP/Grafana dashboards
* All services persist across system reboots (systemd enabled services)
* Blog post is published with clear architecture diagram, step-by-step setup instructions, and screenshots
* Blog post includes section on extending the stack and exporting to external platforms
* All configuration files and scripts are available in a public GitHub repository

**Use Cases - i.e. User Experience & Workflow:**

**Main Success Scenario:**

1. User has a RHEL AI system with GPU installed and wants to set up monitoring
2. User follows blog post to install pcp-zeroconf package and reboot system
3. User deploys Redis container as systemd service for PCP backend storage
4. User deploys Grafana container with PCP plugins as systemd service
5. User configures PCP openmetrics PMDA to connect to Redis
6. User verifies PCP and Grafana are working by viewing system metrics dashboards
7. User deploys vLLM container with Prometheus metrics enabled
8. User deploys OpenLit GPU collector container
9. User deploys and configures OpenTelemetry Collector to scrape vLLM and receive GPU metrics
10. User configures PCP openmetrics PMDA to ingest unified metrics from OTEL Collector
11. User imports vLLM and GPU Grafana dashboards
12. User sends inference requests to vLLM and observes metrics updating in real-time in Grafana
13. User monitors GPU utilization, vLLM performance, and system resources from unified Grafana interface

**Alternative Flow - Existing vLLM Deployment:**

1. User already has vLLM running on RHEL AI
2. User follows blog to add monitoring stack (PCP, Redis, Grafana, OTEL Collector, OpenLit)
3. User updates vLLM deployment to expose metrics endpoint
4. User configures OTEL Collector to scrape existing vLLM instance
5. User immediately gains visibility into existing workload

**Alternative Flow - Export to External Observability Platform:**

1. User completes main setup
2. User wants to send telemetry to Dynatrace/Splunk/OpenShift monitoring
3. User follows extension section in blog to add OTLP exporters to OTEL Collector config
4. User configures authentication and endpoints for external platform
5. User verifies telemetry is flowing to both local Grafana and external platform

**Documentation Considerations:**

* This demo serves as the primary documentation for PCP-based monitoring on RHEL AI
* Existing RHEL AI documentation should be updated to link to this demo/blog
* PCP documentation already exists for RHEL, but needs RHEL AI specific guidance with AI workload examples
* vLLM metrics endpoint documentation exists upstream, but needs RHEL AI context
* OpenLit GPU collector is a third-party tool - provide link to upstream docs with RHEL AI specific notes
* Blog post should be written for users familiar with containers/systemd but may be new to observability concepts
* Include a "Prerequisites" section listing required packages, system requirements (GPU, memory, etc.)
* Provide a "Quick Start" section for users who want to run commands first, understand details later
* Include a "Troubleshooting" section with common issues and solutions
* Link to upstream PCP, Grafana, vLLM, and OpenTelemetry documentation for deep dives
* Consider creating a video walkthrough to accompany the written blog post
* Documentation should be maintained in the ai-observability GitHub repository

**Questions to answer:**

* Should we use valkey instead of Redis for the PCP backend? (valkey is the open-source Redis fork)
* What specific vLLM model should the demo use? (Currently targeting Llama-3.2-3B-Instruct)
* Should we include Tempo for distributed tracing in the MVP, or save for follow-up blog post?
* Do we need to demonstrate the ilab serve workflow in addition to direct vLLM deployment?
* Should the blog post cover building custom vLLM containers with OTLP support, or use pre-built images?
* What level of GPU access is required? (all GPUs vs. specific GPU selection)
* Should we provide Ansible playbooks in addition to manual setup instructions?
* What's the minimum RHEL AI version we're targeting? (Need to verify pcp-zeroconf availability)
* Should dashboards be optimized for single-GPU or multi-GPU systems?
* Do we need to address SELinux considerations for the podman containers?
* Should the demo include cost/resource optimization guidance based on the metrics collected?
* What's the expected retention period for metrics in Redis? Do we need to document cleanup/pruning?

**Background & Strategic Fit:**

Red Hat Enterprise Linux AI (RHEL AI) is Red Hat's platform for deploying, serving, and fine-tuning Large Language Models on-premises. As organizations adopt RHEL AI for production AI inference workloads, comprehensive monitoring becomes critical for:

* **Performance optimization**: Understanding model inference latency, throughput, and queue depths
* **Resource utilization**: Tracking expensive GPU resources, memory consumption, and CPU usage
* **Capacity planning**: Historical metrics for forecasting growth and scaling decisions
* **Troubleshooting**: Rapid identification of performance degradation or resource exhaustion
* **Cost management**: Visibility into GPU utilization to maximize ROI on expensive hardware

Performance Co-Pilot (PCP) is the standard monitoring framework for RHEL and is already included in RHEL AI base images. However, many users are unaware of PCP's capabilities or default to Prometheus/Grafana stacks designed for Kubernetes. This demo and blog will:

* Position PCP + Grafana as the "blessed" monitoring approach for RHEL AI
* Demonstrate that PCP can monitor both system-level and AI workload metrics in a unified way
* Show the value of OpenTelemetry Collector as a metrics aggregation layer
* Highlight RHEL AI's integration with the broader observability ecosystem (OpenTelemetry, Grafana, etc.)
* Provide a reference architecture that users can extend to their specific needs

This effort aligns with Red Hat's strategy to make RHEL AI production-ready out of the box, reducing time-to-value for enterprise AI deployments.

**Customer Considerations**

* **Enterprise environments**: Many customers require all telemetry to remain on-premises for security/compliance. The local Grafana deployment addresses this, while OTEL Collector architecture allows export when permitted.
* **GPU costs**: Customers deploying on expensive GPU infrastructure need to maximize utilization. Real-time GPU monitoring helps identify idle resources and optimization opportunities.
* **Multi-tenant scenarios**: While this demo focuses on single-host monitoring, enterprise customers may need to extend to multi-tenant architectures. The blog should note extension points for this use case.
* **Air-gapped environments**: Some customers deploy in disconnected environments. All container images and packages must be available via Red Hat registries that support disconnected installations.
* **Support and lifecycle**: All components must be supported by Red Hat or be clearly documented as community/upstream projects (OpenLit). PCP, Grafana, and OTEL Collector have Red Hat support paths.
* **Integration with existing tools**: Customers often have existing monitoring infrastructure (Splunk, Dynatrace, Datadog, etc.). The OTEL Collector architecture provides clear integration paths without requiring customers to rip-and-replace.
* **Skills and training**: Many DevOps teams are unfamiliar with PCP but know Prometheus/Grafana. The demo should emphasize that Grafana provides the familiar UI while PCP works in the background.
* **Performance overhead**: Customers need assurance that monitoring won't impact AI inference performance. The blog should include notes on PCP's low overhead and configurable scrape intervals.
* **Security considerations**: Document any security implications of running monitoring stack (exposed ports, authentication, SELinux contexts, etc.)
