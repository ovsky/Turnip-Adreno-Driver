# Adreno Mesa Drivers Toolkit

A comprehensive, automated toolset for sharing, building, and distributing the latest upstream Mesa (Turnip/Zink) drivers for Qualcomm Adreno GPUs.

This project aims to bridge the gap between upstream Linux graphics development and end-user Android emulation/gaming by providing an optimized CI/CD pipeline and local build system for Turnip drivers.

## 🤝 Acknowledgments & Inspirations

This project stands on the shoulders of giants. It is highly inspired by and deeply grateful to the pioneering work done by:
* **[StevenMXZ / Adreno-Tools-Drivers](https://github.com/StevenMXZ/Adreno-Tools-Drivers/)**
* **[K11MCH1 / AdrenoToolsDrivers](https://github.com/K11MCH1/AdrenoToolsDrivers)**

Without their continuous effort to democratize and distribute bleeding-edge Adreno drivers, the landscape of Android gaming and emulation (via Skyline, Strato, Yuzu, Vita3K, etc.) would not be where it is today. Thank you for your immense contributions to the community.

---

## 🧠 Technical Overview & Architecture

Building a graphics driver for Android user-space that effectively intercepts and overrides the system's vendor implementation requires navigating a complex labyrinth of APIs, linking protocols, and hardware architectures.

### 1. The Freedreno & Turnip Stack
Qualcomm's Adreno GPUs utilize a Tile-Based Deferred Rendering (TBDR) architecture, distinct from traditional immediate-mode desktop GPUs. Upstream Mesa supports this via the **Freedreno** project. Within Freedreno, **Turnip** is the open-source Vulkan implementation. By utilizing this toolset, we compile Turnip directly from the latest Mesa `main` branch, enabling bug fixes and Vulkan extensions (like `VK_KHR_portability_subset` and dynamic rendering features) months or years before they reach official OEM firmware.

### 2. Bypassing DRM for KGSL
On standard Linux environments, Mesa interacts with the GPU via the Direct Rendering Manager (DRM) and Kernel Mode Setting (KMS). However, Android devices abstract GPU access through Qualcomm's proprietary **KGSL (Kernel Graphics Support Layer)**. The drivers built via this toolset are compiled with specialized Meson flags that strip out the DRM dependency, substituting it with KGSL bindings. This allows the Turnip driver to communicate directly with the Adreno GPU using Android's native `ioctl` calls without requiring root access.

### 3. User-Space Injection (AdrenoTools)
Because Android heavily restricts library loading via the Bionic linker (relying on `sphall` namespaces and vendor partitions), we cannot easily overwrite the system `libvulkan.so`. Instead, these drivers are packaged to be consumed by `libadrenotools`. This library works by:
* Hooking the Android Vulkan loader in user-space.
* Patching the custom driver ELF headers (e.g., overriding `DT_SONAME`).
* Redirecting application Vulkan calls into our locally extracted, freshly compiled Turnip driver.

### 4. Zink: OpenGL over Vulkan
Alongside Turnip, this toolkit can optionally compile **Zink**. By running a highly optimized OpenGL translation layer over our Turnip Vulkan driver, we provide performant, bug-free OpenGL ES and Desktop OpenGL support for legacy translation layers, fully bypassing the notoriously unstable closed-source Adreno OpenGL drivers.

---

## 🛠️ Build Toolchain

This repository leverages a robust, scriptable pipeline designed for predictability and performance:
* **Compiler:** LLVM / Clang (via Android NDK)
* **Build System:** Meson + Ninja
* **Target Architectures:** `aarch64`
* **Optimization:** `-O3`, LTO (Link Time Optimization) where applicable, and `-Bsymbolic` to prevent dynamic symbol interposition overhead.

### Prerequisites
* Linux environment (Ubuntu 22.04 LTS or Arch Linux recommended)
* Android NDK (r25c or higher recommended)
* Python 3.10+, Meson, Ninja, Flex, Bison, `pkg-config`

### Local Compilation

```bash
# 1. Clone the repository and submodules
git clone https://github.com/YourName/Adreno-Mesa-Drivers.git
cd Adreno-Mesa-Drivers

# 2. Set your environment variables
export NDK_HOME=/path/to/android-ndk

# 3. Run the automated build script
./build_turnip.sh --release
```

The build script will:
1. Fetch the latest Mesa source tree.
2. Generate a cross-compilation file for Meson targeting `aarch64-linux-android`.
3. Configure Mesa with `-Dgallium-drivers=freedreno,zink`, `-Dvulkan-drivers=freedreno`, and `-Dfreedreno-kmds=kgsl`.
4. Compile the target `libvulkan_freedreno.so`.
5. Package the resulting binary, alongside the required `meta.json`, into a flashable/loadable `.zip` archive.

---

## 🚀 CI/CD Pipeline

This repository includes GitHub Actions workflows (`.github/workflows/build.yml`) that automatically pull the latest Mesa commits, build the driver matrix, and publish release artifacts.

Developers simply need to fork the repository, enable GitHub Actions, and watch the bleeding-edge drivers roll into the "Releases" tab.

## 📜 License

The build scripts and tools in this repository are provided under the MIT License.

*Note: The resulting driver binaries are subject to the upstream Mesa license (MIT / X11), and the Android NDK components are subject to their respective Google licenses.*