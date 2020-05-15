# Scanning a Docker Container with InSpec
## 1. Configure your workstation
1. Create a remote desktop connection to your Windows workstation, login using `chef` as the username, ask your instructor for the password.<br />
2. Open the remote desktop connection and wait for the background to change to `DevOps Better Together`, the Chrome browser and a Powershell window to open.
3. We are going to use vscode to write our code and a Linux Node to run our InSpec tests on - we will setup vscode to be our one stop environment for this:<br />
   i. In the Powershell window type<br />
   `code .`<br />
   ii. While in vscode press the F1 function key and start typing<br />
   `Remote-SSH: Connect to Host...`, select it and type<br />
   `ec2-user@<ask your instructor for the ip address>`, select it and then select `Linux` (and `continue` if this is the first time you have connected).<br />
   iii. Close the Welcome page and click on the `Explorer icon` on the top left, Select `Open Folder` and fill in `/home/ec2-user/inspec-labs` and click `OK`.</br >
   iv. From the vscode menu click `Terminal` and then `New Terminal`.<br />
   Your setup should now look similar to this:<br />
   ![Lab Setup Image](/labs/images/vscode-setup.png "Lab Setup")
4. Login to Chef Automate via the Chrome Browser, the browser should be open at the correct URL, if not ask the instructor for the URL.<br />
```
Username = workstation-x
Password = workstation!
```
Replace x with your workstation number given to you by the instructor.
## 2. Scan a Docker Container Directly
1. Docker is installed on the Linux node, we can see that there are no Docker containers running at the moment with:<br />
`docker ps` <br />
Lets create a container that we can scan, I have chosen the Docker delivered "docker/getting-started" container for this exercise: <br />
`docker run -dp 80:80 docker/getting-started` <br />
You can verifiy that it is running by going to your browser: <br />
`http://<public ip of your Linux Node>/` <br />
Now lets find out its container ID to use for InSpec scanning:<br />
`docker ps` <br />
``` docker
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                NAMES
b6d89059de64        docker/getting-started   "nginx -g 'daemon of…"   2 hours ago         Up 2 hours          0.0.0.0:80->80/tcp   zealous_buck
```
2. InSpec allows you to scan directly against against a Docker container. Lets first create a profile to use for our scan tests: <br />
`inspec init profile docker` <br >
(accept the license if prompted) <br />
`cd docker` <br />
Open the `controls/example.rb` file to see the InSpec tests that we are going to run. These are some out of the box tests to test the /tmp diretory. <br />
Run the InSpec profile against our Docker container (use your container id): <br />
`inspec exec . -t docker://b6d89059de64` <br />
```
Profile: InSpec Profile (docker)
Version: 0.1.0
Target:  docker://b6d89059de6439b02d6c84a3140109ce688924fb0d31f2e3ef5271d78bc98031

  ✔  tmp-1.0: Create /tmp directory
     ✔  File /tmp is expected to be directory

  File /tmp
     ✔  is expected to be directory

Profile Summary: 1 successful control, 0 control failures, 0 controls skipped
Test Summary: 2 successful, 0 failures, 0 skipped
```
You can create your own tests using the out of the box InSpec Resources detailed [here](https://www.inspec.io/docs/reference/resources/) or make use of one of the CIS InSpec profiles suitable for the operating system that your containers are sharing, look at the end of this lab to see a CIS profile being executed for the Docker Host. <br />

## 3. Scan the Docker Host
1. Using the InSpec out of the box resource `docker_container`. Lets create a new InSpec profile to scan the Docker Host: <br />
`cd ..`<br />
`inspec init profile docker_host`<br />
`cd docker_host` <br />
Edit `controls/example.rb` in the new profile to be like below: <br />
``` ruby
title "Check Containers on the Docker Host"

container_name = input("container_name")

# you add controls here
control "docker-1.0" do
  impact 0.7
  title "Check to the container configuration"
  desc "A longer description if required"
  describe docker_container(container_name) do
    it { should exist }
    it { should be_running }
    its('id') { should_not eq '' }
    its('image') { should eq 'docker/getting-started' }
    its('repo') { should eq 'docker/getting-started' }
    its('tag') { should eq nil }
    its('ports') { should eq '0.0.0.0:80->80/tcp' }
    its('command') { should eq 'nginx -g \'daemon off;\'' }
  end
end
```

Edit `inspec.yml` in the new profile to add our `inputs:` parameter section: <br />
``` ruby
name: docker_host
title: InSpec Profile
maintainer: The Authors
copyright: The Authors
copyright_email: you@example.com
license: Apache-2.0
summary: An InSpec Compliance Profile
version: 0.1.0
inputs:
- name: container_name
  required: true
  description: 'The name of the container to test'
  type: string
supports:
  platform: os
```

We are running all of our tests on a Linux machine that is running Docker, so we do not need to specify a target this time. If your Docker Host is remote then you can use `-t ssh://` or `-t winrm://` to set the target InSpec will use. <br />

Run this control against our Docker Host with: <br/>
`inspec exec . --input container_name=<contianer_name>` <br />
(Use the container_name you got from running `docker ps` earlier) <br /> <br />
```
Profile: InSpec Profile (docker_host)
Version: 0.1.0
Target:  local://

  ✔  docker-1.0: Check to the container configuration
     ✔  Docker Container strange_khorana is expected to exist
     ✔  Docker Container strange_khorana is expected to be running
     ✔  Docker Container strange_khorana id is expected not to eq ""
     ✔  Docker Container strange_khorana image is expected to eq "docker/getting-started"
     ✔  Docker Container strange_khorana repo is expected to eq "docker/getting-started"
     ✔  Docker Container strange_khorana tag is expected to eq nil
     ✔  Docker Container strange_khorana ports is expected to eq "0.0.0.0:80->80/tcp"
     ✔  Docker Container strange_khorana command is expected to eq "nginx -g 'daemon off;'"


Profile Summary: 1 successful control, 0 control failures, 0 controls skipped
Test Summary: 8 successful, 0 failures, 0 skipped
```
2. We can also specify the input parameters using an input file <br />
Create a new file under your `docker` directory called `inputs.yml` and add this content: <br />
`container_name: '<container_name>'` <br />
Again replace `<container_name>` with the name of your container. <br />
Lets run the test again but this time specifying the input file: <br />
`inspec exec . --input-file=inputs.yml` <br />

```
Profile: InSpec Profile (docker_host)
Version: 0.1.0
Target:  local://

  ✔  docker-1.0: Check to the container configuration
     ✔  Docker Container strange_khorana is expected to exist
     ✔  Docker Container strange_khorana is expected to be running
     ✔  Docker Container strange_khorana id is expected not to eq ""
     ✔  Docker Container strange_khorana image is expected to eq "docker/getting-started"
     ✔  Docker Container strange_khorana repo is expected to eq "docker/getting-started"
     ✔  Docker Container strange_khorana tag is expected to eq nil
     ✔  Docker Container strange_khorana ports is expected to eq "0.0.0.0:80->80/tcp"
     ✔  Docker Container strange_khorana command is expected to eq "nginx -g 'daemon off;'"


Profile Summary: 1 successful control, 0 control failures, 0 controls skipped
Test Summary: 8 successful, 0 failures, 0 skipped
```

3. Send the InSpec scan results to Chef Automate. First you need to create a `reporter.json` file like this in the `docker_host` directory, your instructor should have given you the Chef Automate Hostname and Token. Replace `<x>` with you workstation number. You can create the file by right clicking on the `docker_host` directory or in the terminal with `touch reporter.json`<br />
You will also need to create a UUID for your Docker Host scan, run `uuidgen` in your terminal.
``` json
{
  "reporter": {
    "automate" : {
      "stdout" : false,
      "url" : "https://<Chef Automate Hostname>/data-collector/v0/",
      "token" : "<Chef Automate Token>",
      "insecure" : true,
      "node_name" : "workstation-<x>-docker-host",
      "environment" : "docker",
      "node_uuid" : "<uuidgen>"
    },
    "cli" : {
      "stdout" : true
    }
  }
}
```
Lets run the scan again and report the results to Chef Automate.<br />
`inspec exec . --input-file=inputs.yml --config=reporter.json`<br />
The Chrome Browser should already be open, if not open it and ask your instructor for the Chef Automate URL to use.<br />
In Chef Automate Click the `Compliance` menu and then  the `Node` tab, observe your Docker Host node, there may be other nodes from your classmates. In Chef Automate we refer to everything as a node, so in this case our Docker Host scan is our node.<br />
![Chef Automate Compliance](/labs/images/docker-host-node.png "Chef Automate Compliance")<br />
Notice that we have one Passed Control and 8 tests executed.<br />
Click on the node that you scanned and expand the `+` on the right to see the detail of the scan.<br />
![Chef Automate Compliance](/labs/images/docker-host-node-detail.png "Chef Automate Compliance")<br />
Click the `Source` tab - the actual InSpec code that ran to perform the check:<br />
![Control Source](/labs/images/docker-host-node-source.png)<br />

4. What if we really do not want to run a controls in a profile? InSpec has a Waiver capability to allow you to do this.<br />
Under the `docker-host` directory create a `waiver.yml` file and add the following to it:<br />
``` yml
docker-1.0:
  expiration_date: 2021-02-28
  run: false
  justification: "Security have signed off not doing this check until the end of February 2021"
```
Run InSpec with the waiver like this:<br />
`inspec exec . --config=reporter.json --input-file=inputs.yml --waiver-file waiver.yml`<br />
Look in Chef Automate to see the waiver results, including the reason for the waiver:<br />
![Control Results](/labs/images/docker-host-waiver.png)<br />

5. Debugging a control (optional step). As you build out your tests it is useful to be able to debug them, we can use `pry` for that. Add the following `require "pry";binding.pry` line to your `example.rb` file.<br />

``` ruby
title "Check Containers on the Docker Host"

container_name = input("container_name")

# you add controls here
control "docker-1.0" do
  impact 0.7
  title "Check to the container configuration"
  desc "A longer description if required"
  require "pry";binding.pry
  describe docker_container(container_name) do
    it { should exist }
    it { should be_running }
    its('id') { should_not eq '' }
    its('image') { should eq 'docker/getting-started' }
    its('repo') { should eq 'docker/getting-started' }
    its('tag') { should eq nil }
    its('ports') { should eq '0.0.0.0:80->80/tcp' }
    its('command') { should eq 'nginx -g \'daemon off;\'' }
  end
end
```
Run your InSpec test again without the waiver this time:<br />
`inspec exec . --config=reporter.json --input-file=inputs.yml` <br/>

You should see output like this:<br />
``` ruby
From: /home/ec2-user/inspec-labs/docker_host/controls/example.rb:10 self.load_with_context:

     5: # you add controls here
     6: control "docker-1.0" do
     7:   impact 0.7
     8:   title "Check to the container configuration"
     9:   desc "A longer description if required"
 => 10:   require "pry";binding.pry
    11:   describe docker_container(container_name) do
    12:     it { should exist }
    13:     it { should be_running }
    14:     its('id') { should_not eq '' }
    15:     its('image') { should eq 'docker/getting-started' }

[1] pry(#<Inspec::Rule>)> 
```
This allows you to stop and inspect what is going on, for example you could execute the `docker_container` resource and see what methods are available: <br />
Type: <br />
`container=docker_container(container_name)`
``` ruby
[1] pry(#<Inspec::Rule>)> container=docker_container(container_name)
=> #<#<Class:0x0000000007656588>:0x00000000076560d8
 @__backend_runner__=Inspec::Backend::Class @transport=Train::Transports::Local::Connection,
 @__resource_name__=:docker_container,
 @opts={:name=>"strange_khorana"},
 @resource_exception_message=nil,
 @resource_failed=false,
 @resource_skipped=false,
 @supports=[{:platform=>"unix"}]>
 ```
 `cd container` <br />
 `ls`
 ``` ruby
[2] pry(#<Inspec::Rule>)> cd container
[3] pry(#<#<Class:0x0000000007656588>>):1> ls
Inspec::Resource#methods: check_supported!  check_supports  fail_resource  inspec  resource_exception_message  resource_failed?  resource_skipped?  skip_resource  supersuper_initialize
Inspec::Resources::DockerObject#methods: exist?  id
docker_container#methods: command  image  labels  ports  repo  running?  status  tag  to_s
self.methods: __pry__
instance variables: @__backend_runner__  @__resource_name__  @opts  @resource_exception_message  @resource_failed  @resource_skipped  @supports
locals: _  __  _dir_  _ex_  _file_  _in_  _out_  pry_instance
```
You can then execute methods to see the data, try calling some methods as below: <br />
``` ruby
[4] pry(#<#<Class:0x0000000007656588>>):1> id
=> "60805c06b35aeb543de126a447a98da1afae60c9b2bf29c38a28ef899094a296"
[5] pry(#<#<Class:0x0000000007656588>>):1> image
=> "docker/getting-started"
[6] pry(#<#<Class:0x0000000007656588>>):1> repo
=> "docker/getting-started"
[7] pry(#<#<Class:0x0000000007656588>>):1> tag
=> nil
[8] pry(#<#<Class:0x0000000007656588>>):1> ports
=> "0.0.0.0:80->80/tcp"
[9] pry(#<#<Class:0x0000000007656588>>):1> command
=> "nginx -g 'daemon off;'"
[10] pry(#<#<Class:0x0000000007656588>>):1> exit-program
```

6. Using the Chef Automate supplied CIS Docker Benchmark.

[Back to the Lab Index](../README.md)
