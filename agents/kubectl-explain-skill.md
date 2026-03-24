---
name: kubectl-explain-skill
description: 'Skill to guide users on using k explain for Kubernetes resources.'
---

## Purpose
This skill is designed to assist users in understanding Kubernetes resources and their fields using the `k explain` command. It provides step-by-step guidance on how to retrieve detailed documentation for specific Kubernetes resources and their paths.

## Tools
- `k explain`: To fetch documentation for Kubernetes resources and their fields.

## Workflow
1. Identify the Kubernetes resource you want to explore (e.g., Pod, Deployment, Service).
2. Use the command `k explain <resourceName>` to get an overview of the resource.
3. If you need details about a specific field, use `k explain <resourceName>.<resourcePath>`.
   - Example: `k explain pod.spec.containers`.
4. Review the output to understand the purpose and usage of the resource or field.

## Example Prompts
- "Explain the fields of a Pod resource."
- "What does the `spec.containers` field in a Deployment mean?"
- "Guide me on using `k explain` for the Service resource."

## Limitations
- This skill assumes that the user has `kubectl` installed and configured to interact with a Kubernetes cluster.
- The skill relies on the Kubernetes API documentation available in the cluster.

## Example Commands
- `k explain pod`
- `k explain deployment.spec`
- `k explain service.spec.ports`

## Adjustments for Local Cluster
This skill is optimized for users working with local Kubernetes clusters. Ensure that your `kubectl` context is set correctly to interact with the desired cluster.

## Related Skills
- Kubernetes resource management
- Debugging Kubernetes configurations