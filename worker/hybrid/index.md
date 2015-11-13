# Hybrid IronWorker

Hybrid IronWorker enables you to get all the power and benefits of IronWorker platform while running your workloads on your own
hardware. You can run them on your own servers on any cloud or even in your own datacenter, behind the firewall.

## Installation / Setup

It's very easy to get started using Hybrid IronWorker. Just follow the steps below to get started. 

NOTE: You must have Hybrid access enabled on your account. Email <support@iron.io> to upgrade your account. 

### 1. Create a Cluster

Login to [HUD](http://hud.iron.io), click IronWorker, then click your name in the top right and then click Clusters.
You'll see a list of existing clusters if any and a link to Create a new one. Click the _Create Cluster_ link. 
Fill out the form and submit it. You'll get a `CLUSTER_ID` and `CLUSTER_TOKEN` that you'll need in the next steps. 

### 2. Launch the iron/runner image

On any machine that has Docker installed, just run our iron/runner image with the following flags:

```sh
docker run --name ironrunner -it --privileged -d -e "CLUSTER_ID={CLUSTER_ID}" -e "CLUSTER_TOKEN={CLUSTER_TOKEN}" --net=host iron/runner
```

Replace `{CLUSTER_ID}` and `{CLUSTER_TOKEN}` with the id and token you obtained in step 1 above. 

That's it!  Launch as many of these as you want/need. 

## Using your new Cluster

Everything is the same as using IronWorker on the public cloud, except When queuing jobs,
simply pass in the `CLUSTER_ID` in the "cluster" param ([API docs](http://dev.iron.io/worker/reference/api/#queue_a_task)).
Here is a quick example you can use with [IronCLI]

First upload your worker (or use a Docker image):

```sh
iron worker upload --name hello.rb treeder/hello.rb
```

Example from the cli:

```sh
iron worker queue --cluster CLUSTER_ID --wait hello.rb
```

## Running on AWS

Nothing special is required to run on AWS, the same steps apply above, but you can use the following 
cloud-init/User Data script to install Docker and start IronWorker.
This has been tested on Ubuntu 15.04 and 14.04 AMI.

```
#!/bin/sh
curl -sSL https://get.docker.com/ | sh
sudo service docker start
echo \"Starting runners\"
sudo docker run --name ironrunner -it --privileged -d -e "CLUSTER_ID={CLUSTER_ID}" -e "CLUSTER_TOKEN=#{CLUSTER_TOKEN}" --net=host iron/runner
```

## End-to-End encryption of payloads

While it's possible to do end to end encryption of payloads yourself, we do
offer a convenient way to help accomplish this. Currently, we support a scheme
using `AES-CFB` via [IronCLI] and the hybrid runner, with the encryption and
decryption keys never reaching the cloud.

To get started, you'll need to generate an AES key. The easiest way to do this is
with OpenSSL:

```sh
$ openssl enc -aes-256-cfb -k secret -P -md sha1
salt=2C712245226CC707
key=35CF247CF1D6444829732186470F393F7988C97659198B46D65747B66C9DA419
iv =26BAE4C73F2C5143F7D2435AD4A73F78
```

Copy the `key` field and save it somewhere safe. You won't need the other
fields. Since we're using AES, the key is symmetric and will be used as both
the encryption and decryption key.

After this, you'll need to make sure your hybrid cluster runners are started
with this key before queueing tasks with encrypted payloads. To do this, simply
add the env var `DECRYPTION_KEY` passed to the `iron/runner` docker container, like so:

```sh
$ docker run --name ironrunner--privileged -d -e "DECRYPTION_KEY={KEY}" \
 -e "CLUSTER_ID={CLUSTER_ID}" -e "CLUSTER_TOKEN=#{CLUSTER_TOKEN}" --net=host iron/runner
```

This key will be passed to the runner which will decrypt any encrypted payloads
it comes across before passing them to the job. We only store this key in memory
and it won't leave the runner container.

After getting all the runners started up with that decryption key you may begin
queueing jobs with encrypted payloads to your cluster. The CLI will take care of
the encryption before queueing the job and the key will not leave the CLI process.
You should use the same key as the runner since we're using AES. An example
command is:

```sh
$ iron worker queue --encryption-key {KEY} --cluster {CLUSTER_ID} --wait hello.rb
```

You won't have to update any of your tasks to start using the encrypted payloads
feature, and your payloads are now never stored or sent in plaintext within the Iron
platform. If your hybrid cluster is behind a firewall and you queue tasks from
behind the firewall, this means your payloads never leave your firewall as
plaintext. As we like to say, BOOM.

[IronCLI]:https://github.com/iron-io/ironcli
