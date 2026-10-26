# Variable Scheduler

**Project Specification | Version 0.1**

## 1. Project Overview

The goal is to build an automated employee scheduling system that generates schedules based on staffing requirements, employee availability, and scheduling constraints.

Unlike basic scheduling software that treats all declared availability equally, this system distinguishes between hours an employee *can* work and hours they *prefer* to work.

The initial implementation will prioritize simplicity, transparency, and correctness over optimization.

## 2. Initial Technology Stack

The first iteration will use only three files:

```text
scheduler/
├── employees.json
├── requirements.json
└── scheduler.py
```

- **`employees.json`**: Employee information and availability.
- **`requirements.json`**: Hourly staffing requirements.
- **`scheduler.py`**: Data processing, priority calculation, scheduling, validation, and output.

Python will use pandas for DataFrame operations. The core scheduling engine will remain independent of any web framework.

## 3. Scheduling Rules

### 3.1 Availability

Employee availability has two levels:

- **Hard availability:** Hours during which an employee is permitted to work.
- **Soft availability:** Hours an employee prefers to work, which must be a subset of their hard availability.

Hours that are hard-available but not soft-available remain valid scheduling options, but should generally be avoided when alternatives exist.

For the initial priority calculation, availability receives the following weights:

| Availability status | Weight |
|---|---:|
| Preferred availability | 1.0 |
| Fallback availability | 0.5 |
| Unavailable | 0.0 |

These weights are scheduling heuristics, not literal estimates of employee capacity.

### 3.2 Scheduling Constraints

Constraints will eventually be divided into two categories:

- **Hard constraints:** Rules that must never be violated, such as maximum weekly hours or prohibited shift lengths.
- **Soft constraints:** Preferences the algorithm should satisfy where practical, such as preferred shift lengths or limits on consecutive working hours.

The first iteration should establish this distinction in the design, even if only a small number of constraints are initially implemented.

### 3.3 Staffing Requirements

Each day and hour can specify:

- **Minimum staffing:** The minimum number of employees required.
- **Ideal staffing:** The preferred number of employees to schedule.

The minimum must not exceed the ideal. Hours without a staffing requirement do not need to be represented.

The initial version will not distinguish between employee roles, such as cashier or cook.

## 4. Input Data Structures

### 4.1 `employees.json`

Stores employee identifiers and their availability.

Example:

```json
{
  "employees": [
    {
      "employee_id": 1001,
      "first_name": "Alex",
      "last_name": "Martin",
      "hard_availability": {
        "monday": [9, 10, 11, 12, 13],
        "tuesday": [10, 11, 12, 13, 14]
      },
      "soft_availability": {
        "monday": [9, 10, 11],
        "tuesday": [11, 12, 13]
      }
    }
  ]
}
```

Hours are represented as integers corresponding to the start of an hourly block. For example, `9` represents 9:00 AM to 10:00 AM.

Employee IDs must be unique. Every soft-available hour must also be hard-available.

### 4.2 `requirements.json`

Stores the minimum and ideal staffing levels for each required hour.

Example:

```json
{
  "monday": {
    "11": {
      "minimum": 2,
      "ideal": 3
    },
    "12": {
      "minimum": 3,
      "ideal": 4
    }
  },
  "tuesday": {
    "11": {
      "minimum": 2,
      "ideal": 3
    }
  }
}
```

Requirements are defined per hour rather than per shift. The scheduling algorithm will determine employee assignments across those hours.

## 5. DataFrame Architecture

The Python script will construct four principal DataFrames.

### 5.1 DataFrame 1: Employee Availability

Derived from `employees.json`.

Recommended columns:

| Employee ID | Day | Hour | Hard Available | Soft Available |
|---|---|---:|---|---|
| 1001 | Monday | 9 | True | True |
| 1001 | Monday | 10 | True | True |
| 1001 | Monday | 11 | True | True |
| 1001 | Monday | 12 | True | False |

Each row represents one employee's availability for one hourly block.

This normalized structure makes it easier to filter employees by day, hour, and availability status.

### 5.2 DataFrame 2: Staffing Requirements

Derived from `requirements.json`.

Recommended columns:

| Day | Hour | Minimum Staff | Ideal Staff |
|---|---:|---:|---:|
| Monday | 11 | 2 | 3 |
| Monday | 12 | 3 | 4 |
| Tuesday | 11 | 2 | 3 |

Each row represents one hourly staffing requirement.

### 5.3 DataFrame 3: Scheduling Priority / Heat Map

Derived from the availability and requirements DataFrames.

Recommended columns:

| Day | Hour | Required Staff | Weighted Availability | Delta |
|---|---:|---:|---:|---:|
| Monday | 11 | 2 | 2.0 | 0.0 |
| Monday | 12 | 3 | 2.5 | -0.5 |

The primary calculations are:

**Weighted availability**

Sum the availability weight of each eligible, unassigned employee for the relevant hour.

**Delta**

`Delta = Weighted Availability - Required Staff`

A lower delta indicates a more constrained hour and therefore a higher scheduling priority.

- A positive delta indicates weighted availability beyond the minimum requirement.
- A zero delta indicates no weighted slack.
- A negative delta indicates that weighted availability is below the minimum requirement.

Delta is a heuristic for prioritization, not a guarantee that sufficient eligible employees exist. Actual eligibility must account for hard constraints.

The heat map will be recalculated as assignments are made, since assigning an employee to one hour can affect their availability for other hours.

### 5.4 DataFrame 4: Actual Schedule

Stores the assignments generated by the algorithm.

The internal representation should initially use one row per employee per assigned hour:

| Employee ID | Day | Hour |
|---|---|---:|
| 1001 | Monday | 11 |
| 1003 | Monday | 11 |
| 1002 | Monday | 12 |

This structure simplifies validation and analysis. A presentation-oriented version can later group employee IDs into a list for each day and hour.

The final schedule should be exportable to a readable file.

## 6. Scheduling Algorithm

The initial algorithm will use a greedy, iterative approach.

1. Load and validate both JSON files.
2. Construct the availability and requirements DataFrames.
3. Calculate the initial scheduling priority for each required hour.
4. Identify the most constrained hour using the lowest delta.
5. Select an eligible employee while respecting hard constraints.
6. Assign the employee to the hour.
7. Update the schedule and recalculate affected priorities.
8. Repeat until minimum staffing requirements are met or no valid assignment can be made.
9. Validate the completed schedule independently.
10. Save the schedule and report any unmet requirements or concerns.

The algorithm should not silently return an invalid schedule. If it cannot find a valid schedule, it must report the failure and identify the unmet requirements where possible.

The greedy approach may not always find a valid schedule even when one exists. This limitation is acceptable for the initial iteration and can guide future improvements.

## 7. Validation and Error Handling

Validation should be separated from the scheduling algorithm.

Initial checks should include:

- JSON files can be loaded and parsed.
- Required fields and data types are valid.
- Employee IDs are unique.
- Soft availability is a subset of hard availability.
- Staffing levels are nonnegative integers.
- Minimum staffing does not exceed ideal staffing.
- No employee is assigned during an unavailable hour.
- No employee is assigned more than once to the same hour.
- All implemented hard constraints are satisfied.
- Minimum staffing requirements are met.

The script should distinguish between invalid input, an infeasible schedule, and a successfully generated schedule.

## 8. Project Scope

### Must-have Features

- Load employee availability from JSON.
- Load staffing requirements from JSON.
- Construct the four principal DataFrames.
- Calculate scheduling priorities.
- Generate a schedule.
- Validate the schedule against implemented hard constraints.
- Save or export the resulting schedule.
- Report failures and unmet requirements.

### Should-have Features

- A schedule quality score.
- Warnings about unmet preferences or potential scheduling problems.
- Readable input and output formats.
- Clear reporting of why a schedule failed.

### Future Improvements

- A web interface using Flask and JavaScript.
- Convenient tools for editing employee availability and requirements.
- CSV import and export for manager usability.
- Employee roles and role-specific staffing requirements.
- More advanced optimization techniques if the greedy algorithm proves insufficient.

### Out of Scope for the Initial Version

- Agentic or autonomous decision-making functionality.
- Hosting and deployment.
- A graphical user interface.
- Advanced optimization before the basic scheduling engine works reliably.

## 9. Initial Development Milestones

### Milestone 1: Input Processing

Load both JSON files, validate their structure, and construct the availability and requirements DataFrames.

### Milestone 2: Priority Calculation

Generate the heat map DataFrame and rank required hours by scheduling priority. No employee assignments are necessary yet.

### Milestone 3: Basic Scheduling

Implement the greedy algorithm and generate employee-hour assignments.

### Milestone 4: Independent Validation

Check the generated schedule against all implemented hard constraints and staffing requirements.

### Milestone 5: Output and Reporting

Save the schedule and provide a summary of its validity, unmet requirements, and scheduling concerns.

## 10. Initial Acceptance Criteria

The first functional version is complete when it can:

1. Load valid employee and requirements JSON files.
2. Produce the availability, requirements, and priority DataFrames.
3. Generate a schedule where possible.
4. Verify that the schedule satisfies minimum staffing and all implemented hard constraints.
5. Save a valid schedule or clearly report why a valid schedule could not be produced.

The immediate next step is **Milestone 1: define and validate the two JSON input formats, then load them into pandas DataFrames.**
