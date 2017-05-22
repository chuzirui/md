How to update build cron?
=========================

NOTE: Get a perforce and dbc account before running the below
instructions. See links 3, 4.

1.  Login to your dbc account.

::

:   ssh <gangila@pa-dbc1130.eng.vmware.com>

2.  Login to perforce hosts.

::

:   /build/apps/bin/p4\_login -aA

3.  Create a perforce client

::

:   mkdir scheduler-client cd scheduler-client p4 -p
    perforce-releng:1850 client -t scheduler-template
    ujo-scheduler-client

4.  "Checkout" the project.

::

:   p4 -p perforce-releng:1850 -c ujo-scheduler-client sync

5.  Edit the cron.d/nsx-ujo/nsx-ujo file
6.  Add file to change

::

:   p4 -p perforce-releng:1850 -c ujo-scheduler-client add
    nsx-ujo/nsx-ujo

(If you want to discard changes just use revert instead of add)

7.  Create a changelist

::

:   p4 change

8.  Submit the changelist

::

:   p4 submit -c &lt;changelist id returned in previous step&gt;

Helpful Links: 
1.
<https://wiki.eng.vmware.com/SCM/Frequently_Asked_Questions> 
-- How to use perforce 
2.
<https://wiki.eng.vmware.com/Build/RequestOfficialBuilds#Scheduling_a_Recurring_Official_Build>
3. DBC Account Request: <https://buildweb.eng.vmware.com/dbc/> 
4. Perforce Account Request: <https://p4user.eng.vmware.com/>
