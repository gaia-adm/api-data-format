# Gaia Events

**Gaia Dev Monitoring** is able to accept new measurements for different event types. Some fields for reported events are common for every event type and some are available for specific types.

## Event types

We are using **postfix** to define event type. Currently we are supporting following types:
- **change** - when updating record: change fields, attach files, add comments and similar
- **commit** - when committing a new changeset
- **testrun** - when running test, either by CI build job, command line or from Test Management

## Points

Multiple event points can be passed in single request. `points` is an array of events reported together.

## Common fields

There is a common set of fields for each event type.

- **event** - An *event type*
- **time** - A *time of event* in [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format
- **source** - An *event source* object: set of attributes that together describe an event source in unique way
- **id** - A *record id* object: set of attributes that uniquely define record in event source
- **tags** - A set of free tags. Each tag can be used to filter events. Avoid using tags with unlimited cardinality.

``` json
{
  "points": [
    {
      "event": "issue_change",
      "time": "2015-11-10T23:00:00Z",
      "source": {
        "server": "http://alm-saas.hp.com",
        "domain": "IT",
        "project": "Project A"
      },
      "id": {
        "uid": "1122"
      },
      "tags": {
        "workspace": "CRM",
        "user": "bob"
      }
    }
  ]
}
```

When multiple events share same fields, it's possible to define shared fields only once, at `points` level.

``` json
{
  "event": "issue_change",
  "source": {
    "server": "http://alm-saas.hp.com",
    "domain": "IT",
    "project": "Project A"
  },
  "tags": {
    "workspace": "CRM",
    "user": "bob"
  },
  "points": [
    {
      "time": "2015-11-10T23:00:00Z",
      "id": {
        "uid": "1122"
      },
      "fields": []
    },
    {
      "time": "2015-11-10T23:30:00Z",
      "id": {
        "uid": "1125"
      },
      "fields": []
    }
  ]
}
```

## issue_change event

This is an event issue change, reported from some Issue Tracking System (ALM, MQM, Jira, GitHub and others)

Event specific data:
- **fields** - An array of "field change" objects
  - **name** - name/label of changed field
  - **from** - (optional) previous field value
  - **to** - updated field value
  - **ttc** - (optional) time to change: time passed since last field change (hours)
- **comments**
- **attachments**

``` json
{
  "points": [
    {
      "event": "issue_change",
      "time": "2015-11-10T23:00:00Z",
      "source": {
        "server": "http://alm-saas.hp.com",
        "domain": "IT",
        "project": "Project A"
      },
      "id": {
        "uid": "1122"
      },
      "tags": {
        "workspace": "CRM",
        "user": "bob"
      },
      "fields": [
        {
          "name": "Status",
          "from": "New",
          "to": "Open",
          "ttc": 124
        },
        {
          "name": "Priority",
          "to": "2-Medium"
        }
      ],
      "comments": [
        {
          "topic": "re: Problem to delpoy on AWS",
          "text": "larin fdsfsdf, fsdfds fsdfsfs",
          "time_since_last_post(h)": 12.5
        }
      ]
    }
  ]
}
```

## code_testrun event

This is an event of single coded test execution (run). Coded test usually stored in code repository (or file system) and run as part of CI flow (build, test, deploy, test, etc.).
For coded tests **source** usually will be some SCM repository (and branch within it), while **id** will be code level unique identifier, for example in Java: package, class and method.
We assume that most coded testing frameworks resemble xUnit approach, where each test is an independent and stateless code function with an option to run additional functions to prepare (setup) and cleanup (teardown) test environment.

Event specific data:
- **result** - A *result of test execution* object
  - **status** - test status, can be one of: **passed** , **failed**, **error**, but not limited to these values
  - **error** - (optional) error details: exception message, stack trace, etc.
  - **setup_time** - (optional) test setup time
  - **teardown_time** - (optional) test cleanup (teardown) time
  - **run_time** - actual test execution time

``` json
{
  "points":  [
    {
      "event": "code_testrun",
      "time": "2015-11-10T23:00:00Z",
      "source": {
        "repository": "git://github.com/hp/mqm-server",
        "branch": "master",
      },
      "id" : {
        "package": "com.hp.mqm",
        "class": "FilterBuilder",
        "method": "TestLogicalOperators"
      },
      "tags": {
        "build_job": "backend_job",
        "browser": "firefox",
        "build_label": "1.7.0"
      },
      "result": {
        "status": "error",
        "error": "NullPointerException: ...",
        "setup_time": 35,
        "tear_down_time": 20,
        "run_time": 130
      }
    }
  ]
}
```

## tm_testrun

This is an event of single execution of test managed in some Test Management (TM) Repository (QC, ALM, TFS). This can be either manual (text/step based test) or automated (contains also some associated script) test.
For managed tests **source** usually will be some TM repository (and may be specific location within it: project, workspace, etc.), while **id** will be some unique identifier, for example in ALM: test id.

Event specific data:
- **result** - A *result of test execution* object
  - **status** - test status, can be one of: **passed** , **failed**, **skipped**, but not limited to these values
  - **run_time** - time it took to execute the test
  - **steps** - contains more detailed status reporting at individual step level
    - **name** - test step name
    - **status** - test step execution status
    - **run_time** - test step run time: how long did it take to run the step

``` json
{
  "points":  [
    {
      "event": "tm_testrun",
      "time": "2015-11-10T23:00:00Z",
      "source": {
        "server": "http://alm-saas.hp.com",
        "domain": "IT",
        "project": "Project A",
      },
      "id" : {
        "instance": "245",
        "test_id": "34",
      },
      "tags": {
        "workspace": "CRM",
        "testset": "Regression",
        "user": "john",
        "type": "Manual"
      },
      "result": {
        "status": "Passed",
        "run_time": 380,
        "steps" : [
          {
            "name": "step 1",
            "status": "Passed",
            "runt_time": 12
          },
          {
            "name": "step 2",
            "status": "Skipped"
          },
          {
            "name": "step 3",
            "status": "Passed",
            "runt_time": 368
          },
        ]
      }
    }
  ]
}
```
