# Installation of a jupyter notebook server
Overview reference: http://jupyter-notebook.readthedocs.io/en/stable/public_server.html
Prerequisites:  a *nix vm (on-prem, AWS, OPC, hardware, etc. etc) 
Example: https://jupyter.aws.gis.cloud.vt.edu:8888

### VM
* https://console.aws.amazon.com → EC2 → Launch Instance
* Ubuntu in latest LTS is fine.  Or if you have an AMI you like, probably that too.  And there is a non-free-tier Deep Learning AMI that has lots of this stuff already, but we're not using that here. 
* t2.micro free tier
* up the base EBS storage from 8GB to something more reasonable.  say 20GB
* make a key pair, either in the console and download the PEM to your bastion server (e.g.:'jupyter2-aws-ec2.pem') , or upload one you already generated (ipad→termius)
* Launch it
* Add a route 53 DNS entry for it (e.g., 'jupyter2.aws.gis.cloud.vt.edu')
* connect to it when it initializes using ssh ubuntu@jupyter2.aws.gis.cloud.vt.edu -i ~/.ssh/jupyter2-aws-ec2.pem


## Installing Python, the ArcGIS Python API, and the Jupyter notebook itself

### Anaconda
download anaconda to a home dir.  this installs jupyter notebook.
Latest Anaconda is at https://www.anaconda.com/distribution/#download-section
wget https://repo.anaconda.com/archive/Anaconda3-2019.03-Linux-x86_64.sh

** note:  what you get is a python environment running out of 
/home/ubuntu/anaconda3/bin/

** note: You have to logout and log back into 


https://developers.arcgis.com/python/guide/install-and-set-up/
https://developers.arcgis.com/python/guide/install-and-set-up/#Install-using-Anaconda-for-Python-Distribution

### ArcGIS Python API
    conda install -c esri arcgis
    jupyter nbextension enable arcgis --py --sys-prefix
    conda install -c conda-forge jupyter_nbextensions_configurator
Installation of python modules that may be rather useful

    conda install boto3                      # python module for aws
    conda install psycopg2                   # python module for postgresql
    conda install -c damianavila82 rise      # RISE module for slideshows


<hr>

## Obtaining an SSL Certificate

In order for us to access our jupyter notebook server, we have to have a real cert.  Fortunately, Lets Encrypt makes this easy.

http://jupyter-notebook.readthedocs.io/en/stable/public_server.html#using-lets-encrypt
Lets Encrypt provides free SSL/TLS certificates. You can also set up a public server using a Lets Encrypt certificate.
Running a public notebook server will be similar when using a Lets Encrypt certificate with a few configuration changes. Here are the steps:
### Create a Lets Encrypt certificate.  
        sudo apt-get update
        sudo apt-get install software-properties-common
        sudo add-apt-repository ppa:certbot/certbot
        sudo apt-get update
        sudo apt-get install certbot


### Before we can go any further we need to setup the jupyter notebook config directory.


    $ jupyter notebook --generate-config

... and password ... 
$ jupyter notebook password
Enter password: ****
Verify password: ****
[NotebookPasswordApp] Wrote hashed password to /home/ubuntu/.jupyter/jupyter_notebook_config.json

OK, now do the cert and copy it to the jupyter config dir you created. 




    sudo certbot certonly --standalone -d jupyter.aws.gis.cloud.vt.edu
    mkdir -p /home/ubuntu/.jupyter/certs/

    sudo chmod 755 /etc/letsencrypt/live/   #this is so the renewal tool can pick them up in a userland ubuntu cron

    cp /etc/letsencrypt/live/jupyter2.aws.gis.cloud.vt.edu/fullchain.pem /home/ubuntu/.jupyter/certs/fullchain.pem
    cp /etc/letsencrypt/live/jupyter2.aws.gis.cloud.vt.edu/privkey.pem /home/ubuntu/.jupyter/certs/privkey.pem
and

    mkdir ~/jupyter

<hr>

## NOW, we need to modify some parameters in the jupyter config file.




In the ~/.jupyter directory, edit the notebook config file, jupyter_notebook_config.py. By default, the notebook config file has all fields commented out. The minimum set of configuration options that you should to uncomment and edit in jupyter_notebook_config.py is the following:

    c.NotebookApp.certfile = u'/absolute/path/to/your/certificate/fullchain.pem'
    c.NotebookApp.keyfile = u'/absolute/path/to/your/certificate/privkey.pem'
    c.NotebookApp.ip = '0.0.0.0'  # new for jupyter 5.7; used to be '*'
    c.NotebookApp.password = u'sha1:bcd259ccf...<your hashed password here... find it in the .json file>'
    c.NotebookApp.port = 8888
    c.NotebookApp.open_browser = False
    c.NotebookApp.base_url = '/jupyter'    #NOTE that you need to make sure this dir exists.

to set up renewal for jupyter certs  
https://www.digitalocean.com/community/tutorials/how-to-use-certbot-standalone-mode-to-retrieve-let-s-encrypt-ssl-certificates
make a script like the one below...  (example == /home/ubuntu/scripts/updateJupyterCerts.sh)
then add the following line to /etc/letsencrypt/renewal/jupyter2.aws.gis.cloud.vt.edu.conf
     renew_hook = /home/ubuntu/scripts/updateJupyterCerts.sh
    #!/bin/sh
    # Copy LetsEncrypt updated certs to a location that the jupyter notebook server
    # can access, and change permissions
    
    cp /etc/letsencrypt/live/jupyter2.aws.gis.cloud.vt.edu/fullchain.pem /home/ubuntu/.jupyter/certs/fullchain.pem
    cp /etc/letsencrypt/live/jupyter2.aws.gis.cloud.vt.edu/privkey.pem /home/ubuntu/.jupyter/certs/privkey.pem
    chown ubuntu:ubuntu /home/ubuntu/.jupyter/certs/fullchain.pem
    chown ubuntu:ubuntu /home/ubuntu/.jupyter/certs/privkey.pem
    chmod 755 /home/ubuntu/.jupyter/certs/fullchain.pem
    chmod 755 /home/ubuntu/.jupyter/certs/privkey.pem
    
    # Kill any running instances of the jupyter notebook server
    killall jupyter-notebook >/dev/null 2>&1
    # Restart the jupyter notebook server
    su ubuntu -c "nohup /home/ubuntu/anaconda3/bin/jupyter notebook >/home/ubuntu/.jupyter/jupyter.log 2>&1 &"



## Retrieving all your jupyter notebooks to run on this server
(it's not the only place you're going to keep them, right? RIGHT?)

Either download a gitLab private key to this machine you already have or make one and upload it to gitlab

creating a public key with ssh-keygen 

likewise for github

git clone your repo to /home/ubuntu/jupyter/<project-root>





## Starting the notebook server
    su ubuntu -c "nohup /home/ubuntu/anaconda3/bin/jupyter notebook >/home/ubuntu/.jupyter/jupyter.log 2>&1 &"


