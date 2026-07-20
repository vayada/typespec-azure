---
title: Agent Base Type
description: Define ARM Agent resources with the experimental Agent base type
llmstxt: true
---

The **Agent base type** provides templates and models for defining Azure AI Foundry
agent resources that conform to the ARM `Agent` base type contract. A base type is an
ARM-managed schema descriptor (`@azureBaseType`) that declares the structured
constraints a resource conforms to.

:::caution
Azure Resource Manager base types are **experimental** and may be subject to breaking
changes. Applying the `Agent` template (or `@azureBaseType`) in a provider namespace
emits the `basetypes-experimental` warning, which you must suppress at the usage site:

```tsp
#suppress "@azure-tools/typespec-azure-resource-manager/basetypes-experimental" "Experimental BaseTypes"
```

:::

All Agent models live in the `Azure.ResourceManager.BaseTypes.Agents` namespace:

```tsp
using Azure.ResourceManager.BaseTypes.Agents;
```

## Deployment models

The Agent base type supports two deployment models that differ in who owns the agent
configuration:

- **Appliance** — the service owns and reports the agent configuration. Definition and
  properties fields are **read-only**. Use the `*Appliance` models.
- **Platform** — the client owns and manages the agent configuration. Fields are
  **writable** (except `baseTypes`, which is always ARM-managed). Use the `*Platform` models.

Pick the model that matches how your resource provider manages agents.

## Defining an agent resource

An agent resource is a `TrackedResource` created with the `Agent<Properties>` template.
Its properties come from `AgentPropertiesAppliance` or `AgentPropertiesPlatform`, which
in turn take a user-defined agent definition constrained to `AgentDefinition`.

The definition templates accept two `boolean` parameters that control optional fields:

| Template parameter      | Effect                                  |
| ----------------------- | --------------------------------------- |
| `HasModelDeploymentRef` | Include a `modelDeploymentRef` property |
| `HasInstructions`       | Include an `instructions` property      |

```tsp
@armProviderNamespace
namespace Microsoft.ContosoAgent;

using Azure.ResourceManager.BaseTypes.Agents;

// Platform deployment: client-managed, writable definition fields.
model ContosoAgentDefinition is AgentDefinitionPlatform<true, true>;

model ContosoAgentProperties is AgentPropertiesPlatform<ContosoAgentDefinition> {
  ...DefaultProvisioningStateProperty;
}

// The Agent template applies @azureBaseType automatically.
#suppress "@azure-tools/typespec-azure-resource-manager/basetypes-experimental" "Experimental BaseTypes"
model ContosoAgent is Agent<ContosoAgentProperties> {
  ...ResourceNameParameter<ContosoAgent>;
}
```

For an appliance-deployed agent, swap in the `*Appliance` models:

```tsp
model ContosoAgentDefinition is AgentDefinitionAppliance<true, true>;

model ContosoAgentProperties is AgentPropertiesAppliance<ContosoAgentDefinition> {
  ...DefaultProvisioningStateProperty;
}
```

## Required child resources

An Agent resource **must** define both a `Conversation` and a `Response` proxy child
resource. Omitting either triggers the
[`arm-agent-base-type-child-resources`](/docs/libraries/azure-resource-manager/rules/arm-agent-base-type-child-resources)
rule.

Use the `AgentConversation` and `AgentResponse` templates, extending
`ConversationProperties` and `ResponseProperties` with any RP-specific fields:

```tsp
model ContosoConversationProperties is ConversationProperties {
  ...DefaultProvisioningStateProperty;
}

model ContosoConversation is AgentConversation<ContosoConversationProperties, ContosoAgent> {
  ...ResourceNameParameter<ContosoConversation>;
}

model ContosoResponseProperties is ResponseProperties {
  ...PreviousResponseProperty;
  ...DefaultProvisioningStateProperty;
}

model ContosoResponse is AgentResponse<ContosoResponseProperties, ContosoAgent> {
  ...ResourceNameParameter<ContosoResponse>;
}
```

## Operations

The agent resource uses the standard [ARM resource operation templates](/docs/howtos/arm/resource-operations/).
In addition, each `Conversation` and `Response` child resource **must** define create,
read, update, and delete lifecycle operations, or the
[`arm-agent-base-type-lifecycle-operations`](/docs/libraries/azure-resource-manager/rules/arm-agent-base-type-lifecycle-operations)
rule will fire.

```tsp
@armResourceOperations
interface ContosoAgents {
  get is ArmResourceRead<ContosoAgent>;
  createOrUpdate is ArmResourceCreateOrReplaceAsync<ContosoAgent>;
  update is ArmTagsPatchSync<ContosoAgent>;
  delete is ArmResourceDeleteWithoutOkAsync<ContosoAgent>;
  listBySubscription is ArmListBySubscription<ContosoAgent>;
  listByResourceGroup is ArmResourceListByParent<ContosoAgent>;
}

@armResourceOperations
interface ContosoConversations {
  get is ArmResourceRead<ContosoConversation>;
  createOrUpdate is ArmResourceCreateOrReplaceAsync<ContosoConversation>;
  update is ArmCustomPatchSync<
    ContosoConversation,
    Azure.ResourceManager.Foundations.ResourceUpdateModel<
      ContosoConversation,
      ContosoConversationProperties
    >
  >;
  delete is ArmResourceDeleteWithoutOkAsync<ContosoConversation>;
  listByAgent is ArmResourceListByParent<ContosoConversation>;
}

@armResourceOperations
interface ContosoResponses {
  get is ArmResourceRead<ContosoResponse>;
  createOrUpdate is ArmResourceCreateOrReplaceAsync<ContosoResponse>;
  update is ArmCustomPatchSync<
    ContosoResponse,
    Azure.ResourceManager.Foundations.ResourceUpdateModel<
      ContosoResponse,
      ContosoResponseProperties
    >
  >;
  delete is ArmResourceDeleteWithoutOkAsync<ContosoResponse>;
  listByAgent is ArmResourceListByParent<ContosoResponse>;
}
```

## Related linting rules

- [`arm-agent-base-type-child-resources`](/docs/libraries/azure-resource-manager/rules/arm-agent-base-type-child-resources) — an Agent must have both a Conversation and a Response child resource.
- [`arm-agent-base-type-lifecycle-operations`](/docs/libraries/azure-resource-manager/rules/arm-agent-base-type-lifecycle-operations) — Conversation and Response children must define create, read, update, and delete operations.
