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
## 2. Scan a docker container
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
1. Using InSpec out of the box resource docker_container. Lets create a new InSpec profile to scab the Docker Host: <br />
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


2. Using the Chef Automate supplied CIS Docker Benchmark.

[Back to the Lab Index](../LAB_INDEX.md)
