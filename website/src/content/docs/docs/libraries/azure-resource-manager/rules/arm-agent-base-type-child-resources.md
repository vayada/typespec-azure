---
title: arm-agent-base-type-child-resources
---

```text title="Full name"
@azure-tools/typespec-azure-resource-manager/arm-agent-base-type-child-resources
```

Resources decorated with `@azureBaseType` for the Agent base type must have both a Conversation and a Response child resource.

#### ❌ Incorrect

```tsp
@armProviderNamespace
namespace Microsoft.Contoso;

using Azure.ResourceManager.BaseTypes.Agents;

model MyAgentDefinition is AgentDefinitionPlatform<true, true>;

model MyAgentProperties is AgentPropertiesPlatform<MyAgentDefinition> {
  ...DefaultProvisioningStateProperty;
}

// No Conversation or Response child resources are defined.
#suppress "@azure-tools/typespec-azure-resource-manager/basetypes-experimental" "Experimental BaseTypes"
model MyAgent is Agent<MyAgentProperties> {
  ...ResourceNameParameter<MyAgent>;
}
```

#### ✅ Correct

```tsp
@armProviderNamespace
namespace Microsoft.Contoso;

using Azure.ResourceManager.BaseTypes.Agents;

model MyAgentDefinition is AgentDefinitionPlatform<true, true>;

model MyAgentProperties is AgentPropertiesPlatform<MyAgentDefinition> {
  ...DefaultProvisioningStateProperty;
}

#suppress "@azure-tools/typespec-azure-resource-manager/basetypes-experimental" "Experimental BaseTypes"
model MyAgent is Agent<MyAgentProperties> {
  ...ResourceNameParameter<MyAgent>;
}

model MyConversationProperties is ConversationProperties {
  ...DefaultProvisioningStateProperty;
}

model MyConversation is AgentConversation<MyConversationProperties, MyAgent> {
  ...ResourceNameParameter<MyConversation>;
}

model MyResponseProperties is ResponseProperties {
  ...DefaultProvisioningStateProperty;
}

model MyResponse is AgentResponse<MyResponseProperties, MyAgent> {
  ...ResourceNameParameter<MyResponse>;
}
```
