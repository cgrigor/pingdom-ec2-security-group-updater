# Pingdom-ec2-security-group-updater


Update an EC2 security group from the Pingdom probe RSS feed
The script pulls its feed from RSS at https://my.pingdom.com/probes/feed

### Requirements

To use this script you should have the aws cli tools installed, if you're running this locally (outside of aws) you will
need to provide an access-key and serect-access-key to the aws cli tools. More information can be found here....

If you're running this from an instance in ec2, a role should be assigned at boot to the instance that has permissions
to access the ec2 api. An example policy to assign to the role is below (you should update this to disallow any services not required and not as full access in this example).

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "ec2:*",
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```

Also you need the curl command and perl installed - I will assume you know how to do this.

### Usage

./pingdom-group-update.sh

By running the command without any inputs the usage is displayed to stdout

```
user@anonhost ~/D/w/c/pingdom-ec2-security-group-updater (master) [1]> ./pingdom-group-update.sh
Usage: ./pingdom-group-update.sh <security-group-id> <port-to-allow-inbound-for-pingdom>

<security-group-id> - This is the security group id of the group that you want to update
<port-to-allow-inbound-for-pingdom> - The port to allow on the inbound rules

./pingdom-group-update.sh sg-xxxxxxx 8080
```

### Script breakdown

group=$1 # the security group id

port=$2 # the port of the rule that is being added

: ${port:="8080"} # default port is 8080 if you don't specify

out=/tmp/pingdom.xml # The path to the xml rss feed downloaded from pingdom

old_ips=/tmp/pingdom-old-ips.txt # File which will hold a list of the current IP's in the security group

old_ports=/tmp/pingdom-old-ports.txt # File which will hold a list of the ports associated to the current IP's in the security group

preips=/tmp/pingdom-ips-regional.txt # A filtered list of IP's for the Country you have chosen

ips=/tmp/pingdom-ips.txt # A list of IP's that will be added to the security group

country=USA # The country of the probes that you want to add to the security group (check the contents of the RSS feed file for a list of countries)

---
1.
Get a list of the current IP's and ports and output them into the relevant files.
```
cho "Fetching old IPs in security group..."
aws ec2 describe-security-groups --group-id $group --query 'SecurityGroups[*].{Inbound:IpPermissions}' | grep FromPort | cut -d: -f2 | rev | cut -c 3- | rev > $old_ports
aws ec2 describe-security-groups --group-id $group --query 'SecurityGroups[*].{Inbound:IpPermissions}' | grep CidrIp | cut -d: -f2 | cut -c 3- | rev | cut -c 2- | rev > $old_ip
old_ip_count=`cat $old_ips | wc -l`
```
---
2.
Revoke the rules from the security group
```
for i in `seq 1 $old_ip_count`;
do
	old_port=`sed "${i}q;d" $old_ports`
	old_ip=`sed "${i}q;d" $old_ips`
	aws ec2 revoke-security-group-ingress --group-id $group --cidr $old_ip --port $old_port --protocol tcp
done
```
---
3.
fetch the RSS feed and prep the data
```
echo "Fetching new pingdom probe ips..."
curl https://my.pingdom.com/probes/feed > $out

# Strip down only to the country needed from pingdom (hosts in the US)
echo "Parsing IPs for $country..."
perl -ne 'BEGIN{$/="</item>\n";} print m|(<item>.*Country: '${country}'.*$/)|ms' /tmp/pingdom.xml | grep pingdom:ip > $preips

echo "Parsing IPs..."
grep pingdom:ip $preips | sed -n 's:.*<pingdom\:ip>\(.*\)</pingdom\:ip>.*:\1:p' > $ips
lines=`cat $ips`
```
---
4.
Print out a list of the IP's that will be added and then add them
```
# print out the IPs
echo "The following IP's will be added to the security group"
for ip in $lines ; do
	echo $ip
done
# for each ip, call the ec2 cli to add ips to a predefined pingdom only security group
# see: http://docs.amazonwebservices.com/AWSEC2/latest/CommandLineReference/ApiReference-cmd-AuthorizeSecurityGroupIngress.html
echo "Adding IPs to security group: $group"
for ip in $lines ; do
	aws ec2 authorize-security-group-ingress --group-id $group --cidr $ip/32 --port $port --protocol tcp
done
```
---

### Background

Amazon security groups by default allow only 50 ingress and 50 egress rules to be added in one security group. Since this script was branched from another source which allowed all 90+ IP's to be added to the security group, this would result in a failure (so pretty useless). One way around this is to request the soft limit on security groups to be increased to 100, however this decreases the number of availble security groups that can be assigned to a network interface, so no use there.

It was decided to run pingdom in the region the service is launched, in our case we chose Oregon, meaning we adpated the script to only add IP's of nodes in the USA.

### Maintainers

Chris Grigor
email: chris.grigor@irdeto.com
