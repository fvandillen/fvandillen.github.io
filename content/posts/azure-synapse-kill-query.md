---
title: "How to kill a Synapse query"
date: 2022-06-30T13:56:48+02:00
draft: false
author: ["Florian van Dillen"]
tags: ["Azure", "Synapse", "Performance"]
summary: Explains how to kill a Synapse query that is stuck in SQL serverless pool.
ShowToc: true
---

# Introduction
Sometimes a query in Synapse serverless SQL doesn't work as expected and times out, or worse. It may happen that it gets stuck entirely. Luckily, there is a way to get things going again! Here's how.

In your favorite SQL tool, run the following query to identify the `process ID` of the stuck query:

# SQL snippet
{{< highlight tsql "linenos=table" >}}
	SELECT 
	    'Running' as [Status],
        Transaction_id as [Request ID],
        'SQL On-demand' as [SQL Resource],
        s.login_name as [Submitter],
        s.Session_Id as [Session ID],
        req.start_time as [Submit time],
        req.command as [Request Type],
    SUBSTRING(
	        sqltext.text, 
	        (req.statement_start_offset/2)+1,   
	        (
	            (
	                CASE req.statement_end_offset  
	                    WHEN -1 THEN DATALENGTH(sqltext.text)  
	                    ELSE req.statement_end_offset  
	                END - req.statement_start_offset
	            )/2
	        ) + 1
	    ) as [Query Text],
	    req.total_elapsed_time as [Duration]
	FROM 
	    sys.dm_exec_requests req
	    CROSS APPLY sys.dm_exec_sql_text(sql_handle) sqltext
	    JOIN sys.dm_exec_sessions s ON req.session_id = s.session_id 
{{< /highlight >}}

# Killing the process
Using your obtained `process ID` (in this example it will be `81`), run the following SQL to kill the process:
{{< highlight tsql "linenos=table" >}}
KILL 81
{{< /highlight >}}

Be warned, this might take a while. If everything went well, you will have things moving again.

# What if this doesn't work?
It could be that things are unable to get started again. In that case, you will have one more trick in the book: Azure support. If you open a ticket with them, they can restart your SQL serverless pool.