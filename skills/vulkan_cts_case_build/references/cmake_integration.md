# CMake Integration

This reference explains how to integrate Vulkan CTS test cases into the CMake build system used by VK-GL-CTS.

## Table of Contents

1. [Adding Files to an Existing Module](#1-adding-files-to-an-existing-module)
2. [Creating a New Test Module](#2-creating-a-new-test-module)
3. [Registering a New Module in the Main Build](#3-registering-a-new-module-in-the-main-build)
4. [Build and Run](#4-build-and-run)

---

## 1. Adding Files to an Existing Module

To add new test files to an existing module (e.g., `compute/`), edit the module's `CMakeLists.txt`.

**Example**: Adding `vktComputeMyNewTests.cpp` and `vktComputeMyNewTests.hpp` to the compute module.

Edit `external/vulkancts/modules/vulkan/compute/CMakeLists.txt`:

```cmake
set(DEQP_VK_VKSC_COMPUTE_SRCS
    vktComputeTests.cpp
    vktComputeTests.hpp
    vktComputeBasicComputeShaderTests.cpp
    vktComputeBasicComputeShaderTests.hpp
    # ... existing files ...
    vktComputeMyNewTests.cpp          # <-- Add your new files
    vktComputeMyNewTests.hpp
)
```

Then include the new test group in the module's entry point (`vktComputeTests.cpp`):

```cpp
#include "vktComputeMyNewTests.hpp"

void createChildren(tcu::TestCaseGroup* computeTests, ...)
{
    tcu::TestContext& testCtx = computeTests->getTestContext();
    // ... existing children ...
    computeTests->addChild(createMyNewTests(testCtx, computePipelineConstructionType));
}
```

## 2. Creating a New Test Module

To create a completely new test module (e.g., `my_feature/`):

### Step 2a: Create the Directory

```
external/vulkancts/modules/vulkan/my_feature/
```

### Step 2b: Create CMakeLists.txt

```cmake
include_directories(
    ..
    ../amber
    ${DEQP_INL_DIR}
)

set(DEQP_VK_VKSC_MY_FEATURE_SRCS
    vktMyFeatureTests.cpp
    vktMyFeatureTests.hpp
    vktMyFeatureBasicTests.cpp
    vktMyFeatureBasicTests.hpp
)

set(DEQP_VK_MY_FEATURE_SRCS
    # Vulkan-only sources (not for VulkanSC)
)

PCH(DEQP_VK_MY_FEATURE_SRCS ../pch.cpp)

add_library(deqp-vk-my-feature STATIC
    ${DEQP_VK_VKSC_MY_FEATURE_SRCS}
    ${DEQP_VK_MY_FEATURE_SRCS})
target_link_libraries(deqp-vk-my-feature tcutil vkutil)

# Optional: VulkanSC variant
# add_library(deqp-vksc-my-feature STATIC ${DEQP_VK_VKSC_MY_FEATURE_SRCS})
# target_link_libraries(deqp-vksc-my-feature PUBLIC deqp-vksc-util tcutil vkscutil)
```

### Step 2c: Create Module Entry Point

**`vktMyFeatureTests.hpp`**:

```cpp
#ifndef _VKTMYFEATURETESTS_HPP
#define _VKTMYFEATURETESTS_HPP

#include "tcuDefs.hpp"
#include "tcuTestCase.hpp"

namespace vkt
{
namespace my_feature
{

tcu::TestCaseGroup* createTests(tcu::TestContext& testCtx, const std::string& name);

} // namespace my_feature
} // namespace vkt

#endif // _VKTMYFEATURETESTS_HPP
```

**`vktMyFeatureTests.cpp`**:

```cpp
#include "vktMyFeatureTests.hpp"
#include "vktMyFeatureBasicTests.hpp"
#include "vktTestGroupUtil.hpp"

namespace vkt
{
namespace my_feature
{

namespace
{

void createChildren(tcu::TestCaseGroup* group)
{
    tcu::TestContext& testCtx = group->getTestContext();
    group->addChild(createBasicTests(testCtx));
}

} // anonymous namespace

tcu::TestCaseGroup* createTests(tcu::TestContext& testCtx, const std::string& name)
{
    return createTestGroup(testCtx, name, createChildren);
}

} // namespace my_feature
} // namespace vkt
```

## 3. Registering a New Module in the Main Build

After creating the module, register it in three places within the parent `CMakeLists.txt` at `external/vulkancts/modules/vulkan/CMakeLists.txt`:

### 3a: Add Subdirectory

```cmake
add_subdirectory(my_feature)
```

### 3b: Add to Include Directories

```cmake
include_directories(
    # ... existing modules ...
    my_feature
)
```

### 3c: Add to Library List

```cmake
set(DEQP_VK_LIBS
    # ... existing libraries ...
    deqp-vk-my-feature
)
```

### 3d: Register in Test Package

Edit `vktTestPackage.cpp` to include and register the new module:

```cpp
#include "vktMyFeatureTests.hpp"

// In createChildren():
addChild(my_feature::createTests(m_testCtx, "my_feature"));
```

## 4. Build and Run

### Building

```bash
# From the VK-GL-CTS root directory
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc) deqp-vk
```

### Running Tests

```bash
# Run all tests in your module
./external/vulkancts/modules/vulkan/deqp-vk \
    --deqp-case=dEQP-VK.my_feature.*

# Run a specific test
./external/vulkancts/modules/vulkan/deqp-vk \
    --deqp-case=dEQP-VK.my_feature.basic.buffer_fill_64

# Run with detailed logging
./external/vulkancts/modules/vulkan/deqp-vk \
    --deqp-case=dEQP-VK.my_feature.* \
    --deqp-log-filename=TestResults.qpa \
    --deqp-log-images=enable \
    --deqp-log-shader-sources=enable

# List all test cases without running them
./external/vulkancts/modules/vulkan/deqp-vk \
    --deqp-runmode=txt-caselist \
    --deqp-case=dEQP-VK.my_feature.*
```

### Useful Build Flags

| Flag | Description |
|------|-------------|
| `-DCMAKE_BUILD_TYPE=Debug` | Debug build with assertions |
| `-DCMAKE_BUILD_TYPE=Release` | Optimized build |
| `-DDEQP_TARGET=default` | Default platform target |
| `-DDEQP_TARGET=x11_egl` | X11/EGL target on Linux |
