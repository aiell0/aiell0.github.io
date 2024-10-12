---
title: "Arch Adventures: Installing AWS SSM"
date: 2023-07-04T16:25:16-04:00
draft: false
---

# Arch Adventures: Installing AWS SSM

One of my DevOps side projects as of late has been setting up blockchain nodes. Since proof of stake has really taken hold a few years back, we are seeing a Cambrian explosion of projects all using proof of stake to secure their network. It is the perfect environment for DevOps engineers to practice their craft as there is and will need to be a huge DevOps need to power these networks as they continue to grow.

One such project suggested using Arch Linux as a means to run their software. I love a good challenge and have never used Arch Linux before. So, I dove in.

## So what is Arch Linux?

Arch Linux, also known as the "Linux for power users," has been making waves in the open-source community since its inception in 2002. Its philosophy of simplicity, user-centricity, and adherence to the KISS (Keep It Simple, Stupid) principle has earned it a reputation as a versatile and customizable distribution that allows users to create a tailor-made Linux experience. While other distributions may hold your hand with pre-configured setups and polished graphical interfaces, Arch Linux leaves it up to you to design your system from the ground up, making it a popular choice for those who love to tinker and learn.

This is a great use case for blockchain nodes because they need to be as performant as possible and with the least bloat. As we will be running this in a cloud environment, storage and compute is not cheap, and having extra to run software you don't need will eat into your profits as an operator. The wiki for Arch is quite extensive and you can find what you need to help you customize your OS on there.

## Running Arch on AWS
Arch is not currently supported by AWS, which means that many of the products that come baked in with Amazon Linux, Ubuntu, and CentOS are not installed on to Arch by default. That being said, there is a fair amount of support from the Linux community, as documented here on the wiki. There are even APIs you can use to curl them yourself. I'm more of a Terraform kind of guy, so I just use the following code to get it.

```
data "aws_ami" "arch" {
  most_recent = true

  filter {
      name   = "name"
      values = ["arch-linux-ec2-*"]
    }
    filter {
      name   = "root-device-type"
      values = ["ebs"]
    }
    filter {
      name   = "architecture"
      values = ["x86_64"]
    }
    filter {
      name   = "virtualization-type"
      values = ["hvm"]
    }
    owners = ["647457786197"]
  }
```
You will have to use SSH key pairs at first, because Session Manager does not come baked in to it. We will fix this later. You will notice however that the AWS CLI is already installed though. Yay!

You will then have to initialize the instance, which we do using the following code.
```
pacman-key --init
pacman-key --populate
reflector --country "US" --protocol https,http --score 20 --sort rate --save /etc/pacman.d/mirrorlist
pacman -Syu --noconfirm
```

Pacman is the package manager for Arch Linux. It works a bit differently than other distros. The command `pacman-key --init` sets up the initial configuration and generates a master key pair for your system. This key pair is used to manage the signing and verification of package keys. Running this command is a crucial step in setting up a secure environment for pacman to operate in.

The `pacman-key --populate` command is used to import the public keys of the developers and packagers for a specific repository into your local pacman keyring. These public keys are used by pacman to verify the signatures of packages and repository databases, ensuring their authenticity and integrity.

The `reflector` command is used to retrieve and filter a list of the most up-to-date and fastest Arch Linux mirror servers based on certain criteria, and save the resulting list to a file. I am using the US as my server is in us-east-1, but you should use the CLI to query your region and change the country code accordingly to make the script idempotent.

Finally, the `pacman -Syu --noconfirm` command updates all of the packages. Now we are ready to install AWS Session Manager.

## Installing AWS Session Manager
First things first, we need to install the required packages:

`pacman -S go git make cmake gcc dhcpcd zip --noconfirm`

Since SSM is not supported on Arch, we will have to build it ourselves which is why you need to install all of these tools. You then set up your go environment like so:

```
export GOCACHE="/root/.cache/go-build"
export GOMODCACHE="/root/go/pkg/mod"
export GOENV="/root/.config/go/env"
export GOPATH="/root/go"
export GOARCH=amd64
export GOOS=1inux
```
I was having networking issues when running the SSM agent, the following two commands (along with installing the service above) fixed them for me:

```
systemctl start dhcpcd.service
systemctl enable dhcpcd.service
```

Now we build the agent! The Github repository for the agent is located here, and we clone it like so. Note that we use the opt directory because this is third-party software that we don't want to conflict with other software on our system:

```
git clone https://github.com/aws/amazon-ssm-agent.git /opt/aws/amazon-ssm-agent
```

Then run the following commands to build it. The SSM agent is pure Go code so we don't need the C compiler:

```
cd /opt/aws/amazon-ssm-agent
CGO_ENABLED=0 make build-linux
```

This takes a while so go make a coffee while you wait. Once it's completed, you will see a bin directory containing the SSM agent binaries. Congrats!
Now the next step is to make the SSM agent itself a systemd service. Use the following command to do that:

```
cat << EOF > /etc/systemd/system/ssmagent.service
[Unit]
Description=AWS SSM Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Restart=always
RestartSec=2
ExecStart=/opt/aws/amazon-ssm-agent/bin/linux_amd64/amazon-ssm-agent

[Install]
WantedBy=multi-user.target
EOF
```

Then we enable it so it starts on boot, and start to start the service now:

```
systemctl start ssmagent
systemctl enable ssmagent
```

Now type in `systemctl status ssmagent` and you should see the service running!

## Conclusion

I hope you found this article on getting started with Arch Linux on AWS useful. I plan on posting more about Arch Linux in the future as I continue into these side journeys, so stay tuned!
