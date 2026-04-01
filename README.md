# gh-actions-outputs-experiments
Experiments with Github Actions outputs in various scenarios

# Outputs: Summary

In Github Actions, outputs are used to pass small pieces of data from one job/workflow step to another (not necessarily the same workflow)
TODO ELABORATE LENGTH, CONTENT, STEP SETS

# The outputs block

TODO ELABORATE

# Passing Outputs between different entities

The whole point of outputs is to set them in one place, and utilize them in another. While a workflow step is reponsible for setting the value of an output, it is common to say something like "this workflow/job/action sets this output" or "this step/workflow/job/action uses the output." Thus the use of "entities" in the header for this section--an "entity" can can mean a step, job, workflow, or composite action. 

Each entity-relationship is described in the following list, along with the level of the outputs block(s). Finally, the process of how the output gets set to how it gets received/used is described.

Outputs can be passed between the following entities:
- Same workflow, same job: No need to define an outputs block. One step (with an id) sets the output, another step uses it (referencing that step's id) ex. `steps.<step-id>.outputs.output-name`
- Same workflow, different jobs: The 'outputs' block is defined at the job level (of the job whose step(s) set an output). The 'outputs' block defines the value of the output by referencing the step that is setting the output. For example, say there is a 'Job A' whose id is 'job-a'. This job has a step that sets the value of an output. The id of this step is 'step-id'. The outputs block has an output named example-output, whose value is defined using the following syntax: steps.step-id.outputs.example-output. A different job, 'Job B' can reference this output from Job A. Job B uses 'needs' to ensure Job A sets the output. To get the example-output value, Job B references Job A's output using needs.job-a.outputs.example-output
- Reusable workflow and caller workflow:
  In the reusable workflow, the 'outputs' block is defined at the workflow level AND another at the job level. 
  A step within a reusable workflow sets the output value. The job-level outputs block sets the value of its output by referencing the step id and output name (ex. steps.rw-step-id.outputs.example-rw-step-output). The workflow-level 'outputs' block gets its value from the job-level output (ex. jobs.rw-job-id.outputs.example-rw-job-output). TODO MAPPING ELAB
  Since calling a reusable workflow is strictly a one-step job, there must be a subsequent job
  in the caller workflow to access that output. The subsequent job will reference the "call reusable workflow" job using the job id, NOT the step id (because
  a job that calls a reusable workflow only has no steps). The syntax of that would be something like needs.cw-job-id.outputs.name-of-output. 'name-of-output' is the name of the workflow-level output in the reusable workflow
- Caller workflow and composite action: In this context, the 'caller workflow' refers to a workflow that calls a composite action. A composite action
  is essentially a bundle of steps that you include in the job. Outputs are defined at the top-level in the action.yml (An action.yml technically doesn't have a workflow-level, but you could view it as that). Since composite actions do not have a job (again, they're a bundle of job-agnostic steps), the outputs block references a step id directly (instead of having to reference a job) to get the output. 
  In the caller workflow, a step (with an id) job calls the composite action. A future step in that caller workflow can use the step id to get the output (No outputs block needed, but you MUST reference the step that calls the composite action, not a step within the composite action). Another job in the caller workflow can reference this output as well using a job-level reference. This means that the job with the step that calls the composite action must have an outputs block defined at job-level.  

Chained Reusable Workflows

`workflow-c` is reusable, and is called from `workflow-b`. `workflow-b` is also reusable, and is called by `workflow-a`.
The cadence of calls is `workflow-a` --calls--> `workflow-b` --calls--> `workflow-c`

`workflow-a` can get the output from `workflow-c` like so:
1. `workflow-c` has both a job-level and workflow-level `outputs` block. A job with the id `set-workflow-c-output-job` has a step that sets an output which the job-level `outputs` block references. The workflow-level `outputs` block references the job-level reference so the output can be passed to a caller workflow.
2. `workflow-b` has a job with the ID of 'b-job-1' set up to call `workflow-c`. A subsequent job in `workflow-b` with the ID `b-job-2` has a job-level `outputs` block. 
A workflow-level `outputs` block references the value from the job-level `outputs` block. `workflow-b` is also resuable (`workflow-a` calls it), thus the need for the workflow-level `outputs` block. Note `workflow-b` does NOT need a job-level `outputs` block because `workflow-c`'s `outputs` are already exposed at the job-level?
b-job-2 has a step with the ID 'get-output-from-b-job-1' uses 'needs' and b-job-1's id to get the output from b-job-1. b-job-2's job-level `outputs` block will reference the step ID 'get-output-from-b-job-1' to get the value of the output. 
3. `workflow-a` has a job with the ID 'call-`workflow-b`' which shockingly, calls `workflow-b`. `workflow-a` has another job which uses uses 'needs' and call-`workflow-b`'s id to get the output from call-`workflow-b`. 