# Jetson-Stream-Lite

A trimmed version of [jetson-inference](https://github.com/dusty-nv/jetson-inference).

# How to build it

```
apt-get update
apt-get install git cmake libpython3-dev python3-numpy
apt-get install libglew-dev
mkdir build && cd build && cmake ../ && make -j8
ldconfig
```