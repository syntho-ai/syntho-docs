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

- [e.g., compromising quality attribute, follow-up decisions required, …]
- …

## Pros and Cons of the Options <!-- optional -->

### [option 1]

[example | description | pointer to more information | …] <!-- optional -->

- Good, because [argument a]
- Good, because [argument b]
- Bad, because [argument c]
- … <!-- numbers of pros and cons can vary -->

### [option 2]

[example | description | pointer to more information | …] <!-- optional -->

- Good, because [argument a]
- Good, because [argument b]
- Bad, because [argument c]
- … <!-- numbers of pros and cons can vary -->

### [option 3]

[example | description | pointer to more information | …] <!-- optional -->

- Good, because [argument a]
- Good, because [argument b]
- Bad, because [argument c]
- … <!-- numbers of pros and cons can vary -->

## Links <!-- optional -->

- [Link type](link to adr) <!-- example: Refined by [xxx](yyyymmdd-xxx.md) -->
- … <!-- numbers of links can vary -->
