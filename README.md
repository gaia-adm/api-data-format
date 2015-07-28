# Gaia Events

**Gaia Dev Monitoring** is able to accept new measurements for different event types. Some fields for reported events are common for every event type and some are available for specific types.

## Event types

We are using **postfix** to define event type. Currently we are supporting following types:
- **change** - when updating record: change fields, attach files, add comments and similar
- **commit** - when committing a new changeset
- **testrun** - when running test, either by CI build job, command line or from Test Management

## Event format

There are several event types supported.
Each type has a number of elements that unique only for this specific type but there are some common parts:

## Common fields

There is a common set of fields for each event type.
- **event** - An *event type* (**mandatory**, unique string)
- **time** - A *time of event* in [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format; for example: 2015-07-28T23:00:00Z. Time is set to current time by metrics-gateway-service, if not provided as a part of event.
- **source** - An *event source* object: set of attributes that together describe an event source in unique way. Generally it is a map where both keys and values are string. All values of source elements become tags in database.
- **id** - A *record id* object: set of attributes that uniquely define record in event source. Generally it is a map of string keys and object values. All values of id elements become data fields in database.
- **tags** - A set of free tags. Each tag can be used to filter events. Tags cardinality is potentially limited; only elements with the limited set of values should be used as tags. Generally it is a map where both keys and values are string. All values of source elements become tags in database.

Multiple events can be passed in single request as a part of **JSON Array** (even you send only one event); see more details [here]. (https://github.com/gaia-adm/metrics-gateway-service/)
Every event must include all relevant fields described above, so that it can be sent as single event without changing, if you need.

**Example**:
``` json
[
    {
        "event": "issue_change",
        "time": "2015-07-27T23:00:00Z",
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
    },
    {
        "time": "2015-07-27T23:00:00Z",
        "id": {
            "uid": "2341"
        },
        "event": "test_change",
        "source": {
            "server": "http://alm-saas.hp.com",
            "domain": "IT",
            "project": "Project A"
        },
        "tags": {
            "workspace": "CRM",
            "user": "bob"
        },
        "fields": [
            {
                "name": "State",
                "from": "Maintenance",
                "to": "Ready",
                "ttc(d)": 11
            }
        ],
        "steps": [
            {
                "new": 10,
                "modified": 3,
                "deleted": 1
            }
        ],
        "attachments": [
            {
                "name": "readme.docx",
                "size": "1.3M"
            }
        ]
    }
]
```

## issue_change event

This is an event issue change, reported from some Issue Tracking System (ALM, MQM, Jira, GitHub and others)

Event specific data:
- **fields** - An array of "field change" objects. Each event **must** have field name ("to") and a new value ("to"). It also can include more attributes, like "from" (old value), ttc (time passed since previous change) or any other - just as a valid key/value pair
- **comments** - Generally, List<Map<String, Object>> that includes data about comments updated on the issue

**Example**:
``` json
{
        "event": "issue_change",
        "time": "2015-07-27T23:00:00Z",
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
```

## test_change event

This is an event issue change, reported from some Issue Tracking System (ALM, MQM, Jira, GitHub and others)

Event specific data:
- **fields** - An array of "field change" objects. Each event **must** have field name ("to") and a new value ("to"). It also can include more attributes, like "from" (old value), ttc (time passed since previous change) or any other - just as a valid key/value pair
- **attachments** - Generally, List<Map<String, Object>> that includes data about the test attachments
- **steps** - Generally, List<Map<String, Object>> that includes summary data about the test attachments. Note that only summary data about steps is supported by now, so that it should be a list with single element (see example below)

**Example**:
``` json
{
        "time": "2015-07-27T23:00:00Z",
        "id": {
            "uid": "2341"
        },
        "event": "test_change",
        "source": {
            "server": "http://alm-saas.hp.com",
            "domain": "IT",
            "project": "Project A"
        },
        "tags": {
            "workspace": "CRM",
            "user": "bob"
        },
        "fields": [
            {
                "name": "State",
                "from": "Maintenance",
                "to": "Ready",
                "ttc(d)": 11
            }
        ],
        "steps": [
            {
                "new": 10,
                "modified": 3,
                "deleted": 1
            }
        ],
        "attachments": [
            {
                "name": "readme.docx",
                "size": "1.3M"
            }
        ]
    }
```

## code_testrun event

This is an event of single coded test execution (run). Coded test usually stored in code repository (or file system) and run as part of CI flow (build, test, deploy, test, etc.).
For coded tests **source** usually will be some SCM repository (and branch within it), while **id** will be code level unique identifier, for example in Java: package, class and method.
We assume that most coded testing frameworks resemble xUnit approach, where each test is an independent and stateless code function with an option to run additional functions to prepare (setup) and cleanup (teardown) test environment.

Event specific data:
- **result** - A *result of test execution* object. It **must** include a status field and may include also other fields, like "runTime" (test execution time), "errorString" or any other - just as a valid key/value pair

**Example**
``` json
{
    "event": "code_testrun",
    "time": "2015-07-27T23:00:00Z",
    "source": {
        "repository": "git://github.com/hp/mqm-server",
        "branch": "master"
    },
    "id": {
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
```

## tm_testrun event

This is an event of single execution of test managed in some Test Management (TM) Repository (QC, ALM, TFS). This can be either manual (text/step based test) or automated (contains also some associated script) test.
For managed tests **source** usually will be some TM repository (and may be specific location within it: project, workspace, etc.), while **id** will be some unique identifier, for example in ALM: test id.

Event specific data:
- **result** - A *result of test execution* object. It **must** include a status field and may include also other fields, like "runTime" (test execution time), "errorString" or any other - just as a valid key/value pair
  - **steps** - Unlike code_testrun result mentioned earlier, tm_testrun result can include an array of steps. Step attributes are not limited (just a valid key:value pairs) but it makes sense to send attributes like name and status at least

**Example**:
``` json
{
    "event": "tm_testrun",
    "time": "2015-07-27T23:00:00Z",
    "source": {
        "server": "http://alm-saas.hp.com",
        "domain": "IT",
        "project": "Project A"
    },
    "id": {
        "instance": "245",
        "test_id": "34"
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
        "steps": [
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
            }
        ]
    }
}
```

## codeCommit event

This event represents single commit done in any SCM system.

Event specific data:
- **changedFilesList** - list of files affected by the commit. Generally, it is List<Map<String,Object>> with any reasonable parameters (map per file, list for all files)

**Example**:
``` json
{
    "event": "code_commit",
    "time": "2015-07-27T23:00:00Z",
    "source": {
        "repository": "git://github.com/hp/mqm-server",
        "branch": "master"
    },
    "id": {
        "uid": "8ad3535acb2a724eb0058fa071c788d48ab6978e"
    },
    "tags": {
        "user": "alex"
    },
    "files": [
        {
            "file": "README.md",
            "loc": 10
        },
        {
            "file": " src/main/java/managers/RabbitmqManager.java",
            "loc": -14
        }
    ]
}
```

## general event

This event is introduced mainly for *internal testing purpose* and can represent anything you need

Event specific data:
- **data** - Map<String, Object> that keeps data collected

**Example**:
``` json
{
    "event": "general",
    "time": "2015-07-27T23:00:00Z",
    "source": {
        "origin": "behind the scene"
    },
    "id": {
        "uid": "12345"
    },
    "tags": {
        "tag1": "foo",
        "tag2": "boo"
    },
    "data": {
        "field1": "value1",
        "field2": "value2",
        "field3": 3
    }
}
```