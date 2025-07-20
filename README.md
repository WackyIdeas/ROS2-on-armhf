# ROS2 on armv7

ROS2 no longer provides prebuilt images for `armhf` architectures (Raspberry Pi 2, 3), but still offers Tier 3 support, which requires the user to build ROS2 from source.

Sources: 
- [Building ROS2 on Ubuntu](https://docs.ros.org/en/kilted/Installation/Alternatives/Ubuntu-Development-Setup.html)
- [Installing ROS2 on Ubuntu](https://docs.ros.org/en/kilted/Installation/Alternatives/Ubuntu-Install-Binary.html)
- Arch Wiki articles for [QEMU](https://wiki.archlinux.org/title/QEMU) and [Docker](https://wiki.archlinux.org/title/Docker)

## Dependencies

Arch Linux:

```bash
$ sudo pacman -S qemu-full qemu-user-static qemu-user-static-binfmt docker docker-buildx
```

```bash
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

Confirm that the system can run other architectures:

```bash
$ sudo docker buildx ls --no-trunc
NAME/NODE     DRIVER/ENDPOINT   STATUS    BUILDKIT   PLATFORMS
default*      docker                                 
 \_ default    \_ default       running   v0.23.2    linux/amd64, linux/amd64/v2, linux/386, linux/arm64, linux/riscv64, linux/ppc64, linux/ppc64le, linux/s390x, linux/mips64le, linux/mips64, linux/loong64, linux/arm/v7, linux/arm/v6
```

For our purposes, we're going to need `linux/arm/v7`

Pull the Ubuntu 22.04 LTS image for armv7 (This is the last version of Ubuntu to feature prebuilt images compatible with RPi2):

```bash
$ sudo docker pull --platform=linux/arm/v7 arm32v7/ubuntu:22.04

$ sudo docker images # confirm the image is downloaded 
REPOSITORY       TAG       IMAGE ID       CREATED       SIZE
arm32v7/ubuntu   22.04     2ce6f7cf5275   2 days ago    56.4MB
```

To test if the docker image works correctly: 

```bash
$ sudo docker run --platform=linux/arm/v7 -it --rm arm32v7/ubuntu:22.04 bash -c "echo hello world"
```

Expected output:

```
hello world
```

Run a bash session:

```bash
$ sudo docker run --platform=linux/arm/v7 -it --rm arm32v7/ubuntu:22.04 bash
# In Docker
cat /etc/*release | grep PRETTY_NAME # Check Ubuntu version
PRETTY_NAME="Ubuntu 22.04.5 LTS"
```

Following the guide from [Ubuntu (source)](https://docs.ros.org/en/kilted/Installation/Alternatives/Ubuntu-Development-Setup.html):

### Locale 

The docker image will come with a minimal setup which includes only the POSIX locale. Since ROS2 requires UTF-8, we need to install the appropriate locale:

```bash
apt update && apt install locales curl
locale-gen en_US en_US.UTF-8
update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
locale # verify locale
```

If correctly done, everything should have the value `"en_US.UTF-8"`.

### Universe repository 

```bash
apt install software-properties-common
add-apt-repository universe
```

Note: During installation apt will halt progress and ask to set the timezone.

Download `ros2-apt-sources.deb` to `/tmp`:

```bash
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo $VERSION_CODENAME)_all.deb"
dpkg -i /tmp/ros2-apt-source.deb
```

Installing more dependencies:

```bash
apt update && apt install -y python3-flake8-blind-except python3-flake8-class-newline python3-flake8-deprecated python3-mypy python3-pip python3-pytest python3-pytest-cov python3-pytest-mock python3-pytest-repeat python3-pytest-rerunfailures python3-pytest-runner python3-pytest-timeout ros-dev-tools
```

### Building ROS2 base

This will install the base set of packages required for ROS2. For a more comprehensive set of packages, build `desktop` or `desktop_full` (ROS2 Github releases seem to build `desktop` by default)

```bash
apt upgrade 
cd /opt
mkdir -p ros2/src 
cd ros2
vcs import --input https://raw.githubusercontent.com/ros2/ros2/jazzy-release/ros2.repos src
git clone https://github.com/ros2/variants.git -b jazzy src/ros2/variants # For ros_base
rosdep init # Needs root to write to /etc/ros
rosdep update # Will throw warning if ran as root, but this seems to only affect file permissions which we can fix later
rosdep install -r --from-paths src --ignore-src -y --rosdistro jazzy --skip-keys "fastcdr rti-connext-dds-7.3.0 urdfdom_headers"
colcon build --packages-up-to ros_base --merge-install --cmake-args -DCMAKE_CXX_FLAGS="-Wno-maybe-uninitialized" -DCMAKE_BUILD_TYPE=Release
```

To install examples for testing, packages `example_interfaces`, `demo_nodes_py` and `demo_nodes_cpp` can be built with `--packages-select`:

```bash
colcon build --packages-select example_interfaces demo_nodes_cpp demo_nodes_py --merge-install --cmake-args -DCMAKE_CXX_FLAGS="-Wno-maybe-uninitialized" -DCMAKE_BUILD_TYPE=Release
```

## NOTE: 
- For faster compilation, pass `-G Ninja` as a CMake argument, or run `export CMAKE_MAKE_PROGRAM="make -j x"` before running colcon, replacing `x` with the number of cores on your system.
- This installs ROS2 Jazzy, leaving out `--rosdistro jazzy` will implicitly mean building the latest distro of ROS2 which as of 2025-07-17 is `kilted`. 
- May require [manual intervention on Ubuntu 22.04 LTS](https://github.com/ros/resource_retriever/pull/64/files#diff-8daddde267e48c7092fd992169e35576ab10c795fe1e6fddd91f3bafa294ee0c) as `resource_retriever` may try to pull a version of libcurl not present in `jammy`. This also means that `libcurl_vendor` can be omitted via `--packages-skip libcurl_vendor` as this is essentially telling `resource_retriever` to use the system-provided libcurl library.
- Colcon seems to implicitly treat all warnings as errors during compilation, which might make `fastdds` fail due to a possible use of uninitialized variables, so it's recommended to pass -Wno-maybe-uninitialized to the compiler flags. Everything else seems to be working fine.
- As the compilation is happening within a Docker image, running things using sudo as a non-root user will generate an error:

```
sudo: effective uid is not 0, is /usr/bin/sudo on a file system with the 'nosuid' option set or an NFS file system without root privileges?
```

but running everything as root seems to work fine, with the catch that file permissions need to be fixed after compilation using `chown`.


Once everything has been compiled, we can extract this from the container:

```bash
# From the host machine
$ sudo docker ps 
# Output should show the currently running containers and their IDs
$ sudo docker cp abcd:/root/ros2 /target/path/on/host/machine
```

Where `abcd` are the first 4 digits of the docker's container ID, which is usually enough to identify it. This can be later pushed to the Pi using `rsync`:

```bash
$ rsync -avP /target/path/on/host/machine pi@raspberry.local:/home/pi/ros2

# On the Pi
$ sudo mv ~/ros2 /opt/ros2
```


## Setting up Ubuntu 22.04 on the Raspberry Pi 2

### Wait for Network to be configured service hangs and fails 

To fix this, remove the `optional: true` line in `/etc/netplan/50-cloud-init.yaml` and reboot.

### Hostname isn't working

Ubuntu 22.04 doesn't come with `avahi-daemon` installed by default, which is required for mDNS hostname resolution to work. This can be resolved by installing the package and rebooting the system.

### Setting up the locale 

Just like in the build process, the locale needs to be updated to work correctly. This can be either done the same way as in the process, or using `raspi-config`.

### Including ROS repository 

```bash
$ sudo apt install curl software-properties-common tar bzip2 wget locales raspi-config -y
$ sudo add-apt-repository universe
```

### Setting up 

```bash
$ export ROS_DISTRO=jazzy
$ export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
$ curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo $VERSION_CODENAME)_all.deb"
$ sudo dpkg -i /tmp/ros2-apt-source.deb
$ sudo apt update && sudo apt install ros-dev-tools
```

The system will ask to restart outdated systemd services after this.

### Installing dependencies using rosdep

Download the ROS2 binaries from the [Releases](https://github.com/WackyIdeas/ROS2-on-armhf/releases/latest) page:

```bash
$ cd /opt
$ sudo wget https://github.com/WackyIdeas/ROS2-on-armhf/releases/download/ros2-jazzy/ros2-jazzy-20250720-linux-jammy-armhf.tar.bz2
$ sudo tar xf ros2-jazzy-20250720-linux-jammy-armhf.tar.bz2
$ sudo rm ros2-jazzy-20250720-linux-jammy-armhf.tar.bz2
```

```bash
$ sudo apt upgrade # Reboot recommended
$ sudo apt update
$ sudo apt install -y python3-rosdep
$ sudo rosdep init
$ rosdep update
$ rosdep install --from-paths /opt/ros2/install/share/ --ignore-src -y --rosdistro $ROS_DISTRO --skip-keys "cyclonedds fastcdr fastrtps iceoryx_binding_c rmw_connextdds rti-connext-dds-6.0.1 urdfdom_headers"
```

Finally, append the following line to `.bashrc` to source the ROS2 environment automatically on login (requires starting a new bash session):

```bash
source /opt/ros2/setup.bash
```

## Testing 

To test ROS2 functionality, run the following in two separate bash sessions on the Pi:

Session 1:
```bash
$ ros2 run demo_nodes_cpp talker
```

Session 2:
```bash
$ ros2 run demo_nodes_py listener
```


## Additional notes if building ROS2 Jammy for later Ubuntu releases:

- During the docker setup, omitting the version number will implicitly pull the latest Ubuntu image
- The latest versions of Ubuntu (24.04.02 LTS as of 2025-07-15) no longer ship with prebuilt armhf images. This means that we need to install something like 22.02 LTS onto the Raspberry Pi and upgrade the system to the new version using `do-release-upgrade`.
- As of 2025-07-15, pkg_resources is marked as deprecated.
- ROS_PYTHON_VERSION is unset, so ROS implicitly uses the value 3 instead, which doesn't seem to be a problem.
- Ubuntu noble has changed the package naming scheme for certain packages (notably dependencies for `rosdep` packages `libqt5-gui`, `libqt5-widgets`, `libqt5-core`, `libqt5-opengl`), so these should be omitted when running `rosdep`. Everything builds fine despite this, but the parts of ROS2 that rely on the GUI haven't been tested, so expect things not to work.


