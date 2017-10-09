# Puppet

## Introduction

- Puppet manages configurtion data, including users, packages, processes and services.
  - Brings systems into compliance with predefined computer policy.
- __Declarative__: You are not required to write out hte process for evaluating and adjusting each platform.
  - You utilize Puppet's configuration langauge to declare the final state of compute resources.
- __Imperative__: Language that describes the actions to perform.
  - It must define every change that should be followed to acheive the desired configuration.

### How Puppet Works

- Any managed node contains ana application, the Puppet agent.
- The agent evaluates and applies Puppet manigests.
  - __Manifest__: A file containing Puppet code that describes the desired state of a node.
- If configured to use a Puppet server (also called a master), the agent will send the node's data to the server and receive back a catalog.
  - __Catalog__: A document containing the specificy policy to apply to a node.
- Puppet allows you to classify and categorize nodes to limit what resources are applied using information about the nodes themselves.
  - Hostname
  - OS
  - Node Type
  - Puppet version
  - Other Custom Data
- A single agent is responsible only for the node it is installed on.

### Is Puppet DevOps

- It does not replace DevOps practices adopted by a team, it only facilitates them.
- Puppet makes it possible to track and manage changes in any workflow.

