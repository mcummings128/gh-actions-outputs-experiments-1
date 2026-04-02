# gh-actions-outputs-experiments
Experiments with Github Actions outputs in various scenarios

# Outputs: Summary

In Github Actions, outputs are used to pass small pieces of data from one job/workflow step to another (not necessarily the same workflow)
TODO ELABORATE LENGTH, CONTENT, STEP SETS >> GITHUB_OUTPUT

# The outputs block

TODO ELABORATE
TODO NEEDED FOR MOST OUTPUT PASSING EXCEPT FOR STEPS IN THE SAME JOB
OUTPUTS BLOCK NOT!! UPDATED DYNAMICALLY, UPDATED AT END OF JOB (EXPRESSIONS INCLUDING ${{}} ARE EVALUATED ON THE RUNNER AT THE END OF EACH JOB)
TODO SECRETS 

# Passing Outputs between different entities

The whole point of outputs is to set them in one place, and utilize them in another. While a workflow step is reponsible for setting the value of an output, it is common to say something like "this workflow/job/action sets this output" or "this step/workflow/job/action uses the output." Thus the use of "entities" in the header for this section--an "entity" can can mean a step, job, workflow, or composite action. 

Each entity-relationship in the following list has its own subsection. Each subsection describes the level of the outputs block(s) as well as the process of how the output gets set to how it gets received/used is described.

Outputs can be passed between the following entities:
- Same workflow, same job
- Same workflow, different jobs
- From reusable workflow and caller workflow*
- Chained reusable workflows (i.e. caller calls a reusable workflow, which in turn calls another reusable workflow, ad infinitum)
- Caller workflow and composite action

*Regarding caller/reusable workflows, the subsection discussed here is specifically discussing how a reusable workflow can set an output, and how the caller can reference that output. The opposite (a caller workflow setting an output that the reusable workflow can use) is not discussed, because that is solved via defining inputs for the reusable workflow. 

## Same workflow, same job:

### outputs block Location
No need to define an outputs block--if the outputs are only to be used in the same job they are set, steps can refer to these outputs using <step-id> syntax (see below). 

### Process
1. A step (with an id) within the job sets the output,
2. Another, further down step references the step's id to use the output (ex. `steps.<step-id>.outputs.output-name`)

## Same workflow, different jobs

### outputs block Location

The `outputs` block is defined at the job level (of the job whose step(s) set an output). The `outputs` block defines the value of the output by referencing the step that is setting the output. 

### Process

To explain this process, imagine this example scenario: There is a 'Job A' whose id is 'job-a'. This job has a step that sets the value of an output. A different job, 'Job B' needs to reference this output. 

1. 'job-a' has a step with the ID 'job-a-step-id'. The job-level `outputs` block has an output named example-output, whose value is defined using the following syntax steps.job-a-step-id.outputs.example-output. 
2. At the end of job-a, the outputs block is updated.
3. 'job-b' can reference this output from job-a: it uses 'needs' to indicate it is dependent on job-a and thus wait until it completes. To get the example-output value, 'job-b' references job-a's output using needs.job-a.outputs.example-output

## From reusable workflow and caller workflow:

### outputs block Location

The `outputs` block is defined at the job level (of the job whose step(s) set an output). The `outputs` block defines the value of the output by referencing the step that is setting the output. In the reusable workflow, the `outputs` block is defined at the workflow level AND another at the job level. It should be noted that the `outputs` block is a direct child of workflow_call.

### Process

<ins>Caller Workflow</ins>
1. The caller workflow executes a job that calls the reusable workflow. 
2. Since a workflow-calling job does not and can not involve any steps, only a subsequent job(s) can reference the output. The subsequent job will use 'needs' to indicate it is dependent on the workflow-calling job and also reference the workflow-calling job using the job id. The syntax of that would be something like `needs.<calling-workflow-job-id>.outputs.<name-of-output>`. `<name-of-output>` is the name of the workflow-level output in the reusable workflow.
<br>
<ins>Reusable workflow</ins>

1. A step (with an id) within a reusable workflow job sets the output value. 
2. Once the job is complete, the job-level `outputs` block sets the value of its output by referencing the step id 
and output name (ex. steps.rw-step-id.outputs.example-rw-step-output). 
3. The workflow-level `outputs` block gets its value from the job-level output (ex. jobs.rw-job-id.outputs.example-rw-job-output). It is important to emphasize this happens AFTER any job-level blocks are resolved. 

## Chained Reusable Workflows

TODO MAKE THIS INTO A SENTENCE(i.e. caller calls a reusable workflow, which in turn calls another reusable workflow, ad infinitum)

### outputs block Location

Note: When considering reusable workflows, saying "the outputs block [is] at workflow-level" is slightly misleading--really, it means that the block is a direct child of workflow_call. In practice this is pretty much the same as being at workflow-level, since workflow_call is at workflow level. It's something to keep in the back of your head. 

The location of the outputs block varies depending on the nesting level of the reusable workflow: 
- The 'outermost' workflow does not need an outputs block if no outputs are being set. 
- Any 'inner' reusable workflows will ALWAYS need a workflow-level outputs block. If the reusable workflow is setting an output to be used by other workflows, a job-level outputs block will be required as well. 

### Process 

Passing outputs between chained reusable workflows can be a confusing thing to read through. This repository covers such a scenario to explain how using the workflows `workflow-a`, `workflow-b`, and `workflow-c`.

`workflow-c` is reusable, and is called from `workflow-b`. `workflow-b` is also reusable, and is called by `workflow-a`.
<br>
The cadence of calls is `workflow-a --calls--> workflow-b --calls--> workflow-c`

`workflow-a` can get the output from `workflow-c` like so:
1. `workflow-c` has both a job-level and workflow-level `outputs` block. A job with the id `set-workflow-c-output-job` has a step that sets an output which the job-level `outputs` block references. 
2. `workflow-c`'s workflow-level `outputs` block references the job-level reference so the output can be passed to a caller workflow.
2. `workflow-b` has a job with the ID of 'b-job-1' set up to call `workflow-c`. A subsequent job in `workflow-b` with the ID `b-job-2` has a job-level `outputs` block. 
A workflow-level `outputs` block references the value from the job-level `outputs` block. `workflow-b` is also resuable (`workflow-a` calls it), thus the need for the workflow-level `outputs` block. Note `workflow-b` does NOT need a job-level `outputs` block because `workflow-c`'s `outputs` are already exposed at the job-level?
b-job-2 has a step with the ID 'get-output-from-b-job-1' uses 'needs' and b-job-1's id to get the output from b-job-1. b-job-2's job-level `outputs` block will reference the step ID 'get-output-from-b-job-1' to get the value of the output. 
3. `workflow-a` has a job with the ID 'call-`workflow-b`' which shockingly, calls `workflow-b`. `workflow-a` has another job which uses uses 'needs' and call-`workflow-b`'s id to get the output from call-`workflow-b`. 

## Caller workflow and composite action:

Note: In this context, the 'caller workflow' refers to a workflow that calls a composite action. 
A composite action is essentially a bundle of steps that you include in the job. Outputs are defined at the top-level in the action.yml (An action.yml technically doesn't have a workflow-level, but you could view it as that). Since composite actions do not have a job (again, they're a bundle of job-agnostic steps), the `outputs` block references a step id directly (instead of having to reference a job) to get the output. 
In the caller workflow, a step (with an id) job calls the composite action. A future step in that caller workflow can use the step id to get the output (No `outputs` block needed, but you MUST reference the step that calls the composite action, not a step within the composite action). Another job in the caller workflow can reference this output as well using a job-level reference. This means that the job with the step that calls the composite action must have an `outputs` block defined at job-level.  

