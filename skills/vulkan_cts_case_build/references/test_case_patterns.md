# Test Case Patterns

This reference provides complete code examples for creating Vulkan CTS test cases following the VK-GL-CTS project conventions.

## Table of Contents

1. [Function-Based Test (No Shaders)](#1-function-based-test-no-shaders)
2. [Function-Based Test (With Shaders)](#2-function-based-test-with-shaders)
3. [Class-Based Test (Compute Shader)](#3-class-based-test-compute-shader)
4. [Class-Based Test (Parameterized)](#4-class-based-test-parameterized)
5. [Test Group Registration](#5-test-group-registration)
6. [Module Entry Point](#6-module-entry-point)
7. [Support Check Patterns](#7-support-check-patterns)

---

## 1. Function-Based Test (No Shaders)

Use for simple API-level tests that don't need shader programs. Ideal for testing device properties, format queries, object creation, etc.

```cpp
#include "vktTestCase.hpp"
#include "vktTestCaseUtil.hpp"
#include "vktTestGroupUtil.hpp"

#include "vkQueryUtil.hpp"
#include "vkTypeUtil.hpp"

using namespace vk;

namespace vkt
{
namespace api
{
namespace
{

tcu::TestStatus testDeviceProperties(Context& context)
{
    const InstanceInterface&     vki            = context.getInstanceInterface();
    const VkPhysicalDevice       physicalDevice = context.getPhysicalDevice();
    VkPhysicalDeviceProperties   properties;

    vki.getPhysicalDeviceProperties(physicalDevice, &properties);

    if (properties.limits.maxComputeWorkGroupCount[0] < 65535)
        return tcu::TestStatus::fail("maxComputeWorkGroupCount[0] below minimum");

    return tcu::TestStatus::pass("Device properties within spec limits");
}

void createTests(tcu::TestCaseGroup* group)
{
    addFunctionCase(group, "device_properties", testDeviceProperties);
}

} // anonymous namespace

tcu::TestCaseGroup* createDevicePropertyTests(tcu::TestContext& testCtx)
{
    return createTestGroup(testCtx, "device_properties", createTests);
}

} // namespace api
} // namespace vkt
```

## 2. Function-Based Test (With Shaders)

Use when you need shader programs but the test logic is relatively simple.

```cpp
#include "vktTestCase.hpp"
#include "vktTestCaseUtil.hpp"
#include "vktTestGroupUtil.hpp"

#include "vkDefs.hpp"
#include "vkRef.hpp"
#include "vkRefUtil.hpp"
#include "vkPrograms.hpp"
#include "vkMemUtil.hpp"
#include "vkBuilderUtil.hpp"
#include "vkCmdUtil.hpp"
#include "vkObjUtil.hpp"
#include "vkBufferWithMemory.hpp"

#include "tcuTestLog.hpp"

using namespace vk;

namespace vkt
{
namespace compute
{
namespace
{

void initPrograms(vk::SourceCollections& programCollection)
{
    programCollection.glslSources.add("comp") << glu::ComputeSource(
        "#version 310 es\n"
        "layout(local_size_x = 64) in;\n"
        "layout(binding = 0) buffer Output {\n"
        "    uint values[];\n"
        "} sb_out;\n"
        "void main() {\n"
        "    uint idx = gl_GlobalInvocationID.x;\n"
        "    sb_out.values[idx] = idx * 2u;\n"
        "}\n");
}

tcu::TestStatus testSimpleCompute(Context& context)
{
    const DeviceInterface&  vk              = context.getDeviceInterface();
    const VkDevice          device          = context.getDevice();
    const VkQueue           queue           = context.getUniversalQueue();
    const uint32_t          queueFamilyIdx  = context.getUniversalQueueFamilyIndex();
    Allocator&              allocator       = context.getDefaultAllocator();

    const uint32_t          numElements     = 64u;
    const VkDeviceSize      bufferSize      = sizeof(uint32_t) * numElements;

    // Create output buffer
    const BufferWithMemory outputBuffer(vk, device, allocator,
        makeBufferCreateInfo(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT),
        MemoryRequirement::HostVisible);

    // Create descriptor set
    const Unique<VkDescriptorSetLayout> descriptorSetLayout(
        DescriptorSetLayoutBuilder()
            .addSingleBinding(VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, VK_SHADER_STAGE_COMPUTE_BIT)
            .build(vk, device));

    const Unique<VkDescriptorPool> descriptorPool(
        DescriptorPoolBuilder()
            .addType(VK_DESCRIPTOR_TYPE_STORAGE_BUFFER)
            .build(vk, device, VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT, 1u));

    const Unique<VkDescriptorSet> descriptorSet(
        makeDescriptorSet(vk, device, *descriptorPool, *descriptorSetLayout));

    const VkDescriptorBufferInfo descriptorInfo =
        makeDescriptorBufferInfo(*outputBuffer, 0ull, bufferSize);

    DescriptorSetUpdateBuilder()
        .writeSingle(*descriptorSet,
            DescriptorSetUpdateBuilder::Location::binding(0u),
            VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, &descriptorInfo)
        .update(vk, device);

    // Create compute pipeline
    ComputePipelineWrapper pipeline(vk, device,
        COMPUTE_PIPELINE_CONSTRUCTION_TYPE_PIPELINE,
        context.getBinaryCollection().get("comp"));
    pipeline.setDescriptorSetLayout(descriptorSetLayout.get());
    pipeline.buildPipeline();

    // Record and submit command buffer
    const Unique<VkCommandPool> cmdPool(makeCommandPool(vk, device, queueFamilyIdx));
    const Unique<VkCommandBuffer> cmdBuffer(
        allocateCommandBuffer(vk, device, *cmdPool, VK_COMMAND_BUFFER_LEVEL_PRIMARY));

    beginCommandBuffer(vk, *cmdBuffer);
    pipeline.bind(*cmdBuffer);
    vk.cmdBindDescriptorSets(*cmdBuffer, VK_PIPELINE_BIND_POINT_COMPUTE,
        pipeline.getPipelineLayout(), 0u, 1u, &descriptorSet.get(), 0u, nullptr);
    vk.cmdDispatch(*cmdBuffer, 1u, 1u, 1u);

    const VkBufferMemoryBarrier barrier = makeBufferMemoryBarrier(
        VK_ACCESS_SHADER_WRITE_BIT, VK_ACCESS_HOST_READ_BIT,
        *outputBuffer, 0ull, bufferSize);
    vk.cmdPipelineBarrier(*cmdBuffer, VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT,
        VK_PIPELINE_STAGE_HOST_BIT, 0, 0, nullptr, 1, &barrier, 0, nullptr);

    endCommandBuffer(vk, *cmdBuffer);
    submitCommandsAndWait(vk, device, queue, *cmdBuffer);

    // Validate results
    invalidateAlloc(vk, device, outputBuffer.getAllocation());
    const uint32_t* result =
        static_cast<const uint32_t*>(outputBuffer.getAllocation().getHostPtr());

    for (uint32_t i = 0; i < numElements; ++i)
    {
        if (result[i] != i * 2u)
        {
            return tcu::TestStatus::fail("Mismatch at index " + de::toString(i));
        }
    }

    return tcu::TestStatus::pass("Compute succeeded");
}

void createTests(tcu::TestCaseGroup* group)
{
    addFunctionCaseWithPrograms(group, "simple_compute", initPrograms, testSimpleCompute);
}

} // anonymous namespace
} // namespace compute
} // namespace vkt
```

## 3. Class-Based Test (Compute Shader)

Use for tests that need parameterization or complex state management.

```cpp
#include "vktTestCase.hpp"
#include "vktTestCaseUtil.hpp"

#include "vkDefs.hpp"
#include "vkRef.hpp"
#include "vkRefUtil.hpp"
#include "vkPrograms.hpp"
#include "vkMemUtil.hpp"
#include "vkBarrierUtil.hpp"
#include "vkBuilderUtil.hpp"
#include "vkCmdUtil.hpp"
#include "vkObjUtil.hpp"
#include "vkBufferWithMemory.hpp"

#include "tcuTestLog.hpp"
#include "deStringUtil.hpp"

using namespace vk;

namespace vkt
{
namespace compute
{
namespace
{

// --- TestCase class: defines the test, provides shaders ---

class BufferFillTest : public vkt::TestCase
{
public:
    BufferFillTest(tcu::TestContext& testCtx,
                   const std::string& name,
                   uint32_t numElements);

    void            initPrograms    (SourceCollections& programCollection) const override;
    TestInstance*    createInstance   (Context& context) const override;
    void            checkSupport     (Context& context) const override;

private:
    const uint32_t  m_numElements;
};

// --- TestInstance class: runs the test logic ---

class BufferFillTestInstance : public vkt::TestInstance
{
public:
    BufferFillTestInstance(Context& context, uint32_t numElements);

    tcu::TestStatus iterate(void) override;

private:
    const uint32_t  m_numElements;
};

// --- TestCase implementation ---

BufferFillTest::BufferFillTest(tcu::TestContext& testCtx,
                               const std::string& name,
                               uint32_t numElements)
    : TestCase(testCtx, name)
    , m_numElements(numElements)
{
}

void BufferFillTest::checkSupport(Context& context) const
{
    // Example: require a minimum buffer size
    const VkPhysicalDeviceLimits& limits =
        context.getDeviceProperties().limits;

    if (limits.maxStorageBufferRange < m_numElements * sizeof(uint32_t))
        TCU_THROW(NotSupportedError, "Storage buffer range too small");
}

void BufferFillTest::initPrograms(SourceCollections& programCollection) const
{
    std::ostringstream src;
    src << "#version 310 es\n"
        << "layout(local_size_x = 64) in;\n"
        << "layout(binding = 0) buffer Output {\n"
        << "    uint values[" << m_numElements << "];\n"
        << "} sb_out;\n"
        << "void main() {\n"
        << "    uint idx = gl_GlobalInvocationID.x;\n"
        << "    if (idx < " << m_numElements << "u)\n"
        << "        sb_out.values[idx] = idx;\n"
        << "}\n";

    programCollection.glslSources.add("comp") << glu::ComputeSource(src.str());
}

TestInstance* BufferFillTest::createInstance(Context& context) const
{
    return new BufferFillTestInstance(context, m_numElements);
}

// --- TestInstance implementation ---

BufferFillTestInstance::BufferFillTestInstance(Context& context, uint32_t numElements)
    : TestInstance(context)
    , m_numElements(numElements)
{
}

tcu::TestStatus BufferFillTestInstance::iterate(void)
{
    const DeviceInterface&  vk              = m_context.getDeviceInterface();
    const VkDevice          device          = m_context.getDevice();
    const VkQueue           queue           = m_context.getUniversalQueue();
    const uint32_t          queueFamilyIdx  = m_context.getUniversalQueueFamilyIndex();
    Allocator&              allocator       = m_context.getDefaultAllocator();

    const VkDeviceSize bufferSize = sizeof(uint32_t) * m_numElements;

    // Create output buffer
    const BufferWithMemory buffer(vk, device, allocator,
        makeBufferCreateInfo(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT),
        MemoryRequirement::HostVisible);

    // Set up descriptor set
    const Unique<VkDescriptorSetLayout> descriptorSetLayout(
        DescriptorSetLayoutBuilder()
            .addSingleBinding(VK_DESCRIPTOR_TYPE_STORAGE_BUFFER,
                              VK_SHADER_STAGE_COMPUTE_BIT)
            .build(vk, device));

    const Unique<VkDescriptorPool> descriptorPool(
        DescriptorPoolBuilder()
            .addType(VK_DESCRIPTOR_TYPE_STORAGE_BUFFER)
            .build(vk, device, VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT, 1u));

    const Unique<VkDescriptorSet> descriptorSet(
        makeDescriptorSet(vk, device, *descriptorPool, *descriptorSetLayout));

    const VkDescriptorBufferInfo descriptorInfo =
        makeDescriptorBufferInfo(*buffer, 0ull, bufferSize);
    DescriptorSetUpdateBuilder()
        .writeSingle(*descriptorSet,
            DescriptorSetUpdateBuilder::Location::binding(0u),
            VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, &descriptorInfo)
        .update(vk, device);

    // Create and dispatch compute pipeline
    ComputePipelineWrapper pipeline(vk, device,
        COMPUTE_PIPELINE_CONSTRUCTION_TYPE_PIPELINE,
        m_context.getBinaryCollection().get("comp"));
    pipeline.setDescriptorSetLayout(descriptorSetLayout.get());
    pipeline.buildPipeline();

    const Unique<VkCommandPool> cmdPool(
        makeCommandPool(vk, device, queueFamilyIdx));
    const Unique<VkCommandBuffer> cmdBuffer(
        allocateCommandBuffer(vk, device, *cmdPool, VK_COMMAND_BUFFER_LEVEL_PRIMARY));

    const uint32_t workGroupCount = (m_numElements + 63u) / 64u;

    beginCommandBuffer(vk, *cmdBuffer);
    pipeline.bind(*cmdBuffer);
    vk.cmdBindDescriptorSets(*cmdBuffer, VK_PIPELINE_BIND_POINT_COMPUTE,
        pipeline.getPipelineLayout(), 0u, 1u, &descriptorSet.get(), 0u, nullptr);
    vk.cmdDispatch(*cmdBuffer, workGroupCount, 1u, 1u);

    const VkBufferMemoryBarrier barrier = makeBufferMemoryBarrier(
        VK_ACCESS_SHADER_WRITE_BIT, VK_ACCESS_HOST_READ_BIT,
        *buffer, 0ull, bufferSize);
    vk.cmdPipelineBarrier(*cmdBuffer,
        VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT, VK_PIPELINE_STAGE_HOST_BIT,
        0, 0, nullptr, 1, &barrier, 0, nullptr);

    endCommandBuffer(vk, *cmdBuffer);
    submitCommandsAndWait(vk, device, queue, *cmdBuffer);

    // Validate results
    invalidateAlloc(vk, device, buffer.getAllocation());
    const uint32_t* resultPtr =
        static_cast<const uint32_t*>(buffer.getAllocation().getHostPtr());

    for (uint32_t i = 0; i < m_numElements; ++i)
    {
        if (resultPtr[i] != i)
        {
            return tcu::TestStatus::fail(
                "Mismatch at index " + de::toString(i) +
                ": expected " + de::toString(i) +
                ", got " + de::toString(resultPtr[i]));
        }
    }

    return tcu::TestStatus::pass("Buffer fill succeeded");
}

} // anonymous namespace
} // namespace compute
} // namespace vkt
```

## 4. Class-Based Test (Parameterized)

Use when you want to generate many tests from a set of parameters (e.g., different buffer sizes, formats, pipeline types).

```cpp
#include "vktTestCase.hpp"
#include "vktTestCaseUtil.hpp"
#include "vktTestGroupUtil.hpp"

#include "vkDefs.hpp"
#include "vkPrograms.hpp"
#include "vkBufferWithMemory.hpp"
#include "vkMemUtil.hpp"
#include "vkCmdUtil.hpp"
#include "vkObjUtil.hpp"
#include "vkBuilderUtil.hpp"
#include "vkBarrierUtil.hpp"

#include "deStringUtil.hpp"
#include "tcuTestLog.hpp"

using namespace vk;

namespace vkt
{
namespace compute
{
namespace
{

struct TestParams
{
    tcu::IVec3  localSize;
    tcu::IVec3  workSize;
    bool        useAtomics;
};

// Derive the test name from parameters
std::string getTestName(const TestParams& params)
{
    std::ostringstream name;
    name << "local_" << params.localSize.x()
         << "x" << params.localSize.y()
         << "x" << params.localSize.z()
         << "_work_" << params.workSize.x()
         << "x" << params.workSize.y()
         << "x" << params.workSize.z();
    if (params.useAtomics)
        name << "_atomics";
    return name.str();
}

// --- Function-based parameterized test ---

void initPrograms(vk::SourceCollections& dst, TestParams params)
{
    std::ostringstream src;
    src << "#version 310 es\n"
        << "layout(local_size_x = " << params.localSize.x()
        << ", local_size_y = " << params.localSize.y()
        << ", local_size_z = " << params.localSize.z() << ") in;\n"
        << "layout(binding = 0) buffer Data { uint values[]; } data;\n"
        << "void main() {\n"
        << "    uint idx = gl_GlobalInvocationID.x;\n"
        << "    data.values[idx] = idx;\n"
        << "}\n";

    dst.glslSources.add("comp") << glu::ComputeSource(src.str());
}

tcu::TestStatus testCompute(Context& context, TestParams params)
{
    // ... test implementation using params ...
    (void)params;
    return tcu::TestStatus::pass("Pass");
}

void createParameterizedTests(tcu::TestCaseGroup* group)
{
    const tcu::IVec3 localSizes[] =
    {
        tcu::IVec3(1, 1, 1),
        tcu::IVec3(64, 1, 1),
        tcu::IVec3(8, 8, 1),
        tcu::IVec3(4, 4, 4),
    };

    const tcu::IVec3 workSizes[] =
    {
        tcu::IVec3(1, 1, 1),
        tcu::IVec3(2, 2, 2),
        tcu::IVec3(4, 4, 4),
    };

    for (const auto& localSize : localSizes)
    {
        for (const auto& workSize : workSizes)
        {
            for (int useAtomics = 0; useAtomics < 2; ++useAtomics)
            {
                TestParams params;
                params.localSize  = localSize;
                params.workSize   = workSize;
                params.useAtomics = (useAtomics != 0);

                addFunctionCaseWithPrograms<TestParams>(
                    group,
                    getTestName(params),
                    initPrograms,
                    testCompute,
                    params);
            }
        }
    }
}

} // anonymous namespace
} // namespace compute
} // namespace vkt
```

## 5. Test Group Registration

Register test groups in the module's `createChildren` function:

```cpp
void createChildren(tcu::TestCaseGroup* moduleGroup)
{
    tcu::TestContext& testCtx = moduleGroup->getTestContext();

    // Add sub-groups for different feature areas
    moduleGroup->addChild(createBasicTests(testCtx));
    moduleGroup->addChild(createAdvancedTests(testCtx));

    // Or add individual tests directly
    addFunctionCase(moduleGroup, "simple_test", simpleTestFunction);
}
```

## 6. Module Entry Point

The module entry point creates the top-level test group. This is what gets included in `vktTestPackage.cpp`.

**Header** (`vkt<Module>Tests.hpp`):

```cpp
#ifndef _VKT<MODULE>TESTS_HPP
#define _VKT<MODULE>TESTS_HPP

#include "tcuDefs.hpp"
#include "tcuTestCase.hpp"

namespace vkt
{
namespace <module>
{

tcu::TestCaseGroup* createTests(tcu::TestContext& testCtx, const std::string& name);

} // namespace <module>
} // namespace vkt

#endif // _VKT<MODULE>TESTS_HPP
```

**Implementation** (`vkt<Module>Tests.cpp`):

```cpp
#include "vkt<Module>Tests.hpp"
#include "vkt<Module>FeatureATests.hpp"
#include "vkt<Module>FeatureBTests.hpp"
#include "vktTestGroupUtil.hpp"

namespace vkt
{
namespace <module>
{

namespace
{

void createChildren(tcu::TestCaseGroup* group)
{
    tcu::TestContext& testCtx = group->getTestContext();

    group->addChild(createFeatureATests(testCtx));
    group->addChild(createFeatureBTests(testCtx));
}

} // anonymous namespace

tcu::TestCaseGroup* createTests(tcu::TestContext& testCtx, const std::string& name)
{
    return createTestGroup(testCtx, name, createChildren);
}

} // namespace <module>
} // namespace vkt
```

## 7. Support Check Patterns

Common patterns for checking device support:

```cpp
void checkSupport(Context& context) const
{
    // Require a specific extension
    context.requireDeviceFunctionality("VK_KHR_storage_buffer_storage_class");

    // Require a specific API version
    if (!context.contextSupports(vk::ApiVersion(0, 1, 1, 0)))
        TCU_THROW(NotSupportedError, "Vulkan 1.1 required");

    // Check a feature flag
    const auto& features = context.getDeviceFeatures();
    if (!features.shaderInt64)
        TCU_THROW(NotSupportedError, "shaderInt64 not supported");

    // Check Vulkan 1.2 features
    const auto& vk12Features = context.getDeviceVulkan12Features();
    if (!vk12Features.bufferDeviceAddress)
        TCU_THROW(NotSupportedError, "bufferDeviceAddress not supported");

    // Check extension-specific features
    if (!context.isDeviceFunctionalitySupported("VK_EXT_mesh_shader"))
        TCU_THROW(NotSupportedError, "VK_EXT_mesh_shader not supported");
}
```
