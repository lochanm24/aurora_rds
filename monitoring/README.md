==========================
Enable Enhanced Monitoring
==========================
1. Create the role using CloudFormation
   * Stack name = rds-postgresql-enhanced-monitoring-role
2. In RDS Console
   * Select node-01
   * Click on Modify
3. In the monitoring tab
   * Check the box = Enable Enhanced monitoring
   * Select the role created in #1
   * Select "Apply Immediately" and apply changes


====================
Aurora Postgres Logs
====================

Part-1  Checkout logs in RDS console
------------------------------------
* Select the instance
* Select 'Monitoring & Logs'
* Select Log
* Select View | Watch

Part-2  Publish logs to CloudWatch
----------------------------------
* Modify instance
* Under Additional Config - select publish logs to CW

===============================
enhanced-monitoring-log-sample.json
===============================
This is a sample JSON for a log record published by RDS/EM to CloudWatch. Refer to the link below for the details of each metric:
https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_Monitoring.OS.CloudWatchLogs.html

For viewing the JSON you may use the viewer: http://jsonviewer.stack.hu/

=========================================
Recipe: RDS Event Retention in CloudWatch
=========================================
1. Setup a CloudWatch Log Group
/aws/events/RDSAllEvents
Set the retention on log group = 6 months

2. Set up a rule on Event Bridge
* Name = All-RDS-Events
* Service Name = RDS
* Event Type = All 
* Add Target = CloudWatch group created in #1

3. Take some action on RDS cluster & check events in CloudWatch
* E.g., reboot instance
* E.g., take snapshot



