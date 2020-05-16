---
One of the valid criticisms of Ansible is that it has no knowledge of previous state, so on
every run it has to work out the current state, determine the difference between current state
and desired state, and then, if differences exist, make reality match that desired state. This
can be quite slow, particularly for remote resources accessed over rate-limited or long-latency
APIs.

Additionally, Ansible can't clean up after itself - there's no way of running a playbook to
create an environment, run a bunch of tests and then destroy that environment without lovingly
handcrafting the teardown playbook as well.

And then there's maintaining changes to what can be considered Ansible's primary key of resources.
For example, the primary key of an AWS subnet is really the cidr block, in that that's all that's
required to delete one. If you create a bunch of subnets, and then decide to change the CIDR block
of those subnets, it will create a whole new set of subnets, without cleaning up the old subnets.
Given that Ansible would also have no idea of what's using the old subnets, that's not a terrible
thing, but there's no way to manage CIDR block changes without explicitly adding a task to clean
up the old subnets, a task that would be completely redundant in all future playbooks.


## ansible + state

So, given that there are benefits to knowing about state, what needs to be done to add state to
Ansible? There are a few changes required:

* Add a `state` strategy plugin to Ansible that allows Ansible to determine a sensible task
  ordering and only run the tasks that need changes, as well as cleanup tasks to remove resources
  that are no longer required.
* Add an `AnsibleStateModule` from which state-based modules can inherit, to provide some common
  functionality such as a `compare` method and a `run` method
* Ensure that action plugins (debug, fail, assert et al. can run in state mode)
* Add command line flags and playbook attributes to manage whether state should be validated
  and enforced (this helps protect against drift)
* Add state file settings to ansible configuration. TODO: Allow state file to be on S3 or similar.
* Update modules to make use of state. Such modules should inherit from AnsibleStateModule, and
  provide methods such as `create`, `delete`, `update`, `get` etc.

To use these changes, the following is required:
* Setting of state file and lock file locations in ansible.cfg or environment variables
* Use `strategy: state` in playbooks or
  [configuration](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-strategy)
* Set `resource_id` for all tasks that support state - this is the new primary key


## Testing scenarios

* Create a simple module and verify that it works as expected, including validating state that
  no longer matches reality, and enforcing state (the latter really corrects configuration drift)
* Test various task orderings, particularly with `depends_on`
* Build up a proper AWS environment and tear it down (if VPC deletion works as part of teardown,
  then teardown is pretty good!)
* Implement state for k8s module (the current implementation lends itself to this quite readily)
* Test facts modules
* Check mode + tidy up

## Missing functionality

* What to do when a resource needs to be recreated to match desired state, particularly in terms
  of dependencies
* Handling of loops
* Defining `depends_on` properly
* Safe mode (merely a report of what would be tidied)
* Handling of resource recreation due to state change (e.g. moving subnet CIDR blocks that other
  resources depend upon) - either delete dependent, delete depended, create depended, create dependent,
  or create depended, modify dependent, delete old depended (both scenarios will be required for
  different dependent resources)

## Questions

* How to handle facts modules (or info modules in the preferred vernacular). Such modules should not
  be needed for self-contained environments, but the scenario where each state file represents a
  small part of a larger environment is both expected and encouraged. As such data from outside of the
  state will be required - this should likely not be cached *or* stored in state.


## Desired benefits

* Improved performance, particularly of re-runs. We expect the EKS playbook to be faster than
  it's non-state equivalent on a re-run.
