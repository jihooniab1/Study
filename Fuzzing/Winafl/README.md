# WinAFL
Study WinAFL

## How to Build
1. Download DynamoRIO
Download DynamoRIO package file and unzip it from -> https://dynamorio.org/page_download.html 

2. Install cmake
Install cmake from ->https://cmake.org/download/

3. WinAFL clone
```
git clone https://github.com/googleprojectzero/winafl.git

git submodule update --init --recursive
```

4. Install Visual Studio
https://visualstudio.microsoft.com/ko/vs/community/ -> Select **Desktop development with C++**

5. Open Visual Studio Command Prompt (or Visual Studio x64 Win64 Command Prompt)
Need 64-bit winafl.dll build to fuzz 64-bit target

6. Run script(64-bit)
```
cd winafl
mkdir build64
cd build64
cmake -G"Visual Studio 17 2022" -A x64 .. -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -DDynamoRIO_DIR=C:\path\to\DynamoRIO\cmake -DINTELPT=1 -DUSE_COLOR=1
cmake --build .  --config Release 
```

Following is build configuration options

- -DDynamoRIO_DIR=..\path\to\DynamoRIO\cmake - Needed to build the winafl.dll DynamoRIO client
- -DTINYINST=1 - Enable TinyInst mode
- -DINTELPT=1 - Enable Intel PT mode
- -DUSE_COLOR=1 - color support
- -DUSE_DRSYMS=1 - Drsyms support (use symbols when available to obtain -target_offset from -target_method)

