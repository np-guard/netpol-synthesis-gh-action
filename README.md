# Automatically generate K8s NetworkPolicies for your app

## About
This action automates the generation of Kubernetes NetworkPolicies for a given application. It will first scan your repository for YAML files which define various Kubernetes resources (e.g., Deployments, Services, ConfigMaps). It will then analyze these files and extract all network connections required for your application to work. Finally, this action will synthesise K8s NetworkPolicies that allow these connections and nothing more.

This action is part of a wider attempt to provide [shift-left automation for generating and maintaining Kubernetes Network Policies](https://np-guard.github.io/).

## Inputs
### `corporate-policies`
(Optional) A list of space-separated corporate policy files to consider during synthesis. Generated NetworkPolicies will never violate any of these policies. Files can be given as a relative path to files in the current repository or as URLs to files stored on GitHub. File format is described [here](https://github.com/np-guard/baseline-rules).

## Outputs
### `synth-artifact`
The name of the GitHub Action Artifact containing synthesis results
### `synth-netpols-file-name`
The name of the file in the artifact, which contains the synthesized NetworkPolicies yaml
### `connection-list-file-name`
The name of the actual file in the artifact, which contains the topology-analysis output

## Example usage
### Basic usage - discovered connectivity and synthesized NetworkPolicies are stored in a workflow artifact
```yaml
name: synth-network-policies
on:
  push:
    branches:
      - 'master'
jobs:
  synthesize-netpols:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: np-guard/netpol-synthesis-gh-action@v3
```

### Automatically open a pull request for synthesized NetworkPolicies
```yaml
name: synth-network-policies-and-open-PR
on:
  workflow_dispatch:

jobs:
  synthesize-netpols:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Synthesize netpols
        id: synth-netpol
        uses: np-guard/netpol-synthesis-gh-action@v3
        with:
          corporate-policies: >
            https://github.com/np-guard/baseline-rules/blob/master/examples/ciso_denied_ports.yaml
      - uses: actions/download-artifact@v2
        with:
          name: ${{ steps.synth-netpol.outputs.synth-artifact }}
          path: release
      - name: Commit changes and create a PR
        shell: sh
        run: |
          git config user.name ${{ github.actor }}
          git config user.email '${{ github.actor }}@users.noreply.github.com'
          export branch_name=set_netpols_$(date +%s)
          git checkout -b $branch_name
          git add release/${{ steps.synth-netpol.outputs.synth-netpols-file-name }}
          git commit -m"adding network policies to enforce minimal connectivity"
          git push --set-upstream origin $branch_name
          gh pr create -B master -H $branch_name -t "Adding network policies to enforce minimal connectivity" -b "Automatically generated NetworkPolicies" 
        env:
          GITHUB_TOKEN: ${{ secrets.REPO_SCOPED_TOKEN }}
```
### Synthesize with corporate policies
```yaml
name: synth-network-policies
on:
  workflow_dispatch:

jobs:
  synthesize-netpols:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: np-guard/netpol-synthesis-gh-action@v3
        with:
          corporate-policies: >
            https://github.com/np-guard/baseline-rules/blob/master/examples/ciso_denied_ports.yaml
            https://github.com/np-guard/baseline-rules/blob/master/examples/restrict_access_to_payment.yaml
```
