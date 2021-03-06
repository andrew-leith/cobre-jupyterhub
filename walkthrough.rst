Walkthrough
===========

Kubernetes Configuration
========================

Install kubectl (Kubernetes control)::

	gcloud components install kubectl

Use kubectl to generate a temporary cluster::

	gcloud container clusters create cobre-jupyter \
    	--num-nodes=3 \
    	--machine-type=n1-standard-2 \
    	--zone=us-east4-a

Here, we are using three nodes in the ``us-east4-a`` region due to its proximity to our location

The successful initialization of these nodes can be confirmed using::

	kubectl get node

Next, Helm must be started::

	kubectl --namespace kube-system create sa tiller
	kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
	helm init --service-account tiller

and its initilization can be confirmed with::

	helm version

Next, add Jupyterhub to the Helm installation::

	helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
	helm repo update

YAML file
=========

Now, create a yaml file describing the config.  This file is needed by Helm to configure various aspects of the Jupyterhub server.  First, two hash strings are required to run the hub, a 'cookieSecret' and a 'secretToken' (here denoted as 'xxx' and 'yyy' - you must generate your own using the openssl command listed above those lines).  The quotation marks are required.::

	hub:
  		# output of first execution of 'openssl rand -hex 32'
  		cookieSecret: "xxx"
	proxy:
  		# output of second execution of 'openssl rand -hex 32'
  		secretToken: "yyy"

The YAML file can also be used to designate administrators for the hub.  Admins can manually start and stop individual users' Jupyter instances as well as start and stop the hub itself, and they can also view the contents of user Jupyters.  Admins can also elevate other users to admin level from the hub's admin panel.  Here, you can see a specification that creates two admins, 'aleith' and 'ashok'::

	admin:
  		users:
    	# CIS admins
    		- 'aleith'
    		- 'ashok'

Note that it is not (by default) necessary to explicitly create all users that will be using the hub.  However, at least one administrator must be created in the YAML file if you want to have any administrator accounts.

The YAML file is also used to specify the Docker image that Helm will use to spawn individual user notebooks.  This image is pulled from Dockerhub and can be specified in the following manner::

	singleuser:
  		image:
     		name: aleith/repo-docker
     		tag: v0.1

The ``name`` argument is the host and image name, while the ``tag`` argument is the specific tag corresponding to the desired image's desired version.

By defauly, Jupyterhub uses a so-called 'dummy authenticatior', a system of login where anyone can log in with any username and the process of doing so will generate an account with that username.  This configuration is insecure, so it is advisable to set a specific password that users must input in order to generate their account::

	auth:
  		dummy:
    	password: 1234

The complete YAML file is therefore as follows::

	hub:
 		cookieSecret: "xxx"
	proxy:
  		secretToken: "yyy"
  		
	admin:
  		users:
    		- 'aleith'
    		- 'ashok'
    		
	singleuser:
  		image:
     		name: aleith/repo-docker
     		tag: v0.1
     		
	auth:
  		dummy:
    		password: 1234

Running the Hub
===============

then generate the COBRE hub from the directory containing the yaml::

	helm install jupyterhub/jupyterhub \
    	--version=v0.4 \
    	--name=cobre-hub \
    	--namespace=cobre-hub \
    	-f config.yaml

The status of the JupyterHub can be determined with::

	kubectl --namespace=cobre-hub get pod

Once it is running, the IP of the JupyterHub can then be determined with::

	kubectl --namespace=cobre-hub get svc

The correct IP will be found under ``EXTERNAL-IP``