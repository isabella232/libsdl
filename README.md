# qmd.graphics.test

Host: X64+Win10+VS2019
Host: X64+Ubuntu18

"C:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvarsall.bat" x64
mkdir buildwin
cd buildwin
cmake --build . --target help
cmake -G "NMake Makefiles" ..
cmake -G "NMake Makefiles" -DSDL_TEST=ON ..
cmake --build .
