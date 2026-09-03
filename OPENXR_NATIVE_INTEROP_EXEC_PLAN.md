# Expose Vulkan resources needed by OpenXR hosts

This ExecPlan is a living document. The sections Progress, Surprises and Discoveries, Decision Log, and Outcomes and Retrospective must be kept up to date as work proceeds.

This plan follows `.agents/PLANS.md` from the repository root.

## Purpose / Big Picture

After this change, a Python OpenXR host can use a SlangPy-created Vulkan device directly: it can wrap runtime-owned `VkImage` swapchain images without taking ownership, and it can obtain the graphics `VkQueue` plus the family and queue indices required by `XrGraphicsBindingVulkanKHR`. The behavior is visible through two new `Device` methods and focused device tests.

## Progress

- [x] (2026-09-03) Confirmed slang-rhi already implements non-owning `VkImage` wrapping through `IDevice::createTextureFromNativeHandle`.
- [x] (2026-09-03) Confirmed SlangPy creates one Vulkan graphics/compute queue at index zero in the first compatible family.
- [x] (2026-09-03) Added the two SGL and Python binding APIs.
- [x] (2026-09-03) Added Vulkan device tests for handle identity, non-ownership, and queue metadata.
- [x] (2026-09-03) Configured and built the Windows release preset, then ran the focused device tests (30 passed).
- [x] (2026-09-03) Ran all pre-commit hooks successfully with an isolated current pre-commit executable.

## Surprises and Discoveries

- Observation: No slang-rhi change is required for texture ownership.
  Evidence: `external/slang-rhi/src/vulkan/vk-texture.cpp` sets `m_shouldDestroyImage = false` in `createTextureFromNativeHandle`.

- Observation: The public slang-rhi command queue interface exposes `VkQueue` but not its family index.
  Evidence: `ICommandQueue::getNativeHandle` is public, while the selected family is stored only in Vulkan backend internals.

- Observation: The focused Vulkan tests run successfully on the development host rather than taking their no-adapter skip path.
  Evidence: `python -m pytest slangpy/tests/device/test_device.py -q` reported 30 passed.

## Decision Log

- Decision: Keep the texture API Vulkan-only and accept the raw integer `VkImage` plus a normal `TextureDesc`.
  Rationale: This matches the OpenXR Python binding and prevents silently accepting a native handle whose API cannot be validated.
  Date/Author: 2026-09-03 / Codex

- Decision: Determine the family with the same first-graphics-and-compute-family rule used by the pinned slang-rhi, and report queue index zero.
  Rationale: The pinned RHI creates exactly one queue with that rule, while its public ABI does not expose family metadata. Repeating the deterministic selection avoids modifying and repinning a second repository.
  Date/Author: 2026-09-03 / Codex

## Outcomes and Retrospective

The public device surface now supplies both pieces required by an OpenXR Vulkan
host. The implementation stays local to SlangPy/SGL: no slang-rhi pin changed,
and the ownership test proves that releasing an imported wrapper leaves the
original Vulkan texture usable. The Windows release build and all 30 focused
device tests pass, and every repository pre-commit hook completes successfully.

## Context and Orientation

`src/sgl/device/device.h` and `src/sgl/device/device.cpp` define the C++ `Device` wrapper around slang-rhi. `src/slangpy_ext/device/device.cpp` publishes that wrapper through nanobind. `slangpy/tests/device/test_device_api.py` contains public device API tests. A native Vulkan image is a `VkImage` supplied by another owner, in this case OpenXR; closing the SlangPy texture must release only wrapper state and image views, not call `vkDestroyImage` for that image.

The command queue data consists of the native `VkQueue`, a queue-family index describing the hardware queue family, and queue index zero within that family. OpenXR requires all three when creating a Vulkan session.

## Plan of Work

Add a small `NativeCommandQueueInfo` value type and `Device::get_native_command_queue_info()` in `src/sgl/device/device.h`. Implement it in `src/sgl/device/device.cpp` by validating that the device is Vulkan, obtaining its native handles, loading the Vulkan loader through the existing platform shared-library helpers, and enumerating queue families with the same graphics-plus-compute selection used by slang-rhi. Add `Device::create_texture_from_native_handle(uint64_t, TextureDesc)` and delegate to slang-rhi's non-owning import method.

Bind the value type and both methods in `src/slangpy_ext/device/device.cpp`. Add focused Vulkan tests which compare native image handles before and after wrapping, close the wrapper while leaving the source image usable, and verify that queue metadata contains a `VkQueue` and nonnegative indices.

## Concrete Steps

From `E:\OmniSurg\slangpy-openxr`, apply the source edits, then run:

    cmake --preset windows-msvc
    cmake --build --preset windows-msvc-release
    python -m pytest slangpy/tests/device/test_device.py -q
    pre-commit run --all-files

The focused tests should pass and report Vulkan skips only if no Vulkan adapter is available.

## Validation and Acceptance

Acceptance requires a successful SlangPy build, a test proving the wrapped texture exposes the same `VkImage`, continued use of the original texture after the wrapper is closed, and queue info whose handle type is `VkQueue`. Existing device API tests must remain green. Pre-commit must complete without unresolved formatting or lint failures.

## Idempotence and Recovery

All edits are additive and builds are repeatable. If CMake configuration is stale, rerun the preset with `--fresh`; do not delete source or unrelated build directories. If Vulkan is unavailable, retain the skip and rely on CI or the OmniSurg Windows Vulkan hardware run for hardware coverage.

## Artifacts and Notes

The important existing ownership line is:

    texture->m_shouldDestroyImage = false;

The selected queue rule in the pinned RHI is the first family whose flags contain both graphics and compute, with queue index zero.

## Interfaces and Dependencies

The final Python interfaces are `Device.create_texture_from_native_handle(handle, desc) -> Texture` and `Device.get_native_command_queue_info() -> NativeCommandQueueInfo`. The latter exposes `handle`, `family_index`, and `queue_index`. No new dependency is added; the implementation uses the platform loader and Vulkan headers already linked into SGL.

Revision note (2026-09-03): Initial plan created after verifying the pinned slang-rhi ownership and queue-selection behavior.

Revision note (2026-09-03): Recorded the completed implementation, successful
Windows release build, and 30 passing focused device tests before pre-commit.

Revision note (2026-09-03): Recorded the successful final pre-commit run and
post-format rebuild/test pass.
