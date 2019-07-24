# Installation on MS Windows

It is possible to install pyang tools to MS Windows using Cygwin.

1. Download Cygwin from http://cygwin.com/install.html. Choose 32-bit or 64-bit installer depending on your Windows version.

2. Install Cygwin. The installation path should not contain any spaces or other special characters. The following packages are needed for pyang to work, so choose them during installation process.
  * `bash`
  * `git`
  * `python`
  * `python-setuptools`
  * `libxml2`
  * `libxslt`
  
  In case you need to add new packages to your current Cygwin installation, just run the installer again.

3. Run the Cygwin terminal
4. Download pyang from github
  
  `git clone https://github.com/mbj4668/pyang.git`

5. Install pyang
  
  `cd pyang`  
  `python setup.py install`

6. You can test the installation by running `yang2dsdl -h` command. You should see the help of yang2dsdl tool.

7. Unfortunately, there is an issue in pyang setup, which probably does not expect installation inside Cygwin, see [#184](https://github.com/mbj4668/pyang/issues/184). Some of pyang's files are installed inside python library directory instead of the correct path.

  To workaroud this, you need to copy `share` directory inside `<CYGWIN_INSTALL>\lib\python2.7\site-packages\pyang-1.6-py2.7.egg\` into `<CYGWIN_INSTALL>\usr\local\` .
  
  Otherwise, you will see error like this when trying to process yang file:
  
  `warning: failed to load external entity "/usr/local/share/yang/xslt/basename.xsl"`

8. Now, it should be possible to test functionality on example yang file, e.g. in pyang repository cloned from GitHub:

  `cd doc/tutorial/examples`  
  `yang2dsdl turing-machine.yang`

  The tool should work without any errors.