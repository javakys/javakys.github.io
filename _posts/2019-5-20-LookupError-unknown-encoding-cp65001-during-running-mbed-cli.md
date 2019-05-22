---
layout: post
title: LookupError unknown encoding cp65001 during running mbed cli 
date:   2019-05-06
author: James Kim
categories: W6100
---

mbed cli를 설치해서 실행할 때, "LookupError: unknown encoding: cp65001"이라는 error message가 표시되면서 프로그램 실행이 멈추는 현상이 있다. 

```shell
> pip --version
  Traceback (most recent call last):
    File "c:\python27\lib\runpy.py", line 174, in _run_module_as_main
    "__main__", fname, loader, pkg_name)
    File "c:\python27\lib\runpy.py", line 72, in _run_code
    exec code in run_globals
    File "C:\Python27\Scripts\pip.exe\__main__.py", line 5, in <module>
    File "c:\python27\lib\site-packages\pip\__init__.py", line 28, in <module>
    from pip.vcs import git, mercurial, subversion, bazaar  # noqa
    File "c:\python27\lib\site-packages\pip\vcs\mercurial.py", line 9, in <module>
    from pip.download import path_to_url
    File "c:\python27\lib\site-packages\pip\download.py", line 37, in <module>
    from pip.utils.ui import DownloadProgressBar, DownloadProgressSpinner
    File "c:\python27\lib\site-packages\pip\utils\ui.py", line 57, in <module>
    _BaseBar = _select_progress_class(IncrementalBar, Bar)
    File "c:\python27\lib\site-packages\pip\utils\ui.py", line 50, in _select_progress_class
    six.text_type().join(characters).encode(encoding)
LookupError: unknown encoding: cp65001
```

이때, 아래와 같이 실행하면 문제가 해결된다.

```shell
> set PYTHONIOENCODING=UTF-8
```