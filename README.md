# Automatically generate K8s NetworkPolicies for your app

## About
This action automates the generation of Kubernetes NetworkPolicies for a given application. It will first scan your repository for YAML files which define various Kubernetes resources (e.g., Deployments, Services, ConfigMaps). It will then analyze these files and extract all network connections required for your application to work. Finally, this action will synthesise K8s NetworkPolicies that allow these connections and nothing more.

## Inputs
### `corporate-policies`
(Optional) A list of space-separated corporate policy files to consider during synthesis. Generated NetworkPolicies will never violate any of these policies. Files can be given as a relative path to files in the current repository or as URLs to files stored on GitHub. File format is described [here](https://github.com/shift-left-netconfig/baseline-rules).

## Outputs
### `netpols`
Full path to the synthesized NetworkPolicies yaml (under the workflow's GitHub workspace)
### `topology`
Full path to the topology-analysis output (under the workflow's GitHub workspace)

## Example usage
### Storing discovered connectivity and synthesized NetworkPolicies as workflow artifacts
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
      - name: Synthesize netpols
        id: synth-netpol
        uses: shift-left-netconfig/netpol-synthesis-gh-action@v1
      - name:  Upload netpols yaml
        uses: actions/upload-artifact@v2
        with:
          name: netpols.yaml
          path: ${{ steps.synth-netpol.outputs.netpols }}
      - name:  Upload app network topology
        uses: actions/upload-artifact@v2
        with:
          name: app-net-top.json
          path: ${{ steps.synth-netpol.outputs.topology }}
```

### Open a pull request for synthesized NetworkPolicies (with corporate policies)
```yaml
name: synth-network-policies
on:
  workflow_dispatch:

jobs:
  synthesize-netpols:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Synthesize netpols
        id: synth-netpol
        uses: shift-left-netconfig/netpol-synthesis-gh-action@v1
        with:
          corporate-policies: >
            https://github.com/shift-left-netconfig/baseline-rules/blob/master/examples/ciso_denied_ports.yaml
            https://github.com/shift-left-netconfig/baseline-rules/blob/master/examples/restrict_access_to_payment.yaml
      - name: Commit changes
        shell: sh
        run: |
          cd ${{ github.workspace }}
          cp ${{ steps.synth-netpol.outputs.netpols }} release/netpols.yaml
          git config user.name ${{ github.actor }}
          git config user.email '${{ github.actor }}@users.noreply.github.com'
          git add release/netpols.yaml
          git commit -m"adding network policies to enforce minimal connectivity"
      - name: Open PR
        uses: peter-evans/create-pull-request@v3
        with:
          title: Automatic updates to NetworkPolicies
          branch: update-netpols
          branch-suffix: timestamp
  ```
