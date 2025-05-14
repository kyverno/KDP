# Kyverno Design Proposal - Allow time-based window for policy enforcement behavior

ISSUE : [#2233](https://github.com/kyverno/kyverno/issues/2233)

Author: [Prateek Nandle](https://github.com/Prateeknandle)

## Contents

- [Kyverno Design Proposal - Allow time-based window for policy enforcement behavior](#kyverno-design-proposal---allow-time-based-window-for-policy-enforcement-behaviour)
  * [Overview](#overview)
  * [Proposal](#proposal)
  * [Goals](#goals)
  * [Example](#example)
  * [Implementation](#implementation)
    + [Change in Policy Behaviour]
    + [Activity Status Report]

## Overview

I want to propose a new feature for policies, which will allow Time-based window policy enforcement behavior. This new feature would help to change the activity of the policies, for a particular defined Time window, eg : not allowing config changes during the weekends. Also a report will be created to see which policies are active at present.

## Proposal 

Currently, there is no such concept which will allow the user/ admin to make a time window where a policy’s activity would be changed automatically. Once a policy is defined then only manually it could be changed.

Changing manually is a tedious job, for eg, if an admin wants that no developer should change config file during a time slot, let's say weekend, then the admin needs to manually change the policy. This is tedious, because every weekend the admin needs to change the policy manually.

After this proposal the problem would be solved, a new tag would be added in the policy named, so that within a specified time window the policy would be active and outside the window it’ll be inactive.

## Goals

1. To add features for allowing users to define a time window where they want the particular policy to be active.

2. To support users with a report for active/inactive status of policies at present time.

3. To support the same feature for the CLI `test` command, i.e. functionality to add a time window to specify active time for a policy.

## Example

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  activeWindow: "00 17 * * * - 00 23 * * * "
  rules:
  - name: check-for-labels
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "label 'app.kubernetes.io/name' is required"
      pattern:
        metadata:
          labels:
            app.kubernetes.io/name: "?*"
```
In the above policy we defined `activeWindow`, which contains the time-window for which policy would be active, i.e. when time window hits the current time according to UTC standards, `validationFailureAction` value will change to audit mode, which will change the policy behaviour.

## Implementation

### Change in Policy Behaviour

Adding a new variable under spec(in policy), which will provide a way to declare/define time windows, naming `activeWindow`. Example : `activeWindow: "00 15 1 4 * - 00 18 2 4 *"`
The values mentioned above are in following order - Minutes Hours Day_of_month Month Day_of_week
The string will split on spaces and ‘-’ and separate values will be assigned to StartWindow(starting time) and EndWindow(ending time).
```
type StartWindow struct {
    Minute     string
    Hour       string
    DayOfMonth string
    Month      string
    DayOfWeek  string
}
type EndWindow struct {
    Minute     string
    Hour       string
    DayOfMonth string
    Month      string
    DayOfWeek  string
}
func TimeWindow() {
    if spec.activeWindow != "" {
        a := strings.FieldsFunc(spec.activeWindow, Split)
        b := StartWindow{a[0], a[1], a[2], a[3], a[4]}
        e := EndWindow{a[5], a[6], a[7], a[8], a[9]}
    }
    func Split(r rune) bool() {
        return r == '' || r == '-'
    }
}
```

Validation - We need to validate the values that user will assign to `activeWindow`, the values should follow cron format and values should be in defined range.

For cluster/nodes which based on different time zones, we’ll use the system's time and convert it to UTC time zone, then we’ll split it on space, ‘-’& ‘:’ according to the format.
```
type Time struct {
    Minute     string
    Hour       string
    DayOfMonth string
    Month      string
    DayOfWeek  string
}

func matchTimeWindow() {
    now := time.Now()
    
    loc,_ := time.LoadLocation("UTC")
    time := now.In(loc) //current time in UTC timezone
    a := strings.FieldsFunc(time, Split)
    b := Time{a[5], a[4], a[3], a[2], time.Now().Weekday()}
    func Split(r rune) bool() {
        return r == '' || r == '-' || r == ':'
    }
}
```

For changing policy activity we need to change the value of `validationFailureAction` to audit, we use the above splitted times for matching time, below is just one case, more cases need to be addressed.
For this implementation we can create a new file under the `pkg/engine` folder.
```
func policyChange() {
    if s.Minute == t.Minute || s.Minute == * && s.Hour == t.Hour || s.Hour == * && 
    s.DayOfMonth == t.DayOfMonth || s.DayOfMonth == * && s.Month == t.Month || s.Month == * {
        if Policy.GetSpec().ValidationFaliureAction == audit || Policy.GetSpec().ValidationFaliureAction == '' {
            Policy.GetSpec().ValidationFaliureAction = enforce
        }
    }
    else if {.......}//Many cases are yet to be acknowledge
}
```

After adding the functionality we need to add Unit Tests and E2E Tests for the new added feature.

### Activity Status Report

For policy activity status report, we can use a global variable, let's say `activeStatus`, whenever the time window for inactivity starts, set `activeStatus` to false and when the time window ends, set `activeStatus` to true, and use the variable `activeStatus` for the report.
```
func policyChange() {
    if s.Minute == t.Minute || s.Minute == * && s.Hour == t.Hour || s.Hour == * && 
    s.DayOfMonth == t.DayOfMonth || s.DayOfMonth == * && s.Month == t.Month || s.Month == * {
        if Policy.GetSpec().ValidationFaliureAction == audit || Policy.GetSpec().ValidationFaliureAction == '' {
            Policy.GetSpec().ValidationFaliureAction = enforce
            activeStatus = false //We can use this variable for activity report.
        }
    }
    else if {.......}//Many cases are yet to be acknowledge
}
```