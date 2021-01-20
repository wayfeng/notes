# OpenVINO Notes

## Source code

Get source code of OpenVINO toolkit from github.com. Choose specific release by useing tag.

```bash
git clone -b 2021.1 https://github.com/openvinotoolkit/openvino.git
git submodule update --init --recursive
```

Create =cscope= cross reference.

```bash
find . \
  \( -path "*inference-engine/src/*" \
     -o -path "*inference-engine/include/*" \
     -o -path "./build*" -prune \) \
  -a \( -name "*.[ch]" -o -name "*.[ch]pp" \) > /tmp/iefiles
cscope -b -i /tmp/iefiles
```

## Build Inference Engine on Ubuntu

## Build Inference Engine on Alpine

### Build OpenCV

### Build TBB.
```bash
apk update
apk add gcc g++ git make linux-headers
git clone --depth 1 -b v2020.3 https://github.com/oneapi-src/oneTBB.git build_tbb
cd build_tbb
make tbb
mkdir -p /opt/tbb/lib
cp -r include /opt/tbb/
cp -a build/linux_intel64_gcc_cc9.3.0_libc_kernel5.4.0_release/libtbb.so* /opt/tbb/lib/
```

Before building IE, add 'TBBROOT=/opt/tbb' to cmake configuration.
```bash
mkdir build
cd build
cmake -DTBBROOT=/opt/tbb ..
```

### Build IE

### Test
