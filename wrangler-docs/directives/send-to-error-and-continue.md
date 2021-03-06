# Send to Error and continue processing

The SEND-TO-ERROR-AND-CONTINUE directive allows the filtering of records and directs the filtered
records that match a given condition to an error collector, but, continues processing with the record.
If the error collector is not connected as the next stage in a pipeline, then the filtered records will be dropped.


## Syntax
```
send-to-error-and-continue <condition> [[metric-name] [error-message]]
```

The `<condition>` is a EL specifing the condition that governs if the record
should be sent to the error collector. Optionally you can specify the metric
name that should be registered everytime a record is sent to error combined
with optional ability to specify a error message that should be recorded.


## Usage Notes

The most common use of the SEND-TO-ERROR-AND-CONTINUE directive is to evaluate data quality of the record.
This is a data cleansing directive to flag records that do not conform to specified rules.

The record is *NOT* sent to the error collector (if connected) when the condition for the record
evaluates to `true`. But, a internal state is maintained of the checks that record fail. 

## Example

Assume a record that has these three fields:

* Name
* Age
* DOB

As part of a data cleansing process, check that all the records being ingested follow
these rules:

* `Name` is not empty
* `Age` is not empty and not less than 1 or greater 130
* `DOB` is a valid date

These directives will implement these rules; any records that match any of these
conditions will be sent to the error collector for further investigation:

```
send-to-error-and-continue exp:{ Name == null }
send-to-error-and-continue exp:{ Age.isEmpty() }
send-to-error-and-continue exp:{ !date:isDate(DOB) }
send-to-error-and-continue exp:{ Age.isEmpty()} age_empty 'Age field is empty'
send-to-error-and-continue exp:{ Name == null} name_null 
send-to-error-and-continue exp:{ Age < 1 || Age > 130} 'Age not in range between 1 - 130'
```
Each invocation of `send-to-error-and-continue` will increment a internal transient variable `dq_total` and `dq_failure`  variable when the condition evaluates to `false`. Using the combination of transient variables, one can determine if it's worth proceeding further with processing of record. This can be achieved using the `send-to-error` to compute the percentage and set a threshold to emit the record as error.  

```
set-column error_rate (dq_failure / dq_total)*100
send-to-error error_rate > 50.0
```

OR 

```
send-to-error ((dq_failure / dq_total))*100 > 50.0
```

In this case, for every condition that is matched the input record is emitted on the output.
