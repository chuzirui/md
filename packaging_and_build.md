Build Deliverables
==================

**NOTE**: This document contains obsolete information and may be
deprecated in the future

We leverage Central Engineering build infra which we interface via
buildweb [^1].

Right now we generated two build deliverables:

1.  NCP docker image tar. Approx 290MB
2.  NCP Code as .tar.gz. Users can pip install for CNI/nsx-kube-proxy.
    (Even NCP if they want to run it as a process, not container)

All the code which enables the above build process resides within our
repository is in the support/[^2] and image/[^3] folders in the nsx-ujo
project base.

How to run NCP as a pod/container?
==================================

NCP can be run as a container in three simple steps: 1. Download the tar
image from the buidlweb. 2. Load the image into your local docker images
repo.

::

:   docker load -i &lt;tar file&gt;

3.  Substitute that image name into ncp-rc.yaml, which can be found here
    [^4]
4.  Create the config file ncp.ini and change the values from your
    current kubernetes cluster and NSX system.
5.  Copy it to /etc/nsx-ujo/ncp.ini, which is the default config
    file location.
6.  Create a configmap which takes in the contents of that file.

::

:   kubectl create configmap ncp-config --from-file=ncp.ini

where ncp-config could technically be any string of the name of config,
but ncp-rc.yaml is using the name "ncp-config" by default

7.  Create the Replication controller.

::

:   kubectl create -f ncp-rc.yaml

This should bring up NCP as a container.

You can use,

::

:   kubectl logs &lt;ncp\_pod\_name&gt;

> to check the logs for NCP.

Build job schedule and how to configure?
========================================

The build is managed by a cron job which is set to 18:00 UTC now [^5] .
So one can just to build-web and search for nsx-ujo to get the latest
build. Updating this cron job is tedious, steps are documented in
build\_cron\_howto In addition one can also request a build ad-hoc by
using the "Offical Build Request" from buildweb. Approx time for a build
is 10 min.

[^1]: <https://buildweb.eng.vmware.com/ob/?query=nsx-ujo>

[^2]: <https://opengrok.eng.vmware.com/source/xref/nsx-ujo.git/support/>

[^3]: <https://opengrok.eng.vmware.com/source/xref/nsx-ujo.git/images/>

[^4]: <https://opengrok.eng.vmware.com/source/xref/nsx-ujo.git/ncp-rc.yaml>

[^5]: <https://buildweb.eng.vmware.com/schedules/ob/list/>
