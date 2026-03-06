---
name: vulkan_cts_case_build
description: Create Vulkan CTS (Conformance Test Suite) test cases following VK-GL-CTS project conventions. Use whenever the user wants to create, add, or modify Vulkan CTS test cases, test modules, or conformance tests. Triggers on mentions of CTS, deqp-vk, VK-GL-CTS, Vulkan conformance, or testing Vulkan API features (compute, rendering, synchronization, memory, ray tracing, etc.).
---

# Vulkan CTS Case Build

A skill for creating Vulkan Conformance Test Suite (CTS) test cases that follow the VK-GL-CTS project structure and coding conventions.

The Vulkan CTS is part of the [VK-GL-CTS](https://github.com/KhronosGroup/VK-GL-CTS) project maintained by the Khronos Group. Test cases are located under `external/vulkancts/modules/vulkan/` and are organized into test modules by Vulkan feature area.

## When to Use

- Creating a new Vulkan CTS test case from scratch
- Adding tests to an existing VK-GL-CTS test module
- Creating a new test module for a Vulkan feature area
- Writing compute, graphics, or API-level conformance tests

## When Not to Use

- Non-Vulkan test development (for example OpenGL CTS or non-CTS projects)
- General Vulkan application development that is not part of VK-GL-CTS
- CI/build pipeline setup tasks unrelated to adding or modifying test cases

## Quick Start: Creating a Test Case

Follow these steps to create a new Vulkan CTS test case. Read `references/test_case_patterns.md` for detailed code examples and `references/cmake_integration.md` for build system integration.

### Step 1: Decide on the Approach

There are two main approaches for writing test cases:

1. **Function-based tests** — Best for simple tests. Use `addFunctionCase` or `addFunctionCaseWithPrograms` helper functions. The test logic is a standalone function with signature `tcu::TestStatus testFunction(Context& context)`.

2. **Class-based tests** — Best for complex tests with parameters. Create a `TestCase` subclass and a `TestInstance` subclass. The TestCase defines shader programs and creates TestInstance objects; the TestInstance runs the actual test logic.

Choose function-based for simple tests and class-based when you need parameterization or complex setup.

### Step 2: Understand the File Structure

Each test module lives in its own subdirectory. A typical module has these files:

```
external/vulkancts/modules/vulkan/<module_name>/
├── CMakeLists.txt                          # Build configuration
├── vkt<Module>Tests.cpp                    # Module entry point, creates test groups
├── vkt<Module>Tests.hpp                    # Module entry point header
├── vkt<Module><Feature>Tests.cpp           # Feature-specific test implementations
├── vkt<Module><Feature>Tests.hpp           # Feature-specific test headers
└── vkt<Module>TestsUtil.cpp/hpp (optional) # Shared utilities
```

**Module** is the PascalCase name of the module (e.g., `Compute`, `Api`, `Image`).

### Step 3: Write the Test Code

Every test case follows this lifecycle:

1. **`checkSupport(Context&)`** — Verify device capabilities and extensions. Call `context.requireDeviceFunctionality("VK_EXT_something")` or check feature flags. Throw `tcu::NotSupportedError` if unsupported.

2. **`initPrograms(vk::SourceCollections&)`** — Add shader source code (GLSL or SPIR-V). This runs during the test framework's program setup stage before test execution.

3. **`createInstance(Context&)` / `iterate()`** — The actual test logic. Create Vulkan resources, record command buffers, submit work, and validate results. Return `tcu::TestStatus::pass()` or `tcu::TestStatus::fail()`.

### Step 4: Register the Test

Tests are registered in a test group's `createChildren` function. Use the appropriate helper:

```cpp
// Function-based (no shaders)
addFunctionCase(group, "test_name", testFunction);

// Function-based (with shaders)
addFunctionCaseWithPrograms(group, "test_name", initPrograms, testFunction);

// Function-based (with support check)
addFunctionCase(group, "test_name", checkSupport, testFunction);

// Class-based
group->addChild(new MyTestCase(testCtx, "test_name", param));
```

### Step 5: Integrate with the Build System

See `references/cmake_integration.md` for detailed steps on:

- Adding files to an existing module's `CMakeLists.txt`
- Creating a new test module from scratch
- Linking new modules into the main build

### Step 6: Verify the Test

After building, run your test with:

```bash
./deqp-vk --deqp-case=dEQP-VK.<module>.<group>.<test_name>
```

Use `--deqp-log-filename=TestResults.qpa` to save results. Inspect the `.qpa` log for pass/fail status and any validation layer messages.

## Key Concepts

### Context Object

`vkt::Context` provides access to the Vulkan device, queues, allocators, and binary collection. Every test receives it:

- `context.getDeviceInterface()` — the `vk::DeviceInterface` for calling Vulkan functions
- `context.getDevice()` — `VkDevice`
- `context.getUniversalQueue()` — `VkQueue`
- `context.getUniversalQueueFamilyIndex()` — queue family index
- `context.getDefaultAllocator()` — memory allocator
- `context.getBinaryCollection()` — precompiled shader binaries
- `context.requireDeviceFunctionality(ext)` — throws if extension is not available

### Common Vulkan Helpers

The framework provides many utility functions in the `vk` namespace to simplify Vulkan operations:

- `makeBufferCreateInfo`, `makeImageCreateInfo` — create info helpers
- `BufferWithMemory`, `ImageWithMemory` — RAII wrappers for buffer/image + memory
- `makeCommandPool`, `allocateCommandBuffer` — command buffer setup
- `beginCommandBuffer`, `endCommandBuffer`, `submitCommandsAndWait` — command buffer lifecycle
- `DescriptorSetLayoutBuilder`, `DescriptorPoolBuilder`, `DescriptorSetUpdateBuilder` — descriptor set helpers
- `ComputePipelineWrapper` — compute pipeline abstraction
- `invalidateAlloc` — invalidate host memory before reading back results

### Test Return Values

- `tcu::TestStatus::pass("message")` — test passed
- `tcu::TestStatus::fail("message")` — test failed
- `tcu::TestStatus::incomplete()` — test did not finish
- Throw `tcu::NotSupportedError` in `checkSupport()` to skip

### Shader Programs

Add shaders in `initPrograms()`:

```cpp
void initPrograms(vk::SourceCollections& dst) const
{
    // GLSL compute shader
    dst.glslSources.add("comp") << glu::ComputeSource(
        "#version 310 es\n"
        "layout(local_size_x = 64) in;\n"
        "layout(binding = 0) buffer Data { uint values[]; } data;\n"
        "void main() {\n"
        "    data.values[gl_GlobalInvocationID.x] = gl_GlobalInvocationID.x;\n"
        "}\n");

    // GLSL vertex shader
    dst.glslSources.add("vert") << glu::VertexSource(vertexShaderSource);

    // GLSL fragment shader
    dst.glslSources.add("frag") << glu::FragmentSource(fragmentShaderSource);
}
```

## Naming Conventions

See `references/coding_conventions.md` for the full guide. Key rules:

- **Files**: `vkt<Module><Feature>Tests.cpp` — PascalCase module, PascalCase feature
- **Namespaces**: `vkt::<module>` — lowercase module name (e.g., `vkt::compute`)
- **Test names**: `snake_case` — descriptive, lowercase with underscores (e.g., `shared_var_atomic`)
- **Classes**: `PascalCase` — e.g., `SharedVarTest`, `SharedVarTestInstance`
- **Include guards**: `_VKT<MODULE><FEATURE>TESTS_HPP` — all uppercase

## Existing Test Modules

The VK-GL-CTS project organizes tests into these modules (each a subdirectory under `vulkan/`):

| Module | Area |
|--------|------|
| `api` | Core API (device creation, object lifecycle, format properties) |
| `pipeline` | Graphics and compute pipeline management |
| `binding_model` | Descriptor sets, push constants, bindings |
| `compute` | Compute shaders (dispatch, shared memory, built-in vars) |
| `draw` | Drawing commands (indexed, indirect, multi-draw) |
| `image` | Image operations (views, copies, blits, resolves) |
| `memory` | Memory allocation, mapping, device memory |
| `synchronization` | Barriers, semaphores, fences, events |
| `renderpass` | Render passes, subpasses, attachments |
| `fragment_ops` | Depth/stencil, blending, scissor, multisample |
| `rasterization` | Rasterization rules and precision |
| `ray_tracing` | Ray tracing pipeline and acceleration structures |
| `spirv_assembly` | SPIR-V assembly level tests |
| `subgroups` | Subgroup operations |
| `tessellation` | Tessellation shaders |
| `geometry` | Geometry shaders |
| `wsi` | Window system integration |
| `sparse_resources` | Sparse memory binding |
| `robustness` | Buffer/image access robustness |
| `dynamic_state` | Dynamic pipeline state |
| `query_pool` | Occlusion, timestamp, pipeline stats queries |
| `mesh_shader` | Mesh and task shaders |
| `video` | Video decode/encode |
| `transform_feedback` | Transform feedback |
| `descriptor_indexing` | Non-uniform descriptor indexing |

## Reference Documents

For detailed patterns and examples, read these references as needed:

- **`references/test_case_patterns.md`** — Complete code examples for function-based and class-based tests, including compute shader tests, graphics pipeline tests, and parameterized test patterns
- **`references/cmake_integration.md`** — Step-by-step guide for CMake build system integration, including adding to existing modules and creating new modules
- **`references/coding_conventions.md`** — Full naming conventions, code style rules, header guard format, license header template, and common patterns
