+++
title = "Ansible Inventory Diff Github Action"
date = 2020-09-09T09:00:00Z
+++

At Skedulo we have a [pretty sophisticated inventory](/2017/10/31/making-the-most-of-inventory.html)
set up that uses the [generator plugin](/2017/11/01/generating-inventory.html)
and hundreds of inventory directories to manage the configuration of dozens of
different microservices and infrastructure services across several environments and regions.

[ansible-inventory-diff](https://github.com/Skedulo/ansible-inventory-diff) is a tool
that takes two git branches and runs `ansible-inventory` against both branches and compares
the results, showing what hosts have inventory changes and what those changes are.

We've been using ansible-inventory-diff to
protect ourselves from unexpected issues as a result of inventory changes for
a while now. But with Github Actions being widely available for a while now,
we wanted to make it more obvious in the pull request what will change

It took quite a lot of iterations to get the action behaviour correct
but it's now on the [Github Marketplace](https://github.com/marketplace/actions/ansible-inventory-diff)

We use the configuration below to do the following:

* Check out the code
* See if any inventory files differ between the pull request and main branch
* If they do, run ansible-inventory-diff
* Output the full result in the action workflow (from the `result` output) to easily check
  that full output from the pull request
* Add a comment containing at most the first 200 lines (available as the `snippet` output)
* If there is no difference, or we didn't run ansible-inventory-diff, explicitly comment
  to say that

Running ansible-inventory-diff against our inventory takes 5 minutes
(I haven't checked for a while how big the output from `ansible-inventory` is but I'm pretty
sure it's huge and rendering that is what takes the time) - I hope I can find some optimizations
at some point before we run out of Github Action minutes!

```
on: pull_request

jobs:
  ansible_inventory_diff:
    runs-on: ubuntu-latest
    name: ansible-inventory-diff
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
      - name: Check if ansible-inventory-diff should run
        run:
          echo "::set-output name=diff::$(git diff --name-only origin/main -- inventory)"
        id: check
      - name: Run ansible inventory diff
        id: run
        uses: Skedulo/ansible-inventory-diff@v1.5
        with:
          base-ref: origin/main
        if: ${{ steps.check.outputs.diff != '' }}
      - name: Ansible Inventory Diff output
        shell: bash
        run: echo "${{ steps.run.outputs.result }}"
        if: ${{ steps.check.outputs.diff != '' }}
      - name: Comment PR
        uses: unsplash/comment-on-pr@v1.2.0
        with:
          msg: "*ansible-inventory-diff*\n\n```${{ steps.run.outputs.snippet }}\n```"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        if: ${{ steps.check.outputs.diff != '' && steps.run.outputs.snippet != '' }}
      - name: Comment PR when diff is empty
        uses: unsplash/comment-on-pr@v1.2.0
        with:
          msg: "*ansible-inventory-diff*\n\nEmpty diff"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        if: ${{ steps.check.outputs.diff == '' || steps.run.outputs.snippet == '' }}
```
