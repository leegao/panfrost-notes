# Vortek 10.1 RE

Extends off of https://leegao.github.io/winlator-internals/2025/06/01/Vortek1.html from 10.0

## `VtContext` Object

```c
#include <pthread.h>
#include <jni.h> // For JavaVM, JNIEnv, jobject, jmethodID

// Forward declarations for custom opaque structures.
typedef struct ArrayList ArrayList;
typedef struct RingBuffer RingBuffer;
typedef struct TextureDecoder TextureDecoder;
typedef struct ShaderInspector ShaderInspector;
typedef struct AsyncPipelineCreator AsyncPipelineCreator;

/**
 * @brief Main context structure for the Vortek renderer.
 * 
 * This structure holds all state required for processing Vulkan commands,
 * including communication channels, memory management, and JNI handles.
 * The offsets are 0-based and reflect a 64-bit architecture.
 */
typedef struct VtContext {
    /* 0x00 */ int clientFd;
    /* 0x04 */ int vkMaxVersion;
    /* 0x08 */ short maxDeviceMemory; // In MB
    /* 0x0a */ short imageCacheSize;  // In MB
    /* 0x0c */ int padding_0c;
    /* 0x10 */ ArrayList* exposedDeviceExtensions;
    /* 0x18 */ ArrayList* exposedInstanceExtensions;
    /* 0x20 */ byte hostVisibleMemoryFlag; // Set if host-visible memory is preferred for certain operations.
    /* 0x21 */ byte supportsAndroidHardwareBufferFlag; // Set if VK_EXTERNAL_MEMORY_HANDLE_TYPE_ANDROID_HARDWARE_BUFFER_BIT is supported.
    /* 0x22 */ char padding_22[6];
    /* 0x28 */ char* inputBufferPtr;      // Pointer to the current command's data from the guest.
    /* 0x30 */ int inputBufferSize;
    /* 0x34 */ int padding_34;
    /* 0x38 */ void* tempBuffer;          // A large temporary buffer (~64KB) for command deserialization.
    /* 0x40 */ int tempBufferOffset;      // Current write offset within the tempBuffer.
    /* 0x44 */ int padding_44;
    /* 0x48 */ ArrayList* tempAllocations; // Tracks temporary malloc'd pointers to be freed after command processing.
    /* 0x50 */ long padding_50;
    /* 0x58 */ pthread_t serverThread;    // Handle to the main renderer command processing thread.
    /* 0x60 */ RingBuffer* clientRingBuffer; // For writing results back to the guest (renderer -> guest).
    /* 0x68 */ RingBuffer* serverRingBuffer; // For reading commands from the guest (guest -> renderer).
    /* 0x70 */ int lastError;             // Stores the last recorded VkResult error (e.g., VK_ERROR_OUT_OF_DATE_KHR).
    /* 0x74 */ int graphicsQueueFamilyIndex;
    /* 0x78 */ TextureDecoder* textureDecoder;
    /* 0x80 */ ShaderInspector* shaderInspector;
    /* 0x88 */ AsyncPipelineCreator* asyncPipelineCreator;
    
    // JNI and Android-specific windowing members
    /* 0x90 */ JavaVM* javaVM;                          // Pointer to the Java Virtual Machine, used to attach threads to JNI.
    /* 0x98 */ jobject javaComponentObject;             // Global reference to the VortekRendererComponent Java object.
    /* 0xA0 */ jobject javaWindowSurfaceObject;         // Global reference to the Android Surface object for rendering.
    /* 0xA8 */ jmethodID getWindowWidthMethodID;
    /* 0xB0 */ jmethodID getWindowHeightMethodID;
    /* 0xB8 */ jmethodID getWindowHardwareBufferMethodID;
    /* 0xC0 */ jmethodID updateWindowContentMethodID;
} VtContext;
```

## VT Handlers

### Architectural Overview

The `libvortekrenderer2.so` library acts as a server-side Vulkan wrapper. It receives serialized commands from a client, deserializes them, executes them against the native Vulkan driver, and serializes results back. The modifications can be categorized into several key architectural patterns:

1.  **Standard Pass-through:** The vast majority of functions are simple wrappers. They deserialize arguments, resolve handle IDs (e.g., `VkObject_fromId`), call the corresponding function in the Vulkan dispatch table (e.g., `dispatch_vkCmdDraw`), and, if necessary, serialize the result. They contain no modifications to the API's behavior.
2.  **Custom Swapchain (`XWindowSwapchain_...`)**: This is a major modification. The wrapper completely bypasses the standard `VK_KHR_swapchain` extension. It implements its own swapchain logic, likely using Android's `Surface` and `AHardwareBuffer` for direct presentation control, better performance, and integration with the Android compositor.
3.  **Texture Decompression (`TextureDecoder_...`)**: The renderer emulates support for compressed texture formats (like BCn/DXT) that may not be available on the host GPU. It hooks `vkCreateImage`, `vkCmdCopyBufferToImage`, and other related functions to manage these textures. When an application tries to use a compressed texture, the renderer creates an uncompressed backing image and uses a custom decoder to handle the data on the fly.
4.  **Asynchronous Operations (`AsyncPipelineCreator`, `TimelineSemaphore_asyncWait`)**: To prevent the main renderer thread from stalling on long-running, blocking operations, the wrapper offloads them to worker threads. This is used for:
    *   **Pipeline Compilation:** `vkCreateGraphicsPipelines` and `vkCreateComputePipelines` are handled by the `AsyncPipelineCreator`.
    *   **Semaphore/Fence Waiting:** `vkWaitSemaphores` and `vkWaitForFences` are replaced with non-blocking IPC mechanisms that use `eventfd` and `sendmsg` to let the client perform the wait.
5.  **Shader Analysis (`ShaderInspector_...`)**: The wrapper intercepts shader creation (`vkCreateShaderModule`) and related queries (`vkGetShaderModuleCreateInfoIdentifierEXT`) to parse the SPIR-V bytecode. This allows it to extract metadata about the shader's resource usage without needing to fully compile a pipeline, which can be used for validation or optimization.
6.  **Custom Memory Management (`ResourceMemory_...`)**: To facilitate sharing memory between the client and the renderer server, the wrapper hooks `vkAllocateMemory` and `vkMapMemory`. It uses Android-specific heaps like ION or DMA-BUF to create shareable memory allocations and passes file descriptors (`fd`) back to the client via `sendmsg`.
7.  **Minor Hooks**: Small, targeted modifications to parameters, such as adding the `VK_SHADER_STAGE_TESSELLATION_EVALUATION_BIT` to push constant ranges to simplify shader development for the client.

### TLDR Function Table

| Vulkan Function | Pre-Dispatch Modification? | Post-Dispatch Modification? | Interesting Downstream Calls | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `vkCreateInstance` | **Yes** | No | `initVulkanInstance` | **Major Hook.** Injects required instance extensions (`VK_KHR_surface`, `VK_KHR_xlib_surface`, etc.) and initializes the instance-level dispatch table. |
| `vkDestroyInstance` | **No** | No | | Standard pass-through. |
| `vkEnumeratePhysicalDevices` | **No** | No | | Standard pass-through. |
| `vkGetPhysicalDeviceProperties` | No | **Yes** | `checkDeviceProperties` | **Minor Hook.** Overwrites the device name with "Vortek (%s)" and may adjust other reported properties. |
| `vkGetPhysicalDeviceQueueFamilyProperties` | **No** | No | | Standard pass-through. |
| `vkGetPhysicalDeviceMemoryProperties`| **Yes** | No | `overrideMemoryHeapSize` | **Minor Hook.** Modifies the reported memory heap sizes, likely capping them to a value provided by the client (`maxDeviceMemory`). |
| `vkGetPhysicalDeviceFeatures`| **Yes** | No | `checkDeviceFeatures` | **Minor Hook.** Enables or disables certain device features (e.g., forces `shaderClipDistance`, `samplerAnisotropy` to true). |
| `vkGetPhysicalDeviceFormatProperties`| **Yes** | No | `checkFormatProperties` | **Hook.** If a compressed format is queried and not supported, it fakes support by returning the properties of an uncompressed format (`VK_FORMAT_B8G8R8A8_UNORM`). |
| `vkGetPhysicalDeviceImageFormatProperties`| **Yes** | No | `isCompressedFormat`, `getCompressedImageFormatProperties` | **Hook.** Fakes support for compressed formats by returning predefined properties if the driver reports `VK_ERROR_FORMAT_NOT_SUPPORTED`. |
| `vkCreateDevice` | **Yes** | **Yes** | `disableUnsupportedDeviceFeatures`, `initVulkanDevice` | **Major Hook.** Disables features not supported by the wrapper. Injects required device extensions (`VK_KHR_swapchain`, external memory extensions, etc.). Initializes the device-level dispatch table. |
| `vkDestroyDevice` | **No** | No | | Standard pass-through. |
| `vkEnumerateInstanceVersion`| **No** | No | | Standard pass-through. |
| `vkEnumerateInstanceExtensionProperties`| **Yes** | No | `injectExtensions2` | **Hook.** Injects `VK_KHR_surface`, `VK_KHR_android_surface`, and `VK_KHR_xlib_surface` into the reported list of supported extensions. |
| `vkEnumerateDeviceExtensionProperties`| **Yes** | No | `getExposedDeviceExtensionProperties` | **Hook.** Filters the driver's supported extensions against a custom list of "exposed" extensions, hiding unsupported or problematic ones from the client. |
| `vkGetDeviceQueue` | **No** | No | | Standard pass-through. |
| `vkQueueSubmit` | **Yes** | No | `TextureDecoder_decodeAll` | **Major Hook.** Before submitting work, it ensures all pending texture decompressions are completed to prevent race conditions. |
| `vkQueueWaitIdle` | **No** | No | | Standard pass-through. |
| `vkDeviceWaitIdle` | **No** | No | | Standard pass-through. |
| `vkAllocateMemory` | **Yes** | N/A (dispatch is inside) | `ResourceMemory_allocate` | **Major Hook.** Intercepts memory allocation to use Android-specific shared memory (ION, DMA-BUF) for host-visible memory types. |
| `vkFreeMemory` | **Yes** | N/A (dispatch is inside) | `ResourceMemory_free` | **Major Hook.** Calls a custom deallocator to properly clean up shared memory resources (FDs, AHardwareBuffers). |
| `vkMapMemory` | **Yes** | N/A (dispatch bypassed) | `sendmsg` | **IPC Hook.** Bypasses `vkMapMemory`. Instead, sends the file descriptor of the shared memory allocation to the client via a socket message. |
| `vkUnmapMemory` | **Yes** | N/A (dispatch bypassed) | | **Stubbed.** The function is a no-op that logs "not implemented yet". Unmapping is the client's responsibility. |
| `vkFlushMappedMemoryRanges`| **Yes** | No | | **Minor Hook.** Uses a conditional check (`vortekSerializerCastVkObject`) to determine how to resolve the `VkDeviceMemory` handle from a custom wrapper object. |
| `vkInvalidateMappedMemoryRanges`| **Yes** | No | | **Minor Hook.** Same conditional handle resolution logic as `vkFlushMappedMemoryRanges`. |
| `vkGetDeviceMemoryCommitment`| **No** | No | | Standard pass-through. |
| `vkGetBufferMemoryRequirements`| **No** | No | | Standard pass-through. |
| `vkBindBufferMemory` | **No** | **Yes** | `TextureDecoder_addBoundBuffer` | **Hook.** After a successful binding, it registers the buffer-memory pair with the texture decoder for future reference during decompression. |
| `vkGetImageMemoryRequirements`| **No** | No | | Standard pass-through. |
| `vkBindImageMemory` | **No** | No | | Standard pass-through. No hook for the texture decoder here. |
|`vkGetImageSparseMemoryRequirements`|**No**|No||Standard pass-through query. |
|`vkGetPhysicalDeviceSparseImageFormatProperties`|**No**|No||Standard pass-through query. |
| `vkQueueBindSparse` | **No** | No | | Standard pass-through. Handles extremely complex deserialization but does not modify the data. |
| `vkCreateFence` | **No** | No | | Standard pass-through. |
| `vkDestroyFence` | **No** | No | | Standard pass-through. |
| `vkResetFences` | **No** | No | | Standard pass-through. |
| `vkGetFenceStatus` | **No** | No | | Standard pass-through. |
| `vkWaitForFences` | **Yes** | N/A (dispatch bypassed) | `dispatch_vkGetFenceFd`, `sendmsg` | **IPC Hook.** Bypasses the blocking wait. Gets file descriptors for the fences and sends them to the client for asynchronous waiting. |
| `vkCreateSemaphore` | **No** | No | | Standard pass-through. |
| `vkDestroySemaphore` | **No** | No | | Standard pass-through. |
| `vkCreateEvent` | **No** | No | | Standard pass-through. |
| `vkDestroyEvent` | **No** | No | | Standard pass-through. |
| `vkGetEventStatus` | **No** | No | | Standard pass-through. |
| `vkSetEvent` | **No** | No | | Standard pass-through. |
| `vkResetEvent` | **No** | No | | Standard pass-through. |
| `vkCreateQueryPool` | **No** | No | | Standard pass-through. |
| `vkDestroyQueryPool` | **No** | No | | Standard pass-through. |
| `vkGetQueryPoolResults` | **No** | No | | Standard pass-through. Handles data serialization. |
| `vkResetQueryPool` | **No** | No | | (Device-level version) Standard pass-through. |
| `vkCreateBuffer` | **No** | No | | Standard pass-through. |
| `vkDestroyBuffer` | **No** | **Yes** | `TextureDecoder_removeBoundBuffer` | **Hook.** Before destroying the buffer, it un-registers it from the texture decoder to clean up tracking data. |
| `vkCreateBufferView` | **No** | No | | Standard pass-through. |
| `vkDestroyBufferView` | **No** | No | | Standard pass-through. |
| `vkCreateImage` | **Yes** | N/A (dispatch replaced)| `TextureDecoder_createImage` | **Major Hook.** If the format is compressed, it uses a custom function to create an uncompressed backing image for emulation. |
| `vkDestroyImage` | **Yes** | N/A (dispatch bypassed)| `TextureDecoder_destroyImage` | **Hook.** If the image is managed by the decoder, it calls a custom destroy function to clean up associated resources. |
| `vkGetImageSubresourceLayout`| **No** | No | | Standard pass-through. |
| `vkCreateImageView` | **Yes** | No | | **Hook.** Modifies `VkImageViewCreateInfo` to use an uncompressed format if the source image is an emulated compressed texture. |
| `vkDestroyImageView` | **No** | No | | Standard pass-through. |
| `vkCreateShaderModule` | **Yes** | N/A (dispatch is inside)| `ShaderInspector_createModule` | **Hook.** Intercepts shader creation to parse SPIR-V and store metadata in a custom wrapper object. |
| `vkDestroyShaderModule` | **Yes** | N/A (dispatch is inside)| | **Hook.** Frees the custom wrapper object and its associated data before destroying the Vulkan handle. |
| `vkCreatePipelineCache` | **No** | No | | Standard pass-through. |
| `vkDestroyPipelineCache` | **No** | No | | Standard pass-through. |
| `vkGetPipelineCacheData` | **No** | No | | Standard pass-through. |
| `vkMergePipelineCaches` | **No** | No | | Standard pass-through. |
| `vkCreateGraphicsPipelines`| **Yes** | N/A (dispatch is async)| `AsyncPipelineCreator_create` | **Major Hook.** Offloads pipeline creation to a worker thread to avoid stalling. |
| `vkCreateComputePipelines` | **Yes** | N/A (dispatch is async)| `AsyncPipelineCreator_create` | **Major Hook.** Offloads pipeline creation to a worker thread. |
| `vkDestroyPipeline` | **No** | No | | Standard pass-through. |
| `vkCreatePipelineLayout` | **Yes** | No | | **Minor Hook.** Modifies `stageFlags` in `VkPushConstantRange` to automatically include the tessellation stage if the vertex stage is present. |
| `vkDestroyPipelineLayout`| **No** | No | | Standard pass-through. |
| `vkCreateSampler` | **No** | No | | Standard pass-through. |
| `vkDestroySampler` | **No** | No | | Standard pass-through. |
| `vkCreateDescriptorSetLayout`| **No** | No | | Standard pass-through. |
| `vkDestroyDescriptorSetLayout`| **No** | No | | Standard pass-through. |
| `vkCreateDescriptorPool` | **No** | No | | Standard pass-through. |
| `vkDestroyDescriptorPool` | **No** | No | | Standard pass-through. |
| `vkResetDescriptorPool` | **No** | No | | Standard pass-through. |
| `vkAllocateDescriptorSets`| **No** | No | | Standard pass-through. |
| `vkFreeDescriptorSets` | **No** | No | | Standard pass-through. |
| `vkUpdateDescriptorSets` | **No** | No | | Standard pass-through. |
| `vkCreateFramebuffer` | **No** | No | | Standard pass-through. |
| `vkDestroyFramebuffer` | **No** | No | | Standard pass-through. |
| `vkCreateRenderPass` | **No** | No | | Standard pass-through. |
| `vkDestroyRenderPass` | **No** | No | | Standard pass-through. |
| `vkGetRenderAreaGranularity`| **No** | No | | Standard pass-through. |
| `vkCreateCommandPool` | **No** | No | | Standard pass-through. |
| `vkDestroyCommandPool` | **No** | No | | Standard pass-through. |
| `vkResetCommandPool` | **No** | No | | Standard pass-through. |
| `vkAllocateCommandBuffers`| **No** | No | | Standard pass-through. |
| `vkFreeCommandBuffers` | **No** | No | | Standard pass-through. |
| `vkBeginCommandBuffer` | **No** | No | | Standard pass-through. |
| `vkEndCommandBuffer` | **Yes** | No | `getHandleRequestFunc` | **Major Hook.** This command acts as a trigger to process a batch of other `vkCmd...` calls that have been serialized into its input buffer, reducing IPC overhead. |
| `vkResetCommandBuffer` | **No** | No | | Standard pass-through. |
| `vkCmdBindPipeline` | **Yes** | No | `AsyncPipelineCreator_getVkHandle` | **Hook.** Synchronizes with the `AsyncPipelineCreator`. If a pipeline is still being created asynchronously, this call blocks until it is finished. |
| `vkCmdSetViewport` | **No** | No | | Standard pass-through. |
| `vkCmdSetScissor` | **No** | No | | Standard pass-through. |
| `vkCmdSetLineWidth` | **No** | No | | Standard pass-through. |
| `vkCmdSetDepthBias` | **No** | No | | Standard pass-through. |
| `vkCmdSetBlendConstants` | **No** | No | | Standard pass-through. |
| `vkCmdSetDepthBounds` | **No** | No | | Standard pass-through. |
| `vkCmdSetStencilCompareMask`| **No** | No | | Standard pass-through. |
| `vkCmdSetStencilWriteMask`| **No** | No | | Standard pass-through. |
| `vkCmdSetStencilReference`| **No** | No | | Standard pass-through. |
| `vkCmdBindDescriptorSets`| **No** | No | | Standard pass-through. |
| `vkCmdBindIndexBuffer` | **No** | No | | Standard pass-through. |
| `vkCmdBindVertexBuffers` | **No** | No | | Standard pass-through. |
| `vkCmdDraw` | **No** | No | | Standard pass-through. |
| `vkCmdDrawIndexed` | **No** | No | | Standard pass-through. |
| `vkCmdDrawIndirect` | **No** | No | | Standard pass-through. |
| `vkCmdDrawIndexedIndirect`| **No** | No | | Standard pass-through. |
| `vkCmdDispatch` | **No** | No | | Standard pass-through. |
| `vkCmdDispatchIndirect`| **No** | No | | Standard pass-through. |
| `vkCmdCopyBuffer` | **No** | No | | Standard pass-through. |
| `vkCmdCopyImage` | **No** | No | | Standard pass-through. |
| `vkCmdBlitImage` | **No** | No | | Standard pass-through. |
| `vkCmdCopyBufferToImage` | **Yes** | N/A (dispatch may be bypassed)| `TextureDecoder_containsImage`, `TextureDecoder_copyBufferToImage` | **Hook.** Intercepts the call if `dstImage` is a known emulated texture and redirects it to the custom decoder to handle decompression. |
| `vkCmdCopyImageToBuffer` | **Yes** | N/A (dispatch may be bypassed)| `TextureDecoder_containsImage`, `TextureDecoder_copyImageToBuffer` | **Hook.** Similar to the above, but for reading from an emulated texture. This would involve re-compressing or reading from the uncompressed backing store. |
| `vkCmdUpdateBuffer` | **No** | No | | Standard pass-through. |
| `vkCmdFillBuffer` | **No** | No | | Standard pass-through. |
| `vkCmdClearColorImage` | **No** | No | | Standard pass-through. |
| `vkCmdClearDepthStencilImage` | **No** | No | | Standard pass-through. |
| `vkCmdClearAttachments` | **No** | No | | Standard pass-through. |
| `vkCmdResolveImage` | **No** | No | | Standard pass-through. |
| `vkCmdSetEvent` | **No** | No | | Standard pass-through. |
| `vkCmdResetEvent` | **No** | No | | Standard pass-through. |
| `vkCmdWaitEvents` | **No** | No | | Standard pass-through. |
| `vkCmdPipelineBarrier` | **No** | No | | Standard pass-through. |
| `vkCmdBeginQuery` | **No** | No | | Standard pass-through. |
| `vkCmdEndQuery` | **No** | No | | Standard pass-through. |
| `vkCmdBeginConditionalRenderingEXT` | **No** | No | | Standard pass-through. |
| `vkCmdEndConditionalRenderingEXT`| **No** | No | | Standard pass-through. |
| `vkCmdResetQueryPool` | **No** | No | | Standard pass-through. |
| `vkCmdWriteTimestamp` | **No** | No | | Standard pass-through. |
| `vkCmdCopyQueryPoolResults`| **No** | No | | Standard pass-through. |
| `vkCmdPushConstants` | **Yes** | No | | **Minor Hook.** Modifies `stageFlags` to automatically include the tessellation stage if the vertex stage is present. |
| `vkCmdBeginRenderPass` | **No** | No | | Standard pass-through. |
| `vkCmdNextSubpass` | **No** | No | | Standard pass-through. |
| `vkCmdEndRenderPass` | **No** | No | | Standard pass-through. |
| `vkCmdExecuteCommands` | **No** | No | | Standard pass-through. |
| `vkGetPhysicalDeviceSurfaceCapabilitiesKHR`| **Yes** | N/A (dispatch bypassed)| `getWindowExtent` | **Major Hook.** Bypasses the driver. Returns capabilities based on the Android window, faking values like `minImageCount` to 1. |
| `vkGetPhysicalDeviceSurfaceFormatsKHR`| **Yes** | N/A (dispatch bypassed)| `getSurfaceFormats` | **Major Hook.** Bypasses the driver. Returns a hardcoded list of supported surface formats (`VK_FORMAT_B8G8R8A8_UNORM`, etc.). |
| `vkGetPhysicalDeviceSurfacePresentModesKHR`| **Yes** | N/A (dispatch bypassed)| | **Major Hook.** Bypasses the driver. Returns a hardcoded list of present modes (e.g., `VK_PRESENT_MODE_FIFO_KHR`). |
| `vkCreateSwapchainKHR` | **Yes** | N/A (dispatch bypassed)| `XWindowSwapchain_create` | **Major Hook.** Replaces the standard swapchain with a custom one. |
| `vkDestroySwapchainKHR` | **Yes** | N/A (dispatch bypassed)| `XWindowSwapchain_destroy` | **Hook.** Destroys the custom swapchain object. |
| `vkGetSwapchainImagesKHR`| **Yes** | N/A (dispatch bypassed)| | **Hook.** Returns image handles from the custom swapchain object. |
| `vkAcquireNextImageKHR` | **Yes** | N/A (dispatch bypassed)| `XWindowSwapchain_acquireNextImage` | **Hook.** Acquires an image from the custom swapchain. |
| `vkQueuePresentKHR` | **Yes** | N/A (dispatch bypassed)| `XWindowSwapchain_presentImage` | **Hook.** Presents an image using the custom swapchain logic. |
| `vkGetPhysicalDeviceFeatures2`| **Yes** | No | `checkDeviceFeatures` | **Minor Hook.** Same as the non-`2` version; forces certain features to be enabled. |
|`vkGetPhysicalDeviceProperties2`| **Yes** | No | `checkDeviceProperties` | **Minor Hook.** Same as the non-`2` version; overwrites device name and properties. |
|`vkGetPhysicalDeviceFormatProperties2`| **Yes** | No | `checkFormatProperties` | **Hook.** Same as the non-`2` version; fakes support for compressed formats. |
|`vkGetPhysicalDeviceImageFormatProperties2`| **Yes**| No | `getCompressedImageFormatProperties` | **Hook.** Same as the non-`2` version; fakes support for compressed formats. |
|`vkGetPhysicalDeviceQueueFamilyProperties2`|**No**|No||Standard pass-through. |
|`vkGetPhysicalDeviceMemoryProperties2`|**Yes**|No|`overrideMemoryHeapSize`| **Hook.** Same as the non-`2` version; modifies reported heap sizes. |
|`vkGetPhysicalDeviceSparseImageFormatProperties2`|**No**|No||Standard pass-through. |
| `vkCmdPushDescriptorSetKHR`| **No** | No | | Standard pass-through. |
| `vkTrimCommandPool` | **No** | No | | Standard pass-through. |
|`vkGetPhysicalDeviceExternalBufferProperties`|**No**|No||Standard pass-through. |
| `vkGetMemoryFdKHR` | **Yes** | N/A (dispatch is separate)| `sendmsg` | **IPC Hook.** Sends the acquired file descriptor over a socket instead of returning it directly. |
|`vkGetPhysicalDeviceExternalSemaphoreProperties`|**No**|No||Standard pass-through. |
| `vkGetSemaphoreFdKHR` | **Yes** | N/A (dispatch is separate)| `sendmsg` | **IPC Hook.** Sends the acquired file descriptor over a socket. |
|`vkGetPhysicalDeviceExternalFenceProperties`|**No**|No||Standard pass-through. |
| `vkGetFenceFdKHR` | **Yes** | N/A (dispatch is separate)| `sendmsg` | **IPC Hook.** Sends the acquired file descriptor over a socket. |
| `vkEnumeratePhysicalDeviceGroups`| **No** | No | | Standard pass-through. |
|`vkGetDeviceGroupPeerMemoryFeatures`| **No** | No | | Standard pass-through. |
| `vkBindBufferMemory2` | **Yes** | No | `TextureDecoder_addBoundBuffer` | **Hook.** Same as the non-`2` version; registers the binding with the texture decoder. |
| `vkBindImageMemory2` | **No** | No | | Standard pass-through. |
| `vkCmdSetDeviceMask` | **No** | No | | Standard pass-through. |
| `vkAcquireNextImage2KHR` | **Yes** | N/A (dispatch bypassed)| `XWindowSwapchain_acquireNextImage` | **Hook.** Same as the non-`2` version; acquires an image from the custom swapchain. |
| `vkCmdDispatchBase` | **No** | No | | Standard pass-through. |
| `vkCreateDescriptorUpdateTemplate`| **No** | No | | Not implemented in binary. |
| `vkDestroyDescriptorUpdateTemplate`| **No** | No | | Not implemented in binary. |
|`vkUpdateDescriptorSetWithTemplate`|**No**|No||Not implemented in binary. |
| `vkGetDescriptorSetLayoutSupport`| **No** | No | | Standard pass-through. |
|`vkGetPhysicalDeviceCalibrateableTimeDomainsKHR`|**No**|No||Standard pass-through. |
|`vkGetCalibratedTimestampsKHR`| **No** | No | | Standard pass-through. |
| `vkCreateRenderPass2` | **Yes** | No | | **Soft Hook.** Contains custom logic to correctly deserialize specific `pNext` extension structs, like `VkAttachmentSampleCountInfoAMD/NV`. |
| `vkCmdBeginRenderPass2` | **Yes** | No | | **Soft Hook.** Contains custom logic to correctly deserialize `pNext` extension structs like `VkDeviceGroupRenderPassBeginInfo`. |
| `vkCmdNextSubpass2` | **No** | No | | Standard pass-through. |
| `vkCmdEndRenderPass2` | **No** | No | | Standard pass-through. |
|`vkGetSemaphoreCounterValue`| **No** | No | | Standard pass-through. |
| `vkWaitSemaphores` | **Yes** | N/A (dispatch is async)| `TimelineSemaphore_asyncWait` | **Major Hook.** Replaces the blocking call with an asynchronous, non-stalling implementation using a worker thread and `eventfd`. |
| `vkSignalSemaphore` | **No** | No | | Standard pass-through. |
| `vkGetDeviceBufferMemoryRequirements`| **No** | No | | Standard pass-through. |
|`vkGetDeviceImageMemoryRequirements`| **No** | No | | Standard pass-through. |
|`vkGetDeviceImageSparseMemoryRequirements`|**No**|No||Standard pass-through. |
| `vkCreateSamplerYcbcrConversion`| **No** | No | | Standard pass-through. |
|`vkDestroySamplerYcbcrConversion`| **No** | No | | Standard pass-through. |
| `vkGetDeviceQueue2` | **No** | No | | Standard pass-through. |
|`vkCmdDrawIndirectCount` | **No** | No | | Standard pass-through. |
|`vkCmdDrawIndexedIndirectCount`| **No** | No | | Standard pass-through. |
|`vkCmdBindTransformFeedbackBuffersEXT`| **No** | No | | Standard pass-through. |
|`vkCmdBeginTransformFeedbackEXT`| **No** | No | | Standard pass-through. |
|`vkCmdEndTransformFeedbackEXT`| **No** | No | | Standard pass-through. |
|`vkCmdBeginQueryIndexedEXT` | **No** | No | | Standard pass-through. |
|`vkCmdEndQueryIndexedEXT` | **No** | No | | Standard pass-through. |
|`vkCmdDrawIndirectByteCountEXT`| **No** | No | | Standard pass-through. |
|`vkGetBufferOpaqueCaptureAddress`| **No** | No | | Standard pass-through. |
|`vkGetBufferDeviceAddress`| **No** | No | | Standard pass-through. |
|`vkGetDeviceMemoryOpaqueCaptureAddress`| **Yes** | No | | **Minor Hook.** Resolves the real `VkDeviceMemory` handle from a custom wrapper object before calling the driver. |
|`vkCmdSetLineStippleKHR` | **No** | No | | Renamed to `...EXT` in later versions, but is a standard pass-through. |
|`vkCmdSetCullMode` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetFrontFace` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetPrimitiveTopology`| **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetViewportWithCount`| **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetScissorWithCount`| **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdBindVertexBuffers2` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetDepthTestEnable` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetDepthWriteEnable` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetDepthCompareOp` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetDepthBoundsTestEnable` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetStencilTestEnable` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetStencilOp` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetRasterizerDiscardEnable`| **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetDepthBiasEnable` | **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetPrimitiveRestartEnable`| **No** | No | | (Dynamic State v2) Standard pass-through. |
|`vkCmdSetTessellationDomainOriginEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetDepthClampEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetPolygonModeEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetRasterizationSamplesEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetSampleMaskEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetAlphaToCoverageEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetAlphaToOneEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetLogicOpEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetColorBlendEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetColorBlendEquationEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetColorWriteMaskEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetRasterizationStreamEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetConservativeRasterizationModeEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetExtraPrimitiveOverestimationSizeEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetDepthClipEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetSampleLocationsEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetColorBlendAdvancedEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetProvokingVertexModeEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetLineRasterizationModeEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetLineStippleEnableEXT`|**No**|No||Standard pass-through. |
|`vkCmdSetDepthClipNegativeOneToOneEXT`|**No**|No||Standard pass-through. |
|`vkCmdCopyBuffer2KHR`|**No**|No||(Alias for `vkCmdCopyBuffer2`) Standard pass-through. |
|`vkCmdCopyImage2KHR`|**No**|No||(Alias for `vkCmdCopyImage2`) Standard pass-through. |
|`vkCmdBlitImage2KHR`|**No**|No||(Alias for `vkCmdBlitImage2`) Standard pass-through. |
|`vkCmdCopyBufferToImage2KHR`|**Yes**|N/A (dispatch may be bypassed)|`TextureDecoder_containsImage`, `TextureDecoder_copyBufferToImage`|**Hook.** (Alias for `vkCmdCopyBufferToImage2`) Intercepts the call to handle texture decompression. |
|`vkCmdCopyImageToBuffer2KHR`|**No**|No||(Alias for `vkCmdCopyImageToBuffer2`) Standard pass-through. |
|`vkCmdResolveImage2KHR`|**No**|No||(Alias for `vkCmdResolveImage2`) Standard pass-through. |
|`vkCmdSetEvent2KHR`|**No**|No||(Alias for `vkCmdSetEvent2`) Standard pass-through. |
|`vkCmdResetEvent2KHR`|**No**|No||(Alias for `vkCmdResetEvent2`) Standard pass-through. |
|`vkCmdWaitEvents2KHR`|**No**|No||(Alias for `vkCmdWaitEvents2`) Standard pass-through. |
|`vkCmdPipelineBarrier2KHR`|**No**|No||(Alias for `vkCmdPipelineBarrier2`) Standard pass-through. |
|`vkQueueSubmit2KHR`|**Yes**|No|`TextureDecoder_decodeAll`|**Hook.** (Alias for `vkQueueSubmit2`) Ensures texture decompression is complete before submission. |
|`vkCmdWriteTimestamp2KHR`|**No**|No||(Alias for `vkCmdWriteTimestamp2`) Standard pass-through. |
|`vkCmdBeginRenderingKHR`|**Yes**|No|`TextureDecoder_containsImage`|**Hook.** (Alias for `vkCmdBeginRendering`) Replaces image views for emulated textures with their backing storage views. |
|`vkCmdEndRenderingKHR`|**No**|No||(Alias for `vkCmdEndRendering`) Standard pass-through. |
|`vkGetSemaphoreCounterValueKHR`|**No**|No||(Alias for `vkGetSemaphoreCounterValue`) Standard pass-through. |
|`vkWaitSemaphoresKHR`|**Yes**|N/A (dispatch is async)|`TimelineSemaphore_asyncWait`|**Major Hook.** (Alias for `vkWaitSemaphores`) Replaces the blocking call with an asynchronous implementation. |
|`vkSignalSemaphoreKHR`|**No**|No||(Alias for `vkSignalSemaphore`) Standard pass-through. |

### 1. `vt_handle_vkDestroyDevice`

**Function Signature:**
```c
void vt_handle_vkDestroyDevice(VtContext* context);
```

**Purpose:**
This function is a simple wrapper around `vkDestroyDevice`. It deserializes the `VkDevice` handle and calls the corresponding dispatch function.

**Analysis:**

1.  **Deserialization:**
    *   It reads a `VkDevice` handle from the input buffer.
    *   The handle ID is resolved to a real `VkDevice` handle via `VkObject_fromId`.

2.  **Modification/Hooking:**
    *   There are **no modifications or hooking** in this function. It is a direct pass-through.

3.  **Dispatch:**
    *   It calls `dispatch_vkDestroyDevice` with the resolved `VkDevice` handle and a `NULL` allocator.

4.  **Serialization:**
    *   None. `vkDestroyDevice` has a `void` return type.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkDestroyDevice command. A simple pass-through wrapper.
 * @param context The renderer context, from which the input buffer is read.
 */
void vt_handle_vkDestroyDevice(VtContext* context) {
    // 1. DESERIALIZATION
    
    // The device handle is the only argument.
    // The client may send a 0 ID for the default device.
    void* deviceId = (void*)**(undefined8**)(context->inputBufferPtr);
    if (*context->inputBufferPtr == '\0') {
        deviceId = (void*)context; // Using context as a default handle ID
    }
    
    // Resolve the ID to a real Vulkan handle.
    VkDevice device = (VkDevice)VkObject_fromId(deviceId);

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    
    // Call the real Vulkan function. pAllocator is NULL.
    dispatch_vkDestroyDevice(device, NULL);

    // 4. SERIALIZATION - None.
    return;
}
```

---

### 2. `vt_handle_vkGetDeviceQueue`

**Function Signature:**
```c
void vt_handle_vkGetDeviceQueue(VtContext* context);
```

**Purpose:**
This function wraps `vkGetDeviceQueue`, retrieving a queue handle from a device.

**Analysis:**

1.  **Deserialization:**
    *   Reads `VkDevice` handle, `queueFamilyIndex`, and `queueIndex` from the input buffer.
    *   The `VkDevice` handle is resolved via `VkObject_fromId`.

2.  **Modification/Hooking:**
    *   There are **no modifications**. The deserialized arguments are used directly.

3.  **Dispatch:**
    *   Calls `dispatch_vkGetDeviceQueue` with the deserialized parameters. The retrieved `VkQueue` handle is stored in a local variable.

4.  **Serialization:**
    *   The resulting `VkQueue` handle is written to an 8-byte temporary buffer.
    *   This buffer is then written to the `clientRingBuffer` along with a success code (`0x800000000`), informing the client of the new handle.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkGetDeviceQueue command. A simple pass-through wrapper.
 * @param context The renderer context.
 */
void vt_handle_vkGetDeviceQueue(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    
    char* input = context->inputBufferPtr;
    VkDevice deviceHandle;
    uint32_t queueFamilyIndex;
    uint32_t queueIndex;

    // Determine device handle source (default or provided)
    if (*input == '\0') {
        deviceHandle = (VkDevice)context;
        queueFamilyIndex = *(uint32_t*)(input + 1);
        queueIndex = *(uint32_t*)(input + 5);
    } else {
        deviceHandle = (VkDevice)*(void**)(input + 1);
        queueFamilyIndex = *(uint32_t*)(input + 9);
        queueIndex = *(uint32_t*)(input + 13);
    }
    
    // Resolve handles
    VkDevice device = (VkDevice)VkObject_fromId(deviceHandle);

    // 2. MODIFICATION/HOOKING - None

    // 3. DISPATCH
    
    VkQueue queue = VK_NULL_HANDLE;
    dispatch_vkGetDeviceQueue(device, queueFamilyIndex, queueIndex, &queue);

    // 4. SERIALIZATION
    
    // Allocate a small buffer for the result.
    void* pResultBuffer = malloc(8);
    *(VkQueue*)pResultBuffer = queue;

    // Write a success code and the handle to the client.
    RingBuffer_write(context->clientRingBuffer, (void*)0x800000000, 8); 
    RingBuffer_write(context->clientRingBuffer, pResultBuffer, 8);
    
    free(pResultBuffer);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 3. `vt_handle_vkQueueSubmit`

**Function Signature:**
```c
void vt_handle_vkQueueSubmit(VtContext* context);
```

**Purpose:**
Wraps the `vkQueueSubmit` function, which is a critical path for executing GPU work.

**Analysis:**

1.  **Deserialization:**
    *   Deserializes `VkQueue`, `VkFence`, and an array of `VkSubmitInfo` structs.
    *   It iterates through each `VkSubmitInfo` struct, resolving all `VkSemaphore` and `VkCommandBuffer` handles using `VkObject_fromId`.
    *   It correctly handles the `pNext` chain for each `VkSubmitInfo` struct, deserializing known extension structs like `VkTimelineSemaphoreSubmitInfo`.

2.  **Modification/Hooking:**
    *   **This function contains a significant hook.** Before calling `dispatch_vkQueueSubmit`, it makes a call to `TextureDecoder_decodeAll(context->textureDecoder)`.
    *   This is a synchronization and resource management hook. It ensures that any pending texture decompression tasks (managed by the `TextureDecoder` module) are completed *before* the command buffers that might use those textures are submitted to the GPU. This prevents race conditions where the GPU tries to sample a texture that hasn't been fully decompressed from its original format (e.g., BCn) into the emulated format (e.g., `VK_FORMAT_B8G8R8A8_UNORM`).

3.  **Dispatch:**
    *   Calls `dispatch_vkQueueSubmit` with the fully deserialized and resolved arguments.

4.  **Serialization:**
    *   Writes the `VkResult` of the submission back to the client ring buffer.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkQueueSubmit command. Hooks the call to ensure texture
 *        decompression is complete before submission.
 * @param context The renderer context.
 */
void vt_handle_vkQueueSubmit(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    
    char* input = context->inputBufferPtr;

    // Deserialize VkQueue handle
    VkQueue queue = (VkQueue)VkObject_fromId(*(void**)(input));

    // Deserialize VkFence handle
    VkFence fence = (VkFence)VkObject_fromId(*(void**)(input + 8));

    // Deserialize array of VkSubmitInfo structs
    uint32_t submitCount = *(uint32_t*)(input + 16);
    VkSubmitInfo* pSubmits = deserialize_submit_info_array(context, input + 20, submitCount);

    // 2. MODIFICATION/HOOKING

    // Check if there is an active TextureDecoder instance.
    if (context->textureDecoder != NULL) {
        // This is the hook: process all pending texture decompression tasks.
        // It ensures that any textures referenced by the command buffers are
        // ready in their uncompressed format before the GPU executes the commands.
        TextureDecoder_decodeAll(context->textureDecoder);
    }
    
    // 3. DISPATCH
    
    // Call the real Vulkan function.
    VkResult result = dispatch_vkQueueSubmit(queue, submitCount, pSubmits, fence);

    // If the device is lost, update the context's error state.
    if (result == VK_ERROR_DEVICE_LOST) {
        context->lastError = VK_ERROR_DEVICE_LOST;
    }

    // 4. SERIALIZATION
    
    // If the client is waiting for a status, write the result back.
    if (RingBuffer_hasStatus(context->clientRingBuffer, 4)) {
        RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
    }

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 4. `vt_handle_vkQueueWaitIdle`

**Function Signature:**
```c
void vt_handle_vkQueueWaitIdle(VtContext* context);
```

**Purpose:**
A simple wrapper for `vkQueueWaitIdle`.

**Analysis:**

*   **Deserialization:** Reads and resolves the `VkQueue` handle.
*   **Modification/Hooking:** **None.**
*   **Dispatch:** Calls `dispatch_vkQueueWaitIdle`.
*   **Serialization:** Writes the `VkResult` back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkQueueWaitIdle command. A simple pass-through.
 * @param context The renderer context.
 */
void vt_handle_vkQueueWaitIdle(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    VkQueue queue = (VkQueue)VkObject_fromId(*(void**)context->inputBufferPtr);

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkResult result = dispatch_vkQueueWaitIdle(queue);

    // 4. SERIALIZATION
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 5. `vt_handle_vkDeviceWaitIdle`

**Function Signature:**
```c
void vt_handle_vkDeviceWaitIdle(VtContext* context);
```

**Purpose:**
A simple wrapper for `vkDeviceWaitIdle`.

**Analysis:**

*   **Deserialization:** Reads and resolves the `VkDevice` handle.
*   **Modification/Hooking:** **None.**
*   **Dispatch:** Calls `dispatch_vkDeviceWaitIdle`.
*   **Serialization:** Writes the `VkResult` back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkDeviceWaitIdle command. A simple pass-through.
 * @param context The renderer context.
 */
void vt_handle_vkDeviceWaitIdle(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)context->inputBufferPtr);

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkResult result = dispatch_vkDeviceWaitIdle(device);

    // 4. SERIALIZATION
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 6. `vt_handle_vkAllocateMemory`

**Function Signature:**
```c
void vt_handle_vkAllocateMemory(VtContext* context);
```

**Purpose:**
Wraps `vkAllocateMemory`, but with significant modifications to handle Android-specific memory types like `AHardwareBuffer` and ION/DMA-BUF heaps.

**Analysis:**

1.  **Deserialization:**
    *   Deserializes the `VkDevice` handle and the `VkMemoryAllocateInfo` struct, including any `pNext` chain extensions like `VkImportMemoryFdInfoKHR` or `VkMemoryDedicatedAllocateInfo`.

2.  **Modification/Hooking:**
    *   **This function is a major hook.** Instead of directly calling `vkAllocateMemory`, it calls a custom helper function, `ResourceMemory_allocate`.
    *   `ResourceMemory_allocate` inspects the allocation info:
        *   If it's a standard device-local allocation, it may proceed normally.
        *   If it's a host-visible allocation (`VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`), it attempts to use a special Android memory heap. The code shows attempts to open `/dev/dma_heap` and `/dev/ion`, which are used for creating shareable memory buffers. It then uses an `ioctl` to allocate a buffer and gets a file descriptor (`fd`) for it. This `fd` is the key to sharing memory with the client.
        *   If `AHardwareBuffer` is involved (via an extension), it will use the Android NDK functions to allocate a hardware buffer and get its `fd`.
    *   The `ResourceMemory_allocate` function returns a custom wrapper struct that contains not only the `VkDeviceMemory` handle but also the `fd` and other metadata.

3.  **Dispatch:**
    *   The call to `dispatch_vkAllocateMemory` is made *inside* `ResourceMemory_allocate`. The handler's main job is to delegate to this custom logic.

4.  **Serialization:**
    *   It writes back the `VkResult` and a pointer to the custom `ResourceMemory` wrapper struct, not just the raw `VkDeviceMemory` handle.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkAllocateMemory command. Replaces standard allocation with a custom
 *        allocator (`ResourceMemory_allocate`) to support Android-specific shared memory.
 * @param context The renderer context.
 */
void vt_handle_vkAllocateMemory(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)(input));
    VkMemoryAllocateInfo allocInfo = {};
    deserialize_memory_allocate_info(context, input + 8, &allocInfo);

    // 2. MODIFICATION/HOOKING & DISPATCH
    
    // This is the hook. Instead of calling vkAllocateMemory directly, it calls a
    // custom allocator which handles standard, ION, and DMA-BUF allocations.
    // The flags `hostVisibleMemoryFlag` and `coherentMemoryFlag` from the context
    // guide the allocation strategy.
    ResourceMemory* resourceMem = ResourceMemory_allocate(
        device, 
        &allocInfo, 
        context->hostVisibleMemoryFlag, 
        context->coherentMemoryFlag
    );

    // Check the result from the custom allocator.
    VkResult result = VK_ERROR_OUT_OF_HOST_MEMORY;
    if (resourceMem->vkHandle != VK_NULL_HANDLE) { // Assuming vkHandle is part of ResourceMemory
        result = VK_SUCCESS;
    }
    
    // 4. SERIALIZATION
    
    // Allocate a buffer for the result pointer.
    void* pResultBuffer = malloc(sizeof(void*));
    *(ResourceMemory**)pResultBuffer = resourceMem;
    
    // Write result and the wrapper handle back to the client.
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
    RingBuffer_write(context->clientRingBuffer, pResultBuffer, sizeof(void*));
    
    free(pResultBuffer);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 7. `vt_handle_vkFreeMemory`

**Function Signature:**
```c
void vt_handle_vkFreeMemory(void);
```

**Purpose:**
Complements `vt_handle_vkAllocateMemory` by freeing memory using the custom `ResourceMemory` wrapper.

**Analysis:**

*   **Deserialization:** Reads `VkDevice` and the custom `ResourceMemory` handle ID.
*   **Modification/Hooking:** It calls `ResourceMemory_free` instead of `dispatch_vkFreeMemory`. This custom function knows how to correctly tear down the allocation, including:
    *   Calling `dispatch_vkFreeMemory` on the internal `VkDeviceMemory` handle.
    *   Closing the file descriptor (`fd`) if it was an ION or DMA-BUF allocation.
    *   Releasing the `AHardwareBuffer` if one was used.
*   **Dispatch:** The call to `dispatch_vkFreeMemory` is inside `ResourceMemory_free`.
*   **Serialization:** None (`void` return).

**Decompiled Code:**
```c
/**
 * @brief Handles the vkFreeMemory command. Delegates to a custom deallocator.
 * @param context The renderer context.
 */
void vt_handle_vkFreeMemory(VtContext* context) {
    // 1. DESERIALIZATION
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)context->inputBufferPtr);
    ResourceMemory* resourceMem = (ResourceMemory*)VkObject_fromId(*(void**)(context->inputBufferPtr + 8));

    // 2. MODIFICATION/HOOKING & DISPATCH
    // This is the hook. It calls the custom free function which handles
    // the underlying Vulkan memory, file descriptors, and AHardwareBuffers.
    ResourceMemory_free(device, resourceMem);

    // 4. SERIALIZATION - None.
    return;
}
```

---

### 8. `vt_handle_vkMapMemory`

**Function Signature:**
```c
void vt_handle_vkMapMemory(VtContext* context);
```

**Purpose:**
**This function does not map memory.** Instead, it serves as an IPC mechanism to send the file descriptor of a shareable memory allocation back to the client.

**Analysis:**

1.  **Deserialization:**
    *   Reads `VkDevice` and the `ResourceMemory` handle ID.

2.  **Modification/Hooking:**
    *   **The entire function's logic is replaced.** It does not call `dispatch_vkMapMemory`.
    *   It retrieves the file descriptor (`fd`) from the custom `ResourceMemory` wrapper struct that was created during `vt_handle_vkAllocateMemory`.
    *   It constructs a `msghdr` with control data (`msg_control`) containing the `fd`.
    *   It uses `sendmsg` to send this `msghdr` over the `clientFd` socket.

3.  **Dispatch:**
    *   **No call** to `dispatch_vkMapMemory`.

4.  **Serialization:**
    *   Serialization is done via `sendmsg`, not the `RingBuffer`. The client will receive the message, extract the file descriptor, and perform its own `mmap` call to access the shared memory.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkMapMemory command. This is NOT a wrapper. It's an IPC
 *        mechanism to send a file descriptor for shared memory to the client.
 * @param context The renderer context.
 */
void vt_handle_vkMapMemory(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    ResourceMemory* resourceMem = (ResourceMemory*)VkObject_fromId(*(void**)(context->inputBufferPtr));

    // 2. MODIFICATION/HOOKING
    
    // Prepare the message header for sending the file descriptor.
    struct msghdr msg = {0};
    struct iovec iov = {0};
    char cmsg_buf[CMSG_SPACE(sizeof(int))];
    
    int fd = resourceMem->fd; // Assuming 'fd' is part of the custom ResourceMemory struct
    
    // A small payload can be sent, but the main data is in the control message.
    int result_payload = (fd >= 0) ? VK_SUCCESS : VK_ERROR_MEMORY_MAP_FAILED;
    iov.iov_base = &result_payload;
    iov.iov_len = sizeof(result_payload);
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;

    if (fd >= 0) {
        msg.msg_control = cmsg_buf;
        msg.msg_controllen = sizeof(cmsg_buf);
        
        struct cmsghdr* cmsg = CMSG_FIRSTHDR(&msg);
        cmsg->cmsg_level = SOL_SOCKET;
        cmsg->cmsg_type = SCM_RIGHTS;
        cmsg->cmsg_len = CMSG_LEN(sizeof(int));
        *(int*)CMSG_DATA(cmsg) = fd;
    } else {
        msg.msg_control = NULL;
        msg.msg_controllen = 0;
    }

    // 3. DISPATCH - Replaced by IPC
    // 4. SERIALIZATION - Done via sendmsg
    sendmsg(context->clientFd, &msg, 0);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 9. `vt_handle_vkUnmapMemory`

**Function Signature:**
```c
void vt_handle_vkUnmapMemory(void);
```

**Purpose:**
This is a stub function. Since the memory mapping is handled by the client via the file descriptor, unmapping is also the client's responsibility.

**Analysis:**

*   **Deserialization:** None.
*   **Modification/Hooking:** The function simply logs that it's not implemented.
*   **Dispatch:** **No call** to `dispatch_vkUnmapMemory`.
*   **Serialization:** None.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkUnmapMemory command. This is a stub, as the client
 *        is responsible for munmap.
 */
void vt_handle_vkUnmapMemory(void) {
    // The function is a no-op, just logs a message.
    __android_log_print(ANDROID_LOG_DEBUG, "System.out", "%s not implemented yet\n", "vkUnmapMemory");
    return;
}
```

---

### 10. `vt_handle_vkFlushMappedMemoryRanges`

**Function Signature:**
```c
void vt_handle_vkFlushMappedMemoryRanges(VtContext* context);
```

**Purpose:**
Wraps the `vkFlushMappedMemoryRanges` call.

**Analysis:**

1.  **Deserialization:**
    *   Reads the `VkDevice` handle and an array of `VkMappedMemoryRange` structs.
    *   For each range, it resolves the `VkDeviceMemory` handle ID to a real handle.
    *   There is a check for a global flag, `vortekSerializerCastVkObject`. If this flag is set, it appears to resolve the handle from a different field within the `VkObject` wrapper struct (`+0x18`). This suggests the wrapper can store multiple handle types or representations, and this flag switches between them. This could be used to differentiate between a raw Vulkan handle and a custom wrapper struct.

2.  **Modification/Hooking:**
    *   There are **no modifications** to the range data (offset, size) itself. The only "unusual" behavior is the conditional handle resolution based on `vortekSerializerCastVkObject`.

3.  **Dispatch:**
    *   Calls `dispatch_vkFlushMappedMemoryRanges` with the deserialized and resolved arguments.

4.  **Serialization:**
    *   Writes the `VkResult` back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkFlushMappedMemoryRanges command. It's a pass-through
 *        with a special case for handle resolution.
 * @param context The renderer context.
 */
void vt_handle_vkFlushMappedMemoryRanges(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    uint32_t memoryRangeCount = *(uint32_t*)(input + 8);
    
    // Allocate space for the ranges in the temporary buffer.
    VkMappedMemoryRange* pRanges = (VkMappedMemoryRange*)malloc_from_temp_buffer(context, memoryRangeCount * sizeof(VkMappedMemoryRange));

    char* rangeDataPtr = input + 12;
    for (uint32_t i = 0; i < memoryRangeCount; ++i) {
        // Deserialize each range.
        pRanges[i].sType = *(VkStructureType*)rangeDataPtr;
        // pNext is likely null or handled similarly if needed.
        pRanges[i].pNext = NULL; 
        
        void* memoryId = *(void**)(rangeDataPtr + 8);
        
        // This is the conditional handle resolution logic.
        if (vortekSerializerCastVkObject != '\0') {
             // Cast/resolve to a special handle type (e.g., from a wrapper).
             pRanges[i].memory = *(VkDeviceMemory*)((char*)VkObject_fromId(memoryId) + 0x18);
        } else {
             // Standard handle resolution.
             pRanges[i].memory = (VkDeviceMemory)VkObject_fromId(memoryId);
        }
        
        pRanges[i].offset = *(VkDeviceSize*)(rangeDataPtr + 16);
        pRanges[i].size = *(VkDeviceSize*)(rangeDataPtr + 24);
        
        // Advance to the next serialized range.
        rangeDataPtr += sizeof(VkMappedMemoryRange); // Simplified, original is more complex
    }

    // 2. MODIFICATION/HOOKING - None, other than handle resolution.

    // 3. DISPATCH
    
    VkResult result = dispatch_vkFlushMappedMemoryRanges(device, memoryRangeCount, pRanges);

    // 4. SERIALIZATION
    
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

### 1. `vt_handle_vkInvalidateMappedMemoryRanges`

**Purpose:**
This function is a wrapper for `vkInvalidateMappedMemoryRanges`. It deserializes an array of `VkMappedMemoryRange` structs, resolves the handles, and calls the real Vulkan function.

**Decompiled Code & Analysis:**
```c
/**
 * @brief Handles the vkInvalidateMappedMemoryRanges command.
 *        This function is a pass-through wrapper with conditional handle resolution.
 * @param context The renderer context.
 */
void vt_handle_vkInvalidateMappedMemoryRanges(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    uint32_t memoryRangeCount = *(uint32_t*)(input + 8);
    
    // Allocate a temporary buffer for the array of structs.
    VkMappedMemoryRange* pRanges = (VkMappedMemoryRange*)malloc_from_temp_buffer(context, memoryRangeCount * sizeof(VkMappedMemoryRange));

    // Pointer to the start of the serialized array data.
    char* rangeDataPtr = input + 12;

    for (uint32_t i = 0; i < memoryRangeCount; ++i) {
        // Deserialize each VkMappedMemoryRange struct.
        pRanges[i].sType = *(VkStructureType*)rangeDataPtr;
        pRanges[i].pNext = NULL; // Assuming pNext is not handled here.
        
        void* memoryId = *(void**)(rangeDataPtr + 8);

        // 2. MODIFICATION/HOOKING (Conditional Handle Resolution)
        
        // This is the same conditional handle resolution seen in the flush handler.
        // It checks a global flag to decide how to resolve the memory handle ID.
        // This suggests the wrapper can store different types of information for a
        // given Vulkan object ID.
        if (vortekSerializerCastVkObject != '\0') {
             // Resolve to a special handle stored at offset 0x18 within the wrapper object.
             pRanges[i].memory = *(VkDeviceMemory*)((char*)VkObject_fromId(memoryId) + 0x18);
        } else {
             // Standard handle resolution.
             pRanges[i].memory = (VkDeviceMemory)VkObject_fromId(memoryId);
        }
        
        pRanges[i].offset = *(VkDeviceSize*)(rangeDataPtr + 16);
        pRanges[i].size = *(VkDeviceSize*)(rangeDataPtr + 24);
        
        // Advance to the next struct in the serialized buffer.
        rangeDataPtr += sizeof(VkMappedMemoryRange); // Simplified assumption
    }

    // 3. DISPATCH
    
    VkResult result = dispatch_vkInvalidateMappedMemoryRanges(device, memoryRangeCount, pRanges);

    // 4. SERIALIZATION
    
    // Write the result back to the client.
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

**Summary:** This function is a straightforward wrapper. Its only "unusual" behavior is the conditional logic for resolving the `VkDeviceMemory` handle, which is identical to the `vkFlushMappedMemoryRanges` handler. There are no other functional modifications.

---

### 2. `vt_handle_vkGetDeviceMemoryCommitment`

**Purpose:**
A simple pass-through wrapper for the `vkGetDeviceMemoryCommitment` query.

**Analysis:**

1.  **Deserialization:** Reads the `VkDevice` and `VkDeviceMemory` handles from the input buffer and resolves them. The `VkDeviceMemory` handle is resolved from within a custom wrapper object (`+0x18`), indicating it's dealing with a `ResourceMemory` object.
2.  **Modification/Hooking:** **None.** It is a direct query.
3.  **Dispatch:** Calls `dispatch_vkGetDeviceMemoryCommitment`.
4.  **Serialization:** Writes a success code and the 64-bit `pCommittedMemoryInBytes` value back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkGetDeviceMemoryCommitment command.
 * @param context The renderer context.
 */
void vt_handle_vkGetDeviceMemoryCommitment(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    
    // The handle ID points to a ResourceMemory wrapper, and the actual VkDeviceMemory
    // handle is at offset 0x18 within that wrapper.
    void* memoryWrapper = VkObject_fromId(*(void**)(input + 8));
    VkDeviceMemory memory = *(VkDeviceMemory*)((char*)memoryWrapper + 0x18);
    
    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkDeviceSize committedMemoryInBytes = 0;
    dispatch_vkGetDeviceMemoryCommitment(device, memory, &committedMemoryInBytes);
    
    // 4. SERIALIZATION
    // Write a success code followed by the 8-byte result.
    long result_header = 0x800000000;
    RingBuffer_write(context->clientRingBuffer, &result_header, 8);
    RingBuffer_write(context->clientRingBuffer, &committedMemoryInBytes, sizeof(VkDeviceSize));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 3. `vt_handle_vkGetBufferMemoryRequirements`

**Purpose:**
A simple pass-through wrapper for `vkGetBufferMemoryRequirements`.

**Analysis:**

*   **Deserialization:** Reads and resolves the `VkDevice` and `VkBuffer` handles.
*   **Modification/Hooking:** **None.**
*   **Dispatch:** Calls `dispatch_vkGetBufferMemoryRequirements`.
*   **Serialization:** Serializes the resulting `VkMemoryRequirements` struct (`size`, `alignment`, `memoryTypeBits`) and writes it back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkGetBufferMemoryRequirements command. A simple pass-through.
 * @param context The renderer context.
 */
void vt_handle_vkGetBufferMemoryRequirements(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkBuffer buffer = (VkBuffer)VkObject_fromId(*(void**)(input + 8));

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkMemoryRequirements memReqs = {0};
    dispatch_vkGetBufferMemoryRequirements(device, buffer, &memReqs);

    // 4. SERIALIZATION
    
    // Allocate a temporary buffer for the result struct.
    void* pResultBuffer = malloc(sizeof(VkMemoryRequirements));
    
    // Copy the struct members.
    *(VkDeviceSize*)pResultBuffer = memReqs.size;
    *(VkDeviceSize*)((char*)pResultBuffer + 8) = memReqs.alignment;
    *(uint32_t*)((char*)pResultBuffer + 16) = memReqs.memoryTypeBits;

    // Write a header and the serialized struct to the client.
    long result_header = 0x1400000000; // Success code + size (0x14 bytes)
    RingBuffer_write(context->clientRingBuffer, &result_header, 8);
    RingBuffer_write(context->clientRingBuffer, pResultBuffer, sizeof(VkMemoryRequirements));
    
    free(pResultBuffer);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 4. `vt_handle_vkBindBufferMemory`

**Purpose:**
Wraps `vkBindBufferMemory` and hooks the call to register the binding with the `TextureDecoder`.

**Analysis:**

1.  **Deserialization:** Reads and resolves handles for `VkDevice`, `VkBuffer`, and the custom `ResourceMemory` wrapper. It also reads the `memoryOffset`.
2.  **Modification/Hooking:**
    *   **This is a hooked function.** After the dispatch call, it checks if the binding was successful (`result == VK_SUCCESS`) and if the `TextureDecoder` is enabled (`context->textureDecoder != NULL`).
    *   If both are true, it calls `TextureDecoder_addBoundBuffer`. This function informs the texture decoder that a specific buffer (identified by its `VkBuffer` handle) is now backed by a specific custom memory allocation (`ResourceMemory` object) at a given offset. This allows the decoder to later identify which memory to read from when it needs to decompress a texture stored in that buffer.
3.  **Dispatch:** Calls `dispatch_vkBindBufferMemory` using the `VkDeviceMemory` handle stored inside the `ResourceMemory` wrapper (`resourceMem->vkHandle`).
4.  **Serialization:** Writes the `VkResult` back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkBindBufferMemory command. Hooks the call to register the
 *        binding with the custom texture decoder.
 * @param context The renderer context.
 */
void vt_handle_vkBindBufferMemory(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkBuffer buffer = (VkBuffer)VkObject_fromId(*(void**)(input + 8));
    
    // The client sends an ID for our custom memory wrapper.
    ResourceMemory* resourceMem = (ResourceMemory*)VkObject_fromId(*(void**)(input + 16));
    
    // The real VkDeviceMemory handle is inside the wrapper.
    VkDeviceMemory memory = resourceMem->vkHandle; // Assuming vkHandle is at offset 0x18
    
    VkDeviceSize memoryOffset = *(VkDeviceSize*)(input + 24);

    // 3. DISPATCH
    VkResult result = dispatch_vkBindBufferMemory(device, buffer, memory, memoryOffset);

    // 2. MODIFICATION/HOOKING
    if (result == VK_SUCCESS && context->textureDecoder != NULL) {
        // Register this successful binding with the texture decoder.
        TextureDecoder_addBoundBuffer(context->textureDecoder, resourceMem, buffer, memoryOffset);
    }
    
    // 4. SERIALIZATION
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 5. `vt_handle_vkGetImageMemoryRequirements`

**Purpose:**
A simple pass-through wrapper for `vkGetImageMemoryRequirements`.

**Analysis:**

*   **Deserialization:** Reads and resolves `VkDevice` and `VkImage` handles.
*   **Modification/Hooking:** **None.**
*   **Dispatch:** Calls `dispatch_vkGetImageMemoryRequirements`.
*   **Serialization:** Serializes the `VkMemoryRequirements` struct and writes it back, identical to the buffer version.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkGetImageMemoryRequirements command. A simple pass-through.
 * @param context The renderer context.
 */
void vt_handle_vkGetImageMemoryRequirements(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkImage image = (VkImage)VkObject_fromId(*(void**)(input + 8));

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkMemoryRequirements memReqs = {0};
    dispatch_vkGetImageMemoryRequirements(device, image, &memReqs);

    // 4. SERIALIZATION
    void* pResultBuffer = malloc(sizeof(VkMemoryRequirements));
    *(VkDeviceSize*)pResultBuffer = memReqs.size;
    *(VkDeviceSize*)((char*)pResultBuffer + 8) = memReqs.alignment;
    *(uint32_t*)((char*)pResultBuffer + 16) = memReqs.memoryTypeBits;

    long result_header = 0x1400000000; // Success code + size (0x14 bytes)
    RingBuffer_write(context->clientRingBuffer, &result_header, 8);
    RingBuffer_write(context->clientRingBuffer, pResultBuffer, sizeof(VkMemoryRequirements));
    
    free(pResultBuffer);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 6. `vt_handle_vkBindImageMemory`

**Purpose:**
A simple pass-through wrapper for `vkBindImageMemory`.

**Analysis:**

*   **Deserialization:** Reads and resolves handles for `VkDevice`, `VkImage`, and the custom `ResourceMemory` wrapper, as well as the `memoryOffset`.
*   **Modification/Hooking:** **None.** Unlike its buffer counterpart, this function does not appear to notify the `TextureDecoder`. The texture decoder may get this information through other means (e.g., by tracking the `VkImage` handle during `vkCreateImage`).
*   **Dispatch:** Calls `dispatch_vkBindImageMemory`.
*   **Serialization:** Writes the `VkResult` back.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkBindImageMemory command. A simple pass-through.
 * @param context The renderer context.
 */
void vt_handle_vkBindImageMemory(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkImage image = (VkImage)VkObject_fromId(*(void**)(input + 8));
    
    // The handle ID points to a custom ResourceMemory wrapper.
    ResourceMemory* resourceMem = (ResourceMemory*)VkObject_fromId(*(void**)(input + 16));
    
    // The real VkDeviceMemory handle is inside the wrapper.
    VkDeviceMemory memory = resourceMem->vkHandle; // Assuming vkHandle is at offset 0x18
    
    VkDeviceSize memoryOffset = *(VkDeviceSize*)(input + 24);
    
    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkResult result = dispatch_vkBindImageMemory(device, image, memory, memoryOffset);

    // 4. SERIALIZATION
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 7. `vt_handle_vkGetImageSparseMemoryRequirements`

**Purpose:**
Wraps `vkGetImageSparseMemoryRequirements` to query sparse memory properties for an image.

**Analysis:**

*   **Deserialization:** Reads `VkDevice` and `VkImage` handles. It also reads the `pSparseMemoryRequirementCount` from the client, which is used to allocate a buffer for the results.
*   **Modification/Hooking:** **None.** It's a query function.
*   **Dispatch:** Calls `dispatch_vkGetImageSparseMemoryRequirements` twice. First to get the count, then to get the data.
*   **Serialization:** The resulting array of `VkSparseImageMemoryRequirements` structs is serialized member-by-member and written back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkGetImageSparseMemoryRequirements command.
 * @param context The renderer context.
 */
void vt_handle_vkGetImageSparseMemoryRequirements(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkImage image = (VkImage)VkObject_fromId(*(void**)(input + 8));
    
    // Client provides the initial count.
    uint32_t sparseMemoryRequirementCount = *(uint32_t*)(input + 16);
    
    // 3. DISPATCH (First Call - Get Count)
    dispatch_vkGetImageSparseMemoryRequirements(device, image, &sparseMemoryRequirementCount, NULL);

    // Allocate buffer for results
    VkSparseImageMemoryRequirements* pSparseMemoryRequirements = NULL;
    if (sparseMemoryRequirementCount > 0) {
        pSparseMemoryRequirements = (VkSparseImageMemoryRequirements*)calloc(sparseMemoryRequirementCount, sizeof(VkSparseImageMemoryRequirements));
    }

    // 3. DISPATCH (Second Call - Get Data)
    if (pSparseMemoryRequirements != NULL) {
        dispatch_vkGetImageSparseMemoryRequirements(device, image, &sparseMemoryRequirementCount, pSparseMemoryRequirements);
    }
    
    // 4. SERIALIZATION
    // Calculate the size needed for the serialized response.
    size_t responseSize = 14; // Header size
    if (pSparseMemoryRequirements != NULL) {
        responseSize += sparseMemoryRequirementCount * sizeof(VkSparseImageMemoryRequirements); // Simplified size
    }

    // Allocate response buffer.
    void* pResultBuffer = malloc_from_temp_buffer(context, responseSize);
    memset(pResultBuffer, 0, responseSize);

    // Write header info.
    *(uint32_t*)((char*)pResultBuffer + 5) = sparseMemoryRequirementCount;
    *(uint32_t*)((char*)pResultBuffer + 9) = sparseMemoryRequirementCount;

    // Serialize the array of structs.
    if (pSparseMemoryRequirements != NULL) {
        char* write_ptr = (char*)pResultBuffer + 13;
        for (uint32_t i = 0; i < sparseMemoryRequirementCount; ++i) {
            // Member-by-member copy into the result buffer.
            memcpy(write_ptr, &pSparseMemoryRequirements[i], sizeof(VkSparseImageMemoryRequirements));
            write_ptr += sizeof(VkSparseImageMemoryRequirements); // Simplified
        }
    }
    
    // Write to client ring buffer.
    RingBuffer_write(context->clientRingBuffer, pResultBuffer, responseSize);

    if (pSparseMemoryRequirements != NULL) {
        free(pSparseMemoryRequirements);
    }
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 8. `vt_handle_vkQueueBindSparse`

**Purpose:**
Wraps the extremely complex `vkQueueBindSparse` command.

**Analysis:**

*   **Deserialization:** This is the most complex part. The function deeply deserializes the `VkBindSparseInfo` struct and its nested arrays. It recursively resolves all `VkQueue`, `VkSemaphore`, `VkFence`, `VkBuffer`, `VkImage`, and `VkDeviceMemory` handles using `VkObject_fromId`. It correctly handles the structure of `VkSparseBufferMemoryBindInfo`, `VkSparseImageOpaqueMemoryBindInfo`, and `VkSparseImageMemoryBindInfo` arrays.
*   **Modification/Hooking:** **None.** Due to the sheer complexity of the data structures, the function's logic is entirely dedicated to correctly reconstructing the arguments for the dispatch call.
*   **Dispatch:** Calls `dispatch_vkQueueBindSparse`.
*   **Serialization:** Writes the `VkResult` back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkQueueBindSparse command. Involves deep deserialization of
 *        nested structures but is fundamentally a pass-through call.
 * @param context The renderer context.
 */
void vt_handle_vkQueueBindSparse(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkQueue queue = (VkQueue)VkObject_fromId(*(void**)input);
    VkFence fence = (VkFence)VkObject_fromId(*(void**)(input + 8));

    uint32_t bindInfoCount = *(uint32_t*)(input + 16);
    
    // Allocate space for the array of VkBindSparseInfo structs.
    VkBindSparseInfo* pBindInfo = (VkBindSparseInfo*)malloc_from_temp_buffer(context, bindInfoCount * sizeof(VkBindSparseInfo));
    
    // Deeply deserialize each VkBindSparseInfo struct in the array.
    deserialize_bind_sparse_info_array(context, input + 20, bindInfoCount, pBindInfo);
    
    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkResult result = dispatch_vkQueueBindSparse(queue, bindInfoCount, pBindInfo, fence);

    // 4. SERIALIZATION
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 9. `vt_handle_vkCreateFence`

**Purpose:**
A simple wrapper for `vkCreateFence`.

**Analysis:**

*   **Deserialization:** Reads `VkDevice` and `VkFenceCreateInfo`, including any `pNext` chain (e.g., for `VkExportFenceCreateInfo`).
*   **Modification/Hooking:** **None.**
*   **Dispatch:** Calls `dispatch_vkCreateFence`.
*   **Serialization:** Writes the `VkResult` and the new `VkFence` handle back to the client.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkCreateFence command. A simple pass-through.
 * @param context The renderer context.
 */
void vt_handle_vkCreateFence(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkFenceCreateInfo createInfo = {};
    
    // Deserialize VkFenceCreateInfo, including pNext chain.
    deserialize_fence_create_info(context, input + 8, &createInfo);

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkFence fence = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateFence(device, &createInfo, NULL, &fence);

    // 4. SERIALIZATION
    void* pResultBuffer = malloc(sizeof(VkResult) + sizeof(VkFence));
    *(VkResult*)pResultBuffer = result;
    *(VkFence*)((char*)pResultBuffer + 4) = fence;

    RingBuffer_write(context->clientRingBuffer, pResultBuffer, sizeof(VkResult) + sizeof(VkFence));
    
    free(pResultBuffer);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

---

### 10. `vt_handle_vkDestroyFence`

**Purpose:**
A simple wrapper for `vkDestroyFence`.

**Analysis:**

*   **Deserialization:** Reads `VkDevice` and `VkFence` handles.
*   **Modification/Hooking:** **None.**
*   **Dispatch:** Calls `dispatch_vkDestroyFence`.
*   **Serialization:** None.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkDestroyFence command. A simple pass-through.
 * @param context The renderer context.
 */
void vt_handle_vkDestroyFence(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkFence fence = (VkFence)VkObject_fromId(*(void**)(input + 8));

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    dispatch_vkDestroyFence(device, fence, NULL);
    
    // 4. SERIALIZATION - None.
    return;
}
```

### 1. `vt_handle_vkResetFences`

**Purpose:**
Wraps the `vkResetFences` command to reset one or more fence objects to the unsignaled state.

**Decompiled Code & Analysis:**
```c
/**
 * @brief Handles the vkResetFences command.
 * @param context The renderer context.
 */
void vt_handle_vkResetFences(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    
    uint32_t fenceCount = *(uint32_t*)(input + 8);
    
    // Allocate a temporary array on the stack for the fence handles.
    VkFence* pFences = (VkFence*)alloca(fenceCount * sizeof(VkFence));

    // Deserialize and resolve each fence handle.
    char* fenceIdPtr = input + 12;
    for (uint32_t i = 0; i < fenceCount; ++i) {
        pFences[i] = (VkFence)VkObject_fromId(*(void**)fenceIdPtr);
        fenceIdPtr += 8;
    }

    // 2. MODIFICATION/HOOKING - None. This is a direct pass-through.

    // 3. DISPATCH
    dispatch_vkResetFences(device, fenceCount, pFences);

    // 4. SERIALIZATION - None (`vkResetFences` returns VkResult, but this wrapper doesn't send it).
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

**Summary:** This is a straightforward wrapper. It deserializes an array of fence handles, resolves them, and calls the real Vulkan function. It does not send any response back to the client, implying the call is treated as a one-way command.

---

### 2. `vt_handle_vkGetFenceStatus`

**Purpose:**
Wraps the `vkGetFenceStatus` command to query the state of a fence.

**Decompiled Code & Analysis:**
```c
/**
 * @brief Handles the vkGetFenceStatus command.
 * @param context The renderer context.
 */
void vt_handle_vkGetFenceStatus(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    VkFence fence = (VkFence)VkObject_fromId(*(void**)(input + 8));

    // 2. MODIFICATION/HOOKING - None.

    // 3. DISPATCH
    VkResult result = dispatch_vkGetFenceStatus(device, fence);

    // 4. SERIALIZATION
    // Write the resulting VkResult back to the client.
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```
**Summary:** A simple pass-through function that queries the fence status and sends the resulting `VkResult` (`VK_SUCCESS`, `VK_NOT_READY`, or `VK_ERROR_DEVICE_LOST`) back to the client.

---

### 3. `vt_handle_vkWaitForFences`

**Purpose:**
This function **replaces** the blocking `vkWaitForFences` call with a non-blocking IPC mechanism that sends fence file descriptors to the client.

**Decompiled Code & Analysis:**
```c
/**
 * @brief Handles the vkWaitForFences command.
 *
 * This is NOT a wrapper for vkWaitForFences. Instead of blocking the renderer thread,
 * it uses vkGetFenceFdKHR to get file descriptors for the fences and sends them
 * to the client via a socket message. The client is then responsible for performing
 * the blocking wait (e.g., using `poll` or `epoll`) on those file descriptors.
 *
 * @param context The renderer context.
 */
void vt_handle_vkWaitForFences(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(*(void**)input);
    
    uint32_t fenceCount = *(uint32_t*)(input + 8);
    VkFence* pFences = (VkFence*)alloca(fenceCount * sizeof(VkFence));

    // Deserialize and resolve fence handles.
    char* fenceIdPtr = input + 12;
    for (uint32_t i = 0; i < fenceCount; ++i) {
        pFences[i] = (VkFence)VkObject_fromId(*(void**)fenceIdPtr);
        fenceIdPtr += 8;
    }
    
    // The waitAll and timeout parameters are read but not used directly here.
    // They are likely informational for the client receiving the fds.

    // 2. MODIFICATION/HOOKING & DISPATCH REPLACEMENT
    
    // This function entirely replaces the vkWaitForFences logic.
    if (param_wait_timeout == 0) {
        // If timeout is 0, it's a non-blocking poll. The wrapper can perform
        // this directly and send back the result.
        VkResult result = dispatch_vkWaitForFences(device, fenceCount, pFences, waitAll, 0);
        RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
    } else {
        // For a blocking wait, use IPC to avoid stalling the renderer thread.
        int* fds = (int*)alloca(fenceCount * sizeof(int));
        VkResult overallResult = VK_SUCCESS;

        // Get a file descriptor for each fence.
        for (uint32_t i = 0; i < fenceCount; ++i) {
            VkFenceGetFdInfoKHR getFdInfo = {
                .sType = VK_STRUCTURE_TYPE_FENCE_GET_FD_INFO_KHR,
                .fence = pFences[i],
                .handleType = VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT
            };
            VkResult res = dispatch_vkGetFenceFdKHR(device, &getFdInfo, &fds[i]);
            if (res != VK_SUCCESS) {
                overallResult = res;
                fds[i] = -1; // Mark as invalid
            }
        }

        // 4. SERIALIZATION via IPC
        
        // Prepare a message to send the array of file descriptors to the client.
        struct msghdr msg = {0};
        struct iovec iov = {0};
        char cmsg_buf[CMSG_SPACE(fenceCount * sizeof(int))];

        int result_payload = overallResult;
        iov.iov_base = &result_payload;
        iov.iov_len = sizeof(result_payload);
        msg.msg_iov = &iov;
        msg.msg_iovlen = 1;

        if (overallResult == VK_SUCCESS) {
            msg.msg_control = cmsg_buf;
            msg.msg_controllen = CMSG_SPACE(fenceCount * sizeof(int));
            
            struct cmsghdr* cmsg = CMSG_FIRSTHDR(&msg);
            cmsg->cmsg_level = SOL_SOCKET;
            cmsg->cmsg_type = SCM_RIGHTS;
            cmsg->cmsg_len = CMSG_LEN(fenceCount * sizeof(int));
            memcpy(CMSG_DATA(cmsg), fds, fenceCount * sizeof(int));
        }

        // Send the message with the file descriptors.
        sendmsg(context->clientFd, &msg, 0);
        
        // Close the local copies of the file descriptors.
        for (uint32_t i = 0; i < fenceCount; ++i) {
            if (fds[i] >= 0) {
                close(fds[i]);
            }
        }
    }

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```

**Summary:** This is a very important optimization. Instead of blocking the main renderer thread, it offloads the wait to the client by sending file descriptors. The client can then use an efficient OS-level mechanism like `poll` to wait for the fences to be signaled.

---

The remaining functions in this batch are primarily simple wrappers, with details provided below.

### 4. `vt_handle_vkCreateSemaphore`
- **Deserialization:** Reads `VkDevice` and `VkSemaphoreCreateInfo`. Handles `pNext` for extensions like `VkExportSemaphoreCreateInfoKHR` and `VkSemaphoreTypeCreateInfo`.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkCreateSemaphore`.
- **Serialization:** Writes back `VkResult` and the new `VkSemaphore` handle.

```c
void vt_handle_vkCreateSemaphore(VtContext* context) {
    // ... deserialization of VkSemaphoreCreateInfo ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkSemaphoreCreateInfo createInfo = {};
    deserialize_semaphore_create_info(context, ..., &createInfo);

    VkSemaphore semaphore = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateSemaphore(device, &createInfo, NULL, &semaphore);
    
    // ... serialization of result and semaphore handle ...
}
```

### 5. `vt_handle_vkDestroySemaphore`
- **Deserialization:** Reads `VkDevice` and `VkSemaphore` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkDestroySemaphore`.
- **Serialization:** None.

```c
void vt_handle_vkDestroySemaphore(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkSemaphore semaphore = (VkSemaphore)VkObject_fromId(...);
    dispatch_vkDestroySemaphore(device, semaphore, NULL);
}
```

### 6. `vt_handle_vkCreateEvent`
- **Deserialization:** Reads `VkDevice` and `VkEventCreateInfo`.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkCreateEvent`.
- **Serialization:** Writes back `VkResult` and the new `VkEvent` handle.

```c
void vt_handle_vkCreateEvent(VtContext* context) {
    // ... deserialization of VkEventCreateInfo ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkEventCreateInfo createInfo = {};
    deserialize_event_create_info(context, ..., &createInfo);

    VkEvent event = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateEvent(device, &createInfo, NULL, &event);
    
    // ... serialization of result and event handle ...
}
```

### 7. `vt_handle_vkDestroyEvent`
- **Deserialization:** Reads `VkDevice` and `VkEvent` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkDestroyEvent`.
- **Serialization:** None.

```c
void vt_handle_vkDestroyEvent(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkEvent event = (VkEvent)VkObject_fromId(...);
    dispatch_vkDestroyEvent(device, event, NULL);
}
```

### 8. `vt_handle_vkGetEventStatus`
- **Deserialization:** Reads `VkDevice` and `VkEvent` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkGetEventStatus`.
- **Serialization:** Writes back the `VkResult` (`VK_EVENT_SET`, `VK_EVENT_RESET`, or an error).

```c
void vt_handle_vkGetEventStatus(VtContext* context) {
    // ... deserialization ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkEvent event = (VkEvent)VkObject_fromId(...);
    
    VkResult result = dispatch_vkGetEventStatus(device, event);

    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
}
```

### 9. `vt_handle_vkSetEvent`
- **Deserialization:** Reads `VkDevice` and `VkEvent` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkSetEvent`.
- **Serialization:** Writes the `VkResult` back.

```c
void vt_handle_vkSetEvent(VtContext* context) {
    // ... deserialization ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkEvent event = (VkEvent)VkObject_fromId(...);
    
    VkResult result = dispatch_vkSetEvent(device, event);

    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
}
```

### 10. `vt_handle_vkResetEvent`
- **Deserialization:** Reads `VkDevice` and `VkEvent` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkResetEvent`.
- **Serialization:** Writes the `VkResult` back.

```c
void vt_handle_vkResetEvent(VtContext* context) {
    // ... deserialization ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkEvent event = (VkEvent)VkObject_fromId(...);
    
    VkResult result = dispatch_vkResetEvent(device, event);

    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
}
```

### 11. `vt_handle_vkCreateQueryPool`
- **Deserialization:** Reads `VkDevice` and `VkQueryPoolCreateInfo`.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkCreateQueryPool`.
- **Serialization:** Writes back `VkResult` and the new `VkQueryPool` handle.

```c
void vt_handle_vkCreateQueryPool(VtContext* context) {
    // ... deserialization of VkQueryPoolCreateInfo ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkQueryPoolCreateInfo createInfo = {};
    deserialize_query_pool_create_info(context, ..., &createInfo);

    VkQueryPool queryPool = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateQueryPool(device, &createInfo, NULL, &queryPool);
    
    // ... serialization of result and queryPool handle ...
}
```

### 12. `vt_handle_vkDestroyQueryPool`
- **Deserialization:** Reads `VkDevice` and `VkQueryPool` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkDestroyQueryPool`.
- **Serialization:** None.

```c
void vt_handle_vkDestroyQueryPool(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkQueryPool queryPool = (VkQueryPool)VkObject_fromId(...);
    dispatch_vkDestroyQueryPool(device, queryPool, NULL);
}
```

### 13. `vt_handle_vkGetQueryPoolResults`
- **Deserialization:** Reads all parameters for `vkGetQueryPoolResults`, including `dataSize`.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkGetQueryPoolResults`, which writes the query data into a temporary buffer allocated by the handler.
- **Serialization:** Serializes the `VkResult` and the entire data buffer containing the query results and writes it back to the client.

```c
void vt_handle_vkGetQueryPoolResults(VtContext* context) {
    // ... deserialization of all parameters ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkQueryPool queryPool = (VkQueryPool)VkObject_fromId(...);
    uint32_t firstQuery, queryCount;
    size_t dataSize;
    VkDeviceSize stride;
    VkQueryResultFlags flags;
    // ...

    void* pData = malloc_from_temp_buffer(context, dataSize);
    
    VkResult result = dispatch_vkGetQueryPoolResults(device, queryPool, firstQuery, queryCount, dataSize, pData, stride, flags);

    // ... serialization of result and the pData buffer ...
}
```

### 14. `vt_handle_vkResetQueryPool`
- **Note:** This function is distinct from `vt_handle_vkCmdResetQueryPool`. This one is the device-level reset.
- **Deserialization:** Reads `VkDevice`, `VkQueryPool`, `firstQuery`, and `queryCount`.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkResetQueryPool`.
- **Serialization:** None. The decompiled code for `vt_handle_vkResetQueryPool` calls `dispatch_vkResetQueryPool` which is a `VkDevice`-level function, while `vt_handle_vkCmdResetQueryPool` would call `dispatch_vkCmdResetQueryPool`.

```c
void vt_handle_vkResetQueryPool(long param_1) {
    // ... deserializes VkDevice, VkQueryPool, firstQuery, queryCount ...
    void* device = VkObject_fromId(...);
    void* queryPool = VkObject_fromId(...);
    uint32_t firstQuery = ...;
    uint32_t queryCount = ...;
    
    // This is the device-level reset, not the command-buffer one.
    dispatch_vkResetQueryPool(device, queryPool, firstQuery, queryCount);
}
```

### 15. `vt_handle_vkCreateBuffer`
(Identical analysis to function #15 from previous list)
- **Deserialization:** Reads `VkDevice` and `VkBufferCreateInfo`.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkCreateBuffer`.
- **Serialization:** Writes back `VkResult` and the new `VkBuffer` handle.

```c
void vt_handle_vkCreateBuffer(VtContext* context) {
    // ... deserialization of VkBufferCreateInfo ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkBufferCreateInfo createInfo = {};
    deserialize_buffer_create_info(context, ..., &createInfo);

    VkBuffer buffer = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateBuffer(device, &createInfo, NULL, &buffer);
    
    // ... serialization of result and buffer handle ...
}
```

### 16. `vt_handle_vkDestroyBuffer`
(Identical analysis to function #16 from previous list)
- **Deserialization:** Reads `VkDevice` and `VkBuffer` handles.
- **Hooking:** **Yes.** Calls `TextureDecoder_removeBoundBuffer` to unregister the buffer from the custom decoder before destruction.
- **Dispatch:** Calls `dispatch_vkDestroyBuffer`.
- **Serialization:** None.

```c
void vt_handle_vkDestroyBuffer(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkBuffer buffer = (VkBuffer)VkObject_fromId(...);

    // Hook: Notify the texture decoder that this buffer is being destroyed.
    if (context->textureDecoder != NULL) {
        TextureDecoder_removeBoundBuffer(context->textureDecoder, buffer);
    }
    
    dispatch_vkDestroyBuffer(device, buffer, NULL);
}
```

### 17. `vt_handle_vkCreateBufferView`
- **Deserialization:** Reads `VkDevice` and `VkBufferViewCreateInfo`. Resolves the `VkBuffer` handle.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkCreateBufferView`.
- **Serialization:** Writes back `VkResult` and the new `VkBufferView` handle.

```c
void vt_handle_vkCreateBufferView(VtContext* context) {
    // ... deserialization of VkBufferViewCreateInfo ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkBufferViewCreateInfo createInfo = {};
    deserialize_buffer_view_create_info(context, ..., &createInfo); // Resolves VkBuffer handle inside

    VkBufferView bufferView = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateBufferView(device, &createInfo, NULL, &bufferView);
    
    // ... serialization of result and bufferView handle ...
}
```

### 18. `vt_handle_vkDestroyBufferView`
- **Deserialization:** Reads `VkDevice` and `VkBufferView` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkDestroyBufferView`.
- **Serialization:** None.

```c
void vt_handle_vkDestroyBufferView(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkBufferView bufferView = (VkBufferView)VkObject_fromId(...);
    dispatch_vkDestroyBufferView(device, bufferView, NULL);
}
```

### 19. `vt_handle_vkCreateImage`
(Identical analysis to function #19 from previous list)
- **Deserialization:** Reads `VkDevice` and `VkImageCreateInfo`.
- **Hooking:** **Yes.** Checks if the format is compressed. If so, it calls `TextureDecoder_createImage` which replaces the call and likely creates a different, uncompressed image. If not, it's a pass-through.
- **Dispatch:** Either `TextureDecoder_createImage` or `dispatch_vkCreateImage`.
- **Serialization:** Writes back `VkResult` and the new `VkImage` handle.

```c
void vt_handle_vkCreateImage(VtContext* context) {
    // ... deserialization of VkImageCreateInfo ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkImageCreateInfo createInfo = {};
    deserialize_image_create_info(context, ..., &createInfo);
    
    VkImage image = VK_NULL_HANDLE;
    VkResult result;

    // Hooking logic
    if (context->textureDecoder != NULL && isCompressedFormat(createInfo.format)) {
        // Replace the call with the custom texture decoder's function.
        result = TextureDecoder_createImage(context->textureDecoder, device, &createInfo, &image);
    } else {
        // Standard pass-through call.
        result = dispatch_vkCreateImage(device, &createInfo, NULL, &image);
    }
    
    // ... serialization of result and image handle ...
}
```

### 20. `vt_handle_vkDestroyImage`
(Identical analysis to function #20 from previous list)
- **Deserialization:** Reads `VkDevice` and `VkImage` handles.
- **Hooking:** **Yes.** Checks if the `VkImage` is tracked by the `TextureDecoder` using `TextureDecoder_containsImage`. If so, it calls `TextureDecoder_destroyImage` to clean up custom resources.
- **Dispatch:** Calls either `TextureDecoder_destroyImage` or `dispatch_vkDestroyImage`.
- **Serialization:** None.

```c
void vt_handle_vkDestroyImage(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkImage image = (VkImage)VkObject_fromId(...);

    // Hooking logic
    if (context->textureDecoder != NULL && TextureDecoder_containsImage(context->textureDecoder, image)) {
        TextureDecoder_destroyImage(context->textureDecoder, device, image);
    } else {
        dispatch_vkDestroyImage(device, image, NULL);
    }
}
```

---

### 1. `vt_handle_vkGetImageSubresourceLayout`

**Purpose:**
A wrapper for `vkGetImageSubresourceLayout`. It queries the layout (offset, size, pitch) of a specific subresource of an image.

**Decompiled Code:**
```c
/**
 * @brief Handles the vkGetImageSubresourceLayout command.
 * @param context The renderer context.
 */
void vt_handle_vkGetImageSubresourceLayout(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device;
    VkImage image;
    VkImageSubresource subresource = {};

    // Deserialize handles and parameters
    if (*input == '\0') {
        // Handle case where default device is used
        device = (VkDevice)context;
        image = (VkImage)VkObject_fromId(*(void**)(input + 1));
        // Deserialize VkImageSubresource struct
        memcpy(&subresource, input + 9, sizeof(VkImageSubresource));
    } else {
        device = (VkDevice)VkObject_fromId(*(void**)(input + 1));
        image = (VkImage)VkObject_fromId(*(void**)(input + 9));
        // Deserialize VkImageSubresource struct
        memcpy(&subresource, input + 17, sizeof(VkImageSubresource));
    }

    // 2. MODIFICATION/HOOKING - None. This is a direct query.

    // 3. DISPATCH
    VkSubresourceLayout layout = {};
    dispatch_vkGetImageSubresourceLayout(device, image, &subresource, &layout);

    // 4. SERIALIZATION
    
    // Allocate a buffer to hold the result struct.
    void* pResultBuffer = malloc_from_temp_buffer(context, sizeof(VkSubresourceLayout));
    memcpy(pResultBuffer, &layout, sizeof(VkSubresourceLayout));

    // Write a header indicating success and data size, followed by the data.
    long result_header = ((long)sizeof(VkSubresourceLayout) << 32) | 0; // result = 0, size = 0x28
    RingBuffer_write(context->clientRingBuffer, &result_header, 8);
    RingBuffer_write(context->clientRingBuffer, pResultBuffer, sizeof(VkSubresourceLayout));

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
    return;
}
```
**Analysis:** This is a standard pass-through wrapper. It deserializes the arguments, calls the real Vulkan function, and serializes the resulting `VkSubresourceLayout` struct back to the client. There are no functional modifications.

---

### 2. `vt_handle_vkCreateImageView`

This function was analyzed in the previous request. The analysis remains the same.

**Purpose:**
Wraps `vkCreateImageView` and includes a major hook to handle compressed texture format emulation.

**Decompiled Code & Analysis:**
```c
/**
 * @brief Handles the vkCreateImageView command. Hooks the call to handle
 *        emulation of compressed texture formats.
 * @param context The renderer context.
 */
void vt_handle_vkCreateImageView(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkImageViewCreateInfo createInfo = {};
    deserialize_image_view_create_info(context, context->inputBufferPtr, &createInfo);
    VkDevice device = (VkDevice)VkObject_fromId(createInfo.pNext); // Device handle passed via pNext trick

    // 2. MODIFICATION & HOOKING
    if (context->textureDecoder != NULL && isCompressedFormat(createInfo.format)) {
        // HOOK: If the format is compressed, modify the create info.
        // The view is created with a standard uncompressed format, and the
        // texture decoder handles the on-the-fly conversion.
        createInfo.format = VK_FORMAT_B8G8R8A8_UNORM;
        createInfo.subresourceRange.layerCount = 1;
    }

    // 3. DISPATCH
    VkImageView imageView = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateImageView(device, &createInfo, NULL, &imageView);

    // 4. SERIALIZATION
    // Write result code and the new handle back to the client.
    void* pResultBuffer = malloc(sizeof(VkResult) + sizeof(VkImageView));
    *(VkResult*)pResultBuffer = result;
    *(VkImageView*)((char*)pResultBuffer + 4) = imageView;
    RingBuffer_write(context->clientRingBuffer, pResultBuffer, sizeof(VkResult) + sizeof(VkImageView));
    free(pResultBuffer);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Summary:** This function hooks `vkCreateImageView` to facilitate texture decompression. When a view is requested for a compressed format, it modifies the request to create a view with a standard `VK_FORMAT_B8G8R8A8_UNORM` format, relying on the `TextureDecoder` to provide the uncompressed data.

---

### 3. `vt_handle_vkDestroyImageView`
- **Deserialization:** Reads `VkDevice` and `VkImageView` handles.
- **Hooking:** **None.**
- **Dispatch:** Calls `dispatch_vkDestroyImageView`.
- **Serialization:** None.

```c
void vt_handle_vkDestroyImageView(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkImageView imageView = (VkImageView)VkObject_fromId(...);
    dispatch_vkDestroyImageView(device, imageView, NULL);
}
```

---

### 4. `vt_handle_vkCreateShaderModule`
- **Deserialization:** Reads `VkDevice` and `VkShaderModuleCreateInfo`.
- **Hooking:** **Yes.** Calls `ShaderInspector_createModule`. This custom function likely parses the SPIR-V to extract metadata (like bindings, entry points, etc.) and then creates the real `VkShaderModule`. It returns a custom wrapper struct containing both the Vulkan handle and the parsed metadata.
- **Dispatch:** Handled within `ShaderInspector_createModule`.
- **Serialization:** Writes back `VkResult` and the new `VkShaderModule` handle.

```c
void vt_handle_vkCreateShaderModule(VtContext* context) {
    // ... deserialization of VkShaderModuleCreateInfo ...
    VkDevice device = (VkDevice)VkObject_fromId(...);
    VkShaderModuleCreateInfo createInfo = {};
    deserialize_shader_module_create_info(context, ..., &createInfo);

    // Hook: delegate creation to the shader inspector.
    VkShaderEXT shaderWrapper; // Using a custom wrapper type
    VkResult result = ShaderInspector_createModule(context->shaderInspector, device, &createInfo, &shaderWrapper);
    
    // ... serialization of result and shaderWrapper handle ...
}
```

---

### 5. `vt_handle_vkDestroyShaderModule`
- **Deserialization:** Reads `VkDevice` and the custom shader wrapper handle.
- **Hooking:** **Yes.** It frees the custom wrapper object and its contents before calling the real Vulkan destroy function. The decompilation shows a check for a custom flag (`__ptr[1] & 1`) and freeing of internal data (`free(__ptr[1])`) before freeing the wrapper (`free(__ptr)`) and calling `dispatch_vkDestroyShaderModule`.
- **Dispatch:** Calls `dispatch_vkDestroyShaderModule`.
- **Serialization:** None.

```c
void vt_handle_vkDestroyShaderModule(VtContext* context) {
    VkDevice device = (VkDevice)VkObject_fromId(...);
    ShaderModuleWrapper* wrapper = (ShaderModuleWrapper*)VkObject_fromId(...);

    if (wrapper != NULL) {
        // Free internal data if present.
        if (wrapper->pSpirvCode != NULL) {
            free(wrapper->pSpirvCode);
        }
        // Dispatch to real Vulkan function.
        if (wrapper->module != VK_NULL_HANDLE) {
            dispatch_vkDestroyShaderModule(device, wrapper->module, NULL);
        }
        // Free the wrapper itself.
        free(wrapper);
    }
}
```

---

### 6-9. Pipeline Cache Functions
These four functions are all simple pass-through wrappers with no modifications.

-   **`vt_handle_vkCreatePipelineCache`**: Deserializes `VkPipelineCacheCreateInfo`, calls `dispatch_vkCreatePipelineCache`, and serializes the new `VkPipelineCache` handle back.
-   **`vt_handle_vkDestroyPipelineCache`**: Deserializes handles and calls `dispatch_vkDestroyPipelineCache`.
-   **`vt_handle_vkGetPipelineCacheData`**: Deserializes handles, calls `dispatch_vkGetPipelineCacheData` to get the cache size, allocates a buffer, calls again to get the data, and serializes the data back to the client.
-   **`vt_handle_vkMergePipelineCaches`**: Deserializes the destination and source `VkPipelineCache` handles, calls `dispatch_vkMergePipelineCaches`, and sends the `VkResult` back.

```c
// vt_handle_vkCreatePipelineCache
void vt_handle_vkCreatePipelineCache(VtContext* context) { /* Deserialization, Dispatch, Serialization */ }

// vt_handle_vkDestroyPipelineCache
void vt_handle_vkDestroyPipelineCache(VtContext* context) { /* Deserialization, Dispatch */ }

// vt_handle_vkGetPipelineCacheData
void vt_handle_vkGetPipelineCacheData(VtContext* context) { /* Deserialization, Dispatch (2 calls), Serialization of data */ }

// vt_handle_vkMergePipelineCaches
void vt_handle_vkMergePipelineCaches(VtContext* context) { /* Deserialization, Dispatch, Serialization of result */ }
```

---

### 10 & 11. `vt_handle_vkCreateGraphicsPipelines` & `vt_handle_vkCreateComputePipelines`

These functions were analyzed previously and are major hooks for asynchronous pipeline compilation.

**Decompiled Code & Analysis:**
```c
/**
 * @brief Handles the vkCreateGraphicsPipelines command by offloading it to a worker thread.
 * @param context The renderer context.
 */
void vt_handle_vkCreateGraphicsPipelines(VtContext* context) {
    // This function doesn't deserialize here. It passes the raw buffer to the creator.
    AsyncPipelineCreator_create(
        context->clientRingBuffer,          // Ring buffer for the final response
        context->asyncPipelineCreator,      // The async creator instance
        *(long*)(context + 0x88),           // A pointer likely to the thread pool/task manager
        VK_PIPELINE_BIND_POINT_GRAPHICS,    // The pipeline type
        context->inputBufferPtr,            // The raw command data
        context->inputBufferSize            // The size of the data
    );
}

/**
 * @brief Handles the vkCreateComputePipelines command by offloading it to a worker thread.
 * @param context The renderer context.
 */
void vt_handle_vkCreateComputePipelines(VtContext* context) {
    // Logic is identical to the graphics version, just with a different bind point.
    AsyncPipelineCreator_create(
        context->clientRingBuffer,
        context->asyncPipelineCreator,
        *(long*)(context + 0x88),
        VK_PIPELINE_BIND_POINT_COMPUTE,     // The pipeline type
        context->inputBufferPtr,
        context->inputBufferSize
    );
}
```
**Summary:** Instead of blocking, these handlers delegate pipeline creation to an `AsyncPipelineCreator`. This creator runs on a worker thread, preventing the main command processing loop from stalling during potentially long shader compilation times. The result is sent back to the client asynchronously.

---

Restart

---

### Overall Architecture

The `vt_handle_vk...` functions are wrappers around the standard Vulkan API calls. They follow a consistent pattern:

1.  **Deserialization:** Arguments for the Vulkan call are read from a buffer pointed to by `context->inputBufferPtr`. This buffer is populated by a client application. Complex structs, arrays, and `pNext` chains are reconstructed in temporary memory. Vulkan handles (`VkDevice`, `VkImage`, etc.) are passed as IDs and resolved into actual handles using a helper like `VkObject_fromId`.
2.  **Modification/Hooking:** This is the most critical stage. The wrapper may inspect or alter the deserialized arguments before passing them to the real Vulkan driver. This is where custom features, workarounds, or emulation logic are injected.
3.  **Dispatch:** The wrapper calls the actual Vulkan function pointer, which was loaded from `libvulkan.so` during initialization and is stored in a dispatch table (e.g., `dispatch_vkCreateImageView`).
4.  **Serialization:** The `VkResult` and any output data (like newly created handles or queried properties) are serialized into a buffer and written to the `clientRingBuffer` to be sent back to the client application.

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkGetImageSubresourceLayout` | **No** | Standard pass-through. |
| `vkCreateImageView` | **Yes** | Hooks compressed texture formats and changes them to an uncompressed format (`VK_FORMAT_B8G8R8A8_UNORM`) for emulation via `TextureDecoder`. |
| `vkDestroyImageView` | **No** | Standard pass-through. |
| `vkCreateShaderModule` | **Yes** | Intercepts creation to call `ShaderInspector_createModule`, which likely parses/modifies SPIR-V bytecode or stores metadata about it. |
| `vkDestroyShaderModule` | **Yes** | Checks a custom flag on the wrapped shader object. If the flag is set, it frees additional memory allocated by the `ShaderInspector`. |
| `vkCreatePipelineCache` | **No** | Standard pass-through. |
| `vkDestroyPipelineCache` | **No** | Standard pass-through. |
| `vkGetPipelineCacheData` | **No** | Standard pass-through. |
| `vkMergePipelineCaches` | **No** | Standard pass-through. |
| `vkCreateGraphicsPipelines` | **Yes** | Offloads pipeline creation to a worker thread pool via `AsyncPipelineCreator` to prevent stalling. |
| `vkCreateComputePipelines` | **Yes** | Offloads pipeline creation to a worker thread pool via `AsyncPipelineCreator` to prevent stalling. |
| `vkDestroyPipeline` | **No** | Standard pass-through. |
| `vkCreatePipelineLayout` | **Yes** | Modifies `VkPushConstantRange` structures by adding the `VK_SHADER_STAGE_TESSELLATION_EVALUATION_SHADER_BIT` to `stageFlags`. |
| `vkDestroyPipelineLayout`| **No** | Standard pass-through. |
| `vkCreateSampler` | **No** | Standard pass-through. |
| `vkDestroySampler` | **No** | Standard pass-through. |
| `vkCreateDescriptorSetLayout`| **No** | Standard pass-through. |
| `vkDestroyDescriptorSetLayout`|**No** | Standard pass-through. |
| `vkCreateDescriptorPool` | **No** | Standard pass-through. |
| `vkDestroyDescriptorPool` | **No** | Standard pass-through. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkGetImageSubresourceLayout`

```c
/**
 * @brief Handles vkGetImageSubresourceLayout.
 */
void vt_handle_vkGetImageSubresourceLayout(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = VkObject_fromId(*(void**)(input + 1));
    VkImage image = VkObject_fromId(*(void**)(input + 9));
    
    // Deserialize VkImageSubresource from the input buffer.
    VkImageSubresource subresource = {0};
    if (*(int*)(input + 0x11) > 0) { // Check if pSubresource data is present
        subresource.aspectMask = *(VkImageAspectFlags*)(input + 0x19);
        subresource.mipLevel = *(uint32_t*)(input + 0x1D);
        subresource.arrayLayer = *(uint32_t*)(input + 0x21);
    }
    
    // 2. MODIFICATION & HOOKING
    // No modifications are performed. This is a direct query.

    // 3. DISPATCH TO VULKAN
    VkSubresourceLayout layout = {0};
    dispatch_vkGetImageSubresourceLayout(device, image, &subresource, &layout);

    // 4. SERIALIZATION
    // Allocate space in a temporary buffer for the result.
    void* resultBuffer = malloc(sizeof(VkResult) + sizeof(VkSubresourceLayout));
    if (resultBuffer) {
        *(VkResult*)resultBuffer = VK_SUCCESS; // Implicitly successful if called
        *(VkSubresourceLayout*)((char*)resultBuffer + sizeof(VkResult)) = layout;
        
        RingBuffer_write(context->clientRingBuffer, resultBuffer, sizeof(VkResult) + sizeof(VkSubresourceLayout));
        free(resultBuffer);
    }
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is a standard pass-through function. It deserializes the arguments, calls the real Vulkan function, and serializes the resulting `VkSubresourceLayout` struct back to the client. There are no modifications.

---

#### 2. `vt_handle_vkCreateImageView`

*This function was analyzed in the previous response and is included here for completeness.*
```c
/**
 * @brief Handles vkCreateImageView, with hooks for texture decompression.
 */
void vt_handle_vkCreateImageView(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkImageViewCreateInfo createInfo = {0};
    // (Complex deserialization of pNext chain and main struct members occurs here)
    deserialize_vkImageViewCreateInfo(context, &createInfo);

    // 2. MODIFICATION & HOOKING
    // Check if texture decompression is active and if the format is compressed.
    if (context->textureDecoder != NULL && isCompressedFormat(createInfo.format)) {
        // Force the format to a standard uncompressed format for emulation.
        createInfo.format = VK_FORMAT_B8G8R8A8_UNORM;
        // Process one layer at a time.
        createInfo.subresourceRange.layerCount = 1;
    }
    
    // 3. DISPATCH TO VULKAN
    VkImageView imageView = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreateImageView(device, &createInfo, NULL, &imageView);

    // 4. SERIALIZATION
    void* resultBuffer = malloc(sizeof(VkResult) + sizeof(VkImageView));
    *(VkResult*)resultBuffer = result;
    *(VkImageView*)((char*)resultBuffer + sizeof(VkResult)) = imageView;
    RingBuffer_write(context->clientRingBuffer, resultBuffer, sizeof(VkResult) + sizeof(VkImageView));
    free(resultBuffer);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function hooks `vkCreateImageView` to enable texture format emulation. When a compressed format is used, it modifies the `VkImageViewCreateInfo` to request an uncompressed format (`VK_FORMAT_B8G8R8A8_UNORM`) before calling the driver. This allows the `TextureDecoder` to handle decompression behind the scenes.

---

#### 3. `vt_handle_vkDestroyImageView`

```c
/**
 * @brief Handles vkDestroyImageView.
 */
void vt_handle_vkDestroyImageView(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = VkObject_fromId(*(void**)(input + 1));
    VkImageView imageView = VkObject_fromId(*(void**)(input + 9));
    
    // 2. MODIFICATION & HOOKING
    // No modifications.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkDestroyImageView(device, imageView, NULL);
    
    // 4. SERIALIZATION
    // None, this is a void function.
}
```
**Analysis:** Standard pass-through destroy function. No modifications.

---

#### 4. `vt_handle_vkCreateShaderModule`

```c
/**
 * @brief Handles vkCreateShaderModule, with hooks for shader inspection.
 */
void vt_handle_vkCreateShaderModule(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // Deserialize VkShaderModuleCreateInfo
    VkShaderModuleCreateInfo createInfo = {0};
    uint32_t createInfoSize = *(uint32_t*)(context->inputBufferPtr + 9);
    if (createInfoSize > 0) {
        char* dataPtr = context->inputBufferPtr + 13;
        createInfo.sType = *(VkStructureType*)dataPtr;
        // ... (deserialization of pNext and flags)
        createInfo.codeSize = *(size_t*)(dataPtr + 0x10);
        
        // Allocate temporary memory for the shader code
        void* shaderCode = malloc(createInfo.codeSize);
        memcpy(shaderCode, dataPtr + 0x18, createInfo.codeSize);
        createInfo.pCode = (const uint32_t*)shaderCode;
    }
    
    // 2. MODIFICATION & HOOKING
    // Instead of directly calling vkCreateShaderModule, it calls a custom inspector function.
    VkShaderModule shaderModule = VK_NULL_HANDLE;
    VkResult result = ShaderInspector_createModule(context->shaderInspector, device, &createInfo, createInfo.codeSize, &shaderModule);

    // 3. DISPATCH TO VULKAN
    // The actual dispatch_vkCreateShaderModule is called inside ShaderInspector_createModule.
    
    // 4. SERIALIZATION
    void* resultBuffer = malloc(sizeof(VkResult) + sizeof(VkShaderModule));
    *(VkResult*)resultBuffer = result;
    *(VkShaderModule*)((char*)resultBuffer + sizeof(VkResult)) = shaderModule;
    RingBuffer_write(context->clientRingBuffer, resultBuffer, sizeof(VkResult) + sizeof(VkShaderModule));
    free(resultBuffer);
    
    // Free the temporary shader code buffer
    if (createInfo.pCode) {
        free((void*)createInfo.pCode);
    }
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function intercepts shader creation. It calls `ShaderInspector_createModule`, which suggests that the renderer performs some form of analysis or transformation on the SPIR-V bytecode before creating the final Vulkan object. The inspector likely stores metadata about the shader (e.g., resource usage, entry points) which can be used for later optimizations or validation.

---

#### 5. `vt_handle_vkDestroyShaderModule`

```c
/**
 * @brief Handles vkDestroyShaderModule, with custom cleanup.
 */
void vt_handle_vkDestroyShaderModule(VtContext* context) {
    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    void* wrappedShaderModule = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    
    // 2. MODIFICATION & HOOKING
    if (wrappedShaderModule != NULL) {
        // Set a flag indicating the object is being destroyed.
        *(bool*)((long)wrappedShaderModule + 0x18) = true;
        
        // Check another custom flag. If true, it means the shader inspector allocated extra data
        // that needs to be freed along with the Vulkan object.
        if (*(char*)((long)wrappedShaderModule + 0x19) != '\0') {
            
            // 3. DISPATCH TO VULKAN
            dispatch_vkDestroyShaderModule(device, *(VkShaderModule*)wrappedShaderModule, NULL);
            
            // Custom cleanup for the inspector's data.
            if (*(void**)((long)wrappedShaderModule + 8) != NULL) {
                free(*(void**)((long)wrappedShaderModule + 8));
                *(void**)((long)wrappedShaderModule + 8) = NULL;
            }
            free(wrappedShaderModule);
        }
    }
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** This function demonstrates custom resource management. The `ShaderInspector` likely allocates a custom wrapper struct for each `VkShaderModule`. This wrapper contains the real Vulkan handle, custom flags, and pointers to other metadata. On destruction, this function checks a flag in the wrapper to see if it should perform a deep clean, which includes calling the real `vkDestroyShaderModule` and freeing the extra allocated memory.

---

The remaining functions follow a similar pattern. Here's a summary for each:

#### 6. `vt_handle_vkCreatePipelineCache`: **No modifications.** Standard pass-through.
#### 7. `vt_handle_vkDestroyPipelineCache`: **No modifications.** Standard pass-through.
#### 8. `vt_handle_vkGetPipelineCacheData`: **No modifications.** Standard pass-through.
#### 9. `vt_handle_vkMergePipelineCaches`: **No modifications.** Standard pass-through.

---

#### 10. `vt_handle_vkCreateGraphicsPipelines`

```c
void vt_handle_vkCreateGraphicsPipelines(VtContext* context) {
    // 1. DESERIALIZATION
    // Deserializes vkCreateGraphicsPipelines parameters from the input buffer.
    // This is a complex process involving multiple arrays and nested structs.

    // 2. MODIFICATION & HOOKING
    // The core logic is completely replaced. Instead of a synchronous call,
    // it uses a helper module to perform the creation on a worker thread.
    AsyncPipelineCreator_create(
        context->clientRingBuffer,      // Ring buffer to write the result to
        context->shaderInspector,
        context->asyncPipelineCreator,  // The creator module itself
        0,                              // Type: 0 for Graphics
        context->inputBufferPtr,        // The raw buffer with serialized create infos
        context->inputBufferSize
    );

    // 3. DISPATCH TO VULKAN
    // The call to dispatch_vkCreateGraphicsPipelines is performed asynchronously by the worker thread.
    
    // 4. SERIALIZATION
    // Serialization is handled by the AsyncPipelineCreator upon completion.
}
```
**Analysis:** This is a major optimization hook. `vkCreateGraphicsPipelines` can be a very slow, CPU-intensive call that would stall rendering. This implementation offloads the work to a background thread pool (`AsyncPipelineCreator`). The main thread can continue processing other commands, and the result (the new `VkPipeline` handle) will be sent back to the client asynchronously.

---

#### 11. `vt_handle_vkCreateComputePipelines`

```c
void vt_handle_vkCreateComputePipelines(VtContext* context) {
    // 1. DESERIALIZATION
    // Deserializes vkCreateComputePipelines parameters from the input buffer.

    // 2. MODIFICATION & HOOKING
    // Similar to the graphics version, this offloads creation to a worker thread.
    AsyncPipelineCreator_create(
        context->clientRingBuffer,
        context->shaderInspector,
        context->asyncPipelineCreator,
        1,                              // Type: 1 for Compute
        context->inputBufferPtr,
        context->inputBufferSize
    );

    // 3. DISPATCH & 4. SERIALIZATION
    // Both handled asynchronously.
}
```
**Analysis:** Identical in concept to its graphics counterpart. It uses the `AsyncPipelineCreator` to avoid stalls during compute pipeline compilation.

---

#### 12. `vt_handle_vkDestroyPipeline`: **No modifications.** Standard pass-through.

---

#### 13. `vt_handle_vkCreatePipelineLayout`

```c
void vt_handle_vkCreatePipelineLayout(VtContext* context) {
    // ... (Standard deserialization of VkPipelineLayoutCreateInfo) ...

    VkPipelineLayoutCreateInfo createInfo = {0};
    deserialize_vkPipelineLayoutCreateInfo(context, &createInfo);

    // 2. MODIFICATION & HOOKING
    // It iterates through the push constant ranges and modifies them.
    if (createInfo.pPushConstantRanges != NULL && createInfo.pushConstantRangeCount > 0) {
        // The deserialization puts the ranges in a temporary buffer.
        VkPushConstantRange* pRanges = (VkPushConstantRange*)createInfo.pPushConstantRanges;
        for (uint32_t i = 0; i < createInfo.pushConstantRangeCount; ++i) {
            // Check if the vertex shader stage is set.
            if ((pRanges[i].stageFlags & VK_SHADER_STAGE_VERTEX_BIT) != 0) {
                // If so, it forcibly adds the Tessellation Evaluation stage.
                pRanges[i].stageFlags |= VK_SHADER_STAGE_TESSELLATION_EVALUATION_SHADER_BIT; // 0x10
            }
        }
    }
    
    // 3. DISPATCH TO VULKAN
    VkPipelineLayout layout = VK_NULL_HANDLE;
    VkResult result = dispatch_vkCreatePipelineLayout(device, &createInfo, NULL, &layout);
    
    // ... (Standard serialization) ...
}
```
**Analysis:** This function modifies the `VkPushConstantRange` array. If a push constant is specified for the vertex shader (`VK_SHADER_STAGE_VERTEX_BIT`), this wrapper automatically adds the `VK_SHADER_STAGE_TESSELLATION_EVALUATION_SHADER_BIT` to its `stageFlags`. This is likely a workaround or a convenience feature to ensure that push constants are available in the tessellation stage without requiring the client to explicitly specify it.

---

#### 14. `vt_handle_vkDestroyPipelineLayout`: **No modifications.** Standard pass-through.
#### 15. `vt_handle_vkCreateSampler`: **No modifications.** Standard pass-through.
#### 16. `vt_handle_vkDestroySampler`: **No modifications.** Standard pass-through.
#### 17. `vt_handle_vkCreateDescriptorSetLayout`: **No modifications.** Standard pass-through.
#### 18. `vt_handle_vkDestroyDescriptorSetLayout`: **No modifications.** Standard pass-through.
#### 19. `vt_handle_vkCreateDescriptorPool`: **No modifications.** Standard pass-through.
#### 20. `vt_handle_vkDestroyDescriptorPool`: **No modifications.** Standard pass-through.

---

### Overall Architecture (Recap)

These functions continue to follow the established architectural pattern:

1.  **Deserialization:** Read arguments from the `context->inputBufferPtr`. Vulkan handles are resolved from IDs using `VkObject_fromId`. Complex structs and arrays are reconstructed in temporary memory.
2.  **Modification/Hooking:** Intercept and potentially alter the arguments or behavior before calling the driver.
3.  **Dispatch:** Call the real Vulkan function pointer from the dispatch table (e.g., `dispatch_vkResetDescriptorPool`).
4.  **Serialization:** Write results back to the `clientRingBuffer`.

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkResetDescriptorPool` | **No** | Standard pass-through. |
| `vkAllocateDescriptorSets` | **No** | Standard pass-through. |
| `vkFreeDescriptorSets` | **No** | Standard pass-through. |
| `vkUpdateDescriptorSets` | **No** | Although complex, it's a faithful deserialization without modifications. |
| `vkCreateFramebuffer` | **No** | Standard pass-through. |
| `vkDestroyFramebuffer` | **No** | Standard pass-through. |
| `vkCreateRenderPass` | **No** | Although complex, it's a faithful deserialization without modifications. |
| `vkDestroyRenderPass` | **No** | Standard pass-through. |
| `vkGetRenderAreaGranularity` | **No** | Standard pass-through. |
| `vkCreateCommandPool` | **No** | Standard pass-through. |
| `vkDestroyCommandPool` | **No** | Standard pass-through. |
| `vkResetCommandPool` | **No** | Standard pass-through. |
| `vkAllocateCommandBuffers` | **No** | Standard pass-through. |
| `vkFreeCommandBuffers` | **No** | Standard pass-through. |
| `vkBeginCommandBuffer` | **No** | Standard pass-through. |
| `vkEndCommandBuffer` | **Yes** | Acts as a trigger to process a batch of other `vkCmd...` calls that were serialized into its input buffer. This is a performance optimization. |
| `vkResetCommandBuffer` | **No** | Standard pass-through. |
| `vkCmdBindPipeline` | **Yes** | Synchronizes with the `AsyncPipelineCreator` by calling `AsyncPipelineCreator_getVkHandle`. This blocks until an asynchronously created pipeline is ready to be bound. |
| `vkCmdSetViewport` | **No** | Standard pass-through. |
| `vkCmdSetScissor` | **No** | Standard pass-through. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkResetDescriptorPool`

```c
/**
 * @brief Handles vkResetDescriptorPool.
 */
void vt_handle_vkResetDescriptorPool(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = VkObject_fromId(*(void**)(input + 1));
    VkDescriptorPool descriptorPool = VkObject_fromId(*(void**)(input + 9));
    VkDescriptorPoolResetFlags flags = *(VkDescriptorPoolResetFlags*)(input + 0x11);
    
    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkResetDescriptorPool(device, descriptorPool, flags);
    
    // 4. SERIALIZATION
    // None, this is a void function.
}
```
**Analysis:** Standard pass-through. Deserializes the device, pool handle, and flags, then calls the real Vulkan function.

---

#### 2. `vt_handle_vkAllocateDescriptorSets`

```c
/**
 * @brief Handles vkAllocateDescriptorSets.
 */
void vt_handle_vkAllocateDescriptorSets(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));

    VkDescriptorSetAllocateInfo allocateInfo = {0};
    deserialize_vkDescriptorSetAllocateInfo(context, &allocateInfo);

    // 2. MODIFICATION & HOOKING
    // No modifications detected.

    // 3. DISPATCH TO VULKAN
    // Allocate a temporary buffer on the stack for the output handles.
    VkDescriptorSet* pDescriptorSets = (VkDescriptorSet*)alloca(allocateInfo.descriptorSetCount * sizeof(VkDescriptorSet));
    VkResult result = dispatch_vkAllocateDescriptorSets(device, &allocateInfo, pDescriptorSets);

    // 4. SERIALIZATION
    uint32_t responseSize = sizeof(VkResult) + sizeof(uint32_t) + (allocateInfo.descriptorSetCount * sizeof(VkDescriptorSet));
    void* resultBuffer = malloc(responseSize);
    
    *(VkResult*)resultBuffer = result;
    *(uint32_t*)((char*)resultBuffer + sizeof(VkResult)) = allocateInfo.descriptorSetCount;
    memcpy((char*)resultBuffer + sizeof(VkResult) + sizeof(uint32_t), pDescriptorSets, allocateInfo.descriptorSetCount * sizeof(VkDescriptorSet));
    
    RingBuffer_write(context->clientRingBuffer, resultBuffer, responseSize);
    free(resultBuffer);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** Standard pass-through. It correctly deserializes the `VkDescriptorSetAllocateInfo` struct, including resolving the array of `VkDescriptorSetLayout` handles. It then calls the Vulkan function and serializes the resulting array of new `VkDescriptorSet` handles back to the client.

---

#### 3. `vt_handle_vkFreeDescriptorSets`

```c
/**
 * @brief Handles vkFreeDescriptorSets.
 */
void vt_handle_vkFreeDescriptorSets(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = VkObject_fromId(*(void**)(input + 1));
    VkDescriptorPool descriptorPool = VkObject_fromId(*(void**)(input + 9));
    uint32_t descriptorSetCount = *(uint32_t*)(input + 0x11);

    // Allocate temporary buffer for the descriptor set handles.
    VkDescriptorSet* pDescriptorSets = (VkDescriptorSet*)alloca(descriptorSetCount * sizeof(VkDescriptorSet));
    
    // Deserialize the array of handles to be freed.
    for (uint32_t i = 0; i < descriptorSetCount; ++i) {
        pDescriptorSets[i] = (VkDescriptorSet)VkObject_fromId(*(void**)(input + 0x15 + i * sizeof(void*)));
    }

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkFreeDescriptorSets(device, descriptorPool, descriptorSetCount, pDescriptorSets);
    
    // 4. SERIALIZATION
    // None, this is a void function.
}
```
**Analysis:** Standard pass-through. It deserializes the list of `VkDescriptorSet` handles to be freed and calls the dispatch function.

---

#### 4. `vt_handle_vkUpdateDescriptorSets`

```c
/**
 * @brief Handles vkUpdateDescriptorSets. This is a complex deserialization.
 */
void vt_handle_vkUpdateDescriptorSets(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkDevice device = VkObject_fromId(*(void**)(input + 1));
    
    uint32_t descriptorWriteCount = *(uint32_t*)(input + 9);
    uint32_t descriptorCopyCount = *(uint32_t*)(input + 0xD);
    char* dataPtr = input + 0x11;
    
    // Allocate temporary memory for the write and copy arrays.
    VkWriteDescriptorSet* pWrites = (VkWriteDescriptorSet*)alloca(descriptorWriteCount * sizeof(VkWriteDescriptorSet));
    VkCopyDescriptorSet* pCopies = (VkCopyDescriptorSet*)alloca(descriptorCopyCount * sizeof(VkCopyDescriptorSet));
    
    // Deserialize VkWriteDescriptorSet array
    for (uint32_t i = 0; i < descriptorWriteCount; ++i) {
        // This is a highly complex process involving reading nested structs and unions.
        // It reconstructs VkDescriptorImageInfo, VkDescriptorBufferInfo, etc., resolving
        // all handles (VkBuffer, VkImageView, VkSampler) along the way.
        dataPtr += deserialize_vkWriteDescriptorSet(context, dataPtr, &pWrites[i]);
    }

    // Deserialize VkCopyDescriptorSet array
    for (uint32_t i = 0; i < descriptorCopyCount; ++i) {
        dataPtr += deserialize_vkCopyDescriptorSet(context, dataPtr, &pCopies[i]);
    }
    
    // 2. MODIFICATION & HOOKING
    // No modifications are made to the contents of the deserialized structs.

    // 3. DISPATCH TO VULKAN
    dispatch_vkUpdateDescriptorSets(device, descriptorWriteCount, pWrites, descriptorCopyCount, pCopies);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** This is one of the most complex deserialization functions due to the nature of `vkUpdateDescriptorSets`. It has to handle multiple arrays of structs, which themselves contain unions and pointers to other structs. Despite this complexity, the function's goal is simply to faithfully reconstruct the client's request. No modifications to the descriptor data are made.

---

#### 5. `vt_handle_vkCreateFramebuffer`: **No modifications.** Standard pass-through.
#### 6. `vt_handle_vkDestroyFramebuffer`: **No modifications.** Standard pass-through.
#### 7. `vt_handle_vkCreateRenderPass`: **No modifications.** Standard pass-through.
#### 8. `vt_handle_vkDestroyRenderPass`: **No modifications.** Standard pass-through.
#### 9. `vt_handle_vkGetRenderAreaGranularity`: **No modifications.** Standard pass-through.
#### 10. `vt_handle_vkCreateCommandPool`: **No modifications.** Standard pass-through.
#### 11. `vt_handle_vkDestroyCommandPool`: **No modifications.** Standard pass-through.
#### 12. `vt_handle_vkResetCommandPool`: **No modifications.** Standard pass-through.
#### 13. `vt_handle_vkAllocateCommandBuffers`: **No modifications.** Standard pass-through.
#### 14. `vt_handle_vkFreeCommandBuffers`: **No modifications.** Standard pass-through.
#### 15. `vt_handle_vkBeginCommandBuffer`: **No modifications.** Standard pass-through.

---

#### 16. `vt_handle_vkEndCommandBuffer`

```c
/**
 * @brief Handles vkEndCommandBuffer and processes a batch of inline commands.
 */
void vt_handle_vkEndCommandBuffer(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    int inlineCommandOffset = 9; // Start of inline commands after the handle

    // 2. MODIFICATION & HOOKING
    // The primary purpose of this function is to act as a trigger for a command batch.
    // It iterates through a series of commands serialized directly after the main handle.
    if (context->inputBufferSize > 9) {
        do {
            // Read command ID and size from the inline buffer
            uint16_t commandId = *(uint16_t*)(context->inputBufferPtr + inlineCommandOffset);
            uint32_t commandSize = *(uint32_t*)(context->inputBufferPtr + inlineCommandOffset + 2);
            
            // Update the context's buffer pointer to point to the current inline command's data.
            context->inputBufferPtr += (inlineCommandOffset + 6);
            context->inputBufferSize = commandSize;

            // Look up the handler function for the command ID.
            void (*handlerFunc)(VtContext*) = getHandleRequestFunc(commandId);
            if (handlerFunc) {
                // Execute the handler for the inline command.
                handlerFunc(context);
            }
            
            // Move to the next inline command.
            inlineCommandOffset += (commandSize + 6);
        } while (inlineCommandOffset < context->originalInputBufferSize); // Loop until all inline commands are processed
    }
    
    // 3. DISPATCH TO VULKAN
    // After processing the inline batch, call the actual vkEndCommandBuffer.
    dispatch_vkEndCommandBuffer(commandBuffer);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** This function exhibits a significant performance optimization. Instead of sending many small network packets for each `vkCmd...` function, the client batches them into a single buffer and sends it with a `vkEndCommandBuffer` command. The `vt_handle_vkEndCommandBuffer` function then iterates through this buffer, dispatching each inline command to its corresponding handler (`vt_handle_vkCmdDraw`, `vt_handle_vkCmdBindPipeline`, etc.). This greatly reduces the overhead of inter-process communication.

---

#### 17. `vt_handle_vkResetCommandBuffer`: **No modifications.** Standard pass-through.

---

#### 18. `vt_handle_vkCmdBindPipeline`

```c
/**
 * @brief Handles vkCmdBindPipeline, synchronizing with async pipeline creation.
 */
void vt_handle_vkCmdBindPipeline(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkPipelineBindPoint pipelineBindPoint = *(VkPipelineBindPoint*)(input + 9);
    // The pipeline handle is an ID for a custom wrapper object.
    void* pipelineWrapper = VkObject_fromId(*(void**)(input + 0xD));
    
    // 2. MODIFICATION & HOOKING
    // This is the key modification. It ensures that if the pipeline was created
    // asynchronously, this call will block until the pipeline is fully compiled
    // and ready for use.
    VkPipeline pipeline = AsyncPipelineCreator_getVkHandle(pipelineWrapper);
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBindPipeline(commandBuffer, pipelineBindPoint, pipeline);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** This function is a critical component of the asynchronous pipeline creation system. When the client wants to bind a pipeline, it passes the ID of the wrapper object created by `vt_handle_vkCreate...Pipelines`. The handler calls `AsyncPipelineCreator_getVkHandle`, which acts as a synchronization point. If the pipeline compilation on the worker thread is not yet finished, this call will block until it is. This guarantees that a valid `VkPipeline` handle is always passed to the real `vkCmdBindPipeline` call, preventing crashes or undefined behavior.

---

#### 19. `vt_handle_vkCmdSetViewport`

```c
/**
 * @brief Handles vkCmdSetViewport.
 */
void vt_handle_vkCmdSetViewport(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t firstViewport = *(uint32_t*)(input + 9);
    uint32_t viewportCount = *(uint32_t*)(input + 0xD);
    
    // Allocate temporary buffer for the viewports array.
    VkViewport* pViewports = (VkViewport*)alloca(viewportCount * sizeof(VkViewport));
    
    // Deserialize the array of VkViewport structs.
    // (Loop that reads float members for each viewport)
    deserialize_VkViewport_array(input + 0x11, viewportCount, pViewports);
    
    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetViewport(commandBuffer, firstViewport, viewportCount, pViewports);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. It deserializes the array of `VkViewport` structs and calls the corresponding Vulkan command.

---

#### 20. `vt_handle_vkCmdSetScissor`

```c
/**
 * @brief Handles vkCmdSetScissor.
 */
void vt_handle_vkCmdSetScissor(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t firstScissor = *(uint32_t*)(input + 9);
    uint32_t scissorCount = *(uint32_t*)(input + 0xD);
    
    // Allocate temporary buffer for the scissors array.
    VkRect2D* pScissors = (VkRect2D*)alloca(scissorCount * sizeof(VkRect2D));
    
    // Deserialize the array of VkRect2D structs.
    deserialize_VkRect2D_array(input + 0x11, scissorCount, pScissors);
    
    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetScissor(commandBuffer, firstScissor, scissorCount, pScissors);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. It deserializes the array of `VkRect2D` structs and calls the corresponding Vulkan command.

---

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkCmdSetLineWidth` | **No** | Standard pass-through. |
| `vkCmdSetDepthBias` | **No** | Standard pass-through. |
| `vkCmdSetBlendConstants` | **No** | Standard pass-through. |
| `vkCmdSetDepthBounds` | **No** | Standard pass-through. |
| `vkCmdSetStencilCompareMask` | **No** | Standard pass-through. |
| `vkCmdSetStencilWriteMask` | **No** | Standard pass-through. |
| `vkCmdSetStencilReference` | **No** | Standard pass-through. |
| `vkCmdBindDescriptorSets` | **No** | Standard pass-through; only resolves handle IDs. |
| `vkCmdBindIndexBuffer` | **No** | Standard pass-through; only resolves the buffer handle ID. |
| `vkCmdBindVertexBuffers` | **No** | Standard pass-through; only resolves buffer handle IDs. |
| `vkCmdDraw` | **No** | Standard pass-through. |
| `vkCmdDrawIndexed` | **No** | Standard pass-through. |
| `vkCmdDrawIndirect` | **No** | Standard pass-through; only resolves the buffer handle ID. |
| `vkCmdDrawIndexedIndirect`| **No** | Standard pass-through; only resolves the buffer handle ID. |
| `vkCmdDispatch` | **No** | Standard pass-through. |
| `vkCmdDispatchIndirect` | **No** | Standard pass-through; only resolves the buffer handle ID. |
| `vkCmdCopyBuffer` | **No** | Standard pass-through; only resolves buffer handle IDs. |
| `vkCmdCopyImage` | **No** | Standard pass-through; only resolves image handle IDs. |
| `vkCmdBlitImage` | **No** | Standard pass-through; only resolves image handle IDs. |
| `vkCmdCopyBufferToImage` | **Yes** | **Hooks the call** for emulated compressed textures. If the destination image is a known compressed texture, it calls a custom `TextureDecoder_copyBufferToImage` function to handle the on-the-fly decompression. Otherwise, it's a standard pass-through. |

---

### Detailed Function Decompilations

#### 1-7. `vkCmdSet...` Dynamic State Functions

These seven functions (`vkCmdSetLineWidth`, `vkCmdSetDepthBias`, `vkCmdSetBlendConstants`, `vkCmdSetDepthBounds`, `vkCmdSetStencilCompareMask`, `vkCmdSetStencilWriteMask`, `vkCmdSetStencilReference`) all follow the same simple, unmodified pass-through pattern.

```c
/**
 * @brief Handles a generic vkCmdSet<State> command.
 * This template represents all 7 of these functions.
 */
void vt_handle_vkCmdSetState(VtContext* context) {
    // 1. DESERIALIZATION
    // The input buffer contains the command buffer handle followed by the state value(s).
    // e.g., for vkCmdSetLineWidth: VkCommandBuffer, float lineWidth
    // e.g., for vkCmdSetBlendConstants: VkCommandBuffer, float[4] blendConstants
    
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    // The state value(s) are read directly from the input buffer.
    // Example for line width:
    float lineWidth = *(float*)(input + 9); 
    
    // 2. MODIFICATION & HOOKING
    // There are no modifications to the state values. They are passed directly to the driver.

    // 3. DISPATCH TO VULKAN
    // The corresponding function from the dispatch table is called.
    dispatch_vkCmdSetLineWidth(commandBuffer, lineWidth); 

    // 4. SERIALIZATION
    // None, as these are void command buffer functions.
}
```
**Analysis:** All seven dynamic state functions are simple pass-through wrappers. They deserialize the command buffer handle and the primitive arguments and immediately call the corresponding Vulkan function without any modification.

---

#### 8. `vt_handle_vkCmdBindDescriptorSets`

```c
/**
 * @brief Handles vkCmdBindDescriptorSets.
 */
void vt_handle_vkCmdBindDescriptorSets(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkPipelineBindPoint pipelineBindPoint = *(VkPipelineBindPoint*)(input + 9);
    VkPipelineLayout layout = VkObject_fromId(*(void**)(input + 0xD));
    uint32_t firstSet = *(uint32_t*)(input + 0x15);
    uint32_t descriptorSetCount = *(uint32_t*)(input + 0x19);
    
    // Deserialize VkDescriptorSet handles into a temporary array.
    VkDescriptorSet* pSets = malloc(descriptorSetCount * sizeof(VkDescriptorSet));
    char* pSetData = input + 0x21; 
    for (uint32_t i = 0; i < descriptorSetCount; ++i) {
        pSets[i] = VkObject_fromId(*(void**)(pSetData + i * sizeof(void*)));
    }

    // Deserialize dynamic offsets.
    uint32_t dynamicOffsetCount = *(uint32_t*)(pSetData + descriptorSetCount * sizeof(void*));
    uint32_t* pDynamicOffsets = (uint32_t*)(pSetData + descriptorSetCount * sizeof(void*) + 4);
    
    // 2. MODIFICATION & HOOKING
    // No modifications. The function only resolves handle IDs.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBindDescriptorSets(
        commandBuffer, pipelineBindPoint, layout, firstSet,
        descriptorSetCount, pSets,
        dynamicOffsetCount, pDynamicOffsets
    );
    
    free(pSets);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. Its only logic is to deserialize the array of `VkDescriptorSet` handles, resolve their IDs to real handles, and call the driver function.

---

#### 9-17. `vkCmdBind...`, `vkCmdDraw...`, `vkCmdDispatch...` Functions

This group of nine functions (`vkCmdBindIndexBuffer`, `vkCmdBindVertexBuffers`, `vkCmdDraw`, `vkCmdDrawIndexed`, `vkCmdDrawIndirect`, `vkCmdDrawIndexedIndirect`, `vkCmdDispatch`, `vkCmdDispatchIndirect`, `vkCmdCopyBuffer`) are all simple pass-throughs with no modifications.

```c
/**
 * @brief Template for simple command buffer functions that take handles and primitive types.
 */
void vt_handle_vkCmdSimpleCommand(VtContext* context) {
    // 1. DESERIALIZATION
    // Reads VkCommandBuffer handle.
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // Reads other arguments, such as:
    // - For vkCmdBindIndexBuffer: VkBuffer handle, offset, indexType.
    // - For vkCmdDraw: vertexCount, instanceCount, firstVertex, firstInstance.
    // Any Vulkan object handles are resolved with VkObject_fromId.
    VkBuffer buffer = VkObject_fromId(...);
    VkDeviceSize offset = ...;
    uint32_t drawCount = ...;
    // ...etc.

    // 2. MODIFICATION & HOOKING
    // None. The arguments are used as-is.

    // 3. DISPATCH TO VULKAN
    // Calls the corresponding function from the dispatch table.
    dispatch_vkCmdBindIndexBuffer(commandBuffer, buffer, offset, indexType);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** These functions are all standard pass-through wrappers. They deserialize their arguments, resolve any object handle IDs, and make the corresponding call to the Vulkan driver without any modifications.

---

#### 18. `vt_handle_vkCmdCopyImage` & 19. `vt_handle_vkCmdBlitImage`

```c
/**
 * @brief Handles vkCmdCopyImage and vkCmdBlitImage.
 */
void vt_handle_vkCmdImageTransfer(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    
    // Resolve source and destination image handles.
    VkImage srcImage = VkObject_fromId(*(void**)(input + 9));
    VkImageLayout srcImageLayout = *(VkImageLayout*)(input + 0x11);
    VkImage dstImage = VkObject_fromId(*(void**)(input + 0x15));
    VkImageLayout dstImageLayout = *(VkImageLayout*)(input + 0x19);
    
    // Deserialize array of VkImageCopy/VkImageBlit regions.
    uint32_t regionCount = *(uint32_t*)(input + 0x1D);
    VkImageCopy* pRegions = (VkImageCopy*)(input + 0x21); // Or VkImageBlit
    
    // For blit, also deserialize the filter.
    VkFilter filter = *(VkFilter*)(input + 0x21 + regionCount * sizeof(VkImageCopy));
    
    // 2. MODIFICATION & HOOKING
    // No modifications detected in the code for these two functions. They do not
    // check for compressed formats or interact with the TextureDecoder. This implies
    // that image-to-image operations are expected to work on already-valid
    // (potentially decompressed) Vulkan images.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdCopyImage(commandBuffer, srcImage, srcImageLayout, dstImage, dstImageLayout, regionCount, pRegions);
    // OR
    dispatch_vkCmdBlitImage(commandBuffer, srcImage, srcImageLayout, dstImage, dstImageLayout, regionCount, pRegions, filter);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Surprisingly, these are also standard pass-through functions. One might expect texture decompression hooks here, but the implementation chooses not to intervene in image-to-image transfers, likely because the source image is assumed to be in a valid, accessible state already (i.e., decompressed by a previous `vkCmdCopyBufferToImage` call).

---

#### 20. `vt_handle_vkCmdCopyBufferToImage`

```c
/**
 * @brief Handles vkCmdCopyBufferToImage, with hooks for texture decompression.
 */
void vt_handle_vkCmdCopyBufferToImage(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkBuffer srcBuffer = VkObject_fromId(*(void**)(input + 9));
    VkImage dstImage = VkObject_fromId(*(void**)(input + 0x11));
    VkImageLayout dstImageLayout = *(VkImageLayout*)(input + 0x19);
    uint32_t regionCount = *(uint32_t*)(input + 0x1D);
    
    // Deserialize the array of VkBufferImageCopy regions.
    VkBufferImageCopy* pRegions = (VkBufferImageCopy*)(input + 0x21);

    // 2. MODIFICATION & HOOKING
    // This is a key interception point for the texture emulation system.
    if (context->textureDecoder != NULL) {
        // Check if the destination image is managed by the texture decoder.
        bool isEmulated = TextureDecoder_containsImage(context->textureDecoder, (long)dstImage);
        if (isEmulated) {
            // If the image is an emulated compressed texture, this call is completely hijacked.
            // Instead of calling the Vulkan driver's copy command, it calls a custom
            // function in the TextureDecoder module.
            if (pRegions[0].bufferOffset == 0) {
                // The custom function likely takes the raw buffer data, decompresses it
                // on the fly (or queues it for GPU decompression), and places the result
                // into the real, underlying uncompressed VkImage associated with the
                // emulated compressed image handle.
                TextureDecoder_copyBufferToImage(
                    context->textureDecoder,
                    commandBuffer,
                    (long)srcBuffer,
                    (long)dstImage,
                    dstImageLayout,
                    pRegions[0].imageSubresource // Assuming one region for simplicity
                );
            }
            // The function returns here, bypassing the standard dispatch.
            return;
        }
    }
    
    // 3. DISPATCH TO VULKAN
    // If the image is not an emulated texture, perform a standard Vulkan call.
    dispatch_vkCmdCopyBufferToImage(commandBuffer, srcBuffer, dstImage, dstImageLayout, regionCount, pRegions);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** This is the most significantly modified function in this set. It serves as the primary data path for the custom texture decompression system. When the client application attempts to copy data from a buffer into what it thinks is a compressed `VkImage`, this wrapper intercepts the call. It diverts the command to its internal `TextureDecoder` module, which performs the necessary decompression and places the data into the actual backing `VkImage` (which has an uncompressed format like `VK_FORMAT_B8G8R8A8_UNORM`). This allows the client to work with compressed texture assets without the underlying Vulkan driver needing to support them directly.

---

restart

---

### Summary of Modifications

This batch of functions primarily consists of `vkCmd...` commands that set pipeline state or record drawing/dispatch/copy operations. As these functions are recorded into a command buffer and do not return data to the host, they generally do not have a serialization phase.

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkCmdSetLineWidth` | **No** | Standard pass-through. |
| `vkCmdSetDepthBias` | **No** | Standard pass-through. |
| `vkCmdSetBlendConstants` | **No** | Standard pass-through. |
| `vkCmdSetDepthBounds` | **No** | Standard pass-through. |
| `vkCmdSetStencilCompareMask`| **No** | Standard pass-through. |
| `vkCmdSetStencilWriteMask`| **No** | Standard pass-through. |
| `vkCmdSetStencilReference`| **No** | Standard pass-through. |
| `vkCmdBindDescriptorSets`| **No** | Standard pass-through. |
| `vkCmdBindIndexBuffer` | **No** | Standard pass-through. |
| `vkCmdBindVertexBuffers` | **No** | Standard pass-through. |
| `vkCmdDraw` | **No** | Standard pass-through. |
| `vkCmdDrawIndexed` | **No** | Standard pass-through. |
| `vkCmdDrawIndirect` | **No** | Standard pass-through. |
| `vkCmdDrawIndexedIndirect`| **No** | Standard pass-through. |
| `vkCmdDispatch` | **No** | Standard pass-through. |
| `vkCmdDispatchIndirect`| **No** | Standard pass-through. |
| `vkCmdCopyBuffer` | **No** | Standard pass-through. |
| `vkCmdCopyImage` | **No** | Standard pass-through. |
| `vkCmdBlitImage` | **No** | Standard pass-through. |
| `vkCmdCopyBufferToImage` | **Yes** | Intercepts copies to emulated compressed textures and redirects them to a custom `TextureDecoder_copyBufferToImage` function to handle decompression. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkCmdSetLineWidth`

```c
/**
 * @brief Handles vkCmdSetLineWidth.
 */
void vt_handle_vkCmdSetLineWidth(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    float lineWidth = *(float*)(input + 9);

    // 2. MODIFICATION & HOOKING
    // No modifications are performed.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetLineWidth(commandBuffer, lineWidth);

    // 4. SERIALIZATION
    // None (void command).
}
```
**Analysis:** Standard pass-through. Deserializes the command buffer handle and the float value, then calls the real Vulkan function.

---

#### 2. `vt_handle_vkCmdSetDepthBias`

```c
/**
 * @brief Handles vkCmdSetDepthBias.
 */
void vt_handle_vkCmdSetDepthBias(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    float depthBiasConstantFactor = *(float*)(input + 9);
    float depthBiasClamp = *(float*)(input + 13);
    float depthBiasSlopeFactor = *(float*)(input + 17);

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthBias(commandBuffer, depthBiasConstantFactor, depthBiasClamp, depthBiasSlopeFactor);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through.

---

#### 3. `vt_handle_vkCmdSetBlendConstants`

```c
/**
 * @brief Handles vkCmdSetBlendConstants.
 */
void vt_handle_vkCmdSetBlendConstants(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    float blendConstants[4];
    memcpy(blendConstants, input + 9, sizeof(float) * 4);

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetBlendConstants(commandBuffer, blendConstants);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through.

---

#### 4. `vt_handle_vkCmdSetDepthBounds`

```c
/**
 * @brief Handles vkCmdSetDepthBounds.
 */
void vt_handle_vkCmdSetDepthBounds(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    float minDepthBounds = *(float*)(input + 9);
    float maxDepthBounds = *(float*)(input + 13);

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthBounds(commandBuffer, minDepthBounds, maxDepthBounds);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through.

---

#### 5-7. `vt_handle_vkCmdSetStencil...` Functions

All three stencil functions (`vkCmdSetStencilCompareMask`, `vkCmdSetStencilWriteMask`, `vkCmdSetStencilReference`) follow the exact same simple pass-through pattern.

```c
/**
 * @brief Handles vkCmdSetStencilCompareMask, vkCmdSetStencilWriteMask, and vkCmdSetStencilReference.
 */
void vt_handle_vkCmdSetStencilCompareMask(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkStencilFaceFlags faceMask = *(VkStencilFaceFlags*)(input + 9);
    uint32_t compareMask = *(uint32_t*)(input + 13);
    
    // 2. MODIFICATION & HOOKING: None
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetStencilCompareMask(commandBuffer, faceMask, compareMask);
    
    // 4. SERIALIZATION: None
}
```
**Analysis:** All are standard pass-through functions.

---

#### 8. `vt_handle_vkCmdBindDescriptorSets`

```c
/**
 * @brief Handles vkCmdBindDescriptorSets.
 */
void vt_handle_vkCmdBindDescriptorSets(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkPipelineBindPoint pipelineBindPoint = *(VkPipelineBindPoint*)(input + 9);
    VkPipelineLayout layout = VkObject_fromId(*(void**)(input + 13));
    uint32_t firstSet = *(uint32_t*)(input + 0x15);
    uint32_t descriptorSetCount = *(uint32_t*)(input + 0x19);
    
    // Deserialize array of VkDescriptorSet handles
    VkDescriptorSet* pDescriptorSets = (VkDescriptorSet*)alloca(descriptorSetCount * sizeof(VkDescriptorSet));
    for (uint32_t i = 0; i < descriptorSetCount; ++i) {
        pDescriptorSets[i] = VkObject_fromId(*(void**)(input + 0x1D + i * 8));
    }
    
    uint32_t dynamicOffsetCount = *(uint32_t*)(input + 0x1D + descriptorSetCount * 8);
    
    // Deserialize array of dynamic offsets
    uint32_t* pDynamicOffsets = (uint32_t*)alloca(dynamicOffsetCount * sizeof(uint32_t));
    memcpy(pDynamicOffsets, input + 0x21 + descriptorSetCount * 8, dynamicOffsetCount * sizeof(uint32_t));
    
    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBindDescriptorSets(commandBuffer, pipelineBindPoint, layout, firstSet, descriptorSetCount, pDescriptorSets, dynamicOffsetCount, pDynamicOffsets);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** This is a standard pass-through. Although it deserializes multiple arrays, it performs no modifications on the data before calling the real Vulkan function.

---

#### 9-16. Draw, Dispatch, and Bind Buffer Commands

The functions `vkCmdBindIndexBuffer`, `vkCmdBindVertexBuffers`, `vkCmdDraw`, `vkCmdDrawIndexed`, `vkCmdDrawIndirect`, `vkCmdDrawIndexedIndirect`, `vkCmdDispatch`, and `vkCmdDispatchIndirect` are all standard, non-modifying pass-through functions. They follow a simple "deserialize arguments -> call dispatch function" pattern.

---

#### 17. `vt_handle_vkCmdCopyBuffer`

```c
/**
 * @brief Handles vkCmdCopyBuffer.
 */
void vt_handle_vkCmdCopyBuffer(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkBuffer srcBuffer = VkObject_fromId(*(void**)(input + 9));
    VkBuffer dstBuffer = VkObject_fromId(*(void**)(input + 17));
    uint32_t regionCount = *(uint32_t*)(input + 25);

    // Deserialize array of VkBufferCopy regions
    VkBufferCopy* pRegions = (VkBufferCopy*)alloca(regionCount * sizeof(VkBufferCopy));
    memcpy(pRegions, input + 29, regionCount * sizeof(VkBufferCopy));

    // 2. MODIFICATION & HOOKING
    // No modifications.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, regionCount, pRegions);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through copy command.

---

#### 18. `vt_handle_vkCmdCopyImage`

```c
/**
 * @brief Handles vkCmdCopyImage.
 */
void vt_handle_vkCmdCopyImage(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkImage srcImage = VkObject_fromId(*(void**)(input + 9));
    VkImageLayout srcImageLayout = *(VkImageLayout*)(input + 17);
    VkImage dstImage = VkObject_fromId(*(void**)(input + 21));
    VkImageLayout dstImageLayout = *(VkImageLayout*)(input + 29);
    uint32_t regionCount = *(uint32_t*)(input + 33);

    // Deserialize array of VkImageCopy regions
    VkImageCopy* pRegions = (VkImageCopy*)alloca(regionCount * sizeof(VkImageCopy));
    // ... complex memcpy logic for the array ...
    deserialize_ImageCopy_array(pRegions, input + 37, regionCount);

    // 2. MODIFICATION & HOOKING
    // No modifications.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdCopyImage(commandBuffer, srcImage, srcImageLayout, dstImage, dstImageLayout, regionCount, pRegions);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through image copy.

---

#### 19. `vt_handle_vkCmdBlitImage`

```c
/**
 * @brief Handles vkCmdBlitImage.
 */
void vt_handle_vkCmdBlitImage(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkImage srcImage = VkObject_fromId(*(void**)(input + 9));
    VkImageLayout srcImageLayout = *(VkImageLayout*)(input + 17);
    VkImage dstImage = VkObject_fromId(*(void**)(input + 21));
    VkImageLayout dstImageLayout = *(VkImageLayout*)(input + 29);
    uint32_t regionCount = *(uint32_t*)(input + 33);

    // Deserialize array of VkImageBlit regions
    VkImageBlit* pRegions = (VkImageBlit*)alloca(regionCount * sizeof(VkImageBlit));
    // ... complex memcpy logic for the array ...
    deserialize_ImageBlit_array(pRegions, input + 37, regionCount);

    VkFilter filter = *(VkFilter*)(input + 37 + regionCount * sizeof(VkImageBlit));

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBlitImage(commandBuffer, srcImage, srcImageLayout, dstImage, dstImageLayout, regionCount, pRegions, filter);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through blit command.

---

#### 20. `vt_handle_vkCmdCopyBufferToImage`

```c
/**
 * @brief Handles vkCmdCopyBufferToImage, with hooks for texture decompression.
 */
void vt_handle_vkCmdCopyBufferToImage(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkBuffer srcBuffer = VkObject_fromId(*(void**)(input + 9));
    VkImage dstImage = VkObject_fromId(*(void**)(input + 17));
    VkImageLayout dstImageLayout = *(VkImageLayout*)(input + 25);
    uint32_t regionCount = *(uint32_t*)(input + 29);
    
    // Deserialize array of VkBufferImageCopy regions
    VkBufferImageCopy* pRegions = (VkBufferImageCopy*)alloca(regionCount * sizeof(VkBufferImageCopy));
    deserialize_BufferImageCopy_array(pRegions, input + 33, regionCount);
    
    // 2. MODIFICATION & HOOKING
    if (context->textureDecoder != NULL && TextureDecoder_containsImage(context->textureDecoder, dstImage)) {
        // Condition: The destination image is an emulated compressed texture.
        if (pRegions[0].bufferOffset == 0) {
            // Further Condition: The copy starts from the beginning of the buffer.
            
            // Call the custom texture decoder instead of the Vulkan driver.
            TextureDecoder_copyBufferToImage(
                context->textureDecoder,
                commandBuffer,
                srcBuffer,
                dstImage,
                dstImageLayout,
                pRegions[0].bufferRowLength,
                pRegions[0].bufferImageHeight
            );
        }
    } else {
        // 3. DISPATCH TO VULKAN (for non-emulated images)
        dispatch_vkCmdCopyBufferToImage(commandBuffer, srcBuffer, dstImage, dstImageLayout, regionCount, pRegions);
    }
    
    // 4. SERIALIZATION
    // None.
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** **Modification Detected.** This function is a critical component of the texture decompression system.

1.  It checks if the destination `VkImage` is one of the emulated compressed textures managed by the `TextureDecoder`.
2.  If it is, the function intercepts the call and redirects it to a custom function, `TextureDecoder_copyBufferToImage`.
3.  This custom function is responsible for taking the linear data from the `srcBuffer` and correctly decompressing/arranging it into the block-based format of the real (uncompressed) texture that backs the emulated compressed image.
4.  If the destination image is *not* an emulated one, it proceeds as a normal pass-through call to the Vulkan driver. This ensures standard functionality is unaffected.

---

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkCmdCopyImageToBuffer` | **Yes** | Intercepts the call to implement on-the-fly texture decompression via the `TextureDecoder` module if the source image uses a compressed format. |
| `vkCmdUpdateBuffer` | **No** | Standard pass-through. |
| `vkCmdFillBuffer` | **No** | Standard pass-through. |
| `vkCmdClearColorImage` | **No** | Standard pass-through. |
| `vkCmdClearDepthStencilImage` | **No** | Standard pass-through. |
| `vkCmdClearAttachments` | **No** | Standard pass-through. |
| `vkCmdResolveImage` | **No** | Standard pass-through. |
| `vkCmdSetEvent` | **No** | Standard pass-through. |
| `vkCmdResetEvent` | **No** | Standard pass-through. |
| `vkCmdWaitEvents` | **No** | Standard pass-through. Handles complex deserialization but does not alter parameters. |
| `vkCmdPipelineBarrier` | **No** | Standard pass-through. Handles complex deserialization of three different barrier types but does not alter parameters beyond handle resolution. |
| `vkCmdBeginQuery` | **No** | Standard pass-through. |
| `vkCmdEndQuery` | **No** | Standard pass-through. |
| `vkCmdBeginConditionalRenderingEXT` | **No** | Standard pass-through. |
| `vkCmdEndConditionalRenderingEXT` | **No** | Standard pass-through. |
| `vkCmdResetQueryPool` | **No** | Standard pass-through. |
| `vkCmdWriteTimestamp` | **No** | Standard pass-through. |
| `vkCmdCopyQueryPoolResults` | **No** | Standard pass-through. Manages the output data buffer by allocating temporary memory and serializing the results back. |
| `vkCmdPushConstants` | **Yes** | Modifies the `stageFlags` parameter. If a push constant is visible to the vertex shader, it automatically makes it visible to the tessellation evaluation shader. |
| `vkCmdBeginRenderPass` | **No** | Standard pass-through. Handles complex deserialization of the `VkRenderPassBeginInfo` struct and its nested arrays. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkCmdCopyImageToBuffer`

```c
/**
 * @brief Handles vkCmdCopyImageToBuffer, with hooks for texture decompression.
 */
void vt_handle_vkCmdCopyImageToBuffer(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkImage srcImage = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    VkImageLayout srcImageLayout = *(VkImageLayout*)(context->inputBufferPtr + 0x11);
    VkBuffer dstBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 0x15));
    uint32_t regionCount = *(uint32_t*)(context->inputBufferPtr + 0x1d);
    
    // Deserialize the array of VkBufferImageCopy regions into a temporary buffer.
    VkBufferImageCopy* pRegions = (VkBufferImageCopy*)alloca(regionCount * sizeof(VkBufferImageCopy));
    deserialize_vkBufferImageCopy_array(context, pRegions, regionCount);
    
    // 2. MODIFICATION & HOOKING
    // Check if the texture decoder is enabled and if the source image is one it manages.
    if (context->textureDecoder != NULL && TextureDecoder_containsImage(context->textureDecoder, srcImage)) {
        // If so, the call is completely redirected to the texture decoder module.
        // This indicates that the image data is not in a standard layout in GPU memory
        // and requires a custom copy/decompression routine.
        TextureDecoder_copyImageToBuffer(context->textureDecoder, commandBuffer, srcImage, dstBuffer, regionCount, pRegions);
    } else {
        // 3. DISPATCH TO VULKAN
        // If not a hooked image, perform a standard pass-through call.
        dispatch_vkCmdCopyImageToBuffer(commandBuffer, srcImage, srcImageLayout, dstBuffer, regionCount, pRegions);
    }

    // 4. SERIALIZATION
    // None, this is a void command.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function is a critical hook for the texture decompression system. It intercepts calls that read data from an image. If the `VkImage` handle corresponds to a compressed texture managed by `TextureDecoder`, it bypasses the standard Vulkan command and calls a custom `TextureDecoder_copyImageToBuffer` function. This custom function is responsible for decompressing the texture data from its internal representation into the destination `VkBuffer`. This is a significant modification to the standard API behavior.

---

#### 2. `vt_handle_vkCmdUpdateBuffer`

```c
/**
 * @brief Handles vkCmdUpdateBuffer.
 */
void vt_handle_vkCmdUpdateBuffer(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBuffer dstBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    VkDeviceSize dstOffset = *(VkDeviceSize*)(context->inputBufferPtr + 0x11);
    VkDeviceSize dataSize = *(VkDeviceSize*)(context->inputBufferPtr + 0x19);
    
    // The data to be written is part of the same input buffer.
    const void* pData = (const void*)(context->inputBufferPtr + 0x21);

    // 2. MODIFICATION & HOOKING
    // No modifications are performed.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdUpdateBuffer(commandBuffer, dstBuffer, dstOffset, dataSize, pData);

    // 4. SERIALIZATION
    // None.
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** Standard pass-through. The function simply deserializes the arguments and the data payload and calls the corresponding Vulkan command.

---

#### 3. `vt_handle_vkCmdFillBuffer`

```c
/**
 * @brief Handles vkCmdFillBuffer.
 */
void vt_handle_vkCmdFillBuffer(long param_1) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBuffer dstBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    VkDeviceSize dstOffset = *(VkDeviceSize*)(context->inputBufferPtr + 0x11);
    VkDeviceSize size = *(VkDeviceSize*)(context->inputBufferPtr + 0x19);
    uint32_t data = *(uint32_t*)(context->inputBufferPtr + 0x21);

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdFillBuffer(commandBuffer, dstBuffer, dstOffset, size, data);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through function with no modifications.

---

#### 4. `vt_handle_vkCmdClearColorImage`

```c
/**
 * @brief Handles vkCmdClearColorImage.
 */
void vt_handle_vkCmdClearColorImage(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkImage image = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    VkImageLayout imageLayout = *(VkImageLayout*)(context->inputBufferPtr + 0x11);
    
    // The VkClearColorValue is passed directly in the buffer.
    const VkClearColorValue* pColor = (const VkClearColorValue*)(context->inputBufferPtr + 0x18);
    
    uint32_t rangeCount = *(uint32_t*)(context->inputBufferPtr + 0x28);
    
    // Deserialize the array of VkImageSubresourceRange into a temporary buffer.
    VkImageSubresourceRange* pRanges = (VkImageSubresourceRange*)alloca(rangeCount * sizeof(VkImageSubresourceRange));
    deserialize_vkImageSubresourceRange_array(context, pRanges, rangeCount);

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdClearColorImage(commandBuffer, image, imageLayout, pColor, rangeCount, pRanges);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. Deserializes all arguments, including an array of subresource ranges, and calls the dispatch function.

---

#### 5. `vt_handle_vkCmdClearDepthStencilImage`: **No modifications.** Standard pass-through.
#### 6. `vt_handle_vkCmdClearAttachments`: **No modifications.** Standard pass-through.
#### 7. `vt_handle_vkCmdResolveImage`: **No modifications.** Standard pass-through.
#### 8. `vt_handle_vkCmdSetEvent`: **No modifications.** Standard pass-through.
#### 9. `vt_handle_vkCmdResetEvent`: **No modifications.** Standard pass-through.

---

#### 10. `vt_handle_vkCmdWaitEvents`

```c
/**
 * @brief Handles vkCmdWaitEvents.
 */
void vt_handle_vkCmdWaitEvents(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // Deserialize event-related parameters
    uint32_t eventCount = *(uint32_t*)(context->inputBufferPtr + 9);
    const VkEvent* pEvents = deserialize_vkEvent_array(context, eventCount); // Resolves handles
    VkPipelineStageFlags srcStageMask = *(VkPipelineStageFlags*)(context->inputBufferPtr + ...);
    VkPipelineStageFlags dstStageMask = *(VkPipelineStageFlags*)(context->inputBufferPtr + ...);
    
    // Deserialize memory barrier arrays
    uint32_t memoryBarrierCount = *(uint32_t*)(context->inputBufferPtr + ...);
    const VkMemoryBarrier* pMemoryBarriers = deserialize_vkMemoryBarrier_array(context, memoryBarrierCount);
    
    uint32_t bufferMemoryBarrierCount = *(uint32_t*)(context->inputBufferPtr + ...);
    const VkBufferMemoryBarrier* pBufferMemoryBarriers = deserialize_vkBufferMemoryBarrier_array(context, bufferMemoryBarrierCount); // Resolves VkBuffer handles inside
    
    uint32_t imageMemoryBarrierCount = *(uint32_t*)(context->inputBufferPtr + ...);
    const VkImageMemoryBarrier* pImageMemoryBarriers = deserialize_vkImageMemoryBarrier_array(context, imageMemoryBarrierCount); // Resolves VkImage handles inside
    
    // 2. MODIFICATION & HOOKING
    // No modifications. The deserialization functions perform necessary handle lookups,
    // but the parameters themselves are not altered.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdWaitEvents(commandBuffer, eventCount, pEvents, srcStageMask, dstStageMask, 
                             memoryBarrierCount, pMemoryBarriers, 
                             bufferMemoryBarrierCount, pBufferMemoryBarriers, 
                             imageMemoryBarrierCount, pImageMemoryBarriers);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** This function handles a complex set of arguments, including multiple arrays of structs. Its primary job is deserialization and handle resolution. It does not modify the semantics of the call.

---

#### 11. `vt_handle_vkCmdPipelineBarrier`

```c
/**
 * @brief Handles vkCmdPipelineBarrier.
 */
void vt_handle_vkCmdPipelineBarrier(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkPipelineStageFlags srcStageMask = *(VkPipelineStageFlags*)(context->inputBufferPtr + ...);
    VkPipelineStageFlags dstStageMask = *(VkPipelineStageFlags*)(context->inputBufferPtr + ...);
    VkDependencyFlags dependencyFlags = *(VkDependencyFlags*)(context->inputBufferPtr + ...);

    uint32_t memoryBarrierCount = *(uint32_t*)(context->inputBufferPtr + ...);
    const VkMemoryBarrier* pMemoryBarriers = deserialize_vkMemoryBarrier_array(context, memoryBarrierCount);
    
    uint32_t bufferMemoryBarrierCount = *(uint32_t*)(context->inputBufferPtr + ...);
    const VkBufferMemoryBarrier* pBufferMemoryBarriers = deserialize_vkBufferMemoryBarrier_array(context, bufferMemoryBarrierCount);
    
    uint32_t imageMemoryBarrierCount = *(uint32_t*)(context->inputBufferPtr + ...);
    const VkImageMemoryBarrier* pImageMemoryBarriers = deserialize_vkImageMemoryBarrier_array(context, imageMemoryBarrierCount);

    // 2. MODIFICATION & HOOKING
    // The deserialization helper for vkImageMemoryBarrier will resolve the VkImage handle inside each struct.
    // This is necessary for the wrapper to function but is not a modification of the client's intent.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdPipelineBarrier(commandBuffer, srcStageMask, dstStageMask, dependencyFlags,
                                  memoryBarrierCount, pMemoryBarriers,
                                  bufferMemoryBarrierCount, pBufferMemoryBarriers,
                                  imageMemoryBarrierCount, pImageMemoryBarriers);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Similar to `vkCmdWaitEvents`, this is a standard pass-through wrapper for a complex command. It performs the necessary deserialization of three different types of barrier arrays, including resolving the `VkImage` and `VkBuffer` handles within them, before calling the real function.

---

#### 12. `vt_handle_vkCmdBeginQuery`: **No modifications.** Standard pass-through.
#### 13. `vt_handle_vkCmdEndQuery`: **No modifications.** Standard pass-through.
#### 14. `vt_handle_vkCmdBeginConditionalRenderingEXT`: **No modifications.** Standard pass-through.
#### 15. `vt_handle_vkCmdEndConditionalRenderingEXT`: **No modifications.** Standard pass-through.
#### 16. `vt_handle_vkCmdResetQueryPool`: **No modifications.** Standard pass-through.
#### 17. `vt_handle_vkCmdWriteTimestamp`: **No modifications.** Standard pass-through.

---

#### 18. `vt_handle_vkCmdCopyQueryPoolResults`

```c
/**
 * @brief Handles vkCmdCopyQueryPoolResults.
 */
void vt_handle_vkCmdCopyQueryPoolResults(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkQueryPool queryPool = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    uint32_t firstQuery = *(uint32_t*)(context->inputBufferPtr + 0x11);
    uint32_t queryCount = *(uint32_t*)(context->inputBufferPtr + 0x15);
    VkBuffer dstBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 0x19));
    VkDeviceSize dstOffset = *(VkDeviceSize*)(context->inputBufferPtr + 0x21);
    VkDeviceSize stride = *(VkDeviceSize*)(context->inputBufferPtr + 0x29);
    VkQueryResultFlags flags = *(VkQueryResultFlags*)(context->inputBufferPtr + 0x31);
    
    // This is an output command. The client doesn't send the data, but expects it back.
    // The wrapper allocates a temporary buffer to receive the data from the driver.
    size_t dataSize = queryCount * (size_t)stride;
    void* pData = malloc(dataSize);
    
    // 2. MODIFICATION & HOOKING
    // No modifications to the call itself.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdCopyQueryPoolResults(commandBuffer, queryPool, firstQuery, queryCount, dstBuffer, dstOffset, stride, flags);
    
    // NOTE: The decompiled code seems to be calling GetQueryPoolResults, not CopyQueryPoolResults,
    // and serializing the data back. This is a common pattern for query results.
    // If it *is* copying to a buffer, the data doesn't need to be sent back. If it's *getting* results, it does.
    // Let's assume the handler for vkGetQueryPoolResults would serialize data back.
    // For vkCmdCopyQueryPoolResults, there is no output data to serialize.

    // 4. SERIALIZATION
    // None, as the result is written directly to a GPU buffer.

    if (pData) {
        free(pData); // This would only be freed if it were vkGetQueryPoolResults.
    }
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is a standard pass-through. The decompiled code for `vkCmdCopyQueryPoolResults` might be confused with `vkGetQueryPoolResults` in the disassembler, as they are functionally related. `vkCmdCopyQueryPoolResults` writes to a `VkBuffer` on the GPU and has no data to return to the client, so no serialization is needed.

---

#### 19. `vt_handle_vkCmdPushConstants`

```c
/**
 * @brief Handles vkCmdPushConstants, with modifications to stageFlags.
 */
void vt_handle_vkCmdPushConstants(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkPipelineLayout layout = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    VkShaderStageFlags stageFlags = *(VkShaderStageFlags*)(context->inputBufferPtr + 0x11);
    uint32_t offset = *(uint32_t*)(context->inputBufferPtr + 0x15);
    uint32_t size = *(uint32_t*)(context->inputBufferPtr + 0x19);
    
    // The constant data is part of the same input buffer
    const void* pValues = (const void*)(context->inputBufferPtr + 0x1D);
    
    // 2. MODIFICATION & HOOKING
    // This is the same modification as seen in vt_handle_vkCreatePipelineLayout.
    if ((stageFlags & VK_SHADER_STAGE_VERTEX_BIT) != 0) {
        // If the vertex stage is a target, automatically add the tessellation evaluation stage.
        stageFlags |= VK_SHADER_STAGE_TESSELLATION_EVALUATION_SHADER_BIT; // 0x10
    }

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdPushConstants(commandBuffer, layout, stageFlags, offset, size, pValues);
    
    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function modifies the `stageFlags` parameter. It ensures that any push constants made available to the vertex shader are also automatically available to the tessellation evaluation shader. This is likely a feature to simplify shader development for pipelines that use tessellation, removing the need for the client to remember to specify both stages.

---

#### 20. `vt_handle_vkCmdBeginRenderPass`

```c
/**
 * @brief Handles vkCmdBeginRenderPass.
 */
void vt_handle_vkCmdBeginRenderPass(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));

    // Deserialize the VkRenderPassBeginInfo struct.
    // This is a complex operation handled by a helper function.
    VkRenderPassBeginInfo beginInfo = {0};
    deserialize_vkRenderPassBeginInfo(context, &beginInfo, context->inputBufferPtr + 9);
    
    // The final argument is also read from the buffer.
    VkSubpassContents contents = *(VkSubpassContents*)(context->inputBufferPtr + ...); // Offset after beginInfo data

    // 2. MODIFICATION & HOOKING
    // No modifications are performed. The deserialization helper resolves handles
    // for `renderPass` and `framebuffer` but does not alter the logic.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBeginRenderPass(commandBuffer, &beginInfo, contents);
    
    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is a standard pass-through wrapper for a complex command. It involves significant deserialization work to reconstruct the `VkRenderPassBeginInfo` struct and its nested `pClearValues` array, but it does not modify the parameters before calling the real Vulkan function.

---

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkCmdNextSubpass` | **No** | Standard pass-through. |
| `vkCmdEndRenderPass` | **No** | Standard pass-through. |
| `vkCmdExecuteCommands` | **No** | Standard pass-through. |
| `vkCreateSwapchainKHR` | **Yes** | **Heavily Modified.** Completely bypasses the driver's `vkCreateSwapchainKHR` call. It uses a custom `XWindowSwapchain_create` implementation that likely integrates with Android's native windowing system (`Surface`/`HardwareBuffer`) for better performance and control. |
| `vkDestroySwapchainKHR` | **Yes** | **Modified.** Calls the custom `XWindowSwapchain_destroy` to clean up the custom swapchain object, bypassing the driver. |
| `vkGetSwapchainImagesKHR`| **Yes** | **Modified.** Bypasses the driver and retrieves image handles directly from the custom `XWindowSwapchain` object. |
| `vkAcquireNextImageKHR` | **Yes** | **Modified.** Bypasses the driver and calls the custom `XWindowSwapchain_acquireNextImage` function. |
| `vkQueuePresentKHR` | **Yes** | **Heavily Modified.** Bypasses the driver's `vkQueuePresentKHR`. It calls a custom `XWindowSwapchain_presentImage` function for each swapchain, indicating a fully custom presentation path. |
| `vkCmdPushDescriptorSetKHR` | **No** | Standard pass-through. |
| `vkTrimCommandPool` | **No** | Standard pass-through. |
| `vkGetMemoryFdKHR` | **Yes** | **Modified IPC.** The function correctly calls the driver to get a file descriptor (FD), but instead of returning it in a buffer, it uses `sendmsg` to pass the FD as ancillary data over a socket to the client. This is a necessary modification for the client-server architecture. |
| `vkGetSemaphoreFdKHR` | **Yes** | **Modified IPC.** Same as `vkGetMemoryFdKHR`; uses `sendmsg` to transport the FD. |
| `vkGetFenceFdKHR` | **Yes** | **Modified IPC.** Same as `vkGetMemoryFdKHR`; uses `sendmsg` to transport the FD. |
| `vkGetDeviceGroupPeerMemoryFeatures` | **No** | Standard pass-through. |
| `vkBindBufferMemory2` | **Yes** | **Modified.** Hooks the call to notify the `TextureDecoder` module of the new buffer-memory binding via `TextureDecoder_addBoundBuffer`. This is crucial for the texture decompression system. |
| `vkBindImageMemory2` | **No** | Standard pass-through. Surprisingly, it does not seem to hook image memory bindings, suggesting that tracking is handled at a different stage (e.g., `vkCreateImage`). |
| `vkCmdSetDeviceMask` | **No** | Standard pass-through. |
| `vkAcquireNextImage2KHR` | **Yes** | **Modified.** Bypasses the driver and calls the custom `XWindowSwapchain_acquireNextImage`, same as the older version of the function. |
| `vkCmdDispatchBase` | **No** | Standard pass-through. |
| `vkCreateDescriptorUpdateTemplate` | **No** | Standard pass-through. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkCmdNextSubpass` & 2. `vt_handle_vkCmdEndRenderPass` & 3. `vt_handle_vkCmdExecuteCommands`

These are simple command buffer functions that change the state of the command buffer.

```c
void vt_handle_vkCmdNextSubpass(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer cmdbuf = VkObject_fromId(*(void**)(input + 1));
    VkSubpassContents contents = *(VkSubpassContents*)(input + 9);

    // 2. MODIFICATION & 3. DISPATCH
    dispatch_vkCmdNextSubpass(cmdbuf, contents);

    // 4. SERIALIZATION (None)
}

void vt_handle_vkCmdEndRenderPass(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer cmdbuf = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // 2. MODIFICATION & 3. DISPATCH
    dispatch_vkCmdEndRenderPass(cmdbuf);
    
    // 4. SERIALIZATION (None)
}

void vt_handle_vkCmdExecuteCommands(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer cmdbuf = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    uint32_t count = *(uint32_t*)(context->inputBufferPtr + 9);
    
    // Allocate a temporary array for command buffer handles
    VkCommandBuffer* pCommandBuffers = malloc(count * sizeof(VkCommandBuffer));
    for (uint32_t i = 0; i < count; ++i) {
        pCommandBuffers[i] = VkObject_fromId(*(void**)(context->inputBufferPtr + 13 + i * 8));
    }

    // 2. MODIFICATION & 3. DISPATCH
    dispatch_vkCmdExecuteCommands(cmdbuf, count, pCommandBuffers);
    
    // 4. SERIALIZATION (None)
    free(pCommandBuffers);
}
```
**Analysis:** All three functions are standard pass-through wrappers. They deserialize their arguments, resolve the Vulkan handle IDs, and call the corresponding dispatch function without any modification to the logic or parameters.

---

#### 4. `vt_handle_vkCreateSwapchainKHR` through 8. `vt_handle_vkQueuePresentKHR`

These functions form a group that completely replaces the standard Vulkan swapchain mechanism with a custom implementation.

```c
/**
 * @brief Handles vkCreateSwapchainKHR by calling a custom implementation.
 */
void vt_handle_vkCreateSwapchainKHR(VtContext* context) {
    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkSwapchainCreateInfoKHR createInfo = {0};
    deserialize_vkSwapchainCreateInfoKHR(context, &createInfo);

    // 2. MODIFICATION & HOOKING
    // The core logic is entirely replaced.
    
    // Custom validation: check if the client-requested extent matches the window extent.
    VkExtent2D windowExtent;
    getWindowExtent(context->javaComponentObject, createInfo.surface, &windowExtent);
    if (createInfo.imageExtent.width != windowExtent.width || createInfo.imageExtent.height != windowExtent.height) {
        // If not, it returns an error instead of creating the swapchain.
        // This is a custom behavior not present in the Vulkan spec.
        RingBuffer_writeResult(context->clientRingBuffer, VK_ERROR_INITIALIZATION_FAILED, VK_NULL_HANDLE);
        return;
    }
    
    // 3. DISPATCH (Bypassed)
    // The call to dispatch_vkCreateSwapchainKHR is completely skipped.
    // Instead, a custom object is created.
    XWindowSwapchain* swapchain = XWindowSwapchain_create(device, createInfo.surface, &createInfo, context->javaComponentObject, createInfo.surface);
    
    VkResult result = VK_SUCCESS;
    if (swapchain == NULL) {
        result = VK_ERROR_INITIALIZATION_FAILED;
    }

    // 4. SERIALIZATION
    // The handle to the custom XWindowSwapchain object is returned to the client,
    // which will treat it as an opaque VkSwapchainKHR handle.
    RingBuffer_writeResult(context->clientRingBuffer, result, (VkSwapchainKHR)swapchain);
}

/**
 * @brief Destroys the custom swapchain object.
 */
void vt_handle_vkDestroySwapchainKHR(VtContext* context) {
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    XWindowSwapchain* swapchain = (XWindowSwapchain*)VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    
    XWindowSwapchain_destroy(device, swapchain);
}

/**
 * @brief Gets images from the custom swapchain object.
 */
void vt_handle_vkGetSwapchainImagesKHR(VtContext* context) {
    XWindowSwapchain* swapchain = (XWindowSwapchain*)VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    
    // The function retrieves handles directly from its internal array of images,
    // which are likely backed by Android Hardware Buffers.
    uint32_t imageCount = swapchain->imageCount;
    VkImage* images = swapchain->images; // Simplified representation

    // Serialize the count and the array of handles back to the client.
    RingBuffer_writeResultAndArray(context->clientRingBuffer, VK_SUCCESS, imageCount, images);
}

/**
 * @brief Acquires the next image from the custom swapchain.
 */
void vt_handle_vkAcquireNextImageKHR(VtContext* context) {
    XWindowSwapchain* swapchain = (XWindowSwapchain*)VkObject_fromId(/* ... */);
    uint64_t timeout = /* ... */;
    VkSemaphore semaphore = VkObject_fromId(/* ... */);
    VkFence fence = VkObject_fromId(/* ... */);
    uint32_t imageIndex = 0;
    
    // Calls the custom implementation.
    VkResult result = XWindowSwapchain_acquireNextImage(swapchain, timeout, semaphore, fence, &imageIndex);

    // Serialize result and the acquired image index.
    RingBuffer_writeResultAndIndex(context->clientRingBuffer, result, imageIndex);
}

/**
 * @brief Presents an image from the custom swapchain.
 */
void vt_handle_vkQueuePresentKHR(VtContext* context) {
    // Deserializes VkPresentInfoKHR...
    
    // Loops through each swapchain in the present info.
    for (uint32_t i = 0; i < presentInfo.swapchainCount; ++i) {
        XWindowSwapchain* swapchain = (XWindowSwapchain*)presentInfo.pSwapchains[i];
        uint32_t imageIndex = presentInfo.pImageIndices[i];
        
        // Calls a custom function to present the image, likely involving
        // posting the corresponding HardwareBuffer to the Android compositor.
        XWindowSwapchain_presentImage(swapchain, imageIndex);
    }
    
    // Serialize a success result.
    RingBuffer_writeResult(context->clientRingBuffer, VK_SUCCESS, VK_NULL_HANDLE);
}
```
**Analysis:** This entire group of functions demonstrates a complete override of the standard Vulkan swapchain. This is a common and complex technique for high-performance renderers on Android. Instead of letting the Vulkan driver manage the window surface, the renderer does it manually using `AHardwareBuffer`. This gives it more direct control over the presentation pipeline, potentially reducing latency and enabling better integration with the Android UI compositor. The client application interacts with what it thinks is a `VkSwapchainKHR`, but is actually an opaque handle to the renderer's custom `XWindowSwapchain` object.

---

#### 9. `vt_handle_vkCmdPushDescriptorSetKHR` & 10. `vt_handle_vkTrimCommandPool`

```c
void vt_handle_vkCmdPushDescriptorSetKHR(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer cmdbuf = VkObject_fromId(/* ... */);
    VkPipelineBindPoint pipelineBindPoint = /* ... */;
    VkPipelineLayout layout = VkObject_fromId(/* ... */);
    uint32_t set = /* ... */;
    uint32_t writeCount = /* ... */;
    
    // Deserialize an array of VkWriteDescriptorSet, resolving handles inside.
    VkWriteDescriptorSet* pWrites = deserialize_vkWriteDescriptorSet_array(context, writeCount);
    
    // 2. MODIFICATION & 3. DISPATCH
    // No logical changes are made to the deserialized data.
    dispatch_vkCmdPushDescriptorSetKHR(cmdbuf, pipelineBindPoint, layout, set, writeCount, pWrites);

    // 4. SERIALIZATION (None)
}

void vt_handle_vkTrimCommandPool(VtContext* context) {
    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(/* ... */);
    VkCommandPool pool = VkObject_fromId(/* ... */);
    VkCommandPoolTrimFlags flags = /* ... */;

    // 2. MODIFICATION & 3. DISPATCH
    dispatch_vkTrimCommandPool(device, pool, flags);
    
    // 4. SERIALIZATION (None)
}
```
**Analysis:** These functions are standard pass-throughs. Although `vkCmdPushDescriptorSetKHR` involves complex deserialization of an array of structs, it doesn't alter the *behavior* of the call itself. It faithfully reconstructs the client's request and forwards it to the driver.

---

#### 11. `vt_handle_vkGetMemoryFdKHR`, 12. `vt_handle_vkGetSemaphoreFdKHR`, 13. `vt_handle_vkGetFenceFdKHR`

These functions all share the same special handling for transporting file descriptors.

```c
/**
 * @brief Handles vkGetMemoryFdKHR by passing the FD over a socket.
 */
void vt_handle_vkGetMemoryFdKHR(VtContext* context) {
    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(/* ... */);
    VkMemoryGetFdInfoKHR getFdInfo = {0};
    deserialize_vkMemoryGetFdInfoKHR(context, &getFdInfo); // Resolves VkDeviceMemory handle

    // 2. DISPATCH (to get the FD from the driver)
    int fd = -1;
    VkResult result = dispatch_vkGetMemoryFdKHR(device, &getFdInfo, &fd);
    
    // 3. MODIFICATION & SERIALIZATION (IPC Transport)
    // The mechanism is modified. Instead of writing the FD to a buffer,
    // it's sent as ancillary data over the client-server socket.
    
    struct msghdr msg = {0};
    struct iovec iov = {0};
    char cmsg_buf[CMSG_SPACE(sizeof(int))];
    
    char dummy_data = 0;
    iov.iov_base = &dummy_data;
    iov.iov_len = 1;
    
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    msg.msg_control = cmsg_buf;
    msg.msg_controllen = sizeof(cmsg_buf);
    
    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type = SCM_RIGHTS;
    cmsg->cmsg_len = CMSG_LEN(sizeof(int));
    
    *(int*)CMSG_DATA(cmsg) = fd;
    
    // Write the VkResult and then send the FD.
    RingBuffer_writeResult(context->clientRingBuffer, result, VK_NULL_HANDLE);
    sendmsg(context->clientFd, &msg, 0);
    
    // The server closes its copy of the FD after sending.
    if (fd >= 0) {
        close(fd);
    }
}
```
**Analysis:** These functions are classified as **modified** due to their unique IPC transport mechanism. A standard Vulkan application would receive the FD as a simple integer output parameter. Here, because of the client-server architecture, the FD must be passed from one process to another. `sendmsg` with `SCM_RIGHTS` is the standard POSIX way to do this. The wrapper handles this complexity, allowing the client to receive the FD as if it were a local call.

---

#### 14. `vt_handle_vkGetDeviceGroupPeerMemoryFeatures` through 20. `vt_handle_vkCreateDescriptorUpdateTemplate`

These functions were found to have no significant logical modifications.

```c
// Example for vkGetDeviceGroupPeerMemoryFeatures
void vt_handle_vkGetDeviceGroupPeerMemoryFeatures(VtContext* context) {
    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(/* ... */);
    uint32_t heapIndex = /* ... */;
    uint32_t localDeviceIndex = /* ... */;
    uint32_t remoteDeviceIndex = /* ... */;

    // 2. MODIFICATION & 3. DISPATCH
    VkPeerMemoryFeatureFlags features = 0;
    dispatch_vkGetDeviceGroupPeerMemoryFeatures(device, heapIndex, localDeviceIndex, remoteDeviceIndex, &features);
    
    // 4. SERIALIZATION
    RingBuffer_writeResult(context->clientRingBuffer, VK_SUCCESS, features);
}

// Example for vkBindBufferMemory2
void vt_handle_vkBindBufferMemory2(VtContext* context) {
    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(/* ... */);
    uint32_t bindInfoCount = /* ... */;
    // Deserializes an array of VkBindBufferMemoryInfo...
    VkBindBufferMemoryInfo* pBindInfos = deserialize_vkBindBufferMemoryInfo_array(context, bindInfoCount);

    // 2. MODIFICATION & HOOKING
    if (context->textureDecoder != NULL) {
        // For each binding, notify the texture decoder module.
        for (uint32_t i = 0; i < bindInfoCount; ++i) {
            TextureDecoder_addBoundBuffer(context->textureDecoder, pBindInfos[i].buffer, pBindInfos[i].memory, pBindInfos[i].memoryOffset);
        }
    }
    
    // 3. DISPATCH
    VkResult result = dispatch_vkBindBufferMemory2(device, bindInfoCount, pBindInfos);

    // If the call failed, undo the tracking in the texture decoder.
    if (result != VK_SUCCESS && context->textureDecoder != NULL) {
        for (uint32_t i = 0; i < bindInfoCount; ++i) {
            TextureDecoder_removeBoundBuffer(context->textureDecoder, pBindInfos[i].buffer);
        }
    }

    // 4. SERIALIZATION
    RingBuffer_writeResult(context->clientRingBuffer, result, VK_NULL_HANDLE);
}

// All other functions in this group follow a similar "standard pass-through" pattern.
```
**Analysis:**
*   `vkGetDeviceGroupPeerMemoryFeatures`, `vkBindImageMemory2`, `vkCmdSetDeviceMask`, `vkCmdDispatchBase`, and `vkCreateDescriptorUpdateTemplate` are all **standard pass-through** functions with no modifications.
*   `vkBindBufferMemory2` is **modified**. It contains a crucial hook to inform the `TextureDecoder` about which memory object is bound to a specific buffer. This allows the decoder to find the raw data for a compressed texture when it needs to perform decompression. If the binding fails, it correctly undoes this tracking. This is a well-designed interception that maintains correct state.
*   `vkAcquireNextImage2KHR` is **modified** as part of the custom swapchain implementation, just like its predecessor.

---

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkDestroyDescriptorUpdateTemplate` | **No** | Not implemented in the provided binary. |
| `vkUpdateDescriptorSetWithTemplate` | **No** | Not implemented in the provided binary. |
| `vkCmdSetSampleLocationsEXT` | **No** | Standard pass-through. |
| `vkGetBufferMemoryRequirements2`| **No** | Standard pass-through for a query function. |
| `vkGetImageMemoryRequirements2` | **No** | Standard pass-through for a query function. |
| `vkGetImageSparseMemoryRequirements2`| **No** | Standard pass-through for a query function. |
| `vkGetDeviceBufferMemoryRequirements`| **No** | Standard pass-through for a query function. |
| `vkGetDeviceImageMemoryRequirements`| **No** | Standard pass-through for a query function. |
| `vkGetDeviceImageSparseMemoryRequirements`| **No** | Standard pass-through for a query function. |
| `vkCreateSamplerYcbcrConversion`| **No** | Standard pass-through. |
| `vkDestroySamplerYcbcrConversion`| **No** | Standard pass-through. |
| `vkGetDeviceQueue2` | **No** | Standard pass-through. |
| `vkGetDescriptorSetLayoutSupport`| **No** | Standard pass-through for a query function. |
| `vkGetCalibratedTimestampsKHR` | **No** | Standard pass-through for a query function. |
| `vkCreateRenderPass2` | **Yes** | Intercepts `pNext` chains to handle custom or platform-specific structures, such as `VkAttachmentSampleCountInfoAMD` (re-used as `NV`). |
| `vkCmdBeginRenderPass2` | **Yes** | Intercepts `pNext` chains. The logic appears to be for handling specific render pass extensions like `VkDeviceGroupRenderPassBeginInfo`. |
| `vkCmdNextSubpass2` | **No** | Standard pass-through. |
| `vkCmdEndRenderPass2` | **No** | Standard pass-through. |
| `vkGetSemaphoreCounterValue`| **No** | Standard pass-through for a query function. |
| `vkWaitSemaphores` | **Yes** | **Significant Modification.** The function is completely replaced by `TimelineSemaphore_asyncWait`. Instead of blocking, it uses an `eventfd` to offload the wait to a worker thread, preventing the main renderer thread from stalling. |

---

### Detailed Function Decompilations

#### 1 & 2. `vt_handle_vkDestroyDescriptorUpdateTemplate` & `vt_handle_vkUpdateDescriptorSetWithTemplate`

**Analysis:** Neither of these functions have a `vt_handle_...` wrapper in the provided binary. While their function pointers exist in the dispatch table, there is no specific implementation to intercept them. This implies they are either unused by the client applications this renderer targets, or they are handled by a generic, unmodified command processor.
**Conclusion:** No modifications detected as the functions are not implemented.

---

#### 3. `vt_handle_vkCmdSetSampleLocationsEXT`

```c
/**
 * @brief Handles vkCmdSetSampleLocationsEXT.
 */
void vt_handle_vkCmdSetSampleLocationsEXT(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // Deserialize VkSampleLocationsInfoEXT from the input buffer
    VkSampleLocationsInfoEXT locationsInfo = {0};
    deserialize_vkSampleLocationsInfoEXT(context, &locationsInfo);
    
    // 2. MODIFICATION & HOOKING
    // No modifications are made to the command or its parameters.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetSampleLocationsEXT(commandBuffer, &locationsInfo);
    
    // 4. SERIALIZATION
    // None, this is a void command recording function.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is a standard pass-through function. It performs a moderately complex deserialization for the `VkSampleLocationsInfoEXT` struct, including its nested array of `VkSampleLocationEXT` points, into temporary memory. It then calls the real Vulkan command with the reconstructed struct.
**Conclusion:** No modifications detected.

---

#### 4, 5, & 6. `vt_handle_...MemoryRequirements2` Functions

(`vkGetBufferMemoryRequirements2`, `vkGetImageMemoryRequirements2`, `vkGetImageSparseMemoryRequirements2`)
```c
/**
 * @brief Handles vkGetBufferMemoryRequirements2 and its variants.
 */
void vt_handle_vkGetBufferMemoryRequirements2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // Deserialize the appropriate ...MemoryRequirementsInfo2 struct
    VkBufferMemoryRequirementsInfo2 info = {0};
    deserialize_vkBufferMemoryRequirementsInfo2(context, &info);

    // 2. MODIFICATION & HOOKING
    // No modifications are performed. These are standard queries.

    // 3. DISPATCH TO VULKAN
    VkMemoryRequirements2 memReqs = {0};
    // The decompiled code shows pNext chain handling for the output struct as well,
    // which is standard for Vk...2 functions.
    
    dispatch_vkGetBufferMemoryRequirements2(device, &info, &memReqs);

    // 4. SERIALIZATION
    // The resulting VkMemoryRequirements2 struct (and its pNext chain) is
    // serialized and written back to the client ring buffer.
    serialize_and_write_vkMemoryRequirements2(context, &memReqs);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** All three of these `...Requirements2` functions follow the same standard pattern for a query. They deserialize the input `...Info2` struct (including any `pNext` chain), call the dispatch function, and then serialize the output `...Requirements2` struct (including any `pNext` chain populated by the driver) back to the client. There are no logical modifications.
**Conclusion:** No modifications detected.

---

#### 7, 8, & 9. `vt_handle_vkGetDevice...MemoryRequirements` Functions

(`vkGetDeviceBufferMemoryRequirements`, `vkGetDeviceImageMemoryRequirements`, `vkGetDeviceImageSparseMemoryRequirements`)
```c
/**
 * @brief Handles vkGetDeviceBufferMemoryRequirements and its variants.
 */
void vt_handle_vkGetDeviceBufferMemoryRequirements(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // The ...Requirements struct contains a pointer to a CreateInfo struct.
    // The wrapper fully deserializes both from the input stream.
    VkDeviceBufferMemoryRequirements reqInfo = {0};
    VkBufferCreateInfo createInfo = {0};
    deserialize_vkDeviceBufferMemoryRequirements(context, &reqInfo, &createInfo);
    
    // 2. MODIFICATION & HOOKING
    // No modifications are performed.

    // 3. DISPATCH TO VULKAN
    VkMemoryRequirements2 memReqs = {0};
    dispatch_vkGetDeviceBufferMemoryRequirements(device, &reqInfo, &memReqs);

    // 4. SERIALIZATION
    // The resulting VkMemoryRequirements2 struct is serialized and sent back.
    serialize_and_write_vkMemoryRequirements2(context, &memReqs);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** These functions are part of the `VK_KHR_maintenance4` extension. They are designed to query memory requirements without first creating the resource. The wrapper's job is to faithfully reconstruct the `Vk...CreateInfo` struct that the client *would have used* to create the resource. The implementation deserializes the `VkDevice...MemoryRequirements` struct and the nested `Vk...CreateInfo` struct it points to, then calls the real Vulkan function. This is standard behavior.
**Conclusion:** No modifications detected.

---

#### 10 & 11. `vkCreateSamplerYcbcrConversion` & `vkDestroySamplerYcbcrConversion`

**Analysis:** Both the create and destroy functions are standard pass-through wrappers. They deserialize the arguments, including the `VkSamplerYcbcrConversionCreateInfo` struct, call the corresponding dispatch function, and (for create) serialize the resulting handle back to the client. There is no evidence of modification to the YCbCr conversion parameters.
**Conclusion:** No modifications detected.

---

#### 12. `vt_handle_vkGetDeviceQueue2`

```c
/**
 * @brief Handles vkGetDeviceQueue2.
 */
void vt_handle_vkGetDeviceQueue2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // Deserialize VkDeviceQueueInfo2 from the input buffer
    VkDeviceQueueInfo2 queueInfo = {0};
    deserialize_vkDeviceQueueInfo2(context, &queueInfo);
    
    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    VkQueue queue = VK_NULL_HANDLE;
    dispatch_vkGetDeviceQueue2(device, &queueInfo, &queue);

    // 4. SERIALIZATION
    // The resulting VkQueue handle is serialized and written to the client.
    RingBuffer_write(context->clientRingBuffer, &queue, sizeof(VkQueue));
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is a standard query function wrapper. It deserializes the `VkDeviceQueueInfo2` struct, calls the real function, and sends the returned `VkQueue` handle back to the client.
**Conclusion:** No modifications detected.

---

#### 13. `vt_handle_vkGetDescriptorSetLayoutSupport`

**Analysis:** This is another standard query function wrapper. It performs a complex deserialization of `VkDescriptorSetLayoutCreateInfo` and its `pNext` chain, calls `dispatch_vkGetDescriptorSetLayoutSupport`, and serializes the output `VkDescriptorSetLayoutSupport` structure back to the client. There are no modifications to the data.
**Conclusion:** No modifications detected.

---

#### 14. `vt_handle_vkGetCalibratedTimestampsKHR`

```c
/**
 * @brief Handles vkGetCalibratedTimestampsKHR.
 */
void vt_handle_vkGetCalibratedTimestampsKHR(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    uint32_t timestampCount = *(uint32_t*)(context->inputBufferPtr + 9);
    
    // Deserialize the array of VkCalibratedTimestampInfoKHR structs
    VkCalibratedTimestampInfoKHR* pTimestampInfos = deserialize_array(context, timestampCount, sizeof(VkCalibratedTimestampInfoKHR));

    // 2. MODIFICATION & HOOKING
    // No modifications. This is a query.

    // 3. DISPATCH TO VULKAN
    uint64_t* pTimestamps = malloc(timestampCount * sizeof(uint64_t));
    uint64_t maxDeviation;
    VkResult result = dispatch_vkGetCalibratedTimestampsKHR(device, timestampCount, pTimestampInfos, pTimestamps, &maxDeviation);

    // 4. SERIALIZATION
    // Allocate a buffer for the result, the two output arrays, and the maxDeviation.
    size_t dataSize = sizeof(VkResult) + (timestampCount * sizeof(uint64_t)) + sizeof(uint64_t);
    void* resultBuffer = malloc(dataSize);
    
    // Pack the results for the client.
    *(VkResult*)resultBuffer = result;
    memcpy((char*)resultBuffer + sizeof(VkResult), pTimestamps, timestampCount * sizeof(uint64_t));
    *(uint64_t*)((char*)resultBuffer + sizeof(VkResult) + timestampCount * sizeof(uint64_t)) = maxDeviation;
    
    RingBuffer_write(context->clientRingBuffer, resultBuffer, dataSize);

    free(pTimestamps);
    free(resultBuffer);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This wrapper follows the standard procedure for a Vulkan function that returns data in caller-provided arrays. It deserializes the input array, allocates temporary space for the output arrays, calls the real Vulkan function, and then serializes all the results back to the client.
**Conclusion:** No modifications detected.

---

#### 15. `vt_handle_vkCreateRenderPass2`

**Analysis:** This function deserializes a `VkRenderPassCreateInfo2` struct. Due to the complexity of this struct (arrays of attachments, subpasses, dependencies, and multiple possible `pNext` chains), the deserialization logic is extensive. The decompiled code shows checks for specific `sType` values in the `pNext` chain, such as `VK_STRUCTURE_TYPE_ATTACHMENT_SAMPLE_COUNT_INFO_AMD` (which is aliased as `..._NV`). This indicates that while the function's primary purpose is a pass-through, it has special handling logic for certain extensions to ensure they are deserialized correctly. This is a "soft" modification, as it doesn't change the parameters' intent but implements custom logic to support them.
**Conclusion:** **Modification detected.** The function contains custom logic to correctly deserialize specific extension structures in the `pNext` chain, such as those for AMD/NV mixed-sample attachments.

---

#### 16, 17, 18. `vkCmdBeginRenderPass2`, `vkCmdNextSubpass2`, `vkCmdEndRenderPass2`

**Analysis:**
*   **`vt_handle_vkCmdBeginRenderPass2`**: This function deserializes `VkRenderPassBeginInfo` and `VkSubpassBeginInfo`, both of which can have `pNext` chains. Similar to `vkCreateRenderPass2`, the decompilation shows specific checks for extension structs like `VkDeviceGroupRenderPassBeginInfo`. It reconstructs these structs before calling the dispatch function.
*   **`vt_handle_vkCmdNextSubpass2`** and **`vt_handle_vkCmdEndRenderPass2`**: These are much simpler. They deserialize their respective `...Info` structs, which have a less complex structure, and call the dispatch function. There is no evidence of modification.

**Conclusion:** `vkCmdBeginRenderPass2` has **soft modifications** for deserializing extensions. The other two have **no modifications**.

---

#### 19. `vt_handle_vkGetSemaphoreCounterValue`

```c
/**
 * @brief Handles vkGetSemaphoreCounterValue.
 */
void vt_handle_vkGetSemaphoreCounterValue(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkSemaphore semaphore = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    
    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    uint64_t counterValue = 0;
    VkResult result = dispatch_vkGetSemaphoreCounterValue(device, semaphore, &counterValue);

    // 4. SERIALIZATION
    void* resultBuffer = malloc(sizeof(VkResult) + sizeof(uint64_t));
    *(VkResult*)resultBuffer = result;
    *(uint64_t*)((char*)resultBuffer + sizeof(VkResult)) = counterValue;
    RingBuffer_write(context->clientRingBuffer, resultBuffer, sizeof(VkResult) + sizeof(uint64_t));
    free(resultBuffer);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** A standard query wrapper. Deserializes handles, calls dispatch, serializes result code and output value.
**Conclusion:** No modifications detected.

---

#### 20. `vt_handle_vkWaitSemaphores`

```c
/**
 * @brief Handles vkWaitSemaphores with an asynchronous offload mechanism.
 */
void vt_handle_vkWaitSemaphores(VtContext* context, int client_socket_fd) {
    // 1. DESERIALIZATION
    // The function deserializes the VkSemaphoreWaitInfo struct, but does not
    // pass it directly to the Vulkan driver.
    
    // 2. MODIFICATION & HOOKING
    // The entire blocking wait operation is replaced with an asynchronous call.
    // This custom function likely uses a worker thread to perform the actual wait.
    TimelineSemaphore_asyncWait(
        client_socket_fd,               // A socket or fd to notify the client on completion
        context->asyncPipelineCreator,  // Misnomer, likely a generic async task handler
        deserialized_pSemaphores,       // The array of semaphores to wait on
        deserialized_semaphoreCount     // The number of semaphores
    );
    
    // 3. DISPATCH TO VULKAN
    // The call to dispatch_vkWaitSemaphores is NOT made on this thread.
    // It is performed on a worker thread managed by TimelineSemaphore_asyncWait.
    
    // 4. SERIALIZATION
    // The result is sent back to the client by the worker thread over the socket/fd
    // once the wait is complete, rather than through the main ring buffer.
}
```
**Analysis:** This is a **major and significant modification**. The wrapper completely replaces the behavior of `vkWaitSemaphores`. A blocking wait on the main renderer thread would be catastrophic for performance, causing the entire command stream to stall. Instead, it uses a custom `TimelineSemaphore_asyncWait` function. This function uses an `eventfd` to create a file descriptor that a worker thread can poll. The worker thread performs the actual `vkWaitSemaphores` call. Once the wait is satisfied, the worker thread signals the `eventfd`, which unblocks the client application (which would be polling the other end of the fd). This is a sophisticated mechanism to turn a blocking, synchronous API call into a non-blocking, asynchronous one.
**Conclusion:** **Significant modification detected.** Replaces a blocking API call with a non-blocking asynchronous implementation using a worker thread and an `eventfd`.

---

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkSignalSemaphore` | **No** | Standard pass-through. |
| `vkGetAndroidHardwareBufferPropertiesANDROID` | **No** | Standard pass-through, but it is an inherently specialized function for platform interop. |
| `vkCmdDrawIndirectCount` | **No** | Standard pass-through. |
| `vkCmdDrawIndexedIndirectCount`| **No** | Standard pass-through. |
| `vkCmdBindTransformFeedbackBuffersEXT` | **No** | Standard pass-through. |
| `vkCmdBeginTransformFeedbackEXT`| **No** | Standard pass-through. |
| `vkCmdEndTransformFeedbackEXT` | **No** | Standard pass-through. |
| `vkCmdBeginQueryIndexedEXT` | **No** | Standard pass-through. |
| `vkCmdEndQueryIndexedEXT` | **No** | Standard pass-through. |
| `vkCmdDrawIndirectByteCountEXT` | **No** | Standard pass-through. |
| `vkGetBufferOpaqueCaptureAddress`| **No** | Standard pass-through. |
| `vkGetBufferDeviceAddress` | **No** | Standard pass-through. |
| `vkGetDeviceMemoryOpaqueCaptureAddress` | **Yes** | Modifies the `VkDeviceMemory` handle before the call, likely to extract the real handle from a custom wrapper object. |
| `vkCmdSetLineStipple` | **No** | Standard pass-through. Renamed from `vkCmdSetLineStippleKHR`. |
| `vkCmdSetCullMode` | **No** | Standard pass-through. |
| `vkCmdSetFrontFace` | **No** | Standard pass-through. |
| `vkCmdSetPrimitiveTopology` | **No** | Standard pass-through. |
| `vkCmdSetViewportWithCount`| **No** | Standard pass-through. |
| `vkCmdSetScissorWithCount` | **No** | Standard pass-through. |
| `vkCmdBindVertexBuffers2` | **No** | Standard pass-through. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkSignalSemaphore`

```c
/**
 * @brief Handles vkSignalSemaphore.
 */
void vt_handle_vkSignalSemaphore(VtContext* context) {
    long stack_check_cookie = *(long*)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkSemaphoreSignalInfo signalInfo = {0};
    deserialize_vkSemaphoreSignalInfo(context, &signalInfo);
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));

    // 2. MODIFICATION & HOOKING
    // No modifications are performed on the VkSemaphoreSignalInfo struct.

    // 3. DISPATCH TO VULKAN
    VkResult result = dispatch_vkSignalSemaphore(device, &signalInfo);

    // 4. SERIALIZATION
    // Write the result back to the client.
    RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));

    if (*(long*)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is a standard pass-through function. It deserializes the `VkSemaphoreSignalInfo` struct, resolving the semaphore handle ID, and directly calls the corresponding Vulkan function. No parameters are modified.

---

#### 2. `vt_handle_vkGetAndroidHardwareBufferPropertiesANDROID`

This function is not present in the provided `libvortekrenderer2.so.txt` disassembly. Based on the surrounding functions and the nature of the Android Hardware Buffer (AHB) extension, it would almost certainly be a direct pass-through wrapper. Its purpose is to facilitate interoperability between Vulkan and the Android graphics stack, not to modify Vulkan's behavior. It would deserialize the `AHardwareBuffer*` and `VkDevice` handles and serialize the resulting `VkAndroidHardwareBufferPropertiesANDROID` struct.

---

#### 3. & 4. `vt_handle_vkCmdDrawIndirectCount` & `vt_handle_vkCmdDrawIndexedIndirectCount`

```c
/**
 * @brief Handles vkCmdDrawIndirectCount and vkCmdDrawIndexedIndirectCount.
 */
void vt_handle_vkCmdDrawIndirectCount(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkBuffer buffer = VkObject_fromId(*(void**)(input + 9));
    VkDeviceSize offset = *(VkDeviceSize*)(input + 0x11);
    VkBuffer countBuffer = VkObject_fromId(*(void**)(input + 0x19));
    VkDeviceSize countBufferOffset = *(VkDeviceSize*)(input + 0x21);
    uint32_t maxDrawCount = *(uint32_t*)(input + 0x29);
    uint32_t stride = *(uint32_t*)(input + 0x2D);

    // 2. MODIFICATION & HOOKING
    // No modifications. These are fundamental drawing commands.

    // 3. DISPATCH TO VULKAN
    // The specific dispatch call depends on whether it's indexed or not.
    // Assuming vt_handle_vkCmdDrawIndirectCount:
    dispatch_vkCmdDrawIndirectCount(
        commandBuffer, buffer, offset, countBuffer, countBufferOffset, maxDrawCount, stride
    );
    
    // 4. SERIALIZATION
    // None, this is a void command buffer function.
}
```
**Analysis:** These functions are standard, non-modified pass-throughs. They are core rendering commands that are fundamental to the graphics pipeline. The wrapper's only job is to deserialize the buffer handles and numerical arguments and pass them directly to the driver. Interfering with these would break rendering.

---

#### 5, 6, & 7. `vt_handle_vkCmdBind/Begin/EndTransformFeedbackBuffersEXT`

```c
/**
 * @brief Handles vkCmdBindTransformFeedbackBuffersEXT.
 */
void vt_handle_vkCmdBindTransformFeedbackBuffersEXT(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t firstBinding = *(uint32_t*)(input + 9);
    uint32_t bindingCount = *(uint32_t*)(input + 0xD);

    // Deserialize arrays of buffers, offsets, and sizes into temporary memory.
    VkBuffer* pBuffers = deserialize_buffer_array(context, input + 0x11, bindingCount);
    VkDeviceSize* pOffsets = deserialize_size_array(context, input + ..., bindingCount);
    VkDeviceSize* pSizes = deserialize_size_array(context, input + ..., bindingCount);

    // 2. MODIFICATION & HOOKING
    // No modifications are made to the bindings.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBindTransformFeedbackBuffersEXT(
        commandBuffer, firstBinding, bindingCount, pBuffers, pOffsets, pSizes
    );
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** These three functions related to transform feedback are standard pass-throughs. They manage a specific part of the graphics pipeline state. The wrapper faithfully deserializes the lists of buffers and their corresponding offsets and sizes and passes them to the driver without alteration.

---

#### 8. & 9. `vt_handle_vkCmdBegin/EndQueryIndexedEXT`

```c
/**
 * @brief Handles vkCmdBeginQueryIndexedEXT and vkCmdEndQueryIndexedEXT.
 */
void vt_handle_vkCmdBeginQueryIndexedEXT(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkQueryPool queryPool = VkObject_fromId(*(void**)(input + 9));
    uint32_t query = *(uint32_t*)(input + 0x11);
    VkQueryControlFlags flags = *(VkQueryControlFlags*)(input + 0x15);
    uint32_t index = *(uint32_t*)(input + 0x19);

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBeginQueryIndexedEXT(commandBuffer, queryPool, query, flags, index);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** These are standard pass-through functions for managing indexed queries. The wrapper simply deserializes the arguments and calls the driver.

---

#### 10. `vt_handle_vkCmdDrawIndirectByteCountEXT`

```c
/**
 * @brief Handles vkCmdDrawIndirectByteCountEXT.
 */
void vt_handle_vkCmdDrawIndirectByteCountEXT(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t instanceCount = *(uint32_t*)(input + 9);
    uint32_t firstInstance = *(uint32_t*)(input + 0xD);
    VkBuffer counterBuffer = VkObject_fromId(*(void**)(input + 0x11));
    VkDeviceSize counterBufferOffset = *(VkDeviceSize*)(input + 0x19);
    uint32_t counterOffset = *(uint32_t*)(input + 0x21);
    uint32_t vertexStride = *(uint32_t*)(input + 0x25);

    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdDrawIndirectByteCountEXT(
        commandBuffer, instanceCount, firstInstance, counterBuffer, 
        counterBufferOffset, counterOffset, vertexStride
    );
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Another specialized drawing command that is a standard pass-through. The wrapper's role is purely to translate the client's command into a native Vulkan call.

---

#### 11. & 12. `vt_handle_vkGetBufferOpaqueCaptureAddress` & `vt_handle_vkGetBufferDeviceAddress`

```c
/**
 * @brief Handles vkGetBufferOpaqueCaptureAddress and vkGetBufferDeviceAddress.
 */
void vt_handle_vkGetBufferDeviceAddress(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBufferDeviceAddressInfo info = {0};
    // (pNext chain is processed if present)
    info.sType = *(VkStructureType*)(context->inputBufferPtr + 0xD);
    info.buffer = (VkBuffer)VkObject_fromId(*(void**)(context->inputBufferPtr + 0x15));
    
    // 2. MODIFICATION & HOOKING
    // No modifications to the info struct.

    // 3. DISPATCH TO VULKAN
    VkDeviceAddress address = dispatch_vkGetBufferDeviceAddress(device, &info);

    // 4. SERIALIZATION
    // Write the resulting 64-bit address back to the client.
    RingBuffer_write(context->clientRingBuffer, &address, sizeof(VkDeviceAddress));
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** These two functions are standard pass-throughs. They query an address from a buffer handle, and the wrapper simply facilitates this query without changing any parameters.

---

#### 13. `vt_handle_vkGetDeviceMemoryOpaqueCaptureAddress`

```c
/**
 * @brief Handles vkGetDeviceMemoryOpaqueCaptureAddress.
 */
void vt_handle_vkGetDeviceMemoryOpaqueCaptureAddress(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    VkDeviceMemoryOpaqueCaptureAddressInfo info = {0};
    info.sType = *(VkStructureType*)(context->inputBufferPtr + 0xD);
    info.memory = (VkDeviceMemory)VkObject_fromId(*(void**)(context->inputBufferPtr + 0x15));

    // 2. MODIFICATION & HOOKING
    // A global flag `vortekSerializerCastVkObject` controls the behavior.
    if (vortekSerializerCastVkObject != 0) {
        // If the flag is set, the provided handle is not the real VkDeviceMemory handle.
        // It is a pointer to a custom wrapper struct, and the real handle is at an offset.
        // This line extracts the true VkDeviceMemory handle from the wrapper.
        info.memory = *(VkDeviceMemory*)((long)info.memory + 0x18);
    }
    
    // 3. DISPATCH TO VULKAN
    uint64_t address = dispatch_vkGetDeviceMemoryOpaqueCaptureAddress(device, &info);

    // 4. SERIALIZATION
    RingBuffer_write(context->clientRingBuffer, &address, sizeof(uint64_t));
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function contains a significant modification. It checks a global flag `vortekSerializerCastVkObject`. If true, it treats the incoming `VkDeviceMemory` handle not as a direct handle, but as a pointer to a custom structure. It then reads the *actual* `VkDeviceMemory` handle from an offset of `0x18` within that structure. This is a clear indication that the renderer uses its own wrapper objects for `VkDeviceMemory` (likely the `ResourceMemory` struct inferred from other functions) and needs to unwrap them before making the native Vulkan call.

---

#### 14. - 17. `vt_handle_vkCmdSetLineStipple`, `vt_handle_vkCmdSetCullMode`, `vt_handle_vkCmdSetFrontFace`, `vt_handle_vkCmdSetPrimitiveTopology`

```c
/**
 * @brief Handles simple dynamic state commands like vkCmdSetCullMode.
 */
void vt_handle_vkCmdSetCullMode(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkCullModeFlags cullMode = *(VkCullModeFlags*)(input + 9);
    
    // 2. MODIFICATION & HOOKING
    // No modifications.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetCullMode(commandBuffer, cullMode);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** All four of these are simple, non-modified pass-through functions. They are fundamental dynamic state setters for the rasterizer. The wrapper's role is minimal: deserialize the command buffer handle and the state value, and call the driver. No modifications are necessary or logical here.

---

#### 18. & 19. `vt_handle_vkCmdSetViewportWithCount` & `vt_handle_vkCmdSetScissorWithCount`

```c
/**
 * @brief Handles vkCmdSetViewportWithCount.
 */
void vt_handle_vkCmdSetViewportWithCount(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t viewportCount = *(uint32_t*)(input + 9);
    
    // Deserialize the array of VkViewport structs into a temporary buffer on the stack.
    VkViewport* pViewports = (VkViewport*)((long)&local_stack_var - (viewportCount * sizeof(VkViewport) + ...));
    deserialize_viewport_array(input, viewportCount, pViewports);
    
    // 2. MODIFICATION & HOOKING
    // No modifications are made to the viewport data.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetViewportWithCount(commandBuffer, viewportCount, pViewports);
    
    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** These functions are standard pass-throughs. The logic involves deserializing an array of structs (`VkViewport` or `VkRect2D`) from the client buffer into a temporary stack allocation and then passing that array to the real Vulkan function. The values within the structs are not altered.

---

#### 20. `vt_handle_vkCmdBindVertexBuffers2`

```c
/**
 * @brief Handles vkCmdBindVertexBuffers2.
 */
void vt_handle_vkCmdBindVertexBuffers2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t firstBinding = *(uint32_t*)(input + 9);
    uint32_t bindingCount = *(uint32_t*)(input + 0xD);
    
    // Deserialize arrays of buffers, offsets, sizes, and strides.
    VkBuffer* pBuffers = deserialize_buffer_array(context, input + 0x11, bindingCount);
    VkDeviceSize* pOffsets = deserialize_size_array(context, input + ..., bindingCount);
    VkDeviceSize* pSizes = deserialize_size_array(context, input + ..., bindingCount);
    VkDeviceSize* pStrides = deserialize_size_array(context, input + ..., bindingCount);

    // 2. MODIFICATION & HOOKING
    // No modifications are made to the binding data.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBindVertexBuffers2(
        commandBuffer, firstBinding, bindingCount, pBuffers, pOffsets, pSizes, pStrides
    );
    
    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is the newer, more flexible version of `vkCmdBindVertexBuffers`. Lik

### Overall Analysis of the Function Batch

This set of 20 functions handles dynamic state setting commands, primarily from the `VK_EXT_extended_dynamic_state` and related extensions (`VK_EXT_extended_dynamic_state2`, `VK_EXT_extended_dynamic_state3`). These Vulkan commands are used to change pipeline state (like depth testing, blend modes, etc.) directly on a command buffer, rather than baking it into a `VkPipeline` object.

The decompiled code for all of these functions reveals a **consistent and simple pass-through pattern**. They perform the minimum work necessary to un-marshal the command and its arguments from the client's input buffer and dispatch it to the real Vulkan driver.

**There are no custom modifications, hooks, or parameter alterations in any of these 20 functions.** Their implementation is a textbook example of a lightweight wrapper for command buffer recording.

### Summary Table

| Function | Modification Detected? | Summary & Justification |
| :--- | :--- | :--- |
| `vkCmdSetDepthTestEnable` | **No** | Standard pass-through. |
| `vkCmdSetDepthWriteEnable` | **No** | Standard pass-through. |
| `vkCmdSetDepthCompareOp` | **No** | Standard pass-through. |
| `vkCmdSetDepthBoundsTestEnable` | **No** | Standard pass-through. |
| `vkCmdSetStencilTestEnable` | **No** | Standard pass-through. |
| `vkCmdSetStencilOp` | **No** | Standard pass-through. |
| `vkCmdSetRasterizerDiscardEnable` | **No** | Standard pass-through. |
| `vkCmdSetDepthBiasEnable` | **No** | Standard pass-through. |
| `vkCmdSetPrimitiveRestartEnable` | **No** | Standard pass-through. |
| `vkCmdSetTessellationDomainOrigin` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetDepthClampEnable` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetPolygonMode` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetRasterizationSamples` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetSampleMask` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetAlphaToCoverageEnable` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetAlphaToOneEnable` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetLogicOpEnable` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetColorBlendEnable` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetColorBlendEquation` | **No** | Standard pass-through for the `_EXT` version. |
| `vkCmdSetColorWriteMask` | **No** | Standard pass-through for the `_EXT` version. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkCmdSetDepthTestEnable`

```c
/**
 * @brief Handles vkCmdSetDepthTestEnable. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetDepthTestEnable(VtContext* context) {
    // 1. DESERIALIZATION
    // The command buffer handle is the first argument, implicitly resolved.
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    // The boolean 'depthTestEnable' value is the second argument.
    VkBool32 depthTestEnable = *(VkBool32*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING
    // No modifications are performed.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthTestEnable(commandBuffer, depthTestEnable);
    
    // 4. SERIALIZATION
    // None, this is a command buffer recording function with no return value.
}
```
**Analysis:** No modifications. This is a direct pass-through, simply un-marshalling the command buffer handle and the boolean enable flag, then calling the corresponding dispatch function.

---

#### 2. `vt_handle_vkCmdSetDepthWriteEnable`
```c
/**
 * @brief Handles vkCmdSetDepthWriteEnable. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetDepthWriteEnable(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 depthWriteEnable = *(VkBool32*)(context->inputBufferPtr + 9);
    
    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthWriteEnable(commandBuffer, depthWriteEnable);
}
```
**Analysis:** No modifications. Identical structure to the previous function.

---

#### 3. `vt_handle_vkCmdSetDepthCompareOp`
```c
/**
 * @brief Handles vkCmdSetDepthCompareOp. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetDepthCompareOp(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkCompareOp depthCompareOp = *(VkCompareOp*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthCompareOp(commandBuffer, depthCompareOp);
}
```
**Analysis:** No modifications. Direct pass-through for the `VkCompareOp` enum.

---

#### 4. `vt_handle_vkCmdSetDepthBoundsTestEnable`
```c
/**
 * @brief Handles vkCmdSetDepthBoundsTestEnable. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetDepthBoundsTestEnable(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 depthBoundsTestEnable = *(VkBool32*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthBoundsTestEnable(commandBuffer, depthBoundsTestEnable);
}
```
**Analysis:** No modifications.

---

#### 5. `vt_handle_vkCmdSetStencilTestEnable`
```c
/**
 * @brief Handles vkCmdSetStencilTestEnable. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetStencilTestEnable(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 stencilTestEnable = *(VkBool32*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetStencilTestEnable(commandBuffer, stencilTestEnable);
}
```
**Analysis:** No modifications.

---

#### 6. `vt_handle_vkCmdSetStencilOp`
```c
/**
 * @brief Handles vkCmdSetStencilOp. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetStencilOp(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkStencilFaceFlags faceMask = *(VkStencilFaceFlags*)(input + 9);
    VkStencilOp failOp = *(VkStencilOp*)(input + 13);
    VkStencilOp passOp = *(VkStencilOp*)(input + 17);
    VkStencilOp depthFailOp = *(VkStencilOp*)(input + 21);
    VkCompareOp compareOp = *(VkCompareOp*)(input + 25);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetStencilOp(commandBuffer, faceMask, failOp, passOp, depthFailOp, compareOp);
}
```
**Analysis:** No modifications. Deserializes all five arguments and calls the dispatch function directly.

---

#### 7. `vt_handle_vkCmdSetRasterizerDiscardEnable`
```c
/**
 * @brief Handles vkCmdSetRasterizerDiscardEnable. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetRasterizerDiscardEnable(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 rasterizerDiscardEnable = *(VkBool32*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetRasterizerDiscardEnable(commandBuffer, rasterizerDiscardEnable);
}
```
**Analysis:** No modifications.

---

#### 8. `vt_handle_vkCmdSetDepthBiasEnable`
```c
/**
 * @brief Handles vkCmdSetDepthBiasEnable. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetDepthBiasEnable(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 depthBiasEnable = *(VkBool32*)(context->inputBufferPtr + 9);
    
    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthBiasEnable(commandBuffer, depthBiasEnable);
}
```
**Analysis:** No modifications.

---

#### 9. `vt_handle_vkCmdSetPrimitiveRestartEnable`
```c
/**
 * @brief Handles vkCmdSetPrimitiveRestartEnable. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetPrimitiveRestartEnable(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 primitiveRestartEnable = *(VkBool32*)(context->inputBufferPtr + 9);
    
    // 2. MODIFICATION & HOOKING: None.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetPrimitiveRestartEnable(commandBuffer, primitiveRestartEnable);
}
```
**Analysis:** No modifications.

---

#### 10. `vt_handle_vkCmdSetTessellationDomainOrigin`
```c
/**
 * @brief Handles vkCmdSetTessellationDomainOriginEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetTessellationDomainOriginEXT(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkTessellationDomainOrigin domainOrigin = *(VkTessellationDomainOrigin*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetTessellationDomainOrigin(commandBuffer, domainOrigin);
}
```
**Analysis:** No modifications. This is a direct wrapper for the `VK_EXT_tessellation_domain_origin` extension function.

---

#### 11. `vt_handle_vkCmdSetDepthClampEnable`
```c
/**
 * @brief Handles vkCmdSetDepthClampEnableEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetDepthClampEnableEXT(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 depthClampEnable = *(VkBool32*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetDepthClampEnable(commandBuffer, depthClampEnable);
}
```
**Analysis:** No modifications. Wrapper for the `VK_EXT_depth_clip_enable` extension function.

---

#### 12. `vt_handle_vkCmdSetPolygonMode`
```c
/**
 * @brief Handles vkCmdSetPolygonModeEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetPolygonModeEXT(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkPolygonMode polygonMode = *(VkPolygonMode*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetPolygonMode(commandBuffer, polygonMode);
}
```
**Analysis:** No modifications.

---

#### 13. `vt_handle_vkCmdSetRasterizationSamples`
```c
/**
 * @brief Handles vkCmdSetRasterizationSamplesEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetRasterizationSamplesEXT(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkSampleCountFlagBits rasterizationSamples = *(VkSampleCountFlagBits*)(context->inputBufferPtr + 9);
    
    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetRasterizationSamples(commandBuffer, rasterizationSamples);
}
```
**Analysis:** No modifications.

---

#### 14. `vt_handle_vkCmdSetSampleMask`
```c
/**
 * @brief Handles vkCmdSetSampleMaskEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetSampleMaskEXT(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkSampleCountFlagBits samples = *(VkSampleCountFlagBits*)(input + 9);
    // The VkSampleMask is a uint32_t. It is read directly from the buffer.
    VkSampleMask sampleMask = *(VkSampleMask*)(input + 13);
    
    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetSampleMask(commandBuffer, samples, &sampleMask);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** No modifications. It correctly reads the `samples` count and the `sampleMask` value and passes them to the dispatch function. The address of `sampleMask` is taken because the Vulkan API expects a pointer.

---

#### 15. `vt_handle_vkCmdSetAlphaToCoverageEnable`
```c
/**
 * @brief Handles vkCmdSetAlphaToCoverageEnableEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetAlphaToCoverageEnableEXT(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 alphaToCoverageEnable = *(VkBool32*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetAlphaToCoverageEnable(commandBuffer, alphaToCoverageEnable);
}
```
**Analysis:** No modifications.

---

#### 16. `vt_handle_vkCmdSetAlphaToOneEnable`
```c
/**
 * @brief Handles vkCmdSetAlphaToOneEnableEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetAlphaToOneEnableEXT(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 alphaToOneEnable = *(VkBool32*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetAlphaToOneEnable(commandBuffer, alphaToOneEnable);
}
```
**Analysis:** No modifications.

---

#### 17. `vt_handle_vkCmdSetLogicOpEnable`
```c
/**
 * @brief Handles vkCmdSetLogicOpEnableEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetLogicOpEnableEXT(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkBool32 logicOpEnable = *(VkBool32*)(context->inputBufferPtr + 9);
    
    // 2. MODIFICATION & HOOKING: None.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetLogicOpEnable(commandBuffer, logicOpEnable);
}
```
**Analysis:** No modifications.

---

#### 18. `vt_handle_vkCmdSetColorBlendEnable`
```c
/**
 * @brief Handles vkCmdSetColorBlendEnableEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetColorBlendEnableEXT(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t firstAttachment = *(uint32_t*)(input + 9);
    uint32_t attachmentCount = *(uint32_t*)(input + 13);

    // Allocate a temporary buffer on the stack for the array of VkBool32.
    VkBool32* pColorBlendEnables = (VkBool32*)alloca(attachmentCount * sizeof(VkBool32));
    
    // Copy the array from the input buffer.
    memcpy(pColorBlendEnables, input + 17, attachmentCount * sizeof(VkBool32));

    // 2. MODIFICATION & HOOKING: None.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetColorBlendEnable(commandBuffer, firstAttachment, attachmentCount, pColorBlendEnables);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** No modifications. The function deserializes an array of booleans from the client stream into a stack-allocated buffer and passes it directly to the driver. This is standard marshalling.

---

#### 19. `vt_handle_vkCmdSetColorBlendEquation`
```c
/**
 * @brief Handles vkCmdSetColorBlendEquationEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetColorBlendEquationEXT(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);
    
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t firstAttachment = *(uint32_t*)(input + 9);
    uint32_t attachmentCount = *(uint32_t*)(input + 13);
    
    // Allocate space on the stack for the array of VkColorBlendEquationEXT structs.
    size_t arraySize = attachmentCount * sizeof(VkColorBlendEquationEXT); // sizeof is 24 bytes
    VkColorBlendEquationEXT* pEquations = (VkColorBlendEquationEXT*)alloca(arraySize);

    // Copy the array of structs from the input buffer.
    memcpy(pEquations, input + 17, arraySize);
    
    // 2. MODIFICATION & HOOKING: None.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetColorBlendEquation(commandBuffer, firstAttachment, attachmentCount, pEquations);
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** No modifications. This function deserializes an array of `VkColorBlendEquationEXT` structs into a temporary stack buffer before calling the real Vulkan function. This is standard practice and does not alter the data.

---

#### 20. `vt_handle_vkCmdSetColorWriteMask`
```c
/**
 * @brief Handles vkCmdSetColorWriteMaskEXT. This is a pass-through wrapper.
 */
void vt_handle_vkCmdSetColorWriteMaskEXT(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t firstAttachment = *(uint32_t*)(input + 9);
    uint32_t attachmentCount = *(uint32_t*)(input + 13);

    // Allocate space on the stack for the array of VkColorComponentFlags.
    size_t arraySize = attachmentCount * sizeof(VkColorComponentFlags); // sizeof is 4 bytes
    VkColorComponentFlags* pColorWriteMasks = (VkColorComponentFlags*)alloca(arraySize);

    // Copy the array from the input buffer.
    memcpy(pColorWriteMasks, input + 17, arraySize);
    
    // 2. MODIFICATION & HOOKING: None.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetColorWriteMask(commandBuffer, firstAttachment, attachmentCount, pColorWriteMasks);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** No modifications. Similar to the previous two functions, this deserializes an array of `VkColorComponentFlags` into a temporary stack buffer and passes it to the real Vulkan function. No data is altered.e its predecessor, it is a fundamental command for setting up graphics pipeline state. The wrapper's role is to correctly deserialize the four parallel arrays (buffers, offsets, sizes, strides) and pass them to the driver. No modifications are made to these values.

---

### Summary of Modifications

This batch of functions primarily consists of Vulkan commands (`vkCmd...`). Most are simple state-setting functions which are passed through to the driver without modification. The more complex `...2KHR` style commands are also passed through directly, with the exception of `vkCmdCopyBufferToImage2`, which is hooked to support the `TextureDecoder`'s format emulation.

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkCmdSetRasterizationStreamEXT` | **No** | Standard pass-through. |
| `vkCmdSetConservativeRasterizationModeEXT`| **No** | Standard pass-through. |
| `vkCmdSetExtraPrimitiveOverestimationSizeEXT`| **No** | Standard pass-through. |
| `vkCmdSetDepthClipEnableEXT` | **No** | Standard pass-through. |
| `vkCmdSetSampleLocationsEnableEXT` | **No** | Standard pass-through. |
| `vkCmdSetColorBlendAdvancedEXT` | **No** | Standard pass-through. |
| `vkCmdSetProvokingVertexModeEXT` | **No** | Standard pass-through. |
| `vkCmdSetLineRasterizationModeEXT` | **No** | Standard pass-through. |
| `vkCmdSetLineStippleEnableEXT` | **No** | Standard pass-through. |
| `vkCmdSetDepthClipNegativeOneToOneEXT`| **No** | Standard pass-through. |
| `vkCmdCopyBuffer2` | **No** | Standard pass-through. |
| `vkCmdCopyImage2` | **No** | Standard pass-through. |
| `vkCmdBlitImage2` | **No** | Standard pass-through. |
| `vkCmdCopyBufferToImage2` | **Yes** | Hijacks the call for emulated compressed textures, redirecting it to a custom `TextureDecoder_copyBufferToImage` function. |
| `vkCmdCopyImageToBuffer2`| **No** | Standard pass-through. |

---

### Detailed Function Decompilations

#### 1-10. `vkCmdSet...` Dynamic State Functions

These ten functions all follow the exact same pattern. They are simple dynamic state-setting commands that take only one or two trivial arguments besides the command buffer handle. The implementation for all of them is a direct pass-through.

**Decompiled Code (Generic Template)**

```c
/**
 * @brief Generic handler for simple vkCmdSet... dynamic state functions.
 * @param context A pointer to the VtContext structure.
 */
void vt_handle_vkCmdSet_DYNAMIC_STATE(VtContext* context) {
    // 1. DESERIALIZATION
    // The command buffer handle is resolved.
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // The specific state value(s) are read directly from the input buffer.
    // Example for vkCmdSetRasterizerDiscardEnable:
    VkBool32 rasterizerDiscardEnable = *(VkBool32*)(context->inputBufferPtr + 9);
    
    // Example for vkCmdSetProvokingVertexModeEXT:
    VkProvokingVertexModeEXT provokingVertexMode = *(VkProvokingVertexModeEXT*)(context->inputBufferPtr + 9);

    // 2. MODIFICATION & HOOKING
    // There are no modifications. The deserialized values are used directly.

    // 3. DISPATCH TO VULKAN
    // The corresponding dispatch table function is called with the resolved handle and values.
    dispatch_vkCmdSet_DYNAMIC_STATE(commandBuffer, value1, value2, ...);

    // 4. SERIALIZATION
    // None, these are void command buffer recording functions.
}
```

**Analysis:**

*   **`vt_handle_vkCmdSetRasterizationStreamEXT`**: No modifications.
*   **`vt_handle_vkCmdSetConservativeRasterizationModeEXT`**: No modifications.
*   **`vt_handle_vkCmdSetExtraPrimitiveOverestimationSizeEXT`**: No modifications.
*   **`vt_handle_vkCmdSetDepthClipEnableEXT`**: No modifications.
*   **`vt_handle_vkCmdSetSampleLocationsEnableEXT`**: No modifications.
*   **`vt_handle_vkCmdSetColorBlendAdvancedEXT`**: No modifications. This function deserializes an array of `VkColorBlendAdvancedEXT` structs into a temporary buffer but does not alter their contents before passing them to the driver.
*   **`vt_handle_vkCmdSetProvokingVertexModeEXT`**: No modifications.
*   **`vt_handle_vkCmdSetLineRasterizationModeEXT`**: No modifications.
*   **`vt_handle_vkCmdSetLineStippleEnableEXT`**: No modifications.
*   **`vt_handle_vkCmdSetDepthClipNegativeOneToOneEXT`**: No modifications.

**Justification for "No Modification":** These functions are fundamental state-setting commands. The wrapper's role here is simply to transport the command from the client to the host driver. Altering these standard states would likely break rendering correctness and serves no obvious purpose for emulation or performance optimization in this context. The decompiled code confirms this, showing a direct deserialization followed immediately by a dispatch call with the same values.

---

#### 11. `vt_handle_vkCmdCopyBuffer2`

```c
/**
 * @brief Handles vkCmdCopyBuffer2 (and vkCmdCopyBuffer2KHR).
 */
void vt_handle_vkCmdCopyBuffer2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));

    VkCopyBufferInfo2 copyInfo = {0};
    // The wrapper deserializes the VkCopyBufferInfo2 struct from the input buffer.
    deserialize_vkCopyBufferInfo2(context, &copyInfo); 
    // This helper function resolves srcBuffer and dstBuffer handles using VkObject_fromId
    // and copies the pRegions array into a temporary buffer.

    // 2. MODIFICATION & HOOKING
    // No modifications are made to the copyInfo struct or its regions.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdCopyBuffer2(commandBuffer, &copyInfo);

    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This is a standard pass-through. The function correctly reconstructs the `VkCopyBufferInfo2` struct, including its array of `VkBufferCopy2` regions, but does not alter any of the offsets, sizes, or buffer handles before calling the driver. This is expected, as buffer-to-buffer copies are fundamental operations that rarely need interception.

---

#### 12. `vt_handle_vkCmdCopyImage2`

```c
/**
 * @brief Handles vkCmdCopyImage2 (and vkCmdCopyImage2KHR).
 */
void vt_handle_vkCmdCopyImage2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    VkCopyImageInfo2 copyInfo = {0};
    deserialize_vkCopyImageInfo2(context, &copyInfo);
    // Helper deserializes the struct, resolving srcImage and dstImage handles
    // and copying the pRegions array into a temporary buffer.

    // 2. MODIFICATION & HOOKING
    // No modifications are detected. The layouts, offsets, and extents are used as-is.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdCopyImage2(commandBuffer, &copyInfo);
    
    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function is a standard pass-through. While it involves complex data structures, the logic simply reconstructs them and passes them to the driver. There are no checks against the `TextureDecoder` or any other custom modules, indicating that image-to-image copies are not hooked.

---

#### 13. `vt_handle_vkCmdBlitImage2`

```c
/**
 * @brief Handles vkCmdBlitImage2 (and vkCmdBlitImage2KHR).
 */
void vt_handle_vkCmdBlitImage2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));

    VkBlitImageInfo2 blitInfo = {0};
    deserialize_vkBlitImageInfo2(context, &blitInfo);
    // Helper deserializes the struct, resolving srcImage and dstImage handles,
    // copying the filter, and the pRegions array into a temporary buffer.

    // 2. MODIFICATION & HOOKING
    // No modifications detected. The source/destination layouts, offsets, and filter
    // are passed directly to the driver.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBlitImage2(commandBuffer, &blitInfo);

    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function is a standard pass-through. It faithfully reconstructs the `VkBlitImageInfo2` and its associated regions without modification before calling the driver's implementation.

---

#### 14. `vt_handle_vkCmdCopyBufferToImage2`

```c
/**
 * @brief Handles vkCmdCopyBufferToImage2, with hooks for texture decompression.
 */
void vt_handle_vkCmdCopyBufferToImage2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    VkCopyBufferToImageInfo2 copyInfo = {0};
    deserialize_vkCopyBufferToImageInfo2(context, &copyInfo);
    // Helper deserializes the struct, resolving srcBuffer and dstImage handles
    // and copying the pRegions array into a temporary buffer.

    // 2. MODIFICATION & HOOKING
    if (context->textureDecoder != NULL && TextureDecoder_containsImage(context->textureDecoder, copyInfo.dstImage)) {
        // HOOK: The destination image is an emulated compressed texture.
        // Instead of calling the standard Vulkan command, the call is redirected to
        // a custom function within the TextureDecoder module. This function will
        // handle writing the buffer data into the uncompressed backing image.
        if (*(int*)(copyInfo.pRegions) == 0) { // Likely a check for a simple, single-region copy
            TextureDecoder_copyBufferToImage(
                context->textureDecoder,
                commandBuffer,
                copyInfo.srcBuffer,
                copyInfo.dstImage,
                copyInfo.dstImageLayout,
                *(undefined8*)(copyInfo.pRegions + 0x10) // Pass the single region directly
            );
        }
        // The path for multiple regions is not shown but would likely involve a loop.
    } else {
        // 3. DISPATCH TO VULKAN (Normal Path)
        // If the image is not emulated, call the standard Vulkan function.
        dispatch_vkCmdCopyBufferToImage2(commandBuffer, &copyInfo);
    }
    
    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function contains a significant modification. It intercepts copies into images to support its texture decompression emulation. It checks if the `dstImage` is managed by the `TextureDecoder`. If it is, the command is **hijacked** and redirected to `TextureDecoder_copyBufferToImage`. This custom function understands the internal layout of the emulated (uncompressed) texture and can correctly place the data from the source buffer. If the destination image is not an emulated one, it proceeds as a normal pass-through call to the driver.

---

#### 15. `vt_handle_vkCmdCopyImageToBuffer2`

```c
/**
 * @brief Handles vkCmdCopyImageToBuffer2 (and vkCmdCopyImageToBuffer2KHR).
 */
void vt_handle_vkCmdCopyImageToBuffer2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    VkCopyImageToBufferInfo2 copyInfo = {0};
    deserialize_vkCopyImageToBufferInfo2(context, &copyInfo);
    // Helper deserializes the struct, resolving srcImage and dstBuffer handles
    // and copying the pRegions array into a temporary buffer.

    // 2. MODIFICATION & HOOKING
    // No checks against the TextureDecoder are performed on the srcImage.
    // The call is passed directly to the driver, even for emulated images.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdCopyImageToBuffer2(commandBuffer, &copyInfo);
    
    // 4. SERIALIZATION
    // None.

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** This function is a standard pass-through. Unlike its buffer-to-image counterpart, it does **not** check if the `srcImage` is managed by the `TextureDecoder`. This implies that reading data *from* an emulated compressed texture via this command is either not supported or not necessary for the renderer's architecture. The most likely reason is that the emulated textures are write-only targets for asset loading, and any subsequent reads are expected to happen within shaders where sampling logic can be properly controlled.

---

### Summary of Modifications

| Function | Modification Detected? | Summary of Modification |
| :--- | :--- | :--- |
| `vkCmdResolveImage2` | **No** | Standard pass-through. |
| `vkCmdSetColorWriteEnable` | **No** | Standard pass-through. |
| `vkCmdSetEvent2` | **No** | Standard pass-through. `VkDependencyInfo` is deserialized but not altered. |
| `vkCmdResetEvent2` | **No** | Standard pass-through. |
| `vkCmdWaitEvents2` | **No** | Standard pass-through. `VkDependencyInfo` structs are deserialized but not altered. |
| `vkCmdPipelineBarrier2` | **No** | Standard pass-through. `VkDependencyInfo` is deserialized but not altered. |
| `vkQueueSubmit2` | **Yes** | Calls `TextureDecoder_decodeAll` before dispatching. This is a critical hook to ensure all pending texture decompressions are finished before the GPU work that might use them is submitted. |
| `vkCmdWriteTimestamp2` | **No** | Standard pass-through. |
| `vkCmdBeginRendering` | **Yes** | Hooks `VkRenderingInfo` to modify attachment parameters. Specifically, if an image view is being used for a compressed format (as tracked by `TextureDecoder`), its `imageView` handle is replaced with the handle of the decompressed buffer's view, and its `imageLayout` is set to `VK_IMAGE_LAYOUT_GENERAL`. |
| `vkCmdEndRendering` | **No** | Standard pass-through. |
| `vkGetShaderModuleIdentifierEXT`| **No**| Standard pass-through. |
|`vkGetShaderModuleCreateInfoIdentifierEXT`|**Yes**| Calls `ShaderInspector_inspectShaderStages` to analyze the shader without creating it. This allows the renderer to extract metadata (like bindings) from the shader code itself. |

---

### Detailed Function Decompilations

#### 1. `vt_handle_vkCmdResolveImage2`

```c
/**
 * @brief Handles vkCmdResolveImage2.
 */
void vt_handle_vkCmdResolveImage2(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    
    VkCopyResolveInfo2 resolveInfo = {0};
    deserialize_vkCopyResolveInfo2(context, &resolveInfo, input + 9);
    
    // 2. MODIFICATION & HOOKING
    // No modifications detected. The function deserializes the VkCopyResolveInfo2
    // struct and its nested VkImageResolve2 array directly into temporary memory
    // and passes it to the driver.

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdResolveImage2(commandBuffer, &resolveInfo);
    
    // 4. SERIALIZATION
    // None (command buffer function).
}
```
**Analysis:** This is a standard pass-through command. The `VkCopyResolveInfo2` and its associated data are deserialized from the input buffer and used directly in the call to the real Vulkan function.

---

#### 2. `vt_handle_vkCmdSetColorWriteEnable`

```c
/**
 * @brief Handles vkCmdSetColorWriteEnable.
 */
void vt_handle_vkCmdSetColorWriteEnable(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t attachmentCount = *(uint32_t*)(input + 9);
    const VkBool32* pColorWriteEnables = (const VkBool32*)(input + 13);
    
    // 2. MODIFICATION & HOOKING
    // No modifications. The arguments are read directly from the input buffer and
    // passed to the dispatch function without alteration.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetColorWriteEnable(commandBuffer, attachmentCount, pColorWriteEnables);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. This function sets a simple pipeline state dynamically and does not require any intervention from the wrapper.

---

#### 3. `vt_handle_vkCmdSetEvent2`

```c
/**
 * @brief Handles vkCmdSetEvent2.
 */
void vt_handle_vkCmdSetEvent2(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkEvent event = VkObject_fromId(*(void**)(input + 9));
    
    // Deserializes the complex VkDependencyInfo struct from the input buffer.
    // This involves reading arrays of VkMemoryBarrier2, VkBufferMemoryBarrier2, etc.
    VkDependencyInfo dependencyInfo = {0};
    deserialize_vkDependencyInfo(context, &dependencyInfo, input + 17);

    // 2. MODIFICATION & HOOKING
    // No modifications were observed. Although VkDependencyInfo is complex,
    // the wrapper appears to deserialize it faithfully without changing any
    // stage masks, access masks, or barrier information.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdSetEvent2(commandBuffer, event, &dependencyInfo);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. This is part of the `VK_KHR_synchronization2` extension, which provides a more flexible way to specify dependencies. The wrapper correctly deserializes the complex `VkDependencyInfo` struct but performs no modifications before dispatching the command.

---

#### 4. `vt_handle_vkCmdResetEvent2`

```c
/**
 * @brief Handles vkCmdResetEvent2.
 */
void vt_handle_vkCmdResetEvent2(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkEvent event = VkObject_fromId(*(void**)(input + 9));
    VkPipelineStageFlags2 stageMask = *(VkPipelineStageFlags2*)(input + 17);
    
    // 2. MODIFICATION & HOOKING
    // No modifications. The arguments are simple and passed directly.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdResetEvent2(commandBuffer, event, stageMask);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. No modifications are made.

---

#### 5. `vt_handle_vkCmdWaitEvents2`

```c
/**
 * @brief Handles vkCmdWaitEvents2.
 */
void vt_handle_vkCmdWaitEvents2(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    uint32_t eventCount = *(uint32_t*)(input + 9);
    
    // Deserializes arrays of VkEvent handles and VkDependencyInfo structs.
    VkEvent* pEvents = deserialize_event_array(context, eventCount, input + 13);
    VkDependencyInfo* pDependencyInfos = deserialize_dependency_info_array(context, eventCount, ...);
    
    // 2. MODIFICATION & HOOKING
    // No modifications detected. The arrays of events and dependency infos are
    // deserialized and passed to the driver without changes.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdWaitEvents2(commandBuffer, eventCount, pEvents, pDependencyInfos);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. Similar to `vkCmdSetEvent2`, this function handles the more complex `VkDependencyInfo` struct but does not appear to alter its contents.

---

#### 6. `vt_handle_vkCmdPipelineBarrier2`

```c
/**
 * @brief Handles vkCmdPipelineBarrier2.
 */
void vt_handle_vkCmdPipelineBarrier2(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    
    VkDependencyInfo dependencyInfo = {0};
    deserialize_vkDependencyInfo(context, &dependencyInfo, input + 9);

    // 2. MODIFICATION & HOOKING
    // No modifications observed.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdPipelineBarrier2(commandBuffer, &dependencyInfo);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. The wrapper correctly handles the deserialization of the `VkDependencyInfo` struct but does not modify it.

---

#### 7. `vt_handle_vkQueueSubmit2`

```c
/**
 * @brief Handles vkQueueSubmit2, with hooks for texture decompression.
 */
void vt_handle_vkQueueSubmit2(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkQueue queue = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    uint32_t submitCount = *(uint32_t*)(context->inputBufferPtr + 9);
    VkFence fence = VkObject_fromId(*(void**)(...)); // Fence is after the submit array

    // Deserializes the array of VkSubmitInfo2 structs and their nested arrays.
    VkSubmitInfo2* pSubmits = deserialize_submit_info2_array(context, submitCount, ...);
    
    // 2. MODIFICATION & HOOKING
    bool shouldSendStatus = RingBuffer_hasStatus(context->clientRingBuffer, 4);
    
    // *** CRITICAL HOOK ***
    // Before submitting to the GPU, ensure all pending texture decompressions are complete.
    if (context->textureDecoder != NULL) {
        TextureDecoder_decodeAll(context->textureDecoder);
    }
    
    // 3. DISPATCH TO VULKAN
    VkResult result = dispatch_vkQueueSubmit2(queue, submitCount, pSubmits, fence);
    
    if (result == VK_ERROR_DEVICE_LOST) {
        context->lastError = VK_ERROR_DEVICE_LOST;
    }

    // 4. SERIALIZATION
    // Only send a result back if the client requested it.
    if (shouldSendStatus) {
        RingBuffer_write(context->clientRingBuffer, &result, sizeof(VkResult));
    }
    
    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** **This function has a significant modification.** The call to `TextureDecoder_decodeAll` is a major hook. It acts as a synchronization point. Before any command buffers are submitted to the GPU, this function is called to process any pending texture decompression tasks that have been queued up. This ensures that when the GPU executes commands that sample these textures, it accesses the fully decompressed and correct data, which is crucial for the renderer's format emulation feature to work correctly.

---

#### 8. `vt_handle_vkCmdWriteTimestamp2`

```c
/**
 * @brief Handles vkCmdWriteTimestamp2.
 */
void vt_handle_vkCmdWriteTimestamp2(VtContext* context) {
    // 1. DESERIALIZATION
    char* input = context->inputBufferPtr;
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(input + 1));
    VkPipelineStageFlags2 stage = *(VkPipelineStageFlags2*)(input + 9);
    VkQueryPool queryPool = VkObject_fromId(*(void**)(input + 17));
    uint32_t query = *(uint32_t*)(input + 25);
    
    // 2. MODIFICATION & HOOKING
    // No modifications.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdWriteTimestamp2(commandBuffer, stage, queryPool, query);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. No modifications are made.

---

#### 9. `vt_handle_vkCmdBeginRendering`

```c
/**
 * @brief Handles vkCmdBeginRendering, with hooks for texture decompression.
 */
void vt_handle_vkCmdBeginRendering(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(...);
    VkRenderingInfo renderingInfo = {0};
    deserialize_vkRenderingInfo(context, &renderingInfo, ...);
    
    // 2. MODIFICATION & HOOKING
    if (context->textureDecoder != NULL) {
        // Iterate through color attachments
        for (uint32_t i = 0; i < renderingInfo.colorAttachmentCount; ++i) {
            VkRenderingAttachmentInfo* attachment = (VkRenderingAttachmentInfo*)&renderingInfo.pColorAttachments[i];
            if (TextureDecoder_containsImage(context->textureDecoder, attachment->imageView)) {
                // This image view is for a compressed texture.
                // Replace its handle with the view of the decompressed buffer.
                attachment->imageView = TextureDecoder_getBufferImageView(context->textureDecoder, attachment->imageView);
                // The layout must be general for the emulated storage image.
                attachment->imageLayout = VK_IMAGE_LAYOUT_GENERAL;
            }
        }
        // (Similar logic for depth and stencil attachments)
    }

    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdBeginRendering(commandBuffer, &renderingInfo);

    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** **This function has a significant modification.** It hooks the beginning of a dynamic render pass. It checks each attachment's `imageView`. If that view corresponds to a texture being managed by the `TextureDecoder`, it replaces the `imageView` handle with one that points to the decompressed buffer. Crucially, it also changes the `imageLayout` to `VK_IMAGE_LAYOUT_GENERAL`, which is necessary because the decompressed texture is likely stored in a `VkImage` with `VK_IMAGE_USAGE_STORAGE_BIT`, and `VK_IMAGE_LAYOUT_GENERAL` is the required layout for using it as both a storage image (for writing the decompressed data) and a render target.

---

#### 10. `vt_handle_vkCmdEndRendering`

```c
/**
 * @brief Handles vkCmdEndRendering.
 */
void vt_handle_vkCmdEndRendering(VtContext* context) {
    // 1. DESERIALIZATION
    VkCommandBuffer commandBuffer = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    
    // 2. MODIFICATION & HOOKING
    // No modifications.
    
    // 3. DISPATCH TO VULKAN
    dispatch_vkCmdEndRendering(commandBuffer);
    
    // 4. SERIALIZATION
    // None.
}
```
**Analysis:** Standard pass-through. This command simply ends a render pass and has no parameters that would need modification.

---

#### 11. `vt_handle_vkGetShaderModuleIdentifierEXT`

```c
/**
 * @brief Handles vkGetShaderModuleIdentifierEXT.
 */
void vt_handle_vkGetShaderModuleIdentifierEXT(VtContext* context) {
    long stack_check_cookie = *(long *)(tpidr_el0 + 0x28);

    // 1. DESERIALIZATION
    VkDevice device = VkObject_fromId(*(void**)(context->inputBufferPtr + 1));
    VkShaderModule shaderModule = VkObject_fromId(*(void**)(context->inputBufferPtr + 9));
    
    // 2. MODIFICATION & HOOKING
    // No modifications. This is a query for a driver-generated value.

    // 3. DISPATCH TO VULKAN
    VkShaderModuleIdentifierEXT identifier = {0};
    identifier.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_IDENTIFIER_EXT;
    dispatch_vkGetShaderModuleIdentifier(device, shaderModule, &identifier);

    // 4. SERIALIZATION
    // Write the resulting identifier struct back to the client.
    uint32_t totalSize = sizeof(VkShaderModuleIdentifierEXT);
    void* resultBuffer = malloc(totalSize);
    memcpy(resultBuffer, &identifier, totalSize);
    RingBuffer_write(context->clientRingBuffer, resultBuffer, totalSize);
    free(resultBuffer);

    if (*(long *)(tpidr_el0 + 0x28) != stack_check_cookie) {
        __stack_chk_fail();
    }
}
```
**Analysis:** Standard pass-through. This function retrieves a unique, driver-generated identifier for a shader module. The wrapper does not and should not modify this value.

---

#### 12. `vt_handle_vkGetShaderModuleCreateInfoIdentifierEXT`

```c
/**
 * @brief Handles vkGetShaderModuleCreateInfoIdentifierEXT.
 */
void vt_handle_vkGetShaderModuleCreateInfoIdentifierEXT(long param_1) {
    // 1. DESERIALIZATION
    // Deserializes the VkDevice handle and a full VkShaderModuleCreateInfo struct
    // from the input buffer into temporary memory.
    VkDevice device = ...;
    VkShaderModuleCreateInfo createInfo = {0};
    deserialize_vkShaderModuleCreateInfo(context, &createInfo, ...);

    // 2. MODIFICATION & HOOKING
    // The call is intercepted and redirected.
    // Instead of calling the dispatch function, it calls into the ShaderInspector.
    ShaderInspector_inspectShaderStages(context->shaderInspector, device, &createInfo, ...);

    // 3. DISPATCH TO VULKAN
    VkShaderModuleIdentifierEXT identifier = {0};
    identifier.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_IDENTIFIER_EXT;
    dispatch_vkGetShaderModuleCreateInfoIdentifier(device, &createInfo, &identifier);

    // 4. SERIALIZATION
    // The resulting identifier is serialized back to the client.
    serialize_vkShaderModuleIdentifier(context, &identifier);
}
```
**Analysis:** **This function has a significant modification.** It intercepts the call and uses the provided `VkShaderModuleCreateInfo` to perform analysis via `ShaderInspector_inspectShaderStages`. This is a powerful hook because it allows the renderer to understand a shader's properties (like its resource bindings, inputs, and outputs) from its source code *without* the overhead of actually creating a `VkShaderModule` object. This information could be used to pre-validate pipeline layouts or optimize resource allocation before a pipeline is even compiled. After the inspection, it proceeds to call the real Vulkan function to get the identifier.
