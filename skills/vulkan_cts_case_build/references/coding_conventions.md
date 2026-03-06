# Coding Conventions

This reference covers the naming conventions, code style, and patterns used in the VK-GL-CTS Vulkan test suite.

## Table of Contents

1. [File Naming](#1-file-naming)
2. [Namespace Conventions](#2-namespace-conventions)
3. [Class Naming](#3-class-naming)
4. [Test Case Naming](#4-test-case-naming)
5. [Include Guard Format](#5-include-guard-format)
6. [License Header](#6-license-header)
7. [Code Style Rules](#7-code-style-rules)
8. [Common Patterns](#8-common-patterns)

---

## 1. File Naming

All Vulkan CTS test files follow the prefix `vkt` (Vulkan Khronos Test):

| Pattern | Example | Usage |
|---------|---------|-------|
| `vkt<Module>Tests.cpp/hpp` | `vktComputeTests.cpp` | Module entry point |
| `vkt<Module><Feature>Tests.cpp/hpp` | `vktComputeBasicComputeShaderTests.cpp` | Feature-specific tests |
| `vkt<Module>TestsUtil.cpp/hpp` | `vktComputeTestsUtil.cpp` | Module-shared utilities |

- `<Module>` is PascalCase: `Compute`, `Api`, `Image`, `RayTracing`, `MeshShader`
- `<Feature>` is PascalCase: `BasicComputeShader`, `IndirectComputeDispatch`, `ShaderBuiltinVar`

## 2. Namespace Conventions

Tests are organized in a two-level namespace:

```cpp
namespace vkt          // Top-level: all Vulkan CTS tests
{
namespace compute      // Module-level: lowercase
{
namespace              // Anonymous namespace for implementation details
{
// test implementations
} // anonymous namespace
} // namespace compute
} // namespace vkt
```

Module namespace names use lowercase with underscores for multi-word names:

| Module | Namespace | Style |
|--------|-----------|-------|
| `compute` | `vkt::compute` | lowercase (preferred) |
| `api` | `vkt::api` | lowercase (preferred) |
| `ray_tracing` | `vkt::RayTracing` | PascalCase (legacy) |
| `mesh_shader` | `vkt::MeshShader` | PascalCase (legacy) |
| `fragment_ops` | `vkt::FragmentOps` | PascalCase (legacy) |
| `sparse_resources` | `vkt::sparse` | abbreviated (legacy) |
| `dynamic_state` | `vkt::DynamicState` | PascalCase (legacy) |

Note: Namespace naming varies across modules due to historical evolution. For new modules, prefer lowercase with underscores (e.g., `vkt::my_feature`) to match the directory name.

## 3. Class Naming

Classes use PascalCase and follow these patterns:

| Type | Pattern | Example |
|------|---------|---------|
| Test case | `<Feature>Test` | `SharedVarTest`, `BufferFillTest` |
| Test instance | `<Feature>TestInstance` | `SharedVarTestInstance`, `BufferFillTestInstance` |
| Test group | `<Feature>Tests` (in function names) | `createBasicComputeShaderTests()` |

Member variables use the `m_` prefix:

```cpp
class SharedVarTest : public vkt::TestCase
{
private:
    const tcu::IVec3  m_localSize;
    const tcu::IVec3  m_workSize;
};
```

## 4. Test Case Naming

Test case names registered in the framework use `snake_case`:

```cpp
// Good
addFunctionCase(group, "shared_var_basic", testFunction);
addFunctionCase(group, "buffer_fill_64_elements", testFunction);
group->addChild(new MyTest(testCtx, "atomic_add_shared_memory", params));

// Bad - don't use camelCase or PascalCase for test names
addFunctionCase(group, "sharedVarBasic", testFunction);     // Wrong
addFunctionCase(group, "SharedVarBasic", testFunction);     // Wrong
```

The full test path in dEQP follows this hierarchy:

```
dEQP-VK.<module>.<group>.<subgroup>.<test_name>
```

Example: `dEQP-VK.compute.basic.shared_var_atomic`

## 5. Include Guard Format

Include guards use the all-uppercase underscore-separated pattern:

```cpp
#ifndef _VKT<MODULE><FEATURE>TESTS_HPP
#define _VKT<MODULE><FEATURE>TESTS_HPP

// ... content ...

#endif // _VKT<MODULE><FEATURE>TESTS_HPP
```

Examples:
- `_VKTCOMPUTETESTS_HPP`
- `_VKTCOMPUTEBASICCOMPUTESHADERTESTS_HPP`
- `_VKTAPIDEVICEINITIALIZATION_HPP`

## 6. License Header

Every source file must include the Apache 2.0 license header:

```cpp
/*------------------------------------------------------------------------
 * Vulkan Conformance Tests
 * ------------------------
 *
 * Copyright (c) <year> <author>
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
 *//*!
 * \file
 * \brief <Brief description of the file>
 *//*--------------------------------------------------------------------*/
```

## 7. Code Style Rules

### Indentation and Formatting

- Use tabs for indentation (not spaces) in the upstream project
- Opening braces on the same line for control structures, next line for function definitions
- Align related variable declarations for readability:

```cpp
const DeviceInterface&  vk              = m_context.getDeviceInterface();
const VkDevice          device          = m_context.getDevice();
const VkQueue           queue           = m_context.getUniversalQueue();
const uint32_t          queueFamilyIdx  = m_context.getUniversalQueueFamilyIndex();
Allocator&              allocator       = m_context.getDefaultAllocator();
```

### Vulkan Type Conventions

- Use `vk::` namespace prefix for Vulkan types: `vk::VkDevice`, `vk::VkBuffer`
- Within `using namespace vk;` blocks, you can omit the prefix
- Use `VK_NULL_HANDLE` for null Vulkan handles
- Use `nullptr` for null pointers (not `NULL` or `0`)

### Memory and Resource Management

- Use RAII wrappers: `Unique<VkBuffer>`, `Move<VkDevice>`, `BufferWithMemory`
- Use `de::UniquePtr`, `de::SharedPtr`, `de::MovePtr` for heap allocations
- Prefer `makeXxx()` factory functions over direct `vkCreateXxx()` calls

### Error Checking

- Use `VK_CHECK(result)` macro after Vulkan API calls that return `VkResult`
- The framework's wrapper functions typically handle error checking
- Use `TCU_THROW(NotSupportedError, "message")` for unsupported features
- Use `TCU_FAIL("message")` for internal test errors

## 8. Common Patterns

### Resource Cleanup

RAII handles cleanup automatically — resources are destroyed when wrappers go out of scope:

```cpp
{
    const Unique<VkCommandPool> cmdPool(makeCommandPool(vk, device, queueFamilyIdx));
    const Unique<VkCommandBuffer> cmdBuffer(
        allocateCommandBuffer(vk, device, *cmdPool, VK_COMMAND_BUFFER_LEVEL_PRIMARY));
    // ... use resources ...
} // cmdBuffer and cmdPool destroyed here automatically
```

### Descriptor Set Setup

```cpp
// 1. Layout
const Unique<VkDescriptorSetLayout> layout(
    DescriptorSetLayoutBuilder()
        .addSingleBinding(VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_COMPUTE_BIT)
        .addSingleBinding(VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, VK_SHADER_STAGE_COMPUTE_BIT)
        .build(vk, device));

// 2. Pool
const Unique<VkDescriptorPool> pool(
    DescriptorPoolBuilder()
        .addType(VK_DESCRIPTOR_TYPE_STORAGE_BUFFER)
        .addType(VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER)
        .build(vk, device, VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT, 1u));

// 3. Set
const Unique<VkDescriptorSet> set(makeDescriptorSet(vk, device, *pool, *layout));

// 4. Update
const VkDescriptorBufferInfo storageInfo = makeDescriptorBufferInfo(*storageBuf, 0, storageSize);
const VkDescriptorBufferInfo uniformInfo = makeDescriptorBufferInfo(*uniformBuf, 0, uniformSize);
DescriptorSetUpdateBuilder()
    .writeSingle(*set, DescriptorSetUpdateBuilder::Location::binding(0u),
        VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, &storageInfo)
    .writeSingle(*set, DescriptorSetUpdateBuilder::Location::binding(1u),
        VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER, &uniformInfo)
    .update(vk, device);
```

### Command Buffer Recording

```cpp
const Unique<VkCommandPool> cmdPool(makeCommandPool(vk, device, queueFamilyIdx));
const Unique<VkCommandBuffer> cmdBuffer(
    allocateCommandBuffer(vk, device, *cmdPool, VK_COMMAND_BUFFER_LEVEL_PRIMARY));

beginCommandBuffer(vk, *cmdBuffer);

// ... record commands (binds, dispatches, draws, barriers) ...

endCommandBuffer(vk, *cmdBuffer);
submitCommandsAndWait(vk, device, queue, *cmdBuffer);
```

### Reading Back Results

```cpp
// After GPU work completes
invalidateAlloc(vk, device, buffer.getAllocation());
const uint32_t* data = static_cast<const uint32_t*>(
    buffer.getAllocation().getHostPtr());

// Validate
for (uint32_t i = 0; i < count; ++i)
{
    if (data[i] != expected[i])
        return tcu::TestStatus::fail("Mismatch at index " + de::toString(i));
}
return tcu::TestStatus::pass("Results match");
```
