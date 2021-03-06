# Disk-Space-monitoring


#########################################################
#
# Disk space monitoring and reporting script
#
#########################################################
 

$users = "sansari@globus.com.au", "helpdesk@globus.com.au", "mshiroma@globus.com.au", "skoirala@globus.com.au", "gmisra@globus.com.au" # List of users to email your report to (separate by comma)
$fromemail = "notification@globus.com.au"
$server = "EXCHANGE001.globus.com.au" #enter your own SMTP server DNS name / IP address here
$list = "\\srvsyd15\C$\Automation\diskspacemonitor20percent1\diskspacemonitoring20percentESX.txt" #This accepts the argument you add to your scheduled task for the list of servers. i.e. list.txt
$computers = get-content $list #grab the names of the servers/computers to check from the list.txt file.
# Set free disk space threshold below in percent (default at 20%)
[decimal]$thresholdspace = 20
 

#assemble together all of the free disk space data from the list of servers and only include it if the percentage free is below the threshold we set above.
$tableFragment= Get-WMIObject  -ComputerName $computers Win32_LogicalDisk `
| select __SERVER, DriveType, VolumeName, Name, @{n='Size (Gb)' ;e={"{0:n2}" -f ($_.size/1gb)}},@{n='FreeSpace (Gb)';e={"{0:n2}" -f ($_.freespace/1gb)}}, @{n='PercentFree';e={"{0:n2}" -f ($_.freespace/$_.size*100)}} `
| Where-Object {$_.DriveType -eq 3 -and [decimal]$_.PercentFree -lt [decimal]$thresholdspace} `
| ConvertTo-HTML -fragment 
 

# assemble the HTML for our body of the email report.
$HTMLmessage = @"
<font color=""black"" face=""Arial, Verdana"" size=""3"">
<u><b>Disk Space Storage Report 20%</b></u><br>
<br>This report was generated because the driver(s) listed below have less than $thresholdspace % free space. Drivers above this threshold will not be listed.
<br><br>
<style type=""text/css"">body{font: .8em ""Lucida Grande"", Tahoma, Arial, Helvetica, sans-serif;}
ol{margin:0;padding: 0 1.5em;}
table{color:#000;background:#CC9;border-collapse:collapse;width:647px;border:5px solid #900;}
thead{}
thead th{padding:1em 1em .5em;border-bottom:1px dotted #FFF;font-size:120%;text-align:left;}
thead tr{}
td{padding:.5em 1em;}
tfoot{}
tfoot td{padding-bottom:1.5em;}
tfoot tr{}
#middle{background-color:#999;}
</style>
<body BGCOLOR=""white"">
$tableFragment
</body>
"@ 
 

# Set up a regex search and match to look for any <td> tags in our body. These would only be present if the script above found disks below the threshold of free space.
# We use this regex matching method to determine whether or not we should send the email and report.
$regexsubject = $HTMLmessage
$regex = [regex] '(?im)<td>'
 

# if there was any row at all, send the email
if ($regex.IsMatch($regexsubject)) {
                        send-mailmessage -from $fromemail -to $users -subject "Disk Space Monitoring Report" -BodyAsHTML -body $HTMLmessage -priority High -smtpServer $server
}
 

# End of Script
