# gh-actions-outputs-experiments
Experiments with Github Actions outputs in various scenarios. This repository is a supplement to the article 'Github Actions: Almost Everything You Need To Know About Outputs' TODO LINK. A lot of the text written in this README has been lifted to write the article, so be sure to look at both.

# Outputs: Summary

In Github Actions, outputs are used to pass small pieces of data from one job/workflow step to another (not necessarily the same workflow). It's important to note that outputs are strings. They're usually one line, but can be multi-lined on occasion. 

The contents of an output depend on the use case, but common examples include things like version numbers, text versions of booleans (often tied to success/failure of some activity), commit SHAs, filepaths, and even temporary keys/tokens.

# Setting an output

To set an output, you must do so in a job step (or a script the step calls). Generally, that step must have an `id` set (details for this explained in **__The outputs block__** section). 

An output is set by writing to `GITHUB_OUTPUT` using `echo` (or other commands that write to a file, like `printf`) (You can see more details about the underlying surrounding logic related to `GITHUB_OUTPUT` in the **__More details about GITHUB_OUTPUT (Optional)__** section.)

Outputs can consist of one-line (most commonly), but can also be a multi-lined string. Refer to the corresponding subections for details on how to set each.

# Single-line output

Most outputs are single-lined.  

The following syntax is used to set a single-line output (pay particular attention to the `run` line):
<br>
```- name: Set output
     id: set-output-step
     run: echo 'output-name=some output value' >> $GITHUB_OUTPUT
```


# Multi-line output

Multi-line outputs can be set by treating the multi-line string as a here document (a piece of code that gets treated as if it were input/a file). Combined with `EOF` (End of File) syntax one can feed the multi-line string to `GITHUB_OUTPUT`. Using `EOF` is possible across all types of Github-hosted runners (`ubuntu-*` (Bash), `windows-*` (Powershell), and `macos-*` (Mac) (bash) )

The below syntax shows how to set a multi-line output from a Bash perspective:

```
echo "multi-line-output<<EOF" >> $GITHUB_OUTPUT
echo "This is the first line of a multi-line output." >> $GITHUB_OUTPUT
echo "This is the second line of a multi-line output." >> $GITHUB_OUTPUT
echo "EOF" >> $GITHUB_OUTPUT
```

## More details about GITHUB_OUTPUT (Optional)

`GITHUB_OUTPUT` is the variable that represents a special Github Actions-specific file. It's a runner-provided environment file that is technically different (i.e. its path is different) per step. Writing to `GITHUB_OUTPUT` is considered a [workflow command] (https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-commands), just like writing to `GITHUB_ENV`. Workflow commands are executed via using `echo` or by writing to a file--for `GITHUB_OUTPUT`, it's obviously the latter. 

`GITHUB_OUTPUT` can be viewed as a vector of how outputs get stored/referencable in outputs syntax. It's a bit hard to find sources on this (try the actions/runner repo), but generally what happens is that:
1. The `GITHUB_OUTPUT` file is written to
2. After the step finishes, the runner reads the file
3. `key=value` pairs are parsed and converted into step outputs syntax (`steps.<step-id>.outputs.<name-of-output>`)

# The outputs block


The outputs block is used to define the outputs. It's needed when passing outputs almost anywhere, except when passing outputs between steps in the same workflow job. 

Generally, the job-level (of the job whose step(s) set an output). The `outputs` block defines the value of the output by referencing the step that is setting the output (See the above **__Setting an output__** section).

When utilized, the outputs block can be present at job-level, workflow-level/top-level, or both depending on the context (see the various scenarios under **__Passing outputs between different entities__** for more information). 

It's important to note the outputs block gets its values at the __end__ of a job, not during/incrementally. While knowing this distinction usually isn't necessary during most output setup, it's good to remember when thinking about how outputs get set conceptually.

## General structure

The `outputs` block is a map/dictionary/hash that is un-nested--that is, all key value pairs are at the top level.
For example:

```
example-job:
    outputs:
        example-output-1: ${{steps.<step-id>.outputs.<output-name>}}
        example-output-2: ${{steps.<step-id>.outputs.<output-name>}}
```

The above is an example of a job-level outputs block. Witness how nothing is nested, as well as the general syntax.

## At job-level

When the `outputs` block is defined at the job-level (see previous section for an example), it will always reference a step id. The step id is used to point to where the output is getting its value from. 

Consider the following line:
```example-output-1: ${{steps.<step-id>.outputs.<output-name>}}```

Note: <> used to signify placeholder values. You don't use <> in actual references.
This is saying "set a job-level output named 'example-output-1' to the value being written to GITHUB_OUTPUT in the step with the id <step-id>. The step may be setting multiple outputs at once, so be sure to set example-output-1 to the value <output-name> is being set to in the step"

For example, say you had a step with the `id` `set-ex-output-1`
```- name: Set output
     id: set-ex-output-1
     run: echo "example-output-1=this is example-output-1\'s value value" >> $GITHUB_OUTPUT
```

Additionally, imagine the job-level outputs block has a line that looks like this:
```example-output-1: ${{steps.set-ex-output-1.outputs.example-output-1}}```

Putting those lines together, to define the job-level output named example-output-1, the outputs block "looks" for the step with the id set-ex-output-1, then sees that it has a line where the output example-output-1 is written to GITHUB_OUTPUT. In other words, the step-level output gets mapped to the job-level output.

You'll notice that the output name example-output-1 is the same at both the step-level and job-level. While that might seem confusing at first glance, this is actually a common approach. Keeping the naming consistent in this case actually makes the output setting/passing easier to follow. If desired, the outputs can be named differently if need be. If you wanted the step-level output name to be different, you would change it in the echo statement, and reflect that change in the corresponding reference.

## At workflow-level (top-level)

The outputs block is present at workflow-level/top-level in two different scenarios:

- In a reusable workflow
- In a composite action

In both of these scenarios, the outputs block has __is__ nested. Instead of a `key-name=value` syntax, the key is on its own level, having an optional `description` key and a required `value` key. 

### Reusable workflows

The outputs block in a reusable workflow is a child of workflow_call. This outputs block can map outputs from a job-level outputs block, if present. See **__Passing Outputs between different entities: From reusable workflow and caller workflow__** for more information.

# Passing outputs between different entities

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

### `outputs` block Location
No need to define an outputs block--if the outputs are only to be used in the same job they are set, steps can refer to these outputs using <step-id> syntax (see below). 

### Process
1. A step (with an id) within the job sets the output,
2. Another, further down step references the step's id to use the output (ex. `steps.<step-id>.outputs.output-name`)

## Same workflow, different jobs

### `outputs` block Location

The `outputs` block is defined at the job level (of the job whose step(s) set an output). The `outputs` block defines the value of the output by referencing the step that is setting the output. 

### Process

To explain this process, imagine this example scenario: There is a 'Job A' whose id is 'job-a'. This job has a step that sets the value of an output. A different job, 'Job B' needs to reference this output. 

1. 'job-a' has a step with the ID 'job-a-step-id'. The job-level `outputs` block has an output named example-output, whose value is defined using the following syntax steps.job-a-step-id.outputs.example-output. 
2. At the end of job-a, the outputs block is updated.
3. 'job-b' can reference this output from job-a: it uses 'needs' to indicate it is dependent on job-a and thus wait until it completes. To get the example-output value, 'job-b' references job-a's output using needs.job-a.outputs.example-output

## From reusable workflow and caller workflow:

### `outputs` block Location

Before continuing, note that a reusable workflow can itself function as a caller workflow. We shall refer to such a workflow as a 'mixed reusable workflow' or 'mixed caller workflow'. This is opposed to a 'pure caller workflow'--that is, a workflow that only calls reusable workflows, and is NOT called itself.  

#### **Caller Workflow** 

A pure caller workflow (that is a workflow that only calls reusable workflows, and is NOT called itself) does not necessarily need an outputs block--for example, see `workflow-a.yml`. That workflow's first job `call-workflow-b` calls a reusable workflow (which sets an output that `call-workflow-b` will have access to). `call-workflow-b` does not need to set any other output, so the other job in workflow `workflow-a.yml` can reference `call-workflow-b`'s output directly.

A mixed reusable workflow/mixed caller workflow will always have an `outputs` block defined at the workflow level so it can pass its outputs to the caller workflow that called it. As a mixed caller workflow is a type of reusable workflow, it may or may not also have a job-level outputs block (see below section for more information)

#### **Reusable Workflow** 

For reusable workflows, the `outputs` block has the potential to be defined at the workflow level AND at the job level. In other words, there is a potential for multiple `outputs` blocks. At workflow-level, the `outputs` block is a direct child of `workflow_call`. 

A reusable workflow will have a job-level `outputs` block when it has a job that directly sets an output via the typical writing to GITHUB_OUTPUT (see **__Setting an output__** section). It will not need a job-level `outputs` block if it is not setting any outputs. Additionally, it will not need a job-level `outputs` block if it is calling another reusable workflow to get an output, even if it has other jobs that may need to access that output (similar to a pure caller workflow; see the first paragraph in the **__Caller Workflow__** section above). 

### Process

<ins>Caller Workflow</ins>
1. The caller workflow executes a job that calls the reusable workflow. 
2. Since a workflow-calling job does not and can not involve any steps, only a subsequent job(s) can reference the output. The subsequent job will use 'needs' to indicate it is dependent on the workflow-calling job and also reference the workflow-calling job using the job id. The syntax of that would be something like `needs.<calling-workflow-job-id>.outputs.<name-of-output>`. `<name-of-output>` is the name of the workflow-level output in the reusable workflow.
<br>
<ins>Reusable workflow</ins>

1. A step (with an `id`) within a reusable workflow job sets the output value. 
2. Once the job is complete, the job-level `outputs` block sets the value of its output by referencing the step `id` 
and output name (ex. `steps.rw-step-id.outputs.example-rw-step-output`). 
3. The workflow-level `outputs` block gets its value from the job-level output (ex. `jobs.rw-job-id.outputs.example-rw-job-output`). It is important to emphasize this happens AFTER any job-level blocks are resolved. 

## Chained Reusable Workflows

The concept of a chained reusable workflow is resuable workflows being called in succession (i.e. caller calls a reusable workflow, which in turn calls another reusable workflow, ad infinitum)

### `outputs` block Location

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
1. `workflow-a` has a job with the ID `call-workflow-b` which shockingly, calls `workflow-b`. 
(`workflow-a` has another job `echo-output-from-b` which uses uses `needs` and call-`workflow-b`'s id to get the output from call-`workflow-b`.) 
2. `workflow-b` has a job with the ID of `get-workflow-c-output` which calls `workflow-c`.
3. `workflow-c` has both a job-level and workflow-level `outputs` block. A job with the id `set-workflow-c-output-job` has a step that sets an output which the job-level `outputs` block references. When the job finishes, the __job-level__ outputs block `workflow-c-output` is assigned a value.
4. After `set-workflow-c-output-job` completes, `workflow-c`'s workflow-level `outputs` block references the job-level `workflow-c-output` reference, so that the __workflow-level__ `workflow-c-output` can be given a value and thus be passed to a caller workflow. 
5. After the `get-workflow-c-output` job is complete, the workflow-level outputs block in `workflow-b` gets the value of `workflow-c-output` via referencing `get-workflow-c-output` like so `jobs.get-workflow-c-output.outputs.workflow-c-output`
6. `workflow-a`'s echo-output-from-b job fires since `call-workflow-b` completed. It is able to reference  `workflow-c-output` since this is made available via calling `workflow-b`

## Caller workflow and composite action:

Note: In this context, the 'caller workflow' refers to a workflow that calls a composite action. 

A composite action is essentially a bundle of steps that you include in the job. Those steps are stored in the action.yml file

### `outputs` block Location

Outputs are defined at the top-level in the action.yml (An action.yml technically doesn't have a workflow-level, but you could view it as that). 

### Process

Since composite actions do not have a job (again, they're a bundle of job-agnostic steps), the `outputs` block references a step id directly (instead of having to reference a job) to get the output. 

1. In the caller workflow, a step (with an id) job calls the composite action. 
2. In the composite action, a step (with an id) sets an output.
3. In the composite action, the top-level outputs block takes the value from the composite action step.
4. In the caller workflow:
- A future step can reference the step id that called the composite action to get the output (No `outputs` block needed, but you MUST reference the step that calls the composite action, NOT a step within the composite action). 
- Another job in the caller workflow can reference this output as well using a job-level reference. This means that the job with the step that calls the composite action must have an `outputs` block defined at job-level.  

