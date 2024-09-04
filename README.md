# Ansible Playbook to detect installed Services in Amazon Machine Images 

**Hello!** I appreciate your interest in this file.
I am outlining a challenge I encountered when verifying running services on a newly built Packer Linux Amazon machine image. I will make a few assumptions regarding previous knowledge of AMIs and some Packer experience. 

## Links and Research

You have likely encountered Amazon machine images in your work, but if you want to delve deeper, I recommend reading the documentation to enhance your understanding. 
https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html

I will maybe create another post in the future with more details on the Packer build steps, but for now, you should check out their docs to get familiar with the Amazon Provisioner:

https://developer.hashicorp.com/packer/integrations/hashicorp/amazon

Before I share how I arrived at the solution, I want to share a few more links I used to research this and see how others tackled this problem.

This post from a cloud engineer at Experian got me thinking about going beyond the build process to validate. 
https://www.hashicorp.com/resources/managing-a-golden-image-factory-across-all-major-cloud-platforms

If you watch this, you will notice that they briefly mention using the Ansible provisioner to pull down all the agents, set up the hardening configs, etc.

My provisioning, installations, and everything else are part of the build, so I didn't need to rewrite that in Ansible. 

The example workflows all involved launching one of the AMIs you had built and performing other scans. Either using another set of tools or using a "Pre-Authorized Appliance" from AWS. Here is an example: https://aws.amazon.com/marketplace/reviews/reviews-list/B01LXCD58S


### What I learned
This approach would increase my complexity and introduce further overhead, such as new keys, managing key lifecycles, and developing permission models. I would also have to figure out how to report and monitor these findings. 

This approach all seemed like ***pre-mature over-optimization*** from the start. (This is usually something that will thwart productivity) 

I decided to shift this process left and actually work it out before the AMI is shared across all of the accounts in the organization.

### What would the solution look like?
I initially thought the easiest way to do this would be to use another script (either Bash or Python) and create some conditionals.

In theory, it seemed trivial to obtain the operating system's list of running services and create a loop of only the services I was interested in.

Then, traverse over the loop and call out exceptions as needed. No big deal, right? As I started, I soon found that the layers of abstraction would be an issue. 

For instance, the Packer process, which would handle the shell provisioning or scripting, runs from within a Docker container and uses an SSH session to instantiate the new AMI. This makes it challenging to interpret variables or certain strings and pass them along. 

I thought about it more and started testing the Ansible Packer provisioner. https://developer.hashicorp.com/packer/integrations/hashicorp/ansible/latest/components/provisioner/ansible

One nice aspect of this approach is that it leverages the ephemeral SSH keys used in the Packer session.

"It dynamically creates an Ansible inventory file configured to use SSH, runs an SSH server, executes ansible-playbook, and marshals Ansible plays through the SSH server to the machine being provisioned by Packer"

Once I figured out how to get this to work, I wanted to ensure that I only needed 1 playbook that would work across Ubuntu, Debian, Amazon, Linux, etc.

```
provisioner "ansible" {
    extra_arguments = [
        "--check",
        "-vv"
      ]

    playbook_file = "../../service-check-playbook.yml"
  }
}
```
With this playbook in the root directory and the Packer provisioner all configured, I was ready to start debugging the operating systems and extracting information from them. Note the extra arguments here. The -vv is for verbosity. This can be turned off if you prefer. The other key component here is the --check switch. This ensures that these commands are only printed. This is the equivalent of a "dry run" meaning no changes will be made but we can read , debug and create variables to loop over. Keep reading to see how.

## Making Progress

https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html#conditionals-based-on-ansible-facts

Ansible has a built-in function for assertioons as well as conditionals. It is also able to set the Operating system values as conditionals. This is something known as Ansible Facts. This feature is great because I would not want to have to extract those myself with something like:

`cat /etc/os-release | grep PRETTY_NAME
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"`

I will allow the comments to explain how the playbook would work on your systems. I will use some mainstream tools and agents that you may find in a standard corporate enterprise.

Another aspect that I like about this approach is the portability of it. If I had only used Bash or Python I may encounter more errors across different operating systems and environments. I may have also had some inconsistent results when probing the operating system.

This modular ability is functional right away. One example of this is that in AWS, there is a security tool called SSM. https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html

SSM can be managed in a few ways depending on the type of Amazon instance. The ability to quickly discern between operating systems and apply the checks made this task easier. Another tool, td-agent, also has a different package name depending on the operating system. This approach allowed me to take those types of variances into consideration.


Please take a look at the playbook and try it out.

## Further reading and other links

https://mywiki.wooledge.org/BashWeaknesses

https://docs.ansible.com/ansible/2.9/modules/assert_module.html

https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html

https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html
