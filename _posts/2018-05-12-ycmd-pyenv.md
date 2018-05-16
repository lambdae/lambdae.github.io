---
layout: post
title: 开发环境配置
description:  vim ycmd pyenv配置
categories: tech
author: lambdae
tags: [配置]
comments: true
---




* **python环境配置**

    pyenv安装
    ```C
    curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash
    ```

    创建虚拟环境
    ```C
    pyenv install 2.7.13   
    pyenv uninstall 2.7.13
    pyenv virtualenv 2.7.13 env2
    pyenv activate env2
    ```
    pip源配置
    ```C
    $ vim ~/.pip/pip.conf
    [global]
    index-url = http://mirrors.aliyun.com/pypi/simple/
    [install]
    trusted-host=mirrors.aliyun.com
    ```



*  **vim配置**

    YouCompleteMe
    ```C
    cd ~/.vim/bundle/YouCompleteMe
    ./install.py  --racer-completer  --clang-completer --system-libclang --go-completer
    mkdir -p _ycm && cd _ycm
    $CMAKE_DIR/bin/cmake -G "Unix Makefiles" -DEXTERNAL_LIBCLANG_PATH=$LIBCLANG . $YCM_DIR/third_party/ycmd/cpp && make
    ```

    vimrc配置
    ```C
    Bundle 'Valloric/YouCompleteMe'
    let g:ycm_rust_src_path = '~/.rustup/toolchains/stable-x86_64-apple-darwin/lib/rustlib/src/rust/src'
    let g:ycm_global_ycm_extra_conf = '~/.vim/bundle/YouCompleteMe/third_party/ycmd/cpp/ycm/.ycm_extra_conf.py'
    let g:EclimCompletionMethod = 'omnifunc'
    # 使用系统python, 隔离pyenv
    let g:ycm_path_to_python_interpreter = '/usr/local/bin/python'

    Bundle 'davidhalter/jedi-vim'
    Bundle 'fatih/vim-go'
    ```


