I would like to share my initial idea and get feedback on the approach

cc: @sbueringer @fabriziopandini (tagging @ykakarap also as he has contributed to MachineDeployment rolloutAfter feature https://github.com/kubernetes-sigs/cluster-api/pull/8216 which is similar to this)

#### Feature Description
We want to specify a duration in spec after which machines should get replaced. `MachineExpiry{Minutes,Hours,Days}`. This feature should be available for machines managed by KCP and for machines managed by MachineDeployments/MachineSets.

#### Related exisiting features
- RolloutAfter in MachineDeployment and KCP. 
```go
    // RolloutAfter is a field to indicate a rollout should be performed
	// after the specified time even if no changes have been made to the
	// KubeadmControlPlane.
	// Example: In the YAML the time can be specified in the RFC3339 format.
	// To specify the rolloutAfter target as March 9, 2023, at 9 am UTC
	// use "2023-03-09T09:00:00Z".
    // +optional
	RolloutAfter *metav1.Time `json:"rolloutAfter,omitempty"`
```
- RolloutBefore.CertificatesExpiryDays in KCP (cert based rollout is only for controlplane machines)
```go
type KubeadmControlPlaneSpec struct {
    // RolloutBefore is a field to indicate a rollout should be performed
	// if the specified criteria is met.
	// +optional
	RolloutBefore *RolloutBefore `json:"rolloutBefore,omitempty"`
}

// RolloutBefore describes when a rollout should be performed on the KCP machines.
type RolloutBefore struct {
  // CertificatesExpiryDays indicates a rollout needs to be performed if the
  // certificates of the machine will expire within the specified days.
  // +optional
  CertificatesExpiryDays *int32 `json:"certificatesExpiryDays,omitempty"`
}
```


### API Change Proposal

#### Option 1: Specify machine expiry under RolloutBefore
- Add `MachineExpiry` to existing RolloutBefore of KubeadmControlPlaneSpec
- Add `RolloutBefore.MachineExpiry` for MachineDeploymentSpec

In this approach, we are trying to extend the existing API. We want to rollout once it reached the specified duration. Adding it under `RolloutBefore` struct looks not appropriate. Please suggest your inputs

#### Option 2: Specify machine expiry under new struct Rollout
```go
type KubeadmControlPlaneSpec struct {
  // Rollout indicates different capabilites to
  // rollout the machine when the specified conditions are met
  Rollout *Rollout `json:"rollout,omitempty"`
}  
```
```go
type MachineMaintenance struct {
  // MachineExpiry indicates the duration after which the machine
  // will be rolled out. If Creation time - Current time  > duration,
  // then rollout and replace the expired machine
  MachineExpiry *MachineExpiry `json:"machineexpiry,omitempty"`
}

type MachineExpiry struct {
  Days     *int32 `json:"days,omitempty"`
  Hours    *int32 `json:"hours,omitempty"`
  Minutes  *int32 `json:"minutes,omitempty"`
}
```

In this new `MachineMaintenance` struct, we can define MachineExpiry. We can add other machine maintenance to this struct in the future.

#### Code Changes:
- Webhook: Validate if MachineExpiry is greater than or equal to 2 hours
- Controller Logic: When MachineExpiry is specified and if it crossed for the machine's creationTimestamp, then recreate the machine
  - For KCP, return true in https://github.com/kubernetes-sigs/cluster-api/blob/main/util/collections/machine_filters.go#L223 which will rollout the controlplane machines
  - For MachineDeployment, create nil NewMachineSet from https://github.com/kubernetes-sigs/cluster-api/blob/main/internal/controllers/machinedeployment/mdutil/util.go#L420 which will create a new MachineSet

> Note: I am new to cluster-api repo and my logic may not be fully correct. I'll add the complete logic, unit tests, manually test it and then finally raise the PR. Please let me know if I am in the right direction regarding the api changes and code changes.