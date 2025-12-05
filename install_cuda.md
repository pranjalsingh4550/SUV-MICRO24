# Installing CUDA Toolkit 525.60.13

If you want to run SUV on a system with an existing CUDA installation, we
recommend downloading libraries in the home directory.

## CUDA Toolkit and Runtime Components

- Driver - in memory, not in disk.
- CUDA Runtime - `libcudart.so`
- NVIDIA Management Library - `libnvidia-ml.so`
- Firmware - `/lib/firmware/nvidia/525.60.13`

## Download the CUDA Toolkit

- Download the correct version (525.60.13) from
[https://developer.nvidia.com/cuda-12-0-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=runfile_local]
(https://developer.nvidia.com/cuda-12-0-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=22.04&target_type=runfile_local)

```sh
wget https://developer.download.nvidia.com/compute/cuda/12.0.0/local_installers/cuda_12.0.0_525.60.13_linux.run
```

To inspect the run script, use `less(1)`. It does not load the entire file in memory.  
You can also use the `--help` message.

## Local Installation

Do **NOT** run any of this as root. Instead, use environment variables
to use the correct libraries/headers.

Install the runfile.

```sh
mkdir cuda-525
install_path=`realpath cuda-525` # backticks, not single quotes.

# can take tens of minutes
sh cuda_12*run                          \
    --no-man-page                       \
    --kernel-output-path=$install_path  \
    --toolkit                           \
    --installpath=$install_path         \
    -m=kernel-open                      \
    --silent
```

Then, extract the contents of the _second_ `run` file, inside the first,
to get `libnvidia-ml.so`.

```sh
mkdir cuda-extract
extract_path=`realpath cuda-extract`
sh cuda_12*run                          \
    --extract=$extract_path
cd $extract_path
sh NVIDIA-Linux-x86_64-525.60.13.run --extract-only

# check!
ls NVIDIA-Linux-x86_64-525.60.13/libnvidia-ml*
cp NVIDIA-Linux-x86_64-525.60.13/libnvidia-ml* $install_path/lib64 -P
```

The linker expects specific suffixes. Make a lot of symlinks to be safe, for
`nvml`, `libcudart`, and `libstdc++`.

```sh
cd $install_path/lib64
ln -s libnvidia-ml.so.525.60.13 libnvidia-ml.so
ln -s libnvidia-ml.so.525.60.13 libnvidia-ml.so.1
ln -s libnvidia-ml.so.525.60.13 libnvidia-ml.so.12
ln -s libnvidia-ml.so.525.60.13 libnvidia-ml.so.12.0

# The C++ library isn't supposed to be here, but this is sometimes needed.
# The linker isn't very good with -lstdc++.
ln -s /usr/lib/x86_64-linux-gnu/libstdc++.so.6
ln -s libstdc++.so.6 libstdc++.so
ln -s libstdc++.so.6 libstdc++.so.1
```

Now, `$install_path/lib64` should have the required libraries.  

Finally, add the new installation to `startup.sh`. (Replace `install_path`
with the actual path.)

```sh
export LD_LIBRARY_PATH=$install_path/lib64:$LD_LIBRARY_PATH
export CPATH="$install_path/include:/usr/include/c++/11/:/usr/include/x86_64-linux-gnu/c++/11:$CPATH"
export PATH="$install_path/bin:$PATH"
```

## Copy Firmware

```sh
cd $runfile_path

mkdir /lib/firmware/nvidia/525.60.13
sudo cp NVIDIA-Linux-x86_64-525.60.13/firmware/* /lib/firmware/nvidia/525.60.13
```

The driver can be configured to _not_ use the firmware files.
This may have a performance penalty.
```sh
# Details in $runfile_path/NVIDIA-Linux-x86_64-525.60.13/html/gsp.html

sudo insmod nvidia.ko                       \
        NVreg_EnableGpuFirmware=0           \
        NVreg_OpenRmEnableUnsupportedGpus=1
```

## Loading the Driver

Manually load and unload the driver using `insmod`, `rmmod`, and `modprobe`.  
This does _not_ modify the global or local CUDA installations - the driver runs
entirely in memory.

Doing a `sudo make modules_install`, as the instructions in the driver's `README.md`
suggests, might change the installed driver version, and break the system.

To check the currently loaded driver version:
```sh
cat /proc/driver/nvidia/version
```
