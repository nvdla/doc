NVDLA Open Source Project documentation
=======================================

If you just want to read the latest version of the NVDLA documentation, go
to http://nvdla.org/.

The NVDLA documentation is written in RestructuredText and built using
Sphinx.

Building documentation inside of NVIDIA
---------------------------------------

To build documentation on the NVIDIA farm, you need to first install Sphinx
in your working directory.  To do this:

 * Make sure that you have a version of git in your PATH --
   /home/utils/git-2.9.2/bin/ will do.

 * Set up the VirtualEnv.  `cd` into where you want Sphinx to be installed,
   and then run:
  
     $ /home/utils/Python-2.7.9/bin/virtualenv sphinx
   
   This will create a directory called 'sphinx' in your current directory. 
   (If you want the directory to be named something else, you can change it
   as the parameter to virtualenv.)

 * Install Sphinx in the virtualenv.  I set up my virtualenv in
   /home/scratch.jwise_t19x/sphinx, so I will run:

     $ /home/scratch.jwise_t19x/sphinx/bin/pip install sphinx

 * Sphinx should be installed now.  To set up your environment to run
   Sphinx, if you're using csh, you can run (changing the paths for your
   install):

     % source /home/scratch.jwise_t19x/sphinx/bin/activate.csh

   or, for bash users:

     $ . /home/scratch.jwise_t19x/sphinx/bin/activate

You should now be able to run build-html.sh inside of this "doc" directory.

Infrastructure
--------------

Travis-CI is set up to push to github-pages, as recommended in:

  https://blog.ionelmc.ro/2015/07/11/publishing-to-github-pages-from-travis-ci/

