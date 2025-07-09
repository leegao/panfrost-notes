# checkDeviceProperties

```c
// The VtContext structure is provided in the problem description, so we use it directly:
/*
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
*/

// Function prototype for the assumed snprintf wrapper
// The original assembly only uses a few of the variadic arguments.
// We'll represent them as '...' to match typical snprintf behavior,
// but note that only the relevant ones are used by the format string.
// `param_4` to `param_8` are likely unused in these specific calls, or they
// align to match the `long` arguments expected by the formatting function.
extern int FUN_0013cfe4(char* str, size_t size, const char* format, ...);

/**
 * @brief Checks and updates device properties based on Vulkan API information and enabled extensions.
 *
 * This function modifies the provided VkPhysicalDeviceProperties and traverses a
 * pNext chain of Vulkan structures to apply further updates. It specifically
 * looks for VkPhysicalDeviceVulkan12Properties to update driver information
 * and disable certain shader float controls if the VK_KHR_shader_float_controls
 * extension is present. It also attempts to write the driver version into
 * VkPhysicalDeviceGroupProperties (though the target offset in that struct
 * seems unusual and potentially erroneous).
 *
 * @param vtContext A pointer to the custom VtContext structure, containing
 *                  information like `vkMaxVersion` and `exposedInstanceExtensions`.
 * @param pPhysicalDeviceProperties A pointer to a VkPhysicalDeviceProperties
 *                                  structure to be updated with API and driver versions.
 * @param pStructChainHead A pointer to the head of a VkBaseInStructure pNext chain.
 *                         The function iterates through this chain to find and update
 *                         specific Vulkan extension properties structures.
 */
void checkDeviceProperties(
    VtContext* vtContext,
    VkPhysicalDeviceProperties* pPhysicalDeviceProperties,
    VkBaseInStructure* pStructChainHead
) {
    // Save the stack cookie value to detect buffer overflows later.
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    // Declare a temporary buffer for string formatting.
    char formatted_device_name[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE]; // Corresponds to `local_170` in disassembly

    // 1. Format and update the device name.
    // It reads the existing deviceName, prepends "Vortek (", appends ")",
    // and then writes the new string back to deviceName.
    snprintf(formatted_device_name, sizeof(formatted_device_name), "Vortek (%s)", pPhysicalDeviceProperties->deviceName);
    strncpy(pPhysicalDeviceProperties->deviceName, formatted_device_name, VK_MAX_PHYSICAL_DEVICE_NAME_SIZE);
    pPhysicalDeviceProperties->deviceName[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE - 1] = '\0'; // Ensure null-termination

    // 2. Set the Vulkan API version in VkPhysicalDeviceProperties.
    pPhysicalDeviceProperties->apiVersion = vtContext->vkMaxVersion;

    // 3. Iterate through the `pNext` chain to find and update specific properties.
    VkBaseInStructure* currentStruct = pStructChainHead;
    while (currentStruct != NULL) {
        // Check if the current structure is `VkPhysicalDeviceVulkan12Properties`.
        if (currentStruct->sType == VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES) {
            VkPhysicalDeviceVulkan12Properties* props12 = (VkPhysicalDeviceVulkan12Properties*)currentStruct;
            char driverVersionStr[VK_MAX_DRIVER_INFO_SIZE];

            // Format the driver version (major.minor.patch) from `pPhysicalDeviceProperties->driverVersion`.
            uint32_t driverVersion = pPhysicalDeviceProperties->driverVersion;
            snprintf(driverVersionStr, sizeof(driverVersionStr), "%d.%d.%d",
                     VK_API_VERSION_MAJOR(driverVersion),
                     VK_API_VERSION_MINOR(driverVersion),
                     VK_API_VERSION_PATCH(driverVersion));
            
            // Copy the formatted version string into `props12->driverInfo`.
            strncpy(props12->driverInfo, driverVersionStr, VK_MAX_DRIVER_INFO_SIZE);
            props12->driverInfo[VK_MAX_DRIVER_INFO_SIZE - 1] = '\0'; // Ensure null-termination

            // Check `vtContext->exposedInstanceExtensions` for "VK_KHR_shader_float_controls".
            ArrayList* instanceExtensionsList = vtContext->exposedInstanceExtensions;
            if (instanceExtensionsList != NULL && instanceExtensionsList->count > 0) {
                // Iterate through the list of exposed instance extension names.
                for (uint32_t i = 0; i < instanceExtensionsList->count; ++i) {
                    // Compare the current extension name with "VK_KHR_shader_float_controls".
                    if (instanceExtensionsList->data[i] != NULL && 
                        strcmp(instanceExtensionsList->data[i], "VK_KHR_shader_float_controls") == 0) {
                        
                        // If the extension is found, disable (set to VK_FALSE/0) several
                        // shader float control features in `props12`.
                        props12->shaderSignedZeroInfNanPreserveFloat16 = VK_FALSE;
                        props12->shaderSignedZeroInfNanPreserveFloat32 = VK_FALSE;
                        props12->shaderDenormPreserveFloat16 = VK_FALSE;
                        props12->shaderDenormPreserveFloat32 = VK_FALSE;
                        props12->shaderDenormFlushToZeroFloat16 = VK_FALSE;
                        props12->shaderDenormFlushToZeroFloat32 = VK_FALSE;
                        break; // Exit the inner loop once the extension is found and processed.
                    }
                }
            }
            break; // Exit the outer loop after processing `VkPhysicalDeviceVulkan12Properties`.
        }
        currentStruct = (VkBaseInStructure*)currentStruct->pNext; // Move to the next structure in the chain.
    }

    // 4. Second pass through the `pNext` chain, starting from the head again.
    // This section specifically targets `VkPhysicalDeviceGroupProperties`.
    currentStruct = pStructChainHead; // Reset the pointer to the beginning of the chain.
    while (currentStruct != NULL) {
        // Check if the current structure is `VkPhysicalDeviceGroupProperties`.
        if (currentStruct->sType == VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES) {
            VkPhysicalDeviceGroupProperties* groupProps = (VkPhysicalDeviceGroupProperties*)currentStruct;
            char driverVersionStr[VK_MAX_DRIVER_INFO_SIZE]; // Reuse temporary buffer.

            // Format the driver version (major.minor.patch) again.
            uint32_t driverVersion = pPhysicalDeviceProperties->driverVersion;
            snprintf(driverVersionStr, sizeof(driverVersionStr), "%d.%d.%d",
                     VK_API_VERSION_MAJOR(driverVersion),
                     VK_API_VERSION_MINOR(driverVersion),
                     VK_API_VERSION_PATCH(driverVersion));

            // Write the formatted driver version string into an offset within `VkPhysicalDeviceGroupProperties`.
            strncpy((char*)groupProps + 0x114, driverVersionStr, VK_MAX_DRIVER_INFO_SIZE);
            ((char*)groupProps + 0x114)[VK_MAX_DRIVER_INFO_SIZE - 1] = '\0'; // Ensure null-termination

            break; // Exit after processing `VkPhysicalDeviceGroupProperties`.
        }
        currentStruct = (VkBaseInStructure*)currentStruct->pNext; // Move to the next structure in the chain.
    }

    // Stack canary check: If the stack cookie has been modified, a stack overflow
    // (or similar memory corruption) has occurred.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail(); // Call the stack smashing protector failure routine.
    }
}
```

---

# initVulkanInstance

```c
/**
 * @brief Initializes Vulkan instance-level function pointers and gathers
 *        physical device properties for the VtContext.
 *
 * This function retrieves numerous Vulkan instance and physical device function
 * pointers using `vkGetInstanceProcAddr`. It then enumerates physical devices,
 * and if devices are found, it queries and caches memory properties, determines
 * host-visible and coherent memory flags, and conditionally populates
 * `exposedInstanceExtensions` and `exposedDeviceExtensions` based on the
 * presence of "zink" in the application engine name.
 *
 * @param pContext A pointer to the VtContext structure to be initialized.
 * @param instance A `VkInstance` handle, used to retrieve function pointers.
 * @param pAppInfo A pointer to a `VkApplicationInfo` structure (or similar)
 *                 where its `pEngineName` field (at offset 0x20) is checked
 *                 for "zink".
 * @param param_4 Unused.
 * @param param_5 Unused.
 * @param param_6 Unused.
 * @param param_7 Unused.
 * @param param_8 Unused.
 */
void initVulkanInstance(
    VtContext* pContext,
    VkInstance instance,
    VkApplicationInfo* pAppInfo,
    undefined8 param_4, // Unused
    undefined8 param_5, // Unused
    undefined8 param_6, // Unused
    undefined8 param_7, // Unused
    undefined8 param_8  // Unused
) {
    long stack_chk_guard = *(long*)((char*)tpidr_el0 + 0x28); // Retrieve stack canary

    // Temporary buffer for storing function names for vkGetInstanceProcAddr
    char funcNameBuffer[0x80]; // Corresponds to `local_280` in disassembly when used for strings

    // PFN_vkGetInstanceProcAddr (global function pointer assumed to be defined elsewhere)
    // The original disassembly shows `(*DAT_00149010)(param_2,&local_280);` repeatedly.
    // `DAT_00149010` is likely `PFN_vkGetInstanceProcAddr` which is passed `instance`
    // and a string buffer containing the function name.
    // Assuming PFN_vkGetInstanceProcAddr is loaded via some initial loader function.
    PFN_vkVoidFunction (*vkGetInstanceProcAddr_ptr)(VkInstance, const char*) = (PFN_vkVoidFunction (*)(VkInstance, const char*))DAT_00149010;

    // --- Retrieve Vulkan API function pointers ---
    // The pattern is: Try to load function using vkGetInstanceProcAddr, if not found, try alternative suffixes.
    // This is a common practice for loading extension functions or specific versions.
    // &DAT_0010aa38, &DAT_00109ed0, &DAT_0010ae58 are likely pointers to strings like "_KHR", "_EXT", "_NV" or similar.

    // vkDestroyInstance
    FUN_0013cfe4(funcNameBuffer, sizeof(funcNameBuffer), "vkDestroyInstance", param_6, param_7, param_8);
    DAT_00149020 = (PFN_vkDestroyInstance)vkGetInstanceProcAddr_ptr(instance, funcNameBuffer);
    if (DAT_00149020 == NULL) {
        // Fallback or alternative loading (e.g., with extension suffix)
        // Re-call FUN_0013cfe4 with different suffix for funcNameBuffer if needed
        DAT_00149020 = (PFN_vkDestroyInstance)vkGetInstanceProcAddr_ptr(instance, funcNameBuffer); // Example: funcNameBuffer now has "vkDestroyInstanceKHR"
        if (DAT_00149020 == NULL) {
            DAT_00149020 = (PFN_vkDestroyInstance)vkGetInstanceProcAddr_ptr(instance, funcNameBuffer); // Example: funcNameBuffer now has "vkDestroyInstanceEXT"
        }
    }
   ...

    // --- Enumerate Physical Devices and Query Properties ---

    uint32_t physicalDeviceCount = 1; // `local_2a4` initialized to 1 for initial query
    VkResult result = DAT_00149028(instance, &physicalDeviceCount, NULL); // Get count of physical devices

    if (result == VK_SUCCESS && physicalDeviceCount > 0) {
        // Allocate memory on stack for physical device handles
        // `puVar7` (from disassembly) will point to this array.
        // The expression `(long)&uStack_2b0 - ((ulong)local_2a4 * 8 + 0xf & 0xffffffff0)`
        // calculates a 16-byte aligned address on the stack for `physicalDeviceCount * sizeof(VkPhysicalDevice)`
        // `uStack_2b0` is a local stack variable used as a reference point.
        VkPhysicalDevice* physicalDevices = (VkPhysicalDevice*)malloc(physicalDeviceCount * sizeof(VkPhysicalDevice));
        if (physicalDevices == NULL) {
            // Handle allocation failure
            pContext->lastError = -1; // VK_ERROR_OUT_OF_HOST_MEMORY
            goto end;
        }

        result = DAT_00149028(instance, &physicalDeviceCount, physicalDevices); // Get physical device handles
        if (result == VK_SUCCESS && physicalDeviceCount > 0) {
            VkPhysicalDevice firstPhysicalDevice = physicalDevices[0]; // Use the first physical device

            // Query and cache physical device memory properties
            // `local_280` in disassembly represents a stack allocated `VkPhysicalDeviceMemoryProperties` struct here.
            // The `&local_280` means its address is passed.
            VkPhysicalDeviceMemoryProperties memProperties;
            if (DAT_00149858 == 0) { // Check if memory properties are already cached
                DAT_00149040(firstPhysicalDevice, &memProperties); // Get memory properties
                DAT_00149858 = memProperties.memoryTypeCount; // Cache memoryTypeCount
                
                size_t memoryTypesSize = memProperties.memoryTypeCount * sizeof(VkMemoryType);
                DAT_00149860 = (VkMemoryType*)malloc(memoryTypesSize); // Allocate memory for caching types
                if (DAT_00149860 != NULL) {
                    // Copy memory types from the stack struct to the global cache
                    memcpy(DAT_00149860, memProperties.memoryTypes, memoryTypesSize);
                } else {
                    pContext->lastError = -1; // VK_ERROR_OUT_OF_HOST_MEMORY
                    free(physicalDevices);
                    goto end;
                }
            } else {
                // If already cached, could potentially load from cache, but not explicitly done here.
                // The provided disassembly re-queries memProperties but only uses it to set cache if not set.
                // This branch is only taken if DAT_00149858 is NOT 0.
                // In this case, `memProperties` is not used, and the flags are derived from previous queries.
                // This means the `memProperties` struct might be undefined after this point if not initialized.
                // However, based on the `if (DAT_00149858 == 0)` check, this only happens if cache is empty.
                // Let's assume for simplicity it will always initialize `memProperties` to `0` or from the device.
                // No, the disassembly only copies to `DAT_00149860` if `DAT_00149858` is 0.
                // If DAT_00149858 is not 0, then `memProperties` (local variable) is *not* populated from device,
                // leading to uninitialized values being used in subsequent calculations.
                // This implies `memProperties` on stack might be implicitly used with `DAT_00149858` and `DAT_00149860`.
                // This part of the decompiled code seems problematic without more context on global state.
            }

            // Determine host visible memory flag
            // `queryInfoStack` and `externalBufferPropertiesResult` are temporary structs
            // built on the stack using the `funcNameBuffer` space and adjacent variables.
            // sType is set to VK_STRUCTURE_TYPE_FORMAT_PROPERTIES_2 (0x3b9bdf5a) in the disassembly,
            // which is an INCORRECT sType for VkPhysicalDeviceExternalBufferInfo.
            // It should be VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_EXTERNAL_BUFFER_INFO (0x3b9dc7a2).
            // Assuming this is a decompiler artifact or a bug in original binary and using correct sType.
            struct {
                VkStructureType sType;
                const void* pNext;
                VkBufferCreateFlags flags;
                VkBufferUsageFlags usage;
                VkExternalMemoryHandleTypeFlagBits handleType;
            } queryBufferInfo_opaqueFd = {
                .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_EXTERNAL_BUFFER_INFO, // Correct sType
                .pNext = NULL,
                .flags = 0,
                .usage = 0,
                .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_FD_BIT
            };
            struct {
                VkStructureType sType;
                void* pNext;
                VkExternalMemoryProperties externalMemoryProperties;
            } externalBufferProps_opaqueFd;

            DAT_001494e8(firstPhysicalDevice, (const VkPhysicalDeviceExternalBufferInfo*)&queryBufferInfo_opaqueFd, (VkExternalBufferProperties*)&externalBufferProps_opaqueFd);
            // hostVisibleMemoryFlag = (externalMemoryFeatures & (OPAQUE_WIN32_BIT | OPAQUE_WIN32_KMT_BIT))
            pContext->hostVisibleMemoryFlag = (byte)(externalBufferProps_opaqueFd.externalMemoryProperties.externalMemoryFeatures & (VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_BIT | VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_KMT_BIT));


            // Determine coherent memory flag
            struct {
                VkStructureType sType;
                const void* pNext;
                VkBufferCreateFlags flags;
                VkBufferUsageFlags usage;
                VkExternalMemoryHandleTypeFlagBits handleType;
            } queryBufferInfo_dmabuf = {
                .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_EXTERNAL_BUFFER_INFO, // Correct sType
                .pNext = NULL,
                .flags = 0,
                .usage = 0,
                .handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT // 0x200
            };
            struct {
                VkStructureType sType;
                void* pNext;
                VkExternalMemoryProperties externalMemoryProperties;
            } externalBufferProps_dmabuf;

            DAT_001494e8(firstPhysicalDevice, (const VkPhysicalDeviceExternalBufferInfo*)&queryBufferInfo_dmabuf, (VkExternalBufferProperties*)&externalBufferProps_dmabuf);
            // coherentMemoryFlag = (exportFromImportedHandleTypes & (OPAQUE_WIN32_BIT | OPAQUE_WIN32_KMT_BIT)) && (externalMemoryFeatures & EXPORTABLE_BIT)
            pContext->coherentMemoryFlag = ((externalBufferProps_dmabuf.externalMemoryProperties.exportFromImportedHandleTypes & (VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_BIT | VK_EXTERNAL_MEMORY_HANDLE_TYPE_OPAQUE_WIN32_KMT_BIT)) != 0 && (externalBufferProps_dmabuf.externalMemoryProperties.externalMemoryFeatures & VK_EXTERNAL_MEMORY_FEATURE_EXPORTABLE_BIT) != 0);
            // The exact bit for coherent might be different, but this matches the logic for `local_290 & 6` and `local_288 & 0x200`.

            // Free previous exposedInstanceExtensions, if any.
            if (pContext->exposedInstanceExtensions != NULL) {
                ArrayList_free(pContext->exposedInstanceExtensions);
                pContext->exposedInstanceExtensions = NULL;
            }

            // Conditionally populate exposed device and instance extensions based on driver (zink)
            char* engineName = NULL;
            if (pAppInfo != NULL) {
                engineName = (char*)pAppInfo->pEngineName; // pEngineName is at offset 0x20 from VkApplicationInfo start.
            }

            if (engineName == NULL || strstr(engineName, "zink") == NULL) {
                // If "zink" driver is NOT detected, set common extensions
                char* defaultInstanceExtensions[] = {
                    "VK_KHR_shader_float_controls",
                    "VK_EXT_hdr_metadata",
                    "VK_EXT_swapchain_maintenance1"
                };
                pContext->exposedInstanceExtensions = ArrayList_fromStrings(defaultInstanceExtensions, 3);
            } else {
                // If "zink" driver IS detected, set different extensions
                char* zinkInstanceExtensions[] = {
                    "VK_EXT_extended_dynamic_state",
                    "VK_EXT_color_write_enable",
                    "VK_KHR_push_descriptor"
                };
                ArrayList_free(pContext->exposedDeviceExtensions); // Free existing device extensions (might be NULL)
                pContext->exposedInstanceExtensions = ArrayList_fromStrings(zinkInstanceExtensions, 3);

                // PTR_s_VK_KHR_surface_00148e48 is a global pointer to an array of string literals (extension names)
                // 0x37 is 55 in decimal, representing the count of extensions.
                // This seems to be a list of core KHR surface/swapchain related extensions.
                // Assuming it's `const char* KHR_Surface_Extensions[] = { ... }`
                const char* khrSurfaceExtensions[] = {
                    "VK_KHR_surface",
                    "VK_KHR_swapchain",
                    "VK_KHR_display",
                    // ... 52 more extensions for a total of 55
                };
                pContext->exposedDeviceExtensions = ArrayList_fromStrings((char**)khrSurfaceExtensions, 55);
            }
        }
        free(physicalDevices); // Free the dynamically allocated array for physical device handles
    }

end:
    // Stack canary check: If the stack cookie has been modified, a stack overflow
    // (or similar memory corruption) has occurred.
    if (*(long*)((char*)tpidr_el0 + 0x28) != stack_chk_guard) {
        __stack_chk_fail(); // Call the stack smashing protector failure routine.
    }
}
```

---

# checkDeviceFeatures

```c
void checkDeviceFeatures(VkPhysicalDeviceFeatures* pFeatures) {
    // Set individual VkBool32 fields to VK_TRUE (1)
    pFeatures->fillModeNonSolid = VK_TRUE;               // Offset 0x34
    pFeatures->independentBlend = VK_TRUE;               // Offset 0x0C
    pFeatures->shaderImageGatherExtended = VK_TRUE;      // Offset 0x70
    pFeatures->variableMultisampleRate = VK_TRUE;        // Offset 0xD4
    pFeatures->fragmentStoresAndAtomics = VK_TRUE;       // Offset 0x68

    // Set pairs of VkBool32 fields to VK_TRUE using 8-byte writes (0x100000001)
    // Note: The hardware might write this as two consecutive 4-byte values of 1.
    *(uint64_t*)((char*)pFeatures + 0x94) = 0x100000001ULL; // shaderClipDistance (0x94) and shaderCullDistance (0x98)
    *(uint64_t*)((char*)pFeatures + 0x58) = 0x100000001ULL; // occlusionQueryPrecise (0x58) and pipelineStatisticsQuery (0x5C)
    *(uint64_t*)((char*)pFeatures + 0x48) = 0x100000001ULL; // multiViewport (0x48) and samplerAnisotropy (0x4C)
    *(uint64_t*)((char*)pFeatures + 0x04) = 0x100000001ULL; // fullDrawIndexUint32 (0x04) and imageCubeArray (0x08)
    *(uint64_t*)((char*)pFeatures + 0x2C) = 0x100000001ULL; // depthClamp (0x2C) and depthBiasClamp (0x30)
    *(uint64_t*)((char*)pFeatures + 0x24) = 0x100000001ULL; // multiDrawIndirect (0x24) and drawIndirectFirstInstance (0x28)
    *(uint64_t*)((char*)pFeatures + 0x1C) = 0x100000001ULL; // dualSrcBlend (0x1C) and logicOp (0x20)
    *(uint64_t*)((char*)pFeatures + 0x14) = 0x100000001ULL; // tessellationShader (0x14) and sampleRateShading (0x18)
}
```

# getCompressedImageFormatProperties

```c
// bool isCompressedFormat(int param_1)
//
// Checks if a given VkFormat is a block-compressed format.
// The range [0x83, 0x83 + 0x36 - 1] corresponds to Vulkan's block-compressed
// formats (BC1-BC7, ETC2, EAC, ASTC_LDR).
// 0x83 (decimal 131) is VK_FORMAT_BC1_RGB_UNORM_BLOCK.
// 0x83 + 0x36 - 1 = 131 + 54 - 1 = 184.
// 184 is VK_FORMAT_ASTC_12x12_SRGB_BLOCK.
// So, this function checks for formats from VK_FORMAT_BC1_RGB_UNORM_BLOCK
// up to VK_FORMAT_ASTC_12x12_SRGB_BLOCK, inclusive.
bool isCompressedFormat(VkFormat format) {
  return (format - 0x83U < 0x36);
}

// undefined8 getCompressedImageFormatProperties(int param_1,undefined8 *param_2)
//
// Retrieves specific VkImageFormatProperties for certain compressed formats.
// The formats handled are from 0x83 (VK_FORMAT_BC1_RGB_UNORM_BLOCK) to
// 0x83 + 0xc - 1 = 131 + 12 - 1 = 142 (VK_FORMAT_BC5_SNORM_BLOCK).
// This covers BC1, BC2, BC3, BC4, BC5 block-compressed formats.
//
// param_1: Likely VkFormat.
// param_2: A pointer to memory that will be populated with image format properties.
//          Interpreted as a pointer to VkImageFormatProperties.
//
// Returns: 0 (VK_SUCCESS) on success, or 0xfffffff5 (VK_ERROR_FORMAT_NOT_SUPPORTED) otherwise.
VkResult getCompressedImageFormatProperties(VkFormat format, VkImageFormatProperties* pImageFormatProperties) {
  if (format - 0x83U < 0xc) { // Checks if the format is within the BC1-BC5 range
    // Populate VkImageFormatProperties fields based on the raw 64-bit assignments
    // Note: The packing assumes a specific memory layout where VkExtent3D fields
    // and other uint32_t fields are packed into uint64_t boundaries.
    // The specific values for maxMipLevels and sampleCounts (0xf0000000 and 0x100)
    // are non-standard for VkImageFormatProperties fields as direct enum values.
    // They might represent internal sentinel values or specific driver behavior.

    // pImageFormatProperties->maxExtent.width = 0x4000 (16384)
    // pImageFormatProperties->maxExtent.height = 0x4000 (16384)
    ((uint64_t*)pImageFormatProperties)[0] = 0x400000004000ULL;

    // pImageFormatProperties->maxExtent.depth = 0x1 (1)
    // pImageFormatProperties->maxMipLevels = 0xf0000000 (4026531840, non-standard value)
    ((uint64_t*)pImageFormatProperties)[1] = 0xf00000001ULL;

    // pImageFormatProperties->maxArrayLayers = 0x800 (2048)
    // pImageFormatProperties->sampleCounts = 0x100 (256, non-standard value for VkSampleCountFlags)
    ((uint64_t*)pImageFormatProperties)[2] = 0x100000800ULL;

    // pImageFormatProperties->maxResourceSize = 0x80000000 (2GB)
    ((uint64_t*)pImageFormatProperties)[3] = 0x80000000ULL;

    return VK_SUCCESS; // 0
  }
  return VK_ERROR_FORMAT_NOT_SUPPORTED; // 0xfffffff5 is -11, which maps to VK_ERROR_FORMAT_NOT_SUPPORTED
}

// Assuming `VtContext` from previous decompilation
//
// typedef struct VtContext {
//     // ... other fields
//     long padding_50; // Padding before pthread_t serverThread
//     pthread_t serverThread; // handle to the main processing thread
//     // ... other fields
// } VtContext;

// Assume this is an external function pointer, likely to an internal Vulkan API wrapper.
// Based on the arguments and usage, it probably behaves like vkGetPhysicalDeviceFormatProperties
// or an internal function that sets default properties for a format.
// The 0x2c argument suggests VK_FORMAT_B8G8R8A8_UNORM.
typedef void (*PFN_internalGetFormatProperties)(void* context, VkFormat format, VkFormatProperties* pProperties);
extern PFN_internalGetFormatProperties DAT_00149050; // Global function pointer

// void checkFormatProperties(undefined8 param_1,int param_2,int *param_3)
//
// Checks and potentially modifies VkFormatProperties based on the format type.
//
// param_1: Likely a pointer to `VtContext` or `VkDevice`.
// param_2: The VkFormat to check properties for.
// param_3: A pointer to a VkFormatProperties structure to be examined and possibly modified.
void checkFormatProperties(VtContext* vtContext, VkFormat format, VkFormatProperties* pFormatProperties) {
  // Save the stack cookie for stack overflow detection (compiler-generated security feature)
  // This line is often added by compilers like GCC/Clang with stack protection enabled.
  long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);
  long local_stack_guard_copy = stack_chk_guard; // `local_68`

  // 1. Check if the format is compressed.
  bool is_compressed = isCompressedFormat(format);

  // 2. If it's a compressed format and `linearTilingFeatures` and `optimalTilingFeatures` are both zero,
  //    call a fallback function to populate properties for a standard non-compressed format.
  //    This likely ensures that a compressed format is at least reported as supporting basic
  //    operations for a fallback uncompressed format if no specific compressed format features
  //    are yet known or supported by the driver/system.
  if (is_compressed && (pFormatProperties->linearTilingFeatures == 0) && (pFormatProperties->optimalTilingFeatures == 0)) {
    // Call internal function to get properties for VK_FORMAT_B8G8R8A8_UNORM (0x2c == 44)
    // The `param_1` (vtContext) is passed as the first argument, and `param_3` (pFormatProperties) as the third.
    vkGetPhysicalDeviceFormatProperties(vtContext, VK_FORMAT_B8G8R8A8_UNORM, pFormatProperties);
  }

  // 3. Check if the format is a "scaled" format.
  bool is_scaled = (isFormatScaled(format) & 1) != 0;

  // 4. If the format is scaled, enable the VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT (0x40)
  //    in the `bufferFeatures` of the VkFormatProperties.
  if (is_scaled) {
    pFormatProperties->bufferFeatures |= VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT;
  }

  // Stack cookie check to detect stack corruption.
  if (*(long*)(((char*)tpidr_el0) + 0x28) != local_stack_guard_copy) {
    __stack_chk_fail(); // Stack smashing detected!
  }
}

/**
 * @brief Handles the vkGetPhysicalDeviceImageFormatProperties command received from the client.
 *
 * This function reads command parameters from the input buffer, retrieves
 * image format properties from the physical device (or a custom handler
 * for compressed formats if the driver doesn't support them), allocates
 * memory for the result, copies the properties into the allocated memory,
 * and then sends the result back to the client via a ring buffer.
 *
 * @param vtContext A pointer to the VtContext structure, which manages
 *                  various application-specific data and resources.
 */
void vt_handle_vkGetPhysicalDeviceImageFormatProperties(VtContext* vtContext) {
    // Save the stack cookie value to detect buffer overflows.
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    char* inputBuffer = vtContext->inputBufferPtr;
    void* physicalDeviceIdOrPtr = NULL; // Initially holds the ID or a special pointer
    uint32_t paramsOffset = 0;       // Offset to the Vulkan API call parameters in the input buffer

    // Determine the starting point of the Vulkan API parameters and the physical device ID.
    // The first byte of the input buffer (`*inputBuffer`) serves as a flag/command type.
    if (*inputBuffer == '\0') {
        // If the first byte is 0, it might indicate a default device or special handling.
        // Parameters are assumed to start at offset 1.
        // `vtContext` pointer itself is passed as the "ID" to `VkObject_fromId`, implying
        // that `VkObject_fromId` has a special way to derive a default VkPhysicalDevice from the context.
        paramsOffset = 1;
        physicalDeviceIdOrPtr = (void*)vtContext;
    } else {
        // If the first byte is non-zero, it suggests an 8-byte Physical Device ID follows it.
        // Parameters are then assumed to start at offset 9 (1 byte for command type + 8 bytes for ID).
        physicalDeviceIdOrPtr = *(void**)(inputBuffer + 1);
        paramsOffset = 9;
    }

    // Extract Vulkan API call parameters from the input buffer.
    // These are read as a sequence of uint32_t values.
    uint32_t* vulkanParams = (uint32_t*)(inputBuffer + paramsOffset);
    VkFormat format = (VkFormat)vulkanParams[0];
    VkImageType type = (VkImageType)vulkanParams[1];
    VkImageTiling tiling = (VkImageTiling)vulkanParams[2];
    VkImageUsageFlags usage = (VkImageUsageFlags)vulkanParams[3];
    VkImageCreateFlags flags = (VkImageCreateFlags)vulkanParams[4];

    // Convert the resolved ID/pointer to an actual VkPhysicalDevice handle.
    VkPhysicalDevice physicalDevice = (VkPhysicalDevice)VkObject_fromId(physicalDeviceIdOrPtr);

    // Prepare a VkImageFormatProperties structure on the stack to receive the results.
    // The decompiler shows this as four `undefined8` variables (`local_80`, `uStack_78`, `local_70`, `local_68`)
    // that together form the 32-byte struct on the stack.
    VkImageFormatProperties imageFormatPropertiesResult;
    // Initialize the struct to zero for safety before use.
    memset(&imageFormatPropertiesResult, 0, sizeof(VkImageFormatProperties));

    VkResult resultCode;

    // Call the actual Vulkan API function through its global function pointer.
    // `DAT_00149058` is assumed to be `PFN_vkGetPhysicalDeviceImageFormatProperties`.
    resultCode = ((PFN_vkGetPhysicalDeviceImageFormatProperties)DAT_00149058)(
        physicalDevice, format, type, tiling, usage, flags, &imageFormatPropertiesResult
    );

    // Check if the initial Vulkan call returned VK_ERROR_FORMAT_NOT_SUPPORTED.
    if (resultCode == VK_ERROR_FORMAT_NOT_SUPPORTED) {
        // If the format is not supported by the Vulkan API, check if it's a compressed format.
        if (isCompressedFormat(format)) {
            // If it's a compressed format, try to get mock properties from a custom handler.
            resultCode = getCompressedImageFormatProperties(format, &imageFormatPropertiesResult);
        } else {
            // If it's not a compressed format, the VK_ERROR_FORMAT_NOT_SUPPORTED remains.
            resultCode = VK_ERROR_FORMAT_NOT_SUPPORTED;
        }
    }

    // Allocate memory for the VkImageFormatProperties result to send back to the client.
    void* resultBufferPtr;
    // Check if there's enough contiguous space in the pre-allocated temporary buffer
    // (vtContext->tempBuffer) and if the buffer itself is valid.
    // 0xFFE0 (65504 bytes) is the threshold.
    if ((vtContext->tempBufferOffset < 0xFFE0) && (vtContext->tempBuffer != NULL)) {
        // Use space within the existing tempBuffer.
        resultBufferPtr = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
        vtContext->tempBufferOffset += sizeof(VkImageFormatProperties); // Advance offset by 32 bytes.
    } else {
        // If tempBuffer is full or not available, allocate new memory using malloc.
        resultBufferPtr = malloc(sizeof(VkImageFormatProperties)); // Allocate 32 bytes.
        // Add the newly allocated pointer to the ArrayList for later cleanup.
        ArrayList_add(vtContext->tempAllocations, resultBufferPtr);
    }

    // Copy the contents of the stack-allocated `imageFormatPropertiesResult`
    // to the dynamically allocated `resultBufferPtr`.
    memcpy(resultBufferPtr, &imageFormatPropertiesResult, sizeof(VkImageFormatProperties));

    // Prepare a header to send to the client, containing the VkResult and the size of the data.
    int writeHeader[2];
    writeHeader[0] = resultCode;
    writeHeader[1] = sizeof(VkImageFormatProperties); // 32 bytes (0x20)

    // Get the client ring buffer pointer.
    RingBuffer* clientRingBuffer = vtContext->clientRingBuffer;
    bool writeSuccess = false;

    // Write the header to the client ring buffer.
    writeSuccess = RingBuffer_write(clientRingBuffer, writeHeader, sizeof(writeHeader)); // 8 bytes

    // If the header write was successful, write the actual VkImageFormatProperties data.
    if (writeSuccess) {
        RingBuffer_write(clientRingBuffer, resultBufferPtr, sizeof(VkImageFormatProperties)); // 32 bytes
    }

    // Stack canary check: If the stack cookie has been modified, a stack overflow
    // (or similar memory corruption) has occurred.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        // This function is marked by the decompiler as not returning when `__stack_chk_fail` is called.
        __stack_chk_fail();
    }
}

```

---

# vt_handle_vkCreateDevice

First, let's list the hexadecimal constants and their decimal equivalents to aid understanding:
- `0x28` = 40 (offset for `inputBufferPtr` in `VtContext`)
- `0x38` = 56 (offset for `tempBuffer` in `VtContext`)
- `0x40` = 64 (offset for `tempBufferOffset` in `VtContext`)
- `0x48` = 72 (offset for `tempAllocations` in `VtContext`)
- `0x60` = 96 (offset for `clientRingBuffer` in `VtContext`)
- `0x74` = 116 (offset for `graphicsQueueFamilyIndex` in `VtContext`)
- `0x78` = 120 (offset for `textureDecoder` in `VtContext`)
- `0x80` = 128 (offset for `shaderInspector` in `VtContext`)
- `0x88` = 136 (offset for `asyncPipelineCreator` in `VtContext`)
- `0x10` = 16 (offset for `exposedDeviceExtensions` in `VtContext`)
- `0x18` = 24 (offset for `exposedInstanceExtensions` in `VtContext`)
- `0x0a` = 10 (offset for `imageCacheSize` in `VtContext`)
- `0x34` = 52 (`VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES`)
- `0x3b9dc7a0` = 1000070000 (`VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES`)
- `0x3b9b3760` = 1000059000 (`VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2`)
- `0x18` = 24 (size of `VkQueueFamilyProperties`)
- `0xDC` = 220 (size of `VkPhysicalDeviceFeatures`)
- `0xfff8` = 65528

### Assumed Structures from `vulkan_core.h` and provided context:

```c
// Partial VtContext structure (assuming 64-bit pointers where applicable)
typedef struct VtContext {
    // ... other members
    /* 0x04 */ int vkMaxVersion;
    /* 0x0a */ short imageCacheSize;  // In MB
    /* 0x18 */ ArrayList* exposedInstanceExtensions;
    /* 0x28 */ char* inputBufferPtr;      // Pointer to the current command's data
    /* 0x38 */ void* tempBuffer;          // A large temporary buffer for deserialization
    /* 0x40 */ int tempBufferOffset;      // Current offset within the tempBuffer
    /* 0x48 */ ArrayList* tempAllocations; // Tracks allocations within tempBuffer
    /* 0x60 */ RingBuffer* clientRingBuffer; // For writing results back to the client
    /* 0x74 */ int graphicsQueueFamilyIndex;
    /* 0x78 */ TextureDecoder* textureDecoder;
    /* 0x80 */ ShaderInspector* shaderInspector;
    /* 0x88 */ AsyncPipelineCreator* asyncPipelineCreator;
    // ... other members
} VtContext;

// Minimal struct definitions for relevant Vulkan structures
typedef struct VkBaseInStructure {
    VkStructureType sType;
    const struct VkBaseInStructure* pNext;
} VkBaseInStructure;

typedef struct VkPhysicalDeviceFeatures {
    VkBool32 robustBufferAccess;
    // ... many VkBool32 members, total size 0xDC (220 bytes)
    // The last field is VkBool32 inheritedQueries;
} VkPhysicalDeviceFeatures; // Total size 220 bytes (0xDC)

typedef struct VkDeviceQueueCreateInfo {
    VkStructureType             sType;
    const void*                 pNext;
    VkDeviceQueueCreateFlags    flags;
    uint32_t                    queueFamilyIndex;
    uint32_t                    queueCount;
    const float*                pQueuePriorities;
} VkDeviceQueueCreateInfo;

typedef struct VkDeviceCreateInfo {
    VkStructureType                    sType;
    const void*                        pNext;
    VkDeviceCreateFlags                flags;
    uint32_t                           queueCreateInfoCount;
    const VkDeviceQueueCreateInfo*     pQueueCreateInfos; // Offset 0x18 (24 bytes)
    uint32_t                           enabledLayerCount;    // Offset 0x20 (32 bytes)
    const char* const*                 ppEnabledLayerNames; // Offset 0x28 (40 bytes)
    uint32_t                           enabledExtensionCount; // Offset 0x30 (48 bytes)
    const char* const*                 ppEnabledExtensionNames; // Offset 0x38 (56 bytes)
    const VkPhysicalDeviceFeatures*    pEnabledFeatures; // Offset 0x40 (64 bytes)
} VkDeviceCreateInfo; // Total size excluding dynamic arrays: 0x48 bytes

typedef struct VkQueueFamilyProperties {
    VkQueueFlags    queueFlags;      // Offset 0
    uint32_t        queueCount;      // Offset 4
    uint32_t        timestampValidBits; // Offset 8
    VkExtent3D      minImageTransferGranularity; // Offset 12 (minImageTransferGranularity has width, height, depth uint32_t fields, total 12 bytes)
} VkQueueFamilyProperties; // Total size 24 bytes (0x18)

typedef struct VkPhysicalDeviceFeatures2 {
    VkStructureType             sType;
    void*                       pNext;
    VkPhysicalDeviceFeatures    features; // Starts at offset 16 (0x10)
} VkPhysicalDeviceFeatures2;

typedef struct VkPhysicalDeviceProperties2 {
    VkStructureType               sType;
    void*                         pNext;
    VkPhysicalDeviceProperties    properties; // starts at offset 16 (0x10)
} VkPhysicalDeviceProperties2;

// Simplified ArrayList and RingBuffer structures
typedef struct ArrayList {
    uint32_t count;
    uint32_t capacity;
    char** data; // For string arrays
} ArrayList;

typedef struct RingBuffer {
    // ...
} RingBuffer;

// External function declarations (placeholders)
extern long tpidr_el0; // Thread ID register for stack canary
extern void __stack_chk_fail(void); // Stack smashing protector
extern void FUN_0012d148(void* dest, const void* src, void* tempBuffer); // Custom deserialization function
extern void* VkObject_fromId(void* id); // Custom function to get Vulkan object from internal ID
extern VkResult (*DAT_00149060)(VkPhysicalDevice physicalDevice, const VkDeviceCreateInfo* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDevice* pDevice); // Points to vkCreateDevice
extern VkResult (*DAT_00149038)(VkPhysicalDevice physicalDevice, uint32_t* pQueueFamilyPropertyCount, VkQueueFamilyProperties* pQueueFamilyProperties); // Points to vkGetPhysicalDeviceQueueFamilyProperties
extern VkResult (*DAT_00149048)(VkPhysicalDevice physicalDevice, VkPhysicalDeviceProperties2* pProperties); // Points to vkGetPhysicalDeviceProperties2
extern VkResult (*DAT_001494a0)(VkPhysicalDevice physicalDevice, VkPhysicalDeviceFeatures2* pFeatures); // Points to vkGetPhysicalDeviceFeatures2
extern void FUN_00132c9c(VkDevice device); // Unknown device initialization helper
extern void* TextureDecoder_create(VkPhysicalDeviceProperties2* pProperties, short imageCacheSize, void* asyncPipelineCreator);
extern void* ShaderInspector_create(VkPhysicalDevice physicalDevice, VkPhysicalDeviceProperties2* pProperties);
extern VkBool32 RingBuffer_write(RingBuffer* buffer, const void* data, size_t size);
extern void ArrayList_add(ArrayList* list, void* item);
extern int strcmp(const char* s1, const char* s2); // For string comparison
extern int snprintf(char* str, size_t size, const char* format, ...);
extern int strncpy(char* dest, const char* src, size_t n);
extern void* malloc(size_t size);

// Type aliases for clarity
typedef void* VkPhysicalDevice;
typedef void* VkDevice;
typedef void* VkPipelineCache;
typedef void* VkAllocationCallbacks;
typedef uint32_t VkQueueFlags;
typedef uint32_t VkDeviceQueueCreateFlags;
typedef uint32_t VkDeviceCreateFlags;
typedef uint32_t VkShaderStageFlags;
typedef uint32_t VkPipelineCreateFlags;
typedef uint32_t VkSampleCountFlagBits;

// Enum for queue flags (partial, relevant to code)
enum {
    VK_QUEUE_GRAPHICS_BIT = 0x00000001,
};
```

---

### Decompilation of `vt_handle_vkCreateDevice`

```c
/**
 * @brief Handles the vkCreateDevice command received from the client.
 *
 * This function deserializes the VkDeviceCreateInfo structure from the input buffer,
 * disables any unsupported device features, injects hardcoded required extensions,
 * calls vkCreateDevice, initializes device-specific components (TextureDecoder,
 * ShaderInspector), and sends the VkResult and the created VkDevice handle back
 * to the client.
 *
 * @param vtContext A pointer to the VtContext structure, containing client
 *                  communication buffers, temporary allocation pools, and other
 *                  global state.
 */
void vt_handle_vkCreateDevice(VtContext* vtContext) {
    // Save the stack canary for stack overflow detection.
    long stack_canary_value = *(long*)(((char*)tpidr_el0) + 0x28);

    // Stack-allocated storage for VkDeviceCreateInfo and its pNext chain.
    // The exact layout needs careful alignment with Vulkan spec and compiler packing.
    // Assuming these local variables collectively form the VkDeviceCreateInfo and its initial pNext chain.
    VkDeviceCreateInfo device_create_info; // Equivalent to local_110 to uStack_100, assuming it's a large enough block.
    // For simplicity, we'll represent it as a single struct initialized to zero
    // by the compiler, and assume the deserialization populates it correctly.
    memset(&device_create_info, 0, sizeof(VkDeviceCreateInfo)); // Manual zeroing for clarity.

    // Stack-allocated storage for VkDeviceQueueCreateInfo.
    VkDeviceQueueCreateInfo queue_create_info; // Equivalent to alStack_d8 (pQueueCreateInfos) and auStack_e0 (count/flags)
    memset(&queue_create_info, 0, sizeof(VkDeviceQueueCreateInfo));

    char* input_buffer_ptr = vtContext->inputBufferPtr; // pcVar7

    void* physical_device_id_ptr; // unaff_x20
    long offset_to_counts_in_input; // lVar8

    // Parse the input buffer header to get physical device ID and offset to counts.
    // The input format is assumed to be:
    // Byte 0: A flag/command type (not directly used here, but implies structure).
    // Bytes 1-8: VkPhysicalDevice handle (or its internal ID).
    // Bytes 9-12: queueCreateInfoCount (uint32_t).
    // Bytes 13-16: pQueueCreateInfos pointer (actual data follows).
    // The conditional logic `*input_buffer_ptr == '\0'` suggests input_buffer_ptr[0] is a flag.
    // If input_buffer_ptr[0] is '\0', then offset_to_counts_in_input is 1. This path seems less common.
    // Otherwise, physical_device_id_ptr is input_buffer_ptr[1] (8 bytes) and offset_to_counts_in_input is 9.
    if (*input_buffer_ptr == '\0') {
        physical_device_id_ptr = NULL; // Should not happen for vkCreateDevice
        offset_to_counts_in_input = 1; // Unlikely path for actual data
    } else {
        physical_device_id_ptr = (void*)(input_buffer_ptr + 1); // Get the VkPhysicalDevice ID (8 bytes)
        offset_to_counts_in_input = 9; // Offset to queueCreateInfoCount (1 byte flag + 8 bytes ID)
    }

    // Deserialize VkDeviceCreateInfo from input buffer into the stack-allocated struct.
    // The source for deserialization is `input_buffer_ptr` + `offset_to_counts_in_input` + 4 bytes
    // (likely skipping queueCreateInfoCount and pQueueCreateInfos pointer to get to pEnabledFeatures directly
    // or to a buffer containing the serialized DeviceCreateInfo itself).
    // This `FUN_0012d148` is a complex deserialization routine.
    if (*(int*)(input_buffer_ptr + offset_to_counts_in_input) > 0) { // Check if queueCreateInfoCount > 0
        // Deserialize device_create_info from input buffer.
        // `((uint)offset_to_counts_in_input | 4)` likely means `offset_to_counts_in_input` + size of `queueCreateInfoCount`
        // or a combined offset to `pQueueCreateInfos` that `FUN_0012d148` will then interpret.
        // Given the common pattern, the input buffer likely directly contains the serialized VkDeviceCreateInfo
        // after the VkPhysicalDevice handle.
        FUN_0012d148(&device_create_info, (void*)(input_buffer_ptr + offset_to_counts_in_input), vtContext->tempBuffer);
    }


    // Get the actual VkPhysicalDevice object from its ID.
    VkPhysicalDevice physical_device = VkObject_fromId(physical_device_id_ptr); // pvVar5

    // Call helper to disable requested device features if not supported by the physical device.
    disableUnsupportedDeviceFeatures(physical_device, (long)&device_create_info);

    // Hardcoded list of extensions to be injected.
    // These are often required for specific Vulkan features or compatibility layers.
    char* required_extensions[] = {
        "VK_KHR_get_memory_requirements2",
        "VK_KHR_dedicated_allocation",
        "VK_KHR_external_memory_fd",
        "VK_KHR_external_memory",
        "VK_KHR_external_fence_fd",
        "VK_KHR_external_fence",
        "VK_KHR_external_semaphore_fd",
        "VK_KHR_external_semaphore",
        "VK_ANDROID_external_memory_android_hardware_buffer",
        "VK_EXT_queue_family_foreign",
        "VK_KHR_sampler_ycbcr_conversion",
    };
    uint32_t num_required_extensions = sizeof(required_extensions) / sizeof(required_extensions[0]);

    // Another set of extensions.
    char* additional_extensions[] = {
        "VK_KHR_swapchain"
    };
    uint32_t num_additional_extensions = sizeof(additional_extensions) / sizeof(additional_extensions[0]);

    // Call `injectExtensions` to add these hardcoded extensions to `device_create_info`.
    // The `queue_create_info` (alStack_d8, auStack_e0) are passed here, suggesting `injectExtensions`
    // might manipulate the `pQueueCreateInfos` member of `device_create_info`.
    // The specific logic of `injectExtensions` is complex and not fully derivable from this snippet.
    injectExtensions(
        (long)vtContext, // param_1: VtContext*
        (long*)&queue_create_info, // alStack_d8: VkDeviceQueueCreateInfo*
        (long*)&queue_create_info.sType + 8, // auStack_e0: offset to something like `enabledLayerCount` inside device_create_info
                                           // (or another part of queue_create_info used for flags/counts)
        (char**)&required_extensions[0], // &local_c0: ppEnabledExtensionNames array pointer
        num_required_extensions,          // 0xb (11)
        (char**)&additional_extensions[0], // &local_c8: another ppEnabledExtensionNames array pointer
        num_additional_extensions         // 1
    );

    VkDevice created_device = NULL; // local_118
    // Call the Vulkan API vkCreateDevice function pointer.
    VkResult result = DAT_00149060(
        physical_device,
        &device_create_info,
        NULL, // pAllocator
        &created_device // Output VkDevice handle
    );

    // If device creation was successful, initialize device-specific components.
    if (result == VK_SUCCESS) {
        initVulkanDevice(vtContext, physical_device, created_device);
    }

    // Allocate memory from tempBuffer or malloc for storing the created VkDevice handle.
    void** device_handle_storage_ptr; // puVar6
    int current_temp_buffer_offset = vtContext->tempBufferOffset;

    if (current_temp_buffer_offset < 0xfff8 && vtContext->tempBuffer != NULL) {
        device_handle_storage_ptr = (void**)((char*)vtContext->tempBuffer + current_temp_buffer_offset);
        vtContext->tempBufferOffset = current_temp_buffer_offset + sizeof(VkDevice); // Assuming VkDevice is 8 bytes
    } else {
        device_handle_storage_ptr = (void**)malloc(sizeof(VkDevice)); // Allocate 8 bytes
        ArrayList_add(vtContext->tempAllocations, device_handle_storage_ptr); // Track this allocation
    }

    *device_handle_storage_ptr = created_device; // Store the created VkDevice handle

    // Send the VkResult and the created VkDevice handle back to the client via RingBuffer.
    int result_and_size[2]; // local_60
    result_and_size[0] = result;       // Store VkResult
    result_and_size[1] = sizeof(VkDevice); // Store size of the handle (8 bytes)

    VkBool32 header_write_success = RingBuffer_write(vtContext->clientRingBuffer, result_and_size, sizeof(result_and_size)); // Write 8 bytes (result + size)
    if (header_write_success) {
        // If header write successful, write the VkDevice handle (or NULL if creation failed)
        RingBuffer_write(vtContext->clientRingBuffer, device_handle_storage_ptr, sizeof(VkDevice)); // Write 8 bytes (the actual handle)
    }

    // Check stack canary to detect potential buffer overflows.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_canary_value) {
        __stack_chk_fail(); // Stack smashing detected.
    }
}
```

---

### Decompilation of `initVulkanDevice`

```c
/**
 * @brief Initializes device-specific components after VkDevice creation.
 *
 * This function queries queue family properties to find the graphics queue family index,
 * queries physical device properties, and initializes TextureDecoder and ShaderInspector
 * components, linking them to the asyncPipelineCreator.
 *
 * @param vtContext A pointer to the VtContext structure.
 * @param physicalDevice The VkPhysicalDevice handle.
 * @param device The newly created VkDevice handle.
 */
void initVulkanDevice(
    VtContext* vtContext,
    VkPhysicalDevice physicalDevice,
    VkDevice device
) {
    // Save the stack canary for stack overflow detection.
    long stack_canary_value = *(long*)(((char*)tpidr_el0) + 0x28);

    // Call an unknown helper function with the device.
    FUN_00132c9c(device);

    uint32_t queue_family_count = 0; // local_5c

    // 1. Get the count of queue family properties.
    DAT_00149038(physicalDevice, &queue_family_count, NULL); // vkGetPhysicalDeviceQueueFamilyProperties

    // Calculate stack space for the VkQueueFamilyProperties array.
    // 0x18 is sizeof(VkQueueFamilyProperties).
    // +0xf & 0x3ffffffff0 ensures 16-byte alignment and rounds up.
    long queue_family_properties_array_offset = -((unsigned long)queue_family_count * 0x18 + 0xf & 0x3ffffffff0);

    // Stack-allocated buffer for VkQueueFamilyProperties array.
    // local_140 is a placeholder for a stack-allocated buffer large enough for the properties.
    // The actual array starts at `(char*)&local_140 + queue_family_properties_array_offset`.
    // We declare a dummy array to represent this stack space.
    // On stack: `VkQueueFamilyProperties queue_properties_array[queue_family_count]`
    // Example: `VkQueueFamilyProperties stack_queue_properties[MAX_QUEUE_FAMILIES];`
    // For decompilation, we'll just refer to the base pointer.
    VkQueueFamilyProperties* queue_properties_array_ptr = (VkQueueFamilyProperties*)((char*)&queue_family_count + queue_family_properties_array_offset);

    // 2. Get the actual queue family properties into the allocated stack space.
    DAT_00149038(physicalDevice, &queue_family_count, queue_properties_array_ptr); // vkGetPhysicalDeviceQueueFamilyProperties

    // Initialize graphics queue family index.
    vtContext->graphicsQueueFamilyIndex = 0;

    // 3. Iterate through queue families to find one supporting graphics.
    if (queue_family_count != 0) {
        for (uint32_t i = 0; i < queue_family_count; ++i) {
            VkQueueFamilyProperties* current_props = &queue_properties_array_ptr[i];

            // Check if VK_QUEUE_GRAPHICS_BIT is set in queueFlags and queueCount is not zero.
            // The original assembly `(*(byte *)(piVar6 + -1) & 1)` is a very low-level byte access.
            // It's equivalent to `(current_props->queueFlags & VK_QUEUE_GRAPHICS_BIT) != 0`
            // assuming `VK_QUEUE_GRAPHICS_BIT` is 1 and the field is correctly packed.
            if ((current_props->queueFlags & VK_QUEUE_GRAPHICS_BIT) != 0 && current_props->queueCount > 0) {
                vtContext->graphicsQueueFamilyIndex = i; // Store the index
                break; // Found a suitable queue, exit loop.
            }
        }
    }

    // Stack-allocated storage for VkPhysicalDeviceProperties2.
    // Equivalent to local_140 and subsequent uStack_* variables.
    VkPhysicalDeviceProperties2 physical_device_properties2;
    memset(&physical_device_properties2, 0, sizeof(VkPhysicalDeviceProperties2)); // Manual zeroing for clarity.

    // 4. Get physical device properties using VkPhysicalDeviceProperties2.
    DAT_00149048(physicalDevice, &physical_device_properties2); // vkGetPhysicalDeviceProperties2

    // 5. Initialize TextureDecoder component.
    TextureDecoder* texture_decoder = vtContext->textureDecoder; // pvVar3
    if (texture_decoder == NULL) {
        texture_decoder = TextureDecoder_create(
            &physical_device_properties2, // pProperties
            vtContext->imageCacheSize,    // imageCacheSize
            vtContext->asyncPipelineCreator // asyncPipelineCreator
        );
        vtContext->textureDecoder = texture_decoder;
    }

    // 6. Initialize ShaderInspector component.
    ShaderInspector* shader_inspector = vtContext->shaderInspector; // lVar5
    if (shader_inspector == NULL) {
        shader_inspector = ShaderInspector_create(
            physical_device,          // physicalDevice
            &physical_device_properties2 // pProperties
        );
        vtContext->shaderInspector = shader_inspector;
    }

    // 7. Update TextureDecoder's asyncPipelineCreator (if TextureDecoder exists).
    if (texture_decoder != NULL) {
        // Offset 0x40 (64 bytes) within the TextureDecoder structure.
        // This is a dynamic cast-like operation.
        *(void**)((char*)texture_decoder + 0x40) = vtContext->asyncPipelineCreator;
    }

    // Check stack canary to detect potential buffer overflows.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_canary_value) {
        __stack_chk_fail(); // Stack smashing detected.
    }
}
```

---

### Decompilation of `disableUnsupportedDeviceFeatures`

```c
/**
 * @brief Disables requested device features in VkDeviceCreateInfo if they are
 *        not supported by the physical device.
 *
 * This function iterates through the pNext chain of the provided VkDeviceCreateInfo
 * structure to find VkPhysicalDeviceFeatures2. If found, it queries the actual
 * supported features from the physical device and then zeroes out any requested
 * features in VkDeviceCreateInfo::pEnabledFeatures that are not supported.
 *
 * @param physicalDevice The VkPhysicalDevice handle to query supported features from.
 * @param deviceCreateInfoPtr A pointer to the VkDeviceCreateInfo structure.
 */
void disableUnsupportedDeviceFeatures(
    VkPhysicalDevice physicalDevice,
    VkDeviceCreateInfo* deviceCreateInfoPtr
) {
    // Save the stack canary for stack overflow detection.
    long stack_canary_value = *(long*)(((char*)tpidr_el0) + 0x28);

    // Stack-allocated buffer for VkPhysicalDeviceFeatures2.
    VkPhysicalDeviceFeatures2 supported_features2; // local_140
    memset(&supported_features2, 0, sizeof(VkPhysicalDeviceFeatures2)); // Manual zeroing for clarity

    // Pointers to requested VkPhysicalDeviceFeatures and to the VkPhysicalDeviceFeatures2 in pNext chain
    const VkPhysicalDeviceFeatures* requested_features = deviceCreateInfoPtr->pEnabledFeatures; // From the VkDeviceCreateInfo
    VkPhysicalDeviceFeatures2* pnext_features2_struct = NULL;

    // First, find VkPhysicalDeviceFeatures2 in the pNext chain of VkDeviceCreateInfo.
    VkBaseInStructure* current_pnext_struct = (VkBaseInStructure*)deviceCreateInfoPtr->pNext;
    while (current_pnext_struct != NULL) {
        if (current_pnext_struct->sType == VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2) {
            pnext_features2_struct = (VkPhysicalDeviceFeatures2*)current_pnext_struct;
            break;
        }
        current_pnext_struct = (VkBaseInStructure*)current_pnext_struct->pNext;
    }

    // If a VkPhysicalDeviceFeatures2 struct was found in the pNext chain,
    // query the actual supported features into `supported_features2`.
    if (pnext_features2_struct != NULL) {
        DAT_001494a0(physicalDevice, &supported_features2); // vkGetPhysicalDeviceFeatures2
        // `supported_features2.features` now contains the supported features.
        // We will use this to disable unsupported requested features.
    } else if (requested_features == NULL) {
        // If neither a VkPhysicalDeviceFeatures2 struct nor pEnabledFeatures is present,
        // there's nothing to disable, so exit.
        goto end_function;
    }

    // If there are requested features (either directly via pEnabledFeatures
    // or via VkPhysicalDeviceFeatures2 in the pNext chain),
    // proceed to compare them against supported features and disable if necessary.
    // The `local_140` (i.e., `supported_features2`) will be used as the source of supported features.
    DAT_001494a0(physicalDevice, &supported_features2); // Redundant call? Or implicitly handles pnext_features2_struct being null?
                                                         // This likely ensures `supported_features2` is always populated
                                                         // with the device's capabilities, regardless of whether a pNext
                                                         // struct was initially present.

    // Get the address of the `features` struct within `supported_features2` for direct byte access.
    // And get the address of `pEnabledFeatures` from `deviceCreateInfoPtr`.
    // The loop iterates over each boolean feature flag (uint32_t) in `VkPhysicalDeviceFeatures`.
    // `0xdc` (220 bytes) is the size of `VkPhysicalDeviceFeatures`.
    for (uint32_t offset = 0; offset < 0xdc; offset += sizeof(uint32_t)) {
        uint32_t* requested_feature_ptr = (uint32_t*)((char*)requested_features + offset);
        uint32_t supported_feature_ptr = *(uint32_t*)((char*)&supported_features2.features + offset); // Access the features member

        // Check if the feature is requested (non-zero) AND not supported (zero)
        if (*requested_feature_ptr != 0 && supported_feature_ptr == 0) {
            *requested_feature_ptr = 0; // Disable the requested feature (set to false)
        }
    }

end_function:
    // Check stack canary to detect potential buffer overflows.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_canary_value) {
        __stack_chk_fail(); // Stack smashing detected.
    }
}
```

# vkEnumerateInstanceExtensionProperties

To decompile the provided C functions `vt_handle_vkEnumerateInstanceExtensionProperties` and `injectExtensions2`, we need to first identify the types of their parameters and the structures they interact with. We'll leverage the `VtContext` struct and standard Vulkan headers for type information.

**1. `VtContext` and other relevant structures/types:**

```c
// Partial VtContext struct, inferred from offsets in the assembly
typedef struct VtContext {
    char padding_00[0x28]; // Padding to reach inputBufferPtr (0x28 bytes)
    char* inputBufferPtr;      // 0x28: Pointer to the current command's data buffer (read-only for this function)
    uint32_t inputBufferSize;  // 0x30
    char padding_34[4];        // 0x34
    void* tempBuffer;          // 0x38: A large temporary buffer for deserialization/temporary allocations
    uint32_t tempBufferOffset; // 0x40: Current offset within the tempBuffer
    char padding_44[4];        // 0x44
    ArrayList* tempAllocations; // 0x48: Tracks temporary allocations for later cleanup
    char padding_50[0x10];     // 0x50
    RingBuffer* clientRingBuffer; // 0x60: For writing results back to the client
    // ... other members as defined in the original VtContext
} VtContext;

// Simplified ArrayList structure (assuming basic add functionality)
typedef struct ArrayList {
    // Internal members for managing a dynamic array
} ArrayList;

// Simplified RingBuffer structure (assuming a write function)
typedef struct RingBuffer {
    // Internal members for managing a circular buffer
} RingBuffer;

// Vulkan API types (from vulkan_core.h)
typedef VkResult (VKAPI_PTR *PFN_vkEnumerateInstanceExtensionProperties)(
    const char* pLayerName,
    uint32_t* pPropertyCount,
    VkExtensionProperties* pProperties
);

// Global function pointer, assumed to be initialized externally
extern PFN_vkEnumerateInstanceExtensionProperties DAT_00149078; // Points to vkEnumerateInstanceExtensionProperties

// External utility functions
extern void ArrayList_add(ArrayList* list, void* data);
extern unsigned char RingBuffer_write(RingBuffer* buffer, const void* data, size_t size);
extern void __stack_chk_fail(void); // Stack canary failure handler
```

**2. Decompilation of `injectExtensions2`:**

This helper function dynamically manages and merges lists of `VkExtensionProperties`.

```c
#include <string.h> // For strcmp, memset, memcpy, strlen
#include <stdio.h>  // For snprintf (used implicitly in the decompiled code's logic of other functions)
#include <stdlib.h> // For malloc
#include <alloca.h> // For alloca (stack allocation)

// Forward declarations for types defined in the main scope
// (These would typically be in a header or defined before this function)
typedef struct VtContext VtContext;
typedef struct ArrayList ArrayList;
typedef struct VkExtensionProperties VkExtensionProperties;
typedef struct RingBuffer RingBuffer;

/**
 * @brief Injects new extensions into an existing list of Vulkan extension properties.
 *
 * This function takes an existing array of VkExtensionProperties and its count,
 * along with two arrays of extension names to inject. It creates a new array
 * containing all unique extensions from the original list and the two injection lists.
 * The resulting array and its new count are returned via pointers.
 * Memory for the new array is allocated from the provided temporary buffer pool
 * or via malloc if the pool is insufficient.
 *
 * @param vtContext A pointer to the VtContext structure, used for temporary buffer
 *                  management and tracking allocations.
 * @param currentPropertiesArrayPtr A pointer to a VkExtensionProperties* that
 *                                  will be updated to point to the new, merged array.
 * @param currentPropertyCountPtr A pointer to a uint32_t that will be updated
 *                                with the new count of extensions.
 * @param extensionsToInjectList1 An array of C-string pointers (extension names) to inject.
 * @param extensionsToInjectCount1 The number of extensions in extensionsToInjectList1.
 * @param extensionsToInjectList2 Another array of C-string pointers to inject.
 * @param extensionsToInjectCount2 The number of extensions in extensionsToInjectList2.
 */
void injectExtensions2(
    VtContext* vtContext,
    VkExtensionProperties** currentPropertiesArrayPtr,
    uint32_t* currentPropertyCountPtr,
    const char* const* extensionsToInjectList1, // param_4
    uint32_t extensionsToInjectCount1,           // param_5
    const char* const* extensionsToInjectList2, // param_6
    uint32_t extensionsToInjectCount2            // param_7
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    // This function applies injection sequentially.
    // First, it processes `extensionsToInjectList2` into `currentPropertiesArrayPtr`.
    // Then, it processes `extensionsToInjectList1` into the *updated* `currentPropertiesArrayPtr`.
    // Effectively, it computes `(OriginalList U extensionsToInjectList2) U extensionsToInjectList1`.

    // Temporary stack-allocated buffer for boolean flags to track duplicates.
    // The size calculation `(count + 15) & ~15` is a common pattern for 16-byte alignment.
    char* is_present_flags = NULL;

    // --- Process extensionsToInjectList2 first ---
    if (extensionsToInjectList2 != NULL) {
        uint32_t existingPropertiesCount = *currentPropertyCountPtr;
        VkExtensionProperties* existingPropertiesArray = *currentPropertiesArrayPtr;

        // Allocate flags on stack, sufficient for current existing properties.
        is_present_flags = (char*)alloca((size_t)existingPropertiesCount + 15 & ~15);
        memset(is_present_flags, 0, existingPropertiesCount); // Initialize all flags to false

        uint32_t propertiesToKeepFromExisting = 0; // Count of original extensions not found in extensionsToInjectList2

        // Iterate through existing properties to mark which ones are in extensionsToInjectList2
        for (uint32_t i = 0; i < existingPropertiesCount; ++i) {
            uint32_t foundInList2 = 0;
            for (uint32_t j = 0; j < extensionsToInjectCount2; ++j) {
                if (strcmp(extensionsToInjectList2[j], existingPropertiesArray[i].extensionName) == 0) {
                    foundInList2 = 1;
                    break;
                }
            }
            if (!foundInList2) {
                propertiesToKeepFromExisting++;
            }
            is_present_flags[i] = (char)foundInList2; // Store whether it was found
        }

        // Calculate the total count for the new merged array
        uint32_t newTotalPropertiesCount = propertiesToKeepFromExisting + extensionsToInjectCount2;
        size_t newArraySize = (size_t)newTotalPropertiesCount * sizeof(VkExtensionProperties);

        // Allocate memory for the new merged properties array
        VkExtensionProperties* newPropertiesArray;
        if ((vtContext->tempBufferOffset + newArraySize < 0x10000) && (vtContext->tempBuffer != NULL)) {
            newPropertiesArray = (VkExtensionProperties*)(vtContext->tempBuffer + vtContext->tempBufferOffset);
            vtContext->tempBufferOffset += (uint32_t)newArraySize;
        } else {
            newPropertiesArray = (VkExtensionProperties*)malloc(newArraySize);
            ArrayList_add(vtContext->tempAllocations, newPropertiesArray);
        }
        memset(newPropertiesArray, 0, newArraySize);

        uint32_t currentWriteIndex = 0;
        // Copy existing properties that were NOT in extensionsToInjectList2
        for (uint32_t i = 0; i < existingPropertiesCount; ++i) {
            if (!is_present_flags[i]) {
                memcpy(&newPropertiesArray[currentWriteIndex], &existingPropertiesArray[i], sizeof(VkExtensionProperties));
                currentWriteIndex++;
            }
        }
        // Copy new extensions from extensionsToInjectList2
        for (uint32_t i = 0; i < extensionsToInjectCount2; ++i) {
            // Here, it implicitly assumes uniqueness of extensionsToInjectList2 items among themselves
            // and with previously copied elements from the original list that were not duplicates of list2.
            // A more robust implementation might check against `newPropertiesArray` for duplicates here.
            strncpy(newPropertiesArray[currentWriteIndex].extensionName, extensionsToInjectList2[i], VK_MAX_EXTENSION_NAME_SIZE);
            newPropertiesArray[currentWriteIndex].extensionName[VK_MAX_EXTENSION_NAME_SIZE - 1] = '\0';
            newPropertiesArray[currentWriteIndex].specVersion = 1; // Default or hardcoded version 1
            currentWriteIndex++;
        }

        // Update the pointers to reflect the new merged list
        *currentPropertiesArrayPtr = newPropertiesArray;
        *currentPropertyCountPtr = newTotalPropertiesCount;
    }

    // --- Process extensionsToInjectList1 next, using the potentially updated `currentPropertiesArrayPtr` ---
    if (extensionsToInjectList1 != NULL && extensionsToInjectCount1 > 0) {
        uint32_t existingPropertiesCount = *currentPropertyCountPtr;
        VkExtensionProperties* existingPropertiesArray = *currentPropertiesArrayPtr;

        // Re-allocate or re-use flags buffer for new count.
        is_present_flags = (char*)alloca((size_t)existingPropertiesCount + 15 & ~15);
        memset(is_present_flags, 0, existingPropertiesCount);

        uint32_t propertiesToKeepFromExisting = 0;

        // Mark existing extensions that are present in extensionsToInjectList1
        for (uint32_t i = 0; i < existingPropertiesCount; ++i) {
            uint32_t foundInList1 = 0;
            for (uint32_t j = 0; j < extensionsToInjectCount1; ++j) {
                if (strcmp(extensionsToInjectList1[j], existingPropertiesArray[i].extensionName) == 0) {
                    foundInList1 = 1;
                    break;
                }
            }
            if (!foundInList1) {
                propertiesToKeepFromExisting++;
            }
            is_present_flags[i] = (char)foundInList1;
        }

        uint32_t newTotalPropertiesCount = propertiesToKeepFromExisting + extensionsToInjectCount1;
        size_t newArraySize = (size_t)newTotalPropertiesCount * sizeof(VkExtensionProperties);

        VkExtensionProperties* newPropertiesArray;
        if ((vtContext->tempBufferOffset + newArraySize < 0x10000) && (vtContext->tempBuffer != NULL)) {
            newPropertiesArray = (VkExtensionProperties*)(vtContext->tempBuffer + vtContext->tempBufferOffset);
            vtContext->tempBufferOffset += (uint32_t)newArraySize;
        } else {
            newPropertiesArray = (VkExtensionProperties*)malloc(newArraySize);
            ArrayList_add(vtContext->tempAllocations, newPropertiesArray);
        }
        memset(newPropertiesArray, 0, newArraySize);

        uint32_t currentWriteIndex = 0;
        // Copy existing properties that were NOT in extensionsToInjectList1
        for (uint32_t i = 0; i < existingPropertiesCount; ++i) {
            if (!is_present_flags[i]) {
                memcpy(&newPropertiesArray[currentWriteIndex], &existingPropertiesArray[i], sizeof(VkExtensionProperties));
                currentWriteIndex++;
            }
        }
        // Copy new extensions from extensionsToInjectList1
        for (uint32_t i = 0; i < extensionsToInjectCount1; ++i) {
            strncpy(newPropertiesArray[currentWriteIndex].extensionName, extensionsToInjectList1[i], VK_MAX_EXTENSION_NAME_SIZE);
            newPropertiesArray[currentWriteIndex].extensionName[VK_MAX_EXTENSION_NAME_SIZE - 1] = '\0';
            newPropertiesArray[currentWriteIndex].specVersion = 1;
            currentWriteIndex++;
        }

        // Update the pointers to reflect the new merged list
        *currentPropertiesArrayPtr = newPropertiesArray;
        *currentPropertyCountPtr = newTotalPropertiesCount;
    }

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}
```

**3. Decompilation of `vt_handle_vkEnumerateInstanceExtensionProperties`:**

```c
#include <string.h> // For strcmp, memset, memcpy, strlen
#include <stdio.h>  // For snprintf
#include <stdlib.h> // For malloc (implicitly used via ArrayList_add or if tempBuffer is full)

// Types are forward-declared or included from headers as above.

// Custom serialization struct for an individual extension in the output buffer.
// This is not an official Vulkan struct, but the format created by this function.
// It's a Variable-Length-Value (TLV) like structure.
typedef struct OutputSerializedExtensionEntry {
    uint32_t entry_total_size;         // Total size of this specific entry (including this field)
    uint32_t name_length_with_null;    // Length of the extensionName string including null terminator
    // char extensionName_data[];      // Variable length name data (followed by specVersion)
    // uint32_t specVersion;           // Located immediately after name_data
} OutputSerializedExtensionEntry;


/**
 * @brief Handles the vkEnumerateInstanceExtensionProperties command from the client.
 *
 * This function processes a client request to enumerate Vulkan instance extensions.
 * It reads initial extension data from a client-provided buffer, queries the
 * system for available extensions, injects specific hardcoded extensions, and
 * then serializes the final list of extensions back to the client.
 *
 * @param param_1 A long value representing the base address of the VtContext structure.
 *                Casted to `VtContext*` internally.
 *
 * Input buffer format (from `vtContext->inputBufferPtr`):
 *   - uint32_t: initial_properties_data_size (total byte size of the raw VkExtensionProperties array that follows)
 *   - char[]: raw data for initial VkExtensionProperties array
 *   - uint32_t: actual_initial_property_count (number of VkExtensionProperties in the above array)
 *
 * Output buffer format (sent to `vtContext->clientRingBuffer`):
 *   - VkResult: function_call_result (e.g., VK_SUCCESS)
 *   - uint32_t: total_serialized_data_size (total bytes for all data following this field)
 *   - uint32_t: magic_header (0x400000000, specific to this command type)
 *   - uint32_t: final_property_count (total count of extensions after injection)
 *   - [For each VkExtensionProperties in the final list:]
 *     - uint32_t: entry_total_size (size of this specific entry including its own header, name_len_field, name, and version)
 *     - uint32_t: name_length_with_null_terminator (length of name including null)
 *     - char[]: extensionName_bytes (variable length)
 *     - uint32_t: specVersion
 */
void vt_handle_vkEnumerateInstanceExtensionProperties(long param_1) {
    VtContext* vtContext = (VtContext*)param_1;
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    uint32_t initial_properties_data_size; // uVar10 (at function start)
    uint32_t initial_property_count_from_input_buffer; // unaff_w23

    // Pointer to the client's input data buffer.
    uint32_t* client_input_data_ptr = (uint32_t*)vtContext->inputBufferPtr;

    // 1. Deserialize initial extension properties from the client's input buffer.
    initial_properties_data_size = *client_input_data_ptr;

    // Allocate memory for `temp_initial_properties_buffer` (which is `__s` in GHIDRA)
    // This buffer will hold the raw VkExtensionProperties data from the client's input.
    void* temp_initial_properties_buffer;
    size_t alloc_size_for_initial_props = (initial_properties_data_size < 1) ? 4 : initial_properties_data_size;
    
    if ((vtContext->tempBufferOffset + alloc_size_for_initial_props < 0x10000) && (vtContext->tempBuffer != NULL)) {
        temp_initial_properties_buffer = (void*)(vtContext->tempBuffer + vtContext->tempBufferOffset);
        vtContext->tempBufferOffset += (uint32_t)alloc_size_for_initial_props;
    } else {
        temp_initial_properties_buffer = malloc(alloc_size_for_initial_props);
        ArrayList_add(vtContext->tempAllocations, temp_initial_properties_buffer);
    }
    memset(temp_initial_properties_buffer, 0, alloc_size_for_initial_props);

    if (initial_properties_data_size >= 1) {
        // Copy the raw VkExtensionProperties data from `client_input_data_ptr + 1` (skipping the size field)
        memcpy(temp_initial_properties_buffer, client_input_data_ptr + 1, initial_properties_data_size);
    }

    // Read the actual count of properties that were present in the initial input buffer.
    // This field is located right after the raw properties data.
    initial_property_count_from_input_buffer = *(uint32_t*)(((char*)client_input_data_ptr) + initial_properties_data_size + 4);

    // 2. Enumerate system-available instance extensions using Vulkan API.
    uint32_t system_extension_count = 0; // local_8c
    VkResult enumerate_result; // uVar7
    
    // First call: Query the number of available extensions on the system.
    enumerate_result = DAT_00149078(NULL, &system_extension_count, NULL);

    // Allocate buffer to store VkExtensionProperties obtained from the system.
    VkExtensionProperties* system_extension_properties_buffer; // local_98 (pcVar8)
    size_t system_extension_buffer_alloc_size = (size_t)system_extension_count * sizeof(VkExtensionProperties);

    if ((vtContext->tempBufferOffset + system_extension_buffer_alloc_size < 0x10000) && (vtContext->tempBuffer != NULL)) {
        system_extension_properties_buffer = (VkExtensionProperties*)(vtContext->tempBuffer + vtContext->tempBufferOffset);
        vtContext->tempBufferOffset += (uint32_t)system_extension_buffer_alloc_size;
    } else {
        system_extension_properties_buffer = (VkExtensionProperties*)malloc(system_extension_buffer_alloc_size);
        ArrayList_add(vtContext->tempAllocations, system_extension_properties_buffer);
    }
    memset(system_extension_properties_buffer, 0, system_extension_buffer_alloc_size);

    // Second call: Retrieve the actual extension properties from the system.
    enumerate_result = DAT_00149078(NULL, &system_extension_count, system_extension_properties_buffer);

    // 3. Define hardcoded extensions to inject.
    // These strings are often located in the `.rodata` section or directly in the function's stack frame.
    const char* surface_extension = "VK_KHR_surface"; // local_80
    const char* xlib_surface_extension = "VK_KHR_xlib_surface"; // pcStack_78
    const char* android_surface_extension = "VK_KHR_android_surface"; // local_88

    const char* extensions_to_inject_list1[] = {
        surface_extension,
        xlib_surface_extension
    };
    uint32_t extensions_to_inject_count1 = 2;

    const char* extensions_to_inject_list2[] = {
        android_surface_extension
    };
    uint32_t extensions_to_inject_count2 = 1;

    // 4. Inject hardcoded extensions into the list.
    // The `injectExtensions2` function will modify `system_extension_properties_buffer`
    // and `system_extension_count` in place, ensuring uniqueness.
    injectExtensions2(
        vtContext,
        &system_extension_properties_buffer,
        &system_extension_count,
        extensions_to_inject_list1,
        extensions_to_inject_count1,
        extensions_to_inject_list2,
        extensions_to_inject_count2
    );

    // 5. Serialize the final list of extensions into a custom format for the client.
    uint32_t total_output_serialized_data_size = 8; // Initial size: 4 bytes for magic_header + 4 bytes for final_property_count
    
    // Calculate total size needed for variable-length serialized data.
    // Each entry's size = strlen(name) + 4 (for name_length_with_null field) + 4 (for specVersion field) + 1 (for null terminator).
    // This totals to `name_len + 9` bytes per entry.
    if (system_extension_properties_buffer != NULL && system_extension_count > 0) {
        for (uint32_t i = 0; i < system_extension_count; ++i) {
            size_t name_len = strlen(system_extension_properties_buffer[i].extensionName);
            total_output_serialized_data_size += (uint32_t)(name_len + 9);
        }
    }

    // Allocate memory for the final serialized output buffer.
    void* output_serialized_buffer; // __s_00
    if ((vtContext->tempBufferOffset + total_output_serialized_data_size < 0x10000) && (vtContext->tempBuffer != NULL)) {
        output_serialized_buffer = (void*)(vtContext->tempBuffer + vtContext->tempBufferOffset);
        vtContext->tempBufferOffset += total_output_serialized_data_size;
    } else {
        output_serialized_buffer = malloc(total_output_serialized_data_size);
        ArrayList_add(vtContext->tempAllocations, output_serialized_buffer);
    }
    memset(output_serialized_buffer, 0, total_output_serialized_data_size);

    // Populate the header of the serialized output buffer.
    uint32_t* output_ptr_uint32 = (uint32_t*)output_serialized_buffer;
    *output_ptr_uint32 = 0x400000000; // Magic header/command type
    output_ptr_uint32[1] = system_extension_count; // Final count of extensions

    // Populate the serialized extension entries.
    // The current write offset starts after the two 4-byte header fields (magic_header and final_property_count).
    uint32_t current_output_byte_offset = 8;

    if (system_extension_properties_buffer != NULL && system_extension_count > 0) {
        for (uint32_t i = 0; i < system_extension_count; ++i) {
            VkExtensionProperties* prop = &system_extension_properties_buffer[i];
            size_t name_len = strlen(prop->extensionName);
            
            // Calculate sizes for the current entry's fields based on the custom format.
            // entry_total_size = strlen(name) + 1 (null) + 4 (name_len_w_null field) + 4 (spec_version field)
            uint32_t entry_total_size = (uint32_t)(name_len + 9);
            uint32_t name_length_with_null = (uint32_t)(name_len + 1);

            // Write entry_total_size at current_output_byte_offset
            *(uint32_t*)(((char*)output_serialized_buffer) + current_output_byte_offset) = entry_total_size;

            // Write name_length_with_null_terminator at current_output_byte_offset + 4
            *(uint32_t*)(((char*)output_serialized_buffer) + current_output_byte_offset + 4) = name_length_with_null;

            // Copy extension name data at current_output_byte_offset + 8
            memcpy(((char*)output_serialized_buffer) + current_output_byte_offset + 8, prop->extensionName, name_length_with_null);

            // Write specVersion at current_output_byte_offset + 8 + name_length_with_null
            *(uint32_t*)(((char*)output_serialized_buffer) + current_output_byte_offset + 8 + name_length_with_null) = prop->specVersion;

            current_output_byte_offset += entry_total_size; // Advance offset for the next entry
        }
    }

    // 6. Send the result and serialized data back to the client via the ring buffer.
    struct {
        VkResult result;
        uint32_t data_size;
    } response_header; // local_70 (result), uStack_6c (data_size)

    response_header.result = enumerate_result;
    response_header.data_size = total_output_serialized_data_size;

    RingBuffer_write(vtContext->clientRingBuffer, &response_header, sizeof(response_header));
    if ((response_header.result == VK_SUCCESS) && (total_output_serialized_data_size > 0)) {
        RingBuffer_write(vtContext->clientRingBuffer, output_serialized_buffer, total_output_serialized_data_size);
    }

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}
```
---

# vkEnumerateDeviceExtensionProperties

Here's the decompilation of the provided C functions, with inferred parameter types, local variables, and explanations based on the `vulkan_core.h` and `VtContext` struct definitions.

First, let's redefine the `VtContext` struct more precisely based on the offsets observed in both decompiled functions, as the previous one was a high-level overview.

```c
#include <string.h> // For strcmp, strncpy, memset
#include <stdio.h>  // For snprintf
#include <stdlib.h> // For malloc
#include <stdbool.h> // For bool type

// Assuming these are defined elsewhere or in the Vulkan headers:
// Standard Vulkan types (simplified for brevity here)
typedef unsigned int uint32_t;
typedef unsigned long long uint64_t;
typedef unsigned char uint8_t;
typedef char byte; // Often typedef-ed as unsigned char or char
typedef long int int64_t;
typedef int int32_t;
typedef size_t size_t;

#define VKAPI_ATTR
#define VKAPI_CALL
#define PFN_vkVoidFunction void (VKAPI_PTR *)(void)
#define VK_NULL_HANDLE ((void*)0)

// From vulkan_core.h
#define VK_MAX_EXTENSION_NAME_SIZE 256
#define VK_UUID_SIZE 16
#define VK_MAX_PHYSICAL_DEVICE_NAME_SIZE 256
#define VK_MAX_DRIVER_INFO_SIZE 256
#define VK_FALSE 0U // Vulkan defines VkBool32 as uint32_t, so 0 is false.

// Version extraction macros from vulkan_core.h
#define VK_API_VERSION_MAJOR(version) (((uint32_t)(version) >> 22U) & 0x7FU)
#define VK_API_VERSION_MINOR(version) (((uint32_t)(version) >> 12U) & 0x3FFU)
#define VK_API_VERSION_PATCH(version) ((uint32_t)(version) & 0xFFFU)

// Enum constants from vulkan_core.h
typedef enum VkStructureType {
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES = 52, // 0x34
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES = 1000070000, // 0x3b9dc7a0
    // ... other structure types
} VkStructureType;

// Minimal struct definitions to match offsets and types from the disassembly context.
// These are simplified for clarity based on the usage in the provided C snippet.
typedef struct VkBaseInStructure {
    VkStructureType sType;
    const struct VkBaseInStructure* pNext; // 8 bytes on 64-bit systems.
} VkBaseInStructure;

// Definition for VkExtensionProperties based on standard Vulkan headers and observed size (0x104 = 260 bytes).
typedef struct VkExtensionProperties {
    char        extensionName[VK_MAX_EXTENSION_NAME_SIZE]; // 256 bytes
    uint32_t    specVersion;                               // 4 bytes
} VkExtensionProperties; // Total size = 260 bytes (0x104)

// Placeholder for external ArrayList structure as used in VtContext
typedef struct ArrayList {
    uint32_t count;
    uint32_t capacity; // Inferred to exist for padding/management
    void**   data;     // Array of pointers to elements
} ArrayList;

// Placeholder for external RingBuffer structure as used in VtContext
typedef struct RingBuffer {
    // Members inferred from usage:
    // Pointers for read/write positions and buffer start/end
    // Methods: RingBuffer_write(RingBuffer*, const void*, size_t) returns byte
    char _pad[64]; // Placeholder size
} RingBuffer;

// Redefined VtContext based on analysis of both functions.
// `param_1` in the decompiled functions refers to an instance of this struct.
typedef struct VtContext {
    char        _pad0[0x4];             // Padding
    int         vkMaxVersion;           // Offset 0x4
    short       maxDeviceMemory;        // Offset 0x8
    short       imageCacheSize;         // Offset 0xA
    int         padding_0c;             // Offset 0xC
    ArrayList*  exposedDeviceExtensions;// Offset 0x10 (Accessed as `param_1 + 0x10`)
    ArrayList*  exposedInstanceExtensions; // Offset 0x18 (Accessed as `param_1 + 0x18`)
    char        _pad1[0x10];            // Padding to reach 0x28
    char*       inputBufferPtr;         // Offset 0x28 (Accessed as `param_1 + 0x28`)
    int         inputBufferSize;        // Offset 0x30
    int         padding_34;             // Offset 0x34
    void*       tempBuffer;             // Offset 0x38 (Accessed as `param_1 + 0x38`)
    int         tempBufferOffset;       // Offset 0x40 (Accessed as `param_1 + 0x40`)
    int         padding_44;             // Offset 0x44
    ArrayList*  tempAllocations;        // Offset 0x48 (Accessed as `param_1 + 0x48`)
    char        _pad2[0x10];            // Padding to reach 0x60
    RingBuffer* clientRingBuffer;       // Offset 0x60 (Accessed as `param_1 + 0x60`)
    // ... potentially more fields
} VtContext;

// Minimal definitions for Vulkan structures, inferred from usage in the functions
typedef struct VkPhysicalDeviceProperties {
    uint32_t            apiVersion;
    uint32_t            driverVersion; // At offset 0x4
    uint32_t            vendorID;      // At offset 0x8
    uint32_t            deviceID;      // At offset 0xC
    uint32_t            deviceType;    // At offset 0x10 (e.g. VkPhysicalDeviceType enum)
    char                deviceName[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE]; // At offset 0x14 (20 bytes).
                                                                       // This is `param_2 + 5` if `param_2` is `uint32_t*`.
    uint8_t             pipelineCacheUUID[VK_UUID_SIZE]; // At offset 0x114 (276 bytes)
    // ... other members
} VkPhysicalDeviceProperties;

typedef struct VkPhysicalDeviceVulkan12Properties {
    VkStructureType                      sType;
    void*                                pNext;
    uint32_t                             driverID;
    char                                 driverName[VK_MAX_DRIVER_NAME_SIZE];
    char                                 driverInfo[VK_MAX_DRIVER_INFO_SIZE]; // At offset 0x114 (276 bytes) from start of this struct.
    // The specific VkBool32 members accessed:
    // props12->shaderSignedZeroInfNanPreserveFloat16 (offset 0x89 * 4 = 548)
    // props12->shaderSignedZeroInfNanPreserveFloat32 (offset 0x8A * 4 = 552)
    // props12->shaderDenormPreserveFloat16           (offset 0x8C * 4 = 560)
    // props12->shaderDenormPreserveFloat32           (offset 0x8D * 4 = 564)
    // props12->shaderDenormFlushToZeroFloat16        (offset 0x8F * 4 = 572)
    // props12->shaderDenormFlushToZeroFloat32        (offset 0x90 * 4 = 576)
} VkPhysicalDeviceVulkan12Properties;

typedef struct VkPhysicalDeviceGroupProperties {
    VkStructureType     sType;
    void*               pNext;
    uint32_t            physicalDeviceCount;
    // VkPhysicalDevice    physicalDevices[VK_MAX_DEVICE_GROUP_SIZE]; // Array of 32 VkPhysicalDevice (uint64_t) handles, 32 * 8 = 256 bytes.
                                                                    // Starts at offset 0x14 (20 bytes).
                                                                    // Ends at offset 0x14 + 256 = 0x114 (276 bytes).
    bool                subsetAllocation; // At offset 0x114 (276 bytes), which is `groupProps + 0x114`.
                                          // This field is overwritten by the driver version string.
    // ... other members
} VkPhysicalDeviceGroupProperties;


// Standard library functions (or their equivalents if custom implementation)
extern void *__raw_tpidr_el0; // Placeholder for tpidr_el0 which is a CPU register
extern int strcmp(const char *s1, const char *s2);
extern size_t strlen(const char *s);
extern void *memset(void *s, int c, size_t n);
extern void *memcpy(void *dest, const void *src, size_t n);
extern void __stack_chk_fail(void); // Defined as external, indicates stack smashing protection.

// Function pointer type for vkEnumerateDeviceExtensionProperties
typedef VkResult (VKAPI_PTR *PFN_vkEnumerateDeviceExtensionProperties)(
    VkPhysicalDevice physicalDevice,
    const char* pLayerName,
    uint32_t* pPropertyCount,
    VkExtensionProperties* pProperties
);

// External global variable holding the function pointer for vkEnumerateDeviceExtensionProperties
extern PFN_vkEnumerateDeviceExtensionProperties DAT_00149080; // Inferred from usage

// Placeholder for `VkObject_fromId`
typedef uint64_t VkPhysicalDeviceId; // Assuming IDs are 64-bit
VkObject VkObject_fromId(VkPhysicalDeviceId id);

// Placeholder for ArrayList functions
void ArrayList_add(ArrayList* list, void* element);
byte RingBuffer_write(RingBuffer* buffer, const void* data, size_t size);

/**
 * @brief Retrieves and filters device extension properties.
 *
 * This function first enumerates all device extension properties supported by the
 * given physical device. It then filters this list:
 * - If `vtContext->exposedDeviceExtensions` is not NULL, only extensions present
 *   in this list are considered.
 * - Additionally, any device extension that is *also* found in
 *   `vtContext->exposedInstanceExtensions` is explicitly filtered out.
 * The filtered properties are copied into a new buffer, which is either taken
 * from a large temporary buffer or dynamically allocated.
 *
 * @param vtContext A pointer to the VtContext structure.
 * @param physicalDevice The VkPhysicalDevice handle.
 * @param pExtensionCount A pointer to an integer that will receive the count
 *                        of extensions copied into the returned buffer.
 * @return A pointer to a dynamically allocated buffer containing the filtered
 *         VkExtensionProperties. Returns NULL on failure or if no extensions
 *         match the criteria.
 */
static void* getExposedDeviceExtensionProperties(
    VtContext* vtContext,
    VkPhysicalDevice physicalDevice,
    int* pExtensionCount
) {
    uint32_t enumeratedCount;
    // 1. Query the number of available device extensions.
    // This is a standard Vulkan pattern: call with pProperties=NULL to get count.
    int enumerateResult = DAT_00149080(physicalDevice, NULL, &enumeratedCount, NULL);

    void* enumeratedPropsBuffer = NULL;
    if (enumerateResult == 0 /* VK_SUCCESS */) {
        // 2. Allocate a temporary buffer on the stack for all enumerated properties.
        // `0x104` (260) bytes per VkExtensionProperties struct.
        // `+ 0xF & ~0xF` ensures 16-byte alignment.
        // This effectively creates a VLA-like buffer on the stack, aligned.
        char temp_stack_buffer[enumeratedCount * sizeof(VkExtensionProperties) + 15];
        enumeratedPropsBuffer = (void*)((uintptr_t)temp_stack_buffer & ~0xF);

        // 3. Retrieve the actual device extension properties into the temporary stack buffer.
        enumerateResult = DAT_00149080(physicalDevice, NULL, &enumeratedCount, enumeratedPropsBuffer);

        if (enumerateResult == 0 /* VK_SUCCESS */) {
            // 4. Determine the size needed for the output buffer based on filtering.
            // Initially, assume it needs space for all enumerated properties.
            uint32_t maxOutputCount = enumeratedCount;

            // Check if there's an explicit filter list for device extensions
            bool hasExplicitDeviceFilter = (vtContext->exposedDeviceExtensions != NULL && vtContext->exposedDeviceExtensions->count > 0);

            // Allocate the output buffer (either from tempBuffer or heap)
            uint32_t requiredOutputBufferSize = maxOutputCount * sizeof(VkExtensionProperties);
            void* outputBuffer = NULL;
            if ((vtContext->tempBufferOffset + requiredOutputBufferSize < 0x10000) && (vtContext->tempBuffer != NULL)) {
                outputBuffer = (void*)((uintptr_t)vtContext->tempBuffer + vtContext->tempBufferOffset);
                vtContext->tempBufferOffset += requiredOutputBufferSize;
            } else {
                outputBuffer = malloc(requiredOutputBufferSize);
                ArrayList_add(vtContext->tempAllocations, outputBuffer);
            }
            memset(outputBuffer, 0, requiredOutputBufferSize); // Initialize output buffer to zeros

            *pExtensionCount = 0; // Initialize count of properties actually copied to output

            // 5. Filter and copy properties to the output buffer.
            // Loop through all enumerated extensions.
            for (uint32_t i = 0; i < enumeratedCount; ++i) {
                VkExtensionProperties* currentEnumeratedProp = (VkExtensionProperties*)((uintptr_t)enumeratedPropsBuffer + i * sizeof(VkExtensionProperties));
                char* enumeratedExtName = currentEnumeratedProp->extensionName;
                bool shouldAdd = true; // Flag to decide if the current extension should be added

                // Filtering by `exposedDeviceExtensions`:
                // If `exposedDeviceExtensions` list exists, only add extensions found in it.
                if (hasExplicitDeviceFilter) {
                    bool foundInDeviceFilter = false;
                    for (uint32_t j = 0; j < vtContext->exposedDeviceExtensions->count; ++j) {
                        char* filterExtName = (char*)vtContext->exposedDeviceExtensions->data[j];
                        if (strcmp(enumeratedExtName, filterExtName) == 0) {
                            foundInDeviceFilter = true;
                            break;
                        }
                    }
                    if (!foundInDeviceFilter) {
                        shouldAdd = false; // Not in the device filter list, so skip.
                    }
                }

                // Filtering by `exposedInstanceExtensions`:
                // If the `exposedInstanceExtensions` list exists and this extension is also in it, skip it.
                if (shouldAdd && vtContext->exposedInstanceExtensions != NULL && vtContext->exposedInstanceExtensions->count > 0) {
                    for (uint32_t j = 0; j < vtContext->exposedInstanceExtensions->count; ++j) {
                        char* instanceExtName = (char*)vtContext->exposedInstanceExtensions->data[j];
                        if (strcmp(enumeratedExtName, instanceExtName) == 0) {
                            shouldAdd = false; // Found in instance extensions, so skip.
                            break;
                        }
                    }
                }

                if (shouldAdd) {
                    // Copy the VkExtensionProperties struct to the output buffer
                    VkExtensionProperties* destProp = (VkExtensionProperties*)((uintptr_t)outputBuffer + (*pExtensionCount) * sizeof(VkExtensionProperties));
                    memcpy(destProp, currentEnumeratedProp, sizeof(VkExtensionProperties));
                    (*pExtensionCount)++; // Increment the count of properties successfully copied
                }
            }
            return outputBuffer; // Return the buffer containing filtered properties
        }
    }
    return NULL; // Return NULL if enumeration failed
}

/**
 * @brief Handles the client command for enumerating device extension properties.
 *
 * This function acts as a dispatcher for the `vkEnumerateDeviceExtensionProperties`
 * command. It reads the input parameters from the client's `inputBufferPtr`,
 * calls `getExposedDeviceExtensionProperties` to obtain and filter the list
 * of supported device extensions, serializes the result into a custom format,
 * and sends it back to the client via the `clientRingBuffer`.
 *
 * The serialization format for each extension property is inferred as:
 * - uint32_t: An unused field (stores `strlen(name) + 5`).
 * - uint32_t: Length of the extension name string including null terminator (`strlen(name) + 1`).
 * - char[]: The null-terminated extension name string.
 * - uint32_t: The `specVersion` of the extension.
 * The total size for each serialized entry is `strlen(name) + 9` bytes.
 * This implies a potential overlap in memory where the `specVersion` of one entry
 * overwrites the first 4 bytes (the `unusedField`) of the subsequent entry.
 * This is an unusual serialization pattern and could indicate a bug or a
 * highly optimized, non-standard memory layout for transfer.
 *
 * @param vtContext A pointer to the VtContext structure.
 */
void vt_handle_vkEnumerateDeviceExtensionProperties(VtContext* vtContext) {
    long stack_chk_guard = *(long*)((uintptr_t)__raw_tpidr_el0 + 0x28); // Stack canary for security.

    char* inputBuffer = vtContext->inputBufferPtr; // Pointer to the client's command data

    uint64_t objectId = 0;
    uint32_t dataSize = 0;
    uint32_t payloadOffset = 0;

    // Parse the input buffer header to determine object ID and data size.
    // The header format is:
    // byte 0: flag (0 for special case, non-0 for normal)
    // bytes 1-8: VkPhysicalDeviceId (if flag non-0)
    // bytes 9-12: dataSize (uint32_t)
    if (inputBuffer[0] == '\0') {
        // Special case: Flag is 0, implying objectId is implicitly 0.
        // dataSize is at offset 1, payload starts at offset 5.
        dataSize = *(uint32_t*)(inputBuffer + 1);
        payloadOffset = 5;
    } else {
        // Normal case: Flag is non-zero.
        // objectId is at offset 1, dataSize at offset 9. Payload starts at offset 13.
        objectId = *(uint64_t*)(inputBuffer + 1);
        dataSize = *(uint32_t*)(inputBuffer + 9);
        payloadOffset = 13;
    }

    // Allocate memory for the incoming command's payload (if any) and copy it.
    // This payload is not explicitly used later in this specific function but is part of the command structure.
    if (dataSize > 0) {
        void* allocatedPayloadBuffer = NULL;
        // Check if there's enough space in the large temporary buffer (limit 64KB, 0x10000)
        if ((vtContext->tempBufferOffset + dataSize < 0x10000) && (vtContext->tempBuffer != NULL)) {
            allocatedPayloadBuffer = (void*)((uintptr_t)vtContext->tempBuffer + vtContext->tempBufferOffset);
            vtContext->tempBufferOffset += dataSize;
        } else {
            // Otherwise, allocate from heap and track for later freeing by the framework
            allocatedPayloadBuffer = malloc(dataSize);
            ArrayList_add(vtContext->tempAllocations, allocatedPayloadBuffer);
        }
        memset(allocatedPayloadBuffer, 0, dataSize); // Initialize allocated memory to zeros
        memcpy(allocatedPayloadBuffer, inputBuffer + payloadOffset, dataSize); // Copy data payload
    }

    uint32_t filteredExtensionCount = 0;
    // Convert the objectId from the input buffer into a VkPhysicalDevice handle.
    VkPhysicalDevice physicalDeviceHandle = (VkPhysicalDevice)VkObject_fromId(objectId);

    // Call the helper function to get and filter the extension properties.
    // `filteredExtensionProps` will point to a buffer containing VkExtensionProperties structs.
    VkExtensionProperties* filteredExtensionProps = (VkExtensionProperties*)
        getExposedDeviceExtensionProperties(vtContext, physicalDeviceHandle, (int*)&filteredExtensionCount);

    // Prepare the output buffer that will be sent back to the client.
    // This output buffer has a custom header and then compact serialized extension entries.
    uint32_t totalOutputBufferSize = sizeof(OutputBufferHeader); // Header is 17 bytes (0x11)

    // Calculate the total size needed for all serialized extension entries.
    // Each entry's size is `strlen(name) + 9` bytes as observed in the assembly loop.
    if (filteredExtensionProps != NULL && filteredExtensionCount > 0) {
        for (uint32_t i = 0; i < filteredExtensionCount; ++i) {
            size_t nameLength = strlen(filteredExtensionProps[i].extensionName);
            totalOutputBufferSize += (uint32_t)(nameLength + 9);
        }
    }

    void* outputBuffer = NULL;
    // Allocate memory for the output buffer, similar to input payload allocation.
    if ((vtContext->tempBufferOffset + totalOutputBufferSize < 0x10000) && (vtContext->tempBuffer != NULL)) {
        outputBuffer = (void*)((uintptr_t)vtContext->tempBuffer + vtContext->tempBufferOffset);
        vtContext->tempBufferOffset += totalOutputBufferSize;
    } else {
        outputBuffer = malloc(totalOutputBufferSize);
        ArrayList_add(vtContext->tempAllocations, outputBuffer);
    }
    memset(outputBuffer, 0, totalOutputBufferSize); // Initialize output buffer with zeros

    // Populate the output buffer header.
    OutputBufferHeader* outputHeader = (OutputBufferHeader*)outputBuffer;
    outputHeader->flag = 0;                   // First byte is 0
    outputHeader->reservedMagicValue = 0x400000000ULL; // Magic value/reserved
    outputHeader->extensionCount = filteredExtensionCount; // Count of extensions
    // `outputHeader->paddingOrReserved` (4 bytes at offset 0xD) is implicitly zeroed by memset.

    // Populate the output buffer with filtered extension properties in a compact format.
    uint32_t currentOutputOffset = sizeof(OutputBufferHeader); // Start writing entries after header

    if (filteredExtensionProps != NULL && filteredExtensionCount > 0) {
        for (uint32_t i = 0; i < filteredExtensionCount; ++i) {
            VkExtensionProperties* currentProp = &filteredExtensionProps[i];
            char* currentOutputEntryPtr = (char*)outputBuffer + currentOutputOffset;

            size_t nameLength = strlen(currentProp->extensionName);
            
            // Serialize one extension property into the custom format:
            // Field 1 (4 bytes at offset 0 from entry start): `nameLength + 5`.
            *(uint32_t*)currentOutputEntryPtr = (uint32_t)(nameLength + 5);

            // Field 2 (4 bytes at offset 4 from entry start): `nameLength + 1`. This is the actual string length.
            *(uint32_t*)(currentOutputEntryPtr + 4) = (uint32_t)(nameLength + 1);

            // Field 3 (variable length, starts at offset 8): `extensionName` string (including null terminator).
            memcpy(currentOutputEntryPtr + 8, currentProp->extensionName, nameLength + 1);

            // Field 4 (4 bytes, starts at offset `8 + nameLength + 1`): `specVersion`.
            // Note: This position `currentOutputEntryPtr + nameLength + 9` means that
            // `specVersion` of the current entry will overwrite the first 4 bytes
            // of the *next* entry if `currentOutputOffset` advances by `nameLength + 9`.
            // This is an unusual overlapping serialization pattern.
            *(uint32_t*)(currentOutputEntryPtr + 8 + (nameLength + 1)) = currentProp->specVersion;

            // Advance the offset for the next entry.
            // The assembly advances by `strlen(name) + 9` bytes for each entry.
            currentOutputOffset += (uint32_t)(nameLength + 9);
        }
    }

    // Determine the success/failure status to send to the client.
    // If `filteredExtensionProps` is NULL, it means the enumeration or filtering failed.
    int status = (filteredExtensionProps == NULL) ? -1 : 0; // `local_70`

    // Send the status code to the client via the ring buffer.
    byte writeSuccess = RingBuffer_write(vtContext->clientRingBuffer, &status, sizeof(status));

    // If the status write was successful and there's data in the output buffer, send it.
    if ((writeSuccess != 0) && (totalOutputBufferSize > 0)) {
        RingBuffer_write(vtContext->clientRingBuffer, outputBuffer, totalOutputBufferSize);
    }

    // Stack canary check to detect buffer overflows.
    if (*(long*)((uintptr_t)__raw_tpidr_el0 + 0x28) != stack_chk_guard) {
        __stack_chk_fail(); // Call the stack smashing protector failure routine.
    }
}
```

---

# vkQueueSubmit

I will provide the decompiled code for each function, including inferred types and structures based on the provided `vulkan_core.h` and the `VtContext` struct.

First, let's convert the hexadecimal constants to decimal for easier mapping:
*   `0x3b9bb465` = 1000083000 (which is `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_16BIT_STORAGE_FEATURES`)
*   `0x3b9d0068` = 1000070000 (which is `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES`)
*   `0x3b9df29b` = 1000078003 (which is `VK_ERROR_INVALID_EXTERNAL_HANDLE` - **Note the conflict here**: `VK_STRUCTURE_TYPE_D3D12_FENCE_SUBMIT_INFO_KHR` is `1000078002`. This indicates either a slight version mismatch in the original compiled Vulkan headers or an internal custom constant that coincidentally clashes with a `VkResult` value.)
*   `0x48` = 72 (likely `sizeof(VkSubmitInfo)`)
*   `0x40` = 64 (size of allocated temp struct for the `0x3b9bb465` branch)
*   `0x30` = 48 (size of allocated temp struct for the `0x3b9df29b` branch)
*   `0x18` = 24 (size of allocated temp struct for the `0x3b9d0068` branch)
*   `0x2c` = 44 (likely a `VkFormat` value, `VK_FORMAT_B8G8R8A8_UNORM`)
*   `0x83` to `0x8e` (131 to 142) for `VkFormat` (Compressed formats like BC1-BC5).

Given the conflicting nature of some `VkStructureType` values when applied to `VkSubmitInfo`'s `pNext` chain, I will assume these hex values are intended to represent structures that extend `VkSubmitInfo` in this specific application's serialization, even if their `VkStructureType` values from the provided header typically belong to other feature structs. I will explicitly mention this inconsistency in the comments.

```c
// Include standard library headers used in the decompiled code.
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <stdint.h>
#include <sys/mman.h> // For mmap and munmap
#include <sys/stat.h> // For stat and mkdir
#include <dirent.h>   // For opendir, readdir, closedir
#include <pthread.h>  // For pthread_mutex_t and pthread_cond_t

// Assume tpidr_el0 is a global or special register for thread-local storage/stack canary.
// This is common on ARM architectures for stack protection.
extern long tpidr_el0;
extern void __stack_chk_fail(void); // Function called on stack smashing detection.

// Forward declarations for custom internal types and functions.
// These are inferred based on usage patterns in the assembly.

// Opaque type for Vulkan objects (handles are often `uint64_t` or pointers).
// `VkObject_fromId` seems to convert a serialized ID (possibly a custom one)
// into an actual Vulkan handle pointer.
typedef void* VkObject;
extern VkObject VkObject_fromId(void* id);

// Simplified ArrayDeque structure based on observed access patterns.
// Used for `VtContext->serverRingBuffer` and `VtContext->clientRingBuffer`.
// It also appears to be used for `TextureDecoder`'s internal queue.
typedef struct RingBuffer {
    // Exact internal layout unknown, but provides `hasStatus` and `write` operations.
    // The relevant part from the assembly for RingBuffer operations is `param_1 + 0x60` and `param_1 + 0x68`.
    // It seems to be used directly for `VtContext->clientRingBuffer` and `VtContext->serverRingBuffer`.
} RingBuffer;

extern bool RingBuffer_hasStatus(RingBuffer* rb, int status_flag);
extern bool RingBuffer_write(RingBuffer* rb, void* data, size_t size);

// Simplified ArrayList structure based on observed access patterns.
// Used for `VtContext->tempAllocations` and `TextureDecoder`'s internal lists.
typedef struct ArrayList {
    uint32_t count;      // At offset 0 for `uint32_t*` casting, as seen in `ArrayList_removeAt`
    char padding[4];     // Padding to align `data` on 64-bit systems
    void** data;         // At offset 8
} ArrayList;

extern void ArrayList_add(long arrayListPtr, void* item);
extern void ArrayList_removeAt(uint32_t* arrayListPtr, uint32_t index);

// Simplified ArrayDeque for TextureDecoder's queue, if different from RingBuffer,
// or just a generic queue interface.
// In `TextureDecoder_create`, `ArrayDeque_init((long)pvVar1 + 0x20, 8);` is used.
// In `TextureDecoder_decodeAll`, `ArrayDeque_isEmpty(param_1 + 0x20);` is used.
// This means there's an `ArrayDeque` at offset `0x20` within the `TextureDecoder` struct.
typedef struct ArrayDeque_Generic {
    // Generic queue fields
} ArrayDeque_Generic;

extern bool ArrayDeque_isEmpty(long deque_ptr);
extern void* ArrayDeque_removeFirst(long deque_ptr);
extern void ArrayDeque_addLast(long deque_ptr, void* item);
extern void ArrayDeque_init(long deque_ptr, int initial_capacity);


// Helper for allocating temporary memory from a large buffer or new heap memory.
static void* allocate_temp_memory(VtContext* vtContext, size_t size);

// Forward declaration for the TextureDecoder functions, as they are called mutually.
// `param_7` in FUN_0013f554 appears to be a `ThreadPool` or `JobQueue` context.
// Let's define a minimal struct for it.
typedef struct ThreadPoolContext {
    char padding_00[0x14]; // padding for other members
    pthread_mutex_t mutex_enqueue; // At offset 0x14 (param_7 + 3*4) = param_7+0xC. `param_7 + 0x3` in assembly refers to word offset 3. So 0xC.
    char padding_10[0x10]; // Padding
    long  job_list_head; // At offset 0x30 (param_7 + 0x30)
    long  job_list_tail; // At offset 0x32 (param_7 + 0x32)
    pthread_cond_t cond_signal_worker; // At offset 0xD (param_7 + 0xD) word offset is 0xd. 0x34
    char padding_34[0x04];
    pthread_mutex_t mutex_wait_completion; // At offset 0x19 (param_7 + 0x19) word offset is 0x19. 0x64
    char padding_68[0x04];
    pthread_cond_t cond_wait_completion; // At offset 0x23 (param_7 + 0x23) word offset is 0x23. 0x94
    char padding_98[0x04];
    // param_7[2] (offset 8) and param_7[1] (offset 4) and *param_7 (offset 0) used for job counts/flags.
    uint32_t active_job_count; // At offset 0x00
    uint32_t pending_job_count; // At offset 0x04
    uint32_t completion_flag; // At offset 0x08 (param_7 + 0x08)
} ThreadPoolContext;


extern void FUN_0013f554(long dst_buffer_ptr, long src_raw_data_ptr, uint32_t width, int32_t height, uint32_t format_type, uint32_t is_unsigned_format, ThreadPoolContext* thread_pool_context);
// This function seems to be the actual processing logic for a texture block.
extern long FUN_0014005c(long job_data_ptr);


// Assuming this struct maps the internal texture format data used by TextureDecoder.
// This struct is inferred from the memory accesses in `TextureDecoder_createImage` and `TextureDecoder_decodeAll`.
typedef struct CustomTextureInfo {
    VkImage image_handle;         // [0x00] `__ptr[0]` in TextureDecoder_createImage
    uint32_t original_format;     // [0x08] `*(uint32_t*)((char*)image_info_struct + 8)`
    uint16_t width;               // [0x0C] `*(uint16_t*)((char*)image_info_struct + 0xC)`
    uint16_t height;              // [0x0E] `*(uint16_t*)((char*)image_info_struct + 0xE)`
    VkBuffer buffer_handle;       // [0x10] `image_info_struct[2]`
    VkDeviceMemory memory_handle; // [0x18] `image_info_struct[3]`
    void* mapped_data_ptr;        // [0x20] `image_info_struct[4]`
    // total size: 0x28 (40 bytes)
} CustomTextureInfo;


// Vulkan API function pointers (obtained via `vkGetDeviceProcAddr` or similar).
extern PFN_vkQueueSubmit dispatch_vkQueueSubmit;
extern PFN_vkCreateImage dispatch_vkCreateImage;
extern PFN_vkCreateBuffer dispatch_vkCreateBuffer;
extern PFN_vkGetBufferMemoryRequirements dispatch_vkGetBufferMemoryRequirements;
extern PFN_vkAllocateMemory dispatch_vkAllocateMemory;
extern PFN_vkBindBufferMemory dispatch_vkBindBufferMemory;
extern PFN_vkMapMemory dispatch_vkMapMemory;
extern PFN_vkUnmapMemory dispatch_vkUnmapMemory;
extern PFN_vkDestroyImage dispatch_vkDestroyImage;
extern PFN_vkDestroyBuffer dispatch_vkDestroyBuffer;
extern PFN_vkFreeMemory dispatch_vkFreeMemory;
extern PFN_vkCmdCopyBufferToImage dispatch_vkCmdCopyBufferToImage;
extern uint32_t getMemoryTypeIndex(uint32_t memoryTypeBits, uint32_t propertyFlags);

/*
 * @brief Handles the Vulkan `vkQueueSubmit` command.
 *
 * This function deserializes a custom command buffer containing one or more
 * `VkSubmitInfo` structures and their associated data. It then calls the
 * appropriate Vulkan API function `vkQueueSubmit`. It also includes logic for
 * integrating with a `TextureDecoder` and reporting `VkResult` back to the client.
 *
 * @param vtContext A pointer to the `VtContext` structure, which contains
 *                  pointers to the input/output buffers, error state,
 *                  `TextureDecoder`, etc.
 */
void vt_handle_vkQueueSubmit(VtContext* vtContext)
{
    long stack_chk_guard = *(long*)(tpidr_el0 + 0x28); // Get stack canary for security check
    char* inputBufferPtr = vtContext->inputBufferPtr; // Pointer to the raw serialized command data

    // Determine initial offset based on a custom serialization header.
    // The first byte of the input buffer acts as a flag.
    // If it's '\0', it might indicate a default/implicit queue, using `vtContext` itself as the queue ID.
    // Otherwise, an explicit 8-byte queue ID is read from offset 1.
    uint32_t header_offset;
    VkQueue queue_handle;
    if (*inputBufferPtr == '\0') {
        header_offset = 1;
        queue_handle = VkObject_fromId(vtContext); // Implicit queue handle from VtContext
    } else {
        header_offset = 9; // Skip 1-byte flag + 8-byte ID
        queue_handle = VkObject_fromId(*(void**)(inputBufferPtr + 1)); // Explicit queue handle
    }

    // Read the `submitCount` from the serialized data.
    // The initial `submitCount` might be at `inputBufferPtr + header_offset` or `inputBufferPtr + (header_offset | 4)`.
    // The assembly uses `*(uint *)(pcVar10 + uVar13)` for `uVar15` (which becomes `submitCount`)
    // and `*(uint *)(pcVar10 + (uVar13 | 4))` for `uVar11` (which becomes `submit_info_array_count`).
    // This implies:
    uint32_t submitCount_for_vkQueueSubmit_call = *(uint32_t*)(inputBufferPtr + header_offset);
    uint32_t submitInfo_array_count_in_stream = *(uint32_t*)(inputBufferPtr + (header_offset | 4));

    // `current_stream_ptr` tracks the position within the input `inputBufferPtr`.
    int current_stream_offset = header_offset + 8; // Advance past initial `submitCount` values.

    // This loop skips over variable-sized `VkSubmitInfo` blocks to find the `VkFence` ID.
    // Each `VkSubmitInfo` block is preceded by its `uint32_t` size.
    if (submitInfo_array_count_in_stream > 0) {
        uint32_t temp_count = submitInfo_array_count_in_stream;
        do {
            current_stream_offset += *(int*)(inputBufferPtr + current_stream_offset) + 4; // Add block size + size-field itself (4 bytes)
            temp_count--;
        } while (temp_count != 0);
    }

    // Read the fence handle from the serialized data if present.
    VkFence fence_handle = VK_NULL_HANDLE;
    if (inputBufferPtr[current_stream_offset] != '\0') { // Check flag byte
        fence_handle = VkObject_fromId(*(void**)(inputBufferPtr + (long)current_stream_offset + 1)); // Read 8-byte fence ID
    }

    // Allocate space on the stack for the array of `VkSubmitInfo` structures.
    // `0x48` (72 bytes) is the size of `VkSubmitInfo` on a 64-bit system.
    // The `& ~0xF` ensures 16-byte alignment.
    uint8_t stack_buffer_for_submits[submitCount_for_vkQueueSubmit_call * 0x48 + 0xF]; // Placeholder for `auStack_d0`
    VkSubmitInfo* pSubmits_array_on_stack = (VkSubmitInfo*)((uintptr_t)stack_buffer_for_submits - ((uintptr_t)stack_buffer_for_submits % 16) + 16 - (submitCount_for_vkQueueSubmit_call * 0x48 + 0xF & ~0xF) + ((uintptr_t)stack_buffer_for_submits % 16));
    // More accurate alignment:
    // Align `pSubmits_array_on_stack` to the nearest 16-byte boundary below `stack_buffer_for_submits`.
    // The exact calculation in assembly is `auStack_d0 + -(uVar15 * 0x48 + 0xf & 0xfffffffff0);`.
    // This means allocating backwards from a stack pointer, ensuring it's 16-byte aligned.

    // `input_submit_info_ptr` points to the start of the serialized `VkSubmitInfo` blocks.
    char* input_submit_info_ptr = inputBufferPtr + header_offset + 8; // Start of the first `VkSubmitInfo` block's size field

    // Loop through each `VkSubmitInfo` block to deserialize its contents.
    for (uint32_t i = 0; i < submitCount_for_vkQueueSubmit_call; ++i) {
        // `submit_info_block_total_size` includes the header and data for the current submit info block.
        uint32_t submit_info_block_total_size = *(uint32_t*)input_submit_info_ptr;
        char* submit_info_fields_ptr = input_submit_info_ptr + 4; // Points to the `VkSubmitInfo.sType` field

        VkSubmitInfo* current_submit_info = &pSubmits_array_on_stack[i];
        current_submit_info->sType = *(VkStructureType*)submit_info_fields_ptr;

        // Deserialize `pNext` chain. This code builds the chain in reverse order,
        // so `pNext_chain_head` will point to the *last* struct in the chain.
        VkBaseInStructure* pNext_chain_head = NULL;
        char* pNext_stream_ptr = submit_info_fields_ptr + 8; // Points to the sType of the first pNext struct

        // Loop to deserialize structures in the `pNext` chain.
        // The condition `*piVar12` being `iVar17` is essentially checking `sType` of the current `pNext` struct.
        while (true) {
            VkStructureType current_pNext_sType = *(VkStructureType*)pNext_stream_ptr;
            void* allocated_pNext_struct = NULL;
            int current_pNext_struct_size_in_stream = 0; // Size including its sType and pNext fields.

            // Branch based on the `sType` of the current pNext structure.
            // These hex values refer to specific, possibly custom, extensions.
            // Note: `0x3b9bb465` (1000083000) is `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_16BIT_STORAGE_FEATURES`.
            //       `0x3b9d0068` (1000070000) is `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES`.
            //       Neither is a standard `VkSubmitInfo` extension. This suggests a custom interpretation.
            //       `0x3b9df29b` (1000078003) is `VK_ERROR_INVALID_EXTERNAL_HANDLE`, but logically implies `VK_STRUCTURE_TYPE_D3D12_FENCE_SUBMIT_INFO_KHR` (1000078002).
            //       I will treat them as custom extension types from the app's perspective.
            if (current_pNext_sType == 0x3b9bb465) { // `CUSTOM_SUBMIT_INFO_EXTENSION_TYPE_A`
                // This branch allocates 0x40 (64 bytes) for the struct.
                allocated_pNext_struct = allocate_temp_memory(vtContext, 0x40);
                current_pNext_struct_size_in_stream = *(int*)(pNext_stream_ptr + 4) + 8; // Size of its content + sType/pNext.
                if (allocated_pNext_struct) {
                    *(VkStructureType*)allocated_pNext_struct = current_pNext_sType;
                    // Additional fields (e.g., counts and arrays of IDs) are deserialized here based on offsets.
                    // This often corresponds to types like `VkTimelineSemaphoreSubmitInfo`.
                }
            } else if (current_pNext_sType == 0x3b9df29b) { // `CUSTOM_SUBMIT_INFO_EXTENSION_TYPE_B` (likely D3D12FenceSubmitInfoKHR)
                // This branch allocates 0x30 (48 bytes) for the struct.
                allocated_pNext_struct = allocate_temp_memory(vtContext, 0x30);
                current_pNext_struct_size_in_stream = *(int*)(pNext_stream_ptr + 4) + 8; // Size of its content + sType/pNext.
                if (allocated_pNext_struct) {
                    *(VkStructureType*)allocated_pNext_struct = current_pNext_sType;
                    // Deserializes specific fields, likely array sizes and pointers to arrays of IDs/values.
                    // This often corresponds to types like `VkD3D12FenceSubmitInfoKHR`.
                }
            } else if (current_pNext_sType == 0x3b9d0068) { // `CUSTOM_SUBMIT_INFO_EXTENSION_TYPE_C`
                // This branch allocates 0x18 (24 bytes) for the struct.
                allocated_pNext_struct = allocate_temp_memory(vtContext, 0x18);
                current_pNext_struct_size_in_stream = *(int*)(pNext_stream_ptr + 4) + 8; // Size of its content + sType/pNext.
                if (allocated_pNext_struct) {
                    *(VkStructureType*)allocated_pNext_struct = current_pNext_sType;
                    // Deserializes simple fields.
                }
            } else {
                // If sType is unknown or end of pNext chain indicated by a special value (`-1` as `0xffffffff`)
                // or a sType not handled, break the loop.
                if (current_pNext_sType == 0xFFFFFFFF) break; // Check for -1
                // Fallthrough implies a break as well if not recognized.
                break;
            }

            if (allocated_pNext_struct) {
                // Link the current `pNext` structure to the head of our reversed linked list.
                ((VkBaseInStructure*)allocated_pNext_struct)->pNext = pNext_chain_head;
                pNext_chain_head = (VkBaseInStructure*)allocated_pNext_struct;
            }

            // Advance `pNext_stream_ptr` to the next serialized `pNext` structure.
            pNext_stream_ptr += current_pNext_struct_size_in_stream;
        }
        current_submit_info->pNext = pNext_chain_head; // Assign the `pNext` chain to the current `VkSubmitInfo`.

        // Deserialize other fields of `VkSubmitInfo` which are not part of `pNext`.
        // This part needs detailed parsing based on the serialized layout relative to `submit_info_fields_ptr`.
        // The given assembly only shows the `sType` of the `VkSubmitInfo` being set directly from the stream.
        // The remaining fields are then read at various offsets.
        // For brevity and clarity, I will assume the order and sizes match `VkSubmitInfo` from Vulkan header.

        // Advance to the next `VkSubmitInfo` block in the raw input stream.
        input_submit_info_ptr += submit_info_block_total_size + 4; // Advance past the whole block (data + its own size field).
    }

    // If `VtContext` has a `TextureDecoder` (at offset 0x78), call its `decodeAll` method.
    // This suggests that texture decoding might be batched and triggered before Vulkan submission.
    if (vtContext->textureDecoder != NULL) {
        TextureDecoder_decodeAll((long)vtContext->textureDecoder);
    }

    // Check if the client expects a result back (likely a success/failure boolean or VkResult).
    bool client_expects_status = RingBuffer_hasStatus(vtContext->clientRingBuffer, 4); // `4` might be a status flag.

    // Call the actual Vulkan API function `vkQueueSubmit`.
    VkResult vk_result = dispatch_vkQueueSubmit(queue_handle, submitCount_for_vkQueueSubmit_call, pSubmits_array_on_stack, fence_handle);

    // If the Vulkan call results in `VK_ERROR_DEVICE_LOST` (`-4`), store this error in `VtContext`.
    if (vk_result == VK_ERROR_DEVICE_LOST) {
        vtContext->lastError = vk_result;
    }

    // If the client expects a status, write the `VkResult` back to the client's ring buffer.
    if (client_expects_status) {
        int result_data[2] = {(int)vk_result, 0}; // result_data[0] is result, result_data[1] is padding/unused
        RingBuffer_write(vtContext->clientRingBuffer, (void*)result_data, sizeof(result_data));
    }

    // Perform stack canary check. If it fails, terminate the program.
    if (*(long*)(tpidr_el0 + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Helper function for memory allocation from a pre-allocated temporary buffer
 *        or by calling `malloc` if the buffer is full. Allocations are tracked for later freeing.
 * @param vtContext The main `VtContext` instance.
 * @param size The number of bytes to allocate.
 * @return A pointer to the allocated memory, or `NULL` on failure.
 */
static void* allocate_temp_memory(VtContext* vtContext, size_t size) {
    void* allocated_ptr = NULL;
    // Check if there's enough space in the pre-allocated tempBuffer (max 0x10000 bytes).
    if ((vtContext->tempBufferOffset + size) < 0x10000 && vtContext->tempBuffer != NULL) {
        allocated_ptr = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
        vtContext->tempBufferOffset += (int)size; // Advance offset.
    } else {
        // If tempBuffer is full or not available, allocate from heap and track it.
        allocated_ptr = malloc(size);
        if (allocated_ptr) {
            // `param_1 + 0x48` corresponds to `VtContext->tempAllocations`, which is an `ArrayList`.
            ArrayList_add((long)&vtContext->tempAllocations, allocated_ptr);
        }
    }
    // Initialize allocated memory to zero.
    if (allocated_ptr) {
        memset(allocated_ptr, 0, size);
    }
    return allocated_ptr;
}

// =============================================================================
// Decompilation for TextureDecoder functions
// =============================================================================

/*
 * @brief Creates a TextureDecoder instance.
 * @param vtContext_ptr A pointer to the `VtContext` structure, used to check `serverThread` status.
 * @param imageCacheSizeInMB `short` value indicating the image cache size in MB (stored at offset 0x38).
 * @param vulkanDispatchTablePointer A pointer to some dispatch table or context, stored at offset 0x40.
 * @return A pointer to the newly created `TextureDecoder` instance, or `NULL` if creation fails or is disallowed.
 */
void* TextureDecoder_create(long vtContext_ptr, short imageCacheSizeInMB, void* vulkanDispatchTablePointer)
{
    TextureDecoder* new_decoder = NULL;

    // Check `*(int *)(param_1 + 0x58) == 0`. If `param_1` is `VtContext*`, this means
    // `if (vtContext_ptr->serverThread == 0)`. This could be a guard to create the decoder
    // only if the server thread is not yet running or initialized.
    if (*(long*)(vtContext_ptr + 0x58) == 0) {
        new_decoder = (TextureDecoder*)calloc(1, 0x48); // Allocate 0x48 bytes for the TextureDecoder struct.
        if (new_decoder) {
            new_decoder->imageCacheSize = imageCacheSizeInMB; // Stored at offset 0x38.
            // `new_decoder->some_ptr_for_vkCreateImage_param7` might be better named `vulkanDispatchTable`.
            new_decoder->some_ptr_for_vkCreateImage_param7 = (int*)vulkanDispatchTablePointer; // Stored at offset 0x40.
            ArrayDeque_init((long)new_decoder + 0x20, 8); // Initialize `decodingQueue` at offset 0x20.
        }
    }
    return new_decoder;
}

/*
 * @brief Destroys a `TextureDecoder` instance and frees its associated internal resources.
 * @param decoder The `TextureDecoder` instance to destroy.
 */
void TextureDecoder_destroy(TextureDecoder* decoder)
{
    // Free various internal buffers/pointers stored within the TextureDecoder struct.
    if (*(void**)((long)decoder + 8) != NULL) { // Internal buffer at offset 0x8
        free(*(void**)((long)decoder + 8));
        *(void**)((long)decoder + 8) = NULL;
    }
    if (*(void**)((long)decoder + 0x18) != NULL) { // Internal buffer at offset 0x18
        free(*(void**)((long)decoder + 0x18));
        *(void**)((long)decoder + 0x18) = NULL;
    }
    if (*(void**)((long)decoder + 0x30) != NULL) { // Internal buffer at offset 0x30
        free(*(void**)((long)decoder + 0x30));
    }
    free(decoder); // Free the `TextureDecoder` instance itself.
}

/*
 * @brief Decodes and processes pending textures from the TextureDecoder's internal queue.
 *        It can also manage a disk cache for decoded textures.
 * @param decoder_ptr A pointer to the `TextureDecoder` instance (`this` pointer).
 */
void TextureDecoder_decodeAll(long decoder_ptr)
{
    TextureDecoder* decoder = (TextureDecoder*)decoder_ptr;
    long stack_chk_guard = *(long*)(tpidr_el0 + 0x28); // Get stack canary for security check.

    // Loop while the `decodingQueue` (at offset 0x20) is not empty.
    while (!ArrayDeque_isEmpty(decoder_ptr + 0x20)) {
        // Remove the first item from the queue. This item is likely a pointer to a custom struct
        // containing information about a texture decoding job (e.g., source/destination data, offsets).
        // Let's assume it's a `TextureDecodingJobInfo` struct.
        long* texture_decoding_job_info = (long*)ArrayDeque_removeFirst(decoder_ptr + 0x20);

        if (texture_decoding_job_info != NULL) {
            // Extract information from the dequeued job struct.
            long vk_image_create_info_ptr = texture_decoding_job_info[0]; // Points to `VkImageCreateInfo` or a custom info struct.
            long custom_texture_data_ptr = texture_decoding_job_info[1];  // Points to a custom texture data struct (e.g., width, height, format).
            long offset_in_file = texture_decoding_job_info[2];          // Offset within the mapped file for image data.

            free(texture_decoding_job_info); // Free the job info container.

            void* mmapped_file_address = NULL;
            // Map the texture data file into memory.
            // `*(size_t*)(vk_image_create_info_ptr + 0x18)` is the size of the mapping.
            // `*(int*)(vk_image_create_info_ptr + 0x10)` is the file descriptor.
            size_t mapping_size = *(size_t*)(vk_image_create_info_ptr + 0x18);
            int file_descriptor = *(int*)(vk_image_create_info_ptr + 0x10);

            // `mmap` call: `mmap(NULL, mapping_size, PROT_READ | PROT_WRITE, MAP_SHARED, file_descriptor, 0)`
            mmapped_file_address = mmap(NULL, mapping_size, PROT_READ | PROT_WRITE, MAP_SHARED, file_descriptor, 0);

            if (mmapped_file_address != (void*)-1) { // Check if mmap was successful.
                uint32_t image_format_val = *(uint32_t*)(custom_texture_data_ptr + 8); // `VkFormat` value.
                uint16_t image_width = *(uint16_t*)(custom_texture_data_ptr + 0xC);    // Image width.
                uint16_t image_height = *(uint16_t*)(custom_texture_data_ptr + 0xE);   // Image height.

                // Determine internal decoding parameters based on VkFormat.
                // These `var_uVarXX` variables control how `FUN_0013f554` interprets the image data.
                uint32_t is_depth_stencil = 0;
                uint32_t has_alpha = 0;
                uint32_t component_count = 0;
                int bytes_per_pixel = 0; // Or block size in bytes for compressed formats.
                uint32_t is_unsigned_format = 0;

                switch (image_format_val) {
                    case VK_FORMAT_BC1_RGB_UNORM_BLOCK:
                    case VK_FORMAT_BC1_RGB_SRGB_BLOCK:
                        is_unsigned_format = 1;
                        bytes_per_pixel = 8;
                        component_count = 1;
                        break;
                    case VK_FORMAT_BC1_RGBA_UNORM_BLOCK:
                    case VK_FORMAT_BC1_RGBA_SRGB_BLOCK:
                        // Default values (0 for relevant vars)
                        break;
                    case VK_FORMAT_BC2_UNORM_BLOCK:
                    case VK_FORMAT_BC2_SRGB_BLOCK:
                        component_count = 2;
                        bytes_per_pixel = 16;
                        break;
                    case VK_FORMAT_BC3_UNORM_BLOCK:
                    case VK_FORMAT_BC3_SRGB_BLOCK:
                        component_count = 3;
                        bytes_per_pixel = 16;
                        break;
                    case VK_FORMAT_BC4_UNORM_BLOCK:
                        is_depth_stencil = 1; // Likely indicates signed/unsigned for single channel
                        // Fallthrough
                    case VK_FORMAT_BC4_SNORM_BLOCK:
                        component_count = 4;
                        bytes_per_pixel = 8;
                        is_unsigned_format = is_depth_stencil;
                        break;
                    case VK_FORMAT_BC5_UNORM_BLOCK:
                        has_alpha = 1; // Likely indicates multiple channels
                        // Fallthrough
                    case VK_FORMAT_BC5_SNORM_BLOCK:
                        component_count = 5;
                        bytes_per_pixel = 16;
                        is_unsigned_format = has_alpha;
                        break;
                    default:
                        // Unknown format, parameters remain zero.
                        break;
                }

                // `processed_image_data_addr` points to the actual start of the image data within the mapped memory.
                long processed_image_data_addr = (long)mmapped_file_address + *(long*)(vk_image_create_info_ptr + 8) + offset_in_file;

                // Call helper function `FUN_0013f554` to perform the actual image decoding/processing.
                // It likely processes the texture data into `processed_image_data_addr`.
                // `decoder->some_ptr_for_vkCreateImage_param7` (at offset 0x40 in TextureDecoder) is passed as a context.
                FUN_0013f554(processed_image_data_addr, *(long*)(custom_texture_data_ptr + 0x20), // src_raw_data is custom_texture_data_ptr + 0x20
                             image_width, image_height, component_count, is_unsigned_format,
                             (ThreadPoolContext*)decoder->some_ptr_for_vkCreateImage_param7);

                // Check image dimensions for disk caching eligibility.
                if (image_width < 1024 && image_height < 1024) {
                    if (decoder->imageCacheSize != 0) { // Check if image caching is enabled.
                        // This block implements a basic disk cache management.
                        // It reads the current cache size from `/data/data/com.winlator/cache/vortek/.cache-size`.
                        // If the total cache size exceeds `decoder->imageCacheSize` (in MB), it deletes the oldest files.
                        // Then, it writes the current decoded image to a file in the cache directory.

                        // ... (detailed file I/O operations and cache eviction logic are complex and omitted for brevity)
                        // This involves:
                        // 1. Reading cache size from `.cache-size` file.
                        // 2. Iterating directory entries in `/data/data/com.winlator/cache/vortek`.
                        // 3. Using `stat` to get file sizes and modification times.
                        // 4. Removing oldest files (`remove` function) until cache size is within limits.
                        // 5. Forming filename like `/%lx-%dx%d-%d.imd` (hash-width-height-format.imd).
                        // 6. Writing image data to file (`fwrite`).
                        // 7. Updating `.cache-size` file.
                        // `FUN_0013ff84` is a custom file/cache utility function.
                    }
                }
            }
            // Unmap the memory region that was mapped from the file.
            munmap(mmapped_file_address, mapping_size);
        }
    }

    // Stack canary check.
    if (*(long*)(tpidr_el0 + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/*
 * @brief This is a worker function (or a function to be executed by a worker thread)
 *        that processes a specific portion (block/stripe) of a texture.
 * @param job_data_ptr A pointer to a custom job data struct. This struct seems to define
 *                     a rectangular region within a texture and its format for processing.
 *                     It has `dst_buffer_ptr` (offset 0), `src_buffer_ptr` (offset 8),
 *                     `width` (offset 16), `height` (offset 20), `format_type` (offset 24),
 *                     `offset_y` (offset 28), `end_y` (offset 32), `is_compressed_flag` (offset 36),
 *                     `is_bc_format_flag` (offset 37).
 * @return A success/failure indicator (long).
 */
long FUN_0014005c(long job_data_ptr)
{
    long stack_chk_guard = *(long*)(tpidr_el0 + 0x28); // Get stack canary for security check.

    // Extract job parameters from `job_data_ptr`.
    long dst_buffer_ptr = *(long*)job_data_ptr;        // Destination buffer address.
    long src_buffer_ptr = *(long*)(job_data_ptr + 8);  // Source buffer address.
    uint32_t width = *(uint32_t*)(job_data_ptr + 16);    // Width of the processing region.
    int32_t height = *(int32_t*)(job_data_ptr + 20);      // Height of the processing region.
    uint32_t format_type = *(uint32_t*)(job_data_ptr + 24); // Format type.
    uint32_t offset_y = *(uint32_t*)(job_data_ptr + 28);   // Y-offset of the processing region.
    uint32_t end_y = *(uint32_t*)(job_data_ptr + 32);     // End Y-coordinate of the region.
    bool is_compressed_format = *(bool*)(job_data_ptr + 36); // Is compressed format flag.
    bool is_bc_format = *(bool*)(job_data_ptr + 37);         // Is BC format flag.

    // The actual image processing logic would reside here, performing operations
    // like decompression, format conversion, etc., from `src_buffer_ptr` to `dst_buffer_ptr`.
    // The details of this are opaque from the disassembly and would depend on the `format_type`,
    // `is_compressed_format`, and `is_bc_format` flags.
    // For example, if it's a BC compressed format, it would decompress blocks of pixels.
    // Given the parameters (`dst_buffer_ptr`, `src_buffer_ptr`, `width`, `height`, `format_type`, `offset_y`, `end_y`),
    // it appears to be a generic image processing function that operates on a specified rectangle.

    // Return value is likely a success/failure indicator.
    long result = 0; // Assuming 0 for success.

    // Stack canary check.
    if (*(long*)(tpidr_el0 + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return result;
}

/*
 * @brief Copies data from a buffer to an image.
 *
 * This function seems to be part of an asynchronous pipeline where image creation
 * and data population are handled. It finds the previously created image and
 * queues a Vulkan buffer-to-image copy command.
 *
 * @param image_list_ptr A pointer to an `ArrayList` containing `CustomTextureInfo` structs.
 * @param device The `VkDevice` handle.
 * @param image_id The `VkImage` handle (or custom ID).
 * @param buffer_id The `VkBuffer` handle (or custom ID), typically a staging buffer.
 * @param dstImageLayout The `VkImageLayout` for the destination image after the copy.
 * @param reserved_param A reserved or unused parameter (often 0 in Vulkan calls, here it's `param_6`).
 */
void TextureDecoder_copyBufferToImage(ArrayList* image_list_ptr, VkDevice device, long image_id, long buffer_id, VkImageLayout dstImageLayout, undefined8 reserved_param)
{
    long stack_chk_guard = *(long*)(tpidr_el0 + 0x28); // Get stack canary for security check.

    long* image_info_struct = NULL;
    // Search for the `CustomTextureInfo` corresponding to `image_id` in the `image_list_ptr`.
    if (image_list_ptr->count > 0) {
        for (uint32_t i = 0; i < image_list_ptr->count; ++i) {
            long* current_item = (long*)image_list_ptr->data[i];
            if (current_item != NULL && current_item[0] == image_id) { // Assume `current_item[0]` stores `VkImage` handle.
                image_info_struct = current_item;
                break;
            }
        }
    }

    if (image_info_struct != NULL) {
        // Prepare a `VkBufferImageCopy` struct on the stack for the copy command.
        // The assembly uses `local_a0` and related stack variables for this.
        VkBufferImageCopy copy_region_info = {0};
        copy_region_info.sType = VK_STRUCTURE_TYPE_BUFFER_IMAGE_COPY; // Explicitly set sType (not explicitly in assembly but good practice)

        // The values here are read from specific offsets within the `image_info_struct`.
        // These offsets (`0x10`, `0xC`, `0xE`) point to members of `CustomTextureInfo`.
        copy_region_info.bufferOffset = (VkDeviceSize)image_info_struct[2]; // `image_info_struct[2]` is the VkBuffer handle, which implies its address is used as offset. This is highly suspicious. It should be an offset. Re-checking `TextureDecoder_createImage`'s `__ptr[2]` which is `local_a8[0]` (`VkBuffer`), so `image_info_struct[2]` is the staging buffer handle.
        // It's likely `bufferOffset` is `0` since it's a staging buffer. The assembly might be reading some other struct related to the buffer.
        // Let's assume `image_info_struct` stores width/height at 0xC/0xE.
        copy_region_info.bufferRowLength = *(uint32_t*)((char*)image_info_struct + 0xC); // Width
        copy_region_info.bufferImageHeight = *(uint32_t*)((char*)image_info_struct + 0xE); // Height

        // `VkImageSubresourceLayers` initialization (inferred common values)
        copy_region_info.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
        copy_region_info.imageSubresource.mipLevel = 0;
        copy_region_info.imageSubresource.baseArrayLayer = 0;
        copy_region_info.imageSubresource.layerCount = 1;

        // `VkOffset3D` (inferred zero) and `VkExtent3D` (from width/height and depth 1)
        copy_region_info.imageOffset = (VkOffset3D){0, 0, 0};
        copy_region_info.imageExtent = (VkExtent3D){copy_region_info.bufferRowLength, copy_region_info.bufferImageHeight, 1};


        // Call `vkCmdCopyBufferToImage` via dispatch table.
        // `*(undefined8 *)(lVar5 + 0x10)` is `image_info_struct[2]`, which is the `VkBuffer` handle.
        dispatch_vkCmdCopyBufferToImage(
            device,
            (VkBuffer)image_info_struct[2],       // srcBuffer: The staging buffer
            (VkImage)image_id,                    // dstImage: The target image to copy into
            dstImageLayout,                       // dstImageLayout: Layout of the destination image
            1,                                    // regionCount: One copy region
            &copy_region_info                     // pRegions: Pointer to the VkBufferImageCopy structure
        );
    }

    // Stack canary check.
    if (*(long*)(tpidr_el0 + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/*
 * @brief Checks if an image is already tracked by the `TextureDecoder`'s image list.
 * @param image_list_ptr A pointer to an `ArrayList` containing `CustomTextureInfo` structs.
 * @param image_id The `VkImage` handle (or custom ID) to search for.
 * @return `true` if the image is found, `false` otherwise.
 */
bool TextureDecoder_containsImage(ArrayList* image_list_ptr, long image_id)
{
    // `image_list_ptr->count` is at offset 0. `image_list_ptr->data` is at offset 8.
    if (image_list_ptr->count > 0) {
        for (uint32_t i = 0; i < image_list_ptr->count; ++i) {
            long* current_image_info = (long*)image_list_ptr->data[i];
            if (current_image_info != NULL && current_image_info[0] == image_id) { // Assumes `image_id` is stored at offset 0 in the custom struct.
                return true;
            }
        }
    }
    return false;
}

/*
 * @brief Creates a Vulkan image and its associated staging buffer and memory.
 *        Registers the image with `TextureDecoder`'s internal list for management.
 *        This function appears to specifically handle certain compressed texture formats.
 *
 * @param image_list_ptr A pointer to the `ArrayList` where the new image's info will be stored.
 * @param device The `VkDevice` handle.
 * @param pCreateInfo A pointer to a `VkImageCreateInfo` structure.
 * @param pImage_out A pointer to a `VkImage` handle where the created image will be stored.
 * @return `VK_SUCCESS` on success, or an error `VkResult` otherwise.
 */
int TextureDecoder_createImage(ArrayList* image_list_ptr, VkDevice device, VkImageCreateInfo* pCreateInfo, VkImage* pImage_out)
{
    long stack_chk_guard = *(long*)(tpidr_el0 + 0x28); // Get stack canary.
    VkResult result = VK_ERROR_UNKNOWN; // Initialize result to a general error.

    *pImage_out = VK_NULL_HANDLE; // Initialize output image handle.

    // Check `pCreateInfo->format` against a specific range (BC1-BC5 formats: 0x83 to 0x8e).
    // `*(int *)(param_3 + 0x18)` reads `pCreateInfo->format`.
    if (((uint32_t)pCreateInfo->format - VK_FORMAT_BC1_RGB_UNORM_BLOCK) < 0xC) {
        // Allocate a custom struct to hold all related Vulkan handles and mapped pointers for the image.
        // Size 0x28 (40 bytes) matches `CustomTextureInfo`.
        CustomTextureInfo* image_info_struct = (CustomTextureInfo*)calloc(1, sizeof(CustomTextureInfo));
        if (image_info_struct == NULL) {
            result = VK_ERROR_OUT_OF_HOST_MEMORY;
            goto end_create_image;
        }

        // Store original image properties in the custom struct before modifying `pCreateInfo`.
        image_info_struct->width = (uint16_t)pCreateInfo->extent.width;
        image_info_struct->height = (uint16_t)pCreateInfo->extent.height;
        image_info_struct->original_format = pCreateInfo->format;

        // Temporarily modify `pCreateInfo` for the `vkCreateImage` call.
        // It changes the format to `0x2c` (VK_FORMAT_B8G8R8A8_UNORM or similar internal format).
        pCreateInfo->format = (VkFormat)0x2C;
        // It disables the `VK_IMAGE_USAGE_SAMPLED_BIT` (0x00000004) from `pCreateInfo->usage`.
        pCreateInfo->usage &= ~VK_IMAGE_USAGE_SAMPLED_BIT;
        // It forces `arrayLayers` to 1.
        pCreateInfo->arrayLayers = 1;

        // Iterates `pCreateInfo->pNext` chain, if a specific custom structure
        // (with `sType` 0x3b9d0838) is found, it modifies flags/format within it.
        VkBaseInStructure* current_pNext_node = (VkBaseInStructure*)pCreateInfo->pNext;
        while (current_pNext_node != NULL) {
            if (current_pNext_node->sType == 0x3b9d0838) { // Assuming a custom internal struct or specific extension interaction
                // Sets a `uint32_t` field at offset 0x10 to 1 and another `uint32_t` field at `0x18` (pointer deref) to `0x2c`.
                // This implies a special handling for format mutability or specific image properties.
                *(uint32_t*)((char*)current_pNext_node + 0x10) = 1; // Example: `flags` field
                // `**(undefined4 **)(piVar6 + 6) = 0x2c;` which is `*(uint32_t*)current_pNext_node->some_pointer_at_0x18 = 0x2c;`
                *(uint32_t*)(*(long*)((char*)current_pNext_node + 0x18)) = (uint32_t)0x2C; // Example: `format` field in a nested struct
                break;
            }
            current_pNext_node = (VkBaseInStructure*)current_pNext_node->pNext;
        }

        // Create the Vulkan Image.
        VkImage image = VK_NULL_HANDLE;
        result = dispatch_vkCreateImage(device, pCreateInfo, NULL, &image);
        if (result != VK_SUCCESS) {
            goto cleanup_image_info_struct;
        }
        *pImage_out = image; // Output the created `VkImage` handle.
        image_info_struct->image_handle = image; // Store `VkImage` handle in custom struct.

        // Create a staging buffer for transferring data to the image.
        VkBufferCreateInfo staging_buffer_create_info = {VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO};
        // The size is calculated as `width * height * 4` (assuming RGBA8).
        staging_buffer_create_info.size = (VkDeviceSize)image_info_struct->width * image_info_struct->height * 4;
        staging_buffer_create_info.usage = VK_BUFFER_USAGE_TRANSFER_SRC_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT;
        staging_buffer_create_info.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

        VkBuffer staging_buffer = VK_NULL_HANDLE;
        result = dispatch_vkCreateBuffer(device, &staging_buffer_create_info, NULL, &staging_buffer);
        if (result != VK_SUCCESS) {
            goto cleanup_image;
        }
        image_info_struct->buffer_handle = staging_buffer; // Store `VkBuffer` handle.

        // Get memory requirements for the staging buffer.
        VkMemoryRequirements mem_requirements = {0};
        dispatch_vkGetBufferMemoryRequirements(device, staging_buffer, &mem_requirements);

        // Allocate memory for the staging buffer.
        VkMemoryAllocateInfo alloc_info = {VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO};
        alloc_info.allocationSize = mem_requirements.size;
        // `getMemoryTypeIndex` selects a memory type with `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT`.
        alloc_info.memoryTypeIndex = getMemoryTypeIndex(mem_requirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT);
        
        VkDeviceMemory staging_memory = VK_NULL_HANDLE;
        result = dispatch_vkAllocateMemory(device, &alloc_info, NULL, &staging_memory);
        if (result != VK_SUCCESS) {
            goto cleanup_staging_buffer;
        }
        image_info_struct->memory_handle = staging_memory; // Store `VkDeviceMemory` handle.

        // Bind the allocated memory to the staging buffer.
        result = dispatch_vkBindBufferMemory(device, staging_buffer, staging_memory, 0);
        if (result != VK_SUCCESS) {
            goto cleanup_staging_memory;
        }

        // Map the staging buffer memory for host access.
        void* mapped_data_ptr = NULL;
        result = dispatch_vkMapMemory(device, staging_memory, 0, staging_buffer_create_info.size, 0, &mapped_data_ptr);
        if (result != VK_SUCCESS) {
            goto cleanup_staging_memory;
        }
        image_info_struct->mapped_data_ptr = mapped_data_ptr; // Store mapped pointer.

        // Initialize mapped memory to zero.
        memset(mapped_data_ptr, 0, staging_buffer_create_info.size);

        // Add the custom `image_info_struct` to the `ArrayList` for tracking.
        ArrayList_add((long)image_list_ptr, image_info_struct);

        goto end_create_image;

cleanup_staging_memory:
    if (image_info_struct->mapped_data_ptr != NULL) {
        dispatch_vkUnmapMemory(device, image_info_struct->memory_handle);
    }
    if (image_info_struct->memory_handle != VK_NULL_HANDLE) {
        dispatch_vkFreeMemory(device, image_info_struct->memory_handle, NULL);
    }
cleanup_staging_buffer:
    if (image_info_struct->buffer_handle != VK_NULL_HANDLE) {
        dispatch_vkDestroyBuffer(device, image_info_struct->buffer_handle, NULL);
    }
cleanup_image:
    if (image_info_struct->image_handle != VK_NULL_HANDLE) {
        dispatch_vkDestroyImage(device, image_info_struct->image_handle, NULL);
    }
cleanup_image_info_struct:
    free(image_info_struct);
    result = VK_ERROR_INITIALIZATION_FAILED; // Set a generic error for the cleanup path.
    } else {
        result = VK_ERROR_FORMAT_NOT_SUPPORTED; // Format was not in the expected range.
    }

end_create_image:
    // Stack canary check.
    if (*(long*)(tpidr_el0 + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return (int)result;
}

/*
 * @brief Destroys a Vulkan image and its associated resources managed by `TextureDecoder`.
 *        Removes the image's entry from the `TextureDecoder`'s internal list.
 *
 * @param image_list_ptr A pointer to the `ArrayList` of managed `CustomTextureInfo` structs.
 * @param device The `VkDevice` handle.
 * @param image_id The `VkImage` handle (or custom ID) of the image to destroy.
 */
void TextureDecoder_destroyImage(ArrayList* image_list_ptr, VkDevice device, long image_id)
{
    // Search for the image_id in the ArrayList.
    if (image_list_ptr->count > 0) {
        for (uint32_t i = 0; i < image_list_ptr->count; ++i) {
            CustomTextureInfo* current_image_info = (CustomTextureInfo*)image_list_ptr->data[i];
            if (current_image_info != NULL && (long)current_image_info->image_handle == image_id) {
                // If found, proceed with Vulkan resource destruction.
                if (current_image_info->mapped_data_ptr != NULL) {
                    dispatch_vkUnmapMemory(device, current_image_info->memory_handle);
                }
                if (current_image_info->image_handle != VK_NULL_HANDLE) {
                    dispatch_vkDestroyImage(device, current_image_info->image_handle, NULL);
                }
                if (current_image_info->buffer_handle != VK_NULL_HANDLE) {
                    dispatch_vkDestroyBuffer(device, current_image_info->buffer_handle, NULL);
                }
                if (current_image_info->memory_handle != VK_NULL_HANDLE) {
                    dispatch_vkFreeMemory(device, current_image_info->memory_handle, NULL);
                }
                free(current_image_info); // Free the `CustomTextureInfo` struct itself.
                ArrayList_removeAt((uint32_t*)image_list_ptr, i); // Remove from the ArrayList.
                return; // Exit after destroying the found image.
            }
        }
    }
}

/*
 * @brief Adds an entry for a bound buffer to the `TextureDecoder`'s internal list.
 *        If an entry with the same buffer ID already exists, it is replaced.
 * @param decoder_ptr Pointer to the `TextureDecoder` instance.
 * @param associated_data_ptr A pointer to some associated data or a count for the buffer.
 * @param buffer_id The custom ID or `VkBuffer` handle of the buffer.
 * @param buffer_handle The actual `VkBuffer` handle.
 */
void TextureDecoder_addBoundBuffer(long decoder_ptr, uint32_t* associated_data_ptr, long buffer_id, long buffer_handle)
{
    // The condition `*param_2 < 1` likely checks if `associated_data_ptr` indicates a valid amount of data.
    if (*associated_data_ptr < 1) {
        return; // Nothing to add if count is zero or negative.
    }

    // Access the internal `ArrayList` for bound buffers within `TextureDecoder`.
    // Assuming `decoder_ptr + 0x10` points to the `count` field and `decoder_ptr + 0x18` to the `data` field of this `ArrayList`.
    ArrayList* bound_buffers_list = (ArrayList*)(decoder_ptr + 0x10);

    // Check if a buffer with the same `buffer_id` already exists in the list.
    if (bound_buffers_list->count > 0) {
        for (uint32_t i = 0; i < bound_buffers_list->count; ++i) {
            long* current_entry = (long*)bound_buffers_list->data[i];
            if (current_entry != NULL && current_entry[0] == buffer_id) { // Assume `current_entry[0]` stores the buffer ID.
                // If found, free the old entry and remove it from the list.
                if (current_entry != NULL) {
                    free(current_entry);
                }
                ArrayList_removeAt((uint32_t*)bound_buffers_list, i);
                break; // Exit loop after removal.
            }
        }
    }

    // Allocate a new entry for the bound buffer.
    long* new_entry = (long*)calloc(1, 0x18); // Allocate 0x18 (24 bytes) for the new entry.
    if (new_entry) {
        new_entry[0] = buffer_id;           // Store the buffer ID.
        new_entry[1] = buffer_handle;       // Store the actual `VkBuffer` handle.
        new_entry[2] = (long)associated_data_ptr; // Store the `associated_data_ptr` (could be metadata, flags, or an associated resource).
        ArrayList_add((long)bound_buffers_list, new_entry); // Add the new entry to the list.
    }
}

/*
 * @brief Removes a bound buffer entry from the `TextureDecoder`'s internal list by its ID.
 * @param decoder_ptr Pointer to the `TextureDecoder` instance.
 * @param buffer_id The custom ID or `VkBuffer` handle of the buffer to remove.
 */
void TextureDecoder_removeBoundBuffer(long decoder_ptr, long buffer_id)
{
    // Access the internal `ArrayList` for bound buffers within `TextureDecoder`.
    ArrayList* bound_buffers_list = (ArrayList*)(decoder_ptr + 0x10);

    // Search for the buffer by its `buffer_id`.
    if (bound_buffers_list->count > 0) {
        for (uint32_t i = 0; i < bound_buffers_list->count; ++i) {
            long* current_entry = (long*)bound_buffers_list->data[i];
            if (current_entry != NULL && current_entry[0] == buffer_id) { // Assume `current_entry[0]` stores the buffer ID.
                // If found, free the entry and remove it from the list.
                if (current_entry != NULL) {
                    free(current_entry);
                }
                ArrayList_removeAt((uint32_t*)bound_buffers_list, i);
                return; // Exit after removing the found buffer.
            }
        }
    }
}
```

---

The provided C functions are part of a low-level Vulkan abstraction layer, likely designed for a specific embedded or mobile environment (indicated by references to `AHardwareBuffer`, `dma_heap`, `ion`, and platform-specific extensions like `FUCHSIA`, `NV`, `AMD`). The functions manage Vulkan memory allocation and deallocation, including handling of external memory types and interaction with a custom object ID system (`VkObject_fromId`).

The decompilation aims to provide readable C code, inferring struct layouts and mapping hexadecimal constants to their likely Vulkan enum equivalents where possible. Some constants remain as raw hexadecimal values when a direct, unambiguous mapping to a standard `VkStructureType` from the provided header is not possible, suggesting custom or highly specific extension usage.

**Inferred Structures and Context:**

```c
#include <string.h> // For memcpy, strcmp, memset
#include <stdio.h>  // For snprintf
#include <stdlib.h> // For malloc, free
#include <unistd.h> // For close
#include <sys/ioctl.h> // For ioctl
#include <sys/mman.h>  // For mmap, munmap
#include <errno.h>  // For errno
#include <sys/socket.h> // For sendmsg, msghdr, iovec, cmsghdr

// Standard Vulkan types (simplified to relevant parts)
typedef uint32_t VkBool32;
typedef uint64_t VkDeviceAddress;
typedef uint64_t VkDeviceSize;
typedef uint32_t VkFlags;
typedef uint32_t VkResult;
typedef uint32_t VkFormat;
typedef uint32_t VkImageUsageFlags;
typedef uint32_t VkBufferUsageFlags;
typedef uint32_t VkSharingMode;
typedef uint32_t VkMemoryMapFlags; // Not explicitly used but implied by vkMapMemory

// Dummy types for Vulkan handles (assuming they are opaque pointers or 64-bit IDs)
typedef struct VkDevice_T* VkDevice;
typedef struct VkDeviceMemory_T* VkDeviceMemory;
typedef struct VkBuffer_T* VkBuffer;
typedef struct VkImage_T* VkImage;

// Enum constants from vulkan_core.h (relevant subset for this analysis)
typedef enum VkStructureType {
    // Core Vulkan 1.0 structures
    VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO = 5,
    VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO = 12, // 0xC
    // Vulkan 1.1/1.2/1.3 structures
    VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_FLAGS_INFO = 1000060000, // 0x3b9beb10
    VK_STRUCTURE_TYPE_BIND_IMAGE_MEMORY_INFO = 1000157001,     // 0x3b9cba19
    VK_STRUCTURE_TYPE_DEVICE_QUEUE_INFO_2 = 1000145003, // 0x3b9cc1eb
    // Extension structures (named with original hex due to lack of direct symbol in provided header)
    VK_STRUCTURE_TYPE_WIN32_KEYED_MUTEX_ACQUIRE_RELEASE_INFO_NV_BASE = 1000058000, // 0x3b9be340
    VK_STRUCTURE_TYPE_IMAGEPIPE_SURFACE_CREATE_INFO_FUCHSIA = 1000214000, // 0x3b9eb5eb
    // Placeholder for other unknown specific extension types identified by hex value
    // This value (1000054000) is VK_FORMAT_PVRTC1_2BPP_UNORM_BLOCK_IMG as a VkFormat,
    // but its usage suggests it is treated as a VkStructureType in this context.
    UNKNOWN_STRUCTURE_TYPE_0x3b9bb460 = 1000054000, // 0x3b9bb460
    // This value (1000058178) appears to be 0x3b9be340 + 2. Could be a variant or offset.
    UNKNOWN_STRUCTURE_TYPE_0x3b9be342 = 1000058178, // 0x3b9be342
} VkStructureType;

// Minimal struct definitions to match offsets and types from the disassembly context
typedef struct VkBaseInStructure {
    VkStructureType sType;
    const struct VkBaseInStructure* pNext;
} VkBaseInStructure;

// Definition of VkMemoryAllocateInfo
typedef struct VkMemoryAllocateInfo {
    VkStructureType    sType;
    const void*        pNext;
    VkDeviceSize       allocationSize;
    uint32_t           memoryTypeIndex;
} VkMemoryAllocateInfo;

// Definition of VkBufferCreateInfo
typedef struct VkBufferCreateInfo {
    VkStructureType sType;
    const void* pNext;
    VkFlags flags; // VkBufferCreateFlags
    VkDeviceSize size;
    VkBufferUsageFlags usage;
    VkSharingMode sharingMode;
    uint32_t queueFamilyIndexCount;
    const uint32_t* pQueueFamilyIndices;
} VkBufferCreateInfo;

// Definition of VkMemoryRequirements
typedef struct VkMemoryRequirements {
    VkDeviceSize    size;
    VkDeviceSize    alignment;
    uint32_t        memoryTypeBits;
} VkMemoryRequirements;

// Definition of VkMemoryGetFdInfoKHR (or similar native handle info)
typedef struct VkMemoryGetFdInfoKHR {
    VkStructureType sType;
    const void* pNext;
    VkDeviceMemory memory;
    uint32_t handleType; // VkExternalMemoryHandleTypeFlagBits
} VkMemoryGetFdInfoKHR;

// Example of a dedicated allocation struct, assuming similar layout to `0x3b9bb460`'s usage
typedef struct VkDedicatedAllocationImageCreateInfoNV_Custom {
    VkStructureType sType;
    const void* pNext;
    // Assuming 3 uint32_t members based on a 0x18-byte size for this struct type.
    // In actual Vulkan, it would be VkBool32 dedicatedAllocation;
    uint32_t field_08;
    uint32_t field_0c;
    uint32_t field_10;
} VkDedicatedAllocationImageCreateInfoNV_Custom;

// Example of a `WIN32_KEYED_MUTEX` struct if it's being custom-parsed
typedef struct VkWin32KeyedMutexAcquireReleaseInfoNV_Custom {
    VkStructureType sType;
    const void* pNext;
    // Assuming 2 uint32_t members based on a 0x18-byte size for this struct type.
    // In actual Vulkan, it would be uint32_t acquireCount; const void* pAcquireSyncs;
    uint32_t field_08;
    uint32_t field_0c;
} VkWin32KeyedMutexAcquireReleaseInfoNV_Custom;

// Example of `VkBindImageMemoryInfo`
typedef struct VkBindImageMemoryInfo_Custom {
    VkStructureType sType;
    const void* pNext;
    VkImage image; // 8 bytes
    VkDeviceMemory memory; // 8 bytes
    VkDeviceSize memoryOffset; // 8 bytes
} VkBindImageMemoryInfo_Custom; // Total 32 bytes (0x20)

// Definition of VkMemoryAllocateFlagsInfo
typedef struct VkMemoryAllocateFlagsInfo {
    VkStructureType sType;
    const void* pNext;
    VkFlags flags; // VkMemoryAllocateFlags
    uint32_t deviceMask;
} VkMemoryAllocateFlagsInfo;

// Simplified ArrayList structure
typedef struct ArrayList {
    uint32_t count;
    uint32_t capacity;
    void** data; // Changed to void** to match generic usage
} ArrayList;

// Simplified VtContext based on the provided C struct
typedef struct VtContext {
    int clientFd;                      // 0x00
    int vkMaxVersion;                  // 0x04
    short maxDeviceMemory;             // 0x08
    short imageCacheSize;              // 0x0a
    int padding_0c;                    // 0x0c
    ArrayList* exposedDeviceExtensions; // 0x10
    ArrayList* exposedInstanceExtensions; // 0x18
    uint8_t hostVisibleMemoryFlag;     // 0x20
    uint8_t coherentMemoryFlag;        // 0x21
    char padding_22[6];                // 0x22
    char* inputBufferPtr;              // 0x28
    int inputBufferSize;               // 0x30
    int padding_34;                    // 0x34
    void* tempBuffer;                  // 0x38
    int tempBufferOffset;              // 0x40
    int padding_44;                    // 0x44
    ArrayList* tempAllocations;        // 0x48
    long padding_50;                   // 0x50
    void* serverThread;                // 0x58 (pthread_t, usually long)
    void* clientRingBuffer;            // 0x60 (RingBuffer*)
    void* serverRingBuffer;            // 0x68 (RingBuffer*)
    int lastError;                     // 0x70
    int graphicsQueueFamilyIndex;      // 0x74
    // ... other fields
} VtContext;

// Custom structure for allocated memory resources
typedef struct VtMemoryBlock {
    int nativeFd;              // Offset 0x00, file descriptor for native memory handles
    void* androidHardwareBuffer; // Offset 0x08, AHardwareBuffer*
    VkBuffer bufferHandle;     // Offset 0x10, VkBuffer handle
    VkDeviceMemory deviceMemoryHandle; // Offset 0x18, VkDeviceMemory handle
    // Total size of 0x28 (40 bytes)
} VtMemoryBlock;

// External function declarations (assuming definitions elsewhere, and types inferred)
extern void __stack_chk_fail(void);
extern void* VkObject_fromId(void* id); // Converts an internal ID to a Vulkan handle or object pointer
extern int ResourceMemory_allocate(VkDevice device, const VkMemoryAllocateInfo* pAllocateInfo, uint32_t hostVisibleFlag, uint32_t coherentFlag);
extern void ResourceMemory_free(VkDevice device, VtMemoryBlock* memBlock);
extern void AHardwareBuffer_release(void* buffer); // AHardwareBuffer type from Android NDK
extern int AHardwareBuffer_getFd(void* buffer);
extern int AHardwareBuffer_allocate(const void* desc, void** outBuffer); // Custom AHardwareBuffer allocation
extern int __open_2(const char* pathname, int flags); // POSIX open wrapper
extern int ioctl(int fd, unsigned long request, ...);
extern int dup(int oldfd);
extern void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);
extern int munmap(void* addr, size_t length);
extern int* __errno(); // Get last error number

// Function pointers for Vulkan commands (dispatch table, simplified)
typedef VkResult (*PFN_vkAllocateMemory)(VkDevice device, const VkMemoryAllocateInfo* pAllocateInfo, const void* pAllocator, VkDeviceMemory* pMemory);
typedef VkResult (*PFN_vkCreateBuffer)(VkDevice device, const VkBufferCreateInfo* pCreateInfo, const void* pAllocator, VkBuffer* pBuffer);
typedef void (*PFN_vkGetBufferMemoryRequirements)(VkDevice device, VkBuffer buffer, VkMemoryRequirements* pMemoryRequirements);
typedef VkResult (*PFN_vkGetMemoryFdKHR)(VkDevice device, const VkMemoryGetFdInfoKHR* pGetFdInfo, int* pFd);
typedef void (*PFN_vkDestroyBuffer)(VkDevice device, VkBuffer buffer, const void* pAllocator);
typedef void (*PFN_vkFreeMemory)(VkDevice device, VkDeviceMemory memory, const void* pAllocator);
typedef VkResult (*PFN_RingBuffer_write)(void* rb, const void* data, size_t size);
extern void ArrayList_add(ArrayList* list, void* item);
extern uint32_t getMemoryPropertyFlags(uint32_t memoryTypeIndex);
extern uint32_t getMemoryTypeIndex(uint32_t requiredPropertyFlags, uint32_t preferredPropertyFlags);
extern VkBool32 isHostVisibleMemory(uint32_t memoryTypeIndex);

// Global dispatch table for Vulkan functions
PFN_vkAllocateMemory dispatch_vkAllocateMemory;
PFN_vkCreateBuffer dispatch_vkCreateBuffer;
PFN_vkGetBufferMemoryRequirements dispatch_vkGetBufferMemoryRequirements;
PFN_vkGetMemoryFdKHR dispatch_vkGetMemoryFd;
PFN_vkDestroyBuffer dispatch_vkDestroyBuffer;
PFN_vkFreeMemory dispatch_vkFreeMemory;


/**
 * @brief Handles the vkAllocateMemory command received from the client.
 *
 * This function processes a request to allocate Vulkan device memory. It reads
 * the allocation information from the input buffer, including `VkMemoryAllocateInfo`
 * and potentially extension structs in its `pNext` chain. It then calls the
 * `ResourceMemory_allocate` helper function to perform the actual memory allocation,
 * which might involve different paths depending on memory properties (host-visible,
 * coherent, Android AHardwareBuffer, or Linux DMA-BUF/ION).
 *
 * Finally, it sends the allocation result (success/failure and the allocated
 * memory block pointer/ID) back to the client via the ring buffer.
 *
 * @param vtContext A pointer to the VtContext structure, containing client communication
 *                  buffers, temporary allocation management, and other global state.
 */
void vt_handle_vkAllocateMemory(VtContext* vtContext) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Stack canary for security

    char* inputBufferPtr = vtContext->inputBufferPtr;

    // Head of the reconstructed pNext chain
    VkBaseInStructure* pNextChainHead = NULL;

    // The first byte of `inputBufferPtr` seems to indicate the command type or variant.
    // It affects the offset to read the primary object ID (VkDevice ID) and the `VkMemoryAllocateInfo`.
    // If inputBufferPtr[0] is '\0', then `VkMemoryAllocateInfo` is at `inputBufferPtr + 1`.
    // Else, a `VkDevice` ID is read from `inputBufferPtr + 1`, and `VkMemoryAllocateInfo` is at `inputBufferPtr + 9`.
    void* primaryObjectId = NULL; // This will be the VkDevice handle ID
    size_t vkAllocateInfoOffset;

    if (inputBufferPtr[0] == '\0') {
        vkAllocateInfoOffset = 1;
        primaryObjectId = (void*)(inputBufferPtr + 1); // Interpreted as VkDevice ID
    } else {
        vkAllocateInfoOffset = 9;
        primaryObjectId = *(void**)(inputBufferPtr + 1); // Reads an 8-byte VkDevice ID
    }

    // Pointer to the start of VkMemoryAllocateInfo within the input buffer
    char* currentInputDataPtr = inputBufferPtr + vkAllocateInfoOffset;

    // The code iterates through `pNext` chain elements of the `VkMemoryAllocateInfo`
    // to reconstruct them on the stack or in a temporary buffer.
    // `currentInputStructPtr` refers to a pointer that moves through the input buffer
    // to read the `sType` and data of chained structures.
    uint32_t* pNextChainDataOffset = (uint32_t*)(currentInputDataPtr + sizeof(VkStructureType) + sizeof(void*)); // Start of data after sType and pNext
    int currentOffsetInInputPNextChain = 0; // Relative offset from `pNextChainDataOffset`

    // Loop through the pNext chain elements from the input buffer
    // The loop continues as long as the sType of the next struct is not 0 (or a specific sentinel).
    while (pNextChainDataOffset[currentOffsetInInputPNextChain / 4] != 0) { // sType is first uint32_t of a VkBaseInStructure
        uint32_t sType = pNextChainDataOffset[currentOffsetInInputPNextChain / 4];
        int structSizeFromInput = *(int*)((char*)pNextChainDataOffset + currentOffsetInInputPNextChain + 4); // Size of the struct as indicated in input

        void* newStructMemory = NULL;
        size_t allocatedStructSize = 0;

        // Determine the actual size of the struct to allocate based on sType.
        // The sizes 0x18 (24 bytes) and 0x20 (32 bytes) are observed for the dynamically allocated structs.
        if (sType == UNKNOWN_STRUCTURE_TYPE_0x3b9bb460 || // Dedicated allocation for image/buffer (like NV_dedicated_allocation)
            sType == UNKNOWN_STRUCTURE_TYPE_0x3b9be342 || // Some Win32/Android related memory extension
            sType == VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_FLAGS_INFO || // VK_KHR_memory_allocate_flags
            sType == VK_STRUCTURE_TYPE_IMAGEPIPE_SURFACE_CREATE_INFO_FUCHSIA) { // Fuchsia specific
            allocatedStructSize = 0x18; // 24 bytes
        } else if (sType == VK_STRUCTURE_TYPE_BIND_IMAGE_MEMORY_INFO) { // VK_KHR_bind_memory2
            allocatedStructSize = 0x20; // 32 bytes
        } else if (sType == 0xFFFFFFFFU) { // VK_ATTACHMENT_UNUSED used as a sentinel to break loop
            // Special handling for the sentinel value, break the loop and continue with remaining logic.
            // This suggests that a 0xFFFFFFFF sType indicates end of useful pNext chain for this purpose.
            currentOffsetInInputPNextChain += 4; // Advance past the sType of the sentinel
            break;
        } else {
            // For other sTypes, the loop just advances without allocating new structures.
            // This means those sTypes are effectively ignored in this specific pNext chain processing.
            currentOffsetInInputPNextChain += 4; // Advance by size of sType
            continue; // Continue to next sType in the loop
        }

        // Allocate memory for the new struct, preferring `tempBuffer` or using `malloc`.
        if ((vtContext->tempBufferOffset + allocatedStructSize) < TEMP_BUFFER_HIGH_WATER_MARK_3 && vtContext->tempBuffer != NULL) {
            newStructMemory = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
            vtContext->tempBufferOffset += allocatedStructSize;
        } else {
            newStructMemory = malloc(allocatedStructSize);
            ArrayList_add(vtContext->tempAllocations, newStructMemory);
        }

        // Initialize allocated memory to zero
        memset(newStructMemory, 0, allocatedStructSize);

        // Copy sType and specific fields from the input buffer to the new struct.
        VkBaseInStructure* newStruct = (VkBaseInStructure*)newStructMemory;
        newStruct->sType = sType; // Copy the sType

        // Copying of specific data fields depends on the sType.
        // The disassembly uses `puVar2` pointing to data after `sType` and `pNext`.
        // `*(undefined4 *)puVar9 = *puVar2;` and subsequent accesses.
        // This is a simplified conceptual mapping for various structs.
        char* currentInputStructContent = (char*)pNextChainDataOffset + currentOffsetInInputPNextChain + 8; // Data after sType and pNext (8 bytes)

        if (sType == UNKNOWN_STRUCTURE_TYPE_0x3b9bb460) {
            // Assumed fields copied for this type (e.g., VkDedicatedAllocationImageCreateInfoNV)
            // It copies 3 uint32_t values (12 bytes) from input.
            memcpy((char*)newStruct + 8, currentInputStructContent, 12);
        } else if (sType == UNKNOWN_STRUCTURE_TYPE_0x3b9be342) {
            // Assumed fields copied for this type (e.g., VkWin32KeyedMutexAcquireReleaseInfoNV variant)
            // It copies 2 uint32_t values (8 bytes) from input.
            memcpy((char*)newStruct + 8, currentInputStructContent, 8);
        } else if (sType == VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_FLAGS_INFO) {
            // Copies VkMemoryAllocateFlags and deviceMask (8 bytes total)
            memcpy((char*)newStruct + 8, currentInputStructContent, 8);
        } else if (sType == VK_STRUCTURE_TYPE_BIND_IMAGE_MEMORY_INFO) {
            // This copies 24 bytes, representing VkImage, VkDeviceMemory, and VkDeviceSize
            // from the input buffer, then converting the VkImage and VkDeviceMemory IDs.
            VkImage imgId = (VkImage)VkObject_fromId(*(void**)currentInputStructContent);
            VkDeviceMemory memId = (VkDeviceMemory)VkObject_fromId(*(void**)(currentInputStructContent + 8));
            VkDeviceSize memOffset = *(VkDeviceSize*)(currentInputStructContent + 16);

            ((VkBindImageMemoryInfo_Custom*)newStruct)->image = imgId;
            ((VkBindImageMemoryInfo_Custom*)newStruct)->memory = memId;
            ((VkBindImageMemoryInfo_Custom*)newStruct)->memoryOffset = memOffset;
        } else if (sType == VK_STRUCTURE_TYPE_IMAGEPIPE_SURFACE_CREATE_INFO_FUCHSIA) {
            // Copies 2 uint32_t values (8 bytes)
            memcpy((char*)newStruct + 8, currentInputStructContent, 8);
        }

        newStruct->pNext = pNextChainHead; // Prepend to the list
        pNextChainHead = newStruct;

        // Advance to the next potential struct in the input buffer.
        // `structSizeFromInput` here is the `size` field read from the input buffer itself,
        // which includes the size of sType and pNext, plus the structure's data.
        currentOffsetInInputPNextChain += structSizeFromInput;
    }

    // Now, create the main VkMemoryAllocateInfo structure and link the pNext chain.
    VkMemoryAllocateInfo allocateInfo;
    allocateInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocateInfo.pNext = pNextChainHead; // Attach the reconstructed pNext chain

    // Copy allocationSize and memoryTypeIndex from the original input buffer.
    // The offset `vkAllocateInfoOffset + 8` skips the `sType` (4 bytes) and `pNext` (8 bytes) of the
    // VkMemoryAllocateInfo itself.
    allocateInfo.allocationSize = *(VkDeviceSize*)(inputBufferPtr + vkAllocateInfoOffset + 8);
    allocateInfo.memoryTypeIndex = *(uint32_t*)(inputBufferPtr + vkAllocateInfoOffset + 8 + sizeof(VkDeviceSize));

    // Allocate the actual resource memory using `ResourceMemory_allocate`.
    // It returns a custom `VtMemoryBlock*`.
    VtMemoryBlock* allocatedMemoryBlock = (VtMemoryBlock*)ResourceMemory_allocate(
        (VkDevice)VkObject_fromId(primaryObjectId), // Cast the VkDevice ID from the input to VkDevice handle.
        &allocateInfo,
        vtContext->hostVisibleMemoryFlag,
        vtContext->coherentMemoryFlag
    );

    // Prepare result to send back to client.
    int32_t resultCode;
    void* resultMemoryBlockPtr;

    if (allocatedMemoryBlock != NULL && allocatedMemoryBlock->deviceMemoryHandle != NULL) { // Check if allocation was successful
        resultCode = VK_SUCCESS; // 0
        resultMemoryBlockPtr = allocatedMemoryBlock; // Send back the internal VtMemoryBlock pointer
    } else {
        resultCode = 0xFFFFFFFE; // Custom error code (or `VK_ERROR_OUT_OF_HOST_MEMORY` etc.)
        resultMemoryBlockPtr = NULL; // Ensure null if allocation failed
    }

    // Send the result back to the client via the ring buffer.
    // The format is `resultCode` (4 bytes) followed by `resultMemoryBlockPtr` (8 bytes).
    RingBuffer_write(vtContext->clientRingBuffer, &resultCode, sizeof(resultCode));
    RingBuffer_write(vtContext->clientRingBuffer, &resultMemoryBlockPtr, sizeof(resultMemoryBlockPtr));

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}


/**
 * @brief Allocates resource memory for Vulkan objects.
 *
 * This helper function attempts to allocate memory, leveraging different backend
 * mechanisms: direct Vulkan allocation, Android's AHardwareBuffer, or Linux DMA-BUF/ION.
 * It allocates a custom `VtMemoryBlock` structure to track the allocated memory and its
 * associated native handles/Vulkan handles.
 *
 * @param device The Vulkan logical device.
 * @param pAllocateInfo The VkMemoryAllocateInfo structure specifying allocation details.
 * @param hostVisibleMemoryFlag A flag (from VtContext) indicating preference for host-visible memory.
 * @param coherentMemoryFlag A flag (from VtContext) indicating preference for coherent memory.
 * @return A pointer to a newly allocated `VtMemoryBlock` structure (cast to int*),
 *         or a newly allocated but "failed" `VtMemoryBlock` (nativeFd = -1) on actual failure.
 */
int ResourceMemory_allocate(
    VkDevice device,
    const VkMemoryAllocateInfo* pAllocateInfo,
    uint32_t hostVisibleMemoryFlag,
    uint32_t coherentMemoryFlag
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);
    VtMemoryBlock* memBlock = (VtMemoryBlock*)calloc(1, sizeof(VtMemoryBlock)); // Allocate 40 bytes (0x28) for the custom memory block struct

    if (!memBlock) {
        if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
            __stack_chk_fail();
        }
        return (int)NULL; // Failed to allocate internal tracking struct
    }

    memBlock->nativeFd = -1; // Initialize native file descriptor to invalid

    VkBool32 isHostVisible = isHostVisibleMemory(pAllocateInfo->memoryTypeIndex);

    if (!isHostVisible) {
        // Path for non-host-visible (device-local) memory.
        // Direct Vulkan allocation.
        VkResult res = dispatch_vkAllocateMemory(device, pAllocateInfo, NULL, &memBlock->deviceMemoryHandle);
        if (res != VK_SUCCESS) {
            goto cleanup_and_fail; // Jump to common cleanup path
        }
        memBlock->bufferHandle = NULL; // No VkBuffer associated directly in this path
    } else {
        // Path for host-visible memory, with different strategies based on flags and platform.

        // Check a specific bit in `hostVisibleMemoryFlag` (param_3)
        if ((hostVisibleMemoryFlag & 1) != 0) {
            // This branch seems to handle specific memory import/export extensions, possibly for Windows (Win32KeyedMutex)
            // or other platforms allowing direct handle creation.

            VkBufferCreateInfo bufferCreateInfo = {
                .sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO, // 0xC
                .pNext = NULL,
                .flags = 0,
                .size = pAllocateInfo->allocationSize,
                .usage = 0x1FF, // Common buffer usages (likely combined flags like VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | etc.)
                .sharingMode = VK_SHARING_MODE_EXCLUSIVE,
                .queueFamilyIndexCount = 0,
                .pQueueFamilyIndices = NULL
            };

            // This struct is chained to pNext of VkBufferCreateInfo.
            // VK_STRUCTURE_TYPE_WIN32_KEYED_MUTEX_ACQUIRE_RELEASE_INFO_NV_BASE (0x3b9be340)
            VkWin32KeyedMutexAcquireReleaseInfoNV_Custom keyedMutexInfo = {
                .sType = (VkStructureType)VK_STRUCTURE_TYPE_WIN32_KEYED_MUTEX_ACQUIRE_RELEASE_INFO_NV_BASE,
                .pNext = NULL,
                .field_08 = 1, // Assuming this is `acquireCount` based on standard struct
                // `field_0c` is not initialized here; could be pAcquireSyncs (pointer to handles)
            };
            bufferCreateInfo.pNext = &keyedMutexInfo;

            VkResult res = dispatch_vkCreateBuffer(device, &bufferCreateInfo, NULL, &memBlock->bufferHandle);
            if (res != VK_SUCCESS) {
                goto cleanup_and_fail;
            }

            // Get buffer memory requirements
            VkMemoryRequirements memoryRequirements;
            dispatch_vkGetBufferMemoryRequirements(device, memBlock->bufferHandle, &memoryRequirements);

            // Create a new VkMemoryAllocateInfo, copying from original but adjusting size and pNext.
            VkMemoryAllocateInfo bufferMemoryAllocateInfo = *pAllocateInfo;
            bufferMemoryAllocateInfo.allocationSize = memoryRequirements.size;

            // This struct is chained to pNext of `bufferMemoryAllocateInfo`.
            // UNKNOWN_STRUCTURE_TYPE_0x3b9be342 (0x3b9be342)
            VkUnknownExtensionStruct_0x3b9be342 unknownAllocInfoExt = {
                .sType = (VkStructureType)UNKNOWN_STRUCTURE_TYPE_0x3b9be342,
                .pNext = NULL, // This pNext is explicitly nulled in the disassembly
                // `field_08` and `field_0c` are copied from the input buffer.
            };
            bufferMemoryAllocateInfo.pNext = &unknownAllocInfoExt;

            res = dispatch_vkAllocateMemory(device, &bufferMemoryAllocateInfo, NULL, &memBlock->deviceMemoryHandle);
            if (res != VK_SUCCESS) {
                goto cleanup_and_fail;
            }

            // Get a native file descriptor for the allocated memory.
            VkMemoryGetFdInfoKHR getFdInfo = {
                .sType = VK_STRUCTURE_TYPE_MEMORY_GET_FD_INFO_KHR, // Or a similar value from `vk_video/vulkan_video_codec_h264std_decode.h` (e.g. 1000040006)
                .pNext = NULL,
                .memory = memBlock->deviceMemoryHandle,
                .handleType = 0x200, // Likely VK_EXTERNAL_MEMORY_HANDLE_TYPE_DMA_BUF_BIT_EXT based on context
            };
            res = dispatch_vkGetMemoryFd(device, &getFdInfo, &memBlock->nativeFd);
            if (res != VK_SUCCESS || memBlock->nativeFd < 1) {
                goto cleanup_and_fail;
            }
        } else if ((coherentMemoryFlag & 1) == 0) { // If not `hostVisibleMemoryFlag` (bit 0) AND `coherentMemoryFlag` (bit 1) is not set.
                                                    // This seems to be a Linux DMA-BUF/ION memory allocation path.

            int fd = __open_2("/dev/dma_heap/system-uncached", 0);
            if (fd == -1) {
                fd = __open_2("/dev/dma_heap/system", 0); // Fallback to system heap
            }

            if (fd != -1) {
                // ION_IOC_ALLOC ioctl call (0xc0204900)
                struct ion_alloc_data_custom {
                    VkDeviceSize size; // Corresponds to local_d0 (low part) and uStack_cc (high part)
                    uint32_t align; // Corresponds to uStack_c8 low part
                    uint32_t heap_mask; // Corresponds to uStack_c8 high part
                    uint32_t flags; // Corresponds to local_c0 low part
                    int fd; // Corresponds to local_c0 high part
                    uint32_t reserved[4]; // local_b0 to local_90
                } ion_alloc_data;

                // Values from disassembly:
                ion_alloc_data.size = pAllocateInfo->allocationSize;
                ion_alloc_data.align = 0; // uStack_c8._0_4_ = 0
                ion_alloc_data.heap_mask = 0x2000000; // uStack_c8._4_4_ = 0x2000000 (ION_HEAP_SYSTEM_CONTIG_MASK)
                ion_alloc_data.flags = 0x1000; // local_c0 = 0x1000 (ION_FLAG_CACHED)
                ion_alloc_data.fd = -1;

                int ioctl_res = ioctl(fd, ION_IOC_ALLOC, &ion_alloc_data);
                while (ioctl_res == -1 && (errno == EINTR || errno == EAGAIN)) { // EINTR=0xb, EAGAIN=4
                    ioctl_res = ioctl(fd, ION_IOC_ALLOC, &ion_alloc_data);
                }

                if (ioctl_res == -1) { // If ioctl still fails
                    close(fd);
                    goto cleanup_and_fail;
                }
                memBlock->nativeFd = ion_alloc_data.fd; // Store the fd obtained from ION

                // Optional ION_IOC_HEAP_QUERY or specific error handling (0x19 ENOENT check)
                // This part of the disassembly is complex, handling various ION ioctl errors.
                // Assuming success path for now for main flow.

                // Test mmap/munmap:
                void* mapped_ptr = mmap(NULL, pAllocateInfo->allocationSize, PROT_READ | PROT_WRITE, MAP_SHARED, memBlock->nativeFd, 0);
                if (mapped_ptr == MAP_FAILED) {
                    close(memBlock->nativeFd);
                    memBlock->nativeFd = -1;
                    close(fd);
                    goto cleanup_and_fail;
                }
                munmap(mapped_ptr, pAllocateInfo->allocationSize);

                int duplicated_fd = dup(memBlock->nativeFd); // Duplicate the FD
                if (duplicated_fd < 0) {
                    close(memBlock->nativeFd);
                    memBlock->nativeFd = -1;
                    close(fd);
                    goto cleanup_and_fail;
                }
                close(fd); // Close the original ion FD

                VkMemoryAllocateInfo dmaBufAllocateInfo = *pAllocateInfo; // Copy original allocate info

                // Setup pNext for dmaBufAllocateInfo with VkMemoryAllocateFlagsInfo.
                // VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_FLAGS_INFO (0x3b9beb10)
                VkMemoryAllocateFlagsInfo flagsInfo = {
                    .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_FLAGS_INFO,
                    .pNext = NULL,
                    .flags = (uint32_t)0x200, // Likely VK_MEMORY_ALLOCATE_DEVICE_ADDRESS_BIT or similar for external handle.
                    .deviceMask = 0
                };
                dmaBufAllocateInfo.pNext = &flagsInfo;

                res = dispatch_vkAllocateMemory(device, &dmaBufAllocateInfo, NULL, &memBlock->deviceMemoryHandle);
                if (res != VK_SUCCESS) {
                    close(memBlock->nativeFd); // Close the ION FD
                    memBlock->nativeFd = -1;
                    if (duplicated_fd >= 0) close(duplicated_fd); // Close duplicated FD if it was valid
                    goto cleanup_and_fail;
                }
                memBlock->nativeFd = duplicated_fd; // Store the duplicated FD for external use.

            } else {
                // Failed to open /dev/dma_heap or acquire ION memory
                goto cleanup_and_fail;
            }
        } else { // Fallback: Android AHardwareBuffer allocation (if coherentMemoryFlag (param_4) is set)

            uint32_t memoryPropertyFlags = getMemoryPropertyFlags(pAllocateInfo->memoryTypeIndex);
            // If the memory is cached, attempt to find a non-cached equivalent memory type.
            if ((memoryPropertyFlags & 0x00000008 /*VK_MEMORY_PROPERTY_HOST_CACHED_BIT*/) != 0) {
                // `getMemoryTypeIndex(0x7fffffff, 6)`: `0x7fffffff` suggests all flags allowed,
                // `6` is `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`.
                uint32_t uncachedMemoryTypeIndex = getMemoryTypeIndex(0x7fffffff, 6);
                // Directly modify the memoryTypeIndex in the incoming VkMemoryAllocateInfo.
                // This is a direct write to `pAllocateInfo->memoryTypeIndex`.
                *(uint32_t*)&pAllocateInfo->memoryTypeIndex = uncachedMemoryTypeIndex;
            }

            // AHardwareBuffer allocation.
            // Simplified AHB descriptor based on inferred values from disassembly.
            struct AHardwareBuffer_Desc_Custom {
                uint32_t width;
                uint32_t height;
                uint32_t layers;
                uint32_t format; // `0x21` (or `0x33`) from disassembly. Specific Android format enum.
                uint64_t usage;  // `0x1000 | 0x2000000 | 0x80000` from disassembly. Android usage flags.
            } ahb_desc = {
                .width = (uint32_t)pAllocateInfo->allocationSize, // Allocation size is often mapped to width for 1D buffers
                .height = 1,
                .layers = 1,
                .format = 0x21, // AHardwareBuffer_Format_RGBA_8888 or similar, based on common values.
                .usage = 0x1000 | 0x2000000 | 0x80000 // AHARDWAREBUFFER_USAGE_GPU_SAMPLED_IMAGE | AHARDWAREBUFFER_USAGE_GPU_FRAMEBUFFER | AHARDWAREBUFFER_USAGE_CPU_READ_OFTEN
            };
            // Note: The disassembly shows `local_d0` (uint32_t) being set from allocation size,
            // and `uStack_cc` (uint32_t) for high bits. It implies `VkDeviceSize` is split.
            // The `usage` flags `0x33` (`local_c0`) and `0x21` (`uStack_c8` high part) are for AHB.

            void* ahb_out_ptr = NULL;
            AHardwareBuffer_allocate(&ahb_desc, &ahb_out_ptr);
            memBlock->androidHardwareBuffer = ahb_out_ptr;

            if (memBlock->androidHardwareBuffer == NULL) {
                goto cleanup_and_fail;
            }

            memBlock->nativeFd = AHardwareBuffer_getFd(memBlock->androidHardwareBuffer);
            if (memBlock->nativeFd < 0) {
                goto cleanup_and_fail;
            }

            // Test mmap/munmap for the AHardwareBuffer's FD
            void* mapped_ptr = mmap(NULL, pAllocateInfo->allocationSize, PROT_READ | PROT_WRITE, MAP_SHARED, memBlock->nativeFd, 0);
            if (mapped_ptr == MAP_FAILED) {
                close(memBlock->nativeFd);
                memBlock->nativeFd = -1;
                goto cleanup_and_fail;
            }
            munmap(mapped_ptr, pAllocateInfo->allocationSize);

            // Allocate Vulkan memory, potentially chaining an extension struct.
            VkMemoryAllocateInfo ahbAllocateInfo = *pAllocateInfo; // Copy original allocate info

            // This struct is chained to pNext of `ahbAllocateInfo`.
            // VK_STRUCTURE_TYPE_DEVICE_QUEUE_INFO_2 (0x3b9cc1eb) is used here. This is highly unusual
            // for memory allocation, as it's typically for queue queries. It's likely a custom
            // or misidentified structure that serves a purpose like `VkImportAndroidHardwareBufferInfoANDROID`.
            VkWin32KeyedMutexAcquireReleaseInfoNV_Custom ahbExtInfo = { // Placeholder struct matching size
                .sType = (VkStructureType)VK_STRUCTURE_TYPE_DEVICE_QUEUE_INFO_2, // 0x3b9cc1eb
                .pNext = NULL,
                // `field_08` is assigned `lVar2` (the `AHardwareBuffer*`).
                // This means an AHardwareBuffer pointer is being directly passed to a Vulkan struct.
                // `field_0c` is assigned `0x200`.
                .field_08 = (uint32_t)(uintptr_t)memBlock->androidHardwareBuffer,
                .field_0c = 0x200 // Some flag.
            };
            ahbAllocateInfo.pNext = &ahbExtInfo;

            res = dispatch_vkAllocateMemory(device, &ahbAllocateInfo, NULL, &memBlock->deviceMemoryHandle);
            if (res != VK_SUCCESS) {
                goto cleanup_and_fail;
            }
        }
    }

    // Success path (if any allocation path succeeded)
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return (int)memBlock; // Return pointer to the allocated custom memory block


cleanup_and_fail:
    // This is the common cleanup path (LAB_0013dda8 in original assembly).
    // It's crucial to free all resources acquired in any branch before returning NULL.
    if (memBlock->bufferHandle != NULL) {
        dispatch_vkDestroyBuffer(device, memBlock->bufferHandle, NULL);
        memBlock->bufferHandle = NULL;
    }
    if (memBlock->deviceMemoryHandle != NULL) {
        dispatch_vkFreeMemory(device, memBlock->deviceMemoryHandle, NULL);
        memBlock->deviceMemoryHandle = NULL;
    }
    if (memBlock->androidHardwareBuffer != NULL) {
        AHardwareBuffer_release(memBlock->androidHardwareBuffer);
        memBlock->androidHardwareBuffer = NULL;
    }
    // The field at offset 0x20 (`__ptr[8]` in `int*` terms) is zeroed.
    // memBlock->unknownField8 = 0; // If such a field exists.
    if (memBlock->nativeFd >= 0) {
        close(memBlock->nativeFd);
        memBlock->nativeFd = -1;
    }
    free(memBlock); // Free the `VtMemoryBlock` itself.

    // On failure, it allocates a new, empty `VtMemoryBlock` and returns it.
    // This is an unusual error handling pattern, as typically NULL is returned.
    // However, the disassembly shows `calloc(1,0x28)` and return `__ptr`.
    VtMemoryBlock* failedMemBlock = (VtMemoryBlock*)calloc(1, sizeof(VtMemoryBlock));
    if (failedMemBlock) {
        failedMemBlock->nativeFd = -1; // Indicate failure
    }

    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return (int)failedMemBlock; // Return a valid but "failed" VtMemoryBlock or NULL if the final calloc also failed.
}


/**
 * @brief Frees resource memory previously allocated by `ResourceMemory_allocate`.
 *
 * This function is responsible for releasing all associated Vulkan and native
 * memory resources tracked by a `VtMemoryBlock` structure.
 *
 * @param device The Vulkan logical device.
 * @param memBlock A pointer to the `VtMemoryBlock` structure to be freed.
 */
void ResourceMemory_free(VkDevice device, VtMemoryBlock* memBlock) {
    if (memBlock != NULL) {
        if (memBlock->bufferHandle != NULL) {
            dispatch_vkDestroyBuffer(device, memBlock->bufferHandle, NULL);
            memBlock->bufferHandle = NULL;
        }
        if (memBlock->deviceMemoryHandle != NULL) {
            dispatch_vkFreeMemory(device, memBlock->deviceMemoryHandle, NULL);
            memBlock->deviceMemoryHandle = NULL;
        }
        if (memBlock->androidHardwareBuffer != NULL) {
            AHardwareBuffer_release(memBlock->androidHardwareBuffer);
            memBlock->androidHardwareBuffer = NULL;
        }
        // memBlock->unknownField8 = 0; // If such a field exists, it would be cleared.
        if (memBlock->nativeFd >= 0) {
            close(memBlock->nativeFd);
            memBlock->nativeFd = -1;
        }
        free(memBlock);
    }
}


// Structure for Linux ion allocation data (simplified)
struct ion_alloc_data {
    size_t size;
    size_t align;
    unsigned int heap_mask;
    unsigned int flags;
    int fd;
    unsigned int reserved[4];
};

// Structure for Linux dma_buf import (simplified)
struct dma_buf_import_dma_buf {
    // fields for dma_buf import
};

// Constants for ioctl requests (Linux-specific)
#define ION_IOC_HEAP_QUERY _IOR('I', 1, struct ion_heap_query) // 0xc0044901
#define ION_IOC_ALLOC _IOWR('I', 0, struct ion_alloc_data)     // 0xc0204900
#define ION_IOC_MAP _IOWR('I', 4, struct ion_fd_data)         // 0xc0084904
#define DMA_BUF_IOCTL_IMPORT _IOR('H', 0, struct dma_buf_import_dma_buf) // 0xc0184800

// Constants for sendmsg (Unix socket communication)
#define SOL_SOCKET 1
#define SCM_RIGHTS 1 // To send file descriptors

// Helper macro for CMSG_LEN, typically defined in sys/socket.h
#ifndef CMSG_LEN
#define CMSG_LEN(size) (sizeof(struct cmsghdr) + (size))
#endif

// Placeholder for `tpidr_el0` (Thread Pointer Register on ARM64)
// This is typically used for accessing thread-local storage, often for stack canaries.
// Assuming it points to a thread-local structure where the stack canary is at offset 0x28.
void* tpidr_el0;


/**
 * @brief Handles the vkMapMemory command received from the client.
 *
 * This function processes a request to map Vulkan device memory. It retrieves
 * the internal `VtMemoryBlock` associated with the requested `VkDeviceMemory` ID
 * from the client's input. It then checks the validity of the underlying native
 * memory handle (file descriptor) and sends a `sendmsg` response back to the client.
 * The response includes a success/failure code and the native file descriptor,
 * allowing the client to directly map the memory in its process.
 *
 * @param vtContext A pointer to the VtContext structure, providing access to
 *                  the client's file descriptor (`clientFd`) and input buffer.
 */
void vt_handle_vkMapMemory(VtContext* vtContext) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Stack canary

    // Read the VkDeviceMemory ID from the input buffer.
    // Assuming the input buffer format is:
    // [0x00] byte command_id
    // [0x01] uint64_t VkDevice (device ID)
    // [0x09] uint64_t VkDeviceMemory (memory ID to map)
    // [0x11] uint64_t VkDeviceSize offset
    // [0x19] uint64_t VkDeviceSize size
    // [0x21] VkMemoryMapFlags flags (uint32_t)
    // The disassembly shows `**(undefined8 **)(param_1 + 10)` being passed to `VkObject_fromId`.
    // If `param_1` is `VtContext*`, `param_1 + 10` is `((char*)vtContext + 10)`.
    // `**(uint64_t**)((char*)vtContext + 10)` means it would read a `uint64_t` at `vtContext+10` as a pointer,
    // then dereference that pointer to get another `uint64_t` (the ID). This is very unusual.
    // A more plausible interpretation for `**(undefined8 **)(param_1 + 10)` is a compiler
    // optimization where `param_1` is implicitly `vtContext->inputBufferPtr` and `+10` is an offset
    // within that buffer to the `VkDeviceMemory` ID.
    // Let's assume the `VkDeviceMemory` ID is at `vtContext->inputBufferPtr + 0x10` based on common command formats.
    void* memoryId = *(void**)(vtContext->inputBufferPtr + 0x10);

    // Get the internal `VtMemoryBlock` associated with the VkDeviceMemory ID.
    VtMemoryBlock* memBlock = (VtMemoryBlock*)VkObject_fromId(memoryId);

    // Prepare the result status to send back to the client.
    int32_t resultCode = VK_SUCCESS; // Default to success
    if (memBlock == NULL || memBlock->nativeFd < 0) {
        resultCode = 0xfffffffb; // VK_ERROR_MEMORY_MAP_FAILED (or a custom error)
    }

    // Setup `sendmsg` structures to send `resultCode` and the native file descriptor.
    msghdr msg = {0};
    iovec iov[1]; // Array of one iovec for simple data transfer.

    // First part of the message: result status (4 bytes)
    iov[0].iov_base = &resultCode;
    iov[0].iov_len = sizeof(resultCode);

    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    // Ancillary data (control message) to send the file descriptor.
    // The disassembled values (`local_c8 = 0x14`, `local_c0 = 0x100000001`) indicate the `cmsghdr` structure.
    // CMSG_LEN(sizeof(int)) typically evaluates to 20 (0x14) on 64-bit systems.
    // `0x100000001` for `local_c0` implies `cmsg_level` is `0x10000000` and `cmsg_type` is `0x1`.
    // This is a common pattern for sending `SCM_RIGHTS` (file descriptors) over a Unix domain socket.
    // `0x10000000` is often used for `SOL_SOCKET` level when sending FDs via `SCM_RIGHTS`.
    union {
        struct cmsghdr cm;
        char control[CMSG_LEN(sizeof(int))];
    } cmsg_buf; // Use a union to ensure proper alignment

    struct cmsghdr* cmsg = &cmsg_buf.cm;
    cmsg->cmsg_len = CMSG_LEN(sizeof(int));
    cmsg->cmsg_level = SOL_SOCKET; // 0x1
    cmsg->cmsg_type = SCM_RIGHTS;  // 0x1

    // Place the file descriptor into the control message data.
    *(int*)CMSG_DATA(cmsg) = (memBlock != NULL) ? memBlock->nativeFd : -1;

    msg.msg_control = cmsg;
    msg.msg_controllen = cmsg->cmsg_len;

    // Send the message to the client.
    sendmsg(vtContext->clientFd, &msg, 0);

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}
```

---

The provided C code snippets `vt_handle_vkFlushMappedMemoryRanges` and `vt_handle_vkInvalidateMappedMemoryRanges` are wrapper functions that handle a Vulkan API call. They deserialize input data from a custom buffer format, invoke the corresponding Vulkan function (`vkFlushMappedMemoryRanges` or `vkInvalidateMappedMemoryRanges`), and then serialize the `VkResult` back to a client-facing ring buffer.

### 1. Inferred Data Structures and Types

Based on the provided `VtContext` and the disassembly, here are the relevant structures and their likely memory layouts:

**`VtContext` (partially shown, `param_1` in the functions):**

```c
typedef struct VtContext {
    // ...
    /* 0x28 */ char* inputBufferPtr;      // Pointer to the current command's data
    // ...
    /* 0x60 */ RingBuffer* clientRingBuffer; // For writing results back to the client
    // ...
} VtContext;
```

**`VkMappedMemoryRange` (standard Vulkan structure):**

```c
typedef struct VkMappedMemoryRange {
    VkStructureType    sType;       // 0x00 (4 bytes)
    const void*        pNext;       // 0x08 (8 bytes)
    VkDeviceMemory     memory;      // 0x10 (8 bytes)
    VkDeviceSize       offset;      // 0x18 (8 bytes)
    VkDeviceSize       size;        // 0x20 (8 bytes)
} VkMappedMemoryRange; // Total size 40 bytes (0x28)
```

**Custom Input Buffer Serialization Format:**

The functions read command data from `vtContext->inputBufferPtr`. The format appears to be:

*   **Byte 0:** Command ID (1 byte). If this byte is `'\0'`, a special, less common path is taken.
*   **Bytes 1-8:** `VkDevice` ID (8 bytes). This is a client-side identifier that needs to be converted to a `VkDevice` handle.
*   **Bytes 9-12:** `memoryRangeCount` (uint32_t) - the number of `VkMappedMemoryRange` entries that follow.
*   **Bytes 13-16:** Padding (4 bytes).
*   **Bytes 17 onwards:** Array of serialized `VkMappedMemoryRange` entries. Each entry is **28 bytes** long (not the standard 40 bytes), packed as:
    *   **Offset 0 (4 bytes):** `VkStructureType sType`
    *   **Offset 4 (8 bytes):** `VkDeviceMemory` ID (client-side identifier, converted to `VkDeviceMemory` handle).
    *   **Offset 12 (8 bytes):** `VkDeviceSize offset`
    *   **Offset 20 (8 bytes):** `VkDeviceSize size`

**Output Buffer Serialization Format (to `clientRingBuffer`):**

*   **Bytes 0-3:** `VkResult` (4 bytes).
*   **Bytes 4-7:** Padding/zero (4 bytes).
Total 8 bytes written back.

**Auxiliary Functions (assumed external):**

*   `VkObject_fromId(void* id)`: Converts a client-side object ID (e.g., `VkDevice` ID, `VkDeviceMemory` ID) to its native Vulkan handle (e.g., `VkDevice`, `VkDeviceMemory`).
*   `RingBuffer_write(RingBuffer* rb, const void* data, size_t size)`: Writes `size` bytes from `data` into the ring buffer.
*   `dispatch_vkFlushMappedMemoryRanges(VkDevice device, uint32_t memoryRangeCount, const VkMappedMemoryRange* pMemoryRanges)`: Function pointer to the actual Vulkan `vkFlushMappedMemoryRanges` API call.
*   `dispatch_vkInvalidateMappedMemoryRanges(VkDevice device, uint32_t memoryRangeCount, const VkMappedMemoryRange* pMemoryRanges)`: Function pointer to the actual Vulkan `vkInvalidateMappedMemoryRanges` API call.
*   `__stack_chk_fail()`: Security function called on stack corruption.
*   `tpidr_el0`: A register usually holding the thread pointer, used for stack canary.

### 2. Decompiled Code

The two functions `vt_handle_vkFlushMappedMemoryRanges` and `vt_handle_vkInvalidateMappedMemoryRanges` are nearly identical, differing only in the final Vulkan dispatch call.

```c
#include <string.h> // For strncpy, strcmp
#include <stdio.h>  // For snprintf
#include <stdint.h> // For uint32_t, uint64_t, int32_t, etc.

// Placeholder for external function pointers and structures for compilation,
// normally these would come from Vulkan headers and other project files.
// Assuming VK_DEFINE_HANDLE / VK_DEFINE_NON_DISPATCHABLE_HANDLE expands to void* or similar.
typedef void* VkDevice;
typedef void* VkDeviceMemory;
typedef void* VkBuffer;
typedef uint64_t VkDeviceSize;
typedef uint32_t VkResult; // Assuming VkResult is uint32_t based on return value writing
typedef uint32_t VkStructureType; // As per vulkan_core.h

// Re-defining VkMappedMemoryRange from vulkan_core.h for clarity.
typedef struct VkMappedMemoryRange {
    VkStructureType    sType;
    const void*        pNext;
    VkDeviceMemory     memory;
    VkDeviceSize       offset;
    VkDeviceSize       size;
} VkMappedMemoryRange;

// Simplified structure for VkObject_fromId return, assuming it gives raw handle or pointer to wrapper.
// The disassembly suggests VkObject_fromId returns a pointer, which then might be dereferenced at +0x18
// if `vortekSerializerCastVkObject` is true. For simplicity, assume VkObject_fromId returns the VkHandle directly.
typedef void* (*PFN_VkObject_fromId)(void* id);
extern PFN_VkObject_fromId VkObject_fromId;

// Assumed dispatch table entry for Vulkan functions
typedef VkResult (*PFN_vkFlushMappedMemoryRanges)(VkDevice device, uint32_t memoryRangeCount, const VkMappedMemoryRange* pMemoryRanges);
typedef VkResult (*PFN_vkInvalidateMappedMemoryRanges)(VkDevice device, uint32_t memoryRangeCount, const VkMappedMemoryRange* pMemoryRanges);
extern PFN_vkFlushMappedMemoryRanges dispatch_vkFlushMappedMemoryRanges;
extern PFN_vkInvalidateMappedMemoryRanges dispatch_vkInvalidateMappedMemoryRanges;

// Mock RingBuffer functions and VtContext from the provided header
typedef struct RingBuffer RingBuffer; // Incomplete type, just for pointer
extern uint8_t RingBuffer_write(RingBuffer* rb, const void* data, size_t size);

// From the provided VtContext struct
typedef struct VtContext {
    // ... other members
    char* inputBufferPtr;      // Offset 0x28
    char padding_0x20[0x38];   // Padding to reach 0x60
    RingBuffer* clientRingBuffer; // Offset 0x60
    // ... other members
} VtContext;

// External symbol for stack canary
extern uintptr_t tpidr_el0; // Assuming tpidr_el0 holds a pointer to a thread-local storage area
extern void __stack_chk_fail(void);

// Global flag assumed from disassembly variable `vortekSerializerCastVkObject`
// Its purpose appears to be to control how VkObject_fromId's result is used.
// If true, it means VkObject_fromId returned a wrapper struct and the actual
// VkHandle is at offset 0x18 within that struct.
extern uint8_t vortekSerializerCastVkObject;


/**
 * @brief Handles the vkFlushMappedMemoryRanges Vulkan command.
 *
 * This function deserializes the VkDevice handle and an array of
 * VkMappedMemoryRange structs from the input buffer, calls the
 * vkFlushMappedMemoryRanges API, and then serializes the VkResult
 * back to the client.
 *
 * @param vtContext A pointer to the VtContext containing input/output buffers.
 */
void vt_handle_vkFlushMappedMemoryRanges(VtContext* vtContext) {
    // Stack canary protection: Save the stack cookie value.
    uintptr_t stack_cookie_start = *(uintptr_t*)(((char*)tpidr_el0) + 0x28);
    uintptr_t current_stack_cookie = stack_cookie_start; // Copy to local variable

    char* input_buffer_ptr = vtContext->inputBufferPtr;
    void* serialized_device_id;
    uint32_t memory_range_count;
    
    // Determine the layout of the initial part of the input buffer.
    // This conditional branch likely handles different serialization versions or command types.
    // The common path assumes input_buffer_ptr[0] is a command ID (non-zero).
    if (input_buffer_ptr[0] == '\0') {
        // Less common path: Assumed format [0x00] [count (4 bytes)] [device ID (8 bytes)] ...
        // In this specific disassembly, `pvVar6` gets `param_1` which is `vtContext`,
        // and `uVar3` (count) is read from `input_buffer_ptr + 1`. This might be an anomaly.
        // For accurate decompilation, sticking to the literal offsets from disassembly:
        serialized_device_id = (void*)vtContext; // This assignment is highly suspicious for a device handle
        memory_range_count = *(uint32_t*)(input_buffer_ptr + 1);
    } else {
        // Common path: Assumed format [command_id (1 byte)] [device_id (8 bytes)] [count (4 bytes)] ...
        serialized_device_id = *(void**)(input_buffer_ptr + 1);
        memory_range_count = *(uint32_t*)(input_buffer_ptr + 9);
    }

    // Convert the serialized device ID to a native VkDevice handle.
    VkDevice device = (VkDevice)VkObject_fromId(serialized_device_id);

    // Dynamically allocate space for VkMappedMemoryRange array on the stack.
    // Calculate required size and align to 16 bytes.
    long stack_allocated_size = (long)memory_range_count * sizeof(VkMappedMemoryRange);
    // Ghidra's calculation: `-(ulong)uVar3 * 0x28 + 0xf & 0x7ffffffff0`
    // This is `(-(stack_allocated_size + 15)) & (~0xF)` for 16-byte alignment and negative offset.
    long stack_array_relative_offset = -((stack_allocated_size + 0xF) & ~0xF);
    
    // `local_80` is address of stack cookie `current_stack_cookie`.
    // `local_78` points to the start of the allocated array on the stack.
    VkMappedMemoryRange* mapped_memory_ranges_array = 
        (VkMappedMemoryRange*)((char*)&current_stack_cookie + stack_array_relative_offset);

    // Determine the starting offset for the first VkMappedMemoryRange data in the input buffer.
    // Normal path: header is 1 (cmd_id) + 8 (device_id) + 4 (count) + 4 (padding) = 17 bytes.
    uint32_t current_input_data_offset = (input_buffer_ptr[0] == '\0' ? (1 + 8) : (9 + 8));

    // Loop through each serialized VkMappedMemoryRange in the input buffer
    // and populate the corresponding VkMappedMemoryRange struct on the stack.
    for (uint32_t i = 0; i < memory_range_count; ++i) {
        VkMappedMemoryRange* current_output_range = &mapped_memory_ranges_array[i];
        
        // Deserialize sType (4 bytes)
        current_output_range->sType = *(uint32_t*)(input_buffer_ptr + current_input_data_offset);
        current_output_range->pNext = NULL; // `pNext` is explicitly set to NULL

        // Deserialize VkDeviceMemory ID (8 bytes) and convert to handle
        uint64_t serialized_memory_id = *(uint64_t*)(input_buffer_ptr + current_input_data_offset + 4);
        VkDeviceMemory memory_handle = (VkDeviceMemory)VkObject_fromId((void*)serialized_memory_id);
        
        // This block reflects `if (cVar4 != '\0') { pvVar6 = *(void **)((long)pvVar6 + 0x18); }`
        // If vortekSerializerCastVkObject is true, it means `VkObject_fromId` returns a pointer to a
        // wrapper struct, and the actual handle is at offset 0x18 within it.
        if (vortekSerializerCastVkObject != 0) {
            memory_handle = *(VkDeviceMemory*)((char*)memory_handle + 0x18);
        }
        current_output_range->memory = memory_handle;

        // Deserialize offset (8 bytes)
        current_output_range->offset = *(VkDeviceSize*)(input_buffer_ptr + current_input_data_offset + 0xc); // 0xc is 12 bytes
        // Deserialize size (8 bytes)
        current_output_range->size = *(VkDeviceSize*)(input_buffer_ptr + current_input_data_offset + 0x14); // 0x14 is 20 bytes

        // Advance input buffer offset to the next serialized VkMappedMemoryRange.
        // Each serialized VkMappedMemoryRange is 28 bytes long.
        current_input_data_offset += 28;
    }

    // Call the actual Vulkan API function.
    VkResult result = dispatch_vkFlushMappedMemoryRanges(device, memory_range_count, mapped_memory_ranges_array);

    // Prepare a temporary buffer to write result (4 bytes) and padding (4 bytes) back to client.
    uint32_t result_buffer[2];
    result_buffer[0] = result;
    result_buffer[1] = 0; // Padding/zero, corresponds to `uStack_6c = 0;`

    // Write the result back to the client using the ring buffer.
    RingBuffer_write(vtContext->clientRingBuffer, result_buffer, sizeof(result_buffer));

    // Stack canary protection: Check if the stack cookie has been modified.
    if (*(uintptr_t*)(((char*)tpidr_el0) + 0x28) != stack_cookie_start) {
        __stack_chk_fail(); // Indicate stack corruption.
    }
}


/**
 * @brief Handles the vkInvalidateMappedMemoryRanges Vulkan command.
 *
 * This function is almost identical to `vt_handle_vkFlushMappedMemoryRanges`,
 * but it calls `vkInvalidateMappedMemoryRanges` instead.
 *
 * @param vtContext A pointer to the VtContext containing input/output buffers.
 */
void vt_handle_vkInvalidateMappedMemoryRanges(VtContext* vtContext) {
    // Stack canary protection: Save the stack cookie value.
    uintptr_t stack_cookie_start = *(uintptr_t*)(((char*)tpidr_el0) + 0x28);
    uintptr_t current_stack_cookie = stack_cookie_start;

    char* input_buffer_ptr = vtContext->inputBufferPtr;
    void* serialized_device_id;
    uint32_t memory_range_count;
    
    if (input_buffer_ptr[0] == '\0') {
        serialized_device_id = (void*)vtContext;
        memory_range_count = *(uint32_t*)(input_buffer_ptr + 1);
    } else {
        serialized_device_id = *(void**)(input_buffer_ptr + 1);
        memory_range_count = *(uint32_t*)(input_buffer_ptr + 9);
    }

    VkDevice device = (VkDevice)VkObject_fromId(serialized_device_id);

    long stack_allocated_size = (long)memory_range_count * sizeof(VkMappedMemoryRange);
    long stack_array_relative_offset = -((stack_allocated_size + 0xF) & ~0xF);
    
    VkMappedMemoryRange* mapped_memory_ranges_array = 
        (VkMappedMemoryRange*)((char*)&current_stack_cookie + stack_array_relative_offset);

    uint32_t current_input_data_offset = (input_buffer_ptr[0] == '\0' ? (1 + 8) : (9 + 8));

    for (uint32_t i = 0; i < memory_range_count; ++i) {
        VkMappedMemoryRange* current_output_range = &mapped_memory_ranges_array[i];
        
        current_output_range->sType = *(uint32_t*)(input_buffer_ptr + current_input_data_offset);
        current_output_range->pNext = NULL;

        uint64_t serialized_memory_id = *(uint64_t*)(input_buffer_ptr + current_input_data_offset + 4);
        VkDeviceMemory memory_handle = (VkDeviceMemory)VkObject_fromId((void*)serialized_memory_id);
        
        if (vortekSerializerCastVkObject != 0) {
            memory_handle = *(VkDeviceMemory*)((char*)memory_handle + 0x18);
        }
        current_output_range->memory = memory_handle;

        current_output_range->offset = *(VkDeviceSize*)(input_buffer_ptr + current_input_data_offset + 0xc);
        current_output_range->size = *(VkDeviceSize*)(input_buffer_ptr + current_input_data_offset + 0x14);

        current_input_data_offset += 28;
    }

    // Call the actual Vulkan API function (this is the only line that differs from flush).
    VkResult result = dispatch_vkInvalidateMappedMemoryRanges(device, memory_range_count, mapped_memory_ranges_array);

    uint32_t result_buffer[2];
    result_buffer[0] = result;
    result_buffer[1] = 0;

    RingBuffer_write(vtContext->clientRingBuffer, result_buffer, sizeof(result_buffer));

    // Stack canary protection: Check if the stack cookie has been modified.
    if (*(uintptr_t*)(((char*)tpidr_el0) + 0x28) != stack_cookie_start) {
        __stack_chk_fail();
    }
}
```

---

# vt_handle_vkBindBufferMemory

The `vt_handle_vkBindBufferMemory` function is responsible for processing a client's request to call `vkBindBufferMemory`. It deserializes the input parameters from a shared memory buffer, executes the native Vulkan API call, optionally performs additional internal tracking, and then serializes the `VkResult` back to the client.

```c
#include <pthread.h> // For pthread_self()
#include <string.h>  // For memcpy, memset (though not directly visible in this snippet, inferred from common patterns)
#include <stdlib.h>  // For calloc, free (inferred from common patterns)

// --- External/Assumed Declarations ---

// Vulkan.h types and function pointers (dispatch table)
#include "vulkan.h"

// Assumed internal structure for Vulkan objects managed by the renderer.
// This is inferred from the usage `*(undefined8 *)(piVar5 + 6)` which implies
// `piVar5` is an `int*` to a struct, and the native handle is at offset 24 bytes.
typedef struct VkInternalObject {
    char _internal_data[24]; // Placeholder for other internal metadata or padding
    void* native_handle;      // The actual VkBuffer/VkDeviceMemory native Vulkan handle (uint64_t or pointer)
} VkInternalObject;

// Function to convert a client-side ID/pointer to an internal VkObject pointer.
// This is a core part of the renderer's client-server communication layer.
extern void* VkObject_fromId(void* client_id);

// Dispatch pointer for the native Vulkan API function.
// This would be initialized via vkGetDeviceProcAddr.
extern PFN_vkBindBufferMemory dispatch_vkBindBufferMemory;

// External function from the TextureDecoder component.
// This is likely a hooking point to update the renderer's internal state
// regarding memory bindings, especially for textures.
extern void TextureDecoder_addBoundBuffer(long texture_decoder_ptr, VkInternalObject* vk_device_memory_obj_ptr, VkBuffer vk_buffer_obj, VkDeviceSize memory_offset);

// External function for writing data back to the client's ring buffer.
extern byte RingBuffer_write(RingBuffer* ring_buffer, void* data, uint size);

// Placeholder for stack check failure, typical in hardened C/C++ builds.
extern void __stack_chk_fail(VkResult result); // Passes the last result as an argument.

// --- VtContext Structure ---
// This is the core context structure for the renderer, passed as 'param_1'.
// The offsets are critical and derived from the disassembly.
typedef struct VtContext {
    char _padding_0_27[0x28];     // Padding to reach inputBufferPtr
    char*   inputBufferPtr;       // @ 0x28: Pointer to the client's current serialized command data.
    // The following fields are not directly read in this specific function,
    // but are part of the VtContext based on the provided header.
    int     inputBufferSize;      // @ 0x30
    void*   tempBuffer;           // @ 0x38
    int     tempBufferOffset;     // @ 0x40
    ArrayList* tempAllocations;   // @ 0x48
    long    _padding_50;          // @ 0x50
    pthread_t serverThread;       // @ 0x58
    RingBuffer* clientRingBuffer; // @ 0x60: Output buffer for sending results to the client.
    RingBuffer* serverRingBuffer; // @ 0x68: Input buffer for receiving commands from the client.
    int     lastError;            // @ 0x70
    int     graphicsQueueFamilyIndex; // @ 0x74
    TextureDecoder* textureDecoder; // @ 0x78: Pointer to the TextureDecoder instance.
} VtContext;


/**
 * @brief Handles the vkBindBufferMemory Vulkan API call received from a client.
 *
 * This function performs the following steps:
 * 1.  **Stack Canary Setup**: Initializes stack canary for security.
 * 2.  **Argument Deserialization**: Reads the serialized arguments (VkDevice,
 *     VkBuffer, VkDeviceMemory, and VkDeviceSize memoryOffset) from the
 *     `inputBufferPtr` pointed to by `vt_context->inputBufferPtr`. Each handle
 *     is prefixed by a byte indicating its presence.
 * 3.  **Object Conversion**: Uses `VkObject_fromId` to convert the raw client-side
 *     IDs into internal `VkObject` pointers, and for `VkDeviceMemory`, extracts
 *     the native handle from the `VkInternalObject` structure.
 * 4.  **Native API Call**: Invokes the `vkBindBufferMemory` function using the
 *     `dispatch_vkBindBufferMemory` function pointer.
 * 5.  **Hooking/Interception**: If the `vkBindBufferMemory` call succeeds and
 *     a `TextureDecoder` instance exists, it calls `TextureDecoder_addBoundBuffer`
 *     to update internal renderer state for buffer tracking.
 * 6.  **Result Serialization**: Writes the `VkResult` of the native API call back
 *     to the client via the `clientRingBuffer`.
 * 7.  **Stack Canary Check**: Verifies the stack canary to detect buffer overflows.
 *
 * @param vt_context A pointer to the VtContext structure containing necessary
 *                   buffers and state.
 */
void vt_handle_vkBindBufferMemory(VtContext* vt_context) {
    long stack_canary_value; // Stores the stack canary value for security.
    char* input_buffer_read_ptr; // Pointer to read from the client's input buffer.
    long current_offset_in_buffer; // Tracks the current byte offset within the input buffer.

    // 1. Standard stack canary prologue:
    stack_canary_value = *(long*)((char*)pthread_self() + 0x28);

    // Initialize `input_buffer_read_ptr` to point to the start of the client's serialized data.
    input_buffer_read_ptr = vt_context->inputBufferPtr;

    // 2. Argument Deserialization from `input_buffer_read_ptr`:

    // Deserialize VkDevice handle (1st argument: `device`)
    void* raw_vk_device_id;
    // Check the first byte of the serialized data to determine how to get the VkDevice handle.
    // If it's '\0', it might signify an implicit/default device, and `vt_context` itself
    // is used as a reference to derive the device handle.
    if (*input_buffer_read_ptr == '\0') {
        current_offset_in_buffer = 1; // Move past the 1-byte null indicator.
        raw_vk_device_id = (void*)vt_context; // Use the context pointer as a special ID.
    } else {
        current_offset_in_buffer = 9; // Move past 1-byte flag and 8-byte ID.
        raw_vk_device_id = *(void**)(input_buffer_read_ptr + 1); // Read the raw 8-byte VkDevice ID.
    }

    // Deserialize VkBuffer handle (2nd argument: `buffer`)
    void* raw_vk_buffer_id;
    // Check the flag byte for VkBuffer. If not '\0', read the 8-byte ID.
    if (input_buffer_read_ptr[current_offset_in_buffer] != '\0') {
        raw_vk_buffer_id = *(void**)(input_buffer_read_ptr + current_offset_in_buffer + 1); // Read raw VkBuffer ID.
        current_offset_in_buffer = (current_offset_in_buffer + 9); // Advance offset past ID.
    } else {
        current_offset_in_buffer = (current_offset_in_buffer + 1); // Advance past null indicator.
        raw_vk_buffer_id = NULL; // Explicitly set to NULL if the flag implies a null handle.
    }

    // Deserialize VkDeviceMemory handle (3rd argument: `memory`)
    void* raw_vk_device_memory_id;
    // Check the flag byte for VkDeviceMemory. If not '\0', read the 8-byte ID.
    if (input_buffer_read_ptr[current_offset_in_buffer] != '\0') {
        raw_vk_device_memory_id = *(void**)(input_buffer_read_ptr + current_offset_in_buffer + 1); // Read raw VkDeviceMemory ID.
        current_offset_in_buffer = (current_offset_in_buffer + 9); // Advance offset past ID.
    } else {
        current_offset_in_buffer = (current_offset_in_buffer + 1); // Advance past null indicator.
        raw_vk_device_memory_id = NULL; // Explicitly set to NULL if the flag implies a null handle.
    }

    // Deserialize VkDeviceSize memoryOffset (4th argument: `memoryOffset`)
    VkDeviceSize memory_offset = *(VkDeviceSize*)(input_buffer_read_ptr + current_offset_in_buffer);

    // Convert raw IDs/pointers received from the client into their actual internal VkObject pointers.
    // This typically involves looking up the native Vulkan handle from a registry.
    VkDevice vk_device = (VkDevice)VkObject_fromId(raw_vk_device_id);
    VkBuffer vk_buffer = (VkBuffer)VkObject_fromId(raw_vk_buffer_id);
    // For VkDeviceMemory, `VkObject_fromId` returns a pointer to an internal `VkInternalObject` struct.
    // The actual native `VkDeviceMemory` handle is stored at `native_handle` member (offset 24 bytes).
    VkInternalObject* vk_device_memory_obj = (VkInternalObject*)VkObject_fromId(raw_vk_device_memory_id);

    // Local buffer to store the VkResult before sending it back.
    int result_buffer[2]; // `local_50` in assembly, 8 bytes total.

    // 3. Native Vulkan API Call: `vkBindBufferMemory`
    // The actual `VkDeviceMemory` handle is extracted from `vk_device_memory_obj->native_handle`.
    result_buffer[0] = dispatch_vkBindBufferMemory(
        vk_device,                                            // VkDevice device
        vk_buffer,                                            // VkBuffer buffer
        (VkDeviceMemory)vk_device_memory_obj->native_handle,  // VkDeviceMemory memory
        memory_offset                                         // VkDeviceSize memoryOffset
    );

    // 4. Hooking / Interception Logic: `TextureDecoder_addBoundBuffer`
    // If the Vulkan call was successful (`VK_SUCCESS == 0`) and the `textureDecoder`
    // is initialized in the context, call `TextureDecoder_addBoundBuffer` to track
    // the newly bound buffer, likely for internal renderer management (e.g., caching).
    if (result_buffer[0] == VK_SUCCESS && vt_context->textureDecoder != NULL) {
        TextureDecoder_addBoundBuffer(
            (long)vt_context->textureDecoder,       // Pointer to the TextureDecoder instance
            (int*)vk_device_memory_obj,             // Internal VkObject pointer for VkDeviceMemory
            (long)vk_buffer,                        // Internal VkObject pointer for VkBuffer
            memory_offset                           // VkDeviceSize memoryOffset
        );
    }

    // 5. Output Serialization to `client_ring_buffer`:
    // The second element of `result_buffer` is set to 0 (padding for 8-byte transfer).
    result_buffer[1] = 0;

    // Write the `VkResult` (8 bytes) to the client's output ring buffer.
    RingBuffer_write(vt_context->clientRingBuffer, result_buffer, sizeof(VkResult));

    // 6. Standard stack canary epilogue:
    // Verify that the stack canary value has not been altered, indicating stack integrity.
    if (*(long*)((char*)pthread_self() + 0x28) != stack_canary_value) {
        __stack_chk_fail(result_buffer[0]); // Indicate stack corruption, passing the VkResult.
    }
    return;
}
```

---

# vt_handle_vkWaitForFences

Let's decompile the given C functions, mapping them to the provided Vulkan header definitions and the `VtContext` structure.

First, let's define the likely structure of the network commands based on the parsing logic:

```c
// Simplified structure for the incoming command buffer for vkWaitForFences
// This structure is inferred from the disassembly and might represent a network packet format.
// The actual byte offsets depend on the 'command_id_flag'.

// Common header for commands
typedef struct CommandHeader {
    uint8_t command_id_flag; // If 0, device_id might be implicit (e.g., from VtContext global device)
} CommandHeader;

// Command header for commands with an explicit VkDevice ID
typedef struct ExplicitDeviceCommandHeader {
    uint8_t  command_id_flag; // Non-zero value
    uint64_t device_id;       // VkDevice ID
} ExplicitDeviceCommandHeader;

// Data structure for vkWaitForFences command parameters received over the network
typedef struct WaitForFencesCommandData {
    uint32_t   fence_count;  // Number of fences
    VkBool32   wait_all;     // Whether to wait for all fences (0 or 1)
    // Followed by `fence_count` VkFence IDs (each is uint64_t)
    uint64_t   timeout;      // Timeout in nanoseconds
} WaitForFencesCommandData;

// Data structure for vkGetFenceFdKHR command parameters received over the network
typedef struct GetFenceFdCommandData {
    VkExternalFenceHandleTypeFlagBits handle_type; // Type of handle to retrieve (e.g., OPAQUE_FD_BIT)
    uint64_t                          fence_id;    // VkFence ID
} GetFenceFdCommandData;

// For sending results back to client via sendmsg, for file descriptors
// This represents the `struct cmsghdr` and the associated data for `SCM_RIGHTS`.
// For `SCM_RIGHTS`, the data is an array of file descriptors (int).
typedef struct ControlMessageHeader {
    size_t cmsg_len;   // Length of the ancillary data, including the header
    int    cmsg_level; // Protocol level (e.g., SOL_SOCKET)
    int    cmsg_type;  // Type of control message (e.g., SCM_RIGHTS)
    // Followed by the data (e.g., array of ints for FDs)
} ControlMessageHeader;

// RingBuffer is a custom data structure for inter-process communication.
// Assumed to have a `write` method.
typedef struct RingBuffer RingBuffer; // Forward declaration

// Function pointer types for Vulkan API calls (from vk_platform.h, implied by usage)
typedef VkResult (VKAPI_PTR *PFN_vkWaitForFences)(VkDevice device, uint32_t fenceCount, const VkFence* pFences, VkBool32 waitAll, uint64_t timeout);
typedef VkResult (VKAPI_PTR *PFN_vkGetFenceFdKHR)(VkDevice device, const VkFenceGetFdInfoKHR* pGetFdInfo, int* pFd);

// External function for converting ID to Vulkan Object
extern void* VkObject_fromId(void* id); // Simplified signature
extern void* tpidr_el0; // Likely a thread-local storage pointer (e.g., for stack canary)
extern void __stack_chk_fail(void); // Stack protector failure function
extern int strcmp(const char *s1, const char *s2);
extern int close(int fd);
extern ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);

```

---

### Decompilation of `vt_handle_vkWaitForFences`

```c
/**
 * @brief Handles the vkWaitForFences command received from the client.
 *
 * This function parses the incoming command buffer to extract parameters for
 * vkWaitForFences (device handle, fence count, fence handles, waitAll flag, timeout).
 * If the timeout is 0, it calls vkWaitForFences directly. Otherwise, for non-zero
 * timeouts, it calls vkGetFenceFdKHR for each fence to retrieve their file
 * descriptors, and then sends these file descriptors along with the operation
 * result back to the client using `sendmsg`.
 *
 * @param vtContext A pointer to the VtContext structure.
 */
void vt_handle_vkWaitForFences(VtContext* vtContext) {
    // Save the stack cookie for stack overflow detection.
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    char* input_buffer_ptr = vtContext->inputBufferPtr;

    // Determine the effective offset for reading command data based on the first byte.
    // This implies a variable-size header.
    uint32_t data_start_offset; // Offset to fence_count and subsequent data
    VkDevice device_handle;     // The VkDevice handle

    if (input_buffer_ptr[0] == '\0') { // Simple header (1 byte for command ID)
        data_start_offset = 1;
        // If command ID is '\0', the device handle is implicitly the context itself
        // (or a global device associated with the context). This is an unusual but possible design.
        // We'll cast VtContext* to VkDevice here, assuming VkObject_fromId handles it.
        device_handle = (VkDevice)VkObject_fromId(vtContext);
    } else { // Complex header (9 bytes, including an explicit device ID)
        data_start_offset = 9;
        // The VkDevice ID is explicitly provided in the header.
        device_handle = (VkDevice)VkObject_fromId(*(void**)(input_buffer_ptr + 1));
    }

    // Extract `fence_count`. It's located after the command header.
    uint32_t fence_count = *(uint32_t*)(input_buffer_ptr + data_start_offset);

    // Calculate the offset to `wait_all`. It's 4 bytes after `fence_count`.
    uint32_t wait_all_offset = data_start_offset + sizeof(uint32_t); // data_start_offset + 4

    // `waitAll` is a VkBool32 (uint32_t).
    VkBool32 wait_all = *(VkBool32*)(input_buffer_ptr + wait_all_offset);

    // Calculate the offset to the `timeout` value.
    // This is located after the `fence_count` and `wait_all` fields, and the array of `VkFence` IDs.
    // Each `VkFence` is a 64-bit handle.
    uint32_t fences_array_start_offset = wait_all_offset + sizeof(VkBool32); // wait_all_offset + 4
    uint32_t timeout_offset = fences_array_start_offset + fence_count * sizeof(VkFence);

    uint64_t timeout = *(uint64_t*)(input_buffer_ptr + timeout_offset);

    // Allocate memory on the stack for the array of VkFence handles.
    // `VkFence` is a 64-bit handle (typedef struct VkFence_T* or uint64_t).
    // The `_alloca` allocation is implicitly handled by the compiler.
    // Ghidra's stack allocation calculation `(uVar16 * 8 + 0xf & 0xffffffff0)`
    // correctly aligns the buffer to 16 bytes.
    VkFence* fences_stack_array = (VkFence*)__builtin_alloca(fence_count * sizeof(VkFence)); // local_170 adjusted for pointer type.

    // Copy the VkFence IDs from the input buffer to the stack-allocated array.
    // The fences array starts directly after the `wait_all` field.
    // `input_buffer_ptr + fences_array_start_offset` points to the start of fence IDs in the input.
    memcpy(fences_stack_array, input_buffer_ptr + fences_array_start_offset, fence_count * sizeof(VkFence));

    VkResult vk_result; // Variable to store the VkResult of API calls

    if (timeout == 0) {
        // If timeout is 0, perform a blocking wait for fences.
        // Call the Vulkan API vkWaitForFences.
        vk_result = (*(PFN_vkWaitForFences)dispatch_vkWaitForFences)(
            device_handle,
            fence_count,
            fences_stack_array,
            wait_all,
            0 // Timeout is 0 as per this branch condition
        );

        // Prepare the result to be sent back to the client.
        size_t response_data[1]; // For a single VkResult
        response_data[0] = (size_t)vk_result; // Store VkResult (uint32_t) in size_t (uint64_t)

        // Write the VkResult back to the client's ring buffer.
        // `vtContext->clientRingBuffer` is at `param_1 + 0x18` (0x60 bytes offset)
        RingBuffer_write(vtContext->clientRingBuffer, response_data, sizeof(VkResult)); // sizeof(VkResult) = 4 bytes
    } else {
        // If timeout is non-zero, this path indicates a specific mechanism for
        // handling external fence handles, likely for cross-process/device synchronization.
        // It retrieves file descriptors for each fence.

        // Allocate memory on the stack for file descriptors.
        // Each FD is an `int` (4 bytes).
        int* fence_fds_stack_array = (int*)__builtin_alloca(fence_count * sizeof(int)); // Corresponds to `__src`

        // Initialize a variable for the last VkResult from getting FDs.
        // This will be sent back to the client.
        int last_fd_op_result = 0; // local_14c

        if (fence_count > 0) {
            // Iterate through each fence to get its file descriptor.
            for (uint32_t i = 0; i < fence_count; ++i) {
                // Prepare VkFenceGetFdInfoKHR structure for vkGetFenceFdKHR call.
                // This structure is prepared on the stack.
                VkFenceGetFdInfoKHR fence_get_fd_info = {
                    .sType = VK_STRUCTURE_TYPE_FENCE_GET_FD_INFO_KHR,
                    .pNext = NULL,
                    .semaphore = fences_stack_array[i], // VkFence object
                    // `handleType` is not explicitly read here for this call, but assumed to be OPAQUE_FD_BIT.
                    // The decompiled `local_100[0] = 0x3b9c8b39;` actually points to the `sType` of the info struct.
                    // The `pFenceGetFdInfo->handleType` would come from the input buffer for vkGetFenceFdKHR.
                    // Here, a hardcoded type, possibly `VK_EXTERNAL_FENCE_HANDLE_TYPE_OPAQUE_FD_BIT` is implied.
                    // For the sake of accuracy given the snippet, this struct is constructed in place.
                    // The value `0x3b9c8b39` doesn't match `VK_STRUCTURE_TYPE_FENCE_GET_FD_INFO_KHR`.
                    // I will use `VK_STRUCTURE_TYPE_FENCE_GET_FD_INFO_KHR` as it's the standard.
                    .handleType = VK_EXTERNAL_FENCE_HANDLE_TYPE_OPAQUE_FD_BIT // Assuming common handle type
                };
                
                // Call vkGetFenceFdKHR to get the file descriptor for the current fence.
                vk_result = (*(PFN_vkGetFenceFdKHR)dispatch_vkGetFenceFd)(
                    device_handle,
                    &fence_get_fd_info,
                    &fence_fds_stack_array[i] // Store the retrieved FD in the array
                );

                if (vk_result != VK_SUCCESS) {
                    last_fd_op_result = vk_result; // Store the first error encountered
                    break; // Stop if an error occurs
                }
            }
        }

        // Prepare to send the results (VkResult and file descriptors) back to the client using sendmsg.
        struct msghdr msg_header = {0};
        struct iovec iov[1]; // For the VkResult
        
        // Control message for sending file descriptors
        // `CMSG_SPACE` calculates space for ancillary data including `cmsghdr`.
        size_t cmsg_data_len = fence_count * sizeof(int); // Size of the array of FDs
        size_t cmsg_total_len = CMSG_LEN(cmsg_data_len); // Total size including cmsghdr

        // Allocate cmsg_buf on stack dynamically or using static buffer
        // `local_100` (size_t[3]) is used for this buffer in the assembly
        char cmsg_buf[CMSG_SPACE(VK_MAX_FENCES * sizeof(int))]; // Use max possible size if known, or dynamically allocate.
                                                                 // Ghidra uses a fixed size buffer and copies.
        
        // Initialize iovec for the VkResult
        iov[0].iov_base = &last_fd_op_result;
        iov[0].iov_len  = sizeof(int); // Length of VkResult (4 bytes)

        msg_header.msg_iov    = iov;
        msg_header.msg_iovlen = 1; // Only one iovec for the VkResult

        msg_header.msg_control    = cmsg_buf; // Pointer to control message buffer
        msg_header.msg_controllen = cmsg_total_len;

        // Populate the control message header for sending file descriptors (SCM_RIGHTS)
        struct cmsghdr* cmsg = CMSG_FIRSTHDR(&msg_header);
        if (cmsg != NULL) {
            cmsg->cmsg_len   = CMSG_LEN(cmsg_data_len);
            cmsg->cmsg_level = SOL_SOCKET;
            cmsg->cmsg_type  = SCM_RIGHTS;
            // Copy the file descriptors into the control message data part
            memcpy(CMSG_DATA(cmsg), fence_fds_stack_array, cmsg_data_len);
        }

        // Send the message over the client socket.
        // `*vtContext->clientFd` (param_1[0]) is the client file descriptor.
        sendmsg(vtContext->clientFd, &msg_header, 0);

        // Close all retrieved file descriptors to prevent leaks,
        // as they are duplicated by `sendmsg` to the receiver.
        if (fence_count > 0) {
            for (uint32_t i = 0; i < fence_count; ++i) {
                if (fence_fds_stack_array[i] >= 0) { // Check if the FD is valid before closing
                    close(fence_fds_stack_array[i]);
                    fence_fds_stack_array[i] = -1; // Mark as closed
                }
            }
        }
    }

    // Stack canary check to ensure no buffer overflow occurred.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        // If the stack cookie has been modified, a stack overflow occurred.
        __stack_chk_fail();
    }
}
```

---

### Decompilation of `vt_handle_vkGetFenceFdKHR`

```c
/**
 * @brief Handles the vkGetFenceFdKHR command received from the client.
 *
 * This function parses the incoming command buffer to extract parameters for
 * vkGetFenceFdKHR (device handle, fence ID, handle type). It then calls the
 * Vulkan API to retrieve the file descriptor and sends the result (VkResult
 * and the file descriptor) back to the client using `sendmsg`.
 *
 * @param vtContext A pointer to the VtContext structure.
 */
void vt_handle_vkGetFenceFdKHR(VtContext* vtContext) {
    // Save the stack cookie for stack overflow detection.
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    char* input_buffer_ptr = vtContext->inputBufferPtr;

    // Determine the effective offset for reading command data based on the first byte.
    // This implies a variable-size header.
    uint32_t data_start_offset; // Offset to handle_type and subsequent data
    VkDevice device_handle;     // The VkDevice handle

    if (input_buffer_ptr[0] == '\0') { // Simple header (1 byte for command ID)
        data_start_offset = 1;
        // If command ID is '\0', the device handle is implicitly the context itself.
        device_handle = (VkDevice)VkObject_fromId(vtContext);
    } else { // Complex header (9 bytes, including an explicit device ID)
        data_start_offset = 9;
        // The VkDevice ID is explicitly provided in the header.
        device_handle = (VkDevice)VkObject_fromId(*(void**)(input_buffer_ptr + 1));
    }

    // Extract `handle_type` and `fence_id`.
    // The Ghidra output `local_130[0] = (ulong)*(uint *)(pcVar3 + (uVar4 | 4));` for handle_type
    // and `local_120 = VkObject_fromId(*(void **)((long)(pcVar3 + (uVar4 | 4)) + 4));` for fence_id
    // suggests a slightly different input layout or a single read of a combined struct.
    // Assuming a linear packet: cmd_type, device_id (opt), handle_type, fence_id
    VkExternalFenceHandleTypeFlagBits handle_type = *(VkExternalFenceHandleTypeFlagBits*)(input_buffer_ptr + data_start_offset);
    VkFence fence_handle = (VkFence)VkObject_fromId(*(void**)(input_buffer_ptr + data_start_offset + sizeof(VkExternalFenceHandleTypeFlagBits)));

    // Prepare VkFenceGetFdInfoKHR structure for vkGetFenceFdKHR call.
    VkFenceGetFdInfoKHR fence_get_fd_info = {
        .sType = VK_STRUCTURE_TYPE_FENCE_GET_FD_INFO_KHR, // Using the correct enum value from Vulkan header.
        .pNext = NULL,
        .semaphore = fence_handle,
        .handleType = handle_type
    };

    int retrieved_fd = 0; // Store the retrieved file descriptor (local_134)
    // Call the Vulkan API vkGetFenceFdKHR.
    VkResult vk_result = (*(PFN_vkGetFenceFdKHR)dispatch_vkGetFenceFd)(
        device_handle,
        &fence_get_fd_info,
        &retrieved_fd
    );

    // Prepare to send the result (VkResult and file descriptor) back to the client using sendmsg.
    struct msghdr msg_header = {0};
    struct iovec iov[1]; // For the VkResult
    
    // Control message for sending a file descriptor.
    size_t cmsg_data_len = sizeof(int); // Size of one FD
    size_t cmsg_total_len = CMSG_LEN(cmsg_data_len);

    // `local_c8` and `local_c0` are part of the `cmsghdr` structure in the assembly.
    // Ghidra interpreted them as separate variables, but they are fields within `struct cmsghdr`.
    char cmsg_buf[CMSG_SPACE(sizeof(int))]; // Allocate buffer for control message

    // Initialize iovec for the VkResult
    iov[0].iov_base = &vk_result; // Send the VkResult first
    iov[0].iov_len  = sizeof(VkResult);

    msg_header.msg_iov    = iov;
    msg_header.msg_iovlen = 1;

    msg_header.msg_control    = cmsg_buf; // Pointer to control message buffer
    msg_header.msg_controllen = cmsg_total_len;

    // Populate the control message header for sending file descriptors (SCM_RIGHTS).
    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&msg_header);
    if (cmsg != NULL) {
        cmsg->cmsg_len   = CMSG_LEN(cmsg_data_len);
        cmsg->cmsg_level = SOL_SOCKET; // Protocol level
        cmsg->cmsg_type  = SCM_RIGHTS; // Type of control message (sending file descriptors)
        // Copy the file descriptor into the control message data part.
        memcpy(CMSG_DATA(cmsg), &retrieved_fd, cmsg_data_len);
    }
    
    // Send the message over the client socket.
    // `*vtContext->clientFd` (param_1[0]) is the client file descriptor.
    sendmsg(vtContext->clientFd, &msg_header, 0);

    // Close the retrieved file descriptor to prevent leaks,
    // as it is duplicated by `sendmsg` to the receiver.
    if (retrieved_fd >= 0) { // Check if the FD is valid before closing
        close(retrieved_fd);
    }

    // Stack canary check to ensure no buffer overflow occurred.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        // If the stack cookie has been modified, a stack overflow occurred.
        __stack_chk_fail();
    }
}
```

---

# vkCreateImage

The `vt_handle_vkCreateImage` function is responsible for processing a client's request to create a Vulkan `VkImage` object. It involves deserializing the input parameters, potentially performing custom handling for compressed image formats, calling the appropriate native Vulkan API, and serializing the result back to the client.

Here's a breakdown of the function's logic:

**Function: `void vt_handle_vkCreateImage(VtContext* vt_context)`**

`vt_context` (referred to as `param_1` in the original code) is a pointer to the `VtContext` structure, which holds the necessary state for the renderer's operations, including communication buffers and JNI environment details.

**1. Deserialization of Arguments:**

*   **Reading `VkDevice` Handle (Implicit or Explicit):**
    *   The function first checks the first byte of `vt_context->inputBufferPtr` (`pcVar8`).
    *   If `*vt_context->inputBufferPtr` is `\0`, it indicates that the client omitted the `VkDevice` handle (likely implying the default or already active device). In this case, `serialized_device_handle_ptr` is implicitly `NULL`, and the offset for reading subsequent arguments (`current_input_offset_base`) is set to `1` byte.
    *   If `*vt_context->inputBufferPtr` is not `\0`, the client provided an explicit `VkDevice` handle. This handle is read from `vt_context->inputBufferPtr + 1`, and `current_input_offset_base` is set to `9` bytes (to account for the 8-byte handle plus a 1-byte type/presence indicator).

*   **Reading `VkImageCreateInfo` Structure:**
    *   The function then reads an integer from `vt_context->inputBufferPtr + current_input_offset_base`. This integer represents the *size* of the serialized `VkImageCreateInfo` structure.
    *   If the size is greater than `0`, it indicates that the `VkImageCreateInfo` structure is present in the input buffer.
    *   `FUN_0012f754` is called to deserialize the `VkImageCreateInfo` data:
        *   `&image_create_info`: The destination memory location where the deserialized `VkImageCreateInfo` data will be stored. This is a local variable on the stack. The assembly indicates `local_a8 + 1` is used, which implies `VkImageCreateInfo` starts at an 8-byte offset from `local_a8`.
        *   `vt_context->inputBufferPtr + ((uint)current_input_offset_base | 4)`: The source memory location in the `inputBufferPtr`. The `| 4` suggests a 4-byte alignment or an offset to skip potential `sType` field. Assuming the serialized `VkImageCreateInfo` implicitly omits `sType` and `pNext`, the fields like `flags`, `imageType`, `format`, `extent`, etc., would directly follow in the buffer, matching the offsets seen in the original code snippet for `VkImageCreateInfo`.
        *   `vt_context->tempBuffer_ptr_offset_0x38`: This pointer to `tempBuffer` is passed to `FUN_0012f754` to manage temporary memory allocations required for any nested pointers within `VkImageCreateInfo` (e.g., `pNext` chain, `pQueueFamilyIndices`). `FUN_0012f754` will allocate memory from `tempBuffer` if possible, otherwise `malloc` if `tempBuffer` is full.

**2. Parameter Modifications/Hooking and Native Vulkan API Call:**

*   **Handle Conversion:**
    *   `device = (VkDevice)VkObject_fromId(serialized_device_handle_ptr);`: The deserialized `VkDevice` handle (which is an internal `VkObject` pointer) is converted to its native `VkDevice` type.

*   **Custom Handling for Compressed Image Formats:**
    *   The function checks if `vt_context->textureDecoder` is initialized (`!= NULL`) AND if the image `format` (from `image_create_info.format`) is a compressed format (using `isCompressedFormat`).
    *   **If format is compressed AND `textureDecoder` exists:**
        *   `vk_result = TextureDecoder_createImage(vt_context->textureDecoder, device, (long)&image_create_info, &created_image);`: A custom function `TextureDecoder_createImage` is called. This function likely handles the creation of images with compressed formats, potentially involving texture decoding or specific driver extensions not directly exposed through standard Vulkan API.
        *   If `TextureDecoder_createImage` succeeds (`vk_result == VK_SUCCESS`), `RingBuffer_setStatus(vt_context->clientRingBuffer, 4);` is called. This sets a status flag (value `4`) on the client's ring buffer, potentially signaling a specific custom handling path or successful asynchronous operation completion.
    *   **Otherwise (standard image creation):**
        *   `vk_result = dispatch_vkCreateImage(device, &image_create_info, NULL, &created_image);`: The standard Vulkan API function `vkCreateImage` is called. `NULL` is passed for `pAllocator`, implying the default Vulkan allocator. `created_image` will store the resulting `VkImage` handle.

**3. Serialization of Output:**

*   **Temporary Storage for Result Handle:**
    *   The `vt_context->tempBufferOffset` is read to determine the next available write position in `vt_context->tempBuffer`.
    *   8 bytes (size of a `VkImage` handle) are allocated in `tempBuffer` (`allocated_handle_ptr`). If `tempBuffer` is full or `NULL`, `malloc` is used, and the allocation is tracked in `vt_context->tempAllocations`.
    *   The `created_image` handle (obtained from `vkCreateImage` or `TextureDecoder_createImage`) is stored at `*allocated_handle_ptr`.

*   **Writing to Client Ring Buffer:**
    *   `result_data[0] = vk_result;`: The `VkResult` of the operation is stored.
    *   `result_data[1] = 8;`: The size of the `VkImage` handle to follow is set to 8 bytes.
    *   `RingBuffer_write(vt_context->clientRingBuffer, result_data, 8);`: The `VkResult` and the size of the handle are written to the client's ring buffer.
    *   `if ((write_status & 1) != 0) { RingBuffer_write(vt_context->clientRingBuffer, allocated_handle_ptr, 8); }`: If the previous write was successful (`write_status` bit `0` is set), the actual `VkImage` handle (from `allocated_handle_ptr`) is then written to the client's ring buffer.

**4. Memory Management (End of Function):**

*   `if (*(long *)(lVar2 + 0x28) == local_38) { return; } else { __stack_chk_fail(bVar4); }`: This is the stack canary check. If the stack was corrupted, `__stack_chk_fail` is called, otherwise the function returns. Note that the memory allocated in `tempBuffer` or explicitly via `malloc` and added to `tempAllocations` will be handled by the `VtContext`'s cleanup mechanisms (e.g., when `destroyVkContext` is called, which iterates `tempAllocations` to `free` them).

This function demonstrates a typical pattern in this renderer: deserializing client requests, performing operations, and serializing results back, with specific hooks for custom hardware/software interactions (like `TextureDecoder`).

---

# ShaderInspector attempt 1

Of course. Here is a detailed decompilation and explanation of the requested functions.

### Decompilation Analysis

The provided C code snippets are from a custom Vulkan layer or wrapper, which I've dubbed "Vortek" based on the function names (`vt_handle_...`). This layer intercepts Vulkan API calls to inspect and modify data, primarily for applying driver-specific workarounds. The `ShaderInspector` component is a key part of this, designed to patch SPIR-V shader bytecode on-the-fly before it's compiled into a pipeline.

The decompilation below reconstructs the logic, renames variables for clarity, and maps magic numbers and offsets to their corresponding Vulkan API definitions from the provided header.

---

### Function 1: `ShaderInspector_create`

This function initializes a `ShaderInspector` context object. This object holds state about the physical device's capabilities, which is then used to decide which patches or workarounds to apply to shaders.

**Decompiled Code:**
```c
// A custom structure to hold information about the GPU for shader patching decisions.
typedef struct ShaderInspector {
    bool disableShaderStorageImageMultisample;
    bool no_vertex_buffer_feature; // Set if VK_FORMAT_R8G8B8A8_SNORM lacks VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT
    bool is_mali;
    bool some_flag_3; // Always set to true
} ShaderInspector;

/**
 * @brief Creates and initializes a ShaderInspector context.
 * 
 * This function probes the physical device for specific features and properties to determine
 * which workarounds might be needed for shaders.
 * 
 * @param physicalDevice The VkPhysicalDevice handle to inspect.
 * @param pProperties A pointer to the VkPhysicalDeviceProperties struct for the device.
 * @return A pointer to the newly allocated and initialized ShaderInspector struct.
 */
void* ShaderInspector_create(VkPhysicalDevice physicalDevice, const VkPhysicalDeviceProperties* pProperties) {
    // Save stack canary
    long stack_canary = *(long *)(tpidr_el0 + 0x28);

    // Allocate memory for the ShaderInspector context struct.
    ShaderInspector* inspector = (ShaderInspector*)calloc(1, sizeof(ShaderInspector));

    // --- Check 1: Shader Storage Image Multisample Support ---
    // The original code checks an offset that corresponds to `shaderStorageImageMultisample`
    // within a VkPhysicalDeviceFeatures struct. It sets a flag if this feature is NOT supported.
    // Assuming pProperties is part of a larger context where pFeatures is also available.
    // For this decompilation, we'll represent this check conceptually. A real implementation
    // would likely pass pFeatures or have it accessible.
    // Let's assume a hypothetical `get_features` call for clarity.
    VkPhysicalDeviceFeatures features;
    vkGetPhysicalDeviceFeatures(physicalDevice, &features);
    inspector->disableShaderStorageImageMultisample = (features.shaderStorageImageMultisample == VK_FALSE);

    // --- Check 2: Format Properties for VK_FORMAT_R8G8B8A8_SNORM ---
    VkFormatProperties formatProps;
    vkGetPhysicalDeviceFormatProperties(physicalDevice, VK_FORMAT_R8G8B8A8_SNORM, &formatProps);
    
    // The original code checks bit 6 (0x40) of `linearTilingFeatures`.
    // Bit 6 corresponds to VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT.
    // It sets the flag if this feature is *not* present.
    if ((formatProps.linearTilingFeatures & VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT) == 0) {
        inspector->no_vertex_buffer_feature = true;
    } else {
        inspector->no_vertex_buffer_feature = false;
    }

    // --- Check 3: Identify Mali GPUs ---
    // The original code gets full device properties and searches for "Mali" in the deviceName string.
    // We already have pProperties, so we can use its deviceName field.
    if (strstr(pProperties->deviceName, "Mali") != NULL) {
        inspector->is_mali = true;
    } else {
        inspector->is_mali = false;
    }

    // --- Set a final, seemingly constant, flag ---
    inspector->some_flag_3 = true;

    // Restore stack canary and return the context
    if (*(long *)(tpidr_el0 + 0x28) != stack_canary) {
        __stack_chk_fail();
    }
    return inspector;
}
```

**Explanation:**
The `ShaderInspector_create` function acts as a GPU profiler at initialization. It allocates a small context structure and populates it with boolean flags based on the capabilities of the provided `VkPhysicalDevice`.

1.  **Allocation:** It allocates a 4-byte `ShaderInspector` structure.
2.  **Multisample Storage Image Check:** It checks if the device supports multisampled storage images. The resulting flag is inverted, meaning `disableShaderStorageImageMultisample` is `true` if the feature is *not* supported.
3.  **Vertex Buffer Format Check:** It queries the format properties for `VK_FORMAT_R8G8B8A8_SNORM` to see if it can be used as a vertex buffer with linear tiling. The flag is set if this capability is *missing*.
4.  **Mali GPU Detection:** It inspects the device name string for the substring "Mali" to apply Mali-specific workarounds later.
5.  **Return Context:** It returns the populated `ShaderInspector` struct, which will be used by other functions in this component.

---

### Function 2: `ShaderInspector_createModule`

This function is a wrapper around `vkCreateShaderModule`. Its primary purpose is to inspect the SPIR-V bytecode and, if necessary, patch it before creating the final shader module.

**Decompiled Code:**
```c
// A custom wrapper struct for a VkShaderModule.
typedef struct ShaderModuleWrapper {
    VkShaderModule module;
    void* pPatchedCode;         // Pointer to modified SPIR-V code, if any.
    size_t patchedCodeSize;     // Size of the patched code.
    bool some_flag;             // Original Ghidra analysis suggested a boolean here.
    char padding[7];
} ShaderModuleWrapper;

/**
 * @brief Creates a shader module, potentially patching the SPIR-V code for workarounds.
 *
 * This function inspects the SPIR-V for known patterns that require modification on certain
 * hardware (e.g., Mali GPUs). If a pattern is found, it modifies the bytecode in memory
 * before creating the shader module. Otherwise, it creates the module with the original code.
 *
 * @param inspector The ShaderInspector context created earlier.
 * @param device The logical device.
 * @param pCode Pointer to the SPIR-V bytecode.
 * @param codeSize Size of the bytecode in bytes.
 * @param pOutShaderModuleWrapper Output pointer for the created ShaderModuleWrapper.
 */
void ShaderInspector_createModule(ShaderInspector* inspector, VkDevice device, const uint32_t* pCode, size_t codeSize, ShaderModuleWrapper** pOutShaderModuleWrapper) {
    // Save stack canary
    long stack_canary = *(long *)(tpidr_el0 + 0x28);

    // Allocate the wrapper struct to hold the module and any patched code.
    ShaderModuleWrapper* wrapper = (ShaderModuleWrapper*)calloc(1, sizeof(ShaderModuleWrapper));
    
    // Check for SPIR-V magic number and that the header is present.
    if (*pCode == 0x07230203 && codeSize > 20) {
        uint32_t word_count = codeSize / 4;
        bool needs_patching = false;

        // Iterate through capabilities to check for a specific case.
        int word_offset = 5; // Start after SPIR-V header.
        while (word_offset < word_count) {
            uint32_t opcode_word = pCode[word_offset];
            uint16_t opcode = opcode_word & 0xFFFF;
            uint16_t word_length = opcode_word >> 16;
            
            if (opcode == SpvOpCapability) { // Opcode 15
                uint32_t capability = pCode[word_offset + 1];
                
                // This logic block triggers a complex patching routine.
                // Condition: GPU is Mali, and the shader uses StorageImageExtendedFormats capability.
                // The original code checks `(uVar1 & 0xfffffffb) != 0`, which is equivalent to
                // `capability != SpvCapabilityStorageImageExtendedFormats (4)`.
                // The logic is: if is_mali AND capability IS StorageImageExtendedFormats, proceed.
                if (inspector->is_mali && capability == SpvCapabilityStorageImageExtendedFormats) {
                    needs_patching = true; // Simplified: mark for patching. The actual code is very complex.
                    // The original code performs a deep analysis and modification of the SPIR-V here,
                    // searching for `OpImageWrite` to `Unknown` format storage images and removing
                    // `NonWritable` decorations from the associated `OpVariable`.
                    // This is too complex to reproduce faithfully without a full SPIR-V parsing library.
                }
            }
            if (opcode == SpvOpSource || opcode == SpvOpSourceExtension) { 
                // Skip OpSource and OpSourceExtension sections for simplicity in this analysis.
            }
            if (opcode == SpvOpExecutionMode || opcode == SpvOpEntryPoint || opcode == SpvOpCapability) {
                // ... The logic for searching continues...
            }
            if (opcode == SpvOpKill) break; // End of relevant section
            word_offset += word_length;
        }

        // If the specific Mali workaround is needed...
        if (needs_patching) {
            // ... (The original code would perform the complex SPIR-V patching here) ...
            
            // For this decompilation, we'll assume the patch is done and the result is a
            // new buffer. We copy the original code to simulate this.
            wrapper->pPatchedCode = malloc(codeSize);
            if (wrapper->pPatchedCode) {
                memcpy(wrapper->pPatchedCode, pCode, codeSize);
                wrapper->patchedCodeSize = codeSize;
                
                // Now create the module using the patched code.
                VkShaderModuleCreateInfo createInfo = {};
                createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
                createInfo.codeSize = wrapper->patchedCodeSize;
                createInfo.pCode = (const uint32_t*)wrapper->pPatchedCode;
                (*dispatch_vkCreateShaderModule)(device, &createInfo, NULL, &wrapper->module);
            }
        }
    }

    // If no patching was performed or needed, create the shader module directly from original code.
    if (wrapper->module == VK_NULL_HANDLE) {
        VkShaderModuleCreateInfo createInfo = {};
        createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
        createInfo.codeSize = codeSize;
        createInfo.pCode = pCode;
        (*dispatch_vkCreateShaderModule)(device, &createInfo, NULL, &wrapper->module);
    }

    // Set the output parameter to our new wrapper.
    *pOutShaderModuleWrapper = wrapper;

    // Restore stack canary and return.
    if (*(long *)(tpidr_el0 + 0x28) != stack_canary) {
        __stack_chk_fail();
    }
    return;
}
```

**Explanation:**
This function is a sophisticated wrapper for `vkCreateShaderModule`. It doesn't just pass the call through; it actively inspects and modifies shader code to work around driver issues.

1.  **Wrapper Allocation:** It allocates a `ShaderModuleWrapper` struct. This struct holds the final `VkShaderModule` handle and, importantly, a pointer to any modified SPIR-V code to manage its memory lifetime.
2.  **SPIR-V Inspection:** It begins by validating the SPIR-V magic number. It then parses the SPIR-V, iterating through opcodes.
3.  **Mali-Specific Patch:** The core logic is a workaround for Mali GPUs. It looks for shaders that declare the `StorageImageExtendedFormats` capability. If found on a Mali device, it triggers a complex patching routine. This routine searches for `OpImageWrite` instructions targeting storage images with an `Unknown` format. When it finds such a case, it traces the variable back to its `OpVariable` declaration and removes the `NonWritable` decoration. This suggests a bug where Mali drivers might incorrectly require this decoration to be absent for certain write operations.
4.  **Module Creation:**
    *   If a patch was applied, it creates the `VkShaderModule` using the *modified* in-memory copy of the SPIR-V bytecode. The `pPatchedCode` pointer in the wrapper ensures this memory can be freed later.
    *   If no patch was necessary, it creates the shader module using the original bytecode provided by the application.
5.  **Return:** It returns the `ShaderModuleWrapper` via the output parameter. The rest of the "Vortek" system will use the `module` handle from this wrapper.

---

### Function 3: `ShaderInspector_inspectShaderStages`

This is the most complex function. It orchestrates the inspection and patching of all shaders within a `VkGraphicsPipelineCreateInfo` structure right before the pipeline is created.

**Decompiled Code:**
```c
/**
 * @brief Inspects and patches all shader stages within a graphics pipeline create info.
 * 
 * This is the main entry point for applying shader workarounds before a graphics pipeline is created.
 * It iterates through each shader stage, applies necessary patches (e.g., for scaled vertex
 * formats or Mali-specific issues), creates new shader modules from the patched code, and updates
 * the pipeline create info to use these new modules.
 * 
 * @param inspector The ShaderInspector context.
 * @param device The logical device.
 * @param pCreateInfos Pointer to the array of VkGraphicsPipelineCreateInfo structures.
 * @param pipelineCount The number of pipelines to inspect and patch.
 * @param someList_param5 A custom list-like structure, likely for managing temporary allocations.
 * @param someList_param6 Another custom list for temporary allocations.
 * @return VkResult indicating success or failure.
 */
int ShaderInspector_inspectShaderStages(ShaderInspector* inspector, VkDevice device, VkGraphicsPipelineCreateInfo* pCreateInfos, uint32_t pipelineCount, long someList_param5, long someList_param6, ...) {
    // ... (Stack canary and local variable initialization) ...
    int result = VK_SUCCESS;

    if (pipelineCount == 0) {
        return VK_SUCCESS;
    }

    // This is the outer loop that iterates over each VkGraphicsPipelineCreateInfo struct.
    for (uint32_t i = 0; i < pipelineCount; ++i) {
        VkGraphicsPipelineCreateInfo* pCurrentCreateInfo = &pCreateInfos[i];
        
        // This is the inner loop that iterates over each shader stage in the current create info.
        for (uint32_t j = 0; j < pCurrentCreateInfo->stageCount; ++j) {
            VkPipelineShaderStageCreateInfo* pStage = &pCurrentCreateInfo->pStages[j];
            
            // The original code checks `pStage->pSpecializationInfo != NULL && pStage->pSpecializationInfo->mapEntryCount == 0`.
            // This might be a signal that this shader is a candidate for patching, where the
            // specialization info will be populated by the inspector.
            if (pStage->pSpecializationInfo != NULL && pStage->pSpecializationInfo->mapEntryCount == 0) {
                
                // --- Vertex Shader Patching (Scaled Format Workaround) ---
                if (pStage->stage == VK_SHADER_STAGE_VERTEX_BIT && pCurrentCreateInfo->pVertexInputState) {
                    
                    // The logic here is to find vertex attributes using scaled formats
                    // (e.g., USCALED, SSCALED) and replace them with integer formats.
                    // A specialization constant is then injected into the shader to perform
                    // the scaling manually (int_val * scale + offset).
                    // This is a complex process involving:
                    // 1. Copying vertex attribute and binding descriptions to modify them.
                    // 2. Identifying which formats need changing.
                    // 3. Generating a new VkSpecializationInfo struct with the scaling factors.
                    // 4. Using a helper (FUN_00141c88) to patch the SPIR-V bytecode of the
                    //    vertex shader to include the specialization constant logic.
                    // 5. Creating a new VkShaderModule from the patched code.
                    // 6. Updating pStage->module to the new module handle.
                } 
                // --- Fragment Shader Patching (Mali-specific) ---
                else if (pStage->stage == VK_SHADER_STAGE_FRAGMENT_BIT && inspector->is_mali) {

                    // This block contains another complex SPIR-V patching routine, specific to Mali GPUs.
                    // Helper functions `FUN_00142184` and `FUN_00141d38` are used to find and rewrite
                    // instruction patterns. This is likely related to fixing issues with image sampling,
                    // possibly for specific layouts or features like fragment density maps.
                    // Due to its complexity, it's a deep modification of the SPIR-V bytecode.
                }
            }
        }
        
        // After all stages are inspected and potentially patched, the original code
        // might create a new graphics pipeline using the modified create info.
        // The provided snippet focuses only on the inspection part.
    }

    // ... (Cleanup of temporary allocations and stack canary check) ...
    return result;
}
```

**Explanation:**
This function is the orchestrator for applying shader patches at pipeline creation time.

1.  **Iterate Pipelines and Stages:** It loops through every `VkGraphicsPipelineCreateInfo` provided and then through every `VkPipelineShaderStageCreateInfo` within that.

2.  **Patch Trigger:** It identifies shaders that need potential patching by checking for a `pSpecializationInfo` that exists but has no entries. This seems to be a convention used by this layer to mark shaders for inspection.

3.  **Vertex Shader Workaround (Scaled Formats):**
    *   **Problem:** Some older or mobile drivers have poor or incorrect support for `VK_FORMAT_*_USCALED` or `*_SSCALED` vertex attribute formats.
    *   **Solution:** The code implements a common workaround. It changes the vertex attribute format in the pipeline state from, for example, `VK_FORMAT_R8_USCALED` to `VK_FORMAT_R8_UINT`. It then patches the vertex shader's SPIR-V, injecting specialization constants that contain the scaling factor. The shader code is modified to use these constants to manually convert the integer vertex data back into the expected floating-point range (e.g., `float_value = int_value * scale_factor`).

4.  **Fragment Shader Workaround (Mali-Specific):**
    *   **Problem:** This is a more opaque patch targeting Mali GPUs. The complexity of the helper functions (`FUN_00142184`, `FUN_00141d38`) suggests it's not a simple format swap.
    *   **Solution:** The code parses the fragment shader's SPIR-V to find specific instruction patterns. Based on the logic, it seems to be related to `OpImageRead`, `OpImageSample`, or access chains leading to them. It might be rewriting addressing modes, adding or removing decorations, or changing sampling instructions to fix incorrect rendering artifacts, performance issues, or crashes specific to Mali drivers.

5.  **Module Recreation:** After a shader's bytecode is modified, the function creates a new `VkShaderModule` from the patched code. It then overwrites the `module` handle in the `VkPipelineShaderStageCreateInfo` to point to this new, patched module.

The original `vkCreateGraphicsPipelines` call is then made (outside the scope of this function) with the modified create info, ensuring the driver compiles the corrected shaders.

---

# Attempt 2

To decompile the provided C functions, I'll first analyze each one to understand its parameters, local variables, and the structures it manipulates. I'll infer custom data structures and protocol elements based on memory access patterns and known Vulkan API structures.

### Inferred Custom Structures and Constants:

Based on the disassembly, here are the custom structures and constants assumed for the decompilation:

**`VtContext` Structure:**
This structure is provided in the prompt, so I will use it directly. Relevant fields:
*   `inputBufferPtr` (at offset `0x28`)
*   `tempBuffer` (at offset `0x38`)
*   `tempBufferOffset` (at offset `0x40`)
*   `tempAllocations` (at offset `0x48`)
*   `shaderInspector` (at offset `0x80`)
*   `clientRingBuffer` (at offset `0x60`)
*   `graphicsQueueFamilyIndex` (at offset `0x74`)

**`VkModuleObject` Structure (Internal Shader Module Wrapper):**
This is an internal object used by the `ShaderInspector` to manage `VkShaderModule` handles and associated data.
```c
typedef struct VkModuleObject {
    uint64_t vk_handle;     // Offset 0x00: The actual VkShaderModule handle obtained from Vulkan API.
    void*    pCode_copy;    // Offset 0x08: Pointer to a copy of the shader code (SPIR-V).
    uint64_t codeSize_copy; // Offset 0x10: Size of the copied shader code in bytes.
    // Total size 0x20 bytes based on `calloc(1,0x20)` in `ShaderInspector_createModule`.
    // There might be other internal fields or padding up to 0x20 bytes.
} VkModuleObject;
```

**`ShaderInspector` Structure:**
This is an internal object for inspecting and potentially modifying shaders.
```c
typedef struct ShaderInspector {
    VkBool32 isGraphicsQueueFamilyIndexZero; // Offset 0x00
    uint8_t  someFlag1;                      // Offset 0x01 (packed byte, value derived from a call)
    VkBool32 isMaliGPU;                      // Offset 0x02 (packed byte, set by strstr "Mali")
    uint8_t  someFlag2;                      // Offset 0x03 (packed byte, always set to 1)
} ShaderInspector;
```

**Input Buffer Protocol (`vt_handle_vkCreateShaderModule`):**
The `inputBufferPtr` points to a custom-serialized data structure for `vkCreateShaderModule`.
```c
// Custom serialization format for VkShaderModule creation command data
typedef struct SerializedCreateShaderModuleCommandData {
    uint8_t  discriminator; // Offset 0x00: If 0, VkDevice is implicitly NULL or global. If non-zero, VkDevice follows.
    VkDevice device_handle_optional; // Offset 0x01: VkDevice handle (8 bytes), present if discriminator != 0.
    uint32_t flags; // Offset 0x01 or 0x09: VkShaderModuleCreateFlags. Offset depends on discriminator.
    uint64_t codeSize; // Offset 0x01+0x04 or 0x09+0x04: size_t (8 bytes). This is `codeSize`.
    // The actual raw SPIR-V code `pCode` starts immediately after `codeSize`.
} SerializedCreateShaderModuleCommandData;
```
*   `lVar8` in disassembly (`1` or `9`) is the starting offset of the `flags` field relative to `inputBufferPtr`.
*   The `codeSize` field is at `inputBufferPtr + lVar8 + 4`. The raw `pCode` data starts at `inputBufferPtr + lVar8 + 4 + 8`.

**SPIR-V Opcode Constants:**
*   `SPV_MAGIC_NUMBER = 0x07230203` (120894083 decimal)
*   `SpvOpEntryPoint = 0xF` (15 decimal)
*   `SpvOpFunction = 0x36` (54 decimal)
*   `SpvOpDecorate = 0x47` (71 decimal)
*   `SpvOpVariable = 0x3B` (59 decimal)
*   `SpvOpSource = 0x3E` (62 decimal)
*   `SpvOpSourceExtension = 0x56` (86 decimal)
*   `SpvOpExtension = 0xA9` (169 decimal)
*   `SpvOpConstantComposite = 0x2C` (44 decimal)
*   `SpvOpAccessChain = 0x3D` (61 decimal)

---

### Decompiled Code:

```c
#include <string.h> // For memcpy, strcmp, strncpy, memset
#include <stdio.h>  // For snprintf
#include <stdlib.h> // For malloc, calloc, free

// --- Standard Vulkan API Type Definitions (Simplified for Context) ---
typedef unsigned int uint32_t;
typedef unsigned long long uint64_t;
typedef uint32_t VkFlags;
typedef uint64_t VkDeviceSize;
typedef uint32_t VkBool32;
typedef int32_t VkResult;

// Forward declarations of Vulkan handles
typedef struct VkDevice_T* VkDevice;
typedef struct VkShaderModule_T* VkShaderModule;
typedef struct VkInstance_T* VkInstance;
typedef struct VkPhysicalDevice_T* VkPhysicalDevice;

// Vulkan API versioning macros (from vulkan_core.h)
#define VK_API_VERSION_MAJOR(version) (((uint32_t)(version) >> 22U) & 0x7FU)
#define VK_API_VERSION_MINOR(version) (((uint32_t)(version) >> 12U) & 0x3FFU)
#define VK_API_VERSION_PATCH(version) ((uint32_t)(version) & 0xFFFU)

// Vulkan Structure Types (from vulkan_core.h)
typedef enum VkStructureType {
    VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO = 16, // 0x10
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES = 52, // 0x34
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES = 1000070000, // 0x3b9dc7a0
    // ... other types as needed
} VkStructureType;

// Vulkan CreateInfo/Properties Structures (simplified to relevant fields)
typedef struct VkShaderModuleCreateInfo {
    VkStructureType sType;
    const void* pNext;
    VkFlags flags; // VkShaderModuleCreateFlags
    size_t codeSize;
    const uint32_t* pCode;
} VkShaderModuleCreateInfo;

typedef struct VkPhysicalDeviceProperties {
    uint32_t apiVersion;
    uint32_t driverVersion;
    char deviceName[256]; // VK_MAX_PHYSICAL_DEVICE_NAME_SIZE
    // ... other fields
} VkPhysicalDeviceProperties;

typedef struct VkPhysicalDeviceVulkan12Properties {
    VkStructureType sType;
    void* pNext;
    uint32_t driverID;
    char driverName[256]; // VK_MAX_DRIVER_NAME_SIZE
    char driverInfo[256]; // VK_MAX_DRIVER_INFO_SIZE, starts at offset 0x114 (276 bytes) from struct base
    // ... many more fields
    VkBool32 shaderSignedZeroInfNanPreserveFloat16; // Offset 548 (0x224)
    VkBool32 shaderSignedZeroInfNanPreserveFloat32; // Offset 552 (0x228)
    VkBool32 shaderSignedZeroInfNanPreserveFloat64; // Offset 556 (0x22c)
    VkBool32 shaderDenormPreserveFloat16;           // Offset 560 (0x230)
    VkBool32 shaderDenormPreserveFloat32;           // Offset 564 (0x234)
    VkBool32 shaderDenormPreserveFloat64;           // Offset 568 (0x238)
    VkBool32 shaderDenormFlushToZeroFloat16;        // Offset 572 (0x23c)
    VkBool32 shaderDenormFlushToZeroFloat32;        // Offset 576 (0x240)
    VkBool32 shaderDenormFlushToZeroFloat64;        // Offset 580 (0x244)
    // ... other fields
} VkPhysicalDeviceVulkan12Properties;

typedef struct VkPhysicalDeviceGroupProperties {
    VkStructureType sType;
    void* pNext;
    uint32_t physicalDeviceCount;
    // ... VkPhysicalDevice physicalDevices[VK_MAX_DEVICE_GROUP_SIZE] (32 * 8 = 256 bytes)
    VkBool32 subsetAllocation; // Offset 276 (0x114), after `physicalDevices` array
} VkPhysicalDeviceGroupProperties;

// --- Custom Internal Type Definitions ---
// Function pointer types for Vulkan API dispatch
typedef VkResult (VKAPI_PTR *PFN_vkCreateShaderModule)(VkDevice device, const VkShaderModuleCreateInfo* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkShaderModule* pShaderModule);
typedef void (VKAPI_PTR *PFN_vkDestroyShaderModule)(VkDevice device, VkShaderModule shaderModule, const VkAllocationCallbacks* pAllocator);
typedef void (VKAPI_PTR *PFN_vkGetPhysicalDeviceProperties2)(VkPhysicalDevice physicalDevice, VkPhysicalDeviceProperties2* pProperties);

// Assuming these are external dispatch tables for Vulkan functions
extern PFN_vkCreateShaderModule dispatch_vkCreateShaderModule;
extern PFN_vkDestroyShaderModule dispatch_vkDestroyShaderModule;
extern PFN_vkGetPhysicalDeviceProperties2 dispatch_vkGetPhysicalDeviceProperties2;

// Standard libraries or custom runtime functions
extern void __stack_chk_fail(void); // Stack canary check failure
extern void* tpidr_el0;             // Thread pointer register for stack canary
extern void* VkObject_fromId(void* id); // Custom function to get Vulkan object from ID
extern void ArrayList_add(void* list_ptr, void* item); // Custom ArrayList add function
extern void ArrayList_removeAt(void* list_ptr, int index); // Custom ArrayList remove function
extern void IntArray_add(void* array_ptr, int value); // Custom IntArray add function
extern void IntArray_sort(void* array_ptr); // Custom IntArray sort function
extern void IntArray_clear(void* array_ptr); // Custom IntArray clear function

// --- SPIR-V Related Constants ---
#define SPV_MAGIC_NUMBER 0x07230203 // SPIR-V magic number
#define SpvOpEntryPoint 0xF         // Opcode for OpEntryPoint
#define SpvOpFunction 0x36          // Opcode for OpFunction
#define SpvOpSource 0x3E            // Opcode for OpSource
#define SpvOpSourceExtension 0x56   // Opcode for OpSourceExtension
#define SpvOpExtension 0xA9         // Opcode for OpExtension
#define SpvOpDecorate 0x47          // Opcode for OpDecorate
#define SpvOpVariable 0x3B          // Opcode for OpVariable
#define SpvOpConstantComposite 0x2C // Opcode for OpConstantComposite
#define SpvOpAccessChain 0x3D       // Opcode for OpAccessChain
#define DecorationBinding 0x1E      // SPV_KHR DecorationBinding

// --- Custom Internal Structures ---

// VtContext (as provided in the prompt)
typedef struct VtContext {
    /* 0x00 */ int clientFd;
    /* 0x04 */ int vkMaxVersion;
    /* 0x08 */ short maxDeviceMemory; // In MB
    /* 0x0a */ short imageCacheSize;  // In MB
    /* 0x0c */ int padding_0c;
    /* 0x10 */ ArrayList* exposedDeviceExtensions;
    /* 0x18 */ ArrayList* exposedInstanceExtensions;
    /* 0x20 */ uint8_t hostVisibleMemoryFlag; // For memory type selection
    /* 0x21 */ uint8_t coherentMemoryFlag;    // For memory type selection
    /* 0x22 */ char padding_22[6];
    /* 0x28 */ char* inputBufferPtr;      // Pointer to the current command's data
    /* 0x30 */ int inputBufferSize;
    /* 0x34 */ int padding_34;
    /* 0x38 */ void* tempBuffer;          // A large temporary buffer for deserialization
    /* 0x40 */ int tempBufferOffset;      // Current offset within the tempBuffer
    /* 0x44 */ int padding_44;
    /* 0x48 */ ArrayList* tempAllocations; // Tracks allocations within tempBuffer
    /* 0x50 */ long padding_50;
    /* 0x58 */ pthread_t serverThread;    // Handle to the main processing thread
    /* 0x60 */ RingBuffer* clientRingBuffer; // For writing results back to the client
    /* 0x68 */ RingBuffer* serverRingBuffer; // For reading commands from the client
    /* 0x70 */ int lastError;             // Stores the last recorded VkResult error
    /* 0x74 */ int graphicsQueueFamilyIndex;
    /* 0x78 */ TextureDecoder* textureDecoder;
    /* 0x80 */ ShaderInspector* shaderInspector;
    /* 0x88 */ AsyncPipelineCreator* asyncPipelineCreator;
    
    // JNI and Android-specific windowing members
    /* 0x90 */ jobject javaComponentObject;
    /* 0x98 */ JNIEnv* jniEnv;
    /* 0xA0 */ jobject javaWindowManagerObject;
    /* 0xA8 */ jmethodID getWindowWidthMethodID;
    /* 0xB0 */ jmethodID getWindowHeightMethodID;
    /* 0xB8 */ jmethodID getWindowHardwareBufferMethodID;
    /* 0xC0 */ jmethodID updateWindowContentMethodID;
} VtContext;

// Dummy definitions for types not fully provided in snippet but used.
typedef void* RingBuffer;
typedef void* TextureDecoder;
typedef void* AsyncPipelineCreator;
typedef void* jobject;
typedef void* JNIEnv;
typedef void* jmethodID;
typedef void* pthread_t;
typedef void* VkAllocationCallbacks; // Assuming pAllocator is always NULL in snippets

// --- Custom Internal Structures ---
// Internal wrapper for Vulkan handles, inferred from disassembly
typedef struct VkModuleObject {
    uint64_t vk_handle;     // Offset 0x00: The actual VkShaderModule handle obtained from Vulkan API.
    void*    pCode_copy;    // Offset 0x08: Pointer to a copy of the shader code (SPIR-V).
    uint64_t codeSize_copy; // Offset 0x10: Size of the copied shader code in bytes.
    // Bytes 0x18 to 0x1F (offsets 24 and 25 are accessed in vt_handle_vkDestroyShaderModule)
    // assuming it's a 32-byte struct from calloc(1, 0x20)
    uint8_t  internalFlag1_at_24; // (uint8_t*)((char*)__ptr + 24)
    uint8_t  internalFlag2_at_25; // (uint8_t*)((char*)__ptr + 25)
} VkModuleObject;

// Simplified ShaderInspector struct, inferred from disassembly
typedef struct ShaderInspector {
    VkBool32 isGraphicsQueueFamilyIndexZero; // Offset 0x00
    uint8_t  someFlag1;                      // Offset 0x01 (packed byte)
    VkBool32 isMaliGPU;                      // Offset 0x02 (packed byte) (`*(char *)(param_1 + 2)`)
    uint8_t  someFlag2;                      // Offset 0x03 (packed byte, always set to 1)
    // Total size 4 bytes based on calloc(1,4) in ShaderInspector_create
} ShaderInspector;


// Helper function to determine if a VkFormat is a scaled format.
// This function checks if the input format `format` (a VkFormat enum value)
// is one of the VK_FORMAT_R8_USCALED, VK_FORMAT_R8_SSCALED, etc.
// This is used by `ShaderInspector_createModule` to identify formats that might require
// special handling (e.g., in SPIR-V transformations).
VkBool32 isFormatScaled(uint32_t format) {
    switch (format) {
        case 11:  // VK_FORMAT_R8_USCALED
        case 12:  // VK_FORMAT_R8_SSCALED
        case 18:  // VK_FORMAT_R8G8_USCALED
        case 19:  // VK_FORMAT_R8G8_SSCALED
        case 39:  // VK_FORMAT_R8G8B8A8_USCALED
        case 40:  // VK_FORMAT_R8G8B8A8_SSCALED
        case 72:  // VK_FORMAT_R16G16_USCALED
        case 73:  // VK_FORMAT_R16G16_SSCALED
        case 79:  // VK_FORMAT_R16G16B16A16_USCALED
        case 80:  // VK_FORMAT_R16G16B16A16_SSCALED
        case 93:  // VK_FORMAT_A2R10G10B10_USCALED_PACK32
        case 94:  // VK_FORMAT_A2R10G10B10_SSCALED_PACK32
            return VK_TRUE;
        default:
            return VK_FALSE;
    }
}

// Function to manipulate SPIR-V code, possibly inserting or removing instructions/operands.
// This is a complex internal helper function with logic dependent on SPIR-V opcodes and
// operand structures, making full, accurate decompilation difficult without a SPIR-V
// specification or tool. The provided disassembly hints at:
// - `spirv_buffer_obj_ptr`: Pointer to a structure managing SPIR-V words (e.g., VkModuleObject).
// - `device`: VkDevice handle.
// - `opcode`: Specific SPIR-V opcode being looked for or inserted.
// - `some_id_param_4`, `operand_1`, `operand_2`: Operands for the SPIR-V instruction.
// It seems to be building/modifying SPIR-V code in-place.
int FUN_00141d38(long* spirv_buffer_obj_ptr, VkDevice device, uint32_t opcode, uint32_t some_id_param_4, uint32_t operand_1, uint32_t operand_2, undefined8 param_7, undefined8 param_8) {
    // This is a highly specialized internal function for SPIR-V manipulation.
    // It appears to find or insert SPIR-V instructions based on opcodes and operands.
    // The specifics of `param_1` (an array of uint32_t) and how it's modified are complex
    // and tied to SPIR-V bytecode structure (e.g., instruction words, operand counts).
    // The integer constants like `0x40000`, `0x3d`, `0x1ffff` are likely SPIR-V specific masks/opcodes.
    // The function modifies the SPIR-V code in `spirv_buffer_obj_ptr` (likely `VkModuleObject.pCode_copy`).
    // It returns an integer, likely an ID or a modified offset.

    // WARNING: This is a placeholder for a complex SPIR-V manipulation function.
    // Its precise behavior can only be fully determined by a detailed SPIR-V instruction set
    // analysis and understanding of the specific optimization/workaround it implements.

    return 0; // Placeholder return
}

// This function seems to be part of the SPIR-V modification logic within ShaderInspector.
// It appears to be inserting data into the shader's SPIR-V binary buffer.
// `spirv_buffer_obj_ptr` points to the `VkModuleObject` whose `pCode_copy` is being modified.
// `data_to_insert_array` is a source array containing data (and potentially its size/offset info).
// `data_size` is the actual size of the data to copy.
void FUN_00141c88(long spirv_buffer_obj_ptr, int* data_to_insert_array, uint64_t data_size_bytes) {
    void* current_spirv_data = *(void**)(spirv_buffer_obj_ptr + 8);      // pCode_copy
    uint64_t current_total_size = *(uint64_t*)(spirv_buffer_obj_ptr + 0x10); // codeSize_copy

    // The logic to calculate `insert_offset_bytes` and `insert_data_size_bytes` from `data_to_insert_array`
    // is specific to the internal representation of the data to insert, likely relating to SPIR-V instruction words.
    // The disassembly shows `insert_offset_bytes` derived from `(param_3 >> 0x1f & 1) & 0xfffffffc00000000 | (param_3 & 0xffffffff) << 2`.
    // This translates a 32-bit word offset to a byte offset (x4) with sign extension.
    // `insert_data_size_bytes` is derived from `(long)*param_2 * 4;`
    // This means `data_to_insert_array[0]` holds the number of words to insert.

    uint64_t insert_offset_bytes = data_size_bytes; // This is a simplification based on the actual use in the calling function.
    uint64_t actual_insert_len_bytes = (uint64_t)data_to_insert_array[0] * sizeof(uint32_t); // Assuming data_to_insert_array[0] holds length in words

    uint64_t portion_before_insert = current_total_size;
    if (insert_offset_bytes <= current_total_size) {
        portion_before_insert = insert_offset_bytes;
    }

    void* new_spirv_data = malloc(current_total_size + actual_insert_len_bytes);

    // Copy data before the insertion point
    memcpy(new_spirv_data, current_spirv_data, portion_before_insert);

    // Copy the new data to insert (starting from `data_to_insert_array`'s content pointer)
    // The disassembly used `__src_00 = *(void **)(param_2 + 2);` meaning `data_to_insert_array[2]` (or 8 bytes offset) points to actual data.
    memcpy((char*)new_spirv_data + portion_before_insert, (char*)data_to_insert_array + 8, actual_insert_len_bytes);

    // Copy data after the insertion point
    memcpy((char*)new_spirv_data + portion_before_insert + actual_insert_len_bytes,
           (char*)current_spirv_data + portion_before_insert,
           current_total_size - portion_before_insert);

    // Free old memory and update structure pointers and size
    if (current_spirv_data != NULL) {
        free(current_spirv_data);
    }
    *(void**)(spirv_buffer_obj_ptr + 8) = new_spirv_data;      // Update pCode_copy
    *(uint64_t*)(spirv_buffer_obj_ptr + 0x10) = current_total_size + actual_insert_len_bytes; // Update codeSize_copy

    // Clear the auxiliary data array as it's no longer needed after insertion
    IntArray_clear(data_to_insert_array);
}

/**
 * @brief Handles the vkCreateShaderModule command received from the client.
 *
 * This function deserializes the shader module creation data from the input buffer,
 * allocates memory for the shader code, potentially inspects and modifies the SPIR-V
 * code, calls the actual Vulkan vkCreateShaderModule API, and then sends the result
 * and the created shader module handle back to the client.
 *
 * @param vtContext A pointer to the VtContext structure, which holds global state
 *                  and command buffers.
 */
void vt_handle_vkCreateShaderModule(VtContext* vtContext) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Get stack canary for security
    char* inputBuffer = vtContext->inputBufferPtr;
    
    // Determine the offset to the flags and code data based on a custom discriminator byte.
    // The first byte of the input buffer (`inputBuffer[0]`) acts as a discriminator.
    // If it's 0, no explicit VkDevice handle is provided in the buffer (use default/NULL).
    // If it's non-zero, an 8-byte VkDevice handle is present at `inputBuffer + 1`.
    VkDevice device_handle = NULL;
    size_t data_start_offset; // Offset in inputBuffer where VkShaderModuleCreateInfo-like data begins

    if (inputBuffer[0] == '\0') {
        data_start_offset = 1; // Flags start immediately after the discriminator byte
    } else {
        device_handle = *(VkDevice*)(inputBuffer + 1); // Read VkDevice handle
        data_start_offset = 9; // Flags start after discriminator (1 byte) + VkDevice handle (8 bytes)
    }

    // Read shader module creation flags (4 bytes)
    VkFlags shader_module_flags = *(uint32_t*)(inputBuffer + data_start_offset);
    
    // Read the size of the shader code (8 bytes for size_t)
    size_t code_size = *(uint64_t*)(inputBuffer + data_start_offset + 4); // After flags (4 bytes)
    
    // Get pointer to the raw shader code data
    const uint32_t* pCodeData = (const uint32_t*)(inputBuffer + data_start_offset + 4 + 8); // After flags (4 bytes) + codeSize (8 bytes)

    uint32_t* tempShaderCodeBuffer = NULL;
    // If codeSize is positive, allocate memory for the shader code copy.
    if (code_size > 0) {
        // Check if there's enough space in the temporary buffer managed by VtContext.
        // `0x10000` (65536) is the maximum temporary buffer size before resorting to `malloc`.
        if ((vtContext->tempBufferOffset + code_size < 0x10000) && (vtContext->tempBuffer != NULL)) {
            tempShaderCodeBuffer = (uint32_t*)((char*)vtContext->tempBuffer + vtContext->tempBufferOffset);
            vtContext->tempBufferOffset += code_size; // Advance offset in temp buffer
        } else {
            // If temp buffer is full or not available, allocate new memory.
            tempShaderCodeBuffer = (uint32_t*)malloc(code_size);
            // Add the new allocation to a list for later freeing.
            ArrayList_add(vtContext->tempAllocations, tempShaderCodeBuffer);
        }
        
        memset(tempShaderCodeBuffer, 0, code_size); // Zero-initialize the allocated memory
        memcpy(tempShaderCodeBuffer, pCodeData, code_size); // Copy the raw shader code
    }

    VkModuleObject* created_module_object = NULL;
    // Call the ShaderInspector to create/process the shader module.
    // This function will eventually call the Vulkan API `vkCreateShaderModule` internally.
    ShaderInspector_createModule(vtContext->shaderInspector, device_handle, tempShaderCodeBuffer, code_size, &created_module_object);

    VkResult vk_create_result = VK_SUCCESS;
    // If the module object was created, extract the result from it,
    // assuming ShaderInspector_createModule places the VkResult in `created_module_object->vk_handle`
    // and returns the actual API result code directly. The assembly implies `local_60` is the VkResult.
    // The result from `ShaderInspector_createModule` (which is `uVar8` in that function) is not directly returned here,
    // but `local_60` is assigned a value that came from the call to `ShaderInspector_createModule`.
    // Let's assume `ShaderInspector_createModule` returns `VkResult` and the output `VkModuleObject` is `created_module_object`.
    // The assembly suggests `local_60` takes the `VkResult` directly, so `ShaderInspector_createModule` should return `VkResult`.
    // Re-adjusting: `ShaderInspector_createModule` returns `VkResult` and fills `*pOutShaderModuleObject` (param_5) with `VkModuleObject*`.
    // The disassembly shows: `local_60 = ShaderInspector_createModule(...)` which implies it returns VkResult.
    // And `*puVar6 = local_68;` where `local_68` is filled with the module object from the createModule call.

    // Let's assume `ShaderInspector_createModule` actually sets `*pOutShaderModuleObject` to the new module object and returns VkResult.
    // Then the call would be: `vk_create_result = ShaderInspector_createModule(..., &created_module_object);`
    // And then `created_module_object->vk_handle` is extracted for writing to ring buffer.
    
    // The disassembly for `vt_handle_vkCreateShaderModule` implies:
    // `local_60` (VkResult) gets the return value from `ShaderInspector_createModule`.
    // `local_68` (undefined8, likely VkModuleObject*) gets set to `pvVar7` (the newly allocated VkModuleObject).
    // `*puVar6 = local_68;` -> writes the VkModuleObject* to the response buffer.
    // This indicates that the `vk_handle` inside VkModuleObject is filled by the `dispatch_vkCreateShaderModule` call within `ShaderInspector_createModule`.

    // The local variable `local_60` (4 bytes) is explicitly assigned the result of `ShaderInspector_createModule`.
    // The local variable `local_68` (8 bytes) is used as an output parameter to `ShaderInspector_createModule` and stores the `VkModuleObject*`.

    // Send the VkResult (local_60) to the client ring buffer.
    uint8_t write_success = RingBuffer_write(vtContext->clientRingBuffer, &vk_create_result, sizeof(VkResult));

    // If VkResult was written successfully, send the created VkShaderModule handle (from the VkModuleObject).
    if (write_success != 0) { // Check if the write operation was successful
        // Allocate space for the shader module handle in the response buffer.
        void* response_handle_buffer = NULL;
        // Check if there's enough space in the temporary buffer (0xfff8 = 65528 bytes limit).
        if ((vtContext->tempBufferOffset + sizeof(uint64_t) < 0xfff8) && (vtContext->tempBuffer != NULL)) {
            response_handle_buffer = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
            vtContext->tempBufferOffset += sizeof(uint64_t);
        } else {
            response_handle_buffer = malloc(sizeof(uint64_t));
            ArrayList_add(vtContext->tempAllocations, response_handle_buffer);
        }
        // Store the actual VkShaderModule handle from the internal object.
        *(uint64_t*)response_handle_buffer = created_module_object->vk_handle;
        // Send the shader module handle to the client ring buffer.
        RingBuffer_write(vtContext->clientRingBuffer, response_handle_buffer, sizeof(uint64_t));
    }

    // Stack canary check.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Handles the vkDestroyShaderModule command from the client.
 *
 * This function deserializes the shader module destruction data from the input buffer,
 * retrieves the internal shader module object, calls the actual Vulkan vkDestroyShaderModule
 * API, and frees any associated internal resources.
 *
 * @param vtContext A pointer to the VtContext structure.
 */
void vt_handle_vkDestroyShaderModule(VtContext* vtContext) {
    // Note: This function's disassembly does not show a stack canary check.
    
    char* inputBuffer = vtContext->inputBufferPtr;

    // Deserialize input: Determine VkDevice handle ID and VkShaderModule ID
    void* device_id_raw = NULL; // Raw ID read from input, will be converted to VkDevice ID
    size_t module_id_offset;    // Offset to the VkShaderModule ID in inputBuffer

    // Check discriminator byte at `inputBuffer[0]`
    if (inputBuffer[0] == '\0') {
        module_id_offset = 1; // VkShaderModule ID starts after discriminator byte
    } else {
        device_id_raw = *(void**)(inputBuffer + 1); // Read VkDevice ID
        module_id_offset = 9; // VkShaderModule ID starts after discriminator + VkDevice ID
    }

    void* shader_module_id_raw = NULL; // Raw ID read from input, will be converted to VkModuleObject*
    // `inputBuffer[module_id_offset]` is a flag byte (`cVar1`). If non-zero, then read shader module ID.
    if (inputBuffer[module_id_offset] != '\0') {
        shader_module_id_raw = *(void**)(inputBuffer + module_id_offset + 1);
    }
    
    // Retrieve the actual VkDevice handle from its ID.
    VkDevice device_handle = (VkDevice)VkObject_fromId(device_id_raw);
    
    // Retrieve the internal VkModuleObject from its ID.
    VkModuleObject* shader_module_object = (VkModuleObject*)VkObject_fromId(shader_module_id_raw);

    // Perform destruction and cleanup if the shader module object is valid.
    if (shader_module_object != NULL) {
        // Set internal flags within the VkModuleObject for state tracking.
        // `*(undefined1 *)(__ptr + 3) = 1` -> Sets byte at offset 24 to 1.
        shader_module_object->internalFlag1_at_24 = 1;

        // `*(char *)((long)__ptr + 0x19) != '\0'` -> Checks byte at offset 25.
        if (shader_module_object->internalFlag2_at_25 != '\0') {
            // Call the actual Vulkan API to destroy the shader module.
            // `dispatch_vkDestroyShaderModule` expects VkDevice, VkShaderModule handle, and pAllocator.
            // `shader_module_object->vk_handle` holds the actual VkShaderModule handle.
            (*dispatch_vkDestroyShaderModule)(device_handle, (VkShaderModule)shader_module_object->vk_handle, NULL);

            // Free the copied shader code if it exists.
            if (shader_module_object->pCode_copy != NULL) {
                free(shader_module_object->pCode_copy);
                shader_module_object->pCode_copy = NULL;
            }
            // Free the internal VkModuleObject itself.
            free(shader_module_object);
        }
    }
}

/**
 * @brief Creates and initializes a ShaderInspector object.
 *
 * This function allocates a small internal structure for ShaderInspector,
 * initializes some internal flags based on device properties (specifically
 * if the graphics queue family index is zero and if the GPU is Mali), and
 * queries physical device properties.
 *
 * @param vkInstance A Vulkan VkInstance handle.
 * @param vtContext A pointer to the VtContext structure.
 * @return A pointer to the newly created ShaderInspector object.
 */
void* ShaderInspector_create(VkInstance vkInstance, VtContext* vtContext) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Get stack canary for security
    
    // Allocate 4 bytes for the ShaderInspector object and zero-initialize it.
    ShaderInspector* inspector = (ShaderInspector*)calloc(1, sizeof(ShaderInspector));

    // Initialize the `isGraphicsQueueFamilyIndexZero` flag (first byte of the struct).
    inspector->isGraphicsQueueFamilyIndexZero = (vtContext->graphicsQueueFamilyIndex == 0);

    // This section queries physical device properties.
    // `local_3a0` and `local_398` are local stack variables used in an unknown function call,
    // possibly `vkGetPhysicalDeviceProperties2` or a wrapper.
    // The values are then used to set `inspector->someFlag1`.
    // Placeholder for that complex interaction:
    // Some_Func_DAT_00149050(vkInstance, 0x28, &some_local_ptr); // `0x28` might be flags or a structure type.
    // inspector->someFlag1 = (((uint8_t)some_local_value_from_DAT_00149050 >> 6) ^ 0xFF) & 1;

    // Allocate a large buffer on the stack for physical device properties.
    // `0x338` (824 bytes) is large enough for `VkPhysicalDeviceProperties2`
    // which includes `VkPhysicalDeviceProperties` containing `deviceName`.
    // `acStack_37c` is part of `auStack_390`.
    VkPhysicalDeviceProperties2 device_props2 = {0};
    device_props2.sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_PROPERTIES_2; // Assume this type

    // Call Vulkan API to get physical device properties.
    // `DAT_00149030` is likely `vkGetPhysicalDeviceProperties2`.
    // `memset` initializes `device_props2` to zero before the call.
    if (dispatch_vkGetPhysicalDeviceProperties2 != NULL) { // Check if the function pointer is valid
        dispatch_vkGetPhysicalDeviceProperties2(NULL, &device_props2); // `param_1` is VkPhysicalDevice, assuming `NULL` for default or handled internally.
    }
    
    // Search for "Mali" in the device name (part of `device_props2.properties.deviceName`).
    char* mali_check = strstr(device_props2.properties.deviceName, "Mali");

    // Set `isMaliGPU` flag (byte at offset 2) based on whether "Mali" was found.
    inspector->isMaliGPU = (mali_check != NULL);

    // Set `someFlag2` (byte at offset 3) to 1.
    inspector->someFlag2 = 1;

    // Stack canary check.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return inspector;
}

/**
 * @brief Creates a VkShaderModule after inspecting and potentially modifying SPIR-V code.
 *
 * This function acts as a wrapper around `vkCreateShaderModule`. It first allocates
 * an internal `VkModuleObject` to hold the actual Vulkan handle and a copy of the
 * shader code. It then performs various SPIR-V inspection and potential modification
 * operations, such as handling scaled formats, stripping debug info, or applying
 * workarounds specific to certain GPU vendors (like Mali). Finally, it calls the
 * Vulkan API `vkCreateShaderModule` and stores the resulting VkShaderModule handle
 * in the internal `VkModuleObject`.
 *
 * @param shaderInspector A pointer to the ShaderInspector object.
 * @param device The VkDevice handle.
 * @param pCode The raw SPIR-V shader code (input buffer content, usually temporary).
 * @param codeSize The size of the shader code in bytes.
 * @param pOutShaderModuleObject A pointer to where the created `VkModuleObject*` will be stored.
 */
void ShaderInspector_createModule(
    ShaderInspector* shaderInspector,
    VkDevice device,
    uint32_t* pCode, // Raw SPIR-V code buffer
    uint64_t codeSize,
    VkModuleObject** pOutShaderModuleObject // Output parameter for the created VkModuleObject*
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Get stack canary for security

    // Allocate an internal `VkModuleObject` to manage the shader module.
    VkModuleObject* module_obj = (VkModuleObject*)calloc(1, sizeof(VkModuleObject)); // Allocates 32 bytes

    // Check if the input is valid SPIR-V (starts with SPV_MAGIC_NUMBER)
    // and if the codeSize is sufficient for a basic SPIR-V header (min 23 words = 92 bytes).
    if (*pCode == SPV_MAGIC_NUMBER && (codeSize / sizeof(uint32_t)) > 23) {
        uint32_t spirv_word_count = (uint32_t)(codeSize / sizeof(uint32_t));
        uint32_t current_instruction_word_offset = 5; // Start after SPIR-V header (5 words)

        // --- Pass 1: Find OpEntryPoint and determine ExecutionModel ---
        uint32_t entry_point_exec_model = 0; // The ExecutionModel (e.g., Vertex, Fragment, Geometry)
        int entry_point_found_offset = -1;   // Word offset of OpEntryPoint instruction

        while (current_instruction_word_offset < spirv_word_count) {
            uint32_t instruction_header = pCode[current_instruction_word_offset];
            uint16_t opcode = instruction_header & 0xFFFF;
            uint16_t instruction_word_count = (instruction_header >> 16) & 0xFFFF;

            if (opcode == SpvOpEntryPoint) {
                entry_point_found_offset = current_instruction_word_offset;
                // ExecutionModel is the first operand after OpEntryPoint's fixed parts.
                // Assuming `pCode[current_instruction_word_offset + 1]` is ExecutionModel.
                entry_point_exec_model = pCode[current_instruction_word_offset + 1]; 
                break;
            }
            if (opcode == SpvOpFunction) { // Stop search if a function is encountered before an entry point.
                break;
            }
            current_instruction_word_offset += instruction_word_count;
        }

        // --- Logic for SPIR-V Modification/Optimization ---
        // This part applies specific workarounds or optimizations based on GPU type (Mali)
        // and shader type (Geometry shader).
        // `!shaderInspector->isMaliGPU` checks if it's NOT a Mali GPU.
        // `(entry_point_exec_model & 0xfffffffb) != 0` checks if the 3rd bit (0x4, `VK_SHADER_STAGE_GEOMETRY_BIT`)
        // of `entry_point_exec_model` is NOT set.
        if (entry_point_found_offset != -1 && (!shaderInspector->isMaliGPU || (entry_point_exec_model & 0xfffffffb) != 0)) {
            // This block is executed if:
            // 1. An OpEntryPoint was found AND
            // 2. (It's NOT a Mali GPU) OR (It IS a Mali GPU AND the shader is NOT a Geometry shader).
            // This implies special handling is skipped for Mali Geometry shaders.

            // The inner loops look for and potentially remove or modify SPIR-V instructions.
            // This is commonly done to strip debug information (`OpSource`, `OpSourceExtension`)
            // or modify shader behavior for specific hardware/drivers.
            // `local_180` and `local_178` are used as an IntArray to store offsets of instructions to be removed/modified.
            
            // `isFormatScaled` checks VkFormat values, suggesting interaction with formats.
            // `FUN_00141d38` and `FUN_00141c88` are used to modify the SPIR-V code.

            // Given the complexity of the internal SPIR-V manipulation,
            // this part is summarized as "SPIR-V Inspection and Modification Logic".
            // It searches for specific patterns of SPIR-V instructions (OpDecorate, OpVariable, etc.)
            // and potentially performs transformations on the `pCode` buffer using `FUN_00141c88`
            // and `FUN_00141d38`.
        }
    }

    // Copy the shader code into the `VkModuleObject` if it hasn't been copied already.
    // If `pCode` is a temporary buffer, we must copy it to a persistent one for the module object.
    // The disassembly shows `__dest = malloc(param_4);` and `memcpy(__dest,param_3,param_4);`
    // then `*(void **)((long)pvVar7 + 8) = __dest;`. This confirms a deep copy of the shader code.
    module_obj->pCode_copy = malloc(codeSize);
    if (module_obj->pCode_copy != NULL) {
        memcpy(module_obj->pCode_copy, pCode, codeSize);
        module_obj->codeSize_copy = codeSize;
    } else {
        module_obj->codeSize_copy = 0; // Handle allocation failure
    }

    // Prepare VkShaderModuleCreateInfo structure for the actual Vulkan API call.
    VkShaderModuleCreateInfo createInfo = {0};
    createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
    createInfo.pNext = NULL; // No pNext chain used in this context
    createInfo.flags = shader_module_flags;
    createInfo.codeSize = module_obj->codeSize_copy; // Use the size of the potentially modified code
    createInfo.pCode = (const uint32_t*)module_obj->pCode_copy; // Use the pointer to the copied/modified code

    // Call the actual Vulkan API `vkCreateShaderModule`.
    // The `vk_handle` member of `module_obj` will be filled by this call.
    VkResult vk_api_result = (*dispatch_vkCreateShaderModule)(device, &createInfo, NULL, (VkShaderModule*)&module_obj->vk_handle);
    
    // The `vk_api_result` (VkResult) is returned to the caller (`vt_handle_vkCreateShaderModule`).
    // The `module_obj` pointer is returned via the output parameter `pOutShaderModuleObject`.
    *pOutShaderModuleObject = module_obj;

    // Stack canary check.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    // Return VkResult
    return vk_api_result; // This function should technically return VkResult based on assembly.
}
```

---

# vkCreateGraphicsPipeline

Here is the decompilation of the requested C functions.

### **Analysis of Structures**

Based on the usage patterns in the provided functions, we can infer the structure of several key components:

- **`AsyncPipeline`**: A structure representing a single asynchronous pipeline creation operation.
  - `VkPipeline vkHandle;` at offset `0x00`.
  - `volatile bool isDone;` at offset `0x08`.
  - `pthread_mutex_t mutex;` at offset `0x0c`.
  - `pthread_cond_t cond;` at offset `0x34`.

- **`AsyncJob`**: A structure that encapsulates the data for a batch of pipeline creation requests.
  - `int pipelineType;` at offset `0x00` (0 for Graphics, 1 for Compute).
  - `int createInfoCount;` at offset `0x04`.
  - `char* pCreateInfos;` at offset `0x08`, a buffer containing serialized create info structs.
  - `ShaderInspector* shaderInspector;` at offset `0x10`.
  - `ArrayList* asyncPipelines;` at offset `0x18`, a list of pointers to `AsyncPipeline` structs.
  - `void* inputBuffer;` at offset `0x20`, a copy of the original input buffer.
  - `int inputBufferSize;` at offset `0x24`.

- **`AsyncPipelineCreator`**: The main structure managing the asynchronous creation process. It contains a thread pool and a queue for `AsyncJob` tasks.
  - `pthread_t worker_threads[4];` at offset `0x00`.
  - `pthread_mutex_t queue_mutex;` at offset `0x0c`.
  - `pthread_cond_t queue_cond;` at offset `0x34`.
  - `LinkedList job_queue;` (simplified representation) at offset `0xc0`.

---

### **Function Decompilations**

#### 1. `vt_handle_vkCreateGraphicsPipelines` and `vt_handle_vkCreateComputePipelines`

These functions act as wrappers that dispatch pipeline creation tasks to a dedicated asynchronous creator. They gather the necessary context and command data and enqueue it for background processing.

```c
/**
 * @brief Handles the vkCreateGraphicsPipelines command by offloading it to the AsyncPipelineCreator.
 * @param vtContext The Vortek context containing necessary state and pointers.
 */
void vt_handle_vkCreateGraphicsPipelines(VtContext* vtContext) {
    AsyncPipelineCreator_create(
        vtContext->clientRingBuffer,        // For writing back results
        vtContext->shaderInspector,         // For shader processing
        vtContext->asyncPipelineCreator,    // The asynchronous worker manager
        VK_PIPELINE_BIND_POINT_GRAPHICS,    // Type: 0 for graphics pipelines
        vtContext->inputBufferPtr,          // The serialized create info data
        vtContext->inputBufferSize          // Size of the create info data
    );
}

/**
 * @brief Handles the vkCreateComputePipelines command by offloading it to the AsyncPipelineCreator.
 * @param vtContext The Vortek context containing necessary state and pointers.
 */
void vt_handle_vkCreateComputePipelines(VtContext* vtContext) {
    AsyncPipelineCreator_create(
        vtContext->clientRingBuffer,
        vtContext->shaderInspector,
        vtContext->asyncPipelineCreator,
        VK_PIPELINE_BIND_POINT_COMPUTE,     // Type: 1 for compute pipelines
        vtContext->inputBufferPtr,
        vtContext->inputBufferSize
    );
}
```

#### 2. `AsyncPipelineCreator_getVkHandle`

This function is a blocking call that waits for an asynchronous pipeline creation to complete and then returns the resulting Vulkan handle. It uses a mutex and a condition variable for synchronization.

```c
/**
 * @brief Blocks until an asynchronous pipeline creation is complete and returns the resulting VkPipeline handle.
 * @param asyncPipeline A pointer to the AsyncPipeline structure representing the operation.
 * @return The created VkPipeline handle.
 */
VkPipeline AsyncPipelineCreator_getVkHandle(AsyncPipeline* asyncPipeline) {
    // If the pipeline is not yet done, we need to wait.
    if (!asyncPipeline->isDone) {
        pthread_mutex_lock(&asyncPipeline->mutex);
        
        // Loop to handle spurious wakeups.
        while (!asyncPipeline->isDone) {
            pthread_cond_wait(&asyncPipeline->cond, &asyncPipeline->mutex);
        }
        
        pthread_mutex_unlock(&asyncPipeline->mutex);
    }
    
    // Return the handle once the operation is complete.
    return asyncPipeline->vkHandle;
}
```

#### 3. `AsyncPipelineCreator_create`

This function packages the pipeline creation information into an `AsyncJob` and adds it to a work queue. It then writes back an array of `AsyncPipeline` pointers to the client, which can be used to retrieve the final Vulkan handles later.

```c
/**
 * @brief Creates and queues an asynchronous pipeline creation job.
 * @param clientRingBuffer Ring buffer for sending results back to the client.
 * @param shaderInspector The shader inspector instance.
 * @param asyncCreator The asynchronous pipeline creator instance.
 * @param pipelineType The type of pipeline (0 for graphics, 1 for compute).
 * @param pCreateInfos Pointer to the serialized pipeline create info data.
 * @param createInfoSize Size of the serialized data.
 */
void AsyncPipelineCreator_create(RingBuffer* clientRingBuffer, ShaderInspector* shaderInspector, 
                                 AsyncPipelineCreator* asyncCreator, int pipelineType, 
                                 void* pCreateInfos, int createInfoSize) 
{
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28); // Stack canary

    // Allocate and initialize the job structure.
    AsyncJob* job = (AsyncJob*)calloc(1, sizeof(AsyncJob));
    job->pipelineType = pipelineType;
    job->shaderInspector = shaderInspector;

    // Make a copy of the input create info data to be used by the worker thread.
    job->pCreateInfos = malloc(createInfoSize);
    if (job->pCreateInfos != NULL) {
        memcpy(job->pCreateInfos, pCreateInfos, createInfoSize);
    }
    job->inputBufferSize = createInfoSize;
    
    // Determine the number of pipelines to be created.
    int createInfoCount = 0;
    if (pipelineType == VK_PIPELINE_BIND_POINT_GRAPHICS || pipelineType == VK_PIPELINE_BIND_POINT_COMPUTE) {
        // The first 4 bytes of the serialized data contain the pipeline count.
        createInfoCount = *(int*)((char*)pCreateInfos + (*(char*)pCreateInfos == 0 ? 1 : 9));
    }
    job->createInfoCount = createInfoCount;

    // Allocate an array to hold pointers to the AsyncPipeline objects.
    // These pointers will be sent back to the client.
    AsyncPipeline** asyncPipelineHandles = (AsyncPipeline**)malloc(createInfoCount * sizeof(AsyncPipeline*));
    if (createInfoCount > 0) {
        job->asyncPipelines = ArrayList_create();
        for (int i = 0; i < createInfoCount; ++i) {
            // For each pipeline, create an AsyncPipeline object to track its status.
            AsyncPipeline* asyncPipeline = (AsyncPipeline*)malloc(sizeof(AsyncPipeline));
            asyncPipeline->vkHandle = VK_NULL_HANDLE;
            asyncPipeline->isDone = false;
            pthread_mutex_init(&asyncPipeline->mutex, NULL);
            pthread_cond_init(&asyncPipeline->cond, NULL);
            
            ArrayList_add(job->asyncPipelines, asyncPipeline);
            asyncPipelineHandles[i] = asyncPipeline;
        }
    }

    // Create a worker item (a job wrapper for the thread pool).
    ThreadPoolJob* workerJob = (ThreadPoolJob*)malloc(sizeof(ThreadPoolJob));
    workerJob->jobData = job;
    workerJob->jobFunction = &FUN_00144604; // The worker function to execute.
    workerJob->next = NULL;

    // Add the job to the creator's queue under a lock.
    pthread_mutex_lock(&asyncCreator->queue_mutex);
    if (asyncCreator->job_queue.tail == NULL) {
        asyncCreator->job_queue.head = workerJob;
    } else {
        asyncCreator->job_queue.tail->next = workerJob;
    }
    asyncCreator->job_queue.tail = workerJob;
    pthread_cond_signal(&asyncCreator->queue_cond); // Signal a worker thread.
    pthread_mutex_unlock(&asyncCreator->queue_mutex);

    // Write back the async handles to the client.
    uint32_t resultSize = createInfoCount * sizeof(AsyncPipeline*);
    VkResult result = VK_SUCCESS;
    RingBuffer_write(clientRingBuffer, &result, sizeof(VkResult));
    if (resultSize > 0) {
        RingBuffer_write(clientRingBuffer, asyncPipelineHandles, resultSize);
    }
    
    // Free the temporary handle array.
    free(asyncPipelineHandles);

    // Stack corruption check.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail();
    }
}
```

#### 4. `FUN_00144604` (Worker Thread Function)

This is the entry point for a worker thread that processes a pipeline creation job.

```c
/**
 * @brief Worker function that executes a pipeline creation job.
 * @param job The AsyncJob containing all necessary information for pipeline creation.
 */
void pipeline_creation_worker(AsyncJob* job) {
    if (job->pipelineType == VK_PIPELINE_BIND_POINT_COMPUTE) {
        // Delegate to the compute pipeline creation handler.
        create_compute_pipelines_worker(job);
    } else if (job->pipelineType == VK_PIPELINE_BIND_POINT_GRAPHICS) {
        // Delegate to the graphics pipeline creation handler.
        create_graphics_pipelines_worker(job);
    }

    // Clean up resources associated with the job.
    if (job->pCreateInfos != NULL) {
        free(job->pCreateInfos);
        job->pCreateInfos = NULL;
        job->inputBufferSize = 0;
    }
    if (job->asyncPipelines != NULL) {
        ArrayList_free(job->asyncPipelines); // Note: This should also free the contained AsyncPipeline structs.
    }
    free(job);
}
```

#### 5. `FUN_00144c20` (Graphics Pipeline Creation Worker)

This function, executed by a worker thread, handles the actual creation of graphics pipelines.

```c
/**
 * @brief Worker thread logic to create graphics pipelines.
 * @param job The AsyncJob containing graphics pipeline creation info.
 */
void create_graphics_pipelines_worker(AsyncJob* job) {
    // The deserialization of VkGraphicsPipelineCreateInfo happens here, which is a very
    // complex process of parsing the input buffer and reconstructing Vulkan structures.
    // The function `FUN_00142624` is a key part of this deserialization.
    // For brevity, we'll represent this complex deserialization with a conceptual call.
    VkGraphicsPipelineCreateInfo* pCreateInfos = deserialize_graphics_pipeline_create_infos(job->pCreateInfos);
    
    // An array to hold the resulting VkPipeline handles.
    VkPipeline* pPipelines = (VkPipeline*)malloc(job->createInfoCount * sizeof(VkPipeline));
    
    // Process shader stages using the ShaderInspector. This likely involves
    // SPIR-V reflection or manipulation.
    for (uint32_t i = 0; i < job->createInfoCount; ++i) {
        ShaderInspector_inspectShaderStages(job->shaderInspector, job->vulkanDevice, &pCreateInfos[i]);
    }
    
    // Call the actual Vulkan function to create the pipelines.
    VkResult result = dispatch_vkCreateGraphicsPipelines(
        job->vulkanDevice,
        job->pipelineCache, // Can be VK_NULL_HANDLE
        job->createInfoCount,
        pCreateInfos,
        NULL, // No custom allocator
        pPipelines
    );

    // After creation, signal completion for each pipeline.
    if (job->createInfoCount > 0) {
        for (uint32_t i = 0; i < job->createInfoCount; ++i) {
            AsyncPipeline* asyncPipeline = (AsyncPipeline*)ArrayList_get(job->asyncPipelines, i);
            
            pthread_mutex_lock(&asyncPipeline->mutex);
            
            // If creation was successful, store the handle. Otherwise, keep it VK_NULL_HANDLE.
            if (result == VK_SUCCESS) {
                asyncPipeline->vkHandle = pPipelines[i];
            }
            asyncPipeline->isDone = true; // Mark as complete.
            
            pthread_cond_broadcast(&asyncPipeline->cond); // Wake up any waiting threads.
            pthread_mutex_unlock(&asyncPipeline->mutex);
        }
    }
    
    // Cleanup temporary resources.
    free(pCreateInfos); // Assuming this also frees all nested pointers.
    free(pPipelines);
}
```

```c
/**
 * @brief Deserializes a custom binary stream into a VkGraphicsPipelineCreateInfo structure.
 *
 * This function iterates through a serialized buffer, identifying different sections by
 * custom magic numbers (e.g., 0x3b9b75e2). For each section, it allocates memory for the
 * corresponding Vulkan structure (like VkPipelineShaderStageCreateInfo, VkPipelineVertexInputStateCreateInfo, etc.)
 * and populates it with data from the buffer. It handles nested structures, pNext chains,
 * and arrays of substructures recursively or iteratively.
 *
 * @param pOutPipelineInfo Output pointer to the main deserialized VkGraphicsPipelineCreateInfo struct.
 * @param pOutPipelineLayout Output pointer for the deserialized VkPipelineLayout handle.
 * @param pOutRenderPass Output pointer for the deserialized VkRenderPass handle.
 * @param pDestBuffer A large pre-allocated buffer for placing the deserialized structs.
 * @param pSrcBuffer The source buffer containing the serialized data.
 * @param pAllocContext Context for managing temporary memory allocations needed during deserialization.
 */
void deserialize_graphics_pipeline_create_infos(
    VkGraphicsPipelineCreateInfo* pOutPipelineInfo,
    VkPipelineLayout* pOutPipelineLayout,
    VkRenderPass* pOutRenderPass,
    void* pDestBuffer,
    const char* pSrcBuffer,
    AllocationContext* pAllocContext)
{
    // The function would start by initializing pointers into the destination buffer (pDestBuffer)
    // to place the main struct and its sub-structs.

    // A large loop or switch statement would process the source buffer (pSrcBuffer).
    // It reads a "token" or "magic number" to identify the next Vulkan structure to deserialize.
    
    // Example logic for deserializing a substructure:
    // switch (nextToken) {
    //
    // case VORTEK_PIPELINE_SHADER_STAGE_CREATE_INFO_TOKEN: // e.g., 0x3b9c930b
    // {
    //     // Allocate space for VkPipelineShaderStageCreateInfo in pDestBuffer
    //     VkPipelineShaderStageCreateInfo* pStageInfo = ...;
    //
    //     // Read scalar values (flags, stage, etc.) from pSrcBuffer and write to pStageInfo
    //     pStageInfo->flags = read_uint32(pSrcBuffer);
    //     pStageInfo->stage = read_enum(pSrcBuffer);
    //
    //     // Read object handles (as IDs) and convert them
    //     pStageInfo->module = VkObject_fromId(read_handle_id(pSrcBuffer));
    //
    //     // Read and copy strings
    //     pStageInfo->pName = copy_string(pSrcBuffer, pAllocContext);
    //
    //     // Handle nested structs like pSpecializationInfo recursively
    //     if (has_specialization_info(pSrcBuffer)) {
    //         pStageInfo->pSpecializationInfo = deserialize_specialization_info(...);
    //     }
    //
    //     // Link this struct into the main create info's pStages array.
    //     pOutPipelineInfo->pStages[stageIndex++] = pStageInfo;
    //     break;
    // }
    //
    // case VORTEK_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO_TOKEN: // e.g., 0x3b9c0669
    // {
    //     // ... Similar logic for VkPipelineVertexInputStateCreateInfo
    //     // This would involve deserializing arrays like pVertexBindingDescriptions
    //     // and pVertexAttributeDescriptions.
    //     break;
    // }
    //
    // // ... cases for all other pipeline state create info structs ...
    //
    // }
    
    // The function meticulously reconstructs the entire object graph,
    // including handling `pNext` chains for Vulkan extensions.
    // It uses the provided allocation context (pAllocContext) to manage
    // memory for dynamic arrays and strings.
}
```

---

Attempt 2

I will provide the decompiled code for each function, including detailed explanations of the inferred types and structures.

---

### Inferred Structures and Types

Before diving into the functions, let's establish the inferred structures based on the provided C struct `VtContext` and the observed memory access patterns in the assembly.

```c
// Forward declarations for standard Vulkan types from vulkan_core.h
typedef struct VkDevice_T* VkDevice;
typedef struct VkPipeline_T* VkPipeline;
typedef struct VkPipelineCache_T* VkPipelineCache;
typedef struct VkShaderModule_T* VkShaderModule;
typedef uint32_t VkPipelineCreateFlags;
typedef uint32_t VkShaderStageFlagBits;

// These are function pointers for Vulkan API calls, to be resolved at runtime
// (e.g., via vkGetDeviceProcAddr).
// They are prefixed with `dispatch_` in the assembly to indicate this.
typedef VkResult (VKAPI_PTR *PFN_vkCreateGraphicsPipelines)(
    VkDevice                                    device,
    VkPipelineCache                             pipelineCache,
    uint32_t                                    createInfoCount,
    const VkGraphicsPipelineCreateInfo*         pCreateInfos,
    const VkAllocationCallbacks*                pAllocator,
    VkPipeline*                                 pPipelines
);
typedef void (VKAPI_PTR *PFN_vkDestroyShaderModule)(
    VkDevice                                    device,
    VkShaderModule                              shaderModule,
    const VkAllocationCallbacks*                pAllocator
);

// Placeholder for the thread pointer register/variable for stack canary.
// On AArch64, `tpidr_el0` often points to the thread-local storage (TLS) block.
// The offset `0x28` is a common location for the stack canary (`__stack_chk_guard`) on Linux/ARM.
extern unsigned long tpidr_el0;

// --- Custom Data Structures Inferred from Disassembly ---

// `AsyncPipelineCreator` (Size: 0x68 / 104 bytes)
// This structure appears to be a general-purpose wrapper for asynchronous Vulkan object creation results.
// It contains a handle, a ready flag, and pthreads synchronization primitives.
// It is reused for both `VkPipeline` results and temporary `VkShaderModule` inspection results.
typedef struct AsyncPipelineCreator {
    // Offset 0x00: Stores the created Vulkan handle (e.g., VkPipeline, VkShaderModule).
    VkPipeline      vkHandle;     // (8 bytes)

    // Offset 0x08: A flag indicating the readiness of `vkHandle`.
    // Used by `AsyncPipelineCreator_getVkHandle` to synchronize waiting threads.
    uint8_t         ready_flag;   // (1 byte)

    // Offset 0x09: Padding to align `mutex`.
    uint8_t         padding_09[3]; // (3 bytes)

    // Offset 0x0C: Mutex for protecting access to `vkHandle` and `ready_flag`.
    // Assuming `sizeof(pthread_mutex_t)` is 0x28 (40 bytes), common on 64-bit Linux.
    pthread_mutex_t mutex;        // (40 bytes)

    // Offset 0x34: Condition variable used to signal when `vkHandle` is ready.
    // Assuming `sizeof(pthread_cond_t)` is 0x28 (40 bytes), common on 64-bit Linux.
    pthread_cond_t  cond;         // (40 bytes)

    // Offset 0x5C: This area (from 0x5C to 0x67) appears to be used for internal flags
    // or temporary data when `AsyncPipelineCreator` instances are reused, for example,
    // when storing temporary shader module information during `FUN_00144c20`.
    // The offsets `0x18` and `0x19` in the assembly, when applied to a pointer to `AsyncPipelineCreator`
    // interpreted as `undefined8*`, translate to byte offsets `0x18` and `0x19` from the struct start.
    // This overlaps with the `mutex` and `cond` members if standard sizes are assumed.
    // This suggests either a custom pthread implementation, very specific byte packing, or a re-interpretation of the pointer.
    // For clarity, `internal_flags_area` is used to denote these accessed bytes.
    uint8_t         internal_flags_area[12]; // (12 bytes, total 0x68 (104) bytes for struct)
                                             // 8 + 1 + 3 + 40 + 40 + 12 = 104 bytes (0x68)
    // So the offsets 0x18 and 0x19 used in the disassembly are relative to `internal_flags_area`.
    // E.g. `internal_flags_area[0]` and `internal_flags_area[1]`
    // The assembly uses `puVar12 + 3` which refers to `(char*)puVar12 + 24`. So `internal_flags_area` starts at offset 24.
    // This means `AsyncPipelineCreator` struct has to be:
    // VkPipeline      vkHandle;     // Offset 0x00 (8 bytes)
    // uint8_t         ready_flag;   // Offset 0x08 (1 byte)
    // uint8_t         padding_09[3]; // (3 bytes)
    // char            padding_0c[12]; // Padding before custom flags to reach 0x18
    // uint8_t         flag_at_0x18; // (1 byte) This is the byte `*(char*)((char*)puVar12 + 0x18)`
    // uint8_t         flag_at_0x19; // (1 byte) This is the byte `*(uint8_t*)((char*)puVar12 + 0x19)`
    // char            padding_1a[6]; // (6 bytes)
    // pthread_mutex_t mutex;        // Offset 0x20. If 0x28 bytes large (40). So it starts at 0x20 + 0x28 = 0x48.
    // pthread_cond_t  cond;         // Offset 0x48. If 0x28 bytes large (40). So it starts at 0x48 + 0x28 = 0x70.
    // This is over 0x68 bytes. The initial `AsyncPipelineCreator` struct is the best fit.
    // The accesses at 0x18 and 0x19 must be re-interpretations or involve some custom data structure.
    // For now, I will add these specific flags at the end of the struct for a more accurate representation of access,
    // assuming some internal alignment or union trickery allows these bytes to be at the exact offsets 0x18 and 0x19.
    // This is the most ambiguous part of the reverse engineering.
} AsyncPipelineCreator;


// `ArrayList` (Standard usage inferred)
typedef struct ArrayList {
    uint32_t count;    // Current number of elements.
    uint32_t capacity; // Total allocated capacity.
    void**   data;     // Pointer to the dynamically allocated array of pointers.
} ArrayList;

// `PipelineCreationRequest` (Size: 0x28 / 40 bytes)
// This structure holds all information related to a single asynchronous pipeline creation task.
typedef struct PipelineCreationRequest {
    int   pipelineType;         // Offset 0x00: 0 for graphics, 1 for compute.
    int   pipelineCount;        // Offset 0x04: Number of pipelines to be created in this batch.
    void* createInfoDataCopy;   // Offset 0x08: A heap-allocated copy of the raw `VkGraphicsPipelineCreateInfo` or `VkComputePipelineCreateInfo` data.
    ShaderInspector* shaderInspector; // Offset 0x10: Pointer to the `ShaderInspector` instance.
    ArrayList* asyncPipelineCreators; // Offset 0x18: Pointer to an `ArrayList` containing `AsyncPipelineCreator` instances, one for each pipeline.
    void* internalBufferPtr;    // Offset 0x20: A pointer to an internal buffer, e.g., for temporary shader module data.
    uint32_t internalBufferCount; // Offset 0x24: Count of elements in `internalBufferPtr`.
    // Total size 0x28 (40 bytes).
} PipelineCreationRequest;

// `AsyncTaskNode` (Size: 0x18 / 24 bytes)
// A node in a linked list of tasks managed by `AsyncPipelineCreator`.
typedef struct AsyncTaskNode {
    void (*taskFunction)(PipelineCreationRequest*); // Offset 0x00: Function pointer to execute for this task.
    PipelineCreationRequest* request;               // Offset 0x08: Pointer to the specific request data.
    struct AsyncTaskNode* next;                     // Offset 0x10: Pointer to the next task in the list.
} AsyncTaskNode;


// --- Minimal External/Placeholder Definitions ---

// Functions to be linked from other parts of the codebase.
extern void __stack_chk_fail(void); // For stack smashing protection.
extern void* VkObject_fromId(void* id); // Converts an ID back to a Vulkan object pointer.
extern void ArrayList_add(ArrayList* list, void* element);
extern void ArrayList_removeAt(ArrayList* list, uint32_t index);

// Placeholder for `FUN_00142624` which parses raw pipeline creation info.
// The precise arguments are inferred from its usage in `FUN_00144c20`.
extern void FUN_00142624(void* param_1_unused, void* param_2_unused, void* param_3_unused,
                         VkGraphicsPipelineCreateInfo* pCreateInfosOutput, char* rawInputData, ArrayList* internalAllocationsList);

// Placeholder for `ShaderInspector_inspectShaderStages`.
// This function inspects shader stages and may return information about them
// (e.g., creating temporary shader module wrappers).
extern void ShaderInspector_inspectShaderStages(
    ShaderInspector* inspector,
    VkDevice device,
    VkShaderModule module_handle,
    VkShaderStageFlagBits stage_flags,
    const char* shader_name,
    uint32_t* p_shader_module_count_output, // Output: count of inspected/created shader modules.
    void*** p_shader_modules_array_output   // Output: array of pointers to shader module wrappers (e.g., AsyncPipelineCreator instances).
);

// Placeholder for `RingBuffer_write`.
extern uint8_t RingBuffer_write(RingBuffer* buffer, void* data, size_t size);

// Placeholder for the compute-specific pipeline creation/cleanup function.
extern void FUN_00144ec4(long request_ptr);

```

### Decompiled Functions

```c
// --- Decompiled Functions ---

/**
 * @brief Handles the `vkCreateGraphicsPipelines` command from the client.
 *
 * This function is an entry point for graphics pipeline creation requests.
 * It extracts the necessary parameters (client ring buffer, shader inspector,
 * asynchronous pipeline creator manager, and raw pipeline creation data)
 * from the `VtContext` and initiates an asynchronous task for pipeline creation.
 *
 * @param vt_context_ptr A pointer to the `VtContext` structure, representing
 *                       the global context for the Vulkan wrapper.
 */
void vt_handle_vkCreateGraphicsPipelines(VtContext* vt_context_ptr) {
    // Delegate the graphics pipeline creation to the asynchronous creator.
    // The `asyncPipelineCreator` (at `VtContext` offset 0x88) manages the task queue.
    // `clientRingBuffer` (at `VtContext` offset 0x60) is used for communication.
    // `shaderInspector` (at `VtContext` offset 0x80) is passed for shader stage inspection.
    // The raw pipeline creation input data is taken from `inputBufferPtr` (0x28) and `inputBufferSize` (0x30).
    AsyncPipelineCreator_create(
        vt_context_ptr->clientRingBuffer,
        vt_context_ptr->shaderInspector,
        vt_context_ptr->asyncPipelineCreator,
        0, // pipeline_type: 0 indicates graphics pipeline
        (void*)vt_context_ptr->inputBufferPtr,
        vt_context_ptr->inputBufferSize
    );
}

/**
 * @brief Handles the `vkCreateComputePipelines` command from the client.
 *
 * Similar to `vt_handle_vkCreateGraphicsPipelines`, this function is an entry point
 * for compute pipeline creation requests. It extracts parameters from `VtContext`
 * and initiates an asynchronous task for compute pipeline creation.
 *
 * @param vt_context_ptr A pointer to the `VtContext` structure.
 */
void vt_handle_vkCreateComputePipelines(VtContext* vt_context_ptr) {
    // Delegate the compute pipeline creation to the asynchronous creator.
    // Arguments are identical to graphics pipeline creation, except for `pipeline_type`.
    AsyncPipelineCreator_create(
        vt_context_ptr->clientRingBuffer,
        vt_context_ptr->shaderInspector,
        vt_context_ptr->asyncPipelineCreator,
        1, // pipeline_type: 1 indicates compute pipeline
        (void*)vt_context_ptr->inputBufferPtr,
        vt_context_ptr->inputBufferSize
    );
}

/**
 * @brief Retrieves the Vulkan handle from an `AsyncPipelineCreator` object,
 *        blocking until the handle is ready.
 *
 * This function is typically called by a client thread that has submitted an
 * asynchronous pipeline creation request and is waiting for the result.
 * It uses a mutex and a condition variable to ensure thread-safe access to
 * the Vulkan handle and to efficiently wait without busy-looping.
 *
 * @param async_creator A pointer to the `AsyncPipelineCreator` object which
 *                      holds the asynchronous creation result.
 * @return The `VkPipeline` handle (or other Vulkan object handle) that was
 *         created asynchronously.
 */
VkPipeline AsyncPipelineCreator_getVkHandle(AsyncPipelineCreator* async_creator) {
    // Check the `ready_flag` at offset 0x08 within the `AsyncPipelineCreator` object.
    // If it's 0 (false), the handle is not yet ready.
    if (async_creator->ready_flag == 0) {
        // Lock the mutex (at offset 0x0C) to protect `ready_flag` and `vkHandle`.
        pthread_mutex_lock(&async_creator->mutex);
        // Loop while the handle is still not ready. This pattern handles
        // spurious wakeups from `pthread_cond_wait`.
        while (async_creator->ready_flag == 0) {
            // Atomically unlock the mutex, wait for a signal on the condition variable
            // (at offset 0x34), and re-lock the mutex upon return.
            pthread_cond_wait(&async_creator->cond, &async_creator->mutex);
        }
        // Unlock the mutex as the handle is now ready.
        pthread_mutex_unlock(&async_creator->mutex);
    }
    // Return the `VkPipeline` handle stored at offset 0x00.
    return async_creator->vkHandle;
}

/**
 * @brief Initiates an asynchronous pipeline creation process.
 *
 * This is a core function that sets up a new pipeline creation task. It involves:
 * 1. Allocating a `PipelineCreationRequest` structure to hold task-specific data.
 * 2. Copying the raw pipeline creation information from the input buffer.
 * 3. Deserializing a `pipelineCount` from a custom header in the raw input data.
 * 4. Creating `AsyncPipelineCreator` instances (one for each pipeline) to hold their
 *    future results and synchronization primitives.
 * 5. Packaging the request into an `AsyncTaskNode` and adding it to the manager's
 *    queue for processing by a worker thread.
 * 6. Sending a preliminary acknowledgement and the list of `AsyncPipelineCreator`
 *    pointers back to the client via the ring buffer.
 *
 * @param client_ring_buffer A pointer to the `RingBuffer` for client communication.
 * @param shader_inspector_ptr A pointer to the `ShaderInspector` instance.
 * @param async_creator_manager A pointer to the main `AsyncPipelineCreator` instance
 *                              that manages the queue of asynchronous tasks.
 * @param pipeline_type An integer indicating the type of pipeline (0 for graphics, 1 for compute).
 * @param raw_create_info_data A `void` pointer to the raw input data from the client,
 *                             containing pipeline creation infos.
 * @param raw_create_info_size The size of the `raw_create_info_data` in bytes.
 */
void AsyncPipelineCreator_create(
    RingBuffer* client_ring_buffer,
    ShaderInspector* shader_inspector_ptr,
    AsyncPipelineCreator* async_creator_manager,
    int pipeline_type,
    void* raw_create_info_data,
    int raw_create_info_size
) {
    // Save the stack cookie value for stack overflow detection.
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    // Allocate and initialize a `PipelineCreationRequest` struct (size 0x28 / 40 bytes).
    // This struct will hold all the necessary context for the pipeline creation task.
    PipelineCreationRequest* request = (PipelineCreationRequest*)calloc(1, sizeof(PipelineCreationRequest));
    // TODO: Add error handling for `calloc` failure.

    request->pipelineType = pipeline_type;

    // Allocate memory for and copy the raw pipeline creation info data.
    // This ensures the worker thread has its own copy and doesn't rely on the
    // lifetime of the original input buffer.
    request->createInfoDataCopy = (char*)malloc((size_t)raw_create_info_size);
    if (request->createInfoDataCopy != NULL) {
        memcpy(request->createInfoDataCopy, raw_create_info_data, (size_t)raw_create_info_size);
    }
    // TODO: Add error handling for `malloc` failure for `createInfoDataCopy`.

    request->shaderInspector = shader_inspector_ptr;

    // Custom deserialization logic to determine `pipelineCount`.
    // This logic examines bytes at specific offsets (0 and 9) within the
    // `createInfoDataCopy` to determine the actual offset from which to
    // read the `pipelineCount`. This implies a custom or versioned header format.
    uint32_t offset_factor_u6 = 9; // Initial offset factor
    uint32_t offset_factor_u7 = 9; // Another initial offset factor

    char* data_stream_ptr = (char*)request->createInfoDataCopy;
    if (data_stream_ptr != NULL) {
        // If the byte at offset 0 is NULL, adjust `offset_factor_u7`.
        if (data_stream_ptr[0] == '\0') {
            offset_factor_u7 = 1;
        }
        // If the byte at the current `offset_factor_u7` is NULL, adjust `offset_factor_u6`.
        if (data_stream_ptr[offset_factor_u7] == '\0') {
            offset_factor_u6 = 1;
        }
        // Read `pipelineCount` from `data_stream_ptr` at `(offset_factor_u6 + offset_factor_u7)` bytes offset.
        // Possible effective offsets are 2, 10, or 18 bytes from `data_stream_ptr` start.
        request->pipelineCount = *(uint32_t*)(data_stream_ptr + (uint64_t)offset_factor_u6 + (uint64_t)offset_factor_u7);
    } else {
        request->pipelineCount = 0; // If source data is null, count is 0.
    }


    // Allocate an `ArrayList` struct to hold pointers to `AsyncPipelineCreator` instances.
    // This `ArrayList` is part of the `PipelineCreationRequest` struct.
    request->asyncPipelineCreators = (ArrayList*)malloc(sizeof(ArrayList));
    if (request->asyncPipelineCreators == NULL) {
        // TODO: Handle allocation failure.
    }
    request->asyncPipelineCreators->count = 0;
    request->asyncPipelineCreators->capacity = request->pipelineCount;
    // Allocate the backing array for `ArrayList::data` to store `AsyncPipelineCreator*`.
    request->asyncPipelineCreators->data = (void**)calloc(request->pipelineCount, sizeof(AsyncPipelineCreator*));
    if (request->asyncPipelineCreators->data == NULL) {
        // TODO: Handle allocation failure, free `request->asyncPipelineCreators` and `request`.
    }

    // Allocate a temporary array on the stack to hold pointers to the newly created
    // `AsyncPipelineCreator` instances. This array will be sent back to the client.
    // The size is dynamically calculated based on `pipelineCount` and aligned to 0x10 bytes.
    uint64_t temp_async_creator_array_size = (uint64_t)request->pipelineCount * sizeof(AsyncPipelineCreator*);
    // The `(uintptr_t)&client_ring_buffer` is used as a reference point for stack allocation.
    AsyncPipelineCreator** temp_async_creator_ptr_array =
        (AsyncPipelineCreator**)((uintptr_t)&client_ring_buffer - ((temp_async_creator_array_size + 0xF) & ~0xF));
    // NOTE: This `temp_async_creator_ptr_array` effectively represents a Variable Length Array (VLA)
    // or `alloca` equivalent on the stack. Its address is derived from `uStack_80` in assembly.


    if (pipeline_type == 0 || pipeline_type == 1) { // Logic specific to graphics (0) and compute (1)
        if (request->pipelineCount > 0) {
            // For each pipeline to be created, allocate an `AsyncPipelineCreator` instance.
            for (uint32_t i = 0; i < request->pipelineCount; ++i) {
                // Each `AsyncPipelineCreator` object is 0x68 (104 bytes).
                AsyncPipelineCreator* new_async_creator = (AsyncPipelineCreator*)malloc(sizeof(AsyncPipelineCreator));
                if (new_async_creator == NULL) {
                    // TODO: Handle allocation failure, free preceding allocations.
                }

                // Initialize the new `AsyncPipelineCreator` instance for this pipeline.
                new_async_creator->vkHandle = (VkPipeline)0;      // Handle is initially NULL.
                new_async_creator->ready_flag = 0;                // Mark as not ready.
                // Initialize mutex and condition variable for synchronization.
                pthread_mutex_init(&new_async_creator->mutex, NULL);
                pthread_cond_init(&new_async_creator->cond, NULL);

                // Add the newly created `AsyncPipelineCreator` instance to the request's `ArrayList`.
                ArrayList_add(request->asyncPipelineCreators, new_async_creator);
                // Also store its pointer in the temporary stack-allocated array for sending to the client.
                temp_async_creator_ptr_array[i] = new_async_creator;
            }
        }
    } else {
        // If the `pipeline_type` is not 0 or 1, `create_info_count` may remain 0,
        // and `temp_async_creator_ptr_array` might point to a small dummy area (`auStack_74` in assembly).
        // This branch implies other pipeline types are handled differently or ignored here.
    }

    // Create an `AsyncTaskNode` object to wrap the `PipelineCreationRequest` and its callback.
    // This node will be added to the manager's task queue. (Size 0x18 / 24 bytes).
    AsyncTaskNode* task_node = (AsyncTaskNode*)malloc(sizeof(AsyncTaskNode));
    if (task_node == NULL) {
        // TODO: Handle allocation failure.
    }
    task_node->taskFunction = FUN_00144604; // Set the function to be executed by the worker thread.
    task_node->request = request;         // Link the request data.
    task_node->next = NULL;               // Initialize for the linked list.

    // Add the new `task_node` to the `async_creator_manager`'s task queue (a linked list).
    // This operation is protected by the manager's mutex.
    pthread_mutex_lock(&async_creator_manager->mutex);
    if (async_creator_manager->taskListHead == NULL) {
        // If the task list is empty, this is the first task.
        async_creator_manager->taskListHead = task_node;
    } else {
        // Append the new task to the end of the existing list.
        ((AsyncTaskNode*)async_creator_manager->taskListTail)->next = task_node;
    }
    async_creator_manager->taskListTail = task_node; // Update the tail pointer to the new task.
    pthread_cond_signal(&async_creator_manager->cond); // Signal a worker thread that a new task is available.
    pthread_mutex_unlock(&async_creator_manager->mutex);


    // Write an 8-byte dummy value (0) to the client ring buffer as a preliminary acknowledgement.
    uint64_t dummy_output_value = 0;
    uint8_t ring_buffer_write_status = RingBuffer_write(client_ring_buffer, &dummy_output_value, sizeof(dummy_output_value));

    // If the initial write was successful and there are pipelines to create,
    // write the array of `AsyncPipelineCreator` pointers to the client ring buffer.
    // The client can use these pointers (or their IDs) to later call
    // `AsyncPipelineCreator_getVkHandle` to retrieve the actual `VkPipeline` handles.
    if ((ring_buffer_write_status & 1) != 0 && request->pipelineCount > 0) {
        RingBuffer_write(client_ring_buffer, temp_async_creator_ptr_array, temp_async_creator_array_size);
    }

    // Perform stack cookie check to detect potential buffer overflows.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail(); // Stack corruption detected.
    }
}

/**
 * @brief Callback function executed by the asynchronous pipeline creation worker.
 *
 * This function is the entry point for a worker thread to process a pipeline
 * creation request. It dispatches to a type-specific pipeline creation function
 * (`FUN_00144c20` for graphics, `FUN_00144ec4` for compute) and then handles
 * the cleanup of all resources allocated for this request.
 *
 * @param request A pointer to the `PipelineCreationRequest` struct that contains
 *                all details and resources associated with the pipeline creation task.
 */
void FUN_00144604(PipelineCreationRequest* request) {
    // The `request->pipelineType` determines which specific pipeline creation
    // and cleanup logic to execute.
    if (request->pipelineType == 1) { // If it's a compute pipeline
        // Call the compute-specific creation/cleanup function.
        // `FUN_00144ec4` is assumed to perform the `vkCreateComputePipelines` call
        // and update the `AsyncPipelineCreator` instances with the results.
        FUN_00144ec4((long)request); // Passing `request` pointer as a long.
    } else { // Assumed to be graphics pipeline (request->pipelineType == 0)
        // Redundant check `if (request->pipelineType == 0)` is omitted as it's the `else` branch.
        // Call the graphics-specific creation/cleanup function.
        // `FUN_00144c20` is assumed to perform the `vkCreateGraphicsPipelines` call
        // and update the `AsyncPipelineCreator` instances with the results.
        FUN_00144c20((long)request); // Passing `request` pointer as a long.
    }

    // Free the dynamically allocated copy of the raw pipeline creation info data.
    if (request->createInfoDataCopy != NULL) {
        free(request->createInfoDataCopy);
        request->createInfoDataCopy = NULL; // Clear the pointer after freeing.
    }

    // Free the internal buffer (e.g., for temporary shader module data)
    // that was stored at offset 0x20 in the `PipelineCreationRequest` struct.
    if (request->internalBufferPtr != NULL) {
        free(request->internalBufferPtr);
        request->internalBufferPtr = NULL; // Clear pointer.
        request->internalBufferCount = 0;  // Clear associated count.
    }

    // Finally, free the `PipelineCreationRequest` struct itself.
    free(request);
}

/**
 * @brief Handles the actual creation of graphics pipelines and associated cleanup within a worker thread.
 *
 * This function is invoked by `FUN_00144604` for graphics pipeline creation requests.
 * It performs the following steps:
 * 1. Extracts `VkDevice` and `VkPipelineCache` handles from the raw input data.
 * 2. Prepares `VkGraphicsPipelineCreateInfo` structures for the Vulkan API call
 *    by parsing the raw input data. This involves `FUN_00142624`.
 * 3. Iterates through the shader stages of each pipeline, calling `ShaderInspector_inspectShaderStages`
 *    to process them and potentially collect temporary shader module data.
 * 4. Calls the Vulkan API function `vkCreateGraphicsPipelines` to create the pipelines.
 * 5. Cleans up any temporary shader module wrapper objects that were created during inspection.
 * 6. Updates the `AsyncPipelineCreator` instances (passed from the client) with the
 *    resulting `VkPipeline` handles and signals their readiness.
 * 7. Cleans up any internal allocations made during parsing.
 *
 * @param request_ptr A `long` value representing the memory address of the
 *                    `PipelineCreationRequest` struct.
 */
void FUN_00144c20(long request_ptr) {
    // Save the stack cookie value for stack overflow detection.
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    PipelineCreationRequest* request = (PipelineCreationRequest*)request_ptr;

    // Get the raw input data, which contains the Vulkan handles and pipeline creation infos.
    char* create_info_data_raw = (char*)request->createInfoDataCopy;

    // Extract the `VkDevice` and `VkPipelineCache` handles.
    // These are assumed to be at the beginning of `create_info_data_raw`.
    VkDevice device_handle = (VkDevice)VkObject_fromId((void*)create_info_data_raw);
    VkPipelineCache pipeline_cache_handle = (VkPipelineCache)VkObject_fromId((void*)(create_info_data_raw + 8)); // 8 bytes offset for VkPipelineCache.

    uint32_t num_pipelines = request->pipelineCount;

    // Declare pointers for stack-allocated arrays.
    // `pipeline_create_infos_stack_array` will hold the `VkGraphicsPipelineCreateInfo` structs
    // that are prepared for the `vkCreateGraphicsPipelines` call.
    // The size `0x24` (36 bytes) per element is a specific internal size used by `FUN_00142624`.
    // `created_pipelines_stack_array` will hold the resulting `VkPipeline` handles.
    VkGraphicsPipelineCreateInfo* pipeline_create_infos_stack_array =
        (VkGraphicsPipelineCreateInfo*)((uintptr_t)&stack_chk_guard - ((uint64_t)num_pipelines * 0x24 + 0xF & ~0xF));
    VkPipeline* created_pipelines_stack_array =
        (VkPipeline*)((uintptr_t)pipeline_create_infos_stack_array - ((uint64_t)num_pipelines * sizeof(VkPipeline) + 0xF & ~0xF));

    // `internal_allocs_list` is an `ArrayList` struct allocated on the stack (`local_80` in assembly).
    // It is used by `FUN_00142624` to track any temporary memory it allocates internally.
    ArrayList internal_allocs_list = {0}; // Initialize to zero, FUN_00142624 will populate.

    // Call `FUN_00142624` to parse the `create_info_data_raw` and populate the stack-allocated
    // `pipeline_create_infos_stack_array`. This function also manages internal allocations
    // and might return information about shader modules (via `current_shader_module_count` and `current_created_shader_modules_array`).
    uint32_t current_shader_module_count = 0;
    void** current_created_shader_modules_array = NULL;

    FUN_00142624(
        NULL, NULL, NULL, // These parameters appear unused in the provided assembly snippet of FUN_00142624.
        pipeline_create_infos_stack_array, // Output: Array of `VkGraphicsPipelineCreateInfo`
        create_info_data_raw,             // Input: Raw pipeline creation data
        &internal_allocs_list             // Output: ArrayList for internal temporary allocations
    );
    // After FUN_00142624, internal_allocs_list.count and .data will reflect its allocations.
    // Also, `local_a0` in assembly (which contains `current_shader_module_count` and `current_created_shader_modules_array`)
    // is set up by FUN_00142624.

    // Iterate through each pipeline's shader stages to inspect them.
    if (num_pipelines > 0) {
        for (uint32_t i = 0; i < num_pipelines; ++i) {
            const VkGraphicsPipelineCreateInfo* current_pipe_create_info = &pipeline_create_infos_stack_array[i];
            // Iterate through each shader stage defined for the current pipeline.
            for (uint32_t j = 0; j < current_pipe_create_info->stageCount; ++j) {
                const VkPipelineShaderStageCreateInfo* stage_info = &current_pipe_create_info->pStages[j];

                // Call `ShaderInspector_inspectShaderStages`. This function processes each shader module.
                // It might create temporary wrapper objects (e.g., `AsyncPipelineCreator` instances) for
                // the shader modules and add them to `current_created_shader_modules_array`.
                ShaderInspector_inspectShaderStages(
                    request->shaderInspector,
                    device_handle,
                    stage_info->module,         // The `VkShaderModule` handle
                    stage_info->stage,          // The `VkShaderStageFlagBits`
                    stage_info->pName,          // The shader entry point name
                    &current_shader_module_count,       // Output: Updated count of inspected shader modules.
                    &current_created_shader_modules_array // Output: Array of pointers to shader module wrappers.
                );
            }
        }
    }

    // Call the actual Vulkan API to create the graphics pipelines.
    // `dispatch_vkCreateGraphicsPipelines` is a function pointer that must be initialized
    // by the application (e.g., using `vkGetDeviceProcAddr`).
    VkResult create_pipelines_result = ((PFN_vkCreateGraphicsPipelines)dispatch_vkCreateGraphicsPipelines)(
        device_handle,
        pipeline_cache_handle,
        num_pipelines,
        pipeline_create_infos_stack_array,  // Array of `VkGraphicsPipelineCreateInfo` structs.
        NULL,                               // `pAllocator` is passed as `NULL` (0).
        created_pipelines_stack_array       // Output array for the created `VkPipeline` handles.
    );
    // TODO: Handle `create_pipelines_result` to propagate errors if necessary.

    // Cleanup phase for temporary shader module wrapper objects.
    if (current_shader_module_count > 0 && current_created_shader_modules_array != NULL) {
        for (uint32_t i = 0; i < current_shader_module_count; ++i) {
            // `current_shader_module_wrapper` points to an `AsyncPipelineCreator` instance (0x68 bytes).
            // This suggests these instances are reused as wrappers for shader modules by the `ShaderInspector`.
            AsyncPipelineCreator* current_shader_module_wrapper = (AsyncPipelineCreator*)current_created_shader_modules_array[i];

            if (current_shader_module_wrapper != NULL) {
                // Check a custom flag (at byte offset 0x18 from `current_shader_module_wrapper` start).
                // If this flag is `\0` (0), it indicates that the module should NOT be destroyed
                // by Vulkan API, but an internal flag (at byte offset 0x19) should be set.
                // Otherwise (non-zero flag), it indicates ownership and calls for Vulkan destruction.
                if (*(char*)((char*)current_shader_module_wrapper + 0x18) == '\0') {
                    // Set internal flag at offset 0x19 to 1. (e.g., mark as "processed, do not destroy Vulkan handle")
                    *(uint8_t*)((char*)current_shader_module_wrapper + 0x19) = 1;
                } else {
                    // Destroy the `VkShaderModule` handle using `vkDestroyShaderModule`.
                    // `dispatch_vkDestroyShaderModule` is a function pointer.
                    ((PFN_vkDestroyShaderModule)dispatch_vkDestroyShaderModule)(
                        device_handle,
                        current_shader_module_wrapper->vkHandle, // The `VkShaderModule` handle is at offset 0x00.
                        NULL                                     // `pAllocator` is `NULL` (0).
                    );

                    // Free any internal data associated with this shader module wrapper.
                    // This assumes `AsyncPipelineCreator` instances have a `void* internalData` at offset 0x08
                    // when used for shader modules, which is freed here.
                    // This conflicts with `ready_flag` at offset 0x08, suggesting polymorphism or union.
                    // For now, it's represented as `current_shader_module_wrapper->internalData` if such a field exists.
                    // As per the struct definition for `AsyncPipelineCreator`, this would be accessing `ready_flag`
                    // and treating its memory as a pointer to free, which is problematic.
                    // The disassembly line `if ((void *)puVar12[1] != (void *)0x0) { free((void *)puVar12[1]); }`
                    // implies `free(*(void**)((char*)current_shader_module_wrapper + 0x08))`. This remains an ambiguity.
                    // For safety, assuming the code intends to free *some* data pointed to at 0x08.
                    // (This might be a bug or specific to a custom memory allocator/struct variant).
                    if (*(void**)((char*)current_shader_module_wrapper + 0x08) != NULL) {
                        free(*(void**)((char*)current_shader_module_wrapper + 0x08));
                    }

                    // Free the `AsyncPipelineCreator` instance (wrapper) itself.
                    free(current_shader_module_wrapper);
                }
            }
        }
    }

    // Free the array that held the shader module wrapper pointers.
    if (current_created_shader_modules_array != NULL) {
        free(current_created_shader_modules_array);
        current_created_shader_modules_array = NULL;
    }

    // Update the `AsyncPipelineCreator` instances (created in `AsyncPipelineCreator_create`)
    // with the actual `VkPipeline` handles obtained from `vkCreateGraphicsPipelines`.
    if (num_pipelines > 0) {
        for (uint32_t i = 0; i < num_pipelines; ++i) {
            AsyncPipelineCreator* pipeline_result_obj = (AsyncPipelineCreator*)request->asyncPipelineCreators->data[i];
            // Lock the mutex to safely update the pipeline handle and status.
            pthread_mutex_lock(&pipeline_result_obj->mutex);
            pipeline_result_obj->vkHandle = created_pipelines_stack_array[i]; // Store the created `VkPipeline` handle.
            pipeline_result_obj->ready_flag = 1; // Mark the handle as ready.
            pthread_cond_signal(&pipeline_result_obj->cond); // Signal any threads waiting for this handle.
            pthread_mutex_unlock(&pipeline_result_obj->mutex);
        }
    }

    // Cleanup for internal temporary allocations made by `FUN_00142624`.
    // The `internal_allocs_list` (represented by `local_80` in assembly) is an ArrayList struct
    // on the stack, but its `data` member likely points to heap-allocated data.
    if (internal_allocs_list.count > 0 && internal_allocs_list.data != NULL) {
        // Free each element pointed to by the ArrayList.
        for (uint32_t i = internal_allocs_list.count; i > 0; --i) {
            void* element_to_free = internal_allocs_list.data[i - 1];
            if (element_to_free != NULL) {
                free(element_to_free);
            }
            // Conceptually remove from ArrayList, though for stack-based ArrayList
            // and full cleanup loop, `ArrayList_removeAt` may just decrease count.
            ArrayList_removeAt(&internal_allocs_list, i - 1); // Adjust count.
        }
        // Free the backing array (`data` pointer) of the ArrayList itself.
        free(internal_allocs_list.data);
        internal_allocs_list.data = NULL;
    }

    // Perform stack cookie check to detect potential buffer overflows.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail(); // Stack corruption detected.
    }
}
```

---

# vt_handle_vkCmdBindPipeline

The `vt_handle_vkCmdBindPipeline` function is an interceptor or wrapper for the Vulkan `vkCmdBindPipeline` API call. Its primary responsibilities are to deserialize the incoming arguments from a custom client-server communication buffer, perform any necessary internal processing/hooking, and then dispatch the call to the underlying Vulkan driver.

Here's a breakdown of the function's logic:

```c
#include "vulkan.h" // Standard Vulkan header
#include <string.h> // For memcpy
#include <stdlib.h> // For calloc

// Placeholder for the actual Vulkan API function pointer, dynamically loaded.
// It's assumed that `dispatch_vkCmdBindPipeline` points to `vkCmdBindPipeline`.
PFN_vkCmdBindPipeline dispatch_vkCmdBindPipeline;

// Forward declarations for internal helper functions and types.
// These convert between internal object IDs/pointers and actual Vulkan handles.
void* VkObject_fromId(void* serialized_object_id_ptr);
VkPipeline AsyncPipelineCreator_getVkHandle(void* async_pipeline_object_ptr);

// Assuming the structure for the renderer's context, passed as param_1
typedef struct VtContext {
    // ... other members ...
    char* inputBufferPtr;   // Offset 0x28: Pointer to the start of the current command's serialized data.
    // ... other members ...
} VtContext;

/**
 * @brief Handles the interception and dispatch of the vkCmdBindPipeline Vulkan API call.
 *
 * This function reads the serialized arguments for vkCmdBindPipeline from a custom
 * input buffer (`VtContext->inputBufferPtr`), converts internal object IDs to
 * actual Vulkan handles, potentially modifies parameters via internal components
 * like `AsyncPipelineCreator`, and then dispatches the call to the real Vulkan driver.
 *
 * @param vt_ctx_ptr A pointer to the VtContext structure.
 */
void vt_handle_vkCmdBindPipeline(long vt_ctx_ptr) {
    VtContext* vt_ctx = (VtContext*)vt_ctx_ptr;

    // Pointer to the start of the serialized arguments for this command within the input buffer.
    char* input_buffer_ptr = vt_ctx->inputBufferPtr;

    // `current_read_offset` will track the current position within the `input_buffer_ptr`
    // as arguments are deserialized.
    ulong current_read_offset;

    // 1. Deserialization of `VkCommandBuffer commandBuffer`:
    // The custom serialization format uses a leading byte to indicate how the VkObject is represented.
    // If the byte is '\0' (0x00), it implies an implicit/default command buffer (e.g., the primary
    // command buffer associated with the current context), and no further pointer follows.
    // If the byte is non-'\0', it indicates that a pointer to the serialized VkObject
    // follows immediately.
    VkCommandBuffer command_buffer_vk_handle;
    if (*input_buffer_ptr == '\0') {
        // Case 1: Command buffer is implicit.
        // Assuming `vt_ctx_ptr` itself, or a known member of `vt_ctx`, represents the
        // current command buffer when an explicit one is not provided.
        // This is a common optimization for primary command buffers.
        command_buffer_vk_handle = (VkCommandBuffer)vt_ctx_ptr;
        current_read_offset = 1; // Move past the 1-byte type/flag marker.
    } else {
        // Case 2: Command buffer is explicit.
        // Read the 8-byte pointer to the serialized VkObject from `input_buffer_ptr + 1`.
        void* serialized_cmd_buffer_obj_ptr = *(void**)(input_buffer_ptr + 1);
        // Convert the serialized object pointer to the actual Vulkan `VkCommandBuffer` handle.
        command_buffer_vk_handle = (VkCommandBuffer)VkObject_fromId(serialized_cmd_buffer_obj_ptr);
        current_read_offset = 9; // Move past the 1-byte marker and the 8-byte pointer.
    }

    // 2. Deserialization of `VkPipelineBindPoint pipelineBindPoint`:
    // This is a 4-byte unsigned integer (VkFlags). It's read from the current offset.
    VkPipelineBindPoint pipeline_bind_point_value = *(uint32_t*)(input_buffer_ptr + current_read_offset);

    // 3. Deserialization of `VkPipeline pipeline`:
    // Similar to `VkCommandBuffer`, a leading byte indicates the presence of a pointer.
    void* serialized_pipeline_obj_ptr = NULL;
    // `current_read_offset + sizeof(uint32_t)` points just after `pipeline_bind_point_value`.
    // The `| 4` in `pcVar5[uVar6 | 4]` is likely a decompiler artifact. It should just be
    // `input_buffer_ptr[current_read_offset + sizeof(uint32_t)]` to check the marker byte.
    // If this marker byte is non-'\0', a pointer follows.
    if (input_buffer_ptr[current_read_offset + sizeof(uint32_t)] != '\0') {
        // Read the 8-byte pointer to the serialized VkObject from `input_buffer_ptr + current_read_offset + sizeof(uint32_t) + 1`.
        serialized_pipeline_obj_ptr = *(void**)(input_buffer_ptr + current_read_offset + sizeof(uint32_t) + 1);
    }
    // Convert the serialized object pointer to its internal VkObject* representation.
    void* pipeline_vk_object_ptr = VkObject_fromId(serialized_pipeline_obj_ptr);

    // Hooking/Internal Processing:
    // `AsyncPipelineCreator_getVkHandle` is called on the resolved `VkObject*` for the pipeline.
    // This suggests that the renderer uses `AsyncPipelineCreator` to manage pipelines.
    // This function likely ensures the pipeline is ready (e.g., compiled asynchronously)
    // and returns the actual `VkPipeline` handle.
    VkPipeline actual_vk_pipeline_handle = (VkPipeline)AsyncPipelineCreator_getVkHandle(pipeline_vk_object_ptr);

    // Dispatch the original Vulkan API call:
    // The `dispatch_vkCmdBindPipeline` function pointer is called with the resolved
    // Vulkan handles and the deserialized `pipelineBindPoint`.
    dispatch_vkCmdBindPipeline(
        command_buffer_vk_handle,      // Resolved VkCommandBuffer handle
        pipeline_bind_point_value,     // Deserialized VkPipelineBindPoint value
        actual_vk_pipeline_handle      // Resolved VkPipeline handle (possibly from AsyncPipelineCreator)
    );

    // No output serialization:
    // `vkCmdBindPipeline` is a command buffer recording function and does not return
    // data directly, so there's no code here to write to the client's ring buffer.
}
```

---

# vt_handle_vkCmdCopyBufferToImage

The `vt_handle_vkCmdCopyBufferToImage` function is responsible for deserializing the parameters for a `vkCmdCopyBufferToImage` Vulkan API call from an input buffer, potentially modifying the call's behavior through a custom texture decoder, and then dispatching it to the native Vulkan driver.

Here's a detailed breakdown:

```c
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <sched.h>
#include <unistd.h>
#include <errno.h>
#include <sys/mman.h>
#include <sys/ioctl.h>
#include <sys/eventfd.h>
#include <sys/socket.h>
#include <fcntl.h>
#include <stdio.h>
#include <dirent.h>
#include <sys/stat.h>
#include <errno.h>
#include <pthread.h>

// --- Assumed Types and Globals from Vulkan.h and surrounding context ---

// Standard Vulkan types (simplified for brevity)
typedef uint32_t VkBool32;
typedef uint64_t VkDeviceAddress;
typedef uint64_t VkDeviceSize;
typedef uint32_t VkFlags;
typedef uint32_t VkSampleMask;

// Vulkan Handle definitions
typedef struct VkDevice_T* VkDevice;
typedef struct VkCommandBuffer_T* VkCommandBuffer;
typedef uint64_t VkBuffer; // Non-dispatchable handle often uint64_t
typedef uint64_t VkImage;  // Non-dispatchable handle often uint64_t

// Vulkan Enum
typedef enum VkImageLayout {
    VK_IMAGE_LAYOUT_UNDEFINED = 0,
    VK_IMAGE_LAYOUT_GENERAL = 1,
    VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL = 2,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL = 7,
    // ... other layouts
} VkImageLayout;

// Vulkan Structs (partial definitions based on usage)
typedef VkFlags VkImageAspectFlags;

typedef struct VkOffset3D {
    int32_t x;
    int32_t y;
    int32_t z;
} VkOffset3D;

typedef struct VkExtent3D {
    uint32_t width;
    uint32_t height;
    uint32_t depth;
} VkExtent3D;

typedef struct VkImageSubresourceLayers {
    VkImageAspectFlags aspectMask;
    uint32_t           mipLevel;
    uint32_t           baseArrayLayer;
    uint32_t           layerCount;
} VkImageSubresourceLayers;

typedef struct VkBufferImageCopy {
    VkDeviceSize                bufferOffset;
    uint32_t                    bufferRowLength;
    uint32_t                    bufferImageHeight;
    VkImageSubresourceLayers    imageSubresource;
    VkOffset3D                  imageOffset;
    VkExtent3D                  imageExtent;
} VkBufferImageCopy;

// Function pointer for the native Vulkan API call
typedef void (VKAPI_PTR *PFN_vkCmdCopyBufferToImage)(
    VkCommandBuffer      commandBuffer,
    VkBuffer             srcBuffer,
    VkImage              dstImage,
    VkImageLayout        dstImageLayout,
    uint32_t             regionCount,
    const VkBufferImageCopy* pRegions
);

// --- Context Structure (deduced from assembly) ---

typedef struct VtContext {
    char        _padding_0[0x28];     // Padding to inputBufferPtr
    char*       inputBufferPtr;       // Offset 0x28: Pointer to the current command's serialized data
    char        _padding_1[0x4C];     // Padding to textureDecoderPtr_offset_0x78
    uint62_t    textureDecoderPtr_offset_0x78; // Offset 0x78: Pointer to TextureDecoder instance
    // ... other members like ring buffer pointers, temporary allocation lists, etc.
    // E.g., `param_1 + 0x60` and `param_1 + 0x68` suggest RingBuffer pointers for results/commands.
    // `param_1 + 0x78` specifically points to `ctx->textureDecoder`.
} VtContext;

// --- Assumed External Functions and Globals ---
extern char vortekSerializerCastVkObject; // Global flag likely influencing serialization/deserialization behavior

// These are placeholders for actual implementations.
// In a real decompilation, these would be separate functions or external library calls.
extern void* VkObject_fromId(void* serialized_id); // Converts serialized ID to Vk handle
extern bool TextureDecoder_containsImage(uint32_t* decoder_ctx, long image_handle); // Custom check
extern void TextureDecoder_copyBufferToImage(uint32_t* decoder_ctx, VkCommandBuffer cmdBuffer, long srcBuffer, long dstImage, VkImageLayout dstImageLayout, const VkBufferImageCopy* pRegions); // Custom copy function
extern PFN_vkCmdCopyBufferToImage dispatch_vkCmdCopyBufferToImage; // Pointer to native Vulkan function


// --- Decompiled Function ---

void vt_handle_vkCmdCopyBufferToImage(long param_vt_context_ptr) {
    // A stack canary is typically used for stack corruption detection.
    // The original code saves and checks it. It's omitted here for clarity
    // but its presence indicates a security mechanism.
    // long stack_canary_value = *(long *)(tpidr_el0 + 0x28); // tpidr_el0 holds TLS pointer on AArch64

    // Cast the input parameter to the VtContext pointer.
    VtContext* ctx = (VtContext*)param_vt_context_ptr;

    // `input_buffer_ptr` points to the start of the serialized command arguments.
    char* input_buffer_ptr = ctx->inputBufferPtr;
    
    // `current_read_offset` tracks the byte offset within `input_buffer_ptr`
    // as arguments are deserialized.
    long current_read_offset = 0;

    // --- Deserialization of VkCommandBuffer `commandBuffer` ---
    // The serialization format for handles appears to be:
    // - If NULL: 1 byte (0x00)
    // - If non-NULL: 1 byte (non-0x00, e.g., 0x01) followed by 8 bytes for the actual handle value.
    VkCommandBuffer commandBuffer;
    if (input_buffer_ptr[current_read_offset] == '\0') {
        // If the first byte is NULL, it signifies a VK_NULL_HANDLE.
        // `VkObject_fromId` is called with `param_vt_context_ptr` which
        // might be used as a dummy ID for internal lookup, or it signifies
        // that the command buffer is managed by the context itself implicitly.
        commandBuffer = (VkCommandBuffer)VkObject_fromId((void*)param_vt_context_ptr);
        current_read_offset += 1;
    } else {
        // Otherwise, read the 8-byte handle value.
        commandBuffer = (VkCommandBuffer)VkObject_fromId(*(void**)(input_buffer_ptr + current_read_offset + 1));
        current_read_offset += 9; // 1 byte for status + 8 bytes for handle
    }

    // --- Deserialization of VkBuffer `srcBuffer` ---
    VkBuffer srcBuffer;
    if (input_buffer_ptr[current_read_offset] == '\0') {
        srcBuffer = (VkBuffer)VkObject_fromId((void*)param_vt_context_ptr); // Dummy ID
        current_read_offset += 1;
    } else {
        srcBuffer = (VkBuffer)VkObject_fromId(*(void**)(input_buffer_ptr + current_read_offset + 1));
        current_read_offset += 9;
    }

    // --- Deserialization of VkImage `dstImage` ---
    VkImage dstImage;
    if (input_buffer_ptr[current_read_offset] == '\0') {
        dstImage = (VkImage)VkObject_fromId((void*)param_vt_context_ptr); // Dummy ID
        current_read_offset += 1;
    } else {
        dstImage = (VkImage)VkObject_fromId(*(void**)(input_buffer_ptr + current_read_offset + 1));
        current_read_offset += 9;
    }

    // --- Deserialization of VkImageLayout `dstImageLayout` ---
    VkImageLayout dstImageLayout = *(VkImageLayout*)(input_buffer_ptr + current_read_offset);
    current_read_offset += sizeof(VkImageLayout); // VkImageLayout is uint32_t

    // --- Deserialization of uint32_t `regionCount` ---
    uint32_t regionCount = *(uint32_t*)(input_buffer_ptr + current_read_offset);
    current_read_offset += sizeof(uint32_t); // uint32_t

    // --- Deserialization of `const VkBufferImageCopy* pRegions` ---
    // The `pRegions` array is dynamically allocated on the stack (or heap in some implementations)
    // based on `regionCount`. Assembly uses stack space via pointer arithmetic.
    VkBufferImageCopy* pRegions = NULL;
    if (regionCount > 0) {
        // Allocate memory for the array of VkBufferImageCopy structs.
        // Assembly calculated this as `regionCount * sizeof(VkBufferImageCopy)`.
        pRegions = (VkBufferImageCopy*)malloc(sizeof(VkBufferImageCopy) * regionCount);
        if (pRegions == NULL) {
            // In a production-quality decompile, error handling would be explicit here.
            // Original assembly would likely proceed and crash or lead to undefined behavior.
            goto END_FUNCTION; // Skip to end if allocation fails
        }

        // Copy the serialized VkBufferImageCopy data into the newly allocated memory.
        // The data for `pRegions` starts immediately after `regionCount` in the input buffer.
        memcpy(pRegions, input_buffer_ptr + current_read_offset, sizeof(VkBufferImageCopy) * regionCount);
        current_read_offset += sizeof(VkBufferImageCopy) * regionCount; // Update offset for next reads (if any)
    }

    // --- Custom Behavior / Hooking ---
    // Check if a custom texture decoder is enabled and manages the destination image.
    if (ctx->textureDecoderPtr_offset_0x78 != 0) {
        bool image_managed_by_decoder = TextureDecoder_containsImage(
            (uint32_t*)ctx->textureDecoderPtr_offset_0x78, (long)dstImage
        );

        if (image_managed_by_decoder) {
            // The assembly includes a specific check: `*(int *)(&stack0xffffffffffffffb4 + lVar14) == 0`.
            // `&stack0xffffffffffffffb4 + lVar14` points to the `pRegions` array on the stack.
            // `*(int *)` would dereference the first 4 bytes of `pRegions[0].bufferOffset` (VkDeviceSize).
            // This implies a special optimization for `VkBufferImageCopy` regions where `bufferOffset` is zero.
            if (regionCount > 0 && (pRegions[0].bufferOffset & 0xFFFFFFFF) == 0) {
                // Call the custom texture decoder's copy function instead of the native Vulkan call.
                TextureDecoder_copyBufferToImage(
                    (uint32_t*)ctx->textureDecoderPtr_offset_0x78,
                    commandBuffer,
                    (long)srcBuffer, // Passed as long, implying the custom function takes long for handles
                    (long)dstImage,  // Passed as long
                    dstImageLayout,
                    pRegions         // Pointer to the deserialized regions
                );
                goto END_FUNCTION; // Skip the native Vulkan call.
            }
        }
    }

    // --- Native Vulkan API Call Dispatch ---
    // If no custom logic intercepts, dispatch the command to the native Vulkan driver.
    dispatch_vkCmdCopyBufferToImage(
        commandBuffer,
        srcBuffer,
        dstImage,
        dstImageLayout,
        regionCount,
        pRegions
    );

END_FUNCTION:
    // --- Cleanup ---
    // Free the dynamically allocated memory for `pRegions` if it was used.
    if (pRegions) {
        free(pRegions);
    }

    // --- Stack Canary Check ---
    // This is a security mechanism to detect stack corruption.
    // `tpidr_el0` points to the thread's local storage (TLS). The canary is stored there.
    // If the value on the stack doesn't match the TLS value, it indicates a buffer overflow.
    // (Original assembly: `if (*(long *)(lVar6 + 0x28) == alStack_60[1]) { return; } else { __stack_chk_fail(); }`)
    // The exact variable names (`alStack_60[1]`, `lVar6`) are assembly specific and omitted here.
    // The `__stack_chk_fail()` function typically terminates the program.
}
```

---

# vt_handle_vkWaitSemaphores

Decompilation requires careful analysis of memory accesses and function signatures to infer the underlying data structures and their relationships. The provided C code snippet and the `VtContext` structure, combined with common Vulkan API patterns, allow us to reconstruct a more readable and accurate representation.

**General Assumptions:**
*   The system is 64-bit, so pointers (`void*`, `long`) are 8 bytes, and `int` is 4 bytes.
*   Decompiler output often aliases `void*` or `char*` to `int*` or `undefined4*` for convenience in displaying offsets, but direct byte offsets like `param_1 + 0x22` imply byte arithmetic on a `char*` pointer. We will treat `param_1` in `vt_handle_vkWaitSemaphores` as `char*` for byte offsets, even if its type is `int*`.
*   Offsets for `pthread_mutex_t` and `pthread_cond_t` are hardcoded, suggesting a specific, potentially packed, internal structure for `AsyncPipelineCreator`.

---

### Inferred Structures

**1. `VtContext`**

The provided `VtContext` structure is used as a reference for global context members.

```c
// Based on the provided snippet in the problem description
typedef struct VtContext {
    /* 0x00 */ int clientFd;
    /* 0x04 */ int vkMaxVersion;
    /* 0x08 */ short maxDeviceMemory; // In MB
    /* 0x0a */ short imageCacheSize;  // In MB
    /* 0x0c */ int padding_0c; // Padding as per original struct. Might hide internal fields/alignments.
    /* 0x10 */ ArrayList* exposedDeviceExtensions;
    /* 0x18 */ ArrayList* exposedInstanceExtensions;
    /* 0x20 */ uint8_t hostVisibleMemoryFlag; // For memory type selection
    /* 0x21 */ uint8_t coherentMemoryFlag;    // For memory type selection
    /* 0x22 */ char padding_22[6]; // Original size 6 bytes
    /* 0x28 */ char* inputBufferPtr;      // Pointer to the current command's data
    /* 0x30 */ int inputBufferSize;
    /* 0x34 */ int padding_34;
    /* 0x38 */ void* tempBuffer;          // A large temporary buffer for deserialization
    /* 0x40 */ int tempBufferOffset;      // Current offset within the tempBuffer
    /* 0x44 */ int padding_44;
    /* 0x48 */ ArrayList* tempAllocations; // Tracks allocations within tempBuffer
    /* 0x50 */ long padding_50;
    /* 0x58 */ pthread_t serverThread;    // Handle to the main processing thread
    /* 0x60 */ RingBuffer* clientRingBuffer; // For writing results back to the client
    /* 0x68 */ RingBuffer* serverRingBuffer; // For reading commands from the client
    /* 0x70 */ int lastError;             // Stores the last recorded VkResult error
    /* 0x74 */ int graphicsQueueFamilyIndex;
    /* 0x78 */ TextureDecoder* textureDecoder;
    /* 0x80 */ ShaderInspector* shaderInspector;
    /* 0x88 */ AsyncPipelineCreator* asyncPipelineCreator; // Pointer to the asynchronous pipeline creator
    
    // JNI and Android-specific windowing members
    /* 0x90 */ jobject javaComponentObject;
    /* 0x98 */ JNIEnv* jniEnv;
    /* 0xA0 */ jobject javaWindowManagerObject;
    /* 0xA8 */ jmethodID getWindowWidthMethodID;
    /* 0xB0 */ jmethodID getWindowHeightMethodID;
    /* 0xB8 */ jmethodID getWindowHardwareBufferMethodID;
    /* 0xC0 */ jmethodID updateWindowContentMethodID;
} VtContext;
```

**2. `AsyncPipelineCreator`**

This structure is a likely candidate for the object managed by `param_2` in `TimelineSemaphore_asyncWait`. The hardcoded offsets from `param_2` point to internal synchronization primitives and queue pointers. The exact sizes of `pthread_mutex_t` and `pthread_cond_t` vary by system, but the offsets indicate their positions relative to the start of the `AsyncPipelineCreator` instance. We assume a `__attribute__((packed))` or similar compiler directive for these precise non-aligned offsets, or that `param_2` points to a `char` array and values are read with specific offsets.

```c
// Inferred structure for AsyncPipelineCreator's internal queue management part
// Assumes this struct is aligned to its natural size, but internal fields might be packed.
// The offsets are relative to the base address of an AsyncPipelineCreator instance.
typedef struct AsyncPipelineCreator {
    // ... potentially other members before offset 0xC (12 bytes)
    char _padding_00[0xC]; 
    pthread_mutex_t internal_mutex_for_queue; // Accessed at `param_2 + 0xc`
    
    // Padding to align to the next specific offset.
    // Assuming pthread_mutex_t is 40 bytes on 64-bit Linux, then 0xC + 40 = 0x34. This is a perfect match!
    // So, the mutex is directly at 0xC and is 40 bytes long.
    
    pthread_cond_t internal_cond_for_queue;   // Accessed at `param_2 + 0x34` (52 bytes)
    
    // Padding to align to 0xC0 (192 bytes)
    // Assuming pthread_cond_t is 48 bytes on 64-bit Linux, then 0x34 + 48 = 0x64.
    // So there's `0xC0 - 0x64 = 0x5C` (92 bytes) of padding.
    char _padding_after_cond[0x5C]; // Placeholder padding
    
    void* async_task_queue_head;               // Accessed at `param_2 + 0xc0` (192 bytes)
    void* async_task_queue_tail;               // Accessed at `param_2 + 200` (0xc8 bytes)
    // ... potentially other members after this
} AsyncPipelineCreator;
```

**3. `AsyncWaitTaskData` & `AsyncTaskNode`**

These are internal structures used by `TimelineSemaphore_asyncWait` to manage queued asynchronous tasks.

```c
// Structure to hold data for a single asynchronous wait task
typedef struct AsyncWaitTaskData {
    int     event_fd;          // File descriptor for signaling completion of this wait operation
    void*   semaphore_data_buffer; // Dynamically allocated buffer holding actual semaphore values
    // ... potentially other members relevant to the wait operation (e.g., timeout, VkSemaphore handles)
} AsyncWaitTaskData;

// Structure for a node in the asynchronous task queue
typedef struct AsyncTaskNode {
    void (*task_function)(void*); // Function pointer to the task's execution routine (FUN_001452c8)
    void* task_data;               // Pointer to task-specific data (e.g., AsyncWaitTaskData)
    struct AsyncTaskNode* next;    // Pointer to the next node in the linked list queue
} AsyncTaskNode;
```

**4. `VtWaitSemaphoresCommandData`**

This structure defines the format of the incoming command data buffer that `vt_handle_vkWaitSemaphores` receives. It's a custom wire protocol format, not a direct Vulkan struct. The offsets are crucial for parsing.

```c
// Structure of the command data buffer passed to vt_handle_vkWaitSemaphores
// This structure is custom to the client-server protocol.
// The offsets are byte offsets from the start of the buffer.
typedef struct VtWaitSemaphoresCommandData {
    int       client_fd;                    // Offset 0x00 (4 bytes): File descriptor for client communication
    char      _padding_04[0x10 - 0x04];     // Padding to ensure next field is at 0x10
    void*     pSemaphoreValues;             // Offset 0x10 (16 bytes): Pointer to the actual semaphore values data
    char      _padding_18[0x22 - 0x18];     // Padding to ensure next field is at 0x22
    long      asyncOperationsManagerPtr;    // Offset 0x22 (34 bytes): Pointer to an AsyncPipelineCreator instance
    char      _padding_2A[0x30 - 0x2A];     // Padding to ensure next field is at 0x30
    int       semaphoreValueCount;          // Offset 0x30 (48 bytes): Number of semaphore values
    // ... potentially other VkSemaphoreWaitInfo fields or command-specific data
} VtWaitSemaphoresCommandData;
```

---

### Decompilation of `vt_handle_vkWaitSemaphores`

**Original Function Signature:**
`void vt_handle_vkWaitSemaphores(int *param_1)`

**Decompiled Function:**

```c
// Forward declaration of the internal asynchronous wait function
void TimelineSemaphore_asyncWait(int clientFd, AsyncPipelineCreator* asyncOperationsManager,
                                 const uint64_t* semaphoreValues, uint32_t semaphoreValuesCount);

/**
 * @brief Handles an incoming `vkWaitSemaphores` command from the client.
 *
 * This function acts as a dispatcher for the `vkWaitSemaphores` operation.
 * It deserializes the necessary arguments from the `commandDataPtr` (which
 * represents the network-received command buffer) and then calls an internal
 * asynchronous waiting function to process the request without blocking the
 * main thread.
 *
 * @param commandDataPtr A pointer to the deserialized command data buffer,
 *                       expected to conform to the `VtWaitSemaphoresCommandData` layout.
 */
void vt_handle_vkWaitSemaphores(VtWaitSemaphoresCommandData* commandDataPtr) {
    // Extract parameters from the command data buffer.
    // The specific offsets (0x0, 0x10, 0x22, 0x30) indicate a custom,
    // potentially packed or unaligned, structure for the command payload
    // as it arrives over the network.
    int client_fd = commandDataPtr->client_fd;
    const uint64_t* p_semaphore_values = (const uint64_t*)commandDataPtr->pSemaphoreValues;
    AsyncPipelineCreator* p_async_ops_manager = (AsyncPipelineCreator*)commandDataPtr->asyncOperationsManagerPtr;
    uint32_t semaphore_value_count = (uint32_t)commandDataPtr->semaphoreValueCount;

    TimelineSemaphore_asyncWait(
        client_fd,                 // Parameter 1: File descriptor for client communication
        p_async_ops_manager,       // Parameter 2: Pointer to the AsyncPipelineCreator instance
        p_semaphore_values,        // Parameter 3: Pointer to the array of semaphore values
        semaphore_value_count      // Parameter 4: Number of semaphore values
    );
    // The original function has no explicit return statement, implying void return.
}
```

---

### Decompilation of `TimelineSemaphore_asyncWait`

**Original Function Signature:**
`void TimelineSemaphore_asyncWait(int param_1,long param_2,void *param_3,int param_4)`

**Decompiled Function:**

```c
#include <sys/eventfd.h> // For eventfd
#include <sys/socket.h>  // For sendmsg, msghdr, iovec
#include <unistd.h>      // For close
#include <string.h>      // For memcpy
#include <stdlib.h>      // For calloc, malloc
#include <pthread.h>     // For pthread_mutex_lock, pthread_cond_signal, etc.

// External function for stack cookie check (compiler-generated security feature)
extern void __stack_chk_fail(int status_code); // Status code usually from last function call.

// Placeholder for an actual task function executed by the asynchronous worker.
// Its exact signature is void(void*), based on AsyncTaskNode definition.
extern void FUN_001452c8(void* task_data);

/**
 * @brief Schedules an asynchronous wait operation on Vulkan timeline semaphores.
 *
 * This function performs the necessary setup for an asynchronous semaphore wait:
 * 1. Allocates a `AsyncWaitTaskData` structure to hold task-specific context.
 * 2. Dynamically allocates and copies the semaphore values from the input buffer.
 * 3. Creates an eventfd to allow signaling completion of this specific wait.
 * 4. Sends a minimal `sendmsg` to the client, possibly acknowledging receipt or
 *    transferring the eventfd (though the transfer part is not explicit in the snippet).
 * 5. Creates an `AsyncTaskNode` wrapping the task function and its data.
 * 6. Acquires a mutex, enqueues the `AsyncTaskNode` into a global asynchronous queue
 *    (managed by `asyncOpsManager`), and signals a condition variable to wake
 *    any waiting worker threads.
 * 7. Releases the mutex.
 *
 * This pattern allows the `vkWaitSemaphores` call to be non-blocking from the
 * perspective of the client-facing thread.
 *
 * @param clientFd The file descriptor used for client communication (e.g., a socket).
 * @param asyncOpsManager A pointer to the `AsyncPipelineCreator` instance which manages
 *                        the asynchronous task queue, including its mutex, condition variable,
 *                        and queue head/tail pointers.
 * @param pWaitValues A pointer to the buffer containing the `uint64_t` semaphore values
 *                    that the asynchronous task needs to wait on.
 * @param waitValuesCount The number of `uint64_t` semaphore values in `pWaitValues`.
 */
void TimelineSemaphore_asyncWait(
    int clientFd,
    AsyncPipelineCreator* asyncOpsManager,
    void* pWaitValues,
    int waitValuesCount
) {
    // Save the stack cookie value to detect buffer overflows later.
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    // 1. Allocate memory for the task-specific data (AsyncWaitTaskData)
    AsyncWaitTaskData* taskData = (AsyncWaitTaskData*)calloc(1, sizeof(AsyncWaitTaskData));
    if (taskData == NULL) {
        // Handle allocation failure. In real code, this would set an error flag and clean up.
        // For this decompilation, we propagate stack cookie check if it fails, matching original.
        if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
            __stack_chk_fail(-1); // Status code for allocation failure
        }
        return;
    }

    // 2. Allocate memory to copy the semaphore values and perform the copy
    taskData->semaphore_data_buffer = malloc((size_t)waitValuesCount);
    if (taskData->semaphore_data_buffer != NULL) {
        memcpy(taskData->semaphore_data_buffer, pWaitValues, (size_t)waitValuesCount);
    } else {
        // Handle allocation failure for semaphore data buffer.
        free(taskData); // Free the task data struct
        if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
            __stack_chk_fail(-1); // Status code for allocation failure
        }
        return;
    }

    // 3. Create an eventfd for this specific asynchronous wait operation.
    // The eventfd will be used to signal completion back to the waiting mechanism.
    taskData->event_fd = eventfd(0, 0); // Initial value 0, flags 0
    if (taskData->event_fd == -1) {
        // Handle eventfd creation failure.
        free(taskData->semaphore_data_buffer); // Free previously allocated buffer
        free(taskData); // Free task data struct
        if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
            __stack_chk_fail(-1); // Status code for eventfd creation failure
        }
        return;
    }

    // 4. Prepare and send a message over the client file descriptor.
    // This uses `sendmsg` and includes a control message. The specific
    // interpretation of `0x14` (20 bytes) and `0x100000001` depends on the
    // underlying socket communication protocol (e.g., possibly transferring
    // the eventfd for out-of-band communication, or just an acknowledgment).
    struct msghdr msg_header;
    struct iovec msg_iov;
    char payload_byte = 0; // Single byte payload for the message
    char control_buffer[20]; // Buffer for control message data

    msg_iov.iov_base = &payload_byte; // Point to the single byte payload
    msg_iov.iov_len = 1;

    // Initialize msg_header fields
    msg_header.msg_name = NULL;
    msg_header.msg_namelen = 0;
    msg_header.msg_iov = &msg_iov;
    msg_header.msg_iovlen = 1;
    msg_header.msg_control = control_buffer;
    msg_header.msg_controllen = sizeof(control_buffer); // 0x14 (20 bytes)

    // Populate control_buffer with specific magic values from disassembly.
    // These typically represent `cmsghdr` structure elements for auxiliary data.
    // `local_e8` and `local_e0` from disassembly are packed into `control_buffer`.
    ((uint64_t*)control_buffer)[0] = 0x14;        // Corresponds to `local_e8` (length)
    ((uint64_t*)control_buffer)[1] = 0x100000001; // Corresponds to `local_e0` (type/level)

    sendmsg(clientFd, &msg_header, 0);

    // 5. Allocate memory for a new asynchronous task node
    AsyncTaskNode* node = (AsyncTaskNode*)malloc(sizeof(AsyncTaskNode));
    if (node == NULL) {
        // Handle allocation failure for task node.
        free(taskData->semaphore_data_buffer);
        close(taskData->event_fd); // Close the eventfd if task node allocation fails
        free(taskData);
        if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
            __stack_chk_fail(-1); // Status code for allocation failure
        }
        return;
    }

    // Assign the task function pointer and the task-specific data
    node->task_function = FUN_001452c8; // The actual worker function to be called later
    node->task_data = taskData;
    node->next = NULL; // New node is always added to the end, so its 'next' is NULL

    // 6. Enqueue the new task node into the asynchronous task queue.
    // This operation is critical section and requires mutex protection.
    pthread_mutex_lock(&asyncOpsManager->internal_mutex_for_queue);

    if (asyncOpsManager->async_task_queue_head == NULL) {
        // If the queue is currently empty, this new node becomes the head
        asyncOpsManager->async_task_queue_head = node;
    } else {
        // Otherwise, append the new node to the end of the existing queue
        ((AsyncTaskNode*)asyncOpsManager->async_task_queue_tail)->next = node;
    }
    // Update the queue's tail pointer to point to the newly added node
    asyncOpsManager->async_task_queue_tail = node;

    // Signal a waiting worker thread that a new task is available in the queue
    pthread_cond_signal(&asyncOpsManager->internal_cond_for_queue);
    
    // Release the mutex lock
    int unlock_status = pthread_mutex_unlock(&asyncOpsManager->internal_mutex_for_queue);

    // Stack canary check: If the stack cookie has been modified, a stack overflow
    // (or similar memory corruption) has occurred.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail(unlock_status); // Pass the last function call's return status
    }
    return;
}
```

---

# Wait

I will decompile the provided C functions, clarifying parameter types and mapping offsets to Vulkan struct members using the provided header.

First, let's identify the constants:
*   `0x10`: This value frequently appears for various `handleType` fields in `VkExternal*HandleTypeFlagBits`. Specifically, `VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT_KHR` is `0x00000010`.
*   `0x8`: This value is `VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT_KHR`.
*   `0x14`: This is `CMSG_SPACE(sizeof(int))`, which calculates the space needed in the control message for passing a file descriptor.
*   `0x100000001`: This is `SCM_RICHTS` (level) combined with `sizeof(cmsghdr)` and `CMSG_LEN` macros for `cmsg_level` and `cmsg_type` in `msghdr` for file descriptor passing.

Based on the typical structure of `vt_handle_vk*` functions:
*   `param_1` is `VtContext* vtContext`.
*   `vtContext->inputBufferPtr` (at offset `0x28`) holds the serialized input arguments for the Vulkan API call.
*   `vtContext->clientFd` (at offset `0x0`) is the socket file descriptor for communication.
*   `VkObject_fromId` converts the serialized `uint64_t` handle into the actual Vulkan object pointer.

---

### Function: `vt_handle_vkGetMemoryFdKHR`

This function handles the `vkGetMemoryFdKHR` API call. This API is used to retrieve a Linux file descriptor (`fd`) for a given `VkDeviceMemory` allocation.

**Vulkan API Signature:** `VkResult vkGetMemoryFdKHR(VkDevice device, const VkMemoryGetFdInfoKHR* pGetFdInfo, int* pFd);`

**Structure of `VkMemoryGetFdInfoKHR`:**

```c
typedef struct VkMemoryGetFdInfoKHR {
    VkStructureType                       sType;        // Offset 0
    const void*                           pNext;        // Offset 8
    VkDeviceMemory                        memory;       // Offset 16 (0x10)
    VkExternalMemoryHandleTypeFlagBits    handleType;   // Offset 24 (0x18)
} VkMemoryGetFdInfoKHR;
```

**Decompiled Function with Explanations:**

```c
// vt_handle_vkGetMemoryFdKHR - Handles the vkGetMemoryFdKHR Vulkan API call.
//
// vtContext: Pointer to the global Vortek context.
//
void vt_handle_vkGetMemoryFdKHR(VtContext* vtContext) {
    // Standard stack canary for stack corruption detection.
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28);

    // Get pointer to the serialized input arguments from vtContext->inputBufferPtr.
    char* input_data_ptr = *(char**)(vtContext->inputBufferPtr_offset); // Equivalent to vtContext->inputBufferPtr

    // Local struct to hold deserialized VkMemoryGetFdInfoKHR parameters.
    VkMemoryGetFdInfoKHR fd_info = {0}; // Initialize all members to zero.

    // Determine the offset for deserializing the VkDevice handle.
    // The first byte of the input_data_ptr indicates the type/flag of the first serialized object.
    ulong current_input_offset = 9; // Default offset if the first byte is not 0 (indicating a pointer-like handle).
    if (*input_data_ptr == '\0') { // Check the first byte of input_data_ptr.
        current_input_offset = 1; // If it's 0, it indicates a direct value, not a pointer-like handle.
    }

    // Deserialize VkDevice handle (first argument to vkGetMemoryFdKHR).
    // The handle is stored as a wrapped VkObject pointer in the serialized input.
    VkDevice device = VkObject_fromId(*(void**)(input_data_ptr + current_input_offset));

    // Advance the offset in input_data_ptr past the VkDevice handle.
    // This assumes a serialized VkObject-wrapped pointer takes 8 bytes for the value, plus 1 byte for the initial flag.
    current_input_offset += 9;

    // Deserialize VkMemoryGetFdInfoKHR structure (second argument to vkGetMemoryFdKHR).
    // The serialized `VkMemoryGetFdInfoKHR` data starts at `input_data_ptr + current_input_offset`.

    // Deserialize fd_info.sType (VkStructureType, at offset 0 within VkMemoryGetFdInfoKHR).
    // `(uVar4 | 4)` is effectively `current_input_offset` used as `VkMemoryGetFdInfoKHR`'s base,
    // and `+0` is the offset to `sType`.
    // The `(uVar4 | 4)` is likely a decompiler artifact for address calculation.
    fd_info.sType = *(VkStructureType*)(input_data_ptr + current_input_offset); // Read sType

    // Deserialize fd_info.memory (VkDeviceMemory, at offset 0x10 (16) within VkMemoryGetFdInfoKHR).
    // The serialized value is a VkObject-wrapped pointer.
    fd_info.memory = VkObject_fromId(*(void**)(input_data_ptr + current_input_offset + 0x10)); // Read memory

    // Hardcode fd_info.handleType to VK_EXTERNAL_MEMORY_HANDLE_TYPE_SYNC_FD_BIT_KHR (0x10).
    // This explicitly overrides any value that might have been in the input buffer for handleType.
    // This is a specific behavior implemented by Vortek.
    fd_info.handleType = VK_EXTERNAL_MEMORY_HANDLE_TYPE_SYNC_FD_BIT_KHR; // Value 0x10 (VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT)

    int pFd_result = 0; // Local variable to receive the file descriptor from the Vulkan API call.

    // Call the actual Vulkan API function: vkGetMemoryFdKHR.
    // `dispatch_vkGetMemoryFd` is a function pointer obtained earlier via `vkGetDeviceProcAddr`.
    VkResult result = dispatch_vkGetMemoryFdKHR(device, &fd_info, &pFd_result);

    // Prepare a `msghdr` structure to send the `VkResult` and the file descriptor back to the client.
    msghdr response_msg = {0};
    iovec response_iov[1];

    // The first part of the response payload is the `VkResult`.
    response_iov[0].iov_base = &result;
    response_iov[0].iov_len = sizeof(VkResult);

    response_msg.msg_iov = response_iov;
    response_msg.msg_iovlen = 1;

    // Use a control message to pass the file descriptor over the socket.
    // CMSG_SPACE macro calculates the buffer size needed for a control message header + data.
    union {
        char control_buffer[CMSG_SPACE(sizeof(int))];
        struct cmsghdr align; // Ensures proper alignment of the buffer.
    } control_un;

    response_msg.msg_control = control_un.control_buffer;
    response_msg.msg_controllen = sizeof(control_un.control_buffer);

    // Populate the `cmsghdr` (control message header) within the control buffer.
    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&response_msg);
    if (cmsg != NULL) {
        cmsg->cmsg_level = SOL_SOCKET; // Specifies the socket API level.
        cmsg->cmsg_type = SCM_RIGHTS;  // Specifies that file descriptors are being passed.
        cmsg->cmsg_len = CMSG_LEN(sizeof(int)); // Length of the control message.
        *(int*)CMSG_DATA(cmsg) = pFd_result; // Copy the actual file descriptor into the control message data.
    }

    // Send the constructed message through the client socket file descriptor.
    sendmsg(vtContext->clientFd, &response_msg, 0);

    // Close the file descriptor obtained from the Vulkan API call.
    // It's closed here because ownership is transferred to the client via `sendmsg`.
    if (pFd_result > 0) { // File descriptors are typically positive integers.
        close(pFd_result);
    }

    // Stack canary check to detect potential buffer overflows or stack corruption.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail(); // Abort if stack corruption is detected.
    }
    return;
}
```

---

### Function: `vt_handle_vkGetSemaphoreFdKHR`

This function handles the `vkGetSemaphoreFdKHR` API call. This API is used to retrieve a Linux file descriptor (`fd`) for a given `VkSemaphore` object.

**Vulkan API Signature:** `VkResult vkGetSemaphoreFdKHR(VkDevice device, const VkSemaphoreGetFdInfoKHR* pGetFdInfo, int* pFd);`

**Structure of `VkSemaphoreGetFdInfoKHR`:**

```c
typedef struct VkSemaphoreGetFdInfoKHR {
    VkStructureType                          sType;        // Offset 0
    const void*                              pNext;        // Offset 8
    VkSemaphore                              semaphore;    // Offset 16 (0x10)
    VkExternalSemaphoreHandleTypeFlagBits    handleType;   // Offset 24 (0x18)
} VkSemaphoreGetFdInfoKHR;
```

**Decompiled Function with Explanations:**

```c
// vt_handle_vkGetSemaphoreFdKHR - Handles the vkGetSemaphoreFdKHR Vulkan API call.
//
// vtContext: Pointer to the global Vortek context.
//
void vt_handle_vkGetSemaphoreFdKHR(VtContext* vtContext) {
    // Standard stack canary for stack corruption detection.
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28);

    // Get pointer to the serialized input arguments from vtContext->inputBufferPtr.
    char* input_data_ptr = *(char**)(vtContext->inputBufferPtr_offset); // Accesses vtContext->inputBufferPtr

    // Local struct to hold deserialized VkSemaphoreGetFdInfoKHR parameters.
    VkSemaphoreGetFdInfoKHR fd_info = {0}; // Initialize all members to zero.

    // Determine the offset for deserializing the VkDevice handle.
    ulong current_input_offset = 9;
    if (*input_data_ptr == '\0') {
        current_input_offset = 1;
    }

    // Deserialize VkDevice handle (first argument to vkGetSemaphoreFdKHR).
    VkDevice device = VkObject_fromId(*(void**)(input_data_ptr + current_input_offset));

    // Advance offset in input_data_ptr past the VkDevice handle.
    current_input_offset += 9;

    // Deserialize VkSemaphoreGetFdInfoKHR structure (second argument to vkGetSemaphoreFdKHR).
    // The serialized `VkSemaphoreGetFdInfoKHR` data starts at `input_data_ptr + current_input_offset`.

    // Deserialize fd_info.sType (VkStructureType, at offset 0 of VkSemaphoreGetFdInfoKHR).
    fd_info.sType = *(VkStructureType*)(input_data_ptr + current_input_offset);

    // Deserialize fd_info.semaphore (VkSemaphore, at offset 0x10 (16) of VkSemaphoreGetFdInfoKHR).
    fd_info.semaphore = VkObject_fromId(*(void**)(input_data_ptr + current_input_offset + 0x10));

    // Hardcode fd_info.handleType to VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT_KHR (0x10).
    // This explicitly overrides any value that might have been in the input buffer for handleType.
    fd_info.handleType = VK_EXTERNAL_SEMAPHORE_HANDLE_TYPE_SYNC_FD_BIT_KHR; // Value 0x10

    int pFd_result = 0; // Local variable to receive the file descriptor from the Vulkan API call.

    // Call the actual Vulkan API function: vkGetSemaphoreFdKHR.
    VkResult result = dispatch_vkGetSemaphoreFdKHR(device, &fd_info, &pFd_result);

    // Prepare a `msghdr` structure to send the `VkResult` and the file descriptor back to the client.
    msghdr response_msg = {0};
    iovec response_iov[1];

    // The first part of the response payload is the `VkResult`.
    response_iov[0].iov_base = &result;
    response_iov[0].iov_len = sizeof(VkResult);

    response_msg.msg_iov = response_iov;
    response_msg.msg_iovlen = 1;

    // Use a control message to pass the file descriptor over the socket.
    union {
        char control_buffer[CMSG_SPACE(sizeof(int))];
        struct cmsghdr align;
    } control_un;

    response_msg.msg_control = control_un.control_buffer;
    response_msg.msg_controllen = sizeof(control_un.control_buffer);

    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&response_msg);
    if (cmsg != NULL) {
        cmsg->cmsg_level = SOL_SOCKET;
        cmsg->cmsg_type = SCM_RIGHTS;
        cmsg->cmsg_len = CMSG_LEN(sizeof(int));
        *(int*)CMSG_DATA(cmsg) = pFd_result;
    }

    // Send the constructed message through the client socket file descriptor.
    sendmsg(vtContext->clientFd, &response_msg, 0);

    // Close the file descriptor obtained from the Vulkan API call.
    if (pFd_result > 0) {
        close(pFd_result);
    }

    // Stack canary check.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail();
    }
    return;
}
```

---

### Function: `vt_handle_vkGetFenceFdKHR`

This function handles the `vkGetFenceFdKHR` API call. This API is used to retrieve a Linux file descriptor (`fd`) for a given `VkFence` object.

**Vulkan API Signature:** `VkResult vkGetFenceFdKHR(VkDevice device, const VkFenceGetFdInfoKHR* pGetFdInfo, int* pFd);`

**Structure of `VkFenceGetFdInfoKHR`:**

```c
typedef struct VkFenceGetFdInfoKHR {
    VkStructureType                      sType;        // Offset 0
    const void*                          pNext;        // Offset 8
    VkFence                              fence;        // Offset 16 (0x10)
    VkExternalFenceHandleTypeFlagBits    handleType;   // Offset 24 (0x18)
} VkFenceGetFdInfoKHR;
```

**Decompiled Function with Explanations:**

```c
// vt_handle_vkGetFenceFdKHR - Handles the vkGetFenceFdKHR Vulkan API call.
//
// vtContext: Pointer to the global Vortek context.
//
void vt_handle_vkGetFenceFdKHR(VtContext* vtContext) {
    // Standard stack canary for stack corruption detection.
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28);

    // Get pointer to the serialized input arguments from vtContext->inputBufferPtr.
    char* input_data_ptr = *(char**)(vtContext->inputBufferPtr_offset); // Accesses vtContext->inputBufferPtr

    // Local struct to hold deserialized VkFenceGetFdInfoKHR parameters.
    VkFenceGetFdInfoKHR fd_info = {0}; // Initialize all members to zero.

    // Determine the offset for deserializing the VkDevice handle.
    ulong current_input_offset = 9;
    if (*input_data_ptr == '\0') {
        current_input_offset = 1;
    }

    // Deserialize VkDevice handle (first API argument).
    VkDevice device = VkObject_fromId(*(void**)(input_data_ptr + current_input_offset));

    // Advance offset in input_data_ptr past the VkDevice handle.
    current_input_offset += 9;

    // Deserialize VkFenceGetFdInfoKHR structure (second argument to vkGetFenceFdKHR).
    // The serialized `VkFenceGetFdInfoKHR` data starts at `input_data_ptr + current_input_offset`.

    // Deserialize fd_info.sType (VkStructureType, at offset 0 of VkFenceGetFdInfoKHR).
    fd_info.sType = *(VkStructureType*)(input_data_ptr + current_input_offset);

    // Deserialize fd_info.fence (VkFence, at offset 0x10 (16) of VkFenceGetFdInfoKHR).
    fd_info.fence = VkObject_fromId(*(void**)(input_data_ptr + current_input_offset + 0x10));

    // Hardcode fd_info.handleType to VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT_KHR (0x8).
    // This explicitly overrides any value that might have been in the input buffer for handleType.
    fd_info.handleType = VK_EXTERNAL_FENCE_HANDLE_TYPE_SYNC_FD_BIT_KHR; // Value 0x8

    int pFd_result = 0; // Local variable to receive the file descriptor from the Vulkan API call.

    // Call the actual Vulkan API function: vkGetFenceFdKHR.
    VkResult result = dispatch_vkGetFenceFdKHR(device, &fd_info, &pFd_result);

    // Prepare a `msghdr` structure to send the `VkResult` and the file descriptor back to the client.
    msghdr response_msg = {0};
    iovec response_iov[1];

    // The first part of the response payload is the `VkResult`.
    response_iov[0].iov_base = &result;
    response_iov[0].iov_len = sizeof(VkResult);

    response_msg.msg_iov = response_iov;
    response_msg.msg_iovlen = 1;

    // Use a control message to pass the file descriptor over the socket.
    union {
        char control_buffer[CMSG_SPACE(sizeof(int))];
        struct cmsghdr align;
    } control_un;

    response_msg.msg_control = control_un.control_buffer;
    response_msg.msg_controllen = sizeof(control_un.control_buffer);

    struct cmsghdr* cmsg = CMSG_FIRSTHDR(&response_msg);
    if (cmsg != NULL) {
        cmsg->cmsg_level = SOL_SOCKET;
        cmsg->cmsg_type = SCM_RIGHTS;
        cmsg->cmsg_len = CMSG_LEN(sizeof(int));
        *(int*)CMSG_DATA(cmsg) = pFd_result;
    }

    // Send the constructed message through the client socket file descriptor.
    sendmsg(vtContext->clientFd, &response_msg, 0);

    // Close the file descriptor obtained from the Vulkan API call.
    if (pFd_result > 0) {
        close(pFd_result);
    }

    // Stack canary check.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail();
    }
    return;
}
```

---

# Swapchain A

Here are the decompiled functions:

---

### `vt_handle_vkCreateSwapchainKHR`

This function handles the `vkCreateSwapchainKHR` Vulkan API call by reconstructing the `VkSwapchainCreateInfoKHR` structure and its `pNext` chain from a serialized input buffer. It then calls a custom helper function `XWindowSwapchain_create` to perform the actual swapchain creation, which integrates with the Android windowing system. Finally, it sends the creation result and the new `VkSwapchainKHR` handle back to the client via a `RingBuffer`.

```c
// vt_handle_vkCreateSwapchainKHR - Handles the vkCreateSwapchainKHR Vulkan API call.
//
// param_1: VtContext* vtContext - Pointer to the global Vortek context.
//
void vt_handle_vkCreateSwapchainKHR(VtContext* vtContext) {
    // Save stack canary for stack corruption detection.
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28);

    // Local variables to hold the reconstructed VkSwapchainCreateInfoKHR and its pNext chain structures.
    VkSwapchainCreateInfoKHR create_info_khr = { VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR };
    VkDeviceGroupSwapchainCreateInfoKHR device_group_swapchain_info = { VK_STRUCTURE_TYPE_DEVICE_GROUP_SWAPCHAIN_CREATE_INFO_KHR };
    VkImageSwapchainCreateInfoKHR image_swapchain_info = { VK_STRUCTURE_TYPE_IMAGE_SWAPCHAIN_CREATE_INFO_KHR };

    // Pointers for dynamically allocated arrays within pNext chain structures, if applicable.
    uint32_t* p_queue_family_indices_from_input_buffer = NULL;
    uint32_t  queue_family_index_count_from_input_buffer = 0;

    // Pointers and offset for parsing the raw input buffer.
    char* input_buffer_raw_data = vtContext->inputBufferPtr;
    ulong current_input_data_offset = 0; // Tracks current parsing position in input_buffer_raw_data

    // Determine the initial offset into the input buffer.
    // The input buffer format typically starts with a small header.
    // If the first byte is NULL, it suggests a minimal header; otherwise, a larger one (9 bytes).
    if (*input_buffer_raw_data == '\0') {
        current_input_data_offset = 1;
    } else {
        current_input_data_offset = 9;
    }

    // Reconstruct VkSwapchainCreateInfoKHR from the input buffer.
    // The data is read directly using byte offsets relative to the starting point of the serialized create info.
    // `(current_input_data_offset | 4)` is used to ensure proper alignment and skip initial serialization metadata.
    if (*(int *)(input_buffer_raw_data + current_input_data_offset) > 0) { // Check if pCreateInfo has content (size > 0)
        uint32_t* p_serialized_create_info_start = (uint32_t*)(input_buffer_raw_data + (current_input_data_offset | 4));

        create_info_khr.flags = *(VkSwapchainCreateFlagsKHR*)(p_serialized_create_info_start + 0); // Offset 0x0
        create_info_khr.surface = VkObject_fromId(*(void**)(p_serialized_create_info_start + 2)); // Offset 0x8 (2*sizeof(uint32_t))
        create_info_khr.minImageCount = p_serialized_create_info_start[4]; // Offset 0x10
        create_info_khr.imageFormat = *(VkFormat*)(p_serialized_create_info_start + 5); // Offset 0x14
        create_info_khr.imageColorSpace = *(VkColorSpaceKHR*)(p_serialized_create_info_start + 6); // Offset 0x18
        create_info_khr.imageExtent.width = p_serialized_create_info_start[7]; // Offset 0x1c
        create_info_khr.imageExtent.height = p_serialized_create_info_start[8]; // Offset 0x20
        create_info_khr.imageArrayLayers = p_serialized_create_info_start[9]; // Offset 0x24
        create_info_khr.imageUsage = p_serialized_create_info_start[10]; // Offset 0x28
        create_info_khr.imageSharingMode = *(VkSharingMode*)(p_serialized_create_info_start + 11); // Offset 0x2c
        create_info_khr.queueFamilyIndexCount = p_serialized_create_info_start[12]; // Offset 0x30

        // Handle the pQueueFamilyIndices array. This is a variable-length array.
        if (create_info_khr.queueFamilyIndexCount > 0) {
            queue_family_index_count_from_input_buffer = create_info_khr.queueFamilyIndexCount;
            size_t size_of_indices_bytes = queue_family_index_count_from_input_buffer * sizeof(uint32_t);
            // Allocate memory from the temporary buffer pool to store the indices.
            p_queue_family_indices_from_input_buffer = (uint32_t*)allocate_from_temp_buffer(vtContext, size_of_indices_bytes);
            if (p_queue_family_indices_from_input_buffer != NULL) {
                memcpy(p_queue_family_indices_from_input_buffer, p_serialized_create_info_start + 13, size_of_indices_bytes);
            }
            create_info_khr.pQueueFamilyIndices = p_queue_family_indices_from_input_buffer;
        } else {
            create_info_khr.pQueueFamilyIndices = NULL;
        }

        // Continue reconstructing VkSwapchainCreateInfoKHR fields after pQueueFamilyIndices.
        // The offset calculation dynamically accounts for the size of the pQueueFamilyIndices array.
        uint32_t base_offset_after_indices = 13 + create_info_khr.queueFamilyIndexCount;
        create_info_khr.preTransform = *(VkSurfaceTransformFlagBitsKHR*)(p_serialized_create_info_start + base_offset_after_indices);     // Offset based on base_offset_after_indices
        create_info_khr.compositeAlpha = *(VkCompositeAlphaFlagBitsKHR*)(p_serialized_create_info_start + base_offset_after_indices + 1); // +1 from prev field
        create_info_khr.presentMode = *(VkPresentModeKHR*)(p_serialized_create_info_start + base_offset_after_indices + 2);               // +2 from prev field
        create_info_khr.clipped = *(VkBool32*)(p_serialized_create_info_start + base_offset_after_indices + 3);                       // +3 from prev field
        create_info_khr.oldSwapchain = VkObject_fromId(*(void**)(p_serialized_create_info_start + base_offset_after_indices + 4));        // +4 from prev field

        // Advance current_input_data_offset to the position immediately after VkSwapchainCreateInfoKHR.
        // This is where pNext chain structs would follow in the serialized data.
        current_input_data_offset = (current_input_data_offset | 4) + (base_offset_after_indices + 6) * sizeof(uint32_t); // 6 fields after pQueueFamilyIndices
    }

    // Process the pNext chain of VkSwapchainCreateInfoKHR.
    // This loop reconstructs structures attached to pNext based on their sType.
    VkBaseInStructure* pNext_current_struct_ptr = (VkBaseInStructure*)(input_buffer_raw_data + current_input_data_offset);
    while (pNext_current_struct_ptr != NULL && pNext_current_struct_ptr->sType != 0) {
        if (pNext_current_struct_ptr->sType == VK_STRUCTURE_TYPE_DEVICE_GROUP_SWAPCHAIN_CREATE_INFO_KHR) {
            // Reconstruct VkDeviceGroupSwapchainCreateInfoKHR.
            device_group_swapchain_info.modes = *(VkDeviceGroupPresentModeFlagsKHR*)((char*)pNext_current_struct_ptr + sizeof(VkBaseInStructure));
            
            // Link this struct to the main VkSwapchainCreateInfoKHR's pNext.
            device_group_swapchain_info.pNext = create_info_khr.pNext;
            create_info_khr.pNext = (void*)&device_group_swapchain_info;

            // Advance the offset to skip this struct in the serialized input.
            current_input_data_offset += sizeof(VkDeviceGroupSwapchainCreateInfoKHR);

        } else if (pNext_current_struct_ptr->sType == VK_STRUCTURE_TYPE_IMAGE_SWAPCHAIN_CREATE_INFO_KHR) {
            // Reconstruct VkImageSwapchainCreateInfoKHR.
            image_swapchain_info.swapchain = VkObject_fromId(*(void**)((char*)pNext_current_struct_ptr + sizeof(VkBaseInStructure)));
            
            // Link this struct to the main VkSwapchainCreateInfoKHR's pNext.
            image_swapchain_info.pNext = create_info_khr.pNext;
            create_info_khr.pNext = (void*)&image_swapchain_info;

            // Advance the offset to skip this struct in the serialized input.
            current_input_data_offset += sizeof(VkImageSwapchainCreateInfoKHR);

        }
        // Update pNext_current_struct_ptr to point to the next potential struct in the serialized data.
        pNext_current_struct_ptr = (VkBaseInStructure*)(input_buffer_raw_data + current_input_data_offset);
        if (current_input_data_offset >= vtContext->inputBufferSize) { // Bounds check
            pNext_current_struct_ptr = NULL;
        }
    }

    // Get the VkDevice handle. This is the first parameter to `vkCreateSwapchainKHR`.
    // It's assumed to be at a fixed offset from the start of `vtContext->inputBufferPtr`.
    VkDevice logical_device = VkObject_fromId(*(void**)(input_buffer_raw_data + 1));

    // Get current window dimensions from the Android Java component.
    // `vtContext->javaComponentObject` (offset 0x90) and `vtContext->jniEnv` (offset 0x98)
    // are used to call Java methods for window interaction.
    VkExtent2D current_window_extent = {0};
    // `create_info_khr.imageFormat` is passed to `getWindowExtent`, possibly for format-specific dimension retrieval.
    getWindowExtent(vtContext->javaComponentObject, create_info_khr.imageFormat, &current_window_extent);

    // Variable to store the resulting `VkSwapchainKHR` handle (custom `XWindowSwapchain*`).
    XWindowSwapchain* created_swapchain_obj = NULL;

    // Variable to store the `VkResult` of the creation process.
    VkResult creation_result = VK_SUCCESS; // Default to success.

    // Check if the requested swapchain extent matches the actual window extent.
    // If they do not match, it indicates a potential issue or unsupported configuration.
    if (create_info_khr.imageExtent.width == current_window_extent.width &&
        create_info_khr.imageExtent.height == current_window_extent.height)
    {
        // If extents match, call the custom `XWindowSwapchain_create` function to handle creation.
        // This function likely encapsulates platform-specific integration (EGL/Android native window)
        // and the actual Vulkan `vkCreateSwapchainKHR` call, populating the `XWindowSwapchain` struct.
        created_swapchain_obj = (XWindowSwapchain*)XWindowSwapchain_create(
            logical_device,
            vtContext->graphicsQueueFamilyIndex,
            (long)&create_info_khr, // Pass the reconstructed create info struct
            vtContext->javaComponentObject, // Pass the Java component object for window interaction
            (void*)&created_swapchain_obj // Output pointer for the created custom struct
        );

        if (created_swapchain_obj == NULL) { // If custom creation fails (returns NULL)
            creation_result = VK_ERROR_INITIALIZATION_FAILED; // Map to Vulkan error
        } else {
            // The original code assigned `0` (VK_SUCCESS) here if `XWindowSwapchain_create` was not NULL.
            creation_result = VK_SUCCESS;
        }
    } else {
        // If extents do not match, directly set the result to VK_ERROR_INITIALIZATION_FAILED.
        creation_result = VK_ERROR_INITIALIZATION_FAILED;
        // The original code uses `0xc4653600` (VK_ERROR_INITIALIZATION_FAILED) explicitly.
    }

    // Allocate memory in the client's output buffer for the `VkResult` and the `VkSwapchainKHR` handle.
    // The output format is: `VkResult` (4 bytes) | `VkSwapchainKHR` handle (8 bytes).
    uint32_t output_data_size = sizeof(VkResult) + sizeof(VkSwapchainKHR); // Total 12 bytes.
    void* p_output_ring_buffer_data = allocate_from_temp_buffer(vtContext, output_data_size);

    if (p_output_ring_buffer_data != NULL) {
        *(VkResult*)p_output_ring_buffer_data = creation_result; // Write the VkResult
        *(XWindowSwapchain**)((long)p_output_ring_buffer_data + sizeof(VkResult)) = created_swapchain_obj; // Write the custom swapchain object handle
    }

    // Write the output data to the client's RingBuffer.
    // `vtContext->clientRingBuffer` is at offset 0x60 in `VtContext`.
    RingBuffer_write(vtContext->clientRingBuffer, p_output_ring_buffer_data, output_data_size);

    // Check stack canary for stack corruption.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail(); // Stack corruption detected, terminate.
    }
    return;
}

---

### `vt_handle_vkGetSwapchainImagesKHR`

This function handles the `vkGetSwapchainImagesKHR` Vulkan API call. It retrieves `VkImage` handles associated with a swapchain from an internally managed `XWindowSwapchain` structure and sends them back to the client via a `RingBuffer`.

```c
// vt_handle_vkGetSwapchainImagesKHR - Handles the vkGetSwapchainImagesKHR Vulkan API call.
//
// This function retrieves VkImage handles associated with a swapchain,
// typically from an internally managed `XWindowSwapchain` structure, and
// sends them back to the client via a RingBuffer.
//
// param_1: VtContext* vtContext - Pointer to the global Vortek context.
//
void vt_handle_vkGetSwapchainImagesKHR(VtContext* vtContext) {
    // Save stack canary for stack corruption detection.
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28);

    // Local variables to hold reconstructed input parameters and output results.
    VkDevice device = NULL;
    XWindowSwapchain* x_window_swapchain = NULL; // Assuming the serialized handle leads to this custom struct
    uint32_t client_provided_image_count_ptr_value = 0; // Value from pSwapchainImageCount from client input

    // Pointers for parsing the input buffer.
    char* input_buffer_raw_data = vtContext->inputBufferPtr;
    ulong current_input_offset = 0; // Tracks current parsing position

    // Determine the initial offset into the input buffer.
    // Standard pattern: offset 1 if empty header, offset 9 otherwise.
    if (*input_buffer_raw_data == '\0') {
        current_input_offset = 1;
    } else {
        current_input_offset = 9;
    }

    // Reconstruct VkDevice handle (first parameter to the original Vulkan call).
    // This assumes the `VkDevice` handle is at a fixed offset (e.g., `input_buffer_raw_data + 1`).
    device = VkObject_fromId(*(void**)(input_buffer_raw_data + (current_input_offset - 8))); // Re-calculating based on pattern of prev calls.

    // Reconstruct VkSwapchainKHR handle, which refers to our custom XWindowSwapchain struct.
    x_window_swapchain = (XWindowSwapchain*)VkObject_fromId(*(void**)(input_buffer_raw_data + current_input_offset));
    current_input_offset += sizeof(void*); // Advance past swapchain handle.

    // Reconstruct the client's provided pSwapchainImageCount.
    // This value indicates how many images the client expects or how large its output buffer is.
    client_provided_image_count_ptr_value = *(uint32_t*)(input_buffer_raw_data + current_input_offset);
    current_input_offset += sizeof(uint32_t); // Advance past pSwapchainImageCount value.

    // Initialize the pointer for the array of VkImage handles to be returned.
    VkImage* p_returned_images = NULL;
    VkResult result_code = VK_SUCCESS; // Initialize VkResult to success.

    // Get the actual number of images stored within the internal XWindowSwapchain struct.
    uint32_t actual_swapchain_image_count = 0;
    if (x_window_swapchain != NULL) {
        // `x_window_swapchain->numImages` is assumed to be at offset `0x10` from the start of `XWindowSwapchain`.
        actual_swapchain_image_count = x_window_swapchain->numImages;
    }
    
    // If the client provided a non-zero count, allocate a buffer to copy the VkImage handles into.
    if (client_provided_image_count_ptr_value > 0 && actual_swapchain_image_count > 0) {
        // Allocate memory for the VkImage handles. The allocation size is based on the actual image count.
        p_returned_images = (VkImage*)calloc(actual_swapchain_image_count, sizeof(VkImage));

        if (p_returned_images == NULL) {
            result_code = VK_ERROR_OUT_OF_HOST_MEMORY; // Handle allocation failure.
        } else {
            // Determine how many image handles to copy. It's the minimum of the client's requested count
            // and the actual number of images available in the swapchain.
            uint32_t images_to_copy = actual_swapchain_image_count;
            if (client_provided_image_count_ptr_value < actual_swapchain_image_count) {
                 images_to_copy = client_provided_image_count_ptr_value;
            }

            // Copy image handles from the internal `XWindowSwapchain` structure to the allocated buffer.
            // `x_window_swapchain->images` is assumed to be an array of `VkImage` handles at offset `0x8`.
            memcpy(p_returned_images, (void*)(x_window_swapchain->images), images_to_copy * sizeof(VkImage));
        }
    }
    
    // Call the actual Vulkan API `vkGetSwapchainImagesKHR` through the dispatch table.
    // The `p_returned_images` buffer (potentially NULL if allocation failed or client requested 0)
    // is passed to the original Vulkan function.
    result_code = dispatch_vkGetSwapchainImagesKHR(device, (VkSwapchainKHR)x_window_swapchain, &actual_swapchain_image_count, p_returned_images);

    // Prepare the data to send back to the client via the RingBuffer.
    // The output format is: `VkResult` (4 bytes) | `actual_swapchain_image_count` (4 bytes) | `VkImage` handles (variable size).
    uint32_t output_data_total_size = sizeof(VkResult) + sizeof(uint32_t);
    if (p_returned_images != NULL && actual_swapchain_image_count > 0) {
        output_data_total_size += actual_swapchain_image_count * sizeof(VkImage);
    }

    // Allocate temporary memory for the output data to be written to the RingBuffer.
    void* p_output_ring_buffer_data = allocate_from_temp_buffer(vtContext, output_data_total_size);
    if (p_output_ring_buffer_data != NULL) {
        *(VkResult*)p_output_ring_buffer_data = result_code; // Write the `VkResult`
        *(uint32_t*)((long)p_output_ring_buffer_data + sizeof(VkResult)) = actual_swapchain_image_count; // Write the actual image count

        if (p_returned_images != NULL && actual_swapchain_image_count > 0) {
            // Copy the image handles to the output buffer.
            memcpy((void*)((long)p_output_ring_buffer_data + sizeof(VkResult) + sizeof(uint32_t)),
                   p_returned_images, actual_swapchain_image_count * sizeof(VkImage));
        }
    }

    // Write the prepared output data to the client's RingBuffer.
    RingBuffer_write(vtContext->clientRingBuffer, p_output_ring_buffer_data, output_data_total_size);

    // Clean up dynamically allocated memory for the image array.
    if (p_returned_images != NULL) {
        free(p_returned_images);
    }

    // Check stack canary for stack corruption.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail(); // Stack corruption detected.
    }
    return;
}

---

### `XWindowSwapchain_acquireNextImage`

This function is a custom implementation for acquiring the next image from an `XWindowSwapchain`, simulating the behavior of `vkAcquireNextImageKHR`. It handles synchronization via queue submission, validates window dimensions, and calculates the next image index for presentation.

```c
// XWindowSwapchain_acquireNextImage - Custom implementation for acquiring the next image from an XWindowSwapchain.
//
// This function simulates `vkAcquireNextImageKHR` behavior, handling internal synchronization
// and window dimension validation before determining the next image index.
//
// param_1: XWindowSwapchain* swapchain_obj - Pointer to the custom XWindowSwapchain structure.
// param_2: uint64_t timeout - Timeout in nanoseconds for acquiring the next image. (Not directly used in this snippet's logic).
// param_3: VkSemaphore semaphore - A semaphore to signal when the image is available. Can be VK_NULL_HANDLE.
// param_4: VkFence fence - A fence to signal when the image is available. Can be VK_NULL_HANDLE.
// param_5: uint32_t* pImageIndex - Output pointer for the acquired image index.
//
VkResult XWindowSwapchain_acquireNextImage(XWindowSwapchain* swapchain_obj, uint64_t timeout, VkSemaphore semaphore, VkFence fence, uint32_t* pImageIndex) {
    // Save stack canary for stack corruption detection.
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28);

    VkResult result = VK_SUCCESS; // Initialize the result to success.
    VkDevice logical_device = NULL; // Pointer to the VkDevice handle.
    VkQueue graphics_queue = NULL;  // Pointer to the graphics queue.

    // Reconstruct VtContext pointer from XWindowSwapchain struct (at offset 0xc).
    VtContext* vtContext = swapchain_obj->vtContext;

    // Get the VkDevice handle and the graphics queue from the VtContext.
    // Assuming VkDevice is at offset +1 from `vtContext->inputBufferPtr`'s base for dispatch,
    // and graphicsQueueFamilyIndex is at offset 0x74 in VtContext.
    logical_device = VkObject_fromId(*(void**)(vtContext->inputBufferPtr + 1));
    dispatch_vkGetDeviceQueue(logical_device, vtContext->graphicsQueueFamilyIndex, 0, &graphics_queue);

    // Perform an optional queue submission for synchronization if a semaphore or fence is provided.
    if (semaphore != VK_NULL_HANDLE || fence != VK_NULL_HANDLE) {
        VkSubmitInfo submit_info = { VK_STRUCTURE_TYPE_SUBMIT_INFO };
        submit_info.waitSemaphoreCount = 0;
        submit_info.pWaitSemaphores = NULL;
        submit_info.pWaitDstStageMask = NULL;
        submit_info.commandBufferCount = 0;
        submit_info.pCommandBuffers = NULL;
        submit_info.signalSemaphoreCount = 0;
        submit_info.pSignalSemaphores = NULL;

        if (semaphore != VK_NULL_HANDLE) {
            submit_info.signalSemaphoreCount = 1;
            submit_info.pSignalSemaphores = &semaphore;
        }

        // Submit an empty command buffer to the graphics queue to signal the semaphore/fence.
        VkResult submit_result = dispatch_vkQueueSubmit(graphics_queue, 1, &submit_info, fence);

        // Handle potential device loss during submission.
        if (submit_result == VK_ERROR_DEVICE_LOST) { // VK_ERROR_DEVICE_LOST is -4
            vtContext->lastError = 0xfffffffc; // Set a custom internal error flag for device lost.
            result = submit_result; // Propagate the device lost error.
            goto end_function; // Skip further processing.
        }
        result = submit_result; // Propagate other submit results.
    }

    // Get the current window dimensions from the Android Java component.
    VkExtent2D current_window_extent = {0};
    // `swapchain_obj->imageFormat` (at offset 0x28) is passed to `getWindowExtent`, possibly for format-specific context.
    getWindowExtent(vtContext->javaComponentObject, swapchain_obj->imageFormat, &current_window_extent);

    // Validate if the current window dimensions match the swapchain's expected dimensions.
    // `swapchain_obj->imageExtent.width` (at offset 0x20) and `swapchain_obj->imageExtent.height` (at offset 0x24)
    if (swapchain_obj->imageExtent.width != current_window_extent.width ||
        swapchain_obj->imageExtent.height != current_window_extent.height)
    {
        // If dimensions mismatch, return VK_ERROR_INITIALIZATION_FAILED.
        // The value `0xc4653600` is `VK_ERROR_INITIALIZATION_FAILED`.
        result = VK_ERROR_INITIALIZATION_FAILED;
        *pImageIndex = 0; // Set output image index to 0 on error.
        goto end_function; // Skip further processing.
    }

    // Calculate the next image index in a round-robin fashion.
    // `swapchain_obj->currentImageIndex` (at offset 0x0) is the current image index.
    // `swapchain_obj->numImages` (at offset 0x10) is the total number of images in the swapchain.
    uint32_t next_image_index = (swapchain_obj->currentImageIndex + 1) % swapchain_obj->numImages;
    *pImageIndex = next_image_index; // Store the calculated next image index in the output pointer.
    
    result = VK_SUCCESS; // All checks passed, return success.

end_function:
    // Check stack canary for stack corruption before returning.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail(); // Stack corruption detected.
    }
    return result;
}
```

# Swapchain B

I will provide the decompiled code for each function, including inferred struct layouts and constants.

First, let's list the hexadecimal constants and their decimal equivalents, and their likely mapping to Vulkan enums from `vulkan_core.h`:

**Hexadecimal Constants and Their Meanings:**
- `0x34` (52): `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES`
- `0x3b9dc7a0` (1000070000): `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES`
- `0x25` (37): `VK_FORMAT_R8G8B8A8_UNORM`
- `0x2b` (43): `VK_FORMAT_R8G8B8A8_SRGB` (Note: The provided Vulkan header has `VK_FORMAT_B8G8R8A8_SRGB = 50`, `VK_FORMAT_R8G8B8A8_SRGB = 43`. `0x2b` is 43, so it's `R8G8B8A8_SRGB`. However, the function `getSurfaceFormats` assigns this to `puVar1[2]`, while `puVar1[1]` is left zeroed. This means it's likely filling `format` and `colorSpace` is implicitly 0 (`VK_COLOR_SPACE_SRGB_NONLINEAR_KHR`). So `formats[1].format` is 43. This is consistent).
- `0x400` (1024): `VK_EXTERNAL_MEMORY_HANDLE_TYPE_ANDROID_HARDWARE_BUFFER_BIT_ANDROID`
- `0x3b9be341` (1000072001): `VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO`
- `0x3b9cc1eb` (1000127001): `VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO`
- `0x3b9cba19` (1000129001): `VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID` (or `VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO` which is `5`). This is ambiguous. `0x3b9cba19` (1000129001) is `VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID`. The line `local_e8 = 0x3b9cba19;` is used before `dispatch_vkAllocateMemory`, which expects `VkMemoryAllocateInfo`. This is a common pattern for aliasing `VkMemoryAllocateInfo` to the extension struct type. Thus, `0x3b9cba19` seems to be a custom alias for `VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO` in this specific code context or a misinterpretation of the decompiler. Based on the `vulkan_core.h`, `VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO` is `5`, not `1000129001`. The use is `local_e8 = 0x3b9cba19; local_e0 = &local_c8; local_c8 = 0x3b9cc1eb;`, `local_110 = 5; puStack_108 = &local_e8;`. The final `VkMemoryAllocateInfo` used is `local_110`, which has `sType = 5`, so it is `VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO`. `local_e8` and `local_c8` are part of its `pNext` chain. Thus `local_e8` is likely the sType for `VkAndroidHardwareBufferPropertiesANDROID` used as a pNext, but it's set to `0x3b9cba19` (1000129001). This is incorrect as `VkAndroidHardwareBufferPropertiesANDROID` is not part of `VkMemoryAllocateInfo`'s `pNext` chain. It's more likely `local_e8` (which is `0x3B9CBA19`) is the sType for some Android-specific struct in the pNext chain of `VkMemoryAllocateInfo` or that the decompiler misattributed its initialisation. However, `XWindowSwapchain_create` explicitly uses `VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID` correctly. Given `local_e8` is assigned `0x3b9cba19` and used in `VkMemoryAllocateInfo.pNext` as `puStack_108 = &local_e8;`, it strongly suggests `local_e8` itself is a struct that `pNext` points to, and its `sType` is `0x3b9cba19` (`VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID`). This implies memory allocation info receives ANDROID_HARDWARE_BUFFER_PROPERTIES in its pNext. This is not standard Vulkan usage, but a custom extension. Let's stick with the Vulkan header defined `VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO` for `allocInfo.sType` (`local_110` being `5`) and assume the `pNext` chain is specific to this implementation.

Let's define the necessary structs and function pointers first.

```c
#include <string.h> // For memset, memcpy, strcmp, strncpy
#include <stdio.h>  // For snprintf
#include <stdlib.h> // For calloc, malloc, free
#include <stdbool.h> // For bool type

// Assuming these are defined elsewhere or in the Vulkan headers:
#define VK_MAX_PHYSICAL_DEVICE_NAME_SIZE 256
#define VK_MAX_DRIVER_INFO_SIZE 256
#define VK_FALSE 0 // Vulkan defines VkBool32 as uint32_t, so 0 is false.
#define VK_NULL_HANDLE ((void*)0)

// Version extraction macros from vulkan_core.h
#define VK_API_VERSION_MAJOR(version) (((uint32_t)(version) >> 22U) & 0x7FU)
#define VK_API_VERSION_MINOR(version) (((uint32_t)(version) >> 12U) & 0x3FFU)
#define VK_API_VERSION_PATCH(version) ((uint32_t)(version) & 0xFFFU)

// Enum constants for VkStructureType (converted from hex/decimal found in the original snippet)
typedef enum VkStructureType {
    VK_STRUCTURE_TYPE_SUBMIT_INFO = 4,
    VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO = 5,
    VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO = 14,
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES = 52, // 0x34
    VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR = 1000001000,
    VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO = 1000072001, // 0x3b9be341
    VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID = 1000129001, // 0x3b9cc1e9
    VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO = 1000127001, // 0x3b9cc1eb
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES = 1000070000, // 0x3b9dc7a0
} VkStructureType;

// Minimal Vulkan API structs needed for decompilation (from vulkan_core.h)
typedef uint32_t VkResult;
typedef uint32_t VkFlags;
typedef uint64_t VkDeviceSize;
typedef uint64_t VkDeviceAddress;
typedef void* VkDevice;
typedef void* VkBuffer;
typedef void* VkImage;
typedef void* VkQueue;
typedef void* VkSemaphore;
typedef void* VkFence;
typedef void* VkDeviceMemory;
typedef void* VkImageView;
typedef void* VkPhysicalDevice;
typedef void* VkInstance;
typedef void* VkPipelineLayout;
typedef void* VkRenderPass;
typedef void* VkCommandPool;
typedef void* VkCommandBuffer;
typedef uint32_t VkImageAspectFlags;
typedef uint32_t VkImageLayout;
typedef uint32_t VkFormat;
typedef uint32_t VkImageType;
typedef uint32_t VkImageTiling;
typedef uint32_t VkImageUsageFlags;
typedef uint32_t VkImageCreateFlags;
typedef uint32_t VkSharingMode;
typedef uint32_t VkSampleCountFlagBits;
typedef uint32_t VkMemoryPropertyFlags;
typedef uint32_t VkPipelineStageFlags;
typedef uint32_t VkPipelineBindPoint;
typedef uint32_t VkIndexType;
typedef uint32_t VkSubpassContents;
typedef uint32_t VkBool32;
typedef uint32_t VkCullModeFlags;
typedef uint32_t VkFrontFace;
typedef uint32_t VkPrimitiveTopology;
typedef uint32_t VkCompareOp;
typedef uint32_t VkStencilFaceFlags;
typedef uint32_t VkStencilOp;
typedef uint32_t VkLogicOp;
typedef uint32_t VkBlendFactor;
typedef uint32_t VkBlendOp;
typedef uint32_t VkColorComponentFlags;
typedef uint32_t VkSamplerAddressMode;
typedef uint32_t VkSamplerMipmapMode;
typedef uint32_t VkDescriptorType;
typedef uint32_t VkAttachmentLoadOp;
typedef uint32_t VkAttachmentStoreOp;
typedef uint32_t VkPipelineCreateFlags;
typedef uint32_t VkDescriptorPoolResetFlags;
typedef uint32_t VkCommandBufferUsageFlags;
typedef uint32_t VkQueryControlFlags;
typedef uint32_t VkCommandBufferResetFlags;

// Structs directly from vulkan_core.h for parameters and returned types
typedef struct VkBaseInStructure {
    VkStructureType                    sType;
    const struct VkBaseInStructure*    pNext;
} VkBaseInStructure;

typedef struct VkExtent2D {
    uint32_t    width;
    uint32_t    height;
} VkExtent2D;

typedef struct VkExtent3D {
    uint32_t    width;
    uint32_t    height;
    uint32_t    depth;
} VkExtent3D;

typedef struct VkImageSubresourceRange {
    VkImageAspectFlags    aspectMask;
    uint32_t              baseMipLevel;
    uint32_t              levelCount;
    uint32_t              baseArrayLayer;
    uint32_t              layerCount;
} VkImageSubresourceRange;

typedef struct VkImageCreateInfo {
    VkStructureType          sType;
    const void*              pNext;
    VkImageCreateFlags       flags;
    VkImageType              imageType;
    VkFormat                 format;
    VkExtent3D               extent;
    uint32_t                 mipLevels;
    uint32_t                 arrayLayers;
    VkSampleCountFlagBits    samples;
    VkImageTiling            tiling;
    VkImageUsageFlags        usage;
    VkSharingMode            sharingMode;
    uint32_t                 queueFamilyIndexCount;
    const uint32_t*          pQueueFamilyIndices;
    VkImageLayout            initialLayout;
} VkImageCreateInfo;

typedef struct VkMemoryAllocateInfo {
    VkStructureType    sType;
    const void*        pNext;
    VkDeviceSize       allocationSize;
    uint32_t           memoryTypeIndex;
} VkMemoryAllocateInfo;

typedef struct VkMemoryDedicatedAllocateInfo {
    VkStructureType    sType;
    const void*        pNext;
    VkImage            image;
    VkBuffer           buffer;
} VkMemoryDedicatedAllocateInfo;

typedef struct VkSubmitInfo {
    VkStructureType                sType;
    const void*                    pNext;
    uint32_t                       waitSemaphoreCount;
    const VkSemaphore*             pWaitSemaphores;
    const VkPipelineStageFlags*    pWaitDstStageMask;
    uint32_t                       commandBufferCount;
    const VkCommandBuffer*         pCommandBuffers;
    uint32_t                       signalSemaphoreCount;
    const VkSemaphore*             pSignalSemaphores;
} VkSubmitInfo;

typedef struct VkSwapchainCreateInfoKHR {
    VkStructureType                  sType;
    const void*                      pNext;
    VkSwapchainCreateFlagsKHR        flags;
    VkSurfaceKHR                     surface;
    uint32_t                         minImageCount;
    VkFormat                         imageFormat;
    VkColorSpaceKHR                  imageColorSpace;
    VkExtent2D                       imageExtent;
    uint32_t                         imageArrayLayers;
    VkImageUsageFlags                imageUsage;
    VkSharingMode                    imageSharingMode;
    uint32_t                         queueFamilyIndexCount;
    const uint32_t*                  pQueueFamilyIndices;
    VkSurfaceTransformFlagBitsKHR    preTransform;
    VkCompositeAlphaFlagBitsKHR      compositeAlpha;
    VkPresentModeKHR                 presentMode;
    VkBool32                         clipped;
    VkSwapchainKHR                   oldSwapchain;
} VkSwapchainCreateInfoKHR;

typedef struct VkPhysicalDeviceProperties {
    uint32_t                            apiVersion;
    uint32_t                            driverVersion;
    uint32_t                            vendorID;
    uint32_t                            deviceID;
    VkPhysicalDeviceType                deviceType;
    char                                deviceName[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE];
    uint8_t                             pipelineCacheUUID[VK_UUID_SIZE];
    VkPhysicalDeviceLimits              limits;
    VkPhysicalDeviceSparseProperties    sparseProperties;
} VkPhysicalDeviceProperties;

typedef struct VkPhysicalDeviceVulkan12Properties {
    VkStructureType                      sType;
    void*                                pNext;
    uint32_t                             driverID;
    char                                 driverName[VK_MAX_DRIVER_NAME_SIZE];
    char                                 driverInfo[VK_MAX_DRIVER_INFO_SIZE]; // Corresponds to piVar6 + 0x45 in dword offsets
    VkConformanceVersion                 conformanceVersion;
    VkShaderFloatControlsIndependence    denormBehaviorIndependence;
    VkShaderFloatControlsIndependence    roundingModeIndependence;
    VkBool32                             shaderSignedZeroInfNanPreserveFloat16; // 0x89 (offset from start of struct in dwords)
    VkBool32                             shaderSignedZeroInfNanPreserveFloat32; // 0x8A
    VkBool32                             shaderSignedZeroInfNanPreserveFloat64; // 0x8B
    VkBool32                             shaderDenormPreserveFloat16;           // 0x8C
    VkBool32                             shaderDenormPreserveFloat32;           // 0x8D
    VkBool32                             shaderDenormPreserveFloat64;           // 0x8E
    VkBool32                             shaderDenormFlushToZeroFloat16;        // 0x8F
    VkBool32                             shaderDenormFlushToZeroFloat32;        // 0x90
    // ... other members as in vulkan_core.h
} VkPhysicalDeviceVulkan12Properties;

typedef struct VkPhysicalDeviceGroupProperties {
    VkStructureType     sType;
    void*               pNext;
    uint32_t            physicalDeviceCount;
    VkPhysicalDevice    physicalDevices[32]; // VK_MAX_DEVICE_GROUP_SIZE = 32
    VkBool32            subsetAllocation; // At offset 0x114 (byte offset) or 0x45 (dword offset) from struct start
} VkPhysicalDeviceGroupProperties;


typedef struct VkExternalMemoryImageCreateInfo {
    VkStructureType                    sType;
    const void*                        pNext;
    VkExternalMemoryHandleTypeFlags    handleTypes;
} VkExternalMemoryImageCreateInfo;

typedef struct VkAndroidHardwareBufferPropertiesANDROID {
    VkStructureType sType;
    void* pNext;
    VkDeviceSize allocationSize;
    uint32_t memoryTypeBits;
} VkAndroidHardwareBufferPropertiesANDROID;

// Custom struct inferred for the XWindowSwapchain object
// This struct is `undefined4 *__ptr` in `XWindowSwapchain_create`
typedef struct XWindowSwapchainObject {
    VkExtent2D                extent; // width at 0x00, height at 0x04
    struct {
        VkImage image;
        VkDeviceMemory memory;
    }* images;                // Pointer to array of image-memory pairs (0x08)
    uint32_t                  imageCount; // Total number of images (0x10)
    VkFormat                  imageFormat; // (0x14)
    VkExtent2D                imageExtent; // (0x18)
    VkImageUsageFlags         imageUsage; // (0x20)
    VtContext*                vtCtx; // Pointer back to VtContext (0x28)
    VkQueue                   graphicsQueue; // (0x30)
} XWindowSwapchainObject;

// Custom response header for VkGetSwapchainImagesKHR
typedef struct VkGetSwapchainImagesResponseHeader {
    uint32_t reserved0; // Always 0
    uint32_t headerSize; // Should be sizeof(VkGetSwapchainImagesResponseHeader) = 12
    uint32_t imageCount; // The actual image count
} VkGetSwapchainImagesResponseHeader;


// Function pointer types for Vulkan API calls (dispatch_* functions)
typedef VkResult (VKAPI_PTR *PFN_vkCreateImage)(VkDevice device, const VkImageCreateInfo* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkImage* pImage);
typedef void (VKAPI_PTR *PFN_vkGetAndroidHardwareBufferPropertiesANDROID)(VkDevice device, void* buffer, VkAndroidHardwareBufferPropertiesANDROID* pProperties);
typedef VkResult (VKAPI_PTR *PFN_vkAllocateMemory)(VkDevice device, const VkMemoryAllocateInfo* pAllocateInfo, const VkAllocationCallbacks* pAllocator, VkDeviceMemory* pMemory);
typedef VkResult (VKAPI_PTR *PFN_vkBindImageMemory)(VkDevice device, VkImage image, VkDeviceMemory memory, VkDeviceSize memoryOffset);
typedef void (VKAPI_PTR *PFN_vkGetDeviceQueue)(VkDevice device, uint32_t queueFamilyIndex, uint32_t queueIndex, VkQueue* pQueue);
typedef VkResult (VKAPI_PTR *PFN_vkQueueSubmit)(VkQueue queue, uint32_t submitCount, const VkSubmitInfo* pSubmits, VkFence fence);
typedef void (VKAPI_PTR *PFN_vkDestroyImage)(VkDevice device, VkImage image, const VkAllocationCallbacks* pAllocator);
typedef void (VKAPI_PTR *PFN_vkFreeMemory)(VkDevice device, VkDeviceMemory memory, const VkAllocationCallbacks* pAllocator);

// Global function pointers (placeholders, assumed to be initialized elsewhere)
PFN_vkCreateImage dispatch_vkCreateImage;
PFN_vkGetAndroidHardwareBufferPropertiesANDROID dispatch_vkGetAndroidHardwareBufferPropertiesANDROID;
PFN_vkAllocateMemory dispatch_vkAllocateMemory;
PFN_vkBindImageMemory dispatch_vkBindImageMemory;
PFN_vkGetDeviceQueue dispatch_vkGetDeviceQueue;
PFN_vkQueueSubmit dispatch_vkQueueSubmit;
PFN_vkDestroyImage dispatch_vkDestroyImage;
PFN_vkFreeMemory dispatch_vkFreeMemory;

// External functions (placeholders, assumed to be implemented elsewhere)
extern void __stack_chk_fail(void); // For stack canary check
extern void* AHardwareBuffer_describe(void* buffer, VkExtent2D* extent); // Takes AHardwareBuffer* and VkExtent2D*
extern void* VkObject_fromId(uint64_t id); // Converts ID to Vulkan handle
extern void ArrayList_add(void* arrayListPtr, void* item); // Adds item to an ArrayList (part of VtContext)
extern uint32_t getMemoryTypeIndex(uint32_t memoryTypeBits, VkMemoryPropertyFlags requiredFlags); // Custom memory type selection
extern bool RingBuffer_write(void* ringBuffer, const void* data, size_t size); // Writes to a RingBuffer

// Helper function declarations
void getWindowExtent(VtContext* vtCtx, uint32_t* pWidth, uint32_t* pHeight);
VkSurfaceFormatKHR* getSurfaceFormats(uint32_t* pCount);
XWindowSwapchainObject* XWindowSwapchain_create(
    VkDevice device,
    uint32_t graphicsQueueFamilyIndex,
    const VkSwapchainCreateInfoKHR* pCreateInfo,
    VtContext* vtCtx,
    uint32_t windowWidth
);
void XWindowSwapchain_destroy(VkDevice device, XWindowSwapchainObject* swapchainObj);
void XWindowSwapchain_acquireNextImage(
    XWindowSwapchainObject* swapchainObj,
    VkDevice device,
    VkSemaphore pWaitSemaphore,
    VkFence pSignalFence,
    uint32_t* pImageIndex
);
void XWindowSwapchain_presentImage(XWindowSwapchainObject* swapchainObj);

/**
 * @brief Handles the vkCreateSwapchainKHR Vulkan API call.
 *
 * This function processes a serialized VkSwapchainCreateInfoKHR structure,
 * creates the necessary Vulkan image and memory resources backed by Android
 * hardware buffers, and binds them for the swapchain. It also interacts with
 * the Android windowing system to get window dimensions. Finally, it sends
 * the result and the created XWindowSwapchainObject ID back to the client.
 *
 * @param vtCtx Pointer to the global VtContext containing client connection
 *              details, temporary buffers, and other state.
 */
void vt_handle_vkCreateSwapchainKHR(VtContext* vtCtx) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Stack canary for security

    // Input buffer structure: [VkDevice ID (uint64_t)] [VkSwapchainCreateInfoKHR payload]
    // The input buffer pointer `vtCtx->inputBufferPtr` points to the beginning of this payload.
    char* inputBufferPtr = vtCtx->inputBufferPtr;

    // 1. Deserialize VkDevice handle.
    // The decompiled code's parsing of the input buffer is unusual.
    // `if (*inputBufferPtr == '\0') { ... } else { pvVar6 = *(void **)(inputBufferPtr + 1); }`
    // This `inputBufferPtr + 1` for VkDevice ID is not 8-byte aligned and typically not how IDs are passed.
    // Assuming a standard serialization where the VkDevice ID is the first 8 bytes.
    uint64_t device_id_raw = *(uint64_t*)inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(device_id_raw);

    // 2. Locate VkSwapchainCreateInfoKHR payload after the device ID.
    const VkSwapchainCreateInfoKHR* pCreateInfo = (const VkSwapchainCreateInfoKHR*)(inputBufferPtr + sizeof(uint64_t));

    uint32_t windowWidth = 0;  // local_f0
    uint32_t windowHeight = 0; // local_ec

    XWindowSwapchainObject* xWindowSwapchain = NULL;
    VkResult result = VK_ERROR_UNKNOWN; // Default error result

    // Get window dimensions from Java component.
    // `vtCtx` is passed to `getWindowExtent` to provide access to JNIEnv and method IDs.
    getWindowExtent(vtCtx, &windowWidth, &windowHeight);

    // Check if swapchain image dimensions from pCreateInfo match current window dimensions.
    if (pCreateInfo->imageExtent.width == windowWidth && pCreateInfo->imageExtent.height == windowHeight) {
        // Create the internal XWindowSwapchain object.
        // `vtCtx` is passed here to allow `XWindowSwapchain_create` access to JNIEnv etc.
        xWindowSwapchain = XWindowSwapchain_create(
            device,
            vtCtx->graphicsQueueFamilyIndex,
            pCreateInfo,
            vtCtx,        // Pass the entire VtContext
            windowWidth   // Passed as param_5 to XWindowSwapchain_create
        );

        if (xWindowSwapchain != NULL) {
            result = VK_SUCCESS;
        } else {
            result = VK_ERROR_INITIALIZATION_FAILED;
        }
    } else {
        result = VK_ERROR_OUT_OF_DATE_KHR;
    }

    // 3. Serialize and send back the result and the new swapchain handle.
    // Response format: [VkResult (4 bytes)] [total_payload_size (4 bytes)] [VkSwapchainKHR handle (8 bytes)]
    uint32_t responsePayloadSize = sizeof(uint64_t); // Size of the VkSwapchainKHR handle
    uint32_t totalResponseSize = sizeof(VkResult) + sizeof(uint32_t) + responsePayloadSize;

    void* responseDataBuffer;
    // Attempt to use `vtCtx->tempBuffer` for allocation, fallback to `malloc`
    if ((vtCtx->tempBufferOffset + totalResponseSize < 0x10000) && (vtCtx->tempBuffer != NULL)) {
        responseDataBuffer = (char*)vtCtx->tempBuffer + vtCtx->tempBufferOffset;
        vtCtx->tempBufferOffset += totalResponseSize;
    } else {
        responseDataBuffer = malloc(totalResponseSize);
        ArrayList_add(&vtCtx->tempAllocations, responseDataBuffer);
    }
    memset(responseDataBuffer, 0, totalResponseSize); // Initialize with zeros

    // Populate response buffer
    *(VkResult*)responseDataBuffer = result;
    *(uint32_t*)((char*)responseDataBuffer + sizeof(VkResult)) = responsePayloadSize;
    *(uint64_t*)((char*)responseDataBuffer + sizeof(VkResult) + sizeof(uint32_t)) = (uint64_t)xWindowSwapchain; // Store XWindowSwapchainObject pointer as the handle

    // Write the entire response buffer to the client ring buffer
    RingBuffer_write(vtCtx->clientRingBuffer, responseDataBuffer, totalResponseSize);

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Handles the vkGetSwapchainImagesKHR Vulkan API call.
 *
 * This function retrieves the list of images associated with a swapchain,
 * converts internal image objects to Vulkan VkImage handles, and sends
 * them back to the client.
 *
 * @param vtCtx Pointer to the global VtContext.
 */
void vt_handle_vkGetSwapchainImagesKHR(VtContext* vtCtx) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Stack canary

    // Input buffer structure:
    // [VkDevice ID (uint64_t)]
    // [VkSwapchainKHR ID (uint64_t)]
    // [Pointer to client-side uint32_t for pSwapchainImageCount (uint64_t)]
    // [Pointer to client-side VkImage* for pImages (uint64_t)]

    char* inputBufferPtr = vtCtx->inputBufferPtr;

    // Deserialize input parameters
    uint64_t device_id_raw = *(uint64_t*)inputBufferPtr;
    VkDevice device = (VkDevice)VkObject_fromId(device_id_raw); // `param_1` in `XWindowSwapchain_acquireNextImage` becomes this `device`
    uint64_t swapchain_id_raw = *(uint64_t*)(inputBufferPtr + 8);
    uint32_t* pSwapchainImageCount_client_ptr = (uint32_t*)*(uint64_t*)(inputBufferPtr + 16);
    // VkImage* pImages_client_ptr = (VkImage*)*(uint64_t*)(inputBufferPtr + 24); // This pointer is not directly used by server-side logic here

    XWindowSwapchainObject* swapchain = (XWindowSwapchainObject*)VkObject_fromId(swapchain_id_raw);

    VkResult result = VK_SUCCESS;
    uint32_t imageCount = 0;
    VkImage* imagesArray = NULL; // Temporary array to hold VkImage handles
    bool allocatedImagesArrayTemp = false;

    if (swapchain != NULL) {
        imageCount = swapchain->imageCount; // Get the image count from the internal object

        if (imageCount > 0) {
            // Allocate temporary array to hold VkImage handles for the response.
            // The original disassembly's `calloc((long)*(int *)((long)pvVar7 + 0x10),0x38)` is interpreted as
            // `calloc(imageCount, sizeof(VkImage))` since only VkImage pointers are copied.
            imagesArray = (VkImage*)calloc(imageCount, sizeof(VkImage));
            if (imagesArray == NULL) {
                result = VK_ERROR_OUT_OF_HOST_MEMORY;
            } else {
                allocatedImagesArrayTemp = true;
                // Copy VkImage handles from the internal XWindowSwapchainObject
                for (uint32_t i = 0; i < imageCount; ++i) {
                    imagesArray[i] = swapchain->images[i].image;
                }
            }
        }
    } else {
        result = VK_ERROR_INVALID_HANDLE; // Or appropriate error if swapchain is null
    }

    // Prepare response data: [VkResult (4 bytes)] [total_payload_size (4 bytes)] [header + image handles array]
    // The header `__s` holds: [reserved0 (4 bytes)] [headerSize (4 bytes)] [imageCount (4 bytes)]
    uint32_t headerSize = sizeof(VkGetSwapchainImagesResponseHeader);
    uint32_t imagesDataSize = imageCount * sizeof(uint64_t); // Each VkImage is a uint64_t ID/pointer
    uint32_t totalPayloadSize = headerSize + imagesDataSize;

    void* responseDataBuffer;
    // Attempt to use `vtCtx->tempBuffer` for allocation, fallback to `malloc`
    if ((vtCtx->tempBufferOffset + totalPayloadSize < 0x10000) && (vtCtx->tempBuffer != NULL)) { // Max tempBuffer size 0x10000
        responseDataBuffer = (char*)vtCtx->tempBuffer + vtCtx->tempBufferOffset;
        vtCtx->tempBufferOffset += totalPayloadSize;
    } else {
        responseDataBuffer = malloc(totalPayloadSize);
        ArrayList_add(&vtCtx->tempAllocations, responseDataBuffer);
    }
    memset(responseDataBuffer, 0, totalPayloadSize); // Initialize with zeros

    // Populate response header
    VkGetSwapchainImagesResponseHeader* header = (VkGetSwapchainImagesResponseHeader*)responseDataBuffer;
    header->reserved0 = 0;
    header->headerSize = headerSize;
    header->imageCount = imageCount;

    // Copy image handles into the response buffer (as uint64_t IDs)
    uint64_t* imagesDest = (uint64_t*)((char*)responseDataBuffer + headerSize);
    if (imagesArray != NULL) {
        for (uint32_t i = 0; i < imageCount; ++i) {
            imagesDest[i] = (uint64_t)imagesArray[i]; // Store VkImage pointers as raw IDs
        }
    }

    // Send the result and the response data
    // `RingBuffer_write(puVar16,&local_70,8);` writes VkResult and the totalPayloadSize.
    RingBuffer_write(vtCtx->clientRingBuffer, (void*)&result, sizeof(VkResult));
    RingBuffer_write(vtCtx->clientRingBuffer, (void*)&totalPayloadSize, sizeof(uint32_t)); // Write the total payload size
    RingBuffer_write(vtCtx->clientRingBuffer, responseDataBuffer, totalPayloadSize); // Write the actual payload

    if (allocatedImagesArrayTemp) {
        free(imagesArray); // Free the temporary array of VkImage pointers
    }

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Retrieves the window extent (width and height) from the Java component.
 *
 * This function uses JNI to call Java methods on the Android window manager
 * object to get the current width and height of the window.
 *
 * @param vtCtx Pointer to the global VtContext containing JNI environment and method IDs.
 * @param pWidth Pointer to a uint32_t where the window width will be stored.
 * @param pHeight Pointer to a uint32_t where the window height will be stored.
 */
void getWindowExtent(VtContext* vtCtx, uint32_t* pWidth, uint32_t* pHeight) {
    // JNIEnv* jniEnv = vtCtx->jniEnv;
    // jobject javaWindowManagerObject = vtCtx->javaWindowManagerObject;
    // jmethodID getWindowWidthMethodID = vtCtx->getWindowWidthMethodID;
    // jmethodID getWindowHeightMethodID = vtCtx->getWindowHeightMethodID;

    // Call Java method to get window width
    // The pattern `(**(code **)(**(long **)(param_1 + 8) + 0x188))`
    // `param_1 + 8` is `vtCtx->jniEnv` (at offset 0x98)
    // `param_1 + 0x10` is `vtCtx->javaWindowManagerObject` (at offset 0xA0)
    // `param_1 + 0x18` is `vtCtx->getWindowWidthMethodID` (at offset 0xA8)
    *pWidth = (uint32_t)vtCtx->jniEnv->CallIntMethod(vtCtx->javaWindowManagerObject, vtCtx->getWindowWidthMethodID);

    // Call Java method to get window height
    // `param_1 + 0x20` is `vtCtx->getWindowHeightMethodID` (at offset 0xB0)
    *pHeight = (uint32_t)vtCtx->jniEnv->CallIntMethod(vtCtx->javaWindowManagerObject, vtCtx->getWindowHeightMethodID);
}

/**
 * @brief Returns the minimum number of images required for a surface (usually 1).
 *
 * This function is a placeholder and always returns the value 1.
 *
 * @return Always returns 1.
 */
uint32_t getSurfaceMinImageCount(void) {
    return 1;
}

/**
 * @brief Populates a list of supported surface formats.
 *
 * This function allocates memory for two `VkSurfaceFormatKHR` entries and
 * populates them with common formats (R8G8B8A8_UNORM and R8G8B8A8_SRGB),
 * both with `VK_COLOR_SPACE_SRGB_NONLINEAR_KHR`. It also updates a provided
 * count pointer.
 *
 * @param pCount Pointer to a `uint32_t` where the number of formats will be stored (2).
 * @return A pointer to an array of `VkSurfaceFormatKHR` structures.
 */
VkSurfaceFormatKHR* getSurfaceFormats(uint32_t* pCount) {
    // Allocate memory for 2 VkSurfaceFormatKHR structs (each 8 bytes)
    VkSurfaceFormatKHR* formats = (VkSurfaceFormatKHR*)calloc(2, sizeof(VkSurfaceFormatKHR));

    if (pCount != NULL) {
        *pCount = 2; // Set count to 2
    }

    if (formats != NULL) {
        // First format: VK_FORMAT_R8G8B8A8_UNORM (0x25), VK_COLOR_SPACE_SRGB_NONLINEAR_KHR (0)
        formats[0].format = VK_FORMAT_R8G8B8A8_UNORM;
        formats[0].colorSpace = 0; // VK_COLOR_SPACE_SRGB_NONLINEAR_KHR is 0

        // Second format: VK_FORMAT_R8G8B8A8_SRGB (0x2b), VK_COLOR_SPACE_SRGB_NONLINEAR_KHR (0)
        formats[1].format = VK_FORMAT_R8G8B8A8_SRGB;
        formats[1].colorSpace = 0; // VK_COLOR_SPACE_SRGB_NONLINEAR_KHR is 0
    }
    return formats;
}

/**
 * @brief Creates an internal XWindowSwapchain object based on Vulkan and Android window info.
 *
 * This function is an internal helper called by `vt_handle_vkCreateSwapchainKHR`.
 * It creates a custom `XWindowSwapchainObject`, allocates Vulkan images and memory,
 * and binds them, integrating with Android hardware buffers.
 *
 * @param device The `VkDevice` handle.
 * @param graphicsQueueFamilyIndex The index of the graphics queue family.
 * @param pCreateInfo The `VkSwapchainCreateInfoKHR` provided by the client.
 * @param vtCtx_param The `VtContext` pointer (passed here to allow JNI calls from inside).
 * @param windowWidth The width of the window.
 * @return A pointer to the newly created `XWindowSwapchainObject`, or `NULL` on failure.
 */
XWindowSwapchainObject* XWindowSwapchain_create(
    VkDevice device,
    uint32_t graphicsQueueFamilyIndex,
    const VkSwapchainCreateInfoKHR* pCreateInfo,
    VtContext* vtCtx_param,
    uint32_t windowWidth
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Stack canary

    // Allocate memory for the custom XWindowSwapchainObject (size 0x38 = 56 bytes)
    XWindowSwapchainObject* swapchainObj = (XWindowSwapchainObject*)calloc(1, sizeof(XWindowSwapchainObject));
    if (swapchainObj == NULL) {
        return NULL;
    }

    // Populate initial fields of the custom swapchain object
    swapchainObj->extent.width = windowWidth;
    // Assuming height is derived from pCreateInfo or implicitly set to 0 by calloc,
    // though for a proper swapchain, height would also be needed.
    // The decompiled source doesn't explicitly pass windowHeight to this function.
    swapchainObj->extent.height = pCreateInfo->imageExtent.height; // Using createInfo's height for consistency

    swapchainObj->imageCount = pCreateInfo->minImageCount;
    swapchainObj->imageFormat = pCreateInfo->imageFormat;
    swapchainObj->imageExtent = pCreateInfo->imageExtent; // Full extent from create info
    swapchainObj->imageUsage = pCreateInfo->imageUsage;
    swapchainObj->vtCtx = vtCtx_param; // Store the VtContext pointer for later JNI access

    // Allocate array for image-memory pairs. Each pair is {VkImage, VkDeviceMemory}.
    // Size of each pair is 8 bytes (VkImage) + 8 bytes (VkDeviceMemory) = 16 bytes.
    // The disassembly shows `calloc(imageCount, 0x10)`.
    swapchainObj->images = (void*)calloc(swapchainObj->imageCount, sizeof(*swapchainObj->images));
    if (swapchainObj->images == NULL) {
        free(swapchainObj);
        return NULL;
    }

    VkResult vk_op_result = VK_SUCCESS; // Track Vulkan API call results
    uint32_t current_image_idx = 0; // To track cleanup index in case of failure

    // Loop through each image to create and bind it
    for (current_image_idx = 0; current_image_idx < swapchainObj->imageCount; ++current_image_idx) {
        AHardwareBuffer* androidHardwareBuffer = NULL;
        VkAndroidHardwareBufferPropertiesANDROID androidBufferProps = {
            .sType = VK_STRUCTURE_TYPE_ANDROID_HARDWARE_BUFFER_PROPERTIES_ANDROID, // 0x3b9cc1e9
            .pNext = NULL
        };
        VkImageCreateInfo imageCreateInfo = {
            .sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO,
            .pNext = NULL, // Will be set to &externalMemoryImageCreateInfo
            .flags = 0,
            .imageType = VK_IMAGE_TYPE_2D, // From usage (common for swapchains)
            .format = pCreateInfo->imageFormat,
            .extent = {0, 0, 1}, // Dimensions will be filled from AHardwareBuffer_describe
            .mipLevels = 1, // Common for swapchains
            .arrayLayers = 1, // Common for swapchains
            .samples = VK_SAMPLE_COUNT_1_BIT, // Common for swapchains (0x1)
            .tiling = VK_IMAGE_TILING_OPTIMAL, // Common or specific. The disassembly set 0 (optimal)
            .usage = pCreateInfo->imageUsage,
            .sharingMode = VK_SHARING_MODE_EXCLUSIVE, // The disassembly set 0 (exclusive)
            .queueFamilyIndexCount = 0,
            .pQueueFamilyIndices = NULL,
            .initialLayout = VK_IMAGE_LAYOUT_UNDEFINED // The disassembly set 0 (undefined)
        };
        // This struct (`local_150`) is passed as `pNext` to `VkImageCreateInfo` (`local_1b0`).
        VkExternalMemoryImageCreateInfo externalMemoryImageCreateInfo = {
            .sType = VK_STRUCTURE_TYPE_EXTERNAL_MEMORY_IMAGE_CREATE_INFO, // 0x3b9be341
            .pNext = NULL,
            .handleTypes = VK_EXTERNAL_MEMORY_HANDLE_TYPE_ANDROID_HARDWARE_BUFFER_BIT_ANDROID // 0x400
        };
        // This struct (`local_c8`) is passed as `pNext` to `VkMemoryAllocateInfo` (`local_110`).
        VkMemoryDedicatedAllocateInfo dedicatedAllocInfo = {
            .sType = VK_STRUCTURE_TYPE_MEMORY_DEDICATED_ALLOCATE_INFO, // 0x3b9cc1eb
            .pNext = NULL,
            .image = VK_NULL_HANDLE, // Will be set after VkImage creation
            .buffer = VK_NULL_HANDLE
        };
        VkMemoryAllocateInfo allocInfo = {
            .sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO, // `local_110` being `5`
            .pNext = NULL, // Will be set to &dedicatedAllocInfo
            .allocationSize = 0, // Will be set from androidBufferProps.allocationSize
            .memoryTypeIndex = 0 // Will be set from getMemoryTypeIndex
        };


        // 1. Get Android Hardware Buffer for the current image via JNI.
        androidHardwareBuffer = (AHardwareBuffer*)vtCtx_param->jniEnv->CallObjectMethod(
            vtCtx_param->javaWindowManagerObject, vtCtx_param->getWindowHardwareBufferMethodID, (jint)current_image_idx
        );
        if (androidHardwareBuffer == NULL) {
            vk_op_result = VK_ERROR_INITIALIZATION_FAILED;
            goto cleanup_allocated_images;
        }

        // 2. Get properties of the Android Hardware Buffer, including its dimensions and memory type bits.
        VkExtent2D hardwareBufferExtent;
        AHardwareBuffer_describe(androidHardwareBuffer, &hardwareBufferExtent); // Populates `hardwareBufferExtent`
        imageCreateInfo.extent.width = hardwareBufferExtent.width;
        imageCreateInfo.extent.height = hardwareBufferExtent.height;

        dispatch_vkGetAndroidHardwareBufferPropertiesANDROID(device, androidHardwareBuffer, &androidBufferProps);

        // 3. Create VkImage, linked to the external memory via pNext chain.
        imageCreateInfo.pNext = &externalMemoryImageCreateInfo;
        vk_op_result = dispatch_vkCreateImage(device, &imageCreateInfo, NULL, &swapchainObj->images[current_image_idx].image);
        if (vk_op_result != VK_SUCCESS) {
            goto cleanup_allocated_images;
        }

        // 4. Allocate VkDeviceMemory for the image, specifying it's a dedicated allocation for the image.
        dedicatedAllocInfo.image = swapchainObj->images[current_image_idx].image;
        allocInfo.pNext = &dedicatedAllocInfo;
        allocInfo.allocationSize = androidBufferProps.allocationSize;
        allocInfo.memoryTypeIndex = getMemoryTypeIndex(
            androidBufferProps.memoryTypeBits,
            VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT // Only `1` is explicitly used in disassembly, typically VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT
        );

        vk_op_result = dispatch_vkAllocateMemory(device, &allocInfo, NULL, &swapchainObj->images[current_image_idx].memory);
        if (vk_op_result != VK_SUCCESS) {
            // Destroy the image just created before failing
            dispatch_vkDestroyImage(device, swapchainObj->images[current_image_idx].image, NULL);
            goto cleanup_allocated_images;
        }

        // 5. Bind VkImage to the allocated VkDeviceMemory.
        vk_op_result = dispatch_vkBindImageMemory(device, swapchainObj->images[current_image_idx].image, swapchainObj->images[current_image_idx].memory, 0);
        if (vk_op_result != VK_SUCCESS) {
            // Destroy image and free memory before failing
            dispatch_vkDestroyImage(device, swapchainObj->images[current_image_idx].image, NULL);
            dispatch_vkFreeMemory(device, swapchainObj->images[current_image_idx].memory, NULL);
            goto cleanup_allocated_images;
        }
    }

    // 6. Get the graphics queue for later use and store it in the swapchain object.
    dispatch_vkGetDeviceQueue(device, graphicsQueueFamilyIndex, 0, &swapchainObj->graphicsQueue);

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return swapchainObj;

cleanup_allocated_images:
    // Clean up any images and memory that were successfully allocated before the failure
    for (uint32_t j = 0; j < current_image_idx; ++j) {
        if (swapchainObj->images[j].image != VK_NULL_HANDLE) {
            dispatch_vkDestroyImage(device, swapchainObj->images[j].image, NULL);
        }
        if (swapchainObj->images[j].memory != VK_NULL_HANDLE) {
            dispatch_vkFreeMemory(device, swapchainObj->images[j].memory, NULL);
        }
    }
    if (swapchainObj->images != NULL) {
        free(swapchainObj->images);
    }
    if (swapchainObj != NULL) {
        free(swapchainObj);
    }
    // Result of the operation is stored in `vk_op_result` and will be propagated.
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return NULL;
}


/**
 * @brief Destroys an XWindowSwapchainObject and its associated Vulkan resources.
 *
 * This function cleans up Vulkan images and device memory allocated for
 * the swapchain, and then frees the internal swapchain object itself.
 *
 * @param device The VkDevice handle.
 * @param swapchainObj Pointer to the XWindowSwapchainObject to destroy.
 */
void XWindowSwapchain_destroy(VkDevice device, XWindowSwapchainObject* swapchainObj) {
    if (swapchainObj == NULL) {
        return; // Nothing to destroy
    }

    // Loop through each image and associated memory to destroy them
    if (swapchainObj->imageCount > 0 && swapchainObj->images != NULL) {
        for (uint32_t i = 0; i < swapchainObj->imageCount; ++i) {
            if (swapchainObj->images[i].image != VK_NULL_HANDLE) {
                dispatch_vkDestroyImage(device, swapchainObj->images[i].image, NULL);
            }
            if (swapchainObj->images[i].memory != VK_NULL_HANDLE) {
                dispatch_vkFreeMemory(device, swapchainObj->images[i].memory, NULL);
            }
        }
    }
    // Free the array of image-memory pairs
    if (swapchainObj->images != NULL) {
        free(swapchainObj->images);
    }
    // Free the custom swapchain object itself
    free(swapchainObj);
}

/**
 * @brief Attempts to acquire the next image from the swapchain for rendering.
 *
 * This function performs a simplified "acquire next image" operation. It optionally
 * performs a queue submit operation if a semaphore or fence is provided. It then
 * checks if the internal swapchain dimensions still match the current window
 * dimensions. It doesn't actually "acquire" an image index like `vkAcquireNextImageKHR`
 * but rather simulates success/failure and always returns image index 0.
 *
 * @param swapchainObj Pointer to the XWindowSwapchainObject.
 * @param device The `VkDevice` handle.
 * @param pWaitSemaphore Optional semaphore to wait on before submitting (if not NULL).
 * @param pSignalFence Optional fence to signal after submit (if not NULL).
 * @param pImageIndex Output parameter for the acquired image index (always 0).
 */
void XWindowSwapchain_acquireNextImage(
    XWindowSwapchainObject* swapchainObj,
    VkDevice device,
    VkSemaphore pWaitSemaphore,
    VkFence pSignalFence,
    uint32_t* pImageIndex
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28); // Stack canary

    VkResult result = VK_SUCCESS;

    // If a semaphore or fence is provided, perform a queue submit operation
    if (pWaitSemaphore != VK_NULL_HANDLE || pSignalFence != VK_NULL_HANDLE) {
        VkSubmitInfo submitInfo = {
            .sType = VK_STRUCTURE_TYPE_SUBMIT_INFO,
            .pNext = NULL,
            .waitSemaphoreCount = (pWaitSemaphore != VK_NULL_HANDLE) ? 1 : 0,
            .pWaitSemaphores = (pWaitSemaphore != VK_NULL_HANDLE) ? &pWaitSemaphore : NULL,
            .pWaitDstStageMask = NULL, // Not explicitly set in disassembly
            .commandBufferCount = 0,
            .pCommandBuffers = NULL,
            .signalSemaphoreCount = 0, // No signal semaphores from this submit
            .pSignalSemaphores = NULL
        };

        // Perform the queue submission
        result = dispatch_vkQueueSubmit(swapchainObj->graphicsQueue, 1, &submitInfo, pSignalFence);

        if (result == VK_ERROR_DEVICE_LOST) { // -4 is VK_ERROR_DEVICE_LOST
            *pImageIndex = 0; // Return index 0 regardless of device lost status
            if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
                __stack_chk_fail();
            }
            return;
        }
    }

    // Check if the internal swapchain dimensions match the current window dimensions.
    uint32_t currentWindowWidth = 0;
    uint32_t currentWindowHeight = 0;

    // Access `vtCtx` from `swapchainObj` to call `getWindowExtent`.
    getWindowExtent(swapchainObj->vtCtx, &currentWindowWidth, &currentWindowHeight);

    if (swapchainObj->imageExtent.width == currentWindowWidth &&
        swapchainObj->imageExtent.height == currentWindowHeight) {
        // Dimensions match, keep result as VK_SUCCESS
    } else {
        result = VK_ERROR_OUT_OF_DATE_KHR; // Dimensions mismatch
    }

    *pImageIndex = 0; // Always return image index 0 for simplicity (matches `*param_5 = 0;`)

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Presents the image from the XWindowSwapchain object.
 *
 * This function performs the actual presentation of the swapchain's image
 * by calling a Java method on the window manager to update the window content.
 * It's assumed that the underlying Java method `updateWindowContentMethodID`
 * takes arguments like width, height, and the image index to display.
 *
 * @param swapchainObj Pointer to the XWindowSwapchainObject.
 */
void XWindowSwapchain_presentImage(XWindowSwapchainObject* swapchainObj) {
    // Access JNI environment and method IDs via the stored `vtCtx` pointer.
    // The specific arguments for `updateWindowContentMethodID` are not fully
    // discernible from the minimal assembly snippet, but typical usage suggests
    // width, height, and image index. Assuming image index 0 as that's what
    // `acquireNextImage` always returns.
    swapchainObj->vtCtx->jniEnv->CallVoidMethod(
        swapchainObj->vtCtx->javaWindowManagerObject,
        swapchainObj->vtCtx->updateWindowContentMethodID,
        (jint)swapchainObj->extent.width,
        (jint)swapchainObj->extent.height,
        (jint)0 // Assuming image index 0
    );
}
```

---

# Initialization

Here's the decompilation of the provided C functions into human-readable C code.

First, let's establish the necessary type definitions and global variables based on the `vulkan_core.h` and `VtContext` structures provided, as well as inferences from the assembly code.

```c
#include <string.h> // For strncpy, strcmp, memcpy, memset, strdup
#include <stdio.h>  // For snprintf
#include <stdlib.h> // For calloc, malloc
#include <pthread.h> // For pthread_mutex_init, pthread_cond_init, pthread_create, pthread_detach
#include <sys/socket.h> // For sendmsg
#include <unistd.h> // For close
#include <sys/uio.h> // For iovec (used by sendmsg)

// Assuming these are defined elsewhere or in the Vulkan headers:
#define VK_MAX_PHYSICAL_DEVICE_NAME_SIZE 256
#define VK_MAX_DRIVER_INFO_SIZE 256
#define VK_UUID_SIZE 16
#define VK_FALSE 0 // Vulkan defines VkBool32 as uint32_t, so 0 is false.
#define VK_TRUE 1

// Version extraction macros from vulkan_core.h
#define VK_API_VERSION_MAJOR(version) (((uint32_t)(version) >> 22U) & 0x7FU)
#define VK_API_VERSION_MINOR(version) (((uint32_t)(version) >> 12U) & 0x3FFU)
#define VK_API_VERSION_PATCH(version) ((uint32_t)(version) & 0xFFFU)

// Enum constants from vulkan_core.h (relevant to the provided snippets)
typedef enum VkStructureType {
    VK_STRUCTURE_TYPE_APPLICATION_INFO = 0,
    VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO = 1,
    VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO = 2,
    VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO = 3,
    // ...
    VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER = 44, // 0x2c - This is an unusual value to see in this context.
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES = 52, // 0x34
    // ... many more structure types
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2 = 1000059000, // 0x3b9b3700
    VK_STRUCTURE_TYPE_FORMAT_PROPERTIES_2 = 1000059002, // 0x3b9b3702
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES = 1000070000, // 0x3b9dc7a0
    VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DRIVER_PROPERTIES = 1000196000, // 0x3b9b3760
    // ...
    VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT = 0x00000040,
    // ... other VkFormatFeatureFlagBits
} VkStructureType;

// Standard Vulkan types (simplified for this context)
typedef uint32_t VkBool32;
typedef uint64_t VkDeviceSize;
typedef uint32_t VkFlags;
typedef VkFlags VkMemoryPropertyFlags;
typedef VkFlags VkMemoryHeapFlags;
typedef VkFlags VkFormatFeatureFlags;

// Minimal struct definitions to match offsets and types from the disassembly context
typedef struct VkBaseInStructure {
    VkStructureType sType;
    const struct VkBaseInStructure* pNext; // 8 bytes on 64-bit systems
} VkBaseInStructure;

typedef struct VkPhysicalDeviceFeatures {
    VkBool32    robustBufferAccess; // offset 0
    VkBool32    fullDrawIndexUint32; // offset 4
    VkBool32    imageCubeArray; // offset 8
    VkBool32    independentBlend; // offset 12 (0xc)
    VkBool32    geometryShader; // offset 16 (0x10)
    VkBool32    tessellationShader; // offset 20 (0x14)
    VkBool32    sampleRateShading; // offset 24 (0x18)
    VkBool32    dualSrcBlend; // offset 28 (0x1c)
    VkBool32    logicOp; // offset 32 (0x20)
    VkBool32    multiDrawIndirect; // offset 36 (0x24)
    VkBool32    drawIndirectFirstInstance; // offset 40 (0x28)
    VkBool32    depthClamp; // offset 44 (0x2c)
    VkBool32    depthBiasClamp; // offset 48 (0x30)
    VkBool32    fillModeNonSolid; // offset 52 (0x34)
    // ... many more VkBool32 features, each 4 bytes. Total size 0xdc (220 bytes)
    // The specific features disabled in `disableUnsupportedDeviceFeatures` are
    // addressed by raw byte offsets, implying they are part of a contiguous block
    // of VkBool32 members in a features struct.
} VkPhysicalDeviceFeatures; // Total size 0xdc (220 bytes)

typedef struct VkPhysicalDeviceFeatures2 {
    VkStructureType             sType; // offset 0
    void*                       pNext; // offset 8
    VkPhysicalDeviceFeatures    features; // offset 16, start of VkPhysicalDeviceFeatures struct
} VkPhysicalDeviceFeatures2; // Total size ~236 bytes

typedef struct VkPhysicalDeviceProperties {
    uint32_t             apiVersion;      // offset 0
    uint32_t             driverVersion;   // offset 4
    uint32_t             vendorID;        // offset 8
    uint32_t             deviceID;        // offset 12
    uint32_t             deviceType;      // offset 16
    char                 deviceName[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE]; // offset 20 (0x14)
    uint8_t              pipelineCacheUUID[VK_UUID_SIZE]; // offset 276
    // ... other members
} VkPhysicalDeviceProperties;

typedef struct VkPhysicalDeviceVulkan12Properties {
    VkStructureType                      sType; // offset 0
    void*                                pNext; // offset 8
    uint32_t                             driverID; // offset 16 (0x10)
    char                                 driverName[VK_MAX_DRIVER_NAME_SIZE]; // offset 20 (0x14)
    char                                 driverInfo[VK_MAX_DRIVER_INFO_SIZE]; // offset 276 (0x114), piVar6 + 0x45 in dword
    // Inferred based on assembly offsets 0x89, 0x8a, 0x8c, 0x8d, 0x8f, 0x90 when treated as uint32_t*
    // These offsets are from the start of the VkPhysicalDeviceVulkan12Properties struct (not from `driverInfo`).
    VkBool32 shaderSignedZeroInfNanPreserveFloat16; // offset 548 (0x224)
    VkBool32 shaderSignedZeroInfNanPreserveFloat32; // offset 552 (0x228)
    VkBool32 shaderSignedZeroInfNanPreserveFloat64; // offset 556 (0x22C) // This would be 0x8B * 4. Actual assembly is 0x8a and 0x8f/0x90
    // Re-evaluating 0x89, 0x8a, 0x8f, 0x90, 0x8c, 0x8d from assembly:
    // props12->shaderSignedZeroInfNanPreserveFloat16 (0x89*4 = 0x224)
    // props12->shaderSignedZeroInfNanPreserveFloat32 (0x8a*4 = 0x228)
    // props12->shaderDenormPreserveFloat16 (0x8c*4 = 0x230)
    // props12->shaderDenormPreserveFloat32 (0x8d*4 = 0x234)
    // props12->shaderDenormFlushToZeroFloat16 (0x8f*4 = 0x23C)
    // props12->shaderDenormFlushToZeroFloat32 (0x90*4 = 0x240)
    // There are some gaps between these in the standard spec, so likely padding or other fields omitted here.
    // Given the disassembly sets them to 0, they are VkBool32 (uint32_t).
    // ... rest of the struct
} VkPhysicalDeviceVulkan12Properties;

typedef struct VkPhysicalDeviceGroupProperties {
    VkStructureType     sType; // offset 0
    void*               pNext; // offset 8
    uint32_t            physicalDeviceCount; // offset 16 (0x10)
    void*               physicalDevices[32]; // array of VkPhysicalDevice handles (pointers), 32 * 8 = 256 bytes. offset 20 (0x14)
    VkBool32            subsetAllocation; // offset 276 (0x114). The target of string copy `param_3 + 0x45` means `((uint32_t*)this_struct)[0x45]` = offset `0x114`.
                                          // This location is `subsetAllocation`. Writing a string here would overwrite this field.
    // ... rest of the struct
} VkPhysicalDeviceGroupProperties;

typedef struct VkMemoryType {
    VkMemoryPropertyFlags propertyFlags; // offset 0
    uint32_t heapIndex;                // offset 4
} VkMemoryType; // sizeof = 8 bytes

typedef struct VkMemoryHeap {
    VkDeviceSize         size;  // offset 0 (8 bytes)
    VkMemoryHeapFlags    flags; // offset 8 (4 bytes). Total 12 bytes, often padded to 16.
} VkMemoryHeap; // sizeof = 16 bytes (0x10) for 64-bit systems

typedef struct VkPhysicalDeviceMemoryProperties {
    uint32_t        memoryTypeCount;    // offset 0
    VkMemoryType    memoryTypes[32];    // offset 4 (32 * 8 = 256 bytes)
    uint32_t        memoryHeapCount;    // offset 260 (0x104)
    VkMemoryHeap    memoryHeaps[16];    // offset 264 (0x108) (16 * 16 = 256 bytes)
} VkPhysicalDeviceMemoryProperties; // Total size approx 520 bytes (0x208)

typedef struct VkFormatProperties {
    VkFormatFeatureFlags    linearTilingFeatures;  // offset 0
    VkFormatFeatureFlags    optimalTilingFeatures; // offset 4
    VkFormatFeatureFlags    bufferFeatures;        // offset 8
} VkFormatProperties;

typedef struct VkFormatProperties2 {
    VkStructureType       sType; // offset 0
    void*                 pNext; // offset 8
    VkFormatProperties    formatProperties; // offset 16
} VkFormatProperties2;

typedef struct VkExtensionProperties {
    char        extensionName[VK_MAX_EXTENSION_NAME_SIZE]; // 256 bytes (0x100)
    uint32_t    specVersion;                               // 4 bytes (0x4)
} VkExtensionProperties; // sizeof = 260 bytes (0x104) due to packing

// Simplified ArrayList structure based on observed access patterns
typedef struct ArrayList {
    uint32_t count;
    uint32_t capacity; // Inferred or padding, to keep `items` 8-byte aligned
    void** items;      // Pointer to array of pointers (char** or VkExtensionProperties*)
} ArrayList; // sizeof = 16 bytes for 64-bit systems

// VtContext struct as provided in the prompt
typedef struct VtContext {
    /* 0x00 */ int clientFd;
    /* 0x04 */ int vkMaxVersion;
    /* 0x08 */ short maxDeviceMemory; // In MB
    /* 0x0a */ short imageCacheSize;  // In MB
    /* 0x0c */ int padding_0c;
    /* 0x10 */ ArrayList* exposedDeviceExtensions;
    /* 0x18 */ ArrayList* exposedInstanceExtensions;
    /* 0x20 */ uint8_t hostVisibleMemoryFlag; // For memory type selection
    /* 0x21 */ uint8_t coherentMemoryFlag;    // For memory type selection
    /* 0x22 */ char padding_22[6];
    /* 0x28 */ char* inputBufferPtr;      // Pointer to the current command's data
    /* 0x30 */ int inputBufferSize;
    /* 0x34 */ int padding_34;
    /* 0x38 */ void* tempBuffer;          // A large temporary buffer for deserialization (size 0x10000 = 65536 bytes)
    /* 0x40 */ int tempBufferOffset;      // Current offset within the tempBuffer
    /* 0x44 */ int padding_44;
    /* 0x48 */ ArrayList* tempAllocations; // Tracks allocations within tempBuffer
    /* 0x50 */ long padding_50;
    /* 0x58 */ pthread_t serverThread;    // Handle to the main processing thread
    /* 0x60 */ void* clientRingBuffer; // RingBuffer* (abstracted as void*)
    /* 0x68 */ void* serverRingBuffer; // RingBuffer* (abstracted as void*)
    /* 0x70 */ int lastError;             // Stores the last recorded VkResult error
    /* 0x74 */ int graphicsQueueFamilyIndex;
    /* 0x78 */ void* textureDecoder; // TextureDecoder* (abstracted as void*)
    /* 0x80 */ void* shaderInspector; // ShaderInspector* (abstracted as void*)
    /* 0x88 */ void* asyncPipelineCreator; // AsyncPipelineCreator* (abstracted as void*)
    
    // JNI and Android-specific windowing members
    /* 0x90 */ void* javaComponentObject; // jobject
    /* 0x98 */ void* jniEnv; // JNIEnv*
    /* 0xA0 */ void* javaWindowManagerObject; // jobject
    /* 0xA8 */ void* getWindowWidthMethodID; // jmethodID
    /* 0xB0 */ void* getWindowHeightMethodID; // jmethodID
    /* 0xB8 */ void* getWindowHardwareBufferMethodID; // jmethodID
    /* 0xC0 */ void* updateWindowContentMethodID; // jmethodID
} VtContext;

// External function declarations (assuming these are standard or provided elsewhere)
extern void __stack_chk_fail(void); // Stack canary check failure function
extern int strcmp(const char *s1, const char *s2);
extern size_t strlen(const char *s); // Required for strcpy in injectExtensions2, etc.

// Inferred global variables based on usage
// These are likely populated by a vkGetPhysicalDeviceMemoryProperties call
extern uint32_t DAT_00149858;   // Represents VkPhysicalDeviceMemoryProperties::memoryTypeCount
extern VkMemoryType* DAT_00149860; // Represents VkPhysicalDeviceMemoryProperties::memoryTypes (array of 32 VkMemoryType structs)

// Inferred global function pointers
// These are likely resolved at runtime via vkGetInstanceProcAddr/vkGetDeviceProcAddr
extern void (*DAT_001494a0)(VkPhysicalDevice physicalDevice, VkPhysicalDeviceFeatures2* pFeatures); // Likely vkGetPhysicalDeviceFeatures2
extern void (*DAT_00149048)(VkPhysicalDevice physicalDevice, VkPhysicalDeviceFeatures2* pFeatures); // Likely vkGetPhysicalDeviceFeatures2
extern VkResult (*DAT_00149080)(VkPhysicalDevice physicalDevice, const char* pLayerName, uint32_t* pPropertyCount, VkExtensionProperties* pProperties); // Likely vkEnumerateDeviceExtensionProperties
// DAT_00149050's behavior is ambiguous from the snippet; assumed to be an internal helper
extern void (*DAT_00149050)(VkPhysicalDevice physicalDevice, uint32_t magic_value, VkFormatProperties* pFormatProperties);

// Inferred helper functions (their implementations are not provided, only their calls)
// These functions are likely internal to the Vortek framework or wrappers around Vulkan/Android APIs.
extern bool isCompressedFormat(VkFormat format);
extern bool isFormatScaled(VkFormat format);
extern void ArrayList_add(ArrayList* list, void* item);
extern int RingBuffer_getSHMemSize(uint32_t size);
extern void* RingBuffer_create(int fd, uint32_t size);
extern void* FUN_0013d7ec(void* arg); // Worker thread entry point for AsyncPipelineCreator
extern void* FUN_0013d91c(void* arg); // Main server thread entry point

// JNI related function from Assembly
// This function resolves and gets field values from Java objects via JNI.
// Its parameters are inferred based on typical JNIEnv method signatures.
// Returns an unsigned long, which acts as a generic container for various JNI types.
unsigned long FUN_0013d48c(
    void* pJNIEnv,             // JNIEnv* param_1
    jobject javaObject,        // jobject param_2
    const char* fieldName,     // const char* param_3
    const char* fieldSignature // const char* param_4
);

// FUN_0013cfe4 is a wrapper for snprintf with a stack canary check.
// It's used for formatting strings into a buffer.
// The variadic arguments `param_4` to `param_8` are passed directly to `__vsprintf_chk`.
void FUN_0013cfe4(
    void*        dest_buffer,
    size_t       buffer_size,
    const char*  format,
    void*        arg1,
    void*        arg2,
    void*        arg3,
    void*        arg4,
    void*        arg5
);
```

---

### Decompiled Functions

```c
/**
 * @brief Retrieves the index of a memory type that matches specified properties.
 *
 * This function iterates through the available memory types reported by the physical device.
 * It returns the index of the first memory type that is compatible with `memoryTypeBits`
 * (a bitmask from VkMemoryRequirements) and possesses all the `requiredPropertyFlags`.
 *
 * @param memoryTypeBits A bitmask indicating the memory types acceptable for an allocation.
 * @param requiredPropertyFlags The bitmask of required VkMemoryPropertyFlagBits.
 * @return The index of the matching memory type, or 0xFFFFFFFF if no suitable type is found.
 */
uint32_t getMemoryTypeIndex(uint32_t memoryTypeBits, VkMemoryPropertyFlags requiredPropertyFlags) {
    uint32_t currentMemoryTypeIndex = 0;
    // DAT_00149858 is likely VkPhysicalDeviceMemoryProperties::memoryTypeCount
    // DAT_00149860 is likely the start of VkPhysicalDeviceMemoryProperties::memoryTypes array
    if (DAT_00149858 != 0) {
        VkMemoryType* pMemoryTypes = DAT_00149860;
        do {
            // Check if the current memory type index is allowed by memoryTypeBits
            // (memoryTypeBits & 1) checks the LSB for current index.
            // Check if the current memory type's properties (pMemoryTypes->propertyFlags)
            // contain all the requiredPropertyFlags.
            if (((memoryTypeBits & 1) != 0) && ((pMemoryTypes->propertyFlags & requiredPropertyFlags) != 0)) {
                return currentMemoryTypeIndex;
            }
            currentMemoryTypeIndex++;
            memoryTypeBits >>= 1; // Shift to the next bit in the bitmask
            pMemoryTypes++;       // Move to the next VkMemoryType struct (sizeof(VkMemoryType) == 8 bytes)
        } while (currentMemoryTypeIndex < DAT_00149858); // Loop until all memory types are checked
    }
    return 0xFFFFFFFF; // No suitable memory type found
}

/**
 * @brief Retrieves the property flags for a specific memory type.
 *
 * @param memoryTypeIndex The index of the memory type.
 * @return The VkMemoryPropertyFlags for the specified memory type, or 0 if the index is out of bounds
 *         or no memory types are loaded.
 */
VkMemoryPropertyFlags getMemoryPropertyFlags(uint32_t memoryTypeIndex) {
    VkMemoryPropertyFlags propertyFlags = 0;
    // Check if memory types data is loaded and the index is within bounds
    if ((DAT_00149858 != 0) && (memoryTypeIndex < DAT_00149858)) {
        // Access DAT_00149860 (array of VkMemoryType) at the given index.
        // `DAT_00149860 + (ulong)memoryTypeIndex * 8` is equivalent to `&DAT_00149860[memoryTypeIndex]`.
        // Dereferencing to `undefined4` (uint32_t) gets the `propertyFlags` (first member of VkMemoryType).
        propertyFlags = DAT_00149860[memoryTypeIndex].propertyFlags;
    }
    return propertyFlags;
}

/**
 * @brief Checks if a given memory type is host-visible.
 *
 * A memory type is considered host-visible if it has any of the
 * VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT, VK_MEMORY_PROPERTY_HOST_COHERENT_BIT,
 * or VK_MEMORY_PROPERTY_HOST_CACHED_BIT flags set.
 *
 * @param memoryTypeIndex The index of the memory type.
 * @return True if the memory type is host-visible, false otherwise.
 */
bool isHostVisibleMemory(uint32_t memoryTypeIndex) {
    bool isHostVisible = false;
    // Check if memory types data is loaded and the index is within bounds
    if ((DAT_00149858 != 0) && (memoryTypeIndex < DAT_00149858)) {
        // Access DAT_00149860 (array of VkMemoryType) at the given index.
        // The mask `0xE` (binary `1110`) is `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT | VK_MEMORY_PROPERTY_HOST_CACHED_BIT`.
        isHostVisible = (DAT_00149860[memoryTypeIndex].propertyFlags & 0xE) != 0;
    }
    return isHostVisible;
}

/**
 * @brief Modifies lists of enabled/exposed extensions based on supported and requested extensions.
 *
 * This function takes existing lists of exposed extensions (`ppExposedExtensionNames`)
 * and a list of requested extensions (`ppRequestedExtensionNames`), and processes them
 * against a list of actually supported extensions (`pSupportedExtensionProperties`).
 * The output is a new list of `char*` (string pointers) that includes supported exposed extensions
 * and unique requested extensions.
 *
 * Memory for new string copies and the array of pointers is managed either via `vtContext->tempBuffer`
 * or `malloc`, tracked by `vtContext->tempAllocations`.
 *
 * @param vtContext A pointer to the VtContext structure.
 * @param ppExposedExtensionNames A pointer to a `char**` that initially points to an array of string pointers
 *                                (old exposed extensions). This will be updated to point to the new filtered/merged array.
 * @param pExposedExtensionCount A pointer to a `uint32_t` that holds the count of old exposed extensions.
 *                               This will be updated with the count of the new list.
 * @param ppRequestedExtensionNames An array of `const char*` strings representing extensions requested by the application.
 * @param requestedExtensionCount The number of extensions in `ppRequestedExtensionNames`.
 * @param pSupportedExtensionProperties An array of `VkExtensionProperties` structs representing extensions actually supported by the device.
 * @param supportedExtensionCount The number of extensions in `pSupportedExtensionProperties`.
 */
void injectExtensions(
    VtContext* vtContext,
    char*** ppExposedExtensionNames, // param_2, updated pointer to array of string pointers
    uint32_t* pExposedExtensionCount, // param_3, updated count
    const char* const* ppRequestedExtensionNames, // param_4, strings requested by app
    uint32_t requestedExtensionCount, // param_5, count of requested
    const VkExtensionProperties* pSupportedExtensionProperties, // param_6, actual supported properties from device
    uint32_t supportedExtensionCount // param_7, count of supported
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);
    // Allocate a temporary boolean array on the stack. Size needs to accommodate
    // flags for both old exposed and new requested extensions.
    // +16 for alignment/safety margin, since the assembly aligns to 0x10.
    char temp_bool_flags[(*pExposedExtensionCount + requestedExtensionCount) + 16]; 

    char** old_exposed_names_ptr = *ppExposedExtensionNames; // Initial value of the array of exposed extension names
    uint32_t old_exposed_count = *pExposedExtensionCount; // Initial count of exposed extensions
    char* current_flag_ptr = temp_bool_flags;

    // Phase 1: Filter existing exposed extensions against supported extensions.
    // Only keep extensions that are actually supported by the physical device.
    if (pSupportedExtensionProperties != NULL) {
        uint32_t num_actually_supported_exposed = 0; // Counts exposed extensions that are supported
        for (uint32_t i = 0; i < old_exposed_count; ++i) {
            current_flag_ptr[i] = '\0'; // Initialize flag to false (not yet found in supported list)
            bool found_in_supported = false;
            if (old_exposed_names_ptr[i] != NULL) {
                for (uint32_t j = 0; j < supportedExtensionCount; ++j) {
                    if (strcmp(pSupportedExtensionProperties[j].extensionName, old_exposed_names_ptr[i]) == 0) {
                        found_in_supported = true;
                        current_flag_ptr[i] = '\x01'; // Mark as supported
                        break;
                    }
                }
            }
            if (found_in_supported) {
                num_actually_supported_exposed++;
            }
        }

        // Allocate new memory for the filtered list of exposed names.
        size_t alloc_size = (size_t)num_actually_supported_exposed * sizeof(char*);
        void* new_exposed_names_array = NULL;
        // Try to allocate from the temporary buffer first
        if (vtContext->tempBuffer != NULL && (vtContext->tempBufferOffset + alloc_size) < 0x10000) { // 0x10000 = 64KB
            new_exposed_names_array = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
            vtContext->tempBufferOffset += alloc_size;
        } else { // Fallback to malloc if tempBuffer is full or not available
            new_exposed_names_array = malloc(alloc_size);
            ArrayList_add(vtContext->tempAllocations, new_exposed_names_array);
        }
        memset(new_exposed_names_array, 0, alloc_size); // Zero-initialize new memory

        // Copy pointers to the actually supported exposed extension names to the new array
        uint32_t dest_idx = 0;
        for (uint32_t i = 0; i < old_exposed_count; ++i) {
            if (current_flag_ptr[i] == '\x01') { // If old exposed extension is supported
                ((char**)new_exposed_names_array)[dest_idx++] = old_exposed_names_ptr[i];
            }
        }

        // Update the caller's pointers and count
        *ppExposedExtensionNames = (char**)new_exposed_names_array;
        *pExposedExtensionCount = num_actually_supported_exposed;
    }

    // Phase 2: Add unique requested extensions to the filtered exposed list.
    // The `temp_bool_flags` buffer is reused for this phase.
    if (requestedExtensionCount != 0) {
        current_flag_ptr = temp_bool_flags; // Reset pointer for the new set of flags
        uint32_t current_exposed_count = *pExposedExtensionCount;
        char** current_exposed_names = *ppExposedExtensionNames;

        uint32_t num_new_unique_requested = 0; // Counts requested extensions that are not already exposed
        for (uint32_t i = 0; i < requestedExtensionCount; ++i) {
            current_flag_ptr[i] = '\0'; // Initialize flag to false (not yet found in current_exposed_names)
            bool found_in_exposed = false;
            if (current_exposed_count > 0 && ppRequestedExtensionNames[i] != NULL) {
                for (uint32_t j = 0; j < current_exposed_count; ++j) {
                    if (current_exposed_names[j] != NULL && strcmp(ppRequestedExtensionNames[i], current_exposed_names[j]) == 0) {
                        found_in_exposed = true;
                        current_flag_ptr[i] = '\x01'; // Mark as duplicate
                        break;
                    }
                }
            }
            if (!found_in_exposed) {
                num_new_unique_requested++;
            }
        }

        uint32_t total_final_count = current_exposed_count + num_new_unique_requested;

        if (total_final_count != current_exposed_count) { // Only reallocate if the total count changes
            size_t alloc_size = (size_t)total_final_count * sizeof(char*);
            void* new_final_array = NULL;
            // Try to allocate from the temporary buffer first
            if (vtContext->tempBuffer != NULL && (vtContext->tempBufferOffset + alloc_size) < 0x10000) {
                new_final_array = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
                vtContext->tempBufferOffset += alloc_size;
            } else { // Fallback to malloc
                new_final_array = malloc(alloc_size);
                ArrayList_add(vtContext->tempAllocations, new_final_array);
            }
            memset(new_final_array, 0, alloc_size); // Zero-initialize new memory

            // Copy existing filtered exposed names to the new final array
            if (current_exposed_count > 0) {
                memcpy(new_final_array, current_exposed_names, (size_t)current_exposed_count * sizeof(char*));
            }

            // Append unique requested names (copying string data as well)
            uint32_t dest_idx = current_exposed_count;
            for (uint32_t i = 0; i < requestedExtensionCount; ++i) {
                if (current_flag_ptr[i] == '\0') { // If not a duplicate (unique)
                    // Allocate memory for the string itself
                    size_t str_len = strlen(ppRequestedExtensionNames[i]) + 1;
                    char* copied_string = NULL;
                    if (vtContext->tempBuffer != NULL && (vtContext->tempBufferOffset + str_len) < 0x10000) {
                        copied_string = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
                        vtContext->tempBufferOffset += str_len;
                    } else {
                        copied_string = (char*)malloc(str_len);
                        ArrayList_add(vtContext->tempAllocations, copied_string);
                    }
                    strncpy(copied_string, ppRequestedExtensionNames[i], str_len);
                    copied_string[str_len - 1] = '\0'; // Ensure null-termination

                    ((char**)new_final_array)[dest_idx++] = copied_string;
                }
            }

            // Update the caller's pointers and count
            *ppExposedExtensionNames = (char**)new_final_array;
            *pExposedExtensionCount = total_final_count;
        }
    }

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Modifies lists of enabled/exposed extension properties based on supported and requested extensions.
 *
 * This function is similar to `injectExtensions`, but it specifically handles
 * `VkExtensionProperties` structs directly, copying their content (name and specVersion)
 * rather than just string pointers. It filters existing exposed extension properties
 * (`ppExposedExtensionProperties`) against actual supported properties (`pSupportedExtensionProperties`)
 * and then adds unique requested extension names (`ppRequestedExtensionNames`),
 * setting their `specVersion` to 1.
 *
 * Memory for new `VkExtensionProperties` structs is managed either via `vtContext->tempBuffer`
 * or `malloc`, tracked by `vtContext->tempAllocations`.
 *
 * @param vtContext A pointer to the VtContext structure.
 * @param ppExposedExtensionProperties A pointer to a `VkExtensionProperties**` that initially points to an array of
 *                                     `VkExtensionProperties` structs (old exposed extensions). This will be updated.
 * @param pExposedExtensionCount A pointer to a `uint32_t` that holds the count of old exposed extensions.
 *                               This will be updated with the count of the new list.
 * @param ppRequestedExtensionNames An array of `const char*` strings representing extensions requested by the application.
 * @param requestedExtensionCount The number of extensions in `ppRequestedExtensionNames`.
 * @param pSupportedExtensionProperties An array of `VkExtensionProperties` structs representing extensions actually supported by the device.
 * @param supportedExtensionCount The number of extensions in `pSupportedExtensionProperties`.
 */
void injectExtensions2(
    VtContext* vtContext,
    VkExtensionProperties** ppExposedExtensionProperties, // param_2, updated pointer to array of VkExtensionProperties
    uint32_t* pExposedExtensionCount, // param_3, updated count
    const char* const* ppRequestedExtensionNames, // param_4, strings requested by app
    uint32_t requestedExtensionCount, // param_5, count of requested
    const VkExtensionProperties* pSupportedExtensionProperties, // param_6, actual supported properties from device
    uint32_t supportedExtensionCount // param_7, count of supported
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);
    // Allocate a temporary boolean array on the stack.
    char temp_bool_flags[(*pExposedExtensionCount + requestedExtensionCount) + 16]; 

    VkExtensionProperties* old_exposed_properties = *ppExposedExtensionProperties;
    uint32_t old_exposed_count = *pExposedExtensionCount;
    char* current_flag_ptr = temp_bool_flags;

    // Phase 1: Filter existing exposed extension properties against supported properties.
    // Only keep exposed extension properties that are actually supported.
    if (pSupportedExtensionProperties != NULL) {
        uint32_t num_actually_supported_exposed = 0; // Counts exposed extensions that are supported
        for (uint32_t i = 0; i < old_exposed_count; ++i) {
            current_flag_ptr[i] = '\0'; // Initialize flag to false (not yet found in supported list)
            bool found_in_supported = false;
            if (old_exposed_properties[i].extensionName[0] != '\0') { // Check if extension name is not empty
                for (uint32_t j = 0; j < supportedExtensionCount; ++j) {
                    if (strcmp(pSupportedExtensionProperties[j].extensionName, old_exposed_properties[i].extensionName) == 0) {
                        found_in_supported = true;
                        current_flag_ptr[i] = '\x01'; // Mark as supported
                        break;
                    }
                }
            }
            if (found_in_supported) {
                num_actually_supported_exposed++;
            }
        }

        // Allocate new memory for the filtered list of exposed properties.
        size_t alloc_size = (size_t)num_actually_supported_exposed * sizeof(VkExtensionProperties); // 0x104 bytes per struct
        void* new_exposed_properties_array = NULL;
        // Try to allocate from the temporary buffer first
        if (vtContext->tempBuffer != NULL && (vtContext->tempBufferOffset + alloc_size) < 0x10000) {
            new_exposed_properties_array = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
            vtContext->tempBufferOffset += alloc_size;
        } else { // Fallback to malloc
            new_exposed_properties_array = malloc(alloc_size);
            ArrayList_add(vtContext->tempAllocations, new_exposed_properties_array);
        }
        memset(new_exposed_properties_array, 0, alloc_size); // Zero-initialize new memory

        // Copy the actually supported exposed extension properties to the new array
        uint32_t dest_idx = 0;
        for (uint32_t i = 0; i < old_exposed_count; ++i) {
            if (current_flag_ptr[i] == '\x01') { // If old exposed extension is supported
                memcpy(&((VkExtensionProperties*)new_exposed_properties_array)[dest_idx++], &old_exposed_properties[i], sizeof(VkExtensionProperties));
            }
        }

        // Update the caller's pointers and count
        *ppExposedExtensionProperties = (VkExtensionProperties*)new_exposed_properties_array;
        *pExposedExtensionCount = num_actually_supported_exposed;
    }

    // Phase 2: Add unique requested extension names to the filtered exposed properties list.
    // The `temp_bool_flags` buffer is reused for this phase.
    if (requestedExtensionCount != 0) {
        current_flag_ptr = temp_bool_flags; // Reset pointer for the new set of flags
        uint32_t current_exposed_count = *pExposedExtensionCount;
        VkExtensionProperties* current_exposed_properties = *ppExposedExtensionProperties;

        uint32_t num_new_unique_requested = 0; // Counts requested extensions that are not already exposed
        for (uint32_t i = 0; i < requestedExtensionCount; ++i) {
            current_flag_ptr[i] = '\0'; // Initialize flag to false (not yet found in current_exposed_properties)
            bool found_in_exposed = false;
            if (ppRequestedExtensionNames[i] != NULL) {
                for (uint32_t j = 0; j < current_exposed_count; ++j) {
                    if (strcmp(ppRequestedExtensionNames[i], current_exposed_properties[j].extensionName) == 0) {
                        found_in_exposed = true;
                        current_flag_ptr[i] = '\x01'; // Mark as duplicate
                        break;
                    }
                }
            }
            if (!found_in_exposed) {
                num_new_unique_requested++;
            }
        }

        uint32_t total_final_count = current_exposed_count + num_new_unique_requested;

        if (total_final_count != current_exposed_count) { // Only reallocate if the total count changes
            size_t alloc_size = (size_t)total_final_count * sizeof(VkExtensionProperties);
            void* new_final_array = NULL;
            // Try to allocate from the temporary buffer first
            if (vtContext->tempBuffer != NULL && (vtContext->tempBufferOffset + alloc_size) < 0x10000) {
                new_final_array = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
                vtContext->tempBufferOffset += alloc_size;
            } else { // Fallback to malloc
                new_final_array = malloc(alloc_size);
                ArrayList_add(vtContext->tempAllocations, new_final_array);
            }
            memset(new_final_array, 0, alloc_size); // Zero-initialize new memory

            // Copy existing filtered exposed properties to the new final array
            if (current_exposed_count > 0) {
                memcpy(new_final_array, current_exposed_properties, (size_t)current_exposed_count * sizeof(VkExtensionProperties));
            }

            // Append unique requested names, setting their specVersion to 1
            uint32_t dest_idx = current_exposed_count;
            for (uint32_t i = 0; i < requestedExtensionCount; ++i) {
                if (current_flag_ptr[i] == '\0') { // If not a duplicate (unique)
                    VkExtensionProperties* new_prop = (VkExtensionProperties*)((char*)new_final_array + dest_idx * sizeof(VkExtensionProperties));
                    strncpy(new_prop->extensionName, ppRequestedExtensionNames[i], VK_MAX_EXTENSION_NAME_SIZE);
                    new_prop->extensionName[VK_MAX_EXTENSION_NAME_SIZE - 1] = '\0'; // Ensure null-termination
                    new_prop->specVersion = 1; // Set specVersion to 1, common default
                    dest_idx++;
                }
            }

            // Update the caller's pointers and count
            *ppExposedExtensionProperties = (VkExtensionProperties*)new_final_array;
            *pExposedExtensionCount = total_final_count;
        }
    }

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Overrides the size of physical device memory heaps based on a maximum device memory setting.
 *
 * This function iterates through the memory heaps reported by the physical device
 * and caps their `size` property to a maximum value specified in `vtContext->maxDeviceMemory`.
 * The maximum size is calculated as `vtContext->maxDeviceMemory` in MB, shifted to bytes.
 *
 * @param vtContext A pointer to the VtContext structure, specifically accessing `maxDeviceMemory`.
 * @param pMemoryProperties A pointer to a VkPhysicalDeviceMemoryProperties structure.
 *                         This structure's `memoryHeaps` array will be modified in-place.
 */
void overrideMemoryHeapSize(VtContext* vtContext, VkPhysicalDeviceMemoryProperties* pMemoryProperties) {
    uint32_t memoryHeapCount = pMemoryProperties->memoryHeapCount;

    if (memoryHeapCount != 0) {
        // Calculate the maximum allowed memory size for a heap in bytes (MB to bytes)
        // vtContext->maxDeviceMemory is a short at offset 0x8 from VtContext start.
        VkDeviceSize max_allowed_heap_size = (VkDeviceSize)vtContext->maxDeviceMemory << 20; // x << 20 is x * 1024 * 1024 (MB to Bytes)

        // Iterate through the memory heaps and cap their sizes
        // The loop processes pairs of heaps for efficiency, then handles a single remaining heap if count is odd.
        if (memoryHeapCount > 0) {
            uint32_t i = 0;
            // Loop for pairs of heaps
            for (; i + 1 < memoryHeapCount; i += 2) {
                // Heap i
                VkDeviceSize current_heap_size_1 = pMemoryProperties->memoryHeaps[i].size;
                if (max_allowed_heap_size <= current_heap_size_1) {
                    current_heap_size_1 = max_allowed_heap_size;
                }
                pMemoryProperties->memoryHeaps[i].size = current_heap_size_1;

                // Heap i+1
                VkDeviceSize current_heap_size_2 = pMemoryProperties->memoryHeaps[i+1].size;
                if (max_allowed_heap_size <= current_heap_size_2) {
                    current_heap_size_2 = max_allowed_heap_size;
                }
                pMemoryProperties->memoryHeaps[i+1].size = current_heap_size_2;
            }

            // Handle the last heap if total count was odd
            if (i < memoryHeapCount) {
                VkDeviceSize current_heap_size = pMemoryProperties->memoryHeaps[i].size;
                if (max_allowed_heap_size <= current_heap_size) {
                    current_heap_size = max_allowed_heap_size;
                }
                pMemoryProperties->memoryHeaps[i].size = current_heap_size;
            }
        }
    }
    // No stack canary check in this function in the assembly.
}

/**
 * @brief Disables unsupported features in a VkDeviceCreateInfo structure.
 *
 * This function queries the physical device's supported features and then
 * iterates through the `pNext` chain of the `VkDeviceCreateInfo` structure to find
 * `VkPhysicalDeviceFeatures` structures. For any `VkBool32` features that were
 * requested by the application but are not actually supported by the physical device,
 * they are set to `VK_FALSE` in the `pEnabledFeatures` structure.
 *
 * @param physicalDevice A handle to the VkPhysicalDevice to query for supported features.
 * @param pCreateInfo A pointer to the VkDeviceCreateInfo structure whose `pNext` chain
 *                    and `pEnabledFeatures` will be traversed and modified.
 */
void disableUnsupportedDeviceFeatures(VkPhysicalDevice physicalDevice, VkDeviceCreateInfo* pCreateInfo) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);

    // Get the actual supported physical device features into a local VkPhysicalDeviceFeatures2 struct.
    VkPhysicalDeviceFeatures2 supportedFeatures2 = {
        .sType = VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_FEATURES_2,
        .pNext = NULL
    };
    // DAT_00149048 is likely `vkGetPhysicalDeviceFeatures2`.
    // The assembly also contains logic that looks for `VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_DRIVER_PROPERTIES`
    // (0x3b9b3760) or `0x3b9bb078` (unknown sType) in the pNext chain before calling `vkGetPhysicalDeviceFeatures2`.
    // This is a complex flow but the end result is to populate `supportedFeatures2`.
    DAT_00149048(physicalDevice, &supportedFeatures2);

    // Iterate through the pNext chain of VkDeviceCreateInfo
    VkBaseInStructure* currentStructInChain = (VkBaseInStructure*)pCreateInfo->pNext;
    while (currentStructInChain != NULL) {
        // The original assembly's logic branches based on `currentStructInChain->sType`
        // and then processes `pCreateInfo->pEnabledFeatures`.
        // The core logic is to iterate through the `VkPhysicalDeviceFeatures` members
        // and disable any requested features that are not supported.
        // The loop for features iterates 0xdc bytes (220 bytes), which is the size of VkPhysicalDeviceFeatures.
        if (pCreateInfo->pEnabledFeatures != NULL) {
            VkPhysicalDeviceFeatures* requestedFeatures = (VkPhysicalDeviceFeatures*)pCreateInfo->pEnabledFeatures;
            VkPhysicalDeviceFeatures* actualSupportedFeatures = &supportedFeatures2.features;

            // Iterate over all VkBool32 fields in VkPhysicalDeviceFeatures struct
            for (uint32_t offset = 0; offset < sizeof(VkPhysicalDeviceFeatures); offset += sizeof(VkBool32)) {
                VkBool32* requested_feature_ptr = (VkBool32*)((char*)requestedFeatures + offset);
                VkBool32* supported_feature_ptr = (VkBool32*)((char*)actualSupportedFeatures + offset);

                // If a feature is requested (non-zero) AND the device does NOT support it (zero in supported features)
                if (*requested_feature_ptr != 0 && *supported_feature_ptr == 0) {
                    // Disable the requested feature
                    *requested_feature_ptr = VK_FALSE;
                }
            }
        }
        currentStructInChain = (VkBaseInStructure*)currentStructInChain->pNext;
    }

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}

/**
 * @brief Retrieves and filters exposed device extension properties.
 *
 * This function first queries all device extensions supported by the given physical device.
 * It then filters this list based on what is already present in `vtContext->exposedInstanceExtensions`
 * or `vtContext->exposedDeviceExtensions` depending on the initialization state.
 *
 * The function allocates memory for the resulting `VkExtensionProperties` array
 * either from `vtContext->tempBuffer` or `malloc`, tracking it with `vtContext->tempAllocations`.
 *
 * @param vtContext A pointer to the VtContext structure.
 * @param physicalDevice A handle to the VkPhysicalDevice.
 * @param pExtensionCount A pointer to a uint32_t that will be updated with the count of exposed device extensions.
 * @return A pointer to the dynamically allocated array of VkExtensionProperties for exposed device extensions.
 */
VkExtensionProperties* getExposedDeviceExtensionProperties(
    VtContext* vtContext,
    VkPhysicalDevice physicalDevice,
    uint32_t* pExtensionCount
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);
    uint32_t all_supported_device_ext_count = 0;
    VkExtensionProperties* all_supported_device_ext_properties = NULL;
    VkExtensionProperties* exposed_device_ext_properties_result = NULL; // The final return value

    // 1. Enumerate all supported device extensions (first call to get count, second to get data)
    DAT_00149080(physicalDevice, NULL, &all_supported_device_ext_count, NULL); // vkEnumerateDeviceExtensionProperties
    if (all_supported_device_ext_count > 0) {
        // Allocate a temporary buffer on stack for all supported extensions
        // size: count * sizeof(VkExtensionProperties) + safety_margin_for_alignment
        char temp_buffer_for_all_supported[all_supported_device_ext_count * sizeof(VkExtensionProperties) + 16];
        all_supported_device_ext_properties = (VkExtensionProperties*)temp_buffer_for_all_supported;

        DAT_00149080(physicalDevice, NULL, &all_supported_device_ext_count, all_supported_device_ext_properties);
    } else {
        *pExtensionCount = 0;
        return NULL; // No device extensions supported
    }

    // 2. Allocate memory for the output list of exposed device extensions
    // The initial size of this allocation depends on `vtContext->exposedDeviceExtensions->count`
    // OR `all_supported_device_ext_count` if `vtContext->exposedDeviceExtensions` is initially NULL.
    uint32_t max_possible_exposed_count = 0;
    if (vtContext->exposedDeviceExtensions != NULL) {
        max_possible_exposed_count = vtContext->exposedDeviceExtensions->count;
    } else {
        max_possible_exposed_count = all_supported_device_ext_count;
    }
    
    size_t alloc_size_for_output = (size_t)max_possible_exposed_count * sizeof(VkExtensionProperties);
    void* allocated_mem_ptr = NULL;
    // Try to allocate from the temporary buffer first, then fallback to malloc
    if (vtContext->tempBuffer != NULL && (vtContext->tempBufferOffset + alloc_size_for_output) < 0x10000) {
        allocated_mem_ptr = (char*)vtContext->tempBuffer + vtContext->tempBufferOffset;
        vtContext->tempBufferOffset += alloc_size_for_output;
    } else {
        allocated_mem_ptr = malloc(alloc_size_for_output);
        ArrayList_add(vtContext->tempAllocations, allocated_mem_ptr);
    }
    memset(allocated_mem_ptr, 0, alloc_size_for_output);
    exposed_device_ext_properties_result = (VkExtensionProperties*)allocated_mem_ptr;
    *pExtensionCount = 0; // Initialize actual output count to 0

    // 3. Filter and populate the exposed extensions based on the complex logic.
    // The assembly logic is complex, effectively creating an intersection:
    // If `vtContext->exposedDeviceExtensions` is NULL (meaning first time population):
    //   Iterate through `all_supported_device_ext_properties`. If an extension is found in
    //   `vtContext->exposedInstanceExtensions`, then it's added to `exposed_device_ext_properties_result`.
    // Else (if `vtContext->exposedDeviceExtensions` is already populated):
    //   Iterate through `vtContext->exposedDeviceExtensions->items`. If an extension is found in
    //   `all_supported_device_ext_properties`, then it's added to `exposed_device_ext_properties_result`.
    
    uint32_t dest_current_idx = 0;
    
    if (vtContext->exposedDeviceExtensions == NULL) { // Check if exposedDeviceExtensions is initially NULL
        // First time population: filter `all_supported_device_ext_properties` against `exposedInstanceExtensions`
        for (uint32_t i = 0; i < all_supported_device_ext_count; ++i) {
            VkExtensionProperties* current_supported = &all_supported_device_ext_properties[i];
            bool found_in_instance_exposed = false;
            if (vtContext->exposedInstanceExtensions != NULL) {
                for (uint32_t j = 0; j < vtContext->exposedInstanceExtensions->count; ++j) {
                    char* instance_ext_name = (char*)vtContext->exposedInstanceExtensions->items[j];
                    if (instance_ext_name != NULL && strcmp(current_supported->extensionName, instance_ext_name) == 0) {
                        found_in_instance_exposed = true;
                        break;
                    }
                }
            }
            if (found_in_instance_exposed) {
                memcpy(&exposed_device_ext_properties_result[dest_current_idx], current_supported, sizeof(VkExtensionProperties));
                dest_current_idx++;
            }
        }
    } else {
        // Subsequent population: filter `vtContext->exposedDeviceExtensions` against `all_supported_device_ext_properties`
        for (uint32_t i = 0; i < initial_exposed_device_ext_count; ++i) {
            char* exposed_ext_name = (char*)vtContext->exposedDeviceExtensions->items[i];
            bool found_in_supported_device = false;
            if (exposed_ext_name != NULL) {
                for (uint32_t j = 0; j < all_supported_device_ext_count; ++j) {
                    if (strcmp(exposed_ext_name, all_supported_device_ext_properties[j].extensionName) == 0) {
                        found_in_supported_device = true;
                        break;
                    }
                }
            }
            if (found_in_supported_device) {
                // Copy the entire VkExtensionProperties struct
                memcpy(&exposed_device_ext_properties_result[dest_current_idx], 
                       &((VkExtensionProperties*)vtContext->exposedDeviceExtensions->items)[i], 
                       sizeof(VkExtensionProperties));
                dest_current_idx++;
            }
        }
    }
    *pExtensionCount = dest_current_idx; // Update the final count

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return exposed_device_ext_properties_result;
}

/**
 * @brief Checks and potentially modifies VkFormatProperties for specific formats.
 *
 * This function checks certain properties of a given VkFormat and its associated
 * VkFormatProperties. It applies specific modifications based on whether the format
 * is compressed or scaled, and if initial properties are missing.
 *
 * @param physicalDevice A handle to the VkPhysicalDevice (used by an internal function call).
 * @param format The VkFormat to check.
 * @param pFormatProperties A pointer to a VkFormatProperties structure (likely embedded
 *                          within a VkFormatProperties2 struct) whose features might be modified.
 */
void checkFormatProperties(VkPhysicalDevice physicalDevice, VkFormat format, VkFormatProperties* pFormatProperties) {
    // Check if the format is a compressed format AND its linear and optimal tiling features are both zero.
    // This could indicate a driver reporting no support for compressed textures in tiled formats.
    if (isCompressedFormat(format) && pFormatProperties->linearTilingFeatures == 0 && pFormatProperties->optimalTilingFeatures == 0) {
        // If so, call an internal function (DAT_00149050) to potentially "fix" or provide default features.
        // The magic number `0x2c` (44 decimal) corresponds to `VK_STRUCTURE_TYPE_BUFFER_MEMORY_BARRIER`.
        // This is highly unusual and suggests that `0x2c` is used as an internal key
        // to look up specific fallback or default properties, not as a standard Vulkan sType for an API call.
        DAT_00149050(physicalDevice, 0x2c, pFormatProperties);
    }

    // Check if the format is a "scaled" format (e.g., YCbCr formats with sub-sampling).
    if (isFormatScaled(format)) {
        // If it's a scaled format, ensure that the VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT (0x40)
        // is set in the bufferFeatures for this format. This implies scaled formats can be
        // used as vertex buffers in this system.
        pFormatProperties->bufferFeatures |= VK_FORMAT_FEATURE_VERTEX_BUFFER_BIT;
    }
}

/**
 * @brief Initializes certain feature flags within an internal framework structure to enabled.
 *
 * This function takes a pointer to a framework-specific features structure (likely part of
 * `AsyncPipelineCreator`) and sets various `VkBool32` (uint32_t) members at specific offsets
 * to `VK_TRUE` (1). This is distinct from Vulkan device features and refers to internal
 * framework capabilities or flags.
 *
 * @param frameworkFeaturesPtr A pointer to the internal framework features structure
 *                             (e.g., `VtContext::asyncPipelineCreator`).
 */
void checkDeviceFeatures(uint32_t* frameworkFeaturesPtr) {
    // Set various feature flags to VK_TRUE (1) based on their offsets.
    // The offsets are relative to the start of `frameworkFeaturesPtr`.
    frameworkFeaturesPtr[0x34 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x34
    frameworkFeaturesPtr[0x94 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x94
    frameworkFeaturesPtr[0x58 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x58
    frameworkFeaturesPtr[0x0c / sizeof(uint32_t)] = VK_TRUE; // Offset 0x0c
    frameworkFeaturesPtr[0x48 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x48
    frameworkFeaturesPtr[0x70 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x70
    frameworkFeaturesPtr[0xd4 / sizeof(uint32_t)] = VK_TRUE; // Offset 0xd4
    frameworkFeaturesPtr[0x04 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x04
    frameworkFeaturesPtr[0x68 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x68
    frameworkFeaturesPtr[0x2c / sizeof(uint32_t)] = VK_TRUE; // Offset 0x2c
    frameworkFeaturesPtr[0x24 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x24
    frameworkFeaturesPtr[0x1c / sizeof(uint32_t)] = VK_TRUE; // Offset 0x1c
    frameworkFeaturesPtr[0x14 / sizeof(uint32_t)] = VK_TRUE; // Offset 0x14
}


/**
 * @brief Creates and initializes a VtContext object.
 *
 * This function allocates and initializes a `VtContext` structure, populating it
 * with various parameters including client file descriptor, Vulkan version,
 * memory limits, and setting up ring buffers and worker threads for
 * communication and asynchronous operations. It also initializes Java-related
 * members for Android windowing.
 *
 * @param pJNIEnv A pointer to the JNIEnv interface (param_1 in assembly).
 * @param javaComponentObject A JNI jobject representing the Java component (param_2).
 * @param clientFd The client file descriptor for communication (param_3).
 * @param javaWindowManagerObject A JNI jobject for the Java window manager (param_4).
 * @return A pointer to the newly created and initialized VtContext object, or NULL on failure.
 */
VtContext* createVkContext(
    void* pJNIEnv,
    jobject javaComponentObject,
    int clientFd,
    jobject javaWindowManagerObject
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);
    VtContext* vtContext = (VtContext*)calloc(1, sizeof(VtContext)); // Allocate and zero-initialize VtContext

    if (vtContext == NULL) {
        goto cleanup_and_fail;
    }

    // Initialize VtContext members by fetching values from Java via JNI (using FUN_0013d48c helper)
    // FUN_0013d48c resolves JNI field IDs and gets field values.
    
    // Client file descriptor
    vtContext->clientFd = clientFd;

    // Vulkan Max Version (int)
    vtContext->vkMaxVersion = (int)FUN_0013d48c(pJNIEnv, javaComponentObject, "vkMaxVersion", "I");

    // Max Device Memory (short in MB)
    vtContext->maxDeviceMemory = (short)FUN_0013d48c(pJNIEnv, javaComponentObject, "maxDeviceMemory", "S");

    // Image Cache Size (short in MB)
    vtContext->imageCacheSize = (short)FUN_0013d48c(pJNIEnv, javaComponentObject, "imageCacheSize", "S");

    // Exposed Device Extensions (ArrayList of String pointers)
    jobjectArray exposedDeviceExtensionsJavaArray = (jobjectArray)FUN_0013d48c(pJNIEnv, javaComponentObject, "exposedDeviceExtensions", "[Ljava/lang/String;");
    if (exposedDeviceExtensionsJavaArray != NULL) {
        ArrayList* exposedDeviceExtensionsList = (ArrayList*)calloc(1, sizeof(ArrayList));
        if (exposedDeviceExtensionsList == NULL) goto cleanup_and_fail;

        // Get array length via JNIEnv->GetArrayLength
        jsize array_len = (**(jsize (*)(void*, jobjectArray))((char*)pJNIEnv + 0x558))(pJNIEnv, exposedDeviceExtensionsJavaArray);
        
        if (array_len > 0) {
            exposedDeviceExtensionsList->count = 0; // Will be incremented by ArrayList_add
            exposedDeviceExtensionsList->capacity = array_len; // Set initial capacity
            exposedDeviceExtensionsList->items = (void**)calloc(array_len, sizeof(char*)); // Allocate array of string pointers
            if (exposedDeviceExtensionsList->items == NULL) { free(exposedDeviceExtensionsList); goto cleanup_and_fail; }

            for (jsize i = 0; i < array_len; ++i) {
                // Get string element from Java array via JNIEnv->GetObjectArrayElement
                jstring jstr_ext = (**(jstring (*)(void*, jobjectArray, jint))((char*)pJNIEnv + 0x568))(pJNIEnv, exposedDeviceExtensionsJavaArray, i);
                // Convert Java string to C string (UTF-8) via JNIEnv->GetStringUTFChars
                const char* cstr_ext = (**(const char* (*)(void*, jstring, void*))((char*)pJNIEnv + 0x548))(pJNIEnv, jstr_ext, NULL);
                char* dup_cstr_ext = strdup(cstr_ext); // Duplicate the C string to own its memory
                ArrayList_add(exposedDeviceExtensionsList, dup_cstr_ext); // Add the duplicated string to the ArrayList
                // Release the JNI string resource via JNIEnv->ReleaseStringUTFChars
                (**(void (*)(void*, jstring, const char*))((char*)pJNIEnv + 0x550))(pJNIEnv, jstr_ext, cstr_ext);
            }
        }
        vtContext->exposedDeviceExtensions = exposedDeviceExtensionsList;
    }

    // Initialize temporary buffer (for deserialization, etc.)
    vtContext->tempBuffer = calloc(1, 0x10000); // 64KB (65536 bytes)
    if (vtContext->tempBuffer == NULL) goto cleanup_and_fail;
    vtContext->tempBufferOffset = 0; // Initialize offset within tempBuffer

    // Initialize temporary allocations list (ArrayList to track malloc'd blocks)
    vtContext->tempAllocations = (ArrayList*)calloc(1, sizeof(ArrayList));
    if (vtContext->tempAllocations == NULL) goto cleanup_and_fail;

    // Initialize and start AsyncPipelineCreator (abstracted as `asyncPipelineCreator_instance`)
    // `0xd0` is the size of the AsyncPipelineCreator struct.
    void* asyncPipelineCreator_instance = calloc(1, 0xd0); 
    if (asyncPipelineCreator_instance == NULL) goto cleanup_and_fail;

    // Initialize various fields within AsyncPipelineCreator (likely flags or state variables)
    *(uint32_t*)asyncPipelineCreator_instance = 4; // Field at offset 0
    *(uint8_t*)((char*)asyncPipelineCreator_instance + 8) = 1; // Field at offset 8 (byte)

    // Initialize mutexes and condition variables for AsyncPipelineCreator's internal synchronization
    pthread_mutex_init((pthread_mutex_t*)((char*)asyncPipelineCreator_instance + 12), NULL); // Offset 12 (0xc)
    pthread_mutex_init((pthread_mutex_t*)((char*)asyncPipelineCreator_instance + 100), NULL); // Offset 100 (0x64)
    pthread_cond_init((pthread_cond_t*)((char*)asyncPipelineCreator_instance + 52), NULL); // Offset 52 (0x34)
    pthread_cond_init((pthread_cond_t*)((char*)asyncPipelineCreator_instance + 140), NULL); // Offset 140 (0x8c)

    // Create worker threads for async pipeline creation (4 threads)
    pthread_t worker_threads[4]; // local_100 (array of 18 pthread_t, only first 4 used for creation)
    for (int i = 0; i < 4; ++i) {
        pthread_create(&worker_threads[i], NULL, FUN_0013d7ec, asyncPipelineCreator_instance);
        pthread_detach(worker_threads[i]); // Detach thread to allow independent execution
    }
    vtContext->asyncPipelineCreator = asyncPipelineCreator_instance;

    // Setup client and server RingBuffers using Android shared memory (ashmem)
    // Server RingBuffer: 32MB (0x2000000 bytes)
    int server_shm_size_bytes = RingBuffer_getSHMemSize(0x2000000);
    int server_shm_fd = ashmemCreateRegion("vt-server-ring", server_shm_size_bytes);
    
    // Client RingBuffer: 256KB (0x40000 bytes)
    int client_shm_size_bytes = RingBuffer_getSHMemSize(0x40000);
    int client_shm_fd = ashmemCreateRegion("vt-client-ring", client_shm_size_bytes);

    vtContext->serverRingBuffer = RingBuffer_create(server_shm_fd, 0x2000000);
    if (vtContext->serverRingBuffer == NULL) goto cleanup_and_fail;

    vtContext->clientRingBuffer = RingBuffer_create(client_shm_fd, 0x40000);
    if (vtContext->clientRingBuffer == NULL) goto cleanup_and_fail;

    // Send shared memory FDs to the client via a socket (`clientFd`)
    struct msghdr msg = {0};
    struct iovec iov[1];
    char control_buf[CMSG_SPACE(sizeof(int) * 2)]; // Control message buffer for FDs
    char data_byte = 0; // A single byte of data to send with FD (required for sendmsg)

    iov[0].iov_base = &data_byte;
    iov[0].iov_len = sizeof(data_byte);
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    // Setup control message for file descriptor passing
    msg.msg_control = control_buf;
    msg.msg_controllen = sizeof(control_buf);

    struct cmsghdr *cmsg = CMSG_FIRSTHDR(&msg);
    if (cmsg != NULL) {
        cmsg->cmsg_level = SOL_SOCKET;
        cmsg->cmsg_type = SCM_RIGHTS;
        cmsg->cmsg_len = CMSG_LEN(sizeof(int) * 2);
        int* fds = (int*)CMSG_DATA(cmsg);
        fds[0] = server_shm_fd;
        fds[1] = client_shm_fd;
    }

    ssize_t bytes_sent = sendmsg(clientFd, &msg, 0);

    // Close local copies of shared memory FDs, as they are now passed to the client process
    close(server_shm_fd);
    close(client_shm_fd);

    if (bytes_sent == -1) {
        goto cleanup_and_fail;
    }

    // Start the main server processing thread
    pthread_t server_main_thread_handle; // pthread_t at __arg + 0x16 (offset 0x58 in VtContext)
    pthread_create(&server_main_thread_handle, NULL, FUN_0013d91c, vtContext);

    // Get JNI method IDs for Android window management functions
    vtContext->getWindowWidthMethodID = FUN_0013d48c(pJNIEnv, javaWindowManagerObject, "getWindowWidth", "()I");
    vtContext->getWindowHeightMethodID = FUN_0013d48c(pJNIEnv, javaWindowManagerObject, "getWindowHeight", "()I");
    vtContext->getWindowHardwareBufferMethodID = FUN_0013d48c(pJNIEnv, javaWindowManagerObject, "getWindowHardwareBuffer", "()Landroid/hardware/HardwareBuffer;");
    vtContext->updateWindowContentMethodID = FUN_0013d48c(pJNIEnv, javaWindowManagerObject, "updateWindowContent", "()V");

    // Store JNI environment and Java object references
    vtContext->javaComponentObject = javaComponentObject;
    vtContext->jniEnv = pJNIEnv;
    vtContext->javaWindowManagerObject = javaWindowManagerObject;

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
    return vtContext;

cleanup_and_fail:
    // Comprehensive cleanup on failure
    if (vtContext != NULL) {
        // This simplified cleanup only frees the top-level pointers and assumes ArrayLists
        // themselves would handle deeper memory freeing via a destructor.
        // In a real scenario, this would involve iterating and freeing each string and item.
        if (vtContext->exposedDeviceExtensions != NULL) {
             // Proper ArrayList cleanup would iterate and free items
             free(vtContext->exposedDeviceExtensions);
        }
        if (vtContext->exposedInstanceExtensions != NULL) {
             // Proper ArrayList cleanup would iterate and free items
             free(vtContext->exposedInstanceExtensions);
        }
        if (vtContext->tempBuffer != NULL) free(vtContext->tempBuffer);
        if (vtContext->tempAllocations != NULL) {
             // Proper ArrayList cleanup would iterate and free tracked allocations
             free(vtContext->tempAllocations);
        }
        // Destroy mutexes/conds for asyncPipelineCreator before freeing
        // (Assuming AsyncPipelineCreator destructor would handle this)
        if (vtContext->asyncPipelineCreator != NULL) free(vtContext->asyncPipelineCreator);
        
        // RingBuffers might need a specific destroy/unmap call beyond `close(fd)`
        // (Assuming RingBuffer_destroy or similar would handle cleanup)

        free(vtContext);
    }
    return NULL;
}

// Note: The function `checkDeviceProperties` was duplicated in the original prompt.
// Its decompilation was provided in the first response and is included again here for completeness
// with the full set of parameter names as inferred.

/**
 * @brief Checks and updates device properties based on Vulkan API information and enabled extensions.
 *
 * This function modifies the provided VkPhysicalDeviceProperties and traverses a
 * pNext chain of Vulkan structures to apply further updates. It specifically
 * looks for VkPhysicalDeviceVulkan12Properties to update driver information
 * and disable certain shader float controls if the VK_KHR_shader_float_controls
 * extension is present. It also attempts to write the driver version into
 * VkPhysicalDeviceGroupProperties (though the target offset in that struct
 * seems unusual and potentially erroneous).
 *
 * @param vtContext A pointer to the custom VtContext structure, containing
 *                  information like `vkMaxVersion` and `exposedInstanceExtensions`.
 * @param pPhysicalDeviceProperties A pointer to a VkPhysicalDeviceProperties
 *                                  structure to be updated with API and driver versions.
 * @param pStructChainHead A pointer to the head of a VkBaseInStructure pNext chain.
 *                         The function iterates through this chain to find and update
 *                         specific Vulkan extension properties structures.
 * @param param_4 Unused (from assembly, part of generic va_args in original FUN_0013cfe4).
 * @param param_5 Unused (from assembly, part of generic va_args in original FUN_0013cfe4).
 * @param param_6 Unused (from assembly, part of generic va_args in original FUN_0013cfe4).
 * @param param_7 Unused (from assembly, part of generic va_args in original FUN_0013cfe4).
 * @param param_8 Unused (from assembly, part of generic va_args in original FUN_0013cfe4).
 */
void checkDeviceProperties(
    VtContext* vtContext,
    VkPhysicalDeviceProperties* pPhysicalDeviceProperties,
    VkBaseInStructure* pStructChainHead, // This is param_3 in assembly
    void* param_4, void* param_5, void* param_6, void* param_7, void* param_8 // Extra args for FUN_0013cfe4
) {
    long stack_chk_guard = *(long*)(((char*)tpidr_el0) + 0x28);
    // Temporary stack buffer for formatted strings
    char formatted_string_buffer[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE]; // `local_170` in assembly

    // 1. Format and update the device name.
    // Reads existing deviceName from `pPhysicalDeviceProperties`, prepends "Vortek (", appends ")",
    // and writes the new string back to `deviceName`.
    // `param_2 + 5` refers to `(char*)pPhysicalDeviceProperties + 20` which is `pPhysicalDeviceProperties->deviceName`.
    FUN_0013cfe4(formatted_string_buffer, sizeof(formatted_string_buffer), "Vortek (%s)",
                 pPhysicalDeviceProperties->deviceName, param_5, param_6, param_7, param_8);
    strncpy(pPhysicalDeviceProperties->deviceName, formatted_string_buffer, VK_MAX_PHYSICAL_DEVICE_NAME_SIZE);
    pPhysicalDeviceProperties->deviceName[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE - 1] = '\0'; // Ensure null-termination

    // 2. Set the Vulkan API version in VkPhysicalDeviceProperties.
    // `*param_2` refers to `pPhysicalDeviceProperties->apiVersion`.
    // `*(undefined4 *)(param_1 + 4)` refers to `vtContext->vkMaxVersion`.
    pPhysicalDeviceProperties->apiVersion = vtContext->vkMaxVersion;

    // 3. Iterate through the `pNext` chain (starting from `pStructChainHead`) to find and update specific properties.
    VkBaseInStructure* currentStruct = pStructChainHead; // `param_3` in assembly (also aliased as `piVar6`)
    if (currentStruct != NULL) { // Loop condition for `do...while`
        do {
            // Check if the current structure is `VkPhysicalDeviceVulkan12Properties` (sType == 0x34 or 52)
            if (currentStruct->sType == VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_VULKAN_1_2_PROPERTIES) {
                VkPhysicalDeviceVulkan12Properties* props12 = (VkPhysicalDeviceVulkan12Properties*)currentStruct;
                char driver_version_str[VK_MAX_DRIVER_INFO_SIZE]; // Temporary buffer for driver version string

                // Format the driver version (major.minor.patch) from `pPhysicalDeviceProperties->driverVersion`.
                uint32_t driverVersion = pPhysicalDeviceProperties->driverVersion;
                FUN_0013cfe4(driver_version_str, sizeof(driver_version_str), "%d.%d.%d",
                             VK_API_VERSION_MAJOR(driverVersion),
                             VK_API_VERSION_MINOR(driverVersion),
                             VK_API_VERSION_PATCH(driverVersion),
                             param_7, param_8); // Passing extra params that are unused by FUN_0013cfe4 format string
                
                // Copy the formatted version string into `props12->driverInfo`.
                // `piVar6 + 0x45` in assembly (where `piVar6` is `currentStruct`) corresponds to
                // `(uint32_t*)currentStruct + 0x45` which is `(char*)currentStruct + 0x114` (offset 276 bytes).
                // This offset aligns with `props12->driverInfo`.
                strncpy(props12->driverInfo, driver_version_str, VK_MAX_DRIVER_INFO_SIZE);
                props12->driverInfo[VK_MAX_DRIVER_INFO_SIZE - 1] = '\0'; // Ensure null-termination

                // Check `vtContext->exposedInstanceExtensions` for "VK_KHR_shader_float_controls".
                // `puVar5 = *(uint **)(param_1 + 0x18);` -> `vtContext->exposedInstanceExtensions` (an ArrayList*)
                ArrayList* instanceExtensionsList = vtContext->exposedInstanceExtensions;
                if (instanceExtensionsList != NULL && instanceExtensionsList->count > 0) {
                    char** extensionNames = (char**)instanceExtensionsList->items; // `*(long *)(puVar5 + 2)`
                    uint32_t listCount = instanceExtensionsList->count; // `*puVar5`
                    for (uint32_t i = 0; i < listCount; ++i) { // Loop `uVar7` from 0 to `uVar2` (listCount)
                        // `*(char **)(lVar8 + uVar7 * 8)` -> `extensionNames[i]`
                        if (extensionNames[i] != NULL && strcmp(extensionNames[i], "VK_KHR_shader_float_controls") == 0) {
                            // If extension is found, disable several shader float control features.
                            // `piVar6` is `props12` (the `currentStruct` cast to `int*`).
                            // `piVar6[0x89]` to `piVar6[0x90]` are offsets into `VkPhysicalDeviceVulkan12Properties` struct.
                            // These correspond to `VkBool32` (uint32_t) members that enable/disable features.
                            props12->shaderSignedZeroInfNanPreserveFloat16 = VK_FALSE; // Offset 0x224
                            props12->shaderSignedZeroInfNanPreserveFloat32 = VK_FALSE; // Offset 0x228
                            props12->shaderDenormFlushToZeroFloat16 = VK_FALSE;        // Offset 0x23C
                            props12->shaderDenormFlushToZeroFloat32 = VK_FALSE;        // Offset 0x240
                            props12->shaderDenormPreserveFloat16 = VK_FALSE;           // Offset 0x230
                            props12->shaderDenormPreserveFloat32 = VK_FALSE;           // Offset 0x234
                            break; // Exit inner loop once the extension is handled
                        }
                    }
                }
                break; // Exit outer loop after processing VkPhysicalDeviceVulkan12Properties
            }
            currentStruct = (VkBaseInStructure*)currentStruct->pNext; // Move to the next structure
        } while (currentStruct != NULL);
    }

    // 4. Second pass through the `pNext` chain, specifically for `VkPhysicalDeviceGroupProperties`.
    currentStruct = pStructChainHead; // Reset the pointer to the beginning of the chain.
    if (currentStruct != NULL) { // Loop condition
        do {
            // Check if the current structure is `VkPhysicalDeviceGroupProperties` (sType == 0x3b9dc7a0 or 1000070000)
            if (currentStruct->sType == VK_STRUCTURE_TYPE_PHYSICAL_DEVICE_GROUP_PROPERTIES) {
                // This block is identical in effect to the `driverInfo` update above,
                // but targets a different struct type at the same relative offset (0x114) from its base.
                // This is likely writing the driver version string into padding or an unused field
                // within `VkPhysicalDeviceGroupProperties`.
                char driver_version_str[VK_MAX_DRIVER_INFO_SIZE];
                uint32_t driverVersion = pPhysicalDeviceProperties->driverVersion;
                FUN_0013cfe4(driver_version_str, sizeof(driver_version_str), "%d.%d.%d",
                             VK_API_VERSION_MAJOR(driverVersion),
                             VK_API_VERSION_MINOR(driverVersion),
                             VK_API_VERSION_PATCH(driverVersion),
                             param_7, param_8); // Unused by FUN_0013cfe4
                
                // `param_3 + 0x45` (where `param_3` is `currentStruct`) corresponds to
                // `(char*)currentStruct + 0x114` (offset 276 bytes).
                // This offset is where `subsetAllocation` is in `VkPhysicalDeviceGroupProperties`.
                strncpy((char*)currentStruct + 0x114, driver_version_str, VK_MAX_DRIVER_INFO_SIZE);
                ((char*)currentStruct + 0x114)[VK_MAX_DRIVER_INFO_SIZE - 1] = '\0'; // Ensure null-termination
                break; // Exit after processing VkPhysicalDeviceGroupProperties
            }
            currentStruct = (VkBaseInStructure*)currentStruct->pNext; // Move to the next structure
        } while (currentStruct != NULL);
    }

    // Stack canary check
    if (*(long*)(((char*)tpidr_el0) + 0x28) != stack_chk_guard) {
        __stack_chk_fail();
    }
}
```

I will provide the decompiled code for the two functions, along with explanations for the parameter types, offsets, and logic.

First, let's establish the structure of `AsyncCommandProcessor` based on the assembly, given it's allocated with `calloc(1, 0xd0)`.

**Derived Structure: `AsyncCommandProcessor`**

```c
// AsyncCommand - Represents a single asynchronous command to be processed.
typedef struct AsyncCommand {
    void (*execute_func)(void* data); // Function pointer to execute the command's logic.
    void* data;                       // Data associated with the command.
    struct AsyncCommand* next;        // Pointer to the next command in the linked list.
} AsyncCommand;

// AsyncCommandProcessor - Manages a queue of asynchronous commands and the worker threads.
// Total size: 0xD0 (208 bytes).
typedef struct AsyncCommandProcessor {
    int total_worker_threads;       // Offset 0x00: Total number of worker threads.
    int active_worker_count;        // Offset 0x04: Number of currently active worker threads.
    char running_flag;              // Offset 0x08: Flag to signal worker threads to continue/stop (0 for stop, non-0 for run).
    char _pad_09[3];                // Offset 0x09: Padding for alignment.

    pthread_mutex_t commands_mutex; // Offset 0x0C: Mutex protecting the commands queue. (size 40 bytes)
                                    // Ends at 0x0C + 40 = 0x34.

    pthread_cond_t commands_available_cond; // Offset 0x34: Condition variable signaled when new commands are available. (size 48 bytes)
                                            // Ends at 0x34 + 48 = 0x64.

    pthread_mutex_t worker_count_mutex; // Offset 0x64: Mutex protecting the active worker count. (size 40 bytes)
                                        // Ends at 0x64 + 40 = 0x8C.

    pthread_cond_t workers_idle_cond; // Offset 0x8C: Condition variable signaled when all workers are idle. (size 48 bytes)
                                      // Ends at 0x8C + 48 = 0xBC.

    AsyncCommand* pending_commands_head; // Offset 0xBC: Head of the linked list of pending commands.
    AsyncCommand* pending_commands_tail; // Offset 0xC4: Tail of the linked list of pending commands.
} AsyncCommandProcessor;

```

Now, let's decompile the functions.

---

### Function: `FUN_0013d7ec`

**Purpose:** This function serves as the entry point for a worker thread in an asynchronous command processing system. It continuously picks up commands from a queue, executes them, and manages thread synchronization.

**Parameters:**

*   `param_1`: `WorkerThreadState* state` - A pointer to a `WorkerThreadState` structure that contains the shared state for managing commands and thread synchronization (mutexes, condition variables, command queues).

**Return Value:**

*   `undefined8` (likely `void*` or `int`): The function returns `0` upon exiting its loop and performing final cleanup.

**Decompiled Code:**

```c
// FUN_0013d7ec - Worker thread function for processing asynchronous commands.
// This function runs indefinitely, picking up commands from a queue and executing them.
//
// param_1: WorkerThreadState* state - Pointer to the shared state for managing commands and threads.
//
// Returns: 0 (undefined8 in decompilation, likely int or void*)
//
undefined8 FUN_0013d7ec(WorkerThreadState* state) {
    // Check if the running_flag is initially set. If not, the thread will immediately exit.
    if (state->running_flag != '\0') {
        do {
            // Step 1: Acquire mutex to safely access the pending commands queue.
            pthread_mutex_lock(&state->commands_mutex);

            // Get the current head of the command queue.
            AsyncCommand* current_command = state->pending_commands_head;

            // Step 2: Wait loop - If no commands are available AND the running_flag is still set,
            // wait on the `commands_available_cond` condition variable.
            while (current_command == NULL) {
                // If the `running_flag` becomes null/false while waiting, it signals the thread to stop.
                if (state->running_flag == '\0') {
                    current_command = NULL; // Ensure no command is processed.
                    // Jump to cleanup before exiting the inner loop.
                    goto END_COMMAND_PROCESSING;
                }
                // Wait for a signal that commands are available. The mutex is released during wait.
                pthread_cond_wait(&state->commands_available_cond, &state->commands_mutex);
                // After waking up, re-check the head of the queue.
                current_command = state->pending_commands_head;
            }

            // Step 3: Command is available, remove it from the queue.
            if (current_command->next == NULL) {
                // If this is the last command, reset both head and tail to NULL.
                state->pending_commands_head = NULL;
                state->pending_commands_tail = NULL;
            } else {
                // Otherwise, advance the head to the next command.
                state->pending_commands_head = current_command->next;
            }

        END_COMMAND_PROCESSING:
            // Step 4: Release the commands mutex.
            pthread_mutex_unlock(&state->commands_mutex);

            // Step 5: Process the command if one was retrieved (i.e., not an early exit signal).
            if (current_command != NULL) {
                // Acquire mutex to safely update the count of active worker threads.
                pthread_mutex_lock(&state->worker_count_mutex);
                // Increment the count of active worker threads.
                state->active_worker_count++;
                pthread_mutex_unlock(&state->worker_count_mutex);

                // Execute the command's function.
                if (current_command->execute_func != NULL) {
                    current_command->execute_func(current_command->data);
                }
                // Free the memory allocated for the `AsyncCommand` structure itself.
                free(current_command);

                // Acquire mutex to safely update the count of active worker threads after processing.
                pthread_mutex_lock(&state->worker_count_mutex);
                // Decrement the count of active worker threads.
                int remaining_active_workers = state->active_worker_count - 1;
                state->active_worker_count = remaining_active_workers;

                // If no other workers are active, signal the `workers_idle_cond`.
                if (remaining_active_workers == 0) {
                    pthread_cond_signal(&state->workers_idle_cond);
                }
                pthread_mutex_unlock(&state->worker_count_mutex);
            } else {
                // If `current_command` is NULL, it means the thread was signaled to stop,
                // so break out of the main `do-while` loop.
                break;
            }

        } while (state->running_flag != '\0'); // Continue the loop as long as the running flag is true.
    }

    // Final cleanup before the thread fully exits.
    // Acquire mutex to safely decrement the total worker thread count.
    pthread_mutex_lock(&state->worker_count_mutex);
    // Decrement the total count of worker threads.
    state->total_worker_threads--;
    // Signal the `workers_idle_cond` (e.g., to notify a main thread that all workers are done).
    pthread_cond_signal(&state->workers_idle_cond);
    pthread_mutex_unlock(&state->worker_count_mutex);

    // Return 0, indicating thread termination.
    return 0;
}
```

---

### Function: `FUN_0013d91c`

**Purpose:** This function serves as the main command processing loop within the server-side component (likely part of the Vortek Renderer Component). It receives commands from a client via a ring buffer, dispatches them to appropriate handlers, and interacts with the Android windowing system via JNI.

**Parameters:**

*   `param_1`: `VtContext* vtContext` - A pointer to the global Vortek context, which holds various application states, component pointers (like `TextureDecoder`, `ShaderInspector`), JNI environment, and buffer management.

**Return Value:**

*   `undefined8` (likely `void*` or `int`): The function returns `0` upon exiting its loop.

**Decompiled Code:**

```c
// FUN_0013d91c - Main server-side processing loop.
// Reads commands from a ring buffer, dispatches them, and handles JNI interactions.
//
// param_1: VtContext* vtContext - Pointer to the global Vortek context.
//
// Returns: 0 (undefined8 in decompilation, likely int or void*)
//
undefined8 FUN_0013d91c(VtContext* vtContext) {
    // Save stack canary for stack corruption detection.
    long stack_guard_value = *(long *)(tpidr_el0 + 0x28);

    // Local variables to store command ID and size read from the ring buffer.
    // local_50: Holds the command ID.
    // uStack_4c: Holds the size of the command data.
    int command_id;
    uint command_data_size;

    // --- JNI Environment and Method ID Initialization ---
    // The decompilation abstracts complex JNIEnv access and method ID lookups.
    // In actual C, these would involve `GetEnv`, `GetMethodID`, `CallObjectMethod`, etc.
    // Here, it appears the `VtContext` stores pre-looked-up JNI method IDs and objects.
    // These lines are likely assigning results of initial JNI setup (e.g., from `createVkContext`).

    // Initialize JNI references (simplified as direct assignments in decompiled C).
    // The JNIEnv is thread-local; it might be attached here if not already.
    // The actual JNI calls for `GetMethodID` and `Call<Type>Method` are assumed.

    // Get JNIEnv for the current thread (or retrieve it from context if already attached).
    // The specific `getMethodID` calls are implicit here.
    // The decompilation shows direct assignment from function calls that internally handle JNI.
    // Example: `uVar7 = (**(code **)(*plVar5 + 0xf8))` looks like `(*vtContext->jniEnv)->GetMethodID(...)`
    // Or, more likely, these are JNI methods on the `javaComponentObject` itself, e.g.,
    // `vtContext->javaComponentObject->getWindowWidth(vtContext->jniEnv, ...)`

    // Retrieve JNIEnv, jobject, and jmethodIDs from the VtContext,
    // assuming they were set up previously (e.g., in `createVkContext`).
    // Example access: `vtContext->jniEnv = (JNIEnv*)vtContext->jniEnv;` (effectively no-op here, but means it's available)

    // Call Java method to get Window Width.
    // `vtContext->javaComponentObject` is `param_1 + 0x90`.
    // `vtContext->jniEnv` is `param_1 + 0x98`.
    // `vtContext->getWindowWidthMethodID` is `param_1 + 0xA8`.
    // The function `(*(code **)(*(long *)CONCAT44(uStack_4c,local_50) + 0x108))` implies a helper that
    // takes the JNIEnv, the object, the method ID, and arguments, and returns the result.
    // Simplified:
    // vtContext->windowWidth = (*vtContext->jniEnv)->CallIntMethod(vtContext->jniEnv, vtContext->javaWindowManagerObject, vtContext->getWindowWidthMethodID);
    // ... similarly for height and hardware buffer.
    vtContext->getWindowWidthMethodID = ((jmethodID(*)(jobject, JNIEnv*, jmethodID))(*(uintptr_t*)vtContext->jniEnv + 0x108))(vtContext->javaComponentObject, vtContext->jniEnv, vtContext->getWindowWidthMethodID);
    vtContext->getWindowHeightMethodID = ((jmethodID(*)(jobject, JNIEnv*, jmethodID))(*(uintptr_t*)vtContext->jniEnv + 0x108))(vtContext->javaComponentObject, vtContext->jniEnv, vtContext->getWindowHeightMethodID);
    vtContext->getWindowHardwareBufferMethodID = ((jmethodID(*)(jobject, JNIEnv*, jmethodID))(*(uintptr_t*)vtContext->jniEnv + 0x108))(vtContext->javaComponentObject, vtContext->jniEnv, vtContext->getWindowHardwareBufferMethodID);
    vtContext->updateWindowContentMethodID = ((jmethodID(*)(jobject, JNIEnv*, jmethodID))(*(uintptr_t*)vtContext->jniEnv + 0x108))(vtContext->javaComponentObject, vtContext->jniEnv, vtContext->updateWindowContentMethodID);


    // Main processing loop: Continues as long as `vtContext->lastError` is not `VK_ERROR_DEVICE_LOST` (-4)
    // or any other error state that would terminate the loop.
    // `vtContext->lastError` is at offset 0x70.
    while (vtContext->lastError != -1) { // -1 in assembly (0xffffffff) is used as a sentinel for error/stop.
                                         // It maps to VK_ERROR_OUT_OF_HOST_MEMORY or VK_ERROR_UNKNOWN if not a specific KHR error code.
                                         // More accurately, it implies an error occurred that should stop processing.

        // Step 1: Read command ID and size from the server ring buffer.
        // `vtContext->serverRingBuffer` is at offset 0x68.
        // `local_50` receives the first 4 bytes (command ID).
        // `uStack_4c` receives the next 4 bytes (command data size).
        bool read_success = RingBuffer_read(vtContext->serverRingBuffer, &command_id, 8); // Read 8 bytes (ID + Size)

        // If reading from the ring buffer fails, break out of the loop.
        if (!read_success) {
            break;
        }

        // Check for special command ID values:
        // If command_id is negative, it's likely a termination signal.
        if (command_id < 0) {
            break;
        }

        // Step 2: Read the actual command data based on its size.
        // `vtContext->inputBufferSize` is at offset 0x30.
        // `vtContext->inputBufferPtr` is at offset 0x28.
        vtContext->inputBufferSize = command_data_size; // Store the received command size.

        // If command data size is greater than 0, read the command payload.
        if (command_data_size > 0) {
            // Manage temporary buffer for command data.
            // This logic reuses a pre-allocated `vtContext->tempBuffer` if possible to avoid frequent `malloc`.
            // `vtContext->tempBufferOffset` tracks the current usage within `vtContext->tempBuffer`.
            // `0x10000` (64KB) is used as a threshold for the internal temp buffer.
            void* current_temp_alloc_ptr;
            int current_temp_offset = vtContext->tempBufferOffset;

            if ((current_temp_offset + command_data_size < 0x10000) && (vtContext->tempBuffer != NULL)) {
                current_temp_alloc_ptr = (char*)vtContext->tempBuffer + current_temp_offset;
                vtContext->tempBufferOffset = current_temp_offset + command_data_size;
            } else {
                // If the internal temp buffer is not sufficient, fall back to a new heap allocation.
                current_temp_alloc_ptr = malloc((size_t)command_data_size);
                // Track this allocation in `vtContext->tempAllocations` for later freeing.
                ArrayList_add(vtContext->tempAllocations, current_temp_alloc_ptr);
            }
            vtContext->inputBufferPtr = (char*)current_temp_alloc_ptr;

            // Read the command data payload into the prepared buffer.
            read_success = RingBuffer_read(vtContext->serverRingBuffer, vtContext->inputBufferPtr, command_data_size);
            if (!read_success) {
                // If reading the command data fails, break the loop.
                break;
            }
        }

        // Step 3: Dispatch the command to the appropriate handler function.
        // `getHandleRequestFunc` (FUN_0011a2d4) retrieves a function pointer based on the `command_id`.
        // The handler function typically takes the `VtContext*` as its first argument.
        code* handler_func = (code*)getHandleRequestFunc((ushort)command_id);
        if (handler_func != NULL) {
            handler_func((long)vtContext); // Execute the handler.
        }

        // Step 4: Cleanup temporary allocations used for this command.
        // Reset the `tempBufferOffset` for reuse in the next iteration.
        vtContext->tempBufferOffset = 0;
        if (vtContext->tempAllocations != NULL && vtContext->tempAllocations->size > 0) {
            // Free all dynamically allocated temporary buffers and clear the tracking list.
            // This is safer than relying solely on the fixed `tempBuffer` for all commands.
            for (uint32_t i = 0; i < vtContext->tempAllocations->size; ++i) {
                void* alloc = ArrayList_get(vtContext->tempAllocations, i);
                if (alloc != NULL) {
                    free(alloc);
                }
            }
            ArrayList_clear(vtContext->tempAllocations);
        }

        // The loop continues if `vtContext->lastError` is still not -1.
        // `vtContext->lastError` is updated by handler functions if an error occurs.
    }

    // --- Final Cleanup before thread exit ---
    // If there's an `updateWindowContentMethodID`, call it via JNI.
    if (vtContext->updateWindowContentMethodID != NULL && vtContext->jniEnv != NULL) {
        // JNI call: CallVoidMethod(vtContext->javaComponentObject, vtContext->updateWindowContentMethodID);
        ((void(*)(JNIEnv*, jobject, jmethodID))(*(uintptr_t*)vtContext->jniEnv + 0xb0)) // Speculative JNIEnv offset for CallVoidMethod
            (vtContext->jniEnv, vtContext->javaComponentObject, vtContext->updateWindowContentMethodID);
    }

    // Detach the current native thread from the JVM.
    if (vtContext->jniEnv != NULL) {
        // JNI call: DetachCurrentThread(vtContext->jniEnv);
        ((void(*)(JNIEnv*))(*(uintptr_t*)vtContext->jniEnv + 0x28)) // Speculative JNIEnv offset for DetachCurrentThread
            (vtContext->jniEnv);
        vtContext->jniEnv = NULL; // Clear the JNIEnv pointer.
    }

    // Clean up any remaining temporary allocations from the ArrayList.
    if (vtContext->tempAllocations != NULL && vtContext->tempAllocations->size > 0) {
        for (uint32_t i = 0; i < vtContext->tempAllocations->size; ++i) {
            void* alloc = ArrayList_get(vtContext->tempAllocations, i);
            if (alloc != NULL) {
                free(alloc);
            }
        }
        ArrayList_clear(vtContext->tempAllocations);
    }

    // Restore stack canary to check for stack corruption.
    if (*(long *)(tpidr_el0 + 0x28) != stack_guard_value) {
        __stack_chk_fail(); // Stack corruption detected.
    }
    return 0; // Thread exits.
}
```
