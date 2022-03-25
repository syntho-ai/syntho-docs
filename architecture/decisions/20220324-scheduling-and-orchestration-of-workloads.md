# Scheduling and orchestration of workloads

- Status: draft
- Deciders: Younes Moustaghfir, Simon Brouwer <!-- optional -->
- Date: 24-03-2022 <!-- optional. To customize the ordering without relying on Git creation dates and filenames -->
- Tags: scheduling,orchestration,workflows

Technical Story: [Orchestrator tool integration (i.e. Airflow)](https://dev.azure.com/syntho/Syntho%20Engine%20Core/_backlogs/backlog/Syntho%20Engine%20Core%20Team/Epics/?workitem=1575)

## Context and Problem Statement

For our various workloads, we need some kind of scheduling and/or orchestration to call tasks in an efficient way or on a time-based schedule. Currently we use Ray for orchestrating Machine Learning models and/or reading/writing to database or filesystems, which is called directly in Syntho Core API and needs more custom solutions for time-based scheduling for example.

## Decision Drivers <!-- optional -->

- Syntho Core API is now using Ray directly and acts as the scheduler, which requires more work in order to improve on it.
- Syntho Core API does not have time-based scheduling as of now.
- Orchestration is done via Ray, but needs to communicate back to Syntho Core API somehow.
- No DAG representation of tasks in Ray or Syntho Core API, harder to keep track of failures and fault-tolerance.

## Considered Options

- Airflow
- Additional custom solutions in Syntho Core API
- Ray Workflow
- Prefect
- Dagster
- Luigi

## Decision Outcome

Initial decision: **Airflow**, because of time-based scheduling, DAG structure and maturity as an application.

After initial problems and delays in integration, **additional custom solutions in Syntho Core API** was chosen.

Airflow was not really made for DAG structures that can be different of each DAG run, so you are forced to define static workflows. This moves the problem of scheduling models to the already existing Orchestrator layer in the syntho-engine package or using additional tools (external storage or database) in Airflow to make this possible via triggering of multiple DAG runs.

### Positive Consequences <!-- optional -->

- Less integration costs compared to Airflow. Syntho Core API is already integrated in product landscape as in https://demo.syntho.ai.
- Focus on less applications as the team.
- Quicker path to MVP.

### Negative Consequences <!-- optional -->

- Follow-up decision required after implementing more features in Syntho Core API.
- Probably more custom solutions needed on Syntho Core API to support wanted features.

## Pros and Cons of the Options <!-- optional -->

### Airflow

Airflow is a platform to programmatically author, schedule and monitor workflows. Used as an application within our landscape, it would become the main point of communication between for the Syntho Core API to trigger certain workflows or update time-based scheduled workflows.

- Pro: Most complete tool for scheduling and executing workflows (DAGs)
- Pro: A lot of integration possibilities and features
- Pro: Extensive documentation
- Pro: Easy Kubernetes integration
- Con: Does not have a ‘core’ API, only available to be used as full airflow package
- Con: Requires infrastructure and monitoring
- Con: DAG structures can't be very complicated of compute heavy and is not a BP if it is dynamic

### Syntho Core API (FastAPI)

The management system for interacting with database, data storages and job configuration for all workflows. Does not include a scheduler at the moment, but is an asynchronous application with a possibility for adding background tasks.

- Pro: Would require less integration, since application is already being developed and in the product landscape
- Pro: Not adding an additional scheduler application would decrease complexity of landscape.
- Con: Will require more Python packages or custom tooling to facilitate some features

### Ray Workflow

Ray workflow is a low-level Airflow variant that support the creating of DAG-based workflows. 

- Pro: Low-level implemention of Airflow in a sense
- Pro: Ray already being used in the application landscape
- **!Con: Still in alpha phase**

## Links <!-- optional -->

- [Airflow](https://airflow.apache.org/) <!-- example: Refined by [xxx](yyyymmdd-xxx.md) -->
- [Ray](https://www.ray.io/)
- [FastAPI](https://www.ray.io/)
