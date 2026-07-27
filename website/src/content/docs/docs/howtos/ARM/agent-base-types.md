---
title: Agent Base Types
description: Defining ARM Agent resources with the experimental BaseTypes templates
---

The `Azure.ResourceManager.BaseTypes.Agents` namespace provides templates for defining
ARM **Agent** resources and their required child resources. An Agent base type resource
models an AI agent managed through ARM, together with the `Conversation` and `Response`
child resources that hold the interactions with that agent.

:::caution
Agent base types are **experimental**. Using the `Agent<>` template requires suppressing
the `@azure-tools/typespec-azure-resource-manager/basetypes-experimental` warning, and the
shape of these templates may change in future releases.
:::

## Deployment models

Every part of an agent comes in two variants, matching the two ways an Agent resource
can be deployed:

- **Appliance** — the service owns and reports the agent configuration. The definition and
  agent properties are **read-only**; clients cannot modify them directly.
- **Platform** — the client owns and manages the agent configuration. Those properties are
  **writable** (except `baseTypes`, which is always ARM-managed and read-only).

Pick the variant that matches how your resource provider is deployed and use it consistently
for the definition, properties, and resource models.

## Agent definition

`AgentDefinitionAppliance` and `AgentDefinitionPlatform` describe the model and behavior of
the agent. Two boolean template parameters control the optional definition fields:

- `HasModelDeploymentRef` — include a `modelDeploymentRef` property.
- `HasInstructions` — include an `instructions` property.

```typespec
// Appliance definition: read-only fields managed by the service
model ContosoApplianceDefinition is AgentDefinitionAppliance<true, true>;

// Platform definition: writable fields managed by the client
model ContosoPlatformDefinition is AgentDefinitionPlatform<true, true>;
```

## Agent properties

`AgentPropertiesAppliance` and `AgentPropertiesPlatform` are the resource property bags. Each
takes the user-defined agent definition model as its template parameter. Add the standard
provisioning state property to the bag:

```typespec
// Appliance agent properties: service-managed, read-only
model ContosoApplianceAgentProperties is AgentPropertiesAppliance<ContosoApplianceDefinition> {
  ...DefaultProvisioningStateProperty;
}

// Platform agent properties: client-managed, writable
model ContosoPlatformAgentProperties is AgentPropertiesPlatform<ContosoPlatformDefinition> {
  ...DefaultProvisioningStateProperty;
}
```

## Agent resource

The `Agent<>` template creates an ARM `TrackedResource` already marked with the Agent base
type. As with any ARM resource, add a `ResourceNameParameter` spread for the key and segment.

```typespec
#suppress "@azure-tools/typespec-azure-resource-manager/basetypes-experimental" "Experimental BaseTypes"
model ContosoApplianceAgent is Agent<ContosoApplianceAgentProperties> {
  ...ResourceNameParameter<ContosoApplianceAgent>;
}
```

## Conversation and Response child resources

An Agent resource **must** have both a `Conversation` and a `Response` proxy child resource.
Omitting either triggers the
[`arm-agent-base-type-child-resources`](https://azure.github.io/typespec-azure/docs/libraries/azure-resource-manager/reference/linter)
rule. Build the property bags from `ConversationProperties` and `ResponseProperties`, then
define the child resources with `AgentConversation<>` and `AgentResponse<>`, passing the
parent agent as the second template parameter:

```typespec
model ContosoConversationProperties is ConversationProperties {
  ...DefaultProvisioningStateProperty;
}

model ContosoResponseProperties is ResponseProperties {
  ...PreviousResponseProperty;
  ...DefaultProvisioningStateProperty;
}

model ApplianceConversation
  is AgentConversation<ContosoConversationProperties, ContosoApplianceAgent> {
  ...ResourceNameParameter<ApplianceConversation>;
}

model ApplianceResponse
  is AgentResponse<ContosoResponseProperties, ContosoApplianceAgent> {
  ...ResourceNameParameter<ApplianceResponse>;
}
```

## Required operations

The `Conversation` and `Response` child resources must each define **create, read, update,
and delete** lifecycle operations. Omitting any of them triggers the
[`arm-agent-base-type-lifecycle-operations`](https://azure.github.io/typespec-azure/docs/libraries/azure-resource-manager/reference/linter)
rule.

```typespec
@armResourceOperations
interface ApplianceConversations {
  get is ArmResourceRead<ApplianceConversation>;
  createOrUpdate is ArmResourceCreateOrReplaceAsync<ApplianceConversation>;
  update is ArmCustomPatchSync<
    ApplianceConversation,
    Azure.ResourceManager.Foundations.ResourceUpdateModel<
      ApplianceConversation,
      ContosoConversationProperties
    >
  >;
  delete is ArmResourceDeleteWithoutOkAsync<ApplianceConversation>;
  listByAgent is ArmResourceListByParent<ApplianceConversation>;
}
```

Define the same set of operations for the `Response` child resource. The Agent resource
itself uses the standard ARM resource operation templates (see
[ARM Resource Operations](./resource-operations.md)).
