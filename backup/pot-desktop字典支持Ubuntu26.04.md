
# release
- https://github.com/Ethan-a2/saladict/releases


# 原因
- 构建脚本通过 pkg-config 查找 javascriptcoregtk-4.0和webkit2gtk-4.0，但 Ubuntu 26.04 只有 javascriptcoregtk-4.1和webkitgtk-4.1。
- 因此需要使用 .pc进行兼容。


# 方法

1. 第一步：创建 .pc 兼容文件，让 pkg-config 能找到 4.0：
```
sudo tee /usr/lib/x86_64-linux-gnu/pkgconfig/javascriptcoregtk-4.0.pc > /dev/null << 'EOF'
prefix=/usr
exec_prefix=${prefix}
libdir=/usr/lib/x86_64-linux-gnu
includedir=${prefix}/include
Name: JavaScriptCoreGTK+
Description: GTK+ version of the JavaScriptCore engine (compat shim for 4.1)
Version: 2.52.3
Requires: glib-2.0 gobject-2.0
Libs: -L${libdir} -ljavascriptcoregtk-4.1
Cflags: -I${includedir}/webkitgtk-4.1
EOF


sudo tee /usr/lib/x86_64-linux-gnu/pkgconfig/webkit2gtk-4.0.pc > /dev/null << 'EOF'
prefix=/usr
exec_prefix=${prefix}
libdir=/usr/lib/x86_64-linux-gnu
includedir=${prefix}/include

Name: WebKitGTK
Description: Web content engine for GTK (compat shim for 4.1)
URL: https://webkitgtk.org
Version: 2.52.3
Requires: glib-2.0 gtk+-3.0 libsoup-3.0 javascriptcoregtk-4.0
Libs: -L${libdir} -lwebkit2gtk-4.1
Cflags: -I${includedir}/webkitgtk-4.1
EOF
```
- 验证：
```
pkg-config --libs --cflags javascriptcoregtk-4.0
pkg-config --libs --cflags webkit2gtk-4.0
```
- 输出: -L... -ljavascriptcoregtk-4.1 -I.../webkitgtk-4.1



2. 第二步：仅靠 .pc 文件还不够，编译通过后链接阶段报了另一个错：
	- rust-lld: error: unable to find library -ljavascriptcoregtk-4.0
	- 因为链接器直接找 libjavascriptcoregtk-4.0.so 文件，所以还需要创建 .so 符号链接：
```
sudo ln -sf /usr/lib/x86_64-linux-gnu/libjavascriptcoregtk-4.1.so.0.10.11 \
            /usr/lib/x86_64-linux-gnu/libjavascriptcoregtk-4.0.so
sudo ln -sf /usr/lib/x86_64-linux-gnu/libjavascriptcoregtk-4.1.so.0.10.11 \
            /usr/lib/x86_64-linux-gnu/libjavascriptcoregtk-4.0.so.0
```



# patch
- https://github.com/allentown521/saladict/commit/6d9da28943cfb4c1784aadc6a22ba7450cbb0a53

