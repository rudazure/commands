# kusto queries commonly used

// pluralsight review 80% operators

// search
Perf | search "Memory"

// search with case sensitive
Perf | search kind=case_sensitive "memory"

// search in multiple table
search in (Perf, Event, Alert) "addsvm"

// search in a specific column inside a table
Perf | search CounterName == "Available MBytes"

// search can be combined
Perf | search "Free*bytes" and ("C:" or "D:")

// search with expression regex
Perf | search InstanceName matches regex "[A-Z]:"

// where
Perf | where TimeGenerated >= ago(1h)

// where and "and"
Perf | where TimeGenerated >= ago(30d)
and CounterName == "Bytes Received/sec"

// where and "and" and "or"
Perf
| where TimeGenerated >= ago(7d)
    and (CounterName == "Bytes Received/sec"
        or
        CounterName == "% Processor Time")

// another example
Perf
| where TimeGenerated >= ago(7d)
    and (CounterName == "Bytes Received/sec"
        or
        CounterName == "% Processor Time")
| where CounterValue > 0

// using search with where clausule
Perf
| where * has "Bytes"

Perf
| where * hasprefix "Bytes" // at the start

Perf
| where * hassuffix "Bytes" // at the end


// where supports regex as well

Perf
| where InstanceName matches regex "[A-Z]:"

// Take

Perf
| where TimeGenerated >= ago(1h)
| take 10

// Limit are synonim for take
Perf
| where TimeGenerated >= ago(1h)
| limit 20

/// Count

Perf
| count

Perf
| where TimeGenerated >= ago(1h)
| count

// summarize with count

// summarize allows you count number of rows by column using the count()
Perf
| summarize count() by CounterName 

// can break down by multiple columns
Perf
| summarize count() by ObjectName, CounterName 

// you can rename the output column for count
Perf
| summarize PerfCount=count()
    by ObjectName, CounterName

// with summarize, you can use other aggregation functions
Perf
| where CounterName == "% Free Space"
| summarize NumberOfEntries=count()
        , AverageFreeSpace=avg(CounterValue) 
        by CounterName

// Bin allows you to summarize into logical groups, like days
Perf
| summarize NumberOfEntries=count()
        by bin(TimeGenerated, 1d) 

// bin can be at multiple level
Perf
| summarize numberofentries=count()
    by CounterName
        , bin(TimeGenerated, 1d) 

// bin can be used by values other than dates.
// here we see number of entries for each level at % Free Space
Perf
| where CounterName == "% Free Space"
| summarize NumberOfRowAtThisPercentLevel=count()
        by bin(CounterValue, 10) 


// Extend

// extend creates a calculated column and adds to the result set
Perf
| where CounterName == "Free Megabytes"
| extend FreeGB = CounterValue / 1000

// can also use with strcat to create new string columns
Perf
| where TimeGenerated >= ago(10m)
| extend ObjectCounter = strcat(ObjectName, " - ", CounterName) 

// project

// project allows you to select a subset of column
Perf
| where CounterName == "Free Megabytes"
| where InstanceName matches regex "[A-Z]:" 
| project ObjectName 
            , CounterName 
            , InstanceName 

// distinct
Event
| where EventLevel == "2"
| distinct Source


// SCALAR OPERATORS

// print, now, ago, sort, extract, parse, datetime, Timespan, startof, endof, between, todynamic, format_datetime, datetime_part, case
// iif, isempty, split, String operator, strcat

// print

print "Hello World"

print 11*3

print resultado = 12 * 24

print data_agora = now()

// ago

print ago(1d)

print ago(1h)

print ago(1m)

print ago(1s)

print ago(1ms)

print ago(1microsecond)

print ago(1tick)

print ago(-1d) // tomorrow

// sort

Perf
| where TimeGenerated > ago(15m)
| where ObjectName == "DNS"
| project Computer , TimeGenerated , ObjectName 
| sort by Computer asc 
        , TimeGenerated asc

// can use order by instead of sort

// extract

// example to start
Perf
| where ObjectName == "LogicalDisk"
    and InstanceName matches regex "[A-Z]:" 

// adding extract command ... when the second parameter is 1, it returns get just the part of the extract inside the parenthesis
Perf
| where ObjectName == "LogicalDisk"
    and InstanceName matches regex "[A-Z]:" 
| project Computer 
        , CounterName
        , extract("([A-Z]):", 1, InstanceName)

// OR giving a name for the column

Perf
| where ObjectName == "LogicalDisk"
    and InstanceName matches regex "[A-Z]:" 
| project Computer 
        , CounterName
        , driverletter = extract("([A-Z]):", 1, InstanceName)


// parse

// break a column with lots of text into multiple new columns 
 
Event
| where RenderedDescription startswith "Error Message"

// Error Message = System error. Context: Severity = Warning Host Name = OpsMgr PowerShell Host Host Version = 7.0.5000.0 
// Host ID = bc9d4f4a-be1c-4d5e-b7ee-a9f0e378eb54 Host Application = C:\Program Files\Microsoft Monitoring Agent\Agent\MonitoringHost.exe -Embedding 
// Engine Version = 5.1.14393.3053 Runspace ID = 5ca244d1-281b-4754-906b-2d735aaecf06 Pipeline ID = 2 Command Name = Command Type = 
// Script Name = Command Path = Sequence Number = 81852 User = WORKGROUP\SYSTEM Connected User = Shell ID = Microsoft.PowerShell User Data: 

Event
| where RenderedDescription startswith "Error Message"
| parse RenderedDescription with "Error Message = " typeOfError
                                "Severity = " severity
                                "Host Name =" hostname
                                "Host Version =" * 
| project Computer, typeOfError , severity , hostname



// datetime and timespan
// can be helpful to count how many events you get in a such a day

Event
| where TimeGenerated >= ago(14d)
| extend DayGenerated = startofday(TimeGenerated) // create a new column to represent the entire day
| project Source
        , DayGenerated 
| summarize EventCount=count() // create a new column based on a count() function
            by DayGenerated // count by the new column DayGenerated and then Source column
            , Source
| sort by DayGenerated asc // put the days in order


// todynamic
// use with columns that store json notation

SecurityAlert

// { "Activity start time (UTC)": "2019/11/11 03:00:01.4600308", "Activity end time (UTC)": "2019/11/11 03:59:46.8920098", 
// "Attacker source IP": "IP Address: 193.188.22.218", "Attacker source computer name": "Unknown", "Number of failed authentication attempts to host": "148", 
// "Number of existing accounts used by source to sign in": "1", "Number of nonexistent accounts used by source to sign in": "93", 
// "Top accounts with failed sign in attempts (count)": "PAULA (2), MARTIN (2), MASTER (2), MARY (2), MIGUEL (2), OFFICE (2), PATRICIA (2), PETER (2), IT (2), ITSUPPORT (2)", 
// "Was RDP session initiated": "No", "End Time UTC": "11/11/2019 03:59:46", "ActionTaken": "Detected", "resourceType": "Virtual Machine", 
// "ServiceId": "865b9b71-794d-4ba2-8e71-3ea41c5043aa", "ReportingSystem": "Azure", "OccuringDatacenter": "eastus" }

SecurityAlert
| extend ExtProps=todynamic(ExtendedProperties) // ExtendedProperties is the name of the column that contains json notation
| project AlertName 
        , TimeGenerated 
        , ExtProps["Activity start time (UTC)"]
        , ExtProps["Attacker source IP"]
        , ExtProps["Ip Address"]
        , ExtProps["Number of failed authentication attempts to host"]
// there are something that needs to be fixed in the query above. I didn't debug it ...

// split

// split an individual column content into multiple values

Perf
| take 5

Perf
| take 100
| extend CounterPathArray = split(CounterPath, "\\")
| extend myComputer = CounterPathArray[2]
    , myObjectInstance = CounterPathArray[3]
    , myCounterName = CounterPathArray[4]
| project Computer 
        , ObjectName 
        , CounterName 
        , InstanceName 
        , myComputer 
        , myObjectInstance 
        , myCounterName 
        , CounterPath 


// string operators
// contains // without case sensitive
// contains_cs // with case sensitive
// !contains // not contain
// in //used to compare a column to a set of values
// example of "in"



// !in // not in ...
// startwith
// endwith
// has
// hasprefix
// hassuffix
// contains
// matches regex


// join
// join two tables together when they have a common column name
Perf
| where TimeGenerated >= ago(30d)
| take 1000
| join (Alert) on Computer

// join two tables - other example
Perf
| where TimeGenerated >= ago(30d)
| take 1000
| join (Alert) on $left.Computer == $right.Computer

// join with some complexity
Perf
| where TimeGenerated >= ago(90d)
| where CounterName == "% Processor Time"
| project Computer
        , CounterName
        , CounterValue
        , PerfTime = TimeGenerated
| join ( Alert
        | where TimeGenerated >= ago(90d)
        | project Computer
                , AlertName
                , AlertDescription
                , ThresholdOperator
                , ThresholdValue
                , AlertTime=TimeGenerated
        | where AlertName == "High CPU Alert"
        )
    on Computer

// several types of join .... represented by "kind"
// innerunique, inner, leftouter, rightouter, fullouter, leftanti, leftsemi (same for right)


// union

// join vs union

// join A & B
// Row: A.Col1 A.Col2 A.Col3 B.Col1 B.Col2 B.Col3

// Union A & B
// Row: A.Col1 A.Col2 A.Col3 A.Col4 <Empty>
// Row: B.Col1 B.Col2 B.Col3 <empty> B.Col5

// union
UpdateSummary
| union Update

UpdateSummary
| union withsource = "SourceTable" Update
