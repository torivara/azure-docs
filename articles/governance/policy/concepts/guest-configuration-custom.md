---
title: Changes to behavior in PowerShell Desired State Configuration for guest configuration
description: This article provides an overview of the platform used to deliver configuration changes to machines through Azure Policy.
ms.date: 05/31/2021
ms.topic: how-to
---
# Changes to behavior in PowerShell Desired State Configuration for guest configuration

Before you begin, it's a good idea to read the overview of
[guest configuration](./guest-configuration.md).

[A video walk-through of this document is available](https://youtu.be/nYd55FiKpgs).

Guest configuration uses
[Desired State Configuration (DSC)](/powershell/scripting/dsc/overview/overview)
version 3 to audit and configure machines. The DSC configuration defines the
state that the machine should be in. There are many notable differences in how
DSC is implemented in guest configuration.

## Guest configuration uses PowerShell 7 cross platform

Guest configuration is designed so the experience of managing Windows and Linux
can be consistent. Across both operating system environments, someone with
PowerShell DSC knowledge can create and publish configurations using scripting
skills.

Guest configuration only uses PowerShell DSC version 3 and doesn't rely on the
previous implementation of
[DSC for Linux](https://github.com/Microsoft/PowerShell-DSC-for-Linux)
or the "nx" providers included in that repository.

Guest configuration operates in PowerShell 7.1.3 for Windows and PowerShell 7.2
preview 6 for Linux. Starting with version 7.2, the `PSDesiredStateConfiguration`
module moved from being part of the PowerShell installation and is instead
installed as a
[module from the PowerShell Gallery](https://www.powershellgallery.com/packages/PSDesiredStateConfiguration).

## Multiple configurations

Guest configuration supports assigning multiple configurations to
the same machine. There are no special steps required within the
operating system of guest configuration extension. There's no need to configure
[partial configurations](/powershell/scripting/dsc/pull-server/partialConfigs).

## Configuration mode is set in the package artifact

When creating the configuration package, the mode is set using the following
options:

- _Audit_: Verifies the compliance of a machine. No changes are made.
- _AuditandSet_: Verifies and Remediates the compliance state of the machine.
  Changes are made if the machine isn't compliant.

The mode is set in the package rather than in the
[Local Configuration Manager](/powershell/scripting/dsc/managing-nodes/metaConfig#basic-settings)
service because it can be different per configuration, when multiple
configurations are assigned.

## Parameter support through Azure Resource Manager

Parameters set by the `configurationParameter` property array in
[guest configuration assignments](guest-configuration-assignments.md)
overwrite the static text within a configuration MOF file when the file is
stored on a machine. Parameters allow for customization and changes to be controlled
by an operator from the service API without needing to run commands within
the machine.

Parameters in Azure Policy that pass values to guest configuration
assignments must be _string_ type. It isn't possible to pass arrays through
parameters, even if the DSC resource supports arrays.

## Sequence of events

When guest configuration audits or configures a machine the same
sequence of events is used for both Windows and Linux. The notable change in
behavior is the `Get` method is called by the service to return details about
the state of the machine.

1. The agent first runs `Test` to determine whether the configuration is in the
   correct state.
1. If the package is set to `Audit`, the Boolean value returned by the function
   determines
   if the Azure Resource Manager status for the Guest Assignment should be
   Compliant/Not-Compliant.
1. If the package is set to `AuditandSet`, the Boolean value determines whether
   to remediate the machine by applying the configuration using the `Set` method.
   If the `Test` method returns False, `Set` is run. If `Test` returns True, then
   `Set` isn't run.
1. Last, the provider runs `Get` to return the current state of each setting so
   details are available both about why a machine isn't compliant and to confirm
   that the current state is compliant.

## Trigger Set from outside machine

A challenge in previous versions of DSC has been correcting drift at scale
without a lot of custom code and reliance on WinRM remote connections. Guest
configuration solves this problem. Users of guest configuration have control
over drift correction through
[Remediation On Demand](./guest-configuration-policy-effects.md#remediation-on-demand-applyandmonitor).

## Maximum size of custom configuration package

In Azure Automation state configuration, DSC configurations were
[limited in size](../../../automation/automation-dsc-compile.md#compile-your-dsc-configuration-in-windows-powershell).
Guest configuration supports a total package size of 100MB (before
compression). There is no specific limit on the size of the MOF file within
the package.

## Special requirements for Get

The function `Get` method has special requirements for Azure Policy guest
configuration that haven't been needed for Windows PowerShell Desired State
Configuration.

- The hashtable that is returned should include a property named **Reasons**.
- The Reasons property must be an array.
- Each item in the array should be a hashtable with keys named **Code** and
  **Phrase**.

The Reasons property is used by the service to standardize how compliance
information is presented. You can think of each item in Reasons as a "reason"
that the resource is or isn't compliant. The property is an array because a
resource could be out of compliance for more than one reason.

The properties **Code** and **Phrase** are expected by the service. When
authoring a custom resource, set the text (typically stdout) you would like to
show as the reason the resource isn't compliant as the value for **Phrase**.
**Code** has specific formatting requirements so reporting can clearly display
information about the resource used to do the audit. This solution makes guest
configuration extensible. Any command could be run as long as the output can be
returned as a string value for the **Phrase** property.

- **Code** (string): The name of the resource, repeated, and then a short name
  with no spaces as an identifier for the reason. These three values should be
  colon-delimited with no spaces.
  - An example would be `registry:registry:keynotpresent`
- **Phrase** (string): Human-readable text to explain why the setting isn't
  compliant.
  - An example would be `The registry key $key isn't present on the machine.`

```powershell
$reasons = @()
$reasons += @{
  Code = 'Name:Name:ReasonIdentifer'
  Phrase = 'Explain why the setting is not compliant'
}
return @{
    reasons = $reasons
}
```

### The Reasons property embedded class

In script-based resources (Windows only), the Reasons class is included in the
schema MOF file as follows.

```mof
[ClassVersion("1.0.0.0")]
class Reason
{
  [Read] String Phrase;
  [Read] String Code;
};

[ClassVersion("1.0.0.0"), FriendlyName("ResourceName")]
class ResourceName : OMI_BaseResource
{
  [Key, Description("Example description")] String Example;
  [Read, EmbeddedInstance("Reason")] String Reasons[];
};
```

In class-based resources (Windows and Linux), the `Reason` class is included in
the PowerShell module as follows. Linux is case-sensitive, so the "C" in Code
and "P" in Phrase must be capitalized.

```powershell
enum ensure {
  Absent
  Present
}

class Reason {
  [DscProperty()]
  [string] $Code

  [DscProperty()]
  [string] $Phrase
}

[DscResource()]
class Example {

  [DscProperty(Key)]
  [ensure] $ensure

  [DscProperty()]
  [Reason[]] $Reasons

  [Example] Get() {
    # return current current state
  }

  [void] Set() {
    # set the state
  }

  [bool] Test() {
    # check whether state is correct
  }
}

```

If the resource has required properties, those properties should also be
returned by `Get` in parallel with the `Reason` class. If `Reason` isn't
included, the service includes a "catch-all" behavior that compares the values
input to `Get` and the values returned by `Get`, and provides a detailed
comparison as `Reason`.

## Configuration names

The name of the custom configuration must be consistent everywhere. The name of
the `.zip` file for the content package, the configuration name in the MOF file,
and the guest assignment name in the Azure Resource Manager template, must be
the same.

## Common DSC features not available during guest configuration public preview

During public preview, guest configuration does not support
[specifying cross-machine dependencies](/powershell/scripting/dsc/configurations/crossnodedependencies)
using "WaitFor*" resources. It isn't possible for one
machine to monitor and wait for another machine to reach a state before
progressing.

[Reboot handling](/powershell/scripting/dsc/configurations/reboot-a-node) isn't
available in the public preview release of guest configuration, including,
the `$global:DSCMachineStatus` isn't available. Configurations aren't able to reboot a node during or at the end of a configuration.

## Known compatibility issues with supported modules

The `PsDscResources` module in the PowerShell Gallery and the `PSDesiredStateConfiguration`
module that ships with Windows are supported by Microsoft and have been a commonly-used
set of resources for DSC. Until the `PSDscResources` module is updated for DSCv3, be aware of the
following known compatibility issues.

- Do not use resources from the `PSDesiredStateConfiguration` module that ships with Windows. Instead,
  switch to `PSDscResources`.
- Do not use the `WindowsFeature` and `WindowsFeatureSet` resources in `PsDscResources`. Instead,
  switch to the `WindowsOptionalFeature` and `WindowsOptionalFeatureSet` resources.

## Coexistance with DSC version 3 and previous versions

DSC version 3 in guest configuration can coexist with older versions installed in
[Windows](/powershell/scripting/dsc/getting-started/wingettingstarted) and
[Linux](/powershell/scripting/dsc/getting-started/lnxgettingstarted).
The implementations are separate. However, there's no conflict detection
across DSC versions, so don't attempt to manage the same settings.

## Next steps

- Read the [guest configuration overview](./guest-configuration.md).
- Setup a custom guest configuration package [development environment](../how-to/guest-configuration-create-setup.md).
- [Create a package artifact](../how-to/guest-configuration-create.md)
  for guest configuration.
- [Test the package artifact](../how-to/guest-configuration-create-test.md)
  from your development environment.
- Use the `GuestConfiguration` module to
  [create an Azure Policy definition](../how-to/guest-configuration-create-definition.md)
  for at-scale management of your environment.
- [Assign your custom policy definition](../assign-policy-portal.md) using
  Azure portal.
- Learn how to view
  [compliance details for guest configuration](../how-to/determine-non-compliance.md#compliance-details-for-guest-configuration) policy assignments.
