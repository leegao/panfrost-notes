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
