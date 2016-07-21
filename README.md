# DevOps Playground 5 - Hands on Habitat

## Setting up the VM
You can find the VM from one of the USB keys which will be provided on the night.
​
To use the VM you will need to have VirtualBox installed, you can download load the correct version for your platform [here](https://www.virtualbox.org/wiki/Downloads) if you don't already have it installed.
​
The username / password for the VM is `ubuntu/ubuntu`

## Setting up Habitat
1. Run `sudo hab setup`
2. Follow these instructions, as illustrated here ![](http://i.imgur.com/jQoVosP.png)![](http://i.imgur.com/UWGUxbA.png)![](http://i.imgur.com/qGMU9jm.png):
  * Set up a default origin? [Yes/no/quit] y
  * Default origin name: [default: ubuntu] <yourname>_habitat
  * Create an origin key for `forest_habitat'? [Yes/no/quit] y
  * Set up a default GitHub access token? [Yes/no/quit] n
  * Enable analytics? [Yes/no/quit] n
  For the sake of this playground, we will skip uploading our package to the community, and won't enable analytics to save what we can in terms of network resources.

## Going through the official tutorial
This will quickly run through the simple tutorial provided on the Habitat website which will give you a feel for how the tool works and some of its capabilities. 

Plans are the blueprint to an application, it will describe what the application is and its associated dependencies. Add the following to the plan.
1. Create the directory structure.
```bash
mkdir -p plans/mytutorialapp
cd plans/mytutorialapp/
vi plan.sh
```
![](http://i.imgur.com/059iv4m.png)

2. Note, you will need to change the pkg_origin to the origin you created in the setup and the maintainer to your own email.
```
pkg_origin=<yourname>_habitat
pkg_name=mytutorialapp
pkg_version=0.1.0
pkg_maintainer="<youremail>"
pkg_license=()
pkg_source=https://s3-us-west-2.amazonaws.com/${pkg_name}/${pkg_name}-${pkg_version}.tar.gz
pkg_shasum=b54f8ada292b0249245385996221751f571e170162e0d464a26b958478cc9bfa
pkg_filename=${pkg_name}-${pkg_version}.tar.gz
pkg_deps=(core/node)
pkg_expose=(8080)
```
3. Then we will be setting up callbacks. Callbacks in a plan are simply overrides to existing functions that are called by the hab-plan-build script at build time.

```
do_build() {
  # The mytutorialapp source code is unpacked into a directory,
  # mytutorialapp-0.1.0, at the root of $HAB_CACHE_SRC_PATH. If you were downloading
  # an archive that didn't match your package name and version, you would have to
  # copy the files into $HAB_CACHE_SRC_PATH.

  # This installs both  npm as well as the nconf module we listed as a
  # dependency in package.json.
  npm install
}

do_install() {
  # Our source files were copied over to the HAB_CACHE_SRC_PATH in do_build(),
  # so now they need to be copied into the root directory of our package through
  # the pkg_prefix variable. This is so that we have the source files available
  # in the package.
  cp package.json ${pkg_prefix}
  cp server.js ${pkg_prefix}

  # Copy over the nconf module to the package that we installed in do_build().
  mkdir -p ${pkg_prefix}/node_modules/
  cp -vr node_modules/* ${pkg_prefix}/node_modules/
}
```

![](http://i.imgur.com/Omjzub2.png)

Here is a list of all the plan settings, callbacks, and runtime hooks that are available: https://www.habitat.sh/docs/reference/plan-syntax/

## Entering the studio

Now we need to enter the studio, this will download its dependencies and run in a chrooted shell environment.  
Run `sudo hab studio enter`

![](http://i.imgur.com/Mcg6pme.png)
![](http://i.imgur.com/CcKua4T.png)

The src directory maps to the ~/plans/mytutorialapp directory you were in before you entered the studio.

## Building

Run `build` and watch the magic happen.  

![](http://i.imgur.com/ebv4UJa.png)
![](http://i.imgur.com/3CaClci.png)

Now the application has been built and compiled but it's not running. 
Under the /src folder create a new folder called `hooks` and go inside: `mkdir hooks` followed by `cd hooks`. 

There, create the `init` file which will create symbolic links between the package and the npm server, allowing to deploy easily. Note that the shebangs are important for Habitat to decide which interpreter is going to be used.
```
#!/bin/sh
#
ln -sf {{pkg.path}}/package.json {{pkg.svc_path}}
ln -sf {{pkg.path}}/server.js {{pkg.svc_path}}
ln -sf {{pkg.path}}/node_modules {{pkg.svc_path}}
```
Create the `run` file which will be used to start npm.
```
#!/bin/sh
#
cd {{pkg.svc_path}}

# `exec` makes it so the process that the Habitat supervisor uses is
# `npm start`, rather than the run hook itself. `2>&1` makes it so both
# standard output and standard error go to the standard output stream,
# so all the logs from the application go to the same place.
exec npm start 2>&1
```

The next step is going into the /src directory, creating a `config` directory and creating a `config.json` file inside.
```
{
    "message": "{{cfg.message}}",
    "port": "{{cfg.port}}"
}
```

Now go back to the /src directory and create the `default.toml` file.
```
# Message of the Day
message = "Hello, World!"

# The port number that is listening for requests.
port = 8080
```

From the /src directory, run `build`.
![](http://i.imgur.com/hHw7375.png)
![](http://i.imgur.com/p2YlBrd.png)

The last step is to export the application in a Docker container.  
`hab pkg export docker <yourname>_habitat/mytutorialapp`

You now have a complete Docker container containing your application. Running the following commands will allow your to run the container and see your application.  
```
exit
ifconfig eth0
sudo docker run -it -p 8080:8080 <yourname>_habitat/mytutorialapp
``` 

Open your browser and go to the IP displayed by the ifconfig command on the port 8080.  
![](http://i.imgur.com/Q98U6cL.png)
![](http://i.imgur.com/bzQJuYQ.png)
