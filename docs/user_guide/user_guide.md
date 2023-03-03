# anacision PLANNING User Guide


Guide on how to model standard use cases. Note that the requests that are illustrated in this section
may be shortened due to readability. The fully functional PLANNING requests are linked and can be
found [here](working_code_samples.md)

## Production tasks

The production tasks are the production processes that need to be scheduled. They can be single step processes or belong
to a chain of multiple processing steps (e.g. 1. cutting raw material, 2. milling the raw cut, 3. painting the finished part).
In this case, each of these steps needs to be modeled as a task and they can be linked through a predecessor successor relationship.

This section highlights some ways of modeling certain use cases. Refer to the PLANNING API documentation ([https://planning.anacision.ai/](https://planning.anacision.ai/))
 to get an in depth
description of the task object's attributes.

### Modeling multi-stage production

Multi-stage production scenarios, can be modeled using the ```predecessor_task_ids``` field.

The following examples are NOT functional. They only serve the purpose
of illustrating how the predecessor - successor relationship can be used to model multi-stage production.

```json hl_lines="12 19"
{
  "tasks": [
    {
      "id": "cutting",
     ...
      "predecessor_task_ids": []
    },
    {
      "id": "milling",
     ...
      "predecessor_task_ids": [
        "cutting"
      ]
    },
    {
      "id": "painting",
     ...
      "predecessor_task_ids": [
        "milling"
      ]
    }
  ]
}
```

This example represents three tasks that form a chain where the product is first cut, then milled and finally
painted. It cannot be painted before it has been milled and the milling cannot start before the cutting.
It is a typical flow shop scenario.

```json hl_lines="17-18"
{
  "tasks": [
    {
      "id": "milling_part_1",
          ...
      "predecessor_task_ids": []
    },
    {
      "id": "milling_part_2",
          ...
      "predecessor_task_ids": []
    },
    {
      "id": "assembly",
          ...
      "predecessor_task_ids": [
        "milling_part_1",
        "milling_part_2"
      ]
    }
  ]
}
```

This is an example of a processing chain where one step has multiple predecessors that are independent of each other.
The assembly cannot start before the single parts have been milled. But the single parts do not have any other
predecessor or successor relationship. As long as both parts are milled before the assembly starts, it is irrelevant
when the milling tasks take place.

### Setting allowed production windows
In many planning and scheduling setups, there is some kind of coarse planning or material resource planning that dictates
the time frame in which the task may be processed. The most common example for this is that the production has to take
place close to the confirmed due date to avoid building up inventory. At the same time, the procurement of raw materials
and the start of the processing of these materials need to be coordinated well.

The ```earliest_start_at``` attribute can be used to ensure that a task is not planned before the raw material has been sourced.
In turn, the ````due_at```` attribute represents the confirmed due date. If possible, the task will be finished before the due date.
Note that it is still possible, that the due date cannot be met due to insufficient production capacities.

```json
{
  "tasks": [
    {
      "id": "cutting",
            ...
      "earliest_start_at":"2022-02-02T07:00:00+00:00",
      "due_at":"2022-02-10T22:59:00+00:00"
    }
  ]
}
```

## Processing stations
The processing stations are the processing entities to which the tasks are scheduled to. In most cases
these are machines, but it can also be stations where tasks can be executed in parallel (e.g. paint shops). 
Therefore, each station can have multiple processing slots. Multiple processing slots means that multiple
steps can run in parallel. A station with 4 slots can process at most 4 tasks at once, whereas a station with a single slot
can be understood as a "conventional" machine, that can process one task at a time. A station can also have
unlimited processing slots. In this case, it can run an infinite number of steps at once. This makes sense to model
external production capacities through service providers.

If the PLANNING API is used in a production environment, it is most likely the case that the stations are not empty at 
the time of the scheduling execution, but have tasks that are being executed and resources that are mounted. This needs to 
be modeled for each production slot within the station.

```json
  "processing_slots": [
    {
      "id": "slot_01",
      "current_task": {
        "id": "task_01",
        "remaining_processing_finishes_at": "2022-02-02T13:00:00+00:00"
      }
      "current_mounted_resources": [
        "R11"
      ]
    }
  ]
```

In the code snippet above, the station is blocked until February 2nd at 1 pm. Therefore, the algorithm
will not schedule any tasks before this timestamp. Also, all predecessors of ```task_01``` will have to wait
until the task is finished. Likewise, the resource ```R11``` is blocked for the same time period.

## Shift definitions
There are three types of availability definitions: repeating shift times, planned downtimes (e.g. for maintenance) or
unlimited availability of 24/7. Stations, as well as resources can be assigned shift definitions during which
they can be utilized. The different types of availabilities can be combined (e.g. unlimited availability except for certain planned downtimes)


```json
{!user_guide/sample_code/availability_definitions.json!lines=5-27}
```
[link to fully functional example](working_code_samples.md#code_shifts)

In the snippet above, the availability combines to a regular shift on monday through friday between 7:00 and 15:00 (8 hours), 
with an additional downtime on February 3rd from 8:00 to 9:00 (1 hour). If a task were to start at the beginning of the 
shift, it would be paused after the first hour of processing for the planned downtime.

## Production equipment

Most production settings include resources. For the scheduling algorithm they can indicate two things: 

* Limited availability 
* Setup changes 

Resources that have the same skills or functionality have to be grouped together as ````resource_types````,
which in turn include the single resource units.

### Limited availability
If there is equipment that is shared between different stations but only available in a limited quantity,
it cannot be scheduled at more stations that it is available. To do this, it can be modeled explicitly as a resource type 
and linked to the tasks that require that piece of equipment.


<a name="sample_sufficient_resources"></a>
In this example, there is one type of resource that is required to process the tasks (```R10```). There are three tasks
that can be produced on three stations. Without any additional limitations, they could be processed at once,
with each on one station.

```json hl_lines="12 23 34 51 62 73 90 101 112 125-126"
{!user_guide/sample_code/sufficient_resource_units.json!lines=46-180}
```

[link to fully functional example](working_code_samples.md#code_sufficient_resources)

Which leads to the following production plan:
{!user_guide/sample_images/sufficient_resource_units.html!}

<a name="sample_insufficient_resources"></a>
However, when reducing the ```n_available_units``` to two units, only two tasks can be processed at once,
since the third task needs to wait for the resource to become available.

```json hl_lines="6-7"
{!user_guide/sample_code/insufficient_resource_units.json!lines=159-174}
```

[link to fully functional example](working_code_samples.md#code_insufficient_resources)

Which leads to the following production plan. Note that the third station is not being used, since
it would lead to a shorter makespan.
{!user_guide/sample_images/insufficient_resource_units.html!}


### Setup changes

There are two ways to model the duration of a changeover process: By modeling the time it takes
to remove one resource and set up another resource (```setup_duration_sec, removal_duration_sec```) in the
```Resource``` object or by explicitly modeling the ```deviating_changeover_durations``` as the required time
to change from one resource to another resource. This automatically includes the removal and setup process.

### Usage of removal and set up durations

<a name="sample_mounting_unmounting_v1"></a>

In this sample, the changeover process is modeled using the removal and setup times for each resource.
While ```task_1``` requires ```R10```, ```task_2``` requires resource ```R11```. Therefore, after ```task_1``` is
finished,
```R10``` needs to be removed and ```R11``` needs to be set up which takes the required durations. In this case,
20 seconds for the removal and 20 seconds for the setup.

```json hl_lines="12 31 46-47 55-56"
{!user_guide/sample_code/mounting_unmounting_v1.json!lines=15-77}
```

[link to fully functional example](working_code_samples.md#code_mounting_unmounting_v1)

Which leads to the following production plan:
{!user_guide/sample_images/mounting_unmounting_v1.html!}

<a name="sample_mounting_unmounting_reverse_order"></a>

Since the mounting and unmounting times can be
modeled independent of each other, the sequence of the setup makes a difference.
When switching the order of the tasks, the changeover duration between the tasks changes.
Instead of the 40 seconds from the example before, it is now reduced to 10 seconds for the removal
of ```R11``` and 10 seconds for the set up of ```R10```.

{!user_guide/sample_images/mounting_unmounting_reverse_order.html!}
[link to fully functional example](working_code_samples.md#code_mounting_unmounting_reverse_order)

#### Usage of deviating changeover times

<a name="sample_changeover_matrix_single_station"></a>

The deviating changeovers can also be used to explicitly model changeover matrices between the
different products independent of resources that have to be mounted or unmounted.
This can be the case if there is no sufficient information about all the resources
or if no resources are involved, but there is still a setup process (e.g. for cleaning).

For example, if there are three tasks that can be produced on one station with the following changeover
durations where the rows are the predecessor tasks and the columns the successors (it takes 200 seconds to change from
task_1 to task_3):

|        | task_1 | task_2 | task_3 |
|--------|--------|--------|--------|
| **task_1** | X      | 120    | 200    |
| **task_2** | 120    | X      | 30     |
| **task_3** | 60     | 90     | X      |

They can be modeled by introducing a new resource type ```production_on_station_1```

```json
{!user_guide/sample_code/changeover_matrix_single_station.json!lines=68-131}
```

[link to fully functional example](working_code_samples.md#code_changeover_matrix_single_station)

Which leads to the production plan, where the shortest overall changeover duration is chosen. In this case,
it is task_2 -> 30 sec changeover -> task_3 -> 60 sec changeover -> task_1

{!user_guide/sample_images/changeover_matrix_single_station.html!}

Multiple stations that have different influences on the changeover durations can be represented by
introducing a new resource_group for each station (```production_on_station_2, ...```) and adding
the relevant resources to each task's processing option.

#### Personnel

Personnel can be characterized through their shift times and skills. In this example, there is one type of worker which is
a machine operator that can process the tasks. There are two operators in the morning shift and one operator in the afternoon
shift. This is represented by ```"n_available_units":2``` and ```"n_available_units":1``` respectively.
Note that the flag ```release_resource_after_use``` is set to ```True``` which indicates that the resources
do not need to be unmounted but instead are released to perform the next task immediately after the task has ended.


```json
{!user_guide/sample_code/sample_workers.json!lines=43-44 93-142 172-173}
```
[link to fully functional example](working_code_samples.md#code_workers)

Setting the operator as a resource requirement within a task ensures that the operators are considered
when scheduling the task. In this case, the workers in both shifts can execute the task. If their skills were
such that only one of the two can operate the task, the other one has to be removed from the resource options.
Note that the skills can be modeled on a more granular model by creating a new resource type that groups together
worker of a certain skill. This is especially needed if two different skills are required to process a task at the same time.

```json
{!user_guide/sample_code/sample_workers.json!lines=19-41}
```

### Additional resources to perform setup changes

Finally, resources that are required to perform a setup change can also be included. Use cases for this are workers
that have a certain skill or equipment that is only needed for the setup change, but not the processing of the task.
This could be cleaning equipment or special tooling. These resources are also set as ````resource_requirements```` but
instead of setting the requirement within the task, it needs to be set within the resource type it corresponds to.

In this case, the resource type ```resource_1``` needs a worker to perform the mounting and unmounting process. The
setup resource requirements have to be modeled per station. This provides flexibility, in case different resources
are required on different stations (e.g. differently skilled personnel or equipment with different technical specifications.)

```json
{!user_guide/sample_code/sample_workers.json!lines=78-89}
```
[link to fully functional example](working_code_samples.md#code_workers)


## Production Goals
<a name="production_goals"></a>
<a name="sample_minimal_tardiness"></a>

anacision PLANNING provides different optimization targets that can be applied and
combined with each other. When optimizing different targets, they can be combined using
weights to indicate the relative importance of each target.

In some scenarios, it may be beneficial to only optimize one target, while in other, there can be
multiple targets that are relevant.

Consider a setting where there is one station that needs to process a set of tasks that require two different tools
depending on the product characteristics (a small tool vs. a large tool). The tasks
also have due dates by which they need to be fulfilled in order to meet customer demand. Tasks that are
not fulfilled in time means that their customer satisfaction will go down, which may lead to less incoming orders.

| Task Id | Processing duration | Due date   | Tool type |
|---------|---------------------|------------|-------|
| task_1  | 2 hrs               | 2022-02-02 | Small |
| task_2  | 3 hrs               | 2022-02-03 | Large |
| task_3  | 2 hrs               | 2022-02-02 | Small |
| task_4  | 2 hrs               | 2022-02-02 | Large |
| task_5  | 4 hrs               | 2022-02-03 | Small |
| task_6  | 3 hrs               | 2022-02-03 | Large |

Removing and setting up a new tool takes 30 mins respectively and the start of the production is on 2022-02-02 with
production taking place between 8:00 and 18:00.

Optimizing the setup by minimizing the number of tardy tasks looks as follows:


{!user_guide/sample_images/minimal_tardiness.html!}
[link to fully functional example](working_code_samples.md#code_minimal_tardiness)

In this production plan, all tasks are finished on time and the total changeover duration is three hours.
This corresponds to three switches between the small tool and the large tool. Since the only production
target is to reduce the delay, the algorithm has no incentive to further reduce the setup times.

To reduce the setup times, an additional target function can be added to reward plans that have the same
on-time delivery but with a shorter makespan and thus less setup times.

```json hl_lines="9"
{
  "optimization_targets": [
    {
      "weight": 1,
      "target_function": "tardiness"
    },
    {
      "weight": 0.1,
      "target_function": "makespan"
    }
  ]
}
```

{!user_guide/sample_images/combined_targets.html!}

In this plan, all tasks are still on time, but one resource change has been avoided. Compared to the single target
optimization of the tardiness, the plan is equally
optimal in terms of the due date fulfillment, but better in the resource changes since one hour of resource
change could be avoided.

Note that the weights of the target functions indicate the tradeoff between the goals. In this case,
a small weight for the ```makespan``` is sufficient, as it should not overshadow the
optimization of the tardiness. When weighting the resource change duration too heavily, it may be that additional changeovers are avoided at the cost of tasks not being on time.

{!user_guide/sample_images/overshadowing_changeover.html!}

One can combine as many targets as desired, but it should be done with care, since too many opposite optimization
targets means that there is no clear optimization direction. This can lead to suboptimal plans.


