---
layout: post
title:  "mac上编译安装nmatrix小记"
date:   2014-04-11 10:06:25
categories: ruby mac nmatrix
---

NMatrix是SciRuby下推出的第一个成熟的项目，主要用于ruby环境下的矩阵计算  
下面是简要的安装记录

安装环境是 mac book air: Mavericks

```bash
sudo ln -s /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.8.sdk/usr/lib/libatlas.dylib /usr/lib/libatlas.dylib`

sudo sudo ln -s /System/Library/Frameworks/Accelerate.framework/Versions/Current/Frameworks/vecLib.framework/Versions/Current/Headers/cblas.h /usr/include/cblas.h

unset C_INCLUDE_PATH
unset CPLUS_INCLUDE_PATH

git clone https://github.com/SciRuby/nmatrix.git
cd nmatrix
bundle install

cd scripts
brew install wget
vim mac-brew-gcc 
# change the lastline from `make install` to `sudo make install`
sudo ./mac-brew-gcc
# 在13年中的air上编译了近50分钟

sudo ln -nfs /usr/gcc-4.7.2/bin/gcc-4.7.2 /usr/bin/gcc
sudo ln -nfs /usr/gcc-4.7.2/bin/g++-4.7.2 /usr/bin/g++

bundle exec rake compile
bundle exec rake repackage
gem install pkg/nmatrix-0.1.0.rc4.gem
``` 

接下来，就可以在ruby环境中，进行矩阵操作了

```ruby
a = NMatrix.new([3,3], 2) # 3x3 matrix containing all 2's

a + a       # Element-wise / Matrix addition.
a - a       # Element-wise / Matrix subtraction.
a ** 3      # Element-wise exponentiation.
a / 2       # Element-wise division.
a * a       # Element-wise multiplication.
a.dot a     # Matrix dot-product multiplication.
``` 

如果你想要在ruby下，学习机器学习，那么NMatrix是必须的了！
