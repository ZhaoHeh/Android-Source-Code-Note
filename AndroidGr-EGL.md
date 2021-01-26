# EGL

在C++编译中，“.in”类型的文件可以被C++源文件（.h或.cpp类型）通过“#include”宏包含。[frameworks/native/opengl/libs/platform_entries.in][platform_entries_link]文件的内容就被包含在[frameworks/native/opengl/libs/hooks.h][hooks_h_link]文件中。  

[platform_entries_link]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/platform_entries.in;l=18
[hooks_h_link]:https://cs.android.com/android/platform/superproject/+/master:frameworks/native/opengl/libs/hooks.h;l=60

```json
// The headers modules are in frameworks/native/opengl/Android.bp.
ndk_library {
    name: "libEGL",
    symbol_file: "libEGL.map.txt",
    first_version: "9",
    unversioned_until: "current",
}
// ... 省略代码
cc_library_shared {
    name: "libEGL",
    defaults: ["egl_libs_defaults"],
    llndk_stubs: "libEGL.llndk",
    srcs: [
        "EGL/egl_tls.cpp",
        "EGL/egl_cache.cpp",
        "EGL/egl_display.cpp",
        "EGL/egl_object.cpp",
        "EGL/egl_layers.cpp",
        "EGL/egl.cpp",
        "EGL/eglApi.cpp",
        "EGL/egl_platform_entries.cpp",
        "EGL/Loader.cpp",
        "EGL/egl_angle_platform.cpp",
    ],
    shared_libs: [
// ... 省略代码
    ],
    static_libs: [
        "libEGL_getProcAddress",
        "libEGL_blobCache",
    ],
    ldflags: ["-Wl,--exclude-libs=ALL,--Bsymbolic-functions"],
    export_include_dirs: ["EGL/include"],
    stubs: {
        symbol_file: "libEGL.map.txt",
        versions: ["29"],
    },
    header_libs: ["libsurfaceflinger_headers"],
}
// 代码路径：frameworks/native/opengl/libs/Android.bp
```
