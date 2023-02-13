:exclamation: The table of contents can be found by clicking on the 3-dot menu at the top-left of this document. :exclamation:

# Summary

Greetings! This guide is composed of "foundational building blocks" and is meant to act as learning examples for the LogScale Query Language. The official LogScale documentation page can be found here:

https://library.humio.com/

# Building Blocks

## Add a single field to `groupBy` results

Let's say you need to use `groupBy()` for aggregation, but also need to include fields such as the `aid`, `aip`, `event_simpleName`, etc. This is where the `stats()` and `collect()` functions come into play. 

Here's a simple example of adding `DomainName` to the results of grouping by `ComputerName` and `FileName`.  This says "group everything by unique pairs of `ComputerName` and `FileName`, and then collect all of the `DomainName` values from each of those unique pairings. 

```
groupBy([ComputerName, FileName], function=collect(DomainName))
```

The results look like this:

<img src=./images/image-2022-11-30_10-16-22.png width=400>

## Add additional fields to `groupBy` results

Instead of the example above, you'd like to collect multiple fields instead of just one. The correct way of doing this is by using an array:

```
groupBy([ComputerName, FileName], function=collect([DomainName, MD5HashData, SHA256HashData]))
```

Notice the `[ ... ]`? That says "pass all of these into the function, and display the output". This is the result:

<img src=./images/image-2022-11-30_10-29-57.png width=400>

## Add a `count` to `groupBy` results when using `collect`

By default, groupBy() without a `function=` argument includes a `count()` function. The result is the `groupby()` results have a `_count` field that displays the total count for each aggregation. However, once you add a `function=` to that, the `count()` is no longer in the results. The solution? Simply add the `count()` function into an array.

```
groupBy(ComputerName, function=[collect(DomainName), count()])
```

The results now include count() and look like this:

<img src=./images/image-2022-11-30_10-38-7.png width=400>

## Pass a `groupBy` result to `timechart`

The `groupBy()` function removes the `@timestamp` field by default. This generally doesn't matter unless you're trying to use something like `timechart()` which requires `@timestamp` to work correctly.

```
groupBy(ComputerName, function=selectLast(@timestamp))
| timechart(ComputerName)
```

If you need to include other fields along with the timestamp, you can pass them all in an array:

```
groupBy(ComputerName, function=[collect([UserName, DomainName]), selectLast(@timestamp)])
| timechart(ComputerName)
```

## Assign or create a dynamic field

The documentation lists a few different methods. By far, the easiest way is to use the `:=` shorthand. The field before `:=` will be assigned the value of whatever is after it. This can be other fields, functions, strings, etc. For example, if bytes already has a byte count and you'd like to convert that to megabytes:

```
megabytes := bytes * 0.000001
```

If you'd like to have `thisBytes` created and assigned the value of `thatBytes`:

```
thisBytes := thatBytes
```

If you'd like to find the average string length of `@rawstring`:

```
eventLength := length(@rawstring)
| avg(eventLength)
```

Keep in mind there fields and values are dynamic. They do not exist outside of the query results, and will not be permanently added to the ingested logs. 

## Round a number to 2 decimal places

The `format()` function is extremely powerful. It allows you to manipulate data using printf formatting. The `round()` function rounds to the nearest integer and does not include decimals. 

This example will round the value of `thatSize` and save it to the field `thisSize` with two decimal places.

```
thisSize := format("%.2f", field = thisSize)
```

`%.1f` would be 1 decimal place, `%.3f` would be 3 decimal places, etc.

## Do a `join` statement

The `join()` function is generally used when you have two query results that you'd like to combine, and both results share some common value. There is also a `selfJoin()` and `selfJoinFilter()` function for certain situations, both described in the official documentation. Please note the field names can be different: it's the *value* of those fields that should be identical. Background on the example:

- We want to combine `ProcessRollup2` and `NetworkListenIP4` to find processes that have created a listener on a port. 
- `ProcessRollup2` contains the process information. The field we're using in the results is `TargetProcessId` to designate the ID of the process.
- `NetworkListenIP4` contains the listener information. Instead of `TargetProcessId`, it uses `ContextProcessId`. This also designates the ID of the process.
- Both `TargetProcessId` and `ContextProcessId` share a common value, e.g. 123456 would represent the same process.

And the `join()` to combine them:

```
#event_simpleName=ProcessRollup2
| join({#event_simpleName=NetworkListenIP4 LocalPort<1024 LocalPort!=0}, field=TargetProcessId, key=ContextProcessId, include=[LocalAddressIP4, LocalPort])
```

What does that query actually mean?

- The first query should be the larger of the two. In this case, `#event_simpleName=ProcessRollup2` is the larger query.
- The query within `join()` should be the smaller of the two queries, and is known as the *subquery*. It's the query within the `{ ... }` after the `join()` function. The query is simple enough: `#event_simpleName=NetworkListenIP4 LocalPort<1024 LocalPort!=0`.
- The `field=` and `key=` are the two fields that share a common *value*.
- `field=` is tied to the larger query, i.e. `#event_simpleName=ProcessRollup2`. This is the field `TargetProcessId` in `ProcessRollup2` events.
- `key=` is tied to the smaller query, i.e. `#event_simpleName=NetworkListenIP4 LocalPort<1024 LocalPort!=0`. This is the field `ContextProcessId` in `NetworkListenIP4` events.
- The `include=[ ... ]` is a list of fields that should be pulled from the *smaller* query into the results. 

And a visualization of everything that was just described:

<img src=./images/image-2022-11-30_17-6-18.png width=400>

## Deal with time zones

LogScale is able to deal with most time zone situations. Two major items to keep in mind:

- Everything internal to LogScale is based around UTC.
- Logs in the UI are displayed relative to the local time zone reported by the browser, i.e. UTC is converted to the user's local time. 

Further details on how LogScale leverages different time-related fields:

- `@rawstring` generally includes some type of timestamp, e.g. a syslog header. 
- `@timestamp` is the time field used for everything by default. This is in UTC.
  - If the parser can grab the time out of `@rawstring`, then it's converted to UTC and stored as `@timestamp`. This leverages the time zone specified in the logs.
  - If the time zone is not specified in the log, e.g. it's local time but does not include a time zone, then by default it's assumed it be UTC. This can cause events with a broken timestamp to appear "in the future", e.g. GMT-5 without a time zone would be logged 5 hours ahead.
- `@ingesttimestamp` is the time in UTC that the event was ingested by LogScale. 
- If a timestamp can't be found, then `@timestamp` is set as the `@ingesttimestamp`. 
- `@timestamp.nanos` is the nanoseconds that are sometimes included in certain timestamps.
- `@timezone` is the assumed time zone that was parsed from the logs. 

The timestamp is handled by the parser assigned to the ingest token. These are primarily `findTimestamp()` and `parseTimestamp()`. The `findTimestamp()` function is generally the "easy" function but assumes the event has a correct timestamp. The `parseTimestamp()` function allows you to be extremely specific about the time format and time zone. For example, let's say you have logs that were being sent over in Eastern time but didn't include a time zone:

```
parseTimestamp("MMM [ ]d HH:mm:ss", field=@timestamp, timezone="America/NewYork")
```

This says "parse the `@timestamp` field in that particular format, and assume it's in the America/NewYork time zone". Plenty of examples are included in the default parsers. 

## Compare the last 31-60 days to the previous 30 days

### `case` method

This can be done by utilizing `case {}` statements and comparing the `@timestamp` field. You'll still need to do a 60-day query since the query will encompass all results. It'll look like this:

```
// Run this query over the last 60 days.
#event_simpleName = ProcessRollup2
| case {
    test(@timestamp < (start()+(30*24*60*60*1000))) | eventSize(as=eventSize31to60);
    * | eventSize(as=eventSize0to30);
}
| stats([avg(as=avg31to60, field=eventSize31to60), avg(as=avg0to30, field=eventSize0to30)])
```

- The first line is a basic filter looking for `ProcessRollup2` events.
- The next line is a simple case statement. It says "if the `@timestamp` is older than 30 days ago, save the event size in the `eventSize31to60` variable.
- The second part of the case statement says "everything else gets the event size saved as the `eventSize0to30` variable.
- The last part grabs the average of both of them and displays them as the output.

The end results looks like this:

<img src=./images/image-2022-12-2_9-20-9.png width=400>

### `bucket` method

This can also be accomplished via the `bucket()` function. The bucket size should be divided by the number of values you'd like to compare. For example, if you're looking at 3 30-day windows over 90 days, each bucket should be 30 days. Please note that you might need to lower the search timeframe if you end up with an extra bucket, e.g. change it to 89 days if you specify a 30-day bucket but end up with 4 buckets. The query looks like this:

```
thisSize := eventSize()
| bucket(span=30d, function=avg(thisSize))
```

<img src=./images/image-2022-12-2_9-26-37.png width=400>

## Add `ComputerName` or `UserName` to FDR search results

Not all FDR events include a `ComputerName` or `UserName` field by default. In those cases, we can add the fields at query time. Version 1.1.1 of the FDR package includes a scheduled search that creates a CSV lookup file every 3 hours. You can find this file by clicking on the `Files` link at the top of the LogScale page. You should see `fdr_aidmaster.csv` in the file list, assuming the scheduled search is running as expected.

### Adding the `ComputerName`

The CSV is used to look up the ComputerName from the aid via the match() function. You would simply add this to your query:

```
| match(file="fdr_aidmaster.csv", field=aid, include=ComputerName, ignoreCase=true, strict=false)
```

A more robust version can be saved as a search, and the saved search referenced in subsequent queries as a user function:

```
// First look for ones missing a ComputerName.
| case {
    // Identify any events that have an aid but not a ComputerName.
    // Neither of these overwrite the value if it already exists.
    aid=* AND ComputerName!=*
      // Grab the ComputerName from the aidmaster file.
      | match(file="fdr_aidmaster.csv", field=aid, include=ComputerName, ignoreCase=true, strict=true);
      // Assign the value NotMatched to anything else. 
    * | default(field=ComputerName, value=NotMatched);
  }
```

If that were saved as `AddComputerName`, then it could be called in a query by using `$"AddComputerName"()`.

### Adding the `UserName` 

The `UserName` field can be added via a `join()` query. This will also show the last known user on the aid in question. Keep that in mind if there are multiple users over an extended timeframe, i.e. it will only be reporting the last user. 

```
| join({#event_simpleName=UserLogon}, field=aid, include=UserName, mode=left)
```

A more robust version of this is included in the package linked above. It can also be called as a function, and is included in several of the example queries. 

```
// Grab the UserName. This excludes any of the generic Windows usernames.
default(field=UserName, value="NotMatched")
| join({#event_simpleName=UserLogon | UserName!=/(\$$|^DWM-|LOCAL\sSERVICE|^UMFD-|^$)/}, field=aid, include=UserName, mode=left)
```

## Add a dynamic URL to query results

The `format()` function can be used to create dynamic URLs, e.g. a URL that takes a value from the search results as an input. This saves the user from needing to copy and paste into another browser tab. For example:

```
format("[Link](https://example.com/%s)", field=repo, as=link)
```

This says "generate a markdown URL that shows up as `link` in the results, with the `%s` being replaced by the value of the `repo` field".

## Pass two averages to a timechart

Let's say you have two fields you'd like to average and pass to a `timechart()` function. However, both functions will create an `_avg` field. This causes `timechart()` to generate an error about two events having the same field but different values. The solution is to rename the fields that are generated by the `avg()` function:

```
timechart(function=[avg(ConfigStateHash, as=avgConfigStateHash), avg(ConnectTime, as=avgConnectTime)])
```

This creates two averages, `avgConfigStateHash` and `avgConnectTime`, and then passes them into the `timechart()` function. 

## Do a regex extraction without filtering data

LogScale allows you to dynamically create fields using named capture groups. For example, let's say you want to create the field `netFlag` from certain events, but still pass the results through that don't match. The solution is to add the `strict=false` flag to the `regex()` function. This means "extract if it matches, but still pass the data through even if it doesn't match" in the query. 

```
#event_simpleName=ProcessRollup2 event_platform=Win
| ImageFileName=/\\(whoami|net1?|systeminfo|ping|nltest)\.exe/i
| regex("net1?\s+(?<netFlag>\S+)\s+", field=CommandLine, flags=i, strict=false)
```

## Get markdown URLs to display as URLs instead of strings when using `groupBy`

Let's say you have a field that contains a markdown URL as the value. You want that displayed as a clickable URL in the `groupBy()` output. The solution is to remove the field from `collect()` and place it into a `selectLast()` instead. Otherwise, you end up collecting multiple URLs into the same field. 

```
| groupBy([aid, ParentProcessId], function=[collect([ComputerName, UserName, ParentProcessId, ParentBaseFileName, FileName, CommandLine], limit=10000), count(FileName, as="FileNameCount"), count(CommandLine, as="CommandLineCount"), selectLast("Process Explorer")], limit=max)
```

<img src=./images/image-2022-12-15_13-12-39.png width=400>

## Get the first and last event of a `groupBy`

Let's say you're doing a `groupBy()` for a particular event, but you'd like to see the time of the first and last occurrence in the results. You'd do this:

```
groupBy(UserName, function=[max(@timestamp, as=lastSeen), min(@timestamp, as=firstSeen)])
| firstSeen:=formatTime(field=firstSeen, format="%Y/%m/%d %H:%M:%S") 
| lastSeen:=formatTime(field=lastSeen, format="%Y/%m/%d %H:%M:%S")
```

Keep in mind that `@timestamp` is epoch, which means you can basically search for the "smallest" which is the oldest, or the "largest" which is the most recent. That query says "group the results by `UserName`, find the smallest/oldest timestamp, find the largest/newest timestamp, and then reformat the epoch times into something readable."

## Create a case-insensitive user input

### Single input

User inputs can be created by putting a `?` in front of the input name, e.g. `?username` would create a username input. You'll see these in dashboards, but they can also be used in queries. The inputs are case-sensitive by default. This means that an input of "ADministrator" would not match "administrator", "Administrator, etc. The solution is to use the `test()` function and compare everything in lowercase. For example:

```
// The input is "?MyInput" and we want to match it to "UserName".
 
// First we have to assign the input to a field, otherwise it doesn't work with test().
thisInput:=?MyInput
 
// Next we compare the two strings in lower-case to see if they're equal.
test(lower(thisInput) == lower(UserName))
```

Because we're comparing everything in lowercase, an input of "administrator" would match "Administrator", "ADMinistrator", "AdMiNiSTRATOR", etc. 

### Multiple inputs

Another example of this is when we have multiple inputs, e.g. `?ComputerName`, `?aid`, and `?cid`. Let's say we only need `?ComputerName` to be case-insensitive. It'd look like this:

```
// Assign a different name for the variable. 
| inputComputerName:=?ComputerName
// If it's a "*" then keep it, if it's blank use a "*", otherwise do a case-insensitive match. 
| case {
    test(?ComputerName == "*")
      | ComputerName = ?ComputerName ;
    test(?ComputerName == "")
      | ComputerName = * ;
    test(lower(inputComputerName) == lower(ComputerName)) ;
  }
// Check the last two strings, no reason to look at case. 
| AgentIdString = ?aid AND CustomerIdString = ?cid
```

## Create a fixed-width column using `format`

It's common to use `collect()` as a function with `groupBy()` to grab other values. However, you'll occasionally hit a situation where one doesn't line up with the other, e.g. there are only two `severity` values but a dozen `detectName` values. For example:

```
#kind=Primary
| eventType=K8SDetectionEvent
| groupBy(resourceName, function=[collect(["Detection Type", clusterName], limit=10000), count(as="Total Events")], limit=max)
```

This results in output similar to this:

<img src=./images/image-2023-01-17_16-42-40.png width=400>

Instead, you'd like the `severity` aligned with the `detectionName` in the query results. This is where `format()` can be used to add "left-aligned right-padded" columns that combine the two values:

```
#kind=Primary
| eventType=K8SDetectionEvent
// The first value is the severity padded to 18 spaces, followed by the detectionName. 
| "Detection Type":=format("%-18s %s", field=[severity, detectionName])
| groupBy(resourceName, function=[collect(["Detection Type", clusterName], limit=10000), count(as="Total Events")], limit=max)
| sort("Total Events", limit=10000)
```

The `format()` line says "pad the first value until it's 18 characters wide". You now end up with this:

<img src=./images/image-2023-01-17_16-47-53.png width=400>

## Link to content in lookup files using `match`

To use the `match()` function, you can use the LogScale UI to upload a file using the Files menu, or can upload a CSV or JSON file using the lookup api.

In this example, when used in a search, let all events pass through (denoted by the `strict=false`), but for events in which the RemoteAddressIP4.country field (in the search) matches the value of the CountryCode column (in the table geo_mapping.csv) enrich those events with the Country, Continent, and Flag values of the matching row from the lookup file.

For example:

```
| match(file="geo_mapping.csv", column=CountryCode, field=RemoteAddressIP4.country, include=["Country", "Continent", "Flag"], ignoreCase=true, strict=false)
```

You now end up with additional columns from the lookup file that were not part of the original event, but are included because values from the search matched in the lookup file, like this:

<img src=./images/2023-02-13_17-02-41_match_geo_example.png width=400>