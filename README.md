# Concourse Overops Resource

## Quick Set Up

Assuming there is already an instance of Concourse up, the next step is to attach the OverOps Resource to your existing pipeline. Configure the example ```overops-resource.yml``` and then run ```fly -t OO-example set-pipeline -c pipeline.yml -p oo-test``` to add it into your Build. Please replace the ```OO-example``` with your own names and make sure the configuration is the correct yml file.

## Configuration example

example file name: `overops-resource.yml`

```yaml
resource_types:
- name: overops-resource
  type: docker-image
  source:
    repository: overops/concourse-resource
    tag: latest

resources:
- name: overops-report
  type: overops-resource
  source:
    overops_url: https://api.overops.com
    overops_sid: S111111
    overops_api_key: ((overops_api_key))
    application_name: App1
    deployment_name: Dep1
    mark_unstable: true
    new_events: true
    resurfaced_errors: true
    debug: false

jobs:
  - name: demo
    serial: true
    plan:
      - task: make-a-version-file
        config:
          platform: linux
          image_resource:
            type: registry-image
            source: { repository: busybox }
          run:
            path: sh
            args:
              - -exc
              - date +%s > ./files/created_file
          outputs:
            - name: files
      - put: overops-report
        inputs:
          - files
        params:
          deployment_name:
            file: ./files/created_file
        get_params:
          mark_unstable: false
          debug: false
```

## Configuration parameters

### Reading parameters from files

Any parameter can be read from existing file on filesystem with following construction:

```yaml
- put: overops-report
  inputs:
    - files
  get_params:
    param_name:
      file: files/my/file
```

Where `inputs` are existing output from previous steps

### `deployment_name`

`deployment_name` is a parameter that used to version the resources and best to provide it dynamically from file (as example above)
into `params` section.

### All parameters references

All of the following parameters can be provided globally in the Resource `source` section as well as can be overwritten on per step basis
in the `get_params` section of `put` step.

`deployment_name` is an exception and need to be provided in the `params` section of `put` step, as described in the example above.

Parameter | Required | Default Value | Description
---------|----------|---------|---------
overops_url | true | --- | The OverOps API Endpoint(Saas: https://api.overops.com)
overops_sid | true | --- | The OverOps environment identifier (e.g S4567) to inspect data for this build
overops_api_key | true | --- | API Key for interaction with OverOps API
application_name | true | --- | [Application Name](https://doc.overops.com/docs/naming-your-application-server-deployment) as specified in OverOps
deployment_name  | true | --- | [Deployment Name](https://doc.overops.com/docs/naming-your-application-server-deployment) as specified in OverOps
regex_filter     | false | | A way to filter out specific event types from affecting the outcome of the OverOps Reliability report.
mark_unstable    | false | false | If set to `true` the build will be failed if any of the above gates are met
print_top_issues  | false | 5 | Prints the top X events (as provided by this parameter) with the highest volume of errors detected within the active time window, This is useful when used in conjunction with Max Error Volume to identify the errors which caused a build to fail
new_events       | false | false | If any new errors is detected, the build will be marked as failed
resurfaced_errors| false | false | If any resurfaced errors is detected, the build will be marked as failed
max_error_volume  | false | 0     | Set the max total error volume allowed. If exceeded the build will be marked as failed
max_unique_errors | false | 0     | Set the max total error volume allowed. If exceeded the build will be marked as failed
critical_exception_types | false | | A comma delimited list of exception types that are deemed as severe regardless of their volume.<br>- If any events of any exceptions listed have a count greater than zero, the build will be marked as unstable. Blank to skip this test.<br>*(For example: `NullPointerException,IndexOutOfBoundsException`)*
active_timespan  | false | 0 | The time window inspected to search for new issues and regressions. Set to zero to use the Deployment Name (which would be the current build).<br>_(For example: `1d` [d - day, h - hour, m - minute] would be one day active time window)_
baseline_timespan| false | 0 | The time window against which events in the active window are compared to test for regressions. If this gate is used, baseline time window is required.<br>_(For example: `14d` [d - day, h - hour, m - minute] would be a two week baseline time window.)_
min_volume_threshold| false | 0 | The minimal number of times an event of a non-critical type (e.g. uncaught) must take place to be considered severe.<br>  - If a New event has a count greater than the set value, it will be evaluated as severe and could break the build if its event rate is above the Event Rate Threshold.<br> - If an Existing event has a count greater than the set value, it will be evaluated as severe and could break the build if its event rate is above the Event Rate Threshold and the Critical Regression Threshold.<br> - If any event has a count less than the set value, it will not be evaluated as severe and will not break the build.
min_error_rate_threshold| false | 0 | Value in range `0-1`. The minimum rate at which event of a non-critical type (e.g. uncaught) must take place to be considered severe. A rate of 0.1 means the events is allowed to take place <= 10% of the time.<br>- If a New event has a rate greater than the set value, it will be evaluated as severe and could break the build if its event volume is above the Event Volume Threshold. <br>- If an Existing event has a rate greater than the set value, it will be evaluated as severe and could break the build if its event volume is above the Event Volume Threshold and the Critical Regression Threshold.<br>- If an event has a rate less than the set value, it will not be evaluated as severe and will not break the build.
regression_delta | false | 0 | Value in range `0-1`. The change in percentage between an event's rate in the active time span compared to the baseline to be considered a regression. The active time span is the Active Time Window or the Deployment Name (whichever is populated). A rate of 0.1 means the events is allowed to take place <= 10% of the time.<br>- If an Existing event has an error rate delta (active window compared to baseline) greater than the set value, it will be marked as a regression, but will not break the build.
critical_regression_delta | false | 0 | The change in percentage between an event's rate in the active time span compared to the baseline to be considered a critical regression. The active time span is the Active Time Window or the Deployment Name (whichever is populated). A rate of 0.1 means the events is allowed to take place <= 10% of the time.<br>- If an Existing event has an error rate delta (active window compared to baseline) greater than the set value, it will be marked as a severe regression and will break the build.
apply_seasonality | false | false | If peaks have been seen in baseline window, then this would be considered normal and not a regression. Should the plugin identify an equal or matching peak in the baseline time window, or two peaks of greater than 50% of the volume seen in the active window, the event will not be marked as a regression.
debug | false | false | For advanced debugging purposes only

## ARC Links

The ARC Links inside of the UI display after a build with the OverOps resource are not clickable they must be copy and pasted to be used.

## Behavior

### `check`

Checks OverOps API responsiveness

### `out`

Creates resource version and triggers the `in` script.

**Always trigger resource via `put` step**

### `in`

Generates the report based on OverOps events, if `mark_unstable` is set to `true` then fails the build, until reported issues are fixed.
