# Installing SIBR Viewer on Ubuntu 20.04 - My Installation Experience

So you want to install SIBR viewer? This took me way longer than expected to figure out, and honestly, half the time I don't even know why things broke or why they suddenly started working. But I hope you'll have an easier time setting up SIBR after going through these steps. I used Claude to help me with the installation

## What is SIBR anyway?

SIBR (System for Image-Based Rendering) is this rendering framework that's mainly used for neural rendering and 3D Gaussian splatting visualization. Basically, you can see the trained models and can also see the training itself happening in real-time.

## Prerequisites (The Boring But Necessary Stuff)

First things first, let's get your system ready. I'm assuming you're on Ubuntu 20.04 like me.

```bash
sudo apt update
sudo apt upgrade
```

Yeah, I know, everyone says this, but seriously do it. I skipped it once and weird things happened.

## Step 1: Install CMake from Source (Ubuntu 20.04's version is too old)

The default CMake version in Ubuntu 20.04 is 3.16, but SIBR needs a newer version. I had to build CMake from source because the apt version wasn't cutting it.

```bash
# Remove old CMake if you have it
sudo apt remove cmake

# Install dependencies for building CMake
sudo apt install build-essential libssl-dev

# Download and build CMake
cd ~
wget https://cmake.org/files/v3.25/cmake-3.25.2.tar.gz
tar -xzvf cmake-3.25.2.tar.gz
cd cmake-3.25.2

# Configure and build
./bootstrap
make -j$(nproc)
sudo make install

# Update PATH to use the new CMake
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify version
cmake --version
```

You should see CMake 3.25.2 or whatever version you installed.

## Step 2: Install Basic Build Tools

```bash
sudo apt install build-essential git pkg-config
```

## Step 3: Install Dependencies (Where Things Start Getting Tricky)

```bash
# Graphics libraries - these are crucial
sudo apt install libglew-dev libglfw3-dev libgl1-mesa-dev libglu1-mesa-dev

# Image processing stuff
sudo apt install libfreeimage-dev libassimp-dev

# Math libraries
sudo apt install libeigen3-dev libboost-all-dev

# X11 stuff (trust me, you'll need these)
sudo apt install libx11-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev
```

## Step 4: The OpenCV Build Process

Here's where I spent most of my time. SIBR needs OpenCV to be built with OpenCV-Contrib, when I tried running it with just OpenCV, it threw me errors because it couldn't find some edge detection stuff that's in contrib.

### Remove any existing OpenCV (optional but recommended)

```bash
sudo apt remove libopencv-dev python3-opencv
sudo apt autoremove
```

### Install OpenCV dependencies

```bash
sudo apt install libavcodec-dev libavformat-dev libswscale-dev
sudo apt install libgstreamer-plugins-base1.0-dev libgstreamer1.0-dev
sudo apt install libgtk-3-dev libpng-dev libjpeg-dev libopenexr-dev libtiff-dev libwebp-dev
```

### Build OpenCV from source (this takes a while)

```bash
# Create a workspace
mkdir ~/opencv_build && cd ~/opencv_build

# Clone OpenCV and OpenCV contrib
git clone https://github.com/opencv/opencv.git
git clone https://github.com/opencv/opencv_contrib.git

# Switch to a stable version (I used 4.5.5, because google and claude said so, but you can try newer ones)
cd opencv
git checkout 4.5.5
cd ../opencv_contrib
git checkout 4.5.5
cd ..

# Create build directory
mkdir build && cd build

# Configure CMake (you can adjust the paths according to how you set the files)
cmake -D CMAKE_BUILD_TYPE=RELEASE \
      -D CMAKE_INSTALL_PREFIX=/usr/local \
      -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib/modules \
      -D OPENCV_GENERATE_PKGCONFIG=ON \
      -D BUILD_EXAMPLES=OFF \
      -D INSTALL_PYTHON_EXAMPLES=OFF \
      -D INSTALL_C_EXAMPLES=OFF \
      -D BUILD_opencv_python2=OFF \
      -D BUILD_opencv_python3=ON \
      ..

# Build it (takes 30-60 minutes depending on your system)
make -j$(nproc)

# Install
sudo make install
sudo ldconfig
```

**Note**: If you get weird errors during the make process, try `make -j2` instead of `make -j$(nproc)`. Sometimes using all cores causes issues like the system freezing. It's a good practise to use `nproc-2` cores, i.e., two less than what you have on your system.(My system had 4 cores, hence the `make -j2`) 

## Step 5: Clone and Build SIBR

```bash
# Clone the repo
git clone https://gitlab.inria.fr/sibr/sibr_core.git
cd sibr_core
git checkout fossa_compatibility

# Create build directory
mkdir build && cd build
```

Now here's where I hit my first major roadblock. The basic cmake command didn't work for me:

```bash
# This didn't work initially (but try it first)
cmake .. -DCMAKE_BUILD_TYPE=Release
```

If everything was followed correctly up till here, then the above command should throw no errors. If that still doesn't work, try this approach:

```bash
# This is what actually worked for me
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DOpenCV_DIR=/usr/local/lib/cmake/opencv4 \
         -DOPENCV_EXTRA_MODULES_PATH=/usr/local/include/opencv4
```

Now build it:

```bash
make -j$(nproc)
```

## Problems I Faced (And How I "Fixed" Them)

### Problem 1: "Could not find OpenCV"
**Error**: CMake couldn't find OpenCV even though I just installed it.

**Solution**: I honestly don't know why this happened, but specifying the OpenCV directory explicitly in the cmake command fixed it. Sometimes life is weird like that.

### Problem 2: "GL/glew.h not found"
**Error**: Missing GLEW headers during compilation.

**Solution**: 
```bash
sudo apt install libglew-dev
```

I thought I already installed this, but apparently not? Or maybe I installed it wrong the first time? Restarting the system after the install helped

### Problem 3: "X11 errors" when running SIBR viewer
**Error**: The viewer would crash immediately with X11-related errors.

**Solution**: Installing those X11 development packages fixed it:
```bash
sudo apt install libx11-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev
```

### Problem 4: "libfreeimage.so not found"
**Error**: Runtime error about missing FreeImage library.

**Solution**: 
```bash
sudo apt install libfreeimage3 libfreeimage-dev
sudo ldconfig
```

I don't fully understand what it does, but it updates the linker cache and fixes library loading issues.

### Problem 5: Segmentation fault on startup
**Error**: SIBR viewer would crash with a segfault.

**Solution**: This one was weird. It turned out to be a graphics driver issue. I had to update my NVIDIA drivers:

```bash
# Check what driver you're using
nvidia-smi

# Update drivers (this will vary based on your GPU)
sudo apt install nvidia-driver-470
sudo reboot
```

I don't know why older drivers caused segfaults, but updating to newer ones fixed it. I checked online later on NVIDIA's community and some users did say that old drivers for CUDA 11.3 were faulty and the newer ones had compatibility with 11.3  

## Testing Your Installation

If everything went well, you should be able to run:

```bash
cd sibr_core/build/install/bin
./SIBR_gaussianViewer_app
```


## Final Thoughts

This installation process took me about 2 days to figure out completely. The CMake version issue and OpenCV building from source were the biggest pain points. I don't know why SIBR can't just use the system OpenCV, but apparently it has specific requirements.

If you're still having issues, try:
1. Double-checking that you have the right CMake version (3.18+)
2. Making sure all graphics drivers are properly installed
3. Verifying that OpenCV was built and installed correctly
4. Starting over with a clean build directory if things get too messed up

The process is pretty involved, but once it's working, the viewer performs well.

Also, I am still learning, I would apreciate any kind of input you have.


---

*Hope this helps if you're going through the same installation process.*