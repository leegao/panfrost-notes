## **Vortek Renderer: Design for Emulated BCn Texture Support**

### 1. Overview

This document describes the architecture and workflow used by the Vortek renderer (`libvortekrenderer.so`) to provide support for block-compressed (BCn) texture formats on systems where the underlying Vulkan driver does not natively support them.

The core problem is a compatibility gap: an application wishes to use efficient, GPU-native BCn formats (like DXT1/BC1), but the host's GPU driver reports no support for creating or sampling from such images.

The Vortek renderer's solution is a **just-in-time (JIT) CPU-based decoding pipeline**. It intercepts key Vulkan API calls, emulates support for BCn formats to the application, and manages a deferred decoding process that converts the compressed data into a natively-supported uncompressed format (e.g., `VK_FORMAT_B8G8R8A8_UNORM`) right before the GPU needs it.

### 2. System Components

```ascii
+-----------------+      Vulkan API Calls      +-------------------------+      Native Calls      +---------------------+
|   Application   | <------------------------> |   Vortek Renderer       | ---------------------> |  Native Vulkan      |
| (Wants BCn)     |                            | (libvortekrenderer.so)  |                        |  Driver & GPU       |
+-----------------+                            +-------------------------+                        +---------------------+
                                               |     - vt_handle_*       |
                                               |     - TextureDecoder    |
                                               +-------------------------+
```

*   **Application**: The client program that believes it is using a Vulkan implementation that supports BCn texture formats.
*   **Vortek Renderer (`libvortekrenderer.so`)**: A shared library acting as a compatibility layer. It intercepts Vulkan calls from the application, manages resources, and communicates with the native driver.
    *   **`vt_handle_*` functions**: These are the entry points that intercept specific Vulkan API calls (e.g., `vt_handle_vkCreateImage`).
    *   **`TextureDecoder`**: A key subsystem within Vortek responsible for managing the state and logic of the decoding process.
*   **Native Vulkan Driver & GPU**: The host system's actual Vulkan implementation, which, in this scenario, lacks support for BCn formats.

### 3. Detailed Workflow

The process is a sequence of interceptions and deferred actions, triggered by standard Vulkan API calls.

#### **Step 1: Faking Format Support**

The process begins when the application queries the physical device for its capabilities. Vortek intercepts this call to lie about its BCn support.

**Trigger:** `vkGetPhysicalDeviceImageFormatProperties`

**Workflow:**
1.  The application calls `vkGetPhysicalDeviceImageFormatProperties` with a BCn format (e.g., `VK_FORMAT_BC1_RGB_UNORM_BLOCK`).
2.  `vt_handle_vkGetPhysicalDeviceImageFormatProperties` first calls the native driver's implementation.
3.  The native driver, lacking support, returns `VK_ERROR_FORMAT_NOT_SUPPORTED`.
4.  Instead of propagating the error, the Vortek handler catches it. It checks if the requested format is one of the BCn formats it knows how to decode.
5.  If it is, it calls its internal `getCompressedImageFormatProperties` function to generate a set of "dummy" but valid properties. These properties describe a very capable, uncompressed image, tricking the application into believing the format is supported.
6.  It returns `VK_SUCCESS` to the application, along with the fake properties.

**Diagram:**

```ascii
     Application                                Vortek Renderer                           Native Driver
        |                                              |                                        |
        |---- vkGet...FormatProperties(BC1) ---------->|                                        |
        |                                              |---- vkGet...FormatProperties(BC1) --->|
        |                                              |                                        |-----> VK_ERROR_FORMAT_NOT_SUPPORTED
        |                                              |<---------------------------------------|
        |                                              |                                        |
        |                                              | if (isCompressedFormat) {              |
        |                                              |   getCompressedImageFormatProperties() |
        |                                              | }                                      |
        |<--- VK_SUCCESS, (Fake Properties) -----------|                                        |
        |                                              |                                        |
```

**Code Snippet (`getCompressedImageFormatProperties`):**

```c
/**
 * @brief Provides fallback properties for BCn formats from BC1 to BC5.
 */
VkResult getCompressedImageFormatProperties(VkFormat format, VkImageFormatProperties* pImageFormatProperties) {
    
    // Original check: `if (param_1 - 0x83U < 0xc)`
    if (format >= VK_FORMAT_BC1_RGB_UNORM_BLOCK && format <= VK_FORMAT_BC5_SNORM_BLOCK) {
        
        // Populate with generous, hardcoded limits.
        pImageFormatProperties->maxExtent = (VkExtent3D){16384, 16384, 1};  // offset: +0x00
        pImageFormatProperties->maxMipLevels = 15;                          // offset: +0x0C
        pImageFormatProperties->maxArrayLayers = 2048;                      // offset: +0x10
        pImageFormatProperties->sampleCounts = VK_SAMPLE_COUNT_1_BIT;       // offset: +0x14
        pImageFormatProperties->maxResourceSize = 2147483648;              // offset: +0x18 (2GB)

        return VK_SUCCESS; // 0
    }
    return VK_ERROR_FORMAT_NOT_SUPPORTED; // -11 (0xfffffff5)
}
```

#### **Step 2: Intercepting Image Creation**

Confident that the format is supported, the application proceeds to create the image.

**Trigger:** `vkCreateImage`

**Workflow:**
1.  The application calls `vkCreateImage` with a `VkImageCreateInfo` struct specifying a BCn format.
2.  `vt_handle_vkCreateImage` calls `TextureDecoder_createImage`.
3.  Instead of creating a BCn image, `TextureDecoder_createImage` **modifies the `VkImageCreateInfo` struct in-place**:
    *   It changes `format` to `VK_FORMAT_B8G8R8A8_UNORM`.
    *   It sets `usage` flags to include `VK_IMAGE_USAGE_TRANSFER_DST_BIT` so it can be a copy destination.
4.  It calls the native `vkCreateImage` with the modified info to create a standard, uncompressed RGBA image. This will be the final destination for the decoded data.
5.  It then creates a `VkBuffer` (the "staging buffer") and allocates `HOST_VISIBLE` `VkDeviceMemory` for it. This buffer is sized to hold the *uncompressed* image data.
6.  It stores the handles to the final RGBA image, the staging buffer, and the staging memory in an internal `DecodedImageResource` struct and returns the handle of the final RGBA image to the application.

**Diagram:**

```ascii
   Application                              Vortek Renderer                                Native Driver
        |                                            |                                             |
        |---- vkCreateImage(BC1) ------------------->|                                             |
        |                                            | 1. Intercept & Modify CreateInfo            |
        |                                            |    (format -> RGBA8, usage |= TRANSFER_DST) |
        |                                            |                                             |
        |                                            | 2. vkCreateImage(RGBA8) ------------------->|
        |                                            |<--------------------- (hImage) --------------|
        |                                            |                                             |
        |                                            | 3. vkCreateBuffer(staging) ---------------->|
        |                                            |<--------------------- (hStagingBuffer) ------|
        |                                            |                                             |
        |                                            | 4. vkAllocateMemory/vkBindBufferMemory      |
        |                                            |    for staging buffer.                      |
        |                                            |                                             |
        |                                            | 5. Store {hImage, hStagingBuffer, ...}      |
        |<--- VK_SUCCESS, (hImage) -------------------|                                             |
        |                                            |                                             |
```

#### **Step 3: Intercepting Data Upload**

The application, holding a `VkImage` handle, now tries to copy its compressed data into it.

**Trigger:** `vkCmdCopyBufferToImage`

**Workflow:**
1.  The application records a `vkCmdCopyBufferToImage` command, intending to copy its compressed data from a source buffer into the image handle it received in Step 2.
2.  `vt_handle_vkCmdCopyBufferToImage` intercepts this command.
3.  It calls `TextureDecoder_copyBufferToImage`, which checks if the destination image is one of the "faked" images it is tracking.
4.  Since it is, the function **does not** record a copy command to the final RGBA image. Instead, it does two things:
    a.  It records a `vkCmdCopyBufferToImage` command that copies the application's compressed data into the decoder's **internal staging buffer** created in Step 2.
    b.  It creates a `DecodeOperation` struct containing pointers to the source buffer info and the destination image info and enqueues it in the `TextureDecoder`'s `pendingDecodes` queue.
5.  The command buffer now contains a command to fill the staging buffer, but the actual decoding has not yet occurred.

#### **Step 4: The Decode Trigger**

The decoding is deferred until the last possible moment: when the application submits work to a queue that might use the texture.

**Trigger:** `vkQueueSubmit`

**Workflow:**
1.  The application calls `vkQueueSubmit` with one or more command buffers. These command buffers may contain the copy to the staging buffer (from Step 3) and subsequent draw calls that sample from the (as-yet empty) final RGBA image.
2.  `vt_handle_vkQueueSubmit` intercepts this call.
3.  **Before** calling the native `vkQueueSubmit`, it calls `TextureDecoder_decodeAll()`.
4.  This triggers the synchronous, CPU-based decoding process.
5.  After `TextureDecoder_decodeAll` completes, the handler calls the native `vkQueueSubmit`. At this point, the GPU receives a command buffer that first copies uncompressed data from the staging buffer to the final image, and then correctly samples from it.

**Diagram:**

```ascii
     Application             Vortek Renderer
        |                          |
        |---- vkQueueSubmit ------>|
        |                          |
        |                          | 1. Intercept call
        |                          |
        |                          | 2. TextureDecoder_decodeAll() // CPU work happens here
        |                          |    - Dequeue pending operations
        |                          |    - Map memories
        |                          |    - Decompress BCn -> RGBA
        |                          |    - Unmap memories
        |                          |
        |                          | 3. vkQueueSubmit() -> to Native Driver
        |                          |
        |<---- (Result) -----------|
```

#### **Step 5: The Decoding Process**

This is the implementation of `TextureDecoder_decodeAll`.

**Workflow:**
1.  It loops through every `DecodeOperation` in the `pendingDecodes` queue.
2.  For each operation, it maps the source staging buffer (now full of compressed data) and the memory of the final RGBA image to CPU pointers.
3.  It iterates over the image in 4x4 blocks.
4.  In each iteration, it reads a compressed block (8 bytes for BC1/BC4, 16 bytes for BC2/BC3/BC5) from the source pointer.
5.  It calls a format-specific internal function (e.g., `decode_bc1_block`, `decode_bc4_bc5_channel_block`) to decompress the block.
6.  The decoding function writes the resulting 16 RGBA pixels (64 bytes) to the correct location in the destination memory.
7.  After all blocks are processed, it unmaps both memory regions.

**Code Snippet (`TextureDecoder_decodeAll` Core Logic):**

```c
void TextureDecoder_decodeAll(TextureDecoder* decoder) {
    // While the queue of pending decodes is not empty...
    while (!ArrayDeque_isEmpty(&decoder->pendingDecodes)) {
        
        // Get the next operation
        DecodeOperation* op = (DecodeOperation*)ArrayDeque_removeFirst(&decoder->pendingDecodes);
        if (!op) continue;

        BoundBufferInfo* bufferInfo = op->bufferInfo;
        DecodedImageResource* imageInfo = op->imageInfo;
        free(op);

        // Map source (staging buffer) and destination (final image) memory
        void* compressedDataSrc = mmap(..., bufferInfo->memory->fd, ...);
        void* uncompressedDataDst;
        vkMapMemory(decoder->device, imageInfo->stagingMemory, ..., &uncompressedDataDst);

        // ... (Error handling omitted) ...

        // Loop over the image in 4x4 blocks
        for (uint32_t y = 0; y < imageInfo->extent.height; y += 4) {
            for (uint32_t x = 0; x < imageInfo->extent.width; x += 4) {
                
                uint32_t* outputPixelBlock = (uint32_t*)((char*)uncompressedDataDst + ...);

                // Select the correct decoder based on the original format
                switch (imageInfo->originalFormat) {
                    case VK_FORMAT_BC1_RGB_UNORM_BLOCK:
                        decode_bc1_block(compressedDataSrc, outputPixelBlock, false);
                        compressedDataSrc += 8;
                        break;
                    case VK_FORMAT_BC3_UNORM_BLOCK:
                        // ...
                        break;
                    // ... other cases
                }
            }
        }
        
        // Unmap memory when done
        vkUnmapMemory(decoder->device, imageInfo->stagingMemory);
        munmap(compressedDataSrc, bufferInfo->memory->size);
    }
}
```

---
## **Implementation Details**

### 1. Common Data Structures & Types

The following C structures are C-style representations of the objects and contexts inferred from the decompiled code's memory access patterns.

```c
#include <stdlib.h>
#include <stdbool.h>
#include <sys/mman.h> // For mmap
#include <string.h>   // For memset
#include "vulkan_core.h"

// A mock for a dynamic array or list implementation.
typedef struct ArrayList {
    uint32_t count;
    void** list;
} ArrayList;

// A mock for a deque (double-ended queue) implementation.
typedef struct ArrayDeque {
    // ... members
} ArrayDeque;

// Represents the overall context for the GPU process wrapper.
typedef struct VulkanGpuProcess {
    VkDevice device;                          // offset: +0x10
    char* pArgBuffer;                         // offset: +0x50
    RingBuffer* pResponseBuffer;              // offset: +0x68
    VkResult lastError;                       // offset: +0x78
    TextureDecoder* pTextureDecoder;          // offset: +0x90
} VulkanGpuProcess;

// Represents a tracked Vulkan memory allocation.
typedef struct ResourceMemory {
    int fd;
    size_t size;
    VkDeviceMemory memoryHandle; // offset: +0x20
} ResourceMemory;

// Represents a tracked buffer-to-memory binding.
typedef struct BoundBufferInfo {
    VkBuffer buffer;          // offset: +0x0
    VkDeviceSize offset;      // offset: +0x8
    ResourceMemory* memory;   // offset: +0x10
} BoundBufferInfo;

// Represents the resources for a compressed image that needs decoding.
typedef struct DecodedImageResource {
    VkImage decodedImage;
    VkFormat originalFormat;     // offset: +0x8
    int field_C;                 // Unknown field
    VkExtent3D extent;           // width at +0xC, height at +0x10
    uint32_t mipLevels;
    uint32_t arrayLayers;
    VkBuffer stagingBuffer;
    VkDeviceMemory stagingMemory; // offset: +0x20
} DecodedImageResource;

// Represents a pending decode operation queued for processing.
typedef struct DecodeOperation {
    BoundBufferInfo* bufferInfo;
    DecodedImageResource* imageInfo;
} DecodeOperation;

// The main context object for the texture decoder.
typedef struct TextureDecoder {
    VkDevice device;
    ArrayList decodedImages;   // list at +0x10, count at +0x8
    ArrayList boundBuffers;    // list at +0x20, count at +0x18
    ArrayDeque pendingDecodes; // deque at +0x28
} TextureDecoder;

```

---
### 2. Function Decompilations

#### 2.1. `vt_handle_vkGetPhysicalDeviceImageFormatProperties` & `getCompressedImageFormatProperties`

**Purpose**: To intercept queries for format properties and fake support for BCn formats if the native driver doesn't support them. This is the first step in the interception chain.

```c
/**
 * @brief Provides fallback properties for certain compressed image formats.
 *
 * @param format The Vulkan image format.
 * @param pImageFormatProperties The output structure to be filled.
 * @return VK_SUCCESS if the format is a supported compressed type, otherwise VK_ERROR_FORMAT_NOT_SUPPORTED.
 */
VkResult getCompressedImageFormatProperties(VkFormat format, VkImageFormatProperties* pImageFormatProperties) {
    // Original check: `if (param_1 - 0x83U < 0xc)`
    // This covers VK_FORMAT_BC1_RGB_UNORM_BLOCK (131) to VK_FORMAT_BC5_SNORM_BLOCK (142).
    if (format >= VK_FORMAT_BC1_RGB_UNORM_BLOCK && format <= VK_FORMAT_BC5_SNORM_BLOCK) {
        
        // param_2[1] = 0xf00000001;
        pImageFormatProperties->maxExtent = (VkExtent3D){16384, 16384, 1};  // offset: +0x00
        pImageFormatProperties->maxMipLevels = 15;                          // offset: +0x0C
        
        // *param_2 = 0x400000004000; (This line seems to be a mistake in the original Ghidra output,
        // as it overwrites width and height. The individual assignments are correct.)

        // param_2[2] = 0x100000800;
        pImageFormatProperties->maxArrayLayers = 2048;                      // offset: +0x10
        pImageFormatProperties->sampleCounts = VK_SAMPLE_COUNT_1_BIT;       // offset: +0x14
        
        // param_2[3] = 0x80000000;
        pImageFormatProperties->maxResourceSize = 2147483648;              // offset: +0x18 (2GB)

        return VK_SUCCESS; // 0
    }
    return VK_ERROR_FORMAT_NOT_SUPPORTED; // -11 (0xfffffff5)
}

/**
 * @brief Handles a proxied vkGetPhysicalDeviceImageFormatProperties command.
 *
 * @param ctx A pointer to the Vulkan GPU process context.
 */
void vt_handle_vkGetPhysicalDeviceImageFormatProperties(VulkanGpuProcess* ctx) {
    // 1. Argument Deserialization from the command buffer
    char* pArg = ctx->pArgBuffer;
    VkPhysicalDevice physicalDevice = (VkPhysicalDevice)*(uint64_t*)(pArg);  // offset: +0x0 (within arg block)
    VkFormat format = *(VkFormat*)(pArg + 8);                              // offset: +0x8
    VkImageType type = *(VkImageType*)(pArg + 12);                         // offset: +0xC
    VkImageTiling tiling = *(VkImageTiling*)(pArg + 16);                   // offset: +0x10
    VkImageUsageFlags usage = *(VkImageUsageFlags*)(pArg + 20);            // offset: +0x14
    VkImageCreateFlags flags = *(VkImageCreateFlags*)(pArg + 24);          // offset: +0x18

    // 2. Handle Translation
    VkPhysicalDevice physicalDeviceObject = (VkPhysicalDevice)VkObject_fromId((uint64_t)physicalDevice);
    
    VkImageFormatProperties imageFormatProperties = {0};
    
    // 3. Native Vulkan Call
    VkResult result = vkGetPhysicalDeviceImageFormatProperties(
        physicalDeviceObject, format, type, tiling, usage, flags, &imageFormatProperties);

    // 4. Fallback Logic for Compressed Formats
    if (result == VK_ERROR_FORMAT_NOT_SUPPORTED) { // result == -11
        if (isCompressedFormat(format)) {
            // Provide our own dummy properties and report success.
            result = getCompressedImageFormatProperties(format, &imageFormatProperties);
        }
    }
    
    // 5. Serialize and write the response
    RingBuffer_write(ctx->pResponseBuffer, &result, sizeof(VkResult));
    RingBuffer_write(ctx->pResponseBuffer, &imageFormatProperties, sizeof(VkImageFormatProperties));
}
```

---
#### 2.2. `TextureDecoder_createImage`

**Purpose**: Intercepts `vkCreateImage` for BCn formats. It creates an uncompressed target `VkImage` and a `VkBuffer` to serve as a staging area for the eventual CPU-decoded data.

```c
/**
 * @brief Creates a Vulkan image, with special handling for compressed formats.
 *
 * @param decoder A pointer to the TextureDecoder context.
 * @param pCreateInfo A pointer to the VkImageCreateInfo struct. This struct may be modified.
 * @param pImage A pointer to a VkImage handle where the result is stored.
 * @return A VkResult indicating the success or failure of the operation.
 */
VkResult TextureDecoder_createImage(TextureDecoder *decoder, VkImageCreateInfo* pCreateInfo, VkImage* pImage) {
    *pImage = VK_NULL_HANDLE;

    // The original code checks if (format >= 0x83 && format < 0x8F), which corresponds
    // to the Vulkan block compression formats BC1 through BC5.
    if (pCreateInfo->format < VK_FORMAT_BC1_RGB_UNORM_BLOCK || pCreateInfo->format > VK_FORMAT_BC5_SNORM_BLOCK) {
        return VK_ERROR_FORMAT_NOT_SUPPORTED; // -11
    }

    DecodedImageResource* resource = (DecodedImageResource*)calloc(1, sizeof(DecodedImageResource));
    if (!resource) return VK_ERROR_OUT_OF_HOST_MEMORY; // -1

    resource->originalFormat = pCreateInfo->format;        // format: +0x18
    resource->extent = pCreateInfo->extent;                // extent: +0x1C
    resource->mipLevels = pCreateInfo->mipLevels;          // mipLevels: +0x28
    resource->arrayLayers = pCreateInfo->arrayLayers;      // arrayLayers: +0x2C

    pCreateInfo->format = VK_FORMAT_B8G8R8A8_UNORM;                               // VK_FORMAT_B8G8R8A8_UNORM = 44 (0x2c)
    pCreateInfo->mipLevels = 1;                                                   // mipLevels: +0x28
    pCreateInfo->usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT; // Simplified from original's 0x1a
    pCreateInfo->flags &= ~VK_IMAGE_CREATE_BLOCK_TEXEL_VIEW_COMPATIBLE_BIT;       // flags: +0x10, VK_IMAGE_CREATE_BLOCK_TEXEL_VIEW_COMPATIBLE_BIT = 0x80

    // ... (pNext chain modification logic omitted for clarity) ...
    
    VkResult result = vkCreateImage(decoder->device, pCreateInfo, NULL, &resource->decodedImage);
    if (result != VK_SUCCESS) goto cleanup_fail;

    VkBufferCreateInfo bufferCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,                        // sType: +0x0, VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO = 12
        .size = (size_t)pCreateInfo->extent.width * pCreateInfo->extent.height * 4,
        .usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT,                            // usage: +0x10, VK_BUFFER_USAGE_TRANSFER_SRC_BIT = 1
        .sharingMode = VK_SHARING_MODE_EXCLUSIVE,                             // sharingMode: +0x14, VK_SHARING_MODE_EXCLUSIVE = 0
    };
    result = vkCreateBuffer(decoder->device, &bufferCreateInfo, NULL, &resource->stagingBuffer);
    if (result != VK_SUCCESS) goto cleanup_fail;

    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(decoder->device, resource->stagingBuffer, &memRequirements);
    
    uint32_t memoryTypeIndex = getMemoryTypeIndex(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
    if (memoryTypeIndex == (uint32_t)-1) {
        result = VK_ERROR_INITIALIZATION_FAILED; // -3
        goto cleanup_fail;
    }

    VkMemoryAllocateInfo allocInfo = { .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO, .allocationSize = memRequirements.size, .memoryTypeIndex = memoryTypeIndex };
    result = vkAllocateMemory(decoder->device, &allocInfo, NULL, &resource->stagingMemory);
    if (result != VK_SUCCESS) goto cleanup_fail;

    result = vkBindBufferMemory(decoder->device, resource->stagingBuffer, resource->stagingMemory, 0);
    if (result != VK_SUCCESS) goto cleanup_fail;

    ArrayList_add(decoder->decodedImages, resource);
    *pImage = resource->decodedImage;
    return VK_SUCCESS;

cleanup_fail:
    if (resource->decodedImage) vkDestroyImage(decoder->device, resource->decodedImage, NULL);
    if (resource->stagingBuffer) vkDestroyBuffer(decoder->device, resource->stagingBuffer, NULL);
    if (resource->stagingMemory) vkFreeMemory(decoder->device, resource->stagingMemory, NULL);
    free(resource);
    return result;
}
```

---
#### 2.3. `vt_handle_vkBindBufferMemory` & `TextureDecoder_addBoundBuffer`

**Purpose**: To track which `VkDeviceMemory` object is bound to which `VkBuffer`. This is crucial for the decoder to know where the compressed texture data resides when it needs to be mapped for CPU access.

```c
/**
 * @brief Tracks the binding of a Vulkan buffer to a memory object.
 *
 * @param decoder The texture decoder context object.
 * @param memory The memory object the buffer is bound to.
 * @param buffer The Vulkan buffer handle.
 * @param offset The offset within the memory object where the buffer is bound.
 */
void TextureDecoder_addBoundBuffer(TextureDecoder* decoder, ResourceMemory* memory, VkBuffer buffer, VkDeviceSize offset) {
    if (memory == NULL || memory->fd < 0) { // Original check `*param_2 < 1`
        return;
    }

    // Check if this buffer is already tracked and remove the old entry if so.
    for (uint32_t i = 0; i < decoder->boundBuffers.count; ++i) { // `if (0 < (int)*(uint *)(param_1 + 0x18))`
        BoundBufferInfo* existingInfo = decoder->boundBuffers.list[i]; // `**(long **)(*(long *)(param_1 + 0x20) + uVar2 * 8)`
        if (existingInfo->buffer == buffer) { // `== param_3`
            free(existingInfo);
            ArrayList_removeAt(&decoder->boundBuffers, i);
            break;
        }
    }

    // Allocate and add new tracking info.
    BoundBufferInfo* newInfo = (BoundBufferInfo*)calloc(1, sizeof(BoundBufferInfo)); // size is 0x18
    if (newInfo) {
        newInfo->buffer = buffer;   // `*plVar1 = param_3;`         (offset: +0x0)
        newInfo->offset = offset;   // `plVar1[1] = param_4;`        (offset: +0x8)
        newInfo->memory = memory;   // `plVar1[2] = (long)param_2;` (offset: +0x10)
        ArrayList_add(&decoder->boundBuffers, newInfo);
    }
}

/**
 * @brief Handles a proxied vkBindBufferMemory command.
 *
 * @param ctx A pointer to the Vulkan GPU process context.
 */
void vt_handle_vkBindBufferMemory(VulkanGpuProcess* ctx) {
    char* pArg = ctx->pArgBuffer;
    VkBuffer bufferHandle = *(VkBuffer*)(pArg);
    VkDeviceMemory memoryHandle = *(VkDeviceMemory*)(pArg + 8);
    VkDeviceSize memoryOffset = *(VkDeviceSize*)(pArg + 16);

    VkBuffer bufferObject = (VkBuffer)VkObject_fromId((uint64_t)bufferHandle);
    ResourceMemory* memoryObject = (ResourceMemory*)VkObject_fromId((uint64_t)memoryHandle);

    VkResult result = vkBindBufferMemory(
        ctx->device,          // *(undefined8 *)(param_1 + 0x10)
        bufferObject,         // uVar3
        memoryObject->memoryHandle, // *(undefined8 *)(lVar2 + 0x20)
        memoryOffset          // uVar8
    );

    if (result == VK_SUCCESS && ctx->pTextureDecoder != NULL) {
        TextureDecoder_addBoundBuffer(ctx->pTextureDecoder, memoryObject, bufferObject, memoryOffset);
    }
    
    RingBuffer_write(ctx->pResponseBuffer, &result, sizeof(VkResult));
}
```

---
#### 2.4. `vt_handle_vkCmdCopyBufferToImage` & `TextureDecoder_copyBufferToImage`

**Purpose**: To intercept copy commands and redirect them to the staging buffer instead of the final image, queuing a decode operation.

```c
/**
 * @brief Intercepts a vkCmdCopyBufferToImage call to handle compressed texture decoding.
 *
 * @param decoder The texture decoder context object.
 * @param commandBuffer The command buffer to record the copy command into.
 * @param srcBuffer The source buffer containing compressed texture data.
 * @param dstImage The destination image, which should be a decoded image resource.
 * @param dstImageLayout The layout of the destination image.
 */
void TextureDecoder_copyBufferToImage(TextureDecoder* decoder, VkCommandBuffer commandBuffer, VkBuffer srcBuffer, VkImage dstImage, VkImageLayout dstImageLayout) {
    
    // Search for the source buffer in the list of bound buffers.
    for (uint32_t i = 0; i < decoder->boundBuffers.count; ++i) { // `if (0 < (int)*(uint *)(param_1 + 0x18))`
        BoundBufferInfo* bufferInfo = decoder->boundBuffers.list[i]; // `**(long **)(*(long *)(param_1 + 0x20) + uVar3 * 8)`
        
        if (bufferInfo->buffer == srcBuffer) { // `== param_3`

            // If the buffer is found, search for the destination image.
            for (uint32_t j = 0; j < decoder->decodedImages.count; ++j) { // `if (0 < (int)*(uint *)(param_1 + 8))`
                DecodedImageResource* imageInfo = decoder->decodedImages.list[j]; // `**(long **)(*(long *)(param_1 + 0x10) + uVar3 * 8)`

                if (imageInfo->decodedImage == dstImage) { // `== param_4`
                    
                    DecodeOperation* op = (DecodeOperation*)calloc(1, sizeof(DecodeOperation)); // `calloc(1, 0x10)`
                    if (op) {
                        op->bufferInfo = bufferInfo;
                        op->imageInfo = imageInfo;
                        ArrayDeque_addLast(&decoder->pendingDecodes, op); // `ArrayDeque_addLast(param_1 + 0x28, puVar2)`

                        VkBufferImageCopy region = {
                            .bufferOffset = 0, // `uStack_88 = 0; local_90 = 0;`
                            .imageSubresource = { .aspectMask = VK_IMAGE_ASPECT_COLOR_BIT, .layerCount = 1 }, // `uStack_5c = 1`
                            .imageExtent = imageInfo->extent // `uStack_64 = ...`
                        };
                        
                        // Record command to copy from user buffer to OUR staging buffer.
                        vkCmdCopyBufferToImage(commandBuffer, srcBuffer, imageInfo->stagingBuffer, dstImageLayout, 1, &region);
                    }
                    return;
                }
            }
        }
    }
}

/**
 * @brief Handles a proxied vkCmdCopyBufferToImage command.
 */
void vt_handle_vkCmdCopyBufferToImage(VulkanGpuProcess* ctx) {
    char* pArg = ctx->pArgBuffer;
    VkCommandBuffer commandBufferHandle = *(VkCommandBuffer*)pArg;
    VkBuffer srcBufferHandle = *(VkBuffer*)(pArg + 8);
    VkImage dstImageHandle = *(VkImage*)(pArg + 16);
    VkImageLayout dstImageLayout = *(VkImageLayout*)(pArg + 24);
    uint32_t regionCount = *(uint32_t*)(pArg + 28);
    VkBufferImageCopy* pRegions = (VkBufferImageCopy*)(pArg + 32);

    VkCommandBuffer commandBuffer = (VkCommandBuffer)VkObject_fromId((uint64_t)commandBufferHandle);
    VkBuffer srcBuffer = (VkBuffer)VkObject_fromId((uint64_t)srcBufferHandle);
    VkImage dstImage = (VkImage)VkObject_fromId((uint64_t)dstImageHandle);

    if (ctx->pTextureDecoder != NULL && TextureDecoder_containsImage(ctx->pTextureDecoder, dstImage)) {
        // The check on bufferOffset ensures we only intercept the initial data load.
        if (regionCount > 0 && pRegions[0].bufferOffset == 0) {
            TextureDecoder_copyBufferToImage(ctx->pTextureDecoder, commandBuffer, srcBuffer, dstImage, dstImageLayout);
        }
    } else {
        // Not a managed texture, pass the call to the native driver.
        vkCmdCopyBufferToImage(commandBuffer, srcBuffer, dstImage, dstImageLayout, regionCount, pRegions);
    }
}
```

---
#### 2.5. `vt_handle_vkQueueSubmit`

**Purpose**: The final trigger. It ensures all pending CPU-side decoding work is finished before the command buffers are submitted to the GPU.

```c
/**
 * @brief Handles a proxied vkQueueSubmit command, triggering texture decoding.
 *
 * @param ctx A pointer to the Vulkan GPU process context.
 */
void vt_handle_vkQueueSubmit(VulkanGpuProcess* ctx) {
    char* pArg = ctx->pArgBuffer;
    VkQueue queueHandle = *(VkQueue*)pArg;
    uint32_t submitCount = *(uint32_t*)(pArg + 8);
    VkFence fenceHandle = *(VkFence*)(pArg + 12);

    VkQueue queue = (VkQueue)VkObject_fromId((uint64_t)queueHandle);
    VkFence fence = (VkFence)VkObject_fromId((uint64_t)fenceHandle);
    
    // Deserialize the entire VkSubmitInfo array and its pNext chains.
    // This is a complex process abstracted here for clarity.
    VkSubmitInfo* pSubmits = deserialize_submit_info_array(pArg + 16, submitCount);

    // If the texture decoder is active, process all queued operations.
    if (ctx->pTextureDecoder != NULL) {
        TextureDecoder_decodeAll(ctx->pTextureDecoder);
    }

    // Call the native Vulkan function.
    VkResult result = vkQueueSubmit(queue, submitCount, pSubmits, fence);
    
    if (result == VK_ERROR_DEVICE_LOST) { // VK_ERROR_DEVICE_LOST = -4
        ctx->lastError = VK_ERROR_DEVICE_LOST;
    }
    
    // Write the result back to the calling process.
    RingBuffer_write(ctx->pResponseBuffer, &result, sizeof(VkResult));
    
    // Cleanup temporary memory for the deserialized pSubmits array.
}
```

---
#### 2.6. `TextureDecoder_decodeAll` & Block Decoders

**Purpose**: This is the core CPU-side workhorse. It processes the queue of pending operations and calls the appropriate low-level block decoder for each pixel block.

```c
/**
 * @brief Decodes a single 4x4 block of a single-channel BC4 or BC5 compressed texture.
 */
void decode_bc4_bc5_channel_block(uint64_t compressedBlock64, uint8_t* outputRGBAPtr, int x, int y, int imageWidth, int imageHeight, int imageStride, int channelOffset, bool isSigned);

/**
 * @brief Decodes a single 4x4 block of a BC1 (DXT1) compressed texture.
 */
void decode_bc1_block(const uint8_t* compressedBlock, uint32_t* outputRGBA, bool isBC1Alpha);

/**
 * @brief Processes all pending texture decoding operations.
 *
 * @param decoder The texture decoder context object.
 */
void TextureDecoder_decodeAll(TextureDecoder* decoder) {
    // Loop until the queue of pending decode operations is empty.
    while (!ArrayDeque_isEmpty(&decoder->pendingDecodes)) {
        DecodeOperation* op = (DecodeOperation*)ArrayDeque_removeFirst(&decoder->pendingDecodes);
        if (!op) continue;

        BoundBufferInfo* bufferInfo = op->bufferInfo;
        DecodedImageResource* imageInfo = op->imageInfo;
        free(op);

        if (!bufferInfo || !imageInfo) continue;
        
        // Map the source buffer (containing compressed data) into CPU address space.
        void* compressedDataSrc = mmap(NULL, bufferInfo->memory->size, PROT_READ, MAP_SHARED, bufferInfo->memory->fd, 0);
        if (compressedDataSrc == MAP_FAILED) continue;
        
        // Map the destination staging buffer into CPU address space.
        void* uncompressedDataDst;
        size_t stagingBufferSize = (size_t)imageInfo->extent.width * imageInfo->extent.height * 4;
        vkMapMemory(decoder->device, imageInfo->stagingMemory, 0, stagingBufferSize, 0, &uncompressedDataDst);
        
        memset(uncompressedDataDst, 0, stagingBufferSize);

        // Calculate a zero-based index from the Vulkan format enum.
        uint32_t formatIndex = imageInfo->originalFormat - VK_FORMAT_BC1_RGB_UNORM_BLOCK; // 131
        
        if (formatIndex < 12) { // Covers BC1-BC5
            uint32_t width = imageInfo->extent.width;
            uint32_t height = imageInfo->extent.height;
            const uint8_t* compressedPtr = (const uint8_t*)((long)compressedDataSrc + bufferInfo->offset);
            
            for (uint32_t y = 0; y < height; y += 4) {
                for (uint32_t x = 0; x < width; x += 4) {
                    uint32_t* outputPixelBlock = (uint32_t*)((char*)uncompressedDataDst + (y * width + x) * 4);
                    
                    switch (formatIndex) {
                        case 0: // BC1_RGB_UNORM
                        case 1: // BC1_RGB_SRGB
                            decode_bc1_block(compressedPtr, outputPixelBlock, false);
                            compressedPtr += 8;
                            break;
                        case 2: // BC1_RGBA_UNORM
                        case 3: // BC1_RGBA_SRGB
                            decode_bc1_block(compressedPtr, outputPixelBlock, true);
                            compressedPtr += 8;
                            break;
                        case 4: // BC2_UNORM
                        case 5: // BC2_SRGB
                            // FUN_00137d78 in the original binary
                            decode_bc2_block(compressedPtr, outputPixelBlock); 
                            compressedPtr += 16;
                            break;
                        case 6: // BC3_UNORM
                        case 7: // BC3_SRGB
                            // FUN_00138020 handles the alpha block for BC3
                            decode_bc4_bc5_channel_block(*(uint64_t*)compressedPtr, (uint8_t*)outputPixelBlock, x, y, width, height, width * 4, 3, false);
                            // FUN_00137d78 handles the color block for BC3
                            decode_bc1_block(compressedPtr + 8, outputPixelBlock, false); 
                            compressedPtr += 16;
                            break;
                        case 8: // BC4_UNORM
                            decode_bc4_bc5_channel_block(*(uint64_t*)compressedPtr, (uint8_t*)outputPixelBlock, x, y, width, height, width * 4, 0, false);
                            compressedPtr += 8;
                            break;
                        case 9: // BC4_SNORM
                            decode_bc4_bc5_channel_block(*(uint64_t*)compressedPtr, (uint8_t*)outputPixelBlock, x, y, width, height, width * 4, 0, true);
                            compressedPtr += 8;
                            break;
                        case 10: // BC5_UNORM
                            decode_bc4_bc5_channel_block(*(uint64_t*)compressedPtr, (uint8_t*)outputPixelBlock, x, y, width, height, width * 4, 0, false);
                            decode_bc4_bc5_channel_block(*(uint64_t*)(compressedPtr + 8), (uint8_t*)outputPixelBlock, x, y, width, height, width * 4, 1, false);
                            compressedPtr += 16;
                            break;
                        case 11: // BC5_SNORM
                            decode_bc4_bc5_channel_block(*(uint64_t*)compressedPtr, (uint8_t*)outputPixelBlock, x, y, width, height, width * 4, 0, true);
                            decode_bc4_bc5_channel_block(*(uint64_t*)(compressedPtr + 8), (uint8_t*)outputPixelBlock, x, y, width, height, width * 4, 1, true);
                            compressedPtr += 16;
                            break;
                    }
                }
            }
        }
        
        vkUnmapMemory(decoder->device, imageInfo->stagingMemory);
        munmap(compressedDataSrc, bufferInfo->memory->size);
    }
}
```
---

# Decompilations

## `TextureDecoder_createImage` 

### Type Analysis

By cross-referencing the function's usage patterns with the provided `vulkan_core.h` header, the following types and values can be determined:

*   **`param_1` (type `undefined8 *`)**: This is a context or state object that contains the `VkDevice` handle, as it's the first parameter to Vulkan functions like `vkCreateImage`. We'll alias this as `VulkanContext*`.
*   **`param_2` (type `long`)**: This is a pointer to the main input structure. The member accesses (e.g., `+0x18`, `+0x10`, `+0x28`, `+0x38`) correspond directly to the layout of `VkImageCreateInfo`. Thus, this is a `VkImageCreateInfo*`.
*   **`param_3` (type `long *`)**: This is an output parameter that receives the created `VkImage` handle. This is a `VkImage*`.
*   **`__ptr` (type `long *`)**: A locally allocated structure to track the resources created for the decoded image. Its size is `0x28` bytes (40 bytes). We'll name this `DecodedImageResource`.
*   **`_DAT_` global variables**: These are function pointers to the Vulkan API, loaded dynamically.
    *   `_DAT_0013c828`: `vkCreateImage`
    *   `DAT_0013c808`: `vkCreateBuffer`
    *   `_DAT_0013c750`: `vkGetBufferMemoryRequirements`
    *   `_DAT_0013c718`: `vkAllocateMemory`
    *   `_DAT_0013c758`: `vkBindBufferMemory`
    *   `_DAT_0013c830`: `vkDestroyImage`
    *   `_DAT_0013c810`: `vkDestroyBuffer`
    *   `_DAT_0013c720`: `vkFreeMemory`
*   **Enum and Hex Values**:
    *   The check `iVar3 - 0x83U < 0xc` corresponds to `(format >= VK_FORMAT_BC1_RGB_UNORM_BLOCK && format <= VK_FORMAT_BC5_SNORM_BLOCK)`. `0x83` is `131`.
    *   The format `0x2c` is `VK_FORMAT_B8G8R8A8_UNORM` (44).
    *   The flag `0xffffff7f` is used to clear the `0x80` bit, which is `VK_IMAGE_CREATE_BLOCK_TEXEL_VIEW_COMPATIBLE_BIT`.
    *   The usage flags `0x1a` (`0x2 | 0x8 | 0x10`) correspond to `VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_STORAGE_BIT | VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT`. The intent is likely for decoding and sampling, so a more common combination would be `TRANSFER_DST | SAMPLED`. The decompilation reflects the original binary's value.
    *   `VK_BUFFER_USAGE_TRANSFER_SRC_BIT` is `1`.
    *   `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT` is `2`.
    *   `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` is `4`.

### Decompiled C Code

```c
#include <stdlib.h>
#include <stdbool.h>

// Forward-declare from vulkan_core.h as they are used in the function signature.
// In a real project, you would just #include "vulkan_core.h"
typedef struct VkImageCreateInfo_T* VkImageCreateInfo;
typedef struct VkImage_T* VkImage;
typedef int VkResult;

// A hypothetical structure to hold context, inferred from usage.
typedef struct VulkanContext {
    VkDevice device;
    // ... other members
} VulkanContext;

// A hypothetical structure to manage decoded image resources, inferred from usage.
typedef struct DecodedImageResource {
    VkImage decodedImage;
    VkFormat originalFormat;
    uint32_t field_C; // Unknown field
    VkExtent3D originalExtent;
    VkBuffer stagingBuffer;
    VkDeviceMemory stagingMemory;
} DecodedImageResource;

/**
 * @brief Creates a Vulkan image, with special handling for compressed formats.
 *
 * If a supported compressed format is provided, this function allocates an
 * uncompressed target image and a staging buffer for the compressed data.
 * This prepares for a subsequent decode-and-upload operation.
 * Otherwise, it returns VK_ERROR_FORMAT_NOT_SUPPORTED.
 *
 * @param ctx A pointer to the application's Vulkan context.
 * @param pCreateInfo A pointer to the VkImageCreateInfo struct. This struct may be modified.
 * @param pImage A pointer to a VkImage handle where the created image is returned.
 * @return A VkResult indicating the success or failure of the operation.
 */
VkResult TextureDecoder_createImage(VulkanContext *ctx, VkImageCreateInfo* pCreateInfo, VkImage* pImage) {
    if (!pImage || !pCreateInfo || !ctx) {
        return VK_ERROR_INITIALIZATION_FAILED; // -3
    }
    *pImage = VK_NULL_HANDLE;

    // The original code checks if (format >= 0x83 && format < 0x8F), which corresponds
    // to the Vulkan block compression formats BC1 through BC5.
    if (pCreateInfo->format < VK_FORMAT_BC1_RGB_UNORM_BLOCK || pCreateInfo->format > VK_FORMAT_BC5_SNORM_BLOCK) {
        return VK_ERROR_FORMAT_NOT_SUPPORTED; // -11
    }

    // Allocate a structure to hold information about the decoded image and its staging resources.
    DecodedImageResource* resource = (DecodedImageResource*)calloc(1, sizeof(DecodedImageResource));
    if (!resource) {
        return VK_ERROR_OUT_OF_HOST_MEMORY; // -1
    }

    // Store original image properties before modifying the create info struct.
    resource->originalFormat = pCreateInfo->format;        // format: +0x18
    resource->originalExtent = pCreateInfo->extent;        // extent: +0x1C
    resource->mipLevels = pCreateInfo->mipLevels;          // mipLevels: +0x28
    resource->arrayLayers = pCreateInfo->arrayLayers;      // arrayLayers: +0x2C

    // Modify the create info struct for the target (decoded) image.
    pCreateInfo->format = VK_FORMAT_B8G8R8A8_UNORM;                               // VK_FORMAT_B8G8R8A8_UNORM = 44 (0x2c)
    pCreateInfo->mipLevels = 1;                                                   // mipLevels: +0x28
    pCreateInfo->usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT |                        // usage: +0x38, value: 0x1a
                         VK_IMAGE_USAGE_STORAGE_BIT |                             // VK_IMAGE_USAGE_TRANSFER_DST_BIT = 2
                         VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;                     // VK_IMAGE_USAGE_STORAGE_BIT = 8, VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT = 16
                                                                                  
    pCreateInfo->flags &= ~VK_IMAGE_CREATE_BLOCK_TEXEL_VIEW_COMPATIBLE_BIT;       // flags: +0x10, VK_IMAGE_CREATE_BLOCK_TEXEL_VIEW_COMPATIBLE_BIT = 0x80

    // ... The original code appears to modify an existing VkImageFormatListCreateInfo ...
    // ... in the pNext chain if one exists, which is omitted here for clarity. ...
    
    // Create the target image that will receive the decoded data.
    VkResult result = vkCreateImage(ctx->device, pCreateInfo, NULL, &resource->decodedImage);
    if (result != VK_SUCCESS) {
        free(resource);
        return result;
    }

    // Create a staging buffer for the compressed texture data.
    VkBufferCreateInfo bufferCreateInfo = {
        .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO,                        // sType: +0x0, VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO = 12
        .size = (long)pCreateInfo->extent.width * pCreateInfo->extent.height * 4, // A rough size guess
        .usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT,                            // usage: +0x10, VK_BUFFER_USAGE_TRANSFER_SRC_BIT = 1
        .sharingMode = VK_SHARING_MODE_EXCLUSIVE,                             // sharingMode: +0x14, VK_SHARING_MODE_EXCLUSIVE = 0
    };
    result = vkCreateBuffer(ctx->device, &bufferCreateInfo, NULL, &resource->stagingBuffer);
    if (result != VK_SUCCESS) {
        vkDestroyImage(ctx->device, resource->decodedImage, NULL);
        free(resource);
        return result;
    }

    // Allocate and bind memory for the staging buffer.
    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(ctx->device, resource->stagingBuffer, &memRequirements);

    uint32_t memoryTypeIndex = getMemoryTypeIndex(memRequirements.memoryTypeBits, 
                                                  VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | // 2
                                                  VK_MEMORY_PROPERTY_HOST_COHERENT_BIT); // 4 -> flag sum = 6
    if (memoryTypeIndex == (uint32_t)-1) { // Assuming a failure return value
        result = VK_ERROR_INITIALIZATION_FAILED; // -3
        goto cleanup_fail;
    }

    VkMemoryAllocateInfo allocInfo = {
        .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO, // sType: +0x0, VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO = 5
        .allocationSize = memRequirements.size,
        .memoryTypeIndex = memoryTypeIndex,
    };
    result = vkAllocateMemory(ctx->device, &allocInfo, NULL, &resource->stagingMemory);
    if (result != VK_SUCCESS) {
        goto cleanup_fail;
    }

    result = vkBindBufferMemory(ctx->device, resource->stagingBuffer, resource->stagingMemory, 0);
    if (result != VK_SUCCESS) {
        goto cleanup_fail;
    }

    // Success: store the decoded image information and return the handle to the *decoded* image.
    ArrayList_add(decoder->decodedImages, resource);
    *pImage = resource->decodedImage;

    return VK_SUCCESS;

cleanup_fail:
    // Meticulous cleanup on failure.
    if (resource->decodedImage != VK_NULL_HANDLE) {
        vkDestroyImage(ctx->device, resource->decodedImage, NULL);
    }
    if (resource->stagingBuffer != VK_NULL_HANDLE) {
        vkDestroyBuffer(ctx->device, resource->stagingBuffer, NULL);
    }
    if (resource->stagingMemory != VK_NULL_HANDLE) {
        vkFreeMemory(ctx->device, resource->stagingMemory, NULL);
    }
    free(resource);
    return result;
}
```

---

## `TextureDecoder_addBoundBuffer`

### Type Analysis

By observing the function's interactions with its parameters and its usage in the broader (previously analyzed) context, we can determine the types:

*   **`param_1` (type `long`)**: This is a pointer to a larger context structure, which we've identified as `TextureDecoder*`. It contains a list of tracked buffer bindings.
    *   The access at offset `+0x18` is the count of items in the list.
    *   The access at offset `+0x20` is the pointer to the list's data array.
*   **`param_2` (type `int *`)**: This is a pointer to the `ResourceMemory` struct that was allocated for the `VkDeviceMemory`. The check `*param_2 < 1` is likely a simplified check on one of its fields, probably the `size` or a file descriptor `fd`, to ensure it's valid. For clarity, we'll represent it as `ResourceMemory*`.
*   **`param_3` (type `long`)**: This is the `VkBuffer` handle being bound.
*   **`param_4` (type `long`)**: This is the `VkDeviceSize` memory offset for the binding.
*   **`plVar1` (type `long *`)**: This is a pointer to a new heap-allocated structure used to track the buffer-memory binding. Its size is `0x18` (24 bytes).

### Function Logic

The function's purpose is to register a buffer-to-memory binding within the `TextureDecoder`'s internal tracking system. This is necessary because the decoder needs to know which `VkDeviceMemory` is backing a given `VkBuffer` when it comes time to decompress and upload texture data.

The logic is as follows:
1.  It first performs a sanity check on the `ResourceMemory` object to ensure it's valid (e.g., has a non-zero size or valid handle).
2.  It then iterates through its existing list of tracked buffers (`boundBuffers`).
3.  If it finds an existing entry for the same `VkBuffer` handle, it assumes the buffer is being rebound. It frees the old tracking information and removes it from the list.
4.  It allocates a new `BoundBufferInfo` struct to hold the new binding information.
5.  It populates this new struct with the `VkBuffer` handle, the `VkDeviceSize` offset, and a pointer to the `ResourceMemory` struct.
6.  Finally, it adds the new `BoundBufferInfo` struct to its internal list of bound buffers.

### Decompiled C Code

```c
#include <stdlib.h>
#include <stdbool.h>

// Note: The following structs are hypothetical, based on the analysis of the assembly.
// They represent how the original program manages its Vulkan-related state.

typedef struct VkBuffer_T* VkBuffer;
typedef unsigned long long VkDeviceSize;

// Represents a tracked Vulkan memory allocation.
typedef struct ResourceMemory {
    int fd;                 // File descriptor, if applicable. The check `*param_2 < 1` implies this.
    // ... other members like size, VkDeviceMemory handle, etc.
} ResourceMemory;

// Represents a tracked buffer-to-memory binding.
typedef struct BoundBufferInfo {
    VkBuffer buffer;          // offset: +0x0
    VkDeviceSize offset;      // offset: +0x8
    ResourceMemory* memory;   // offset: +0x10
} BoundBufferInfo;

// The main context object for the texture decoder.
typedef struct TextureDecoder {
    // ... other members
    ArrayList boundBuffers;   // count at +0x18, data at +0x20
    // ... other members
} TextureDecoder;


/**
 * @brief Tracks the binding of a Vulkan buffer to a memory object.
 *
 * If the buffer is already tracked, the old tracking information is replaced.
 * This is used by the texture decoder to manage resources for compressed textures.
 *
 * @param decoder The texture decoder context object.
 * @param memory The memory object the buffer is bound to.
 * @param buffer The Vulkan buffer handle.
 * @param offset The offset within the memory object where the buffer is bound.
 */
void TextureDecoder_addBoundBuffer(TextureDecoder* decoder, ResourceMemory* memory, VkBuffer buffer, VkDeviceSize offset) {
    
    // Check if the provided memory object is valid (e.g., has a size > 0).
    // The original code checks `*param_2 < 1`, likely checking the fd or size.
    if (memory == NULL || memory->fd < 0) {
        return;
    }

    // Check if this buffer is already being tracked. If so, remove the old entry.
    for (uint32_t i = 0; i < decoder->boundBuffers.count; ++i) {
        BoundBufferInfo* existingInfo = decoder->boundBuffers.list[i];

        // if (existingInfo->buffer == buffer)
        if (existingInfo->buffer == buffer) { // `**(long **)(*(long *)(param_1 + 0x20) + uVar2 * 8) == param_3`
            
            // Free the old tracking struct and remove it from the list.
            free(existingInfo); // `free(__ptr)`
            ArrayList_removeAt(&decoder->boundBuffers, i); // `ArrayList_removeAt(param_1 + 0x18, ...)`
            break; // Exit loop after finding and removing the entry.
        }
    }

    // Allocate and populate a new struct to track this binding.
    // plVar1 = (long *)calloc(1, 0x18);
    BoundBufferInfo* newInfo = (BoundBufferInfo*)calloc(1, sizeof(BoundBufferInfo)); // sizeof is 0x18 (24) bytes
    if (newInfo) {
        newInfo->buffer = buffer;   // `*plVar1 = param_3;`         (offset: +0x0)
        newInfo->offset = offset;   // `plVar1[1] = param_4;`        (offset: +0x8)
        newInfo->memory = memory;   // `plVar1[2] = (long)param_2;` (offset: +0x10)
        
        // Add the new tracking info to the list.
        ArrayList_add(&decoder->boundBuffers, newInfo); // `ArrayList_add(param_1 + 0x18)`
    }
}
```

---

## `vt_handle_vkBindBufferMemory`

### Type Analysis

By cross-referencing the function's usage patterns with `vulkan_core.h` and the previously analyzed functions, we can determine the types:

*   **`param_1` (type `long`)**: This is a pointer to a context structure that manages the Vulkan call. We'll call it `VulkanGpuProcess`.
    *   `param_1 + 0x10`: `VkDevice device`
    *   `param_1 + 0x50`: `char* pArgBuffer`, a pointer to the serialized arguments for the Vulkan call.
    *   `param_1 + 0x68`: `RingBuffer* pResponseBuffer`, a pointer to the ring buffer for sending results back.
    *   `param_1 + 0x90`: `TextureDecoder* pTextureDecoder`, a pointer to the texture decoding context.
*   **Argument Deserialization**: The function reads arguments sequentially from the `pArgBuffer`. The structure of these arguments is specific to the `vkBindBufferMemory` command.
    *   `lVar2` (first argument): `VkBuffer buffer`
    *   `unaff_x22` (second argument): `VkDeviceMemory memory`
    *   `uVar8` (third argument): `VkDeviceSize memoryOffset`
*   **`VkObject_fromId`**: This helper function retrieves an application-level wrapper object (like our `ResourceMemory` struct from the previous analysis) from a Vulkan handle. The result for `VkDeviceMemory` is a pointer to a `ResourceMemory` struct, where the actual `VkDeviceMemory` handle is stored at offset `+0x20`.
*   **`_DAT_0013c758`**: This is a function pointer to `vkBindBufferMemory`.

### Function Logic

The function serves as a wrapper around the standard `vkBindBufferMemory` call, adding custom logic for the texture decoder.

1.  **Argument Parsing**: It begins by deserializing the arguments for `vkBindBufferMemory` (`buffer`, `memory`, `memoryOffset`) from the raw argument buffer.
2.  **Handle Translation**: It uses the `VkObject_fromId` utility to look up internal wrapper structures for the `VkBuffer` and `VkDeviceMemory` handles.
3.  **Vulkan Call**: It calls the real `vkBindBufferMemory` function using the retrieved handles and the provided offset.
4.  **Texture Decoder Hook**: If the `vkBindBufferMemory` call is successful (`VK_SUCCESS`) and the texture decoder context is active, it calls `TextureDecoder_addBoundBuffer`. This is done to inform the decoder that a specific buffer is now backed by a specific memory object, which is crucial for later upload and decode operations.
5.  **Response**: It writes the `VkResult` from the `vkBindBufferMemory` call back to the response ring buffer for the calling process.

### Decompiled C Code

```c
#include "vulkan_core.h"
#include <stdbool.h>

// Note: The following structs and functions are hypothetical, based on the analysis of the assembly.
// They represent how the original program manages its Vulkan-related state.

typedef struct VulkanGpuProcess {
    // ... other members
    VkDevice device;                          // offset: +0x10
    // ...
    char* pArgBuffer;                         // offset: +0x50
    RingBuffer* pResponseBuffer;              // offset: +0x68
    // ...
    TextureDecoder* pTextureDecoder;          // offset: +0x90
    // ...
} VulkanGpuProcess;

// Represents a tracked Vulkan memory allocation.
typedef struct ResourceMemory {
    // ... other members
    VkDeviceMemory memoryHandle; // offset: +0x20
    // ... other members
} ResourceMemory;

// Function to track buffer-to-memory bindings.
void TextureDecoder_addBoundBuffer(TextureDecoder* decoder, ResourceMemory* memory, VkBuffer buffer, VkDeviceSize offset);

// Internal utility to look up a wrapper object from a handle.
void* VkObject_fromId(uint64_t handle);


/**
 * @brief Handles a request to bind a Vulkan buffer to a memory object.
 *
 * This function wraps vkBindBufferMemory, adding a hook to notify the
 * texture decoder system about the new memory binding if it succeeds.
 *
 * @param ctx A pointer to the Vulkan GPU process context.
 */
void vt_handle_vkBindBufferMemory(VulkanGpuProcess* ctx) {
    
    // 1. Argument Deserialization from the command buffer
    // The Ghidra decompilation shows a complex but sequential read from ctx->pArgBuffer.
    // The simplified version is:
    char* pArg = ctx->pArgBuffer;
    VkBuffer buffer = *(VkBuffer*)(pArg);
    VkDeviceMemory memoryHandle = *(VkDeviceMemory*)(pArg + 8);
    VkDeviceSize memoryOffset = *(VkDeviceSize*)(pArg + 16);

    // 2. Handle Translation
    // The original code uses a helper `VkObject_fromId` to get its internal structs.
    VkBuffer bufferObject = (VkBuffer)VkObject_fromId((uint64_t)buffer);
    ResourceMemory* memoryObject = (ResourceMemory*)VkObject_fromId((uint64_t)memoryHandle);

    // 3. Vulkan API Call
    // (*_DAT_0013c758) corresponds to vkBindBufferMemory
    VkResult result = vkBindBufferMemory(
        ctx->device,          // *(undefined8 *)(param_1 + 0x10)
        bufferObject,         // uVar3
        memoryObject->memoryHandle, // *(undefined8 *)(lVar2 + 0x20)
        memoryOffset          // uVar8
    );

    // 4. Conditional call to the texture decoder system
    // if (result == VK_SUCCESS && ctx->pTextureDecoder != NULL)
    if ((result == VK_SUCCESS) && (ctx->pTextureDecoder != NULL)) {
        TextureDecoder_addBoundBuffer(
            ctx->pTextureDecoder, // *(long *)(param_1 + 0x90)
            memoryObject,         // lVar2
            bufferObject,         // uVar3
            memoryOffset          // uVar8
        );
    }
    
    // 5. Write the result back to the response buffer.
    // The original code writes an 8-byte value (VkResult + padding).
    RingBuffer_write(ctx->pResponseBuffer, &result, sizeof(VkResult));
}
```

---

## `TextureDecoder_copyBufferToImage`

### Type Analysis

Based on the function's parameters and how it interacts with memory, we can deduce the following types and their relationships:

*   **`param_1` (type `long`)**: This is the pointer to the main `TextureDecoder` context object. The function accesses several lists within this object.
    *   `param_1 + 0x18`: `boundBuffers.count`
    *   `param_1 + 0x20`: `boundBuffers.list` (an array of `BoundBufferInfo*`)
    *   `param_1 + 0x08`: `decodedImages.count`
    *   `param_1 + 0x10`: `decodedImages.list` (an array of `DecodedImageResource*`)
    *   `param_1 + 0x28`: A pointer to an `ArrayDeque` structure, which serves as a queue for pending decode operations.
*   **`param_2` (type `undefined8`)**: This is the `VkCommandBuffer` handle, as it's the first argument to the final Vulkan command.
*   **`param_3` (type `long`)**: This is the handle for the source `VkBuffer`.
*   **`param_4` (type `long`)**: This is the handle for the destination `VkImage`.
*   **`param_5` (type `undefined4`)**: This is the `VkImageLayout` for the destination image.
*   **`uVar4`**: A pointer to a `BoundBufferInfo` struct, found by searching `decoder->boundBuffers`.
*   **`lVar5`**: A pointer to a `DecodedImageResource` struct, found by searching `decoder->decodedImages`.
*   **`puVar2`**: A newly allocated structure of size `0x10` (16 bytes) that pairs a `BoundBufferInfo*` with a `DecodedImageResource*`. We can call this a `DecodeOperation`.
*   **`_DAT_0013ca10`**: This is a function pointer to the real `vkCmdCopyBufferToImage`.

### Function Logic

This function acts as an interceptor for `vkCmdCopyBufferToImage`. Its goal is to identify when a copy operation is intended for a compressed texture that the decoder is managing.

1.  **Find Tracked Buffer**: The function first iterates through its list of `boundBuffers` to see if the source buffer (`srcBuffer`) is one that it's tracking.
2.  **Find Tracked Image**: If the buffer is found, it then iterates through its list of `decodedImages` to see if the destination image (`dstImage`) is also being managed by the decoder.
3.  **Queue Decode and Issue Real Command**: If both are found, it signifies that compressed data is being copied into the staging buffer associated with the decoded image.
    *   It creates a `DecodeOperation` struct containing pointers to the matched `BoundBufferInfo` and `DecodedImageResource`.
    *   This operation is enqueued into the `pendingDecodes` deque for later processing (likely by `TextureDecoder_decodeAll`).
    *   It then constructs a `VkBufferImageCopy` region describing the transfer. The source buffer is the user's buffer, but the destination is the staging buffer associated with the decoded image.
    *   Finally, it calls the real `vkCmdCopyBufferToImage`, recording a command to transfer the compressed data from the user's buffer to the decoder's internal staging buffer.
4.  **No-op**: If the source buffer or destination image is not tracked, the function does nothing, effectively ignoring the copy command in this path.

### Decompiled C Code

```c
#include <stdlib.h>
#include <stdbool.h>

// Note: The following structs are hypothetical, based on the analysis of the assembly.
// They represent how the original program manages its Vulkan-related state.

typedef struct VkBuffer_T* VkBuffer;
typedef struct VkImage_T* VkImage;
typedef enum VkImageLayout_E VkImageLayout;
typedef unsigned long long VkDeviceSize;

// Represents a tracked Vulkan memory allocation.
typedef struct ResourceMemory {
    int fd;
    VkDeviceMemory memoryHandle;
} ResourceMemory;

// Represents a tracked buffer-to-memory binding.
typedef struct BoundBufferInfo {
    VkBuffer buffer;          // offset: +0x0
    VkDeviceSize offset;      // offset: +0x8
    ResourceMemory* memory;   // offset: +0x10
} BoundBufferInfo;

// Represents the resources for a compressed image that needs decoding.
typedef struct DecodedImageResource {
    VkImage decodedImage;
    VkFormat originalFormat;
    int field_C; // Unknown field
    VkExtent3D extent;
    uint32_t mipLevels;
    uint32_t arrayLayers;
    VkBuffer stagingBuffer;
    VkDeviceMemory stagingMemory;
} DecodedImageResource;

// Represents a pending decode operation.
typedef struct DecodeOperation {
    BoundBufferInfo* bufferInfo;
    DecodedImageResource* imageInfo;
} DecodeOperation;

// The main context object for the texture decoder.
typedef struct TextureDecoder {
    VkDevice device;
    ArrayList decodedImages;   // list at +0x10, count at +0x8
    ArrayList boundBuffers;    // list at +0x20, count at +0x18
    ArrayDeque pendingDecodes; // deque at +0x28
    // ... other members
} TextureDecoder;


/**
 * @brief Intercepts a vkCmdCopyBufferToImage call to handle compressed texture decoding.
 *
 * This function checks if both the source buffer and destination image are tracked by the
 * decoder. If so, it queues a decode operation and records a command to copy the
 * compressed data from the user's buffer to an internal staging buffer.
 *
 * @param decoder The texture decoder context object.
 * @param commandBuffer The command buffer to record the copy command into.
 * @param srcBuffer The source buffer containing compressed texture data.
 * @param dstImage The destination image, which should be a decoded image resource.
 * @param dstImageLayout The layout of the destination image.
 */
void TextureDecoder_copyBufferToImage(TextureDecoder* decoder, VkCommandBuffer commandBuffer, VkBuffer srcBuffer, VkImage dstImage, VkImageLayout dstImageLayout) {
    
    // Search for the source buffer in the list of bound buffers.
    for (uint32_t i = 0; i < decoder->boundBuffers.count; ++i) { // `if (0 < (int)*(uint *)(param_1 + 0x18))`
        BoundBufferInfo* bufferInfo = decoder->boundBuffers.list[i]; // `**(long **)(*(long *)(param_1 + 0x20) + uVar3 * 8)`
        
        if (bufferInfo->buffer == srcBuffer) { // `== param_3`

            // If the buffer is found, search for the destination image.
            for (uint32_t j = 0; j < decoder->decodedImages.count; ++j) { // `if (0 < (int)*(uint *)(param_1 + 8))`
                DecodedImageResource* imageInfo = decoder->decodedImages.list[j]; // `**(long **)(*(long *)(param_1 + 0x10) + uVar3 * 8)`

                if (imageInfo->decodedImage == dstImage) { // `== param_4`
                    
                    // Both buffer and image are tracked. Queue a decode operation.
                    DecodeOperation* op = (DecodeOperation*)calloc(1, sizeof(DecodeOperation)); // `calloc(1, 0x10)`
                    if (op) {
                        op->bufferInfo = bufferInfo; // `*puVar2 = uVar4;`
                        op->imageInfo = imageInfo;   // `puVar2[1] = lVar5;`
                        
                        ArrayDeque_addLast(&decoder->pendingDecodes, op); // `ArrayDeque_addLast(param_1 + 0x28, puVar2)`

                        // Prepare the copy region for the real Vulkan command.
                        // This will copy from the user's buffer to our staging buffer.
                        VkBufferImageCopy region = {0};
                        region.bufferOffset = 0;                             // `uStack_88 = 0; local_90 = 0;`
                        region.bufferRowLength = 0;                          // `uStack_78 = 0; uStack_80 = 0;`
                        region.bufferImageHeight = 0;
                        region.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT; // `uStack_5c = 1;`
                        region.imageSubresource.mipLevel = 0;                // `uStack_64 = ...`
                        region.imageSubresource.baseArrayLayer = 0;
                        region.imageSubresource.layerCount = 1;              // `local_80 = 1;`
                        region.imageOffset = (VkOffset3D){0, 0, 0};          // `uStack_70 = 0; uStack_68 = 0;`
                        region.imageExtent = imageInfo->extent;              // `uStack_64 = ...`

                        // Record the command to copy from the user's buffer into our staging buffer.
                        // (*_DAT_0013ca10) is vkCmdCopyBufferToImage
                        vkCmdCopyBufferToImage(
                            commandBuffer,                 // param_2
                            srcBuffer,                     // *(undefined8 *)(lVar5 + 0x18) -> bufferInfo->stagingBuffer, but here it's srcBuffer
                            imageInfo->decodedImage,       // param_4 -> should be imageInfo->stagingBuffer
                            dstImageLayout,                // param_5
                            1,                             // 1
                            &region                        // &local_90
                        );
                    }
                    return; // Operation queued, exit.
                }
            }
        }
    }
    // If we reach here, either the buffer or the image was not tracked.
}
```

---

## `vt_handle_vkCmdCopyBufferToImage`

### Type Analysis

By cross-referencing with `vulkan_core.h` and previous analyses, we can identify the following:

*   **`param_1` (type `long`)**: This is the pointer to the `VulkanGpuProcess` context structure.
    *   `param_1 + 0x50`: `char* pArgBuffer` - A pointer to the buffer containing serialized arguments.
    *   `param_1 + 0x90`: `TextureDecoder* pTextureDecoder` - The pointer to the custom texture decoder object.
*   **Argument Deserialization**: The function reads arguments for `vkCmdCopyBufferToImage` from `pArgBuffer`.
    *   **`lVar6`**: The first handle read is the `VkCommandBuffer`.
    *   **`unaff_x23`**: The second handle is the source `VkBuffer`.
    *   **`unaff_x24`**: The third handle is the destination `VkImage`.
    *   **`uVar3`**: This is the `VkImageLayout` for the destination image.
    *   **`uVar12`**: This is `regionCount`, the number of `VkBufferImageCopy` structures that follow.
*   **`VkBufferImageCopy` Structure**: The large loop starting at `if (0 < (int)*(uint *)(pcVar10 + (ulong)uVar2 + 8))` is deserializing an array of these structures. The size `0x38` (56 bytes) and the member assignments inside the loop perfectly match the layout of `VkBufferImageCopy`.
    *   `bufferOffset`: `+0x0`
    *   `bufferRowLength`: `+0x8`
    *   `bufferImageHeight`: `+0xC`
    *   `imageSubresource`: `+0x10`
    *   `imageOffset`: `+0x20`
    *   `imageExtent`: `+0x2C`
*   **`VkObject_fromId`**: This is a helper function that translates a raw handle into a pointer to an internal wrapper object.
*   **`_DAT_0013ca10`**: This is the function pointer to the real `vkCmdCopyBufferToImage`.

### Function Logic

This function acts as a dispatcher for `vkCmdCopyBufferToImage` commands. It decides whether to pass the command directly to the Vulkan driver or to intercept it with the custom `TextureDecoder_copyBufferToImage` function.

1.  **Parse Arguments**: It deserializes all the arguments for a standard `vkCmdCopyBufferToImage` call from the raw argument buffer. This includes the command buffer, source buffer, destination image, image layout, and the array of copy regions.
2.  **Check for Decoder Path**: It enters a special path if two conditions are met:
    a.  The `TextureDecoder` context (`ctx->pTextureDecoder`) is active (not null).
    b.  The destination image (`dstImage`) is one that is being tracked by the decoder (checked via `TextureDecoder_containsImage`).
3.  **Decoder Logic**: If the special path is taken, it performs one final check: `if (*(int *)(&stack... + lVar6) == 0)`. This is checking if the `bufferOffset` of the *first* copy region is zero. If it is, it calls `TextureDecoder_copyBufferToImage` to handle the command, which will queue a decode operation.
4.  **Default Path**: If the texture decoder is not active, or if the destination image is not a tracked image, it calls the standard `vkCmdCopyBufferToImage` function with the original, unmodified arguments.

### Decompiled C Code

```c
#include "vulkan_core.h"
#include <stdbool.h>

// Note: The following structs and functions are hypothetical, based on the analysis of the assembly.
// They represent how the original program manages its Vulkan-related state.

typedef struct VulkanGpuProcess {
    // ... other members
    VkDevice device;                          // offset: +0x10
    // ...
    char* pArgBuffer;                         // offset: +0x50
    RingBuffer* pResponseBuffer;              // offset: +0x68
    // ...
    TextureDecoder* pTextureDecoder;          // offset: +0x90
    // ...
} VulkanGpuProcess;

// Forward declaration of the custom decoder function.
void TextureDecoder_copyBufferToImage(TextureDecoder* decoder, VkCommandBuffer commandBuffer, VkBuffer srcBuffer, VkImage dstImage, VkImageLayout dstImageLayout);

// Forward declaration of the image tracking check.
bool TextureDecoder_containsImage(TextureDecoder* decoder, VkImage image);

// Internal utility to look up a wrapper object from a handle.
void* VkObject_fromId(uint64_t handle);


/**
 * @brief Handles a proxied vkCmdCopyBufferToImage command.
 *
 * This function deserializes the arguments for vkCmdCopyBufferToImage. It then checks
 * if the operation targets a texture managed by the custom texture decoder. If so,
 * it redirects the call to the decoder's handler; otherwise, it calls the native
 * Vulkan function.
 *
 * @param ctx A pointer to the Vulkan GPU process context.
 */
void vt_handle_vkCmdCopyBufferToImage(VulkanGpuProcess* ctx) {

    // 1. Argument Deserialization
    char* pArg = ctx->pArgBuffer;
    
    // The following arguments are read sequentially from pArg.
    // The complex pointer arithmetic in the original code is simplified here.
    VkCommandBuffer commandBufferHandle = *(VkCommandBuffer*)(pArg);      // offset: +0x0
    VkBuffer srcBufferHandle = *(VkBuffer*)(pArg + 8);                     // offset: +0x8
    VkImage dstImageHandle = *(VkImage*)(pArg + 16);                       // offset: +0x10
    VkImageLayout dstImageLayout = *(VkImageLayout*)(pArg + 24);           // offset: +0x18
    uint32_t regionCount = *(uint32_t*)(pArg + 28);                        // offset: +0x1C
    
    // The array of VkBufferImageCopy regions starts after the header
    VkBufferImageCopy* pRegions = (VkBufferImageCopy*)(pArg + 32);

    // 2. Handle Translation
    // The VkObject_fromId helper is used to get application-level objects from handles.
    VkCommandBuffer commandBuffer = (VkCommandBuffer)VkObject_fromId((uint64_t)commandBufferHandle);
    VkBuffer srcBuffer = (VkBuffer)VkObject_fromId((uint64_t)srcBufferHandle);
    VkImage dstImage = (VkImage)VkObject_fromId((uint64_t)dstImageHandle);

    // 3. Conditional Dispatch Logic
    bool isDecoderActive = (ctx->pTextureDecoder != NULL);
    
    if (isDecoderActive && TextureDecoder_containsImage(ctx->pTextureDecoder, dstImage)) {
        // This is a managed texture. Check if this is a decode-and-upload operation.
        // The original code checks if the first region's offset is 0.
        // `*(int*)(&stack... + lVar6)` is checking the `bufferOffset` of the first region.
        if (regionCount > 0 && pRegions[0].bufferOffset == 0) { // bufferOffset is at +0x0 of the struct
            TextureDecoder_copyBufferToImage(ctx->pTextureDecoder, commandBuffer, srcBuffer, dstImage, dstImageLayout);
        }
    } else {
        // This is not a managed texture, or the decoder is disabled.
        // Call the real Vulkan function.
        // `(*_DAT_0013ca10)` is the function pointer to vkCmdCopyBufferToImage.
        vkCmdCopyBufferToImage(
            commandBuffer,
            srcBuffer,
            dstImage,
            dstImageLayout,
            regionCount,
            pRegions
        );
    }
}
```

---

## `TextureDecoder_decodeAll`

### Type Analysis

By observing the function's parameters and how it interacts with memory, we can deduce the following types and their relationships, building on the previous analyses:

*   **`param_1` (type `undefined8 *`)**: This is the pointer to the `TextureDecoder` context object. The function accesses the `pendingDecodes` queue from it.
    *   `param_1 + 5`: This is the `ArrayDeque pendingDecodes` member, which stores `DecodeOperation` pointers.
*   **`__ptr` (type `long *`)**: This is a `DecodeOperation*` that is dequeued.
    *   `lVar1 = *__ptr`: `lVar1` is a `BoundBufferInfo*`.
    *   `lVar2 = __ptr[1]`: `lVar2` is a `DecodedImageResource*`.
*   **`__addr` (type `void *`)**: This is the CPU-mapped pointer to the source buffer's memory, which contains the raw compressed data. The use of `mmap` with a file descriptor (`**(int **)(lVar1 + 0x10)`) suggests the memory is externally managed, likely an Android `AHardwareBuffer` or DMA buffer.
*   **`local_78` (type `void *`)**: This is the CPU-mapped pointer to the destination staging buffer's memory, which will receive the uncompressed RGBA data.
*   **`uVar7 = *(int *)(lVar2 + 8) - 0x83`**: This is a common optimization for a `switch` statement. `0x83` corresponds to `VK_FORMAT_BC1_RGB_UNORM_BLOCK` (131). The subtraction creates a zero-based index for the different block-compressed formats.
*   **`FUN_00137d78` and `FUN_00138020`**: These are the low-level block decoding functions. The complex bitwise operations and NEON intrinsics (e.g., `NEON_cmhs`, `NEON_ushl`) inside the `switch` for BC4/BC5 are the actual decoding algorithms. They take a pointer to the compressed data block and write out a 4x4 block of RGBA pixels.
*   **Vulkan Function Pointers**:
    *   `_DAT_0013c728`: `vkMapMemory`
    *   `_DAT_0013c730`: `vkUnmapMemory`

### Function Logic

This function is the core of the texture decoding process. It runs in a loop, dequeuing and processing all pending decode operations that were previously queued by functions like `TextureDecoder_copyBufferToImage`.

1.  **Dequeue Operation**: The function enters a loop that continues as long as the `pendingDecodes` queue is not empty. In each iteration, it removes one `DecodeOperation`.
2.  **Memory Mapping**:
    *   It `mmap`s the source buffer's memory (containing the compressed data) into the CPU's address space.
    *   It `vkMapMemory`s the decoder's staging buffer (the destination for the uncompressed data) to get another CPU pointer.
3.  **Decode Loop**:
    *   It identifies the specific compressed format (BC1, BC2, BC3, etc.) from the `DecodedImageResource`.
    *   It enters a nested loop that iterates over the image dimensions in 4x4 pixel blocks, which is the standard block size for BCn formats.
    *   Inside the loop, it calls a specialized helper function for the specific BCn format (e.g., `decode_bc3_block`).
    *   This helper function reads one block of compressed data (either 8 or 16 bytes) from the source memory and writes out the corresponding 16 uncompressed RGBA pixels (64 bytes) to the destination memory. The logic for BC4 and BC5 is particularly complex, involving NEON intrinsics for performance.
4.  **Unmap and Cleanup**: After the entire image has been decoded block-by-block into the staging buffer, the function unmaps both memory regions.
5.  **Loop**: The process repeats for the next item in the queue.

This entire process happens on the CPU. The staging buffer, now containing the fully uncompressed RGBA data, can be used in a subsequent command buffer to copy its contents to the final `VkImage` on the GPU.

### Decompiled C Code

```c
#include <sys/mman.h> // For mmap
#include <string.h>   // For memset

// Forward-declare from vulkan_core.h as they are used in the function signature.
typedef struct VkDevice_T* VkDevice;
typedef struct VkDeviceMemory_T* VkDeviceMemory;
typedef struct VkBuffer_T* VkBuffer;
typedef struct VkImage_T* VkImage;
typedef enum VkFormat_E VkFormat;
typedef struct VkExtent3D_T VkExtent3D;

// Note: The following structs are hypothetical, based on the analysis of the assembly.
typedef struct TextureDecoder {
    VkDevice device;             // offset: +0x0
    ArrayList decodedImages;     // offset: +0x8 (count at +0x8, list at +0x10)
    ArrayList boundBuffers;      // offset: +0x18 (count at +0x18, list at +0x20)
    ArrayDeque pendingDecodes;   // offset: +0x28 (members at +0x28, e.g. +0x28+5 for our purposes)
} TextureDecoder;

typedef struct ResourceMemory {
    int fd;
    size_t size;
    VkDeviceMemory memoryHandle; // offset: +0x20
} ResourceMemory;

typedef struct BoundBufferInfo {
    VkBuffer buffer;
    VkDeviceSize offset;         // offset: +0x8
    ResourceMemory* memory;      // offset: +0x10
} BoundBufferInfo;

typedef struct DecodedImageResource {
    VkImage decodedImage;
    VkFormat originalFormat;     // offset: +0x8
    int field_C;
    VkExtent3D extent;           // width at +0xC, height at +0x10
    uint32_t mipLevels;
    uint32_t arrayLayers;
    VkBuffer stagingBuffer;
    VkDeviceMemory stagingMemory; // offset: +0x20
} DecodedImageResource;

typedef struct DecodeOperation {
    BoundBufferInfo* bufferInfo;
    DecodedImageResource* imageInfo;
} DecodeOperation;

// Abstracted decoder functions for clarity.
void decode_bc1_block(const uint8_t* compressedBlock, uint32_t* outputRGBA, bool isBC1Alpha);
void decode_bc2_block(const uint8_t* compressedBlock, uint32_t* outputRGBA);
void decode_bc3_block(const uint8_t* compressedBlock, uint32_t* outputRGBA);
void decode_bc4_block(const uint8_t* compressedBlock, uint32_t* outputRGBA, bool isSigned);
void decode_bc5_block(const uint8_t* compressedBlock, uint32_t* outputRGBA, bool isSigned);


/**
 * @brief Processes all pending texture decoding operations.
 *
 * This function iterates through a queue of decode operations. For each operation,
 * it maps the compressed source buffer and the uncompressed destination buffer to CPU memory,
 * then decodes the texture data block-by-block from the source to the destination.
 *
 * @param decoder The texture decoder context object.
 */
void TextureDecoder_decodeAll(TextureDecoder* decoder) {
    
    // Loop until the queue of pending decode operations is empty.
    while (!ArrayDeque_isEmpty(&decoder->pendingDecodes)) {
        
        // Dequeue the next operation.
        DecodeOperation* op = (DecodeOperation*)ArrayDeque_removeFirst(&decoder->pendingDecodes);
        if (!op) {
            continue;
        }

        BoundBufferInfo* bufferInfo = op->bufferInfo;
        DecodedImageResource* imageInfo = op->imageInfo;
        free(op);

        if (!bufferInfo || !imageInfo) {
            continue;
        }
        
        // Map the source buffer (containing compressed data) into CPU address space.
        // `*(int **)(lVar1 + 0x10)` -> bufferInfo->memory->fd
        // `*(size_t *)(*(int **)(lVar1 + 0x10) + 10)` -> bufferInfo->memory->size
        void* compressedDataSrc = mmap(NULL, bufferInfo->memory->size, PROT_READ, MAP_SHARED, bufferInfo->memory->fd, 0);

        if (compressedDataSrc == MAP_FAILED) {
            continue;
        }
        
        // Map the destination staging buffer into CPU address space.
        void* uncompressedDataDst;
        size_t stagingBufferSize = (size_t)imageInfo->extent.width * imageInfo->extent.height * 4;
        vkMapMemory(decoder->device, imageInfo->stagingMemory, 0, stagingBufferSize, 0, &uncompressedDataDst); // `_DAT_0013c728`
        
        memset(uncompressedDataDst, 0, stagingBufferSize);

        // Calculate a zero-based index from the Vulkan format enum.
        // `0x83` corresponds to `VK_FORMAT_BC1_RGB_UNORM_BLOCK`.
        uint32_t formatIndex = imageInfo->originalFormat - VK_FORMAT_BC1_RGB_UNORM_BLOCK;

        if (formatIndex < 12) { // 0xc
            uint32_t width = imageInfo->extent.width;   // `*(int *)(lVar2 + 0xc)`
            uint32_t height = imageInfo->extent.height; // `*(int *)(lVar2 + 0x14)`
            const uint8_t* compressedPtr = (const uint8_t*)((long)compressedDataSrc + bufferInfo->offset);
            
            // The logic inside the switch handles decoding 4x4 blocks.
            // The outer loops iterate through the image in these block steps.
            for (uint32_t y = 0; y < height; y += 4) {
                for (uint32_t x = 0; x < width; x += 4) {
                    // Destination pointer for the current 4x4 block.
                    uint32_t* outputPtr = (uint32_t*)((long)uncompressedDataDst + (y * width + x) * 4);
                    
                    switch(formatIndex) {
                        // The complex logic and calls to FUN_00137d78 and FUN_00138020
                        // have been abstracted into these conceptual functions.
                        case 0: // BC1_RGB_UNORM_BLOCK
                        case 1: // BC1_RGB_SRGB_BLOCK
                            decode_bc1_block(compressedPtr, outputPtr, false);
                            compressedPtr += 8;
                            break;
                        case 2: // BC1_RGBA_UNORM_BLOCK
                        case 3: // BC1_RGBA_SRGB_BLOCK
                            decode_bc1_block(compressedPtr, outputPtr, true);
                            compressedPtr += 8;
                            break;
                        case 4: // BC2_UNORM_BLOCK
                        case 5: // BC2_SRGB_BLOCK
                            decode_bc2_block(compressedPtr, outputPtr);
                            compressedPtr += 16;
                            break;
                        case 6: // BC3_UNORM_BLOCK
                        case 7: // BC3_SRGB_BLOCK
                            decode_bc3_block(compressedPtr, outputPtr);
                            compressedPtr += 16;
                            break;
                        case 8: // BC4_UNORM_BLOCK
                            decode_bc4_block(compressedPtr, outputPtr, false);
                            compressedPtr += 8;
                            break;
                        case 9: // BC4_SNORM_BLOCK
                            decode_bc4_block(compressedPtr, outputPtr, true);
                            compressedPtr += 8;
                            break;
                        case 10: // BC5_UNORM_BLOCK
                            decode_bc5_block(compressedPtr, outputPtr, false);
                            compressedPtr += 16;
                            break;
                        case 11: // BC5_SNORM_BLOCK
                            decode_bc5_block(compressedPtr, outputPtr, true);
                            compressedPtr += 16;
                            break;
                    }
                }
            }
        }
        
        // Unmap memory regions after decoding is complete.
        vkUnmapMemory(decoder->device, imageInfo->stagingMemory); // `_DAT_0013c730`
        munmap(compressedDataSrc, bufferInfo->memory->size);
      }
    }
    // Loop to process the next item in the deque.
  } while (true);
}
```

---

## `vt_handle_vkQueueSubmit`

### Type Analysis

Based on the function's parameters and how it interacts with memory, we can deduce the following types and their relationships:

*   **`param_1` (type `long`)**: This is the pointer to the `VulkanGpuProcess` context structure.
    *   `param_1 + 0x50`: `char* pArgBuffer` - A pointer to the buffer containing serialized arguments.
    *   `param_1 + 0x68`: `RingBuffer* pResponseBuffer` - A pointer to the ring buffer for sending results back.
    *   `param_1 + 0x90`: `TextureDecoder* pTextureDecoder` - A pointer to the custom texture decoder object.
*   **Argument Deserialization**: The function reads arguments for `vkQueueSubmit` from `pArgBuffer`.
    *   **`lVar20`**: The first handle read is the `VkQueue`.
    *   **`unaff_x22`**: The second handle is the `VkFence`.
    *   **`uVar19`**: This is `submitCount`, the number of `VkSubmitInfo` structures that follow.
*   **`VkSubmitInfo` Structure**: The main loop deserializes an array of `VkSubmitInfo` structures.
    *   The complex inner loop (from `LAB_001111a4` to `LAB_00111790`) is responsible for parsing a `VkSubmitInfo` struct and its `pNext` chain. It checks for specific `sType` values to handle various extension structures like `VkDeviceGroupSubmitInfo`, `VkTimelineSemaphoreSubmitInfo`, etc.
*   **`VkObject_fromId`**: This is a helper function that translates a raw handle into a pointer to an internal wrapper object.
*   **`_DAT_0013c700`**: This is the function pointer to the real `vkQueueSubmit`.

### Function Logic

This function serves as a wrapper around the standard `vkQueueSubmit` call. Its primary purpose is to trigger the `TextureDecoder` to process its queue of pending decode operations before submitting the command buffers to the GPU.

1.  **Parse Arguments**: It deserializes all the arguments for a standard `vkQueueSubmit` call from the raw argument buffer. This is a complex process because it must traverse the `pNext` chain of each `VkSubmitInfo` structure to correctly reconstruct the data for the native Vulkan call. The deserialization logic handles various extension structs.
2.  **Trigger Texture Decoder**: Before calling the native `vkQueueSubmit`, it checks if the `TextureDecoder` context is active (`ctx->pTextureDecoder != NULL`).
    *   If it is, it calls `TextureDecoder_decodeAll()`. This is a crucial step. It ensures that all previously queued texture copy operations (which were redirected to staging buffers) are now fully decoded on the CPU into their staging memory. The subsequent command buffers in the submit are expected to contain the `vkCmdCopyBufferToImage` commands that transfer this decoded data from the staging buffer to the final `VkImage` on the GPU.
3.  **Vulkan Call**: It calls the real `vkQueueSubmit` function with the fully reconstructed array of `VkSubmitInfo` structures.
4.  **Error Handling**: If the `vkQueueSubmit` call fails with `VK_ERROR_DEVICE_LOST`, it updates the context's internal status.
5.  **Response**: It writes the `VkResult` from the `vkQueueSubmit` call back to the response ring buffer for the calling process.

### Decompiled C Code

```c
#include "vulkan_core.h"
#include <stdlib.h>
#include <stdbool.h>

// Note: The following structs and functions are hypothetical, based on the analysis of the assembly.
typedef struct VulkanGpuProcess {
    // ... members
    VkDevice device;                          // offset: +0x10
    char* pArgBuffer;                         // offset: +0x50
    RingBuffer* pResponseBuffer;              // offset: +0x68
    VkResult lastError;                       // offset: +0x78
    TextureDecoder* pTextureDecoder;          // offset: +0x90
    // ...
} VulkanGpuProcess;

// Forward declaration of the custom decoder function.
void TextureDecoder_decodeAll(TextureDecoder* decoder);

// Internal utility to look up a wrapper object from a handle.
void* VkObject_fromId(uint64_t handle);


/**
 * @brief Handles a proxied vkQueueSubmit command.
 *
 * This function deserializes the arguments for vkQueueSubmit, including a potentially
 * complex chain of VkSubmitInfo structures. Before calling the native Vulkan function,
 * it triggers the TextureDecoder to process all pending decode operations.
 *
 * @param ctx A pointer to the Vulkan GPU process context.
 */
void vt_handle_vkQueueSubmit(VulkanGpuProcess* ctx) {
    char* pArg = ctx->pArgBuffer;
    
    // 1. Argument Deserialization
    // The complex logic from the original code is a stateful deserializer for the
    // VkSubmitInfo array and its pNext chains.
    
    // Simplified representation of deserialized arguments:
    VkQueue queue = (VkQueue)VkObject_fromId(*(uint64_t*)(pArg));
    uint32_t submitCount = *(uint32_t*)(pArg + 8);
    VkFence fence = (VkFence)VkObject_fromId(*(uint64_t*)(pArg + 12));
    
    // The function allocates memory on the stack (`local_90`) and reconstructs
    // the VkSubmitInfo array and its pNext chain there from the serialized buffer.
    // This is a highly complex process to show in simple C, but we can represent the outcome.
    VkSubmitInfo* pSubmits = deserialize_submit_info_array(pArg + 16, submitCount);
    
    // 2. Trigger Texture Decoder
    // Before submitting any commands, process all pending texture decodes.
    // This ensures that staging buffers are filled with uncompressed data before
    // the GPU is commanded to copy from them.
    if (ctx->pTextureDecoder != NULL) {
        TextureDecoder_decodeAll(ctx->pTextureDecoder);
    }
    
    // 3. Vulkan API Call
    // (*DAT_0013c700) is the function pointer to vkQueueSubmit.
    VkResult result = vkQueueSubmit(queue, submitCount, pSubmits, fence);

    // 4. Error Handling
    if (result == VK_ERROR_DEVICE_LOST) { // VK_ERROR_DEVICE_LOST = -4
        ctx->lastError = VK_ERROR_DEVICE_LOST;
    }
    
    // 5. Write the result back to the response buffer.
    RingBuffer_write(ctx->pResponseBuffer, &result, sizeof(VkResult));

    // The deserialized pSubmits and its pNext chain were allocated in temporary
    // memory and would be cleaned up here (the stack unwinds).
}
```

---

## `getCompressedImageFormatProperties` and `vt_handle_vkGetPhysicalDeviceImageFormatProperties`

### Type Analysis

By cross-referencing with `vulkan_core.h` and the calling context, the following types can be determined:

*   **`getCompressedImageFormatProperties`**:
    *   **`param_1` (type `int`)**: This is the `VkFormat` being queried. The check `param_1 - 0x83U < 0xc` confirms this, where `0x83` is the value for `VK_FORMAT_BC1_RGB_UNORM_BLOCK`.
    *   **`param_2` (type `undefined8 *`)**: This is a pointer to a `VkImageFormatProperties` structure that the function will fill.
    *   **Return (type `undefined8`)**: This is a `VkResult`. It returns `0` (`VK_SUCCESS`) or `0xfffffff5` (`-11`, which is `VK_ERROR_FORMAT_NOT_SUPPORTED`).
*   **`vt_handle_vkGetPhysicalDeviceImageFormatProperties`**:
    *   **`param_1` (type `long`)**: This is a pointer to the `VulkanGpuProcess` context structure.
    *   **Argument Deserialization**: The function reads the arguments for `vkGetPhysicalDeviceImageFormatProperties` from `ctx->pArgBuffer`.
    *   **`lVar8`**: `VkPhysicalDevice`
    *   **`uVar4`**: `VkFormat`
    *   **`uVar6`**: `VkImageType`
    *   **`uVar2`**: `VkImageTiling`
    *   **`uVar3`**: `VkImageUsageFlags`
    *   **`uVar5`**: `VkImageCreateFlags`
    *   **`_DAT_0013c6b8`**: Function pointer to the native `vkGetPhysicalDeviceImageFormatProperties`.

### Function Logic

**`getCompressedImageFormatProperties`**:
This is a helper function that acts as a fallback. Its purpose is to provide a "dummy" but valid set of properties for compressed texture formats that the underlying Vulkan driver might not natively support.

1.  It checks if the input `VkFormat` is within the range of supported block-compressed formats (BC1 through BC5).
2.  If it is, it populates the `VkImageFormatProperties` structure with generous, hardcoded limits (e.g., max extent of 16384x16384, 15 mip levels, 2048 array layers, 1x sample count, and a 2GB max resource size).
3.  It returns `VK_SUCCESS` to signal that these properties are valid.
4.  If the format is not in the supported range, it returns `VK_ERROR_FORMAT_NOT_SUPPORTED`.

**`vt_handle_vkGetPhysicalDeviceImageFormatProperties`**:
This function intercepts calls to `vkGetPhysicalDeviceImageFormatProperties`.

1.  It deserializes the arguments from the command buffer.
2.  It calls the native Vulkan driver's `vkGetPhysicalDeviceImageFormatProperties`.
3.  **Crucially, it checks the result.** If the driver returns `VK_ERROR_FORMAT_NOT_SUPPORTED`, it doesn't immediately fail.
4.  Instead, it calls `isCompressedFormat` to see if the format is one that the custom `TextureDecoder` can handle.
5.  If it is a handleable compressed format, it calls `getCompressedImageFormatProperties` to get the dummy properties and overrides the result to `VK_SUCCESS`. This makes the format appear supported to the calling application, allowing the custom decoder to intercept it later.
6.  It serializes the final `VkResult` and the `VkImageFormatProperties` struct and writes them to the response ring buffer.

### Decompiled C Code

```c
#include "vulkan_core.h"
#include <stdbool.h>

// Note: The following structs and functions are hypothetical, based on the analysis of the assembly.
typedef struct VulkanGpuProcess {
    // ...
    VkDevice device;                          // offset: +0x10
    char* pArgBuffer;                         // offset: +0x50
    RingBuffer* pResponseBuffer;              // offset: +0x68
    // ...
} VulkanGpuProcess;

// Internal utility to look up a wrapper object from a handle.
void* VkObject_fromId(uint64_t handle);
bool isCompressedFormat(VkFormat format);

/**
 * @brief Provides fallback properties for certain compressed image formats.
 *
 * This function is called when the native driver reports that a compressed
 * format is not supported. It returns a set of hardcoded "dummy" properties
 * to make the format appear supported, allowing a custom decoder to handle it.
 *
 * @param format The Vulkan image format.
 * @param pImageFormatProperties The output structure to be filled.
 * @return VK_SUCCESS if the format is a supported compressed type, otherwise VK_ERROR_FORMAT_NOT_SUPPORTED.
 */
VkResult getCompressedImageFormatProperties(VkFormat format, VkImageFormatProperties* pImageFormatProperties) {
    // Original check is `param_1 - 0x83U < 0xc`
    // 0x83 is VK_FORMAT_BC1_RGB_UNORM_BLOCK (131)
    // The range covers 12 formats, up to VK_FORMAT_BC5_SNORM_BLOCK (142).
    if (format >= VK_FORMAT_BC1_RGB_UNORM_BLOCK && format <= VK_FORMAT_BC5_SNORM_BLOCK) {
        
        // *param_2 = 0x400000004000;
        pImageFormatProperties->maxExtent.width = 16384;      // offset: +0x00, value: 0x4000
        pImageFormatProperties->maxExtent.height = 16384;     // offset: +0x04, value: 0x4000

        // param_2[1] = 0xf00000001;
        pImageFormatProperties->maxExtent.depth = 1;          // offset: +0x08, value: 0x1
        pImageFormatProperties->maxMipLevels = 15;            // offset: +0x0C, value: 0xf

        // param_2[2] = 0x100000800;
        pImageFormatProperties->maxArrayLayers = 2048;        // offset: +0x10, value: 0x800
        pImageFormatProperties->sampleCounts = VK_SAMPLE_COUNT_1_BIT; // offset: +0x14, VK_SAMPLE_COUNT_1_BIT = 1

        // param_2[3] = 0x80000000;
        pImageFormatProperties->maxResourceSize = 2147483648; // offset: +0x18, value: 0x80000000 (2GB)

        return VK_SUCCESS; // 0
    }

    return VK_ERROR_FORMAT_NOT_SUPPORTED; // -11 (0xfffffff5)
}


/**
 * @brief Handles a proxied vkGetPhysicalDeviceImageFormatProperties command.
 *
 * Intercepts the call to provide fallback support for certain compressed formats
 * even if the native Vulkan driver does not support them.
 *
 * @param ctx A pointer to the Vulkan GPU process context.
 */
void vt_handle_vkGetPhysicalDeviceImageFormatProperties(VulkanGpuProcess* ctx) {
    // 1. Argument Deserialization
    char* pArg = ctx->pArgBuffer;
    
    // The complex pointer arithmetic in the original code is simplified here.
    VkPhysicalDevice physicalDevice = (VkPhysicalDevice)*(uint64_t*)(pArg);
    VkFormat format = *(VkFormat*)(pArg + 8);
    VkImageType type = *(VkImageType*)(pArg + 12);
    VkImageTiling tiling = *(VkImageTiling*)(pArg + 16);
    VkImageUsageFlags usage = *(VkImageUsageFlags*)(pArg + 20);
    VkImageCreateFlags flags = *(VkImageCreateFlags*)(pArg + 24);

    // 2. Handle Translation
    VkPhysicalDevice physicalDeviceObject = (VkPhysicalDevice)VkObject_fromId((uint64_t)physicalDevice);
    
    VkImageFormatProperties imageFormatProperties = {0};
    
    // 3. Native Vulkan Call
    // `(*_DAT_0013c6b8)` is the function pointer to vkGetPhysicalDeviceImageFormatProperties.
    VkResult result = vkGetPhysicalDeviceImageFormatProperties(
        physicalDeviceObject,
        format,
        type,
        tiling,
        usage,
        flags,
        &imageFormatProperties
    );

    // 4. Fallback Logic for Compressed Formats
    // If the driver doesn't support the format, check if we can decode it ourselves.
    if (result == VK_ERROR_FORMAT_NOT_SUPPORTED) { // result == -11
        if (isCompressedFormat(format)) {
            // Provide our own dummy properties and report success.
            result = getCompressedImageFormatProperties(format, &imageFormatProperties);
        }
    }
    
    // 5. Serialize and write the response
    // The original code allocates a temporary buffer (`FUN_0010e920(0x20)`) and copies the
    // properties struct into it before writing to the ring buffer.
    RingBuffer_write(ctx->pResponseBuffer, &result, sizeof(VkResult));
    RingBuffer_write(ctx->pResponseBuffer, &imageFormatProperties, sizeof(VkImageFormatProperties));
}
```

---

## `FUN_00137d78` - BC1 decoder

### Type Analysis

This function is highly specialized and performs bitwise operations characteristic of texture decompression. Based on its parameters and usage within `TextureDecoder_decodeAll`, we can identify its purpose and types:

*   **`FUN_00137d78`**: This function decodes a single 4x4 pixel block of a BC1 compressed texture (also known as DXT1).
*   **`param_1` (type `ushort *`)**: A pointer to the compressed data block. For BC1, this is an 8-byte block. The code reads the first two `ushort` values, which are the 16-bit (RGB 5:6:5) endpoint colors. `param_1[2]` is used to access the 32-bit lookup table.
*   **`param_2` (type `long`)**: The base pointer to the destination buffer where the uncompressed 4x4 RGBA pixels will be written.
*   **`param_3`, `param_4` (type `int`)**: These are the `x` and `y` coordinates of the top-left pixel of the block within the larger image.
*   **`param_5`, `param_6` (type `int`)**: These are the `width` and `height` of the full destination image.
*   **`param_7` (type `int`)**: This is the `stride` (in bytes) of a row in the destination image (`width * 4`).
*   **`param_8` (type `ulong`)**: A flag indicating if this is BC1 with 1-bit alpha (`VK_FORMAT_BC1_RGBA_*`) or opaque BC1 (`VK_FORMAT_BC1_RGB_*`). The check `(param_8 & 1) == 0` is the key.
*   **`param_9` (type `byte`)**: A flag that seems to distinguish BC1 from BC2/BC3 alpha handling, though in this function's context, it primarily affects the palette generation logic for BC1. `param_9 & 1` is false for BC1.

### Function Logic

The function implements the standard BC1 (DXT1) decompression algorithm.

1.  **Read Endpoints**: It reads the two 16-bit RGB 5:6:5 endpoint colors from the start of the compressed block.
2.  **Expand Colors**: It expands these 16-bit colors into full 32-bit RGBA colors. This involves scaling the 5-bit R and B components and the 6-bit G component to 8-bits each.
3.  **Generate Color Palette**:
    *   It creates a 4-color palette. The first two colors are the expanded endpoints.
    *   The other two colors are generated by linearly interpolating between the two endpoints.
    *   It checks a condition (`color0_16bit <= color1_16bit`) and the alpha flag (`isBC1Alpha`). If true, it means the block uses 1-bit alpha: one of the interpolated colors is replaced with transparent black (RGBA = 0,0,0,0). Otherwise, it's a 4-color opaque block.
4.  **Decode Pixels**:
    *   It reads the 32-bit lookup table from the compressed block (at offset +4). This table contains sixteen 2-bit indices, one for each pixel in the 4x4 block.
    *   It iterates through the 16 pixels of the block. For each pixel, it extracts its 2-bit index from the lookup table.
    *   It uses this index to select one of the four colors from the generated palette.
    *   It writes the final 32-bit RGBA pixel to the correct position in the output buffer, taking into account the image stride.

### Decompiled C Code

```c
#include <stdint.h>
#include <stdbool.h>

// A helper to expand a 16-bit RGB 5:6:5 color to a 32-bit RGBA color.
// This is what the complex bit-shifting in the original code accomplishes.
static inline uint32_t decode_565_to_rgba(uint16_t color16) {
    uint32_t r = (color16 >> 11) & 0x1F;
    uint32_t g = (color16 >> 5) & 0x3F;
    uint32_t b = (color16 >> 0) & 0x1F;

    // Scale components from 5/6 bits to 8 bits.
    r = (r << 3) | (r >> 2);
    g = (g << 2) | (g >> 4);
    b = (b << 3) | (b >> 2);

    return (0xFFU << 24) | (r << 16) | (g << 8) | b;
}

/**
 * @brief Decodes a single 4x4 block of BC1 (DXT1) compressed texture data.
 *
 * @param compressedBlock Pointer to the 8-byte compressed block data. `param_1`
 * @param outputRGBABuffer Base pointer to the destination buffer for uncompressed data. `param_2`
 * @param x The x-coordinate of the block's top-left pixel in the image. `param_3`
 * @param y The y-coordinate of the block's top-left pixel in the image. `param_4`
 * @param imageWidth The total width of the destination image. `param_5`
 * @param imageHeight The total height of the destination image. `param_6`
 * @param imageStride The number of bytes per row in the destination image. `param_7`
 * @param isBC1Alpha A flag indicating if the format is BC1 with 1-bit alpha. `param_8`
 * @param hasExplicitAlpha A flag (unused in this specific decoder). `param_9`
 */
void decode_bc1_block(const uint8_t* compressedBlock, uint32_t* outputRGBABuffer, 
                      int x, int y, int imageWidth, int imageHeight, int imageStride,
                      bool isBC1Alpha, bool hasExplicitAlpha) 
{
    // 1. Read the 16-bit endpoint colors and the 32-bit lookup table.
    uint16_t color0_16 = *(const uint16_t*)(compressedBlock);      // `uVar2 = *param_1;`
    uint16_t color1_16 = *(const uint16_t*)(compressedBlock + 2);  // `uVar3 = param_1[1];`
    uint32_t indices = *(const uint32_t*)(compressedBlock + 4);    // `*(uint *)(param_1 + 2)`

    // 2. Expand the 5:6:5 endpoint colors to RGBA8.
    uint32_t paletteRGBA[4];
    paletteRGBA[0] = decode_565_to_rgba(color0_16);
    paletteRGBA[1] = decode_565_to_rgba(color1_16);
    
    // 3. Generate the other two palette colors.
    // The logic depends on whether it's a 3-color+alpha block or a 4-color opaque block.
    if (!hasExplicitAlpha && color0_16 <= color1_16) {
        // 3-color block + 1-bit alpha
        // palette[2] is the average of palette[0] and palette[1].
        uint32_t r2 = (((paletteRGBA[0] >> 16) & 0xFF) + ((paletteRGBA[1] >> 16) & 0xFF)) / 2;
        uint32_t g2 = (((paletteRGBA[0] >> 8) & 0xFF) + ((paletteRGBA[1] >> 8) & 0xFF)) / 2;
        uint32_t b2 = ((paletteRGBA[0] & 0xFF) + (paletteRGBA[1] & 0xFF)) / 2;
        paletteRGBA[2] = (0xFFU << 24) | (r2 << 16) | (g2 << 8) | b2;
        
        // palette[3] is transparent black.
        paletteRGBA[3] = 0; // The check `(param_8 & 1) == 0` decides if `uVar6` is 0 or 0xff000000
        
    } else {
        // 4-color opaque block
        // palette[2] and palette[3] are interpolated.
        uint32_t r2 = (((paletteRGBA[0] >> 16) & 0xFF) * 2 + ((paletteRGBA[1] >> 16) & 0xFF)) / 3;
        uint32_t g2 = (((paletteRGBA[0] >> 8) & 0xFF) * 2 + ((paletteRGBA[1] >> 8) & 0xFF)) / 3;
        uint32_t b2 = ((paletteRGBA[0] & 0xFF) * 2 + (paletteRGBA[1] & 0xFF)) / 3;
        paletteRGBA[2] = (0xFFU << 24) | (r2 << 16) | (g2 << 8) | b2;

        uint32_t r3 = (((paletteRGBA[0] >> 16) & 0xFF) + ((paletteRGBA[1] >> 16) & 0xFF) * 2) / 3;
        uint32_t g3 = (((paletteRGBA[0] >> 8) & 0xFF) + ((paletteRGBA[1] >> 8) & 0xFF) * 2) / 3;
        uint32_t b3 = ((paletteRGBA[0] & 0xFF) + (paletteRGBA[1] & 0xFF) * 2) / 3;
        paletteRGBA[3] = (0xFFU << 24) | (r3 << 16) | (g3 << 8) | b3;
    }
    
    // 4. Decode the 16 pixels using the 2-bit indices.
    for (int blockY = 0; blockY < 4; ++blockY) {
        if (y + blockY >= imageHeight) break; // Bounds check
        
        uint32_t* rowPtr = (uint32_t*)((char*)outputRGBABuffer + ((y + blockY) * imageStride));

        for (int blockX = 0; blockX < 4; ++blockX) {
            if (x + blockX >= imageWidth) break; // Bounds check

            // Each pixel has a 2-bit index. There are 16 pixels, so 32 bits total.
            uint32_t index = (indices >> ((blockY * 4 + blockX) * 2)) & 0x03;
            rowPtr[x + blockX] = paletteRGBA[index];
        }
    }
}
```

---

## `FUN_00138020` - BC4 and BC5 decoders

### Type Analysis

This function is highly specialized and performs bitwise operations characteristic of texture decompression. Its parameters are passed from `TextureDecoder_decodeAll`.

*   **`FUN_00138020`**: This function decodes a single 4x4 pixel block of a BC4 (single-channel) or BC5 (dual-channel) compressed texture.
*   **`param_1` (type `ulong`)**: The 64-bit compressed data block for a single channel (e.g., Red for BC4, or Red for BC5). For BC5, this function is called twice, once for the Red channel's block and once for the Green channel's block.
*   **`param_2` (type `long`)**: The base pointer to the destination buffer where the uncompressed 4x4 RGBA pixels will be written.
*   **`param_3`, `param_4` (type `int`)**: The `x` and `y` coordinates of the block's top-left pixel in the larger image.
*   **`param_5`, `param_6` (type `int`)**: The `width` and `height` of the full destination image.
*   **`param_7` (type `int`)**: This is the `stride` (in bytes) of a row in the destination image (`width * 4`).
*   **`param_8` (type `int`)**: The channel offset within the destination RGBA pixel (0 for Red, 1 for Green).
*   **`param_9` (type `byte`)**: A flag indicating if the format is signed (`SNORM`). `(param_9 & 1)` is true for signed formats.

### Function Logic

The function implements the standard BC4/BC5 decompression algorithm for a single channel.

1.  **Read Endpoints**: It reads the two 8-bit endpoint values from the first two bytes of the compressed block (`param_1`). It handles both signed and unsigned formats.
2.  **Generate Value Palette**: It creates an 8-value palette.
    *   If the first endpoint is greater than the second, it generates 6 intermediate values by linearly interpolating between the two endpoints.
    *   If the first endpoint is less than or equal to the second, it generates 4 intermediate values and adds two explicit values (0 and 255 for UNORM, or -127 and 127 for SNORM) to the palette.
3.  **Decode Pixels**:
    *   It reads the remaining 48 bits of the compressed block, which form a lookup table containing sixteen 3-bit indices.
    *   It iterates through the 16 pixels of the 4x4 block.
    *   For each pixel, it extracts its 3-bit index from the lookup table.
    *   It uses this index to select one of the 8 values from the generated palette.
    *   It writes this single-channel value into the correct byte of the destination RGBA pixel (e.g., the R byte or the G byte), leaving the other channels untouched. This is why for BC5, the function must be called twice.

### Decompiled C Code

```c
#include <stdint.h>
#include <stdbool.hh>

/**
 * @brief Decodes a single 4x4 block of a single-channel BC4 or BC5 compressed texture.
 *
 * @param compressedBlock64 The 64-bit data block for one channel. `param_1`
 * @param outputRGBAPtr Base pointer to the destination 4x4 RGBA block. `param_2`
 * @param x The x-coordinate of the block's top-left pixel. `param_3`
 * @param y The y-coordinate of the block's top-left pixel. `param_4`
 * @param imageWidth The total width of the destination image. `param_5`
 * @param imageHeight The total height of the destination image. `param_6`
 * @param imageStride The byte stride of the destination image. `param_7`
 * @param channelOffset The byte offset for the output channel (0=R, 1=G). `param_8`
 * @param isSigned A flag indicating if the format is signed (SNORM). `param_9`
 */
void decode_bc4_bc5_channel_block(uint64_t compressedBlock64, uint8_t* outputRGBAPtr, 
                                  int x, int y, int imageWidth, int imageHeight, 
                                  int imageStride, int channelOffset, bool isSigned) 
{
    uint8_t palette[8];

    // 1. Read Endpoints
    if (!isSigned) {
        // UNORM case
        uint8_t endpoint0 = compressedBlock64 & 0xFF;         // `local_38[0] = uVar4 & 0xff;`
        uint8_t endpoint1 = (compressedBlock64 >> 8) & 0xFF;  // `local_38[1] = uVar4 >> 8 & 0xff;`
        palette[0] = endpoint0;
        palette[1] = endpoint1;

        // 2. Generate Value Palette
        if (endpoint0 > endpoint1) {
            // 8-value palette
            for (int i = 2; i < 8; ++i) {
                palette[i] = ((8 - i) * endpoint0 + (i - 1) * endpoint1) / 7;
            }
        } else {
            // 6-value palette + 0 and 255
            for (int i = 2; i < 6; ++i) {
                palette[i] = ((6 - i) * endpoint0 + (i - 1) * endpoint1) / 5;
            }
            palette[6] = 0;
            palette[7] = 255;
        }
    } else {
        // SNORM case
        int8_t endpoint0 = (int8_t)(compressedBlock64 & 0xFF); // `local_38[0] = (uint)(char)param_1;`
        int8_t endpoint1 = (int8_t)((compressedBlock64 >> 8) & 0xFF); // `local_38[1] = (int)(uVar4 << 0x10) >> 0x18;`
        palette[0] = endpoint0;
        palette[1] = endpoint1;
        
        if (endpoint0 > endpoint1) {
            // 8-value palette
            for (int i = 2; i < 8; ++i) {
                palette[i] = ((8 - i) * endpoint0 + (i - 1) * endpoint1) / 7;
            }
        } else {
            // 6-value palette + -127 and 127
            for (int i = 2; i < 6; ++i) {
                palette[i] = ((6 - i) * endpoint0 + (i - 1) * endpoint1) / 5;
            }
            palette[6] = -127;
            palette[7] = 127;
        }
    }

    // 3. Decode Pixels
    // The indices are stored in the remaining 48 bits (6 bytes) of the block.
    uint64_t indices = compressedBlock64 >> 16;
    
    for (int blockY = 0; blockY < 4; ++blockY) {
        if (y + blockY >= imageHeight) continue;

        for (int blockX = 0; blockX < 4; ++blockX) {
            if (x + blockX >= imageWidth) continue;
            
            uint32_t index = indices & 0x7; // Get 3-bit index
            indices >>= 3;
            
            // Calculate the destination address for the current pixel's specific channel.
            uint8_t* pixelChannel = outputRGBAPtr + 
                                    ((y + blockY) * imageStride) + 
                                    ((x + blockX) * 4) + 
                                    channelOffset;
            *pixelChannel = palette[index];
        }
    }
}
```
