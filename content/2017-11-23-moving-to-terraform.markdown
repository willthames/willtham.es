


# Things I think I'll miss about Ansible

Did I mention how much I love hierarchical inventory? Well written hierarchies can massively reduce
the amount of repetition. Terraform doesn't seem to have quite the same capability, but we'll see how it goes.

No one really *loves* YAML, but it's better than JSON (the lowest of bars) and it seems superior to HCL
(the fact that the H stands for Hashicorp is a bit of a sign of NIH).

Jinja templating is super powerful, with loads of filters. Combined with Ansible's lookup plugins

Loops. I like being able to do

```
- ec2_vpc_subnet:
    vpc: "{{ vpc_id }}"
    az: "{{ region }}{{ item.az }}"
    cidr: "{{ item.cidr }}"
    name: "{{ item.name }}"
  with_items:
  - name: vpc-X-app-Y-subnet-a
    az: ap-southeast-2a
    cidr: 10.1.0.0/16
  - name: vpc-X-app-Y-subnet-b
    az: ap-southeast-2b
    cidr: 10.2.0.0/16
```

In reality of course, all the details of all the subnets are just in inventory, and the
`with_items` loop just becomes `with_items: "{{ subnets }}"`

Ease of contribution - it took a long time to get here, but I have commit rights and people know who I am.
I can comfortably review other people's code, and I know python and Ansible's coding convention expectations
well enough to easily make improvements.

Running bleeding edge modules on a stable core. Ansible's pluggable module system means that if I need a fix to
a module, I can just take that module and dump it in my library directory without needing to run all of
Ansible on the bleeding edge.



# Things I look forward to with Terraform

AWS module coverage

Community size - while the size of the Terraform community might be smaller than Ansible's, I'd imagine
Terraform's AWS community will be huge in comparison. This means more eyes to catch bugs, more people who can
fix bugs, and new AWS features will be much more likely readily available



